# VME_FPGA — VME Interface and Configuration FPGA

Stability: C3 - Structural / stable

_Source: `DGS_tools_pack/raw_FPGA/MTRG/Firmware/VME_FPGA/A32D32_VME_FPGA/`. Created: 2026-04-05._

## Table of Contents

- [Target Device](#target-device)
- [Role](#role)
- [Source Files](#source-files)
- [Architecture](#architecture)
  - [VME Interface (A32/D32)](#vme-interface-a32d32)
  - [Configuration Controller](#configuration-controller)
  - [External Flash Interface](#external-flash-interface)
  - [Inter-FPGA Communication](#inter-fpga-communication)
- [Clock Domains](#clock-domains)
- [Bitfile History](#bitfile-history)
- [IP Cores](#ip-cores)

## Target Device

| Field | Value | Verified |
|-------|-------|----------|
| Family | Spartan-3 | ✅ verified 2026-04-10 — `vme_A32_D32.xise`: `Device Family=Spartan3` |
| Part | xc3s400 | ✅ verified 2026-04-10 — `vme_A32_D32.xise`: `Device=xc3s400` |
| Package | fg320 | ✅ verified 2026-04-10 — `vme_A32_D32.xise`: `Package=fg320` |
| Speed Grade | -5 | ✅ verified 2026-04-10 — `vme_A32_D32.xise`: `Device Speed Grade=-5` |
| Tool | Xilinx ISE 13.4 | ✅ verified 2026-04-10 — `Work13.4/` directory name |
| Project File | `Firmware/VME_FPGA/A32D32_VME_FPGA/Work13.4/vme_A32_D32.xise` | |
| Top Entity | `vme_top` | ✅ verified 2026-04-10 — `TOP.VHD:L42` (`entity vme_top is`) |
| Bitfile | `Firmware/VME_FPGA/A32D32_VME_FPGA/Work13.4/20250711.mcs` | |

## Role

The VME FPGA serves two primary functions:

1. **VME Slave** — Presents an A32/D32 VME interface to the host computer, providing access to status and configuration registers
2. **Configuration Manager** — Programs and boots the main FPGA from external flash memory

It bridges the host computer (VME bus) to the main FPGA, and controls all FPGA configuration via serial bitstream download.

## Source Files

**Location:** `Firmware/VME_FPGA/A32D32_VME_FPGA/Source/`

| File | Lines | Description |
|------|-------|-------------|
| `TOP.VHD` | 1,345 | Top-level entity (`vme_top`) — instantiates all submodules | ✅ verified 2026-04-10 — `wc -l TOP.VHD` |
| `vme_addr_decode.vhd` | 314 | VME address space decoder and chip select generation | ✅ verified 2026-04-10 — `wc -l` |
| `external_bus_controller.vhd` | 448 | Flash/FPGA bus multiplexer and access sequencer | ✅ verified 2026-04-10 — `wc -l` |
| `configuration_controller.vhd` | 500 | FPGA programming sequencer (serial bitstream download) | ✅ verified 2026-04-10 — `wc -l` |
| `register_block.vhd` | 392 | Status and control register bank | ✅ verified 2026-04-10 — `wc -l` |

## Architecture

### VME Interface (A32/D32)

- 32-bit address bus
- 32-bit bidirectional data bus
- Address modifier (AM) decoding
- Geographic address (GA/GAP) decoding for slot identification
- Data strobes DS0/DS1 and address strobe AS
- DTACK/BERR acknowledgement generation
- Interrupt request outputs IRQ[7:1]
- IACK interrupt acknowledge routing

### Configuration Controller

Programs the main FPGA after power-on or on demand:
- Reads bitstream from external flash memory
- Drives main FPGA configuration pins: PROGRAM, DONE, INIT, CCLK ✅ verified 2026-04-15 — `configuration_controller.vhd:L10,L43,L51-52,L72,L74,L121` (20200702 tag: entity ports fpga_init_in/out; CCLK from controller; ASSERT_PRGM/RELEASE_PRGM states; config_done_error)
- Supports 3 chip-enable lines for multiple flash devices
- Reports configuration status (success, init error) via VME registers

### External Flash Interface

- Byte/word selectable access
- Multiplexed with main FPGA configuration bus
- Managed by `external_bus_controller.vhd`

### Inter-FPGA Communication

A 10-bit bidirectional bus (`FPGA2FPGA[9:0]`) connects the VME FPGA (Spartan-3) to the main FPGA (Virtex-4). ✅ verified 2026-04-19 — `TOP.VHD:L112` (`FPGA2FPGA : inout std_logic_vector(9 downto 0)`)

Signal direction split (fixed in firmware): ✅ verified 2026-04-19 — `TOP.VHD:L529-548`
- **Bits 0–3:** VME FPGA → Main FPGA (outputs from Spartan-3)
  - Bit 0: LED control (`fpga_ctrl_reg(8)`) — drives an LED on GRETINA Master Trigger
  - Bit 1: Router reset (active-high: `NOT fpga_ctrl_reg(9)`) — holds a GRETINA Router in reset
  - Bit 2: `fpga_ctrl_reg(10)`
  - Bit 3: `fpga_we` — FPGA write-enable signal
- **Bits 4–9:** Main FPGA → VME FPGA (inputs to Spartan-3); driven as `"000000"` in current firmware (reserved for future status readback)

## Clock Domains

| Clock | Frequency | Used For |
|-------|-----------|----------|
| Main clock | 50 MHz | VME interface logic | ✅ verified 2026-04-19 — `TOP.VHD:L142,L622,L644` (`CLK_50MHZ` from DCM CLK0 output; input is MASTER_CLOCK oscillator) |
| Derived | 25 MHz | Flash access / FPGA configuration clock (CCLK) | ✅ verified 2026-04-19 — `TOP.VHD:L304,L628,L647` (CLKDV_DIVIDE=2.0 → 25 MHz; `CLK_25MHZ` → `global_cclk` BUFG → `fpga_cclk`) |
| Derived | 100 MHz | High-speed logic (ChipScope ILA) | ✅ verified 2026-04-19 — `TOP.VHD:L645` (`CLK_100MHZ` from DCM CLK2X output) |

All three clocks derived from the single MASTER_CLOCK oscillator (pin P10, 50 MHz) via one DCM (`vme_clk_dcm`). ✅ verified 2026-04-19 — `TOP.VHD:L618-647`

## Bitfile History

All files located in `Firmware/VME_FPGA/A32D32_VME_FPGA/Work13.4/`:

| File | Notes |
|------|-------|
| `20250711.mcs` | Latest (current as of March 2026) |
| `20250602.mcs` | June 2025 |
| `20250601.mcs` | June 2025 |
| `20250528.mcs` | May 2025 |
| `20250511.mcs` | May 2025 |
| `20230406.mcs` | April 2023 |
| `Trigger_VME.mcs` | Generic/release copy |
| `vme_top.bit` | Raw bitfile |

Active development is ongoing — multiple builds released in 2025.

## IP Cores

**Location:** `Firmware/VME_FPGA/A32D32_VME_FPGA/Cores/`

ChipScope ILA cores for debug:
| Core | Description |
|------|-------------|
| `chipscope_icon` | ChipScope controller |
| `chipscope_ila_1X64_d1024` | 1×64-bit ILA, 1024 depth |
| `chipscope_ila_2X64_d1024` | 2×64-bit ILA, 1024 depth |
| `chipscope_ila_80_64_d1024` | 80+64-bit ILA, 1024 depth |
| `chipscope_ila_80_64_d2048` | 80+64-bit ILA, 2048 depth |
| `chipscope_ila_80_64_d4096` | 80+64-bit ILA, 4096 depth |

ChipScope projects: `ChipScope/A32D32_VME_1x64.cpj`, `A32D32_VME_2x64.cpj`

---

## See Also

- [deep_fpga_MTRG.md](deep_fpga_MTRG.md) — MTRG overview: all three devices (Main FPGA + VME FPGA + CPLD)
- [deep_fpga_MTRG_MAIN.md](deep_fpga_MTRG_MAIN.md) — Main FPGA: the device this VME FPGA programs and controls
- [fpga.md](fpga.md) — VME control hierarchy: how all VME FPGAs fit into the system
- [ioc.md](ioc.md) — EPICS IOC: the software that drives VME A32/D32 register writes
