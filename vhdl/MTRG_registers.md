# MTRG VME Register Map (registers.vhd)

Stability: C3 - Structural / stable

Source: `~/FPGA_svn2git/MTRG_git/MAIN_FPGA/trunk/Source/registers.vhd`  
Entity: `registers`  
Last analyzed: 2026-04-24

---

## Table of Contents

- [Overview](#overview)
  - [Chip-Select (CS) Address Map](#chip-select-cs-address-map)
- [Read-Only Registers (CS=001, 0x0100–0x02FF)](#read-only-registers-cs001-0x0100-0x02ff)
- [Read/Write Registers — Block 0x01E0–0x02CC (CS=001)](#readwrite-registers--block-0x01e0-0x02cc-cs001)
- [Read/Write Registers — Main Block 0x0800–0x08EC (CS=001)](#readwrite-registers--main-block-0x0800-0x08ec-cs001)
- [Special Addresses](#special-addresses)
- [Embedded Lookup RAMs (DPRAM, CS=001)](#embedded-lookup-rams-dpram-cs001)
- [Monitor FIFOs](#monitor-fifos)
- [VME Bus State Machine](#vme-bus-state-machine)
- [Rate Counters](#rate-counters)
- [Notable Defaults](#notable-defaults)
- [Cross-References](#cross-references)

---

## Overview

`registers.vhd` (internally titled `assy_registers.vhd`) implements **all VME-accessible control and status registers** for the MTRG (Master Trigger) FPGA. It is the single register file for the entire MTRG board, handling reads, writes, FIFOs, and lookup RAMs via a VME bus state machine.

### Chip-Select (CS) Address Map

| CS[2:0] | Address Range | Region |
|---------|--------------|--------|
| `001`   | 0x0000–0x08FC | **Main FPGA registers** (R/W and R/O) |
| `000`   | 0x0900–0x09FC | Reserved — VME FPGA block |
| `000`   | 0x0A00–0x0FFC | Unused |
| `010`   | 0x1000–0x3FFC | Main FPGA expansion registers |
| `011`   | 0x4000–0x5FFC | Reserved |
| `100`   | 0x6000–0x7FFC | Reserved |
| `101`   | 0x8000–0x9FFC | Reserved |
| `110`   | 0xA000–0xAFFC | Reserved |
| `000`   | 0xB000–0xDFFC | Unused |
| `111`   | 0xE000–0xEFFC | **Fast strobe CPLD** |
| `000`   | 0xF000–0xFFFC | Unused |

Within CS=001: **R/W registers at 0x0800–0x08FC**; **R/O registers at 0x0100–0x02FF**.

---

## Read-Only Registers (CS=001, 0x0100–0x02FF)

| Address | Signal Name | Description |
|---------|------------|-------------|
| 0x0100  | reg_LOCK_BUS | LOCK* status from all SERDES chips (16-bit bitmap) |
| 0x0104  | reg_DEN_BUS | DEN signal driven to all SERDES chips |
| 0x0108  | reg_REN_BUS | REN signal driven to all SERDES chips |
| 0x010C  | reg_SYNC_BUS | SYNC signal driven to all SERDES chips |
| 0x0110  | Unused_0110 | Unused |
| 0x0114  | REG_114_IN | Timestamp bits [47:32] |
| 0x0118  | REG_118_IN | Timestamp bits [31:16] |
| 0x011C  | REG_11C_IN | Timestamp bits [15:0] |
| 0x0120  | reg_MSTR_MACH_STATE | Snapshot of Master State Machine current state |
| 0x0124  | reg_AUX_INPUT_STATE | State of auxiliary inputs |
| 0x0128  | reg_MISC_STAT | Miscellaneous status (NIM inputs, etc.) |
| 0x012C  | REG_12C_IN | DIAG_COUNTER(1) |
| 0x0130  | REG_130_IN | DIAG_COUNTER(2) |
| 0x0134  | REG_134_IN | DIAG_COUNTER(3) |
| 0x0138  | REG_138_IN | DIAG_COUNTER(4) |
| 0x013C  | REG_13C_IN | DIAG_COUNTER(5) |
| 0x0140  | REG_140_IN | DIAG_COUNTER(6) |
| 0x0144  | REG_144_IN | DIAG_COUNTER(7) |
| 0x0148  | REG_148_IN | DIAG_COUNTER(8) |
| 0x014C  | reg_LINK_LRU_MACH_STAT | Engineering diagnostic bits (definition not fixed) |
| 0x0150  | reg_MISC_STAT2 | Additional miscellaneous status bits |
| 0x0158  | reg_CODE_DATE | Code modification date |
| 0x015C  | reg_CODE_REVISION | PCB/Firmware revision number |
| 0x01B0  | reg_SYSTEM_THROTTLE_MAP | Bitmap of router throttle requests |
| 0x01B8  | reg_UNUSED_1B8 | Unused |
| 0x01BC  | reg_SUM_CONN_BUF_MON | SUM connector monitor |
| 0x01C0  | reg_UNUSED_1C0 | Unused |
| 0x01C4  | reg_FRAME_12_CMD_CNT | Count of Frame 12 (internal trigger) commands issued |
| 0x01C8  | reg_FRAME_14_CMD_CNT | Count of Frame 14 (trigger frame) commands issued |
| 0x01CC  | reg_FRAME_16_CMD_CNT | Count of Frame 16 (sync capture) commands issued |
| 0x01D0  | reg_FRAME_17_CMD_CNT | Count of Frame 17 commands issued |
| 0x01D4–0x01DC | reg_UNUSED_1D4/8/C | Unused |
| 0x0280  | reg_LATCHED_TS_HIGH | Latched timestamp [47:32] (IOC readout assist) |
| 0x0284  | reg_LATCHED_TS_MID | Latched timestamp [31:16] |
| 0x0288  | reg_LATCHED_TS_LOW | Latched timestamp [15:0] |
| 0x028C  | reg_Unused_28C_RO | Unused |

---

## Read/Write Registers — Block 0x01E0–0x02CC (CS=001)

These are writable control registers in the lower address block:

| Address | Signal Name | Default | Description |
|---------|------------|---------|-------------|
| 0x01E0  | reg_STARTING_TIMESTAMP_HI | 0x0000 | Timestamp hi-word for Imperative Sync |
| 0x01E4  | reg_STARTING_TIMESTAMP_MID | 0x0000 | Timestamp mid-word for Imperative Sync |
| 0x01E8  | reg_STARTING_TIMESTAMP_LOW | 0x0000 | Timestamp lo-word for Imperative Sync |
| 0x01EC  | reg_FRAME_17_DATA_1 | 0x0000 | Frame 17 word 1 |
| 0x01F0  | reg_FRAME_17_DATA_2 | 0x0000 | Frame 17 word 2 |
| 0x01F4  | reg_FRAME_17_DATA_3 | 0x0000 | Frame 17 word 3 |
| 0x01F8  | reg_FRAME_17_DATA_4 | 0x0000 | Frame 17 word 4 |
| 0x01FC  | reg_FRAME_17_DATA_5 | 0x0000 | Frame 17 word 5 |
| 0x0200  | reg_ENCODER_SOURCE_SELECT | 0x0000 | Use timestamp as encoder (diagnostics) |
| 0x0204  | reg_MYRIAD_TRIG_DELAY | 0x0001 | Delay applied to MyRIAD trigger messages |
| 0x0208  | reg_MYRIAD_OVERLAP_CTL | 0x0000 | MyRIAD trigger algorithm control |
| 0x020C  | reg_MON7_FILL_CTL | 0x0086 | System monitor FIFO control |
| 0x0210  | reg_ENCODER_TEST | 0x0000 | CPLD testing register |
| 0x0214  | reg_USER_PACKAGE_DATA | 0x0123 | User package data |
| 0x0218  | reg_TDC_TRIG_SEL | 0x0000 | Selects which trigger saves system monitor data |
| 0x021C  | reg_TRIG_ALGO_MUX_SEL | 0x0001 | Select trigger modes for links L, R, and U |
| 0x0220  | reg_TRIGGER_PRESCALE_A | 0x0000 | Prescale for algorithm 1 |
| 0x0224  | reg_TRIGGER_PRESCALE_B | 0x0000 | Prescale for algorithm 2 |
| 0x0228  | reg_TRIGGER_PRESCALE_C | 0x0000 | Prescale for algorithm 3 |
| 0x022C  | reg_TRIGGER_PRESCALE_D | 0x0000 | Prescale for algorithm 4 |
| 0x0230  | reg_TRIGGER_PRESCALE_E | 0x0000 | Prescale for algorithm 5 |
| 0x0234  | reg_TRIGGER_PRESCALE_F | 0x0000 | Prescale for algorithm 6 |
| 0x0238  | reg_TRIGGER_PRESCALE_G | 0x0000 | Prescale for algorithm 7 |
| 0x023C  | reg_TRIGGER_PRESCALE_H | 0x0000 | Prescale for algorithm 8 |
| 0x0240  | reg_REMOTE_TRIGGER_TS_OFFSET_L | 0x0000 | Timestamp offset for remote triggers (Link L) |
| 0x0244  | reg_REMOTE_TRIG_DELAY_L | 0x0000 | Delay for remote trigger messages (Link L) |
| 0x0248  | reg_REMOTE_TRIG_OVERLAP_CTL_L | 0x0000 | Remote trigger algorithm control (Link L) |
| 0x024C  | reg_REMOTE_TRIG_DIG_OFFSET_L | 0x0000 | Secondary TS offset for remote triggers (Link L) |
| 0x0250  | reg_TRIG_VETO_SELECT_A | 0x000F | Veto enables — algorithm 1 |
| 0x0254  | reg_TRIG_VETO_SELECT_B | 0x000F | Veto enables — algorithm 2 |
| 0x0258  | reg_TRIG_VETO_SELECT_C | 0x000F | Veto enables — algorithm 3 |
| 0x025C  | reg_TRIG_VETO_SELECT_D | 0x000F | Veto enables — algorithm 4 |
| 0x0260  | reg_TRIG_VETO_SELECT_E | 0x000F | Veto enables — algorithm 5 |
| 0x0264  | reg_TRIG_VETO_SELECT_F | 0x000F | Veto enables — algorithm 6 |
| 0x0268  | reg_TRIG_VETO_SELECT_G | 0x000F | Veto enables — algorithm 7 |
| 0x026C  | reg_TRIG_VETO_SELECT_H | 0x000F | Veto enables — algorithm 8 |
| 0x0270  | reg_LOCAL_TRIG_DELAY_L | 0x0001 | Delay for local triggers into remote trig algos (Link L) |
| 0x0274  | reg_CPLD_EXTRA | 0x5504 | Controls revision D sum buffer chip |
| 0x0278  | reg_SSI_CTL | 0x0008 | SSI encoder interface control |
| 0x027C  | reg_SSI_ENCODE_TIME | 0x0100 | SSI synchronization timeout |
| 0x0290  | reg_REMOTE_TRIG_OVERLAP_CTL_R | 0x0000 | Remote trigger algorithm control (Link R) |
| 0x0294  | reg_REMOTE_TRIG_OVERLAP_CTL_U | 0x0000 | Remote trigger algorithm control (Link U) |
| 0x0298  | reg_REMOTE_TRIGGER_TS_OFFSET_R | 0x0000 | Timestamp offset for remote triggers (Link R) |
| 0x029C  | reg_REMOTE_TRIGGER_TS_OFFSET_U | 0x0000 | Timestamp offset for remote triggers (Link U) |
| 0x02A0  | reg_REMOTE_TRIG_DIG_OFFSET_R | 0x0000 | Secondary TS offset for remote triggers (Link R) |
| 0x02A4  | reg_REMOTE_TRIG_DIG_OFFSET_U | 0x0000 | Secondary TS offset for remote triggers (Link U) |
| 0x02A8  | reg_REMOTE_TRIG_DELAY_R | 0x0000 | Delay for remote trigger messages (Link R) |
| 0x02AC  | reg_REMOTE_TRIG_DELAY_U | 0x0000 | Delay for remote trigger messages (Link U) |
| 0x02B0  | reg_COINC_TRIG_MASK | 0x0000 | Local/remote trigger coincidence mask |
| 0x02B4  | reg_COINC_OVERLAP_CONTROL | 0x0000 | Coincidence trigger overlap control |
| 0x02B8  | reg_LOCAL_TRIG_DELAY_R | 0x0000 | Local trigger delay into remote trig algos (Link R) |
| 0x02BC  | reg_LOCAL_TRIG_DELAY_U | 0x0000 | Local trigger delay into remote trig algos (Link U) |
| 0x02C0  | reg_X_PLANE_LINK_MASK | 0x0000 | Mask of SERDES links in X-plane |
| 0x02C4  | reg_Y_PLANE_LINK_MASK | 0x0000 | Mask of SERDES links in Y-plane |
| 0x02C8  | reg_TRIGGER_HOLDOFF | 0x0000 | Trigger holdoff control |
| 0x02CC  | reg_UNUSED_2CC | 0x0000 | Unused |

---

## Read/Write Registers — Main Block 0x0800–0x08EC (CS=001)

| Address | Signal Name | Default | Description |
|---------|------------|---------|-------------|
| 0x0800  | reg_INPUT_LINK_MASK | 0x0800 | Mask out unused SERDES links |
| 0x0804  | reg_LED | 0x0000 | Front panel LED function select |
| 0x0808  | reg_SKEW_CTL_A | 0xCCCC | Link clock skew control A |
| 0x080C  | reg_SKEW_CTL_B | 0xCCCC | Link clock skew control B |
| 0x0810  | reg_SKEW_CTL_C | 0xCCCC | Link clock skew control C |
| 0x0814  | reg_MISC_CLK_CTL | 0x0007 | Overall clock control |
| 0x0818  | reg_AUX_IO_CTL | 0x2EFF | AUX I/O direction control |
| 0x081C  | reg_AUX_IO_DATA | 0x0000 | Manual AUX I/O data |
| 0x0820  | reg_TARGET_WHEEL_AUX_CTL | 0x0500 | Routes RS-485 & NIM inputs to logic blocks (target wheel) |
| 0x0824  | reg_TRIGGER_WIDTH | 0x000A | Width of trigger pulse on auxiliary outputs |
| 0x0828  | reg_SERDES_TPOWER | 0x0000 | Power down unused SERDES transmitters |
| 0x082C  | reg_SERDES_RPOWER | 0x0000 | Power down unused SERDES receivers |
| 0x0830  | reg_SERDES_LOCAL_LE | 0x0000 | Enable local SERDES loopback (TX→RX, diagnostic) |
| 0x0834  | reg_SERDES_LINE_LE | 0x0000 | Enable SERDES line loopback (RX→TX, diagnostic) |
| 0x0838  | reg_LVDS_PREEMPHASIS | 0x0000 | LVDS pre-emphasis + power-down for unused buffers |
| 0x083C  | reg_LINK_LRU_CTL | 0x0888 | Manual controls for extra SERDES links (L/R/U) |
| 0x0840  | reg_MISC_CTL1 | 0xFFC0 | Miscellaneous control bits 1 |
| 0x0844  | reg_MISC_CTL2 | 0x0000 | Miscellaneous control bits 2 (DC-balance enables) |
| 0x084C  | reg_DIAG_PIN_CTL | 0x0000 | Selects signals for Chipscope logic analyzer |
| 0x0850  | reg_TRIG_MASK | 0x0000 | Overall trigger selection control |
| 0x0854  | reg_TRIG_DIST_MASK | 0x00FF | Trigger distribution control to digitizers |
| 0x0858  | reg_FRAME_12_DATA_1 | 0x0001 | Frame 12 (internal trigger) word 1 |
| 0x085C  | reg_FRAME_12_DATA_2 | 0x0000 | Frame 12 word 2 |
| 0x0860  | reg_FRAME_12_DATA_3 | 0x0000 | Frame 12 word 3 |
| 0x0864  | reg_FRAME_12_DATA_4 | 0x0000 | Frame 12 word 4 |
| 0x0868  | reg_FRAME_12_DATA_5 | 0x0000 | Frame 12 word 5 |
| 0x086C  | reg_FRAME_14_DATA_1 | 0x0000 | Frame 14 (auxiliary trigger) word 1 |
| 0x0870  | reg_FRAME_14_DATA_2 | 0x0000 | Frame 14 word 2 |
| 0x0874  | reg_FRAME_14_DATA_3 | 0x0000 | Frame 14 word 3 |
| 0x0878  | reg_FRAME_14_DATA_4 | 0x0000 | Frame 14 word 4 |
| 0x087C  | reg_FRAME_14_DATA_5 | 0x0000 | Frame 14 word 5 |
| 0x0880  | reg_FRAME_16_DATA_1 | 0x0000 | Frame 16 (sync capture) word 1 |
| 0x0884  | reg_FRAME_16_DATA_2 | 0x0000 | Frame 16 word 2 |
| 0x0888  | reg_FRAME_16_DATA_3 | 0x0000 | Frame 16 word 3 |
| 0x088C  | reg_FRAME_16_DATA_4 | 0x0000 | Frame 16 word 4 |
| 0x0890  | reg_FRAME_16_DATA_5 | 0x0000 | Frame 16 word 5 |
| 0x0898  | reg_MASTER_COUNTER_RESETS | 0x0000 | Bitmask of counters to reset in Master |
| 0x089C  | reg_REMOTE_TRIGGER_PLANE_THRESHOLD | 0x0000 | Remote trigger plane multiplicity threshold |
| 0x08A0  | reg_MON1_FIFO_SEL | 0x0000 | Board-wide monitor FIFO 1 control |
| 0x08A4  | reg_MON2_FIFO_SEL | 0x0000 | Board-wide monitor FIFO 2 control |
| 0x08A8  | reg_MON3_FIFO_SEL | 0x0000 | Board-wide monitor FIFO 3 control |
| 0x08AC  | reg_MON4_FIFO_SEL | 0x0000 | Board-wide monitor FIFO 4 control |
| 0x08B0  | reg_MON5_FIFO_SEL | 0x0000 | Board-wide monitor FIFO 5 control |
| 0x08B4  | reg_MON6_FIFO_SEL | 0x0000 | Board-wide monitor FIFO 6 control |
| 0x08B8  | reg_MON7_FIFO_SEL | 0x0000 | Board-wide monitor FIFO 7 control (system monitor) |
| 0x08BC  | reg_MON8_FIFO_SEL | 0x0000 | Board-wide monitor FIFO 8 control |
| 0x08C0  | reg_CHAN_FIFO_CTL | 0x0000 | Channel-specific FIFO input selector |
| 0x08C4  | reg_REMOTE_MULTIPLCITY_CONTROL | 0x0000 | Remote multiplicity control (note: typo in source) |
| 0x08C8  | reg_SUM_OF_X_THRESH | 0x00FF | X-plane multiplicity threshold |
| 0x08CC  | reg_SUM_OF_Y_THRESH | 0x00FF | Y-plane multiplicity threshold |
| 0x08D0  | reg_LINK_L_PROPAGATION_CONTROL | 0x0000 | Command propagation control (Link L) |
| 0x08D4  | reg_LINK_R_PROPAGATION_CONTROL | 0x0000 | External trigger collection control (Link R) |
| 0x08D8  | reg_LINK_U_PROPAGATION_CONTROL | 0x0000 | External trigger collection control (Link U) |
| 0x08DC  | reg_RATE_COUNTER_CTL | 0x0000 | Trigger rate counter mode select |
| 0x08E0  | reg_PULSED_CTL1 | 0x0000 | Pulsed control 1 — bits create short pulses for test/reset |
| 0x08E4  | reg_PULSED_CTL2 | 0x0000 | Pulsed control 2 — bits create short pulses for test/reset |
| 0x08E8  | reg_NIM1_DELAY | 0x0001 | NIM IN 1 delay control |
| 0x08EC  | reg_NIM2_DELAY | 0x0001 | NIM IN 2 delay control |

---

## Special Addresses

| Address | Type | Description |
|---------|------|-------------|
| 0x0848  | FIFO (R/W) | Generic sandbox FIFO — VME R/W, no board function (`GENERIC_TEST_FIFO`, 16×1023 async; reset via `xreg_PULSED_CTL2(14)`) ✅ verified 2026-04-25 — `registers.vhd:L1043,L1155,L1168` (REG_ADDR=X"0848" for both read and write) |
| 0x08F0  | Special write | FIFO reset register — resets for MON FIFOs 1–8 and CHAN FIFOs 1–8; `xreg_FIFO_RESETS[7:0]` → MON resets, `[15:8]` → CHAN resets ✅ verified 2026-04-25 — `registers.vhd:L1459,L871-872` (write decoder X"08F0"; stale internal comment at L862 erroneously says 0x00F0, which is the old address) |
| 0x08F4  | FIFO (write) | **Async Command FIFO** — written by VME, read by Master State Machine ✅ verified 2026-04-25 — `registers.vhd:L1157` (`if (REG_ADDR = X"08F4") then ASYNC_CMD_FIFO_WE <= '1'`) |
| 0x08F8  | Reserved | **AUX Command FIFO** — reserved in header comment but **not implemented** in current firmware; no WE logic exists for this address ✅ verified 2026-04-25 — `registers.vhd:L22-23` (header comment says reserved), no X"08F8" write decoder found in source |
| 0x08FC  | Reserved | Reserved for future FIFO type |

---

## Embedded Lookup RAMs (DPRAM, CS=001)

Three dual-port RAMs are accessible via VME (port A, 64×16-bit) and provide single-bit outputs to logic (port B, 1024×1-bit lookup from AUX I/O address):

| Address Range | RAM | Output Signal | Function |
|--------------|-----|--------------|----------|
| 0x0300–0x03FC | VETO RAM | VETO_FROM_VETO_RAM | Level active while address maps to 1 |
| 0x0400–0x04FC | TRIG RAM | TRIG_FROM_TRIG_RAM / LVL_FROM_TRIG_RAM | Edge (one clock) from level output |
| 0x0500–0x05FC | SWEEP RAM | SWEEP_FROM_SWEEP_RAM | Level active while address maps to 1 |

The lookup address comes from `SWEEP_RAM_ADDRESS`, `TRIG_RAM_ADDRESS`, `VETO_RAM_ADDRESS` (10 bits each, normally from AUX I/O in target-wheel mode).

---

## Monitor FIFOs

### Board-wide Monitor FIFOs (MON1–MON8)

- 8 FIFOs, each 16-bit wide
- Written by external logic via `MON_FIFO_WE`/`MON_FIFO_IN`
- Read via VME; selector registers `reg_MON1_FIFO_SEL`–`reg_MON8_FIFO_SEL` (0x08A0–0x08BC)
- MON7 is the **system trigger monitor FIFO** with extended status flags:
  - `MON7_FIFO_ALMOST_EMPTY`, `MON7_FIFO_PROG_EMPTY`, `MON7_FIFO_ALMOST_FULL`, `xMON7_FIFO_PROG_FULL`
  - `MON7_FIFO_OVERFLOW_LAT`, `MON7_FIFO_UNDERFLOW_LAT` (latching error flags)
  - `MON7_VETO_REQUEST` output — asserted when MON7 wants triggers vetoed
  - Depth counter added 2020-06-23 (MBO): 17-bit `FIFO_DEPTH` with boundary tracking for event framing

### Channel-Specific Monitor FIFOs (CHAN1–CHAN8)

- 8 channel FIFOs, 16-bit wide
- Each has independent write clock (`CHAN_FIFO_WCLK`) and write enable (`CHAN_FIFO_WE`)
- Controlled by `reg_CHAN_FIFO_CTL` (0x08C0)

---

## VME Bus State Machine

Internal `STBSTATE` FSM handles bus timing:

| State | Action |
|-------|--------|
| `WAIT_FOR_1` | Waiting for VME strobe assert |
| `WRITE_DELAY1/2` | Two-cycle delay pipeline for write setup |
| `DO_WRITE` | Register write performed on this clock edge |
| `READ_1` | Register read presented to `VME_DATA_OUT` |
| `WAIT_FOR_0` | Waiting for strobe to deassert, holding ACK |

- `VME_RNW='1'` → read; `VME_RNW='0'` → write
- `CHIPSEL` = {VME_CS2, VME_CS1, VME_CS0}
- `READ_CS_001` / `WRITE_CS_001` gating signals for main FPGA block
- `READ_CS_010` / `WRITE_CS_010` for expansion block (0x1000–0x3FFC)

---

## Rate Counters

Two sets of 8×32-bit synchronized rate counters:

| Signal | Description |
|--------|-------------|
| `TRIG_RATE_COUNTERs` | Synchronized capture counters (type: JTA_8X32_Array) |
| `RAW_TRIG_RATE_COUNTERs` | Raw (unsynchronized) rate counters |

Mode controlled by `reg_RATE_COUNTER_CTL` (0x08DC).

---

## Notable Defaults

✅ verified 2026-04-24 — registers.vhd signal declarations (L327, L346, L348–350, L362, L299, L301, L303) + reset assignments confirmed identical

| Register | Default | Significance |
|----------|---------|-------------|
| reg_INPUT_LINK_MASK | 0x0800 | Link 11 masked by default at startup |
| reg_SKEW_CTL_A/B/C | 0xCCCC | Mid-range skew setting for all 16 links |
| reg_MISC_CTL1 | 0xFFC0 | Bits 15:6 set → many features enabled at reset |
| reg_MISC_CTL2 | 0x0000 | DC-balance enables all off at reset |
| reg_TRIG_DIST_MASK | 0x00FF | All 8 digitizer links enabled for trigger distribution |
| reg_TRIG_VETO_SELECT_A–H | 0x000F | Veto sources 0–3 enabled for all algorithms |
| reg_MYRIAD_TRIG_DELAY | 0x0001 | 1-cycle MyRIAD delay |
| reg_MON7_FILL_CTL | 0x0086 | System mon FIFO fill control: captures 6 events, skip mode |
| reg_CPLD_EXTRA | 0x5504 | Rev D sum buffer chip config |
| reg_SSI_CTL | 0x0008 | SSI encoder interface: one option enabled at reset |
| reg_SSI_ENCODE_TIME | 0x0100 | SSI sync timeout: 256 cycles |
| reg_USER_PACKAGE_DATA | 0x0123 | Sentinel/test value |

---

## Cross-References

- [fpga.md](../fpga.md) — MTRG FPGA top-level overview
- [deep_fpga_MTRG_MAIN.md](../deep_fpga_MTRG_MAIN.md) — MTRG trigger algorithm details
- [MTRG_top.md](MTRG_top.md) — top.vhd instantiation of this entity
- [MTRG_mstr_mach.md](MTRG_mstr_mach.md) — consumer of ASYNC_CMD_FIFO and frame data registers
- [MTRG_support_modules.md](MTRG_support_modules.md) — link_tx_block.vhd (reg_MISC_CTL2 DC-balance bits)
