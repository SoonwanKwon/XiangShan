# Memory Disambiguation Analysis (XiangShan)

This note summarizes **where** memory disambiguation checks occur, **what triggers** them, and **what happens** on pass/fail paths. It is based on `doc/memblock/memory_disambiguation.md` and code under `src/main/scala/xiangshan/mem/`.

---

## 1) Check Stages and Trigger Events

| stage_name | trigger event | check performed |
|---|---|---|
| Load S1 | Load enters S1 (valid load) | Store-to-load forwarding **vaddr CAM** query to SQ/Sbuffer; collect `forwardMaskFast` and potential early invalids. |
| Load S2 | Load enters S2 with paddr | Store-to-load forwarding **paddr CAM** query to SQ/Sbuffer; merge with D-channel/MSHR forwarding; compute full-forward vs replay. |
| Store S1 | Store address ready (TLB hit, not HW prefetch) | Broadcast `StoreNukeQueryIO` (robIdx, paddr, mask) to all load pipes for early st-ld violation check. |
| Load S1/S2 | Receive `stld_nuke_query` from Store S1 | Compare age (`robIdx`), `paddr[...:3]`, and `mask` overlap; set nuke/replay if match. |
| Load S2 | `io.lsq.stld_nuke_query.req` fired | Enqueue/check in **LoadQueueRAW** for unresolved older-store addresses (memory ambiguity tracking). |
| Store S3 (store complete) | Store completion into LSQ/RAW | RAW queue CAM query to detect younger loads that read stale data; generate redirect on violation. |

---

## 2) Pass Cases (Load Proceeds After Disambiguation)

There are two categories: **definitely no violation** vs **speculative issue (unknown yet)**.

| category | case | condition in code/logic | effect |
|---|---|---|---|
| No violation (definite) | No matching older store in SQ/Sbuffer | vaddr/paddr CAM has no overlap and no `addrInvalid/dataInvalid/matchInvalid` | Load completes normally with DCache or forwarded data. |
| No violation (definite) | Full forwarding success | All requested bytes are forwarded (`s2_full_fwd` true) | Load completes with forwarded data, no replay. |
| No violation (definite) | Nuke check misses | `stld_nuke_query` arrives but age/addr/mask do not match | Load continues without replay. |
| Speculative issue | Older store address not ready | Load enters RAW queue when older store addr is unknown (`hasAddrInvalidStore`) | Load may complete now, but is tracked for later RAW violation check. |
| Speculative issue | Store addr not ready, no nuke yet | Store hasn’t broadcasted nuke (addr not ready), so load proceeds | Load is speculative; later store completion may trigger RAW redirect. |

---

## 3) Fail Cases (Replay or Redirect)

| failure condition | detection point | recovery | notes |
|---|---|---|---|
| Store data not ready for required forward | Forwarding response (`dataInvalid`) | **Replay** (LoadQueueReplay cause `C_FF`) | Definite dependency, but data unavailable. |
| Store address required but not ready | Forwarding response (`addrInvalid`) | **Replay** (cause `C_MA`) | Definite dependency, but addr unavailable. |
| VA/PA CAM mismatch (synonym/homonym) | Forwarding response (`matchInvalid`) | **Redirect/flush** | Micro-architectural exception to restore ordering. |
| Store-load violation detected by nuke | Load S1/S2 (`s1_nuke` / `s2_nuke`) | **Replay** (cause `C_NK`) | Store addr ready; younger load must replay. |
| RAW queue reports violation | Store completion RAW CAM match | **Redirect/flush** | Final safety net; pipeline rollback to violating load. |
| RAW/RAR query not accepted (queue full or not ready) | Load S2 query nack | **Replay** (cause `C_RAW` / `C_RAR`) | Disambiguation resources unavailable. |

---

## References

- `doc/memblock/memory_disambiguation.md`
- `src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala`
- `src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala`
- `src/main/scala/xiangshan/mem/lsqueue/LoadQueueRAW.scala`
- `src/main/scala/xiangshan/mem/lsqueue/LoadQueueReplay.scala`
- `src/main/scala/xiangshan/mem/MemCommon.scala`
