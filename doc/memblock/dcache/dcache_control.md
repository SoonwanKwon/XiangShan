# DCache Control Module 상세 분석

## 1. 개요

XiangShan DCache의 Control 모듈들은 캐시의 핵심 제어 로직을 담당합니다. 이 문서에서는 다음 모듈들을 상세히 분석합니다:

- **MissQueue**: Cache Miss 처리 및 L2 Acquire 관리
- **ProbeQueue**: L2로부터의 Probe 요청 처리
- **WritebackQueue**: Dirty 라인의 Release/ProbeAck 처리
- **MainPipe**: Store, AMO, Probe, Replace의 실제 처리
- **RefillPipe**: Refill 데이터를 캐시에 기록

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                        DCache Control Module Architecture                                │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│                           ┌────────────────────────────────────┐                        │
│                           │           MainPipe                  │                        │
│                           │  (Store/AMO/Probe/Replace 처리)     │                        │
│                           └───────────────┬────────────────────┘                        │
│                                           │                                             │
│        ┌──────────────────┬───────────────┼───────────────┬──────────────────┐          │
│        │                  │               │               │                  │          │
│        ▼                  ▼               ▼               ▼                  ▼          │
│  ┌───────────┐      ┌───────────┐   ┌──────────┐   ┌────────────┐   ┌───────────────┐   │
│  │ MissQueue │      │ProbeQueue │   │SBuffer   │   │AtomicsUnit │   │WritebackQueue │   │
│  │ (MSHR)    │      │           │   │(Store)   │   │ (AMO)      │   │               │   │
│  └─────┬─────┘      └─────┬─────┘   └──────────┘   └────────────┘   └───────┬───────┘   │
│        │                  │                                                  │          │
│        │                  │                                                  │          │
│        ▼                  │                                                  ▼          │
│  ┌───────────┐            │                                           ┌──────────┐     │
│  │RefillPipe │            │                                           │TileLink C│     │
│  │           │            │                                           │(Release) │     │
│  └─────┬─────┘            │                                           └──────────┘     │
│        │                  │                                                             │
│        ▼                  ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│  │                         Storage Arrays                                           │   │
│  │   TagArray (Dup) | MetaArray | DataArray (Banked) | ErrorArray | PrefetchArray   │   │
│  └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. MissQueue (Miss Status Holding Registers)

### 2.1 기본 파라미터

```scala
// DCacheParameters에서 정의
nMissEntries: Int = 16    // MSHR 엔트리 수 (2의 거듭제곱)
nReleaseEntries: Int = 18 // Release 엔트리 수 (MSHR보다 커야 함)
nMaxPrefetchEntry: Int = 6 // Prefetch 전용 예약 슬롯
```

### 2.2 MissQueue 아키텍처

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              MissQueue Architecture                                      │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   Miss Sources:                                                                         │
│   ┌────────────┐  ┌────────────┐  ┌────────────┐                                        │
│   │ MainPipe   │  │ LoadPipe[0]│  │ LoadPipe[1]│                                        │
│   │(Store miss)│  │(Load miss) │  │(Load miss) │                                        │
│   └─────┬──────┘  └─────┬──────┘  └─────┬──────┘                                        │
│         │               │               │                                               │
│         └───────────────┼───────────────┘                                               │
│                         ▼                                                               │
│                  ┌──────────────┐                                                       │
│                  │   Arbiter    │                                                       │
│                  │  (S0 Stage)  │                                                       │
│                  └──────┬───────┘                                                       │
│                         │                                                               │
│                         ▼                                                               │
│            ┌────────────────────────┐                                                   │
│            │  Miss Req Pipe Reg     │ ──────────────────────────────────┐               │
│            │  (1 cycle latency)     │                                   │               │
│            └────────────┬───────────┘                                   │               │
│                         │                                               │               │
│                         ▼ (S1 Stage)                                    │               │
│   ┌─────────────────────────────────────────────────────────────────────┼───────────┐   │
│   │                        MissEntry Array                              │           │   │
│   │  ┌─────────┐ ┌─────────┐ ┌─────────┐       ┌─────────┐ ┌─────────┐ │           │   │
│   │  │Entry[0] │ │Entry[1] │ │Entry[2] │ ..... │Entry[13]│ │Entry[14]│ │Entry[15]  │   │
│   │  │ (Normal)│ │ (Normal)│ │ (Normal)│       │(Normal) │ │  (PF)   │ │  (PF)     │   │
│   │  └─────────┘ └─────────┘ └─────────┘       └─────────┘ └─────────┘ └───────────┘   │
│   │                                                                                     │
│   │  ※ 마지막 nMaxPrefetchEntry개 슬롯은 Prefetch 전용                                   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘
│                                                                                         │
│   TileLink Interface:                                                                   │
│   ┌───────────┐          ┌───────────┐          ┌───────────┐                           │
│   │ Channel A │ ◄────────│ Acquire   │ ◄────────│ MissEntry │                           │
│   │ (to L2)   │          │ Arbiter   │          │           │                           │
│   └───────────┘          └───────────┘          └───────────┘                           │
│                                                                                         │
│   ┌───────────┐                                 ┌───────────┐                           │
│   │ Channel D │ ────────────────────────────────│ MissEntry │                           │
│   │ (from L2) │          (Grant/GrantData)      │(by source)│                           │
│   └───────────┘                                 └───────────┘                           │
│                                                                                         │
│   ┌───────────┐          ┌───────────┐          ┌───────────┐                           │
│   │ Channel E │ ◄────────│ GrantAck  │ ◄────────│ MissEntry │                           │
│   │ (to L2)   │          │ Arbiter   │          │           │                           │
│   └───────────┘          └───────────┘          └───────────┘                           │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 MissReq 데이터 구조

```scala
class MissReqWoStoreData {
  val source = UInt(sourceTypeWidth.W)  // 요청 출처 (LOAD=0, STORE=1, AMO=2, PREFETCH≥3)
  val pf_source = UInt(L1PfSourceBits.W) // Prefetch 소스 정보
  val cmd = UInt(M_SZ.W)                 // 메모리 명령 (Read/Write/AMO 등)
  val addr = UInt(PAddrBits.W)           // 물리 주소 (블록 정렬됨)
  val vaddr = UInt(VAddrBits.W)          // 가상 주소
  val way_en = UInt(DCacheWays.W)        // Way 활성화 비트 (one-hot)
  val pc = UInt(VAddrBits.W)             // PC 주소 (디버깅용)

  // Store 관련
  val full_overwrite = Bool()            // 전체 블록 덮어쓰기 여부

  // AMO 관련
  val word_idx = UInt()                  // AMO가 작동하는 word 인덱스
  val amo_data = UInt(DataBits.W)        // AMO 데이터
  val amo_mask = UInt((DataBits/8).W)    // AMO 마스크

  // Coherence 정보
  val req_coh = new ClientMetadata       // 요청 시점 캐시 일관성 상태
  val replace_coh = new ClientMetadata   // 교체될 라인의 일관성 상태
  val replace_tag = UInt(tagBits.W)      // 교체될 라인의 태그

  val id = UInt(reqIdWidth.W)            // 요청 ID
  val cancel = Bool()                    // 취소 신호 (늦게 생성됨)

  // 소스 타입 판별 메서드
  def isFromLoad = source === LOAD_SOURCE.U
  def isFromStore = source === STORE_SOURCE.U
  def isFromAMO = source === AMO_SOURCE.U
  def isFromPrefetch = source >= DCACHE_PREFETCH_SOURCE.U
}

class MissReq extends MissReqWoStoreData {
  val store_data = UInt((blockBytes * 8).W)  // Store 데이터 (512 bits)
  val store_mask = UInt(blockBytes.W)         // Store 마스크 (64 bits)
}
```

### 2.4 MissEntry 상태 머신

각 MissEntry는 여러 상태 플래그로 miss 처리 파이프라인을 추적합니다:

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                           MissEntry State Flags                                          │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  State Flags (RegInit로 초기화):                                                         │
│  ┌──────────────────┬────────────────────────────────────────────────────────────────┐  │
│  │ s_acquire        │ Acquire 요청 발행 가능 여부 (true=발행됨/발행 필요 없음)         │  │
│  │ s_grantack       │ GrantAck 발행 가능 여부                                        │  │
│  │ s_replace_req    │ Replace 요청 발행 가능 여부                                     │  │
│  │ s_refill         │ Refill 요청 발행 가능 여부                                      │  │
│  │ s_mainpipe_req   │ MainPipe 요청 발행 가능 여부 (AMO용)                            │  │
│  ├──────────────────┼────────────────────────────────────────────────────────────────┤  │
│  │ w_grantfirst     │ 첫 Grant beat 수신 완료 여부                                   │  │
│  │ w_grantlast      │ 마지막 Grant beat 수신 완료 (refill 데이터 완료)               │  │
│  │ w_replace_resp   │ Replace 완료 응답 수신 여부                                    │  │
│  │ w_refill_resp    │ Refill 완료 응답 수신 여부                                     │  │
│  │ w_mainpipe_resp  │ MainPipe 처리 완료 여부                                        │  │
│  └──────────────────┴────────────────────────────────────────────────────────────────┘  │
│                                                                                         │
│  Entry Release 조건:                                                                    │
│  release_entry = s_grantack && w_refill_resp && w_mainpipe_resp                         │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

**상태 전이 다이어그램**:

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              MissEntry State Transitions                                 │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   ┌──────────┐                                                                          │
│   │  IDLE    │◄───────────────────────────────────────────────────────────────┐         │
│   │req_valid │                                                                │         │
│   │ = false  │                                                                │         │
│   └────┬─────┘                                                                │         │
│        │ primary_fire                                                         │         │
│        │ (ALLOC)                                                              │         │
│        ▼                                                                      │         │
│   ┌──────────────────┐                                                        │         │
│   │ ALLOCATED        │                                                        │         │
│   │ s_acquire=false  │                                                        │         │
│   │ w_grantfirst=false│                                                       │         │
│   │ w_grantlast=false │                                                       │         │
│   └────┬─────────────┘                                                        │         │
│        │                                                                      │         │
│        ▼ io.mem_acquire.fire                                                  │         │
│   ┌──────────────────┐                                                        │         │
│   │ ACQUIRE SENT     │                                                        │         │
│   │ s_acquire=true   │                                                        │         │
│   └────┬─────────────┘                                                        │         │
│        │                                                                      │         │
│        ▼ io.mem_grant.fire (first beat)                                       │         │
│   ┌──────────────────┐                                                        │         │
│   │ GRANT RECEIVED   │◄──── io.mem_grant.fire (each beat)                     │         │
│   │ w_grantfirst=true│      refill_data_raw(beat) := grant.data               │         │
│   └────┬─────────────┘                                                        │         │
│        │                                                                      │         │
│        ▼ refill_done (last beat)                                              │         │
│   ┌──────────────────┐                                                        │         │
│   │ GRANT COMPLETE   │                                                        │         │
│   │ w_grantlast=true │                                                        │         │
│   └────┬──────┬──────┘                                                        │         │
│        │      │                                                               │         │
│        │      └──────────────────────┐                                        │         │
│        │                             │ (replace 필요 시)                       │         │
│        │                             ▼                                        │         │
│        │                    ┌────────────────────┐                            │         │
│        │                    │ REPLACE REQUEST    │                            │         │
│        │                    │ s_replace_req=false│                            │         │
│        │                    └────────┬───────────┘                            │         │
│        │                             │ io.replace_pipe_req.fire               │         │
│        │                             ▼                                        │         │
│        │                    ┌────────────────────┐                            │         │
│        │                    │ REPLACE DONE       │                            │         │
│        │                    │ w_replace_resp=true│                            │         │
│        │                    └────────┬───────────┘                            │         │
│        │                             │                                        │         │
│        └─────────────────────────────┼────────────────────────────────────┐   │         │
│                                      │                                    │   │         │
│                                      ▼                                    │   │         │
│                             ┌────────────────────┐                        │   │         │
│                             │ REFILL REQUEST     │                        │   │         │
│                             │ (w_replace_resp && │                        │   │         │
│                             │  w_grantlast)      │                        │   │         │
│                             └────────┬───────────┘                        │   │         │
│                                      │ io.refill_pipe_req.fire            │   │         │
│                                      ▼                                    │   │         │
│                             ┌────────────────────┐                        │   │         │
│                             │ REFILL DONE        │                        │   │         │
│                             │ w_refill_resp=true │                        │   │         │
│                             └────────┬───────────┘                        │   │         │
│                                      │                                    │   │         │
│        ┌─────────────────────────────┴────────────────────────────────────┘   │         │
│        │                                                                      │         │
│        ▼ (AMO인 경우)                                                         │         │
│   ┌──────────────────┐                                                        │         │
│   │ MAINPIPE REQUEST │                                                        │         │
│   │ s_mainpipe_req   │                                                        │         │
│   │ =false (AMO)     │                                                        │         │
│   └────┬─────────────┘                                                        │         │
│        │ io.main_pipe_req.fire                                                │         │
│        ▼                                                                      │         │
│   ┌──────────────────┐                                                        │         │
│   │ MAINPIPE DONE    │                                                        │         │
│   │ w_mainpipe_resp  │                                                        │         │
│   │ =true            │                                                        │         │
│   └────┬─────────────┘                                                        │         │
│        │                                                                      │         │
│        ▼                                                                      │         │
│   ┌──────────────────┐                                                        │         │
│   │ GRANTACK SEND    │ ◄─── w_grantfirst 이후 언제든                           │         │
│   │ io.mem_finish    │                                                        │         │
│   │ s_grantack=true  │                                                        │         │
│   └────┬─────────────┘                                                        │         │
│        │                                                                      │         │
│        ▼ release_entry = s_grantack && w_refill_resp && w_mainpipe_resp       │         │
│   ┌──────────────────┐                                                        │         │
│   │ RELEASE ENTRY    │────────────────────────────────────────────────────────┘         │
│   │ req_valid=false  │                                                                  │
│   └──────────────────┘                                                                  │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 2.5 Miss 요청 처리 로직: Alloc vs Merge vs Reject

MissQueue의 요청 처리는 2 사이클에 걸쳐 수행됩니다:

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                        Miss Request Decision Logic                                       │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  [Cycle S0: 요청 중재 및 결정]                                                           │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐    │
│  │                                                                                 │    │
│  │  incoming_req                                                                   │    │
│  │       │                                                                         │    │
│  │       ▼                                                                         │    │
│  │  ┌────────────────────────────────────────────────────────────────────────┐     │    │
│  │  │  For each MissEntry[i]:                                                │     │    │
│  │  │    primary_ready[i] = !req_valid && !RegNext(primary_fire)             │     │    │
│  │  │                       && (슬롯 조건 만족)                                │     │    │
│  │  │    secondary_ready[i] = should_merge(req)                              │     │    │
│  │  │    secondary_reject[i] = should_reject(req)                            │     │    │
│  │  └────────────────────────────────────────────────────────────────────────┘     │    │
│  │       │                                                                         │    │
│  │       ▼                                                                         │    │
│  │  merge = secondary_ready_vec.orR || pipe_reg.merge_req(req)                     │    │
│  │  reject = secondary_reject_vec.orR || pipe_reg.reject_req(req)                  │    │
│  │  alloc = !reject && !merge && primary_ready_vec.orR                             │    │
│  │  accept = alloc || merge                                                        │    │
│  │       │                                                                         │    │
│  │       ▼                                                                         │    │
│  │  ┌─────────┐     ┌─────────┐     ┌─────────┐                                    │    │
│  │  │  ALLOC  │     │  MERGE  │     │ REJECT  │                                    │    │
│  │  │새 엔트리│     │기존 엔트│     │ 요청    │                                    │    │
│  │  │ 할당    │     │리에 병합│     │ 거부    │                                    │    │
│  │  └─────────┘     └─────────┘     └─────────┘                                    │    │
│  │                                                                                 │    │
│  └─────────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                         │
│  [Cycle S1: MSHR 업데이트]                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐    │
│  │  miss_req_pipe_reg에서 요청 정보 읽기                                            │    │
│  │                                                                                 │    │
│  │  if (alloc):                                                                    │    │
│  │    req_valid := true.B                                                          │    │
│  │    req := pipe_reg.req                                                          │    │
│  │    상태 플래그 초기화                                                            │    │
│  │    if (Store) Store 데이터/마스크 저장                                           │    │
│  │                                                                                 │    │
│  │  if (merge):                                                                    │    │
│  │    req.req_coh := 최신 coherence 상태                                           │    │
│  │    if (Store) Store 데이터/마스크 병합                                           │    │
│  │    access := true.B  // non-prefetch 요청 병합 시                               │    │
│  │                                                                                 │    │
│  └─────────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

**Merge 조건** (MissEntry.should_merge):
```scala
def should_merge(new_req: MissReqWoStoreData): Bool = {
  val block_match = get_block(req.addr) === get_block(new_req.addr)
  val alias_match = is_alias_match(req.vaddr, new_req.vaddr)
  block_match && alias_match && (
    before_req_sent_can_merge(new_req) ||   // Acquire 발행 전
    before_data_refill_can_merge(new_req)   // Refill 완료 전
  )
}

def before_req_sent_can_merge(new_req): Bool = {
  acquire_not_sent &&                        // !s_acquire && !io.mem_acquire.ready
  (req.isFromLoad || req.isFromPrefetch) &&  // 기존이 Load/Prefetch
  (new_req.isFromLoad || new_req.isFromStore) // 새 요청이 Load/Store
}

def before_data_refill_can_merge(new_req): Bool = {
  data_not_refilled &&                       // !w_grantfirst
  (req.isFromLoad || req.isFromStore || req.isFromPrefetch) &&
  new_req.isFromLoad                         // 새 요청이 Load만 가능
}
```

**Reject 조건** (MissEntry.should_reject):
```scala
def should_reject(new_req: MissReqWoStoreData): Bool = {
  req_valid &&
  Mux(
    block_match,
    // 블록 주소 일치하지만 병합 불가
    (!before_req_sent_can_merge && !before_data_refill_can_merge) || !alias_match,
    // 블록 주소 불일치이나 set/way 충돌
    set_match && new_req.way_en === req.way_en
  )
}
```

### 2.6 Refill 데이터 머지 (Store Miss)

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              Refill Data Merge (Store Miss)                              │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  L2에서 Grant 수신 시 Store 데이터와 병합:                                                │
│                                                                                         │
│  for (i <- 0 until blockRows) {                                                         │
│    val idx = (refill_count << log2Floor(beatRows)) + i.U                                │
│    val grant_row = io.mem_grant.bits.data(rowBits * (i+1) - 1, rowBits * i)             │
│    refill_and_store_data(idx) := mergePutData(grant_row, new_data(idx), new_mask(idx))  │
│  }                                                                                      │
│                                                                                         │
│  def mergePutData(old_data: UInt, new_data: UInt, wmask: UInt): UInt = {                │
│    val full_wmask = FillInterleaved(8, wmask)                                           │
│    (~full_wmask & old_data | full_wmask & new_data)                                     │
│  }                                                                                      │
│                                                                                         │
│  예시:                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐    │
│  │  grant_data:    [AA AA AA AA AA AA AA AA]  ← L2에서 온 원본 데이터               │    │
│  │  store_data:    [XX XX BB BB XX XX XX XX]  ← Store 데이터                        │    │
│  │  store_mask:    [00 00 FF FF 00 00 00 00]  ← Store 마스크 (바이트 단위)          │    │
│  │  merged_data:   [AA AA BB BB AA AA AA AA]  ← 병합 결과                           │    │
│  └─────────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                         │
│  ※ full_overwrite=true인 경우:                                                         │
│     - AcquireBlock 대신 AcquirePerm만 발행 (데이터 불필요)                              │
│     - L2에서 Grant (without data) 수신                                                 │
│     - Store 데이터만으로 캐시 업데이트                                                  │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 2.7 Early Load Wakeup

```scala
// Load miss 시 Refill 데이터가 도착하면 LoadQueue로 조기 전송
io.refill_to_ldq.valid := RegNext(!w_grantlast && io.mem_grant.fire)
io.refill_to_ldq.bits.addr := RegNext(req.addr + (refill_count << refillOffBits))
io.refill_to_ldq.bits.data := refill_data_splited(RegNext(refill_count))
io.refill_to_ldq.bits.refill_done := RegNext(refill_done && io.mem_grant.fire)

// 이를 통해 Load는 Refill 완료 전에 데이터를 받아 실행 가능
```

---

## 3. ProbeQueue

### 3.1 기본 파라미터

```scala
nProbeEntries: Int  // ProbeQueue 엔트리 수
```

### 3.2 ProbeQueue 아키텍처

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              ProbeQueue Architecture                                     │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   TileLink B Channel (from L2)                                                          │
│        │                                                                                │
│        │ Probe Request                                                                  │
│        ▼                                                                                │
│   ┌────────────────────────────────────────────────────────────────────────────────┐    │
│   │                           ProbeQueue                                            │    │
│   │                                                                                 │    │
│   │   ┌───────────┐                                                                │    │
│   │   │ ProbeReq  │ ← TLBundleB 변환                                               │    │
│   │   │ 생성      │   source, opcode, addr, vaddr, param, needData, id             │    │
│   │   └─────┬─────┘                                                                │    │
│   │         │                                                                       │    │
│   │         │ allocate (primary_ready.asUInt.orR && io.mem_probe.valid)            │    │
│   │         ▼                                                                       │    │
│   │   ┌─────────────────────────────────────────────────────────────────────────┐  │    │
│   │   │                     ProbeEntry Array                                     │  │    │
│   │   │                                                                          │  │    │
│   │   │   ┌─────────────┐  ┌─────────────┐       ┌─────────────┐                 │  │    │
│   │   │   │ ProbeEntry  │  │ ProbeEntry  │ ..... │ ProbeEntry  │                 │  │    │
│   │   │   │    [0]      │  │    [1]      │       │ [n-1]       │                 │  │    │
│   │   │   └──────┬──────┘  └──────┬──────┘       └──────┬──────┘                 │  │    │
│   │   │          │                │                     │                        │  │    │
│   │   │          └────────────────┼─────────────────────┘                        │  │    │
│   │   │                           │                                              │  │    │
│   │   │                           ▼                                              │  │    │
│   │   │                    ┌──────────────┐                                      │  │    │
│   │   │                    │   Arbiter    │                                      │  │    │
│   │   │                    │ (pipe_req)   │                                      │  │    │
│   │   │                    └──────┬───────┘                                      │  │    │
│   │   │                           │                                              │  │    │
│   │   └───────────────────────────┼──────────────────────────────────────────────┘  │    │
│   │                               │                                                 │    │
│   │                               │ (1 cycle delay for timing)                      │    │
│   │                               ▼                                                 │    │
│   │                        ┌──────────────┐                                         │    │
│   │                        │ selected_req │                                         │    │
│   │                        │ (Pipe Reg)   │                                         │    │
│   │                        └──────┬───────┘                                         │    │
│   │                               │                                                 │    │
│   └───────────────────────────────┼─────────────────────────────────────────────────┘    │
│                                   │                                                      │
│                                   ▼                                                      │
│                            io.pipe_req → MainPipe                                        │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 ProbeEntry 상태 머신

```scala
val s_invalid :: s_pipe_req :: s_wait_resp :: Nil = Enum(3)

// 상태 전이:
// s_invalid → s_pipe_req: io.req.fire (새 Probe 요청 할당)
// s_pipe_req → s_wait_resp: io.pipe_req.fire (MainPipe로 요청 전송)
// s_wait_resp → s_invalid: io.pipe_resp.valid && id 일치 (처리 완료)
```

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                            ProbeEntry State Machine                                      │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│       ┌─────────────┐                                                                   │
│       │  s_invalid  │ ◄────────────────────────────────────────────────┐                │
│       │  (idle)     │                                                  │                │
│       └──────┬──────┘                                                  │                │
│              │                                                         │                │
│              │ io.req.fire                                             │                │
│              │ req := io.req.bits                                      │                │
│              ▼                                                         │                │
│       ┌─────────────┐                                                  │                │
│       │ s_pipe_req  │                                                  │                │
│       │             │                                                  │                │
│       │ io.pipe_req │ ── lrsc_blocked 체크 ──┐                         │                │
│       │ .valid :=   │                        │                         │                │
│       │ !lrsc_blocked│                       │ (LR/SC 블록 시 대기)    │                │
│       └──────┬──────┘                        │                         │                │
│              │                               │                         │                │
│              │ io.pipe_req.fire              │                         │                │
│              ▼                               │                         │                │
│       ┌─────────────┐                        │                         │                │
│       │ s_wait_resp │ ◄──────────────────────┘                         │                │
│       │             │                                                  │                │
│       │ MainPipe    │                                                  │                │
│       │ 처리 대기   │                                                  │                │
│       └──────┬──────┘                                                  │                │
│              │                                                         │                │
│              │ io.pipe_resp.valid && io.id === io.pipe_resp.bits.id    │                │
│              │                                                         │                │
│              └─────────────────────────────────────────────────────────┘                │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 3.4 LR/SC Blocking

```scala
// Probe 요청이 LR/SC로 잠긴 블록과 충돌하면 블록
val lrsc_blocked = Mux(
  io.req.fire,
  io.lrsc_locked_block.valid && get_block(io.lrsc_locked_block.bits) === get_block(io.req.bits.addr),
  io.lrsc_locked_block.valid && get_block(io.lrsc_locked_block.bits) === get_block(req.addr)
)

// 다음 사이클에 체크하여 타이밍 개선
io.pipe_req.valid := !RegNext(lrsc_blocked)
```

### 3.5 TLBundleB → ProbeReq 변환

```scala
class ProbeReq {
  val source = UInt()              // TLBundleB.source
  val opcode = UInt()              // TLBundleB.opcode (항상 TLMessages.Probe)
  val addr = UInt(PAddrBits.W)     // TLBundleB.address (물리 주소)
  val vaddr = UInt(VAddrBits.W)    // 가상 주소 (alias 처리용)
  val param = UInt()               // TLBundleB.param (toN/toB/toT)
  val needData = Bool()            // TLBundleB.data(0)
  val id = UInt()                  // ProbeQueue entry ID
}

// Alias 처리 (L2에서 vaddr index 전달)
val alias_addr_frag = io.mem_probe.bits.data(2, 1)
req.vaddr := Cat(
  io.mem_probe.bits.address(PAddrBits - 1, DCacheAboveIndexOffset),
  alias_addr_frag(DCacheAboveIndexOffset - DCacheTagOffset - 1, 0),
  io.mem_probe.bits.address(DCacheTagOffset - 1, 0)
)
```

---

## 4. WritebackQueue

### 4.1 기본 파라미터

```scala
nReleaseEntries: Int = 18  // WritebackQueue 엔트리 수 (nMissEntries보다 커야 함)
```

### 4.2 WritebackQueue 아키텍처

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                           WritebackQueue Architecture                                    │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   Request Sources:                                                                      │
│   ┌──────────────────┐  ┌──────────────────┐                                           │
│   │    MainPipe      │  │    MainPipe      │                                           │
│   │   (Release:      │  │   (ProbeAck:     │                                           │
│   │    voluntary)    │  │    !voluntary)   │                                           │
│   └────────┬─────────┘  └────────┬─────────┘                                           │
│            │                     │                                                      │
│            └──────────┬──────────┘                                                      │
│                       │                                                                 │
│                       ▼                                                                 │
│   ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│   │                         WritebackQueue                                           │   │
│   │                                                                                  │   │
│   │   ┌──────────────────────────────────────────────────────────────────────────┐  │   │
│   │   │                    WritebackEntry Array                                   │  │   │
│   │   │                                                                          │  │   │
│   │   │   ┌───────────┐  ┌───────────┐       ┌───────────┐                       │  │   │
│   │   │   │ WBEntry   │  │ WBEntry   │ ..... │ WBEntry   │                       │  │   │
│   │   │   │   [0]     │  │   [1]     │       │  [17]     │                       │  │   │
│   │   │   └─────┬─────┘  └─────┬─────┘       └─────┬─────┘                       │  │   │
│   │   │         │              │                   │                             │  │   │
│   │   │         └──────────────┼───────────────────┘                             │  │   │
│   │   │                        │                                                 │  │   │
│   │   │                        ▼                                                 │  │   │
│   │   │                 ┌──────────────┐                                         │  │   │
│   │   │                 │ RR Arbiter   │                                         │  │   │
│   │   │                 │ (mem_release)│                                         │  │   │
│   │   │                 └──────┬───────┘                                         │  │   │
│   │   │                        │                                                 │  │   │
│   │   └────────────────────────┼─────────────────────────────────────────────────┘  │   │
│   │                            │                                                    │   │
│   └────────────────────────────┼────────────────────────────────────────────────────┘   │
│                                │                                                        │
│                                ▼                                                        │
│                    ┌────────────────────┐                                               │
│                    │  TileLink C Channel│                                               │
│                    │  (Release/ProbeAck)│                                               │
│                    └─────────┬──────────┘                                               │
│                              │                                                          │
│                              ▼                                                          │
│                    ┌────────────────────┐                                               │
│                    │  TileLink D Channel│                                               │
│                    │   (ReleaseAck)     │                                               │
│                    └────────────────────┘                                               │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 WritebackEntry 상태 머신

```scala
val s_invalid :: s_sleep :: s_release_req :: s_release_resp :: Nil = Enum(4)
```

**상태 전이 시나리오**:

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                        WritebackEntry State Transitions                                  │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  [시나리오 1: ProbeAck (비자발적)]                                                       │
│  s_invalid → s_release_req → s_invalid                                                  │
│              (즉시 발행)      (ProbeAck 완료)                                            │
│                                                                                         │
│  [시나리오 2: Release (자발적)]                                                          │
│  s_invalid → s_sleep → s_release_req → s_release_resp → s_invalid                       │
│              (대기)    (Release 발행)  (ReleaseAck 대기)  (완료)                          │
│                                                                                         │
│  [시나리오 3: Release 중 ProbeAck 병합]                                                  │
│  Case A: Release 아직 발행 안 됨                                                        │
│    s_sleep에서 merge → Release를 ProbeAck으로 변경                                      │
│    s_sleep → s_release_req → s_invalid                                                  │
│                                                                                         │
│  Case B: Release 이미 발행됨                                                            │
│    release_later := true.B로 설정                                                       │
│    s_release_req → s_release_resp → s_release_req (ProbeAck) → s_invalid                │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

**상세 상태 전이 다이어그램**:

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                    WritebackEntry Detailed State Machine                                 │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│       ┌─────────────┐                                                                   │
│       │  s_invalid  │ ◄───────────────────────────────────────────────────────┐         │
│       │   (idle)    │                                                         │         │
│       └──────┬──────┘                                                         │         │
│              │                                                                │         │
│              │ alloc && io.req.valid                                          │         │
│              │                                                                │         │
│        ┌─────┴─────┐                                                          │         │
│        │           │                                                          │         │
│        ▼           ▼                                                          │         │
│  ┌───────────┐  ┌───────────┐                                                 │         │
│  │ s_sleep   │  │s_release_ │ (delay_release=false, 즉 ProbeAck)              │         │
│  │(delay_    │  │   req     │                                                 │         │
│  │release=   │  └─────┬─────┘                                                 │         │
│  │true)      │        │                                                       │         │
│  └─────┬─────┘        │ release_done                                          │         │
│        │              └───────────────────────────────────────────────────────┤         │
│        │                                                                      │         │
│        │ release_wakeup (MissQueue에서)                                       │         │
│        │ 또는 ProbeAck merge                                                  │         │
│        ▼                                                                      │         │
│  ┌───────────────┐                                                            │         │
│  │ s_release_req │                                                            │         │
│  │               │ ◄──── release_later=true 시 ProbeAck 추가 발행             │         │
│  │ remain bits   │       (release_done 후 다시 이 상태로)                      │         │
│  │ 관리          │                                                            │         │
│  └───────┬───────┘                                                            │         │
│          │                                                                    │         │
│          │ release_done (모든 beat 전송 완료)                                  │         │
│          │                                                                    │         │
│    ┌─────┴─────┐                                                              │         │
│    │           │                                                              │         │
│    ▼           ▼                                                              │         │
│ ┌────────┐  ┌────────────────┐                                                │         │
│ │ProbeAck│  │s_release_resp  │ (voluntary=true, Release인 경우)               │         │
│ │완료    │  │                │                                                │         │
│ │        │  │ ReleaseAck 대기│                                                │         │
│ └───┬────┘  └───────┬────────┘                                                │         │
│     │               │                                                         │         │
│     │               │ io.mem_grant.fire (ReleaseAck 수신)                      │         │
│     │               │                                                         │         │
│     │         ┌─────┴─────┐                                                   │         │
│     │         │           │                                                   │         │
│     │         ▼           ▼                                                   │         │
│     │    ┌────────┐  ┌────────────────┐                                       │         │
│     │    │release_│  │ merge 있거나   │                                       │         │
│     │    │later=  │  │ release_later  │                                       │         │
│     │    │false   │  │ =true          │                                       │         │
│     │    └───┬────┘  └───────┬────────┘                                       │         │
│     │        │               │                                                │         │
│     │        │               │ s_release_req로 이동                           │         │
│     │        │               │ (다음 요청 처리)                                │         │
│     │        │               │                                                │         │
│     └────────┴───────────────┴────────────────────────────────────────────────┘         │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 4.4 WritebackReq 데이터 구조

```scala
class WritebackReqCtrl {
  val param = UInt(cWidth.W)         // Release/ProbeAck 권한
  val voluntary = Bool()             // true: Release, false: ProbeAck
  val hasData = Bool()               // 데이터 포함 여부
  val dirty = Bool()                 // Dirty 여부
  val delay_release = Bool()         // Sleep 상태 필요 여부
  val miss_id = UInt()               // 연관된 MissQueue 엔트리 ID
}

class WritebackReq extends WritebackReqCtrl {
  val addr = UInt(PAddrBits.W)       // 물리 주소
  val data = UInt((blockBytes * 8).W) // 캐시라인 데이터 (512 bits)
}
```

### 4.5 Release Wakeup 메커니즘

```scala
// MissQueue에서 Refill 완료 시 WritebackQueue 깨움
when (io.release_wakeup.valid && io.release_wakeup.bits === req.miss_id) {
  state := s_release_req
  req.delay_release := false.B
  remain_set := Mux(req.hasData, ~0.U(refillCycles.W), 1.U(refillCycles.W))
}
```

### 4.6 Beat 전송 로직

```scala
val remain = RegInit(0.U(refillCycles.W))  // 남은 beat 비트맵
val beat = PriorityEncoder(remain)          // 현재 전송할 beat

// Release 또는 ProbeAck 메시지 생성
io.mem_release.valid := busy  // remain.orR && s_data_override && s_data_merge
io.mem_release.bits := Mux(req.voluntary,
  Mux(req.hasData, voluntaryReleaseData, voluntaryRelease),
  Mux(req.hasData, probeResponseData, probeResponse)
)

// beat 전송 후 remain 클리어
when (io.mem_release.fire) {
  remain_clr := PriorityEncoderOH(remain)
}
```

### 4.7 Probe TtoB Override 체크

```scala
// Probe TtoB가 들어왔을 때 해당 블록이 아직 Release 대기 중이면
// TtoB를 TtoN으로 변경하여 캐시 coherence를 N으로 설정
io.probe_ttob_check_resp.bits.toN := RegNext(
  state === s_sleep &&
  io.probe_ttob_check_req.bits.addr === paddr &&
  io.probe_ttob_check_req.valid
)
```

---

## 5. MainPipe

### 5.1 MainPipe 역할

MainPipe는 DCache의 핵심 파이프라인으로, 다음 요청들을 처리합니다:
- **Probe**: L2로부터의 일관성 요청 (최우선)
- **Replace**: MissQueue에서의 교체 요청
- **Store**: SBuffer에서의 쓰기 요청
- **Atomic (AMO)**: 원자적 메모리 연산

### 5.2 4-Stage Pipeline 구조

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              MainPipe 4-Stage Pipeline                                   │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  +-------+     +-------+     +-------+     +-------+                                    │
│  |  S0   | --> |  S1   | --> |  S2   | --> |  S3   |                                    │
│  +-------+     +-------+     +-------+     +-------+                                    │
│   Request      Tag Match     Data Process  Write Arrays                                 │
│   Arbiter      Data Read     Miss Judge    Generate Resp                                │
│   Meta/Tag                   AMO Execute                                                │
│   Read                       (if needed)                                                │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 5.3 요청 우선순위 및 Arbiter

```scala
// S0: 요청 중재 (우선순위: Probe > Replace > Store > Atomic)
arbiter(
  in = Seq(
    io.probe_req,    // ProbeQueue에서 (최우선)
    io.replace_req,  // MissQueue에서
    store_req,       // SBuffer에서 (storeCanAccept 조건부)
    io.atomic_req    // AtomicsUnit에서
  ),
  out = req,
  name = Some("main_pipe_req")
)

// Store 요청은 Load와의 충돌을 고려하여 조건부 수락
val storeCanAccept = storeWaitTooLong || !loadsAreComing || io.force_write
store_req.valid := io.store_req.valid && storeCanAccept
```

### 5.4 Stage 0 (S0): 요청 수신 및 Meta/Tag 읽기

```scala
// S0 로직
val s0_req = req.bits
val s0_idx = get_idx(s0_req.vaddr)
val s0_can_go = io.meta_read.ready && io.tag_read.ready && s1_ready && !set_conflict
val s0_fire = req.valid && s0_can_go

// Set Conflict 체크 (S1, S2, S3와 같은 set 접근 방지)
val set_conflict = s1_s0_set_conflict || s2_s0_set_conlict || s3_s0_set_conflict

// Meta/Tag 읽기 요청 발행
io.meta_read.valid := req.valid
io.meta_read.bits.idx := s0_idx
io.tag_read.valid := req.valid
io.tag_read.bits.idx := s0_idx

// Data 읽기 마스크 생성 (요청 종류에 따라)
val store_need_data = !s0_req.probe && s0_req.isStore && banked_store_rmask.orR
val probe_need_data = s0_req.probe
val amo_need_data = !s0_req.probe && s0_req.isAMO
val miss_need_data = s0_req.miss
val replace_need_data = s0_req.replace
```

### 5.5 Stage 1 (S1): Tag 비교 및 Data 읽기

```scala
// S1 로직
val s1_valid = RegInit(false.B)
val s1_req = RegEnable(s0_req, s0_fire)

// Tag 비교
val s1_tag_eq_way = wayMap((w: Int) => tag_resp(w) === get_tag(s1_req.addr)).asUInt
val s1_tag_match_way = wayMap((w: Int) =>
  s1_tag_eq_way(w) && Meta(meta_resp(w)).coh.isValid()
).asUInt
val s1_tag_match = ParallelORR(s1_tag_match_way)

// Hit 판정
val s1_hit_coh = ClientMetadata(ParallelMux(s1_tag_match_way.asBools, ...))
val s1_has_permission = s1_hit_coh.onAccess(s1_req.cmd)._1
val s1_hit = s1_tag_match && s1_has_permission

// Replacement Way 선택 (Miss 시)
val s1_need_replacement = (s1_req.miss || s1_req.isStore && !s1_req.probe) && !s1_tag_match
val s1_way_en = Mux(s1_req.replace, s1_req.replace_way_en,
                Mux(s1_req.miss, s1_req.miss_way_en,
                Mux(s1_need_replacement, s1_repl_way_en, s1_tag_match_way)))

// Data 읽기 요청 (필요 시)
io.data_readline.valid := s1_valid && s1_need_data
io.data_readline.bits.addr := s1_req.vaddr
io.data_readline.bits.way_en := s1_way_en
io.data_readline.bits.rmask := s1_banked_rmask
```

### 5.6 Stage 2 (S2): Data 응답 및 Miss 판정

```scala
// S2 로직
val s2_valid = RegInit(false.B)
val s2_req = RegEnable(s1_req, s1_fire)
val s2_hit = s2_tag_match && s2_has_permission

// Store Hit/Miss 판정
val s2_store_hit = s2_hit && !s2_req.probe && !s2_req.miss && s2_req.isStore
val s2_amo_hit = s2_hit && !s2_req.probe && !s2_req.miss && s2_req.isAMO

// Store 데이터 머지 (기존 데이터 + 새 데이터)
for (i <- 0 until DCacheBanks) {
  val old_data = s2_data(i)
  val new_data = get_data_of_bank(i, s2_req.store_data)
  val wmask = get_mask_of_bank(i, s2_req.store_mask)
  s2_store_data_merged(i) := mergePutData(old_data, new_data, wmask)
}

// Miss 시 MissQueue로 요청 (Store가 miss인 경우)
val s2_can_go_to_mq = !s2_req.replace && !s2_req.probe && !s2_req.miss &&
                      (s2_req.isStore || s2_req.isAMO) && !s2_hit
io.miss_req.valid := s2_valid && s2_can_go_to_mq
```

### 5.7 Stage 3 (S3): Array 쓰기 및 응답 생성

```scala
// S3 로직
val s3_valid = RegInit(false.B)
val s3_req = RegEnable(s2_req, s2_fire_to_s3)

// LR/SC 처리
val s3_lr = !s3_req.probe && s3_req.isAMO && s3_req.cmd === M_XLR
val s3_sc = !s3_req.probe && s3_req.isAMO && s3_req.cmd === M_XSC
val s3_sc_fail = s3_sc && !s3_lrsc_addr_match

// AMO ALU 연산
val amoalu = Module(new AMOALU(wordBits))
amoalu.io.mask := s3_req.amo_mask
amoalu.io.cmd := s3_req.cmd
amoalu.io.lhs := s3_data_word
amoalu.io.rhs := s3_req.amo_data

// Meta 업데이트 판정
val miss_update_meta = s3_req.miss
val probe_update_meta = s3_req.probe && s3_tag_match && s3_coh =/= probe_new_coh
val store_update_meta = s3_req.isStore && !s3_req.probe && s3_hit_coh =/= s3_new_hit_coh
val amo_update_meta = s3_req.isAMO && !s3_req.probe && s3_hit_coh =/= s3_new_hit_coh
val update_meta = (miss_update_meta || probe_update_meta || store_update_meta || amo_update_meta) && !s3_req.replace

// Data 업데이트 판정
val update_data = s3_req.miss || s3_store_hit || s3_can_do_amo_write

// Writeback 판정
val miss_wb = s3_req.miss && s3_need_replacement && s3_coh.state =/= ClientStates.Nothing
val probe_wb = s3_req.probe
val replace_wb = s3_req.replace
val need_wb = miss_wb || probe_wb || replace_wb

// S3 Can Go 조건
val s3_probe_can_go = s3_req.probe && io.wb.ready && (io.meta_write.ready || !probe_update_meta)
val s3_store_can_go = s3_req.isStore && !s3_req.probe &&
                      (io.meta_write.ready || !store_update_meta) &&
                      (io.data_write.ready || !update_data)
val s3_amo_can_go = s3_amo_hit &&
                    (io.meta_write.ready || !amo_update_meta) &&
                    (io.data_write.ready || !update_data) &&
                    (s3_s_amoalu || !amo_wait_amoalu)
val s3_miss_can_go = s3_req.miss &&
                     (io.meta_write.ready || !amo_update_meta) &&
                     (io.data_write.ready || !update_data) &&
                     (s3_s_amoalu || !amo_wait_amoalu) &&
                     io.tag_write.ready && io.wb.ready
val s3_replace_can_go = s3_req.replace && (s3_replace_nothing || io.wb.ready)

val s3_can_go = s3_probe_can_go || s3_store_can_go || s3_amo_can_go || s3_miss_can_go || s3_replace_can_go
```

### 5.8 Coherence 상태 업데이트

```scala
// Miss 시 새로운 Coherence 상태 생성
def missCohGen(cmd: UInt, param: UInt, dirty: Bool) = {
  val c = categorize(cmd)
  MuxLookup(Cat(c, param, dirty), Nothing, Seq(
    Cat(rd, toB, false.B)  -> Branch,   // Read, toB -> Branch
    Cat(rd, toB, true.B)   -> Branch,
    Cat(rd, toT, false.B)  -> Trunk,    // Read, toT -> Trunk
    Cat(rd, toT, true.B)   -> Dirty,    // Read, toT, dirty -> Dirty
    Cat(wi, toT, false.B)  -> Trunk,    // WriteIntent, toT -> Trunk
    Cat(wi, toT, true.B)   -> Dirty,
    Cat(wr, toT, false.B)  -> Dirty,    // Write, toT -> Dirty
    Cat(wr, toT, true.B)   -> Dirty
  ))
}

// Probe 시 새로운 Coherence 상태
val (_, _, probe_new_coh) = s3_coh.onProbe(s3_req.probe_param)

// Store Hit 시 새로운 Coherence 상태
val (s2_has_permission, _, s2_new_hit_coh) = s2_hit_coh.onAccess(s2_req.cmd)
```

---

## 6. RefillPipe

### 6.1 RefillPipe 역할

RefillPipe는 MissQueue에서 refill 받은 데이터를 캐시 Array에 기록하는 단일 사이클 파이프라인입니다.

### 6.2 RefillPipe 구조

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              RefillPipe Structure                                        │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   MissQueue                                                                             │
│       │                                                                                 │
│       │ RefillPipeReq                                                                   │
│       │ (source, vaddr, addr, way_en, wmask, data, meta, error, prefetch, access)       │
│       ▼                                                                                 │
│   ┌───────────────────────────────────────────────────────────────────────────────┐     │
│   │                           RefillPipe                                           │     │
│   │                                                                                │     │
│   │   io.req.ready := true.B  (항상 수락 가능)                                      │     │
│   │                                                                                │     │
│   │   1 Cycle에 동시 수행:                                                          │     │
│   │   ┌─────────────────────────────────────────────────────────────────────────┐  │     │
│   │   │                                                                         │  │     │
│   │   │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                  │  │     │
│   │   │   │  data_write  │  │  meta_write  │  │  tag_write   │                  │  │     │
│   │   │   │  (8 banks)   │  │              │  │              │                  │  │     │
│   │   │   └──────────────┘  └──────────────┘  └──────────────┘                  │  │     │
│   │   │                                                                         │  │     │
│   │   │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                  │  │     │
│   │   │   │error_flag_   │  │prefetch_flag_│  │access_flag_  │                  │  │     │
│   │   │   │write         │  │write         │  │write         │                  │  │     │
│   │   │   └──────────────┘  └──────────────┘  └──────────────┘                  │  │     │
│   │   │                                                                         │  │     │
│   │   └─────────────────────────────────────────────────────────────────────────┘  │     │
│   │                                                                                │     │
│   │   출력:                                                                        │     │
│   │   - io.resp: miss_id 반환                                                      │     │
│   │   - io.store_resp: Store 요청에 대한 응답                                       │     │
│   │   - io.release_wakeup: WritebackQueue 깨우기                                   │     │
│   │                                                                                │     │
│   └────────────────────────────────────────────────────────────────────────────────┘     │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 6.3 RefillPipeReq 데이터 구조

```scala
class RefillPipeReqCtrl {
  val source = UInt(sourceTypeWidth.W)   // 요청 출처 (LOAD/STORE/AMO/PREFETCH)
  val vaddr = UInt(VAddrBits.W)          // 가상 주소
  val addr = UInt(PAddrBits.W)           // 물리 주소
  val way_en = UInt(DCacheWays.W)        // Way 선택 (one-hot)
  val alias = UInt(2.W)                  // Alias 비트
  val miss_id = UInt()                   // MissQueue 엔트리 ID
  val id = UInt(reqIdWidth.W)            // 요청 ID
  val error = Bool()                     // 에러 플래그
  val prefetch = UInt(L1PfSourceBits.W)  // Prefetch 소스
  val access = Bool()                    // 실제 접근 여부

  def paddrWithVirtualAlias: UInt = Cat(alias, addr(DCacheSameVPAddrLength - 1, 0))
  def idx: UInt = get_idx(paddrWithVirtualAlias)
}

class RefillPipeReq extends RefillPipeReqCtrl {
  val wmask = UInt(DCacheBanks.W)                    // Bank별 쓰기 마스크
  val data = Vec(DCacheBanks, UInt(DCacheSRAMRowBits.W))  // 8 banks × 64 bits
  val meta = new Meta                                // Coherence 상태
}
```

### 6.4 RefillPipe 동작 로직

```scala
class RefillPipe {
  io.req.ready := true.B  // 항상 준비 상태
  io.resp.valid := io.req.fire
  io.resp.bits := refill_w_req.miss_id

  // Data Array 쓰기
  io.data_write.valid := io.req_dup_for_data_w(0).valid
  io.data_write.bits.addr := io.req_dup_for_data_w(0).bits.paddrWithVirtualAlias
  io.data_write.bits.way_en := io.req_dup_for_data_w(0).bits.way_en
  io.data_write.bits.wmask := refill_w_req.wmask
  io.data_write.bits.data := refill_w_req.data

  // Meta Array 쓰기
  io.meta_write.valid := io.req_dup_for_meta_w.valid
  io.meta_write.bits.idx := req_dup_for_meta_w.idx
  io.meta_write.bits.way_en := req_dup_for_meta_w.way_en
  io.meta_write.bits.meta := refill_w_req.meta

  // Tag Array 쓰기
  io.tag_write.valid := io.req_dup_for_tag_w.valid
  io.tag_write.bits.idx := req_dup_for_tag_w.idx
  io.tag_write.bits.way_en := req_dup_for_tag_w.way_en
  io.tag_write.bits.tag := get_tag(refill_w_req.addr)

  // Error/Prefetch/Access Flag 쓰기
  io.error_flag_write.valid := io.req_dup_for_err_w.valid
  io.error_flag_write.bits.flag := refill_w_req.error

  io.prefetch_flag_write.valid := io.req_dup_for_err_w.valid
  io.prefetch_flag_write.bits.source := refill_w_req.prefetch

  io.access_flag_write.valid := io.req_dup_for_err_w.valid
  io.access_flag_write.bits.flag := refill_w_req.access

  // Store 요청에 대한 응답
  io.store_resp.valid := refill_w_valid && refill_w_req.source === STORE_SOURCE.U
  io.store_resp.bits.miss := false.B
  io.store_resp.bits.replay := false.B
  io.store_resp.bits.id := refill_w_req.id

  // WritebackQueue 깨우기 (Refill 완료 알림)
  io.release_wakeup.valid := refill_w_valid
  io.release_wakeup.bits := refill_w_req.miss_id
}
```

---

## 7. Control Module 간 상호작용

### 7.1 전체 데이터 흐름

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                     Control Module Interaction Flow                                      │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  [Load Miss 처리 흐름]                                                                   │
│                                                                                         │
│  LoadPipe S2 (miss) ──► MissQueue (alloc) ──► TileLink A (Acquire)                      │
│                                   │                    │                                │
│                                   │                    ▼                                │
│                                   │              L2 Cache                               │
│                                   │                    │                                │
│                                   │                    ▼                                │
│                                   │◄──────── TileLink D (Grant)                         │
│                                   │                                                     │
│                                   │ (replace 필요 시)                                    │
│                                   ▼                                                     │
│                             MainPipe (replace) ──► WritebackQueue ──► TileLink C        │
│                                   │                                                     │
│                                   │ (replace 완료)                                       │
│                                   ▼                                                     │
│                             RefillPipe ──► Data/Tag/Meta Array                          │
│                                   │                                                     │
│                                   ▼                                                     │
│                             MissQueue (release entry)                                   │
│                                   │                                                     │
│                                   ▼                                                     │
│                         TileLink E (GrantAck)                                           │
│                                                                                         │
│  [Store Miss 처리 흐름]                                                                  │
│                                                                                         │
│  SBuffer ──► MainPipe S0-S2 (miss) ──► MissQueue (alloc) ──► TileLink A                 │
│                                               │                    │                    │
│                                               │                    ▼                    │
│                                               │              L2 Cache                   │
│                                               │                    │                    │
│                                               │◄──────── TileLink D                     │
│                                               │    (데이터 머지)                         │
│                                               ▼                                         │
│                                         RefillPipe ──► Arrays                           │
│                                                                                         │
│  [Probe 처리 흐름]                                                                       │
│                                                                                         │
│  TileLink B ──► ProbeQueue ──► MainPipe ──► WritebackQueue ──► TileLink C               │
│                                    │              (ProbeAck)                            │
│                                    ▼                                                    │
│                              Meta 업데이트                                               │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Wakeup 신호 흐름

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              Wakeup Signal Flow                                          │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  [MissQueue → LoadQueue Wakeup]                                                         │
│                                                                                         │
│    MissQueue.io.refill_to_ldq.valid                                                     │
│         │                                                                               │
│         │  Grant beat 수신 시마다 (마지막 beat 전에도!)                                  │
│         ▼                                                                               │
│    LoadQueue (대기 중인 Load 완료 가능)                                                  │
│                                                                                         │
│  [RefillPipe → WritebackQueue Wakeup]                                                   │
│                                                                                         │
│    RefillPipe.io.release_wakeup.valid                                                   │
│         │                                                                               │
│         │  Refill 완료 시                                                               │
│         ▼                                                                               │
│    WritebackQueue (s_sleep → s_release_req)                                             │
│                                                                                         │
│  [MainPipe → MissQueue Response]                                                        │
│                                                                                         │
│    MainPipe.io.replace_resp (replace 완료)                                              │
│    MainPipe.io.atomic_resp (AMO 완료)                                                   │
│         │                                                                               │
│         ▼                                                                               │
│    MissQueue (w_replace_resp, w_mainpipe_resp 업데이트)                                  │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. 성능 카운터 및 디버깅

### 8.1 MissQueue 성능 카운터

```scala
XSPerfAccumulate("miss_req", io.req.fire && !io.req.bits.cancel)
XSPerfAccumulate("miss_req_allocate", io.req.fire && alloc)
XSPerfAccumulate("miss_req_load_allocate", io.req.fire && alloc && io.req.bits.isFromLoad)
XSPerfAccumulate("miss_req_store_allocate", io.req.fire && alloc && io.req.bits.isFromStore)
XSPerfAccumulate("miss_req_merge_load", io.req.fire && merge && io.req.bits.isFromLoad)
XSPerfAccumulate("miss_req_reject_load", io.req.valid && reject && io.req.bits.isFromLoad)
XSPerfAccumulate("probe_blocked_by_miss", io.probe_block)

// Miss Penalty 히스토그램
XSPerfHistogram("miss_penalty", mshr_penalty, mshr_penalty_sample, 0, 20, 1, true, true)
XSPerfHistogram("load_miss_penalty_to_use", load_miss_penalty, load_miss_penalty_sample, 0, 20, 1, true, true)
XSPerfHistogram("a_to_d_penalty", a_to_d_penalty, a_to_d_penalty_sample, 0, 20, 1, true, true)
```

### 8.2 WritebackQueue 성능 카운터

```scala
XSPerfAccumulate("wb_req", io.req.fire)
XSPerfAccumulate("wb_release", state === s_release_req && release_done && req.voluntary)
XSPerfAccumulate("wb_probe_resp", state === s_release_req && release_done && !req.voluntary)
XSPerfAccumulate("wb_probe_ttob_fix", io.probe_ttob_check_resp.valid && io.probe_ttob_check_resp.bits.toN)
XSPerfAccumulate("penalty_blocked_by_channel_C", io.mem_release.valid && !io.mem_release.ready)
```

### 8.3 ProbeQueue 성능 카운터

```scala
XSPerfAccumulate("probe_req", state === s_invalid && io.req.fire)
XSPerfAccumulate("probe_penalty", state =/= s_invalid)
XSPerfAccumulate("probe_penalty_blocked_by_lrsc", state === s_pipe_req && lrsc_blocked)
XSPerfAccumulate("probe_penalty_blocked_by_pipeline", state === s_pipe_req && io.pipe_req.valid && !io.pipe_req.ready)
```

---

## 9. 핵심 설계 특징

### 9.1 비순차 Miss 처리

- MSHR는 여러 Miss를 동시에 처리 가능 (nMissEntries = 16)
- 같은 블록의 Miss는 병합(Merge)하여 효율성 증가
- Prefetch 전용 슬롯으로 일반 요청과 분리

### 9.2 Early Wakeup

- Grant의 첫 beat 수신 시 LoadQueue로 조기 전송
- Refill 완료 전에 Load 재시도 가능
- 메모리 지연 숨김

### 9.3 2-Cycle Miss Queue Enqueue

- S0: 요청 중재, Alloc/Merge/Reject 결정
- S1: 실제 엔트리 업데이트
- 파이프라인 레지스터로 타이밍 개선

### 9.4 LR/SC 보호

- Probe가 LR/SC로 잠긴 블록에 접근 시 블록
- LRSC 카운터로 유효 기간 관리 (LRSCCycles)
- 무한 대기 방지를 위한 타임아웃

### 9.5 Release/ProbeAck 병합

- 같은 주소의 Release와 ProbeAck 병합 가능
- WritebackQueue에서 자동 처리
- TileLink 트래픽 최적화

---

## 10. 소스 코드 참조

| 모듈 | 파일 위치 |
|------|----------|
| MissQueue | [src/main/scala/xiangshan/cache/dcache/mainpipe/MissQueue.scala](src/main/scala/xiangshan/cache/dcache/mainpipe/MissQueue.scala) |
| ProbeQueue | [src/main/scala/xiangshan/cache/dcache/mainpipe/Probe.scala](src/main/scala/xiangshan/cache/dcache/mainpipe/Probe.scala) |
| WritebackQueue | [src/main/scala/xiangshan/cache/dcache/mainpipe/WritebackQueue.scala](src/main/scala/xiangshan/cache/dcache/mainpipe/WritebackQueue.scala) |
| MainPipe | [src/main/scala/xiangshan/cache/dcache/mainpipe/MainPipe.scala](src/main/scala/xiangshan/cache/dcache/mainpipe/MainPipe.scala) |
| RefillPipe | [src/main/scala/xiangshan/cache/dcache/mainpipe/RefillPipe.scala](src/main/scala/xiangshan/cache/dcache/mainpipe/RefillPipe.scala) |
| DCacheWrapper | [src/main/scala/xiangshan/cache/dcache/DCacheWrapper.scala](src/main/scala/xiangshan/cache/dcache/DCacheWrapper.scala) |
