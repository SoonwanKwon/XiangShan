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

### 3.5 MissQueue와 MSHR (Miss Status Holding Registers)

#### 3.5.1 기본 설정
- **MSHR 엔트리 수**: nMissEntries = 16개 (기본값, 2의 거듭제곱이어야 함)
- **Release 엔트리 수**: nReleaseEntries = 18개 (MSHR보다 커야 함)
- **Prefetch 예약 슬롯**: nMaxPrefetchEntry = 6개 (마지막 6개 슬롯)
- **역할**: Cache Miss 처리, L2 Acquire 발행, Refill 관리

#### 3.5.2 MSHR 엔트리 구조

각 MSHR 엔트리(`MissEntry`)는 다음 정보를 포함합니다:

```scala
// MissQueue.scala의 MissEntry 핵심 필드
class MissEntry {
  // 요청 정보
  val req: MissReqWoStoreData = Reg(...)
    - source: 요청 출처 (LOAD=0, STORE=1, AMO=2, PREFETCH≥3)
    - addr: 물리 주소 (블록 정렬)
    - vaddr: 가상 주소
    - way_en: Way 활성화 비트 (one-hot, 8비트)
    - cmd: 메모리 명령
    - req_coh: 요청 시점 캐시 일관성 상태
    - replace_coh: 교체될 라인의 일관성 상태
    - replace_tag: 교체될 라인의 태그
    - full_overwrite: 전체 블록 덮어쓰기 여부

  // Store 관련 데이터
  val req_store_mask: UInt(64.W)              // Store 마스크 (바이트 단위)
  val refill_and_store_data: Vec[8][64bits]   // 머지된 Store+Refill 데이터

  // Refill 데이터
  val refill_data_raw: Vec[beatRows]          // L2에서 받은 원본 데이터

  // 상태 및 메타데이터
  val grant_param: TileLink 권한 정보
  val error: ECC/L2 에러 플래그
  val prefetch: Prefetch 표시 비트
  val access: 실제 접근 여부 (non-prefetch)
  val isDirty: Dirty 플래그
}
```

#### 3.5.3 MSHR 상태 머신

각 엔트리는 여러 상태 플래그로 miss 처리 파이프라인을 추적합니다:

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                           MSHR Entry State Machine                                       │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  ┌──────────┐     ┌───────────────┐     ┌───────────────┐     ┌──────────────┐          │
│  │  IDLE    │────>│  s_acquire    │────>│ w_grantfirst  │────>│ w_grantlast  │          │
│  │(req_valid│     │(Acquire 발행) │     │(첫 Grant 수신)│     │(마지막 Grant)│          │
│  │ = false) │     └───────────────┘     └───────────────┘     └──────┬───────┘          │
│  └──────────┘                                                        │                  │
│       ▲                                                              │                  │
│       │                                                              ▼                  │
│       │                           ┌──────────────────────────────────────────┐          │
│       │                           │  Replace 필요 시: s_replace_req          │          │
│       │                           │  MainPipe로 victim line writeback 요청   │          │
│       │                           └──────────────────────────────────────────┘          │
│       │                                              │                                  │
│       │                                              ▼                                  │
│       │                                    ┌──────────────────┐                         │
│       │                                    │  s_refill        │                         │
│       │                                    │  RefillPipe로    │                         │
│       │                                    │  데이터 기록 요청 │                         │
│       │                                    └────────┬─────────┘                         │
│       │                                             │                                   │
│       │                                             ▼                                   │
│       │       ┌───────────────┐           ┌──────────────────┐                          │
│       │       │  s_grantack   │<──────────│ w_refill_resp    │                          │
│       │       │  GrantAck 발행│           │ (Refill 완료)    │                          │
│       │       └───────────────┘           └──────────────────┘                          │
│       │               │                                                                 │
│       │               ▼                                                                 │
│       │       ┌───────────────────────────────────────────────┐                         │
│       │       │  AMO인 경우: s_mainpipe_req                    │                         │
│       │       │  MainPipe에서 atomic 연산 수행                  │                         │
│       │       └───────────────────────────────────────────────┘                         │
│       │                               │                                                 │
│       │                               ▼                                                 │
│       │                      ┌─────────────────┐                                        │
│       └──────────────────────│  release_entry  │                                        │
│                              │  (엔트리 해제)   │                                        │
│                              │  s_grantack &&  │                                        │
│                              │  w_refill_resp  │                                        │
│                              │  && w_mainpipe  │                                        │
│                              └─────────────────┘                                        │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

**상태 플래그 설명**:

| 플래그 | 의미 |
|--------|------|
| `s_acquire` | Acquire 요청 발행 가능 (false=발행 전) |
| `w_grantfirst` | 첫 Grant beat 수신 완료 |
| `w_grantlast` | 마지막 Grant beat 수신 (refill 데이터 완료) |
| `s_replace_req` | Replace 요청 발행 가능 |
| `w_replace_resp` | Replace 완료 응답 수신 |
| `s_refill` | Refill 요청 발행 가능 |
| `w_refill_resp` | Refill 완료 응답 수신 |
| `s_mainpipe_req` | MainPipe 요청 발행 가능 (AMO용) |
| `w_mainpipe_resp` | MainPipe 처리 완료 |
| `s_grantack` | GrantAck 발행 가능 |

#### 3.5.4 Miss 요청 처리: Alloc vs Merge vs Reject

Miss 요청은 2 사이클에 걸쳐 처리됩니다:

**Cycle S0: 요청 중재**
```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                         Miss Request Arbiter (S0)                                        │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  Sources:                                                                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                                      │
│  │ MainPipe    │  │ LoadPipe[0] │  │ LoadPipe[1] │                                      │
│  │ (Store miss)│  │ (Load miss) │  │ (Load miss) │                                      │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                                      │
│         │                │                │                                             │
│         └────────────────┼────────────────┘                                             │
│                          ▼                                                              │
│                   ┌──────────────┐                                                      │
│                   │   Arbiter    │                                                      │
│                   └──────┬───────┘                                                      │
│                          │                                                              │
│         ┌────────────────┼────────────────┐                                             │
│         ▼                ▼                ▼                                             │
│   ┌──────────┐    ┌──────────┐    ┌───────────┐                                         │
│   │  ALLOC   │    │  MERGE   │    │  REJECT   │                                         │
│   │새 엔트리  │    │기존 엔트리│    │요청 거부   │                                         │
│   │ 할당     │    │에 병합    │    │(replay)   │                                         │
│   └──────────┘    └──────────┘    └───────────┘                                         │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

**Merge 조건**:
- **before_req_sent_can_merge**: Acquire 발행 전, 동일 블록 주소, 호환 가능한 소스
- **before_data_refill_can_merge**: Refill 완료 전, 동일 블록 주소, 호환 가능한 소스
- Alias 비트 일치 필요

**Reject 조건**:
- 블록 주소 불일치이나 set/way 충돌
- Acquire 이미 발행 후 저장소 수정 필요
- Refill 진행 중 비호환 요청 타입

**Cycle S1: MSHR 업데이트**
- Pipeline register에서 요청 읽기
- 선택된 엔트리에 요청 정보 기록
- Merge 시: Store 데이터 및 마스크 병합

### 3.6 Miss 발생 시 전체 처리 흐름

#### 3.6.1 Load Miss 처리 흐름

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              Load Miss Handling Flow                                     │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  [Phase 1: Miss Detection - LoadPipe S2]                                                │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐    │
│  │ LoadPipe S2:                                                                    │    │
│  │   - Tag 비교 결과: Miss!                                                        │    │
│  │   - miss_req 생성: addr, vaddr, way_en, cmd, source=LOAD                        │    │
│  │   - io.miss_req.valid := s2_valid && !s2_hit                                    │    │
│  └─────────────────────────────────────────────────────────────────────────────────┘    │
│                                              │                                          │
│                                              ▼                                          │
│  [Phase 2: MSHR Allocation - MissQueue]                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐    │
│  │ MissQueue Arbiter (Cycle S0):                                                   │    │
│  │   - primary_ready 확인: 빈 엔트리 있는가?                                         │    │
│  │   - secondary_ready 확인: 병합 가능한 엔트리 있는가?                               │    │
│  │   - 결정: ALLOC / MERGE / REJECT                                                │    │
│  │                                                                                 │    │
│  │ MSHR Entry Update (Cycle S1):                                                   │    │
│  │   - Pipeline reg에서 요청 읽기                                                   │    │
│  │   - req_valid := true                                                           │    │
│  │   - 모든 상태 플래그 초기화                                                       │    │
│  └─────────────────────────────────────────────────────────────────────────────────┘    │
│                                              │                                          │
│                                              ▼                                          │
│  [Phase 3: Acquire - TileLink Channel A]                                                │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐    │
│  │ MSHR Entry:                                                                     │    │
│  │   when(!s_acquire) {                                                            │    │
│  │     io.mem_acquire.valid := true                                                │    │
│  │     io.mem_acquire.bits := AcquireBlock(addr, size=64, grow=NtoB/NtoT)          │    │
│  │   }                                                                             │    │
│  │                                                                                 │    │
│  │ 발행 후: s_acquire := true                                                      │    │
│  └─────────────────────────────────────────────────────────────────────────────────┘    │
│                                              │                                          │
│                                              ▼ TileLink A Channel                       │
│                                        ┌───────────┐                                    │
│                                        │  L2 Cache │                                    │
│                                        └─────┬─────┘                                    │
│                                              │ TileLink D Channel (Grant + Data)        │
│                                              ▼                                          │
│  [Phase 4: Grant Reception]                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐    │
│  │ MSHR Entry:                                                                     │    │
│  │   for each grant beat:                                                          │    │
│  │     refill_data_raw(beat_idx) := grant_data                                     │    │
│  │     refill_count++                                                              │    │
│  │                                                                                 │    │
│  │   첫 beat: w_grantfirst := true                                                 │    │
│  │   마지막 beat: w_grantlast := true                                              │    │
│  │                                                                                 │    │
│  │ Load Queue Wakeup (Early):                                                      │    │
│  │   io.refill_to_ldq.valid := beat 수신 시 (마지막 beat 전에도)                     │    │
│  │   → 대기 중인 Load가 Refill 데이터로 바로 완료 가능                               │    │
│  └─────────────────────────────────────────────────────────────────────────────────┘    │
│                                              │                                          │
│                                              ▼                                          │
│  [Phase 5: Replace (if victim exists)]                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐    │
│  │ if (miss_req.hit == false && replace_coh.valid) {                               │    │
│  │   // Victim line이 있으면 MainPipe로 writeback 요청                              │    │
│  │   io.main_pipe_req.valid := !s_replace_req                                      │    │
│  │   io.main_pipe_req.bits.replace := true                                         │    │
│  │   io.main_pipe_req.bits.way_en := replace_way_en                                │    │
│  │                                                                                 │    │
│  │   MainPipe가 dirty line을 WritebackQueue로 전송                                 │    │
│  │   완료 후: w_replace_resp := true                                               │    │
│  │ }                                                                               │    │
│  └─────────────────────────────────────────────────────────────────────────────────┘    │
│                                              │                                          │
│                                              ▼                                          │
│  [Phase 6: Refill - RefillPipe]                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐    │
│  │ when(w_replace_resp && w_grantlast) {                                           │    │
│  │   io.refill_pipe_req.valid := !s_refill                                         │    │
│  │   io.refill_pipe_req.bits := RefillPipeReq(                                     │    │
│  │     wmask = 8'b11111111,     // 8개 Bank 모두                                   │    │
│  │     data  = refill_data,     // 64 bytes                                        │    │
│  │     way_en = way_en,         // 선택된 Way                                      │    │
│  │     meta  = new_coh_state    // 새 coherency 상태                               │    │
│  │   )                                                                             │    │
│  │ }                                                                               │    │
│  │                                                                                 │    │
│  │ RefillPipe가 1 cycle에:                                                         │    │
│  │   - DataArray (8 banks) 쓰기                                                    │    │
│  │   - TagArray 쓰기                                                               │    │
│  │   - MetaArray 쓰기 (coherency 상태)                                             │    │
│  │                                                                                 │    │
│  │ 완료 후: w_refill_resp := true                                                  │    │
│  └─────────────────────────────────────────────────────────────────────────────────┘    │
│                                              │                                          │
│                                              ▼                                          │
│  [Phase 7: GrantAck - TileLink Channel E]                                               │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐    │
│  │ when(w_grantfirst && !s_grantack) {                                             │    │
│  │   io.mem_finish.valid := true                                                   │    │
│  │   io.mem_finish.bits := GrantAck                                                │    │
│  │ }                                                                               │    │
│  │                                                                                 │    │
│  │ 발행 후: s_grantack := true                                                     │    │
│  └─────────────────────────────────────────────────────────────────────────────────┘    │
│                                              │                                          │
│                                              ▼                                          │
│  [Phase 8: Entry Release]                                                               │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐    │
│  │ release_entry := s_grantack && w_refill_resp && w_mainpipe_resp                 │    │
│  │ when(release_entry) { req_valid := false }                                      │    │
│  │                                                                                 │    │
│  │ → MSHR 엔트리가 다시 사용 가능해짐                                                │    │
│  └─────────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

#### 3.6.2 Store Miss 처리 흐름

Store Miss는 Load Miss와 약간 다른 경로를 따릅니다:

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              Store Miss Handling Flow                                    │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  [1] StorePipe: Miss 감지 (실제 쓰기 안 함, Prefetch 요청만)                             │
│      └─→ Miss 시 MissQueue로 prefetch write 요청                                        │
│                                                                                         │
│  [2] StoreQueue: Store 데이터 보관                                                       │
│      └─→ Commit 후 SBuffer로 전송                                                       │
│                                                                                         │
│  [3] SBuffer: 데이터 합침 (Coalescing)                                                  │
│      └─→ 캐시라인 단위로 MainPipe에 쓰기 요청                                            │
│                                                                                         │
│  [4] MainPipe S0-S3:                                                                    │
│      ┌─────────────────────────────────────────────────────────────────────────────┐    │
│      │ S0: Meta/Tag 읽기                                                           │    │
│      │ S1: Tag 비교, Miss 판정                                                     │    │
│      │ S2: Miss 시 → MissQueue에 miss 요청 (source=STORE, full_overwrite 여부)     │    │
│      │ S3: Hit 시 → Data 쓰기, Meta 업데이트                                       │    │
│      └─────────────────────────────────────────────────────────────────────────────┘    │
│                                              │                                          │
│                                              ▼ (Miss인 경우)                            │
│  [5] MSHR Entry: Store 데이터 머지                                                      │
│      ┌─────────────────────────────────────────────────────────────────────────────┐    │
│      │ // L2에서 refill 데이터 수신 시                                              │    │
│      │ for (i <- 0 until blockRows) {                                              │    │
│      │   refill_and_store_data(i) := mergePutData(                                 │    │
│      │     grant_row,         // L2에서 온 데이터                                   │    │
│      │     store_data(i),     // Store 데이터                                      │    │
│      │     store_mask(i)      // Store 마스크 (바이트 단위)                         │    │
│      │   )                                                                         │    │
│      │ }                                                                           │    │
│      │                                                                             │    │
│      │ // Merge 로직: 마스크된 바이트만 Store 데이터 사용                            │    │
│      │ def mergePutData(old, new, mask) = {                                        │    │
│      │   (0 until rowBytes).map { i =>                                             │    │
│      │     Mux(mask(i), new(i*8+7, i*8), old(i*8+7, i*8))                           │    │
│      │   }.reverse.reduce(Cat(_, _))                                               │    │
│      │ }                                                                           │    │
│      └─────────────────────────────────────────────────────────────────────────────┘    │
│                                              │                                          │
│                                              ▼                                          │
│  [6] RefillPipe: 머지된 데이터로 캐시 업데이트                                           │
│      └─→ 64 bytes 전체를 1 cycle에 기록                                                 │
│                                                                                         │
│  ※ full_overwrite=true인 경우:                                                          │
│     - Acquire 대신 AcquirePerm만 발행 (데이터 불필요)                                    │
│     - L2에서 Grant (no data) 수신                                                       │
│     - Store 데이터만으로 캐시 업데이트                                                   │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

#### 3.6.3 Refill 데이터 경로

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              Refill Data Path Detail                                     │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  L2 Cache (TileLink D Channel)                                                          │
│      │                                                                                  │
│      │ GrantData: 256 bits per beat × N beats = 512 bits (64 bytes) total              │
│      │ (beatBytes = 32, blockBytes = 64 → 2 beats)                                     │
│      ▼                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐    │
│  │                            MSHR Entry                                           │    │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐    │    │
│  │  │  refill_data_raw: Array[beatRows] of GrantData                          │    │    │
│  │  │                                                                         │    │    │
│  │  │  Beat 0: [Row0, Row1, Row2, Row3]  ← 첫 256 bits (4 × 64 bits)          │    │    │
│  │  │  Beat 1: [Row4, Row5, Row6, Row7]  ← 다음 256 bits (4 × 64 bits)        │    │    │
│  │  └─────────────────────────────────────────────────────────────────────────┘    │    │
│  │                                           │                                     │    │
│  │                                           ▼ Store Miss인 경우                   │    │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐    │    │
│  │  │  mergePutData():                                                        │    │    │
│  │  │                                                                         │    │    │
│  │  │  Refill:  [A A A A A A A A]  ← L2에서 온 데이터                          │    │    │
│  │  │  Store:   [X X B B X X X X]  ← Store 데이터                              │    │    │
│  │  │  Mask:    [0 0 1 1 0 0 0 0]  ← Store 마스크                              │    │    │
│  │  │  Result:  [A A B B A A A A]  ← 머지된 최종 데이터                         │    │    │
│  │  └─────────────────────────────────────────────────────────────────────────┘    │    │
│  │                                           │                                     │    │
│  │  refill_and_store_data: Vec[8] of UInt(64.W) ← 최종 refill 데이터              │    │
│  └───────────────────────────────────────────┼─────────────────────────────────────┘    │
│                                              │                                          │
│                                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐    │
│  │                           RefillPipe                                            │    │
│  │                                                                                 │    │
│  │  RefillPipeReq:                                                                 │    │
│  │    wmask = 8'b11111111                                                          │    │
│  │    data[0] = refill_and_store_data[0]  → Bank 0                                 │    │
│  │    data[1] = refill_and_store_data[1]  → Bank 1                                 │    │
│  │    ...                                                                          │    │
│  │    data[7] = refill_and_store_data[7]  → Bank 7                                 │    │
│  │                                                                                 │    │
│  │  동시 업데이트:                                                                  │    │
│  │    - DataArray.write(all 8 banks)                                               │    │
│  │    - TagArray.write(tag)                                                        │    │
│  │    - MetaArray.write(coh_state)                                                 │    │
│  │    - ErrorArray.write(error_flag)                                               │    │
│  │    - PrefetchArray.write(prefetch_flag)                                         │    │
│  │    - AccessArray.write(access_flag)                                             │    │
│  └─────────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 3.7 ProbeQueue
- **수량**: nProbeEntries 개
- **역할**: L2의 Probe 요청 큐잉, MainPipe로 전달

### 3.8 WritebackQueue
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

## 7. Storage Array 구조와 Parallel Access

### 7.1 Tag Array vs Data Array: 핵심 차이점

**중요**: Tag Array와 Data Array는 완전히 다른 구조입니다!

- **Tag Array**: **Duplicated** (복제) - 각 파이프라인마다 전체 복사본 보유
- **Data Array**: **Banked** (분할) - 8개 Bank로 물리적 분할

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                     Tag Array: DUPLICATED (복제 구조)                                    │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   DuplicatedTagArray(readPorts = LoadPipelineWidth + 1 + StorePipelineWidth)            │
│   = 2 + 1 + 2 = 5개 읽기 포트 (Store Prefetch 활성화 시)                                  │
│   = 2 + 1 = 3개 읽기 포트 (기본 구성)                                                     │
│                                                                                         │
│   ┌────────────────┐  ┌────────────────┐  ┌────────────────┐                            │
│   │ Tag Array      │  │ Tag Array      │  │ Tag Array      │                            │
│   │ Copy 0         │  │ Copy 1         │  │ Copy 2         │                            │
│   │ (LoadPipe[0])  │  │ (LoadPipe[1])  │  │ (MainPipe)     │                            │
│   ├────────────────┤  ├────────────────┤  ├────────────────┤                            │
│   │ Set 0: 8 tags  │  │ Set 0: 8 tags  │  │ Set 0: 8 tags  │                            │
│   │ Set 1: 8 tags  │  │ Set 1: 8 tags  │  │ Set 1: 8 tags  │                            │
│   │ ...            │  │ ...            │  │ ...            │                            │
│   │ Set 255: 8 tags│  │ Set 255: 8 tags│  │ Set 255: 8 tags│                            │
│   └────────────────┘  └────────────────┘  └────────────────┘                            │
│         │                   │                   │                                       │
│   ┌─────┴─────┐       ┌─────┴─────┐       ┌─────┴─────┐                                 │
│   │ 1 read =  │       │ 1 read =  │       │ 1 read =  │                                 │
│   │ 8 way tags│       │ 8 way tags│       │ 8 way tags│                                 │
│   └───────────┘       └───────────┘       └───────────┘                                 │
│                                                                                         │
│   ※ Write는 모든 Copy에 Broadcast (동기화 유지)                                          │
│   ※ 각 파이프라인이 독립적으로 Tag 읽기 가능 → 충돌 없음!                                  │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

**Tag Array SRAM 구조** ([TagArray.scala:62](src/main/scala/xiangshan/cache/dcache/meta/TagArray.scala#L62)):
```scala
val tag_array = Module(new SRAMTemplate(
  UInt(tagBits.W),
  set = nSets,    // 256 sets
  way = nWays,    // 8 ways
  singlePort = true
))
// SET으로만 인덱싱 → 한 번 읽으면 해당 Set의 8개 Way 태그 전부 반환
```

**DuplicatedTagArray 구조** ([TagArray.scala:117](src/main/scala/xiangshan/cache/dcache/meta/TagArray.scala#L117)):
```scala
val array = Seq.fill(readPorts) { Module(new TagArray) }
// readPorts개의 완전한 Tag Array 복사본 생성
```

### 7.2 왜 Tag Array는 Bank가 아닌 Duplicate인가?

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                         Load 동작 시 Tag와 Data 접근 비교                                 │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   문제: 4 byte Load 시, 데이터가 어느 Way에 있는지 미리 알 수 없음                         │
│   해결: Tag는 모든 Way를 한번에 읽고, Data는 필요한 Bank만 읽음                            │
│                                                                                         │
│   ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│   │ S0 Stage: Tag Array 읽기 (SET 인덱스만 사용)                                     │   │
│   │                                                                                 │   │
│   │   LoadPipe[0]의 전용 Tag Array Copy에서:                                         │   │
│   │   read(set_idx) → [Tag_Way0, Tag_Way1, ..., Tag_Way7] 전부 반환                  │   │
│   │                                                                                 │   │
│   │   ※ Bank 개념 없음! SET 하나 읽으면 8개 Way 태그 모두 획득                        │   │
│   └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                      │                                                  │
│                                      ▼                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│   │ S1 Stage: Tag 비교 + Data Array 읽기                                             │   │
│   │                                                                                 │   │
│   │   1) paddr의 Tag와 8개 Way Tag 비교 → Hit Way 결정 (예: Way 3 hit)               │   │
│   │                                                                                 │   │
│   │   2) Data Array에서 해당 Bank만 읽기:                                            │   │
│   │      addr[5:3] = Bank Index (예: Bank 2)                                        │   │
│   │      read(set_idx, bank=2) → [Data_Way0, Data_Way1, ..., Data_Way7]             │   │
│   │                                                                                 │   │
│   │   ※ Data Array는 Bank별로 분리되어 있어 다른 Bank 접근과 병렬 가능                 │   │
│   └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                      │                                                  │
│                                      ▼                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│   │ S2 Stage: Hit Way 데이터 선택                                                    │   │
│   │                                                                                 │   │
│   │   Way 3가 hit → Data_Way3 선택하여 반환                                          │   │
│   └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

**핵심 포인트**:

| 구분 | Tag Array | Data Array |
|------|-----------|------------|
| **구조** | Duplicated (복제) | Banked (분할) |
| **인덱싱** | SET만 사용 | SET + Bank |
| **읽기 결과** | 8개 Way 태그 전부 | 해당 Bank의 8개 Way 데이터 |
| **병렬화 방법** | 파이프라인별 복사본 | Bank별 분리 |
| **충돌** | 없음 (각자 복사본) | Bank Conflict 가능 |

### 7.3 Data Array Bank 구성

DCache의 데이터 어레이는 8개의 독립적인 Bank로 구성되어 병렬 접근을 지원합니다.

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              Data Array Bank Layout                                      │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   Cache Block (64 bytes = 512 bits):                                                    │
│   ┌────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┐             │
│   │ Bank 0 │ Bank 1 │ Bank 2 │ Bank 3 │ Bank 4 │ Bank 5 │ Bank 6 │ Bank 7 │             │
│   │ 8 bytes│ 8 bytes│ 8 bytes│ 8 bytes│ 8 bytes│ 8 bytes│ 8 bytes│ 8 bytes│             │
│   │ 64 bits│ 64 bits│ 64 bits│ 64 bits│ 64 bits│ 64 bits│ 64 bits│ 64 bits│             │
│   └────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┘             │
│                                                                                         │
│   Per Bank Structure (for each of 8 banks):                                             │
│   ┌───────────────────────────────────────────────────┐                                 │
│   │                    Bank N                          │                                 │
│   ├─────────┬─────────┬─────────┬─────────────────────┤                                 │
│   │  Way 0  │  Way 1  │  ...    │  Way 7              │                                 │
│   ├─────────┼─────────┼─────────┼─────────────────────┤                                 │
│   │  Set 0  │  Set 0  │  ...    │  Set 0              │                                 │
│   │  Set 1  │  Set 1  │  ...    │  Set 1              │                                 │
│   │  ...    │  ...    │  ...    │  ...                │                                 │
│   │ Set 255 │ Set 255 │  ...    │ Set 255             │                                 │
│   └─────────┴─────────┴─────────┴─────────────────────┘                                 │
│                                                                                         │
│   blockBytes = 64, rowBits = 64, blockRows = 8 (= DCacheBanks)                          │
│   각 Bank가 블록 내 1개 row(8 bytes)를 담당                                               │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 7.4 Bank 인덱싱

```
           Virtual/Physical Address 구조
┌────────────────────────────────────────────────────────────────────┐
│  Tag (상위)  │    Set Index    │  Bank Index │  Byte Offset       │
│              │    [14:6] (9b)  │  [5:3] (3b) │  [2:0] (3b)        │
└────────────────────────────────────────────────────────────────────┘
                        │                │            │
                        ▼                ▼            ▼
              어느 Set인가?      어느 Bank인가?   Bank 내 위치

DCacheBankOffset = log2(8) = 3      // Bank 인덱스 시작 오프셋
DCacheSetOffset = 3 + 3 = 6         // Set 인덱스 시작 오프셋
DCacheAboveIndexOffset = 6 + 9 = 15 // Set 인덱스 끝 오프셋

// 주소에서 Bank/Set 추출
def addr_to_dcache_bank(addr: UInt) = addr(DCacheSetOffset-1, DCacheBankOffset)  // [5:3]
def addr_to_dcache_set(addr: UInt) = addr(DCacheAboveIndexOffset-1, DCacheSetOffset) // [14:6]
```

### 7.5 Parallel Read 동작 (2개 Load Pipeline)

2개의 LoadPipe가 동시에 데이터 어레이에 접근할 수 있습니다:

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                          Parallel Read Access                                            │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   LoadPipe[0]                              LoadPipe[1]                                  │
│      │                                        │                                         │
│      │ addr_a = 0x1000                        │ addr_b = 0x2008                         │
│      │ bank_a = (0x1000 >> 3) & 7 = 0         │ bank_b = (0x2008 >> 3) & 7 = 1          │
│      ▼                                        ▼                                         │
│   ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐               │
│   │Bank 0│ │Bank 1│ │Bank 2│ │Bank 3│ │Bank 4│ │Bank 5│ │Bank 6│ │Bank 7│               │
│   │ READ │ │ READ │ │      │ │      │ │      │ │      │ │      │ │      │               │
│   │ ◄──  │ │  ──► │ │      │ │      │ │      │ │      │ │      │ │      │               │
│   └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └──────┘               │
│      │                │                                                                 │
│      ▼                ▼                                                                 │
│   Data_A           Data_B        ← 서로 다른 Bank이므로 병렬 접근 가능!                    │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

**병렬 접근이 가능한 경우**:
- 두 Load가 서로 다른 Bank를 접근할 때
- 두 Load가 같은 Bank를 접근하더라도 Set이나 Way가 다를 때 (일부 조건)

### 7.6 Bank Conflict 종류

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              Bank Conflict Types                                         │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  1. RR (Read-Read) Bank Conflict:                                                       │
│     ┌───────────────────────────────────────────────────────────────────────────────┐   │
│     │ LoadPipe[0]  ──┐                                                              │   │
│     │                 ├──► 같은 Bank, 같은 div_addr 접근 → CONFLICT!                │   │
│     │ LoadPipe[1]  ──┘                                                              │   │
│     │                                                                               │   │
│     │ 조건: bank_a == bank_b && set_a == set_b                                      │   │
│     │ 결과: bank_conflict_slow 신호 발생 → LoadPipe[1] replay                       │   │
│     └───────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
│  2. RRL (Read-Read-Line) Bank Conflict:                                                 │
│     ┌───────────────────────────────────────────────────────────────────────────────┐   │
│     │ LoadPipe    ──┐                                                               │   │
│     │               ├──► MainPipe readline과 같은 bank/div_addr 접근 → CONFLICT!    │   │
│     │ MainPipe    ──┘    (MainPipe가 전체 line 읽기 수행 시)                         │   │
│     └───────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
│  3. WR (Write-Refill) Bank Conflict:                                                    │
│     ┌───────────────────────────────────────────────────────────────────────────────┐   │
│     │ LoadPipe    ──┐                                                               │   │
│     │               ├──► RefillPipe 쓰기와 동시 읽기 시도 → CONFLICT!               │   │
│     │ RefillPipe  ──┘                                                               │   │
│     │                                                                               │   │
│     │ write_valid_reg로 쓰기 진행 중 추적                                            │   │
│     │ wmask가 겹치는 Bank 접근 시 충돌                                               │   │
│     └───────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 7.7 Bank Conflict 해결

**충돌 감지 및 처리**:
```scala
// BankedDataArray.scala 핵심 로직
val rr_bank_conflict = (bank_addrs(0) === bank_addrs(1)) &&
                       (set_addrs(0) === set_addrs(1)) &&
                       io.read(0).valid && io.read(1).valid

// LoadPipe로 conflict 신호 전달
io.read(1).bank_conflict_slow := rr_bank_conflict

// LoadUnit에서 replay 처리
when(io.dcache.s2_bank_conflict) {
  // LoadQueue에 replay 원인 기록
  // 다음 사이클에 재시도
}
```

**충돌 시 동작**:
1. `bank_conflict_slow` 신호로 충돌 감지
2. 충돌된 요청에 `replay` 신호 발생
3. LoadUnit에서 해당 Load를 LoadQueue로 반환
4. LoadQueue가 적절한 시점에 재발행

### 7.8 Refill 쓰기 동작

RefillPipe는 한 사이클에 모든 8개 Bank를 동시에 쓸 수 있습니다:

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              Refill Write (1 cycle)                                      │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   RefillPipeReq:                                                                        │
│   ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│   │ wmask: 8'b11111111 (8개 Bank 모두 쓰기)                                          │   │
│   │ data:  Vec[8] of UInt(64.W) - 각 Bank별 64비트 데이터                            │   │
│   │ way_en: 쓸 Way 선택 (one-hot)                                                   │   │
│   │ addr:  Set 주소                                                                 │   │
│   └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                      │                                                  │
│                                      ▼                                                  │
│   ┌────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┐             │
│   │ Bank 0 │ Bank 1 │ Bank 2 │ Bank 3 │ Bank 4 │ Bank 5 │ Bank 6 │ Bank 7 │             │
│   │ WRITE  │ WRITE  │ WRITE  │ WRITE  │ WRITE  │ WRITE  │ WRITE  │ WRITE  │             │
│   │data[0] │data[1] │data[2] │data[3] │data[4] │data[5] │data[6] │data[7] │             │
│   └────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┘             │
│                                                                                         │
│   ※ 64 bytes 전체 블록을 한 사이클에 기록                                                 │
│   ※ wmask로 부분 쓰기도 가능 (예: 8'b00001111 → Bank 0-3만 쓰기)                         │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

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
