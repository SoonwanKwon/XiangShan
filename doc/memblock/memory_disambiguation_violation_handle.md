# Memory Disambiguation Violation Handling

This note summarizes **all cases where memory disambiguation violations are resolved by replay or redirect** in XiangShan.

---

## 1. Store → Load (RAW) Violations

### 1.1 Early Detection: Store S1 `s1_nuke`

- **Detection point**: StorePipe S1, when store address becomes ready
- **Mechanism**: Store broadcasts address (`s1_nuke`) and LoadPipe compares against in-flight loads
- **Action**: **Replay** the load (nuke/replay)
- **Reason**: Load executed before older store address was known; conflict discovered once store address resolves

**Result**: Load is replayed (typically `C_NK` / nuke replay path)

---

### 1.2 Late Detection: RAW Queue (Store S3 complete)

- **Detection point**: StorePipe S3 (store complete)
- **Mechanism**: Store performs **CAM search** in LoadQueueRAW
- **Action**: **Redirect / pipeline flush**
- **Reason**: Load already consumed stale data; correctness requires rollback

**Result**: Pipeline redirect (`io.rollback`) and re-execution

---

## 2. Load → Load (RAR) Violations

### 2.1 RAR Queue (Line Release / RVWMO ordering)

- **Detection point**: LoadQueueRAR when cache line is released/modified
- **Mechanism**: LoadQueueRAR compares younger/older loads for ordering violations
- **Action**: **Replay from fetch** (`rep_frm_fetch`)
- **Reason**: Younger load observed data before older load, violating RVWMO ordering guarantees

**Result**: Older load is replayed (fetch replay)

---

## Summary Table

| Violation Type | Detection Point | Trigger | Action |
|---|---|---|---|
| RAW (Store→Load) | Store S1 (`s1_nuke`) | Store addr ready → conflict | **Replay** |
| RAW (Store→Load) | Store S3 (LoadQueueRAW CAM) | Store complete → stale data | **Redirect** |
| RAR (Load→Load) | LoadQueueRAR | Line release/ordering | **Replay** |

---

## Notes

- **StoreSet (SSIT/LFST)** only controls **pre-issue waiting**, not these post-issue violation handlers.
- **RAW Queue and RAR Queue** are the **final safety nets** for correctness when speculative execution violates ordering.

