# LSQ Block Diagram

## Overview
This document describes the detailed block diagram of the Load Store Queue (LSQ) subsystem in XiangShan, showing all modules and their interconnections from Rename to Write-back stages.

## Configuration Parameters

### Pipeline Widths
- **RenameWidth**: 6 (max instructions renamed/dispatched to DispatchQueue per cycle)
- **IntDqDeqWidth**: 4 (Integer Dispatch Queue dequeue width)
- **LsDqDeqWidth**: 4 (LS Dispatch Queue dequeue width) ← **Dispatch throughput bottleneck**
- **FpDqDeqWidth**: 4 (FP Dispatch Queue dequeue width)
- **CommitWidth**: 6 (max instructions committed per cycle)

### LSQ Enqueue
- **LsExuCnt**: **4** (LSQ enqueue interface width, **equals LsDqDeqWidth**)
  - Important: This matches **dispatch throughput**, not execution unit count!

### Execution Units (Pipelines)
- **LduCnt**: 2 (Load Unit Count, number of load execution pipelines)
- **StuCnt**: 2 (Store Unit Count, number of store execution pipelines)
- **LoadPipelineWidth**: 2 (same as LduCnt)
- **StorePipelineWidth**: 2 (same as StuCnt)

**Design Note**: Although LduCnt+StuCnt=4 equals LsExuCnt=4, this is a balanced design choice.
LsExuCnt is fundamentally determined by LsDqDeqWidth (dispatch throughput), not by execution unit count.

### Queue Sizes
- **ROB Size**: 192 (Reorder Buffer)
- **VirtualLoadQueueSize**: 80 (tracks ALL loads, 42% of ROB)
- **LoadQueueReplaySize**: 72 (tracks replay-needed loads only, 38% of ROB)
- **StoreQueueSize**: 64 (tracks ALL stores, 33% of ROB)

**Design Rationale:**
- VirtualLoadQueue (80) must be large enough to hold all in-flight loads in the 192-entry ROB
- Typical workload has ~40% loads, so 80 entries provides adequate coverage
- LoadQueueReplay (72) is smaller since not all loads miss/replay (typical miss rate: 5-20%)
- This sizing allows high Memory Level Parallelism (MLP) for memory-intensive workloads

## Top-Level Block Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              RENAME STAGE                                    │
│  (Dispatch)                                                                  │
└──────────────────────────┬──────────────────────────────────────────────────┘
                           │
                           │ LsqEnqIO
                           │  ├─ canAccept (backpressure from LSQ)
                           │  ├─ needAlloc[LsExuCnt]
                           │  ├─ req[LsExuCnt] (MicroOp)
                           │  └─ resp[LsExuCnt] (LqPtr/SqPtr)
                           │
                           ▼
         ┌─────────────────────────────────────────────────┐
         │            LSQ Wrapper (LsqWrapper)             │
         │                                                 │
         │  ┌───────────────────┐  ┌──────────────────┐   │
         │  │  VirtualLoadQueue │  │   StoreQueue     │   │
         │  │                   │  │                  │   │
         │  │  Entries:         │  │  Entries:        │   │
         │  │  - allocated[]    │  │  - allocated[]   │   │
         │  │  - uop[]          │  │  - uop[]         │   │
         │  │  - addrvalid[]    │◄─┼─►- addrvalid[]  │   │
         │  │  - datavalid[]    │  │  - datavalid[]   │   │
         │  │                   │  │  - committed[]   │   │
         │  │  Pointers:        │  │                  │   │
         │  │  - enqPtr         │  │  Modules:        │   │
         │  │  - deqPtr         │  │  - paddrModule   │   │
         │  │                   │  │  - vaddrModule   │   │
         │  │                   │  │  - dataModule    │   │
         │  │                   │  │                  │   │
         │  │                   │  │  Pointers:       │   │
         │  │                   │  │  - enqPtr        │   │
         │  │                   │  │  - deqPtr        │   │
         │  │                   │  │  - cmtPtr        │   │
         │  │                   │  │  - addrReadyPtr  │   │
         │  │                   │  │  - dataReadyPtr  │   │
         │  └────┬──────────────┘  └──────┬───────────┘   │
         │       │                        │               │
         │       │  Cross-communication:  │               │
         │       │  - sqCanAccept ────────┼───►           │
         │       │           ◄────────────┼─ lqCanAccept  │
         │       │                        │               │
         └───────┼────────────────────────┼───────────────┘
                 │                        │
                 │                        │
    ┌────────────┴────────────┐  ┌────────┴────────────┐
    │    LoadQueue Module     │  │  StoreQueue Module  │
    │                         │  │                     │
    │  Sub-modules:           │  │  Forward Query:     │
    │  ┌──────────────────┐   │  │  ┌──────────────┐  │
    │  │ LoadQueueRAR     │   │  │  │ Address CAM  │  │
    │  │ (Read-After-Read)│   │  │  │ Data Forward │  │
    │  └──────────────────┘   │  │  └──────────────┘  │
    │  ┌──────────────────┐   │  │                     │
    │  │ LoadQueueRAW     │   │  │  Writeback:         │
    │  │(Read-After-Write)│   │  │  ┌──────────────┐  │
    │  └──────────────────┘   │  │  │ SBuffer      │  │
    │  ┌──────────────────┐   │  │  └──────────────┘  │
    │  │LoadQueueReplay   │   │  │                     │
    │  │ (Replay Queue)   │   │  │  MMIO:              │
    │  └──────────────────┘   │  │  ┌──────────────┐  │
    │  ┌──────────────────┐   │  │  │ Uncache I/O  │  │
    │  │ExceptionBuffer   │   │  │  └──────────────┘  │
    │  └──────────────────┘   │  │                     │
    │  ┌──────────────────┐   │  │                     │
    │  │ UncacheBuffer    │   │  │                     │
    │  └──────────────────┘   │  │                     │
    └────────────┬────────────┘  └─────────┬───────────┘
                 │                         │
                 │                         │
                 ▼                         ▼
    ┌────────────────────────┐  ┌──────────────────────┐
    │   Load Pipeline (×2)   │  │  Store Pipeline (×2) │
    │                        │  │                      │
    │  [0] ┌──────────────┐  │  │  [0] ┌────────────┐ │
    │      │  Load Unit 0 │  │  │      │Store Unit 0│ │
    │      └──────────────┘  │  │      └────────────┘ │
    │  [1] ┌──────────────┐  │  │  [1] ┌────────────┐ │
    │      │  Load Unit 1 │  │  │      │Store Unit 1│ │
    │      └──────────────┘  │  │      └────────────┘ │
    │                        │  │                      │
    │  Interfaces:           │  │  Interfaces:         │
    │  - ldin[2]             │  │  - storeAddrIn[2]    │
    │  - ldout[2]            │  │  - storeDataIn[2]    │
    │  - replay[2]           │  │  - storeMaskIn[2]    │
    │  - ld_raw_data[2]      │  │  - stout[2]          │
    │  - stld_nuke_query[2]  │  │  - forward[2]        │
    │  - ldld_nuke_query[2]  │  │                      │
    └────────────┬───────────┘  └─────────┬────────────┘
                 │                        │
                 │                        │
                 ▼                        ▼
         ┌──────────────────────────────────────┐
         │          ROB (Reorder Buffer)        │
         │                                      │
         │  Commit Interface:                   │
         │  - lcommit (load commit count)       │
         │  - scommit (store commit count)      │
         │  - pendingld                         │
         │  - pendingst                         │
         └──────────────────────────────────────┘
```

## Detailed Module Interconnections

### 1. Enqueue Path (Rename → LSQ)

#### A. Top-Level Enqueue Interface (4-wide)

```
Dispatch/Rename
      │
      │ LsqEnqIO (4-wide interface, LsExuCnt = 4)
      │
      ├─► needAlloc[0]: [1]=LQ, [0]=SQ  ──┐
      ├─► needAlloc[1]: [1]=LQ, [0]=SQ  ──┤
      ├─► needAlloc[2]: [1]=LQ, [0]=SQ  ──┼─► Used for offset calculation
      ├─► needAlloc[3]: [1]=LQ, [0]=SQ  ──┘
      │
      ├─► req[0].valid, req[0].bits (MicroOp) ──┐
      ├─► req[1].valid, req[1].bits (MicroOp) ──┤
      ├─► req[2].valid, req[2].bits (MicroOp) ──┼─► Up to 4 instructions/cycle
      ├─► req[3].valid, req[3].bits (MicroOp) ──┘
      │
      ◄── canAccept: backpressure (LQ && SQ both have space)
      │
      ◄── resp[0]: LSIdx (lqIdx + sqIdx) ──┐
      ◄── resp[1]: LSIdx (lqIdx + sqIdx) ──┤
      ◄── resp[2]: LSIdx (lqIdx + sqIdx) ──┼─► Allocated queue indices
      ◄── resp[3]: LSIdx (lqIdx + sqIdx) ──┘
      │
      ▼
   LSQWrapper.io.enq
      │
      ├──────────────┬──────────────┐
      ▼              ▼              ▼
  LoadQueue.io.enq  StoreQueue.io.enq
      │              │
      ├─► canAccept ─┼─► canAccept
      ├─◄ sqCanAccept│
      │  lqCanAccept◄┤
      │              │
      ▼              ▼
VirtualLoadQueue   StoreQueue
  enqueue logic    enqueue logic
```

#### B. VirtualLoadQueue Internal Enqueue Logic

```
┌─────────────────────────────────────────────────────────────────┐
│                  VirtualLoadQueue Enqueue Logic                 │
└─────────────────────────────────────────────────────────────────┘

Step 1: Space Check
  ┌──────────────────────────────────────────────┐
  │ validCount = distanceBetween(enqPtr, deqPtr) │
  │ allowEnqueue = validCount ≤                  │
  │     (VirtualLoadQueueSize - LoadPipelineWidth)│
  └──────────────────────────────────────────────┘
                      │
                      ▼
Step 2: Multiple Enqueue Pointer Management
  ┌──────────────────────────────────────────────┐
  │ enqPtrExt[0] ───► Base pointer               │
  │ enqPtrExt[1] ───► Base + 1                   │
  │ enqPtrExt[2] ───► Base + 2                   │
  │ enqPtrExt[3] ───► Base + 3                   │
  └──────────────────────────────────────────────┘
                      │
                      ▼
Step 3: Offset Calculation & Entry Allocation
  ┌──────────────────────────────────────────────────────────────┐
  │ For i = 0 to 3:                                              │
  │   offset[i] = PopCount(needAlloc.take(i))                    │
  │                                                              │
  │   Example (needAlloc = [1,0,1,1]):                          │
  │     req[0]: offset=0 → enqPtrExt[0]                         │
  │     req[1]: (not needed)                                     │
  │     req[2]: offset=1 → enqPtrExt[1]                         │
  │     req[3]: offset=2 → enqPtrExt[2]                         │
  │                                                              │
  │   Allocation:                                                │
  │     lqIdx = enqPtrExt[offset]                               │
  │     allocated[lqIdx.value] := true.B                        │
  │     uop[lqIdx.value] := req[i].bits                         │
  │     addrvalid[lqIdx.value] := false.B                       │
  │     datavalid[lqIdx.value] := false.B                       │
  │                                                              │
  │   Response:                                                  │
  │     resp[i] := lqIdx                                         │
  └──────────────────────────────────────────────────────────────┘
                      │
                      ▼
Step 4: Pointer Update
  ┌──────────────────────────────────────────────────────────────┐
  │ enqNumber = PopCount(req.map(_.valid))                       │
  │                                                              │
  │ Normal case:                                                 │
  │   enqPtrExt[i] := enqPtrExt[i] + enqNumber  (all advance)   │
  │                                                              │
  │ Redirect case (2 cycles after redirect):                     │
  │   enqPtrExt[i] := enqPtrExt[i] - redirectCancelCount        │
  │                    (rollback cancelled instructions)         │
  └──────────────────────────────────────────────────────────────┘

Queue State Visualization (VirtualLoadQueueSize = 80):
  ┌───┬───┬───┬───┬───┬───┬───┬─  ─┬───┬───┐
  │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │... │78 │79 │
  └───┴───┴───┴───┴───┴───┴───┴─  ─┴───┴───┘
    ▲                           ▲
    │                           │
  deqPtr                    enqPtrExt[0..3]
 (commit)                    (allocate)

  enqPtrExt pointers:
    [0] = current base
    [1] = base + 1
    [2] = base + 2
    [3] = base + 3

  Each cycle, all 4 pointers advance together by enqNumber
```

### 2. Distribution from LSQ to Load/Store Pipelines

After instructions enter the LSQ during dispatch, they must be issued to the physical load/store execution units. XiangShan has **2 Load Units** and **2 Store Units**. This section describes how instructions are distributed from LSQ to these pipelines.

#### 2.1 Load Pipeline Distribution (LSQ → Load Units)

**Configuration:**
- LoadPipelineWidth = 2
- LduCnt = 2 (Load Unit count)
- VirtualLoadQueueSize = 80 (tracks ALL loads from dispatch to commit)
- LoadQueueReplaySize = 72 (tracks ONLY loads that need replay)

**Architecture Overview:**

XiangShan uses **two separate queues** for load tracking:

1. **VirtualLoadQueue (80 entries)**: Tracks **ALL** loads from dispatch to commit
   - Allocated at dispatch (Rename stage)
   - Deallocated at commit
   - Purpose: lqIdx assignment, exception tracking, commit ordering
   - Always has entry for every in-flight load

2. **LoadQueueReplay (72 entries)**: Tracks **ONLY** loads that need replay
   - Allocated when Load Unit S3 detects `need_rep == true`
   - Deallocated when replay completes successfully or instruction commits
   - Purpose: Replay scheduling and distribution to Load Units
   - Smaller than VLQ because not all loads need replay

```
Load Lifecycle:
┌─────────────────────────────────────────────────────────────┐
│ Dispatch → VirtualLoadQueue [Entry X allocated]            │
│     │                                                       │
│     ├──► Load Unit (first issue from RS)                   │
│     │        │                                              │
│     │        ├──► Hit → Commit                              │
│     │        │          └─► VLQ[X] deallocated             │
│     │        │                                              │
│     │        └──► Miss/Replay (S3: need_rep=true)          │
│     │                │                                      │
│     │                └──► LoadQueueReplay [Entry Y allocated]│
│     │                         │                             │
│     │                         ├──► Replay → Hit             │
│     │                         │      └─► LQReplay[Y] dealloc│
│     │                         │          VLQ[X] continues   │
│     │                         │                             │
│     └──────────────────── Commit ─────────────────────────► │
│                                  VLQ[X] deallocated         │
└─────────────────────────────────────────────────────────────┘

Key: VLQ entry lives entire time (dispatch→commit)
     LQReplay entry only during replay period
```

LoadQueueReplay implements a **striped oldest-first distribution** scheme to issue replays to the 2 Load Units in parallel.

**Key Components (LoadQueueReplay.scala):**

```scala
// LoadQueueReplay outputs (LoadQueueReplay.scala:170)
val replay = Vec(LoadPipelineWidth, Decoupled(new LsPipelineBundle))

// Connection in MemBlock.scala:631
loadUnits(i).io.replay <> lsq.io.replay(i)
```

**Distribution Mechanism:**

LoadQueueReplay divides its 72 entries into **2 interleaved stripes** for parallel processing:

```
Entry Striping (LoadQueueReplaySize = 72):
┌──────────────────────────────────────────────────────────────────┐
│ Stripe 0 (for LoadUnit[0]):  0,  2,  4,  6, ..., 68, 70        │
│ Stripe 1 (for LoadUnit[1]):  1,  3,  5,  7, ..., 69, 71        │
└──────────────────────────────────────────────────────────────────┘

Implementation (LoadQueueReplay.scala:342-348):
  def getRemBits(input: UInt)(rem: Int): UInt = {
    VecInit((0 until LoadQueueReplaySize / LoadPipelineWidth).map(i => {
      input(LoadPipelineWidth * i + rem)
    })).asUInt
  }

  // rem=0 extracts entries [0, 2, 4, ...]
  // rem=1 extracts entries [1, 3, 5, ...]
```

**Selection Logic (3-Stage Pipeline):**

LoadQueueReplay uses a **3-stage pipeline** (s0, s1, s2) to select and issue replays:

**Stage 0 (s0): Oldest Selection with Priority**

Each Load Unit gets an independent oldest-first selector operating on its stripe:

```scala
// LoadQueueReplay.scala:351-352
val s0_oldestSel = Wire(Vec(LoadPipelineWidth, Valid(UInt(LoadQueueReplaySize.W))))
val s1_can_go = Wire(Vec(LoadPipelineWidth, Bool()))

// Selection priority (LoadQueueReplay.scala:388-407):
// 1. L2 hint wake-up loads (highest priority)
//    - Loads blocked on cache miss, woken early when L2 GrantData incoming
//    - Condition: s0_loadHintWakeMask(i) = allocated && cause(C_DM) &&
//                 blocking && missMSHRId matches l2_hint.sourceId
//
// 2. Higher-priority replay causes
//    - C_DM: DCache miss
//    - C_FF: Forward fail (store-to-load forwarding failed)
//    - Condition: s0_loadHigherPriorityReplaySelMask(i) =
//                 allocated && !scheduled && !blocking && hasHigherPriority
//
// 3. Lower-priority replay causes
//    - C_MA: Store addr miss (blocked waiting for store address)
//    - C_TM: TLB miss
//    - C_RAW/C_RAR: Memory ordering violation
//    - C_NK: Nuke (pipeline flush)
//    - Condition: s0_loadLowerPriorityReplaySelMask(i) =
//                 allocated && !scheduled && !blocking && !hasHigherPriority

// For each LoadPipelineWidth port (LoadQueueReplay.scala:427-452):
s0_oldestSel := VecInit((0 until LoadPipelineWidth).map(rport => {
  // Age-based selection within stripe
  val ageOldest = AgeDetector(
    LoadQueueReplaySize / LoadPipelineWidth,  // 36 entries per stripe
    s0_remEnqSelVec(rport),      // Enqueue mask
    s0_remFreeSelVec(rport),     // Free mask
    s0_remPriorityReplaySelVec(rport)  // Ready candidates with priority
  )

  // Program-order oldest selection (for L2 hint priority)
  val issOldestValid = l2HintFirst || ParallelORR(s0_remOldestSelVec(rport))
  val issOldestIndexOH = Mux(l2HintFirst,
                             PriorityEncoderOH(s0_remOldestHintSelVec(rport)),
                             PriorityEncoderOH(s0_remOldestSelVec(rport)))

  // Final selection: program order oldest if available, else age oldest
  val oldestSel = Mux(issOldestValid, issOldestIndexOH, ageOldestIndexOH)

  oldest.valid := ageOldest.valid || issOldestValid
  oldest.bits := oldestBitsVec.asUInt  // One-hot encoding
  oldest
}))
```

**Stage 1 (s1): Register and Schedule**

```scala
// LoadQueueReplay.scala:465-478
for (i <- 0 until LoadPipelineWidth) {
  val s0_can_go = s1_can_go(i) ||
                  uop(s1_oldestSel(i).bits).robIdx.needFlush(io.redirect) ||
                  uop(s1_oldestSel(i).bits).robIdx.needFlush(RegNext(io.redirect))

  // Register selection
  s1_oldestSel(i).valid := RegEnable(s0_oldestSel(i).valid, s0_can_go)
  s1_oldestSel(i).bits := RegEnable(OHToUInt(s0_oldestSel(i).bits), s0_can_go)

  // Mark as scheduled when selected
  for (j <- 0 until LoadQueueReplaySize) {
    when (s0_can_go && s0_oldestSel(i).valid && s0_oldestSelIndexOH(j)) {
      scheduled(j) := true.B
    }
  }
}
```

**Stage 2 (s2): Issue to Load Unit**

```scala
// LoadQueueReplay.scala:480-520
for (i <- 0 until LoadPipelineWidth) {
  val s1_cancel = uop(s1_oldestSel(i).bits).robIdx.needFlush(io.redirect) ||
                  uop(s1_oldestSel(i).bits).robIdx.needFlush(RegNext(io.redirect))
  val s1_oldestSelV = s1_oldestSel(i).valid && !s1_cancel

  // Cold-down mechanism: prevents continuous replay flooding
  // Only replay if: coldCounter < ColdDownThreshold
  s1_can_go(i) := replayCanFire(i) && (!s2_oldestSel(i).valid || io.replay(i).fire) ||
                  s2_cancelReplay(i)

  s2_oldestSel(i).valid := RegEnable(Mux(s1_can_go(i), s1_oldestSelV, false.B),
                                     (s1_can_go(i) || io.replay(i).fire))
  s2_oldestSel(i).bits := RegEnable(s1_oldestSel(i).bits, s1_can_go(i))

  // Output to Load Unit
  io.replay(i).valid := s2_oldestSel(i).valid
  io.replay(i).bits.uop := s2_replayUop
  io.replay(i).bits.vaddr := vaddrModule.io.rdata(i)
  io.replay(i).bits.isFirstIssue := false.B
  io.replay(i).bits.isLoadReplay := true.B
  io.replay(i).bits.replayCarry := s2_replayCarry
  io.replay(i).bits.mshrid := s2_replayMSHRId
  io.replay(i).bits.forward_tlDchannel := s2_replayCauses(LoadReplayCauses.C_DM)
}
```

**Replay Cold-Down Mechanism:**

To prevent replay flooding, a cold-down counter limits consecutive replays:

```scala
// LoadQueueReplay.scala:456-532
val ColdDownCycles = 16
val ColdDownThreshold = 12  // Configurable
val coldCounter = RegInit(VecInit(List.fill(LoadPipelineWidth)(0.U)))

def replayCanFire(i: Int) = coldCounter(i) >= 0.U && coldCounter(i) < ColdDownThreshold

// Update logic:
for (i <- 0 until LoadPipelineWidth) {
  when (lastReplay(i) && io.replay(i).fire) {
    coldCounter(i) := coldCounter(i) + 1.U  // Increment if consecutive replay
  } .elsewhen (coldDownNow(i)) {
    coldCounter(i) := coldCounter(i) + 1.U  // Continue counting in cold-down
  } .otherwise {
    coldCounter(i) := 0.U  // Reset if no replay
  }
}
```

**Distribution Summary:**

```
LoadQueueReplay (72 entries)
         │
         ├────────────────────────┬────────────────────────┐
         │                        │                        │
    Stripe 0                 Stripe 1                     │
  (0,2,4,...,70)           (1,3,5,...,71)                │
         │                        │                        │
         │                        │                        │
    ┌────▼────┐              ┌────▼────┐                  │
    │ Oldest  │              │ Oldest  │                  │
    │ Select  │              │ Select  │                  │
    │  (s0)   │              │  (s0)   │                  │
    └────┬────┘              └────┬────┘                  │
         │                        │                        │
    ┌────▼────┐              ┌────▼────┐                  │
    │Register │              │Register │                  │
    │  (s1)   │              │  (s1)   │                  │
    └────┬────┘              └────┬────┘                  │
         │                        │                        │
    ┌────▼────┐              ┌────▼────┐                  │
    │  Issue  │              │  Issue  │                  │
    │  (s2)   │              │  (s2)   │                  │
    └────┬────┘              └────┬────┘                  │
         │                        │                        │
         ▼                        ▼                        │
   LoadUnit[0]              LoadUnit[1]                    │
   io.replay                io.replay                      │
```

**Blocking Conditions Release:**

Loads become ready for replay when their blocking conditions are resolved:

```scala
// LoadQueueReplay.scala:310-338
for (i <- 0 until LoadQueueReplaySize) {
  // C_MA: Store addr miss - wait for store address ready
  when (cause(i)(C_MA)) {
    blocking(i) := Mux(stAddrDeqVec(i), false.B, blocking(i))
  }

  // C_TM: TLB miss - wait for TLB hint response
  when (cause(i)(C_TM)) {
    blocking(i) := Mux(io.tlb_hint.resp.valid &&
                       (io.tlb_hint.resp.bits.replay_all ||
                        io.tlb_hint.resp.bits.id === tlbHintId(i)),
                       false.B, blocking(i))
  }

  // C_FF: Forward fail - wait for store data ready
  when (cause(i)(C_FF)) {
    blocking(i) := Mux(stDataDeqVec(i), false.B, blocking(i))
  }

  // C_DM: DCache miss - wait for TileLink D channel grant
  when (cause(i)(C_DM)) {
    blocking(i) := Mux(io.tl_d_channel.valid &&
                       io.tl_d_channel.mshrid === missMSHRId(i),
                       false.B, blocking(i))
  }

  // C_RAR: RAR queue full - wait for queue space
  when (cause(i)(C_RAR)) {
    blocking(i) := Mux(!io.rarFull || !isAfter(uop(i).lqIdx, io.ldWbPtr),
                       false.B, blocking(i))
  }

  // C_RAW: RAW queue full - wait for queue space
  when (cause(i)(C_RAW)) {
    blocking(i) := Mux(!io.rawFull || !isAfter(uop(i).sqIdx, io.stAddrReadySqPtr),
                       false.B, blocking(i))
  }
}
```

#### 2.2 Store Pipeline Distribution (RS → Store Units)

**Configuration:**
- StorePipelineWidth = 2
- StuCnt = 2 (Store Unit count)
- StoreQueueSize = 64

**Architecture Overview:**

Unlike loads, stores do **NOT** have a replay mechanism from StoreQueue. Stores are issued **directly from Reservation Station** to the 2 Store Units. The store pipeline has two components:
1. **STA (Store Address)**: Calculates store address and writes to StoreQueue
2. **STD (Store Data)**: Sends store data to StoreQueue

**Distribution Mechanism:**

```
Reservation Station
         │
         │ (Issue to Store Units)
         ├────────────────┬────────────────┐
         │                │                │
         ▼                ▼                │
    StoreUnit[0]     StoreUnit[1]         │
         │                │                │
         │ STA            │ STA            │
         │ (S0→S1)        │ (S0→S1)        │
         │                │                │
         ▼                ▼                │
    storeAddrIn[0]   storeAddrIn[1]       │
         │                │                │
         └────────┬───────┘                │
                  │                        │
                  ▼                        │
            StoreQueue                     │
         (Unified Storage)                 │
                  │                        │
         ┌────────┴────────┐               │
         │                 │               │
    addrvalid          datavalid           │
    paddrModule        dataModule          │
    vaddrModule                            │
                                           │
                  │                        │
    (From RS)     │                        │
         │        │                        │
         ├────────┼────────┬───────────────┘
         │        │        │
         ▼        │        ▼
    Std[0]       │    Std[1]
    (S0)         │    (S0)
         │       │        │
         │       │        │
    storeDataIn[0]   storeDataIn[1]
         │                │
         └────────┬───────┘
                  │
                  ▼
            StoreQueue
         (Update datavalid)
```

**Connection Details (MemBlock.scala:671-702):**

```scala
// Store Unit creation (MemBlock.scala:281)
val storeUnits = Seq.fill(exuParameters.StuCnt)(Module(new StoreUnit))

for (i <- 0 until exuParameters.StuCnt) {
  val stu = storeUnits(i)

  // 1. Store Address (STA) issue from RS
  // MemBlock.scala:685
  stu.io.stin <> io.ooo_to_mem.issue(exuParameters.LduCnt + i)
  //   StoreUnit[0] ← issue(2)
  //   StoreUnit[1] ← issue(3)

  // 2. STA writeback to StoreQueue
  // MemBlock.scala:686-687
  stu.io.lsq <> lsq.io.sta.storeAddrIn(i)
  stu.io.lsq_replenish <> lsq.io.sta.storeAddrInRe(i)
  //   Writes: paddr, vaddr, mask, mmio flag

  // 3. Store mask to StoreQueue
  // MemBlock.scala:699
  lsq.io.sta.storeMaskIn(i) <> stu.io.st_mask_out

  // 4. Store Data (STD) issue from RS
  // MemBlock.scala:68, 674-677
  stdExeUnits(i).io.fromInt <> io.ooo_to_mem.issue(i + LduCnt + StuCnt)
  //   StdExeUnit[0] ← issue(4)
  //   StdExeUnit[1] ← issue(5)

  // 5. STD writeback to StoreQueue
  // MemBlock.scala:702
  lsq.io.std.storeDataIn(i) := stData(i)
  //   Writes: store data value
}
```

**Issue Allocation:**

The issue ports are allocated as follows:

```
io.ooo_to_mem.issue[] allocation (LsExuCnt = 4, StuCnt = 2):
┌─────────┬──────────────────────────────────────────┐
│ Port    │ Target                                   │
├─────────┼──────────────────────────────────────────┤
│ [0]     │ LoadUnit[0].io.ldin                     │
│ [1]     │ LoadUnit[1].io.ldin                     │
│ [2]     │ StoreUnit[0].io.stin (STA)              │
│ [3]     │ StoreUnit[1].io.stin (STA)              │
│ [4]     │ StdExeUnit[0].io.fromInt (STD)          │
│ [5]     │ StdExeUnit[1].io.fromInt (STD)          │
└─────────┴──────────────────────────────────────────┘
```

**StoreQueue Writeback (StoreQueue.scala:296-349):**

```scala
// Store Address writeback
for (i <- 0 until StorePipelineWidth) {
  val stWbIndex = io.storeAddrIn(i).bits.uop.sqIdx.value
  when (io.storeAddrIn(i).fire) {
    val addr_valid = !io.storeAddrIn(i).bits.miss
    addrvalid(stWbIndex) := addr_valid

    // Write physical address
    paddrModule.io.waddr(i) := stWbIndex
    paddrModule.io.wdata(i) := io.storeAddrIn(i).bits.paddr
    paddrModule.io.wmask(i) := io.storeAddrIn(i).bits.mask
    paddrModule.io.wen(i) := true.B

    // Write virtual address
    vaddrModule.io.waddr(i) := stWbIndex
    vaddrModule.io.wdata(i) := io.storeAddrIn(i).bits.vaddr
    vaddrModule.io.wen(i) := true.B

    // Update uop control info
    uop(stWbIndex).ctrl := io.storeAddrIn(i).bits.uop.ctrl
  }

  // Store mmio/atomic info (1 cycle later after PMA/PMP check)
  val storeAddrInFireReg = RegNext(io.storeAddrIn(i).fire && !io.storeAddrIn(i).bits.miss)
  val stWbIndexReg = RegNext(stWbIndex)
  when (storeAddrInFireReg) {
    pending(stWbIndexReg) := io.storeAddrInRe(i).mmio
    mmio(stWbIndexReg) := io.storeAddrInRe(i).mmio
    atomic(stWbIndexReg) := io.storeAddrInRe(i).atomic
    prefetch(stWbIndexReg) := io.storeAddrInRe(i).miss
  }
}

// Store Data writeback (not shown in provided excerpt, similar pattern)
// Updates: datavalid flag and data in dataModule
```

**Key Differences: Load vs Store Distribution:**

| Aspect | Load Pipeline | Store Pipeline |
|--------|--------------|----------------|
| **Source** | LoadQueueReplay (replay queue) | Reservation Station (direct issue) |
| **Distribution** | Striped oldest-first (entries 0,2,4,... vs 1,3,5,...) | RS arbiter decides per instruction |
| **Priority** | 3-level: L2 hint > Higher priority (DM,FF) > Lower priority | RS scheduling policy |
| **Selection** | Age-based within stripe, program-order for L2 hint | RS ready-oldest or other policy |
| **Pipeline Stages** | 3-stage (s0: select, s1: register, s2: issue) | Direct (RS → Store Unit) |
| **Throttling** | Cold-down counter prevents flooding | RS rate limiting |
| **Replay Mechanism** | Yes (normal replay, fast replay, super replay) | No (stores don't replay from SQ) |

**Conditions for Distribution:**

**Load Replay Conditions:**
1. **Entry must be allocated**: `allocated(i) === true.B`
2. **Entry must not be scheduled**: `scheduled(i) === false.B`
3. **Entry must not be blocking**: `blocking(i) === false.B`
4. **Cold-down must allow**: `coldCounter < ColdDownThreshold`
5. **Must pass priority selection**: Selected as oldest in its stripe
6. **Must not be flushed**: `!uop(i).robIdx.needFlush(redirect)`

**Store Issue Conditions:**
1. **RS must have ready entry**: Operands ready, no structural hazards
2. **StoreQueue must have space**: Entry was allocated at dispatch
3. **TLB port available**: For address translation
4. **No pipeline hazards**: Store Unit pipeline not stalled

### 3. Load Unit Complete Interface Block Diagram

This diagram shows **ALL** interfaces of a single Load Unit with complete detail.
Each Load Unit (there are 2: LoadUnit[0] and LoadUnit[1]) has identical interfaces.

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                    LOAD UNIT [i]                                       │
│                         (4-Stage Pipeline: S0→S1→S2→S3)                               │
└────────────────────────────────────────────────────────────────────────────────────────┘

╔═══════════════════════════════════════════════════════════════════════════════════════╗
║                              INPUT INTERFACES (S0 Stage)                              ║
╚═══════════════════════════════════════════════════════════════════════════════════════╝

┌─── 7 Input Sources (Priority Arbitration in S0) ────────────────────────────────────┐
│                                                                                      │
│  [Priority 0 - Highest] ─────────────────────────────────────────────────────┐      │
│    io.replay (Super Replay)                                                  │      │
│      ├─► valid: Bool                                                         │      │
│      ├─► bits.forward_tlDchannel: Bool = true  ← Super replay flag           │      │
│      ├─► bits.vaddr: UInt(VAddrBits)                                         │      │
│      ├─► bits.mask: UInt(8.W)                                                │      │
│      ├─► bits.uop: MicroOp                                                   │      │
│      ├─► bits.mshrid: UInt  ← MSHR ID for D channel forward                 │      │
│      └─◄ ready: Bool                                                         │      │
│                                                                               │      │
│  [Priority 1] ───────────────────────────────────────────────────────────────┤      │
│    io.fast_rep_in (Fast Replay)                                              │      │
│      ├─► valid: Bool                                                         │      │
│      ├─► bits: LqWriteBundle (complete load info from fast replay queue)    │      │
│      ├─► bits.lateKill: Bool  ← Kill signal for late cancellation            │      │
│      ├─► bits.delayedLoadError: Bool  ← Delayed error flag                  │      │
│      └─◄ ready: Bool                                                         │      │
│                                                                               │      │
│  [Priority 2] ───────────────────────────────────────────────────────────────┤      │
│    io.replay (Normal Replay)                                                 │      │
│      ├─► valid: Bool                                                         │      │
│      ├─► bits.forward_tlDchannel: Bool = false  ← Normal replay              │      │
│      ├─► bits: LsPipelineBundle                                              │      │
│      ├─► bits.replacementUpdated: Bool                                       │      │
│      └─◄ ready: Bool (gated by !rep_stall)                                  │      │
│                                                                               │      │
│  [Priority 3] ───────────────────────────────────────────────────────────────┤      │
│    io.prefetch_req (High-Confidence HW Prefetch)                             │      │
│      ├─► valid: Bool                                                         │      │
│      ├─► bits.confidence: UInt > 0  ← High confidence                        │      │
│      ├─► bits.vaddr: UInt(VAddrBits)                                         │      │
│      └─► bits.paddr: UInt(PAddrBits) (optional)                              │      │
│                                                                               │      │
│  [Priority 4] ───────────────────────────────────────────────────────────────┤      │
│    io.ldin (RS Issue / SW Prefetch)                                          │      │
│      ├─► valid: Bool                                                         │      │
│      ├─► bits.src: Vec(UInt(XLEN), 3)  ← Source operands                    │      │
│      ├─► bits.uop: MicroOp                                                   │      │
│      └─◄ ready: Bool                                                         │      │
│                                                                               │      │
│  [Priority 5] ───────────────────────────────────────────────────────────────┤      │
│    io.vec_in (Vector Issue) - NOT IMPLEMENTED                                │      │
│                                                                               │      │
│  [Priority 6] ───────────────────────────────────────────────────────────────┤      │
│    io.l2l_fwd_in (Load-to-Load Forwarding / Pointer Chasing)                │      │
│      ├─► valid: Bool                                                         │      │
│      ├─► data: UInt(64.W)  ← Forwarded address from previous load           │      │
│      ├─► dly_ld_err: Bool  ← Error from previous load                       │      │
│      └─◄ ready: Bool                                                         │      │
│                                                                               │      │
│  [Priority 7 - Lowest] ──────────────────────────────────────────────────────┘      │
│    io.prefetch_req (Low-Confidence HW Prefetch)                                     │
│      ├─► valid: Bool                                                                │
│      ├─► bits.confidence: UInt == 0  ← Low confidence                               │
│      └─► bits.vaddr: UInt(VAddrBits)                                                │
│                                                                                      │
│  ┌─ Priority Arbitration Logic (ParallelPriorityMux) ───────────────────────────┐  │
│  │  s0_sel_src = ParallelPriorityMux(                                           │  │
│  │    [s0_super_ld_rep_select,  // Priority 0                                   │  │
│  │     s0_ld_fast_rep_select,   // Priority 1                                   │  │
│  │     s0_ld_rep_select,        // Priority 2                                   │  │
│  │     s0_hw_prf_select,        // Priority 3                                   │  │
│  │     s0_int_iss_select,       // Priority 4                                   │  │
│  │     s0_vec_iss_select,       // Priority 5                                   │  │
│  │     s0_l2l_fwd_select],      // Priority 6                                   │  │
│  │    [super_replay_src, fast_replay_src, normal_replay_src, ...]              │  │
│  │  )                                                                            │  │
│  │  → Only ONE source selected per cycle                                        │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────────┘

┌─── Memory System Interfaces ─────────────────────────────────────────────────────────┐
│                                                                                      │
│  TLB Interface (S0→S1)                                                               │
│    io.tlb.req (S0 - Request)                                                         │
│      ├─► valid: Bool                                                                 │
│      ├─► bits.vaddr: UInt(VAddrBits)                                                 │
│      ├─► bits.cmd: UInt (TlbCmd.read)                                                │
│      ├─► bits.size: UInt                                                             │
│      ├─► bits.memidx.is_ld: Bool = true                                             │
│      ├─► bits.memidx.idx: UInt (lqIdx.value)                                        │
│      ├─► bits.debug.robIdx: RobPtr                                                  │
│      ├─► bits.debug.pc: UInt                                                         │
│      └─► bits.debug.isFirstIssue: Bool                                              │
│                                                                                      │
│    io.tlb.resp (S1 - Response)                                                       │
│      ├─◄ valid: Bool                                                                 │
│      ├─◄ bits.paddr(0): UInt(PAddrBits)  ← For LSU                                  │
│      ├─◄ bits.paddr(1): UInt(PAddrBits)  ← For DCache                               │
│      ├─◄ bits.miss: Bool                                                             │
│      ├─◄ bits.excp(0).pf.ld: Bool  ← Page fault                                     │
│      ├─◄ bits.excp(0).af.ld: Bool  ← Access fault                                   │
│      ├─◄ bits.ptwBack: Bool  ← PTW walk needed                                      │
│      ├─◄ bits.memidx: MemIdx                                                        │
│      └─► ready: Bool = true.B                                                        │
│                                                                                      │
│    io.tlb.req_kill (S1)                                                              │
│      └─► Bool  ← Kill TLB request (s1_kill || s1_dly_err)                           │
│                                                                                      │
│  ──────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  DCache Interface (S0→S1→S2)                                                         │
│    io.dcache.req (S0 - Request)                                                      │
│      ├─► valid: Bool                                                                 │
│      ├─► bits.cmd: UInt (MemoryOpConstants.M_XRD)                                    │
│      ├─► bits.vaddr: UInt(VAddrBits)                                                 │
│      ├─► bits.mask: UInt(8.W)                                                        │
│      └─◄ ready: Bool                                                                 │
│                                                                                      │
│    io.dcache.s0_pc (S0)                                                              │
│      └─► UInt  ← PC for debug                                                        │
│                                                                                      │
│    io.dcache.s1_paddr_dup_lsu (S1)                                                   │
│      └─► UInt(PAddrBits)  ← Physical address for LSU path                           │
│                                                                                      │
│    io.dcache.s1_paddr_dup_dcache (S1)                                                │
│      └─► UInt(PAddrBits)  ← Physical address for DCache path                        │
│                                                                                      │
│    io.dcache.s1_pc (S1)                                                              │
│      └─► UInt  ← PC for debug                                                        │
│                                                                                      │
│    io.dcache.s1_kill (S1)                                                            │
│      └─► Bool  ← Kill DCache access (TLB miss/exception/MMIO/dly_err)               │
│                                                                                      │
│    io.dcache.s1_disable_fast_wakeup (S1)                                             │
│      ├─◄ Bool  ← Disable fast wakeup if uncertain                                   │
│                                                                                      │
│    io.dcache.s2_pc (S2)                                                              │
│      └─► UInt  ← PC for debug                                                        │
│                                                                                      │
│    io.dcache.s2_kill (S2)                                                            │
│      └─► Bool  ← Kill S2 (PMP violation/MMIO/flush)                                  │
│                                                                                      │
│    io.dcache.resp (S2 - Response)                                                    │
│      ├─◄ valid: Bool                                                                 │
│      ├─◄ bits.data: UInt(XLEN)  ← Cache data                                        │
│      ├─◄ bits.data_delayed: UInt(XLEN)  ← Delayed data for S3                       │
│      ├─◄ bits.miss: Bool  ← Cache miss                                              │
│      ├─◄ bits.replay: Bool  ← Needs replay                                          │
│      ├─◄ bits.tag_error: Bool  ← ECC tag error                                      │
│      ├─◄ bits.mshr_id: UInt  ← MSHR entry ID                                        │
│      ├─◄ bits.handled: Bool  ← Handled by MSHR                                      │
│      ├─◄ bits.real_miss: Bool  ← First-time miss                                    │
│      ├─◄ bits.replayCarry: ReplayCarry                                              │
│      ├─◄ bits.replacementUpdated: Bool                                              │
│      ├─◄ bits.meta_prefetch: Bool                                                   │
│      ├─◄ bits.meta_access: Bool                                                     │
│      └─► ready: Bool = true.B                                                        │
│                                                                                      │
│    io.dcache.s2_mq_nack (S2)                                                         │
│      └─◄ Bool  ← MSHR queue full, cannot allocate                                   │
│                                                                                      │
│    io.dcache.s2_bank_conflict (S2)                                                   │
│      └─◄ Bool  ← Bank conflict detected                                             │
│                                                                                      │
│    io.dcache.s2_wpu_pred_fail (S2)                                                   │
│      └─◄ Bool  ← Way predictor prediction failed                                    │
│                                                                                      │
│    io.dcache.replacementUpdated (S0)                                                 │
│      └─► Bool  ← Replacement policy updated (for replay loads)                      │
│                                                                                      │
│  ──────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  PMP Interface (S2)                                                                  │
│    io.pmp (S2 - Response from PMP Checker)                                           │
│      ├─◄ ld: Bool  ← Load access violation                                          │
│      ├─◄ st: Bool  ← Store access violation                                         │
│      ├─◄ instr: Bool  ← Instruction access violation                                │
│      ├─◄ mmio: Bool  ← MMIO region                                                  │
│      └─◄ atomic: Bool  ← Atomic region                                              │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘

┌─── Store-to-Load Forwarding Interfaces (S1) ─────────────────────────────────────────┐
│                                                                                      │
│  StoreBuffer Forward Query (S1)                                                      │
│    io.sbuffer (S1 - Query committed stores in SBuffer)                               │
│      ├─► valid: Bool                                                                 │
│      ├─► vaddr: UInt(VAddrBits)                                                      │
│      ├─► paddr: UInt(PAddrBits)                                                      │
│      ├─► uop: MicroOp                                                                │
│      ├─► sqIdx: SqPtr                                                                │
│      ├─► mask: UInt(8.W)                                                             │
│      ├─► pc: UInt                                                                    │
│      ├─◄ forwardMask: Vec(Bool, 8)  ← Which bytes forwarded                         │
│      ├─◄ forwardData: Vec(UInt(8), 8)  ← Forwarded data per byte                    │
│      └─◄ matchInvalid: Bool  ← VP match invalid                                     │
│                                                                                      │
│  LSQ (StoreQueue) Forward Query (S1→S2)                                              │
│    io.lsq.forward (S1 - Query)                                                       │
│      ├─► valid: Bool                                                                 │
│      ├─► vaddr: UInt(VAddrBits)                                                      │
│      ├─► paddr: UInt(PAddrBits)                                                      │
│      ├─► uop: MicroOp                                                                │
│      ├─► sqIdx: SqPtr                                                                │
│      ├─► sqIdxMask: UInt(StoreQueueSize)  ← Mask of older stores                    │
│      ├─► mask: UInt(8.W)                                                             │
│      ├─► pc: UInt                                                                    │
│      ├─◄ forwardMask: Vec(Bool, 8)  ← Which bytes forwarded (S2)                    │
│      ├─◄ forwardData: Vec(UInt(8), 8)  ← Forwarded data per byte (S2)               │
│      ├─◄ dataInvalid: Bool  ← Store data not ready (S2)                             │
│      ├─◄ dataInvalidFast: Bool  ← Fast check for data invalid (S1)                  │
│      ├─◄ addrInvalid: Bool  ← Store address not ready (S2)                          │
│      ├─◄ dataInvalidSqIdx: SqPtr  ← Which store has invalid data (S2)               │
│      ├─◄ addrInvalidSqIdx: SqPtr  ← Which store has invalid addr (S2)               │
│      └─◄ matchInvalid: Bool  ← VP match invalid (S3)                                │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘

┌─── Memory Ordering Violation Interfaces (S1→S2) ─────────────────────────────────────┐
│                                                                                      │
│  Store-Load Violation Check (from Store Pipelines to Load S1)                       │
│    io.stld_nuke_query[0..StorePipelineWidth-1] (Input from Store Units)             │
│      ├─◄ valid: Bool  ← Store is writing address in S1                              │
│      ├─◄ bits.robIdx: RobPtr  ← Store's ROB index                                   │
│      ├─◄ bits.paddr: UInt(PAddrBits)  ← Store's physical address                    │
│      └─◄ bits.mask: UInt(8.W)  ← Store's byte mask                                  │
│                                                                                      │
│      Detection Logic in S1:                                                          │
│        s1_nuke = any store where:                                                    │
│          - stld_nuke_query[i].valid == true                                          │
│          - isAfter(s1_load.robIdx, store.robIdx)  ← Load younger than store         │
│          - s1_paddr[PAddrBits-1:3] == store.paddr[PAddrBits-1:3]  ← Addr match      │
│          - (s1_mask & store.mask).orR  ← Byte overlap                               │
│                                                                                      │
│  Load-Load Violation Query (to LSQ/LoadQueueRAR)                                    │
│    io.lsq.ldld_nuke_query (S2 - Query)                                              │
│      ├─► req.valid: Bool                                                             │
│      ├─► req.bits.uop: MicroOp                                                       │
│      ├─► req.bits.mask: UInt(8.W)                                                    │
│      ├─► req.bits.paddr: UInt(PAddrBits)                                             │
│      ├─► req.bits.data_valid: Bool                                                   │
│      ├─◄ req.ready: Bool  ← RAR checker ready                                       │
│      ├─◄ resp.valid: Bool  ← Violation detected (S3)                                │
│      ├─◄ resp.bits.rep_frm_fetch: Bool  ← Need flush pipeline                       │
│      └─► revoke: Bool  ← Revoke query if exception/replay (S3)                      │
│                                                                                      │
│  Store-Load Violation Query (to LSQ/LoadQueueRAW)                                   │
│    io.lsq.stld_nuke_query (S2 - Query)                                              │
│      ├─► req.valid: Bool                                                             │
│      ├─► req.bits.uop: MicroOp                                                       │
│      ├─► req.bits.mask: UInt(8.W)                                                    │
│      ├─► req.bits.paddr: UInt(PAddrBits)                                             │
│      ├─► req.bits.data_valid: Bool                                                   │
│      ├─◄ req.ready: Bool  ← RAW checker ready                                       │
│      ├─◄ resp.valid: Bool  ← Violation detected                                     │
│      └─► revoke: Bool  ← Revoke query if exception/replay (S3)                      │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘

┌─── MSHR/Refill Forwarding Interfaces (S1→S2→S3) ─────────────────────────────────────┐
│                                                                                      │
│  TileLink D Channel Forward (for Super Replay)                                      │
│    io.tl_d_channel (S1→S2→S3)                                                        │
│      ├─► valid: Bool  ← Request D channel data                                      │
│      ├─► mshrid: UInt  ← MSHR entry ID                                              │
│      ├─► paddr: UInt(PAddrBits)                                                      │
│      ├─◄ forward(s1/s2_valid, mshrid, paddr): (Bool, UInt)                          │
│      │    ├─ fwd_valid: Bool  ← D channel data available                            │
│      │    └─ fwd_data: UInt(refillBits)  ← Refill data from D channel              │
│      │                                                                               │
│      Function: Forwards cache refill data directly from TileLink D channel          │
│                to minimize super replay latency                                     │
│                                                                                      │
│  MSHR Forward Request (S1)                                                           │
│    io.forward_mshr (S1 - Request to MSHR)                                            │
│      ├─► valid: Bool  ← Request MSHR forward                                        │
│      ├─► mshrid: UInt  ← MSHR entry ID                                              │
│      ├─► paddr: UInt(PAddrBits)                                                      │
│      └─◄ forward(): (Bool, Bool, UInt)  ← (data_valid, from_d, from_mshr, data)    │
│                                                                                      │
│  Cache Refill Notification (for wakeup)                                             │
│    io.refill                                                                         │
│      ├─◄ valid: Bool  ← Refill happening                                            │
│      ├─◄ bits.addr: UInt(PAddrBits)  ← Refilled address                             │
│      └─◄ bits.data: UInt  ← Refill data                                             │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘

┌─── Control & Configuration Interfaces ───────────────────────────────────────────────┐
│                                                                                      │
│  Redirect/Flush (All Stages)                                                        │
│    io.redirect                                                                       │
│      ├─◄ valid: Bool  ← Redirect occurred                                           │
│      ├─◄ bits.robIdx: RobPtr  ← Redirect ROB index                                  │
│      ├─◄ bits.level: RedirectLevel  ← Flush level                                   │
│      └─◄ bits.cfiUpdate: CfiUpdateInfo                                              │
│                                                                                      │
│      Used in all stages to kill instructions:                                       │
│        s0_kill, s1_kill, s2_kill, s3_kill based on robIdx.needFlush(redirect)       │
│                                                                                      │
│  CSR Control                                                                         │
│    io.csrCtrl                                                                        │
│      ├─◄ cache_error_enable: Bool  ← Enable cache error exception                   │
│      ├─◄ ldld_vio_check_enable: Bool  ← Enable load-load violation check            │
│      └─◄ (other CSR controls)                                                        │
│                                                                                      │
│  RS Tracking (from Reservation Station)                                             │
│    io.isFirstIssue                                                                   │
│      └─◄ Bool  ← First time issuing this instruction                                │
│                                                                                      │
│    io.rsIdx                                                                          │
│      └─◄ UInt  ← RS entry index for feedback                                        │
│                                                                                      │
│  TLB Hint (S2)                                                                       │
│    io.tlb_hint                                                                       │
│      ├─◄ id: UInt  ← TLB entry ID                                                   │
│      └─◄ full: Bool  ← TLB full                                                     │
│                                                                                      │
│  Load-to-Load Fast Match (S0, for pointer chasing)                                  │
│    io.ld_fast_imm (S0)                                                               │
│      └─◄ UInt(12.W)  ← Immediate offset for address calculation                     │
│                                                                                      │
│    io.ld_fast_match (S0)                                                             │
│      └─◄ Bool  ← Fast match for pointer chasing successful                          │
│                                                                                      │
│  Replay Queue Full                                                                   │
│    io.lq_rep_full                                                                    │
│      └─◄ Bool  ← LoadQueueReplay full, cannot accept more                           │
│                                                                                      │
│  Prefetch Correction                                                                 │
│    io.correctMissTrain                                                               │
│      └─◄ Bool  ← Prefetch accuracy correction signal                                │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘

╔═══════════════════════════════════════════════════════════════════════════════════════╗
║                              OUTPUT INTERFACES                                        ║
╚═══════════════════════════════════════════════════════════════════════════════════════╝

┌─── Writeback Outputs (S3) ───────────────────────────────────────────────────────────┐
│                                                                                      │
│  Register File Writeback                                                             │
│    io.ldout (S3 - ExuOutput to Backend)                                              │
│      ├─► valid: Bool  ← Load completes successfully                                 │
│      ├─► ready: Bool  ← Backend ready to accept                                     │
│      ├─► bits.uop: MicroOp  ← Complete micro-op                                     │
│      ├─► bits.data: UInt(XLEN)  ← Load data (aligned, sign-extended)                │
│      ├─► bits.redirectValid: Bool = false                                           │
│      ├─► bits.redirect: Redirect                                                    │
│      ├─► bits.debug.isMMIO: Bool                                                    │
│      ├─► bits.debug.paddr: UInt(PAddrBits)                                          │
│      ├─► bits.debug.vaddr: UInt(VAddrBits)                                          │
│      └─► bits.fflags: UInt                                                           │
│                                                                                      │
│      Conditions for valid:                                                           │
│        - s3_valid                                                                    │
│        - !s3_rep_info.need_rep  ← No replay needed                                  │
│        - !s3_mmio  ← Not MMIO (MMIO goes through uncache path)                      │
│                                                                                      │
│  VirtualLoadQueue Update                                                             │
│    io.lsq.ldin (S3 - Write to VirtualLoadQueue)                                      │
│      ├─► valid: Bool                                                                 │
│      ├─► bits.uop: MicroOp  ← Updated with exception info                           │
│      ├─► bits.paddr: UInt(PAddrBits)                                                 │
│      ├─► bits.vaddr: UInt(VAddrBits)                                                 │
│      ├─► bits.mask: UInt(8.W)                                                        │
│      ├─► bits.data: UInt(XLEN)  ← For exception reporting                           │
│      ├─► bits.tlbMiss: Bool                                                          │
│      ├─► bits.ptwBack: Bool                                                          │
│      ├─► bits.mmio: Bool                                                             │
│      ├─► bits.miss: Bool  ← Cache miss (updated with D channel forward)             │
│      ├─► bits.uop.cf.exceptionVec: Vec(Bool)  ← All exceptions                      │
│      ├─► bits.rep_info: ReplayInfo  ← Complete replay information                   │
│      │     ├─ need_rep: Bool                                                         │
│      │     ├─ cause: ReplayCause (mem_amb, tlb_miss, fwd_fail, dcache_miss, ...)   │
│      │     ├─ nuke: Bool                                                             │
│      │     ├─ data_inv_sq_idx: SqPtr                                                │
│      │     └─ addr_inv_sq_idx: SqPtr                                                │
│      ├─► bits.data_wen_dup: Vec(Bool, 6)  ← Which fields to update                  │
│      ├─► bits.replacementUpdated: Bool                                               │
│      ├─► bits.missDbUpdated: Bool                                                    │
│      ├─► bits.dcacheRequireReplay: Bool                                              │
│      └─► bits.forwardData/forwardMask: Forward info                                  │
│                                                                                      │
│  Uncache Load Output (from LSQ for MMIO)                                             │
│    io.lsq.uncache (Input to S3 - MMIO load data)                                     │
│      ├─◄ valid: Bool  ← Uncache load completes                                      │
│      ├─► ready: Bool = !s3_valid  ← Accept when S3 idle                             │
│      └─◄ bits: ExuOutput  ← MMIO load result                                        │
│                                                                                      │
│    io.lsq.ld_raw_data (Input to S3 - Raw MMIO data)                                  │
│      └─◄ LoadDataFromLQBundle  ← Raw data for MMIO loads                            │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘

┌─── Replay/Fast Path Outputs ─────────────────────────────────────────────────────────┐
│                                                                                      │
│  Fast Replay Queue Entry (S3)                                                        │
│    io.fast_rep_out (S3 - To Fast Replay Queue)                                       │
│      ├─► valid: Bool  ← Fast replay needed                                          │
│      ├─► bits: LqWriteBundle  ← Complete pipeline state                             │
│      ├─► bits.lateKill: Bool  ← Late kill flag                                      │
│      └─► bits.delayedLoadError: Bool  ← Delayed error flag                          │
│                                                                                      │
│      Conditions for fast replay:                                                     │
│        - s2_mq_nack  ← MSHR queue full                                              │
│        - s2_bank_conflict  ← Bank conflict                                          │
│        - s2_wpu_pred_fail  ← Way predictor fail                                     │
│        - s2_nuke  ← Store-load violation                                            │
│                                                                                      │
│  Load-to-Load Forwarding Output (S3)                                                 │
│    io.l2l_fwd_out (S3 - For Pointer Chasing)                                         │
│      ├─► valid: Bool  ← L2L forward valid                                           │
│      ├─► data: UInt(64.W)  ← Forwarded data (address for next load)                 │
│      └─► dly_ld_err: Bool  ← Error flag                                             │
│                                                                                      │
│      Conditions:                                                                     │
│        - s3_valid && !s3_mmio && !s3_rep_info.need_rep                              │
│        - EnableLoadToLoadForward enabled                                            │
│                                                                                      │
│  Fast Wakeup (S1→S2)                                                                 │
│    io.fast_uop (S2 - Speculative Wakeup)                                             │
│      ├─► valid: Bool  ← Can speculatively wake dependents                           │
│      └─► bits: MicroOp  ← Which instruction completing                              │
│                                                                                      │
│      Conditions (RegNext from S1):                                                   │
│        - !io.dcache.s1_disable_fast_wakeup                                          │
│        - s1_valid && !s1_kill                                                        │
│        - !io.tlb.resp.bits.miss                                                      │
│        - !io.lsq.forward.dataInvalidFast                                            │
│        AND (in S2):                                                                  │
│        - s2_valid && !s2_out.rep_info.need_rep && !s2_mmio                          │
│                                                                                      │
│  Pointer Chasing Status (S2)                                                         │
│    io.s2_ptr_chasing                                                                 │
│      └─► Bool  ← Pointer chasing successful in S1                                   │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘

┌─── Feedback & Exception Outputs ─────────────────────────────────────────────────────┐
│                                                                                      │
│  Fast Feedback to RS (S2)                                                            │
│    io.feedback_fast (S2)                                                             │
│      ├─► valid: Bool = false.B  ← Currently not used                                │
│      ├─► bits.hit: Bool                                                              │
│      ├─► bits.flushState: Bool                                                       │
│      ├─► bits.rsIdx: UInt                                                            │
│      └─► bits.sourceType: RSFeedbackType                                             │
│                                                                                      │
│  Slow Feedback to RS (S3)                                                            │
│    io.feedback_slow (S3)                                                             │
│      ├─► valid: Bool  ← Feedback valid                                              │
│      ├─► bits.hit: Bool  ← !need_rep || lsq.ldin.ready                              │
│      ├─► bits.flushState: Bool  ← PTW walk needed                                   │
│      ├─► bits.rsIdx: UInt  ← RS entry to update                                     │
│      ├─► bits.sourceType: RSFeedbackType.lrqFull                                     │
│      └─► bits.dataInvalidSqIdx: SqPtr                                                │
│                                                                                      │
│      Conditions:                                                                     │
│        - s3_valid && s3_fb_no_waiting                                               │
│        - !s3_isLoadReplay && !fast_rep && !feedbacked                               │
│                                                                                      │
│  Pipeline Rollback/Flush (S3)                                                        │
│    io.rollback (S3 - Redirect to Frontend)                                           │
│      ├─► valid: Bool  ← Rollback needed                                             │
│      ├─► bits.robIdx: RobPtr                                                         │
│      ├─► bits.ftqIdx: FtqPtr                                                         │
│      ├─► bits.ftqOffset: UInt                                                        │
│      ├─► bits.level: RedirectLevel  ← flush or flushAfter                           │
│      ├─► bits.cfiUpdate.target: UInt  ← PC to restart                               │
│      └─► bits.isRVC: Bool                                                            │
│                                                                                      │
│      Conditions:                                                                     │
│        - s3_valid && (s3_rep_frm_fetch || s3_flushPipe) && !s3_exception            │
│        - s3_rep_frm_fetch: VP match fail                                            │
│        - s3_flushPipe: Load-load violation                                          │
│                                                                                      │
│  Delayed Load Error (S3)                                                             │
│    io.s3_dly_ld_err                                                                  │
│      └─► Bool  ← Delayed load error (ECC error)                                     │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘

┌─── Prefetch Training Outputs ────────────────────────────────────────────────────────┐
│                                                                                      │
│  Prefetch Training (S2→S3)                                                           │
│    io.prefetch_train (S3 - To L2/L3 Prefetcher)                                      │
│      ├─► valid: Bool  ← Training valid                                              │
│      ├─► bits.paddr: UInt(PAddrBits)                                                 │
│      ├─► bits.vaddr: UInt(VAddrBits)                                                 │
│      ├─► bits.miss: Bool  ← RegNext(s2 cache miss)                                  │
│      ├─► bits.meta_prefetch: Bool  ← Was prefetch                                   │
│      └─► bits.meta_access: Bool  ← Access bit                                       │
│                                                                                      │
│    io.prefetch_train_l1 (S3 - To L1 Prefetcher)                                      │
│      └─► (same as prefetch_train)                                                   │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘

┌─── Debug/Topdown Outputs ────────────────────────────────────────────────────────────┐
│                                                                                      │
│  Topdown Performance Counters                                                        │
│    io.lsTopdownInfo.s1                                                               │
│      ├─► robIdx: UInt                                                                │
│      ├─► vaddr_valid: Bool                                                           │
│      └─► vaddr_bits: UInt                                                            │
│                                                                                      │
│    io.lsTopdownInfo.s2                                                               │
│      ├─► robIdx: UInt                                                                │
│      ├─► paddr_valid: Bool                                                           │
│      ├─► paddr_bits: UInt                                                            │
│      ├─► first_real_miss: Bool                                                       │
│      └─► cache_miss_en: Bool                                                         │
│                                                                                      │
│  Debug Load/Store Info                                                               │
│    io.debug_ls                                                                       │
│      └─► Various debug signals                                                       │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘

╔═══════════════════════════════════════════════════════════════════════════════════════╗
║                           INTERNAL PIPELINE FLOW                                      ║
╚═══════════════════════════════════════════════════════════════════════════════════════╝

  S0 (Input Arb)        S1 (TLB/Fwd)          S2 (Cache/Replay)      S3 (Data/WB)
  ──────────────        ────────────          ─────────────────      ────────────

  7 Sources          TLB Response           DCache Response         Data Merge
  Priority Arb   →   + PAddr            →   + Cache Data       →    + Alignment
  Address Gen        SB Forward Query       PMP Check               Sign Extend
  TLB Req            SQ Forward Query       Forward Merge
  DCache Req         Viol Check (S1)        Viol Query (S2)         Outputs:
  Align Check        Ptr Chasing Val        Replay Decision         - ldout (RF)
                     MSHR Fwd Req           Fast Wakeup             - lsq.ldin (VLQ)
                                                                    - fast_rep_out
                                                                    - l2l_fwd_out
                                                                    - rollback
                                                                    - feedback_slow

  Kill Conditions:
  s0: redirect       s1: fast_rep_kill      s2: redirect            s3: redirect
                         ptr_chase_cancel
                         redirect (×2)
```

**Summary**:
- **7 input sources** with strict priority arbitration in S0
- **TLB interface**: 2-port (req in S0, resp in S1), duplicated paddr outputs
- **DCache interface**: Multi-stage (req S0, s1_paddr, s1_kill, s2_kill, resp S2, data_delayed for S3)
- **Forwarding**: StoreBuffer (S1), StoreQueue (S1→S2), TileLink D (S1→S2→S3), MSHR (S1→S2)
- **Violation checks**: From stores (S1), to RAW/RAR checkers (S2), responses (S3)
- **5 major outputs**: Register writeback, VLQ update, fast replay, L2L forward, pipeline rollback
- **Fast wakeup**: Speculative in S2 based on S1 prediction
- **Feedback**: To RS in S3 for replay/miss handling

This shows the complete microarchitecture of a single Load Unit with all interfaces.

### 4. Store Pipeline Data Flow

```
Store Unit [0]              Store Unit [1]
     │                           │
     ├─► storeAddrIn[0]         ├─► storeAddrIn[1]
     │   (S1, address)          │   (S1, address)
     │                          │
     ├─► storeMaskIn[0]         ├─► storeMaskIn[1]
     │   (S0, mask)             │   (S0, mask)
     │                          │
     └─► storeDataIn[0]         └─► storeDataIn[1]
         (S0, data from RS)         (S0, data from RS)
                 │
                 ▼
           StoreQueue
         ┌───┬───┬───┐
         │   │   │   │
         │   │   │   └─► dataModule (data storage)
         │   │   └─────► vaddrModule (vaddr storage)
         │   └─────────► paddrModule (paddr storage)
         └─────────────► uop + flags
                 │
                 │ (after commit)
                 ▼
            SBuffer
         (Store Buffer)
```

### 5. Load-Store Forwarding

```
Load Unit [0]                Load Unit [1]
     │                            │
     │ forward query              │ forward query
     ├─► sqIdx                    ├─► sqIdx
     ├─► paddr                    ├─► paddr
     ├─► vaddr                    ├─► vaddr
     │                            │
     └────────┬───────────────────┘
              │
              ▼
      StoreQueue Forward Logic
              │
      ┌───────┴────────┐
      │                │
      ▼                ▼
  paddrModule     vaddrModule
   CAM Match       CAM Match
      │                │
      └───────┬────────┘
              │ match masks
              ▼
         dataModule
        read matching
         store data
              │
              ▼
     forwardMask[0] ────► Load Unit [0]
     forwardData[0]
     dataInvalid[0]
     addrInvalid[0]
              │
     forwardMask[1] ────► Load Unit [1]
     forwardData[1]
     dataInvalid[1]
     addrInvalid[1]
```

### 6. Memory Ordering Violation Detection

```
Load Unit [0/1] (S2)
      │
      ├─► stld_nuke_query[i].req
      │   - vaddr, paddr, mask
      │
      ▼
LoadQueueRAW (Read-After-Write violation checker)
      │ check against StoreQueue
      │
      ├─◄ storeAddrIn[0/1] (new stores)
      ├─◄ stAddrReadySqPtr
      ├─◄ stIssuePtr
      │
      ├─► resp (to Load Unit)
      │   - violation detected?
      │   - need replay?
      │
      └─► rollback (if violation)
          - redirect signal
          - flush younger instructions

Load Unit [0/1] (S2)
      │
      ├─► ldld_nuke_query[i].req
      │   - check load-load ordering
      │
      ▼
LoadQueueRAR (Read-After-Read violation checker)
      │
      ├─◄ release (from DCache)
      ├─◄ ldWbPtr
      │
      └─► resp (to Load Unit)
```

### 7. Commit Path

```
ROB (Reorder Buffer)
      │
      ├─► lcommit (# of loads committing)
      ├─► scommit (# of stores committing)
      │
      ▼
LSQWrapper
      │
      ├──────────────┬──────────────┐
      ▼              ▼              ▼
VirtualLoadQueue  StoreQueue
      │              │
      │              │ (committed stores)
      │              ▼
      │         DataBuffer
      │              │
      │              ▼
      │         SBuffer[0] ────┐
      │         SBuffer[1] ────┤
      │                        │
      │              ┌─────────┘
      │              ▼
      │         DCache Store Interface
      │
      ├─► deqPtr update
      ├─► allocated[] = false
      │
      ▼
  lqDeq count ────────────► Backend
  lqCancelCnt ────────────► Backend
```

## Backpressure and Flow Control

### Enqueue Backpressure

```
VirtualLoadQueue:
  validCount = distanceBetween(enqPtr, deqPtr)
  allowEnqueue = validCount <= (VirtualLoadQueueSize - LoadPipelineWidth)
  io.enq.canAccept := allowEnqueue
                           ▲
                           │
StoreQueue:                │
  validCount = distanceBetween(enqPtr, deqPtr)
  allowEnqueue = validCount <= (StoreQueueSize - StorePipelineWidth)
  io.enq.canAccept := allowEnqueue
                           │
                           │
                    ┌──────┴──────┐
                    │             │
                    ▼             ▼
              lqCanAccept   sqCanAccept
                    │             │
                    └──────┬──────┘
                           │
                           ▼
            Final Accept = lqCanAccept && sqCanAccept
                           │
                           ▼
                    Rename/Dispatch
                  (gates instruction dispatch)
```

### Replay Backpressure

```
LoadQueueReplay
      │
      ├─► lqFull (replay queue full)
      │
      ▼
Load Units
      │
      ├─► Block new loads if replay queue full
      └─► Prioritize replays over new requests
```

## MUX and Arbitration Details

### Load Writeback MUX

```
Load Unit 0 ──┐
ldout.valid   │
ldout.bits    │     ┌─────────────┐
              ├────►│             │
Atomics Unit ─┘     │  MUX (OR)   │──► writeback[0]
  out.valid         │             │
  out.bits          └─────────────┘
                    Priority: Atomics > Load

Load Unit 1 ────────────────────────► writeback[1]
  ldout
```

### Store Forward Query Arbitration

```
For each Load Unit [i]:
    ┌─────────────────────────────────────┐
    │   Forward Query to StoreQueue       │
    │                                     │
    │   Inputs:                           │
    │   - sqIdx (youngest visible store)  │
    │   - paddr, vaddr, mask              │
    │                                     │
    │   CAM Matching:                     │
    │   ┌──────────────┐                  │
    │   │ For each SQ  │                  │
    │   │ entry from   │                  │
    │   │ deqPtr to    │                  │
    │   │ sqIdx:       │                  │
    │   │              │                  │
    │   │ - Match addr │──► forwardMask   │
    │   │ - Check mask │──► forwardData   │
    │   │ - Get data   │──► dataInvalid   │
    │   └──────────────┘    addrInvalid   │
    │                                     │
    └─────────────────────────────────────┘
```

### Commit Arbitration

```
Stores in StoreQueue:
    committed && !mmio ────┐
                           │
                           ▼
                    ┌──────────────┐
    rdataPtr[0] ───►│              │
    rdataPtr[1] ───►│  Priority    │──► DataBuffer[0]
                    │  Encoder     │──► DataBuffer[1]
                    │              │
                    │ (oldest      │
                    │  first)      │
                    └──────────────┘
                           │
                           ▼
                    SBuffer[0] (EnsbufferWidth = 2)
                    SBuffer[1]
```

## Signal Width Summary

| Signal | Width | Direction | Description |
|--------|-------|-----------|-------------|
| enq.req | LsExuCnt | Input | MicroOp requests from dispatch |
| enq.needAlloc | LsExuCnt × 2 | Input | Allocation flags (LQ/SQ) |
| enq.resp | LsExuCnt | Output | Allocated queue indices |
| ldin | LoadPipelineWidth | Input | Load writeback from S3 |
| ldout | LoadPipelineWidth | Output | Load result to ROB |
| storeAddrIn | StorePipelineWidth | Input | Store address from S1 |
| storeDataIn | StorePipelineWidth | Input | Store data from S0 |
| replay | LoadPipelineWidth | Output | Replay requests |
| forward | LoadPipelineWidth | Input | Forward queries |

---

*Document Status: Initial version covering Rename to Write-back stages*
*Last Updated: 2026-02-03*
