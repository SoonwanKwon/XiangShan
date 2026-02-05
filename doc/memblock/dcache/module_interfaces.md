# DCache Module Interfaces

## 1. LoadPipe 인터페이스

**파일 위치**: `src/main/scala/xiangshan/cache/dcache/loadpipe/LoadPipe.scala`

### 1.1 LoadPipe IO Bundle
```scala
class LoadPipe IO {
  // LSU 연결
  val lsu: Flipped(DCacheLoadIO)        // Load Unit으로부터의 요청/응답

  // DWPU (Way Prediction)
  val dwpu: Flipped(DwpuBaseIO)         // Way Prediction Unit
  val load128Req: Input(Bool)           // 128-bit 로드 요청

  // Nack 신호
  val nack: Input(Bool)                 // S0에서 nack되었는지

  // Meta Array
  val meta_read: DecoupledIO(MetaReadReq)    // Meta 읽기 요청
  val meta_resp: Input(Vec[nWays, Meta])     // Meta 읽기 응답
  val extra_meta_resp: Input(Vec[nWays, DCacheExtraMeta])  // 추가 메타 (error, prefetch, access)

  // Tag Array
  val tag_read: DecoupledIO(TagReadReq)      // Tag 읽기 요청
  val tag_resp: Input(Vec[nWays, UInt])      // Tag 읽기 응답
  val vtag_update: Flipped(DecoupledIO(TagWriteReq))  // VTag 업데이트

  // Data Array
  val banked_data_read: DecoupledIO(L1BankedDataReadReqWithMask)  // 뱅크 데이터 읽기
  val is128Req: Output(Bool)                 // 128-bit 요청 신호
  val banked_data_resp: Input(Vec[L1BankedDataReadResult])  // 데이터 응답
  val read_error_delayed: Input(Vec[Bool])   // 읽기 에러 (1 cycle delayed)

  // Flag 업데이트
  val access_flag_write: DecoupledIO(FlagMetaWriteReq)    // Access flag 쓰기
  val prefetch_flag_write: DecoupledIO(SourceMetaWriteReq) // Prefetch flag 쓰기

  // Bank Conflict
  val bank_conflict_slow: Input(Bool)        // Bank conflict 감지

  // MissQueue
  val miss_req: DecoupledIO(MissReq)         // Miss 요청
  val miss_resp: Input(MissResp)             // Miss 응답

  // Replacement
  val replace_access: ValidIO(ReplacementAccessBundle)  // Replacement 접근
  val replace_way: ReplacementWayReqIO       // Replacement way 요청

  // 기타
  val disable_ld_fast_wakeup: Input(Bool)    // Fast wakeup 비활성화
  val error: Output(L1CacheErrorInfo)        // 에러 정보
  val mq_enq_cancel: Input(Bool)             // MissQueue enqueue 취소
}
```

### 1.2 LoadPipe 내부 신호 흐름

```
S0 Stage:
  - io.lsu.req.fire -> s0_valid
  - vaddr -> tag_read, meta_read
  - s0_bank_oh 계산

S1 Stage:
  - s1_tag_match_way = tag 비교 결과
  - s1_hit = tag_match && has_permission
  - s1_repl_way_en = replacement 선택
  - banked_data_read 요청

S2 Stage:
  - s2_hit 최종 판정
  - miss 시: miss_req 발행
  - replay 조건: bank_conflict || wpu_pred_fail || mq_nack

S3 Stage:
  - ECC 검사
  - data_delayed 반환
  - access_flag, prefetch_flag 업데이트
```

### 1.3 DCacheLoadIO 세부

```scala
class DCacheLoadIO {
  val req: DecoupledIO(DCacheWordReq)
  val resp: Flipped(DecoupledIO(DCacheWordResp))

  // Kill 신호
  val s1_kill: Output(Bool)    // TLB miss 등으로 S1에서 kill
  val s2_kill: Output(Bool)    // S2에서 kill (예: exception)

  // PC (디버그용)
  val s0_pc, s1_pc, s2_pc: Output(UInt)

  // 물리 주소 (S1에서 전달)
  val s1_paddr_dup_lsu: Output(UInt)      // LSU side
  val s1_paddr_dup_dcache: Output(UInt)   // DCache side

  // S2 상태
  val s2_hit: Input(Bool)
  val s2_first_hit: Input(Bool)
  val s2_bank_conflict: Input(Bool)
  val s2_wpu_pred_fail: Input(Bool)
  val s2_mq_nack: Input(Bool)

  // 디버그
  val debug_s1_hit_way: Input(UInt)
}
```

## 2. StorePipe 인터페이스

**파일 위치**: `src/main/scala/xiangshan/cache/dcache/storepipe/StorePipe.scala`

### 2.1 StorePipe IO Bundle
```scala
class StorePipe IO {
  // LSU 연결
  val lsu: Flipped(DCacheStoreIO)

  // Meta Array
  val meta_read: DecoupledIO(MetaReadReq)
  val meta_resp: Input(Vec[nWays, Meta])

  // Tag Array
  val tag_read: DecoupledIO(TagReadReq)
  val tag_resp: Input(Vec[nWays, UInt])

  // MissQueue (Prefetch)
  val miss_req: DecoupledIO(MissReq)       // Store miss -> prefetch write

  // Replacement
  val replace_access: ValidIO(ReplacementAccessBundle)
  val replace_way: ReplacementWayReqIO

  // Error
  val error: Output(L1CacheErrorInfo)
}
```

### 2.2 DCacheStoreIO 세부

```scala
class DCacheStoreIO {
  // 물리 주소 (STA S1에서)
  val s1_paddr: Output(UInt)

  // Kill 신호
  val s1_kill: Output(Bool)    // TLB miss 또는 Exception
  val s2_kill: Output(Bool)    // Access Fault 또는 MMIO

  // PC (디버그)
  val s2_pc: Output(UInt)

  // 요청/응답
  val req: DecoupledIO(DcacheStoreRequestIO)
  val resp: Flipped(DecoupledIO({
    val miss: Bool
    val replay: Bool
    val tag_error: Bool
  }))
}
```

### 2.3 StorePipe 역할

StorePipe는 **데이터 쓰기를 직접 수행하지 않음**:
1. STA (Store Address) 파이프라인에서 hit/miss 검사
2. Miss 시 prefetch write 요청 발행 (EnableStorePrefetchAtIssue가 켜진 경우)
3. 실제 데이터 쓰기는 SBuffer -> MainPipe 경로로 처리

## 3. MainPipe 인터페이스

**파일 위치**: `src/main/scala/xiangshan/cache/dcache/mainpipe/MainPipe.scala`

### 3.1 MainPipe IO Bundle

```scala
class MainPipe IO {
  // ProbeQueue
  val probe_req: Flipped(DecoupledIO(MainPipeReq))

  // MissQueue (Store miss)
  val miss_req: DecoupledIO(MissReq)
  val miss_resp: Input(MissResp)

  // SBuffer (Store)
  val store_req: Flipped(DecoupledIO(DCacheLineReq))
  val store_replay_resp: ValidIO(DCacheLineResp)
  val store_hit_resp: ValidIO(DCacheLineResp)
  val release_update: ValidIO(ReleaseUpdate)

  // Atomics
  val atomic_req: Flipped(DecoupledIO(MainPipeReq))
  val atomic_resp: ValidIO(AtomicsResp)

  // Replace (from MissQueue)
  val replace_req: Flipped(DecoupledIO(MainPipeReq))
  val replace_resp: ValidIO(UInt)

  // WritebackQueue
  val wb: DecoupledIO(WritebackReq)
  val wb_ready_dup: Vec[Input(Bool)]
  val probe_ttob_check_req: ValidIO(ProbeToBCheckReq)
  val probe_ttob_check_resp: Flipped(ValidIO(ProbeToBCheckResp))

  // Data SRAM
  val data_read: Vec[Input(Bool)]           // LoadPipe 읽기 중?
  val data_read_intend: Output(Bool)        // 읽기 의도
  val data_readline: DecoupledIO(L1BankedDataReadLineReq)
  val data_resp: Input(Vec[DCacheBanks, L1BankedDataReadResult])
  val data_write: DecoupledIO(L1BankedDataWriteReq)

  // Meta Array
  val meta_read: DecoupledIO(MetaReadReq)
  val meta_resp: Input(Vec[nWays, Meta])
  val meta_write: DecoupledIO(CohMetaWriteReq)
  val extra_meta_resp: Input(Vec[nWays, DCacheExtraMeta])
  val error_flag_write: DecoupledIO(FlagMetaWriteReq)
  val prefetch_flag_write: DecoupledIO(SourceMetaWriteReq)
  val access_flag_write: DecoupledIO(FlagMetaWriteReq)

  // Tag SRAM
  val tag_read: DecoupledIO(TagReadReq)
  val tag_resp: Input(Vec[nWays, UInt])
  val tag_write: DecoupledIO(TagWriteReq)
  val tag_write_intend: Output(Bool)

  // Replacement
  val replace_access: ValidIO(ReplacementAccessBundle)
  val replace_way: ReplacementWayReqIO

  // Status (for set conflict detection)
  val status: Bundle {
    val s0_set: ValidIO(UInt)
    val s1, s2, s3: ValidIO(MainPipeStatus)
  }

  // LRSC
  val lrsc_locked_block: Output(Valid(UInt))
  val invalid_resv_set: Input(Bool)
  val update_resv_set: Output(Bool)
  val block_lr: Output(Bool)

  // Error
  val error: Output(L1CacheErrorInfo)
  val force_write: Input(Bool)
}
```

### 3.2 MainPipeReq 구조

```scala
class MainPipeReq {
  val miss: Bool                    // AMO miss refill
  val miss_id: UInt                 // Miss entry ID
  val miss_param: UInt              // TL permission
  val miss_dirty: Bool
  val miss_way_en: UInt

  val probe: Bool                   // Probe 요청
  val probe_param: UInt
  val probe_need_data: Bool

  val source: UInt                  // LOAD/STORE/AMO_SOURCE
  val cmd: UInt                     // Memory op command
  val vaddr: UInt
  val addr: UInt                    // Physical addr (block aligned)

  val store_data: UInt              // Store data (blockBytes * 8)
  val store_mask: UInt              // Store mask (blockBytes)

  val word_idx: UInt                // AMO word index
  val amo_data: UInt
  val amo_mask: UInt

  val error: Bool

  val replace: Bool                 // Replace 요청
  val replace_way_en: UInt

  val id: UInt                      // Request ID
}
```

### 3.3 요청 중재 우선순위

```scala
arbiter(
  in = Seq(
    io.probe_req,      // 최고 우선순위: L2 Probe
    io.replace_req,    // 두 번째: MissQueue Replace
    store_req,         // 세 번째: SBuffer Store
    io.atomic_req      // 최저: Atomic
  ),
  out = req
)
```

## 4. MissQueue 인터페이스

**파일 위치**: `src/main/scala/xiangshan/cache/dcache/mainpipe/MissQueue.scala`

### 4.1 MissReq 구조

```scala
class MissReq {
  val source: UInt          // LOAD/STORE/AMO/PREFETCH_SOURCE
  val pf_source: UInt       // Prefetch source bits
  val cmd: UInt             // Memory op
  val addr: UInt            // Physical address
  val vaddr: UInt           // Virtual address
  val way_en: UInt          // Selected way
  val pc: UInt              // PC for debug

  val full_overwrite: Bool  // 전체 라인 덮어쓰기
  val word_idx: UInt        // AMO word index
  val amo_data: UInt
  val amo_mask: UInt

  val req_coh: ClientMetadata     // 요청 시 coherence 상태
  val replace_coh: ClientMetadata // 교체될 라인 상태
  val replace_tag: UInt           // 교체될 라인 tag
  val replace_pf: UInt            // 교체될 라인 prefetch flag

  val store_data: UInt      // Store data
  val store_mask: UInt      // Store mask

  val cancel: Bool          // 취소 신호 (pmp fail 등)
}
```

### 4.2 MissResp 구조

```scala
class MissResp {
  val id: UInt              // MSHR entry ID
  val handled: Bool         // 요청이 처리됨 (alloc 또는 merge)
  val merged: Bool          // 기존 entry에 merge됨
  val repl_way_en: UInt     // 선택된 replacement way
}
```

### 4.3 MissEntry 상태 머신

```
+--------+     +-----------+     +------------+     +---------+
| s_idle | --> | s_acquire | --> | w_grantfirst| --> |w_grantlast|
+--------+     +-----------+     +------------+     +---------+
                    |                                    |
                    |            +-------------+         |
                    +----------->| s_replace_req|<-------+
                                 +-------------+
                                       |
                                 +------------+
                                 | w_replace_resp|
                                 +------------+
                                       |
                                 +-----------+
                                 | s_refill  |
                                 +-----------+
                                       |
                                 +-----------+
                                 |w_refill_resp|
                                 +-----------+
                                       |
                                 +----------+
                                 |s_mainpipe| (AMO only)
                                 +----------+
```

## 5. RefillPipe 인터페이스

**파일 위치**: `src/main/scala/xiangshan/cache/dcache/mainpipe/RefillPipe.scala`

### 5.1 RefillPipeReq 구조

```scala
class RefillPipeReq {
  val source: UInt          // Source type
  val vaddr: UInt
  val addr: UInt            // Physical address
  val way_en: UInt          // Target way
  val alias: UInt           // Alias bits
  val miss_id: UInt         // MSHR entry ID
  val id: UInt              // Request ID
  val error: Bool           // Error flag
  val prefetch: UInt        // Prefetch source
  val access: Bool          // Access flag

  val wmask: UInt           // Write mask (per bank)
  val data: Vec[UInt]       // Write data (per bank)
  val meta: Meta            // New coherence meta
}
```

### 5.2 RefillPipe 동작

```scala
// 단일 사이클 파이프라인
// 입력: MissQueue에서 refill된 데이터
// 출력: Tag/Meta/Data Array 쓰기

io.data_write.valid := io.req_dup_for_data_w(0).valid
io.meta_write.valid := io.req_dup_for_meta_w.valid
io.tag_write.valid := io.req_dup_for_tag_w.valid
io.error_flag_write.valid := io.req_dup_for_err_w.valid
io.prefetch_flag_write.valid := io.req_dup_for_err_w.valid
io.access_flag_write.valid := io.req_dup_for_err_w.valid

// Store에 대한 응답
io.store_resp.valid := refill_w_valid && source === STORE_SOURCE.U
```

## 6. WritebackQueue 인터페이스

**파일 위치**: `src/main/scala/xiangshan/cache/dcache/mainpipe/WritebackQueue.scala`

### 6.1 WritebackReq 구조

```scala
class WritebackReq {
  val addr: UInt            // Physical address
  val param: UInt           // TL permission param
  val voluntary: Bool       // Voluntary release (vs ProbeAck)
  val hasData: Bool         // Data 포함 여부
  val dirty: Bool           // Dirty 여부
  val delay_release: Bool   // 지연된 release (refill 대기)
  val miss_id: UInt         // 관련 MSHR ID
  val data: UInt            // Data (blockBytes * 8)
}
```

### 6.2 WritebackEntry 상태 머신

```
+----------+     +---------+     +---------------+     +----------------+
| s_invalid| --> | s_sleep | --> | s_release_req | --> | s_release_resp |
+----------+     +---------+     +---------------+     +----------------+
                      |                  ^
                      |                  |
                 release_wakeup     TileLink C
                      |                  |
                      +------------------+
```

## 7. ProbeQueue 인터페이스

**파일 위치**: `src/main/scala/xiangshan/cache/dcache/mainpipe/Probe.scala`

### 7.1 ProbeReq 구조

```scala
class ProbeReq {
  val source: UInt          // TL source
  val opcode: UInt          // TL opcode
  val addr: UInt            // Physical address
  val vaddr: UInt           // Virtual address (L2에서 전달)
  val param: UInt           // Probe permission
  val needData: Bool        // Data 필요 여부
  val id: UInt              // Probe entry ID
}
```

### 7.2 ProbeEntry 상태 머신

```
+----------+     +------------+     +-------------+
| s_invalid| --> | s_pipe_req | --> | s_wait_resp |
+----------+     +------------+     +-------------+
     ^                                    |
     |                                    |
     +------------------------------------+
              pipe_resp
```
