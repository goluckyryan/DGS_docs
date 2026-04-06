# DGS_CPLD — Fast Strobe CPLD

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
| Tool | Xilinx ISE |
| Project File | `Firmware/DGS_CPLD/Work/Work.xise` |
| Top Entity | `fast_strb` |

## Role

The CPLD implements fast multiplicity threshold logic for auxiliary detectors. It emulates the analog Gammasphere "Fast OR" function: when germanium center contacts detect gamma rays, the CPLD asserts a `FAST_STRB` signal to auxiliary detectors within approximately **1 µs**.

## Source Files

**Location:** `Firmware/DGS_CPLD/`

| File | Lines | Description |
|------|-------|-------------|
| `fast_strb.vhd` | 20,957 | Top-level entity — all logic contained here |

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

Chip select 6 (binary 110):

| Address | Access | Description |
|---------|--------|-------------|
| 0xA004 (read) | R | Masked discriminator bits from RJ45 connector |
| 0xA004 (write) | W | Mux selects: `sum_conn_ctrl` [1:0], `FS_sel` |
| THRESH_REG | R/W | Multiplicity threshold value |
| MASK_REG | R/W | Discriminator bit mask |

### Key Signals

| Signal | Dir | Width | Description |
|--------|-----|-------|-------------|
| DISC_BITS | In | 8 | Discriminator input bits |
| CONN_A/B/C/D_DATA | In | 8 each | Router sum data inputs |
| FAST_STRB | Out | 1 | Fast strobe output to auxiliary detectors |
| SUM_CONN_OUT | Out | 8 | Sum data output to connector |
| VME_CS/RNW/STRB | In | — | VME bus control |
| BUF_VME_DATA | Bidir | 16 | VME data bus |

### Tristate Bus Management

- VME data bus: bidirectional with direction control
- Connector buffers: separate direction controls for AB, CD, and SUM groups

---

## See Also

- `dgs/deep_fpga_MTRG.md` — MTRG overview: all three devices (Main FPGA + VME FPGA + CPLD)
- `dgs/deep_fpga_MTRG_MAIN.md` — Main FPGA: receives the fast strobe analog multiplicity sum from this CPLD
- `dgs/fpga.md` — System overview: fast strobe latency (~1 µs) vs full SERDES trigger cycle (2 µs)
- `dgs/sbx.md` — Slope Box Extension: sources of the BGO sum signals this CPLD aggregates
