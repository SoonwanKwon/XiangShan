# L1 DCache Architecture Overview

## 1. 전체 구조 개요

XiangShan의 L1 DCache는 Non-Blocking 캐시로, 여러 개의 파이프라인을 통해 Load, Store, Atomic 연산을 병렬로 처리할 수 있습니다.

### 1.1 기본 파라미터 (Default Configuration)
```
- DCacheSets: 256 (64 sets in some configs)
- DCacheWays: 8
- DCacheBanks: 8
- DCacheSRAMRowBits: 64
- BlockBytes: 64 bytes (512 bits)
- Total Size: 256 * 8 * 8 * 8 bytes = 128KB (default) 또는 32KB
```

### 1.2 주소 구조
```
           Physical Address
 --------------------------------------
 |   Physical Tag |  PIndex  | Offset |
 --------------------------------------
                  |
                  DCacheTagOffset

           Virtual Address
 --------------------------------------
 | Above index  | Set | Bank | Offset |
 --------------------------------------
                |     |      |        |
                |     |      |        0
                |     |      DCacheBankOffset (3)
                |     DCacheSetOffset (6)
                DCacheAboveIndexOffset
```

## 2. 핵심 모듈 구성

```
                            +------------------+
                            |     DCache       |
                            +------------------+
                                    |
        +----------+----------+-----+------+----------+----------+
        |          |          |            |          |          |
   +--------+ +--------+ +----------+ +----------+ +-------+ +--------+
   |LoadPipe| |LoadPipe| | StorePipe| | StorePipe| |MainPipe| |RefillPipe|
   |  [0]   | |  [1]   | |   [0]    | |   [1]    | |        | |         |
   +--------+ +--------+ +----------+ +----------+ +-------+ +--------+
        |          |          |            |          |          |
        +----------+----------+------------+----------+----------+
                                    |
        +----------------------------------------------------------+
        |                     Storage Arrays                        |
        | +------------+ +----------+ +------------+ +-----------+ |
        | | TagArray   | | MetaArray| | DataArray  | | ErrorArray| |
        | | (Dup x 3)  | | (Coh)    | | (Banked x8)| | Prefetch  | |
        | +------------+ +----------+ +------------+ +-----------+ |
        +----------------------------------------------------------+
                                    |
        +----------------------------------------------------------+
        |                    Control Modules                        |
        | +-----------+ +------------+ +-----------+ +----------+  |
        | | MissQueue | | ProbeQueue | | WritebackQ| | Replacer |  |
        | +-----------+ +------------+ +-----------+ +----------+  |
        +----------------------------------------------------------+
```

## 3. 주요 파이프라인

### 3.1 LoadPipe (Load Pipeline)
- **수량**: LoadPipelineWidth 개 (기본 2개)
- **역할**: Load 요청 처리, Prefetch 요청 처리
- **스테이지 구성**:

```
+-------+     +-------+     +-------+     +-------+
|  S0   | --> |  S1   | --> |  S2   | --> |  S3   |
+-------+     +-------+     +-------+     +-------+
 Tag/Meta     Tag Match     Data Resp    ECC Check
  Read        Data Read     Miss Judge   Data Return
```

| Stage | 동작 |
|-------|------|
| S0 | Tag/Meta Array 읽기 요청 발행, vaddr 수신 |
| S1 | Tag 비교, Hit/Miss 판정, Data Array 읽기 요청, paddr 수신 |
| S2 | Data 응답 수신, Miss 시 MissQueue 요청, replay 신호 생성 |
| S3 | ECC 검사, Data 반환, prefetch/access flag 업데이트 |

### 3.2 StorePipe (Store Pipeline)
- **수량**: StorePipelineWidth 개 (기본 2개)
- **역할**: Store Address (STA) 파이프라인, Prefetch 트리거
- **스테이지 구성**:

```
+-------+     +-------+     +-------+
|  S0   | --> |  S1   | --> |  S2   |
+-------+     +-------+     +-------+
 Tag/Meta     Tag Match     Prefetch
  Read         Hit Check     Miss Req
```

| Stage | 동작 |
|-------|------|
| S0 | Tag/Meta Array 읽기 요청 발행 |
| S1 | Tag 비교, Hit/Miss 판정, paddr 수신 |
| S2 | Miss 시 Prefetch Write 요청 (MissQueue로) |

**주의**: StorePipe는 실제 데이터 쓰기를 하지 않음. SBuffer를 통해 MainPipe로 쓰기 요청 전달.

### 3.3 MainPipe
- **수량**: 1개
- **역할**: Store 실제 쓰기, AMO, Probe, Replace 처리
- **스테이지 구성**:

```
+-------+     +-------+     +-------+     +-------+
|  S0   | --> |  S1   | --> |  S2   | --> |  S3   |
+-------+     +-------+     +-------+     +-------+
 Meta/Tag     Tag Match     Data Process  Write Back
  Read        Data Read     AMO Execute   Meta/Tag/Data
              Coh Check                    Write
```

| Stage | 동작 |
|-------|------|
| S0 | 요청 중재 (Probe > Replace > Store > Atomic), Meta/Tag 읽기 |
| S1 | Tag 비교, Coherence 상태 확인, Data 읽기, Replace Way 선택 |
| S2 | Data merge, Store miss 시 MissQueue 요청, AMO 연산 |
| S3 | Meta/Tag/Data 쓰기, Writeback 요청, Coherence 상태 업데이트 |

**요청 우선순위**:
1. Probe (L2에서 온 요청)
2. Replace (MissQueue에서 온 교체 요청)
3. Store (SBuffer에서 온 쓰기 요청)
4. Atomic (AMO 연산)

### 3.4 RefillPipe
- **수량**: 1개
- **역할**: MissQueue에서 refill 받은 데이터를 캐시에 기록
- **스테이지**: 1-stage 파이프라인

```
+-------+
|  S0   |
+-------+
 Write Tag/Meta/Data
 Update Error/Prefetch/Access flags
```

### 3.5 MissQueue
- **수량**: nMissEntries 개 (기본 16개)
- **역할**: Cache Miss 처리, L2 Acquire 발행, Refill 관리

**상태 머신**:
```
IDLE --> s_acquire --> w_grantfirst --> w_grantlast --> s_refill --> IDLE
                                    |
                                    +--> s_replace_req (if needed)
```

### 3.6 ProbeQueue
- **수량**: nProbeEntries 개
- **역할**: L2의 Probe 요청 큐잉, MainPipe로 전달

### 3.7 WritebackQueue
- **수량**: nReleaseEntries 개
- **역할**: Dirty 라인의 Release/ProbeAck 처리

## 4. 핵심 인터페이스

### 4.1 DCacheIO (Top-Level Interface)
```scala
class DCacheIO {
  val hartId: Input(UInt)
  val lsu: DCacheToLsuIO          // LSU 연결
  val csr: L1CacheToCsrIO         // CSR 연결
  val error: L1CacheErrorInfo     // 에러 정보
  val mshrFull: Output(Bool)      // MSHR 가득 참
  val force_write: Input(Bool)    // 강제 쓰기
  // ...
}
```

### 4.2 DCacheToLsuIO
```scala
class DCacheToLsuIO {
  val load: Vec[DCacheLoadIO]     // Load 포트 (LoadPipelineWidth)
  val sta: Vec[DCacheStoreIO]     // Store Address 포트 (StorePipelineWidth)
  val store: DCacheToSbufferIO    // SBuffer 연결
  val atomics: AtomicWordIO       // Atomic 연결
  val lsq: Refill                 // LSQ로의 Refill 정보
  val release: Release            // Release hint (ld-ld violation)
  val forward_D: DcacheToLduForwardIO  // TileLink D 채널 포워딩
  val forward_mshr: LduToMissqueueForwardIO  // MSHR 포워딩
}
```

### 4.3 DCacheLoadIO (Load Unit <-> LoadPipe)
```scala
class DCacheLoadIO {
  val req: DecoupledIO(DCacheWordReq)  // Load 요청
  val resp: DecoupledIO(DCacheWordResp) // Load 응답
  val s1_kill: Output(Bool)            // S1에서 kill
  val s2_kill: Output(Bool)            // S2에서 kill
  val s1_paddr: Output(UInt)           // S1 물리주소
  val s2_hit: Input(Bool)              // S2 hit 신호
  val s2_bank_conflict: Input(Bool)    // Bank conflict
  val s2_wpu_pred_fail: Input(Bool)    // Way Prediction 실패
}
```

### 4.4 DCacheToSbufferIO (SBuffer <-> MainPipe)
```scala
class DCacheToSbufferIO {
  val req: Flipped(Decoupled(DCacheLineReq))  // SBuffer -> DCache 쓰기 요청
  val main_pipe_hit_resp: ValidIO(DCacheLineResp)  // MainPipe hit 응답
  val refill_hit_resp: ValidIO(DCacheLineResp)     // Refill hit 응답
  val replay_resp: ValidIO(DCacheLineResp)         // Replay 응답
}
```

## 5. TLB와 DCache 연결

XiangShan에서 DCache는 **VIPT (Virtually Indexed, Physically Tagged)** 캐시입니다.
가상 주소로 인덱싱하고 물리 주소로 태그를 비교하므로, TLB 변환과 캐시 접근이 병렬로 수행됩니다.

### 5.1 DTLB 구성

MemBlock에는 세 가지 DTLB 인스턴스가 존재합니다:
```
- dtlb_ld: Load용 TLB (LduCnt + 1 포트, 2개의 응답 복사본)
- dtlb_st: Store용 TLB (StuCnt 포트, 1개의 응답 복사본)
- dtlb_prefetch: Prefetch용 TLB (1 포트, 2개의 응답 복사본)
```

### 5.2 Load 경로: TLB ↔ LoadUnit ↔ DCache

```
           LoadUnit                    DTLB                    DCache (LoadPipe)
           --------                    ----                    -----------------

[S0] +----------------+         +----------------+         +----------------+
     | vaddr 생성     |-------->| TLB Lookup     |         | Tag/Meta Read  |
     | TLB 요청 발행  |         | (vaddr→paddr)  |         | (vaddr index)  |
     +----------------+         +----------------+         +----------------+
            |                          |                          |
            | tlb.req.valid            |                          |
            | tlb.req.bits.vaddr       |                          |
            | tlb.req.bits.cmd=read    |                          |
            v                          v                          v
[S1] +----------------+         +----------------+         +----------------+
     | TLB 응답 수신  |<--------| paddr 반환     |         | Tag Compare    |
     | paddr를 DCache |         | miss 신호      |-------->| (paddr tag)    |
     | 로 전달        |-------->| exception      |         | Hit/Miss 판정  |
     +----------------+         +----------------+         +----------------+
            |                                                     |
            | s1_paddr_dup_dcache -------------------------------->|
            | s1_kill (if TLB miss) ----------------------------->|
            v                                                     v
[S2] +----------------+                                   +----------------+
     | Hit/Miss 수신  |<----------------------------------| Data Response  |
     | Replay 판정    |                                   | Miss → MissQ   |
     +----------------+                                   +----------------+
```

### 5.3 핵심 인터페이스: TlbRequestIO

```scala
class TlbRequestIO(nRespDups: Int = 1) {
  val req = DecoupledIO(new TlbReq)
  val req_kill = Output(Bool())
  val resp = Flipped(DecoupledIO(new TlbResp(nRespDups)))
}

class TlbReq {
  val vaddr = UInt(VAddrBits.W)         // 가상 주소
  val cmd = TlbCmd()                     // read/write/exec
  val size = UInt(2.W)                   // 접근 크기
  val memidx = new MemBlockidxBundle     // LQ/SQ 인덱스
  // ...
}

class TlbResp(nDups: Int) {
  val paddr = Vec(nDups, UInt(PAddrBits.W))  // 물리 주소 (복사본)
  val miss = Bool()                           // TLB 미스
  val excp = Vec(nDups, new Bundle {
    val pf = TlbExceptionBundle()            // Page Fault
    val af = TlbExceptionBundle()            // Access Fault
  })
  val ptwBack = Bool()                        // PTW 응답
}
```

### 5.4 LoadUnit에서의 TLB-DCache 신호 흐름

```scala
// LoadUnit.scala 주요 신호

// S0: TLB 요청 발행
io.tlb.req.valid := s0_valid
io.tlb.req.bits.vaddr := s0_vaddr
io.tlb.req.bits.cmd := TlbCmd.read

// S1: TLB 응답 수신 및 DCache로 paddr 전달
val s1_paddr_dup_lsu = io.tlb.resp.bits.paddr(0)      // LSU 로직용
val s1_paddr_dup_dcache = io.tlb.resp.bits.paddr(1)   // DCache용
val s1_tlb_miss = io.tlb.resp.bits.miss

// DCache로 물리 주소 전달
io.dcache.s1_paddr_dup_lsu := s1_paddr_dup_lsu
io.dcache.s1_paddr_dup_dcache := s1_paddr_dup_dcache

// TLB miss 시 DCache kill
io.dcache.s1_kill := s1_tlb_miss || s1_exception
```

### 5.5 Store 경로: TLB ↔ StoreUnit ↔ DCache

```
           StoreUnit                   DTLB                    DCache (StorePipe)
           ---------                   ----                    ------------------

[S0] +----------------+         +----------------+         +----------------+
     | vaddr 생성     |-------->| TLB Lookup     |         | Tag/Meta Read  |
     | STA Pipe 요청  |         | (vaddr→paddr)  |         | (vaddr index)  |
     +----------------+         +----------------+         +----------------+
            |                          |                          |
            v                          v                          v
[S1] +----------------+         +----------------+         +----------------+
     | TLB 응답 수신  |<--------| paddr 반환     |-------->| Tag Compare    |
     | StoreQueue에   |         | miss/exception |         | Hit/Miss 판정  |
     | paddr 저장     |         +----------------+         +----------------+
     +----------------+                                           |
            |                                                     |
            | s1_paddr ------------------------------------------->|
            | s1_kill (if TLB miss) ----------------------------->|
```

### 5.6 VIPT 동작 원리

```
가상 주소:  | VPN (Virtual Page Number) | Page Offset |
            |<------- TLB 변환 -------->|<-- 동일 --->|

물리 주소:  | PPN (Physical Page Number)| Page Offset |
            |<------- Tag 비교 -------->|             |

Cache:      |        Tag        |  Index  |  Offset  |
            |<-- paddr[47:12] ->|<-vaddr->|<-vaddr-->|
```

**핵심 포인트**:
- **Index**: 가상 주소의 Page Offset 내 비트 사용 → TLB 변환 없이 접근 가능
- **Tag**: 물리 주소의 상위 비트 → TLB 변환 후 비교
- **병렬 접근**: S0에서 vaddr로 SRAM 접근 + TLB 변환 동시 시작
- **S1에서 paddr 사용**: Tag 비교는 S1에서 TLB 결과(paddr) 사용

### 5.7 TLB Miss 처리

TLB Miss 발생 시:
1. `s1_tlb_miss` 신호 활성화
2. LoadUnit에서 `s1_kill` 발행 → DCache 요청 취소
3. Load를 LoadQueue에서 대기
4. PTW (Page Table Walker)가 페이지 테이블 워크 수행
5. TLB 갱신 후 Load 재시도

```scala
// LoadUnit replay 원인
class LoadToLsqReplayIO {
  val tlb_miss = cause(LoadReplayCauses.C_TM)  // TLB miss 표시
  val tlb_id = UInt()                           // TLB 필터 ID
  val tlb_full = Bool()                         // TLB 리필 버퍼 가득 참
}
```

### 5.8 DCache에서의 paddr 사용처

DCacheLoadIO를 통해 받은 `s1_paddr`는 LoadPipe 내부에서:
```scala
// LoadPipe.scala
val s1_paddr_dup_dcache = io.lsu.s1_paddr_dup_dcache

// Tag 비교
val s1_tag_match_way_dup_dc = wayMap((w: Int) =>
  tag_resp(w) === get_tag(s1_paddr_dup_dcache) &&
  meta_resp(w).coh.isValid()
).asUInt
```

## 6. 데이터 흐름

### 6.1 Load 데이터 흐름 (TLB 포함)
```
LoadUnit                  DTLB                    DCache
--------                  ----                    ------
   |                        |
   | S0: vaddr ------------>| TLB Lookup
   | S0: vaddr ---------------------------------> Tag/Meta Read (vaddr index)
   |                        |                          |
   |                        v                          v
   |<---- S1: paddr --------|                    Tag Compare (paddr tag)
   |      S1: miss?         |                    Data Read
   |                                                   |
   | S1: paddr ---------------------------------------->|
   | S1: kill (if miss) -------------------------------->|
   |                                                   |
   |<--------------------------------- S2: hit/miss ---|
   |                                   S2: data        |
   |                                                   |
   |                                      Miss: MissQueue
   |                                            |
   |                                      L2 Acquire/Grant
   |                                            |
   +<------ LSQ Wakeup <---------------- RefillPipe
```

### 6.2 Store 데이터 흐름 (TLB 포함)
```
StoreUnit                 DTLB                    DCache
---------                 ----                    ------
   |                        |
   | S0: vaddr ------------>| TLB Lookup
   | S0: vaddr ---------------------------------> StorePipe Tag/Meta Read
   |                        |                          |
   |<---- S1: paddr --------|                    Tag Compare (paddr tag)
   |      S1: miss?         |                          |
   | S1: paddr ---------------------------------------->|
   | S1: kill (if miss) -------------------------------->|
   |                                                   |
   v                                              (hit check only)
StoreQueue                                             |
   | (commit)                                     [Prefetch if miss]
   v
SBuffer (data merge, coalesce)
   |
   v (cache line write, paddr already resolved)
MainPipe.S0 --> S1 --> S2 --> S3
   |           |       |       |
Tag Read  Tag Match  Data   Write Complete
                    Merge       |
                          Miss: MissQueue → L2 → RefillPipe
```

### 6.3 Probe/Release 데이터 흐름
```
L2 (TileLink B) --> ProbeQueue --> MainPipe
                                      |
                               Probe Process
                                      |
                    +-----------------+-----------------+
                    |                                   |
              ProbeAck only                     ProbeAck + Data
                    |                                   |
                    v                                   v
              WritebackQueue -----------------> TileLink C --> L2
```

## 7. Bank Conflict 처리

DCache는 8개의 Bank로 구성되어 있어 동일 Bank 접근 시 충돌 발생:
- LoadPipe들 간의 Bank Conflict
- LoadPipe와 MainPipe 간의 Bank Conflict

**해결 방법**:
- `bank_conflict_slow` 신호로 충돌 감지
- 충돌 시 `replay` 신호 발생, LoadUnit에서 재시도

## 8. Way Prediction Unit (WPU)

선택적으로 Way Prediction을 사용하여 데이터 읽기 지연 단축:
```scala
val dwpu = Module(new DCacheWpuWrapper(LoadPipelineWidth))
```
- S0에서 Way 예측
- S1에서 예측 결과 검증
- 예측 실패 시 replay

## 9. TileLink 인터페이스

### 9.1 Channel A (Acquire)
- MissQueue에서 L2로 AcquireBlock/AcquirePerm 발행
- 권한 상승 요청

### 9.2 Channel B (Probe)
- L2에서 DCache로 Probe 요청
- ProbeQueue에서 수신, MainPipe에서 처리

### 9.3 Channel C (Release/ProbeAck)
- WritebackQueue에서 Release/ProbeAck 발행
- Dirty 데이터 L2로 전송

### 9.4 Channel D (Grant)
- MissQueue에서 Grant/GrantData 수신
- RefillPipe로 전달하여 캐시 갱신

### 9.5 Channel E (GrantAck)
- MissQueue에서 GrantAck 발행
- Acquire 트랜잭션 완료

## 10. Prefetch 메커니즘

XiangShan에는 두 가지 종류의 Prefetch 메커니즘이 존재합니다.

### 10.1 Prefetch 종류 비교

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Prefetch 종류 비교                                │
├───────────────────────────────┬─────────────────────────────────────────┤
│  Demand-driven Prefetch       │  Hardware Prefetcher (Speculative)      │
│  (요청 기반)                   │  (패턴 기반 추측)                        │
├───────────────────────────────┼─────────────────────────────────────────┤
│  Load/Store Miss 발생 시      │  접근 패턴을 학습하여                    │
│  해당 캐시라인만 가져옴        │  미래에 필요할 데이터를 미리 가져옴      │
│                               │                                         │
│  예: StorePipe에서 miss →     │  예: A[0], A[1], A[2] 접근 패턴 감지 →  │
│      해당 line prefetch       │      A[3], A[4]를 미리 prefetch         │
├───────────────────────────────┼─────────────────────────────────────────┤
│  확실한 요청 (reactive)       │  추측성 요청 (proactive)                │
│  Miss 발생 후 동작            │  Miss 발생 전에 미리 동작               │
└───────────────────────────────┴─────────────────────────────────────────┘
```

### 10.2 Hardware Prefetcher (SMS/Stream)

XiangShan의 Hardware Prefetcher는 Load/Store 접근 패턴을 모니터링하여 미래에 필요할 데이터를 예측합니다.

```scala
// MemBlock.scala에서 Prefetcher 인스턴스
val prefetcherOpt = ... // SMS (Spatial Memory Streaming) Prefetcher
val l1_pf_req = ...     // L1 Prefetch 요청
```

**SMS (Spatial Memory Streaming) Prefetcher**:
- 접근 패턴 학습: Load/Store의 주소 패턴을 모니터링
- Stride 감지: 일정 간격으로 접근하는 패턴 (배열 순회 등)
- Spatial 감지: 인접 캐시라인 접근 패턴

**Stream Prefetcher**:
- 연속적인 메모리 접근 스트림 감지
- 다음에 접근할 캐시라인 예측

### 10.3 Load Prefetch 경로 (L1 Hardware Prefetch)

Load에 대한 Hardware Prefetch는 **LoadUnit을 통해 LoadPipe로** 전달됩니다:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Load Prefetch 전체 경로                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────┐     prefetch_train      ┌────────────────┐                 │
│  │   LoadUnit  │ ───────────────────────>│ Stream/Stride  │                 │
│  │   (실제 Load)│     (학습 데이터)       │   Prefetcher   │                 │
│  └──────┬──────┘                         └───────┬────────┘                 │
│         │                                        │                          │
│         │                                        │ l1_pf_req                │
│         │                                        │ (L1PrefetchReq)          │
│         │                                        ▼                          │
│         │                                ┌───────────────┐                  │
│         │     prefetch_req               │   LoadUnit    │ ◄── prefetch를   │
│         │ <──────────────────────────────│   (S0 stage)  │     Load처럼 처리│
│         │                                └───────┬───────┘                  │
│         │                                        │                          │
│         ▼                                        ▼                          │
│  ┌─────────────┐                         ┌─────────────┐                    │
│  │  LoadPipe   │                         │  LoadPipe   │                    │
│  │ (실제 Load) │                         │ (Prefetch)  │                    │
│  └─────────────┘                         └─────────────┘                    │
│                                                                             │
│  ※ Prefetch도 일반 Load와 동일한 경로로 DCache에 접근                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**LoadUnit S0 소스 우선순위** ([LoadUnit.scala:207-214](src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala#L207)):
```scala
// src1: fast load replay
// src2: load replayed by LSQ
// src3: hardware prefetch (HIGH confidence) ◄── Prefetch 높은 우선순위
// src4: int read / software prefetch
// src5: vec read
// src6: load try pointchasing
// src7: hardware prefetch (LOW confidence)  ◄── Prefetch 낮은 우선순위

val s0_high_conf_prf_valid = io.prefetch_req.valid && io.prefetch_req.bits.confidence > 0.U
val s0_low_conf_prf_valid  = io.prefetch_req.valid && io.prefetch_req.bits.confidence === 0.U
```

**Prefetch 훈련 데이터 피드백**:
```scala
// LoadUnit.scala - prefetcher 학습용 데이터 제공
io.prefetch_train.valid := RegNext(s2_valid && !s2_actually_mmio && !s2_in.tlbMiss)
io.prefetch_train.bits.miss := RegNext(io.dcache.resp.bits.miss)
io.prefetch_train.bits.meta_prefetch := RegNext(io.dcache.resp.bits.meta_prefetch)
```

### 10.4 Hardware Prefetcher와 DTLB

Hardware Prefetcher(SMS)는 L2 prefetch 시 전용 DTLB(`dtlb_prefetch`)를 사용합니다:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Prefetch DTLB 사용 구분                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [L1 Prefetch - Stream/Stride]          [L2 Prefetch - SMS]                 │
│                                                                             │
│  LoadUnit.prefetch_req 경로 사용         dtlb_prefetch 경로 사용             │
│  → dtlb_ld 공유 (LoadPipe 통해)          → 전용 TLB 사용                     │
│  → 이미 paddr 변환 완료된 상태            → vaddr→paddr 변환 필요             │
│                                                                             │
│       ┌──────────────┐                       ┌──────────────┐               │
│       │ L1 Prefetcher│                       │ SMS (L2 PF)  │               │
│       │Stream/Stride │                       │              │               │
│       └──────┬───────┘                       └──────┬───────┘               │
│              │ paddr (이미 변환됨)                   │ vaddr                 │
│              ▼                                      ▼                       │
│       ┌──────────────┐                       ┌──────────────┐               │
│       │   LoadUnit   │                       │dtlb_prefetch │               │
│       │  (LoadPipe)  │                       │  (전용 TLB)  │               │
│       └──────────────┘                       └──────┬───────┘               │
│                                                     │ paddr                 │
│                                                     ▼                       │
│                                              ┌──────────────┐               │
│                                              │  L2 Cache    │               │
│                                              │  Prefetch    │               │
│                                              └──────────────┘               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**왜 SMS Prefetcher만 별도 DTLB가 필요한가?**
- **L1 Prefetch (Stream/Stride)**: LoadUnit을 통해 LoadPipe로 진입 → `dtlb_ld` 공유
- **L2 Prefetch (SMS)**: 직접 L2로 prefetch 요청 → 별도 주소 변환 필요 → `dtlb_prefetch`

```scala
// MemBlock.scala
val PrefetcherDTLBPortIndex = exuParameters.LduCnt + exuParameters.StuCnt + 1

prefetcherOpt match {
  case Some(pf) => dtlb_reqs(PrefetcherDTLBPortIndex) <> pf.io.tlb_req
  // SMS Prefetcher가 L2 prefetch 시 사용
}
```

### 10.5 StorePipe의 Demand-driven Prefetch

StorePipe에서 발생하는 Prefetch는 Hardware Prefetcher가 아닌 **요청 기반 Prefetch**입니다:

```scala
// StorePipe.scala
if(EnableStorePrefetchAtIssue) {
  // Store miss 시 해당 캐시라인 prefetch write 요청
  io.miss_req.valid := s2_valid && !s2_hit
} else {
  // 명시적 prefetch 명령어만 처리
  io.miss_req.valid := s2_valid && !s2_hit && s2_is_prefetch
}
```

**동작 방식**:
1. Store 주소가 DCache에서 miss
2. 해당 캐시라인을 MissQueue로 prefetch 요청
3. 실제 Store 데이터 쓰기는 SBuffer → MainPipe 경로로 나중에 수행
4. "지금 miss니까 데이터 가져와둬" (demand-driven)

### 10.6 Prefetch vs LSQ 역할 비교

| 구성요소 | 역할 |
|----------|------|
| **LoadQueue** | 실행 중인 Load 추적, Memory Ordering 보장, Replay 관리 |
| **StoreQueue** | Commit 전 Store 보관, Store-to-Load Forwarding |
| **SBuffer** | Commit된 Store를 캐시라인 단위로 합침 (Coalescing) |
| **Hardware Prefetcher** | 미래 접근 패턴 예측하여 미리 데이터 가져옴 |

```
LSQ의 역할:        "지금 실행 중인 메모리 연산 관리"
Prefetcher의 역할: "앞으로 필요할 데이터 미리 가져오기"
```

### 10.7 Prefetch Source 구분

DCache에서는 prefetch 요청의 출처를 구분합니다:

```scala
// DCacheBundle.scala
object DCACHE_PREFETCH_SOURCE {
  def value = 0.U  // 기본 prefetch 소스
}

object L1_HW_PREFETCH_NULL extends L1PrefetchSource  // HW prefetch 없음
object L1_HW_PREFETCH_STRIDE extends L1PrefetchSource  // Stride prefetch
object L1_HW_PREFETCH_STREAM extends L1PrefetchSource  // Stream prefetch
```

이를 통해 prefetch 효과 분석 및 최적화가 가능합니다.
