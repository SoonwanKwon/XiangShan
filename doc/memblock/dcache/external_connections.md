# DCache External Connections

LoadPipe, StorePipe, SBuffer와 DCache의 연결 관계를 상세히 설명합니다.

## 1. 전체 연결 아키텍처

```
                  MemBlock
+----------------------------------------------------------+
|                                                          |
|  +-------------+     +-------------+                     |
|  |  LoadUnit0  |     |  LoadUnit1  |                     |
|  +------+------+     +------+------+                     |
|         |                   |                            |
|         | DCacheLoadIO      | DCacheLoadIO               |
|         |                   |                            |
|  +------v------+     +------v------+                     |
|  |  LoadPipe0  |     |  LoadPipe1  |                     |
|  +------+------+     +------+------+                     |
|         |                   |                            |
|         +-------------------+                            |
|                   |                                      |
|                   v                                      |
|         +------------------+                             |
|         |   Tag/Meta/Data  |                             |
|         |      Arrays      |                             |
|         +------------------+                             |
|                   ^                                      |
|                   |                                      |
|         +--------+--------+                              |
|         |                 |                              |
|  +------+------+   +------v------+                       |
|  |  MainPipe   |   |  RefillPipe |                       |
|  +------+------+   +------+------+                       |
|         ^                 ^                              |
|         |                 |                              |
|    DCacheToSbufferIO  MissQueue                          |
|         |                 |                              |
|  +------+------+   +------+------+                       |
|  |   SBuffer   |   |  MissQueue  |                       |
|  +------+------+   +------+------+                       |
|         ^                 ^                              |
|         |                 |                              |
|  +------+------+   +------+------+                       |
|  | StoreQueue  |   | LoadPipe x2 |                       |
|  +------+------+   | MainPipe    |                       |
|         ^          | StorePipe x2|                       |
|         |          +-------------+                       |
|  +------+------+                                         |
|  |StoreUnit x2 |<---- DCacheStoreIO                      |
|  +-------------+          |                              |
|         ^          +------v------+                       |
|         |          | StorePipe x2|                       |
|         |          +-------------+                       |
+----------------------------------------------------------+
```

## 2. LoadPipe <-> LoadUnit 연결

### 2.1 연결 위치
```scala
// DCacheImp.scala (line 1056-1068)
for (w <- 0 until LoadPipelineWidth) {
  ldu(w).io.lsu <> io.lsu.load(w)
  ldu(w).io.load128Req := false.B
  ldu(w).io.nack := false.B
  ldu(w).io.disable_ld_fast_wakeup :=
    bankedDataArray.io.disable_ld_fast_wakeup(w)
}
```

### 2.2 DCacheLoadIO 신호 흐름

```
LoadUnit                             LoadPipe
--------                             --------

[S0 - 요청 단계]
req.valid ----------------------->  s0_valid
req.bits (vaddr, cmd, mask) ----->  s0_req
                <-----------------  req.ready

[S1 - 주소 변환 단계]
s1_paddr_dup_lsu --------------->   물리주소로 tag 비교
s1_paddr_dup_dcache ------------>   (동일, fanout 감소용)
s1_kill ------------------------>   TLB miss 시 kill
         <----------------------   s1_disable_fast_wakeup

[S2 - 히트/미스 판정]
s2_kill ------------------------>   Exception 등으로 kill
         <----------------------   s2_hit
         <----------------------   s2_first_hit
         <----------------------   s2_bank_conflict
         <----------------------   s2_wpu_pred_fail
         <----------------------   s2_mq_nack
         <----------------------   resp.valid
         <----------------------   resp.bits (data, miss, replay, ...)

[S3 - 최종 데이터]
         <----------------------   resp.bits.data_delayed (ECC 검사 후)
         <----------------------   resp.bits.error_delayed
```

### 2.3 핵심 응답 신호

| 신호 | 의미 | LoadUnit 동작 |
|------|------|---------------|
| `s2_hit` | 캐시 히트 | Writeback 가능 |
| `s2_bank_conflict` | Bank 충돌 | Replay 필요 |
| `s2_wpu_pred_fail` | Way 예측 실패 | Replay 필요 |
| `s2_mq_nack` | MSHR 가득 참 | Replay 필요 |
| `resp.bits.miss` | 캐시 미스 | MissQueue가 처리 중 |
| `resp.bits.replay` | Replay 필요 | ReplayQueue로 |

### 2.4 Miss 시 데이터 포워딩

```scala
// DCacheImp.scala (line 1023-1037)
// TileLink D 채널 포워딩
(0 until LoadPipelineWidth).map(i => {
  when(bus.d.bits.opcode === TLMessages.GrantData) {
    io.lsu.forward_D(i).apply(bus.d.valid, bus.d.bits.data, bus.d.bits.source, done)
  }
})

// MSHR 포워딩
(0 until LoadPipelineWidth).map(i =>
  io.lsu.forward_mshr(i).connect(missQueue.io.forward(i))
)
```

## 3. StorePipe <-> StoreUnit 연결

### 3.1 연결 위치
```scala
// DCacheImp.scala (line 1136-1138)
for (w <- 0 until StorePipelineWidth) {
  stu(w).io.lsu <> io.lsu.sta(w)
}
```

### 3.2 DCacheStoreIO 신호 흐름

```
StoreUnit (STA Pipe)               StorePipe
-------------------                ---------

[S0 - 요청]
req.valid ---------------------->  s0_valid
req.bits (vaddr, cmd) ---------->  s0_req
         <---------------------   req.ready

[S1 - 주소 변환]
s1_paddr ---------------------->  물리주소로 tag 비교
s1_kill ----------------------->  TLB miss 시 kill

[S2 - 응답]
s2_kill ----------------------->  Exception 등
s2_pc ------------------------->  디버그용
         <---------------------   resp.valid
         <---------------------   resp.bits.miss
         <---------------------   resp.bits.replay
```

### 3.3 StorePipe의 역할 (제한적)

StorePipe는 **STA (Store Address) 파이프라인**과 연동되어:
1. Hit/Miss 검사만 수행
2. **데이터 쓰기는 수행하지 않음**
3. Miss 시 Prefetch Write 요청 발행 (옵션)

```scala
// StorePipe.scala (line 177-183)
if(EnableStorePrefetchAtIssue) {
  // 모든 miss store가 MSHR로 prefetch 요청
  io.miss_req.valid := s2_valid && !s2_hit
} else {
  // prefetch 요청만 MSHR로
  io.miss_req.valid := s2_valid && !s2_hit && s2_is_prefetch
}
```

## 4. SBuffer <-> DCache 연결

### 4.1 핵심 인터페이스

```scala
// DCacheToSbufferIO
class DCacheToSbufferIO {
  val req: Flipped(Decoupled(DCacheLineReq))  // SBuffer -> MainPipe

  val main_pipe_hit_resp: ValidIO(DCacheLineResp)  // MainPipe hit 응답
  val refill_hit_resp: ValidIO(DCacheLineResp)     // Refill hit 응답
  val replay_resp: ValidIO(DCacheLineResp)         // Replay 응답
}
```

### 4.2 연결 위치

```scala
// DCacheImp.scala (line 1216-1219)
block_decoupled(io.lsu.store.req, mainPipe.io.store_req, refillPipe.io.req.valid)

io.lsu.store.replay_resp := RegNext(mainPipe.io.store_replay_resp)
io.lsu.store.main_pipe_hit_resp := mainPipe.io.store_hit_resp

// (line 1288)
io.lsu.store.refill_hit_resp := RegNext(refillPipe.io.store_resp)
```

### 4.3 SBuffer -> MainPipe 데이터 흐름

```
SBuffer                              MainPipe
-------                              --------

[요청 단계]
state: x_eviction ---------------->
dcache.req.valid -----------------> store_req.valid
dcache.req.bits:
  - addr (cacheline aligned) ----->
  - vaddr ------------------------>
  - data (512 bits) -------------->
  - mask (64 bytes) -------------->
  - id (sbuffer entry index) ----->
                <-----------------  store_req.ready

[응답 단계 - Hit]
                <-----------------  main_pipe_hit_resp.valid
                <-----------------  main_pipe_hit_resp.bits:
                                      - miss = false
                                      - replay = false
                                      - id

[응답 단계 - Refill Hit]
                <-----------------  refill_hit_resp.valid
                <-----------------  refill_hit_resp.bits

[응답 단계 - Replay 필요]
                <-----------------  replay_resp.valid
                <-----------------  replay_resp.bits:
                                      - replay = true
```

### 4.4 SBuffer 내부 처리 흐름

```scala
// Sbuffer.scala

// [1] SBuffer Entry 상태
class SbufferEntryState {
  val state_valid: Bool        // 유효한 엔트리
  val state_inflight: Bool     // DCache로 전송 중
  val w_timeout: Bool          // timeout 대기 중
  val w_sameblock_inflight: Bool  // 같은 블록이 inflight
}

// [2] DCache 요청 조건
val sbuffer_out_s0_valid = missqReplayHasTimeOut ||
  stateVec(idx).isDcacheReqCandidate() &&
  (need_drain || cohHasTimeOut || need_replace)

// [3] DCache 응답 처리
io.dcache.hit_resps.map(resp => {
  when (resp.fire) {
    stateVec(dcache_resp_id).state_inflight := false.B
    stateVec(dcache_resp_id).state_valid := false.B
  }
})
```

### 4.5 SBuffer 상태 머신

```
         +--------+
         | x_idle |
         +---+----+
             |
     +-------+-------+
     |               |
   flush         do_eviction
     |               |
     v               v
+----------+   +-----------+
|x_drain_all|   | x_replace |
+----+-----+   +-----+-----+
     |               |
     |    sbuffer    |
     |    empty      |
     |               |
     +-------+-------+
             |
             v
         +--------+
         | x_idle |
         +--------+
```

## 5. Store 전체 경로

### 5.1 Store 명령어 처리 흐름

```
[1] Dispatch Stage
    Store 명령어 -> StoreUnit에 할당

[2] Issue Stage
    StoreUnit -> STA Pipe (Address)
              -> STD Pipe (Data)

[3] Address Translation (STA Pipe)
    STA -> DTLB (vaddr -> paddr)
        -> StorePipe (DCache hit check)

[4] Store Queue
    STA 결과 + STD 결과 -> StoreQueue entry
    Commit 시 SBuffer로 이동

[5] SBuffer
    여러 Store를 cacheline 단위로 합침 (coalesce)
    조건 충족 시 DCache로 전송:
      - Threshold 초과
      - Timeout
      - Flush 요청

[6] DCache MainPipe
    SBuffer 요청 수신
    Tag 비교, Hit/Miss 판정
    Hit: Data 병합, 쓰기 완료
    Miss: MissQueue로 전송

[7] MissQueue (Miss 시)
    L2로 Acquire 발행
    GrantData 수신
    RefillPipe로 전달

[8] RefillPipe
    Tag/Meta/Data 갱신
    SBuffer에 응답 (refill_hit_resp)

[9] SBuffer Entry 해제
    응답 수신 시 entry 무효화
    새 Store 수용 가능
```

### 5.2 Block Diagram

```
StoreUnit
    |
    | (STA: vaddr)
    v
+---------+     +------------+
| DTLB    |---->| StorePipe  |
+---------+     |  (hit check|
    |           |   prefetch)|
    | paddr     +------------+
    v
+-----------+
| StoreQueue|
+-----------+
    | (commit)
    v
+---------+      DCacheToSbufferIO     +----------+
| SBuffer |<-------------------------->| MainPipe |
+---------+      (cacheline write)     +----------+
    |                                       |
    |                                  Miss |
    |                                       v
    |                                 +----------+
    |                                 | MissQueue|
    |                                 +-----+----+
    |                                       |
    |                                 TileLink A
    |                                       v
    |                                 +----------+
    |                                 |    L2    |
    |                                 +-----+----+
    |                                       |
    |                                 TileLink D
    |                                       v
    |                                 +----------+
    |<--------------------------------| RefillPipe|
    |        refill_hit_resp          +----------+
    v
Entry Release
```

## 6. Load/Store 충돌 처리

### 6.1 Bank Conflict

```scala
// DCacheImp.scala
// LoadPipe와 MainPipe가 같은 Bank 접근 시

bankedDataArray.io.read(i) <> ldu(i).io.banked_data_read
bankedDataArray.io.readline <> mainPipe.io.data_readline

// bank_conflict_slow 신호로 감지
ldu(i).io.bank_conflict_slow := bankedDataArray.io.bank_conflict_slow(i)
```

### 6.2 Set Conflict

```scala
// MainPipe 내부
// Store/Probe/Atomic이 같은 Set 접근 시 blocking

val s1_s0_set_conflict := s1_valid && s0_idx === s1_idx
val s2_s0_set_conflict := s2_valid && s0_idx === s2_idx
val s3_s0_set_conflict := s3_valid && s0_idx === s3_idx

val set_conflict = s1_s0_set_conflict || s2_s0_set_conflict || s3_s0_set_conflict
```

### 6.3 Store-to-Load Forwarding

DCache는 직접적인 Store-to-Load Forwarding을 제공하지 않음.
LSQ (LoadQueue/StoreQueue)에서 처리:

```scala
// SBuffer forward path
io.forward: Vec[LoadForwardQueryIO]

// SBuffer -> LoadUnit으로 forwarding
// vaddr 기반 매칭
```

## 7. Prefetch 연동

### 7.1 Store Prefetch

```scala
// StorePipe에서 miss 시 prefetch write 요청
io.miss_req.bits.source := DCACHE_PREFETCH_SOURCE.U
io.miss_req.bits.cmd := MemoryOpConstants.M_PFW
```

### 7.2 Load Prefetch

```scala
// LoadPipe에서 prefetch 처리
val total_prefetch = s2_valid && (s2_req.instrtype === DCACHE_PREFETCH_SOURCE.U)
val prefetch_hit = s2_valid && s2_hit && isFromL1Prefetch(s2_hit_prefetch)
```

## 8. Error Handling

### 8.1 ECC Error 경로

```scala
// LoadPipe S3
val s3_tag_error = RegEnable(s2_tag_error, s2_fire)
val s3_data_error = io.read_error_delayed && s3_hit
val s3_flag_error = RegEnable(s2_flag_error, s2_fire)
val s3_error = s3_tag_error || s3_flag_error || s3_data_error

// Error 보고
io.error.report_to_beu := (s3_tag_error || s3_data_error) && s3_valid
io.error.paddr := s3_paddr
io.error.source.tag := s3_tag_error
io.error.source.data := s3_data_error
io.error.source.l2 := s3_flag_error
```

### 8.2 Error 전파

```scala
// DCacheImp.scala
val errors = ldu.map(_.io.error) ++ Seq(mainPipe.io.error)
io.error <> RegNext(Mux1H(errors.map(e => RegNext(e.valid) -> RegNext(e))))
```
