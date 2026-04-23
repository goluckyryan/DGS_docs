# Building the Firmware

Stability: C2 - Active / semi-stable

_Source: `DGS_tools_pack/raw_FPGA/` + `DGS_tools_pack/fpga/` repos. Created: 2026-04-05._

## Toolchain Summary

| Module | Device | Tool | Version |
|--------|--------|------|---------|
| MTRG Main FPGA (ISE) | xc4vlx80 (Virtex-4 LX80) | Xilinx ISE | 14.7 | ✅ verified 2026-04-10 — Work13_4.xise `Device=xc4vlx80` |
| MTRG Main FPGA (Vivado) | xcku060-ffva1517-1L-i (Kintex UltraScale) | Xilinx Vivado | 2018.3 | ✅ verified 2026-04-11 — `MTRG/Firmware/VIVADO_MAIN_FPGA/trunk/project_1/project_1.xpr` (`Part=xcku060-ffva1517-1L-i`, `Vivado v2018.3`) |
| MTRG VME FPGA | xc3s400 (Spartan-3) | Xilinx ISE | 14.7 | ✅ verified 2026-04-10 — vme_A32_D32.xise `Device=xc3s400` |
| RTRG Main FPGA | xc4vlx80 (Virtex-4 LX80) | Xilinx ISE | 14.7 | ✅ verified 2026-04-11 — Work13_4.xise `Device=xc4vlx80` |
| DIG Main FPGA | xc3s5000 (Spartan-3) | Xilinx ISE | 14.7 | ✅ verified 2026-04-10 — BUS_LEFT.xise `Device=xc3s5000` |

---

## ISE 14.7

### Device Support

ISE 14.7 (released 2013, the final ISE version) supports all devices through 7-series, including:

- **Virtex-4** (xc4vlx80) — MTRG Main FPGA, RTRG Main FPGA
- **Spartan-3** (xc3s400, xc3s5000) — all VME FPGAs, DIG Main FPGA
- Virtex-5, Virtex-6, Spartan-6, 7-series

The `.xise` project files record the targets explicitly:

| Module | Project File |
|--------|-------------|
| MTRG Main FPGA | `MTRG/Firmware/MAIN_FPGA/trunk/Work13_4/Work13_4.xise` ✅ verified 2026-04-10 — file exists; `Device=xc4vlx80, DeviceFamily=Virtex4` |
| MTRG VME FPGA | `MTRG/Firmware/VME_FPGA/A32D32_VME_FPGA/Work13.4/vme_A32_D32.xise` ✅ verified 2026-04-10 — file exists; `Device=xc3s400` |
| RTRG Main FPGA | `RTRG/Firmware/DGS_Version/Rtr4704_mod_for_reset/MAIN_FPGA_4704_mod/Work13_4/Work13_4.xise` ✅ verified 2026-04-11 — `Device=xc4vlx80, DeviceFamily=Virtex4` |

### Ubuntu 24.04 Compatibility

ISE 14.7 does **not** run natively on Ubuntu 24.04. It was built for RHEL 6 / CentOS 6 / Ubuntu 12.04–14.04. Key incompatibilities:

| Issue | Root Cause |
|-------|-----------|
| Bundled library conflicts | ISE ships its own `libstdc++` and `libz` that clash with system versions |
| 32-bit binary support | ISE contains mixed 32/64-bit code; modern Ubuntu no longer installs `ia32-libs` by default |
| Python 2 dependency | ISE's Tcl/scripting layer assumes Python 2; Ubuntu 24.04 defaults to Python 3 |
| glibc version mismatch | ISE's bundled libraries were linked against glibc 2.15 |

#### Recommended Approach: Docker / Podman Container

Run ISE 14.7 inside a CentOS 6 or Ubuntu 14.04 container, with the repository mounted from the host:

```bash
docker run -it --rm \
  -v /home/dgsspark/DGS_FPGA:/work \
  -e DISPLAY=$DISPLAY \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  <centos6-ise-image>:14.7 \
  bash
```

Inside the container:

```bash
source /opt/Xilinx/14.7/ISE_DS/settings64.sh
cd /work/MTRG/Firmware/MAIN_FPGA/trunk/Work13_4
# GUI
ise Work13_4.xise
# Command-line batch build
xtclsh run.tcl rebuild_project
```

Pre-built ISE 14.7 container images are available from the community (search `xilinx-ise docker`). Alternatively, install ISE 14.7 into a fresh Ubuntu 14.04 or CentOS 6 container image manually — the ISE installer runs cleanly in those environments.

#### Alternative: LD_PRELOAD Patch (Native, No Container)

Some users run ISE natively on modern Linux by intercepting conflicting library calls via `LD_PRELOAD`. This is more fragile but avoids container overhead. The typical patch replaces ISE's bundled `libstdc++` and `libz` with symlinks to system versions, then sets `LD_LIBRARY_PATH` to exclude ISE's internal lib directories for those specific libraries. Community-maintained patch scripts exist (e.g., on GitHub) for this approach.

---

## Vivado 2018.3

The Kintex UltraScale port of the MTRG Main FPGA uses Vivado 2018.3, which has better long-term Linux support. Vivado 2018.3 runs on Ubuntu 16.04 and 18.04 officially. On Ubuntu 24.04 it may require the same container approach or the `--disable-webtalk` workaround and manual library patches, but the situation is less severe than ISE.

See [`deep_fpga_MTRG_VIVADO.md`](deep_fpga_MTRG_VIVADO.md) for the Vivado project details.

---

## Trigger System VHDL Simulation
_Source: `DGS_tools_pack/FPGA/others/Trig_sys_sim/` — documented 2026-04-12_

A standalone VHDL testbench (`ISE 13.4` project) that simulates two MTRG boards communicating over SERDES links. Useful for validating trigger command logic and SERDES state machine behavior without hardware.

### Files

| File | Purpose |
|------|---------|
| `trigger_data_types.vhd` | Common type definitions shared across MTRG, MyRIAD, Router |
| `MstrTrig_pkg.vhd` | Component declaration for `trigger_top` (the MTRG Main FPGA entity) with `BUILD_TYPE` generic |
| `bus_pkg.vhd` | Bus signal record definitions for test bench |
| `bus_trans.vhd` | Bus transaction helpers (stimulate/check VME transactions) |
| `crate_def_tb.vhd` | Top-level testbench: instantiates two `trigger_top` entities (LOCAL_MASTER + REMOTE_MASTER) as `BUILD_TYPE=4` (DGS Master Trigger) ✅ verified 2026-04-18 — `crate_def_tb.vhd:L36` (`BUILD_TYPE => 4`), `L42` (comment: `4-DGS Master Trigger`), `L19-20` (LOCAL_MASTER + REMOTE_MASTER ports) |
| `regio_tb.vhd` | Register I/O testbench: verifies register reads/writes |
| `top_tb1.VHD` | Alternate top-level testbench |
| `MyRIAD_pkg.vhd` | Copy of MyRIAD package (for cross-system simulation) |

### Usage

Open in ISE 13.4 (`Work13_4/Work13_4.xise`), select the desired testbench as the simulation top, and run ISim (behavioral simulation). No synthesis target is needed.

### Significance

This simulation shares the same `trigger_top` component and `BUILD_TYPE` encoding as the production MTRG firmware (`BUILD_TYPE=4` for DGS Master Trigger) ✅ verified 2026-04-18 — `crate_def_tb.vhd:L36,L42`. It was used during development to validate the SERDES link initialization protocol and command routing logic without requiring physical hardware.

## Firmware_Tags Archive (`FPGA/Firmware_Tags/`)

`DGS_tools_pack/FPGA/Firmware_Tags/` is a **historical firmware release archive** — frozen snapshots of firmware at key experiment/release milestones. Unlike the live `FPGA/DIG/`, `FPGA/MTRG/`, `FPGA/RTRG/` source trees, these are point-in-time copies of compiled bitfiles and sometimes source. Useful for reproducing past configurations or tracking firmware evolution.

_Source: `DGS_tools_pack/FPGA/Firmware_Tags/` (explored 2026-04-19)_

### Directory Structure

| Subdirectory | Contents |
|---|---|
| `Digitizer/MAIN_FPGA_TAGS/` | 28 DIG Main FPGA tags (20140613–20230809) + named branches (HELIOS, EXPERIMENT_*, Release_*) |
| `Digitizer/VME/` | DIG VME FPGA tags |
| `Digitizer/HELIOS/` | HELIOS experiment DIG variant (includes `dg pulse estimator.xls`, simulation configs) |
| `Digitizer/VME_FPGA_LBL/` | Lawrence Berkeley Lab DIG VME FPGA (historical reference) |
| `MasterTrigger/` | 17 MTRG Main FPGA tags (20140318–20220705) + `DGS/`, `DFMA/`, `JTA_TEMP_BRANCH/` |
| `MasterTrigger/VME_FPGA/` | MTRG VME FPGA older tags |
| `MasterTrigger/VME_FPGA_20250822/` | **Most recent MTRG VME FPGA** — `20250511.mcs` (May 2025), plus ChipScope, Cores, Source, Work13.4 ✅ verified 2026-04-19 — file listing confirmed |
| `Router/` | 8 RTRG tags (20140613–20220705) + `DGS/`, `DFMA/`, `Release_*` |
| `SBX/tag_20221020/` | Slope Box FPGA (SBX): `20221020.mcs`, `slopeboxint.bit`, ISE project files |
| `TriggerCPLD/20140613/` | CPLD firmware: `fast_strb.jed` (JEDEC format, 2014) |
| `TriggerVME/20140613/` | Trigger VME FPGA: `VME.mcs`, `VME.cfi`, `VME.prm`, `VME.sig` (2014) |

### Key Notes

- **MCS files** = Xilinx PROM programming files (used with iMPACT to flash hardware)
- **JED files** = JEDEC format for CPLD programming (used with iMPACT for CPLD)
- **BIT files** = raw FPGA configuration bitstream (used in ISE/iMPACT for direct FPGA config)
- The most recent DIG Main FPGA tag in this Firmware_Tags archive is **20230809** — ⚠️ **not** the currently deployed version (deployed: date=`20250704`, rev=`0x4CD8`; stored in `ioc/` via git-LFS, not in this archive)
- The most recent MTRG VME FPGA tag is **20250511.mcs** (inside `VME_FPGA_20250822/`)
- The TriggerCPLD and TriggerVME entries are from 2014 — the CPLD handles fast-strobe logic on the trigger board
- `SBX/` contains the Slope Box FPGA — separate from the main trigger chain; used for preamp slope adjustment

### Digitizer Tag Timeline (MAIN_FPGA_TAGS)

Selected milestones (from directory names):

| Tag | Notes |
|---|---|
| `EXPERIMENT_SEP_2012` | Earliest archived experiment tag |
| `Release_20130829` | First named release |
| `Release_20140318` | Also present in MasterTrigger and Router |
| `20180507` | Present in DIG, MTRG, RTRG — major synchronized release |
| `20211118` | Historical tag — NOT the deployed firmware |
| `20230809` | Most recent DIG tag in the Firmware_Tags archive — ⚠️ **not** the currently deployed version |

> ✅ verified 2026-04-20 — `ioc/README.md:L28-29`: deployed DIG Main FPGA date = `20250704`, rev = `0x4CD8`. The `20250704` build is **not** present in the Firmware_Tags archive (archive only goes to `20230809`). The firmware binary is stored in `ioc/` via git-LFS, not in the Firmware_Tags snapshot archive.

## Cross-References

- `knowledgeBase/fpga.md` — FPGA system overview: which firmware runs on which device
- `knowledgeBase/deep_fpga_DIG.md` — DIG firmware: ISE 14.7 Spartan-3 project, build branches
- `knowledgeBase/deep_fpga_MTRG.md` — MTRG overview: ISE / Vivado projects, 3 devices
- `knowledgeBase/deep_fpga_MTRG_VIVADO.md` — MTRG Vivado port: Kintex UltraScale build details
- `knowledgeBase/deep_fpga_MTRG_VME.md` — MTRG VME FPGA: Spartan-3 VME interface firmware + tag history
- `knowledgeBase/deep_fpga_RTRG.md` — RTRG firmware: Virtex-4, ISE project
- `knowledgeBase/vxworks.md` — VxWorks cross-compilation (IOC driver side of the build chain)

---
*Source: `DGS_tools_pack/raw_FPGA/` + `DGS_tools_pack/fpga/`. Created: 2026-04-07.*
