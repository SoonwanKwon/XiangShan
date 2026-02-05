# LoadQueueRAR (Read-After-Read Violation Checker)

## Overview

LoadQueueRAR tracks loads that **may need re-execution due to cache line release**. When multiple loads access the same cache line and the line gets evicted before all loads complete, younger loads may have read data that is no longer valid.

**File**: `src/main/scala/xiangshan/mem/lsqueue/LoadQueueRAR.scala`

---

## Key Concept: Why LoadQueueRAR Exists

In Out-of-Order execution with non-blocking cache:

```
Scenario:
  Load A (lqIdx = 5)  ← older, cache miss, waiting for refill
  Load B (lqIdx = 10) ← younger, same cacheline, cache hit (reads from cache)

Problem:
  1. Load B hits in cache, reads data, completes
  2. Cache line gets evicted (released) before Load A completes
  3. Load A eventually gets refill from memory (NEW data)
  4. But Load B already read OLD data from cache!
  5. This violates memory consistency!

Solution:
  - Load B registers in LoadQueueRAR (with paddr)
  - When cache release happens, check if released line matches any entry
  - If match found and older load still pending → Load B must re-execute (rep_frm_fetch)
```

**Key Difference from RAW**:
- **RAW**: Load vs older Store conflict (wrong data due to bypassed store)
- **RAR**: Load vs older Load conflict (stale data due to cache eviction)

---

## Queue Structure

```scala
// LoadQueueRAR.scala:61-73
val allocated = RegInit(VecInit(List.fill(LoadQueueRARSize)(false.B)))
val uop = Reg(Vec(LoadQueueRARSize, new MicroOp))
val paddrModule = Module(new LqPAddrModule(
  numEntries = LoadQueueRARSize,
  numRead = LoadPipelineWidth,
  numWrite = LoadPipelineWidth,
  numCamPort = LoadPipelineWidth  // ← For violation query
))
val released = RegInit(VecInit(List.fill(LoadQueueRARSize)(false.B)))
```

**Entry Fields**:
| Field | Purpose |
|-------|---------|
| `allocated` | Entry valid |
| `uop` | MicroOp (robIdx, lqIdx) |
| `paddr` | Physical address (for CAM search) |
| `released` | Cache line has been released (evicted) |

---

## Enqueue

### Timing: Load S2

Load S2 sends query to LoadQueueRAR after paddr is determined (post-TLB).

```scala
// LoadUnit.scala:873-877
io.lsq.ldld_nuke_query.req.valid := s2_valid && s2_can_query
io.lsq.ldld_nuke_query.req.bits.uop := s2_in.uop
io.lsq.ldld_nuke_query.req.bits.paddr := s2_in.paddr
io.lsq.ldld_nuke_query.req.bits.mask := s2_in.mask
io.lsq.ldld_nuke_query.req.bits.data_valid := Mux(s2_full_fwd || s2_fwd_data_valid, true.B, !s2_dcache_miss)
```

### Enqueue Condition

**Critical Code** (LoadQueueRAR.scala:96-102):
```scala
// LoadQueueRAR enqueue condition:
// There are still not completed load instructions before the current load instruction.
// (e.g. "not completed" means that load instruction get the data or exception).
val canEnqueue = io.query.map(_.req.valid)
val cancelEnqueue = io.query.map(_.req.bits.uop.robIdx.needFlush(io.redirect))
val hasNotWritebackedLoad = io.query.map(_.req.bits.uop.lqIdx).map(lqIdx => isAfter(lqIdx, io.ldWbPtr))
val needEnqueue = canEnqueue.zip(hasNotWritebackedLoad).zip(cancelEnqueue).map {
  case ((v, r), c) => v && r && !c
}
```

**Condition Breakdown**:

| Variable | Meaning |
|----------|---------|
| `ldWbPtr` | Last lqIdx that has completed writeback |
| `hasNotWritebackedLoad` | Current load's lqIdx is after ldWbPtr → older loads still pending |

**Enqueue only when `hasNotWritebackedLoad = true`**:

```
Example:
  ldWbPtr = 5 (loads 0~5 have completed)

  Load with lqIdx = 10 (older loads are 0~9):
    → isAfter(10, 5) = true
    → hasNotWritebackedLoad = true
    → "Loads 6~9 haven't completed yet → ENQUEUE!"

  Load with lqIdx = 3:
    → isAfter(3, 5) = false
    → hasNotWritebackedLoad = false
    → "All older loads completed → NO ENQUEUE needed"
```

### Enqueue Logic

```scala
// LoadQueueRAR.scala:122-146
when (needEnqueue(w) && enq.ready) {
  allocated(enqIndex) := true.B

  // Write paddr for CAM search
  paddrModule.io.wen(w) := true.B
  paddrModule.io.waddr(w) := enqIndex
  paddrModule.io.wdata(w) := enq.bits.paddr

  // Fill uop info
  uop(enqIndex) := enq.bits.uop

  // Check if cache line was just released (same cycle)
  released(enqIndex) :=
    enq.bits.data_valid &&
    (release2Cycle.valid &&
     enq.bits.paddr(PAddrBits-1, DCacheLineOffset) === release2Cycle.bits.paddr(...) ||
     release1Cycle.valid &&
     enq.bits.paddr(PAddrBits-1, DCacheLineOffset) === release1Cycle.bits.paddr(...))
}
```

**Note**: `released` flag is set immediately if cache release happens in the same cycle as enqueue.

### C_RAR Replay (Queue Full)

If LoadQueueRAR is full, enqueue fails → Load must replay later:

```scala
// LoadUnit.scala:816-817
val s2_rar_nack = io.lsq.ldld_nuke_query.req.valid &&
                  !io.lsq.ldld_nuke_query.req.ready  // Queue full!

// This triggers C_RAR replay cause
s2_out.rep_info.rar_nack := s2_rar_nack && s2_troublem
```

---

## Dequeue

### Condition 1: Normal Invalidation (Older Loads Completed)

**Timing**: When all older loads have completed writeback

```scala
// LoadQueueRAR.scala:155-165
// when the loads that "older than" current load were writebacked,
// current load will be released.
for (i <- 0 until LoadQueueRARSize) {
  val deqNotBlock = !isBefore(io.ldWbPtr, uop(i).lqIdx)

  when (allocated(i) && deqNotBlock) {
    allocated(i) := false.B
    freeMaskVec(i) := true.B
  }
}
```

**Meaning**: `ldWbPtr` has passed this load's `lqIdx` → All older loads completed → No more RAR risk → **Safe to release**

```
Example:
  Entry has lqIdx = 7
  ldWbPtr advances: 5 → 6 → 7 → 8

  When ldWbPtr = 8:
    → !isBefore(8, 7) = true
    → deqNotBlock = true
    → Entry deallocated (no violation possible anymore)
```

### Condition 2: Flush

**Timing**: When redirect signal indicates this load should be flushed

```scala
// LoadQueueRAR.scala:159-164
val needFlush = uop(i).robIdx.needFlush(io.redirect)

when (allocated(i) && needFlush) {
  allocated(i) := false.B
  freeMaskVec(i) := true.B
}
```

### Condition 3: Load S3 Revoke

**Timing**: Load S3 sends revoke signal

```scala
// LoadQueueRAR.scala:167-179
for ((revoke, w) <- io.query.map(_.revoke).zipWithIndex) {
  val revokeValid = revoke && lastCanAccept(w)
  val revokeIndex = lastAllocIndex(w)

  when (allocated(revokeIndex) && revokeValid) {
    allocated(revokeIndex) := false.B
    freeMaskVec(revokeIndex) := true.B
  }
}
```

---

## Cache Release Handling

### Released Flag Update

When a cache line is evicted (released), all matching entries get their `released` flag set:

```scala
// LoadQueueRAR.scala:210-223
val release1Cycle = io.release
val release2Cycle = RegNext(io.release)

when (release1Cycle.valid) {
  paddrModule.io.releaseMdata.takeRight(1)(0) := release1Cycle.bits.paddr
}

(0 until LoadQueueRARSize).map(i => {
  when (RegNext(paddrModule.io.releaseMmask.takeRight(1)(0)(i) && allocated(i) && release1Cycle.valid)) {
    // Note: if a load has missed in dcache and is waiting for refill in load queue,
    // its released flag still needs to be set as true if addr matches.
    released(i) := true.B
  }
})
```

**Meaning**: When cache evicts a line, mark all loads accessing that line as "released"

---

## Violation Detection (Query Response)

### Timing: Load S2 Query → Response in Next Cycle

When a new load queries LoadQueueRAR, it checks for violations:

```scala
// LoadQueueRAR.scala:183-207
// Load-to-Load violation check condition:
// 1. Physical address match by CAM port.
// 2. release is set.
// 3. Younger than current load instruction.

for ((query, w) <- io.query.zipWithIndex) {
  paddrModule.io.releaseViolationMdata(w) := query.req.bits.paddr

  query.resp.valid := RegNext(query.req.valid)

  // Generate real violation mask
  val robIdxMask = VecInit(uop.map(_.robIdx).map(isAfter(_, query.req.bits.uop.robIdx)))
  val matchMask = (0 until LoadQueueRARSize).map(i => {
    RegNext(allocated(i) &&
            paddrModule.io.releaseViolationMmask(w)(i) &&  // paddr match
            robIdxMask(i) &&                               // entry is OLDER
            released(i))                                    // cache line was released
  })

  // Load-to-Load violation check result
  val ldLdViolationMask = VecInit(matchMask)
  query.resp.bits.rep_frm_fetch := ParallelORR(ldLdViolationMask)
}
```

**Violation Conditions** (ALL must be true):
1. `allocated(i)` - Entry is valid
2. `paddr match` - Same cache line
3. `robIdxMask(i)` - Entry is for an OLDER load
4. `released(i)` - Cache line was evicted

**Result**: `rep_frm_fetch = true` → Current load must re-fetch from memory

---

## Complete Flow Example

```
T0: Load A dispatched (lqIdx = 5), cache miss, waiting for refill
    Load B dispatched (lqIdx = 10), same cacheline

T1: Load B in Load S1 (TLB translation)

T2: Load B in Load S2
    ├─► paddr = 0x2000 (cacheline 0x2000-0x203F)
    ├─► Check: ldWbPtr = 3, Load B's lqIdx = 10
    │   → isAfter(10, 3) = true
    │   → hasNotWritebackedLoad = true (loads 4~9 not completed)
    └─► Enqueue to LoadQueueRAR entry 8
        - paddr = 0x2000
        - lqIdx = 10
        - released = false

T3: Load B in Load S3 → Writeback (cache hit, got data)

T5: Cache eviction! Line 0x2000 released
    └─► LoadQueueRAR entry 8: released := true

T6: Load A still waiting for refill...

T8: Load C dispatched (lqIdx = 15), same cacheline 0x2000

T9: Load C in Load S2
    ├─► Query LoadQueueRAR
    ├─► CAM search: paddr 0x2000 matches entry 8
    ├─► Check entry 8:
    │   - allocated = true ✓
    │   - paddr match = true ✓
    │   - isAfter(Load C robIdx, entry 8 robIdx) = true (entry 8 is older) ✓
    │   - released = true ✓
    └─► resp.rep_frm_fetch = true!
        "Entry 8 (Load B) accessed same line, line was released, Load B is older"
        → Load C must re-fetch from memory, not cache!

T10: Load A finally gets refill, completes
     └─► ldWbPtr advances past lqIdx = 5

T15: ldWbPtr reaches 10
     └─► LoadQueueRAR entry 8: deqNotBlock = true → deallocated
```

---

## Summary Table

| Event | Timing | Condition | Result |
|-------|--------|-----------|--------|
| **Enqueue** | Load S2 | `hasNotWritebackedLoad` (older load not completed) | Entry allocated with paddr |
| **C_RAR Replay** | Load S2 | Queue full (`!req.ready`) | Load replays later |
| **Released Flag Set** | Cache release | `paddr` matches released cacheline | `released := true` |
| **Violation Query** | Load S2 (resp in S3) | paddr match + older + released | `rep_frm_fetch = true` |
| **Dequeue (normal)** | Continuous | `ldWbPtr` passes entry's lqIdx | Entry freed (safe) |
| **Dequeue (flush)** | On redirect | `robIdx.needFlush(redirect)` | Entry freed |
| **Dequeue (revoke)** | Load S3 | `revoke` signal from load | Entry freed |

---

## Key Insight

LoadQueueRAR ensures **cache coherence for load-load ordering**:

- **Enqueue**: "Older loads still pending, I might read stale data if cache evicts"
- **Released**: "Cache line was evicted while older load still pending"
- **Violation Query**: "Another load found: same line + older + released = I read stale data!"
- **Normal Dequeue**: "Older loads all completed, no more risk"

**Critical Difference from RAW**:
| Aspect | RAW | RAR |
|--------|-----|-----|
| **Conflict** | Load vs older Store | Load vs older Load |
| **Trigger** | Store executes | Cache line evicted |
| **Detection** | Store S1 CAM search | Load S2 query response |
| **Problem** | Bypassed store data | Stale cache data |

---

## Design Rationale: Why Separate Queue?

### Question: Why not use VirtualLoadQueue for RAR detection?

VirtualLoadQueue (80 entries) tracks all in-flight loads, but it **cannot** be used for RAR violation detection:

### Reason 1: VirtualLoadQueue Has NO paddr

```
VirtualLoadQueue stores:
  - uop (MicroOp): robIdx, lqIdx, sqIdx, etc.
  - ❌ NO paddr (physical address)

LoadQueueRAR stores:
  - uop (MicroOp)
  - ✅ paddr (for CAM search)
  - ✅ released flag (cache eviction tracking)
```

**Why this matters**:
- Cache release event comes with paddr (which cache line was evicted)
- Need to CAM search by paddr to find affected loads
- New loads query by paddr to check for violations
- Without paddr, these operations are impossible!

### Reason 2: Smaller CAM = Shorter Critical Path

```
If using VirtualLoadQueue for CAM:
  - 80 entries × LoadPipelineWidth CAM ports
  - Every cache release must search ALL 80 entries
  - Every new load must query ALL 80 entries
  - Long critical path, timing issues

With LoadQueueRAR (subset):
  - Only ~16-32 entries (loads with pending older loads)
  - Much smaller CAM, faster search
  - Better timing closure
```

### Comparison Table

| Aspect | VirtualLoadQueue | LoadQueueRAR |
|--------|------------------|--------------|
| **Size** | 80 entries | LoadQueueRARSize (smaller) |
| **paddr storage** | ❌ None | ✅ Yes (CAM module) |
| **released flag** | ❌ None | ✅ Yes |
| **CAM Search** | ❌ Impossible | ✅ Cache release / Load query |
| **Entry condition** | All issued loads | Only "at-risk" loads |
| **Purpose** | Commit ordering, replay tracking | RAR violation detection |

### Key Design Principle

```
"Track subset, not all"

All loads → VirtualLoadQueue (for commit)
     ↓ filter
Loads with pending older loads → LoadQueueRAR (for cache coherence)
```

This filtering reduces:
1. **Storage**: Only "at-risk" loads need paddr stored
2. **CAM size**: Smaller CAM = better timing
3. **Power**: Fewer entries to search per cache release event
