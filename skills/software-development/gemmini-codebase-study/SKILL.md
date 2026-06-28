---
name: gemmini-codebase-study
description: "Use when studying, modifying, or porting the ucb-bar/gemmini codebase: Chipyard/Rocket RoCC integration, Gemmini Chisel module map, key configs, software/runtime layout, simulation commands, and FPGA-fit assessment for boards such as Atum A3 Nano."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [gemmini, chipyard, chisel, rocc, dnn-accelerator, fpga, codebase-study]
    related_skills: [codebase-inspection, atum-a3-nano-quartus-bringup, silicon-eda-toolchain-operations]
---

# Gemmini Codebase Study

## Overview

Gemmini is a Chipyard/Rocket-Chip **RoCC DNN accelerator generator** written in Chisel. It is not a drop-in standalone FPGA top. Treat it as a generator integrated into a Rocket/BOOM SoC with a software stack, custom instructions, DMA/TLB, scratchpad/accumulator memories, and optional DNN features.

Use this skill for future work on `https://github.com/ucb-bar/gemmini`, especially when deciding whether/how to target Hariharan's Atum A3 Nano or a cloud-VM FPGA flow.

## Current Codebase Snapshot Studied

Local path used in the first study: `~/Desktop/gemmini`.

Observed commit: `8c3f992` on `master`.

Submodules are large and may timeout during clone; for architecture study, the top-level source is enough. The first partial clone had submodule checkouts incomplete:

- `software/gemmini-rocc-tests`
- `software/libgemmini`
- `software/onnxruntime-riscv`

LOC snapshot excluding those submodules/build dirs: about **100 files**, **12.1k code lines**, mostly Scala/Chisel (**63 Scala files**, **11.2k Scala code lines**).

## High-Level Architecture

From README and source:

- Gemmini attaches as a **LazyRoCC** accelerator using Rocket custom instructions.
- It has a systolic array for matrix multiplication.
- It uses an explicitly managed scratchpad and accumulator SRAMs.
- DMA moves data between main memory and private SRAMs.
- Controllers are decoupled into load, execute, and store paths.
- Optional hardware supports activations, scaling, pooling, convolution loop helpers, counters, and normalizations.

System context matters: Gemmini needs a Rocket/Chipyard SoC, memory system, and RISC-V software/runtime. Do not assess it like a single Verilog accelerator file.

## Module Map

Key files under `src/main/scala/gemmini/`:

| Area | Files | Purpose |
|---|---|---|
| Top-level RoCC integration | `Controller.scala` | `class Gemmini` extends `LazyRoCC`; writes generated C header; instantiates `Scratchpad`; chooses TL/ATL node. `GemminiModule` wires TLB, counters, reservation station, controllers, and scratchpad. |
| Config/generator parameters | `GemminiConfigs.scala`, `Configs.scala`, `ConfigsFP.scala`, `DSEConfigs.scala`, `Custom*.scala` | Define `GemminiArrayConfig` and mixins such as default, lean, FP32, DSE/PnR configs. |
| ISA/custom instruction encoding | `GemminiISA.scala`, `InstructionCompression.scala`, `LocalAddr.scala` | RoCC funct fields, packed rs1/rs2 formats, local scratchpad/accumulator addresses. |
| Access/execute orchestration | `ReservationStation.scala`, `ExecuteController.scala`, `LoadController.scala`, `StoreController.scala`, `Tiler*.scala`, `Loop*.scala` | Decoupled command queues, ROB/dependencies, matmul/conv CISC loops, DMA command issue/completion. |
| Compute core | `Mesh.scala`, `MeshWithDelays.scala`, `Tile.scala`, `PE.scala`, `Transposer.scala` | Two-level systolic array: combinational tiles plus pipelined mesh; transposition supports dataflow modes. |
| Memory subsystem | `Scratchpad.scala`, `AccumulatorMem.scala`, `DMA.scala`, `DMACommandTracker.scala`, `FrontendTLB.scala`, `SharedExtMem.scala` | Banked scratchpad/accumulator SRAMs, stream reader/writer DMA, TLB, optional external scratchpad memory. |
| DNN helpers | `Activation.scala`, `Normalizer.scala`, `PixelRepeater.scala`, `Im2Col.scala`, `VectorScalarMultiplier.scala`, `AccumulatorScale.scala` | Optional activation/scaling/normalization/im2col/pooling helpers. |
| Tests | `src/test/scala/gemmini/*.scala` | Unit tests for header generation, mesh, pipelines, DMA tracker, transposer. |

## Important Default Parameters

`GemminiConfigs.defaultConfig` in `Configs.scala` uses:

- `inputType = SInt(8.W)`, `weightType = SInt(8.W)`, `accType = SInt(32.W)`.
- Systolic shape: `tileRows=1`, `tileColumns=1`, `meshRows=16`, `meshColumns=16`.
- Dataflow: `Dataflow.BOTH`.
- Scratchpad: `sp_capacity = 256 KiB`, `sp_banks = 4`, single-ported.
- Accumulator: `acc_capacity = 64 KiB`, `acc_banks = 2`.
- Queues: load 8, store 2, execute 8; reservation station entries ld 8/st 4/ex 16.
- DMA: `dma_maxbytes=64`, `dma_buswidth=128`, `max_in_flight_mem_reqs=16`.
- TLB size 4.
- Optional features enabled by default include training convs, max pool, nonlinear activations, counters, and scaling.

For FPGA-fit exploration, start by reducing these: mesh dimensions, scratchpad/acc capacity, queues/ROB, optional conv/pool/activation/scaling, and memory interfaces.

## Chipyard Integration Points

`chipyard/GemminiConfigs.scala` defines SoC configs:

- `GemminiRocketConfig`: `DefaultGemminiConfig` + one Rocket core + 128-bit system bus.
- `FPGemminiRocketConfig`: FP32 Gemmini config.
- `LeanGemminiRocketConfig`: lean Gemmini config; likely the first candidate for smaller FPGA exploration.
- `ReRoCCManyGemminiConfig`: multiple Gemmini accelerators through ReRoCC.

README commands assume a full Chipyard checkout:

```bash
git clone https://github.com/ucb-bar/chipyard.git
cd chipyard
./build-setup.sh
source env.sh
cd generators/gemmini
make -C software/libgemmini install
```

Build software:

```bash
cd chipyard/generators/gemmini/software/gemmini-rocc-tests
./build.sh
```

Build Verilator simulator:

```bash
cd chipyard/sims/verilator
make CONFIG=GemminiRocketConfig
make CONFIG=GemminiRocketConfig run-binary BINARY=../../generators/gemmini/software/gemmini-rocc-tests/build/bareMetalC/template-baremetal
```

Functional Spike check:

```bash
spike --extension=gemmini pk ../../generators/gemmini/software/gemmini-rocc-tests/build/imagenet/resnet50-pk
```

## FPGA/Atum Port Assessment Workflow

Do not begin with Quartus. Use this staged approach:

1. **Clone through Chipyard or verify this repo's submodules.** A standalone partial clone is good for reading but not for builds.
2. **Build the smallest software/header path.** Run the header-generation test or generate `gemmini_params.h`; confirm config constants match the hardware.
3. **Elaborate a small config.** Start with `LeanGemminiRocketConfig`, then create an even smaller Atum-focused config if needed.
4. **Generate Verilog from Chipyard.** Capture generated-src path and exact config name.
5. **Run Verilator smoke before FPGA.** Use a tiny baremetal matrix multiply, not ResNet.
6. **Quartus compile only after generated Verilog is stable.** Treat the result as a fabric-fit experiment first; board execution needs a boot/memory/control plan.
7. **Only then design the Atum platform path.** Decide whether the board run is JTAG-driven baremetal, a tiny ROM/BRAM boot, or external memory-backed. Avoid Linux/FireSim claims unless the platform supports them.

Completion criterion: every claim names exact config, generated Verilog, simulation command, Quartus reports, and runtime environment.

## Practical Atum A3 Nano Starting Point

For Atum, use a resource-minimized Gemmini study first:

- `LeanGemminiRocketConfig` rather than `GemminiRocketConfig`.
- Consider custom config with:
  - Smaller mesh, e.g. 4×4 or 8×8 before 16×16.
  - Smaller scratchpad/accumulator capacities.
  - Weight-stationary-only or output-stationary-only, not `BOTH`.
  - Disable training convolutions, pooling, nonlinear activations, normalization, and extra scaling units initially.
  - Reduce queue lengths and reservation station entries.
- Measure generated Verilog area/timing with Quartus before writing board host software.

Report as a feasibility ladder:

1. Chisel elaborates.
2. Verilator/Spike functional smoke passes.
3. Quartus synthesis fits.
4. Quartus fit/timing passes at a modest clock.
5. Board programming succeeds.
6. Tiny baremetal workload produces correct output.

Do not skip from (1) to (6) in reporting.

## Common Pitfalls

1. **Treating Gemmini as standalone RTL.** It is coupled to Rocket/Chipyard, RoCC, TileLink, TLB, DMA, and software headers.
2. **Ignoring software/header coherence.** Hardware config changes must regenerate `gemmini_params.h` and rebuild software.
3. **Running full ResNet first.** Start with tiny baremetal matrix multiply; large DNNs need too much runtime/platform support.
4. **Assessing only the systolic array.** The SoC, scratchpad/accumulator, DMA, TLB, queues, and Rocket core can dominate FPGA feasibility.
5. **Submodule timeout mistaken for source failure.** Large software submodules can fail checkout while top-level Scala source is intact; distinguish study clone from build clone.
6. **Overpromising Atum board execution.** Atum needs an explicit memory/boot/JTAG runtime plan; FireSim/Linux assumptions do not automatically transfer.

## Verification Checklist

- [ ] Repo commit and submodule state recorded.
- [ ] Config name and parameter deltas stated.
- [ ] Generated C header is regenerated after config changes.
- [ ] Tiny software smoke built before large workloads.
- [ ] Verilator/Spike result captured before Quartus.
- [ ] Quartus result labeled as synthesis-only, fit/timing, or board-validated.
- [ ] For Atum, board-specific clock/pin/JTAG/programming evidence comes from `atum-a3-nano-quartus-bringup`.
