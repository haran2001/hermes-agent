---
name: atum-a3-nano-quartus-bringup
description: "Use when working on Hariharan's Terasic Atum A3 Nano / Intel Agilex 3 FPGA projects, especially Quartus VM builds, pin-map/QSF work, JTAG-Avalon/System Console programming, TALOS-V2 microGPT, Vortex, or assessing whether larger accelerators such as Gemmini fit the board."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [atum-a3-nano, agilex-3, quartus, fpga, jtag, talos, vortex, gemmini]
    related_skills: [silicon-eda-toolchain-operations, cloud-fpga-vm-operations, remote-vm-operations, remote-vm-long-running-jobs]
---

# Atum A3 Nano Quartus Bringup

## Overview

This is the board-specific skill for Hariharan's **Terasic Atum A3 Nano** work. It captures the known board identity, safe programming flow, VM/Quartus blockers, and reusable project discipline for porting accelerator RTL onto this FPGA.

Use this alongside `cloud-fpga-vm-operations` whenever the build host is a VM and the board is programmed elsewhere.

## Board Identity

- **Board name:** Terasic **Atum A3 Nano**.
- **FPGA part:** Intel **Agilex 3 A3CZ135BB18AE7S**.
- **Quartus edition:** Quartus Prime **Pro** 24.3+; Lite/Standard cannot target Agilex 3.
- **Known clock/pins from prior verified pin-map work:**
  - `CLOCK_50` → `PIN_K43`, 1.2 V.
  - `SW[1:0]` → `PIN_A19` / `PIN_B24`, 1.2 V.
  - `LED[3:0]` → `PIN_AG2` / `PIN_AM6` / `PIN_AF1` / `PIN_AF2`, 3.3-V LVCMOS.
- **Programming interface:** USB-Blaster/JTAG via Quartus Programmer on the board-attached machine.

Never invent additional pins. Use the Terasic user manual/System CD/golden QSF as source of truth before assigning GPIO, clocks, reset, DDR, or transceiver pins.

## Known Project Repos

- `~/Desktop/talos-v2-atum` / `haran2001/talos-v2-atum` — TALOS-V2 microGPT port to Atum A3 Nano. Prior pushed commit `86b3c3f` added verified pin map and QSF assignments. Current caveat: PLL and JTAG bridge were still stubs in the last known state.
- `~/Desktop/vortex-atum-a3` — planning-only Vortex bringup plan unless more files have since been added. The plan targets a minimal Vortex GPGPU with custom JTAG-to-AXI + BRAM shell.

## Standard Bringup Phases

### Phase 0 — Toolchain and license preflight

Check on the VM or Linux host:

```bash
quartus_sh --version
quartus_pgm --version
quartus_sh --tcl_eval 'puts [get_part_list -family "Agilex 3"]' | grep A3CZ135BB18AE7S
```

If synthesis fails with:

```text
Error (19286): No license for family Agilex 3
```

then stop. This is not an RTL problem. Generate a fixed/node-locked Quartus Pro Agilex 3 license and set `LM_LICENSE_FILE` in the exact shell/wrapper that runs Quartus.

Completion criterion: Quartus Pro sees `A3CZ135BB18AE7S` and license preflight is clean, or the license blocker is reported explicitly.

### Phase 1 — Cheap functional proof before Quartus

For any imported RTL/generator:

1. Run upstream simulation first if available.
2. Build a board-wrapper behavioral simulation that replaces physical JTAG/IP blocks with testbench drivers.
3. Confirm expected register writes/reads or token/kernel outputs before burning Quartus time.

Completion criterion: upstream or wrapper sim passes with a captured transcript.

### Phase 2 — Board skeleton compile

Start from the golden QSF/pin map. The minimal skeleton must include:

- Top entity.
- `A3CZ135BB18AE7S` device assignment.
- `CLOCK_50` constraint in SDC (`20 ns` for 50 MHz input).
- Only verified pin assignments.
- Real or clearly marked stub IP.

If using stubs, predict and report stub-collapse risk before compiling. A successful compile with ALM≈1/registers≈0 only proves filelist/QSF/plumbing; it does not prove the real design fits.

Completion criterion: compile logs and resource report are parsed and labeled as either `plumbing/stub` or `real-IP` evidence.

### Phase 3 — Replace stubs with real IP

The Atum projects repeatedly need two real IP blocks:

- **Agilex 3 IOPLL** from 50 MHz board clock to the design clock.
- **JTAG-to-Avalon or JTAG-to-AXI bridge** for host MMIO / System Console access.

Do not claim a functional bitstream until these are real IP blocks or a functionally equivalent bridge exists. Stubbed JTAG masters tie off command paths and cause Quartus to constant-propagate large designs away.

Completion criterion: generated IP files are in the repo/workspace, top-level instantiation names match, and Quartus no longer prunes the control path to constants.

### Phase 4 — Full compile and artifact collection

Run `quartus_sh --flow compile <revision>` detached on VM/host. Preserve:

- `.sof`.
- `*.fit.rpt`, `*.sta.rpt`, `*.asm.rpt`, resource summary, timing summary.
- Full compile log.
- Prediction-vs-result note in `context/`.

Completion criterion: `.sof` exists, timing/utilization are parsed, and result is compared to the pre-run prediction.

### Phase 5 — Program and test the physical board

On the board-attached machine:

```bash
jtagconfig
quartus_pgm -c <cable> -m JTAG -o "p;output_files/<revision>.sof"
```

For JTAG-Avalon designs, use `system-console` or `quartus_stp` scripts to read/write a known register before running the full host protocol.

Completion criterion: programming command exits 0 and a board-visible or register-level smoke test passes.

## Design-Fit Assessment Rules

For any accelerator candidate, assess five axes separately:

1. **Fabric:** ALMs/LUTs/registers/DSP/M20K estimate.
2. **On-chip memory:** M20K capacity and banking vs scratchpad/BRAM needs.
3. **External memory/platform:** whether the design assumes DDR/HBM/PCIe/Linux/HPS that Atum does not provide in the MVP.
4. **Host control:** JTAG-Avalon/System Console vs PCIe/driver/runtime assumptions.
5. **Software stack:** baremetal vs proxy-kernel vs Linux workload assumptions.

A design can fit fabric but still be a poor board project if it assumes PCIe, large DDR, or Linux runtime services.

## Gemmini-Specific Atum Notes

Gemmini is a Chipyard/Rocket RoCC accelerator, not a standalone Verilog IP. For Atum A3 Nano:

- Start with **LeanGemmini**-style configs, not the full default, because default Gemmini includes a 16×16 array, 256 KiB scratchpad, 64 KiB accumulator, DMA/TLB/ROB queues, scaling, pooling, and optional convolution features.
- The real gate is not only Gemmini's array; it is the whole Rocket/SoC + memory system + JTAG/debug boot path.
- Avoid promising Linux/FireSim-style workloads on Atum until a memory/boot/runtime story exists.
- A practical Atum plan is: elaborate small Chipyard/Gemmini config → generate Verilog → Quartus compile for fabric fit → then design a board-specific shell/host-loader. Do not jump directly to “run ResNet on board.”

Completion criterion for any Gemmini-on-Atum claim: name the exact `Config`, generated Verilog path, Quartus utilization/timing, and runtime environment used.

## Common Pitfalls

1. **Calling it Fortis when the evidence says Atum.** For this project, the verified board name is Terasic Atum A3 Nano unless the user supplies a different board/link.
2. **Running Quartus on macOS.** Quartus does not run natively on macOS; use Linux/Windows or remote VM for build, and a board-attached programmer host for JTAG.
3. **License blocker hidden by retries.** `Error (19286)` needs license/hostid work, not more RTL edits.
4. **Stub-collapse reported as fit.** ALM≈1/registers≈0 means design is optimized away; say so.
5. **Programming from cloud without hardware path.** A cloud VM cannot program a locally held board unless a verified JTAG-over-IP setup exists.
6. **Big accelerator port without platform inventory.** Before porting Vortex/Gemmini, inventory clock, reset, memory, boot, debug/JTAG, and software runtime assumptions.

## Verification Checklist

- [ ] Board name and part number stated exactly: Terasic Atum A3 Nano / `A3CZ135BB18AE7S`.
- [ ] Quartus Pro/device-support/license preflight captured.
- [ ] Pin assignments sourced from Terasic docs/golden QSF, not guessed.
- [ ] Stub vs real-IP status labeled before interpreting utilization.
- [ ] Long VM compiles detached and observable.
- [ ] `.sof` and reports verified by path/size.
- [ ] Physical programming claim backed by `jtagconfig` + `quartus_pgm` or System Console evidence.
