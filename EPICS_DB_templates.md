# EPICS DB Templates — DGS IOC

**Source:** `ANLDAQ/ioc/db/` (all `.template` files) and `ANLDAQ/ioc/boot/vme*.cmd` (instantiation)
**Date documented:** 2026-04-23
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
15. [Board Type Encoding](#board-type-encoding)
16. [Fifo Select Encoding](#fifo-select-encoding)

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

Raw hardware register readback PVs, prefix `VME$(CRATE):$(BOARD):reg_*`.

Key readback PV families (suffix `_RBV`):
- `reg_channel_controlN_RBV` — per-channel control register (channels 0–9)
- `reg_CFD_fractionN_RBV` — per-channel CFD fraction register
- `reg_baseline_delay_RBV` — baseline delay register
- `reg_d3_windowN_RBV` — D3 coincidence window registers
- `reg_channel_pulsed_controlN` — pulsed control (write-only style)

SDigRegisters.template is identical to MDigRegisters.template (same layout, both use `MDigRegisters`
naming internally).

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

One static `.db` file per VME crate (e.g. `dgsGlobals_DGS_VME01.db`, `_VME06.db`, etc.).

These are **not** templates — they are pre-expanded EPICS DB files generated for a specific crate
configuration. They contain `dfanout` records that fan out global set-all commands to individual
board PVs. Examples:

- `VME99:GLBL:master_fifo_reset` → fans out to `VME99:MDIG1:master_fifo_reset`, `VME99:SDIG1:master_fifo_reset`
- `VME01:GLBL:BGOs_ext_disc_src` → fans to channel-level `ext_disc_src0`–`4` on MDIG1
- `VME01:GLBL:GeC_ext_disc_src` → fans to `ext_disc_src5`–`9` on MDIG1 (GeC = central Ge contact)
- `VMEnn:MDIG1:BGOs_ext_disc_src` — per-board sub-fanout to per-channel PVs

This two-level fanout (global → board → channel) allows setting all BGO or Ge channels at once.

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

After `iocInit()`, each digitizer (and VME12's RTR4) gets a unique 8-bit `user_package_data` value set via `dbpf`. This value is embedded in every data packet for offline crate/board identification.

Formula for digitizers: `[(crate# − 1) × 4] + 101 + board#`
- Board# 0 = MDIG1, 1 = SDIG1, 2 = MDIG2, 3 = SDIG2

| Crate | MDIG1 | SDIG1 | MDIG2 | SDIG2 | Other |
|-------|-------|-------|-------|-------|-------|
| VME01 | 101 | 102 | 103 | 104 | — |
| VME02 | 105 | 106 | 107 | 108 | — |
| VME03 | 109 | 110 | 111 | 112 | — |
| VME04 | 113 | 114 | 115 | 116 | — |
| VME05 | 117 | 118 | 119 | 120 | — |
| VME06 | 121 | 122 | — | — | — |
| VME07 | 123 | 124 | 125 | 126 | — |
| VME08 | 127 | 128 | 129 | 130 | — |
| VME09 | 131 | 132 | 133 | 134 | — |
| VME10 | 135 | 136 | — | — | — |
| VME11 | 137 | 138 | 139 | 140 | — |
| VME12 | 141 | 142 | 143 | 144 | RTR4=154 |
| VME66 | 170 | — | 171 | — | — |
| VME99 | 160 | — | 161 | — | — |

Master trigger: `USER_PACKAGE_DATA = 150` (fixed, applies globally to MTRG).
Router triggers: RTR4 in VME12 = 154; future RTRs = 151, 152, etc. (per comment in vme01.cmd).
Note: VME66/VME99 have non-sequential IDs (170/171 and 160/161); these are lab/special systems outside the main formula.

✅ verified 2026-04-23 — cross-checked all `dbpf "VMExx:*:user_package_data"` lines in all `vme*.cmd` boot files

---

## IOC Boot Sequence

Each VME crate runs the same boot sequence via its `vme<NN>.cmd` script:

1. **cd** to boot directory; source `cdCommands` and `nfsCommands`
2. **ld** the binary: `ld < gretDet.munch` (VxWorks image with all EPICS tasks + inLoop/outLoop/MiniSender)
3. **dbLoadDatabase** `gretDet.dbd` — registers all DTYP/record types
4. **dbLoadRecords** for every board (Registers → User → UserVME templates), then `daqSegment2`, `daqCrate`, `dgsGlobals`
5. **InitializeDaqBoardStructure()** — sets up internal DAQ board mapping
6. **asynDigitizerConfig / asynTrigRouterConfig1 / asynTrigMasterConfig1** — maps board name to board# and physical VME slot
7. **asynDebugConfig("DBG",0)** — enables debug peek/poke interface
8. **iocInit()** — starts EPICS IOC; all PVs become live
9. **setupFIFOReader()** — initializes the three-queue buffer pool (see vxworks.md)
10. **dbpf** `user_package_data` — sets board-ID values (digitizer formula above)
11. **dbpf** `CS_Ena = 1` — software-enables all digitizer boards for readout (MTRG/RTR not set here)
12. **seq &inLoop** — starts inLoop sequencer with `CRATE=NN,B0=…,B6=…` board list
13. **seq &outLoop** — starts outLoop sequencer with `CRATE=NN`
14. **seq &MiniSender** — starts MiniSender sequencer with `CRATE=NN`
15. **dbl > vme<NN>_db.txt** — dumps all PV names to a text file (run in `/startup`)

The `B0`–`B6` parameters in `seq &inLoop` map physical slot positions to board names. Slots with no board use `B<N>=X`. The value `X` causes inLoop to look for a PV `X_CS_Ena`, which doesn't exist and is silently skipped.

✅ verified 2026-04-23 — cross-checked steps against `vme01.cmd`, `vme10.cmd`, `vme99.cmd`

---

## EPICS Channel Access Port Assignments

The boot files for special/lab crates set non-default CA ports:

| System | CA Server Port | Repeater Port | Note |
|--------|---------------|---------------|------|
| DGS (standard) | 5064 | 5065 | Default EPICS ports (not set explicitly in vme01–12) |
| DFMA | 5068 | 5069 | |
| Xarray | 5072 | 5073 | |
| G-wing | 5074 | 5075 | VME99 uses these (lab test stand) |
| F-wing / microball | 5078 | 5079 | |

These port values are from comments in `vme99.cmd`. Standard DGS crates (vme01–12) do not override ports and use the EPICS defaults (5064/5065).

✅ verified 2026-04-23 — from `vme99.cmd` putenv comments; absence of putenv lines confirmed in `vme01.cmd`

---

## Board Type Encoding

Used by `DAQC$(CRATE)_BoardType0`–`6` (mbbi records). Value comes from `code_revision[11:8]`.

| Value | String | Meaning |
|-------|--------|---------|
| 0 | `0?` | Unknown |
| 1 | `GRT` | GRETINA Router Trigger |
| 2 | `GMT` | GRETINA Master Trigger |
| 3 | `GD` | LBNL Digitizer (GRETINA digitizer) |
| 4 | `DMT` | DGS Master Trigger |
| 5 | `5?` | Unknown |
| 6 | `DRT` | DGS Router Trigger |
| 7 | `7?` | Unknown |
| 8 | `MYR` | MyRIAD |
| 9–11 | `9?`–`11?` | Unknown |
| 12 | `AMD` | ANL Master Digitizer |
| 13 | `ASD` | ANL Slave Digitizer |
| 14–15 | `14?`–`15?` | Majorana Digitizer (unimplemented in mbbi) |

Note: For ANL digitizers (`AMD`/`ASD`), the low 16 bits of `code_revision` encode `4XYZ` where
`X`=master(0)/slave(1), `Y`=major rev, `Z`=minor rev.

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

## See Also

- `knowledgeBase/ioc.md` — IOC startup: how `vme*.cmd` files instantiate these templates (cardno, IP, port assignments)
- `knowledgeBase/IOC_cmd.md` — Full IOC shell command reference; asynDigitizerConfig, ProgramFlash, VMERead32 commands that accompany these DB records
- `knowledgeBase/EPICS.md` — EPICS primer: record types (mbbo, mbbi, longout, waveform) used throughout these templates
- `knowledgeBase/EPICS_asyn.md` — How asyn translates caput/caget into VME register reads/writes (the underlying mechanism these templates rely on)
- `knowledgeBase/DGS_PVs.md` — Full DGS PV list (all records instantiated from these templates for the 12-crate DGS system)
- `knowledgeBase/VME_registers.md` — VME register address map for DIG/MTRG/RTRG; matches the register indices referenced in `daqSegment2.template`, `MTrigUser.template`, etc.
- `knowledgeBase/vxworks.md` — VxWorks IOC build pipeline; the `.dbd` files that register these template records with the EPICS database
- `knowledgeBase/data_structures.md` — DIG event packet format; DIG firmware fields that the `MDigUser.template` PVs expose (e.g. `trigger_mux_select`, `code_revision`)

---

*Source: `ANLDAQ/ioc/db/` template files + `ANLDAQ/ioc/boot/vme*.cmd` instantiation scripts. Created: 2026-04-23.*
