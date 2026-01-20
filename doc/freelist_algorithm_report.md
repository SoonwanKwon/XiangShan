# FreeList (Int/FP) Algorithm and Hardware Notes

This document explains XiangShan’s register free lists from two perspectives:
1) **Algorithmic role** in rename/commit/recovery.
2) **Hardware structure** and key implementation choices.

XiangShan uses **two free lists**:
- **Integer FreeList**: `MEFreeList` for integer physical registers.
- **FP FreeList**: `StdFreeList` for floating‑point physical registers.

Both are instantiated in `Rename.scala` and operate per cycle at rename width.

---

## 1. Algorithm Perspective

### 1.1 What the FreeList does

The free list manages the pool of **available physical registers**.
- **Allocate**: provide new `pdest` when a decoded instruction has a destination.
- **Free**: return old `pdest` when an instruction commits (or when reclaiming after walk/redirect).

### 1.2 Rename‑time allocation

For each cycle (RenameWidth instructions):
1. Determine which instructions need a new destination (`allocateReq`).
2. If enough free registers exist, allocate unique physical registers in program order.
3. Provide `pdest` to rename and update speculative RAT.

### 1.3 Commit‑time free

When instructions commit:
- The old physical register (from arch RAT) is **returned** to the free list.
- This ensures the mapping space does not leak and keeps the free pool stable.

### 1.4 Redirect / Walk recovery

On redirect:
- The free list head pointer is **rolled back** to a snapshot or architectural head.
- This discards speculative allocations performed after the snapshot.

On walk (ROB walk for mispredict/exception recovery):
- The free list replays allocation bookkeeping to align with the architected state.

---

## 2. Hardware Implementation Overview

All free lists are implemented as a **circular queue** over a `Vec` of physical register IDs.
Key concepts:
- **headPtr**: points to next allocatable register.
- **tailPtr**: points to next slot to insert a freed register.
- **archHeadPtr**: tracks architectural head for recovery/consistency.
- **snapshot support**: on redirect, head pointer can be restored to a snapshot.

### 2.1 Shared Base Structure

`BaseFreeList` provides:
- Allocate/Free request ports.
- Head/tail pointers and pointer arithmetic.
- Snapshot restore logic (`SnapshotGenerator`).
- `canAllocate` gating based on free count.

### 2.2 StdFreeList (FP)

- Free list initially contains physical registers **32..(N‑1)**.
- Tail pointer updated using `freeReq` each cycle.
- `freeRegCnt` computed from circular distance `(tailPtr - headPtr)`.
- `canAllocate` is asserted when `freeRegCnt >= RenameWidth`.

### 2.3 MEFreeList (Integer)

- Free list includes **0 as a special entry** (x0 handling) and typically registers 1..(N‑1).
- Similar allocation logic, but **move elimination** is handled in integer rename.
- Additional consistency checks ensure all physical regs are in either RAT or free list.

---

## 3. Allocation Algorithm (Pseudo‑code)

```text
# Inputs: allocateReq[i] for i in 0..RenameWidth-1
# headPtr points to next available free reg

if canAllocate:
  for i in 0..RenameWidth-1:
    if allocateReq[i]:
      pdest[i] = freeList[headPtr + popcount(allocateReq[0..i-1])]

# advance head pointer by number of allocations
headPtr += popcount(allocateReq)
```

---

## 4. Free Algorithm (Commit)

```text
for each committing instruction:
  if freeReq[i]:
    freeList[tailPtr + popcount(freeReq[0..i-1])] = freePhyReg[i]

tailPtr += popcount(freeReq)
```

---

## 5. Recovery Behavior

- **Redirect**: head pointer restores to snapshot or arch state.
- **Walk**: allocate/frees are replayed to stay consistent with architectural commit order.

---

## 6. Key Files

- `src/main/scala/xiangshan/backend/rename/freelist/BaseFreeList.scala`
- `src/main/scala/xiangshan/backend/rename/freelist/StdFreeList.scala` (FP)
- `src/main/scala/xiangshan/backend/rename/freelist/MEFreeList.scala` (Int)
- `src/main/scala/xiangshan/backend/rename/Rename.scala`

