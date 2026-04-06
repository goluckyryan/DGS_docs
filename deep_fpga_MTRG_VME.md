# VME_FPGA — VME Interface and Configuration FPGA

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

| Field | Value |
|-------|-------|
| Family | Spartan-3 |
| Part | xc3s400 |
| Package | fg320 |
| Speed Grade | -5 |
| Tool | Xilinx ISE 13.4 |
| Project File | `Firmware/VME_FPGA/A32D32_VME_FPGA/Work13.4/vme_A32_D32.xise` |
| Top Entity | `vme_top` |
| Bitfile | `Firmware/VME_FPGA/A32D32_VME_FPGA/Work13.4/20250711.mcs` |

## Role

The VME FPGA serves two primary functions:

1. **VME Slave** — Presents an A32/D32 VME interface to the host computer, providing access to status and configuration registers
2. **Configuration Manager** — Programs and boots the main FPGA from external flash memory

It bridges the host computer (VME bus) to the main FPGA, and controls all FPGA configuration via serial bitstream download.

## Source Files

**Location:** `Firmware/VME_FPGA/A32D32_VME_FPGA/Source/`

| File | Lines | Description |
|------|-------|-------------|
| `TOP.VHD` | 68,233 | Top-level entity (`vme_top`) — instantiates all submodules |
| `vme_addr_decode.vhd` | 15,625 | VME address space decoder and chip select generation |
| `external_bus_controller.vhd` | 22,964 | Flash/FPGA bus multiplexer and access sequencer |
| `configuration_controller.vhd` | 24,748 | FPGA programming sequencer (serial bitstream download) |
| `register_block.vhd` | 18,451 | Status and control register bank |

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
- Drives main FPGA configuration pins: PROGRAM, DONE, INIT, CCLK
- Supports 3 chip-enable lines for multiple flash devices
- Reports configuration status (success, init error) via VME registers

### External Flash Interface

- Byte/word selectable access
- Multiplexed with main FPGA configuration bus
- Managed by `external_bus_controller.vhd`

### Inter-FPGA Communication

A 10-bit bidirectional bus connects the VME FPGA to the main FPGA:
- Configuration command exchange
- Status signal routing
- Direction control per signal group

## Clock Domains

| Clock | Frequency | Used For |
|-------|-----------|----------|
| Main clock | 50 MHz | VME interface logic |
| Derived | 25 MHz | Flash access timing |
| Derived | 100 MHz | High-speed logic |

All derived from a single oscillator via DCM.

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

- `dgs/deep_fpga_MTRG.md` — MTRG overview: all three devices (Main FPGA + VME FPGA + CPLD)
- `dgs/deep_fpga_MTRG_MAIN.md` — Main FPGA: the device this VME FPGA programs and controls
- `dgs/fpga.md` — VME control hierarchy: how all VME FPGAs fit into the system
- `dgs/ioc.md` — EPICS IOC: the software that drives VME A32/D32 register writes
