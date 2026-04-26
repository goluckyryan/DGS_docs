# gui_SYS.py — ANLDAQ System Overview Window

Stability: C2 - Active / semi-stable

_Source: `DGS_tools_pack/ANLDAQ/gui/gui_SYS.py` (589 lines)_ ✅ verified 2026-04-25 — `wc -l gui_SYS.py` = 589
_Documented: 2026-04-25_

**See also:** [`ANLDAQ_gui_internals.md`](ANLDAQ_gui_internals.md) — other GUI components, [`ANLDAQ_commander.md`](ANLDAQ_commander.md) — commander.py, [`link_sys_analysis.md`](link_sys_analysis.md) — Link System internals

---

## Overview

`gui_SYS.py` implements a multi-tab system overview window in the DGS ANLDAQ commander GUI.
It provides cross-board visibility of timestamps, link status, TCP readout, firmware revisions, and global channel settings.
All tabs inherit from `sysTemplateTab` and use the `pvWidgetList` + `QTimer` pattern for live 500 ms PV polling.

---

## Table of Contents

1. [Base Class: `sysTemplateTab`](#base-class-systemplatetab)
2. [`sysTimestampReadOutTab` — Timestamp + Readout Status](#systimestampreadouttab--timestamp--readout-status)
3. [`sysLinktab` — Link Status, Lock, Mask, and Control](#syslinktab--link-status-lock-mask-and-control)
4. [`sysTCPTab` — TCP Transfer Status](#systcptab--tcp-transfer-status)
5. [`sysCodeRevisionTab` — Firmware Code Revision & Date](#syscoderevisiontab--firmware-code-revision--date)
6. [`globalSettingTab` — Global Channel & Board Settings](#globalsettingtab--global-channel--board-settings)
7. [PV Name Conventions by Board Type](#pv-name-conventions-by-board-type)

---

## Base Class: `sysTemplateTab`

All tab classes extend `sysTemplateTab(QWidget)`. Constructor signature:
```python
sysTemplateTab(MTRG: Board, RTR_list, DIG_list, DAQ_list, parent=None)
```

Key methods:
- `FindPV(pv_name, board, isDAQ=False) -> PV`: finds a PV in `board.Board_PV` by name suffix.
  - Normal boards: matches `pv.name.split(":")[-1] == pv_name`
  - DAQ boards: strips first segment (`"_".join(pv.name.split("_")[1:]) == pv_name`)
  ✅ verified 2026-04-25 — `gui_SYS.py:L29-36` (exact logic confirmed)
- `UpdatePVs(forced=False)`: iterates `pvWidgetList`; calls `pvWidget.UpdatePV(forced)` if tab is visible.
- All tabs call `self.timer.start(500)` → 500 ms live polling. ✅ verified 2026-04-25 — `gui_SYS.py:L171,L311,L351,L417`

---

## `sysTimestampReadOutTab` — Timestamp + Readout Status

Shows real-time timestamps and readout enable state for all boards.

### Timestamp Group
Displays `TIMESTAMP_A/B/C` (hex) for MTRG and all RTRGs; `live_timestamp_msb/lsb` (hex) for DIGs.
Also shows `ClkSrc` toggle for MTRG and each RTRG; `clk_select` for each DIG.
IMP_SYNC button at top right triggers `IMP_SYNC` PV on MTRG.

PV names by board type:

| Board | Timestamp PVs | Clock Source PV |
|-------|--------------|----------------|
| MTRG  | `reg_TIMESTAMP_A`, `reg_TIMESTAMP_B`, `reg_TIMESTAMP_C` | `ClkSrc` |
| RTRG  | `reg_TIMESTAMP_A`, `reg_TIMESTAMP_B`, `reg_TIMESTAMP_C` | `ClkSrc` |
| DIG   | `live_timestamp_msb`, `live_timestamp_lsb` (2-word only) | `clk_select` |

### Readout Group
Shows `CS_Ena` (enable/disable) and `FifoNum` (FIFO select) for MTRG;
`CS_Ena` for each DIG. These control whether each board's readout loop is active.

---

## `sysLinktab` — Link Status, Lock, Mask, and Control

Comprehensive view of SERDES link health for MTRG and all RTRGs.

### Link Status (`reg_MISC_STAT` / `reg_MISC_STAT_REG`)
`RRegisterDisplay` widget shows per-link signal flags decoded from the MISC_STAT register.
- MTRG: `reg_MISC_STAT` (decoded with `isRTR=False`)
- RTRG: `reg_MISC_STAT_REG` (decoded with `isRTR=True`)

### Link Lock Status
`RMapTwoStateButton` grid (1 row per board) showing lock state for all 11 links:
`LOCK_A/B/C/D/E/F/G/H/L/R/U` ✅ verified 2026-04-25 — `gui_SYS.py:L213` (`lock_pvNameList` all 11 links confirmed)
Color inverted (lock lost = red, locked = green).

### Input Link Mask
Same 11-link grid using `ILM_A…ILM_U` PVs.
Masked links are excluded from multiplicity sums. ✅ verified 2026-04-25 — `gui_SYS.py:L241` (`LIM_pvNameList` = ILM_A…ILM_U, 11 links)

### Link L Control
Per-board buttons for MTRG and each RTRG for link control PVs:

| MTRG PV           | RTRG PV                | Label               |
|-------------------|------------------------|---------------------|
| `LOCK_RETRY`      | `LOCK_RETRY`           | Lock Retry          |
| `LOCK_ACK`        | `LOCK_ACK`             | Lock Ack            |
| `RESET_LINK_INIT` | `RESET_LINK_INIT`      | Reset Link Init     |
| `LINK_L_STRINGENT`| `STRINGENT_LOCK`       | Link L Stringent    |
| `LINK_R_STRINGENT`| *(none)*               | Link R Stringent    |
| `LINK_U_STRINGENT`| *(none)*               | Link U Stringent    |
| *(none)*          | `SM_LOST_LOCK_RESET`   | Reset Lock Lost     |
✅ verified 2026-04-25 — `gui_SYS.py:L271` (`mtrg_LLC_pvNameList`) + `L289` (`rtrg_LLC_pvNameList`); 7-entry aligned lists confirmed

---

## `sysTCPTab` — TCP Transfer Status

Monitors TCP data output buffer health for each DAQ computer (IOC):

| Column | PV (isDAQ=True) | Description |
|--------|----------------|-------------|
| Buffers | `CV_BuffersAvail` | Available output buffers |
| Send Buffs | `CV_NumSendBuffers` | Number of pending send buffers |
| TCP Rate | `CV_SendRate` | Current TCP transfer rate |

✅ verified 2026-04-25 — `gui_SYS.py:L337,L341,L345` (all three PV names confirmed)

Uses `isDAQ=True` path in `FindPV()` (strips leading segment from PV name).
One row per entry in `DAQ_list`.

---

## `sysCodeRevisionTab` — Firmware Code Revision & Date

Scrollable table showing firmware revision and build date for all boards.
Displayed as hex values via `RLineEdit(hexBinDec="hex")`.

PV names by board type:

| Board | Revision PV | Date PV |
|-------|-------------|---------|
| MTRG  | `reg_CODE_REVISION` | `reg_CODE_DATE` |
| RTRG  | `Code_Revision`     | `CODE_DATE`     |
| DIG   | `regin_code_revision` | `code_date`   |

Note the inconsistent naming across board types (MTRG uses `reg_` prefix; RTRG drops it; DIG uses `regin_`). ✅ verified 2026-04-25 — `gui_SYS.py:L83-85` (MTRG `reg_TIMESTAMP_A/B/C`), `L104-106` (RTRG same), `L125-126` (DIG `live_timestamp_msb/lsb`); code revision cross-checked `L357-414`

---

## `globalSettingTab` — Global Channel & Board Settings

**Only visible when `SYSTEM=DGS` environment variable is set.** ✅ verified 2026-04-25 — `gui_SYS.py:L429` (`os.environ.get("SYSTEM") != "DGS"` early return)

Write-only mass-set controls — values set here write immediately to all matching boards/channels but do **not** display current values (no readback). Warning label displayed in orange.

### Channel Parameters Group

Sets a named PV for **all channels of a given detector type at once**.
4 detector type columns: `Ge Center`, `BGO`, `Ge Side`, `Aux`.

Detector type → board/channel mapping:

| Det Type | Board Filter | Channels |
|----------|-------------|----------|
| `Ge Center` (GeC) | MDIG boards | 5–9 |
| `BGO`             | MDIG boards | 0–4 |
| `Ge Side` (GeS)   | SDIG boards | 5–9 |
| `Aux`             | SDIG boards | 0–4 |

Parameters (20 total): ✅ verified 2026-04-25 — `gui_SYS.py:L459-478` (20 entries confirmed)

| Label | PV Name | Type |
|-------|---------|------|
| Ch. On/Off | `channel_enable` | Combo: Reset / Run |
| Polarity | `trigger_polarity` | Combo: Disabled / RiseEdge / FallEdge / Both |
| K0 Window | `k0_window` | Numeric |
| K Window | `k_window` | Numeric |
| D Window | `d_window` | Numeric |
| M Window | `m_window` | Numeric |
| D3 Window | `d3_window` | Numeric |
| LED Threshold | `led_threshold` | Numeric |
| Raw Length | `raw_data_length` | Numeric |
| Raw Delay | `raw_data_delay` | Numeric |
| P1 Window | `p1_window` | Numeric |
| P2 Window | `p2_window` | Numeric |
| PARST Delay | `preamp_reset_delay` | Numeric |
| CFD Fraction | `CFD_fraction` | Numeric |
| Disc Width | `disc_width` | Numeric |
| Coarse Disc Width | `coarse_disc_thresh` | Numeric |
| PileUp Mode | `pileup_mode` | Combo: Reject / Accept |
| Preamp Reset En | `preamp_reset_delay_en` | Combo: Disabled / Enabled |
| P2 Mode | `P2_mode` | Combo: Separate / Span |
| Downsample Factor | `downsample_factor` | Combo: 1x–128x (8 steps, powers of 2) |

### Digitizer Board Settings Group

Sets a named PV on **all boards** simultaneously (no per-detector-type filtering).

Parameters (15 total): ✅ verified 2026-04-25 — `gui_SYS.py:L518-534` (15 entries confirmed)

| Label | PV Name | Type / Options |
|-------|---------|----------------|
| LED/CFD Mode | `cfd_mode` | Combo: LED_Mode / CFD_Mode |
| Trig Mode | `trigger_mux_select` | Combo: IntAcptAll / ExtTTL / ExtTTCL / Diag |
| Win Comp Min | `win_comp_min` | Numeric |
| Win Comp Max | `win_comp_max` | Numeric |
| Peak Sens. | `peak_sensitivity` | Numeric |
| Holdoff Time | `holdoff_time` | Numeric |
| HO Stop at Peak | `stop_ho_at_peak` | Combo: OFF / ON *(yellow bg)* |
| Counter Mode | `counter_mode` | Combo: internal / SERDES |
| Diag Mux Control | `diag_mux_control` | Combo: rawADC / RunSumM / FiltSimDisc / NULL |
| Diag Wave Select | `DIAG_WAVE_SEL` | Combo: ADC / cfd / run_pre / run_post |
| Throttle Mode | `rj45_throttle_mode` | Combo: half full / PROG FULL / LFSR / 0 / 1 |
| FIFO Prog. Thresh. | `FIFO_Prog_Thresh` | Combo: ANY / 6.25% … 93.75% (16 steps) |
| LFSR Rate | `lfsr_rate_sel` | Combo: FAST / MED FAST / MED SLOW / SLOW |
| LFSR Seed | `lfsr_seed` | Numeric |
| Load Delays | `load_delays` | Combo: No / Yes *(green bg)* |

`SetAllBoardsPV()` iterates all DIG boards and calls `pv.SetValue(value)`.
`SetDetTypePV()` selects MDIG or SDIG boards, then iterates the matching channel range.
✅ verified 2026-04-25 — `gui_SYS.py:L556-579` (GeC/BGO → MDIG ch 5-9/0-4; GeS/Aux → SDIG ch 5-9/0-4)

Board settings PV names, options, and highlight colors (yellow for `stop_ho_at_peak`, green for `load_delays`) ✅ verified 2026-04-26 — `gui_SYS.py:L518-534` (all 15 entries), `gui_SYS.py:L549-552` (style sheet)

---

## PV Name Conventions by Board Type

Note the inconsistent PV prefix naming across board families (a historical artifact):

| PV Category | MTRG prefix | RTRG prefix | DIG prefix |
|-------------|-------------|-------------|------------|
| Timestamp A | `reg_TIMESTAMP_A` | `reg_TIMESTAMP_A` | `live_timestamp_msb` |
| Code Revision | `reg_CODE_REVISION` | `Code_Revision` | `regin_code_revision` |
| Code Date | `reg_CODE_DATE` | `CODE_DATE` | `code_date` |
| Clock Source | `ClkSrc` | `ClkSrc` | `clk_select` |
| Misc Status | `reg_MISC_STAT` | `reg_MISC_STAT_REG` | — |

---

## Cross-References

| File | Relationship |
|------|--------------|
| `ANLDAQ.md` | Parent ANLDAQ overview; this file split from it |
| `ANLDAQ_GUI_windows.md` | Other ANLDAQ GUI window documentation |
| `ANLDAQ_gui_internals.md` | GUI helper classes and internals |
| `deep_fpga_MTRG_MAIN.md` | MTRG FPGA internals (PVs shown in this window) |
| `deep_fpga_RTRG.md` | RTRG FPGA internals (PVs shown in this window) |
| `VME_registers.md` | Register map for PVs displayed here |

---

*Created: 2026-04-25 | Last reviewed: 2026-04-25*
