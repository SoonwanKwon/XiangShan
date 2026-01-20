# XiangShan Frontend (FE) Module Analysis — Work Instructions

## Goal
Analyze XiangShan Frontend (FE) modules in two phases:
- **Phase 1:** Understand **top-level structure**, **module connections**, and **representative end-to-end behaviors**.
- **Phase 2:** Perform **deep functional analysis** for each submodule with code-level evidence.

The output must be written so that a hardware/architecture engineer can validate it by tracing signal paths and reading code.

Create reports as md file locatig those at `./analysis/frontend` folder 

---

## Repository & Scope
- Target directory (expected): `src/main/scala/xiangshan/frontend/` (+ any dependent folders referenced)
- Focus: **Frontend pipeline and its predictor/ICache/fetch/IBuffer integration**
- Exclude: Backend, D-cache, rename/ROB, etc. (unless FE depends on them).
           
---

## Required Deliverables (Phase 1)
### 1) FE Top-Level Connectivity Diagram (Mermaid)
Create a **Mermaid flowchart** describing:
- FE pipeline stages (e.g., S0/S1/S2/S3... as used in XiangShan)
- Major blocks (BPU components, ICache/ITLB interface, Fetch Queue/IBuffer, redirect/flush path)
- Key buses/signals categories (PC, redirect, prediction meta, fetch bundle, exception, stall/ready/valid)
- Backpressure paths (ready/valid) must be visible

**Constraints**
- Use **subgraphs** for stages and for major subsystems (BPU, ICache path, queue/buffer)
- Every arrow must be labeled with at least a short semantic meaning (e.g., `pc`, `redirect`, `resp`, `meta`)
- Show where “control redirects” originate and where they terminate
- If a module name is uncertain, label as `UNKNOWN(<best guess>)` and list it in “Open Questions” section

### 2) Representative Behavior Sequence Diagram (Mermaid)
Provide at least **3 sequence diagrams** (separate Mermaid blocks) describing typical scenarios:
1. **Normal sequential fetch** (no redirect, ICache hit)
2. **Branch predicted taken** (BPU provides target; fetch switches path)
3. **Redirect/flush scenario** (mispredict or exception/flush causing redirect)

**Constraints**
- Include actors: `IFU/FE`, `BPU`, `uBTB/mBTB (if separate)`, `TAGE/SC (if present)`, `ICache`, `ITLB`, `IBuffer/FQ`, `Backend redirect source`
- To verify the actors, include list of components
- Show handshake timing (valid/ready) at least conceptually
- Include what metadata is produced/consumed (e.g., prediction meta, fetch packet info)

### 3) Phase 1 Written Summary
A concise but technical summary covering:
- Stage-by-stage responsibilities
- Who computes/updates PC each stage
- Where prediction happens (which stage) and what is available at each stage
- How compressed instructions (RVC) are handled in fetch alignment (if applicable)
- Backpressure: what stalls FE and how it propagates
- Known assumptions + open questions (with file/line references when possible)

**Evidence requirement (even in Phase 1):**
- Every major claim should include at least one pointer: `file path + symbol name` (and line numbers if accessible).

---

## Required Deliverables (Phase 2)
For each FE submodule, produce:
- Purpose & inputs/outputs (interfaces)
- Internal pipeline state (regs) and update conditions
- Critical algorithms (pseudo-code is OK, but must map to code)
- Corner cases (redirect during refill, cross-cacheline, RVC alignment, etc.)
- Performance/latency notes (what adds cycles, what is 0-cycle in same stage)

**Phase 2 must include a section for these (if present in code):**
- uBTB / mBTB / uFTB / FTB
- TAGE / SC / BIM / RAS
- FetchQueue / IBuffer / instruction alignment / predecode
- ICache request/response gating
- ITLB interaction & exception generation
- Redirect/flush arbitration logic

---

## Work Method (How to Proceed)
1. Start at FE top-level entry (likely `Frontend.scala` or a `LazyModule` wrapper).
2. Enumerate instantiated submodules and their wiring.
3. Build Phase 1 diagrams from actual connections.
4. Choose 3 representative scenarios and trace signals across stages for sequence diagrams.
5. Only after Phase 1 is complete, deep dive each submodule for Phase 2.

---

## Output Format Rules
- Use Markdown headings (H2/H3) and keep it skimmable.
- Mermaid blocks must be valid and renderable.
- Do not invent signals; if unsure, mark as `UNKNOWN` and justify uncertainty.
- Prefer quoting exact identifiers (class/val names) instead of paraphrase.
- If multiple versions exist across branches, explicitly state which one you analyzed.

---

## Acceptance Checklist (Phase 1)
- [ ] Top-level Mermaid flowchart shows stages + subsystems + redirect/backpressure paths
- [ ] At least 3 Mermaid sequence diagrams for the scenarios listed
- [ ] Stage-by-stage written explanation exists
- [ ] Each major claim has file/symbol pointers
- [ ] Open questions are clearly listed

---

## Notes / Context
- Goal is to match XiangShan’s real FE implementation, not a generic FE.
- Pay special attention to:
  - PC generation and cut/align logic (cross-cacheline + RVC)
  - Prediction timing differences between early stage (uBTB) and later stage (mBTB/TAGE)
  - What metadata is carried and where it is consumed
