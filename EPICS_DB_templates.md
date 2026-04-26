# EPICS DB Templates — DGS IOC

**Source:** `ANLDAQ/ioc/db/` (all `.template` files) and `ANLDAQ/ioc/boot/vme*.cmd` (instantiation)
**Date documented:** 2026-04-23  
**Last reviewed:** 2026-04-26 (duplicate reference sections condensed; see ioc.md for user_package_data, boot sequence, CA ports, board type encoding)
Stability: C2 - Active / semi-stable

---

## Table of Contents

1. [Overview](#overview)
2. [Template Files Summary](#template-files-summary)
3. [PV Naming Scheme](#pv-naming-scheme)
4. [daqCrate.template](#daqcrate-template)
5. [daqSegment2.template](#daqsegment2-template)
6. [MDigRegisters / SDigRegisters.template](#mdigregisters--sdigregisters-template)
7. [MDigUser / SDigUser.template](#mdiguser--sdiguser-template)
8. [MDigUserVME / SDigUserVME.template](#mdigu​servme--sdigusервme-template)
9. [RTrigRegisters.template](#rtrigregisters-template)
10. [RTrigUser.template](#rtriguser-template)
11. [MTrigRegisters.template](#mtrigregisters-template)
12. [MTrigUser.template](#mtriguser-template)
13. [dgsGlobals DB files](#dgsglobals-db-files)
14. [Instantiation per VME Crate](#instantiation-per-vme-crate)
15. [user_package_data Numbering](#user_package_data-numbering) → see ioc.md
16. [IOC Boot Sequence](#ioc-boot-sequence) → see ioc.md
17. [EPICS CA Port Assignments](#epics-channel-access-port-assignments) → see ioc.md
18. [Board Type Encoding](#board-type-encoding) → see ioc.md
19. [Fifo Select Encoding](#fifo-select-encoding)
20. [`gretDet.dbd` — EPICS Database Definition File](#gretdetdbd--epics-database-definition-file)

---

## Overview

The DGS EPICS IOC uses a set of EPICS `.template` files to define process variables (PVs) for every
board in every VME crate. Templates are instantiated with `dbLoadRecords()` at IOC boot time, with
macro substitutions for `$(CRATE)` (e.g. `01`, `06`, `10`) and `$(BOARD)` (e.g. `MDIG1`, `SDIG1`,
`RTR2`, `MTRG`).

Total templates: 8 primary + 1 static globals `.db` per crate.
Total EPICS records: ~115,000 lines across all templates.

---

## Template Files Summary

| File | Lines | Records | Applies To | Purpose |
|------|-------|---------|------------|---------|
| `daqCrate.template` | 471 | 58 | One per VME crate | InLoop/OutLoop/MiniSender status, board-type PVs, trace PVs | ✅ verified 2026-04-23 — `grep -c '^record('` = 58 (prev. documented as ~80)
| `daqSegment2.template` | 67 | 2 | One per board per crate | Per-board software enable (`CS_Ena`) and FIFO select (`FifoNum`) | ✅ verified 2026-04-23 — wc -l=67
| `MDigRegisters.template` | 2,738 | — | MDIG boards | Raw register readback PVs (`reg_*_RBV`) | ✅ verified 2026-04-23 — wc -l=2738
| `SDigRegisters.template` | 2,738 | — | SDIG boards | Same as MDigRegisters (identical layout) | ✅ verified 2026-04-23 — `diff MDigRegisters SDigRegisters` → 0 lines
| `MDigUser.template` | 11,931 | 1,368 | MDIG boards | User-facing per-channel config PVs (thresholds, CFD, DCM, event counts) | ✅ verified 2026-04-23 — wc -l=11931, records=1368
| `SDigUser.template` | 11,931 | 1,368 | SDIG boards | Identical layout to MDigUser | ✅ verified 2026-04-23 — `diff MDigUser SDigUser` → 0 lines (byte-for-byte identical)
| `MDigUserVME.template` | 127 | 10 | MDIG boards | Additional VME-specific PVs (board-level, not per-channel) | ✅ verified 2026-04-23 — wc-l=127, records=10
| `SDigUserVME.template` | 127 | 10 | SDIG boards | Identical to MDigUserVME | ✅ verified 2026-04-23 — wc-l=127
| `RTrigRegisters.template` | 1,390 | — | RTRG boards | Router Trigger raw register readbacks | ✅ verified 2026-04-23 — wc -l=1390
| `RTrigUser.template` | 8,452 | 897 | RTRG boards | User-facing Router Trigger config PVs | ✅ verified 2026-04-23 — wc-l=8452, records=897
| `MTrigRegisters.template` | 4,953 | — | MTRG board | Master Trigger raw register readbacks | ✅ verified 2026-04-23 — wc-l=4953
| `MTrigUser.template` | 70,386 | 7,056 | MTRG board | User-facing Master Trigger config PVs (largest template by far) | ✅ verified 2026-04-23 — wc-l=70386, records=7056

---

## PV Naming Scheme

All PVs follow the pattern:

```
VME<CRATE>:<BOARD>:<param>
```

Examples:
- `VME01:MDIG1:coarse_threshold0` — channel 0 coarse threshold on MDIG1 in crate 01
- `VME10:MTRG:COINC_OVERLAP_DELAY` — coincidence overlap delay on Master Trigger in crate 10
- `VME06:RTR2:CF1_MODE` — connector fanout 1 mode on Router Trigger 2 in crate 06

Readback PVs use `_RBV` suffix (mbbi or longin type).
Setpoint PVs use `ao`, `bo`, `mbbo`, `longout` types.
Some numeric setpoints have a parallel `LONGOUT` variant (e.g. `coarse_threshold0LONGOUT`) used by scripts.

Crate-level PVs use `DAQC$(CRATE)_*` prefix (not `VME$(CRATE):`):
- `DAQC01_CV_InLoop1` — inLoop MBytes/sec read from crate 01
- `DAQC01_BoardType0` — decoded board type for slot 0

---

## daqCrate.template

Instantiated **once per VME crate**. Macros: `$(CRATE)`.

### PV Groups

**InLoop status** (`DAQC$(CRATE)_CV_*`):
| PV | Type | Description |
|----|------|-------------|
| `DAQC$(CRATE)_CV_CRATENUM` | ai | Crate number (static value) |
| `DAQC$(CRATE)_CV_InLoop1` | ai | MBytes/sec read rate |
| `DAQC$(CRATE)_CV_InLoop2` | ai | Type F buffer raw count |
| `DAQC$(CRATE)_CV_InLoop3` | ai | Number of VME transfers |
| `DAQC$(CRATE)_CV_InLoop4` | ai | Result of last transfer |

**Board type** (`DAQC$(CRATE)_BoardType0` through `_BoardType6`): mbbi records that decode the
board type ID from `code_revision[11:8]` into a human-readable string. Up to 7 boards per crate.
See [Board Type Encoding](#board-type-encoding) below.

**OutLoop status** (`DAQC$(CRATE)_CV_OutLoop0`–`6`, `_OL_*`):
| PV | Type | Description |
|----|------|-------------|
| `DAQC$(CRATE)_CV_OutLoop0`–`6` | ai | Data lost from boards 0–6 |
| `DAQC$(CRATE)_CV_BuffersAvail` | ai | Buffers available |
| `DAQC$(CRATE)_CV_NumSendBuffers` | ai | Send buffer count |
| `DAQC$(CRATE)_OL_DataRate0`–`6` | ai | kBytes/sec from boards 0–6 |
| `DAQC$(CRATE)_OL_Data0`–`6` | ai | Total MBytes from boards 0–6 |
| `DAQC$(CRATE)_OL_NumFreeBuffers` | ai | Free buffer count |
| `DAQC$(CRATE)_OL_NumWrittenBuffers` | ai | Written buffer count |
| `DAQC$(CRATE)_OL_TotalBufsWritten` | ai | Cumulative buffers written |
| `DAQC$(CRATE)_OL_TotalFBufsWritten` | ai | Cumulative F-buffers written |
| `DAQC$(CRATE)_OL_TotalBufsLost` | ai | Cumulative buffers lost |
| `DAQC$(CRATE)_OL_BufLostPerecnt` | ai | Percent buffers lost (note: typo in PV name) |

**OutLoop data-quality checks** (ao type, initialized to enabled=1):
- `DAQC$(CRATE)_OL_HeaderCheckEnable` — enable header validation
- `DAQC$(CRATE)_OL_TimestampCheckEnable` — enable timestamp validation
- `DAQC$(CRATE)_OL_DeepCheckEnable` — enable deep packet check
- `DAQC$(CRATE)_OL_HeaderSummaryEnable` — enable summary print (default off)
- `DAQC$(CRATE)_OL_HeaderSummaryPrescale` — prescale factor (default 4096)
- `DAQC$(CRATE)_OL_HeaderSummaryEventPrescale` — event-level prescale (default 256)

**Trace display** (for waveform capture):
- `DAQC$(CRATE)_CV_Trace` — waveform (1024 SHORT samples)
- `DAQC$(CRATE)_CV_TraceLen` — longin, trace length (0–1024)
- `DAQC$(CRATE)_CS_TraceBd` — longout, board to trace (0–15)
- `DAQC$(CRATE)_CS_TraceChan` — longout, channel to trace (0–9)
- `DAQC$(CRATE)_CS_TraceHorns` — bo, show/hide trace triggers

**Send rate** (updated 1 second scan):
- `DAQC$(CRATE)_CV_SendRate` — ai, smoothed KB/sec output (SMOO=0.9)

**MiniSender status**:
- `DAQC$(CRATE)_CV_SenderRunning` — bi, Stopped/Running

**InLoop↔OutLoop IPC**:
- `DAQC$(CRATE):inLoop_Running` — bi, inLoop run state; written by inLoop, monitored by outLoop

**Dummy PVs** (for unoccupied crate slots):
- `VME$(CRATE):X:CS_Ena` — bo, always disabled (DOL=0)
- `VME$(CRATE):X:FifoNum` — mbbo, defaults to MAIN DATA FIFO (value=6)

---

## daqSegment2.template

Instantiated **once per populated board slot** in each crate. Macros: `$(CRATE)`, `$(BOARD)`.

| PV | Type | Description |
|----|------|-------------|
| `VME$(CRATE):$(BOARD):CS_Ena` | bo | Software board enable (default: Disable=0) |
| `VME$(CRATE):$(BOARD):FifoNum` | mbbo | Readout FIFO selector (default: MAIN DATA FIFO=6) |

Both are initialized with `PINI=YES` and `DTYP=Raw Soft Channel` (software PVs, no hardware link).
These are used by inLoop to decide whether to read from a slot and which FIFO to use.

---

## MDigRegisters / SDigRegisters.template

Raw hardware register readback/write PVs for Master and Slave Digitizer boards.
**Source:** `DGS_tools_pack/ANLDAQ/ioc/db/MDigRegisters.template` (2,738 lines, 359 records: 217 longin + 142 longout) ✅ verified 2026-04-25 — wc/grep confirmed

Macros: `$(CRATE)`, `$(BOARD)`. PV prefix: `VME$(CRATE):$(BOARD):`.
SDigRegisters.template is identical (byte-for-byte) to MDigRegisters.template.

Two naming prefixes are used:
- `reg_*` — R/W registers (paired `longout` write + `longin` readback `_RBV`, scanned 1 second)
- `regin_*` — read-only status/counter registers (`longin` only, scanned 1 second)

### Board-Level Identity & Status

| PV Suffix | Type | Description |
|-----------|------|-------------|
| `regin_board_id_RBV` | longin | Contains board address and firmware ID |
| `reg_programming_done` / `_RBV` | longout/longin | FIFO status and firmware programming done flag |
| `regin_hardware_status_RBV` | longin | DCM status info (clock lock, resets) |
| `reg_master_logic_status` / `_RBV` | longout/longin | Global logic control/status |
| `reg_trigger_config` / `_RBV` | longout/longin | Trigger configuration register |
| `regin_code_revision_RBV` | longin | Firmware code revision |
| `regin_code_date_RBV` | longin | Firmware code date (BCD or int) |

### Per-Channel Configuration (channels 0–9)

| PV Group | Type | Description |
|----------|------|-------------|
| `reg_channel_controlN` / `_RBV` | longout/longin | Channel N control register |
| `reg_CFD_fractionN` / `_RBV` | longout/longin | CFD fraction for channel N |
| `reg_led_thresholdN` / `_RBV` | longout/longin | LED threshold for channel N |
| `reg_disc_widthN` / `_RBV` | longout/longin | Discriminator pulse width for channel N |
| `reg_raw_data_delayN` / `_RBV` | longout/longin | Raw data (waveform) pre-trigger delay for channel N |
| `reg_raw_data_lengthN` / `_RBV` | longout/longin | Raw data (waveform) window length for channel N |

### Coincidence / Timing Windows (per-channel, 0–9)

The digitizer firmware uses multiple named coincidence windows per channel. Each is a `longout`/`longin` pair:

| Window Name | Description |
|-------------|-------------|
| `reg_d_windowN` | D window — primary coincidence gate |
| `reg_k_windowN` | K window — second coincidence gate |
| `reg_m_windowN` | M window — multiplicity coincidence gate |
| `reg_d3_windowN` | D3 window — third-level coincidence gate |
| `reg_p1_windowN` | P1 window — pileup/prompt coincidence gate 1 |
| `reg_p2_windowN` | P2 window — pileup/prompt coincidence gate 2 |

### Board-Level Timing & Readout

| PV Suffix | Type | Description |
|-----------|------|-------------|
| `reg_win_comp_min` / `_RBV` | longout/longin | Window comparator lower bound |
| `reg_win_comp_max` / `_RBV` | longout/longin | Window comparator upper bound |
| `reg_baseline_delay` / `_RBV` | longout/longin | Baseline computation delay |
| `reg_downsample_holdoff` / `_RBV` | longout/longin | Downsampler holdoff duration |
| `reg_holdoff_control` / `_RBV` | longout/longin | Holdoff operation control |
| `reg_veto_gate_width` / `_RBV` | longout/longin | Time window for veto gate |
| `reg_vme_ext_delay` / `_RBV` | longout/longin | VME pulsed control delay |
| `reg_user_package_data` / `_RBV` | longout/longin | User-defined data embedded in event header |
| `reg_channel_pulsed_control` | longout | Write-only self-clearing pulsed control |

### Timestamps (read-only)

| PV Suffix | Description |
|-----------|-------------|
| `regin_lat_timestamp_lsb_RBV` | Latched timestamp lower 32 bits |
| `regin_lat_timestamp_msb_RBV` | Latched timestamp upper 16 bits |
| `regin_live_timestamp_lsb_RBV` | Live (running) timestamp lower 32 bits |
| `regin_live_timestamp_msb_RBV` | Live (running) timestamp upper 16 bits |
| `reg_ts_err_count_ctrl` / `_RBV` | longout/longin | Enable/disable timestamp error counting |
| `regin_ts_err_count_RBV` | longin | Timestamp synchronization error count |

### Per-Channel Counters (read-only, channels 0–9)

| PV Group | Description |
|----------|-------------|
| `regin_dropped_event_countN_RBV` | Number of events dropped (FIFO overflow or gate rejection) |
| `regin_accepted_event_countN_RBV` | Number of accepted (output) events |
| `regin_ahit_countN_RBV` | Above-threshold hit count |
| `regin_disc_countN_RBV` | Discriminator event count |
| `regin_hihilolo_N_RBV` | ADC saturation counter — extreme rails (HIHI/LOLO) |
| `regin_hilo_N_RBV` | ADC saturation counter — high/low threshold exceeded |

### External Discriminator Control

| PV Suffix | Description |
|-----------|-------------|
| `reg_external_discriminator_src` / `_RBV` | Selects source for external discriminator input |
| `reg_external_disc_mode` / `_RBV` | How external discriminator is used (veto, gate, trigger…) |

### SERDES / Phase / Diagnostics

| PV Suffix | Description |
|-----------|-------------|
| `reg_sd_config` / `_RBV` | SERDES configuration register |
| `regin_phase_value_RBV` | ADC clock to ACQ clock phase measurement |
| `regin_phase_errors_RBV` | Phase hunter lock status / error flags |
| `regin_phase_offset_a/b/c_RBV` | Per-channel phase offsets (three groups) |
| `regin_serdes_phase_value_RBV` | Current SERDES phase offset applied |
| `reg_ila_config` / `_RBV` | ILA (Integrated Logic Analyzer) mux control |
| `reg_diag_mux_control` / `_RBV` | Selects diagnostic signal routing |
| `reg_diag_channel_input` / `_RBV` | Diagnostic use: allows alternate channel input |

### Front Panel / LED

| PV Suffix | Description |
|-----------|-------------|
| `reg_led_control` / `_RBV` | Front panel LED operation control |
| `regin_led_state_RBV` | Current front panel LED state readback |
| `reg_rj45_spare_dout_control` / `_RBV` | Front panel RJ45 spare digital output control |
| `reg_dac` / `_RBV` | DAC configuration register |

### DAQ Board Identification

| PV Suffix | Description |
|-----------|-------------|
| `regin_hardware_status_RBV` | DCM status info (ACQ lock, ADC lock, phase shift overflow, clock stopped) |

---

## MDigUser / SDigUser.template

User-facing per-board configuration PVs. Macros: `$(CRATE)`, `$(BOARD)`.
SDigUser.template is **identical** to MDigUser.template.

**Record count: 1,368 per board** (298 bi + 239 bo + 230 longin + 164 longout + 156 ao + 156 ai + 61 mbbo + 61 mbbi + 3 calcout).

### Per-Channel Parameters (channels 0–9 each)

| PV Suffix | Type (set/read) | Description |
|-----------|-----------------|-------------|
| `coarse_thresholdN` / `coarse_thresholdN_RBV` | ao/longin | Coarse CFD threshold |
| `coarse_thresholdNLONGOUT` | longout | Script-friendly longout variant |
| `preamp_reset_delayN` / `preamp_reset_delayN_RBV` | ao/longin | Preamp reset inhibit delay |
| `led_thresholdN` / `led_thresholdN_RBV` | ao/longin | LED threshold |
| `CFD_fractionN` / `CFD_fractionN_RBV` | ao/longin | CFD fraction setting |
| `accepted_event_countN_RBV` | longin | Accepted event count readback |
| `ahit_countN_RBV` | longin | Above-threshold hit count |
| `ahit_count_modeN` / `ahit_count_modeN_RBV` | mbbo/mbbi | Hit count mode |
| `cfd_esum_modeN` / `cfd_esum_modeN_RBV` | mbbo/mbbi | CFD energy-sum mode |
| `ext_disc_srcN` / `ext_disc_srcN_RBV` | mbbo/mbbi | External discriminator source |
| `BGO_discbit_selectN` / `_RBV` | mbbo/mbbi | BGO discriminator bit select |

### Board-Level Parameters

| PV Suffix | Type | Description |
|-----------|------|-------------|
| `win_comp_min` / `win_comp_max` | ao/longin | Window comparator min/max |
| `acq_dcm_lock_RBV` | bi | ACQ DCM lock status |
| `acq_dcm_reset_RBV` | bi | ACQ DCM reset status |
| `adc_dcm_lock_RBV` | bi | ADC DCM lock status |
| `acq_dcm_clock_stopped_RBV` | bi | ACQ clock stopped alarm |
| `acq_ph_shift_overflow_RBV` | bi | Phase shift overflow |
| `aux_output_modeN` | mbbo | Auxiliary output mode |

---

## MDigUserVME / SDigUserVME.template

Small supplemental template with VME-side PVs. 10 records per board.

Record types: 6 bi + 2 longin + 1 mbbo + 1 mbbi.
Covers board-level VME interface status flags not in the main User template.

---

## RTrigRegisters.template

Raw register readbacks for Router Trigger boards. 1,390 lines.
Pattern: `VME$(CRATE):$(BOARD):reg_*_RBV`.

---

## RTrigUser.template

User-facing Router Trigger PVs. **897 records per RTR board.**
Record types: 353 bi + 286 bo + 118 longin + 87 longout + 27 mbbo + 26 mbbi.

### Key PV Groups

**Connector Fanout (CF) Modes** (8 connectors):
- `CF1_MODE` through `CF8_MODE` — mbbo, fanout mode selection
- `CF1_MODE_WE` through `CF8_MODE_WE` — mbbo, write-enable gating
- Corresponding `_RBV` readbacks

**Interconnect Direction** (connector A/B, nibbles):
- `A_3_0_DIR`, `A_7_4_DIR`, `B_3_0_DIR`, `B_7_4_DIR` — mbbo, data direction
- Corresponding `_RBV` readbacks

**NIM Inputs**:
- `NIMSrc1`, `NIMSrc2` — mbbo, NIM input source selection
- `NIM_THROTTLE_SELECT` — mbbo, NIM throttle source
- `THROTTLE_TIME_RANGE` — mbbo, throttle time range

**Interlink Masks** (ILM — 8 channels A–H):
- `ILM_A` through `ILM_H` — bo, enable/disable interlink mask per connector

**Routing / Coincidence**:
- `X_SELECT`, `Y_SELECT` — mbbo, X/Y matrix source selects
- `FS_SEL` — mbbo, fanout select
- `conn_a_mask` through `conn_d_mask` — longout, connector port masks
- `ASSERTION_DELAY`, `ASSERTION_DELAY_RBV` — longout/longin, assertion delay

**LED Control**: `LEDControl` — mbbo

**Diagnostics**:
- `CLEAR_DIAG_COUNTERS` — bo, pulse to reset diagnostic counters
- `CHAN_FIFO_FLAGS_RBV` — longin, channel FIFO flag status
- `ALL_LOCKED_RBV` — bi, all links locked
- `CODE_DATE_RBV`, `Code_Revision_RBV` — longin, firmware version readbacks
- `ClkSrc`, `ClkSrc_RBV` — mbbo/mbbi, clock source
- `DEN_A_RBV` — longin, data enable A readback
- `BIT_5_OFFSET`, `BIT_5_OFFSET_RBV` — longout/longin, bit 5 offset

---

## MTrigRegisters.template

Raw register readbacks for the Master Trigger board. 4,953 lines.
Pattern: `VME$(CRATE):$(BOARD):reg_*_RBV`.

---

## MTrigUser.template

The largest template in the IOC: **70,386 lines, 7,056 records per MTRG board.**
Record types: 3,507 bi + 3,430 bo + 36 longin + 28 mbbo + 28 mbbi + 27 longout.

The sheer size reflects the Master Trigger's role as the hub of the full Gammasphere trigger system,
with registers covering every router link, every coincidence combination, sweep RAM, veto RAM, etc.

### Key PV Groups

**Clock and Sync**:
- `SLOW_CLOCK_SEL` — mbbo, slow clock selection
- `ClkSrc`, `ClkSrc_RBV` — mbbo/mbbi
- `ENBL_SYNC_RESET` — bo, enable synchronous reset
- `COUNTER_ROLL_999` — bo, roll-over at 999 mode

**VETO / TRIG / SWEEP RAM**:
- `VETO_RAM_ADDR_SRC` — mbbo, veto RAM address source
- `TRIG_RAM_ADDR_SRC` — mbbo, trigger RAM address source
- `SWEEP_RAM_ADDR_SRC` — mbbo, sweep RAM address source
- `SweepMux` — mbbo, sweep multiplex control

**MYRIAD Interface**:
- `MYR_TRIGGER_TYPE_SELECT` — bo, MYRIAD trigger type select
- `MYR_TS_MODE` — bo, MYRIAD timestamp mode
- `TRIG_MON_SEL` — mbbo, trigger monitor select

**Link / SSI Interface**:
- `LINK_L_CMD_FORMAT` — mbbo, left link command format
- `SSI_InputSelect`, `SSI_BIT_RANGE` — mbbo, SSI input and bit range
- `LINK_INIT_STATE_RBV` — mbbi, link initialization state readback
- `LINK_L_IS_TRIGGER_TYPE`, `LINK_R_IS_TRIGGER_TYPE`, `LINK_U_IS_TRIGGER_TYPE` — bo

**NIM I/O**:
- `NIMSrc1`, `NIMSrc2` — mbbo, NIM source 1 and 2
- `NIM1_SubSelect`, `NIM2_SubSelect` — mbbo, sub-selects
- `AUXPolaritySelect` — mbbo, auxiliary polarity
- `AuxTrig_Width` — mbbo, auxiliary trigger width

**Connector Fanout** (CFC1–CFC7):
- `CFC1`–`CFC7`, `CFCN` — mbbo, connector fanout channel assignment per router link
- Per-router FIFO force: `CFIFO1_FORCE`–`CFIFO8_FORCE` — bo

**Coincidence / Trigger Logic**:
- `COINC_OVERLAP_DELAY` — longout, coincidence overlap delay
- `COINC_TRIG_MASK_A1_RBV`–`A8_RBV` — longin, coincidence trigger mask readbacks per router
- `Rtr1ThrottleReq`–`Rtr8ThrottleReq` — bo, per-router throttle request
- `ALGO_5_SELECT` — bo, algorithm 5 select
- `PEHLRU`, `PEEFG`, `PEABCD` — mbbo, pipeline enable groups

**Connector/Channel Resets**:
- `CF0_F12RESET`–`CF7_F12RESET` — bo, per-CF FIFO 1/2 reset
- `ALL_CHANNEL_RESET` — bo, reset all channels
- `CLEAR_DIAG_COUNTERS`, `CLEAR_RATE_COUNTERS`, `CLEAR_ENCODER_CNTR` — bo, counter resets

**Trigger Monitor**:
- `TRIGMON_ENBL_VME_CLK`, `TRIGMON_ENBL_TEST`, `TRIGMON_ENBL_ACK1`, `TRIGMON_ENBL_ACK2` — bo

**SYSMON** (System Monitor / Vivado SYSMON block):
- `SYSMON_ENABLE`, `SYSMON_ENBL` — bo

**ILA (Integrated Logic Analyzer)**:
- `ILA1_MODE` — mbbo, ILA mode
- `FAST_TDC_ILA_CTL` — mbbo, fast TDC ILA control

**Timestamp**:
- `TS_SAMP_PHASE` — bo, timestamp sample phase
- `TS_EDGE_FLAG_SEL` — mbbo, timestamp edge flag select

**Diagnostics**:
- `ASYNC_CMD_FIFO_RESET`, `ASYNC_CMD_FLAG` — bo, async command FIFO controls
- `ASYNC_FIFO_STATE_RBV` — mbbi
- `SKIP_TDC_DATA` — bo, skip TDC data mode
- `ALL_LOCKED_RBV` — bi, all links locked

---

## dgsGlobals DB files

One static `.db` file per VME crate (e.g. `dgsGlobals_DGS_VME01.db` through `VME12.db`, `VME66.db`, `VME99.db`).

These are **not** templates — they are pre-expanded EPICS DB files generated for a specific crate
configuration. They contain exclusively `dfanout` records (298 records total in VME99 version).
All records are of type `dfanout`; the files contain no other record types.

**Purpose:** Fan out global "set-all" commands to individual board PVs in a two-level chain:
1. **Phase 2A** — `VMEnn:GLBL:<param>` fans to `VMEnn:MDIG1:<param>` and/or `VMEnn:SDIG1:<param>`
2. **Phase 2B** — `VMEnn:MDIG1:<detector_type>_<param>` fans to per-channel `VMEnn:MDIG1:<param>N`

Example (ext_disc_src on VME99):
```
# Phase 2A: global → board
record(dfanout,"VME99:GLBL:BGOs_ext_disc_src") { OUTA: VME99:MDIG1:BGOs_ext_disc_src }
# Phase 2B: board → channels 0–4
record(dfanout,"VME99:MDIG1:BGOs_ext_disc_src") { OUTA–OUTE: VME99:MDIG1:ext_disc_srcN (N=0..4) }
```

**Detector-type prefixes** (32 params each — same set for all 4 types):
- `BGOs_` → MDIG1 channels 0–4 (BGO synchronous/slow board)
- `GeC_` → MDIG1 channels 5–9 (central Ge contact)
- `BGOp_` → SDIG1 channels 0–4 (BGO prompt/fast board)
- `GeS_` → SDIG1 channels 5–9 (segment Ge contact)

✅ verified 2026-04-25 — `ANLDAQ/ioc/db/dgsGlobals_DGS_VME99.db`: `grep -c 'record(dfanout'` = 298; channel routing from `BGOs_ext_disc_src`→ext_disc_src0–4, `GeC_ext_disc_src`→ext_disc_src5–9, `BGOp_ext_disc_src`→SDIG1:ext_disc_src0–4, `GeS_ext_disc_src`→SDIG1:ext_disc_src5–9.

**Per-detector-type parameters (32 params, same set for BGOs/GeC/BGOp/GeS):**
`ahit_count_mode`, `CFD_fraction`, `channel_enable`, `coarse_width`, `counter_reset`,
`d3_window`, `disc_count_mode`, `disc_width`, `downsample_factor`, `dropped_event_count_mode`,
`d_window`, `enable_dec_pause`, `event_count_mode`, `event_extension_mode`, `ext_disc_sel`,
`ext_disc_src`, `k0_window`, `k_window`, `led_threshold`, `m_window`, `p1_window`, `P2_mode`,
`p2_window`, `pileup_extension_enable`, `pileup_mode`, `pileup_waveform_only_mode`,
`preamp_reset_delay`, `preamp_reset_delay_en`, `raw_data_delay`, `raw_data_length`,
`trigger_polarity`, `write_flags`

**System-level (global, no detector-type prefix) — 38 PVs:**

| PV suffix | Description |
|-----------|-------------|
| `master_fifo_reset` | Reset FIFOs on all boards |
| `master_counter_reset` | Reset all event/disc counters |
| `master_logic_enable` | Enable/disable DAQ logic globally |
| `trigger_mux_select` | Select trigger source (ExtTTCL, IntAcptAll, etc.) |
| `veto_enable` | Enable veto logic |
| `ts_counter_mode` | Timestamp counter mode |
| `ts_counter_reset` | Reset timestamp counter |
| `counter_mode` | Event counter mode |
| `cfd_mode` | CFD mode selection |
| `coarse_threshold` | Global coarse discriminator threshold |
| `holdoff_time` | Global holdoff time |
| `peak_sensitivity` | Peak-sensing sensitivity |
| `stop_ho_at_peak` | Stop holdoff at peak flag |
| `load_delays` | Load delay settings |
| `clk_select` | Clock source select |
| `rj45_throttle_mode` | RJ45 link throttle mode |
| `win_comp_min` | Window comparator minimum |
| `win_comp_max` | Window comparator maximum |
| `FIFO_Prog_Thresh` | FIFO programmable threshold |
| `EXT_DISC_REQ` | External discriminator request |
| `ext_disc_ts_sel` | External disc timestamp select |
| `dac_channel_select` | DAC channel select |
| `dac_attenuation` | DAC attenuation setting |
| `DIAG_DISC_SEL` | Diagnostic discriminator select |
| `DIAG_WAVE_SEL` | Diagnostic waveform select |
| `diag_mux_control` | Diagnostic multiplexer control |
| `diag_input` | Diagnostic input select |
| `diag_input_en` | Diagnostic input enable |
| `lfsr_rate_sel` | LFSR pulser rate select |
| `lfsr_seed` | LFSR seed value |
| `sd_rx_pwr` | SERDES RX power control |
| `sd_tx_pwr` | SERDES TX power control |
| `sd_sync` | SERDES sync command |
| `sd_local_loopback_en` | SERDES local loopback enable |
| `sd_line_loopback_en` | SERDES line loopback enable |
| `sd_pem` | SERDES PEM mode |
| `sd_sm_stringent_lock` | SERDES state machine stringent lock |
| `sd_sm_lost_lock_flag_rst` | SERDES lost-lock flag reset |

✅ verified 2026-04-25 — complete list extracted from `dgsGlobals_DGS_VME99.db` system-level GLBL records (non-detector-type-prefixed).

---

## Instantiation per VME Crate

Each VME crate boot file (`ioc/boot/vme0N.cmd`) calls `dbLoadRecords()` for every populated board:

```
dbLoadRecords("db/MDigRegisters.template",  "CRATE=01,BOARD=MDIG1")
dbLoadRecords("db/MDigUser.template",       "CRATE=01,BOARD=MDIG1")
dbLoadRecords("db/MDigUserVME.template",    "CRATE=01,BOARD=MDIG1")
dbLoadRecords("db/daqSegment2.template",    "CRATE=01,BOARD=MDIG1")
dbLoadRecords("db/daqCrate.template",       "CRATE=01")
dbLoadRecords("db/dgsGlobals_DGS_VME01.db")
```

### Board→Template Mapping

| Board Type | Registers template | User template | VME template |
|------------|--------------------|---------------|--------------|
| MDIG (Master Digitizer) | `MDigRegisters` | `MDigUser` | `MDigUserVME` |
| SDIG (Slave Digitizer) | `MDigRegisters`* | `MDigUser`* | `SDigUserVME` |
| RTRG (Router Trigger) | `RTrigRegisters` | `RTrigUser` | — |
| MTRG (Master Trigger) | `MTrigRegisters` | `MTrigUser` | — |

*SDigRegisters and SDigUser are identical files to their M counterparts.

### Known board assignments per crate (from boot files)

✅ verified 2026-04-23 — all entries cross-checked against `ANLDAQ/ioc/boot/vme*.cmd`

| Crate | Boards | Slots (Board#→Slot) |
|-------|--------|---------------------|
| VME01 | MDIG1, SDIG1, MDIG2, SDIG2 | MDIG1@2, SDIG1@3, MDIG2@4, SDIG2@5 |
| VME02 | MDIG1, SDIG1, MDIG2, SDIG2 | MDIG1@2, SDIG1@3, MDIG2@4, SDIG2@5 |
| VME03 | MDIG1, SDIG1, MDIG2, SDIG2, RTR1 | MDIG1@2, SDIG1@3, MDIG2@4, SDIG2@5, RTR1@7 |
| VME04 | MDIG1, SDIG1, MDIG2, SDIG2 | MDIG1@2, SDIG1@3, MDIG2@4, SDIG2@5 |
| VME05 | MDIG1, SDIG1, MDIG2, SDIG2 | MDIG1@2, SDIG1@3, MDIG2@4, SDIG2@5 |
| VME06 | MDIG1, SDIG1, RTR2 | MDIG1@2, SDIG1@3, RTR2@7 |
| VME07 | MDIG1, SDIG1, MDIG2, SDIG2 | MDIG1@2, SDIG1@3, MDIG2@4, SDIG2@5 |
| VME08 | MDIG1, SDIG1, MDIG2, SDIG2 | MDIG1@2, SDIG1@3, MDIG2@4, SDIG2@5 |
| VME09 | MDIG1, SDIG1, MDIG2, SDIG2, RTR3 | MDIG1@2, SDIG1@3, MDIG2@4, SDIG2@5, RTR3@7 |
| VME10 | MDIG1, SDIG1, MTRG | MDIG1@2, SDIG1@3, MTRG@5 |
| VME11 | MDIG1, SDIG1, MDIG2, SDIG2 | MDIG1@2, SDIG1@3, MDIG2@4, SDIG2@5 |
| VME12 | MDIG1, SDIG1, MDIG2, SDIG2, RTR4 | MDIG1@2, SDIG1@3, MDIG2@4, SDIG2@5, RTR4@7 |
| VME66 | MDIG1, MDIG2, RTR1, MTRG | MDIG1@3, MDIG2@4, RTR1@6, MTRG@7 |
| VME99 | MDIG1, MDIG2, RTR1, MTRG | MDIG1@2, MDIG2@4, RTR1@6, MTRG@7 (lab test stand) |

Note: VME66 and VME99 have no SDIG boards — only MDIG. VME10 has MTRG at board# 4 (slot 5).

---

## user_package_data Numbering

> **Reference:** Full table and formula in [`ioc.md` — *Boot Scripts* section](ioc.md#production-gs-crate-slot-map-vme01vme12). ✅ verified 2026-04-23

Formula: `[(crate# − 1) × 4] + 101 + board#` (board# 0=MDIG1, 1=SDIG1, 2=MDIG2, 3=SDIG2).  
MTRG = 150 (fixed). VME66=170/171, VME99=160/161 (lab systems, outside formula).

---

## IOC Boot Sequence

> **Reference:** Full 15-step boot sequence in [`ioc.md` — *Boot Script Details*](ioc.md#boot-script-details). ✅ verified 2026-04-23

DB-loading steps relevant to templates (steps 3–4 of boot sequence):
1. `dbLoadDatabase "dbd/gretDet.dbd"` — registers all DTYP/record types
2. `dbLoadRecords` for every board (Registers → User → UserVME), then `daqSegment2`, `daqCrate`, `dgsGlobals`
3. `iocInit()` — all PVs become live
4. `dbpf user_package_data` — sets board-ID values; `dbpf CS_Ena = 1` enables all boards
5. `seq &inLoop/outLoop/MiniSender` — starts DAQ state machines

---

## EPICS Channel Access Port Assignments

> **Reference:** Full port table in [`ioc.md` — *EPICS CA Port Map*](ioc.md#epics-ca-port-map-from-cdcommands). ✅ verified 2026-04-23

Key values: DGS=5064/5065 (default), Xarray=5072/5073, DUO=5080/5081, vme99 lab=5074/5075.

---

## Board Type Encoding

> **Reference:** Full encoding table in [`ioc.md` — *Board Type Encoding*](ioc.md#board-type-encoding--code_revision118). ✅ verified 2026-04-23

Key values used in DGS production: AMD (0xC) = ANL Master Digitizer; ASD (0xD) = ANL Slave Digitizer; DMT (4) = DGS Master Trigger; DRT (6) = DGS Router Trigger.

---


## Fifo Select Encoding

Used by `VME$(CRATE):$(BOARD):FifoNum` (mbbo, daqSegment2 + dummy in daqCrate).

| Value | String | Meaning |
|-------|--------|---------|
| 0–5 | `MONFIFO 1`–`6` | Monitor FIFO 1–6 |
| 6 | `MAIN DATA FIFO` | Primary data readout (default) |
| 7 | `MONFIFO 8` | Monitor FIFO 8 |
| 8–15 | `CHAN A`–`CHAN H FIFO` | Per-channel FIFOs A–H |

---

## `gretDet.dbd` — EPICS Database Definition File

_Source: `vxworks/dgsIoc/tcDetApp/src/gretDet.dbd`. Loaded by each VME crate IOC at step 3 of the boot sequence above._

The `.dbd` file registers all record types, device support entries, drivers, and registrars with the EPICS IOC. Key architectural notes from its inline comments:

### Device Type (DTYP) Registrations

| DTYP name | Status | Notes |
|-----------|--------|-------|
| `Soft Channel` | Active | Standard EPICS; used for analog-scaled PVs linked to hardware records |
| `Raw Soft Channel` | Active | Same as above but uses `.RVAL` field instead of `.VAL` |
| `Async Soft Channel` | Active (unused) | Same as Soft Channel but returns immediately; **no DGS PV uses this DTYP** as of 2026-04-25 |
| `Soft Timestamp` | Active (unused) | EPICS time access; **no DGS PV uses this DTYP** |
| `General Time` | Active (unused) | Real-time clock access; **no DGS PV uses this DTYP** |
| `asynInt32`, `asynUInt32Digital`, `asynFloat64`, `asynOctet*`, `asynXxxArray` | Active | Asyn wrapper DTYPs — Tim Madden's VME I/O bridge; still required because asyn is linked in, even though the comment notes it "does nothing for us" |
| `Gretina VME Board` | **Commented out** | Legacy Gretina flash-programming DTYPs (`devBoGVME`, `devLIGVME`, etc.); all references to `gretVME.template` were commented out in VME IOC boot scripts as of 2022-07-20 |
| `VX stats` / `VX stats clusts` | **Commented out** | No references in any DGS database; leftover Gretina debris |
| `asynRecordDevice` | Active | Backing device for the `asyn` record type |
| `stdio` | Active | `devSoStdio` — `stringout` record that prints to VxWorks console |

### Registrar Notes (inline comments)

- **`equalSubRegistrar`** — removed 2026-04-25 (JTA); `equalSub.c` was previously registered here but is now removed from the IOC registrar list. ✅ verified 2026-04-26 — `gretDet.dbd` inline comment `<== JTA 20250425 REMOVED`
- **`devGDigRestoreRegistrar`**, **`save_restoreRegister`**, **`dbrestoreRegister`**, **`asInitHooksRegister`** — all removed 2022-07-29 (MBO); save/restore functionality not used in DGS. ✅ verified 2026-04-26 — `gretDet.dbd` inline comments `<== MBO 20220729 REMOVED`
- **`flashOpsRegistrar`** — Gretina flash ops registrar; removed (flash programming done via `devGVME.c` IOCshell commands, not Gretina's method)
- **`BuildSendRegistrar`** — removed; replaced by `MiniSender` state machine
- **`registerReboot`** — old Gretina junk; deleted
- **Active registrars:** `asSub`, `asynRegister`, `asynInterposeFlushRegister`, `asynInterposeEosRegister`, `devGVMERegistrar`, `asynDebugRegister`, `asynDigitizerRegister`, `asynTrigMasterRegister`, `asynTrigRouterRegister`, `inLoopRegistrar`, `outLoopRegistrar`, `MiniSenderRegistrar`

### EPICS Variables

| Variable | Type | Notes |
|----------|------|-------|
| `asCaDebug` | int | Access Security CA debug flag |
| `dbRecordsOnceOnly` | int | EPICS DB flag |
| `dbBptNotMonotonic` | int | EPICS breakpoint table flag |

---

## See Also

- `knowledgeBase/ioc.md` — IOC startup: how `vme*.cmd` files instantiate these templates (cardno, IP, port assignments)
- `knowledgeBase/IOC_cmd.md` — Full IOC shell command reference; asynDigitizerConfig, ProgramFlash, VMERead32 commands that accompany these DB records
- `knowledgeBase/EPICS_RTrig_templates.md` — RTrig DB templates deep-dive: complete RTrigRegisters + RTrigUser PV inventory (split from this file)
- `knowledgeBase/EPICS_implementation_tools.md` — DBGEN macro/substitution workflow that generates templates for new hardware
- `knowledgeBase/EPICS.md` — EPICS primer: record types (mbbo, mbbi, longout, waveform) used throughout these templates
- `knowledgeBase/EPICS_asyn.md` — How asyn translates caput/caget into VME register reads/writes (the underlying mechanism these templates rely on)
- `knowledgeBase/DGS_PVs.md` — Full DGS PV list (all records instantiated from these templates for the 12-crate DGS system)
- `knowledgeBase/VME_registers.md` — VME register address map for DIG/MTRG/RTRG; matches the register indices referenced in `daqSegment2.template`, `MTrigUser.template`, etc.
- `knowledgeBase/vxworks.md` — VxWorks IOC build pipeline; the `.dbd` files that register these template records with the EPICS database
- `knowledgeBase/data_structures.md` — DIG event packet format; DIG firmware fields that the `MDigUser.template` PVs expose (e.g. `trigger_mux_select`, `code_revision`)

---

*Source: `ANLDAQ/ioc/db/` template files + `ANLDAQ/ioc/boot/vme*.cmd` instantiation scripts. Created: 2026-04-23.*
