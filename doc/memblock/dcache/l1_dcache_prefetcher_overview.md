# L1 DCache Prefetcher Overview (Big Picture)

This document provides a structured, top-down view of the L1 DCache prefetch system. It covers the **overall pipeline**, **modules**, and **interfaces**, and explains connections to **TLB**, **L1 DCache**, **LoadPipe**, **StorePipe**, and **SBuffer**.

---

**0. Big Picture (One Page Mental Model)**

At a high level, the prefetch system does **not** pull “ready-but-not-issued” uops from VLQ/SQ. Instead it **learns patterns from executed memory accesses** and then **injects predicted addresses** into the cache hierarchy through dedicated prefetch paths.

Think of it as three stages:

1. **Train (observe real accesses)**  
   Load/Store activity generates training signals. The prefetchers observe these signals to learn **stream/stride/spatial patterns**.

   **What “training signals” mean here**
   - The prefetchers see **retired/executed memory accesses** (vaddr/paddr/PC, miss/hit info) from Load/Store pipelines.
   - The training is **not** “issue a ready uop when the pipe is idle”; it is **learning from completed accesses** and building prediction state.

   **Example 1: Stream pattern (sequential scan)**  
   Suppose a loop walks a large array `A` in cache-line order:
   ```
   A[0], A[1], A[2], A[3], ...
   ```
   - Each access trains the stream detector.
   - The stream tracker observes **consecutive cache-line addresses** and marks a stream region.
   - It then predicts **future lines** ahead of the current one and issues prefetch requests for those lines.

   **Example 2: Stride pattern (fixed offset)**  
   Suppose accesses jump by a fixed stride (e.g., struct-of-arrays or strided loop):
   ```
   A[0], A[16], A[32], A[48], ...
   ```
   - The stride tracker sees the **constant delta** between consecutive accesses.
   - It learns “stride = +16 lines” and prefetches `A[64]`, `A[80]`, etc.

   **Example 3: SMS (spatial memory streaming)**
   Suppose accesses repeatedly touch a few lines within a region (spatial locality):
   ```
   Region R: lines {0, 2, 3, 7} keep appearing across iterations
   ```
   - SMS builds **region-level patterns** (Active Generation Table / Pattern History Table).
   - When a new region access appears, SMS predicts the **rest of the pattern** in that region and prefetches those lines to L2.

   In all cases, the **training source is real load/store traffic**, and the prefetcher generates **new addresses** from learned patterns rather than replaying pending uops.

2. **Predict (generate future addresses)**  
   The prefetchers compute candidate addresses based on learned patterns and filter them to avoid duplicates or low-value requests.

3. **Dispatch (issue prefetches opportunistically)**  
   Prefetch requests are issued to **L1 (via LoadPipe)** or **L2/L3 (via prefetch senders)** only when resources allow.

Key consequences:
- Prefetches are **pattern-based**, not “idle-pipe opportunistic issue” of existing uops.
- **L1 prefetches use LoadPipe resources** and can be stalled or replayed like normal loads.
- **L2 prefetches (SMS + L1 prefetcher L2 output)** bypass LoadPipe and go straight to the L2 prefetch path.

Simplified flow:

```
Real Loads/Stores
   | train signals
   v
Prefetcher (L1 stream/stride + SMS)
   |---- L1 prefetch -> LoadPipe -> DCache
   |---- L2/L3 prefetch -> L2/L3 prefetch sender
   |
   +---- TLB prefetch port (address translation for L1/L2/L3)
```

If you keep this model in mind, the rest of the document maps details onto these three steps.

---

**1. Prefetchers in the System (What Exists)**

This core integrates **two hardware prefetchers**:

1. **L1 Prefetcher (Stream + Stride)**
- Module: `L1Prefetcher` in `L1PrefetchComponent.scala`
- Emits L1 prefetch requests into **LoadPipe** (as pseudo-loads)
- Also emits L2/L3 prefetch addresses

2. **SMS Prefetcher (Spatial Memory Streaming)**
- Module: `SMSPrefetcher` in `SMSPrefetcher.scala`
- Emits **L2 prefetch** requests directly (no L1 injection)
- Runs in parallel with L1 Prefetcher and is selected by `coreParams.prefetcher`

Notes:
- Both are **hardware prefetchers** instantiated in `MemBlock.scala`.
- L1 prefetcher is always used when prefetching is enabled; SMS is optional based on config.

---

**2. System-Level Placement (MemBlock)**

```
LoadUnits / StoreUnits
   | (train info)
   v
L1Prefetcher (stream+stride)
   | l1_req (Decoupled)
   v
LoadUnit.prefetch_req -> LoadPipe (DCache load path)

L1Prefetcher
   | l2_req (Valid)
   v
L2 Prefetch Sender (to L2)

SMSPrefetcher
   | l2_req (Valid)
   v
L2 Prefetch Sender (to L2)

L1Prefetcher
   | tlb_req
   v
DTLB Prefetch Port
```

Primary integration points:
- `MemBlock.scala`: instantiation and wiring for L1Prefetcher and SMSPrefetcher
- `L1PrefetchComponent.scala`: L1 prefetch core logic (stream + stride)
- `SMSPrefetcher.scala`: SMS prefetch core logic
- `BasePrefecher.scala`: Prefetcher IO definition

---

**3. End-to-End Pipeline (Big Picture)**

The L1 prefetch system follows three conceptual stages:

1. **Training** from Load/Store activity
2. **Request generation + filtering** (stream/stride + MLP filter)
3. **Translation + dispatch** (TLB + L1/L2/L3 outputs)

---

**4. L1 Prefetcher (Stream + Stride) — Internal Pipeline**

### 4.1 Training Stage

Inputs:
- `ld_in`: load training input (`LdPrefetchTrainBundle`)
- `stride_train`: filtered load training for stride prefetcher
- `st_in`: store training input exists in IO but is **not used** by L1Prefetcher

Key behavior:
- **TrainFilter** removes duplicates by cache line
- Limits training to **1 entry per cycle**
- Multiple LoadUnits are **reordered by ROB age** before training, so out-of-order execution does not scramble the learning sequence.

### 4.2 Pattern Extraction

Modules:
- `StreamBitVectorArray` for stream detection
- `StrideMetaArray` for stride detection

Priority:
- **Stream** candidates take priority over **stride** candidates

### 4.3 Example: Stream Training (A[0], A[1], A[2]) with Entry Evolution

Assume a loop touches consecutive cache lines in the same region:
`A[0] -> A[1] -> A[2] -> A[3] ...`

Stream table entry (`StreamBitVectorBundle`) evolves as follows:

```
Initial entry:
tag = R, bit_vec = 0000_0000_0000_0000, cnt = 0, active = 0

After A[0]:
tag = R, bit_vec = 0000_0000_0000_0001, cnt = 1, active = 0

After A[1]:
tag = R, bit_vec = 0000_0000_0000_0011, cnt = 2, active = 0

After A[2]:
tag = R, bit_vec = 0000_0000_0000_0111, cnt = 3, active = 0

After more consecutive lines:
cnt reaches ACTIVE_THRESHOLD -> active = 1
```

Once `active = 1`, the stream prefetcher builds a **prefetch bit-vector** ahead of the current position and emits L1/L2/L3 candidates.

### 4.4 Example: Stride Training (A[0], A[16], A[32]) with Entry Evolution

Assume a fixed stride pattern by PC (same load instruction):
`A[0] -> A[16] -> A[32] -> A[48] ...`

Stride table entry (`StrideMetaBundle`) evolves by PC hash:

```
Initial entry:
hash_pc = H, pre_vaddr = 0, stride = 0, confidence = 0

After A[0]:
pre_vaddr = 0, stride = 0, confidence = 0

After A[16]:
new_stride = +16
stride = +16, confidence = 1
pre_vaddr = 16

After A[32]:
new_stride = +16 matches stride
confidence = 2
pre_vaddr = 32

After A[48]:
new_stride = +16 matches stride
confidence reaches MAX_CONF -> can_send_pf = true
```

When confidence reaches `MAX_CONF`, the stride prefetcher starts issuing prefetch requests at the learned stride distance (L1 and L2 depths are derived from this stride).

### 4.5 MLP Filter and Dispatch

Module: `MutiLevelPrefetchFilter`

Pipelines inside the filter:
1. **Enqueue**: region hash match, allocate or update entry
2. **TLB**: request translation, update entry with paddr
3. **L1 dispatch**: arbitrate and send `io.l1_req`
4. **L2/L3 dispatch**: arbitrate and send `io.l2_req` and `io.l3_req`

Gating conditions:
- `pf_ctrl.enable` and `io.enable`
- `io.l1_req.ready` backpressure
- `l2PfqBusy` throttling
- Address filter: paddr >= `0x8000_0000`

---

**5. SMS Prefetcher (HW Prefetcher to L2)**

Module: `SMSPrefetcher`

Big-picture behavior:
- Trains on load/store activity via `SMSTrainFilter`
- Generates **L2 prefetch** addresses (no L1 injection)
- Uses AGT/PHT style tracking (active generation + pattern history)

Key outputs:
- `io.l2_req` with `MemReqSource.Prefetch2L2SMS`

Impact:
- Competes with L1 prefetch for **L2 prefetch bandwidth**
- Independent from LoadPipe arbitration

---

**6. Connection to TLB (DTLB Prefetch Port)**

- L1Prefetcher issues `tlb_req` to **DTLB prefetch port**
- TLB response updates entries in MLP filter
- Needed because L1 DCache is VIPT but prefetch targets still require a valid paddr

---

**7. Connection to LoadPipe (L1 Prefetch Injection)**

L1 prefetch is injected as a **LoadUnit prefetch request**:

- `L1Prefetcher.io.l1_req` → MemBlock `l1_pf_req` → `loadUnit.io.prefetch_req`
- LoadUnits arbitrate acceptance based on **confidence** and availability
- Prefetches consume normal LoadPipe resources

Key implications:
- Prefetch can be replayed due to bank conflicts or MSHR pressure
- Low-confidence prefetch is restricted to a specific LoadUnit port

---

**8. Connection to L1 DCache (Prefetch Metadata)**

DCache maintains prefetch metadata:
- **Prefetch flag array** and **access flag array**
- Updated by LoadPipe and RefillPipe
- Monitored by `PrefetcherMonitor` and `FDPrefetcherMonitor`

Key signals:
- `prefetch_flag_write`
- `access_flag_write`

---

**9. StorePipe and SBuffer Interaction**

Store-related prefetch behavior:
- StorePipe can issue **prefetch write** on store miss (when enabled)
- Flow: StorePipe → MissQueue → RefillPipe → DCache arrays

SBuffer interaction:
- Store data commits via **SBuffer → MainPipe**
- Prefetch traffic competes for MissQueue and refill bandwidth

---

**10. LoadPipe Resource Interaction (Backpressure)**

Prefetch uses LoadPipe like a demand load, so it can be throttled by:

- S0 arbitration (demand load priority)
- Bank conflicts
- MissQueue backpressure
- MSHR availability

Outcome:
- Prefetch is **opportunistic** and should not steal latency-critical cycles from demand loads

---

**11. Full-System Diagram (Mermaid)**

```
flowchart LR
  %% styling
  classDef memory fill:#e8f5e9,stroke:#1b5e20,stroke-width:1px;
  classDef reg fill:#fff3e0,stroke:#e65100,stroke-width:1px;
  classDef logic fill:#e1f5fe,stroke:#01579b,stroke-width:1px;
  classDef io fill:#fce4ec,stroke:#880e4f,stroke-width:1px;

  subgraph Exec[Execution Pipelines]
    LDU[LoadUnit xN]:::logic
    STU[StoreUnit xN]:::logic
    SB[SBuffer]:::logic
  end

  subgraph L1PF[L1 Prefetcher]
    TF[TrainFilter<br>stream/stride]:::logic
    STR[StrideMetaArray]:::logic
    STM[StreamBitVectorArray]:::logic
    MLP[MutiLevelPrefetchFilter]:::logic
  end

  subgraph SMS[SMS Prefetcher]
    SMSPF[SMSPrefetcher]:::logic
  end

  subgraph TLB[DTLB]
    TLBPF[DTLB Prefetch Port]:::logic
  end

  subgraph DCache[L1 DCache]
    LP[LoadPipe xN]:::logic
    SP[StorePipe xN]:::logic
    MP[MainPipe]:::logic
    RP[RefillPipe]:::logic
    MQ[MissQueue]:::logic
    PFARR[Prefetch/Access Flag Arrays]:::memory
  end

  subgraph L2[L2/L3 Prefetch]
    L2PF[L2 PF Sender]:::logic
    L3PF[L3 PF Sender]:::logic
  end

  LDU -->|"ld_in / stride_train"| TF
  STU -.->|"st_in (unused in L1PF)"| TF
  TF --> STR
  TF --> STM
  STR -->|"prefetch_req"| MLP
  STM -->|"prefetch_req"| MLP

  MLP -->|"tlb_req"| TLBPF
  TLBPF -->|"tlb_resp"| MLP

  MLP -->|"l1_req"| LP
  MLP -->|"l2_req"| L2PF
  MLP -->|"l3_req"| L3PF

  SMSPF -->|"l2_req"| L2PF

  LP -->|"prefetch_flag_write"| PFARR
  RP -->|"prefetch_flag_write"| PFARR
  LP -->|"access_flag_write"| PFARR

  STU -->|"store miss -> prefetch write"| SP
  SP -->|"miss_req"| MQ
  SB -->|"store data"| MP
  MQ --> RP
```

---

**12. Files to Reference**

1. `src/main/scala/xiangshan/backend/MemBlock.scala`
2. `src/main/scala/xiangshan/mem/prefetch/L1PrefetchComponent.scala`
3. `src/main/scala/xiangshan/mem/prefetch/SMSPrefetcher.scala`
4. `src/main/scala/xiangshan/mem/prefetch/BasePrefecher.scala`
5. `src/main/scala/xiangshan/cache/dcache/DCacheWrapper.scala`
6. `doc/memblock/dcache/dcache_overview.md`
7. `doc/memblock/dcache/module_interfaces.md`

If you want, I can add cycle-accurate timing charts for L1 prefetch injection and MLP filter arbitration, or expand SMS internals (AGT/PHT/stride) in a separate document.
