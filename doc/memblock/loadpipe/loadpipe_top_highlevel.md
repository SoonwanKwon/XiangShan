# XiangShan LoadPipe High-Level Overview

This document provides a high-level understanding of the XiangShan load pipeline. For detailed analysis, see [loadpipe_top.md](./loadpipe_top.md).

## Table of Contents
- [Pipeline Architecture](#pipeline-architecture)
- [Key Mechanisms](#key-mechanisms)
- [Important Queues](#important-queues)
- [Performance Characteristics](#performance-characteristics)
- [Design Trade-offs](#design-trade-offs)

---

## Pipeline Architecture

The LoadPipe is a **4-stage pipeline** (S0 → S1 → S2 → S3) that executes RISC-V load instructions with out-of-order execution support.

```
┌─────┐     ┌─────┐     ┌─────┐     ┌─────┐
│ S0  │────▶│ S1  │────▶│ S2  │────▶│ S3  │
└─────┘     └─────┘     └─────┘     └─────┘
   │                                    │
   │◀───────── Fast Replay ─────────────┤
   │           (1-2 cycles)             │
   │                                    │
   │◀─── Slow Replay via Replay Queue ─┤
        (3+ cycles, with blocking)
```

### Stage Functions

| Stage | Primary Function | Key Operations |
|-------|------------------|----------------|
| **S0** | Address Generation & Issue | Source selection (fresh issue, fast replay, slow replay), address calculation, LSQ/TLB/DCache issue |
| **S1** | TLB Translation & Forwarding | Virtual→Physical address translation, store-to-load forwarding query (LSQ, Sbuffer, Store Pipeline) |
| **S2** | DCache Response & Replay Decision | Cache hit/miss detection, bank conflict detection, data forwarding selection, replay cause determination |
| **S3** | Writeback & Hazard Detection | Result writeback to backend, CAM mismatch check, load-load violation detection, redirect generation |

### Pipeline Flow

**Normal L1 Hit Path (3-4 cycles):**
```
Cycle 0 (S0): Generate address, issue to DCache
Cycle 1 (S1): TLB translates vaddr→paddr, query forwarding sources
Cycle 2 (S2): DCache returns hit data, select final result
Cycle 3 (S3): Writeback to backend (RegFile)
```

**Why 3-4 cycles?** The 4th cycle occurs when DCache uses the `data_delayed` path (ECC correction, multi-bank collection, or routing delays).

---

## Key Mechanisms

### 1. Store-to-Load Forwarding

**Purpose:** Bypass data from stores that haven't reached DCache yet.

**Four Forwarding Sources (priority order):**
1. **LSQ (Load/Store Queue):** Youngest uncommitted stores
2. **Sbuffer (Store Buffer):** Committed stores waiting to retire to DCache
3. **D-channel:** Stores currently writing to DCache
4. **MSHR (Miss Status Holding Register):** Outstanding cache misses

**Priority:** LSQ > Sbuffer > D-channel > MSHR > DCache (S1 queries LSQ/Sbuffer, S2 selects result)

### 2. Replay Mechanisms

When a load cannot complete normally, it must **replay** (re-execute). There are three replay types:

#### Fast Replay (1-2 cycles)
- **Path:** S3 → S0 directly
- **Latency:** 1 cycle (best case), 2 cycles (if pipeline stalled)
- **Triggers:** Bank conflict, TLB miss (pf_ld path), cache miss (refill match)
- **No writeback on first attempt** (suppressed by `rep_info.need_rep`)

#### Slow Replay (3+ cycles)
- **Path:** S3 → Replay Queue → S0 (when blocking condition clears)
- **Latency:** 3+ cycles (depends on blocking cause)
- **Triggers:** 10 replay causes (data not ready, address conflict, RAW hazard, etc.)
- **Blocking conditions:** Each cause has specific blocking logic (e.g., wait for store retirement, refill completion)

#### Super Replay (Optimized)
- **Enhancement:** L2 cache hint wakes up blocked loads in Replay Queue
- **Benefit:** Reduces slow replay latency by ~10-20 cycles
- **Mechanism:** L2 cache sends hint when refill is imminent, Replay Queue preemptively issues load

### 3. Redirect (Rollback)

**Purpose:** Handle incorrect speculation or address translation issues.

**Two Redirect Types:**
- **flush:** Complete pipeline flush (e.g., CAM mismatch - vaddr/paddr inconsistency)
- **flushAfter:** Flush instructions after violating load (e.g., load-load violation from speculative reordering)

**Redirect Causes in LoadPipe:**
1. **CAM Mismatch:** Virtual address and physical address point to different cache lines (TLB/address inconsistency)
2. **Load-Load Violation:** Speculative load bypassed younger load that should have executed first (RAR Queue detects)

---

## Important Queues

### 1. Replay Queue (LoadQueueReplay)

**Purpose:** Hold loads that need slow replay until blocking conditions clear.

**Key Features:**
- **10 replay causes:** dcacheMiss, tlbMiss, forwardFail, bankConflict, dataInvalid, addrConflict, etc.
- **Blocking logic:** Each cause has specific blocking conditions (lines 310-338 in LoadQueueReplay.scala)
- **Wakeup:** Entries wake up when blocking condition clears or L2 hint arrives
- **Super replay:** L2 cache hint mechanism (lines 369-385)

**Structure:**
- Circular buffer with load queue index (lqIdx)
- Each entry tracks: uop, replay cause, strict flag, dataInvalid flag

### 2. RAR Queue (Load Queue Read-After-Read)

**Purpose:** Detect load-load violations (speculative reordering errors).

**Scenario:**
```
Younger Load A (speculative): ld x1, 0(x2)  // x2 unknown, speculates address 0x1000
Older Load B (executed):      ld x3, 0(x4)  // x4 = 0x1000 (same address!)
```
If Load A speculatively executed before Load B, but they access the same address, **Load B must redirect** to preserve memory ordering.

**Detection (S3):**
- CAM query: Compare S3 load's paddr against all younger loads in RAR queue
- If match found → `s3_rar_nuke` → Redirect (flushAfter)

### 3. RAW Queue (Load Queue Read-After-Write)

**Purpose:** Detect store-load violations (forwarding failures).

**Scenario:**
```
Store:  sw x1, 0(x2)  // Store to 0x1000 (address unknown at load time)
Load:   ld x3, 0(x4)  // Load from 0x1000 (executed before store address known)
```
If load executed before store's address was known, it might have missed forwarding.

**Detection (Store Address Calculation):**
- CAM query: When store calculates address, check against all younger loads in RAW queue
- If match found → Load must replay to get correct data

**Entry Fields:**
- allocated, uop, paddr, mask (byte enable), datavalid (from store)

### 4. Uncache Buffer

**Purpose:** Handle uncacheable/MMIO loads (serialized, single-flight).

**Features:**
- Only one uncache load in flight
- Bypasses DCache, goes directly to memory bus
- Blocks pipeline until completion

---

## Performance Characteristics

### Latency Summary

| Scenario | Total Latency | Details |
|----------|---------------|---------|
| **L1 DCache Hit** | 3-4 cycles | S0→S1→S2→S3 (4th cycle for data_delayed path) |
| **L1 DCache Hit (Forwarded)** | 3-4 cycles | LSQ/Sbuffer/D-channel forwarding in S1/S2 |
| **TLB Miss (L2 TLB Hit)** | ~10-15 cycles | L2 TLB lookup + fast replay + retry |
| **Bank Conflict (Fast Replay)** | 7-9 cycles | 3 cycles (fail) + 1-2 cycles (replay) + 3-4 cycles (retry) |
| **DCache Miss (L2 Hit)** | ~30-40 cycles | L2 cache latency + refill + retry |
| **DCache Miss (L3 Hit)** | ~60-80 cycles | L3 cache latency + refill + retry |
| **DCache Miss (Memory)** | ~150-200 cycles | Memory latency + refill + retry |
| **DCache Miss (Super Replay)** | L2/L3/Memory - 10~20 cycles | L2 hint wakes up load early |
| **Uncache/MMIO Load** | ~100-300 cycles | Device-dependent, serialized |

### Fast Replay Breakdown: Bank Conflict Example

**Total latency: 7-9 cycles**
- **First attempt:** 3 cycles (S0→S1→S2→S3, NO writeback due to `need_rep`)
- **Fast replay:** 1-2 cycles (S3→S0)
- **Second attempt:** 3-4 cycles (S0→S1→S2→S3, writeback with data)

**Penalty:** +4-5 cycles compared to normal L1 hit

**Key code (LoadUnit.scala:1060):**
```scala
s3_out.valid := s3_valid && !io.lsq.ldin.bits.rep_info.need_rep && !s3_in.mmio
```
Writeback is **suppressed** on first attempt when `need_rep = true`.

---

## Design Trade-offs

### 1. Fast Replay vs. Slow Replay

**Fast Replay Design:**
- **Pros:** Minimal latency (1-2 cycles), simple direct path (S3→S0)
- **Cons:** No blocking logic, can cause repeated replays (livelock risk)
- **Use cases:** Transient conditions (bank conflict, TLB miss with pf_ld, refill match)

**Slow Replay Design:**
- **Pros:** Blocking prevents wasted work, frees pipeline for other loads
- **Cons:** Higher latency (3+ cycles), requires queue management
- **Use cases:** Persistent conditions (cache miss, data not ready, RAW hazard)

**Trade-off:** Fast replay optimizes common transient cases, slow replay handles persistent blocking efficiently.

### 2. Replay Queue vs. Reservation Station Re-issue

**Why Replay Queue instead of re-issuing from reservation station?**

1. **Bypass pressure:** LSQ forwarding has limited CAM ports (8 for stores). Re-issuing all blocked loads would cause severe contention.
2. **Queue specialization:** Replay Queue has specific blocking logic per cause, more efficient than generic RS.
3. **Super replay:** L2 hint mechanism only works with Replay Queue (centralized wakeup).
4. **Out-of-order execution:** Loads can complete out-of-order, but must wait in Replay Queue without blocking younger loads.

**Trade-off:** Dedicated Replay Queue adds hardware but improves throughput and latency for blocked loads.

### 3. 4-Stage Pipeline Depth

**Why 4 stages (S0, S1, S2, S3)?**

1. **TLB latency:** S1 dedicated to TLB translation (multi-cycle operation)
2. **Forwarding complexity:** S1 queries forwarding sources, S2 selects result (bypassing adds latency)
3. **DCache timing:** S2 receives DCache response, needs full cycle for muxing and replay decision
4. **Hazard detection:** S3 performs CAM checks (RAR, address mismatch) after data available

**Trade-off:** Deeper pipeline increases load-to-use latency (minimum 3 cycles) but enables higher frequency and complex forwarding.

### 4. CAM Mismatch Detection (Redirect)

**Why check address consistency in S3?**

1. **TLB speculation:** S0 uses virtual address for cache access (VIPT), S1 gets physical address
2. **Synonym/Homonym:** Virtual address aliasing or TLB updates can cause vaddr ≠ paddr mapping change
3. **Correctness:** Must verify cache line accessed in S0/S1 matches paddr from TLB in S2/S3

**Cost:** Redirect (flush) on mismatch → ~20-30 cycle penalty (pipeline flush + refetch)

**Trade-off:** Rare mismatch penalty vs. always waiting for TLB in S0 (adds latency to every load).

### 5. Load-Load Violation Detection (RAR Queue)

**Why detect speculative load reordering?**

1. **Out-of-order execution:** Younger loads can execute before older loads (when older load address unknown)
2. **Memory ordering:** RISC-V TSO requires load-load ordering to same address
3. **Correctness:** If younger load speculatively executed first, older load must redirect on address match

**Cost:** RAR Queue hardware (CAM ports, storage), redirect penalty on violation

**Trade-off:** Enables aggressive speculation (younger loads don't wait for older loads) at cost of rare redirects.

### 6. Super Replay (L2 Hint)

**Why L2 hint mechanism?**

1. **DCache miss latency:** L2 hit takes ~30-40 cycles, loads blocked in Replay Queue waste cycles
2. **Wake-up timing:** L2 cache knows refill is imminent (~10-20 cycles before completion)
3. **Pipeline utilization:** Waking up load early preloads TLB/pipeline, reduces total latency

**Cost:** L2→L1 hint signal wiring, Replay Queue wakeup logic

**Trade-off:** Added complexity for ~10-20 cycle latency reduction on L2 hits (significant performance win).

---

## Key Takeaways

1. **LoadPipe is a 4-stage pipeline** optimized for low-latency load execution with extensive forwarding and replay mechanisms.

2. **Three replay types** handle different failure scenarios:
   - Fast replay (1-2 cycles): transient issues
   - Slow replay (3+ cycles): persistent blocking
   - Super replay: L2 hint optimization

3. **Four important queues** support out-of-order execution:
   - Replay Queue: blocked loads
   - RAR Queue: load-load violation detection
   - RAW Queue: store-load violation detection
   - Uncache Buffer: MMIO/uncacheable loads

4. **Store-to-load forwarding** has 4 sources (LSQ, Sbuffer, D-channel, MSHR) to minimize load latency when store data not yet in cache.

5. **Redirects** handle speculation errors (CAM mismatch, load-load violations) with 20-30 cycle penalty.

6. **Design prioritizes common-case performance** (L1 hits: 3-4 cycles) while handling complex corner cases (misses, conflicts, violations) with specialized mechanisms.

---

## Related Documentation

- **Detailed Analysis:** [loadpipe_top.md](./loadpipe_top.md) - Comprehensive 1400+ line analysis with code references
- **Redirect Deep Dive:** [loadpipe_redirect.md](./loadpipe_redirect.md) - Detailed redirect scenarios with sequence diagrams

## Code References

- **LoadUnit.scala** (`src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala`): Main load pipeline implementation
- **LoadQueueReplay.scala** (`src/main/scala/xiangshan/mem/lsqueue/LoadQueueReplay.scala`): Replay queue logic
- **LoadQueueRAR.scala** (`src/main/scala/xiangshan/mem/lsqueue/LoadQueueRAR.scala`): RAR queue for load-load violations
- **LoadQueueRAW.scala** (`src/main/scala/xiangshan/mem/lsqueue/LoadQueueRAW.scala`): RAW queue for store-load violations
- **LoadPipe.scala** (`src/main/scala/xiangshan/cache/dcache/loadpipe/LoadPipe.scala`): DCache-side load pipeline
