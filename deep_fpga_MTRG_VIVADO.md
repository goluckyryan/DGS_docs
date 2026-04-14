# VIVADO_MAIN_FPGA — Master Trigger Main FPGA (Vivado)
_Source: `DGS_tools_pack/raw_FPGA/MTRG/Firmware/VIVADO_MAIN_FPGA/`. Created: 2026-04-05._

## Table of Contents

- [Target Device](#target-device)
- [Role](#role)
- [Differences from ISE Version](#differences-from-ise-version-main_fpga)
- [Source Files](#source-files)
- [IP Cores](#ip-cores)
- [Project Structure](#project-structure)
- [Build Artifacts](#build-artifacts)

## Target Device

| Field | Value |
|-------|-------|
| Family | Kintex UltraScale |
| Part | xcku060 |
| Package | ffva1517 |
| Speed Grade | -1L (industrial) |
| Tool | Xilinx Vivado 2018.3 |
| Project File | `Firmware/VIVADO_MAIN_FPGA/trunk/project_1/project_1.xpr` |
| Top Entity | `trigger_top` |
| Bitfile | `Firmware/VIVADO_MAIN_FPGA/trunk/Work13_4/trigger_top.bit` |

✅ verified 2026-04-11 — `FPGA/MTRG/Firmware/VIVADO_MAIN_FPGA/trunk/project_1/project_1.xpr`: Part=`xcku060-ffva1517-1L-i`, Vivado v2018.3

## Role

This is the Vivado-based port of the Master Trigger main FPGA, targeting the Kintex UltraScale device. It implements the same trigger logic as the ISE version (`MAIN_FPGA`) but uses updated Xilinx IP cores and the Vivado toolchain. This version is intended for newer hardware revisions.

For full functional description see [deep_fpga_MTRG_MAIN.md](deep_fpga_MTRG_MAIN.md) — the VHDL source files are shared between both versions.

## Differences from ISE Version (MAIN_FPGA)

| Aspect | MAIN_FPGA (ISE) | VIVADO_MAIN_FPGA (Vivado) |
|--------|-----------------|---------------------------|
| Tool | ISE 13.4 | Vivado 2018.3 |
| Target FPGA | Virtex-4 XC4VLX80 | Kintex UltraScale XCK060 |
| SERDES primitives | Virtex-4 SERDES | UltraScale GTH transceivers |
| FIFO/RAM cores | XCO CoreGen | Vivado IP Catalog |
| Debug | ChipScope (.cpj) | Integrated Logic Analyzer (ILA) |
| Synthesis engine | XST | Vivado Synth |
| Place & route | ISE PAR | Vivado P&R |
| VHDL source | Shared | Mostly shared (~55/59 files identical); Vivado adds `matrix_trig.vhd`, `EVENT_FIFO.vhd`; ISE has `Generated_top.vhd`, `jta_odelay.vhd`, `jta_vernier_pos_finder.vhd` | ✅ verified 2026-04-13 — diff of MAIN_FPGA/trunk/Source/ vs VIVADO_MAIN_FPGA/trunk/Source/ |

## Source Files

**Location:** `Firmware/VIVADO_MAIN_FPGA/trunk/Source/`

The VHDL source files are the same as in `MAIN_FPGA/trunk/Source/`. Refer to [deep_fpga_MTRG_MAIN.md](deep_fpga_MTRG_MAIN.md) for the full source file listing.

## IP Cores

**Location:** `Firmware/VIVADO_MAIN_FPGA/trunk/Cores/`

Vivado IP blocks (replacing XCO cores from ISE version). ✅ verified 2026-04-11 — all core names confirmed in `project_1.xpr` Run entries:

| Core | Description |
|------|-------------|
| `fifo_16x64K_async` | 16-bit wide, 64K deep async FIFO |
| `fifo_16x1023_async` | 16-bit wide, 1023 deep async FIFO |
| `FIFO_FWFT_80X16IN_20X64OUT_SEPCLK` | Width-converting FIFO, separate clocks |
| `FIFO_FWFT_48X64` | 48-bit FWFT FIFO |
| `FIFO_FWFT_32X16IN_16X32OUT_SEPCLK` | Width-converting FIFO, separate clocks |
| `FIFO_22W16D_FWFT` | 22-bit wide, 16 deep FWFT FIFO |
| `FIFO_IND_16Wx1024D_STD` | 16-bit, 1024 deep standard FIFO |
| `DPRAM_64Wx16D_A_1kWx1D_B` | Dual-port RAM |
| ILA cores | Integrated Logic Analyzer (64, 80, 128-bit widths) |

## Project Structure

```
VIVADO_MAIN_FPGA/trunk/
├── project_1/
│   ├── project_1.xpr          # Vivado project file
│   ├── project_1.cache/ip/    # Synthesized IP core cache
│   └── project_1.runs/
│       ├── synth_1/           # Top-level synthesis results
│       └── impl_1/            # Implementation results
├── Source/                    # Shared VHDL source (same as MAIN_FPGA)
├── Cores/                     # Vivado IP core definitions
└── Work13_4/
    ├── trigger_top.bit        # Main bitfile
    └── GRET_L_trigger_top.bit # GRETINA Link L variant
```

## Build Artifacts

| File | Description |
|------|-------------|
| `Work13_4/trigger_top.bit` | Main production bitfile |
| `Work13_4/GRET_L_trigger_top.bit` | Variant for GRETINA Link L mode |

✅ verified 2026-04-12 — both bitfiles confirmed present at `FPGA/MTRG/Firmware/VIVADO_MAIN_FPGA/trunk/Work13_4/` (trigger_top.bit + GRET_L_trigger_top.bit)

## Cross-References

- `knowledgeBase/deep_fpga_MTRG.md` — MTRG overview: all 3 devices (Main FPGA, VME FPGA, CPLD)
- `knowledgeBase/deep_fpga_MTRG_MAIN.md` — ISE-based MTRG main FPGA (production); shared VHDL source files
- `knowledgeBase/deep_fpga_MTRG_VME.md` — VME FPGA (Spartan-3): configures and communicates with the main FPGA
- `knowledgeBase/deep_fpga_building.md` — Vivado 2018.3 build toolchain setup
- `knowledgeBase/fpga.md` — FPGA firmware overview; MTRG role in the 3-tier trigger hierarchy
