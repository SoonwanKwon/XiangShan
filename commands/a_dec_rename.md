# Decode/Rename Stage Analysis Guide

## Methodology

This analysis follows the structured approach from `analysis_common.md`:

### Phase 1: Top-Level Analysis
1. **Module Exploration**: Identify all decode/rename modules and their connections
2. **Data Flow Mapping**: Trace data structures through the pipeline (CtrlFlow → CfCtrl → MicroOp)
3. **Connectivity Diagram**: Create Mermaid flowchart showing all major components
4. **Behavior Scenarios**: Document 3+ representative execution scenarios with sequence diagrams
5. **Evidence-Based Writing**: All claims backed by file:line references

### Phase 2: Deep Dive (Completed)
- FreeList implementation details  ✓
- Snapshot mechanism for fast recovery ✓
- Fusion decoder detailed logic (26 fusion patterns) ✓
- Timing analysis and critical paths ✓
- Corner case handling and error recovery ✓

## Key Modules Analyzed

### Decode Stage
- **File**: `src/main/scala/xiangshan/backend/decode/DecodeStage.scala`
- **Purpose**: Convert RISC-V instructions to internal control signals
- **Width**: DecodeWidth (6 instructions/cycle)
- **Key Features**: Parallel decode units, speculative RAT reads

### Rename Stage
- **File**: `src/main/scala/xiangshan/backend/rename/Rename.scala`
- **Purpose**: Allocate physical registers, maintain register mappings
- **Width**: RenameWidth (6 instructions/cycle)
- **Key Features**: Dual freelists, bypass network, move elimination

### Rename Table
- **File**: `src/main/scala/xiangshan/backend/rename/RenameTable.scala`
- **Purpose**: Maintain spec and arch register mappings
- **Structure**: 32 entries × 2 tables (int/FP)
- **Key Features**: Pipelined reads, bypass logic, snapshot support

## Output Locations

- **Phase 1 Report**: `/home/kswan7004/XiangShan/analysis/frontend/phase1_decode_rename_analysis.md`
  - Top-level connectivity, data flows, behavior scenarios

- **Phase 2 Deep Dive**: `/home/kswan7004/XiangShan/analysis/frontend/phase2_decode_rename_deepdive.md`
  - FreeList algorithms (MEFreeList with one-hot optimization, StdFreeList with RegNext)
  - Snapshot mechanism (8 checkpoints, 1-2 cycle recovery)
  - Fusion decoder (26 patterns: shift-add, zero-extend, sign-extend, custom micro-ops)
  - Timing analysis (critical paths: bypass network ~600ps, total ~1.1ns @ 28nm)
  - Corner cases and validation

## Diagram Guidelines

Follow `analysis_common.md` styling:
- Memory structures: Green (#e8f5e9)
- Registers/Pipeline: Orange (#fff3e0)
- Logic blocks: Blue (#e1f5fe)
- I/O: Pink (#fce4ec)

All diagrams use Mermaid for renderability.

## Validation Checklist

### Phase 1 ✓
- [x] Top-level connectivity diagram with all major components
- [x] 3+ representative behavior sequence diagrams
- [x] Data structure flow documented
- [x] All claims have file:line references
- [x] Stage responsibilities clearly defined
- [x] Key algorithms explained with code snippets
- [x] Performance characteristics documented
- [x] Corner cases identified

### Phase 2 ✓
- [x] FreeList implementation fully analyzed
  - [x] MEFreeList: One-hot optimization, multi-enqueue, move elimination
  - [x] StdFreeList: RegNext timing, FP-specific behavior
  - [x] Comparison table and tradeoffs
- [x] Snapshot mechanism fully documented
  - [x] SnapshotGenerator implementation
  - [x] Creation policy (rate limiting)
  - [x] Recovery modes (snapshot vs walk vs arch)
- [x] Fusion decoder complete analysis
  - [x] All 26 fusion patterns documented
  - [x] Example cases with code references
  - [x] Pipeline timing (T0 detect, T1 replace)
  - [x] Integration with Rename stage
- [x] Timing analysis
  - [x] Critical path identification (bypass network)
  - [x] Per-stage timing breakdown
  - [x] Frequency targets
- [x] Corner cases and validation
  - [x] Error scenarios (freelist exhaustion, snapshot full)
  - [x] Assertions and invariants
  - [x] Performance counters
