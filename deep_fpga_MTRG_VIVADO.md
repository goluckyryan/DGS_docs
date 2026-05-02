# VIVADO_MAIN_FPGA — Master Trigger Main FPGA (Vivado)

Stability: C3 - Structural / stable

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
| VHDL source | Shared | Mostly shared (~55/59 files identical); Vivado active source adds `matrix_trig.vhd` (deprecated in ISE `UnusedOrDeprecated/`), `EVENT_FIFO.vhd`; ISE active source has `Generated_top.vhd`, `jta_odelay.vhd`, `jta_vernier_pos_finder.vhd` | ✅ verified 2026-04-13 — diff of MAIN_FPGA/trunk/Source/ vs VIVADO_MAIN_FPGA/trunk/Source/; ✅ verified 2026-04-14 — matrix_trig.vhd found at MAIN_FPGA/trunk/Source/UnusedOrDeprecated/matrix_trig.vhd (not in active ISE build) |

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

## Vivado-Only Source Files

Two source files exist in the Vivado version that are **not** in the ISE active build:

### `matrix_trig.vhd` — Unfinished Matrix Trigger Prototype
- **Description:** Prototype for a generalized matrix trigger algorithm that would fire if any pair of 7 trigger inputs overlapped within a configurable time window.
- **7 inputs:** man/aux, sumX, sumY, CPLD, link L, link R, link U
- **Intended mechanism:** All 21 pairs (7 choose 2) fed into `overlap_machine` components; OR of all pair overlaps → trigger. Uses `trig_algo_support` wrapper with trigger type code `0x56`.
- **Status: Stub/unfinished.** The behavioral body instantiates `trig_algo_support` but never declares `SIMPLE_TRIGGER` as a local signal or instantiates any `overlap_machine`. The file compiles but does nothing useful. `overlap_machine` is declared as a component but has no implementation in any source file.
- **Not instantiated** in `top.vhd` — present in the project source list but never connected.
- **ISE counterpart:** `MAIN_FPGA/trunk/Source/UnusedOrDeprecated/matrix_trig.vhd` (same file, same state).
✅ verified 2026-04-17 — `VIVADO_MAIN_FPGA/trunk/Source/matrix_trig.vhd` + `top.vhd` (no instantiation of matrix_trig found)

### `EVENT_FIFO.vhd` — Event-Counting FIFO Wrapper
- **Author:** John Anderson. Originally from the FLIC project; reused in MTRG for **Monitor FIFO 7 building** (since 2015-10-02).
- **Underlying core:** `FIFO_INDEP_FWFT_18W1024D_AF_AE_PROGFULLPORT_PROGEPORT` — 18-bit wide, 1024 deep, FWFT, independent read/write clocks, with almost-full/empty and programmable threshold flags.
- **Data width:** 18 bits: [17]=event boundary tag, [16]=ILA trigger tag, [15:0]=payload data.
- **Event boundary tag (bit 17):** Set on the second-to-last word of each event written. Allows event-level framing across clock domain crossings.
- **ILA tag (bit 16):** Armed via a Pulsed Control register bit; fires when incoming data matches `ILA_TAG_MATCH_VAL_REG`. Propagated unchanged through all processing stages — used to trigger Chipscope ILA across the full event pipeline.
- **Event counter:** 8-bit counter tracks number of full events in FIFO. `EVENT_END` (write clock) increments; `EVENT_READ` (read clock, on first word of event) decrements. Three clock-domain modes via `COUNTER_MODE` generic: 0=same clock, 1=read faster, 2=write faster.
- **Outputs:** `EVENT_AVAILABLE` (nonzero count), `EVENTS_IN_FIFO` (8-bit count), overflow/underflow flags.
✅ verified 2026-04-17 — `VIVADO_MAIN_FPGA/trunk/Source/EVENT_FIFO.vhd` (368 lines)

## Cross-References

- [deep_fpga_MTRG.md](deep_fpga_MTRG.md) — MTRG overview: all 3 devices (Main FPGA, VME FPGA, CPLD)
- [deep_fpga_MTRG_MAIN.md](deep_fpga_MTRG_MAIN.md) — ISE-based MTRG main FPGA (production); shared VHDL source files
- [deep_fpga_MTRG_VME.md](deep_fpga_MTRG_VME.md) — VME FPGA (Spartan-3): configures and communicates with the main FPGA
- [deep_fpga_building.md](deep_fpga_building.md) — Vivado 2018.3 build toolchain setup
- [fpga.md](fpga.md) — FPGA firmware overview; MTRG role in the 3-tier trigger hierarchy
