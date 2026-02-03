# LoadPipe Documentation Index

This directory contains comprehensive documentation of the XiangShan LoadPipe (Load Pipeline) implementation.

## Document Structure

### Overview Documents

#### [loadpipe_top_highlevel.md](./loadpipe_top_highlevel.md) - High-Level Overview (14 KB)
**Start here for a quick understanding**

High-level architecture and data flow of the load pipeline.

**Best for**: Getting a quick overview of the load pipeline architecture

---

#### [loadpipe_top.md](./loadpipe_top.md) - Complete Top-Level Analysis (53 KB)
**Comprehensive top-level documentation**

Detailed analysis of the complete load pipeline including all stages, interfaces, and control flow.

**Best for**: Understanding overall load pipeline design and integration

---

### Stage-by-Stage Documentation

#### [loadpipe_S0.md](./loadpipe_S0.md) - Stage 0 Analysis (14 KB)
**Address Generation and TLB/DCache Request**

Detailed analysis of S0 stage:
- Address calculation
- TLB request generation
- DCache request generation
- Load Queue allocation

---

#### [loadpipe_S1.md](./loadpipe_S1.md) - Stage 1 Analysis (21 KB)
**TLB Response and Cache Tag Matching**

Detailed analysis of S1 stage:
- TLB response processing
- DCache tag matching
- Bank conflict detection
- Load-load forwarding (fast path)
- S1 nuke conditions

---

#### [loadpipe_S2.md](./loadpipe_S2.md) - Stage 2 Analysis (27 KB)
**Data Selection and Store Forwarding**

Detailed analysis of S2 stage:
- DCache data read
- Store-to-load forwarding
- Data merging and selection
- RAW (Read-After-Write) handling
- Release violation detection

---

#### [loadpipe_S3.md](./loadpipe_S3.md) - Stage 3 Analysis (50 KB)
**Writeback and Exception Handling**

Detailed analysis of S3 stage:
- Load data writeback
- Exception processing
- Load Queue update
- Replay trigger conditions

---

### Replay and Violation Handling

#### [replay.md](./replay.md) - Load Replay Mechanism (29 KB)
**Load replay architecture and flow**

Analysis of load replay mechanism:
- Replay sources and triggers
- Replay queue management
- Fast replay vs. slow replay
- Replay scheduling

---

#### [super_replay.md](./super_replay.md) - Super Replay Mechanism (37 KB)
**Advanced replay handling**

Detailed analysis of super replay:
- Super replay conditions
- Load Queue management during replay
- Performance optimization

---

#### [loadpipe_redirect.md](./loadpipe_redirect.md) - Redirect Handling (32 KB)
**Pipeline flush and redirect**

Analysis of redirect handling in load pipeline:
- Redirect sources
- Pipeline flush mechanism
- Load Queue redirect handling
- State recovery

---

### Detailed Violation and Exception Analysis

#### [z_detail_s1_nuke.md](./z_detail_s1_nuke.md) - S1 Nuke Conditions (28 KB)
**S1 stage kill conditions**

Detailed analysis of S1 nuke (kill) conditions:
- Bank conflict
- TLB miss
- Load-load violations
- RAW hazards

---

#### [z_detail_ldld_violation.md](./z_detail_ldld_violation.md) - Load-Load Violations (36 KB)
**Load-load violation detection and handling**

Comprehensive analysis of load-load violations:
- Detection mechanism
- Address comparison
- Ordering enforcement
- Recovery procedures

---

#### [z_detail_cam_mismatch.md](./z_detail_cam_mismatch.md) - CAM Mismatch (26 KB)
**Content-Addressable Memory mismatch handling**

Analysis of CAM mismatch in store forwarding:
- Virtual vs. physical address CAM
- Mismatch detection
- Exception generation
- Pipeline recovery

---

#### [z_detail_rar.md](./z_detail_rar.md) - RAR (Read-After-Read) (6.1 KB)
**Read-after-read hazard handling**

Analysis of read-after-read hazards and handling mechanisms.

---

## Quick Reference

### Load Pipeline Stages Summary

| Stage | File | Key Operations |
|-------|------|----------------|
| **S0** | loadpipe_S0.md | Address gen, TLB/DCache request, LQ allocation |
| **S1** | loadpipe_S1.md | TLB resp, tag match, bank conflict, fast forward |
| **S2** | loadpipe_S2.md | Data read, store forward, data merge, RAW check |
| **S3** | loadpipe_S3.md | Writeback, exception, LQ update, replay trigger |

### Common Issues Quick Lookup

| Issue | Document | Section |
|-------|----------|---------|
| TLB Miss | loadpipe_S1.md | TLB response handling |
| Bank Conflict | loadpipe_S1.md, z_detail_s1_nuke.md | Bank conflict detection |
| Store Forwarding | loadpipe_S2.md | Store-to-load forwarding |
| CAM Mismatch | z_detail_cam_mismatch.md | Complete analysis |
| Load-Load Violation | z_detail_ldld_violation.md | Complete analysis |
| Replay Handling | replay.md, super_replay.md | Replay mechanisms |
| S1 Nuke | z_detail_s1_nuke.md | All S1 kill conditions |
| Redirect | loadpipe_redirect.md | Pipeline flush |

---

## Reading Guide

### For New Contributors

**Start with**:
1. loadpipe_top_highlevel.md - Get the big picture
2. loadpipe_top.md - Understand complete architecture
3. Read stage documents in order (S0 → S1 → S2 → S3)
4. Deep dive into specific issues as needed (z_detail_*.md)

### For Understanding Load Execution

**Read in order**:
1. loadpipe_top_highlevel.md - High-level flow
2. loadpipe_S0.md through loadpipe_S3.md - Stage by stage
3. loadpipe_S2.md - Focus on store forwarding
4. loadpipe_S3.md - Focus on writeback and exceptions

### For Debugging Specific Issues

**TLB-related**:
- loadpipe_S0.md: TLB request
- loadpipe_S1.md: TLB response and miss handling

**Cache-related**:
- loadpipe_S0.md: DCache request
- loadpipe_S1.md: Tag matching and bank conflicts
- loadpipe_S2.md: Data read and selection

**Forwarding-related**:
- loadpipe_S1.md: Fast forwarding (load-load)
- loadpipe_S2.md: Store-to-load forwarding
- z_detail_cam_mismatch.md: CAM mismatch issues

**Violation-related**:
- z_detail_ldld_violation.md: Load-load violations
- z_detail_s1_nuke.md: S1 stage violations
- loadpipe_S2.md: RAW violations

**Replay-related**:
- replay.md: Standard replay mechanism
- super_replay.md: Advanced replay handling
- loadpipe_S3.md: Replay trigger conditions

**Redirect-related**:
- loadpipe_redirect.md: Complete redirect analysis

---

## Document Organization

```
loadpipe/
├── INDEX.md                          # This file
│
├── Overview & Top-Level
│   ├── loadpipe_top_highlevel.md    # High-level overview
│   └── loadpipe_top.md               # Complete top-level analysis
│
├── Stage Documentation
│   ├── loadpipe_S0.md                # Stage 0: Address generation
│   ├── loadpipe_S1.md                # Stage 1: TLB/Cache response
│   ├── loadpipe_S2.md                # Stage 2: Data selection
│   └── loadpipe_S3.md                # Stage 3: Writeback
│
├── Replay & Redirect
│   ├── replay.md                     # Load replay mechanism
│   ├── super_replay.md               # Super replay
│   └── loadpipe_redirect.md          # Redirect handling
│
└── Detailed Analysis
    ├── z_detail_s1_nuke.md           # S1 nuke conditions
    ├── z_detail_ldld_violation.md    # Load-load violations
    ├── z_detail_cam_mismatch.md      # CAM mismatch
    └── z_detail_rar.md               # RAR handling
```

---

## Related Documentation

### In Parent Directory (memblock/)
- **z_detail_stld_violation.md** - Store-to-load violations (shared concern)
- **memblock_1.md** - General memblock documentation

### In Sibling Directory (storepipe/)
- Store pipeline documentation
- Store-load violation from store perspective

---

## Key Concepts

### Load Pipeline Flow
```
S0: Issue → Address Gen → TLB/DCache Req → LQ Alloc
↓
S1: TLB Resp → Tag Match → Bank Conflict Check → Fast Forward
↓
S2: Data Read → Store Forward → Data Merge → RAW Check
↓
S3: Data Select → Exception Check → Writeback → LQ Update
```

### Replay Triggers
- TLB miss (from S1)
- Bank conflict (from S1)
- Store forwarding data invalid (from S2)
- RAW hazard (from S2)
- DCache miss (from S2)
- Load-load violation (from LQ)
- CAM mismatch (from S2)

### Violation Types
- **Load-Load (LDLD)**: Younger load executes before older load
- **Store-Load (STLD)**: Load misses forwarding from older store
- **CAM Mismatch**: Virtual address CAM ≠ Physical address CAM
- **RAR**: Read-after-read hazard

---

## Acronyms

| Term | Full Name | Description |
|------|-----------|-------------|
| CAM | Content-Addressable Memory | Address matching structure |
| DCache | Data Cache | L1 data cache |
| DTLB | Data TLB | Data Translation Lookaside Buffer |
| LDLD | Load-Load | Load-load violation |
| LQ | Load Queue | Load instruction queue |
| MSHR | Miss Status Holding Register | Cache miss tracking |
| RAR | Read-After-Read | Read-after-read hazard |
| RAW | Read-After-Write | Load depends on store |
| STLD | Store-Load | Store-to-load violation |
| SQ | Store Queue | Store instruction queue |
| TLB | Translation Lookaside Buffer | Virtual-physical address cache |

---

## Contributing

When updating load pipeline documentation:

1. **Maintain consistency** across stage documents
2. **Update cross-references** when modifying interfaces
3. **Add examples** for complex scenarios
4. **Keep diagrams updated** using mermaid syntax
5. **Update this INDEX** when adding new documents

---

## Document Statistics

| Document | Size | Focus Area |
|----------|------|------------|
| loadpipe_top_highlevel.md | 14 KB | Overview |
| loadpipe_top.md | 53 KB | Complete top-level |
| loadpipe_S0.md | 14 KB | Stage 0 |
| loadpipe_S1.md | 21 KB | Stage 1 |
| loadpipe_S2.md | 27 KB | Stage 2 |
| loadpipe_S3.md | 50 KB | Stage 3 |
| replay.md | 29 KB | Replay mechanism |
| super_replay.md | 37 KB | Super replay |
| loadpipe_redirect.md | 32 KB | Redirect handling |
| z_detail_s1_nuke.md | 28 KB | S1 violations |
| z_detail_ldld_violation.md | 36 KB | LDLD violations |
| z_detail_cam_mismatch.md | 26 KB | CAM mismatch |
| z_detail_rar.md | 6.1 KB | RAR hazards |
| **Total** | **~370 KB** | **13 documents** |

---

*Last Updated: 2026-02-02*
