# MγRIAD — Multipurpose γ-Ray Interface to Auxiliary Detectors

**Full name:** Multipurpose γ-Ray Interface to Auxiliary Detectors  
**Source:** `DGS_SVN/dgs/MyRIAD/Documentation/MyRIAD Abridged User Notes.pdf` (v0.0, 2018-04-27, J. Anderson) + `MyRIAD User Manaual.pdf` (v1.2, March 2015, J. Anderson — full register map in Section 3)  
**Also:** `DGS_SVN/dgs/MyRIAD/Documentation/MYRIAD_Module_Specification.pdf`  
**Wiki PDFs:** `https://wiki.anl.gov/wiki_gsdaq/images/4/40/MyRIAD_User_Manaual.pdf`

---

## Purpose

The MγRIAD bridges auxiliary detectors (NIM/ECL-based) to the DGS/GRETINA TTCL trigger system. It:
- Receives the TTCL timestamp from the Master Trigger via RJ45 (Cat5e, same as digitizer)
- Propagates DGS/GRETINA timestamps to auxiliary VME-based DAQs
- Sends trigger messages back to the Master Trigger when the auxiliary detector fires
- Provides local coincidence logic between local detector and auxiliary trigger
- Interfaces legacy FERA ADC systems via ECL

Connected to **link U** of the Master Trigger. ✅ verified 2026-04-07 — `top.vhd:L3562` (`RECEIVED_MYRIAD_DATA => LAT_LINKU_RX`).

---

## Front Panel Connectors

### RJ45 — TTCL Link
- **Not Ethernet** — Cat5e cable to DGS/GRETINA trigger module only
- Carries TTCL (Trigger Timing and Control Link) — same protocol as digitizer RJ45
- LEDs on connector indicate SERDES link lock state

#### MγRIAD → MTRG SERDES Data Frame Format
_Source: `MYRIAD_RCV_MACH.vhd` header comment ✅ verified 2026-04-07_

MγRIAD sends a **5-word repeating frame** over SERDES (16-bit words):

| Word | Bits 15:13 | Bit 12 | Bit 11 | Bits 10:8 | Bits 7:0 |
|------|-----------|--------|--------|-----------|----------|
| All  | bit15=10MHz flag, bits14:13=00 | Raw trigger (NIM or ECL input) | Gated trigger (coincidence logic output) | 3-bit ordinal counter (sync check) | word-specific (below) |
| 00 | — | — | — | — | `0xAD` (part of reset sync `0x0BAD`) |
| 01 | — | — | — | — | NIM input states (8 bits) |
| 02 | — | — | — | — | ECL input states |
| 03 | — | — | — | — | FERA control states |
| 04 | — | — | — | — | 8-bit frame counter (sync check) |

MTRG receiver (`MYRIAD_RCV_MACH`) locks onto this stream and extracts NIM/ECL state, raw trigger, gated trigger, and 10 MHz flag. Reset detected by `0x0BAD` signature in word 00.

### JTAG
- Direct FPGA access via Xilinx JTAG programmer

### ECL CTL Header (10-pin, 2 differential inputs + 3 differential outputs)
- Pinout compatible with FERA ADC system cables ✅ verified 2026-04-06 — MyRIAD Abridged User Notes.pdf p.3
- Firmware-defined; current DGS build uses ECL CTL outputs as diagnostics:
  - **FERA FULL** pair → copy of multiplexed 50 MHz FPGA clock (locked = TTCL sync OK) ✅ verified 2026-04-06 — MyRIAD Abridged User Notes.pdf p.3
  - **FERA ACK** pair → copy of NIM input 1 (level translator) ✅ verified 2026-04-06 — MyRIAD Abridged User Notes.pdf p.3
  - **FERA OVF** pair → copy of MyRIAD's internal 50 MHz oscillator (differs from FERA FULL when CLOCK_SEL selects SerDes clock)
  - **FERA WSI** pair (input) → alternate source for local trigger signal (firmware-selectable vs NIM input 0)
  - **FERA VETO** pair (input) → readable from status register for testing; no active function in current firmware

### ECL I/O Header (16 differential ECL signals)
- Default: 16 receiver inputs (100 Ω termination per differential pair)
- Assembly positions allow installing driver chips instead (reconfigurable)
- All signals available to firmware

### NIM I/O — 8 inputs, 4 outputs
Layout: two groups of 4 inputs (I), two groups of 2 outputs (O). ✅ verified 2026-04-06 — MyRIAD Abridged User Notes.pdf p.5 Fig.1

#### NIM Input Functions
| Input | Function |
|-------|---------|
| **NIM In 0** (upper left) | **Local system trigger input** — latches timestamp on each edge; minimum pulse width **100 ns** ✅ verified 2026-04-08 — MYRIAD_Module_Specification.pdf ("100ns wide should be used") |
| **NIM In 1** | Local coincidence input — starts coincidence timer after NIM In 0 edge; asserts coincidence if NIM In 1 fires before timeout |
| NIM In 2–7 | General purpose — counted only (no trigger function as of 2018-04-27) |

- All NIM inputs connected to 16-bit edge counters (sampled at 100 MHz) ✅ verified 2026-04-06 — MyRIAD Abridged User Notes.pdf p.5
- All NIM input states regularly sent to Master Trigger over SERDES ✅ verified 2026-04-06 — MyRIAD Abridged User Notes.pdf p.5

#### NIM Output Functions

| Output | Function |
|--------|----------|
| **NIM Out 0** | Echoed copy of NIM In 0 (fixed, not configurable) ✅ verified 2026-04-08 — MYRIAD_Module_Specification.pdf §3.5 |
| **NIM Out 1** | Buffered copy of NIM In 1 (fixed, not configurable) ✅ verified 2026-04-08 — MYRIAD_Module_Specification.pdf §3.5 |
| **NIM Out 2** | Copy of 'sync flag' from DGS/Gretina master trigger over SERDES; pulses every 2 µs — useful for timing verification ✅ verified 2026-04-08 — MYRIAD_Module_Specification.pdf §3.5 |
| **NIM Out 3** | Coincidence logic output (fires when NIM In 1 occurs within coincidence window after starting trigger) ✅ verified 2026-04-08 — MYRIAD_Module_Specification.pdf §3.4 |

> Note: GATING_REG (0x0702) bits 0–1 select the coincidence *starting trigger* source (bit 0 = NIM In 0, bit 1 = SERDES trigger from master). The NIM Out 0/1 outputs are hardwired echoes — they are not muxed by GATING_REG. Prior doc claiming bits 3:2 and 6:4 controlled NIM Out 0/1 selection was incorrect.

---

## Front Panel LEDs (3×3 array)

```
P1  P2  V4
F3  F5  S6
S7  S8  S9
```

| LED | Label | Meaning |
|-----|-------|---------|
| 1 | P | +5V VME power present (blue) |
| 2 | P | DC-DC converter subsidiary voltages OK (green) |
| 4 | V | VME access activity (flashes on each VME access) |
| 3 | F | FPGA configuration in progress (blinks) |
| 5 | F | Firmware-specific main FPGA indicator (currently unused) |
| 6 | S | Blinks when internal coincidence logic satisfied |
| 7 | S | Blinks on NIM input 1 leading edges |
| 8 | S | Blinks on local detector TRIGGER IN edges (NIM In 0 or ECL FERA WSI, firmware-selectable) |
| 9 | S | Blinks on NIM input 7 leading edges |

---

## DGS Usage

In DGS, MγRIAD is connected to **link U** of the MTRG: ✅ verified 2026-04-07 — `MTRG/top.vhd:L3562`
- Receives TTCL timestamps → propagates to auxiliary VME DAQs
- Sends auxiliary detector trigger messages back to MTRG
- Local NIM input 0 = aux detector trigger (e.g. ancillary detector, tape station)
- Local NIM input 1 = coincidence gate signal

Coincidence timer is programmable via register — window defines valid auxiliary trigger window relative to NIM In 0.

---

## VME Register Map
_Source: MyRIAD User Manual v1.2, 2015/2018, J. Anderson — Section 3.2/3.3_

All registers are 16-bit, A16/D16. Addresses in hex.

| Address | Mode | Register | Function |
|---------|------|----------|----------|
| 0x0000 | R | `board_id` | Board address + firmware info |
| 0x0004 | RW | `fifo_status` | FIFO status |
| 0x0020 | R | `hardware_status` | DCM status information |
| 0x040C | W | `pulsed_control` | Write-only self-clearing: resets |
| 0x040E | RW | `fifo_control` | FIFO operational modes |
| 0x0410 | RW | `Capture_time` | SerDes capture time (FIFO test) |
| 0x0600 | R | `code_revision` | Firmware revision |
| 0x0604 | R | `code_date` | Compilation date (MMDD) |
| 0x0606 | R | `code_year` | Compilation year (YYYY) |
| 0x0700 | R | `NIM_input_status` | Current state of all NIM inputs |
| 0x0702 | RW | `GATING_REG` | Coincidence starting trigger select: bit 0 = use NIM In 0 as starting trigger; bit 1 = use SERDES trigger from master as starting trigger. (NIM Out 0/1 are hardwired echoes, not controlled by this register.) ✅ verified 2026-04-08 — MYRIAD_Module_Specification.pdf §3.4 |
| 0x0704 | R | `ECL_input_status_A` | Current state of ECL data inputs |
| 0x0706 | R | `ECL_input_status_B` | Current state of ECL control inputs |
| 0x0708 | R | `LATCHED_TIMESTAMP_A` | Timestamp bits 47:32 (latched on NIM In 0) |
| 0x070A | R | `LATCHED_TIMESTAMP_B` | Timestamp bits 31:16 |
| 0x070C | R | `LATCHED_TIMESTAMP_C` | Timestamp bits 15:0 |
| 0x070E | RW | `SerDes_COMMAND_FORMAT` | Select DGS or GRETINA command format |
| 0x0710 | R | `Coincidence_window_delay` | Delay after local trigger before coincidence window opens (×20 ns) |
| 0x0712 | R | `coincidence_window_width` | Width of coincidence gate for GS trigger match |
| 0x0714 | R | `LIVE_TIMESTAMP_A` | Running timestamp bits 47:32 |
| 0x0716 | R | `LIVE_TIMESTAMP_B` | Running timestamp bits 31:16 |
| 0x0718 | R | `LIVE_TIMESTAMP_C` | Running timestamp bits 15:0 |
| 0x071A | RW | `TIMESTAMP_ERROR_CTRL` | Reset timestamp error counter |
| 0x071E | R | `TIMESTAMP_ERROR_CNT_A` | Timestamp error counter bits 31:16 |
| 0x0720 | R | `TIMESTAMP_ERROR_CNT_B` | Timestamp error counter bits 15:0 |
| 0x0722 | RW | `TTCL_TIME_OFFSET` | Master trigger re-issue offset control |
| 0x0724 | R | `MISSED_TRIG_COUNT` | Counter of missed re-issued trigger messages |
| 0x0726 | R | `DLYD_TRIG_ERR_COUNT` | Counter of re-issued trigger errors |
| 0x0728 | RW | `PROPAGATION_CONTROL` | Controls SerDes command processing |
| 0x07EC | R | `FIFO_COUNTER` | Number of triggers stored in FIFO |
| 0x07EE | R | `TRIG_COUNTER` | Total triggers received |
| 0x07F2–0x07Fe | R | `USER_COUNTER_0–6` | Edge counters for NIM inputs 0–6 |
| 0x0800 | R | `USER_COUNTER_7` | Edge counter for NIM input 7 |
| 0x0848 | RW | `sd_config` | SerDes configuration register |
| 0x0860–0x0866 | R | RESERVED | Reserved for future use |
| 0x0900 | RW | `fpga_ctrl_reg` | Main FPGA configuration control |
| 0x0902 | R | `vme_status` | VME FPGA status |
| 0x0904 | R | `vme_aux_status` | VME FPGA auxiliary status |
| 0x0906 | R | `spare_registers` | Spare/reserved |
| 0x0908 | RW | `flash_vpen` | Flash write-protect enable |
| 0x090A | RW | `config_start_low` | FPGA config start address (low) |
| 0x090C | RW | `config_stop_high` | FPGA config stop address (high) |
| 0x090E | RW | `config_stop_low` | FPGA config stop address (low) |
| 0x0910 | RW | `config_start_high` | FPGA config start address (high) |
| 0x0918 | RW | `vme_sandbox1` | VME sandbox register 1 (debug) |
| 0x091A | RW | `vme_sandbox2` | VME sandbox register 2 (debug) |
| 0x091C | RW | `vme_sandbox3` | VME sandbox register 3 (debug) |
| 0x091E | R | `vme_sandbox4` | VME sandbox register 4 (debug) |
| 0x1000 | R | `fifo` | FIFO read port — each read returns one 16-bit word from the trigger FIFO |

_Note: Above 0x0902 entries sourced from `MYRIAD_Module_Specification.pdf` (v1.0, 2014). Some may differ from User Manual v1.2 (2015)._

### Coincidence Logic
State machine: fires when "starting trigger" occurs (NIM In 0 if `GATING_REG[0]`=1, or SERDES trigger if `GATING_REG[1]`=1). After `Coincidence_window_delay` × 20 ns, opens a window of width `coincidence_window_width`. If NIM In 1 fires during window → NIM Out 3 asserts. Timeout without match → return to idle.

---

## Related Files
- `connectors.md` — links R and U on MTRG where MγRIAD connects
- `ttcl.md` — TTCL protocol MγRIAD uses for trigger communication
- `wiki_gsdaq.md` — MγRIAD mentioned in DAQ system overview

## SVN Location
`DGS_tools_pack/DGS_SVN/dgs/MyRIAD/`

---

*Created: 2026-04-05 (from SVN MyRIAD Abridged User Notes PDF)*
