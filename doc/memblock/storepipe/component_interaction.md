# StoreUnit Component Interaction Analysis

This document details how the StoreUnit interacts with other components in the XiangShan memory subsystem.

---

## System Context Diagram

```mermaid
graph TB
    subgraph "Backend"
        ROB[ROB<br>Reorder Buffer]
        RS[Reservation<br>Station]
        DISPATCH[Dispatch]
    end

    subgraph "StoreUnit Pipeline"
        STUNIT[StoreUnit<br>S0-S3-SX]
    end

    subgraph "Memory Subsystem"
        LSQ[Load-Store<br>Queue]
        SQ[Store<br>Queue]
        LQ[Load<br>Queue]
        SBUF[Store<br>Buffer]
    end

    subgraph "Cache & TLB"
        DTLB[DTLB<br>Data TLB]
        L2TLB[L2 TLB<br>Shared]
        PTW[PTW<br>Page Walk]
        DC[DCache<br>L1 Data]
    end

    subgraph "Protection"
        PMP[PMP<br>Physical Memory<br>Protection]
    end

    subgraph "Prefetch"
        SMS[SMS<br>Prefetcher]
        PRF_UNIT[Prefetch<br>Unit]
    end

    DISPATCH -->|uop| RS
    RS -->|stin| STUNIT
    STUNIT -->|stout| ROB
    STUNIT -->|lsq| LSQ
    STUNIT -->|st_mask| SQ
    STUNIT -->|feedback| RS

    STUNIT <-->|req/resp| DTLB
    DTLB <-->|miss/refill| L2TLB
    L2TLB <-->|walk| PTW

    STUNIT <-->|req/resp| DC
    STUNIT -->|stld_query| LQ

    PMP -->|resp| STUNIT

    STUNIT -->|train| SMS
    PRF_UNIT -->|prefetch_req| STUNIT
    SMS -->|hints| PRF_UNIT

    LSQ -->|commit| SBUF
    SBUF -->|drain| DC

    %% styling
    classDef cpu fill:#e3f2fd,stroke:#1565c0,stroke-width:2px;
    classDef mem fill:#f3e5f5,stroke:#6a1b9a,stroke-width:2px;
    classDef cache fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px;
    classDef prot fill:#fff3e0,stroke:#e65100,stroke-width:2px;
    classDef pref fill:#fce4ec,stroke:#880e4f,stroke-width:2px;
    classDef main fill:#ffebee,stroke:#c62828,stroke-width:3px;

    class ROB,RS,DISPATCH cpu;
    class LSQ,SQ,LQ,SBUF mem;
    class DTLB,L2TLB,PTW,DC cache;
    class PMP prot;
    class SMS,PRF_UNIT pref;
    class STUNIT main;
```

---

## Detailed Component Interactions

### 1. Reservation Station (RS) Interface

#### Input: Store Instruction Issue

```mermaid
sequenceDiagram
    participant RS
    participant StoreUnit
    participant RegFile

    RS->>RS: Select store for issue
    RS->>RegFile: Read src(0), src(1)
    RegFile->>RS: base_addr, store_data

    RS->>StoreUnit: stin.valid = 1<br>stin.bits.src(0) = base_addr<br>stin.bits.src(1) = store_data<br>stin.bits.uop = micro_op

    alt StoreUnit Ready
        StoreUnit->>RS: stin.ready = 1
        Note over StoreUnit: Accept and process
    else StoreUnit Busy
        StoreUnit->>RS: stin.ready = 0
        Note over RS: Hold instruction,<br>retry next cycle
    end
```

**Interface Details**:

```scala
// Input from RS (DecoupledIO)
io.stin.valid    : Bool              // RS has a store to issue
io.stin.ready    : Bool              // StoreUnit can accept
io.stin.bits     : ExuInput          // Store instruction bundle
  .src(0)        : UInt[64]          // Base address register
  .src(1)        : UInt[64]          // Store data register
  .uop           : MicroOp           // Micro-operation details
    .ctrl.imm    : UInt[12]          // Immediate offset
    .ctrl.fuOpType : UInt           // Operation type (SB/SH/SW/SD)
    .robIdx      : RobPtr            // ROB index for ordering
    .sqIdx       : SqPtr             // Store Queue index
```

**Ready Condition**: `stin.ready = s1_ready` (backpressure from S1)

#### Output: Feedback to RS

```mermaid
sequenceDiagram
    participant StoreUnit
    participant RS

    Note over StoreUnit: S1: TLB miss detected

    StoreUnit->>StoreUnit: Generate feedback<br>hit=0, sourceType=tlbMiss

    Note over StoreUnit: S2: Register feedback

    StoreUnit->>RS: feedback_slow.valid = 1<br>feedback_slow.bits.hit = 0<br>feedback_slow.bits.rsIdx = X<br>feedback_slow.bits.sourceType = tlbMiss

    RS->>RS: Mark entry X for replay
    RS->>RS: Clear valid bit for entry X

    Note over RS: Wait for TLB refill

    Note over RS: After TLB refill

    RS->>StoreUnit: Re-issue store (stin.valid)
```

**Feedback Interface**:

```scala
// Output to RS (ValidIO, delayed 1 cycle)
io.feedback_slow.valid : Bool        // Feedback valid
io.feedback_slow.bits  : RSFeedback
  .rsIdx        : UInt               // RS entry index
  .hit          : Bool               // TLB hit (0 = miss)
  .flushState   : Bool               // PTW feedback flag
  .sourceType   : UInt[3]            // Feedback type (tlbMiss = 1)
```

**Feedback Timing**: Generated in S1, sent in S2 (1-cycle delay for timing)

**Feedback Types**:
- `tlbMiss (1)`: Replay after TLB/PTW refill

---

### 2. TLB (DTLB) Interface

#### TLB Request Flow

```mermaid
sequenceDiagram
    participant S0 as StoreUnit S0
    participant DTLB
    participant L2TLB
    participant PTW
    participant S1 as StoreUnit S1

    S0->>DTLB: tlb.req.valid = 1<br>vaddr = computed_addr<br>cmd = TlbCmd.write<br>size = operation_size<br>memidx.is_st = 1<br>memidx.idx = sq_index

    alt TLB Hit
        DTLB->>S1: tlb.resp.valid = 1<br>paddr = translated_addr<br>miss = 0<br>excp = {pf.st=0, af.st=0}
        Note over S1: Use paddr,<br>continue pipeline
    else TLB Miss, L2 TLB Hit
        DTLB->>L2TLB: L2 TLB lookup
        L2TLB->>DTLB: Refill DTLB entry
        DTLB->>S1: tlb.resp.valid = 1<br>paddr = translated_addr<br>miss = 1<br>ptwBack = 0
        Note over S1: Kill pipeline,<br>send feedback to RS
    else TLB Miss, L2 TLB Miss
        DTLB->>L2TLB: L2 TLB lookup
        L2TLB->>PTW: Page table walk
        PTW->>PTW: Walk page tables
        PTW->>L2TLB: PTE found
        L2TLB->>DTLB: Refill entry
        DTLB->>S1: tlb.resp.valid = 1<br>miss = 1<br>ptwBack = 1
        Note over S1: Kill pipeline,<br>feedback to RS
    else Page Fault
        DTLB->>S1: tlb.resp.valid = 1<br>excp.pf.st = 1<br>miss = 0
        Note over S1: Record exception,<br>continue for commit
    end
```

**TLB Request Interface**:

```scala
// Stage 0 → TLB
io.tlb.req.valid         : Bool      // Request valid
io.tlb.req.bits          : TlbReq
  .vaddr                 : UInt[39]  // Virtual address
  .cmd                   : UInt      // TlbCmd.write
  .size                  : UInt[2]   // 0=byte, 1=half, 2=word, 3=double
  .kill                  : Bool      // Kill request (always 0)
  .memidx                : Bundle
    .is_ld               : Bool      // 0 for stores
    .is_st               : Bool      // 1 for stores
    .idx                 : UInt      // Store Queue index
  .debug                 : Bundle
    .robIdx              : RobPtr    // For tracking
    .pc                  : UInt[39]  // For debug
    .isFirstIssue        : Bool      // First issue flag
  .no_translate          : Bool      // Always 0 for normal stores
```

**TLB Response Interface**:

```scala
// TLB → Stage 1
io.tlb.resp.valid        : Bool      // Response valid
io.tlb.resp.ready        : Bool      // Always 1 (StoreUnit always accepts)
io.tlb.resp.bits         : TlbResp
  .paddr                 : Vec[UInt[36]]  // paddr(0) for single request
  .miss                  : Bool      // TLB miss flag
  .excp                  : Vec[Bundle]    // excp(0) for single request
    .pf.st               : Bool      // Store page fault
    .af.st               : Bool      // Store access fault
  .ptwBack               : Bool      // PTW feedback flag
  .memidx                : Bundle    // Echo of request memidx
    .is_st               : Bool
    .idx                 : UInt
```

**TLB Timing**:
- Request sent in S0
- Response received in S1 (1-cycle latency for DTLB hit)
- L2 TLB hit: 2-3 cycle latency
- PTW: 10-200+ cycle latency

---

### 3. DCache Interface

#### DCache Store Pipe Interaction

```mermaid
sequenceDiagram
    participant S0 as StoreUnit S0
    participant S1 as StoreUnit S1
    participant S2 as StoreUnit S2
    participant DC_S0 as DCache S0
    participant DC_S1 as DCache S1
    participant DC_S2 as DCache S2

    S0->>DC_S0: dcache.req.valid = 1<br>vaddr = vaddr<br>cmd = M_PFW<br>instrtype = STORE_SOURCE

    alt DCache Ready
        DC_S0->>DC_S0: Read tag/meta arrays
        Note over S0,S1: Pipeline advance

        S1->>DC_S1: dcache.s1_paddr = paddr<br>dcache.s1_kill = kill_signal

        alt Kill Signal
            Note over DC_S1: Cancel tag lookup,<br>discard results
        else No Kill
            DC_S1->>DC_S1: Compare tags,<br>determine hit/miss
            Note over S1,S2: Pipeline advance

            S2->>DC_S2: dcache.s2_kill = kill_signal<br>dcache.s2_pc = pc

            alt Kill Signal
                Note over DC_S2: Cancel operation
            else No Kill
                DC_S2->>S2: dcache.resp.valid = 1<br>miss = 0/1<br>replay = 0/1
                Note over S2: Use miss info<br>for prefetch
            end
        end
    else DCache Not Ready
        Note over S0: Continue anyway<br>(store doesn't wait)
    end
```

**DCache Request Interface**:

```scala
// Stage 0 → DCache
io.dcache.req.valid      : Bool      // Request valid
io.dcache.req.ready      : Bool      // DCache ready (may be ignored)
io.dcache.req.bits       : DcacheStoreRequestIO
  .cmd                   : UInt      // M_PFW (prefetch for write)
  .vaddr                 : UInt[39]  // Virtual address
  .instrtype             : UInt      // STORE_SOURCE or DCACHE_PREFETCH_SOURCE
```

**DCache Control Signals**:

```scala
// Stage 1 → DCache
io.dcache.s1_paddr       : UInt[36]  // Physical address
io.dcache.s1_kill        : Bool      // Kill tag lookup
  // Kill if: TLB miss || exception || MMIO || redirect

// Stage 2 → DCache
io.dcache.s2_kill        : Bool      // Kill operation
  // Kill if: MMIO || exception || redirect
io.dcache.s2_pc          : UInt[39]  // PC for debug

// DCache → Stage 2
io.dcache.resp.valid     : Bool      // Response valid
io.dcache.resp.ready     : Bool      // Always 1
io.dcache.resp.bits      : Bundle
  .miss                  : Bool      // Cache miss flag
  .replay                : Bool      // Replay required flag
  .tag_error             : Bool      // Tag ECC error
```

**Important Notes**:
1. **Not a real write**: DCache request is for tag/meta lookup only
2. **May not wait**: Store doesn't stall if DCache not ready
3. **Actual write**: Happens at commit through SBuffer
4. **Miss info**: Used for prefetch training only

---

### 4. Load-Store Queue (LSQ) Interface

#### LSQ Update Flow

```mermaid
sequenceDiagram
    participant S0 as StoreUnit S0
    participant S1 as StoreUnit S1
    participant S2 as StoreUnit S2
    participant SQ as Store Queue
    participant LQ as Load Queue

    Note over S0: Compute mask

    S0->>SQ: st_mask_out.valid = 1<br>sqIdx = sq_index<br>mask = byte_mask

    SQ->>SQ: Write mask to<br>SQ entry

    Note over S0,S1: Pipeline advance

    S1->>SQ: lsq.valid = 1<br>paddr = physical_addr<br>vaddr = virtual_addr<br>exceptions = exc_vec<br>tlbMiss = miss_flag

    SQ->>SQ: Write paddr,<br>exceptions to<br>SQ entry

    S1->>LQ: stld_nuke_query.valid = 1<br>robIdx = store_rob_idx<br>paddr = store_paddr<br>mask = store_mask

    LQ->>LQ: Check executed loads<br>for address overlap

    alt Violation Detected
        LQ->>LQ: Trigger pipeline flush,<br>replay from violating load
    else No Violation
        Note over LQ: Continue normal operation
    end

    Note over S1,S2: Pipeline advance

    S2->>SQ: lsq_replenish.mmio = mmio_flag<br>lsq_replenish.atomic = atomic_flag<br>lsq_replenish.miss = cache_miss<br>lsq_replenish.exceptions = final_exc

    SQ->>SQ: Update MMIO/atomic flags,<br>final exception status
```

**LSQ Interface Details**:

```scala
// Stage 0 → Store Queue (immediate)
io.st_mask_out.valid     : Bool      // Mask valid
io.st_mask_out.bits      : StoreMaskBundle
  .sqIdx                 : SqPtr     // Store Queue index
  .mask                  : UInt[16]  // Byte mask (VLEN/8 = 16)

// Stage 1 → Store Queue
io.lsq.valid             : Bool      // Update valid
io.lsq.bits              : LsPipelineBundle
  .vaddr                 : UInt[39]  // Virtual address
  .paddr                 : UInt[36]  // Physical address
  .mask                  : UInt[16]  // Byte mask
  .data                  : UInt[129] // Store data
  .uop                   : MicroOp   // Micro-op (includes sqIdx)
    .sqIdx               : SqPtr     // Store Queue index
  .tlbMiss               : Bool      // TLB miss flag
  .uop.cf.exceptionVec   : Vec[Bool] // Exception vector
  // ... other fields

// Stage 1 → Load Queue (violation detection)
io.stld_nuke_query.valid : Bool      // Query valid
io.stld_nuke_query.bits  : StoreNukeQueryIO
  .robIdx                : RobPtr    // Store ROB index
  .paddr                 : UInt[36]  // Store physical address
  .mask                  : UInt[16]  // Store byte mask

// Stage 2 → Store Queue (replenishment)
io.lsq_replenish         : LsPipelineBundle
  .mmio                  : Bool      // Final MMIO flag
  .atomic                : Bool      // Final atomic flag
  .miss                  : Bool      // DCache miss flag
  .uop.cf.exceptionVec   : Vec[Bool] // Final exceptions
  // ... complete bundle
```

**Update Sequence**:
1. **S0**: Send byte mask to SQ (early for timing)
2. **S1**: Send paddr and initial exception info to SQ
3. **S1**: Query LQ for store-load violations
4. **S2**: Send final MMIO/atomic/exception info to SQ

---

### 5. PMP (Physical Memory Protection) Interface

#### PMP Check Flow

```mermaid
sequenceDiagram
    participant S1 as StoreUnit S1
    participant PMP
    participant S2 as StoreUnit S2

    Note over S1: Physical address<br>available from TLB

    S1->>PMP: (implicit) paddr for check

    PMP->>PMP: Check against<br>PMP configuration<br>registers

    alt Normal Memory
        PMP->>S2: pmp.mmio = 0<br>pmp.atomic = 0<br>pmp.st = 0 (no fault)
        Note over S2: Normal store,<br>continue
    else MMIO Region
        PMP->>S2: pmp.mmio = 1<br>pmp.atomic = 0<br>pmp.st = 0
        Note over S2: Mark as MMIO,<br>kill pipeline
    else Atomic Region
        PMP->>S2: pmp.atomic = 1<br>pmp.mmio = 0<br>pmp.st = 0
        Note over S2: Mark as atomic,<br>continue
    else Access Violation
        PMP->>S2: pmp.st = 1<br>(access fault)
        Note over S2: Record exception,<br>continue to commit
    end
```

**PMP Response Interface**:

```scala
// PMP → Stage 2
io.pmp                   : PMPRespBundle
  .mmio                  : Bool      // MMIO region flag
  .atomic                : Bool      // Atomic region flag
  .st                    : Bool      // Store access fault

// Internal processing in S2
val s2_pmp = WireInit(io.pmp)
val s2_mmio = s2_in.mmio || s2_pmp.mmio
val s2_atomic = s2_in.atomic || s2_pmp.atomic

s2_out.uop.cf.exceptionVec[storeAccessFault] :=
  s2_in.uop.cf.exceptionVec[storeAccessFault] || s2_pmp.st
```

**PMP Timing**: Combinational response in S2

---

### 6. Prefetch System Interface

#### Prefetch Training Flow

```mermaid
sequenceDiagram
    participant S2 as StoreUnit S2
    participant DC as DCache
    participant S3 as StoreUnit S3
    participant SMS as SMS Prefetcher
    participant PRF as Prefetch Unit

    DC->>S2: dcache.resp.valid = 1<br>miss = 1 (cache miss)

    Note over S2: Generate training signal

    S2->>S3: Register training data<br>(1 cycle delay)

    S3->>SMS: prefetch_train.valid = 1<br>vaddr = store_vaddr<br>paddr = store_paddr<br>pc = store_pc<br>miss = 1<br>mask = store_mask

    SMS->>SMS: Update prefetch<br>prediction tables

    SMS->>SMS: Decide to issue<br>prefetch request

    SMS->>PRF: Generate prefetch hint

    PRF->>StoreUnit: prefetch_req.valid = 1<br>vaddr = predicted_addr

    Note over StoreUnit: Process as<br>hardware prefetch<br>(isHWPrefetch = 1)
```

**Prefetch Request Input**:

```scala
// Prefetch Unit → Stage 0
io.prefetch_req.valid    : Bool      // Prefetch request valid
io.prefetch_req.ready    : Bool      // StoreUnit can accept
  // ready = s1_ready && dcache.req.ready && !s0_iss_valid
io.prefetch_req.bits     : StorePrefetchReq
  .vaddr                 : UInt[39]  // Prefetch virtual address
```

**Prefetch Training Output**:

```scala
// Stage 3 → SMS Prefetcher
io.prefetch_train.valid  : Bool      // Training signal valid
  // valid = s2_valid && dcache.resp.fire && !mmio && !tlbMiss && !isHWPrefetch
io.prefetch_train.bits   : StPrefetchTrainBundle
  .vaddr                 : UInt[39]  // Store virtual address
  .paddr                 : UInt[36]  // Store physical address
  .mask                  : UInt[16]  // Store mask
  .uop.cf.pc             : UInt[39]  // Store PC
  .miss                  : Bool      // Cache miss flag
  .meta_prefetch         : UInt      // Prefetch source
  .meta_access           : Bool      // Access flag
```

**Training Conditions**:
- DCache response received (hit/miss known)
- Not MMIO operation
- Not TLB miss
- Not hardware prefetch itself
- EnableStorePrefetchSMS = true

**Training Timing**: Registered in S2, sent in S3 (1-cycle delay)

---

### 7. ROB (Reorder Buffer) Interface

#### Store Writeback Flow

```mermaid
sequenceDiagram
    participant SX as StoreUnit SX
    participant ROB
    participant LSQ
    participant SBUF as SBuffer

    SX->>ROB: stout.valid = 1<br>uop = micro_op<br>debug.isMMIO = mmio_flag<br>debug.paddr = paddr<br>debug.vaddr = vaddr<br>exceptionVec = exceptions

    alt No Exception
        ROB->>ROB: Mark store as<br>completed/ready to commit

        Note over ROB: At commit point

        alt MMIO Store
            ROB->>LSQ: Commit MMIO store
            LSQ->>LSQ: Handle MMIO<br>at commit time
        else Normal Store
            ROB->>LSQ: Commit normal store
            LSQ->>SBUF: Drain to SBuffer
            SBUF->>SBUF: Merge and coalesce<br>stores
            SBUF->>SBUF: Write to cache<br>or memory
        end

    else Exception Detected
        ROB->>ROB: Trigger exception<br>handler

        ROB->>ROB: Flush pipeline

        ROB->>ROB: Jump to exception<br>handler PC

        Note over ROB: Store not committed
    end
```

**Writeback Interface**:

```scala
// Stage X → ROB
io.stout.valid           : Bool      // Writeback valid
io.stout.ready           : Bool      // ROB ready to accept
io.stout.bits            : ExuOutput
  .uop                   : MicroOp   // Micro-operation
    .robIdx              : RobPtr    // ROB index
    .cf.exceptionVec     : Vec[Bool] // Exception vector
    .cf.pc               : UInt[39]  // PC for exception handling
  .data                  : UInt[64]  // DontCare for stores
  .redirectValid         : Bool      // Always false for stores
  .redirect              : Redirect  // DontCare
  .debug                 : DebugBundle
    .isMMIO              : Bool      // MMIO flag
    .isPerfCnt           : Bool      // Always false
    .paddr               : UInt[36]  // Physical address
    .vaddr               : UInt[39]  // Virtual address
  .fflags                : UInt[5]   // DontCare for stores
```

**Writeback Timing**: After S0+S1+S2+S3+SX delay cycles

**ROB Actions**:
1. Mark store as completed
2. Check for exceptions
3. At commit: handle MMIO or drain to SBuffer
4. Update retirement counters

---

## Inter-Component Timing Diagram

### Complete Store Execution Timeline

```mermaid
gantt
    title Complete Store Execution with All Components
    dateFormat X
    axisFormat %L

    section StoreUnit
    S0 (Addr Gen)       :s0, 0, 1000
    S1 (TLB Resp)       :s1, 1000, 2000
    S2 (PMP Check)      :s2, 2000, 3000
    S3 (WB Prep)        :s3, 3000, 4000
    SX (Delay)          :sx, 4000, 6000
    Writeback           :wb, 6000, 7000

    section TLB
    Receive Req         :tlb_r, 0, 1000
    Lookup              :tlb_l, 1000, 2000
    Send Resp           :tlb_s, 2000, 3000

    section DCache
    Receive Req         :dc_r, 0, 1000
    Tag Lookup          :dc_t, 1000, 2000
    Send Resp           :dc_s, 2000, 3000

    section LSQ
    Receive Mask        :lsq_m, 0, 1000
    Receive Paddr       :lsq_p, 1000, 2000
    Violation Check     :lsq_v, 2000, 3000
    Receive Replenish   :lsq_r, 3000, 4000
    Wait Commit         :lsq_w, 4000, 7000

    section PMP
    Receive Paddr       :pmp_r, 1000, 2000
    Check               :pmp_c, 2000, 3000
    Send Resp           :pmp_s, 3000, 4000

    section ROB
    Wait WB             :rob_w, 0, 6000
    Receive WB          :rob_r, 6000, 7000
    Mark Complete       :rob_m, 7000, 8000
    Commit              :rob_c, 8000, 9000

    section SBuffer
    Wait Commit         :sb_w, 0, 8000
    Receive Store       :sb_r, 8000, 9000
    Drain to Cache      :sb_d, 9000, 12000
```

---

## Pipeline Bubble Propagation

### Bubble Example: TLB Miss

```mermaid
sequenceDiagram
    participant RS
    participant S0
    participant S1
    participant DTLB
    participant PTW
    participant L2TLB

    Note over RS: Store ready

    RS->>S0: Issue store
    S0->>DTLB: TLB request (vaddr)

    Note over S0,S1: Cycle boundary

    DTLB->>DTLB: Lookup: MISS
    DTLB->>L2TLB: L2 TLB request
    L2TLB->>L2TLB: Lookup: MISS
    L2TLB->>PTW: Start page walk

    DTLB->>S1: resp (miss=1, ptwBack=1)

    S1->>S1: s1_kill = 1
    S1->>S0: Pipeline bubble
    S1->>RS: feedback (tlbMiss)

    Note over PTW: Walk page tables<br>(10-200 cycles)

    PTW->>L2TLB: PTE found
    L2TLB->>DTLB: Refill DTLB

    Note over RS: Wait complete

    RS->>S0: Re-issue store
    S0->>DTLB: TLB request (same vaddr)
    DTLB->>S1: resp (HIT, paddr)

    Note over S1: Continue normally
```

---

## Error and Exception Propagation

### Exception Accumulation Through Pipeline

```mermaid
graph TB
    subgraph "S0: Initial Exceptions"
        E0[storeAddrMisaligned]
    end

    subgraph "S1: TLB Exceptions"
        E0 --> E1A[storeAddrMisaligned]
        TLB_PF[TLB: storePageFault] --> E1B[storePageFault]
        TLB_AF[TLB: storeAccessFault] --> E1C[storeAccessFault]
    end

    subgraph "S2: PMP Exceptions"
        E1A --> E2A[storeAddrMisaligned]
        E1B --> E2B[storePageFault]
        E1C --> E2C[storeAccessFault]
        PMP_AF[PMP: st fault] --> E2D[storeAccessFault]
        E2C -.OR.-> E2D
    end

    subgraph "S3/SX/WB: Exception Vector"
        E2A --> EV[Exception Vector<br>to ROB]
        E2B --> EV
        E2D --> EV
    end

    subgraph "ROB: Exception Handling"
        EV --> PRIOR[Priority Encoder]
        PRIOR --> TRAP[Highest Priority<br>Exception Taken]
    end

    classDef exc fill:#ffccbc,stroke:#d84315,stroke-width:2px;
    classDef vec fill:#fff9c4,stroke:#f57f17,stroke-width:2px;
    classDef rob fill:#c5e1a5,stroke:#558b2f,stroke-width:2px;

    class E0,TLB_PF,TLB_AF,PMP_AF exc;
    class E1A,E1B,E1C,E2A,E2B,E2C,E2D,EV vec;
    class PRIOR,TRAP rob;
```

**Exception Priority** (high to low):
1. Store Address Misaligned (S0)
2. Store Page Fault (S1)
3. Store Access Fault (S1, S2)

---

## Resource Conflict Handling

### TLB Port Arbitration

```mermaid
graph TB
    subgraph "TLB Request Sources"
        L0[LoadUnit 0]
        L1[LoadUnit 1]
        S0[StoreUnit 0]
        S1[StoreUnit 1]
        PF[Prefetch]
    end

    subgraph "TLB Arbiter"
        ARB{Arbiter<br>Priority}
    end

    subgraph "DTLB"
        PORT0[Port 0]
        PORT1[Port 1]
    end

    L0 --> ARB
    L1 --> ARB
    S0 --> ARB
    S1 --> ARB
    PF --> ARB

    ARB -->|High Priority| PORT0
    ARB -->|Low Priority| PORT1

    Note1[Priority:<br>Load > Store > Prefetch]

    classDef source fill:#e3f2fd,stroke:#1565c0,stroke-width:1px;
    classDef arb fill:#fff3e0,stroke:#ef6c00,stroke-width:2px;
    classDef port fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px;

    class L0,L1,S0,S1,PF source;
    class ARB arb;
    class PORT0,PORT1 port;
```

**Conflict Resolution**:
- Loads have higher priority (critical for performance)
- Stores can tolerate some latency
- Prefetch has lowest priority

---

## Configuration-Dependent Behavior

### Impact of Load Queue Size on Delay Stages

```mermaid
graph LR
    subgraph "Configuration Parameters"
        LQ_SIZE[LoadQueueRAWSize]
        RB_SIZE[RollbackGroupSize]
    end

    subgraph "Calculation"
        CALC[TotalSelectCycles =<br>ceil log2 LQ_SIZE / log2 RB_SIZE + 1]
        DELAY[TotalDelayCycles =<br>TotalSelectCycles - 2]
    end

    subgraph "Pipeline Impact"
        DEPTH[Total Pipeline Depth =<br>4 + TotalDelayCycles]
        LAT[Store Latency =<br>DEPTH cycles]
    end

    LQ_SIZE --> CALC
    RB_SIZE --> CALC
    CALC --> DELAY
    DELAY --> DEPTH
    DEPTH --> LAT

    classDef param fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef calc fill:#fff3e0,stroke:#e65100,stroke-width:2px;
    classDef impact fill:#f3e5f5,stroke:#6a1b9a,stroke-width:2px;

    class LQ_SIZE,RB_SIZE param;
    class CALC,DELAY calc;
    class DEPTH,LAT impact;
```

**Example Configurations**:

| Config | LQ RAW Size | Rollback Size | Total Select | Delay Cycles | Total Depth |
|--------|-------------|---------------|--------------|--------------|-------------|
| Default | 80 | 8 | 4 | 2 | 6 cycles |
| Minimal | 40 | 8 | 3 | 1 | 5 cycles |
| Large | 160 | 8 | 5 | 3 | 7 cycles |

---

## Appendix: Interface Summary Table

### Complete Interface Reference

| Interface | Direction | Stage | Purpose | Timing |
|-----------|-----------|-------|---------|--------|
| `stin` | Input | S0 | Store instruction from RS | Ready/Valid |
| `tlb.req` | Output | S0 | TLB translation request | Combinational |
| `tlb.resp` | Input | S1 | TLB translation response | 1-cycle latency |
| `dcache.req` | Output | S0 | DCache tag lookup | May not wait |
| `dcache.s1_paddr` | Output | S1 | Physical address | Combinational |
| `dcache.s1_kill` | Output | S1 | Kill S0 request | Combinational |
| `dcache.s2_kill` | Output | S2 | Kill S1 request | Combinational |
| `dcache.resp` | Input | S2 | DCache hit/miss | 2-cycle latency |
| `pmp` | Input | S2 | PMP check result | Combinational |
| `st_mask_out` | Output | S0 | Store mask to SQ | Combinational |
| `lsq` | Output | S1 | Store info to LSQ | Combinational |
| `lsq_replenish` | Output | S2 | Final store info | Combinational |
| `stld_nuke_query` | Output | S1 | Violation check | Combinational |
| `feedback_slow` | Output | S2 | Feedback to RS | 1-cycle delayed |
| `prefetch_req` | Input | S0 | Prefetch request | Ready/Valid |
| `prefetch_train` | Output | S3 | Prefetch training | 1-cycle delayed |
| `stout` | Output | SX | Writeback to ROB | Ready/Valid |
| `redirect` | Input | All | Pipeline flush | Asynchronous |

---

## Revision History

| Date | Version | Changes |
|------|---------|---------|
| 2026-02-02 | 1.0 | Initial component interaction analysis |
