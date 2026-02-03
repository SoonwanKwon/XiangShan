# StoreUnit Pipeline Documentation Index

This directory contains comprehensive analysis and documentation of the XiangShan StoreUnit pipeline implementation.

## Document Structure

### 1. [README.md](./README.md) - Main Documentation
**Primary comprehensive analysis document**

**Contents**:
- Overview and architecture
- Top-level interface description
- Complete data structure definitions
- Detailed pipeline stage analysis (S0, S1, S2, S3, SX)
- Key execution scenarios with sequence diagrams
- Implementation details and optimizations
- Performance considerations
- Debug and performance counters
- Configuration parameters
- Error handling and special cases

**Best for**: Understanding overall architecture, pipeline flow, and data structures

---

### 2. [pipeline_timing.md](./pipeline_timing.md) - Timing & Logic Details
**Detailed timing analysis and pseudo-code**

**Contents**:
- Complete pipeline timing diagrams
- Detailed stage-by-stage pseudo-code:
  - Address calculation algorithm
  - Mask generation logic
  - Alignment checking
  - Exception merging
  - Feedback generation
  - MMIO detection
  - Delay pipeline implementation
- Control flow state machines
- Critical path timing analysis
- Pipeline bubble analysis
- Data flow diagrams
- Performance metrics and calculations

**Best for**: Understanding detailed logic, timing constraints, and implementation algorithms

---

### 3. [component_interaction.md](./component_interaction.md) - Component Integration
**Inter-component communication and integration**

**Contents**:
- System context diagram
- Detailed interface analysis for each component:
  - Reservation Station (RS)
  - Translation Lookaside Buffer (TLB)
  - Data Cache (DCache)
  - Load-Store Queue (LSQ)
  - Physical Memory Protection (PMP)
  - Prefetch System
  - Reorder Buffer (ROB)
- Complete timing diagrams for component interactions
- Exception propagation through components
- Resource conflict handling
- Configuration-dependent behavior
- Complete interface reference table

**Best for**: Understanding how StoreUnit integrates with the rest of the system

---

## Quick Reference

### Pipeline Stages Summary

| Stage | Cycles | Key Operations |
|-------|--------|----------------|
| **S0** | 1 | Address generation, TLB/DCache request, mask generation |
| **S1** | 1 | TLB response, LSQ update, RS feedback, violation query |
| **S2** | 1 | PMP check, MMIO detection, prefetch training |
| **S3** | 1 | Writeback preparation |
| **SX** | 2 (typical) | Delay stages (varies by config) |
| **Total** | 6 (typical) | Complete latency for store writeback |

### Key Interfaces Quick Lookup

| Interface | File | Section |
|-----------|------|---------|
| RS (Issue/Feedback) | README.md | "Interface Signals" |
| TLB (Request/Response) | component_interaction.md | "2. TLB Interface" |
| DCache (Tag Lookup) | component_interaction.md | "3. DCache Interface" |
| LSQ (Update/Query) | component_interaction.md | "4. LSQ Interface" |
| PMP (Protection Check) | component_interaction.md | "5. PMP Interface" |
| ROB (Writeback) | component_interaction.md | "7. ROB Interface" |

### Key Data Structures Quick Lookup

| Structure | File | Location |
|-----------|------|----------|
| LsPipelineBundle | README.md | "Data Structures" section |
| ExuInput | README.md | "Data Structures" section |
| ExuOutput | README.md | "S3: Writeback Preparation" |
| RSFeedback | README.md | "Data Structures" section |
| StoreMaskBundle | README.md | "Data Structures" section |
| StoreNukeQueryIO | README.md | "Data Structures" section |

---

## Reading Guide

### For New Contributors

**Start with**: README.md
1. Read "Overview" and "Pipeline Architecture"
2. Study "Pipeline Overview Diagram"
3. Read through each stage (S0-SX) sequentially
4. Review "Scenario 1: Normal Store Hit Path"

**Then**: component_interaction.md
1. Study "System Context Diagram"
2. Focus on interfaces relevant to your work
3. Review timing diagrams for those interfaces

**Finally**: pipeline_timing.md
1. Deep dive into specific stages of interest
2. Study pseudo-code for algorithms you'll modify

### For Understanding Store Execution

**Read in order**:
1. README.md: "Scenario 1: Normal Store Hit Path"
2. pipeline_timing.md: "Normal Store Execution Timeline"
3. component_interaction.md: "Complete Store Execution Timeline"

### For Debugging Issues

**TLB-related issues**:
- README.md: "Stage 1 (S1): TLB Response & LSQ Update"
- component_interaction.md: "2. TLB Interface"
- pipeline_timing.md: "S1: TLB Response Processing"

**DCache-related issues**:
- README.md: "Stage 0 (S0): Address Generation" (DCache Request)
- component_interaction.md: "3. DCache Interface"

**Exception-related issues**:
- README.md: "Error Handling" section
- component_interaction.md: "Error and Exception Propagation"
- pipeline_timing.md: Exception merging logic

**Performance issues**:
- README.md: "Performance Considerations"
- pipeline_timing.md: "Performance Analysis"

---

## Code Navigation

### Key Source Files

| File | Description |
|------|-------------|
| `src/main/scala/xiangshan/mem/pipeline/StoreUnit.scala` | Main StoreUnit implementation |
| `src/main/scala/xiangshan/mem/MemCommon.scala` | Common memory data structures |
| `src/main/scala/xiangshan/Bundle.scala` | Core bundle definitions |
| `src/main/scala/xiangshan/cache/dcache/storepipe/StorePipe.scala` | DCache store pipe |

### Key Code Sections

| Feature | File:Line | Description |
|---------|-----------|-------------|
| Address Calculation | StoreUnit.scala:80-88 | Optimized address generation |
| Mask Generation | StoreUnit.scala:90 | Byte mask computation |
| TLB Request | StoreUnit.scala:92-104 | TLB interface |
| TLB Response | StoreUnit.scala:165-214 | Physical address handling |
| PMP Check | StoreUnit.scala:248-257 | PMP integration |
| Delay Pipeline | StoreUnit.scala:321-358 | Variable delay stages |

---

## Diagram Quick Reference

### Pipeline Flow
- **README.md**: "Pipeline Overview Diagram" - Overall pipeline structure
- **pipeline_timing.md**: "Normal Store Execution Timeline" - Gantt chart timing

### Scenarios
- **README.md**: Multiple sequence diagrams for different scenarios
  - Normal store hit
  - TLB miss with replay
  - MMIO operation
  - Address misalignment
  - Hardware prefetch

### Component Interactions
- **component_interaction.md**: Sequence diagrams for each interface
  - RS issue/feedback
  - TLB request/response
  - DCache interaction
  - LSQ update flow
  - PMP check flow
  - Prefetch training

### State Machines
- **pipeline_timing.md**: "Stage Valid State Transitions" - Stage valid bit FSM
- **pipeline_timing.md**: "Ready Signal Propagation" - Backpressure flow

### Data Flow
- **pipeline_timing.md**: "Complete Data Flow Through Pipeline" - End-to-end data flow

---

## Acronyms and Terminology

| Term | Full Name | Description |
|------|-----------|-------------|
| CBO | Cache Block Operation | cbo.clean, cbo.flush, cbo.inval, cbo.zero |
| DCache | Data Cache | L1 data cache |
| DTLB | Data TLB | Data Translation Lookaside Buffer |
| LSQ | Load-Store Queue | Combined load and store queue |
| MMIO | Memory-Mapped I/O | Memory-mapped device access |
| PMP | Physical Memory Protection | RISC-V physical memory protection |
| PTW | Page Table Walker | Hardware page table walker |
| RAW | Read After Write | Load-store dependency |
| ROB | Reorder Buffer | Out-of-order completion buffer |
| RS | Reservation Station | Issue queue for stores |
| SBuffer | Store Buffer | Buffer for committed stores |
| SMS | Spatial Memory Streaming | Prefetch algorithm |
| SQ | Store Queue | Store instruction queue (part of LSQ) |
| TLB | Translation Lookaside Buffer | Virtual-to-physical address cache |

---

## Version Information

**Documentation Version**: 1.0
**Date**: 2026-02-02
**XiangShan Version**: Kunminghu (master branch)
**Chisel Version**: 3.6 / 6.x (cross-compiled)

---

## Related Documentation

### In This Repository
- [LoadPipe Documentation](../loadpipe/) - Load instruction pipeline
- [LDLD Violation Documentation](../z_detail_ldld_violation.md) - Load-load violation detection
- [Frontend Analysis](../../frontend/) - Frontend pipeline documentation

### External References
- [RISC-V ISA Specification](https://riscv.org/technical/specifications/)
- [RISC-V Privileged Specification](https://riscv.org/technical/specifications/) - PMP, virtual memory
- [Chisel Documentation](https://www.chisel-lang.org/)

---

## Contributing

When updating this documentation:

1. **Maintain consistency** across all three documents
2. **Update diagrams** using mermaid syntax (keep styling)
3. **Add examples** for new features or edge cases
4. **Update revision history** in each modified file
5. **Cross-reference** between documents where appropriate

### Diagram Style Guide

All mermaid diagrams should use consistent styling:

```mermaid
%% styling
classDef memory fill:#e8f5e9,stroke:#1b5e20,stroke-width:1px;
classDef reg fill:#fff3e0,stroke:#e65100,stroke-width:2px;
classDef logic fill:#e1f5fe,stroke:#01579b,stroke-width:1px;
classDef io fill:#fce4ec,stroke:#880e4f,stroke-width:2px;
```

- **memory**: SRAM/memory structures (green)
- **reg**: Pipeline registers (orange)
- **logic**: Combinational logic (blue)
- **io**: I/O interfaces (pink)

---

## Feedback and Questions

For questions or suggestions about this documentation:
1. Check existing documentation thoroughly first
2. Review code comments in source files
3. Search for similar cases in other pipeline documentation
4. Raise issues with specific section references

---

## Document Statistics

| Document | Lines | Sections | Diagrams | Code Blocks |
|----------|-------|----------|----------|-------------|
| README.md | ~1100 | 25+ | 5+ | 30+ |
| pipeline_timing.md | ~800 | 15+ | 8+ | 25+ |
| component_interaction.md | ~900 | 20+ | 12+ | 20+ |
| **Total** | **~2800** | **60+** | **25+** | **75+** |

---

*Last Updated: 2026-02-02*
