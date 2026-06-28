---
name: cloud-fpga-vm-operations
description: "Use when using a cloud VM to build/program FPGA projects for Hariharan: GCP instance discovery, Quartus/Vivado disk/license preflight, detached compile jobs, artifact transfer, and the local-vs-remote boundary for physically programming boards such as Atum A3 Nano."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [fpga, cloud-vm, gcp, quartus, vivado, jtag, remote-jobs]
    related_skills: [remote-vm-operations, remote-vm-long-running-jobs, silicon-eda-toolchain-operations]
---

# Cloud FPGA VM Operations

## Overview

Use this skill when the FPGA toolchain runs on a cloud VM but the board is physically somewhere else. The recurring pattern is: compile and archive artifacts on the VM; program only from the machine with the USB/JTAG cable unless a verified JTAG-over-IP bridge exists.

This skill captures the reusable steps learned from Hariharan's Atum A3 Nano / Agilex 3 work and generalizes them to future FPGA boards and VM-hosted flows.

## When to Use

- A Quartus/Vivado/Yosys build is launched on a GCP/AWS/customer VM.
- The user asks for FPGA build status and there may be orphaned remote runs.
- `ssh` times out to an FPGA build VM.
- `gcloud` auth, source-IP allowlists, stopped instances, or ephemeral external IPs are involved.
- A board must be programmed after a remote compile.

Also load:
- `remote-vm-operations` for shared-VM safety and no-delete rules.
- `remote-vm-long-running-jobs` for any build over ~5 minutes.
- `silicon-eda-toolchain-operations` for vendor-specific FPGA/EDA flow dispatch.

## Required Mental Model

A cloud VM usually **cannot program a USB-attached FPGA board** unless the board is physically attached to the VM host or a known-good USB/JTAG-over-IP setup exists. Treat these as two separate phases:

1. **Remote build phase:** elaborate/synthesize/place/route/assemble on the VM; produce `.sof`, `.pof`, `.bit`, `.bin`, reports, logs.
2. **Local programming phase:** transfer the programming file to the machine with the board and run the vendor programmer locally.

Do not claim a board was programmed from a cloud run unless real programmer output (`quartus_pgm`, `vivado hw_server`, `openFPGALoader`, etc.) names the cable/device and exits cleanly.

## Standard Workflow

### 1. Identify the VM from cloud control plane first

If SSH silently times out, do not start with VPN theories. Check instance state and IP from the cloud control plane.

```bash
gcloud compute instances list --project <project>   --format='table(name,zone,status,networkInterfaces[0].accessConfigs[0].natIP)'
```

If `gcloud` reports refresh-token failure:

```text
There was a problem refreshing your current auth tokens: Reauthentication failed.
```

stop and ask Hariharan to run `gcloud auth login` in his own terminal. Do not invent a service-account workaround.

Completion criterion: VM name, project, zone, status, and current external IP are known from live output.

### 2. Probe the VM read-only before launching builds

Run disk/tool/license probes before any big compile:

```bash
ssh <vm> 'set -e
  echo === host ===; hostname; uptime; whoami
  echo === disk ===; df -h | grep -v tmpfs
  echo === tools ===; command -v quartus_sh || true; command -v quartus_pgm || true; command -v vivado || true
  echo === fpga processes ===; ps -axo pid,etime,command | egrep "quartus|qfit|vivado|yosys|nextpnr" | grep -v egrep || true'
```

For Quartus, also prove the target part family is supported:

```bash
quartus_sh --version
quartus_sh --tcl_eval 'puts [get_part_list -family "Agilex 3"]' | grep A3CZ || true
```

Completion criterion: disk has adequate free space, required tool binary is found, and any active builds are identified before launching another.

### 3. Never clean a shared VM without permission

If root is full, do not run `docker prune`, `rm -rf`, `apt clean`, or log truncation as an automatic fix. Show exact paths/sizes and ask. Prefer redirecting FPGA workspace, `$TMPDIR`, and tool caches to a project-owned data disk.

For Quartus/Vivado wrappers, export inside the detached script:

```bash
export HOME=/mnt/<data>/users/<user>
export TMPDIR=/mnt/<data>/users/<user>/tmp
export TMP=$TMPDIR TEMP=$TMPDIR
mkdir -p "$TMPDIR"
```

Completion criterion: build temp/state writes are routed to a safe writable project/data path or user explicitly approved cleanup.

### 4. Launch long builds detached with sentinels

Use a remote script, not a foreground SSH pipe:

```bash
RUN=<abs-run-dir>
mkdir -p "$RUN"
cat > "$RUN/run.sh" <<'SH'
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")/project"
echo "started=$(date -Iseconds)" > ../run.meta
quartus_sh --flow compile <revision> > ../quartus_compile.log 2>&1
rc=$?
echo "exit=$rc" >> ../run.meta
echo "finished=$(date -Iseconds)" >> ../run.meta
if [ "$rc" -eq 0 ] && [ -f output_files/<revision>.sof ]; then touch ../ok.flag; else touch ../failed.flag; fi
exit "$rc"
SH
chmod +x "$RUN/run.sh"
setsid nohup bash "$RUN/run.sh" </dev/null >"$RUN/launcher.log" 2>&1 & echo $! > "$RUN/run.pid"
```

Completion criterion: `run.pid` exists, process is live or `ok.flag/failed.flag` exists, and logs are on the VM disk.

### 5. Preserve artifacts and reports

For every build, collect at minimum:

- Build log (`quartus_compile.log` / `vivado.log`).
- Final programming artifact (`.sof`, `.pof`, `.bit`, `.bin`).
- Utilization and timing reports.
- `run.meta` with start/end/exit.
- A short markdown note under `context/` explaining prediction vs result.

Transfer artifacts with `rsync -avzP --partial` for large files. Do not rely on Discord screenshots as the only evidence.

Completion criterion: artifact path, report path, and transfer destination are named and verified with file sizes.

### 6. Program only from the machine with the board

For Intel/Altera boards:

```bash
jtagconfig
quartus_pgm -c <cable-index-or-name> -m JTAG -o "p;output_files/<revision>.sof"
```

For local programmer-only installs, full Quartus is not required; the Quartus Programmer package is enough if the `.sof` is already built.

Completion criterion: programmer output lists the cable/device and the programming command exits 0.

## Status Report Template

```text
VM: <name> <project>/<zone>, status=<RUNNING|TERMINATED|unknown>, IP=<live IP or auth blocked>
Build workspace: <path>
Active process: <pid/elapsed/command or none>
Latest run: <run-dir>, exit=<code or running>, flags=<ok/failed/running>
Artifact: <path>, size=<bytes>, mtime=<time>
Board programming: <not attempted | programmer output summary>
Blocker: <license/auth/disk/board absent/none>
```

## Common Pitfalls

1. **Cloud VM status guessed from stale SSH config.** External IPs are often ephemeral; always re-grab IP from `gcloud` before diagnosing SSH.
2. **`gcloud auth list` looks logged in but compute commands fail.** Active account is not enough; refresh token may be expired. Ask for `gcloud auth login`.
3. **Foreground SSH compile dies silently.** Use detached remote scripts with disk logs and sentinels.
4. **Remote compile mistaken for board programming.** A `.sof` on a VM is not a programmed board.
5. **License blockers misdiagnosed as RTL bugs.** Quartus `Error (19286)` for Agilex is a license issue; resolve `LM_LICENSE_FILE`/hostid before editing RTL.
6. **Too-good-to-be-true utilization.** ALM/LUT near zero usually means stubs, unpinned outputs, or constant propagation; treat as plumbing evidence only.

## Verification Checklist

- [ ] Cloud control-plane state and current IP checked when using a cloud VM.
- [ ] Disk/tool/license preflight captured before large builds.
- [ ] Long build launched detached with log, pid, meta, and success/failure sentinels.
- [ ] No destructive cleanup on shared VM without explicit approval.
- [ ] Artifact and reports exist with verified file sizes.
- [ ] Programming claim backed by real programmer output from the board-attached machine.
