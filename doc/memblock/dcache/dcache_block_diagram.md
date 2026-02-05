# DCache Block Diagram

## 1. DCache Top-Level Block Diagram

```
                                    +==========================================+
                                    |              DCacheWrapper               |
                                    +==========================================+
                                                       |
                                                       v
+========================================================================================+
|                                        DCache                                          |
|                                                                                        |
|  +-----------------------------------------------------------------------------------+ |
|  |                              DCacheIO (External Interface)                        | |
|  |  +-------------+  +-------------+  +-------------+  +------------+  +----------+ | |
|  |  | load[0..1]  |  | sta[0..1]   |  |   store     |  |  atomics   |  |  lsq     | | |
|  |  | DCacheLoadIO|  |DCacheStoreIO|  |DCacheToSbuf |  |AtomicWordIO|  | Refill   | | |
|  |  +------+------+  +------+------+  +------+------+  +-----+------+  +----+-----+ | |
|  +---------|-----------------|--------------|--------------|--------------|---------+ |
|            |                 |              |              |              |            |
|            v                 v              |              v              |            |
|  +---------+-------+ +-------+-------+      |     +--------+-------+      |            |
|  |    LoadPipe[0]  | |  StorePipe[0] |      |     |   MainPipe     |      |            |
|  +-----------------+ +---------------+      |     +----------------+      |            |
|  |    LoadPipe[1]  | |  StorePipe[1] |      |            ^               |            |
|  +--------+--------+ +-------+-------+      |            |               |            |
|           |                  |              +------------+               |            |
|           |                  |                           |               |            |
|           v                  v                           v               |            |
|  +========================================================================+            |
|  |                         Storage Arrays                                 |            |
|  |  +---------------+  +---------------+  +------------------+            |            |
|  |  | DuplicatedTag |  | L1CohMetaArray|  |  BankedDataArray |            |            |
|  |  |    Array      |  |               |  |    (8 Banks)     |            |            |
|  |  | (x3 for ports)|  +---------------+  +------------------+            |            |
|  |  +---------------+  +---------------+  +------------------+            |            |
|  |                     | L1FlagMetaArr |  | (ErrorArray,     |            |            |
|  |                     | (error,access)|  |  PrefetchArray,  |            |            |
|  |                     +---------------+  |  AccessArray)    |            |            |
|  |                     | L1PfSourceArr |  +------------------+            |            |
|  |                     +---------------+                                  |            |
|  +========================================================================+            |
|           |                  |              |              |               |            |
|           v                  v              v              v               |            |
|  +========================================================================+            |
|  |                        Control Modules                                 |            |
|  |  +--------------+  +--------------+  +--------------+  +------------+  |            |
|  |  |  MissQueue   |  |  ProbeQueue  |  | WritebackQ   |  |  Replacer  |  |            |
|  |  | (16 entries) |  | (4 entries)  |  | (8 entries)  |  |  (PLRU)    |  |            |
|  |  +------+-------+  +------+-------+  +------+-------+  +------------+  |            |
|  +=========|================|================|============================+            |
|            |                |                |                                          |
|            v                v                v                                          |
|  +---------+----------------+----------------+-----------+                              |
|  |              RefillPipe                               |                              |
|  +-------------------------------------------------------+                              |
|            |                                                                            |
|            v                                                                            |
|  +========================================================================+            |
|  |                        TileLink Interface                              |            |
|  |  +------------+  +------------+  +------------+  +------------+        |            |
|  |  | Channel A  |  | Channel B  |  | Channel C  |  | Channel D  |        |            |
|  |  | (Acquire)  |  |  (Probe)   |  | (Release)  |  |  (Grant)   |        |            |
|  |  +-----+------+  +-----+------+  +-----+------+  +-----+------+        |            |
|  +========|==============|==============|==============|==================+            |
+===========|==============|==============|==============|===========================+
            |              |              |              |
            v              ^              v              ^
     +------+------+-------+--------------+------+-------+------+
     |                           L2 Cache                       |
     +----------------------------------------------------------+
```

## 2. Pipeline Timing Diagram

### 2.1 LoadPipe Timing

```
Cycle:    0        1        2        3        4
         +--------+--------+--------+--------+--------+
  S0     |Tag/Meta|        |        |        |        |
         | Read   |        |        |        |        |
         +--------+--------+--------+--------+--------+
  S1              |Tag Cmp |        |        |        |
                  |Data Rd |        |        |        |
                  | paddr  |        |        |        |
         +--------+--------+--------+--------+--------+
  S2                       |Hit/Miss|        |        |
                           |Data Rsp|        |        |
                           |MissReq |        |        |
         +--------+--------+--------+--------+--------+
  S3                                |ECC Chk |        |
                                    |DataRet |        |
                                    |FlagUpd |        |
         +--------+--------+--------+--------+--------+

신호 타이밍:
- vaddr:     S0 시작
- paddr:     S1 시작 (TLB 결과)
- hit/miss:  S2 끝
- data:      S3 끝 (resp.bits.data_delayed)
```

### 2.2 MainPipe Timing (Store)

```
Cycle:    0        1        2        3        4
         +--------+--------+--------+--------+--------+
  S0     |Tag/Meta|        |        |        |        |
         | Read   |        |        |        |        |
         | Arb    |        |        |        |        |
         +--------+--------+--------+--------+--------+
  S1              |Tag Cmp |        |        |        |
                  |Data Rd |        |        |        |
                  |CohChk  |        |        |        |
         +--------+--------+--------+--------+--------+
  S2                       |Data Mrg|        |        |
                           |MissReq?|        |        |
                           |AMO Exec|        |        |
         +--------+--------+--------+--------+--------+
  S3                                |Meta Wr |        |
                                    |Tag Wr  |        |
                                    |Data Wr |        |
                                    |WB Req  |        |
         +--------+--------+--------+--------+--------+

요청 우선순위:
  Probe > Replace > Store > Atomic
```

### 2.3 Miss Handling Timing

```
Cycle:    0    1    2    3    4   ...   N   N+1  N+2  N+3
         +----+----+----+----+----+----+----+----+----+----+
LoadPipe |S0  |S1  |S2  |    |    |    |    |    |    |    |
         |    |    |Miss|    |    |    |    |    |    |    |
         +----+----+----+----+----+----+----+----+----+----+
MissQueue          |Enq |Acq |    |    |Grnt|    |    |    |
                   |    |    |<---L2 latency-->|    |    |
         +----+----+----+----+----+----+----+----+----+----+
RefillPipe                             |    |Ref |    |    |
                                       |    |    |    |    |
         +----+----+----+----+----+----+----+----+----+----+
LSQ Wakeup                             |    |    |Wake|    |
         +----+----+----+----+----+----+----+----+----+----+

- Acq: TileLink Acquire 발행
- Grnt: TileLink Grant 수신
- Ref: RefillPipe 캐시 갱신
- Wake: LoadQueue 재실행 가능
```

## 3. Data Flow Diagrams

### 3.1 Load Hit Path

```
         +-----------+
         | LoadUnit  |
         +-----+-----+
               | vaddr, cmd
               v
         +-----------+     +-----------+
         | LoadPipe  |<--->| Tag Array |
         |    S0     |     +-----------+
         +-----------+
               | tag read
               v
         +-----------+     +-----------+     +------------+
         | LoadPipe  |<--->| Meta Array|     | Data Array |
         |    S1     |     +-----------+     +-----+------+
         +-----------+                             |
               | paddr, tag match                  |
               v                                   v
         +-----------+     +---------------------------+
         | LoadPipe  |<----| Data Bank[0..7] Response  |
         |    S2     |     +---------------------------+
         +-----------+
               | hit, data valid
               v
         +-----------+
         | LoadPipe  |
         |    S3     |---> data to LoadUnit
         +-----------+
```

### 3.2 Load Miss Path

```
         +-----------+
         | LoadPipe  |
         |   S2 Miss |
         +-----+-----+
               |
               v
         +-----------+
         | MissQueue |
         |   Alloc   |
         +-----+-----+
               |
               v
    +----------+----------+
    |  TileLink Channel A |
    |     (Acquire)       |
    +----------+----------+
               |
               v
         +-----------+
         |    L2     |
         +-----------+
               |
               v
    +----------+----------+
    |  TileLink Channel D |
    |     (Grant)         |
    +----------+----------+
               |
      +--------+--------+
      |                 |
      v                 v
+-----------+    +-----------+
| MissQueue |    |  LSQ      |
| Data Buf  |    | Wakeup    |
+-----+-----+    +-----------+
      |
      v
+-----------+
| RefillPipe|
+-----+-----+
      |
      v
+-----+-----+-----+
| Tag | Meta| Data|
| Wr  | Wr  | Wr  |
+-----+-----+-----+
```

### 3.3 Store Path

```
         +-----------+
         | StoreUnit |
         |  (STA)    |
         +-----+-----+
               | vaddr
               v
         +-----------+     +-----------+
         | StorePipe |<--->| Tag/Meta  |
         |   S0-S2   |     |  Array    |
         +-----------+     +-----------+
               | hit/miss (prefetch only)
               |
               v (실제 데이터 경로)
         +-----------+
         | StoreQueue|
         +-----+-----+
               | commit
               v
         +-----------+
         |  SBuffer  |
         | (coalesce)|
         +-----+-----+
               | cacheline
               v
         +-----------+     +-----------+
         | MainPipe  |<--->| Tag/Meta  |
         |   S0-S1   |     |  Array    |
         +-----------+     +-----------+
               |
      +--------+--------+
      |                 |
      v                 v
  +-------+         +-------+
  | Hit   |         | Miss  |
  +---+---+         +---+---+
      |                 |
      v                 v
+-----------+     +-----------+
| MainPipe  |     | MissQueue |
|  S2-S3    |     |  Alloc    |
| Data Merge|     +-----------+
| Array Wr  |           |
+-----------+           v
      |           (... L2 ...)
      v                 |
+-----------+           v
| SBuffer   |     +-----------+
| Resp      |<----| RefillPipe|
+-----------+     +-----------+
```

## 4. Coherence State Diagram

### 4.1 MESI-like Protocol

```
                    +----------+
                    | Invalid  |
                    |   (I)    |
                    +----+-----+
                         |
         +---------------+---------------+
         |               |               |
    Acquire(S)      Acquire(E)      Acquire(M)
         |               |               |
         v               v               v
    +---------+     +---------+     +---------+
    | Shared  |     |Exclusive|     |Modified |
    |   (S)   |     |   (E)   |     |   (M)   |
    +----+----+     +----+----+     +----+----+
         |               |               |
         |          +----+----+          |
         |          |         |          |
         |    ProbeAck(S) ProbeAck(B)    |
         |          |         |          |
         |          v         v          |
         +--------->+---------+<---------+
                    | Branch  |
                    |  (B)    |
                    +---------+
```

### 4.2 Probe Handling

```
Probe Request (L2 -> L1):
  +--------+     +--------+
  | Probe  |---->| DCache |
  | toB    |     +---+----+
  +--------+         |
                     v
              +------+------+
              |             |
              v             v
         +--------+    +--------+
         | TtoB   |    | MtoB   |
         +---+----+    +---+----+
             |             |
             v             v
         ProbeAck     ProbeAckData
             |             |
             +------+------+
                    |
                    v
              +-----+-----+
              | WritebackQ|
              +-----+-----+
                    |
                    v
              +-----+-----+
              | TileLink C|
              +-----------+
```

## 5. Key Parameters Summary

| Parameter | Default | Description |
|-----------|---------|-------------|
| DCacheSets | 256 | Set 수 |
| DCacheWays | 8 | Way 수 |
| DCacheBanks | 8 | Data Bank 수 |
| BlockBytes | 64 | Cache Line 크기 |
| nMissEntries | 16 | MSHR 엔트리 수 |
| nProbeEntries | 4 | Probe Queue 엔트리 |
| nReleaseEntries | 18 | Release Queue 엔트리 |
| LoadPipelineWidth | 2 | Load 파이프라인 수 |
| StorePipelineWidth | 2 | Store 파이프라인 수 |
| StoreBufferSize | 16 | SBuffer 엔트리 수 |

## 6. Source File Locations

| Module | File Path |
|--------|-----------|
| DCache Top | `cache/dcache/DCacheWrapper.scala` |
| LoadPipe | `cache/dcache/loadpipe/LoadPipe.scala` |
| StorePipe | `cache/dcache/storepipe/StorePipe.scala` |
| MainPipe | `cache/dcache/mainpipe/MainPipe.scala` |
| MissQueue | `cache/dcache/mainpipe/MissQueue.scala` |
| RefillPipe | `cache/dcache/mainpipe/RefillPipe.scala` |
| ProbeQueue | `cache/dcache/mainpipe/Probe.scala` |
| WritebackQueue | `cache/dcache/mainpipe/WritebackQueue.scala` |
| BankedDataArray | `cache/dcache/data/BankedDataArray.scala` |
| TagArray | `cache/dcache/meta/TagArray.scala` |
| SBuffer | `mem/sbuffer/Sbuffer.scala` |
