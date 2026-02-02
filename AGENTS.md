# Repository Guidelines

## Project Structure & Module Organization
The main hardware sources live under `src/main/scala/` (core logic in `xiangshan/`, SoC wrappers in `system/`, and top-levels in `top/`). Tests are in `src/test/scala/`. Major submodules live at the repo root: `difftest/` (cosim framework), `fudian/` (FPU), `huancun/` (L2/L3 cache), `coupledL2/` (private L2), `rocket-chip/` (RISC-V infra), and `utility/` (shared helpers). Prebuilt images for sims are under `ready-to-run/`.

## Build, Test, and Development Commands
- `make init`: initialize git submodules (required for first-time setup).
- `make verilog`: generate FPGA Verilog at `build/XSTop.v`.
- `make sim-verilog`: generate simulation Verilog at `build/SimTop.v`.
- `make emu CONFIG=DefaultConfig EMU_THREADS=2 -j10`: build Verilator-based emulator.
- `./build/emu -i ./ready-to-run/coremark-2-iteration.bin --diff ./ready-to-run/riscv64-nemu-interpreter-so`: run a binary with difftest.
- `make bsp` / `make idea`: generate IDE project files for Metals/VS Code or IntelliJ.
- `make test` or `make test-DecodeUnit`: run Scala/Chisel tests via Mill (see `Makefile.test`).
- `make clean`: remove build artifacts.

## Coding Style & Naming Conventions
Scala/Chisel code follows `scalastyle-config.xml` (no tabs, 120-char lines). New Scala files should include the header line `// See README.md for license details.` to satisfy header checks. Class and object names are `UpperCamelCase`, package object names are `lowerCamelCase`. Prefer existing patterns in `xiangshan/` and `top/`.

## Testing Guidelines
Unit tests live under `src/test/scala/` (and submodules like `huancun/src/test/scala/`). Tests typically use ChiselTest and are named `*Test.scala` or `*Tester.scala`. Run all tests with `make test`; run a specific test with `make test-DecodeUnit` or `mill -i xiangshan[chisel3].test.testOnly <TestClass>`.

## Commit & Pull Request Guidelines
Recent commits use short, imperative summaries (sometimes with a component prefix, e.g., `LDU: fix ...`) and occasionally include a PR number like `(#2536)`. Follow this pattern and keep subject lines concise. For pull requests, include a clear description, affected configs/modules, and the tests you ran; add reproduction steps or logs if you changed simulation behavior.

## Environment & Configuration Tips
Simulation requires `NEMU_HOME`, `NOOP_HOME` (this repo), and `AM_HOME` to be set to absolute paths. Common knobs include `CONFIG` (e.g., `DefaultConfig`, `MinimalConfig`), `NUM_CORES`, and `RELEASE=1` for release builds.
