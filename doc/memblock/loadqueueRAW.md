# LoadQueueRAW (Read-After-Write Violation Checker)

## Overview

LoadQueueRAW tracks loads that **may have violated store-load ordering** due to Out-of-Order execution. When a load executes before an older store's address is known, it cannot determine if there's a conflict. LoadQueueRAW holds such loads and checks for violations when the older stores eventually execute.

**File**: `src/main/scala/xiangshan/mem/lsqueue/LoadQueueRAW.scala`

---

## Key Concept: Why LoadQueueRAW Exists

In Out-of-Order execution:
```
Program Order:
  Store A (sqIdx = 10)  ← older, address unknown (not executed yet)
  Load B  (lqIdx = 5)   ← younger, executing now in Load S2

Problem:
  - Load B executes and reads data from memory/cache
  - But Store A hasn't executed yet → Load B doesn't know Store A's address
  - If Store A and Load B access the same address → Load B read WRONG data!
  - This is a RAW (Read-After-Write) violation

Solution:
  - Load B registers itself in LoadQueueRAW (with its paddr)
  - When Store A executes (Store S1), CAM search LoadQueueRAW
  - If conflict found → Rollback (Load B must re-execute)
```

---

## Queue Structure

```scala
// LoadQueueRAW.scala:67-89
val allocated = RegInit(VecInit(List.fill(LoadQueueRAWSize)(false.B)))
val uop = Reg(Vec(LoadQueueRAWSize, new MicroOp))
val paddrModule = Module(new LqPAddrModule(
  numEntries = LoadQueueRAWSize,
  numCamPort = StorePipelineWidth  // ← Store S1 CAM search
))
val maskModule = Module(new LqMaskModule(...))
val datavalid = RegInit(VecInit(List.fill(LoadQueueRAWSize)(false.B)))
```

**Entry Fields**:
| Field | Purpose |
|-------|---------|
| `allocated` | Entry valid |
| `uop` | MicroOp (robIdx, sqIdx, lqIdx) |
| `paddr` | Physical address (for CAM search) |
| `mask` | Data mask |
| `datavalid` | Load data is valid |

---

## Enqueue

### Timing: Load S2

Load S2 sends query to LoadQueueRAW after paddr is determined (post-TLB).

```scala
// LoadUnit.scala:880-884
io.lsq.stld_nuke_query.req.valid := s2_valid && s2_can_query
io.lsq.stld_nuke_query.req.bits.uop := s2_in.uop
io.lsq.stld_nuke_query.req.bits.paddr := s2_in.paddr
io.lsq.stld_nuke_query.req.bits.mask := s2_in.mask
io.lsq.stld_nuke_query.req.bits.data_valid := ...
```

### Enqueue Condition

**Critical Code** (LoadQueueRAW.scala:108-112):
```scala
val allAddrCheck = io.stIssuePtr === io.stAddrReadySqPtr
val hasAddrInvalidStore = io.query.map(_.req.bits.uop.sqIdx).map(sqIdx => {
  Mux(!allAddrCheck, isBefore(io.stAddrReadySqPtr, sqIdx), false.B)
})
val needEnqueue = canEnqueue.zip(hasAddrInvalidStore).zip(cancelEnqueue).map {
  case ((v, r), c) => v && r && !c
}
```

**Condition Breakdown**:

| Variable | Meaning |
|----------|---------|
| `stAddrReadySqPtr` | Last sqPtr whose store address is executed |
| `stIssuePtr` | Last sqPtr that has been issued |
| `allAddrCheck` | All issued stores have executed their addresses |
| `hasAddrInvalidStore` | There exists an older store whose address is NOT yet executed |

**Enqueue only when `hasAddrInvalidStore = true`**:

```
Example:
  stAddrReadySqPtr = 5  (stores 0~5 address executed)
  stIssuePtr = 10       (stores 0~10 issued)

  Load with sqIdx = 8 (older stores are 0~7):
    → isBefore(5, 8) = true
    → hasAddrInvalidStore = true
    → "Stores 6,7 haven't executed address yet → ENQUEUE!"

  Load with sqIdx = 3 (older stores are 0~2):
    → isBefore(5, 3) = false
    → hasAddrInvalidStore = false
    → "All older stores executed → NO ENQUEUE needed"
```

### Enqueue Logic

```scala
// LoadQueueRAW.scala:121-162
when (needEnqueue(w) && enq.ready) {
  allocated(enqIndex) := true.B

  // Write paddr for CAM search
  paddrModule.io.wen(w) := true.B
  paddrModule.io.waddr(w) := enqIndex
  paddrModule.io.wdata(w) := enq.bits.paddr

  // Write mask
  maskModule.io.wen(w) := true.B
  maskModule.io.waddr(w) := enqIndex
  maskModule.io.wdata(w) := enq.bits.mask

  // Fill uop info
  uop(enqIndex) := enq.bits.uop
  datavalid(enqIndex) := enq.bits.data_valid
}
```

### C_RAW Replay (Queue Full)

If LoadQueueRAW is full, enqueue fails → Load must replay later:

```scala
// LoadUnit.scala:819-820
val s2_raw_nack = io.lsq.stld_nuke_query.req.valid &&
                  !io.lsq.stld_nuke_query.req.ready  // Queue full!

// This triggers C_RAW replay cause
s2_out.rep_info.raw_nack := s2_raw_nack && s2_troublem
```

---

## Dequeue

### Condition 1: Normal Invalidation (No Violation)

**Timing**: When all older stores have executed their addresses

```scala
// LoadQueueRAW.scala:176-186
// when the stores that "older than" current load address were ready,
// current load will be released.
for (i <- 0 until LoadQueueRAWSize) {
  val deqNotBlock = Mux(!allAddrCheck, !isBefore(io.stAddrReadySqPtr, uop(i).sqIdx), true.B)

  when (allocated(i) && deqNotBlock) {
    allocated(i) := false.B
    freeMaskVec(i) := true.B
  }
}
```

**Meaning**: `stAddrReadySqPtr` has passed this load's `sqIdx` → All older stores executed → No conflict detected → **Safe to release**

```
Example:
  Entry has sqIdx = 7
  stAddrReadySqPtr advances: 5 → 6 → 7 → 8

  When stAddrReadySqPtr = 8:
    → !isBefore(8, 7) = true
    → deqNotBlock = true
    → Entry deallocated (no violation occurred)
```

### Condition 2: Rollback/Flush (Violation or Other Flush)

**Timing**: When redirect signal indicates this load should be flushed

```scala
// LoadQueueRAW.scala:180-185
val needCancel = uop(i).robIdx.needFlush(io.redirect)

when (allocated(i) && needCancel) {
  allocated(i) := false.B
  freeMaskVec(i) := true.B
}
```

**Meaning**:
- RAW violation detected → rollback triggered → redirect signal
- Or other flush event (branch mispredict, exception, etc.)
- Entry cancelled

### Condition 3: Load S3 Revoke

**Timing**: Load S3 sends revoke signal

```scala
// LoadQueueRAW.scala:188-200
for ((revoke, w) <- io.query.map(_.revoke).zipWithIndex) {
  val revokeValid = revoke && lastCanAccept(w)
  val revokeIndex = lastAllocIndex(w)

  when (allocated(revokeIndex) && revokeValid) {
    allocated(revokeIndex) := false.B
    freeMaskVec(revokeIndex) := true.B
  }
}
```

**Meaning**: Load was enqueued but needs replay for other reasons (not RAW) → revoke the entry

---

## Violation Detection (Store S1 CAM Search)

### Timing: Store S1

When a store executes in S1 and gets its paddr, it searches LoadQueueRAW for conflicts:

```scala
// LoadQueueRAW.scala:43, 205-234
val storeIn = Vec(StorePipelineWidth, Flipped(Valid(new LsPipelineBundle)))
val rollback = Output(Valid(new Redirect))

// Store-Load Memory violation detection
// When store writes back, it searches LoadQueueRAW for younger load instructions
// with the same load physical address. They loaded wrong data and need re-execution.
```

### CAM Search Flow

```
Store S1 executes:
  ├─► paddr = 0x1000, mask = 0xFF
  │
  └─► CAM search LoadQueueRAW
      ├─► Entry 3: paddr = 0x1000, mask = 0x0F, robIdx = 50
      │   → Address match! Mask overlap!
      │   → This load is YOUNGER than store (check robIdx)
      │   → VIOLATION DETECTED!
      │
      └─► Select oldest violating load → Rollback
```

### Rollback Generation

```scala
// LoadQueueRAW.scala:236-289 (selectOldest logic)
// Multi-cycle selection to find oldest violating load
// Then generate rollback request
io.rollback.valid := violation_detected
io.rollback.bits := redirect_to_violating_load
```

---

## Complete Flow Example

```
T0: Load B dispatched (sqIdx boundary = 10, meaning older stores are 0~9)
    Store A (sqIdx = 7) not yet executed

T1: Load B in Load S1 (TLB translation)

T2: Load B in Load S2
    ├─► paddr = 0x2000 determined
    ├─► Check: stAddrReadySqPtr = 5, Load's sqIdx boundary = 10
    │   → isBefore(5, 10) = true
    │   → hasAddrInvalidStore = true (stores 6~9 not executed)
    └─► Enqueue to LoadQueueRAW entry 8
        - paddr = 0x2000
        - sqIdx = 10
        - robIdx = 100

T3: Load B in Load S3 → Writeback (speculatively correct)

T5: Store A (sqIdx = 7) executes in Store S1
    ├─► paddr = 0x2000 (SAME as Load B!)
    ├─► CAM search LoadQueueRAW
    │   → Entry 8 matches! (paddr = 0x2000)
    │   → Load B (robIdx=100) is younger than Store A (robIdx=95)
    └─► VIOLATION! → Rollback to Load B

T6: Redirect signal
    → LoadQueueRAW entry 8: needCancel = true → deallocated
    → Pipeline flushed, Load B will re-execute
```

---

## Summary Table

| Event | Timing | Condition | Result |
|-------|--------|-----------|--------|
| **Enqueue** | Load S2 | `hasAddrInvalidStore` (older store addr unknown) | Entry allocated with paddr |
| **C_RAW Replay** | Load S2 | Queue full (`!req.ready`) | Load replays later |
| **Dequeue (normal)** | Continuous | `stAddrReadySqPtr` passes entry's sqIdx | Entry freed (no violation) |
| **Dequeue (cancel)** | On redirect | `robIdx.needFlush(redirect)` | Entry freed (violation or flush) |
| **Dequeue (revoke)** | Load S3 | `revoke` signal from load | Entry freed (other replay) |
| **Violation Detect** | Store S1 | CAM search finds paddr match with younger load | Rollback generated |

---

## Key Insight

LoadQueueRAW is **NOT** for storing loads that have violated ordering. It's for **tracking loads that MIGHT violate ordering** because their older stores haven't executed yet.

- **Enqueue**: "I don't know if I violated, older store not executed yet"
- **Normal Dequeue**: "Older stores all executed, no conflict, I'm safe"
- **Violation Dequeue**: "Oops, older store conflicted with me, rollback"

---

## Design Rationale: Why Separate Queue?

### Question: Why not use VirtualLoadQueue for RAW detection?

VirtualLoadQueue (80 entries) tracks all in-flight loads, but it **cannot** be used for RAW violation detection:

### Reason 1: VirtualLoadQueue Has NO paddr

```
VirtualLoadQueue stores:
  - uop (MicroOp): robIdx, lqIdx, sqIdx, etc.
  - ❌ NO paddr (physical address)

LoadQueueRAW stores:
  - uop (MicroOp)
  - ✅ paddr (for CAM search)
  - ✅ mask (for byte-level matching)
```

**Why this matters**: Store S1 needs to CAM search by paddr to find conflicting loads. Without paddr, address comparison is impossible!

### Reason 2: Smaller CAM = Shorter Critical Path

```
If using VirtualLoadQueue for CAM:
  - 80 entries × StorePipelineWidth CAM ports
  - Every store must search ALL 80 entries
  - Long critical path, timing issues

With LoadQueueRAW (subset):
  - Only ~16-32 entries (loads with pending older stores)
  - Much smaller CAM, faster search
  - Better timing closure
```

### Comparison Table

| Aspect | VirtualLoadQueue | LoadQueueRAW |
|--------|------------------|--------------|
| **Size** | 80 entries | LoadQueueRAWSize (smaller) |
| **paddr storage** | ❌ None | ✅ Yes (CAM module) |
| **mask storage** | ❌ None | ✅ Yes |
| **CAM Search** | ❌ Impossible | ✅ Store S1 search |
| **Entry condition** | All issued loads | Only "at-risk" loads |
| **Purpose** | Commit ordering, replay tracking | RAW violation detection |

### Key Design Principle

```
"Track subset, not all"

All loads → VirtualLoadQueue (for commit)
     ↓ filter
Loads with pending older stores → LoadQueueRAW (for violation check)
```

This filtering reduces:
1. **Storage**: Only "at-risk" loads need paddr stored
2. **CAM size**: Smaller CAM = better timing
3. **Power**: Fewer entries to search per store
