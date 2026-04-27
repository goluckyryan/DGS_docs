# ANLDAQ — GUI Window Reference

Stability: C2 - Active / semi-stable

_Split from `ANLDAQ.md` on 2026-04-16. Covers all board/subsystem window modules + trigger setup scripts._  
_Source: `DGS_tools_pack/ANLDAQ/gui/`._

**See also:** [`ANLDAQ.md`](ANLDAQ.md) — parent overview | [`ANLDAQ_gui_internals.md`](ANLDAQ_gui_internals.md) — GUI helper classes + deep gui_MTRG.py/gui_RTR.py/gui_SYS.py docs (class_PV, class_Board, findAllPV, RComboBox)

_Last reviewed: 2026-04-27 — gui_MTRG.py and gui_RTR.py sections replaced with stubs pointing to deep docs in ANLDAQ_gui_internals.md (code-read 2026-04-27)_

---

## Table of Contents

1. [GUI: Master Trigger Window (`gui_MTRG.py`)](#gui-master-trigger-window-gui_mtrgpy)
2. [GUI: Detector Array Window (`gui_Det.py`)](#gui-detector-array-window-gui_detpy)
3. [GUI: Per-Detector Collector Box Window (`gui_GS.py`)](#gui-per-detector-collector-box-window-gui_gspy)
4. [GUI: Scalar Window (`gui_scalar.py`)](#gui-scalar-window-gui_scalarpy)
5. [GUI: Router Trigger Window (`gui_RTR.py`)](#gui-router-trigger-window-gui_rtrpy)
6. [GUI: RAM Window (`gui_RAM.py`)](#gui-ram-window-gui_rampy)
7. [GUI: System Window (`gui_SYS.py`)](#gui-system-window-gui_syspy)
8. [Trigger Setup Scripts (`gui/scripts/`)](#trigger-setup-scripts-guiscripts)
9. [GUI: Link System Window (`gui_LinkSys.py`)](#gui-link-system-window-gui_linksyspy)
10. [GUI: Digitizer Board Window (`gui_DIG.py`)](#gui-digitizer-board-window-gui_digpy)
11. [GUI: Per-Channel Window (`gui_CH.py`)](#gui-per-channel-window-gui_chpy)
12. [GUI: Generic Board PV Window (`gui_Board.py`)](#gui-generic-board-pv-window-gui_boardpy)
13. [GUI: Data Taking Window (`gui_DataTaking.py`)](#gui-data-taking-window-gui_datatakingpy)
14. [Guceiver — Online Waveform/Spectrum Viewer](#guceiver--online-waveformspectrum-viewer)
15. [See Also](#see-also)

---

## GUI: Master Trigger Window (`gui_MTRG.py`)

`MTRGWindow` (1,386 lines) — opened by clicking an MTRG board button in DGS Commander. 5-tab QTabWidget plus a top-level board info panel. 500 ms QTimer refresh.

> 📖 **Full documentation** (deep code-read 2026-04-27): **[`ANLDAQ_gui_internals.md § gui_MTRG.py`](ANLDAQ_gui_internals.md)**
> Covers: `templateTab` base class, `MTRGWindow` header panel (rate/diagnostic counters, board info), all 5 tabs (triggerControlTab, wheelRAMTab, linkControlTab, CPLDControlTab, otherControlTab), and timer lifecycle.

_Quick reference:_

| Tab | Class | Contents |
|-----|-------|----------|
| Trigger/Veto Control | `triggerControlTab` (L36) | 8 algorithms, veto masks, prescale, X/Y thresholds |
| Wheel RAM | `wheelRAMTab` (L278) | AUX I/O direction, SSI controls, embedded RAMWindow |
| LINK Control | `linkControlTab` (L484) | SERDES links A–H/L/R/U status + LVDS pre-emphasis |
| Trigger/CPLD map | `CPLDControlTab` (L756) | Remote ML logic, CPLD trigger type map |
| Other Control | `otherControlTab` (L974) | NIM outputs, throttle, overlap/assertion delays |

✅ verified 2026-04-27 — `gui_MTRG.py:L36,L278,L484,L756,L974` (class definitions at exact line numbers confirmed)

---

## GUI: Detector Array Window (`gui_Det.py`)

`DetWindow` — opened via "SBX / CollectorBox" button in DGS Commander. Shows all 110 detectors laid out by collector box group.

### Detector → Collector Box Mapping
_Source: `gui_Det.py` header comment (code-verified 2026-04-06)_

| Group | CollectorBox | Detector Numbers | Count | Formula |
|-------|-------------|-----------------|-------|---------|
| North-East (NE) | CB 203 | 1, 3, 5, …, 59 (odd) | 30 | 2i+1, i=0..29 |
| South-East (SE) | CB 201 | 2, 4, 6, …, 60 (even) | 30 | 2i, i=1..30 |
| North-West (NW) | CB 204 | 61, 63, 65, …, 109 (odd) | 25 | 2i+1, i=30..54 |
| South-West (SW) | CB 202 | 62, 64, 66, …, 110 (even) | 25 | 2i, i=31..55 |

NW and SW only occupy 25 of 30 cable slots; last 5 slots per CB are physically empty (dummy cells).

### Window Layout & Tabs

The window (1100×700, title "SBX / CollectorBox") has:
- **CollectorBox combo** — dropdown (GS 201/202/203/204); index 0 is placeholder (TODO: open CB control window on selection) ✅ verified 2026-04-27 — `gui_Det.py:L91-95` (`addItem("CollectorBox ▾")` as placeholder; L353-354 TODO comment)
- **"All Detectors" button** — stub for future all-detector overview window (currently a no-op) ✅ verified 2026-04-27 — `gui_Det.py:L98-99,L356-357` (`all_det_btn` connected to `_OpenAllDetWindow`; body is TODO comment only)
- **1-second timer** — flushes CA callbacks to widgets; only fires when `isVisible()` ✅ verified 2026-04-27 — `gui_Det.py:L112-114` (`QTimer.start(1000)`); `L344` (`if not self.isVisible(): return`)
- **Lazy CA subscription** — `AddCallback()` not called until first `showEvent()`; `RemoveCallback()` on close ✅ verified 2026-04-27 — `gui_Det.py:L123-140` (`showEvent` calls `pv.AddCallback()`; `closeEvent` calls `pv.RemoveCallback()`)

#### Temperature tab
2×2 `QGridLayout` of `QGroupBox` panels (NE/SE/NW/SW). Each group has 30 slots as **3 columns × 10 rows** (column-major order). Per-slot cell:

| Column | Widget | PV | R/W |
|--------|--------|----|-----|
| ID | `QPushButton` (`MOD###`) | — | click (or Shift+click) opens `GSWindow` |
| TEMP | `RLineEdit` (2 dp) | `MOD###_DV_TEMP` | read-only |
| HIGH | `RLineEdit` | `MOD###_DV_TEMP.HIGH` | R/W (EPICS alarm field, created as manual `PV()` — not in JSON) |
| EN | `RTwoStateButton` | `MOD###_DV_EN` | R/W |

Disabled dummy widgets for absent slots (NW/SW last 5). ✅ verified 2026-04-24 — `gui_Det.py` (Apr 23 commit c2339cb)

#### HV tab
Same 2×2 / 3×10 layout. Per-slot cell:

| Column | Widget | PV | R/W |
|--------|--------|----|-----|
| ID | `QPushButton` (`MOD###`) | — | click opens `GSWindow` for that detector |
| Set | `RLineEdit` (2 dp) | `GS###_GE_HV_DEMAND_VOLTS` | R/W — Ge HV demand voltage |
| Out | `RLineEdit` (2 dp) | `GS###_Conv_GeHV` | read-only — Ge HV converter measured |
| High | `RLineEdit` (2 dp) | `MOD###_DS_GEHV` | read-only — HV setpoint from SlopeBox |

Note: column order in widget layout is ID → Set (dv_w) → Out (conv_w) → High (ds_w). ✅ verified 2026-04-24 — `gui_Det.py:L270-310` (current file, Apr 23 commit)

#### Availability detection
Slot is "available" if `f"MOD{det_id:03d}"` is in `cb_det_list` (MOD-prefixed names from `CollectorBox_PV.json`). Unavailable slots: disabled placeholders with no CA connections. ✅ verified 2026-04-20 — `gui_Det.py:L67-68`

## GUI: Per-Detector Collector Box Window (`gui_GS.py`)

`GSWindow` — per-detector CollectorBox PV viewer. Opened from `DetWindow` (Detector Array Window) when the user clicks any detector button in the array grid. Shows CollectorBox-sourced status and control PVs for a single Gammasphere detector.

_Source: `ANLDAQ/gui/gui_GS.py` (209 lines as of Apr 23 commit c2339cb, code-read 2026-04-24)_ ✅ verified 2026-04-24

### How It Opens

- Clicking a detector ID button (`MOD###`) in `DetWindow`'s Temperature or HV grid calls `_OpenGSWindow(det_id, new_window=shift_held)` ✅ verified 2026-04-24 — `gui_Det.py:L201-207` (Temperature tab) + HV tab
- Default behavior: **reuses a single shared `_gs_window`** — switches to new detector in place; if Shift is held, a new independent `GSWindow` is spawned and appended to `_gs_window_extras` ✅ verified 2026-04-24 — `gui_Det.py`
- On `DetWindow.closeEvent()`, all extra GSWindow instances are closed alongside the primary one ✅ verified 2026-04-27 — `gui_Det.py:L130-140` (`closeEvent` closes `_gs_window` + loops `_gs_window_extras` to close each)

### Window Layout

- Title: `GS###` (e.g. `GS042`); size: 500×400 px ✅ verified 2026-04-27 — `gui_GS.py:L22` (`self.resize(500, 400)`)
- **Detector dropdown** at top: lists all available detector IDs (`GS###` format); switching selection calls `SwitchTo()` which rebuilds the whole content grid ✅ verified 2026-04-27 — `gui_GS.py:L35,L60,L72` (`available_det_ids` loop builds dropdown; `SwitchTo` called on init and on combo change)
- Two side-by-side `QGroupBox` columns:
  - **Info / Status** — identity fields + status bits + voltage readbacks (read-only)
  - **Control** — writable control PVs (scan, HV control)

### Info / Status Column PVs

| Label | PV | Type |
|-------|----|------|
| Ge ID | `GS###_Ge_ID` | int readback |
| SlopeBox ID | `GS###_SlopeBox_ID` | int readback |
| Ge Prefix | `GS###_Ge_Prefix` | int readback |
| VME Index | `GS###_VME_Index` | int readback |
| Dig Index | `GS###_Dig_Index` | int readback |
| Dig Channel | `GS###_Dig_Channel` | int readback |
| Ge HV | `GS###_SlopeBoxGe_HV_On` | enum (read-only combobox) |
| Temp | `GS###_SlopeBoxTempHigh` | enum (read-only combobox) |
| BGO HV | `GS###_SlopeBoxBGO_HV_On` | enum (read-only combobox) |
| BGO Interlock | `GS###_SlopeBoxBGOInterlock` | enum (read-only combobox) |
| BGO 400V | `GS###_Conv_BGO400` | float (2 dp, read-only) |
| BGO 450V | `GS###_Conv_BGO450` | float (2 dp, read-only) |
| 24V | `GS###_Conv_24V` | float (2 dp, read-only) |
| +12V | `GS###_Conv_plus12V` | float (2 dp, read-only) |
| -12V | `GS###_Conv_minus12V` | float (2 dp, read-only) |
| 5V | `GS###_Conv_5V` | float (2 dp, read-only) |

✅ verified 2026-04-23 — `gui_GS.py:L95-162` (int_pvs, status_pvs, voltage_pvs lists)

### Control Column PVs

| Label | PV | Type |
|-------|----|------|
| Scan Control | `GS###_Slopebox_Scan_control` | writable combobox |
| Ge HV Ctrl | `GS###_GE_HV_CTRL` | writable combobox |
| BGO HV Ctrl | `GS###_BGO_HV_CTRL` | writable combobox |

✅ verified 2026-04-23 — `gui_GS.py:L164-176` (ctrl_pvs list)

### Update Behavior

- **1-second `QTimer`** calls `_UpdateWidgets()` which iterates all `RLineEdit`/`RComboBox` widgets and calls `.UpdatePV()` ✅ verified 2026-04-23 — `gui_GS.py:L62-64,L183-187`
- CA callbacks subscribe on `showEvent()` and unsubscribe on `closeEvent()` (lazy subscription, same pattern as `DetWindow`)
- Detector switch: `_OnDetChanged()` unsubscribes current PVs, clears layout, rebuilds with new `GS###` prefix, re-subscribes — all content recreated dynamically ✅ verified 2026-04-23 — `gui_GS.py:L83-101`

### Notes

- PVs are sourced from `cb_pv` (Collector Box PV list, passed in from `DetWindow`) — same `PV` objects shared with `DetWindow`
- `_FindPV(pv_name)` does a linear scan through `cb_pv` by `.name` equality — no dict lookup ✅ verified 2026-04-23 — `gui_GS.py:L72-75`
- All voltage PVs (`_Conv_*`) are explicitly set `ReadONLY=True`; status PVs (`_SlopeBox*`) also set `ReadONLY=True` ✅ verified 2026-04-23 — `gui_GS.py:L127-130,L147-150`

---

## GUI: Scalar Window (`gui_scalar.py`)

`ScalarWindow` — "Scalar - All Digitizers" window. Shows all digitizer boards in a scrollable grid of GroupBoxes. Per channel per board, displays (live PV, timer-updated):
- `led_threshold` — LED threshold setting
- `disc_count` — discriminator fire count
- `ahit_count` — accepted hit count

✅ verified 2026-04-19 — `gui_scalar.py:L115,L121,L127` (FindChannelPV calls for `led_threshold`, `disc_count`, `ahit_count` confirmed)

_Source: `gui_scalar.py` commit `0f3f2df` 2026-04-06 (code-verified)_

---

## GUI: Router Trigger Window (`gui_RTR.py`)

`RTRWindow` (542 lines) — opened by clicking an RTRG board button in DGS Commander. 2-tab QTabWidget plus a top-level board info panel. 500 ms QTimer refresh.

> 📖 **Full documentation** (deep code-read 2026-04-27): **[`ANLDAQ_gui_internals.md § gui_RTR.py`](ANLDAQ_gui_internals.md)**
> Covers: `RTRWindow` header panel (board info, LED controls, diagnostic counters, other controls), both tabs (rtrlinkControlTab, rtrXYMapTab), and timer lifecycle.

_Quick reference:_

| Tab | Class | Contents |
|-----|-------|----------|
| LINK Control | `rtrlinkControlTab` (L13) | Per-link SERDES grid + LRU controls + LVDS pre-emphasis |
| X/Y Map | `rtrXYMapTab` (L221) | XMAP/YMAP bit grids, DISCRIMINATOR_DELAY, X/Y SELECT combos |

---

## GUI: RAM Window (`gui_RAM.py`)

`RAMWindow` (34 lines) — minimal popup window showing a **32×32 grid** of `RMapTwoStateButton` cells, each bound to a PV. Used to visualize MTRG RAM buffer contents (VETO_RAM, TRIG_RAM, SWEEP_RAM). Opened from the MTRG tab via a RAM selector combo box. Updates every **500 ms** via `QTimer`.

**Three RAM types** (from `gui_MTRG.py:L393`):
- `VETO_RAM` — trigger veto pattern RAM
- `TRIG_RAM` — trigger pattern RAM
- `SWEEP_RAM` — sweep pattern RAM

Each RAM has a corresponding `_ADDR_SRC` PV (`VETO_RAM_ADDR_SRC`, etc.) that selects the address source; shown separately in the MTRG tab rather than in the RAM window grid.

_Source: `ANLDAQ/gui/gui_RAM.py` (34 lines) + `gui_MTRG.py:L393-482` (code-read 2026-04-11)_

---

## GUI: System Window (`gui_SYS.py`)

`gui_SYS.py` (427 lines) — provides the system-level tabs embedded in the main `DGS Commander` window. All tab classes inherit from `sysTemplateTab` (same visible-only update pattern as other GUI modules: `QTimer` → `UpdatePVs()` skips hidden tabs). Update interval: **500 ms** for all system tabs.

_Source: `ANLDAQ/gui/gui_SYS.py` (code-read 2026-04-09)_

### Tab Classes

| Class | Tab Name | Contents |
|-------|----------|----------|
| `sysTimestampReadOutTab` | **Timestamp** | MTRG + per-RTR + per-DIG timestamp readbacks (hex); Imp Sync toggle; STARTING_TIMESTAMP HI/MID/LOW writeable fields; scrollable |
| `sysLinktab` | **Link Status** | Three sub-panels: (1) **Link Status** — `reg_MISC_STAT`/`reg_MISC_STAT_REG` full bit-field display for MTRG + each RTR; (2) **Link Lock Status** — per-link lock indicator grid (LOCK_A … LOCK_U) for MTRG + each RTR, inverted color (locked=green); (3) **Input Link Mask** — per-link ILM_A … ILM_U grid for MTRG + each RTR (inverted: masked=red); (4) **Link L Control** — MTRG: LOCK_RETRY/LOCK_ACK/RESET_LINK_INIT/LINK_L/R/U_STRINGENT; per-RTR: LOCK_RETRY/LOCK_ACK/RESET_LINK_INIT/STRINGENT_LOCK/SM_LOST_LOCK_RESET ✅ verified 2026-04-17 — `gui_SYS.py:L175-305` |
| `sysTCPTab` | **TCP Transfer** | Per-IOC DAQ stats: `CV_BuffersAvail`, `CV_NumSendBuffers`, `CV_SendRate` (live readback from EPICS DAQ PVs) |
| `sysCodeRevisionTab` | **Code Revision** | All boards in one scrollable table: MTRG (`reg_CODE_REVISION`/`reg_CODE_DATE`), each RTR (`Code_Revision`/`CODE_DATE`), each DIG (`regin_code_revision`/`code_date`) — all displayed in hex |
| `globalSettingTab` | **Global Settings** | System-wide register controls (threshold multipliers, global enables, etc.) |

✅ verified 2026-04-19 — `commander.py:L301-305` (`addTab` labels confirmed; corrected "Timestamps" → "Timestamp")

### Key Notes
- PV naming differs by board type for code revision: MTRG uses `reg_CODE_REVISION`, RTR uses `Code_Revision`, DIG uses `regin_code_revision` (note prefix `regin_` vs `reg_`) ✅ verified 2026-04-16 — `gui_MTRG.py:L1188`, `gui_RTR.py:L300`, `gui_SYS.py:L407`, `gui_DIG.py:L41`
- TCP tab takes only `DAQ_list` (not MTRG/RTR/DIG); other tabs take all board lists
- Timestamp tab shows MTRG + all RTRs + all DIGs in one scrollable panel — useful for verifying timestamp sync across the full chain

---

## Trigger Setup Scripts (`gui/scripts/`)

The `scripts/` directory contains 5 staged shell scripts that initialize the full trigger chain via `caput`. They are invoked by the GUI's "SERDES Link-Up" button (`Serdes_Linkup.sh`).

| Stage | Script | What It Does |
|-------|--------|--------------|
| 1 | `trig_setup_Stage1.sh` | Set MTRG to local clock; enable all SERDES links; drive SYNC pattern out all A–H/L/U links |
| 2 | `trig_setup_Stage2.sh` | Initialize all RTRGs to local clock; set Link L to receive MTRG SYNC; drive SYNC back to MTRG; set router channel masks |
| 3 | `trig_setup_Stage3.sh` | Read and verify lock status on all active MTRG↔RTRG and RTRG↔DIG links |
| 4 | `trig_setup_Stage4.sh` | Switch digitizers to send SYNC to their Router; verify Router sees lock on all enabled DIG links |
| 5 | `trig_setup_Stage5.sh` | Flip DIGs and RTRGs (in order) from SYNC to real data; verify nothing erroneous after each flip |

### `SYSTEM_DEFINES.sh` — Live Gammasphere Trigger Topology

All 5 stage scripts take `SYSTEM_DEFINES.sh` as their first argument. This file defines the **live system topology** for Gammasphere:

| Variable | Value | Meaning |
|----------|-------|---------|
| `MT_VME_LEADER` | `VME10` | MTRG crate | ✅ verified 2026-04-23 — `SYSTEM_DEFINES.sh:L9`
| `MT_USE_LINK_CLK` | `0` | Local clock (not remote/link clock) | ✅ verified 2026-04-23 — `SYSTEM_DEFINES.sh:L113`
| `DIG_CLOCK_SEL` | `1` | Digitizer clock source = OSC (internal oscillator, enum value 1) | ✅ corrected 2026-04-27 — `MDigUserVME.template:L83-84` (`ONST="OSC",ONVL=1`); `SYSTEM_DEFINES.sh:L120`; `Stage4.sh:L163` echo confirms `1=internal`. Previous doc incorrectly said "SERDES".
| `PROPAGATE_TRIG_FROM_DUB/DFMA/DXA` | `0` | No remote trigger propagation | ✅ verified 2026-04-23 — `SYSTEM_DEFINES.sh:L108-110`
| `PERFORM_ERROR_CHECKS` | `0` | Error checking disabled | ✅ verified 2026-04-23 — `SYSTEM_DEFINES.sh:L124`

**Router config** (`LIST_OF_ROUTERS`): 4 RTRGs, each using links A–F (6 Router channels) + Link L back to MTRG: ✅ verified 2026-04-23 — `SYSTEM_DEFINES.sh:L18-24`
```
VME03:RTR1  A B C D E F X X  L X X
VME06:RTR2  A B C D E F X X  L X X
VME09:RTR3  A B C D E F X X  L X X
VME12:RTR4  A B C D E F X X  L X X
```

**MTRG link map** (`MT_LINK_MAP`): Links A–D → RTR1–RTR4; E–H and L/R/U masked: ✅ verified 2026-04-23 — `SYSTEM_DEFINES.sh:L93-104`
```
RTR1 → Link A    RTR2 → Link B    RTR3 → Link C    RTR4 → Link D
E, F, G, H, L, R, U → MASKED
```

**Digitizer config** (`LIST_OF_DIGITIZERS`): 12 crates; VME06 and VME10 have only 2 DIGs, rest have 4: ✅ verified 2026-04-23 — `SYSTEM_DEFINES.sh:L27-39`
```
VME01–05, 07–09, 11–12: MDIG1 MDIG2 MDIG3 MDIG4  (4 boards each)
VME06, VME10:            MDIG1 MDIG2               (2 boards each)
```
Total: 10×4 + 2×2 = **44 digitizer boards × 10 ch = 440 channels** ✅ verified 2026-04-07 — `SYSTEM_DEFINES.sh`

Key details:
- All stages read a `SYSTEM_DEFINES.sh` file (passed as arg 1) that defines `MT_VME_LEADER`, `LIST_OF_ROUTERS`, etc.
- Stage 1 sets `ClkSrc` and all `LINK_L_PROPAGATE_Fx` registers on MTRG; also sets `EN_RTR_DCBAL 1` (DC balance on MTRG→Router links — required since July 2022 fiber expander installation; see below)
- Stage 2 loops over `LIST_OF_ROUTERS` with bash substring extraction (`${RTR:6:3}`) to parse VME address
- Stage 5 clears SYNC bits (the gotcha described in `troubleshooting.md`) — this is the final step that lets real data flow
- `basic_settings_LED.py` — sets all DIG channels to **LED mode** with hardcoded defaults: threshold=300, `IntAcptAll` trigger, `p1=0.07µs, p2=0.05µs, m=2.5µs, k0=0.5µs, k=0.5µs, d=0.16µs`, RiseEdge polarity, raw_data_delay=0.5µs, raw_data_length=0.32µs. Targets CH 5–9 on VME66 MDIG1/MDIG2 (test stand config). Uses `epics.caput` with wait=True. Ends with `Online_CS_StartStop=Stop`, `Online_CS_SaveData=No Save`.
- `enableScriptList.txt` — lists which scripts are enabled in the GUI

**DC Balance (`EN_RTR_DCBAL`):**
The MTRG SERDES links to Routers use a DC-balance encoding scheme (18-bit physical link = 16-bit data + CG/POL bits). Stage 1 enables this with `caput ${MT_VME_LEADER}:MTRG:EN_RTR_DCBAL 1`. Required since July 2022 when the VME Fiber Expander replaced direct copper SERDES cables — fiber links are more sensitive to DC imbalance. The algorithm (origin: GRETINA/LBNL, `DCBAL.doc`) inverts the 16-bit word when doing so improves DC balance; decision = XNOR of MSB(data disparity) and MSB(channel running disparity); latency = 1 clock cycle.

**Important EPICS write caveat** (from `Serdes_Linkup.sh` comments):
- Writing to a **whole-register PV** (e.g. `VME10:MTRG:reg_INPUT_LINK_MASK`) updates that PV and its `_RBV`, but does **NOT** update the "breakout" bit PVs (e.g. `ILM_A` through `ILM_H`)
- Writing to a **breakout PV** updates the register and the whole-reg `_RBV`, but does **NOT** update the whole-reg PV itself
- **Scripts must always use breakout PVs** (not whole-reg PVs) to match what the GUI displays

_Source: `ANLDAQ/gui/scripts/trig_setup_Stage*.sh` + `Serdes_Linkup.sh` (code-verified 2026-04-06)_

---

## `gui_LinkSys.py` — Link System GUI Window

295-line PyQt6 window that wraps `link_sys.py`'s `LinkSys` class with a graphical interface. Opened from the main GUI's "Link System" button.

**Key constants:**
- `LINK_IDS = ["A","B","C","D","E","F","G","H","L","R","U"]` — all 11 SERDES links
- `MTRG_TYPE_OPTIONS = ["MASKED","PIXIE","DFMA","DUB","DXA"]` — external system types for MTRG link map

**GUI layout:**
- **MTRG Link Map** (`QGroupBox`): one row per link (A–H, L, R, U); columns = link ID / type combo (`rtr_names + MTRG_TYPE_OPTIONS`) / propagate combo (0/1, enabled **only** for L, R, U — hidden for A–H)
- **Router Link Map** (`QGroupBox`): rows=routers, columns=links A–H,L,R,U; each cell is a `QCheckBox` (checked=active=1, unchecked=disabled=0; `state=2` masked+powered is handled inside `link_sys.py` directly, not from this GUI)
- **Settings** (`QGroupBox`): Error Check checkbox (off by default), MTRG Clock Source combo (local/external), DIG Clock combo (0:AUX / 1:SERDES / 2:Oscillator / 3:SERDES, default=1)
- Status label + **Run LinkSys** / **Cancel** buttons

**Config persistence** (`SaveConfig`/`LoadConfig`): JSON `linkMap.json` in the gui directory saves/restores full MTRG map, RTR checkbox grid, error-check flag, clock source, and DIG clock selection. Loaded automatically on window open.

**`LinkSysWorker(QThread)`**: runs the full 5-stage `LinkSys` sequence in a background thread so the GUI stays responsive. Emits `stageUpdate(str)` (progress label) and `finished(bool, str)` (success/failure) signals. Runs all 5 stages sequentially — there is no stage-selector; the GUI always runs Stages 1–5 in order.

_Source: `ANLDAQ/gui/gui_LinkSys.py` (verified 2026-04-23)_

---

## GUI: Digitizer Board Window (`gui_DIG.py`)

`DIGWindow` (374 lines) — opened by clicking a DIG board button in DGS Commander. Shows all status and controls for a single digitizer board in a single-level grid layout (no tabs).

_Source: `ANLDAQ/gui/gui_DIG.py` (code-read 2026-04-16)_ ✅ verified 2026-04-16

### Layout — Group Boxes

| Group | Contents |
|-------|----------|
| **Board Info / Status** | Code Revision, Code Date, VME Code Rev., Serial No., Timestamps (MSB/LSB), Geo Addr, Board ID, FW Type, LED State, Power OK, Over/Under Volt Stat, 3× Temp Sensors, Misc Logic Status, SD Config, VME gp Ctrl, Ext Disc Src/Mode, TS Err Count |
| **SerDes Status/Control** | SERDES Lock, SM Locked, Lost Lock Flag, Rx/Tx Pwr, Local/Line Loopback, PEM, Sync, Stringent Lock |
| **FIFO Status/Control** | Master FIFO Reset, FIFO A/B Empty, FIFO A/B Full, FIFO Almost Full, Prog Flag, FIFO Depth, FIFO Prog Err/Flg |
| **Channel Triggers/Controls** | 10-row table: per-channel Enable toggle, LED Threshold, Disc Count, Accepted Hit Count, Downsample Factor. Bottom row: "All Ch." threshold field, Counter Mode combo. "Open Channel" button → opens `CHWindow` |
| **Throttle Control** | Throttle Mode, LFSR Rate, Prog Throttle Mode, LFSR Seed |
| **Board Control** | Master Logic Enable, Readout (CS_Ena), Trigger Mode (`trigger_mux_select`), CFD Mode, Comp Win Min/Max, Veto Enable, Clock Source, Reset Lost Lock, Ext Disc TS, Downsample Holdoff, Diag Mux Control, Diag Wave Select, VME Disc Req |
| **ADC Status** | Phase Shift Overflow, DCM clock stopped, DCM Reset, DCM Lock, DCM Ctrl Status |
| **Acquisition Status** | Same 5 fields for ACQ clock domain |
| **Phase Status** | Phase Check, Hunting Down/Up, Phase Failure/Success |

### Key behaviors
- `UpdatePVs()` runs every **500 ms** via `QTimer`; only updates if window is **visible**
- During ACQ run (`isACQRunning=True`): forces update on `led_threshold`, `channel_enable`, `disc_count`, `ahit_count`
- Closing the window stops the timer and calls `board.UnsubscribeChannels()` (unregisters CA callbacks)
- `SetAllChThreshold()` writes the same threshold to all 10 channels simultaneously
- Widget type auto-selected: `RComboBox` (>2 states), `RTwoStateButton` (2 states), `RLineEdit` (scalar; hex display if flagged)

---

## GUI: Per-Channel Window (`gui_CH.py`)

`CHWindow` (402 lines) — opened by "Open Channel" in `DIGWindow`. Shows all per-channel parameters for a single digitizer board, organized in **5 tabs**. Each tab updates every **500 ms** via `QTimer`; updates skip hidden tabs.

_Source: `ANLDAQ/gui/gui_CH.py` (code-read 2026-04-16)_ ✅ verified 2026-04-16

### Tab Structure

| Tab | Class | Contents |
|-----|-------|----------|
| **Channel** | `ChannelTab` | Drop-down to select ch 0–9; then shows all settings for that channel in group boxes. Switching channel re-binds all widgets to the new channel's PVs. |
| **Window Settings** | `SettingsTabTemplate` | Scrollable grid: rows=ch 0–9, columns=`led_threshold`, `k0`, `k`, `d`, `d3`, `m`, `p1`, `p2`, Trace Delay, Trace Length. "All" row at bottom sets all channels at once. |
| **General Settings** | `SettingsTabTemplate` | Scrollable grid: same pattern for Enable, Polarity, Pileup, CFD E-Sum Mode, CFD Frac, Preamp Reset En, Preamp Reset, Downsample, Dec Pause, Trig TS Mode, Early Pre-M Capture, Mux Word Select, Control reg |
| **Ext. Discr.** | `SettingsTabTemplate` | Mode and Source per channel |
| **Status** | `SettingsTabTemplate` | Enable, Trigger count, Accepted Trig, Accepted Event, Dropped Event, Count Reset — `forceUpdate=True` so it always polls live even when not changing |

### `ChannelTab` — single-channel detail view
Group boxes within the Channel tab:
- **Window Settings** — Threshold, k0/k/d/d3/m/p1/p2, Trace Delay, Trace Length. "Load Delays" `RSetButton` applies timing parameters to FPGA.
- **Status** — Trigger count, Accepted Trigger count, Accepted Event count, Dropped Event count, Counter Reset, count mode combos.
- **General Settings** — all per-channel boolean/enum controls (Enable, Polarity, Pileup, CFD Mode, etc.)
- **Ext. Discr.** — Mode and Source combos.

### `SettingsTabTemplate` — bulk view
A reusable widget that renders a grid where **rows = channels (0–9)**, **columns = PV fields**. Each cell is an `RComboBox`, `RTwoStateButton`, or `RLineEdit` auto-selected by PV state count. An "All" row at the bottom broadcasts the same value to all channels via `_setAllChannels(pvName, value)`.

Tab switch triggers `UpdatePVs(forced=True)` to re-populate newly visible tab immediately.

---

## GUI: Generic Board PV Window (`gui_Board.py`)

`BoardPVWindow` (432 lines) — a generic catch-all viewer opened for board types that don't have a dedicated window (anything that's not DIG, MTRG, or RTR). Also used internally by `DIGWindow` / `MTRGWindow` / `RTRWindow` for specialized sub-views.

_Source: `ANLDAQ/gui/gui_Board.py` (code-read 2026-04-16)_ ✅ verified 2026-04-16

### Layout strategy

The window auto-detects PV categories and builds specialized group boxes for each:

| Detected category | PV prefix/pattern | Rendered as |
|-------------------|-------------------|-------------|
| XMAP/YMAP | `XMAP_`, `YMAP_`, `DISCRIMINATOR_DELAY` | 2D checkbox grid (same as RTR X/Y Map tab) |
| SERDES links | `LOCK`, `DEN`, `REN`, `SYNC`, `RPwr`, `TPwr`, `SLiL`, `SLoL`, `ILM`, `LINK`, `GATED_THROTTLE`, `RAW_THROTTLE` | Per-link button rows |
| Diagnostics | `Diag_`, `LOCK_COUNT` | Labeled diag display |
| FIFO Reset | `FIFOReset` | Reset buttons |
| RAM (MTRG) | `VETO_RAM`, `TRIG_RAM`, `SWEEP_RAM` | Open `RAMWindow` buttons |
| MTRG Link Propagate | `LINK_L/R/U_PROPAGATE` | Toggle buttons per link |
| Veto enables | `EN_NIM_VETO_`, `EN_RAM_VETO_`, `EN_REMTRIG_VETO_`, `EN_SOFTWARE_VETO_`, `EN_THROTTLE_VETO_` | Button grid |
| Everything else | all remaining PVs | Auto-widget scroll list |

If the board has channels (`NumChannels > 0`), a channel selector combo is shown at the top. Selecting a channel switches the PV list to that channel's PVs.

All widgets update every **500 ms** via `QTimer`; updates skip hidden windows.

---

## GUI: Data Taking Window (`gui_DataTaking.py`)

Two classes in this module handle run start/stop and live receiver output:

_Source: `ANLDAQ/gui/gui_DataTaking.py` (code-read 2026-04-16)_ ✅ verified 2026-04-16

### `IOCConfigDialog`
A modal dialog for editing the IOC connection list fed to `tcpReceiverMT`. Format: one IOC per line — `IP  Port  DataType` (port defaults to 9001, DataType defaults to 8; lines with `#` are comments). ✅ verified 2026-04-19 — `gui_DataTaking.py:L12,L21-23` (class docstring + label text confirm format exactly)

### `RunStatusWindow`
A `QMainWindow` that manages a live DAQ run:

**On open:**
1. Creates run folder: `{expFolder}/{expName}_{runNum:03d}/`
2. Writes `ioc_config.txt` from the IOC config dialog
3. Auto-compiles `tcpReceiverMT` if needed (`make tcpReceiverMT` in `ANLDAQ_DIR/tcpReceiver/`)
4. Spawns `tcpReceiverMT <config_file> <file_prefix>` as a subprocess

**Live display (200 ms poll):**
- Live log output (ANSI codes stripped, max 2000 lines)
- Parses `======  X.XXX Mbytes | ...` lines to show total data size
- Status label: Starting → Running → Finished / Stopped / Exited(N)

**Stop run flow:**
1. Prompts for a stop comment (if manual)
2. Calls `parent.StopAcquisition(comment)` to flush IOC data (triggers EPICS `Online_CS_StartStop=Stop`)
3. Waits **5 seconds**, then sends `SIGTERM` to `tcpReceiverMT` ✅ verified 2026-04-19 — `gui_DataTaking.py:L209` (`QTimer.singleShot(5000, self._TerminateReceiver)` — 5s QTimer before SIGTERM)
4. On `SIGTERM` exit (code = -SIGTERM): status = "Stopped"; on 0: "Finished"; other: "Exited(N)" ✅ verified 2026-04-19 — `gui_DataTaking.py:L188` (`elif retcode == -signal.SIGTERM`)

**Key constants:**
- `ANLDAQ_DIR` — from env var or auto-detected as parent of `gui/`
- `RECEIVER_BIN` — `{ANLDAQ_DIR}/tcpReceiver/tcpReceiverMT`

---

## Guceiver — Online Waveform/Spectrum Viewer

> **Full reference moved to [`guceiver.md`](guceiver.md)** (split 2026-04-20 to eliminate duplication).

**Location:** `ANLDAQ/gui/Guceiver/`  
**Entry point:** `Guceiver.py` — `python3 Guceiver.py`  
**Purpose:** Standalone PyQt6 GUI for online monitoring of DIG and TAC-II data streams in real time. Connects directly to a VxWorks IOC TCP server (port 9001) and decodes raw digitizer packets — **not** via tcpReceiverMT. Separate tool from the main ANLDAQ GUI.

See **[`guceiver.md`](guceiver.md)** for: architecture diagram, class_Receiver.py protocol, DIG/TAC-II decoder field tables, tab descriptions, and key design notes.

---

## See Also

- [`ANLDAQ.md`](ANLDAQ.md) — parent overview (class_PV, class_Board, findAllPV, commander.py)
- [`ANLDAQ_gui_internals.md`](ANLDAQ_gui_internals.md) — GUI helper classes: class_PV, class_Board, findAllPV, RComboBox (used throughout window classes)
- [`ANLDAQ_commander.md`](ANLDAQ_commander.md) — DGS Commander run control GUI (commander.py, board buttons, system tabs)
- [`ANLDAQ_tcpReceiver.md`](ANLDAQ_tcpReceiver.md) — tcpReceiverMT deep-dive: protocol, GEB header, run control scripts
- [`guceiver.md`](guceiver.md) — Guceiver online waveform/spectrum viewer (full reference)
- [`trig_setup_scripts.md`](trig_setup_scripts.md) — trigger setup scripts (detailed cross-reference)
- [`link_sys_analysis.md`](link_sys_analysis.md) — link_sys.py 5-stage sequence (called by gui_LinkSys.py)
- [`deep_fpga_MTRG_MAIN.md`](deep_fpga_MTRG_MAIN.md) — MTRG firmware details for trigger tab PVs
- [`deep_fpga_RTRG.md`](deep_fpga_RTRG.md) — RTRG firmware details for RTR window PVs

*Created: 2026-04-13 | Last reviewed: 2026-04-24*
