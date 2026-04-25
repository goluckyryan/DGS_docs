# MyRIAD — Trigger Expansion Module

**Stability: C3 - Structural / stable**

Source: `DGS_tools_pack/FPGA/others/MyRIAD/`

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

MyRIAD receives TTCL (Trigger, Timestamp, Command Link) messages from the DGS master trigger via SERDES:

- `TTCL_TRIG_TIMESTAMP [47:0]`: timestamp embedded in trigger packet
- `TTCL_TRIG_FLAG`: high when trigger received
- `TTCL_TRIG_TYPE [2:0]`: trigger type field
- `TTCL_TIME_OFFSET` (REG_722): delay applied to trigger timestamp before asserting NIM output (in 10 ns steps)
- Delayed trigger state machine: `IDLE → CALC_OFFSET_TIME → LET_FLAGS_SETTLE1 → LET_FLAGS_SETTLE2 → WAIT_MATCH`
- `DLYD_TTCL_TRIG_OUT`: NIM output pulse at fixed delay relative to trigger timestamp
- `DLYD_TRIG_ERROR_COUNTER`: counts triggers where delay cannot be met

---

## GITMO (Gammasphere Interface to Trigger Module)

`GITMO_TOP.vhd` implements a historical adapter role:
- Collects clock and trigger data from an **analog Gammasphere Master Trigger crate**
- Packs this into the DGS SERDES data stream for the Digital Gammasphere Master Trigger
- Bridges legacy GS analog trigger system to digital DGS infrastructure
- Has same dual-FIFO interface (FIFO A + FIFO B) as MyRIAD main FPGA

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

## Relationship to Other DGS Components

- Receives TTCL data stream from MTRG (DGS Master Trigger) via SERDES
- Sends event data via VME bus (through VME FPGA)
- Can operate in DGS Master, DGS Router, or GRETINA Master mode
- GITMO variant bridges analog GS master trigger crate to digital DGS trigger infrastructure
- Not to be confused with MTRG (deep_fpga_MTRG.md) or RTRG (deep_fpga_RTRG.md) — MyRIAD is an *auxiliary* expansion module, not a primary trigger processor
