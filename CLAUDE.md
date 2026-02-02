# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

XiangShan (香山) is an open-source high-performance RISC-V processor written in Chisel (Scala-based HDL). The current version on master is Kunminghu (昆明湖). Previous stable versions include Nanhu (南湖) and Yanqihu (雁栖湖).

## Build System

XiangShan uses Mill as its build system. Key commands:

```bash
# Initialize submodules (required first time)
make init

# Generate Verilog for FPGA (output: build/XSTop.v)
make verilog

# Generate Verilog for simulation (output: build/SimTop.v)
make sim-verilog

# Build simulator with Verilator
make emu CONFIG=DefaultConfig EMU_THREADS=2 -j10

# Run simulator
./build/emu -b 0 -e 0 -i ./ready-to-run/coremark-2-iteration.bin --diff ./ready-to-run/riscv64-nemu-interpreter-so

# IDE support
make bsp        # Mill BSP for Metals/VS Code
make idea       # IntelliJ IDEA project files

# Run all tests
mill -i xiangshan[chisel3].test.test

# Run specific test
mill -i xiangshan[chisel3].test.testOnly xiangshan.DecodeUnitTest

# Clean build artifacts
make clean
```

### Configuration Options

- `CONFIG`: Processor configuration (DefaultConfig, MinimalConfig, etc.)
- `NUM_CORES`: Number of cores (default: 1)
- `MFC`: Use MLIR-based FIRRTL compiler (0 or 1)
- `RELEASE`: Build release version without debug features

## Environment Variables

Required for simulation:
- `NEMU_HOME`: Path to NEMU reference model
- `NOOP_HOME`: Path to XiangShan project
- `AM_HOME`: Path to AM project

## Architecture Overview

### Core Pipeline (src/main/scala/xiangshan/)

The XSCore is a superscalar out-of-order RISC-V processor with three main components:

1. **Frontend** (`frontend/`): Instruction fetch and branch prediction
   - BPU (Branch Prediction Unit): TAGE, ITTAGE, RAS, FTB, SC predictors
   - IFU (Instruction Fetch Unit)
   - IBuffer: Instruction buffer between fetch and decode
   - FTQ (Fetch Target Queue): Manages prediction/execution decoupling

2. **Backend** (`backend/`): Out-of-order execution engine
   - Decode/Rename: Converts architectural to physical registers
   - Dispatch: Routes instructions to issue queues
   - Issue/Reservation Stations: Schedules ready instructions
   - Execution Units: ALU, MulDiv, FPU, Jump/CSR
   - ROB (Reorder Buffer): Maintains program order for commit
   - Register files: Integer and floating-point

3. **MemBlock** (`backend/MemBlock.scala`, `cache/`, `mem/`): Memory subsystem
   - Load/Store Units with LSQ (Load/Store Queue)
   - DCache: L1 data cache with MSHR
   - TLB: Translation lookaside buffers (ITLB, DTLB, L2TLB)
   - Store buffer, prefetchers

### Cache Hierarchy

- **L1 ICache** (`cache/icache/`): Instruction cache
- **L1 DCache** (`cache/dcache/`): Data cache with write-back
- **L2 Cache** (`coupledL2/`): Private L2, coupled with core
- **L3 Cache** (`huancun/`): Shared L3, HuanCun directory-based cache

### Key Submodules

- `rocket-chip/`: RISC-V infrastructure (diplomacy, TileLink, configs)
- `difftest/`: Co-simulation framework for verification against NEMU
- `fudian/`: Floating-point units
- `huancun/`: L2/L3 cache implementation
- `coupledL2/`: L2 cache coupled with processor
- `utility/`: Shared utility code

### Configuration System

Configs are defined in `src/main/scala/top/Configs.scala`:
- `DefaultConfig`: Full-featured configuration
- `MinimalConfig`: Reduced resources for faster simulation
- Parameters can be customized via `XSCoreParameters` in `Parameters.scala`

### Key Parameters (XSCoreParameters)

- `DecodeWidth`, `RenameWidth`, `CommitWidth`: Pipeline widths
- `FetchWidth`: Instructions fetched per cycle
- `RobSize`, `IssQueSize`: Buffer sizes
- `NRPhyRegs`: Physical register count
- `exuParameters`: Execution unit counts (ALU, MulDiv, FPU, Load, Store)

## Code Patterns

- Uses Chisel 3.6 (chisel3) or Chisel 6.x (chisel) via cross-compilation
- LazyModule pattern from Rocket Chip for parameterized hardware generation
- Diplomacy for TileLink interconnect negotiation
- Parameters passed implicitly via `Parameters` from CDE (Context-Dependent Environments)

## Testing

The difftest framework co-simulates XiangShan against NEMU (reference emulator) for verification. Tests are in `src/test/scala/`.
