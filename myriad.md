# MyRIAD — Trigger Expansion Module

Stability: C3 - Structural / stable

Source: `DGS_tools_pack/FPGA/others/MyRIAD/`

---

## Table of Contents

- [Overview](#overview)
- [FPGA Structure](#fpga-structure)
- [VME FPGA Register Map](#vme-fpga-register-map)
- [MAIN FPGA Source Files](#main-fpga-source-files)
- [Firmware Command Formats (Generic)](#firmware-command-formats-generic)
- [Hardware I/O (MAIN FPGA Top Level)](#hardware-io-main-fpga-top-level)
- [TDC (Time-to-Digital Converter)](#tdc-time-to-digital-converter)
- [Key Internal Registers](#key-internal-registers-from-registersvhd-and-myr-reg-notestxt)
- [Hardware Status Register (0x0020) Bit Map](#hardware-status-register-0x0020-bit-map)
- [Pulsed Control Register (0x040C) Bit Map](#pulsed-control-register-0x040c-bit-map)
- [SERDES Config Register Bit Map](#serdes-config-register-bit-map)
- [Coincidence Logic](#coincidence-logic)
- [TTCL Trigger Interface](#ttcl-trigger-interface)
- [GITMO (Gammasphere Interface to Trigger Module)](#gitmo-gammasphere-interface-to-trigger-module)
- [FIFO Data Write State Machine](#fifo-data-write-state-machine)
- [MAIN FPGA Sub-Module Deep Dives](#main-fpga-sub-module-deep-dives)
  - [mstr_mach.vhd — MyRIAD Master State Machine](#mstr_machvhd--myriad-master-state-machine-968-lines)
  - [SERDES_TX_MACH.vhd — SERDES Transmit State Machine](#serdes_tx_machvhd--serdes-transmit-state-machine-258-lines)
  - [NIM_Delay.vhd — NIM Delay Line](#nim_delayvhd--nim-delay-line-97-lines)
  - [DCBAL_in.vhd — DC-Balance Input Decoder](#dcbal_invhd--dc-balance-input-decoder--clock-domain-crossing-147-lines)
- [Diagnostic / Debug Features](#diagnostic--debug-features)
- [VME FPGA Sub-Module Deep Dives](#vme-fpga-sub-module-deep-dives)
  - [vme_addr_decode.vhd — VME Address Decoder](#vme_addr_decodevhd--vme-address-decoder-293-l)
  - [external_bus_controller.vhd — External Bus Controller](#external_bus_controllervhd--external-bus-controller-285-l)
  - [configuration_controller.vhd — FPGA Configuration FSM](#configuration_controllervhd--fpga-configuration-fsm-349-l)
- [Cross-References](#cross-references)

---

## Overview

**MyRIAD** (name likely derived from "many/multiple I/O module") is a VME-based trigger expansion module designed for Digital Gammasphere (DGS) and GRETINA experiments at Argonne National Laboratory. It was designed by the High Energy Physics Division at ANL for the Physics Division.

Key purpose: provide additional NIM/ECL I/O, coincidence logic, FIFO data buffering, and SERDES-based trigger/timestamp communication to the DGS master trigger system.

Firmware type code: **0xB** = "MyRIAD Trigger expansion module" (from `MyRIAD_pkg.vhd`) ✅ verified 2026-04-25 - MyRIAD_pkg.vhd:L45

Latest firmware revision: `0x0B16` dated `0810/2022` (August 10, 2022) ✅ verified 2026-04-25 - MyRIAD_pkg.vhd:L53-55

FPGA device: **Xilinx Spartan-3 XC3S1000-FG456**

---

## FPGA Structure

MyRIAD has **two FPGAs**:

| FPGA | Source directory | Role |
|------|-----------------|------|
| MAIN FPGA | `MAIN_FPGA/Source/` | All main logic: SERDES, TDC, coincidence, FIFO write, timestamp, NIM/ECL I/O |
| VME FPGA | `VME_FPGA/Source/` | VME bus interface, address decode, register bridge to MAIN FPGA |

### VME FPGA Components
- `TOP.VHD` — top-level VME interface
- `vme_addr_decode.vhd` — VME address decoder
- `external_bus_controller.vhd` — external bus controller to MAIN FPGA
- `configuration_controller.vhd` — configuration FSM
- `register_block.vhd` — register interface
- `myr reg notes.txt` — hardware register bit definitions (see Register Map below)

---

## VME FPGA Register Map

_Source: `FPGA/others/MyRIAD/VME_FPGA/Source/register_block.vhd`. Code-read 2026-04-27._

**Firmware:** `code_revision = 0x0F16` (type F = VME FPGA, major=1, minor=6; PCB rev 0 = proto); `code_date = 0x0531` (May 31, 2016 — last ILA touch). ✅ verified 2026-04-27 — `register_block.vhd:L83-85`

All registers are 16-bit, accessed at A32/D16. The VME FPGA acts as a gateway to the MAIN FPGA via the external bus controller.

| Address | R/W | Description |
|---------|-----|-------------|
| 0x0900 | R/W | `fpga_ctrl_reg` — control register driven to MAIN FPGA |
| 0x0902 | R | `fpga_status_register` — status read-back from MAIN FPGA (read-only) |
| 0x0904 | R | `aux_status` — auxiliary status from MAIN FPGA (read-only) |
| 0x0906 | W | Config request/ack: bit 0 = config_request (loads 8-cycle pulse), bit 1 = config_done_ack |
| 0x0908 | R/W | Flash VPEN control: bit 4 = flash_vpen (enable flash write voltage) |
| 0x090A | R/W | `config_start_low` — low 16 bits of FPGA config start address |
| 0x090C | R/W | `config_start_high` — bits [23:16] of config start address (upper 8 bits of VME data used) |
| 0x090E | R/W | `config_stop_low` — low 16 bits of FPGA config stop address |
| 0x0910 | R/W | `config_stop_high` — bits [23:16] of config stop address |
| 0x0916 | R | Returns 0x0916 (self-identifying register, read-only) |
| 0x0918 | R/W | `aux_rw_reg0` — scratch register (default 0x0000) |
| 0x091A | R/W | `aux_rw_reg1` — scratch register (default 0x1111) |
| 0x091C | R/W | `aux_rw_reg2` — scratch register (default 0x2222) |
| 0x091E | R/W | `aux_rw_reg3` — scratch register (default 0x3333) |
| 0x0924 | R | `code_revision` = 0x0F16 (firmware type/revision, read-only) |
| 0x0928 | R | `code_date` = 0x0531 (firmware build date, read-only) |
| 0x0980 | R/W | Flash address [15:0] |
| 0x0982 | R/W | Flash address [23:16] (bits [7:0] of VME data) |
| 0x0984 | W | Trigger flash read/write (fixed address) — blocks until flash_ack |
| 0x0986 | W | Trigger flash read/write (auto-increment address) — blocks until flash_ack |

**Config stop defaults:** `config_stop_high=0x0007`, `config_stop_low=0x0000` → stop address 0x070000.  
Reference bit-stream sizes (VHDL comment): XC3S1000 = 0x625F8 bytes, XC3S5000 = 0x195070 bytes, XC4VLX80 = 0x2C6C90 bytes.  
✅ verified 2026-04-27 — `register_block.vhd:L120-135` (reset defaults), `L161-280` (read/write decoder)

---

## MAIN FPGA Source Files

| File | Purpose |
|------|---------|
| `MyRIAD.vhd` | Top-level entity, all I/O port declarations |
| `MyRIAD_pkg.vhd` | Package: constants, type arrays, firmware revision, component declarations |
| `registers.vhd` | All VME-accessible registers (read/write logic) |
| `GITMO_TOP.vhd` | GITMO (Gammasphere Interface to Trigger MOdule) — collects GS master trigger clock/trigger info, packs into SERDES data stream for DGS master trigger |
| `GITMO_RCV_MACH.vhd` | GITMO receive state machine |
| `tdc_unit2.vhd` | TDC unit using carry-chain delay lines (64-bit vernier), clocked at 300 MHz |
| `tdc_short_chain.vhd` | Short variant of carry-chain TDC |
| `Timestamp_Generator.vhd` | System timestamp generator |
| `SERDES_RX_Mach_R2.vhd` | SERDES receive state machine (R2 = revision 2) |
| `SERDES_TX_MACH.vhd` | SERDES transmit state machine |
| `Phase_Hunter_SerDes.vhd` | DCM phase hunt for SERDES lock alignment |
| `DCM_CONTROLLER.vhd` | DCM management (lock, reset, logic reset) |
| `dc_balance_mach.vhd` | DC-balance encoder/decoder for SERDES |
| `DCBAL_in.vhd` | DC-balance input (with FIFO) |
| `DCBAL_in_nofifo.vhd` | DC-balance input (no FIFO variant) |
| `disparity_lookup.vhd` | Disparity lookup table for 8B/10B-style coding |
| `NIM_Delay.vhd` | NIM delay line using on-chip RAM (up to 65535 cycles delay per channel) |
| `Fifo.vhd` | Internal FIFO wrapper |
| `mstr_mach.vhd` | Master state machine |
| `basic_capture_counter.vhd` | Basic event capture counter |
| `trigger_data_types.vhd` | Data type definitions for trigger packets |
| `pinlock.ucf` | Pin constraints (XC3S1000 device) |

---

## Firmware Command Formats (Generic)

The MAIN FPGA has a generic parameter `COMMAND_LINE_COMMAND_FORMAT` controlling which command set it responds to:

| Value | Mode |
|-------|------|
| 0 | DGS Master Trigger |
| 1 | DGS Router |
| 2 | GRETINA Master Trigger |

(`myriad_pkg.vhd`: `cCMD_FORMAT_DGS_MASTER`, `cCMD_FORMAT_DGS_ROUTER`, `cCMD_FORMAT_GRETINA_MASTER`) ✅ verified 2026-04-25 - MyRIAD_pkg.vhd:L159-161

---

## Hardware I/O (MAIN FPGA Top Level)

### External FIFOs
- Two external FIFO chips: **FIFO A** and **FIFO B**
- Each is 18-bit wide (16 data bits + 2 flag bits) ✅ verified 2026-04-25 - MyRIAD.vhd:L36-37 (FIFOFLAG0/1 = bits 16/17)
- FIFO A data[15:0] → VME data bits [15:0]; FIFO B data[15:0] → VME data bits [31:16] ✅ verified 2026-04-25 - MyRIAD.vhd:L37,L72
- Flag bits (bits 16/17) are control-only; not driven to VME data bus ✅ verified 2026-04-25 - MyRIAD.vhd:L36-37
- Full first-word fall-through mode supported ✅ verified 2026-04-25 - MyRIAD.vhd:L49 (FIFO_A_FWFTSI_pin = FirstWordFallThru)
- Standard IDT/Cypress-style FIFO interface: WEN, REN, RCLK, WCLK, EF, HF, PAF, PAE, FF ✅ verified 2026-04-25 - MyRIAD.vhd:L44-66

### SERDES Link
- Uses **DS92LV18** LVDS SERDES chip (18-bit parallel ↔ serial) ✅ verified 2026-04-25 - MyRIAD.vhd:L158
- Receive clock: SERDES_RCLK (both I/O pad and GCLK pad present) ✅ verified 2026-04-25 - MyRIAD.vhd:L169-170
- Lock indicator: SERDES_LOCK_pin is **active LOW** (low = locked) ✅ verified 2026-04-25 - MyRIAD.vhd:L167
- RJ45 connector with 2 extra LVDS pairs (STAT0/STAT1) for status I/O ✅ verified 2026-04-25 - MyRIAD.vhd:L175-181
- RJ45 has two LED pairs (SERDES_LED0/LED1 + complement pins) ✅ verified 2026-04-25 - MyRIAD.vhd:L185-187

### NIM I/O
- 8× NIM inputs (`NIM_IN_pin[7:0]`) ✅ verified 2026-04-25 - MyRIAD.vhd:L133
- 4× NIM outputs (`NIM_OUT_pin[3:0]`) ✅ verified 2026-04-25 - MyRIAD.vhd:L134
- NIM delay lines: software-configurable delay per channel using on-chip RAM (65535-cycle max)

### ECL I/O
- 16× ECL inputs (`ECL_IO_pin[15:0]`) — only receivers populated (drivers physically present but not stuffed on board) ✅ verified 2026-04-25 - MyRIAD.vhd:L135
- 4× ECL driver enables (`ECL_DRIVE_EN0-3`) for groups of 4 bits each ✅ verified 2026-04-25 - MyRIAD.vhd:L136-139
- Secondary ECL connector (FERA interface): FERA_FULL, FERA_ACK, FERA_OVF, FERA_WSI, FERA_VETO signals ✅ verified 2026-04-25 - MyRIAD.vhd:L142-148

### Clock Sources
- Oscillator clock: `MAIN_FPGA_LOGIC_CLK_0/1` (redundant, 50 MHz)
- Switchable machine clock: `MAIN_FPGA_MACH_CLK_0/1` — mux selects between oscillator and SERDES recovered clock
- CLOCK_SEL_pin: controls external clock mux chip
- DCM produces: 5 MHz (÷10), 50 MHz (×1), 100 MHz (×2), 250 MHz (×5)

### LEDs
- 5× front-panel LEDs (`LED_pin[9:5]`)
- 2× RJ45 LEDs + complements

---

## TDC (Time-to-Digital Converter)

- `tdc_unit2.vhd`: carry-chain TDC
- 64-bit vernier output (one bit per carry-chain slice) ✅ verified 2026-04-25 - tdc_unit2.vhd:L43
- Sampling clock: 300 MHz (`TDC_CLOCK`) ✅ verified 2026-04-25 - tdc_unit2.vhd:L25
- Input: any logic signal (`BIT_IN`) ✅ verified 2026-04-25 - tdc_unit2.vhd:L26
- Dual interleaved delay chains: `DELAY_CHAIN_ODD` and `DELAY_CHAIN_EVEN` ✅ verified 2026-04-25 - tdc_unit2.vhd:L43
- Uses XORCY_D/XORCY_L Xilinx primitives for carry-chain delay line ✅ verified 2026-04-25 - tdc_unit2.vhd:L49-64
- Reset enables capture; stop event captured as thermometer code

---

## Key Internal Registers (from `registers.vhd` and `myr reg notes.txt`)

| Address | Name | Direction | Description |
|---------|------|-----------|-------------|
| 0x0000 | BOARD_ID | R | Board identification |
| 0x0004 | REG_004 / FIFO_STATUS | R | FIFO status flags |
| 0x0020 | HARDWARE_STATUS | R | Hardware/SERDES/DCM status |
| 0x040C | REG_40C / PULSED_CTRL | W | Pulsed control register (one-shot actions) |
| 0x040E | REG_40E / FIFO_CTRL | W | FIFO control |
| 0x0410 | REG_410 / CAPTURE_TIME | W | FIFO capture window time |
| 0x0600 | CODE_REVISION | R | Firmware revision |
| 0x0604 | CODE_DATE_MMDD | R | Firmware date (MM/DD) |
| 0x0606 | CODE_DATE_YEAR | R | Firmware year |
| 0x0702 | NIM_STATUS / REG_702 | R/W | NIM input status; NIM gating control |
| 0x0704 | ECL_STATUS_A | R | ECL input status A |
| 0x0706 | ECL_STATUS_B | R | ECL input status B |
| 0x0708 | REG_708 | R | Latched timestamp [47:32] |
| 0x070A | REG_70A | R | Latched timestamp [31:16] |
| 0x070C | REG_70C | R | Latched timestamp [15:0] |
| 0x070E | REG_70E | R | SERDES command format readback |
| 0x0710 | REG_710 | W | Coincidence trigger delay |
| 0x0712 | REG_712 | W | Coincidence trigger gate width |
| 0x0714 | REG_714 | R | Live timestamp [47:32] |
| 0x0716 | REG_716 | R | Live timestamp [31:16] |
| 0x0718 | REG_718 | R | Live timestamp [15:0] |
| 0x071A–0x0720 | REG_71A–720 | R/W | Timestamp error count control/readback |
| 0x0722 | REG_722 | W | TTCL time offset (10 ns steps applied to trigger timestamp) |
| 0x0724 | REG_724 | R | Missed trigger count |
| 0x0726 | REG_726 | R | Delayed trigger error count |
| 0x0728 | REG_728 | W | SERDES propagation control |
| 0x072C | REG_72C | W | Coincidence output control |
| 0x07EC | FIFO_COUNTER | R | Number of events written to FIFO |
| 0x07F0 | TRIG_COUNTER | R | Number of triggers received |
| 0x07F2–0x0800 | USER_COUNTERs[7:0] | R | 8× user-defined counters |

---

## Hardware Status Register (0x0020) Bit Map

```
Bit 00: STAT0_R_pin (LVDS status input 0)
Bit 01: STAT1_R_pin (LVDS status input 1)
Bit 02: SERDES_SM_LOST_LOCK_FLAG (SM ever lost lock)
Bit 03–07: reserved
Bit 08: DCM_LOCKED
Bit 09: reserved
Bit 10: SERDES_LOCK_pin (ACTIVE LOW — 0=locked, 1=unlocked)
Bit 11: UNQUALIFIED_SM_LOCKED (state machine lock, ignores stringent check)
Bit 12: serdes_isync_flag (imperative sync received)
Bit 13: serdes_sync_flag (sync received)
Bit 14: '1' (always 1, for Ron's software check — added 2014-02-11)
Bit 15: serdes_sm_locked (full lock including data pattern check)
```
(Source: `myr reg notes.txt`)

---

## Pulsed Control Register (0x040C) Bit Map

```
Bit 00: (reserved)
Bit 01: RESET timestamp logic
Bit 02: SERDES_SM_LOST_LOCK_RST (clear lost lock flag)
Bit 03: clear local counter
Bit 04: LOCAL_TS_RESET_FLAG (simulate imperative sync in local mode)
Bit 05: reset FIFO and clear capture mode
Bit 06–13: (reserved)
Bit 14: put FIFO into SERDES capture mode for CAPTURE_TIME
Bit 15: SYNC_RESET to DCM
```

---

## SERDES Config Register Bit Map

Normal operational value: `0x0063` or `0x8063`

```
Bit 00: SERDES_DEN_pin (driver enable)
Bit 01: SERDES_REN_pin (receiver enable)
Bit 02: SERDES_LINE_LE_pin (line loopback enable)
Bit 03: SERDES_LOCAL_LE_pin (local loopback enable)
Bit 04: SERDES_SYNC_pin (sync pattern control)
Bit 05: SERDES_TPWRDN_pin (TX power down, active low)
Bit 06: SERDES_RPWRDN_pin (RX power down, active low)
Bit 07: stringent_lock_flag (if set: check data pattern everywhere; if clear: only Sync+EOC)
Bit 15: clock select (0=oscillator, 1=SERDES recovered clock)
```

---

## Coincidence Logic

MyRIAD implements a **Auxiliary Detector ↔ Gammasphere coincidence window**:

- `AUX_DETECTOR_TRIG`: input from NIM IN[0] or ECL WSI (multiplexed)
- `AUX_DETECTOR_TRIG_DLY` (REG_710): delay after aux detector trigger before GS gate opens
- `MSTR_TRIG_OVERLAP_TIME` (REG_712): gate width for coincidence window
- State machine: `WAIT_FOR_AUX_DETECTOR` → `AUX_DETECTOR_DLY_WAIT`
- Counters: `COINC_TRIG_COUNTER`, `COINC_TIMEOUT_COUNTER`, `MISSED_TRIG_ERROR_COUNTER`

---

## TTCL Trigger Interface

MyRIAD receives [TTCL](ttcl.md) (Trigger, Timestamp, Command Link) messages from the DGS master trigger via SERDES:

- `TTCL_TRIG_TIMESTAMP [47:0]`: timestamp embedded in trigger packet
- `TTCL_TRIG_FLAG`: high when trigger received
- `TTCL_TRIG_TYPE [2:0]`: trigger type field
- `TTCL_TIME_OFFSET` (REG_722): delay applied to trigger timestamp before asserting NIM output (in 10 ns steps)
- Delayed trigger state machine: `IDLE → CALC_OFFSET_TIME → LET_FLAGS_SETTLE1 → LET_FLAGS_SETTLE2 → WAIT_MATCH`
- `DLYD_TTCL_TRIG_OUT`: NIM output pulse at fixed delay relative to trigger timestamp
- `DLYD_TRIG_ERROR_COUNTER`: counts triggers where delay cannot be met

---

## GITMO (Gammasphere Interface to Trigger Module)

_Source: `FPGA/others/MyRIAD/MAIN_FPGA/Source/GITMO_TOP.vhd` (795 lines) + `GITMO_RCV_MACH.vhd` (372 lines). Code-read 2026-04-27._

`GITMO_TOP.vhd` implements a historical adapter role:
- Collects clock and trigger data from an **[analog Gammasphere](analog_gammasphere.md) Master Trigger crate** via the VXI backplane
- Packs this into the DGS SERDES data stream for the Digital Gammasphere Master Trigger
- Bridges legacy GS analog trigger system to digital DGS infrastructure
- Has same dual-FIFO interface (FIFO A + FIFO B) as MyRIAD main FPGA (but FIFOs are driven to constant 0 — unused in GITMO)
- Author: John T. Anderson (ANL)

### VXI Signal Sources (GITMO)

The GITMO sits in a VXI crate and receives analog Gammasphere backplane signals:

| Signal | VXI source | Description |
|--------|-----------|-------------|
| `TRIG0_FROM_VXI` | VXI ZECLTRIG0 | Gammasphere master trigger (ECL) |
| `VXI_RDY_BSY_IN_T_pin` | VXI backplane | ADC conversion busy (active LOW) |
| `TTLTRIG_pin[0]` | VXI TTLTRIG0 | GS Run active |
| `TTLTRIG_pin[1]` | VXI TTLTRIG1 | Token Busy |
| `TTLTRIG_pin[2]` | VXI TTLTRIG2 | EOE — End of Event (abort trigger) |
| `NIM_IN_pin[7:0]` | Front panel NIM | 8 NIM inputs (arbitrary use) |
| `ECL_IN_pin[15:8]` | Front panel ECL | ECL inputs (receiver-only; driver chips U6/U10 removed) |
| `CLOCK_10_pin` | VXI | 10 MHz VXI reference clock |
| `CLOCK_100_pin` | VXI / ICS582 | Actually 50 MHz from ICS582 PLL; DCM derives 50 MHz master clock |

**ECL bus note:** GITMO has an unusual schematic error where bits 11–8 and 15–12 on the ECL connector are cross-wired through the receiver chip (U3/U5 now removed). Driver side (bits 7..0) is correct. Receiver chips U6/U10 are also not installed — bits 15..8 are receive-only. UCF pins are swapped to correct the connector bit order. ✅ verified 2026-04-27 — `GITMO_TOP.vhd:L160-213` (extensive comments + VHDL port declarations)

### Clock Architecture (GITMO)

- VXI `CLOCK_100_pin` → `IBUFG` → DCM (`GITMO_DCMS`) → **50 MHz system clock** (despite pin name, the ICS582 chip outputs 50 MHz) ✅ verified 2026-04-27 — `GITMO_DCMS` component instantiation comment: `--100MHz VXI clock pin is actually 50MHz from the ICS582`
- VXI `CLOCK_10_pin` → `IBUFG` → 10 MHz clock (drives LED blinker, SYNC pulse generator)
- SERDES TCLK: ICS502 PLL driven from `CLOCK_10_pin` (10 MHz reference). Multiplier: S1=0, S0=1 → ×5 = **50 MHz SERDES transmit clock** ✅ verified 2026-04-27 — `GITMO_TOP.vhd:L786-790` (TCLK_S0/S1 static assignments, mux table comments)
- CLOCK_SEL_pin driven to '1' → selects local oscillator (INA) for MAIN_FPGA_MACH_CLK, not SERDES RCLK ✅ verified 2026-04-27 — `GITMO_TOP.vhd:L792-794`

### SERDES Operation (GITMO)

- **SERDES_SYNC_pin forced to '0'**: originally conditionally asserted when SERDES unlocked, changed 2011-10-26 due to crosstalk issue — GITMO now always transmits, MTRG transmitter is kept off ✅ verified 2026-04-27 — `GITMO_TOP.vhd:L768-775` (comment: "When the Master Trigger sends data, causing LOCK* to be asserted in the GITMO, then the data received by the Master is corrupted")
- **SERDES_RPWRDN / TPWRDN = '1'**: both powered down — SERDES used but power-down pins held asserted (likely inverted logic or chip in use with this config) ✅ verified 2026-04-27 — `GITMO_TOP.vhd:L762-763`
- **SERDES TX data:** `SERDES_TX_pin[17:0] = '0' & COMMAND_OUT[15:0] & '0'` — 16-bit command word zero-padded at MSB and LSB ✅ verified 2026-04-27 — `GITMO_TOP.vhd:L608`
- **Trigger glitch filter:** `TRIG0_FROM_VXI_pin` passed through 2-cycle pipeline at 50 MHz → `SAMP_TRIG0 = pipe1 AND pipe2` — requires trigger to be HIGH for ≥40 ns (2 clocks). Eliminates glitches seen on VXI backplane. ✅ verified 2026-04-27 — `GITMO_TOP.vhd:L726-739`

### SYNC Pulse Generator

8-bit counter at 10 MHz → asserts `SYNC_EQUIVALENT` for 1 cycle at count 0x12, clears at 0x13, wraps at 0x13 → **2.0 µs sync pulse** (matches DGS 20-frame TTCL cycle). ✅ verified 2026-04-27 — `GITMO_TOP.vhd:L706-725` (count 0x12=18, 0x13=19 → 20 clocks × 100 ns = 2 µs)

### NIM Output Monitor Assignments

| NIM OUT pin | Signal |
|-------------|--------|
| NIM_OUT[7] | NON_CLOCK_10MHZ (VXI 10 MHz) |
| NIM_OUT[6] | SAMP_TRIG0 (glitch-filtered GS trigger) |
| NIM_OUT[5] | SYNC_EQUIVALENT (2 µs sync pulse) |
| NIM_OUT[4] | ~TTLTRIG[2] (inverted EOE) |
| NIM_OUT[3] | ~TTLTRIG[1] (inverted Token Busy) |
| NIM_OUT[2] | ~TTLTRIG[0] (inverted GS Run) |
| NIM_OUT[1] | TRIG0_FROM_VXI_pin (raw trigger, unfiltered) |
| NIM_OUT[0] | VXI_RDY_BSY_IN_T_pin |

✅ verified 2026-04-27 — `GITMO_TOP.vhd:L599-609`

### LED Assignments (GITMO front panel)

| LED | Input signal |
|-----|-------------|
| LED5 | SAMP_TRIG0 (glitch-filtered GS trigger) |
| LED6 | VXI_RDY_BSY_IN_T_pin |
| LED7 | TTLTRIG[1] (Token Busy) |
| LED8 | TTLTRIG[0] (GS Run) |
| LED9 | SERDES_LOCK_pin (active LOW = locked) |
| SERDES LED0 (RJ45) | SERDES lock (on when locked) |
| SERDES LED1 (RJ45) | DCM_STATUS[8] (DCM locked) |

✅ verified 2026-04-27 — `GITMO_TOP.vhd:L502-519, L612-615`

---

### `GITMO_RCV_MACH.vhd` — GITMO Receive State Machine (372 lines)

Implemented in the **[DGS Master Trigger](deep_fpga_MTRG_MAIN.md)** (MTRG) `MAIN_FPGA`, not in the GITMO module itself. Locks onto the SERDES data stream from the GITMO (Link L) and extracts GS trigger signals. Author: John T. Anderson, ANL, 2011-09-03.

**Purpose:** Receives the continuous 5-word-per-frame GITMO uplink and decodes Gammasphere trigger/status bits into MTRG-usable signals.

**Port interface:**
- `LINKL_RCLK` — SERDES receive clock (drives the entire machine)
- `LINKL_RX[15:0]` — raw SERDES data from GITMO
- `LINKL_LOCK` — SERDES lock (active LOW)
- Outputs: `TRIG0_FROM_VXI`, `RDY_BSY_FROM_VXI`, `GS_EOE`, `GS_TOKEN_BUSY`, `GS_RUN`, `NIM_STAT[8:1]`, `ECL_STAT[8:1]`, `FERA_STAT[8:1]`, `MACHINE_LOCKED`, `CLK_10_FLAG`
- Error flags: `FRAME_COUNT_ERROR`, `EOF_ERROR`

**GITMO word format (all 5 words per frame):**

| Bits | Meaning |
|------|---------|
| bit 15 | 10 MHz clock flag |
| bits 14–13 | Two-bit sync ordinal (fixed = "00" in current GITMO) |
| bit 12 | TRIG0_FROM_VXI — Gammasphere master trigger (ECL ZECLTRIG0) |
| bit 11 | RDY_BSY_FROM_VXI — ADC conversion busy |
| bit 10 | GS_EOE — TTLTRIG2: End of Event / abort |
| bit 9 | GS_TOKEN_BUSY — TTLTRIG1: token running |
| bit 8 | GS_RUN — TTLTRIG0: GS run active |
| bits 7–0 | Payload, rotated by word (see below) |

**Per-word payload (bits 7–0):**

| Word | Bits 7–0 |
|------|----------|
| W01 (zero-ordinal) | NIM inputs |
| W02 | ECL inputs |
| W03 | FERA control states |
| W04 | Frame counter (1–20) |
| W05 | 0xA5 (sync marker) |

**Lock sequence (two-phase):**
1. **PRELOCK1:** wait for `LINKL_LOCK = '0'` (active LOW = SERDES chip locked) → advance to PRELOCK2
2. **PRELOCK2:** scan for `bits[14:13]="00"` AND `bits[7:0]=0xA5` (W05 sync word) → enter W01
3. **W01–W05:** validate `bits[14:13]="00"` on each word; abort to PRELOCK1 if check fails
4. **MACHINE_LOCKED** asserted from W01 onward; cleared on any framing error

**Frame counter validation (W04):**
- Tracks expected consecutive frame count (1–20 wrap)
- First good W04 seeds the counter; subsequent frames compared
- Mismatch → `FRAME_COUNT_ERROR=1`, counter invalidated → re-seeds on next W04
- Counter wraps: 20 → 1 (not 0) ✅ verified 2026-04-27 — `GITMO_RCV_MACH.vhd:L308,L323`

**Key notes:**
- All trigger bits (`TRIG0_FROM_VXI`, `RDY_BSY_FROM_VXI`, GS_EOE/TOKEN_BUSY/RUN) are updated on **every word** W01–W05, not only on the word that carries the corresponding payload. Only bits 7–0 are word-specific.
- `EOF_ERROR` set when W05 does not contain 0xA5 → frame sync lost
- State machine runs on `LINKL_RCLK` (SERDES chip clock), isolated from board master clock

✅ verified 2026-04-27 — `GITMO_RCV_MACH.vhd` (372 lines, read in full)

---

## MAIN FPGA Sub-Module Deep Dives

_Source: `FPGA/others/MyRIAD/MAIN_FPGA/Source/`. Code-read 2026-04-27._

### `mstr_mach.vhd` — MyRIAD Master State Machine (968 lines)

A **satellite master trigger state machine** that generates the full 20-frame [TTCL](ttcl.md) command sequence. MyRIAD-specific adaptation of the standard DGS `mstr_mach.vhd` — same frame structure, with additions for Frame 17 (Auxiliary Detector) and satellite clock-source synchronization.

**Port highlights:**
- `CLK` (50 MHz), `RST`, `INIT_FLAG` (holds machine in init state via VME)
- `SYS_TIME[47:0]` — 48-bit master timestamp; `ROLLOVER` — timestamp rollover flag
- `IMPERATIVE_FLAG_REQ` / `LATCHED_IMPERATIVE_FLAG` output
- Trigger Decision FIFO: `TRIG_DES_FIFO_RE/DATA/EMPTY`, `TRIG_COLLECT_FLAG/RST`
- Frame command buses: `FRAME_12/14/15_REQ_FLAG`, `_SENT_FLAG`, `_DATA` (JTA_5X16_Array each)
- Async command FIFO (Frame 15): `ASYNC_CMD_FLAG/FIFO_RE/FIFO_DATA/FIFO_EMPTY`
- Aux command FIFO (Frame 17): `AUX_CMD_FLAG/FIFO_RE/FIFO_DATA/FIFO_EMPTY`
- Clock source sync: `CLOCK_SOURCE` flag + `RECEIVE_MACH_SYNC_PULSE`
- Monitor FIFO: `MSM_MON_FIFO_SELECT_REG[15:0]`, `MSM_MON_FIFO_WE`, `MSM_MON_DATA`
- `COMMAND_OUT[15:0]` to DC-balance entity

**20-frame map (5 words/frame at 50 MHz = 2 µs cycle):**

| Frame | Name | Content |
|-------|------|---------|
| F01 | SYNC | Cmd=0x01 (0x81 if imperative); 48-bit timestamp; rollover→arg=0xFF |
| F02 | Debug/Null | Null (0xAAAA); pre-fetches trigger-decision FIFO (W4/W5) |
| F03–F09 | Trigger Decision ×7 | Up to 7 trigger decisions from TRIG_DES FIFO; null if empty |
| F10 | Trigger Decision | 8th trigger decision slot (F09/F10 boundary handling) |
| F11 | Spare | Null; TRIG_COLLECT_RST at W1; arms TRIG_COLLECT_FLAG for F12/W1 |
| F12 | Internal Control | Router counter resets (W2), FIFO resets (W3), Data Generator resets (W4) |
| F13 | Demand Slow Data | Fixed: 0x40FB / A5 / 5A / A5 / A5 |
| F14 | Inter-Trigger Cmd | [Digitizer Tester](digitizer_tester.md) control (cmd/TS compare/pulse count+delay); pipelines async FIFO for F15 |
| F15 | Front End Cmd | Synchronous (FRAME_15_DATA) or async FIFO drain (5 words); null if neither |
| F16 | Spare | Null; pre-fetches first word of aux FIFO at W4/W5 |
| F17 | Auxiliary Detector | Aux FIFO drain (5 words for ancillary detector commands) |
| F18–F19 | Spare | Null; F19/W5 arms ASYNC_REQUEST and AUX_REQUEST synchronously |
| F20 | End of Cycle | 0xFFFF / 0x0000 / 0xFFFF / 0x0000 / 0x5555 |

**Key behavioral details:**
- **Imperative SYNC:** Asynchronously latched on `IMPERATIVE_FLAG_REQ`; released synchronously at F01/W5 when input is also 0. Holds timestamp counter in reset → next cycle restarts at 0.
- **Timestamp capture:** Captured at F20/W4 (end-of-cycle) → embedded in next F01/W2-4.
- **Trigger Decision FIFO pipelining:** RE pre-asserted two clocks early (F02/W4+W5). Uses `LATCHED_TRIG_DES_FIFO_EMPTY` (not live signal) to avoid missing last word.
- **Async command sync (two-flop):** `ASYNC_CMD_FLAG` (1-tick) → `ASYNC_CMD_REQUEST_INT` (immediate) → `ASYNC_CMD_REQUEST` at F19/W5 (prevents partial-frame sends) → cleared by `ASYNC_CMD_ACK` at end of F15.
- **AUX command sync:** Same two-flop pattern for Frame 17; pre-fetch begins at F16/W4.
- **Frame 12/14/15 retiming:** Request flags latched at W4 of preceding frame (F11, F13, F14) to prevent mid-frame switching.
- **Clock source mode:** `CLOCK_SOURCE=0` (remote/satellite) → F20→F01 advance only on `RECEIVE_MACH_SYNC_PULSE`; `CLOCK_SOURCE=1` (local) → advances freely.
- **Monitor FIFO select:** Bit 8 = capture everything; other bits select F01/SYNC, F20/EOC, frames-with-triggers, demand-slow, async events, aux events, non-null F12/14/15. Bit 15 selects data source: 0=COMMAND_OUT, 1=TRIG_DES_FIFO_DATA.

✅ verified 2026-04-27 — `mstr_mach.vhd` (968 lines, read in full)

---

### `SERDES_TX_MACH.vhd` — SERDES Transmit State Machine (258 lines)

Transmits MyRIAD NIM/ECL/trigger status upstream to the DGS Master Trigger at 50 MHz. **Separate** from the main TTCL command stream — carries MyRIAD's own front-panel input state.

**Structure:** 21 outer frame states (F00=wait-for-sync, F01–F20 cycling); 6 inner word states (W00=reset emits 0x0BAD, W01–W05=5 data words/frame).

**Word format (all words, at 50 MHz):**
```
Bit 15:     GITMO 10 MHz flag (always 0 here; '1' only on W01)
Bits 14-13: Two-bit sync ordinal (always "00" in this implementation)
Bit 12:     LATCHED_RAW_TRIGGER (any aux detector trigger)
Bit 11:     LATCHED_GATED_TRIGGER (gated coincidence trigger)
Bits 10-8:  3-bit word index (000-100 for W01-W05)
Bits 7-0:   Payload rotated by word:
              W01: NIM_IN[8:1]
              W02: ECL_IN[8:1]
              W03: FERA_STAT[8:1]
              W04: FRAME_COUNT[7:0] (current outer frame number)
              W05: 0xA5
```

**Trigger latching:** `RAW_TRIGGER`/`GATED_TRIGGER` from 100 MHz domain. Set asynchronously; cleared synchronously on next 50 MHz clock after capture — guarantees at least one 50 MHz frame captures the trigger even if pulse < 20 ns. ✅ verified 2026-04-27 — `SERDES_TX_MACH.vhd` (258 lines, read in full)

---

### `NIM_Delay.vhd` — NIM Delay Line (97 lines)

Entity `NIM_DELAY_LINE`: software-configurable digital delay for 2 NIM channels via on-chip RAM shift register.

- 16-bit write address `NIM_WR_ADDR` increments every clock
- Read address per channel: `NIM_RD_ADDR = NIM_WR_ADDR - reg_NIM_delay`
- 65536-entry shift register per channel (65536-bit VHDL signal → BRAM/distributed RAM)
- Delay resolution: 1 cycle = **20 ns at 50 MHz**
- Maximum delay: 65535 cycles = **1.31071 ms**
- Pipeline overhead: 2 extra cycles (input latch + output latch)
- Reset: only address counters cleared; RAM contents/output latches intentionally not reset (outputs indeterminate until first write-read pair)

✅ verified 2026-04-27 — `NIM_Delay.vhd` (97 lines, read in full)

---

### `DCBAL_in.vhd` — DC-Balance Input Decoder + Clock Domain Crossing (147 lines)

Entity `DCBAL_IN` (J.T. Anderson, ANL, Dec 2006): decodes 18-bit SERDES input (DC-balance protocol) and crosses from SERDES receive clock (`RCLK`) to board master clock (`MCLK`).

**DC-balance decode:**
- `D_IN[0]=0` → data bits [16:1] pass through as-is
- `D_IN[0]=1` → bits [16:1] were inverted for DC balance → restore by inverting
- `ENABLE=0` bypasses DC restore; `ENABLE=1` performs decode
- IOB latch (`LATCHED_D_IN`) added 2013-10-22 to force input flop to IOB for timing

**Clock-domain crossing:** `fifo_16x1023_async` (16-bit × 1023-deep). WEN always '1' (continuous write on RCLK); REN gated during reset.

**Channel reset (`CHAN_RST`) — 3-bit countdown:**
- `CHAN_RST` (1-tick from VME pulsed register) → loads `reset_count` to 7
- Counts 6→5→4: assert `reset_request`, gate off FIFO RE
- Count 0: clear request, re-enable RE
- `reset_request` crosses MCLK→RCLK via single FF (`RCLK_SAMP_RESET`); RCLK flop asynchronously clears MCLK's `reset_request` after 1 RCLK pulse
- Power-up: `reset_count` initializes to "111" → reset runs at startup (added 2013-12-09)

**`DCBAL_in_nofifo.vhd`:** Same decode logic without the async FIFO — no clock-domain crossing. ✅ verified 2026-04-27 — `DCBAL_in.vhd` (147 lines, read in full)

---

## FIFO Data Write State Machine

`FIFO_WRT_STATES`: `WAIT_TRIG → WRT_HDR_A → WRT_HDR_B → WRT1A → WRT1B → WRT2A → WRT2B → WRT3A → WRT3B → ALWAYS_CAPTURE_A → ALWAYS_CAPTURE_B`

Data written per trigger event: header (A+B words) + 3× 32-bit data words (A and B halves each)

---

## Diagnostic / Debug Features

- **Chipscope ILA** projects: multiple `.cpj` files in `MAIN_FPGA/Chipscope/` for different operating modes (FIFO mode, SERDES mode, delayed trigger, default, 80-bit ILA)
- **VIO** (Virtual I/O): 64-bit control bits + 16-bit readback data sampled every LACK edge
- ILA trigger bus: 128-bit wide
- Loopback pins: `N_OUT_DLY_OUT/IN[7:0]`, `TDC_LOOPBACK_OUT/IN` for fine-tuning ODELAY values

---

## VME FPGA Sub-Module Deep Dives

_Source: `FPGA/others/MyRIAD/VME_FPGA/Source/`. Code-read 2026-04-27._

### `vme_addr_decode.vhd` — VME Address Decoder (293 L)

A 4-state FSM (`WAIT_AS → DECODE → ASSERT_CS → END_CYCLE`) that decodes incoming VME A24/D16 cycles and generates chip-select outputs for the three addressable regions: VME FPGA internal registers, Flash memory, and MAIN FPGA.

**Operation:**
- **AM codes accepted:** 0x39 (A24 non-priv data), 0x3B (A24 non-priv BLT), 0x3D (A24 supervisory data), 0x3F (A24 supervisory BLT). All others are silently ignored (no response). ✅ verified 2026-04-27 — `vme_addr_decode.vhd:L124-131`
- **Address space:** responds to `BOARD_BASE:0000h – BOARD_BASE:FFFFh` (16 KB window); `BOARD_BASE` = 8-bit DIP switch value. ✅ verified 2026-04-27 — `vme_addr_decode.vhd:L134` (`VME_ADDR(23 downto 16) = BOARD_BASE`)
- **Clock:** 60 MHz VME bus clock; FSM uses 2-cycle AS* falling-edge detection (`delayed_AS1/AS2` pipeline).

**Chip-select decode table (address = `0xdd????`):**

| Address Range | `internal_CS` | `flash_CS` | `FPGA_CS[2:0]` | Target |
|---|---|---|---|---|
| `0xdd0000–0xdd08FF` | 0 | 0 | `001` | MAIN FPGA (registers) |
| `0xdd0900–0xdd0982` | 1 | 0 | `000` | VME FPGA internal registers |
| `0xdd0984–0xdd098E` | 0 | 1 | `000` | Flash memory |
| `0xdd0990–0xdd09FF` | 0 | 0 | `000` | Unused / no response |
| `0xdd0A00–0xdd0FFF` | 0 | 0 | `000` | No response |
| `0xdd1000–0xdd3FFF` | 0 | 0 | `010` | MAIN FPGA (FIFO) |
| `0xdd4000–0xdd5FFF` | 0 | 0 | `011` | MAIN FPGA (excess decode 011) |
| `0xdd6000–0xdd7FFF` | 0 | 0 | `100` | MAIN FPGA (excess decode 100) |
| `0xdd8000–0xdd9FFF` | 0 | 0 | `101` | MAIN FPGA (excess decode 101) |
| `0xddA000–0xddAFFF` | 0 | 0 | `110` | MAIN FPGA (excess decode 110) |
| `0xddB000–0xddDFFF` | 0 | 0 | `000` | No response |
| `0xddE000–0xddEFFF` | 0 | 0 | `111` | MAIN FPGA (excess decode 111) |
| `0xddF000–0xddFFFF` | 0 | 0 | `000` | No response |

✅ verified 2026-04-27 — `vme_addr_decode.vhd:L181-241` (full case statement)

**Block transfer support:** BLT AM codes (0x3B/0x3F) set `BLOCK_XFR_FLAG`. In BLT mode, `END_CYCLE` loops back to `DECODE` on each successive DS* assertion (rather than waiting for full AS* release), enabling multi-word transfers at full VME speed. ✅ verified 2026-04-27 — `vme_addr_decode.vhd:L108-113, L279-293`

---

### `external_bus_controller.vhd` — External Bus Controller (285 L)

An 8-state FSM that arbitrates between VME cycles and FPGA configuration accesses on the shared external bus (Flash memory + MAIN FPGA). Runs at **50 MHz** (20 ns/cycle).

**States:** `IDLE → CONFIGURE | WRITE_FLASH_INDIRECT | READ_FLASH_INDIRECT | WRITE_MAIN_FPGA | READ_MAIN_FPGA | READ_EXTERNAL_FIFO | HOLD_DTACK`

**Priority:** configuration request (`config_request`) takes highest priority; VME flash/FPGA cycles run from IDLE only when no config is pending. ✅ verified 2026-04-27 — `external_bus_controller.vhd:L117-122`

**VME vs. Flash vs. FPGA routing:**
- **Flash read** (`READ_FLASH_INDIRECT`): 10-cycle sequence; address comes from `flash_reg_addr` register (not from VME address lines); data returned to VME after 8-cycle `flash_OE` assert. ✅ verified 2026-04-27 — `external_bus_controller.vhd:L186-201`
- **Flash write** (`WRITE_FLASH_INDIRECT`): 14-cycle sequence; address from register; WE* pulse at cycles 6–12. ✅ verified 2026-04-27 — `external_bus_controller.vhd:L163-180`
- **MAIN FPGA read** (`READ_MAIN_FPGA`): asserts `fpga_strb` at delay_count=2; waits for `fpga_ack` pipeline (2-stage `fpga_ack_p1/p2`) to capture data; times out at delay_count=14. ✅ verified 2026-04-27 — `external_bus_controller.vhd:L204-229`
- **MAIN FPGA write** (`WRITE_MAIN_FPGA`): asserts address at cycle 0, data at cycle 1, `fpga_strb` at cycle 4; done at cycle 10. ✅ verified 2026-04-27 — `external_bus_controller.vhd:L232-248`
- **FIFO read** (`READ_EXTERNAL_FIFO`, `FPGA_CS="010"`, added 2013-03-05): identical to `READ_MAIN_FPGA` but in a separate state for future differentiation. ✅ verified 2026-04-27 — `external_bus_controller.vhd:L251-273`
- **Configuration** (`CONFIGURE`): passes `cnfg_addr_in` from configuration_controller as flash address; drives `cnfg_data_out ← external_data_in`; holds flash CE/OE asserted; stays until `config_request` deasserts. ✅ verified 2026-04-27 — `external_bus_controller.vhd:L135-153`
- **HOLD_DTACK**: holds DTACK until the address decoder releases chip-selects (end of VME cycle), then returns to IDLE.

**Direction control:** `external_ddir` output controls bidirectional bus buffer direction (`'0'` = output/write, `'1'` = input/read). Default is input.

---

### `configuration_controller.vhd` — FPGA Configuration FSM (349 L)

Controls automatic configuration of the **MAIN FPGA** from Flash memory on power-up (and optionally from VME command). Uses a 12-state FSM that serializes 16-bit Flash words MSB-first to the MAIN FPGA's SelectMap/serial configuration pins.

**States:** `STARTUP → ASSERT_PRGM → HOLD_PRGM → RELEASE_PRGM → WAIT_INIT → READ_WORD → SHIFT_DATA_A → SHIFT_DATA_B → SHIFT_DATA_C → ERROR | DONE → IDLE`

**Configuration bitstream layout in Flash:**
- Starts at address `0x000000` (hard-coded constant `dcf_cnfg_start_addr_i`)
- Stop address: `0x180000` (a round number above the Spartan-3 XC3S5000 bitstream size of ~1,659K words; machine exits early when FPGA asserts DONE). ✅ verified 2026-04-27 — `configuration_controller.vhd:L76-82`
- Diagnostic firmware region: `0x400000–0x7FFFFEh` (defined but not used in normal operation)

**Sequence:**
1. **STARTUP**: reset counters, go to ASSERT_PRGM
2. **ASSERT_PRGM**: hold `fpga_pgm='0'` for 32 cycles (≈1.28 µs; spec: ≥500 ns); also holds external bus via `configuration_flag='1'` to claim Flash access
3. **HOLD_PRGM**: extend PGM assert using `pgm_count` counter (32 × 40 ns = 1.28 µs additional hold)
4. **RELEASE_PRGM**: raise PGM; hold for 12 more cycles; load `init_count = 125000` (5 ms timeout for INIT)
5. **WAIT_INIT**: wait for `fpga_init_in='1'` (MAIN FPGA INIT assertion indicates internal clear complete). Timeout after 125,000 × 20 ns = 5 ms → ERROR if INIT never asserts. ✅ verified 2026-04-27 — `configuration_controller.vhd:L218-228`
6. **READ_WORD**: latch 16 bits from Flash; check if `fpga_done='1'` → DONE. Load `shift_count=16`.
7. **SHIFT_DATA_A/B/C**: shift one bit per iteration (A=data setup, B=CCLK high, C=CCLK low + shift register advance). Address incremented 4 cycles before the word boundary to meet Flash access timing. 16 bits per word = 16 iterations per Flash word. ✅ verified 2026-04-27 — `configuration_controller.vhd:L280-299`
8. **DONE**: configuration complete; deassert `configuration_flag`; assert `config_complete`. Wait for VME software to acknowledge (`config_done_ack='1'`) before going IDLE.
9. **ERROR**: same as DONE but also asserts `config_error`. Triggered by address overflow or INIT timeout.
10. **IDLE**: quiescent; re-triggered by `request_config='1'` from VME register write.

**Key design note:** CCLK is generated directly by this FSM (not by buffering the master clock) per Xilinx app note #5154. `fpga_init_out` is tied high and `fpga_init_t` is tied high (init pin used as input only — "big hack 2009-03-27"). ✅ verified 2026-04-27 — `configuration_controller.vhd:L96-97`

---

## Cross-References

**Role in system:**
- Receives TTCL data stream from MTRG (DGS Master Trigger) via SERDES
- Sends event data via VME bus (through VME FPGA)
- Can operate in DGS Master, DGS Router, or GRETINA Master mode
- GITMO variant bridges analog GS master trigger crate to digital DGS trigger infrastructure
- Not to be confused with MTRG or RTRG — MyRIAD is an *auxiliary* expansion module, not a primary trigger processor

**Related files:**
- [`ttcl.md`](ttcl.md) — TTCL protocol specification; MyRIAD is a TTCL client that also sends triggers back to MTRG via Link U
- [`deep_fpga_MTRG_MAIN.md`](deep_fpga_MTRG_MAIN.md) — MTRG Main FPGA that drives the TTCL stream MyRIAD receives; also handles MYRIAD_RCV_MACH and MYRIAD_TRIGGER algorithms
- [`fpga.md`](fpga.md) — FPGA firmware overview; firmware type codes including 0xB (MyRIAD)
- [`VME_registers.md`](VME_registers.md) — VME register map reference for DIG/MTRG/RTRG boards
- [`XIA_1SFP.md`](XIA_1SFP.md) — Another auxiliary SERDES-based trigger interface (receive-only; no Link U back-channel unlike MyRIAD)
- [`analog_gammasphere.md`](analog_gammasphere.md) — GITMO context: MyRIAD GITMO bridges analog master trigger crate to DGS
- [`multi_system_linking.md`](multi_system_linking.md) — Cross-system trigger sharing; Link U used by MyRIAD
