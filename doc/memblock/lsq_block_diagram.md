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
- **VirtualLoadQueueSize**: 80 (default, configurable)
- **StoreQueueSize**: 64 (default, configurable)

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

### 2. Load Unit Complete Interface Block Diagram

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

### 3. Store Pipeline Data Flow

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

### 4. Load-Store Forwarding

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

### 5. Memory Ordering Violation Detection

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

### 6. Commit Path

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
