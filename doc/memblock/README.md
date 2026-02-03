# XiangShan MemBlock Documentation

This directory contains comprehensive documentation for XiangShan's memory subsystem (MemBlock).

## Directory Structure

```
memblock/
├── README.md                          # This file - overview and navigation
│
├── loadpipe/                          # Load Pipeline Documentation
│   ├── INDEX.md                       # LoadPipe documentation index
│   ├── loadpipe_top_highlevel.md     # High-level overview
│   ├── loadpipe_top.md               # Complete top-level analysis
│   ├── loadpipe_S0.md                # Stage 0: Address generation
│   ├── loadpipe_S1.md                # Stage 1: TLB/Cache response
│   ├── loadpipe_S2.md                # Stage 2: Data selection
│   ├── loadpipe_S3.md                # Stage 3: Writeback
│   ├── replay.md                     # Load replay mechanism
│   ├── super_replay.md               # Super replay handling
│   ├── loadpipe_redirect.md          # Redirect handling
│   ├── z_detail_s1_nuke.md           # S1 nuke conditions
│   ├── z_detail_ldld_violation.md    # Load-load violations
│   ├── z_detail_cam_mismatch.md      # CAM mismatch
│   └── z_detail_rar.md               # RAR handling
│
├── storepipe/                         # Store Pipeline Documentation
│   ├── INDEX.md                       # StorePipe documentation index
│   ├── README.md                      # Main comprehensive analysis
│   ├── pipeline_timing.md             # Timing and logic details
│   └── component_interaction.md       # Component integration
│
├── memblock_1.md                      # General MemBlock documentation
└── z_detail_stld_violation.md         # Store-load violation (shared)
```

---

## Quick Start

### For Load Pipeline
**Start here**: [loadpipe/INDEX.md](./loadpipe/INDEX.md)

The load pipeline handles load instruction execution from issue to writeback, including:
- Virtual-to-physical address translation (TLB)
- Cache tag matching and data reading
- Store-to-load forwarding
- Load replay mechanism
- Violation detection (load-load, CAM mismatch)

**Quick navigation**:
- High-level overview: [loadpipe/loadpipe_top_highlevel.md](./loadpipe/loadpipe_top_highlevel.md)
- Complete analysis: [loadpipe/loadpipe_top.md](./loadpipe/loadpipe_top.md)
- Stage-by-stage: S0, S1, S2, S3 documents in loadpipe/

---

### For Store Pipeline
**Start here**: [storepipe/INDEX.md](./storepipe/INDEX.md)

The store pipeline handles store instruction execution from issue to writeback, including:
- Virtual-to-physical address translation (TLB)
- Cache hit/miss detection for prefetching
- Physical memory protection (PMP)
- Store-load violation detection
- MMIO operation handling

**Quick navigation**:
- Main documentation: [storepipe/README.md](./storepipe/README.md)
- Detailed timing: [storepipe/pipeline_timing.md](./storepipe/pipeline_timing.md)
- Component integration: [storepipe/component_interaction.md](./storepipe/component_interaction.md)

---

## MemBlock Overview

The MemBlock is XiangShan's memory subsystem responsible for:

### Core Components

1. **Load Pipeline** (LoadUnit)
   - 4-stage pipeline (S0-S3)
   - Handles load instruction execution
   - Integrates with Load Queue, TLB, DCache
   - Supports store-to-load forwarding
   - Complex replay and violation handling

2. **Store Pipeline** (StoreUnit)
   - 4+ stage pipeline (S0-S3-SX)
   - Handles store instruction execution
   - Integrates with Store Queue, TLB, DCache, PMP
   - Supports prefetch training
   - MMIO detection and handling

3. **Load-Store Queue (LSQ)**
   - Load Queue (LQ): Track in-flight loads
   - Store Queue (SQ): Track in-flight stores
   - Handles memory disambiguation
   - Manages store-to-load forwarding
   - Detects memory ordering violations

4. **Store Buffer (SBuffer)**
   - Buffers committed stores
   - Merges and coalesces stores
   - Drains to L1 DCache

5. **Memory Interface**
   - TLB (DTLB, L2 TLB, PTW)
   - L1 DCache
   - PMP (Physical Memory Protection)
   - Prefetch units

---

## Key Concepts

### Memory Ordering

**Load-Load Ordering**:
- Younger loads can execute speculatively
- Violations detected by comparing addresses
- Recovery via pipeline flush and replay

**Store-Load Ordering**:
- Loads must check for overlapping older stores
- Store-to-load forwarding for RAW dependencies
- Violations trigger pipeline replay

**Store-Store Ordering**:
- Enforced by Store Queue in program order
- Committed to memory in order via SBuffer

### Replay Mechanisms

**Fast Replay**:
- Quick replay from Load Queue
- For DCache misses, bank conflicts
- Lower latency path

**Slow Replay**:
- Replay from Reservation Station
- For TLB misses, serious violations
- Higher latency but handles complex cases

**Super Replay**:
- Advanced replay handling
- Optimizations for specific scenarios

### Violation Detection

**Load-Load (LDLD)**:
- Younger load executes before older load
- Detected in Load Queue
- Triggers pipeline flush

**Store-Load (STLD)**:
- Load misses forwarding from older store
- Detected when store address becomes known
- Triggers load replay or flush

**CAM Mismatch**:
- Virtual address CAM ≠ Physical address CAM
- Indicates aliasing or microarchitectural error
- Triggers exception or flush

---

## Pipeline Comparison

| Feature | Load Pipeline | Store Pipeline |
|---------|--------------|----------------|
| **Stages** | S0-S1-S2-S3 (4 stages) | S0-S1-S2-S3-SX (4+ stages) |
| **Latency** | 4 cycles (hit) | 6 cycles typical (hit) |
| **TLB Miss** | Replay from RS | Replay from RS |
| **DCache Miss** | Replay, wait MSHR | No stall (only prefetch hint) |
| **Forwarding** | Load-load, store-load | N/A |
| **Violations** | LDLD, CAM mismatch | STLD (query only) |
| **Queue** | Load Queue | Store Queue |
| **Writeback** | Data to register | Signal only (data to SBuffer) |
| **MMIO** | Handled at writeback | Detected and handled separately |

---

## Common Debugging Scenarios

### Load Not Returning Correct Data

**Check**:
1. Store-to-load forwarding (loadpipe/loadpipe_S2.md)
2. DCache data selection (loadpipe/loadpipe_S2.md)
3. Load Queue forwarding (loadpipe/loadpipe_S1.md)
4. CAM mismatch issues (loadpipe/z_detail_cam_mismatch.md)

### Load Stuck in Replay

**Check**:
1. Replay conditions (loadpipe/replay.md)
2. DCache miss handling (loadpipe/loadpipe_S2.md)
3. Bank conflict (loadpipe/z_detail_s1_nuke.md)
4. RAW dependencies (loadpipe/loadpipe_S2.md)

### Store Not Committing

**Check**:
1. Store Queue status (memblock_1.md)
2. ROB status (storepipe/README.md)
3. Exception status (storepipe/README.md - Error Handling)
4. MMIO handling (storepipe/README.md - S2 stage)

### Memory Ordering Violation

**Check**:
1. Load-load violations (loadpipe/z_detail_ldld_violation.md)
2. Store-load violations (z_detail_stld_violation.md)
3. Violation detection logic (loadpipe/loadpipe_S3.md)
4. Pipeline flush mechanism (loadpipe/loadpipe_redirect.md)

### TLB Issues

**Check**:
1. Load TLB request (loadpipe/loadpipe_S0.md)
2. Load TLB response (loadpipe/loadpipe_S1.md)
3. Store TLB request (storepipe/README.md - S0)
4. Store TLB response (storepipe/README.md - S1)
5. TLB miss replay (both pipelines)

---

## Documentation Conventions

### Diagram Styling

All mermaid diagrams use consistent color coding:

- **Memory/SRAM**: Green (#e8f5e9)
- **Registers**: Orange (#fff3e0)
- **Logic**: Blue (#e1f5fe)
- **I/O**: Pink (#fce4ec)

### File Naming

- `loadpipe_*.md`: Load pipeline main documentation
- `storepipe/*.md`: Store pipeline documentation
- `z_detail_*.md`: Detailed analysis of specific issues
- `INDEX.md`: Navigation and quick reference
- `README.md`: Overview and introduction

### Cross-References

Documents frequently cross-reference each other:
- Use relative paths (e.g., `./loadpipe/loadpipe_S0.md`)
- Reference specific sections when possible
- Maintain bidirectional references for related topics

---

## Related Documentation

### Frontend Pipeline
- Located in `../frontend/`
- Instruction fetch and branch prediction
- Interfaces with MemBlock for I-TLB and I-Cache

### Backend Pipeline
- Decode, Rename, Dispatch
- Reservation Stations
- ROB (Reorder Buffer)
- Interfaces with MemBlock for load/store execution

### Cache Hierarchy
- L1 ICache, L1 DCache
- L2 Cache (CoupledL2)
- L3 Cache (HuanCun)

---

## Statistics

### Load Pipeline Documentation
- **Files**: 13 documents
- **Total Size**: ~370 KB
- **Focus**: Load execution, replay, violations

### Store Pipeline Documentation
- **Files**: 4 documents
- **Total Size**: ~95 KB
- **Focus**: Store execution, MMIO, prefetch

### Total MemBlock Documentation
- **Files**: 19 documents
- **Total Size**: ~470 KB
- **Coverage**: Complete load/store pipeline analysis

---

## Contributing

When adding or updating documentation:

1. **Follow naming conventions**: Use consistent prefixes and suffixes
2. **Update indexes**: Modify INDEX.md files when adding documents
3. **Cross-reference**: Link related content in both directions
4. **Use diagrams**: Mermaid diagrams with consistent styling
5. **Add examples**: Include real scenarios and code snippets
6. **Update this README**: When adding major new sections

---

## Version History

| Date | Version | Changes |
|------|---------|---------|
| 2026-02-02 | 2.0 | Reorganized: Created loadpipe/ and storepipe/ subdirectories |
| 2026-02-02 | 1.0 | Initial documentation set |

---

*For questions or suggestions, refer to specific INDEX.md files in subdirectories*
