# CAM Mismatch (Vaddr/Paddr Inconsistency) - Detailed Explanation

## Overview

**CAM mismatch** is a correctness guard in the load pipeline. It detects cases where
the load **received forwarded data** based on a **speculative or early address match**,
but the **final address translation** (vaddr -> paddr) proves the match was incorrect.

When a CAM mismatch is detected, the load **must be re-fetched** (full flush) because:
- The forwarded data might be from the wrong physical address.
- The load might have written incorrect data into the architectural state.

This detection occurs in **S3** using:
```
s3_vp_match_fail = RegNext(io.lsq.forward.matchInvalid || io.sbuffer.matchInvalid) && s3_troublem
s3_rep_frm_fetch = s3_vp_match_fail
```
and triggers **RedirectLevel.flush** (full flush).

---

## What is Being Checked?

The load pipeline performs a **vaddr+paddr CAM check** after forwarding:

- The **forwarding decision** in S1/S2 may use:
  - vaddr match (virtual address)
  - paddr match (physical address)
  - mask overlap (byte enables)
- In S3, LSQ/SBuffer **re-check** that the forwarding source still matches the
  load's **final paddr**.

If the re-check fails, **matchInvalid** is asserted:
- `io.lsq.forward.matchInvalid`
- `io.sbuffer.matchInvalid`

This is **not** a replay; it is a **redirect/flush** because the load might have
already consumed incorrect data.

---

## Where Do LSQ/SBuffer vaddr/paddr Come From?

The **load’s** vaddr/paddr are known in S1/S2, but the **LSQ/SBuffer** compare
against **store entries**, each of which carries its own address state:

### Store Queue (LSQ forward)
- **vaddr**: captured when the store’s effective address is computed.
- **paddr**: captured when the store’s DTLB translation completes.
- **addrValidVec**: indicates which store entries have a valid paddr.
- The forward logic runs **two CAMs** (vaddr CAM and paddr CAM) on the store queue:
  - `vaddrModule.io.forwardMmask` vs `paddrModule.io.forwardMmask`
  - **matchInvalid** when these masks disagree for valid addresses:
    ```
    vpmaskNotEqual = (paddrMask ^ vaddrMask) & needForward & addrValidVec
    vaddrMatchFailed = vpmaskNotEqual && forward.valid
    io.forward.matchInvalid := vaddrMatchFailed
    ```

### Store Buffer (SBuffer forward)
- **vtag**: derived from store vaddr when entering the buffer.
- **ptag**: derived from store paddr when translation is available.
- Forwarding compares **load vaddr** and **load paddr** tags against stored vtag/ptag.
- **matchInvalid** when vtag match disagrees with ptag match for active entries:
  ```
  tag_mismatch = vtag_match != ptag_match  (for active/inflight entries)
  forward.matchInvalid := tag_mismatch
  ```

**Key point**: CAM mismatch is about **store-side address state** (vaddr vs paddr)
being inconsistent, not about the load’s own paddr being invalid.

---

## How Forwarding Actually Selects an Entry (and Where Mismatch Fits)

In XiangShan, **forward selection is driven by the vaddr CAM**, while the **paddr CAM is
used only as a consistency check**. This is a timing choice: vaddr CAM is closer and
faster, paddr CAM is used to validate that the vaddr match was correct.

### 1) Selection uses vaddr CAM (not both)
From `StoreQueue.scala`:
```
dataModule.io.needForward(i)(0) := canForward1 & vaddrModule.io.forwardMmask(i)
dataModule.io.needForward(i)(1) := canForward2 & vaddrModule.io.forwardMmask(i)
io.forward(i).forwardMask := dataModule.io.forwardMask(i)
io.forward(i).forwardData := dataModule.io.forwardData(i)
```
This means the **forwarded store entry is chosen by vaddr match**.

### 2) paddr CAM only checks sanity
```
vpmaskNotEqual = (paddrMask ^ vaddrMask) & needForward & addrValidVec
io.forward(i).matchInvalid := vpmaskNotEqual && forward.valid
```
If the **vaddr CAM mask ≠ paddr CAM mask**, the load might have forwarded from
the wrong physical line. In that case the pipeline **flushes**, rather than trying to
pick a different entry.

### 3) Same idea in SBuffer
SBuffer forwarding uses **vtag match** to select data, and **ptag match** only to
check consistency:
- vtag match = selection
- ptag match = validation
- mismatch → `forward.matchInvalid` → flush

**Conclusion:** both vaddr and paddr are compared, but **only vaddr decides the
forwarding entry**. paddr is used to **validate** that decision; disagreement triggers
a **full flush**.

---

## Where It Is Detected (Pipeline View)

- **S1**: Load gets TLB translation (paddr) and issues forwarding query.
- **S2**: Data comes back (cache or forward).
- **S3**: LSQ/SBuffer re-checks vaddr/paddr consistency.
  - If mismatch, `matchInvalid` is raised and `s3_rep_frm_fetch` triggers full flush.

---

## Typical Forwarding vs CAM Mismatch

### Normal Forward (No Mismatch)
```mermaid
sequenceDiagram
  participant L as Load
  participant T as DTLB
  participant LSQ as LSQ Forward
  participant SB as SBuffer

  L->>T: S1 translate vaddr -> paddr=P1
  L->>LSQ: S1 forward query (vaddr, paddr=P1, mask)
  LSQ-->>L: S2 forward data (match)
  SB-->>L: (optional) forward data
  LSQ-->>L: S3 CAM re-check OK
  L->>L: Commit/writeback
```

### CAM Mismatch (Forward Was Wrong)
```mermaid
sequenceDiagram
  participant L as Load
  participant T as DTLB
  participant LSQ as LSQ Forward
  participant S as Store Pipe

  L->>T: S1 translate vaddr -> paddr=P1
  L->>LSQ: S1 forward query (vaddr, paddr=P1)
  LSQ-->>L: S2 forward data (speculative/early match)
  S->>LSQ: Store addr resolves -> paddr=P2 (P2 != P1)
  LSQ-->>L: S3 CAM re-check fails -> matchInvalid=1
  L->>L: Redirect (flush) + re-fetch
```

---

## Why CAM Mismatch Happens (Expanded)

### 1) Store Address Arrives After the Load Has Already Queried Forwarding

This is a race condition common in out-of-order processors. A "fast" `LOAD` gets ahead of a "slow" older `STORE`.

#### The Problem: Speculative Forwarding with Incomplete Address Information

**Core Issue**: Forwarding uses **virtual address CAM** for performance, but this can be wrong if the store's **physical address isn't known yet**.

#### Step-by-Step Breakdown

Let's trace through what happens with concrete addresses:

**Initial State:**
- **Store** (older): `sw x1, 0x2000`  // Virtual address 0x2000, data = 0xABCD
  - Store is in LSQ, waiting for TLB translation
  - LSQ entry has: `vaddr = 0x2000`, `paddr = INVALID` (not ready yet)

- **Load** (younger): `lw x2, 0x2000`  // Same virtual address 0x2000
  - Load executes out-of-order (fast path)

**Timeline:**

**Cycle 1-2: Load in S0-S1 (Fast)**
```
Load S0: Calculate vaddr = 0x2000
Load S1: TLB translates quickly
  → vaddr 0x2000 → paddr 0x1000 (assume TLB hit, ASID=1, Page A)
```

**Cycle 2: Load Queries Forwarding in S1**
```
Load → LSQ: "Do you have data for vaddr=0x2000, paddr=0x1000?"

LSQ checks its entries:
  Entry[5]: Store instruction
    - vaddr = 0x2000 ✓ MATCH!
    - paddr = INVALID (TLB translation not done yet)
    - addrValid = FALSE
    - data = 0xABCD (data is ready)

LSQ Decision (SPECULATIVE):
  "vaddr matches (0x2000 == 0x2000), assume paddr will also match"
  → Forward data 0xABCD to load!
```

**Why does LSQ forward despite invalid paddr?**
- **Performance optimization**: Waiting for store's paddr would add latency
- **Common case**: Usually vaddr match → paddr match (same mapping)
- **Safety net**: Will verify later in S3

**Cycle 3: Load in S2 (Receives Forwarded Data)**
```
Load S2: Receives data = 0xABCD from LSQ
  → Load thinks it got correct data from store
  → Continues to S3
```

**Cycle 5-10: Store TLB Finally Completes (Slow)**
```
Store: TLB translation finally completes
  → vaddr 0x2000 → paddr 0x2000 (different! ASID=2, Page B)

LSQ updates entry:
  Entry[5]: Store
    - vaddr = 0x2000
    - paddr = 0x2000  ← NOW VALID!
    - addrValid = TRUE
    - data = 0xABCD
```

**Key Revelation**: Load's paddr (0x1000) ≠ Store's paddr (0x2000)!
- **Same virtual address** (0x2000) in different address spaces
- **Different physical addresses** (0x1000 vs 0x2000)
- **Different physical memory locations!**

**Cycle 11: Load in S3 (Validation Check)**
```
Load S3: LSQ re-validates forwarding decision

LSQ checks:
  vaddr CAM: Load(0x2000) vs Entry[5](0x2000) → MATCH
  paddr CAM: Load(0x1000) vs Entry[5](0x2000) → MISMATCH!

LSQ: "vaddr matched but paddr didn't → WRONG FORWARD!"
  → matchInvalid = TRUE
  → s3_vp_match_fail = TRUE

Load S3: Detects matchInvalid
  → Discard forwarded data 0xABCD (it was from wrong physical address!)
  → Trigger REDIRECT (flush pipeline)
  → Re-fetch load from scratch
```

#### Why This Happens: Address Space Disambiguation

**The root cause**: Same **virtual address** can map to different **physical addresses**:

1. **Different ASID (Address Space ID)**:
   ```
   Process A: vaddr 0x2000 → paddr 0x1000 (ASID=1)
   Process B: vaddr 0x2000 → paddr 0x2000 (ASID=2)
   ```
   Store and Load might be from different processes!

2. **TLB Synonym** (Multiple virtual addresses → same physical):
   ```
   vaddr 0x2000 → paddr 0x3000
   vaddr 0x4000 → paddr 0x3000  (alias)
   ```

3. **TLB Homonym** (Same virtual → different physical):
   ```
   Context 1: vaddr 0x2000 → paddr 0x1000
   Context 2: vaddr 0x2000 → paddr 0x5000
   ```

4. **Page Remapping** (TLB update between load and store):
   ```
   Time T1: vaddr 0x2000 → paddr 0x1000
   [TLB invalidation + update]
   Time T2: vaddr 0x2000 → paddr 0x2000  (remapped!)
   ```

#### Why Forwarding Used vaddr (Not paddr)

**Design choice**: Use **vaddr CAM** for forwarding selection because:
- **Timing**: vaddr available earlier than paddr
- **Performance**: Can't wait for all stores' TLB translations
- **Speculative**: Assume vaddr match → paddr match (usually true)
- **Safety**: Verify with paddr CAM in S3, flush if wrong

#### The Mismatch Detection

```
vaddr CAM result:  [Entry 5 matches]  ← Used for forwarding
paddr CAM result:  [Entry 5 no match] ← Used for validation

XOR these results: MISMATCH!
  → Load got data based on vaddr match
  → But paddr proves this was wrong
  → Must flush and retry
```

**Effect**: The load proceeded with **speculative data** (0xABCD from paddr 0x2000) that is now proven to be from the **wrong physical source** (should have been from paddr 0x1000, not 0x2000), requiring a full redirect.

```mermaid
sequenceDiagram
    participant L as Load Pipe
    participant TLB_L as Load TLB
    participant LSQ as LSQ / Store Queue
    participant S as Store Pipe
    participant TLB_S as Store TLB (Slow)

    Note over L,S: Program Order: Store (older) → Load (younger)<br/>Both access vaddr 0x2000

    rect rgb(255, 240, 220)
        Note over S,TLB_S: Cycle 0-1: Store issued but slow

        S->>S: Store S0: vaddr = 0x2000 calculated
        S->>TLB_S: Store S1: TLB query for vaddr 0x2000
        TLB_S-->>S: TLB MISS! (page walk required)
        S->>LSQ: Store enqueued in LSQ<br/>Entry[5]: vaddr=0x2000, paddr=INVALID, data=0xABCD
    end

    rect rgb(220, 255, 220)
        Note over L,TLB_L: Cycle 2-3: Load fast, out-of-order

        L->>L: Load S0: vaddr = 0x2000 calculated
        L->>TLB_L: Load S1: TLB query for vaddr 0x2000
        TLB_L-->>L: TLB HIT! → paddr = 0x1000 (ASID=1, Page A)

        L->>LSQ: Load S1: Forward query<br/>(vaddr=0x2000, paddr=0x1000, mask=0xFF)

        LSQ->>LSQ: vaddr CAM: Entry[5] vaddr(0x2000) == Load vaddr(0x2000) ✓
        LSQ->>LSQ: paddr CAM: Entry[5] paddr(INVALID) - can't check yet!
        LSQ->>LSQ: SPECULATIVE DECISION:<br/>"vaddr matches, assume paddr will too"

        LSQ-->>L: Load S2: Forward data = 0xABCD ✓ (speculative)

        L->>L: Load S2: Receives 0xABCD, proceeds
    end

    Note over TLB_S: ... Cycles 4-10: Page walk ...

    rect rgb(255, 220, 220)
        Note over TLB_S,S: Cycle 11: Store TLB completes (LATE!)

        TLB_S-->>S: Page walk done!<br/>vaddr 0x2000 → paddr 0x2000 (ASID=2, Page B)
        S->>LSQ: Update Entry[5]:<br/>paddr = 0x2000, addrValid = TRUE
    end

    rect rgb(255, 200, 200)
        Note over L,LSQ: Cycle 12: Load S3 - Validation Check

        L->>LSQ: Load S3: Re-validate forwarding

        LSQ->>LSQ: vaddr CAM:<br/>Entry[5] vaddr(0x2000) == Load vaddr(0x2000) ✓
        LSQ->>LSQ: paddr CAM:<br/>Entry[5] paddr(0x2000) != Load paddr(0x1000) ✗

        LSQ->>LSQ: ❌ MISMATCH DETECTED!<br/>vaddr matched, paddr didn't!

        LSQ-->>L: matchInvalid = TRUE ❌

        L->>L: s3_vp_match_fail = TRUE
        L->>L: REDIRECT (level=flush)
        L->>L: Discard wrong data 0xABCD!<br/>Flush pipeline, refetch from FTQ

        Note over L: Load got data from paddr 0x2000<br/>but should have read from paddr 0x1000!<br/>Different physical memory!
    end
```

#### Concrete Example: Two Processes, Same Virtual Address

**Scenario**: Process A and Process B both use virtual address 0x2000 for their stack:

```
Memory Layout:
┌─────────────────────────────────────┐
│ Physical Memory                      │
├─────────────────────────────────────┤
│ 0x1000: Process A's stack page      │
│         Contains: 0x1111             │  ← Load should read from here
├─────────────────────────────────────┤
│ 0x2000: Process B's stack page      │
│         Contains: 0xABCD             │  ← Store writes here
└─────────────────────────────────────┘

TLB Mappings:
Process A (ASID=1): vaddr 0x2000 → paddr 0x1000
Process B (ASID=2): vaddr 0x2000 → paddr 0x2000
```

**What Happens**:
1. **Store** (Process B, older):
   - `sw x1, 0(sp)` where sp = 0x2000
   - Wants to write 0xABCD to vaddr 0x2000
   - **Slow path**: TLB miss, page walk in progress
   - LSQ entry: vaddr=0x2000, paddr=INVALID, data=0xABCD

2. **Context Switch** happens (Process B → Process A)

3. **Load** (Process A, younger, out-of-order):
   - `lw x2, 0(sp)` where sp = 0x2000
   - Wants to read from vaddr 0x2000
   - **Fast path**: TLB hit! (Process A's mapping cached)
   - TLB: vaddr 0x2000 → paddr 0x1000 (Process A's stack)

4. **Forwarding Query** (Load S1):
   - Load asks LSQ: "Do you have data for vaddr=0x2000?"
   - LSQ: "Yes! Entry[5] has vaddr=0x2000, data=0xABCD"
   - LSQ forwards 0xABCD (from Process B's store!)

5. **Store TLB Completes** (later):
   - Store's TLB: vaddr 0x2000 → paddr 0x2000 (Process B's stack)
   - LSQ updates: Entry[5] paddr = 0x2000

6. **Load S3 Validation**:
   - vaddr CAM: Load(0x2000) vs Entry[5](0x2000) → MATCH ✓
   - paddr CAM: Load(0x1000) vs Entry[5](0x2000) → **MISMATCH!** ✗
   - **Violation**: Load got data from wrong process!

**The Bug Without CAM Check**:
- Load would have gotten 0xABCD (Process B's data)
- But Load wanted data from paddr 0x1000 (Process A's data, which is 0x1111)
- **Security violation**: Process A reading Process B's memory!
- **Correctness violation**: Wrong data!

**CAM Mismatch Saves The Day**:
- Detects paddr mismatch in S3
- Flushes load, forces retry
- On retry, store's paddr is known, forwarding check properly fails
- Load reads from DCache at paddr 0x1000 → Gets correct 0x1111 ✓

#### Visual Summary: What Went Wrong

```
Step 1: Store enqueued (paddr unknown)
┌─────────────────────────────────────┐
│ LSQ Entry[5] (Store)                │
│  vaddr:  0x2000  ✓                  │
│  paddr:  INVALID ✗                  │
│  data:   0xABCD                     │
└─────────────────────────────────────┘

Step 2: Load queries (vaddr match!)
┌─────────────────────────────────────┐
│ Load S1                             │
│  vaddr:  0x2000  ← matches Entry[5] │
│  paddr:  0x1000                     │
└─────────────────────────────────────┘
          ↓
    vaddr CAM: MATCH!
          ↓
    Forward 0xABCD to load (WRONG!)

Step 3: Store paddr arrives (too late!)
┌─────────────────────────────────────┐
│ LSQ Entry[5] (Store)                │
│  vaddr:  0x2000                     │
│  paddr:  0x2000  ← NOW KNOWN!       │
│  data:   0xABCD                     │
└─────────────────────────────────────┘

Step 4: Load S3 validation (catch error!)
┌─────────────────────────────────────┐
│ vaddr CAM: 0x2000 == 0x2000  ✓     │
│ paddr CAM: 0x1000 != 0x2000  ✗     │
│                                     │
│ → MISMATCH! matchInvalid = TRUE    │
│ → FLUSH pipeline                    │
└─────────────────────────────────────┘

Physical Memory (what should have happened):
┌──────────────────────────────────┐
│ paddr 0x1000:  0x1111           │ ← Load should read this
│ paddr 0x2000:  0xABCD           │ ← Store wrote this
└──────────────────────────────────┘
       ↑                  ↑
   Different physical addresses!
   Load got wrong data (0xABCD instead of 0x1111)
```

**Key Insight**: The forwarding was based on **vaddr match** (0x2000 == 0x2000), but the **paddr mismatch** (0x1000 != 0x2000) proves they are accessing **different physical memory locations**. Without the CAM mismatch check, the load would have incorrectly used the store's data from a different physical address!

### 2) TLB mapping changes between load and store translations
**When**: Load sees P1, store later sees P2 due to mapping change (TLB invalidation,
context switch, alias/homonym).
**Effect**: Forward match based on P1 is invalid -> mismatch.

### 3) Alias / homonym (same vaddr, different paddr)
**When**: Two different physical pages share the same virtual address in different
contexts (or due to synonym/homonym behavior).
**Effect**: vaddr-based forward match is invalid -> mismatch.

---

## Detailed Timing (Cycle-Level View)

```mermaid
sequenceDiagram
  participant L as Load Pipe
  participant LSQ as LSQ/SBuffer
  participant T as TLB

  Note over L: Cycle N: S1 (TLB translate)
  L->>T: vaddr -> paddr=P1
  L->>LSQ: forward query (vaddr, paddr=P1, mask)

  Note over L: Cycle N+1: S2 (data return)
  LSQ-->>L: forward data (speculative)

  Note over L: Cycle N+2: S3 (CAM re-check)
  LSQ-->>L: matchInvalid=1 (paddr mismatch)
  L->>L: redirect (flush)
```

---

## Signals and Control Flow

**Detection**
```
val s3_vp_match_fail = RegNext(io.lsq.forward.matchInvalid || io.sbuffer.matchInvalid) && s3_troublem
val s3_rep_frm_fetch = s3_vp_match_fail
```

**Redirect**
```
io.rollback.valid := s3_valid && (s3_rep_frm_fetch || s3_flushPipe) && !s3_exception
io.rollback.bits.level := Mux(s3_rep_frm_fetch, RedirectLevel.flush, RedirectLevel.flushAfter)
```

**Result**
- Load itself is flushed.
- All younger instructions are flushed.
- Frontend re-fetches from the load’s PC.

---

## Debug/Bring-up Checklist

If CAM mismatch is frequent:
- Check `io.lsq.forward.matchInvalid` / `io.sbuffer.matchInvalid` rate.
- Check TLB invalidation timing vs in-flight loads.
- Check store address readiness and forwarding timing.
- Monitor `s3_rep_frm_fetch` and `io.rollback` frequency.

---

## Related Docs

- `doc/memblock/loadpipe_redirect.md` (CAM mismatch redirect)
- `doc/memblock/loadpipe_S2.md` (forwarding + hazards)
- `doc/memblock/z_detail_s1_nuke.md` (store-load violation handling)
---

## Real-World Scenarios for TLB Mapping Changes

These scenarios illustrate how interactions with the OS or supervisor-level fences can cause a vaddr-to-paddr mapping to change between a load's translation and a store's translation, leading to a CAM mismatch.

### Scenario 1: `sfence.vma` after Page Table Modification

The `sfence.vma` (fence virtual memory address) instruction invalidates TLB entries. It is used by the OS after it modifies page tables to ensure subsequent memory accesses see the new mappings.

**Timeline:**
1.  **Load Executes**: A younger `LOAD` executes out-of-order. It gets a translation for virtual address `V` from the TLB: `V -> P1`.
2.  **Speculative Forward**: It queries the LSQ with `(V, P1)` and is speculatively forwarded data from an older `STORE` to the same vaddr `V` whose own paddr is not yet known.
3.  **OS Modifies Pages**: In supervisor mode (e.g., on another core, or after an interrupt), the OS changes the page table entry for `V` to point to a new physical page, `P2`.
4.  **TLB Flush**: The OS executes `sfence.vma V` to invalidate the old `V -> P1` translation from all hardware TLBs.
5.  **Store Translates (Late)**: The older `STORE` instruction finally executes its address translation for `V`. It misses the TLB (which was flushed) and the hardware page walker reads the newly modified page tables, returning the new translation: `V -> P2`.
6.  **Mismatch**: The `LOAD` reaches its S3 validation stage. Its original translation (`P1`) is compared against the `STORE`'s final translation (`P2`). The mismatch (`P1 != P2`) is detected, and a pipeline flush is correctly triggered.

```mermaid
sequenceDiagram
    participant Core0 as Core 0 (User)
    participant LSQ as LSQ
    participant TLB as TLB
    participant Core1 as Core 1 (Supervisor)
    participant PageTable as Page Table

    Core0->>TLB: 1. Load executes, translates V -> P1
    Core0->>LSQ: 2. Load speculatively forwards from Store (paddr unknown)
    
    Core1->>PageTable: 3. OS remaps page: V -> P2
    Core1->>Core0: 4. OS executes sfence.vma V
    Core0->>TLB: 4a. TLB entry for V is invalidated

    Core0->>TLB: 5. Store (late) translates V, TLB miss!
    TLB->>PageTable: 5a. Page walk
    PageTable-->>TLB: 5b. Gets new mapping: V -> P2
    
    Core0->>LSQ: 6. Load S3 check: Load(P1) != Store(P2)
    LSQ-->>Core0: 6a. MISMATCH! -> Redirect
```

### Scenario 2: Copy-on-Write (CoW) Page Fault

Copy-on-Write is an OS optimization where a parent and child process initially share the same physical memory pages, marked as read-only. A write attempt by either process triggers a page fault, causing the OS to create a private, writable copy of the page.

**Timeline:**
1.  **Initial State**: A `LOAD` and an older `STORE` from the same process are in the pipeline. They both target virtual address `V`, which currently maps to a shared, read-only physical page `P_shared`.
2.  **Load Executes**: The younger `LOAD` executes first. It translates `V -> P_shared` and is speculatively forwarded data from the `STORE`.
3.  **Store Causes Page Fault**: The older `STORE` attempts to execute. Since it's a write to a read-only page (`P_shared`), it triggers a page fault exception.
4.  **OS Handler**: The CPU traps into the OS page fault handler.
5.  **Create Private Page**: The OS allocates a new physical page, `P_private`, copies the data from `P_shared` to it, and updates the process's page table to map `V -> P_private` with write permissions.
6.  **Store Re-Executes**: The OS returns from the exception, and the `STORE` instruction is re-executed. This time, its translation succeeds: `V -> P_private`.
7.  **Mismatch**: The `LOAD` instruction, which has been waiting, reaches S3. Its validation check compares its original translation (`P_shared`) with the `STORE`'s final translation (`P_private`). The addresses do not match, a CAM mismatch is signaled, and the pipeline is flushed.

```mermaid
sequenceDiagram
    participant CPU as CPU Pipeline
    participant LSQ as LSQ
    participant TLB as TLB
    participant OS as OS Page Fault Handler
    participant PageTable as Page Table

    CPU->>TLB: 1. Load translates V -> P_shared (Read-Only)
    CPU->>LSQ: 2. Load gets speculative forward from Store
    
    CPU->>TLB: 3. Store attempts to write V
    TLB-->>CPU: 3a. Page Fault! (Write to R/O page)

    CPU->>OS: 4. Trap to OS
    OS->>OS: 5. Allocate P_private, copy data
    OS->>PageTable: 5a. Update mapping: V -> P_private (Writable)
    OS-->>CPU: 6. Return from exception

    CPU->>TLB: 6a. Store re-executes, translates V -> P_private
    
    CPU->>LSQ: 7. Load S3 check: Load(P_shared) != Store(P_private)
    LSQ-->>CPU: 8. MISMATCH! -> Redirect
```

### Scenario 3: Context Switch

If a context switch occurs between a load's translation and its store-dependency's translation, the active address space (defined by the `satp` register in RISC-V) can change entirely.

**Timeline:**
1.  **Load in Process A**: A `LOAD` in Process A (ASID 10) executes, translating `V -> P_A` using Process A's page table. It is speculatively forwarded data from a pending `STORE`.
2.  **Interrupt**: A timer interrupt occurs.
3.  **Context Switch**: The OS saves Process A's context and switches to Process B (ASID 20), updating the `satp` register to point to Process B's page tables.
4.  **Store in Process A Translates (Very Late)**: The `STORE` from Process A, which was stalled, finally attempts its translation. Because the hardware context now belongs to Process B, its translation for `V` would either be invalid or map to a completely different physical address `P_B` (if `V` is also valid in Process B).
5.  **Mismatch**: When the `LOAD` (from Process A) is eventually validated, its paddr `P_A` will not match the store's resolved paddr `P_B`.

*Note: In a realistic implementation, an interrupt would likely cause a pipeline flush anyway, but the CAM mismatch provides a crucial correctness backstop if any instructions survive the flush or if the address-space-change event happens through other means.*
