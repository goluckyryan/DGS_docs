# DGS_CPLD — Fast Strobe CPLD

Stability: C3 - Structural / stable

_Source: `DGS_tools_pack/raw_FPGA/MTRG/Firmware/DGS_CPLD/` + CPLD schematic + `deep_fpga_MTRG_MAIN.md`. Created: 2026-04-06._

## Table of Contents

- [Target Device](#target-device)
- [Role](#role)
- [Source Files](#source-files)
- [Architecture](#architecture)
  - [Input Processing](#input-processing)
  - [Threshold Comparison](#threshold-comparison)
  - [VME Register Interface](#vme-register-interface)
  - [Key Signals](#key-signals)
  - [Tristate Bus Management](#tristate-bus-management)

## Target Device

| Field | Value |
|-------|-------|
| Family | XC9500XL CPLDs |
| Part | xc95144xl |
| Package | TQ100 |
| Speed Grade | -7 |

✅ All device fields verified 2026-04-07 — `Firmware/DGS_CPLD/Work/Work.xise` (Device Family, Device, Package, Speed Grade fields)
| Tool | Xilinx ISE |
| Project File | `Firmware/DGS_CPLD/Work/Work.xise` |
| Top Entity | `fast_strb` |

## Role

The CPLD implements fast multiplicity threshold logic for auxiliary detectors. It emulates the analog Gammasphere "Fast OR" function: when germanium center contacts detect gamma rays, the CPLD asserts a `FAST_STRB` signal to auxiliary detectors within approximately **1 µs**. ✅ verified 2026-04-07 — fast_strb.vhd:L37-38,L96

## Source Files

**Location:** `Firmware/DGS_CPLD/`

| File | Lines | Description |
|------|-------|-------------|
| `fast_strb.vhd` | 462 | Top-level entity — all logic contained here ✅ verified 2026-04-07 — `DGS_CPLD/fast_strb.vhd` |

## Architecture

### Input Processing

1. Receives 6-bit SUM data from up to 4 Router connections (CONN_A through CONN_D)
2. Uses bits [3:1] of each sum bus, scaled by 8
3. Accumulates all inputs into an 8-bit total multiplicity sum
4. Applies a mask register (MASK_REG bits [3:0]) to select which discriminator bits count

### Threshold Comparison

- Compares accumulated multiplicity against a programmable threshold register (THRESH_REG)
- Two modes selectable via `FS_sel`:
  - **OR mode:** Assert FAST_STRB if any discriminator bit is set
  - **Multiplicity mode:** Assert FAST_STRB if sum > threshold

### VME Register Interface

Chip select 6 (binary 110): ✅ verified 2026-04-09 — `fast_strb.vhd:L51` ("use chip select 6 (110) to gain access to the Mask register")

| Address | Access | Description |
|---------|--------|-------------|
| 0xA004 (read) | R | Masked discriminator bits from RJ45 connector | ✅ `fast_strb.vhd:L61` ("register at address A004 has different functionality on read vs. write") |
| 0xA004 (write) | W | Mux selects: `sum_conn_ctrl` [1:0], `FS_sel` | ✅ `fast_strb.vhd:L64-65` |
| THRESH_REG | R/W | Multiplicity threshold value | ✅ `fast_strb.vhd:L117` |
| MASK_REG | R/W | Discriminator bit mask | ✅ `fast_strb.vhd:L118` |

Note: `sum_conn_ctrl[1:0]` mux options: `00`=sum data, `01`=raw disc bits, `10`=masked disc bits, `11`=free-running 8-bit counter. ✅ verified 2026-04-09 — `fast_strb.vhd:L142` (comment on signal declaration, added JTA 2015-02-19)

### Key Signals

✅ verified 2026-04-20 — `DGS_CPLD/fast_strb.vhd:L84-100` (entity port declarations)

| Signal | Dir | Width | Description |
|--------|-----|-------|-------------|
| DISC_BITS | In | 8 | Discriminator input bits (one per channel; `7 downto 0`) |
| CONN_A/B/C/D_DATA | In | 8 each | Router sum data inputs (`7 downto 0`) |
| FAST_STRB | Out | 1 | Fast strobe output to auxiliary detectors |
| SUM_CONN_OUT | Out | 8 | Sum data output to connector (`7 downto 0`; tri-stated via OBUFT) |
| VME_CS/RNW/STRB | In | — | VME bus control (CS = 3-bit chip select; RNW = read/write; STRB = data strobe + STRB_IO edge-detect variant) |
| BUF_VME_DATA | Bidir | **8** | VME data bus (`7 downto 0`) — **corrected from earlier erroneous 16-bit entry** |

### Tristate Bus Management

- VME data bus: bidirectional with direction control
- Connector buffers: separate direction controls for AB, CD, and SUM groups

---

## Historical Lineage

The DGS CPLD is a direct descendant of the original **GRETINA/GRETNA fast-strobe CPLD** (2007–2008, J.T. Anderson). The SVN archive preserves snapshots in `DGS_SVN/dgs/GRETNA_CPLD_CHECK/` (Master and Router CPLD variants). Key differences between the GRETINA original and the production DGS CPLD:

| Feature | GRETINA original (2008) | DGS production |
|---------|------------------------|----------------|
| Purpose | GRETINA master trigger | DGS MTRG fast strobe |
| Register map | 0xA000 (mask/counts), 0xE000 (threshold) via 3-bit chip select | Simplified VME A/D decode |
| `LOC_MULT_COUNT_A/B/C/D` | Present (per-connector hit counters) | Not present |
| `SUM_CONN_CTRL` | Present | Not present |
| Source | `GRETNA_CPLD_CHECK/Master_CPLD/fast_strb.vhd` | `FPGA/MTRG/Firmware/DGS_CPLD/fast_strb.vhd` |

The Router CPLD variant (`GRETNA_CPLD_CHECK/Router_CPLD/`) uses `lookup_comp.vhd` (a lookup table adder) not present in the Master CPLD — it **adds two 4-bit multiplicity values** (A_IN + B_IN, each with an enable) using a lookup table rather than an arithmetic adder, producing a 5-bit sum. This pre-sums multiplicities from groups of connected digitizers before forwarding to the master. ✅ verified 2026-04-09 — `Router_CPLD/lookup_comp.vhd:L1` ("implementation of 4 bit + 4 bit adder plus enables done as a lookup") + entity port list (A_IN[3:0], B_IN[3:0], A_ENBL, B_ENBL → Sum_out[4:0])

Multiple GRETINA CPLD snapshots exist in `FPGA/Firmware_Tags/MasterTrigger/20200702/Gretina Trigger/VHDL/` (2007–2011 vintage), providing a full lineage history.

## See Also

- [deep_fpga_MTRG.md](deep_fpga_MTRG.md) — MTRG overview: all three devices (Main FPGA + VME FPGA + CPLD)
- [deep_fpga_MTRG_MAIN.md](deep_fpga_MTRG_MAIN.md) — Main FPGA: receives the fast strobe analog multiplicity sum from this CPLD
- [fpga.md](fpga.md) — System overview: fast strobe latency (~1 µs) vs full SERDES trigger cycle (2 µs)
- [sbx.md](sbx.md) — Slope Box Extension: sources of the BGO sum signals this CPLD aggregates
- [DGS_SVN.md](DGS_SVN.md) — `GRETNA_CPLD_CHECK/` entry for SVN archive context
