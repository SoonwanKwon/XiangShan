# XiangShan Frontend Analysis

This directory contains detailed analysis of the XiangShan frontend modules, following the guidelines from `commands/analysis_common.md`.

## Files

| File | Description |
|------|-------------|
| `phase1_frontend_analysis.md` | Top-level frontend architecture with detailed interface definitions, IFU pipeline diagram, and sequence diagrams |
| `phase2_bpu_analysis.md` | Deep dive into BPU submodules (FauFTB, FTB, TAGE-SC, ITTage, RAS) with detailed data structures and prediction logic |
| `phase2_icache_itlb_analysis.md` | ICache MainPipe pipeline, ITLB integration, miss handling, and prefetch support |
| `phase2_ibuffer_analysis.md` | IBuffer banked organization, enqueue/dequeue datapath, and TopDown integration |
| `tage_algorithm_analysis.md` | **Algorithm-focused analysis** of TAGE predictor: prediction/update algorithms, history management, SC integration, complexity analysis |

## Analysis Methodology

All analysis documents follow the guidelines in `/commands/analysis_common.md`:

1. **Data Types**: Memory and data types are shown with their structure (size and data type), with enough explanation to understand clearly
2. **Pipeline Diagrams**: Pipeline registers are clearly shown, logic is presented as pseudo code, and each connection is clearly noted
3. **Mermaid Styling**: Standard color scheme is used (green=memory, orange=registers, blue=logic, pink=io)

## Key Insights

### BPU Architecture
- **3-stage pipeline** (S0 input + S1/S2/S3 registered) with progressive refinement
- **FauFTB** provides fast S1 predictions (32-way fully-associative, ~1KB)
- **FTB** main predictor with differential target encoding (saves ~8 bits per slot)
- **TAGE-SC** with geometric history lengths (0, 5, 12, 26, 44, 67, ...)
- **ITTage** for indirect targets (stores full 39-bit addresses)

### IFU Pipeline
- **4-stage pipeline** (F0 input + F1/F2/F3 registered + WB)
- Parallel ICache and ITLB access in F0
- Predecode in F2 with verification in F3
- Multiple flush sources: Backend, IFU WB, BPU S2/S3

### ICache
- **2-port design** for cross-cacheline accesses
- **3-stage pipeline** (S0 input + S1/S2 registered)
- Integrated with prefetch buffer (IPF) and queue (PIQ)
- 256 sets x 4 ways, 64-byte lines (total 64KB)

### IBuffer
- **Banked FIFO** (6 banks x 8 entries = 48 total)
- **2-stage dequeue mux** for area efficiency (8→1 then 6→1 vs 48→1)
- Each entry ~128 bits (16 bytes)
- Total storage: 768 bytes

## Data Type Sizes Summary

| Component | Key Structure | Size |
|-----------|---------------|------|
| FTBEntry | Full entry | ~60 bits |
| TageEntry | Per table | ~11-15 bits |
| ITTageEntry | With target | ~50+ bits |
| RASEntry | Addr + counter | 39 + 8 = 47 bits |
| IBufEntry | Full entry | ~128 bits |
| FetchRequestBundle | From FTQ | ~120 bits |
| ICacheMainPipeResp | To IFU | ~650 bits (2x512b data) |

## Mermaid Diagram Conventions

All diagrams use consistent styling:
- 🟢 **Green (memory)**: SRAM arrays, TLBs, register files
- 🟠 **Orange (reg)**: Pipeline registers between stages
- 🔵 **Blue (logic)**: Combinational logic, state machines
- 🔴 **Pink (io)**: I/O interfaces, external modules

Pipeline registers are explicitly marked as:
```
[["Pipeline Reg S0→S1<br/>field1: size<br/>field2: size"]]
```

## Usage

These documents are designed for:
1. **Onboarding**: Understanding frontend architecture quickly
2. **Debugging**: Locating specific data structures and their sizes
3. **Optimization**: Identifying storage costs and critical paths
4. **Verification**: Checking signal widths and pipeline stages

## Related Files

- Source code: `src/main/scala/xiangshan/frontend/`
- Build config: `src/main/scala/xiangshan/Parameters.scala`
- Top-level: `src/main/scala/xiangshan/XSCore.scala`
