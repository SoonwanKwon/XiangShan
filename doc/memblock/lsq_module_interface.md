# LSQ Module Interface and Functionality

## Overview
This document describes each module in the LSQ subsystem from two perspectives:
1. **Interface Description**: Input/output ports and their semantics
2. **Functionality**: What the module does and how it operates

---

## 1. LsqWrapper (Top-Level LSQ Module)

### Interface Description

**File**: [src/main/scala/xiangshan/mem/lsqueue/LSQWrapper.scala](../../../src/main/scala/xiangshan/mem/lsqueue/LSQWrapper.scala)

#### Input Interfaces

| Port | Type | Width/Size | Source | Description |
|------|------|------------|--------|-------------|
| `hartId` | Input | UInt(8.W) | Top | Hardware thread ID |
| `brqRedirect` | Flipped(ValidIO) | Redirect | Backend | Branch misprediction/exception redirect |
| `enq` | LsqEnqIO | - | Dispatch | Enqueue interface from rename/dispatch |
| `enq.needAlloc` | Vec(Input) | LsExuCnt × UInt(2.W) | Dispatch | [1]=need LQ, [0]=need SQ |
| `enq.req` | Vec(Flipped(ValidIO)) | LsExuCnt × MicroOp | Dispatch | Instruction requests |
| `ldu.stld_nuke_query` | Vec(Flipped) | LoadPipelineWidth | Load Units | Store-Load violation check (S2) |
| `ldu.ldld_nuke_query` | Vec(Flipped) | LoadPipelineWidth | Load Units | Load-Load violation check (S2) |
| `ldu.ldin` | Vec(Flipped(Decoupled)) | LoadPipelineWidth | Load Units | Load results from S3 |
| `sta.storeMaskIn` | Vec(Flipped(Valid)) | StorePipelineWidth | Store Units | Store mask from S0 |
| `sta.storeAddrIn` | Vec(Flipped(Valid)) | StorePipelineWidth | Store Units | Store address from S1 |
| `sta.storeAddrInRe` | Vec(Input) | StorePipelineWidth | Store Units | Store addr replenish from S2 (MMIO info) |
| `std.storeDataIn` | Vec(Flipped(Valid)) | StorePipelineWidth | RS | Store data from reservation station |
| `forward` | Vec(Flipped) | LoadPipelineWidth | Load Units | Store-to-load forward query |
| `rob` | Flipped(RobLsqIO) | - | ROB | ROB commit interface |
| `rob.lcommit` | Input | UInt | ROB | Number of loads committing |
| `rob.scommit` | Input | UInt | ROB | Number of stores committing |
| `rob.pendingld` | Input | Bool | ROB | ROB head is pending load |
| `rob.pendingst` | Input | Bool | ROB | ROB head is pending store |
| `release` | Flipped(Valid) | Release | DCache | Cache line release notification |
| `refill` | Flipped(Valid) | Refill | DCache | Cache refill notification |
| `tl_d_channel` | Input | DcacheToLduForwardIO | DCache | TileLink D channel for forwarding |
| `uncacheOutstanding` | Input | Bool | MemBlock | Uncache outstanding enable |

#### Output Interfaces

| Port | Type | Width/Size | Destination | Description |
|------|------|------------|-------------|-------------|
| `enq.canAccept` | Output | Bool | Dispatch | Can accept new instructions |
| `enq.resp` | Vec(Output) | LsExuCnt × LSIdx | Dispatch | Allocated LQ/SQ indices |
| `ldout` | Vec(DecoupledIO) | LoadPipelineWidth | Backend | Load writeback to ROB |
| `ld_raw_data` | Vec(Output) | LoadPipelineWidth | Backend | Raw load data for exceptions |
| `replay` | Vec(Decoupled) | LoadPipelineWidth | Load Units | Load replay requests |
| `sbuffer` | Vec(Decoupled) | EnsbufferWidth | SBuffer | Committed stores to store buffer |
| `nuke_rollback` | Output(Valid) | Redirect | Backend | Store-Load violation rollback |
| `nack_rollback` | Output(Valid) | Redirect | Backend | Uncache NACK rollback |
| `mmioStout` | DecoupledIO | ExuOutput | Backend | MMIO store writeback |
| `sqEmpty` | Output | Bool | SBuffer | Store queue is empty |
| `lqFull` | Output | Bool | Dispatch | Load queue is full |
| `sqFull` | Output | Bool | Dispatch | Store queue is full |
| `lq_rep_full` | Output | Bool | Load Units | Load replay queue is full |
| `lqCanAccept` | Output | Bool | Dispatch | Load queue can accept |
| `sqCanAccept` | Output | Bool | Dispatch | Store queue can accept |
| `lqCancelCnt` | Output | UInt | Backend | Number of cancelled loads |
| `sqCancelCnt` | Output | UInt | Backend | Number of cancelled stores |
| `lqDeq` | Output | UInt | Backend | Number of loads dequeued |
| `sqDeq` | Output | UInt | Backend | Number of stores dequeued |
| `issuePtrExt` | Output | SqPtr | Backend | Store issue pointer |

### Functionality

**Purpose**: Top-level wrapper that instantiates and connects LoadQueue and StoreQueue modules.

**Operation**:

### 1. Enqueue Coordination and Backpressure Mechanism

**Code Reference**: [LSQWrapper.scala:118-140](../../../src/main/scala/xiangshan/mem/lsqueue/LSQWrapper.scala#L118-L140)

#### A. Joint Backpressure Logic

**Why Joint Backpressure is Necessary**:

The LSQ enforces **joint backpressure**, meaning dispatch can only enqueue instructions when **BOTH** LoadQueue AND StoreQueue have available space, regardless of whether a specific instruction needs LQ, SQ, or both.

```scala
// LSQWrapper.scala:121
io.enq.canAccept := loadQueue.io.enq.canAccept && storeQueue.io.enq.canAccept
```

**Reason**: **Maintain Program Order for Memory Operations**

Memory instructions must enter the LSQ in strict program order to ensure:
1. **Correct memory ordering semantics** (TSO for RISC-V)
2. **Proper dependency tracking** between loads and stores
3. **Accurate rollback** on violations or mispredictions

**Example of Why Joint Backpressure is Required**:

```
Scenario WITHOUT joint backpressure (BROKEN):

Instruction Stream:  LOAD1 → STORE1 → LOAD2 → STORE2
                       |       |        |        |
                      lq=0    sq=0     lq=1    sq=1

Cycle 1: LQ has space, SQ is FULL
  - Dispatch accepts LOAD1 (only needs LQ)  ← lq[0] = LOAD1
  - Dispatch rejects STORE1 (needs SQ, but SQ full)  ← STUCK!

Cycle 2: LOAD2 arrives at dispatch
  - Dispatch would want to accept LOAD2 (LQ has space)  ← lq[1] = LOAD2
  - But STORE1 is still stuck!

Result: LOAD2 enters LSQ BEFORE STORE1 → PROGRAM ORDER VIOLATED!
        This breaks memory ordering and dependency tracking.
```

```
Scenario WITH joint backpressure (CORRECT):

Instruction Stream:  LOAD1 → STORE1 → LOAD2 → STORE2

Cycle 1: LQ has space, SQ is FULL
  - canAccept = (LQ ready && SQ ready) = (true && false) = FALSE
  - Dispatch rejects ALL instructions (including LOAD1)

Cycle 2: SQ has space
  - canAccept = (LQ ready && SQ ready) = (true && true) = TRUE
  - Dispatch accepts LOAD1, STORE1, LOAD2, STORE2 in order

Result: Program order maintained!
```

#### B. Individual Queue Readiness Signals

While `canAccept` enforces joint backpressure for dispatch, the LSQWrapper also exports individual queue status:

```scala
// LSQWrapper.scala:122-123
io.lqCanAccept := loadQueue.io.enq.canAccept
io.sqCanAccept := storeQueue.io.enq.canAccept
```

These signals are used for:
- **Performance monitoring**: Track which queue is the bottleneck
- **Cross-queue coordination**: Each queue knows if the other is ready

```scala
// LSQWrapper.scala:124-125
loadQueue.io.enq.sqCanAccept := storeQueue.io.enq.canAccept
storeQueue.io.enq.lqCanAccept := loadQueue.io.enq.canAccept
```

This cross-connection allows each queue to know when the joint condition is satisfied.

#### C. needAlloc Encoding and Routing

**Code Reference**: [Dispatch2Rs.scala:210](../../../src/main/scala/xiangshan/backend/dispatch/Dispatch2Rs.scala#L210)

The `needAlloc` signal (2 bits per instruction) determines which queue(s) to allocate:

```scala
// Dispatch2Rs.scala:210
enqLsq.needAlloc(i) := Mux(io.in(i).valid && isLs(i),
                           Mux(isStore(i) && !isAMO(i), 2.U, 1.U),
                           0.U)
```

**Encoding**:
- `needAlloc = 0` (0b00): No allocation (not a memory instruction)
- `needAlloc = 1` (0b01): **Load or AMO** → Allocate **LQ only** (bit[0]=1)
- `needAlloc = 2` (0b10): **Store (not AMO)** → Allocate **SQ only** (bit[1]=1)
- `needAlloc = 3` (0b11): **Both LQ and SQ** (currently unused, reserved)

**Routing in LSQWrapper**:

```scala
// LSQWrapper.scala:127-136
for (i <- 0 until LsExuCnt) {
  // Bit [0] controls LoadQueue allocation
  loadQueue.io.enq.needAlloc(i) := io.enq.needAlloc(i)(0)
  loadQueue.io.enq.req(i).valid := io.enq.needAlloc(i)(0) && io.enq.req(i).valid
  loadQueue.io.enq.req(i).bits := io.enq.req(i).bits
  loadQueue.io.enq.req(i).bits.sqIdx := storeQueue.io.enq.resp(i)  ← Cross-assign

  // Bit [1] controls StoreQueue allocation
  storeQueue.io.enq.needAlloc(i) := io.enq.needAlloc(i)(1)
  storeQueue.io.enq.req(i).valid := io.enq.needAlloc(i)(1) && io.enq.req(i).valid
  storeQueue.io.enq.req(i).bits := io.enq.req(i).bits
  storeQueue.io.enq.req(i).bits.lqIdx := loadQueue.io.enq.resp(i)  ← Cross-assign
}
```

**Key Point**: Each instruction gets **both lqIdx and sqIdx**, even if it only uses one:
- **Loads**: Get valid lqIdx (for tracking), sqIdx indicates "older stores up to this point"
- **Stores**: Get valid sqIdx (for tracking), lqIdx indicates "older loads up to this point"

#### D. Response Aggregation

```scala
// LSQWrapper.scala:138-139
io.enq.resp(i).lqIdx := loadQueue.io.enq.resp(i)
io.enq.resp(i).sqIdx := storeQueue.io.enq.resp(i)
```

Every dispatched memory instruction receives:
- `lqIdx`: Load queue index (valid if load/AMO, marker if store)
- `sqIdx`: Store queue index (valid if store, marker if load/AMO)

**Usage**:
- **Load**: Uses lqIdx for tracking in VLQ, sqIdx marks older stores for forwarding checks
- **Store**: Uses sqIdx for tracking in SQ, lqIdx marks older loads for violation checks

---

### 2. Pipeline Integration

- Routes load writebacks to VirtualLoadQueue
- Routes store address/data to StoreQueue
- Manages violation detection coordination between LQ and SQ

### 3. Commit Coordination

- Distributes ROB commit signals to both queues
- Aggregates dequeue counts and cancel counts

### 4. Exception Handling

- Collects rollback signals from both queues
- Arbitrates between multiple violation sources

---

## 2. VirtualLoadQueue

### Interface Description

**File**: [src/main/scala/xiangshan/mem/lsqueue/VirtualLoadQueue.scala](../../../src/main/scala/xiangshan/mem/lsqueue/VirtualLoadQueue.scala:28)

#### Input Interfaces

| Port | Type | Width | Source | Description |
|------|------|-------|--------|-------------|
| `redirect` | Flipped(Valid) | Redirect | Backend | Flush signal for misprediction |
| `enq.needAlloc` | Vec(Input) | LsExuCnt × Bool | Dispatch | Which instructions need LQ entry |
| `enq.req` | Vec(Flipped(ValidIO)) | LsExuCnt × MicroOp | Dispatch | Load/Store instruction MicroOps |
| `enq.sqCanAccept` | Input | Bool | StoreQueue | SQ can accept (for joint backpressure) |
| `ldin` | Vec(Flipped(Decoupled)) | LoadPipelineWidth × LqWriteBundle | Load Units | Load results from pipeline S3 |
| `ldin.bits.uop.lqIdx` | - | LqPtr | Load Units | LQ index of this load |
| `ldin.bits.miss` | - | Bool | Load Units | DCache miss |
| `ldin.bits.tlbMiss` | - | Bool | Load Units | TLB miss |
| `ldin.bits.mmio` | - | Bool | Load Units | MMIO access |
| `ldin.bits.paddr` | - | UInt(PAddrBits) | Load Units | Physical address |
| `ldin.bits.vaddr` | - | UInt(VAddrBits) | Load Units | Virtual address |
| `ldin.bits.mask` | - | UInt | Load Units | Byte mask |
| `ldin.bits.rep_info.need_rep` | - | Bool | Load Units | Need replay |

#### Output Interfaces

| Port | Type | Width | Destination | Description |
|------|------|-------|-------------|-------------|
| `enq.canAccept` | Output | Bool | Dispatch | LQ has space for new instructions |
| `enq.resp` | Vec(Output) | LsExuCnt × LqPtr | Dispatch | Allocated LQ index for each inst |
| `ldWbPtr` | Output | LqPtr | LoadQueue modules | Pointer to oldest non-committed load |
| `lqFull` | Output | Bool | Dispatch | LQ is full, block dispatch |
| `lqEmpty` | Output | Bool | DCache | LQ is empty |
| `lqDeq` | Output | UInt | Backend | Number of loads committed this cycle |
| `lqCancelCnt` | Output | UInt | Backend | Number of loads flushed |

#### Entry Structure

Each VirtualLoadQueue entry (indexed by `LqPtr.value`) contains:

```scala
// Control flags
allocated[i]: Bool              // Entry is allocated to an instruction
addrvalid[i]: Bool              // Address has been calculated
datavalid[i]: Bool              // Data is ready (hit or forwarded)

// Instruction information
uop[i]: MicroOp                 // Complete micro-operation
  ├─ lqIdx: LqPtr              // This entry's LQ index
  ├─ sqIdx: SqPtr              // Corresponding SQ index
  ├─ robIdx: RobPtr            // ROB index
  ├─ pdest: UInt               // Physical destination register
  ├─ ctrl: CtrlSignals         // Control signals (fuOpType, etc.)
  └─ cf: CtrlFlow              // Control flow info (PC, exception vec)

// Debug information
debug_mmio[i]: Bool             // Is MMIO access
debug_paddr[i]: UInt(PAddrBits) // Physical address for debug
```

#### Pointer Structure

```scala
enqPtrExt[0..LsExuCnt-1]: LqPtr  // Enqueue pointers (multiple for multi-dispatch)
deqPtr: LqPtr                     // Dequeue pointer (commit pointer)

// Circular queue pointer (flag for wrap-around)
LqPtr {
  flag: Bool              // Wrap-around flag
  value: UInt(log2Up(VirtualLoadQueueSize))  // Index into queue
}
```

### Functionality

**Purpose**: Tracks load instruction state from dispatch to commit. Does NOT store data/addresses (those are in LoadQueueReplay and exception buffer).

**Key Parameters**:
- **RenameWidth**: 6 - Maximum instructions renamed per cycle
- **LsDqDeqWidth**: 4 - LS Dispatch Queue dequeue width (dispatch throughput)
- **LsExuCnt**: 4 - LSQ enqueue interface width (**matches LsDqDeqWidth**)
- **LduCnt**: 2 - Number of load execution pipelines
- **StuCnt**: 2 - Number of store execution pipelines (execution capability, not dispatch width)
- **VirtualLoadQueueSize**: 80 (default queue depth)
- **CommitWidth**: 6 - Maximum instructions committable per cycle

**Important Design Note**:
LsExuCnt = 4 is determined by **LsDqDeqWidth (dispatch throughput bottleneck)**, NOT by LduCnt+StuCnt.
Although LduCnt+StuCnt also equals 4, this is a balanced design choice, not a causal requirement.
Theoretically, LsDqDeqWidth could be set to 6 (matching RenameWidth) for higher dispatch throughput.

**Operation**:

#### 1. Enqueue (Dispatch Stage)

**Complete Pipeline Flow**:
```
Rename (6-wide, RenameWidth)
    ↓
Dispatch (6-wide)
    ↓
LsDq (LS Dispatch Queue)
  - Enqueue: 6-wide (from Rename)
  - Dequeue: 4-wide (LsDqDeqWidth) ← Dispatch throughput bottleneck
    ↓
LsDispatch2Rs (4-wide)
    ↓
LSQ Enqueue (4-wide, LsExuCnt)
```

**Maximum Enqueue Capacity**: Up to **4 instructions per cycle** can be enqueued from LsDq to LSQ.

**Internal Queue Logic**:

**A. Multiple Enqueue Pointer Management** ([VirtualLoadQueue.scala:71-72](../../../src/main/scala/xiangshan/mem/lsqueue/VirtualLoadQueue.scala#L71-L72)):
```scala
// Maintain 4 enqueue pointers simultaneously
val enqPtrExt = RegInit(VecInit((0 until io.enq.req.length).map(_.U.asTypeOf(new LqPtr))))
//                                 ^^^^^^^^^^^^^^^^^^^^^^^^
//                                 io.enq.req.length = LsExuCnt = 4

// enqPtrExt[0] = base pointer
// enqPtrExt[1] = base + 1
// enqPtrExt[2] = base + 2
// enqPtrExt[3] = base + 3
```

**B. Space Check**:
```scala
validCount = distanceBetween(enqPtrExt(0), deqPtr)
allowEnqueue = validCount <= (VirtualLoadQueueSize - LoadPipelineWidth)
canAccept = allowEnqueue
```

**C. Offset Calculation and Entry Allocation** ([VirtualLoadQueue.scala:143-163](../../../src/main/scala/xiangshan/mem/lsqueue/VirtualLoadQueue.scala#L143-L163)):
```scala
for (i <- 0 until io.enq.req.length) {  // i = 0, 1, 2, 3
  // Calculate offset: count how many prior requests need LQ allocation
  val offset = PopCount(io.enq.needAlloc.take(i))

  // Assign LQ index based on offset
  val lqIdx = enqPtrExt(offset)
  val index = io.enq.req(i).bits.lqIdx.value

  // Allocate entry if valid and not cancelled
  when (canEnqueue(i) && !enqCancel(i)) {
    allocated(index) := true.B
    uop(index) := io.enq.req(i).bits
    uop(index).lqIdx := lqIdx
    addrvalid[index] := false.B
    datavalid[index] := false.B
  }

  io.enq.resp(i) := lqIdx  // Return assigned index to dispatch
}
```

**Example**: If requests [0, 2, 3] are valid and need LQ allocation:
```
Request 0: offset = PopCount([]) = 0         → enqPtrExt[0]
Request 1: (invalid, skipped)
Request 2: offset = PopCount([req0]) = 1     → enqPtrExt[1]
Request 3: offset = PopCount([req0,req2]) = 2 → enqPtrExt[2]
```

**D. Pointer Update Logic** ([VirtualLoadQueue.scala:94-111](../../../src/main/scala/xiangshan/mem/lsqueue/VirtualLoadQueue.scala#L94-L111)):

Two cases for pointer update:

**Case 1: Normal Enqueue**
```scala
val enqNumber = Mux(io.enq.canAccept && io.enq.sqCanAccept,
                    PopCount(io.enq.req.map(_.valid)), 0.U)

// All 4 pointers advance by the same amount
enqPtrExtNextVec := VecInit(enqPtrExt.map(_ + enqNumber))
```

**Case 2: Redirect Recovery** (rollback cancelled instructions)
```scala
when (lastLastCycleRedirect.valid) {
  // Roll back pointers by number of cancelled instructions
  enqPtrExtNextVec := VecInit(enqPtrExt.map(_ - redirectCancelCount))
}

// redirectCancelCount =
//   PopCount(cancelled entries in queue) +
//   PopCount(cancelled entries being enqueued)
```

**Safety Check** (prevent overflow):
```scala
when (isAfter(enqPtrExtNextVec(0), deqPtrNext)) {
  enqPtrExtNext := enqPtrExtNextVec  // Normal case
} .otherwise {
  // Queue wrapped around, clamp to deqPtr
  enqPtrExtNext := VecInit((0 until io.enq.req.length).map(i => deqPtrNext + i.U))
}

enqPtrExt := enqPtrExtNext  // Register update
```

2. **Writeback Update (Load S3)**:
   ```scala
   for each load writeback i:
     index = ldin(i).bits.uop.lqIdx.value

     if (!need_replay):
       // Update validity flags
       addrvalid[index] := hasExceptions || !tlbMiss
       datavalid[index] := hasExceptions || mmio || (!miss && !dcacheRequireReplay)

       // Update uop fields if needed
       if (data_wen_dup[1]): uop[index].pdest := updated_pdest
       if (data_wen_dup[2]): uop[index].cf := updated_cf
       if (data_wen_dup[3]): uop[index].ctrl := updated_ctrl
   ```

#### 3. Commit (Dequeue)

**Dequeue Logic** ([VirtualLoadQueue.scala:113-130](../../../src/main/scala/xiangshan/mem/lsqueue/VirtualLoadQueue.scala#L113-L130)):

**Maximum Commit Capacity**: Up to **CommitWidth instructions per cycle** can be committed.

```scala
val DeqPtrMoveStride = CommitWidth  // Typically 6 for XiangShan
require(DeqPtrMoveStride == CommitWidth, "DeqPtrMoveStride must be equal to CommitWidth!")
```

**A. Find Ready-to-Commit Entries**:
```scala
// Check CommitWidth entries starting from deqPtr
val deqLookupVec = VecInit((0 until DeqPtrMoveStride).map(deqPtr + _.U))

// Entry is ready if: allocated, addr valid, data valid, and not equal to enqPtr
val deqLookup = VecInit(deqLookupVec.map(ptr =>
  allocated(ptr.value) &&
  addrvalid(ptr.value) &&
  datavalid(ptr.value) &&
  ptr =/= enqPtrExt(0)  // Not the same as enqueue pointer (queue not empty)
))

// Exclude entries being cancelled in this cycle
val deqInSameRedirectCycle = VecInit(deqLookupVec.map(ptr => needCancel(ptr.value)))
val deqCountMask = deqLookup.asUInt & ~deqInSameRedirectCycle.asUInt
```

**B. Count Consecutive Committable Entries** (using priority encoder):
```scala
// Find first non-committable entry using priority encoder
// Example: deqCountMask = 0b111100
//   → ~deqCountMask = 0b000011
//   → PriorityEncoderOH(0b000011) = 0b000001
//   → 0b000001 - 1 = 0b000000 (6'b0)
//   → PopCount(0b000000) = 0
//
// Example: deqCountMask = 0b111111
//   → ~deqCountMask = 0b000000
//   → PriorityEncoderOH(0b000000) = 0b000000
//   → 0b000000 - 1 = all 1s (underflow wraps)
//   → PopCount counts all bits before first 0
val commitCount = PopCount(PriorityEncoderOH(~deqCountMask) - 1.U)
```

**C. Deallocate Committed Entries**:
```scala
// Mark entries as deallocated
for (i <- 0 until DeqPtrMoveStride) {
  when (commitCount > i.U) {
    allocated((deqPtr + i.U).value) := false.B
  }
}
```

**D. Update Dequeue Pointer** (2-stage pipeline):
```scala
// Cycle 1: Generate next pointer
deqPtrNext := deqPtr + lastCommitCount

// Cycle 2: Update register
val deqPtrUpdateEna = lastCommitCount =/= 0.U
deqPtr := RegEnable(deqPtrNext, 0.U.asTypeOf(new LqPtr), deqPtrUpdateEna)

// Output to backend (3 cycles total from commit decision)
io.lqDeq := RegNext(lastCommitCount)
```

**Pipeline Timing**:
```
Cycle N:   Evaluate deqLookup → commitCount
Cycle N+1: deqPtrNext = deqPtr + commitCount
Cycle N+2: deqPtr updated, allocated bits cleared
Cycle N+3: io.lqDeq reports count to backend
```

4. **Flush on Redirect**:
   ```scala
   // Identify entries to cancel
   for i in 0 until VirtualLoadQueueSize:
     needCancel[i] := uop[i].robIdx.needFlush(redirect) && allocated[i]
     when (needCancel[i]):
       allocated[i] := false

   // Recover enqueue pointer (2 cycles after redirect)
   cancelCount = PopCount(needCancel) + enqCancelCount
   when (lastLastCycleRedirect.valid):
     enqPtrExt := enqPtrExt - redirectCancelCount
   ```

5. **Backpressure Control**:
   ```scala
   // Joint backpressure with Store Queue
   actualEnqueue = enq.canAccept && enq.sqCanAccept

   // Only update pointers if both queues accept
   enqNumber = Mux(actualEnqueue, PopCount(enq.req.valid), 0)
   ```

**Key Characteristics**:
- **Virtual**: Only tracks control state, not data/addresses
- **Circular Queue**: Uses flag bit to distinguish full vs empty
- **Multi-dispatch**: Can allocate multiple entries per cycle
- **In-order Commit**: Dequeue pointer moves to oldest ready load
- **Out-of-order Writeback**: Any load can writeback when ready

---

## 3. StoreQueue

### Interface Description

**File**: [src/main/scala/xiangshan/mem/lsqueue/StoreQueue.scala](../../../src/main/scala/xiangshan/mem/lsqueue/StoreQueue.scala:63)

#### Input Interfaces

| Port | Type | Width | Source | Description |
|------|------|-------|--------|-------------|
| `hartId` | Input | UInt(8.W) | Top | Hardware thread ID |
| `enq` | SqEnqIO | - | Dispatch | Enqueue interface |
| `enq.needAlloc` | Vec(Input) | LsExuCnt × Bool | Dispatch | Need SQ allocation |
| `enq.req` | Vec(Flipped(ValidIO)) | LsExuCnt × MicroOp | Dispatch | Store instructions |
| `enq.lqCanAccept` | Input | Bool | LoadQueue | LQ can accept |
| `brqRedirect` | Flipped(ValidIO) | Redirect | Backend | Branch redirect |
| `storeAddrIn` | Vec(Flipped(Valid)) | StorePipelineWidth | Store Units | Store address from S1 |
| `storeAddrIn.bits.uop.sqIdx` | - | SqPtr | Store Units | SQ index |
| `storeAddrIn.bits.paddr` | - | UInt(PAddrBits) | Store Units | Physical address |
| `storeAddrIn.bits.vaddr` | - | UInt(VAddrBits) | Store Units | Virtual address |
| `storeAddrIn.bits.mask` | - | UInt | Store Units | Byte mask |
| `storeAddrIn.bits.miss` | - | Bool | Store Units | DCache miss (for prefetch) |
| `storeAddrIn.bits.mmio` | - | Bool | Store Units | MMIO access |
| `storeAddrInRe` | Vec(Input) | StorePipelineWidth | Store Units | Replenish MMIO/atomic info (S2) |
| `storeDataIn` | Vec(Flipped(Valid)) | StorePipelineWidth | RS | Store data from S0 |
| `storeDataIn.bits.uop.sqIdx` | - | SqPtr | RS | SQ index |
| `storeDataIn.bits.data` | - | UInt(XLEN) | RS | Store data |
| `storeMaskIn` | Vec(Flipped(Valid)) | StorePipelineWidth | Store Units | Store mask from S0 |
| `forward` | Vec(Flipped) | LoadPipelineWidth | Load Units | Forward query interface |
| `forward.sqIdx` | - | SqPtr | Load Units | Youngest visible store |
| `forward.sqIdxMask` | - | UInt(StoreQueueSize) | Load Units | SQ entry mask |
| `forward.paddr` | - | UInt(PAddrBits) | Load Units | Query address |
| `forward.vaddr` | - | UInt(VAddrBits) | Load Units | Query vaddr |
| `forward.mask` | - | UInt | Load Units | Query byte mask |
| `forward.uop` | - | MicroOp | Load Units | Load uop (for store set) |
| `rob` | Flipped(RobLsqIO) | - | ROB | ROB interface |
| `rob.scommit` | - | UInt | ROB | Number of stores committing |
| `rob.pendingst` | - | Bool | ROB | ROB head is pending store |
| `uncache.resp` | Flipped(Decoupled) | - | Uncache | MMIO response |
| `uncacheOutstanding` | Input | Bool | MemBlock | Outstanding MMIO enable |

#### Output Interfaces

| Port | Type | Width | Destination | Description |
|------|------|-------|-------------|-------------|
| `enq.canAccept` | Output | Bool | Dispatch | SQ can accept new stores |
| `enq.resp` | Vec(Output) | LsExuCnt × SqPtr | Dispatch | Allocated SQ indices |
| `sbuffer` | Vec(Decoupled) | EnsbufferWidth | SBuffer | Committed stores |
| `sbuffer.bits.addr` | - | UInt(PAddrBits) | SBuffer | Physical address |
| `sbuffer.bits.vaddr` | - | UInt(VAddrBits) | SBuffer | Virtual address |
| `sbuffer.bits.data` | - | UInt(VLEN) | SBuffer | Store data |
| `sbuffer.bits.mask` | - | UInt(VLEN/8) | SBuffer | Byte mask |
| `mmioStout` | DecoupledIO | ExuOutput | Backend | MMIO store writeback |
| `forward.forwardMask` | - | Vec(UInt(8)) | Load Units | Forward byte mask |
| `forward.forwardData` | - | Vec(UInt(8)) | Load Units | Forward data |
| `forward.forwardMaskFast` | - | Vec(Bool) | Load Units | Fast forward mask (S1) |
| `forward.dataInvalid` | - | Bool | Load Units | Store data not ready |
| `forward.addrInvalid` | - | Bool | Load Units | Store addr not ready |
| `forward.matchInvalid` | - | Bool | Load Units | Vaddr/paddr mismatch |
| `forward.dataInvalidSqIdx` | - | SqPtr | Load Units | Which store has invalid data |
| `forward.addrInvalidSqIdx` | - | SqPtr | Load Units | Which store has invalid addr |
| `uncache.req` | Decoupled | - | Uncache | MMIO request |
| `exceptionAddr.vaddr` | Output | UInt(VAddrBits) | Backend | Exception address |
| `sqEmpty` | Output | Bool | SBuffer | SQ is empty |
| `sqFull` | Output | Bool | Dispatch | SQ is full |
| `sqCancelCnt` | Output | UInt | Backend | Cancelled store count |
| `sqDeq` | Output | UInt | Backend | Dequeued store count |
| `stAddrReadySqPtr` | Output | SqPtr | LoadQueue | Pointer to oldest addr-ready store |
| `stAddrReadyVec` | Output | Vec(Bool) × StoreQueueSize | LoadQueue | Which stores have addr ready |
| `stDataReadySqPtr` | Output | SqPtr | LoadQueue | Pointer to oldest data-ready store |
| `stDataReadyVec` | Output | Vec(Bool) × StoreQueueSize | LoadQueue | Which stores have data ready |
| `stIssuePtr` | Output | SqPtr | LoadQueue | Next store to issue (enqPtr) |
| `sqDeqPtr` | Output | SqPtr | LoadQueue | Dequeue pointer |
| `force_write` | Output | Bool | DCache | Force write to cache |

#### Entry Structure

Each StoreQueue entry contains:

```scala
// Control state (Register)
allocated[i]: Bool           // Entry allocated
addrvalid[i]: Bool           // Address computed
datavalid[i]: Bool           // Data ready
committed[i]: Bool           // ROB has committed this store
pending[i]: Bool             // MMIO pending (wait for ROB head)
mmio[i]: Bool                // Is MMIO access
atomic[i]: Bool              // Is atomic operation
prefetch[i]: Bool            // Trigger prefetch on commit

// Instruction (Register)
uop[i]: MicroOp              // Micro-operation

// Data (in separate modules for better timing)
paddrModule:
  - Physical address storage
  - CAM for forward matching
  - Line flag (whole cache line write)

vaddrModule:
  - Virtual address storage
  - CAM for forward matching

dataModule:
  - Store data storage
  - Byte mask storage
  - Forward data read ports
```

#### Pointer Structure

```scala
enqPtrExt[0..LsExuCnt-1]: SqPtr   // Enqueue pointers
deqPtrExt[0..EnsbufferWidth-1]: SqPtr  // Dequeue pointers (multiple for bandwidth)
cmtPtrExt[0..CommitWidth-1]: SqPtr     // Commit pointers
rdataPtrExt[0..EnsbufferWidth-1]: SqPtr // Data read pointers
addrReadyPtrExt: SqPtr                  // Oldest addr-not-ready store
dataReadyPtrExt: SqPtr                  // Oldest data-not-ready store
```

### Functionality

**Purpose**: Buffers stores from dispatch to memory system. Provides store-to-load forwarding and enforces memory ordering.

**Operation**:

1. **Enqueue (Dispatch Stage)**:
   ```scala
   // Check space
   validCount = distanceBetween(enqPtrExt(0), deqPtrExt(0))
   allowEnqueue = validCount <= (StoreQueueSize - StorePipelineWidth)
   canAccept = allowEnqueue

   // Allocate entries
   for each valid store i:
     index = enq.req(i).bits.sqIdx.value
     if (canEnqueue && !enqCancel):
       allocated[index] := true
       uop[index] := enq.req(i).bits
       uop[index].sqIdx := sqIdx
       datavalid[index] := false
       addrvalid[index] := false
       committed[index] := false
       pending[index] := false
       mmio[index] := false
   ```

2. **Address Writeback (Store S1)**:
   ```scala
   for each store addr writeback i:
     index = storeAddrIn(i).bits.uop.sqIdx.value
     when (storeAddrIn(i).fire):
       addrvalid[index] := !miss  // Valid if TLB hit

       // Write to address modules
       paddrModule.waddr(i) := index
       paddrModule.wdata(i) := paddr
       paddrModule.wmask(i) := mask
       paddrModule.wen(i) := true

       vaddrModule.waddr(i) := index
       vaddrModule.wdata(i) := vaddr
       vaddrModule.wmask(i) := mask
       vaddrModule.wen(i) := true

   // Replenish MMIO info (1 cycle later, after PMA/PMP check)
   when (RegNext(storeAddrIn(i).fire && !miss)):
     index_reg = RegNext(index)
     pending[index_reg] := storeAddrInRe(i).mmio
     mmio[index_reg] := storeAddrInRe(i).mmio
     atomic[index_reg] := storeAddrInRe(i).atomic
     prefetch[index_reg] := storeAddrInRe(i).miss
   ```

3. **Data Writeback (Store S0)**:
   ```scala
   // Data write is pipelined (2 stages)
   // S0: Send write request
   for each store data writeback i:
     index = storeDataIn(i).bits.uop.sqIdx.value
     when (storeDataIn(i).fire):
       dataModule.data.waddr(i) := index
       dataModule.data.wdata(i) := genWdata(data, fuOpType)
       dataModule.data.wen(i) := true

   // S1: Mark as valid
   when (RegNext(storeDataIn(i).fire)):
     datavalid[RegNext(index)] := true
   ```

4. **Store-to-Load Forwarding**:
   ```scala
   for each load forward query i:
     // Determine search range
     differentFlag = (deqPtr.flag != forward(i).sqIdx.flag)
     forwardMask1 = Mux(differentFlag, ~deqMask, deqMask ^ forward(i).sqIdxMask)
     forwardMask2 = Mux(differentFlag, forward(i).sqIdxMask, 0)

     // Find matching stores (address + data valid)
     canForward1 = forwardMask1 & allValidVec  // allValid = addr && data
     canForward2 = forwardMask2 & allValidVec

     // CAM match on address
     vaddrModule.forwardMdata(i) := forward(i).vaddr
     vaddrModule.forwardDataMask(i) := forward(i).mask
     // vaddrModule produces forwardMmask(i) via CAM

     paddrModule.forwardMdata(i) := forward(i).paddr
     paddrModule.forwardDataMask(i) := forward(i).mask
     // paddrModule produces forwardMmask(i) via CAM

     // Read data from matching stores
     dataModule.needForward(i)(0) := canForward1 & vaddrModule.forwardMmask(i)
     dataModule.needForward(i)(1) := canForward2 & vaddrModule.forwardMmask(i)
     // dataModule produces forwardMask(i) and forwardData(i)

     // Output (S2)
     forward(i).forwardMask := dataModule.forwardMask(i)  // Which bytes forwarded
     forward(i).forwardData := dataModule.forwardData(i)  // Forwarded data

     // Detect data not ready
     dataInvalidMask = (addrValid & ~dataValid & vaddrMatch & forwardMask)
     forward(i).dataInvalid := dataInvalidMask.orR

     // Detect addr not ready (store-set based)
     addrInvalidMask = (~addrValid & storeSetHit & forwardMask)
     forward(i).addrInvalid := addrInvalidMask.orR

     // Detect vaddr/paddr mismatch (requires replay)
     vpmaskNotEqual = (paddrMatch XOR vaddrMatch) & needForward
     forward(i).matchInvalid := vpmaskNotEqual != 0
   ```

5. **Commit (ROB)**:
   ```scala
   commitCount = RegNext(io.rob.scommit)

   for i in 0 until CommitWidth:
     when (commitCount > i):
       if (i == 0 && uncacheState != s_idle):
         // Don't commit MMIO until it completes
         skip
       else:
         committed[cmtPtrExt(i).value] := true

   cmtPtrExt := cmtPtrExt + commitCount
   ```

6. **Dequeue to SBuffer**:
   ```scala
   // Read data from queue (pipelined)
   for i in 0 until EnsbufferWidth:
     ptr = rdataPtrExt(i).value
     dataBuffer.enq(i).valid := allocated[ptr] && committed[ptr] && !mmio
     dataBuffer.enq(i).bits.addr := paddrModule.rdata(i)
     dataBuffer.enq(i).bits.data := dataModule.rdata(i).data
     dataBuffer.enq(i).bits.mask := dataModule.rdata(i).mask

   // Send to SBuffer
   for i in 0 until EnsbufferWidth:
     sbuffer(i) <> dataBuffer.deq(i)

     // Deallocate entry (delayed for store-load forwarding)
     when (RegNext(sbuffer(i).fire)):
       allocated[RegEnable(ptr, sbuffer(i).fire)] := false

   // Update pointers
   deqPtrExt := Mux(RegNext(sbuffer(1).fire), deqPtrExt + 2,
                    Mux(RegNext(sbuffer(0).fire), deqPtrExt + 1, deqPtrExt))
   ```

7. **MMIO Store Handling**:
   ```scala
   // State machine: s_idle -> s_req -> s_resp -> s_wb -> s_wait -> s_idle

   when (state == s_idle && ROB.pendingst && pending[deqPtr] &&
         allocated[deqPtr] && datavalid[deqPtr] && addrvalid[deqPtr]):
     state := s_req

   when (state == s_req && uncache.req.fire):
     pending[deqPtr] := false  // Allow commit
     state := Mux(uncacheOutstanding, s_wb, s_resp)

   when (state == s_resp && uncache.resp.fire):
     state := s_wb

   when (state == s_wb && mmioStout.fire):
     allocated[deqPtr] := false
     state := s_wait

   when (state == s_wait && commitCount > 0):
     state := s_idle
   ```

8. **Ready Pointer Updates**:
   ```scala
   // Track oldest addr-not-ready store
   IssuePtrMoveStride = 4
   addrReadyLookup = (0 until IssuePtrMoveStride).map(i =>
     allocated[(addrReadyPtr + i).value] &&
     (mmio[(addrReadyPtr + i).value] || addrvalid[(addrReadyPtr + i).value])
   )
   nextAddrReadyPtr = addrReadyPtr + PriorityEncoder(~addrReadyLookup :+ true)
   addrReadyPtr := nextAddrReadyPtr

   // Similarly for data ready pointer
   ```

**Key Characteristics**:
- **Physical Storage**: Stores actual addresses and data (unlike VirtualLoadQueue)
- **CAM-based Forwarding**: Parallel address matching for all loads
- **Multi-port**: Multiple stores can writeback per cycle
- **Separate Modules**: Data/addr in separate modules for timing
- **In-order Dequeue**: Committed stores leave in order
- **MMIO Serialization**: MMIO stores wait for ROB head

---

## 4. LoadQueueRAW (Read-After-Write Violation Checker)

### Interface Description

**File**: [src/main/scala/xiangshan/mem/lsqueue/LoadQueueRAW.scala](../../../src/main/scala/xiangshan/mem/lsqueue/LoadQueueRAW.scala)

#### Input Interfaces

| Port | Type | Source | Description |
|------|------|--------|-------------|
| `redirect` | Flipped(Valid(Redirect)) | Backend | Flush signal |
| `storeIn` | Vec(Flipped(Valid(LsPipelineBundle))) | Store Units | New store addresses (S1) |
| `stAddrReadySqPtr` | Input(SqPtr) | StoreQueue | Oldest addr-ready store |
| `stIssuePtr` | Input(SqPtr) | StoreQueue | Store issue pointer (enqPtr) |
| `query(i).req` | Flipped(Valid) | Load Unit i | Query from load S1 |
| `query(i).req.bits.uop.sqIdx` | - | Load Unit i | Load's youngest visible store |
| `query(i).req.bits.paddr` | - | Load Unit i | Load address |
| `query(i).req.bits.mask` | - | Load Unit i | Load mask |
| `query(i).revoke` | Input(Bool) | Load Unit i | Cancel query (from S3) |

#### Output Interfaces

| Port | Type | Destination | Description |
|------|------|-------------|-------------|
| `query(i).resp` | Valid | Load Unit i | Violation detected |
| `rollback` | Output(Valid(Redirect)) | Backend | Rollback redirect |
| `lqFull` | Output(Bool) | LoadQueue | RAW checker is full |

### Functionality

**Purpose**: Detect store-to-load (Read-After-Write) memory ordering violations. Ensures loads don't execute before older stores to the same address.

**Operation**:

1. **Query (Load S1)**:
   ```scala
   for each load query i:
     // Load claims it checked stores up to sqIdx
     // Check if any stores AFTER sqIdx (but before stIssuePtr) match this load

     searchRange = stores from (load.sqIdx + 1) to stIssuePtr

     for each store in searchRange:
       if (store.addrReady && store.addr matches load.addr):
         // Violation! This store came after the load thought it checked
         violation_detected := true
   ```

2. **Monitor New Stores (Store S1)**:
   ```scala
   // When new store address arrives, check against in-flight loads
   for each new store s:
     for each in-flight load query:
       if (store.sqIdx > load.sqIdx &&
           store.sqIdx <= load.sqIdx_checked &&
           address_match):
         // This store is younger than load but should have been checked
         violation := true
   ```

3. **Generate Rollback**:
   ```scala
   when (violation_detected):
     rollback.valid := true
     rollback.bits.robIdx := violating_load.robIdx
     rollback.bits.isReplay := false  // Full flush, not replay
   ```

**Key Characteristics**:
- **Parallel Checker**: Checks all loads against all new stores
- **CAM-based**: Content-addressable memory for address matching
- **Proactive**: Detects violations before load commits

---

## 5. LoadQueueRAR (Read-After-Read Violation Checker)

### Interface Description

**File**: [src/main/scala/xiangshan/mem/lsqueue/LoadQueueRAR.scala](../../../src/main/scala/xiangshan/mem/lsqueue/LoadQueueRAR.scala)

#### Input Interfaces

| Port | Type | Source | Description |
|------|------|--------|-------------|
| `redirect` | Flipped(Valid(Redirect)) | Backend | Flush signal |
| `release` | Flipped(Valid(Release)) | DCache | Cache line release (eviction) |
| `release.bits.paddr` | - | DCache | Released cache line address |
| `ldWbPtr` | Input(LqPtr) | VirtualLoadQueue | Oldest non-committed load |
| `query(i).req` | Flipped(Valid) | Load Unit i | Query from load S1 |
| `query(i).req.bits.paddr` | - | Load Unit i | Load address (cache line) |
| `query(i).revoke` | Input(Bool) | Load Unit i | Cancel query |

#### Output Interfaces

| Port | Type | Destination | Description |
|------|------|-------------|-------------|
| `query(i).resp` | Valid | Load Unit i | Violation/eviction detected |
| `lqFull` | Output(Bool) | LoadQueue | RAR checker full |

### Functionality

**Purpose**: Detect when a cache line accessed by an older load (not yet committed) is evicted. The older load must be replayed to ensure correctness.

**Operation**:

1. **Track Cache Lines (Load S1)**:
   ```scala
   for each load query i:
     // Record cache line addresses of loads in-flight
     cache_line = load.paddr[PAddrBits-1:6]  // Extract cache line addr

     track[load.lqIdx] := cache_line
   ```

2. **Check Evictions (DCache Release)**:
   ```scala
   when (release.valid):
     released_line = release.bits.paddr[PAddrBits-1:6]

     for each in-flight load from ldWbPtr to newest:
       if (tracked[load.lqIdx] == released_line):
         // This load accessed the evicted line
         // Younger loads might have read stale forwarded data
         violation := true
         violating_lqIdx := load.lqIdx
   ```

3. **Signal Violation**:
   ```scala
   // Inform loads in S2 that their data might be stale
   for each load query i:
     query(i).resp.valid := (query.lqIdx matches violating_lqIdx)
   ```

**Key Characteristics**:
- **Cache-line Granularity**: Tracks cache lines, not individual addresses
- **Conservative**: Any eviction causes replay of loads to that line
- **Handles Store-Load Forwarding**: Prevents forwarded data from becoming stale

---

## 6. LoadQueueReplay (Replay Queue)

### Interface Description

**File**: [src/main/scala/xiangshan/mem/lsqueue/LoadQueueReplay.scala](../../../src/main/scala/xiangshan/mem/lsqueue/LoadQueueReplay.scala)

#### Input Interfaces

| Port | Type | Source | Description |
|------|------|--------|-------------|
| `redirect` | Flipped(Valid(Redirect)) | Backend | Flush signal |
| `enq` | Vec(Flipped(Decoupled(LqWriteBundle))) | Load Units | Load results (S3) |
| `enq(i).bits.rep_info.need_rep` | - | Load Units | This load needs replay |
| `enq(i).bits.rep_info.cause` | - | Load Units | Replay reason (TLB, cache, etc.) |
| `refill` | Flipped(Valid(Refill)) | DCache | Cache refill event |
| `tl_d_channel` | Input(DcacheToLduForwardIO) | DCache | TileLink D channel |
| `storeAddrIn` | Vec(Flipped(Valid)) | Store Units | New store addresses |
| `storeDataIn` | Vec(Flipped(Valid)) | Store Units | New store data |
| `stAddrReadySqPtr` | Input(SqPtr) | StoreQueue | Addr ready pointer |
| `stDataReadySqPtr` | Input(SqPtr) | StoreQueue | Data ready pointer |
| `sqEmpty` | Input(Bool) | StoreQueue | SQ is empty |
| `ldWbPtr` | Input(LqPtr) | VirtualLoadQueue | Oldest uncommitted load |
| `l2_hint` | Input(Valid(L2ToL1Hint)) | L2 Cache | L2 hints |
| `tlb_hint` | Flipped(TlbHintIO) | TLB | TLB hints |

#### Output Interfaces

| Port | Type | Destination | Description |
|------|------|-------------|-------------|
| `replay` | Vec(Decoupled(LsPipelineBundle)) | Load Units | Replay requests |
| `lqFull` | Output(Bool) | LoadQueue | Replay queue full |

### Functionality

**Purpose**: Buffers loads that need replay (TLB miss, cache miss, bank conflict, forwarding issues) and re-issues them when ready.

**Operation**:

1. **Enqueue (Load S3)**:
   ```scala
   for each load writeback i:
     when (enq(i).valid && enq(i).bits.rep_info.need_rep):
       // Allocate replay entry
       entry[allocPtr].valid := true
       entry[allocPtr].uop := enq(i).bits.uop
       entry[allocPtr].vaddr := enq(i).bits.vaddr
       entry[allocPtr].mask := enq(i).bits.mask
       entry[allocPtr].cause := enq(i).bits.rep_info.cause

       // Determine replay schedule based on cause
       if (cause == TLB_MISS):
         schedule := wait_for_tlb_hint
       else if (cause == CACHE_MISS):
         schedule := wait_for_cache_refill
       else if (cause == BANK_CONFLICT):
         schedule := immediate_replay
       else if (cause == FORWARD_FAIL):
         schedule := wait_for_store_data
   ```

2. **Wake-up on Events**:
   ```scala
   // TLB refill
   when (tlb_hint.valid):
     for each entry waiting on TLB:
       if (entry.vaddr matches tlb_hint.vaddr):
         entry.ready := true

   // Cache refill
   when (refill.valid):
     for each entry waiting on cache:
       if (entry.paddr matches refill.paddr):
         entry.ready := true

   // Store data ready
   when (storeDataIn(i).valid):
     for each entry waiting on store data:
       sqIdx = storeDataIn(i).bits.uop.sqIdx
       if (entry.waitSqIdx == sqIdx):
         entry.ready := true

   // Store address ready
   when (storeAddrIn(i).valid):
     for each entry waiting on store addr:
       sqIdx = storeAddrIn(i).bits.uop.sqIdx
       if (entry.waitSqIdx == sqIdx):
         entry.ready := true

   // RAR queue full resolved (C_RAR)
   // LoadQueueReplay.scala:331-333
   when (cause(i)(C_RAR)):
     // Unblock when RAR queue has space OR load is old enough to skip check
     blocking(i) := Mux(!io.rarFull || !isAfter(uop(i).lqIdx, io.ldWbPtr),
                        false.B, blocking(i))

   // RAW queue full resolved (C_RAW)
   // LoadQueueReplay.scala:335-337
   when (cause(i)(C_RAW)):
     // Unblock when RAW queue has space OR store is old enough to skip check
     blocking(i) := Mux(!io.rawFull || !isAfter(uop(i).sqIdx, io.stAddrReadySqPtr),
                        false.B, blocking(i))
   ```

3. **Schedule Replay**:
   ```scala
   // Select oldest ready entry
   for i in 0 until LoadPipelineWidth:
     ready_entries = entries.filter(_.valid && _.ready)
     if (ready_entries.nonEmpty):
       oldest = select_oldest(ready_entries)

       replay(i).valid := true
       replay(i).bits := oldest

       when (replay(i).fire):
         entry[oldest].valid := false
   ```

4. **Delayed Replay (for TLB)**:
   ```scala
   // TLB replays use configurable delay
   when (entry.cause == TLB_MISS && tlb_hint.valid):
     entry.delay_counter := tlbReplayDelayCycleCtrl
     entry.ready := false

   when (entry.delay_counter > 0):
     entry.delay_counter := entry.delay_counter - 1
   .elsewhen (entry.delay_counter == 0 && entry.woke):
     entry.ready := true
   ```

5. **Dependency Tracking with blockSqIdx**:

   For C_MA (memory ambiguity) and C_FF (forward fail) causes, the replay queue tracks **which specific store** the load is waiting for using `blockSqIdx`. This enables precise wakeup.

   **Step 1: Violation Detection & sqIdx Capture (Load Unit S2)**

   When a load queries StoreQueue for forwarding, it receives dependency info:
   ```scala
   // LoadUnit.scala:933-934 (S2 stage)
   io.lsq.forward (S1→S2 query)
     ├─► sqIdx, paddr, vaddr, mask
     │
     ├─◄ dataInvalid: Bool           ← Store data not ready
     ├─◄ addrInvalid: Bool           ← Store address not ready
     ├─◄ dataInvalidSqIdx: SqPtr     ← **Which store's data is missing**
     └─◄ addrInvalidSqIdx: SqPtr     ← **Which store's addr is missing**

   // Saved to rep_info for S3
   s2_out.rep_info.data_inv_sq_idx := io.lsq.forward.dataInvalidSqIdx
   s2_out.rep_info.addr_inv_sq_idx := io.lsq.forward.addrInvalidSqIdx
   ```

   **Step 2: Record Dependency in Replay Entry**

   ```scala
   // LoadQueueReplay.scala:234 (Field)
   val blockSqIdx = Reg(Vec(LoadQueueReplaySize, new SqPtr))

   // LoadQueueReplay.scala:615-623 (Enqueue)
   when (needEnqueue(w) && enq.ready) {
     val replayInfo = enq.bits.rep_info

     // C_MA: Memory Ambiguity - wait for store address
     when (replayInfo.cause(LoadReplayCauses.C_MA)) {
       blockSqIdx(enqIndex) := replayInfo.addr_inv_sq_idx
     }

     // C_FF: Forward Fail - wait for store data
     when (replayInfo.cause(LoadReplayCauses.C_FF)) {
       blockSqIdx(enqIndex) := replayInfo.data_inv_sq_idx
     }
   }
   ```

   **Step 3: Resolution Detection (Three Methods)**

   **Method A - Ready Vector Check (registered):**
   ```scala
   // StoreQueue provides per-entry ready vectors
   io.stAddrReadyVec(i) := RegNext(allocated(i) && addrvalid(i))
   io.stDataReadyVec(i) := RegNext(allocated(i) && datavalid(i))

   // LoadQueueReplay.scala:280-281 - Check specific blocking store
   dataNotBlockVec(i) := !isBefore(io.stDataReadySqPtr, blockSqIdx(i)) ||
                         stDataReadyVec(blockSqIdx(i).value) ||  // ← Check THIS store!
                         io.sqEmpty

   addrNotBlockVec(i) := !isBefore(io.stAddrReadySqPtr, blockSqIdx(i)) ||
                         stAddrReadyVec(blockSqIdx(i).value) ||  // ← Check THIS store!
                         io.sqEmpty
   ```

   **Method B - Same-Cycle Match (zero-cycle wakeup):**
   ```scala
   // LoadQueueReplay.scala:284-295 - Detect store executing this cycle
   storeAddrInSameCycleVec(i) := VecInit((0 until StorePipelineWidth).map(w => {
     io.storeAddrIn(w).valid &&
     !io.storeAddrIn(w).bits.miss &&
     blockSqIdx(i) === io.storeAddrIn(w).bits.uop.sqIdx  // ← Match!
   })).asUInt.orR

   storeDataInSameCycleVec(i) := VecInit((0 until StorePipelineWidth).map(w => {
     io.storeDataIn(w).valid &&
     blockSqIdx(i) === io.storeDataIn(w).bits.uop.sqIdx  // ← Match!
   })).asUInt.orR
   ```

   **Method C - Combined Readiness:**
   ```scala
   // LoadQueueReplay.scala:298-308
   stAddrDeqVec(i) := allocated(i) && (addrNotBlockVec(i) | storeAddrInSameCycleVec(i))
   stDataDeqVec(i) := allocated(i) && (dataNotBlockVec(i) | storeDataInSameCycleVec(i))
   ```

   **Step 4: Blocking Release**
   ```scala
   // LoadQueueReplay.scala:311-325
   for (i <- 0 until LoadQueueReplaySize) {
     // C_MA: Wait for store address
     when (cause(i)(LoadReplayCauses.C_MA)) {
       blocking(i) := Mux(stAddrDeqVec(i), false.B, blocking(i))
     }

     // C_FF: Wait for store data
     when (cause(i)(LoadReplayCauses.C_FF)) {
       blocking(i) := Mux(stDataDeqVec(i), false.B, blocking(i))
     }
   }
   ```

   **Complete Flow Example:**
   ```
   T0: Load S2 queries StoreQueue
       ├─► dataInvalid=true, dataInvalidSqIdx=SqPtr(15)
       └─► S3: rep_info.data_inv_sq_idx = SqPtr(15), cause=C_FF

   T1: Load enqueues to LoadQueueReplay entry 8
       ├─► blockSqIdx(8) := SqPtr(15)  ← Dependency recorded
       └─► blocking(8) := true

   T2-T5: Waiting (blocking=true)
       └─► stDataReadyVec(15) = false

   T6: Store #15 executes STD
       ├─► io.storeDataIn(1).valid = true, sqIdx = SqPtr(15)
       ├─► storeDataInSameCycleVec(8) = true  ← Match!
       └─► blocking(8) := false  ← Released!

   T7: Entry 8 becomes ready for selection

   T8: Load replays → Forward succeeds
   ```

   **Why Track Specific sqIdx?**
   - Without: Load waits for ALL older stores → massive latency
   - With blockSqIdx: Load wakes IMMEDIATELY when specific blocker completes
   - Latency saved: (N-1) × store_completion_cycles

**Key Characteristics**:
- **Selective Replay**: Only replays when blocking condition is resolved
- **Priority**: Oldest loads replay first (maintains memory ordering)
- **Backpressure**: Can stall load pipeline if full
- **Event-driven**: Wakes on cache refills, TLB updates, store data
- **Precise Dependency**: Tracks specific blocking sqIdx for C_MA/C_FF causes

---

## 7. LoadUnit (Load Execution Pipeline)

### Interface Description

**File**: [src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala:101)

### All Input Sources & Priority

LoadUnit accepts **7 input sources** with strict priority order (high to low):

| Priority | Source Name | Input Port | Type | Trigger Condition |
|----------|-------------|-----------|------|-------------------|
| **0 (Highest)** | Super Load Replay | `io.replay` | Decoupled(LsPipelineBundle) | `replay.valid && replay.bits.forward_tlDchannel` |
| **1** | Fast Load Replay | `io.fast_rep_in` | Decoupled(LqWriteBundle) | `fast_rep_in.valid` |
| **2** | Normal Load Replay | `io.replay` | Decoupled(LsPipelineBundle) | `replay.valid && !replay.bits.forward_tlDchannel && !rep_stall` |
| **3** | High-Conf HW Prefetch | `io.prefetch_req` | Valid(L1PrefetchReq) | `prefetch_req.valid && confidence > 0` |
| **4** | RS Issue / SW Prefetch | `io.ldin` | Decoupled(ExuInput) | `ldin.valid` |
| **5** | Vector Issue | (TODO) | - | Not implemented yet |
| **6** | Load-to-Load Forward | `io.l2l_fwd_in` | LoadToLoadIO | `l2l_fwd_in.valid` (pointer chasing) |
| **7 (Lowest)** | Low-Conf HW Prefetch | `io.prefetch_req` | Valid(L1PrefetchReq) | `prefetch_req.valid && confidence == 0` |

### Input Source Details

#### 1. Super Load Replay (Priority 0)

**Code Reference**: [LoadUnit.scala:216, 528](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L216)

```scala
val s0_super_ld_rep_valid = io.replay.valid && io.replay.bits.forward_tlDchannel
s0_out.forward_tlDchannel := s0_super_ld_rep_select
```

**When Triggered**:
- Cache miss replay **with D-channel refill data available**
- TileLink D channel is actively returning the requested cache line
- `forward_tlDchannel` flag is set in the replay request

**Purpose**: Minimize latency by forwarding refill data immediately from TileLink D channel (bypass cache write → read)

**Characteristics**:
- Highest priority - preempts all other sources
- Special fast path for refilling loads
- Reduces replay penalty significantly

---

#### 2. Fast Load Replay (Priority 1)

**Code Reference**: [LoadUnit.scala:217](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L217)

```scala
val s0_ld_fast_rep_valid = io.fast_rep_in.valid
```

**When Triggered**:
- **Memory ordering violation detected** (RAW, RAR)
- **Load-to-load dependency violation**
- Requires immediate re-execution

**Purpose**: Fast recovery from violations without going through normal replay queue

**Characteristics**:
- Higher priority than normal replay
- Bypasses LoadQueueReplay for critical replays
- Shorter latency path

---

#### 3. Normal Load Replay (Priority 2)

**Code Reference**: [LoadUnit.scala:218](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L218)

```scala
val s0_ld_rep_valid = io.replay.valid && !io.replay.bits.forward_tlDchannel && !s0_rep_stall
```

**When Triggered**:
- **L1 cache miss** (D-channel refill data **not yet available**)
- **TLB miss** (translation not in TLB)
- **Bank conflict** (cache bank busy)
- **Store-to-load forwarding failed** (see details below)

**Purpose**: Standard replay mechanism when D-channel forwarding is not possible

**Store-to-Load Forwarding Failure Cases**:

Store-to-load forwarding (`fwd_fail`) occurs when a load needs data from an older store, but the forwarding cannot complete. There are two main failure scenarios:

1. **`dataInvalid`** - Address matched but data not ready ([StoreQueue.scala:473-501](../../../src/main/scala/xiangshan/mem/lsqueue/StoreQueue.scala#L473))
   - Store address is computed and matches load address
   - Store data is **not yet written** to StoreQueue (store is still in execution)
   - Load must wait until the store writes its data
   - Tracked via `dataInvalidSqIdx` to wake up when store data becomes valid

2. **`addrInvalid`** (Memory Disambiguation) - SSID matched but address not ready ([StoreQueue.scala:488-554](../../../src/main/scala/xiangshan/mem/lsqueue/StoreQueue.scala#L488))
   - Store-Set predictor indicates potential dependency (same SSID)
   - Store address is **not yet computed**
   - Load must wait until the store computes its address to verify true dependency
   - Tracked via `addrInvalidSqIdx` to wake up when store address becomes valid

**Partial Forwarding** ([LoadUnit.scala:888-894](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L888)):

```scala
s2_full_fwd := ((~s2_fwd_mask.asUInt).asUInt & s2_in.mask) === 0.U && !io.lsq.forward.dataInvalid
```

- `forwardMask`: Byte-level mask indicating which bytes can be forwarded from SQ/SBuffer
- `full_fwd`: True only when **all** requested bytes are covered by forwarding
- If partial (e.g., load requests 4 bytes but only 2 are forwardable):
  - Remaining bytes must come from DCache
  - If DCache misses, load must replay and wait for both store data and cache refill

**Characteristics**:
- From LoadQueueReplay module
- Oldest ready load replays first
- Event-driven wake-up (store data valid, store address valid, refill, TLB update)

---

#### 4. High-Confidence HW Prefetch (Priority 3)

**Code Reference**: [LoadUnit.scala:219](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L219)

```scala
val s0_high_conf_prf_valid = io.prefetch_req.valid && io.prefetch_req.bits.confidence > 0.U
```

**When Triggered**:
- **SMS (Spatial Memory Streaming) prefetcher** detects strong pattern
- **Stride prefetcher** detects regular stride access
- Confidence level > 0 (medium to high confidence)

**Purpose**: Prefetch cache lines likely to be accessed soon

**Characteristics**:
- Higher priority than demand loads from RS
- Can use load pipeline when idle
- Improves cache hit rate

---

#### 5. RS Issue / SW Prefetch (Priority 4)

**Code Reference**: [LoadUnit.scala:220](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L220)

```scala
val s0_int_iss_valid = io.ldin.valid // int flow first issue or software prefetch
```

**When Triggered**:
- **Normal load instruction** issued from Reservation Station
- **Software prefetch** (PREFETCH instruction)
- All operands ready, RS selects this load

**Purpose**: Normal demand load execution path

**Characteristics**:
- Main load execution path
- Medium priority (replays have higher priority)
- Can be stalled by replays

---

#### 6. Load-to-Load Forwarding (Priority 6)

**Code Reference**: [LoadUnit.scala:222](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L222)

```scala
val s0_l2l_fwd_valid = io.l2l_fwd_in.valid
```

**When Triggered**:
- **Pointer chasing detected**: `ld r2, 0(r1)` followed by `ld r3, 0(r2)`
- Previous load calculated address, forwarding to next load
- `EnableLoadToLoadForward` configuration enabled

**Purpose**: Accelerate dependent load chains (common in linked data structures)

**Characteristics**:
- Forwards address calculation result
- Reduces latency for pointer-chasing loads
- Optional feature (can be disabled)

---

#### 7. Low-Confidence HW Prefetch (Priority 7)

**Code Reference**: [LoadUnit.scala:223](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L223)

```scala
val s0_low_conf_prf_valid = io.prefetch_req.valid && io.prefetch_req.bits.confidence === 0.U
```

**When Triggered**:
- **Speculative prefetch** with low confidence
- Prefetcher exploring potential patterns
- Only when pipeline completely idle

**Purpose**: Opportunistic prefetching without impacting critical path

**Characteristics**:
- Lowest priority - only runs when nothing else
- Can be easily evicted
- Avoids polluting cache with useless data

---

### Priority Arbitration Logic

**Code Reference**: [LoadUnit.scala:234-260, 484-502](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L234)

```scala
// Daisy-chain priority: Higher priority blocks lower
val s0_super_ld_rep_ready  = true.B  // Always ready (highest priority)
val s0_ld_fast_rep_ready   = !s0_super_ld_rep_valid
val s0_ld_rep_ready        = !s0_super_ld_rep_valid && !s0_ld_fast_rep_valid
val s0_high_conf_prf_ready = !s0_super_ld_rep_valid && !s0_ld_fast_rep_valid && !s0_ld_rep_valid
val s0_int_iss_ready       = !s0_super_ld_rep_valid && !s0_ld_fast_rep_valid &&
                             !s0_ld_rep_valid && !s0_high_conf_prf_valid
// ... continues for all priorities

// Selection
val s0_src_selector = Seq(
  s0_super_ld_rep_select,
  s0_ld_fast_rep_select,
  s0_ld_rep_select,
  s0_hw_prf_select,
  s0_int_iss_select,
  s0_vec_iss_select,
  s0_l2l_fwd_select
)

s0_sel_src := ParallelPriorityMux(s0_src_selector, s0_src_format)
```

**Key Point**: Only **one source** can be selected per cycle. Higher priority sources **block** lower priority sources.

---

### Other Input Interfaces

#### Memory System Inputs

| Input | Type | Source | Purpose |
|-------|------|--------|---------|
| `tlb` | TlbRequestIO(2) | DTLB | VA→PA translation (2 ports for load + prefetch) |
| `pmp` | PMPRespBundle | PMP Checker | Physical memory protection check |
| `dcache` | DCacheLoadIO | L1 DCache | Cache access for data read |
| `sbuffer` | LoadForwardQueryIO | Store Buffer | Forward from committed stores |
| `tl_d_channel` | DcacheToLduForwardIO | DCache TileLink | Refill data forwarding |
| `forward_mshr` | LduToMissqueueForwardIO | MSHR | Miss queue data forwarding |
| `refill` | Valid(Refill) | DCache | Cache line refill notification |

#### Control & Configuration

| Input | Type | Source | Purpose |
|-------|------|--------|---------|
| `redirect` | Valid(Redirect) | Backend | Branch misprediction/exception flush |
| `csrCtrl` | CustomCSRCtrlIO | CSR | Load barrier, memory ordering control |
| `isFirstIssue` | Bool | RS | Performance tracking |
| `rsIdx` | UInt | RS | RS entry index for feedback |

#### Violation & Feedback

| Input | Type | Source | Purpose |
|-------|------|--------|---------|
| `stld_nuke_query` | Vec(Valid(StoreNukeQueryIO)) | Store Pipelines | Store-load ordering violation check from store side |
| `lq_rep_full` | Bool | LoadQueueReplay | Replay queue full backpressure |
| `correctMissTrain` | Bool | Prefetcher | Prefetch accuracy correction |

---

### Functionality

**Purpose**: Execute load instructions through a 4-stage pipeline (S0→S1→S2→S3) with support for replays, forwarding, and prefetching.

**Key Operations**:

1. **Stage S0**: Source selection (priority arbitration), address generation
2. **Stage S1**: TLB lookup, cache tag access
3. **Stage S2**: Cache data access, forwarding check, violation detection
4. **Stage S3**: Data writeback, replay decision, LSQ update

**Pipeline Characteristics**:
- **4-stage pipeline**: S0, S1, S2, S3
- **Out-of-order execution**: Can overtake older loads (with violation detection)
- **Speculative execution**: Can execute before older stores (with rollback)
- **Multi-path**: Supports normal, replay, and prefetch flows

---

## 8. StoreUnit (Store Address Pipeline)

### Interface Description

**File**: [src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala:32)

### Input Sources

| Input | Type | Source | Trigger Condition |
|-------|------|--------|-------------------|
| `stin` | Decoupled(ExuInput) | Store Address RS | Normal store address issue from RS |
| `prefetch_req` | Decoupled(StorePrefetchReq) | Store Prefetcher | Store prefetch request |

**Note**: StoreUnit has **2 input sources** with simple priority: RS issue has priority over prefetch.

```scala
val s0_iss_valid = io.stin.valid
val s0_prf_valid = io.prefetch_req.valid && io.dcache.req.ready
val s0_valid = s0_iss_valid || s0_prf_valid
val s0_use_flow_rs = s0_iss_valid
```

### Input Interfaces

| Input | Type | Source | Purpose |
|-------|------|--------|---------|
| **Issue & Control** |
| `stin` | Decoupled(ExuInput) | Store Address RS | Store address calculation issue |
| `redirect` | Valid(Redirect) | Backend | Flush on misprediction |
| `isFirstIssue` | Bool | RS | First issue tracking |
| `rsIdx` | UInt | RS | RS entry index for feedback |
| **Memory System** |
| `tlb` | TlbRequestIO | DTLB | Virtual address translation |
| `pmp` | PMPRespBundle | PMP | Physical memory protection |
| `dcache` | DCacheStoreIO | DCache | Cache access for address check |
| **Prefetch** |
| `prefetch_req` | Decoupled(StorePrefetchReq) | Store Prefetcher | Store prefetch request |

### Functionality

**Purpose**: Calculate store address, translate via TLB, and write to StoreQueue.

**Key Operations**:
1. **S0**: Select source (RS issue or prefetch), calculate address
2. **S1**: TLB translation, cache tag check
3. **S2**: Write address to StoreQueue, PMP check
4. **S3**: Feedback to RS, violation query

**Note**: Store **data** is handled separately by StdExeUnit (Store Data Execution Unit).

---

## 9. StdExeUnit (Store Data Pipeline)

### Interface Description

**File**: [src/main/scala/xiangshan/backend/exu/Exu.scala](../../../src/main/scala/xiangshan/backend/exu/Exu.scala:116)

StdExeUnit extends the base `Exu` class with `StdExeUnitCfg` configuration.

### Input Interfaces

| Input | Type | Source | Purpose |
|-------|------|--------|---------|
| `fromInt` | Decoupled(ExuInput) | Store Data RS | Store data issue from RS |
| `redirect` | Valid(Redirect) | Backend | Flush on misprediction |

### Functionality

**Purpose**: Read store data from register file and write to StoreQueue.

**Key Operations**:
1. Issue store data from RS
2. Read source register (data to store)
3. Write data to StoreQueue via `dataModule`

**Note**: StdExeUnit is a **simple execution unit** that only handles data movement. Address is handled separately by StoreUnit.

---

## Summary Table

| Module | Primary Function | Entry Count | Key State |
|--------|-----------------|-------------|-----------|
| **VirtualLoadQueue** | Track load control state | VirtualLoadQueueSize | allocated, addrvalid, datavalid, uop |
| **StoreQueue** | Buffer stores, provide forwarding | StoreQueueSize | allocated, addrvalid, datavalid, committed, paddr, vaddr, data |
| **LoadQueueRAW** | Store-load violation detection | Small tracker | Monitored address ranges |
| **LoadQueueRAR** | Load-load violation detection | Small tracker | Cache line tracking |
| **LoadQueueReplay** | Replay scheduling | Smaller queue | Waiting loads, wake-up conditions |

---

## 10. Pipeline Stage-by-Stage Operations

This section provides a detailed breakdown of what happens at each pipeline stage for both Load and Store execution pipelines.

### 10.1 Load Pipeline Stage Operations

The Load pipeline consists of 4 stages (S0 → S1 → S2 → S3), each performing specific operations.

---

#### **Stage 0 (S0): Input Arbitration and Request Generation**

**File Reference**: [LoadUnit.scala:484-562](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L484-L562)

**Duration**: 1 cycle

**Main Operations**:

1. **Input Source Selection (Priority-based Arbitration)**
   - Selects one input from 7 sources using strict priority (highest to lowest):
     - Priority 0: Super Replay (io.replay with forward_tlDchannel=true)
     - Priority 1: Fast Replay (io.fast_rep_in)
     - Priority 2: Normal Replay (io.replay with forward_tlDchannel=false)
     - Priority 3: High-Confidence HW Prefetch
     - Priority 4: RS Issue / SW Prefetch (io.ldin)
     - Priority 5: Vector Issue (not implemented)
     - Priority 6: Load-to-Load Forwarding (io.l2l_fwd_in)
     - Priority 7: Low-Confidence HW Prefetch
   - Uses `ParallelPriorityMux` for selection ([LoadUnit.scala:502](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L502))

2. **Address Alignment Check**
   - Validates address alignment based on operation size:
     - Byte (b00): Always aligned
     - Halfword (b01): vaddr[0] == 0
     - Word (b10): vaddr[1:0] == 0
     - Doubleword (b11): vaddr[2:0] == 0
   - Sets `loadAddrMisaligned` exception if misaligned ([LoadUnit.scala:505-527](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L505-L527))

3. **TLB Request**
   - Sends virtual address to TLB for translation
   - Provides request metadata (cmd, size, robIdx, pc, etc.)
   - TLB operates in parallel with DCache tag access

4. **DCache Request**
   - Sends request to DCache for tag/data array access
   - Only proceeds if `s0_can_go && io.dcache.req.ready`
   - Sets `replacementUpdated` flag for replay loads

5. **Ready Signal Generation**
   - `io.replay.ready`: Accepts replay if DCache ready and highest priority
   - `io.fast_rep_in.ready`: Accepts fast replay if DCache ready
   - `io.ldin.ready`: Accepts RS issue only if no higher priority requests
   - All gated by `s0_can_go` (S1 must be ready)

**Key Signals Generated**:
- `s0_out.vaddr`: Virtual address
- `s0_out.mask`: Byte mask for operation
- `s0_out.uop`: Micro-operation metadata
- `s0_out.forward_tlDchannel`: Super replay flag
- `s0_out.isFirstIssue`: First issue indicator

**Exit Condition**: `s0_fire` = `s0_valid && s0_can_go && io.dcache.req.ready`

---

#### **Stage 1 (S1): TLB Response and Forward Query**

**File Reference**: [LoadUnit.scala:564-739](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L564-L739)

**Duration**: 1 cycle

**Main Operations**:

1. **TLB Response Processing**
   - Receives physical address from TLB (`io.tlb.resp.bits.paddr`)
   - Detects TLB miss (`s1_tlb_miss`)
   - Checks for page faults and access faults:
     - `loadPageFault`: TLB page fault
     - `loadAccessFault`: TLB access fault or delayed error
   - Duplicated paddr for LSU and DCache paths ([LoadUnit.scala:601-602](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L601-L602))

2. **Store-to-Load Forwarding Query**
   - **StoreBuffer Query** ([LoadUnit.scala:617-623](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L617-L623)):
     - Sends vaddr, paddr, mask, sqIdx to StoreBuffer
     - StoreBuffer checks for matching stores and returns forwarded data
   - **LSQ (StoreQueue) Query** ([LoadUnit.scala:625-632](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L625-L632)):
     - Sends same query to StoreQueue for younger stores
     - StoreQueue has higher priority than StoreBuffer in S2

3. **Store-Load Violation Check**
   - Checks if any store is writing to overlapping address in same cycle
   - Compares with `io.stld_nuke_query` from Store pipelines ([LoadUnit.scala:635-641](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L635-L641)):
     - Must be older store (isAfter check)
     - Physical address match (paddr[PAddrBits-1:3])
     - Byte mask overlap
   - Sets `s1_nuke` if violation detected

4. **DCache S1 Operations**
   - Provides paddr to DCache for data array access
   - `io.dcache.s1_kill`: Kills request if TLB miss/exception/MMIO

5. **Pointer Chasing (Load-to-Load Forwarding)**
   - If enabled and attempted in S0:
     - Validates address calculation correctness
     - Checks for cancellation conditions:
       - Cache set mismatch (vaddr[6] overflow)
       - Address misalignment
       - Wrong operation type (not LD)
       - Fast match failure
     - Updates UOP if valid ([LoadUnit.scala:695-718](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L695-L718))

6. **MSHR Forward Request**
   - For super replay: requests data forwarding from MSHR
   - `io.forward_mshr.valid`, `mshrid`, `paddr` ([LoadUnit.scala:733-735](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L733-L735))

**Key Signals Generated**:
- `s1_out.paddr`: Physical address
- `s1_out.tlbMiss`: TLB miss indicator
- `s1_out.rep_info.nuke`: Violation detected
- `s1_out.delayedLoadError`: Fast replay delayed error
- Updated exception vector

**Kill Conditions** ([LoadUnit.scala:675-679](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L675-L679)):
- Fast replay late kill
- Pointer chasing cancellation
- ROB flush (same cycle or previous cycle)
- Previous stage kill propagation

**Exit Condition**: `s1_fire` = `s1_valid && !s1_kill && s2_ready`

---

#### **Stage 2 (S2): DCache Response and Replay Decision**

**File Reference**: [LoadUnit.scala:741-998](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L741-L998)

**Duration**: 1 cycle

**Main Operations**:

1. **DCache Response Processing**
   - Receives cache hit/miss status (`io.dcache.resp.bits.miss`)
   - Receives cache data (`io.dcache.resp.bits.data`)
   - Checks for:
     - `s2_dcache_miss`: Cache miss (unless forwarded)
     - `s2_mq_nack`: MSHR queue full
     - `s2_bank_conflict`: Bank conflict
     - `s2_wpu_pred_fail`: Way predictor failure
     - `s2_cache_tag_error`: ECC tag error

2. **PMP (Physical Memory Protection) Check**
   - Receives PMP response (`io.pmp`)
   - Checks for access violations (`s2_pmp.ld`)
   - Identifies MMIO regions (`s2_pmp.mmio`)
   - Updates `loadAccessFault` exception

3. **Data Forwarding Merge**
   - **TileLink D Channel Forward** ([LoadUnit.scala:781-783](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L781-L783)):
     - For super replay: gets data directly from D channel
     - `s2_fwd_frm_d_chan`, `s2_fwd_data_frm_d_chan`
   - **MSHR Forward** ([LoadUnit.scala:782](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L782)):
     - Gets data from MSHR if available
     - `s2_fwd_frm_mshr`, `s2_fwd_data_frm_mshr`
   - **StoreQueue + StoreBuffer Forward** ([LoadUnit.scala:887-895](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L887-L895)):
     - Merges forwarded data from both sources
     - LSQ has priority over StoreBuffer
     - Creates per-byte forward mask and data
   - **Full Forward Check** ([LoadUnit.scala:890](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L890)):
     - `s2_full_fwd`: All bytes covered by forwarding
     - `((~s2_fwd_mask.asUInt) & s2_in.mask) === 0.U`

4. **Memory Ordering Violation Queries**
   - **Load-Load Violation Query** ([LoadUnit.scala:873-877](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L873-L877)):
     - `io.lsq.ldld_nuke_query.req`: Sends request to LoadQueueRAR
     - Includes uop, mask, paddr, data_valid
   - **Store-Load Violation Query** ([LoadUnit.scala:880-884](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L880-L884)):
     - `io.lsq.stld_nuke_query.req`: Sends request to LoadQueueRAW
     - Checks for RAW hazards with younger stores
   - Detects new violations from Store pipelines ([LoadUnit.scala:827-833](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L827-L833))

5. **Replay Condition Analysis**
   - **Fast Replay Required** ([LoadUnit.scala:848-862](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L848-L862)):
     - `s2_dcache_fast_rep`: MQ nack, bank conflict, or WPU fail
     - `s2_nuke_fast_rep`: Store-load violation
     - Goes to Fast Replay queue for quick retry
   - **Normal Replay Required** ([LoadUnit.scala:916-941](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L916-L941)):
     - Memory ambiguity (`s2_mem_amb`): StoreSet hit + addr invalid
     - TLB miss (`s2_tlb_miss`)
     - Forward fail (`s2_fwd_fail`): Data invalid from stores
     - DCache miss (`s2_dcache_miss`)
     - RAR/RAW nack: Queue full
     - Nuke: Violation detected
   - Stores detailed cause in `s2_out.rep_info.cause`

6. **Fast Wakeup Signal**
   - Generates speculative wakeup for dependent instructions ([LoadUnit.scala:957-964](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L957-L964))
   - Conditions:
     - S1 valid and not killed
     - No TLB miss
     - No forward data invalid
     - S2 confirms no replay needed and not MMIO

7. **Prefetch Training**
   - Sends load access pattern to prefetcher ([LoadUnit.scala:971-981](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L971-L981))
   - Includes miss status, paddr, access metadata

**Key Signals Generated**:
- `s2_out.data`: Set to 0 (actual data generated in S3)
- `s2_out.forwardMask/forwardData`: Merged forward results
- `s2_out.mmio`: MMIO flag
- `s2_out.miss`: Cache miss flag
- `s2_out.rep_info`: Complete replay info with all causes
- `s2_out.handledByMSHR`: MSHR handling flag

**Kill Conditions** ([LoadUnit.scala:753](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L753)):
- ROB flush

**Exit Condition**: `s2_fire` = `s2_valid && !s2_kill && s3_ready`

---

#### **Stage 3 (S3): Data Generation and Writeback**

**File Reference**: [LoadUnit.scala:1000-1180](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L1000-L1180)

**Duration**: 1 cycle

**Main Operations**:

1. **Data Source Selection and Merging**
   - **From DCache** ([LoadUnit.scala:1124-1155](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L1124-L1155)):
     - Uses delayed cache data: `io.dcache.resp.bits.data_delayed`
     - Merges with forwarded data from StoreQueue/StoreBuffer
     - Applies TileLink D channel data if super replay
     - Applies MSHR forwarded data if available
     - Creates merged data per byte
   - **From Uncache** ([LoadUnit.scala:1108-1121](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L1108-L1121)):
     - Uses `io.lsq.ld_raw_data` for MMIO/uncached loads
     - Different data path for uncached accesses

2. **Data Alignment and Sign Extension**
   - Picks correct bytes based on address offset (paddr[3:0])
   - Performs data alignment using `LookupTree` ([LoadUnit.scala:1137-1154](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L1137-L1154))
   - Applies sign/zero extension using `rdataHelper`
   - Handles different load sizes (LB, LH, LW, LD, LBU, LHU, LWU)

3. **TileLink D Channel Late Forwarding**
   - For super replay: attempts to get data in S3 if not available in S2
   - `s3_fwd_frm_d_chan_valid`: D channel data valid in S3 ([LoadUnit.scala:1016-1018](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L1016-L1018))
   - Updates miss status: not a miss if D channel provides data

4. **Exception and Error Handling**
   - **Delayed Load Error** ([LoadUnit.scala:1030-1035](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L1030-L1035)):
     - `s3_dly_ld_err`: Cache error delayed (ECC error)
     - Sets `loadAccessFault` exception
   - **Virtual-Physical Match Failure** ([LoadUnit.scala:1040](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L1040)):
     - `s3_vp_match_fail`: Forward match invalid
     - Requires replay from fetch
   - **Load-Load Violation** ([LoadUnit.scala:1042-1045](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L1042-L1045)):
     - `s3_ldld_rep_inst`: RAR violation requires flush
     - Sets `flushPipe` flag

5. **Replay Information Finalization**
   - Updates replay cause based on S3 conditions ([LoadUnit.scala:1048-1057](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L1048-L1057))
   - Clears replay causes if exception/error/replay-from-fetch
   - Otherwise keeps S2 replay causes with updates:
     - `dcache_miss` updated with D channel forward status

6. **Writeback to VirtualLoadQueue**
   - **Conditions** ([LoadUnit.scala:1021-1023](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L1021-L1023)):
     - Valid and not fast replay (or fast replay canceled)
     - Not already feedbacked
   - **Information Written** ([LoadUnit.scala:1021-1086](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L1021-L1086)):
     - Complete UOP with updated exception vector
     - Miss status (considering D channel forward)
     - Replay information with all causes
     - Data write enable flags (`data_wen_dup`)
     - Replacement and miss tracking metadata

7. **Writeback to Register File**
   - **Conditions** ([LoadUnit.scala:1060](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L1060)):
     - Valid and no replay needed
     - Not MMIO (MMIO goes through uncache path)
   - **Output** ([LoadUnit.scala:1159-1161](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L1159-L1161)):
     - `io.ldout.valid`: Writeback valid
     - `io.ldout.bits.data`: Final load data (aligned, sign-extended)
     - `io.ldout.bits.uop`: Complete micro-op
     - Debug info: paddr, vaddr, MMIO flag

8. **Fast Replay Queue Entry**
   - **Conditions** ([LoadUnit.scala:1164](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L1164)):
     - Valid and fast replay required
   - **Output** ([LoadUnit.scala:1164-1166](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L1164-L1166)):
     - `io.fast_rep_out.valid`
     - Complete pipeline bundle for fast retry
     - `lateKill` flag if replay-from-fetch

9. **Load-to-Load Forwarding**
   - If enabled ([LoadUnit.scala:1171-1175](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L1171-1175)):
     - `io.l2l_fwd_out.valid`: Forward successful
     - `io.l2l_fwd_out.data`: 64-bit data for pointer chasing
     - `io.l2l_fwd_out.dly_ld_err`: Error flags

10. **Rollback Signal**
    - Generates pipeline flush if needed ([LoadUnit.scala:1074-1083](../../../src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L1074-L1083)):
      - `s3_rep_frm_fetch`: Virtual-physical match fail
      - `s3_flushPipe`: Load-load violation
    - Provides redirect information to frontend

**Key Signals Generated**:
- `io.ldout`: Complete load result for register file writeback
- `io.lsq.ldin`: Complete load info for VirtualLoadQueue
- `io.fast_rep_out`: Fast replay queue entry
- `io.l2l_fwd_out`: Load-to-load forward data
- `io.rollback`: Pipeline flush request

**Data Flow**:
```
Cache Data → Merge with Forwards → Align → Sign Extend → Register File
           → Write to VLQ
           → Fast Replay Queue (if needed)
           → L2L Forward (if enabled)
```

**Exit Condition**: `s3_valid && (!s3_kill || io.ldout.ready)`

---

### 10.2 Store Pipeline Stage Operations

The Store pipeline consists of 4 stages (S0 → S1 → S2 → S3/SX), with SX being optional delay stages for load-store violation detection.

---

#### **Stage 0 (S0): Address Generation and TLB Request**

**File Reference**: [StoreUnit.scala:80-147](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L80-L147)

**Duration**: 1 cycle

**Main Operations**:

1. **Input Source Selection**
   - **RS Issue** (Priority 1): `io.stin.valid`
     - Store address calculation from reservation station
     - Has ROB entry
   - **Store Prefetch** (Priority 2): `io.prefetch_req.valid`
     - Hardware prefetch for store addresses
     - No ROB entry
   - Selection logic ([StoreUnit.scala:61-78](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L61-L78)):
     - `s0_use_flow_rs`: Use RS flow
     - `s0_use_flow_prf`: Use prefetch flow

2. **Address Calculation**
   - Computes effective address ([StoreUnit.scala:82-88](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L82-L88)):
     - `s0_saddr = base (src[0]) + SignExt(imm12)`
     - Split into high and low parts for carry handling
     - `saddr_hi` handles carry from `saddr_lo[12]`
   - Virtual address: `s0_vaddr = Cat(saddr_hi, saddr_lo[11:0])`

3. **Byte Mask Generation**
   - Creates byte mask based on operation size ([StoreUnit.scala:90](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L90)):
     - Uses `genVWmask(vaddr, fuOpType[1:0])`
     - Indicates which bytes are written

4. **TLB Request**
   - Sends virtual address for translation ([StoreUnit.scala:92-104](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L92-L104)):
     - Command: TlbCmd.write
     - Provides size, mem_idx (sqIdx), robIdx, pc
     - `memidx.is_st = true`

5. **DCache Tag Check**
   - Reads DCache tags to predict hit/miss ([StoreUnit.scala:112-115](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L112-L115)):
     - Command: `M_PFW` (prefetch for write)
     - Does NOT wait for DCache ready (unlike loads)
     - Prefetch always waits for DCache ready

6. **Address Alignment Check**
   - Validates alignment based on size ([StoreUnit.scala:134-140](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L134-L140)):
     - Byte: always aligned
     - Halfword: vaddr[0] == 0
     - Word: vaddr[1:0] == 0
     - Doubleword: vaddr[2:0] == 0
   - Sets `storeAddrMisaligned` exception

7. **Store Mask Output**
   - Sends mask to StoreQueue for early allocation ([StoreUnit.scala:142-144](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L142-L144)):
     - `io.st_mask_out.valid`
     - `io.st_mask_out.bits.mask`
     - `io.st_mask_out.bits.sqIdx`

**Key Signals Generated**:
- `s0_out.vaddr`: Calculated virtual address
- `s0_out.mask`: Byte write mask
- `s0_out.uop`: Micro-operation
- `s0_out.isFirstIssue`: First issue flag
- `s0_out.isHWPrefetch`: Prefetch indicator

**Ready Signals** ([StoreUnit.scala:146-147](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L146-L147)):
- `io.stin.ready = s1_ready`
- `io.prefetch_req.ready = s1_ready && dcache.ready && !s0_iss_valid`

**Exit Condition**: `s0_fire` = `s0_valid && s1_ready`

---

#### **Stage 1 (S1): TLB Response and StoreQueue Write**

**File Reference**: [StoreUnit.scala:149-229](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L149-L229)

**Duration**: 1 cycle

**Main Operations**:

1. **TLB Response Processing**
   - Receives physical address ([StoreUnit.scala:165](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L165)):
     - `s1_paddr = io.tlb.resp.bits.paddr(0)`
   - Detects TLB miss ([StoreUnit.scala:166](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L166)):
     - `s1_tlb_miss = io.tlb.resp.bits.miss`
   - Checks for exceptions ([StoreUnit.scala:213-214](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L213-L214)):
     - `storePageFault`: Page fault exception
     - `storeAccessFault`: Access fault exception

2. **MMIO/CBO Detection**
   - Identifies cache management operations ([StoreUnit.scala:162-164](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L162-L164)):
     - `s1_mmio_cbo`: CBO clean/flush/inval operations
     - These are treated as MMIO

3. **Store-Load Violation Query**
   - Sends query to load pipelines ([StoreUnit.scala:178-181](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L178-L181)):
     - `io.stld_nuke_query.valid`: Query valid
     - `io.stld_nuke_query.bits.robIdx`: Store's ROB index
     - `io.stld_nuke_query.bits.paddr`: Physical address
     - `io.stld_nuke_query.bits.mask`: Byte mask
   - Load pipelines check if younger loads accessed same address

4. **Issue to Reservation Station**
   - Notifies RS that store address is ready ([StoreUnit.scala:184-185](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L184-L185)):
     - `io.issue.valid`: Address calculation complete
     - `io.issue.bits`: Original instruction info
   - RS can now issue dependent instructions

5. **TLB Feedback to RS**
   - Generates feedback for replay on TLB miss ([StoreUnit.scala:190-203](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L190-L203)):
     - `s1_feedback.valid`
     - `s1_feedback.bits.hit = !s1_tlb_miss`
     - `s1_feedback.bits.flushState`: PTW walk required
     - `sourceType = RSFeedbackType.tlbMiss`
   - Feedback sent in S2 ([StoreUnit.scala:203](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L203))

6. **Write to StoreQueue**
   - Writes address and metadata to StoreQueue ([StoreUnit.scala:216-218](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L216-L218)):
     - `io.lsq.valid = s1_valid && !isHWPrefetch`
     - `io.lsq.bits.paddr`: Physical address
     - `io.lsq.bits.vaddr`: Virtual address
     - `io.lsq.bits.mask`: Byte mask
     - `io.lsq.bits.uop`: Micro-operation with exceptions
     - `io.lsq.bits.miss = s1_tlb_miss`

7. **DCache Kill Signal**
   - Kills DCache write intent if needed ([StoreUnit.scala:221](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L221)):
     - `io.dcache.s1_kill`: TLB miss/exception/MMIO/flush
     - `io.dcache.s1_paddr`: Physical address for tag check

**Key Signals Generated**:
- `s1_out.paddr`: Physical address
- `s1_out.tlbMiss`: TLB miss flag
- `s1_out.mmio`: MMIO flag
- `s1_out.uop.cf.exceptionVec`: Updated exceptions

**Kill Conditions** ([StoreUnit.scala:169](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L169)):
- ROB flush
- TLB miss (for pipeline progression, but still writes to SQ)

**Exit Condition**: `s1_fire` = `s1_valid && !s1_kill && s2_ready`

---

#### **Stage 2 (S2): PMP Check and MMIO Detection**

**File Reference**: [StoreUnit.scala:231-287](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L231-L287)

**Duration**: 1 cycle

**Main Operations**:

1. **PMP (Physical Memory Protection) Check**
   - Receives PMP response ([StoreUnit.scala:248](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L248)):
     - `s2_pmp = io.pmp`
   - Checks permissions and identifies regions:
     - `s2_pmp.mmio`: MMIO region
     - `s2_pmp.atomic`: Atomic region
     - `s2_pmp.st`: Access violation

2. **MMIO Finalization**
   - Combines TLB and PMP results ([StoreUnit.scala:251](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L251)):
     - `s2_mmio = s2_in.mmio || s2_pmp.mmio`
   - MMIO stores don't proceed through normal writeback

3. **Exception Update**
   - Updates access fault exception ([StoreUnit.scala:257](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L257)):
     - `storeAccessFault |= s2_pmp.st`
   - Combines TLB and PMP exceptions

4. **DCache Response**
   - Receives DCache tag check result ([StoreUnit.scala:263](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L263)):
     - `io.dcache.resp.ready = true.B`
     - Response indicates predicted hit/miss

5. **DCache Kill Signal**
   - Kills DCache write intent for special cases ([StoreUnit.scala:260](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L260)):
     - `io.dcache.s2_kill`: MMIO/exception/flush
     - MMIO stores go through uncache path

6. **TLB Feedback Delayed**
   - Sends feedback generated in S1 to RS ([StoreUnit.scala:266-267](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L266-L267)):
     - `io.feedback_slow.valid = RegNext(s1_feedback.valid && !flush)`
     - `io.feedback_slow.bits = RegNext(s1_feedback.bits)`

7. **Replenish to StoreQueue**
   - Updates StoreQueue with S2 information ([StoreUnit.scala:270](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L270)):
     - `io.lsq_replenish = s2_out`
     - Includes updated MMIO/atomic flags
     - Includes DCache miss prediction ([StoreUnit.scala:273](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L273))

8. **Prefetch Training**
   - If enabled, sends store pattern to prefetcher ([StoreUnit.scala:277-287](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L277-L287)):
     - `io.prefetch_train.valid`
     - `io.prefetch_train.bits`: Access pattern info
     - Only for non-MMIO, non-TLB-miss, non-prefetch stores

**Key Signals Generated**:
- `s2_out.mmio`: Final MMIO flag
- `s2_out.atomic`: Atomic flag
- Updated exception vector

**Kill Conditions** ([StoreUnit.scala:252](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L252)):
- (MMIO && !exception): MMIO without exception kills pipeline
- ROB flush

**Exit Condition**: `s2_fire` = `s2_valid && !s2_kill && s3_ready`

---

#### **Stage 3/SX (S3 + Delay Stages): Writeback to ROB**

**File Reference**: [StoreUnit.scala:289-356](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L289-L356)

**Duration**: Variable (1 + TotalDelayCycles cycles)

**Main Operations**:

1. **S3 Stage Filtering**
   - Only non-MMIO stores with exceptions or normal stores proceed ([StoreUnit.scala:301](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L301)):
     - `s3_valid := (!s2_mmio || s2_exception) && !s2_out.isHWPrefetch`
   - MMIO stores handled separately through uncache path

2. **ExuOutput Preparation**
   - Prepares writeback to ROB ([StoreUnit.scala:310-319](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L310-L319)):
     - `s3_out.uop`: Complete micro-op
     - `s3_out.data`: Don't care (stores don't write registers)
     - `s3_out.redirectValid = false.B`
     - `s3_out.debug.isMMIO = s3_in.mmio`
     - `s3_out.debug.paddr/vaddr`: Address info

3. **Delay Stages (SX)**
   - **Purpose**: Wait for load-store violation detection
     - `TotalSelectCycles = ceil(log2(LoadQueueRAWSize) / log2(RollbackGroupSize)) + 1`
     - `TotalDelayCycles = TotalSelectCycles - 2`
     - Gives time for violation CAM search ([StoreUnit.scala:306-308](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L306-L308))

   - **Pipeline Structure** ([StoreUnit.scala:333-353](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L333-L353)):
     - Creates chain of delay registers: `sx_valid[i]`, `sx_in[i]`
     - Each stage checks for ROB flush
     - Ready signal propagates backward
     - `sx_valid(i) = RegEnable(Mux(prev_fire, true.B, false.B), false.B, valid_can_go)`

4. **Final Writeback**
   - After all delay stages, writes back to ROB ([StoreUnit.scala:355-357](../../../src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala#L355-L357)):
     - `io.stout.valid = sx_last_valid && !flush`
     - `io.stout.bits = sx_last_in`
     - Signals store address calculation complete to ROB
     - Store won't commit until data also ready (from StdExeUnit)

**Key Points**:
- Stores write address (S1) and data (separate pipeline) to StoreQueue
- Both must be ready before commit
- Actual memory write happens at commit time (through StoreQueue → SBuffer → DCache)
- Delay stages ensure violation detection completes before commit

**Data Flow**:
```
S0: Addr Calc → S1: TLB + SQ Write → S2: PMP + Replenish → S3/SX: Delay + ROB WB
                                    ↓
                              StoreQueue (holds until commit)
                                    ↓
                              SBuffer (on commit)
                                    ↓
                              DCache Write
```

**Exit Condition**: `io.stout.fire` = `sx_last_valid && !flush && io.stout.ready`

---

### 10.3 Store Data Pipeline (StdExeUnit)

**File Reference**: [src/main/scala/xiangshan/backend/exu/StdExeUnit.scala](../../../src/main/scala/xiangshan/backend/exu/StdExeUnit.scala)

**Duration**: ~1 cycle (simple execution unit)

**Operations**:

1. **Data Read from Register File**
   - Reads store data from integer or FP register file
   - Input: `io.fromInt` or `io.fromFp` (from [Exu.scala:117-118](../../../src/main/scala/xiangshan/backend/exu/Exu.scala#L117-L118))

2. **Data Write to StoreQueue**
   - Writes data to StoreQueue entry
   - Uses sqIdx from UOP to find correct entry
   - Must coordinate with StoreUnit (which writes address)

3. **ExuOutput Generation**
   - Signals completion to ROB
   - No actual register writeback (stores don't write registers)

**Note**: StdExeUnit is simpler than StoreUnit. It only handles data movement from register file to StoreQueue, while StoreUnit handles complex address calculation, TLB, and violation checks.

---

### 10.4 Summary: Pipeline Timing Diagram

```
Cycle:     0        1        2        3        4        5
         ┌────┐   ┌────┐   ┌────┐   ┌────┐
Load:    │ S0 │──▶│ S1 │──▶│ S2 │──▶│ S3 │──▶ RF WB
         └────┘   └────┘   └────┘   └────┘
           │        │        │        │
           ├─ Arb   ├─ TLB   ├─DCache ├─ Data Gen
           ├─ TLB   ├─ Fwd   ├─ Fwd   ├─ VLQ Write
           └─DCache └─ Viol  ├─ Viol  ├─ Replay
                             └─Replay └─ L2L Fwd

         ┌────┐   ┌────┐   ┌────┐   ┌────┐   ┌────┐
Store:   │ S0 │──▶│ S1 │──▶│ S2 │──▶│ S3 │──▶│ SX │──▶ ROB WB
(Addr)   └────┘   └────┘   └────┘   └────┘   └────┘
           │        │        │        │        │
           ├─ Addr  ├─ TLB   ├─ PMP   ├─Filter └─Delay
           └─ TLB   ├─ SQ    └─ SQ    └─ExuOut  (Viol
                    └─ Viol    Update             Check)

Store:   ┌────┐   ┌────┐
(Data)   │Read│──▶│ SQ │──▶ (Wait for Commit)
         └────┘   └────┘
```

**Key Observations**:
1. Load pipeline is 4 stages with complex forwarding and replay logic
2. Store pipeline is 4+ stages with delay for violation detection
3. Store data and address are separate pipelines that merge in StoreQueue
4. Both use speculative execution with extensive replay mechanisms
5. Memory ordering violations detected in S1-S2 of both pipelines

---

*Document Status: Updated with comprehensive pipeline stage breakdown*
*Last Updated: 2026-02-03*
