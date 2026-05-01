# ANLDAQ GUI Internals — PyQt6 GUI Components

Stability: C2 - Active / semi-stable

_Split from `ANLDAQ.md` 2026-04-25 for size management._
_Source: `DGS_tools_pack/ANLDAQ/gui/`_

**See also:** [`ANLDAQ.md`](ANLDAQ.md) — top-level ANLDAQ overview, [`ANLDAQ_commander.md`](ANLDAQ_commander.md) — commander.py full reference

---

## Table of Contents

- [`class_PV.py` — EPICS PV Abstraction](#class_pvpy--epics-pv-abstraction)
- [`class_Board.py` — Board Abstraction](#class_boardpy--board-abstraction)
- [`findAllPV.py` — PV List Generator (ioc/)](#findallpvpy--pv-list-generator-ioc)
- [CollectorBox PV Utilities (`collectorBox/`)](#collectorbox-pv-utilities-collectorbox)
- [`json2pv.py` — PV List Parser](#json2pvpy--pv-list-parser)
- [`gui_DataTaking.py` — Run Control & Data Taking](#gui_datatakingpy--run-control--data-taking)
- [`custom_QClasses.py` — Custom Qt Base Widgets](#custom_qclassespy--custom-qt-base-widgets)
- [`gui_scalar.py` — Scalar / Rate Monitor Window](#gui_scalarpy--scalar--rate-monitor-window)
- [`class_PVWidgets.py` — PV-Bound Qt Widgets](#class_pvwidgetspy--pv-bound-qt-widgets)
- [`commander.py` — Main Window (summary)](#commanderpy--main-window)
- [`gui_GS.py` — Per-Detector GS Window](#gui_gspy--per-detector-gs-window)
- [`gui_RAM.py` — RAM Visualizer Window](#gui_rampy--ram-visualizer-window)
- [`gui_Det.py` — Detector / CollectorBox Monitor Window](#gui_detpy--detector--collectorbox-monitor-window)
- [`gui_Board.py` — Generic Board PV Window](#gui_boardpy--generic-board-pv-window)
- [`gui_DIG.py` — Digitizer Board Window](#gui_digpy--digitizer-board-window)
- [`gui_CH.py` — Channel Settings Window](#gui_chpy--channel-settings-window)
- [`gui_LinkSys.py` — Link System Configuration Window](#gui_linksyspy--link-system-configuration-window)
- [`gui_MTRG.py` — Master Trigger Board Window](#gui_mtrgpy--master-trigger-board-window)
- [`gui_RTR.py` — Router Trigger Board Window](#gui_rtrpy--router-trigger-board-window)
- [`aux.py` — Utility Helpers](#auxpy--utility-helpers)
- [`gui_SYS.py` — System Status Tab Library](#gui_syspy--system-status-tab-library)
- [`Guceiver/` — GUI Live Receiver & Online Monitor](#guceiver--gui-live-receiver--online-monitor)

---

## `class_PV.py` — EPICS PV Abstraction

Wraps a single EPICS PV with DGS-specific metadata:

| Attribute | Purpose |
|---|---|
| `name` | PV base name (without board prefix) |
| `RBV_exist` | True if a `_RBV` readback PV exists alongside the setpoint |
| `ReadONLY` | True if PV is read-only (no `put`) |
| `States` | List of enum state strings (from `*NAM`/`*ST` fields in JSON) |
| `value`, `char_value` | Cached value from last CA callback |
| `isUpdated` | Flag: True after `OnChange()` callback fires |
| `_cmd_pv`, `_rbv_pv` | Cached `epics.PV` objects (lazy-initialized) |
| `_subscribed` | Ref count — multiple windows can share one PV without duplicate CA connections |

**Key design:** ref-counted subscriptions via `AddCallback()` / `RemoveCallback()`. A PV subscribes to CA only when at least one window is displaying it; unsubscribes when all windows close. This avoids flooding CA with unnecessary monitors.

**RBV pattern:** If `RBV_exist=True`, writes go to `name` and reads/callbacks come from `name_RBV`. If `ReadONLY=True` + `RBV_exist=True`, the PV is display-only (no put).

---

## `class_Board.py` — Board Abstraction

Represents one VME board (DIG, RTRG, MTRG, DAQ):

- `BD_name` — board prefix (e.g. `MDIG0101`, `RTR001`, `MTRG001`) ✅ verified 2026-04-17 — `class_Board.py:L10,L16`
- `Board_PV[]` — list of board-level `PV` objects (subscribed immediately on creation via `AddCallback()`) ✅ verified 2026-04-17 — `class_Board.py:L71` (`pv_b.AddCallback()`)
- `CH_PV[ch][pv_idx]` — 2D array of per-channel `PV` objects (10 ch for DIG) ✅ verified 2026-04-17 — `class_Board.py:L21`
  - Channel PVs are **not subscribed at creation** — only subscribed via `SubscribeChannels()` when a channel window opens, and unsubscribed via `UnsubscribeChannels()` when it closes ✅ verified 2026-04-17 — `class_Board.py:L30,L33-45`
  - This keeps CA connections minimal when boards aren't being inspected
- Board name prefix is stamped onto each PV: `f"{BD_name}:{pv.name}{ch_idx}"` for channels, `f"{BD_name}:{pv.name}"` for board PVs ✅ verified 2026-04-17 — `class_Board.py:L29` (channel), `L67,L69` (board)

---

## `findAllPV.py` — PV List Generator (ioc/)
_Source: `DGS_tools_pack/ioc/findAllPV.py`_

Run once (manually or at build time) to regenerate `All_PV.json` after any IOC boot script or DB template changes.

**Workflow:**
1. Reads `bootFiles.txt` — a list of VxWorks startup script paths (one per line, `#` = comment)
2. **`parse_dbloadrecords(startup_file)`** — scans each startup script for `dbLoadRecords("template", "MACRO=VAL,...")` lines; returns list of `(template_file, {macro: value})` tuples
3. **`parse_template_with_macros(template_file, macros)`** — reads each `.template`/`.db` file:
   - Applies macro substitution (`$(MACRO)` → value) to record names
   - Extracts record fields; ignores display-only fields (`DTYP`, `SCAN`, `DESC`, `DOL`, `ESLO`, `LINR`, `EGU`, `PREC`, `PINI`, `*VL`)
   - Converts `INP`/`OUT` field → `("Type", "IN"/"OUT")`
   - Detects `_RBV` suffix records → merges into base record as `("RBV", "Exist")`; standalone RBV-only records get `("RBV", "ONLY")`
4. Skips `db/asynDebug.template` and `db/dgsGlobals_DGS_VME99.db`
5. Serialises all records to `All_PV.json` as `[["PV_NAME", {field_dict}], ...]`

**Output format** (one entry per record):
```json
["VME32:MTRG:reg_STARTING_TIMESTAMP_HI", {"Type": "OUT", "RBV": "Exist"}]
```
`All_PV.json` is ~113k lines for a full Gammasphere config (all VME crates × all boards × all channels). ✅ verified 2026-04-08 — `ioc/All_PV.json`: 113,059 lines

> 🔧 **Planned simplification:** Replace template-parsing approach with a static per-firmware suffix list + active board config. Design doc: `workspace/GUI_simplify.md`. See QUEUE.md: "ANLDAQ GUI — simplify PV generation".

---

## CollectorBox PV Utilities (`collectorBox/`)

Three files in `ANLDAQ/collectorBox/` handle CollectorBox PV generation, download, and GUI loading:

### `collectorBox/findAllPV.py` — CollectorBox PV List Generator
_Source: `DGS_tools_pack/ANLDAQ/collectorBox/findAllPV.py` (178 lines). Code-read 2026-04-23._ ✅ verified 2026-04-23

CollectorBox-specific variant of `ioc/findAllPV.py`. Parses `.cmd` boot files and `.db` templates from the CollectorBox IOC to generate `CollectorBox_PV.json`.

**Key differences from `ioc/findAllPV.py`:**
- DB files are **flat** in the same directory as the script (no subdirectory); `db/foo.db` maps to `./foo.db` ✅ verified 2026-04-23 — `collectorBox/findAllPV.py:L138-141`
- Supports **both** `$(Macro)` and `${Macro}` substitution styles (CollectorBox EPICS templates use both) ✅ verified 2026-04-23 — `collectorBox/findAllPV.py:L39-42` (`substitute_macros`)
- Skips `unused_*.db` files — these are placeholder records for disconnected detector slots; excluding them keeps those DetNbr values out of the PV list so the GUI correctly treats them as unavailable ✅ verified 2026-04-23 — `collectorBox/findAllPV.py:L146-148`
- Deduplicates `(db_path, macros)` pairs across multiple cmd files using a `seen` set ✅ verified 2026-04-23 — `collectorBox/findAllPV.py:L151-153`

**Workflow:**
1. Reads `bootFiles.txt` — currently contains `st_201.cmd` (single collector box) ✅ verified 2026-04-23 — `collectorBox/bootFiles.txt`
2. `parse_dbloadrecords()` — same as ioc version: scans for `dbLoadRecords("db/foo.db", "MACRO=VAL,...")` lines
3. `parse_template_with_macros()` — per-record macro substitution + field extraction; same `ignore_fields` set as ioc version; merges `_RBV` suffix records
4. Outputs `CollectorBox_PV.json` as `[["GS201_ctl_reset_startup_rom", {field_dict}], ...]`

**PV naming convention:** `GS<DetNbr>_<pvname>` (e.g. `GS201_ctl_reset_startup_rom`). Only prefixes starting with `GS` or `MOD` are kept; others are silently skipped ✅ verified 2026-04-23 — `collectorBox/findAllPV.py:L35-37`

### `collectorBox/rsyncDB.sh` — Download CollectorBox DB Files
_Source: `DGS_tools_pack/ANLDAQ/collectorBox/rsyncDB.sh` (19 lines). Code-read 2026-04-23; corrected 2026-04-27._ ✅ verified 2026-04-27 — `rsyncDB.sh` (line count + IPs confirmed)

One-shot sync script to download the live CollectorBox DB and cmd files from **all 4 running CollectorBox Pi IOC hosts**:
- Requires `ANLDAQ_DIR` environment variable (set by `EPICS_para.sh`); exits with error if absent ✅ verified 2026-04-23 — `rsyncDB.sh:L3-6`
- **Target hosts:** IPs 192.168.203.**42** (pi0), **.26** (pi1), **.88** (pi2), **.149** (pi3); user: `dgs`; key: `~/.ssh/id_ed25519` — iterates all 4 in a loop ✅ verified 2026-04-27 — `rsyncDB.sh:L10,L14-17` (`IPs="42 26 88 149"`; `for IP in $IPs`)
- **⚠️ Correction:** Prior doc said "Target host: 192.168.203.42" (single host) — wrong. Script loops over all 4 Pi IPs and was also documented as 15 lines; actual line count is 19.
- Downloads from each Pi: `db/*.db` and `iocBoot/iocCollectorApp/*.cmd` from `/shared/EPICS/CollectorBox_RevA/` → local `ANLDAQ_DIR/collectorBox/` ✅ verified 2026-04-27 — `rsyncDB.sh:L15-16`
- Run after deploying new CollectorBox firmware/IOC to refresh local templates before re-running `findAllPV.py`

### `collectorBox/cb_json2pv.py` — CollectorBox PV Loader (GUI)
_Source: `DGS_tools_pack/ANLDAQ/gui/cb_json2pv.py` (65 lines). Code-read 2026-04-23._ ✅ verified 2026-04-23

Loads `CollectorBox_PV.json` into `PV` objects for the commander GUI. Called at startup with graceful fallback.

**`LoadCollectorBoxPVs(file_path)`** — single public function:
1. Reads JSON; skips entries without `_` in name or not prefixed with `GS`/`MOD` ✅ verified 2026-04-23 — `cb_json2pv.py:L29-37`
2. Creates `PV` objects via `SetName()`, populates enum state strings from `_STATE_FIELDS` set (ZNAM/ONAM + all ZRST…FFST variants) ✅ verified 2026-04-23 — `cb_json2pv.py:L4-10,L39-42`
3. Sets `ReadONLY=True` if `RBV==ONLY`; `RBVExist=True` if `RBV==Exist`; neither if no RBV field ✅ verified 2026-04-23 — `cb_json2pv.py:L44-52`
4. Returns `(pv_list, board_set)` — `pv_list` sorted by PV name; `board_set` sorted list of unique detector prefixes (e.g. `['GS022', 'GS042', ...]`) ✅ verified 2026-04-23 — `cb_json2pv.py:L54-57`

---

## `json2pv.py` — PV List Parser

Loads `../ioc/All_PV.json` (generated by `ioc/findAllPV.py`) and builds typed PV lists:

**`load_pv_json()`** — groups raw JSON entries by board type:
- `MDIG`/`SDIG` entries → split into per-channel (`*0`–`*9` suffix) vs board-level PVs
- `MTRG` → MTRG board PVs (skips `reg_TRIG_RAM_*`, `reg_VETO_RAM_*`, `reg_SWEEP_RAM_*` — too many)
- `RTR` → RTRG board PVs
- `DAQ` → DAQ PVs (skips `Trace` and `inLoop` entries)

**`FormatPVList()`** — converts raw tuples → `PV` objects:
- Parses `*NAM`/`*ST` fields → `PV.States` (enum strings for combo boxes)
- `RBV=="ONLY"` → ReadOnly + RBV_exist
- `RBV=="Exist"` → RBV_exist (writable setpoint + readback)
- DAQ PVs always ReadOnly

**`GeneratePVLists()`** — top-level call, returns:
```python
DIG_CHANNEL_PV, DIG_BOARD_PV, RTR_BOARD_PV, MTRG_BOARD_PV,
DIG_BOARD_LIST, RTR_BOARD_LIST, MTRG_BOARD_LIST, DAQ_PV, DAQ_LIST
```
All lists sorted by PV name. Called once at startup in `commander.py`. ✅ verified 2026-04-08 — `json2pv.py:L277`

---

## `gui_DataTaking.py` — Run Control & Data Taking

_Source: `ANLDAQ/gui/gui_DataTaking.py` (227 lines). Code-verified 2026-04-09._

Two classes:

**`IOCConfigDialog`** — Modal dialog for editing the IOC connection list before starting a run.
- Text format: one IOC per line: `IP  Port  DataType` (port defaults to 9001, DataType defaults to 8 if omitted; `#` = comment) ✅ verified 2026-04-16 — `tcpReceiverMT.cpp:L55-56,L63` (`cfg.port=9001`, `cfg.dataType=8`)
- User edits inline; result passed to `RunStatusWindow` as `configText` string

**`RunStatusWindow`** — QMainWindow that manages a live run from start to stop.

Run folder structure:
```
{expFolder}/{expName}_{runNum:03d}/
    ioc_config.txt          ← IOC config written here at run start
    {runName}_NNN_BBBB_C    ← per-channel binary data files from tcpReceiverMT
```

Start sequence:
1. Creates `runFolder = expFolder/expName_RRR/` via `os.makedirs(..., exist_ok=True)`
2. Writes `ioc_config.txt` from the dialog's `configText`
3. Auto-compiles `tcpReceiverMT` via `make tcpReceiverMT` in `tcpReceiver/` — skips if already up to date (see [`ANLDAQ_tcpReceiver.md`](ANLDAQ_tcpReceiver.md) for full receiver internals)
4. Spawns `tcpReceiverMT configFile filePrefix` as a subprocess (`subprocess.Popen`, line-buffered stdout)
5. `QTimer` fires every 200 ms to read stdout via `select.select` (non-blocking) ✅ verified 2026-04-18 — `gui_DataTaking.py:L109-111,L156-158` (`timer.start(200)`, `select.select(...,0)`)
6. ANSI color codes stripped (`re.sub(r'\033\[[0-9;]*m', '', line)`) before display
7. Parses `====== X.XXX Mbytes | ...` lines to update live total-size label

Stop sequence:
1. User clicks **Stop Run** → prompts for a stop comment (`QInputDialog.getText`)
2. Calls `parent.StopAcquisition(comment)` (stops EPICS acquisition, lets IOC flush data)
3. Waits **5 seconds** (`QTimer.singleShot(5000, ...)`) then sends **SIGTERM** to `tcpReceiverMT` ✅ verified 2026-04-18 — `gui_DataTaking.py:L209,L215-216`
4. Process exit codes: 0 = finished cleanly, `-SIGTERM` = stopped by user, other = error

Button color scheme: running=green, stopped=orange, finished=blue, error=red.

---

## `custom_QClasses.py` — Custom Qt Base Widgets

| Class | Inherits | Purpose |
|---|---|---|
| `GLabel` | `QLabel` | Right-aligned label (default) |
| `GLineEdit` | `QLineEdit` | Text turns **blue** on edit, black on Enter — visual dirty indicator |
| `GTwoStateButton` | `QPushButton` | Toggles between two text/color states; `setState(bool)` + `stateChanged` signal; supports inverted color logic |
| `GFlagDisplay` | `QWidget` | Label + disabled colored square; green=True, grey=False; tooltip shows pass/fail message |
| `GArrow` | `QWidget` | Custom-painted directional arrow with configurable length, color, angle |

---

## `gui_scalar.py` — Scalar / Rate Monitor Window

_Source: `ANLDAQ/gui/gui_scalar.py` (164 lines). Code-read 2026-04-18._ ✅ verified 2026-04-18

`ScalarWindow` — opened via the "Scalar" button in DGS Commander. Displays per-channel scalar counters for **all DIG boards simultaneously**.

**Per-channel columns** (3 per channel, rows 0–9):

| Column | PV name suffix | Description |
|--------|---------------|-------------|
| Threshold | `led_threshold` | Leading-edge discriminator threshold |
| Trigger | `disc_count` | Discriminator trigger count (raw trigger rate) |
| Ahit | `ahit_count` | Accepted-hit count (triggers passed to trigger chain) |

**Layout algorithm** (`CalcLayout`): ✅ verified 2026-04-18 — `gui_scalar.py:L12-15,75-87`
- Single row if total width ≤ 1500 px (BOX_W=230, GAP=6, PAD=20)
- Wraps to multiple rows if single row too wide but height ≤ 1000 px
- Square-pack with scroll (`QScrollArea`, window capped at 1500×1500) for very large systems

**PV subscription:** channel PVs are looked up from the `Board.CH_PV[ch]` list already subscribed when the `ScalarWindow` is opened (triggered by `commander.py` calling `bd.SubscribeChannels()` for all DIG boards before opening this window). ✅ verified 2026-04-18 — `commander.py:L697`: `for bd in DIG_List: bd.SubscribeChannels()` before `ScalarWindow(...)`

**Timer:** 500 ms refresh via `QTimer`; paused on `closeEvent`, restarted on `showEvent`. ✅ verified 2026-04-18 — `gui_scalar.py:L67-69,146,149`

**Note on `closeEvent`:** contains a branch on `self._system == "DGS"` that is unreachable — `_system` is never set on the instance. In practice the `else` branch always runs (`board.UnsubscribeChannels()` for each DIG board). This is a latent bug (no visible effect since the else path is correct for DGS). ✅ verified 2026-04-18 — `gui_scalar.py:L147-157`

---

## `class_PVWidgets.py` — PV-Bound Qt Widgets

_Source: `ANLDAQ/gui/class_PVWidgets.py` (393 lines, code-read 2026-04-17)_ ✅ verified 2026-04-17

Provides a set of **PV-aware Qt widgets** — each wraps a `PV` object, reads its current value via `UpdatePV()`, and writes back via `SetPV()` on user interaction. All widgets follow the same pattern: on update tick, check `pv.isUpdated` flag; if set, pull `pv.value` and refresh the widget; clear the flag.

| Class | Base | Description |
|---|---|---|
| `RLineEdit` | `GLineEdit` | Text field bound to a PV. Displays value as decimal, hex, or binary (`hexBinDec` param). Special cases: CFD_fraction → `{:.3f}` format; `win_comp_min/max` → divide by 100 if `abs(val) > 10`. Read-only PVs shown in darkgray. `returnPressed` → `SetPV()` |
| `RTwoStateButton` | `GTwoStateButton` | Toggle button bound to a PV. Label text comes from `pv.States[0/1]`. Click toggles state and calls `caput(int(state))`. Read-only PVs are disabled. |
| `RSetButton` | `RTwoStateButton` | Momentary "set" button — writes 1 then 0 immediately (pulse). Used for one-shot actions. Always shows fixed label text. |
| `RComboBox` | `QComboBox` | Dropdown bound to a PV. Populated from `pv.States`. `currentIndexChanged` → `caput(index)`. Emits `whenIndexZero(bool)` signal when value is 0 (used for conditional UI enable/disable). |
| `RMapTwoStateButton` | `QWidget` | Grid of `RTwoStateButton` cells (rows × cols) bound to a flat `pvList`. Optional row/col labels (A–H for cols, 0-N for rows). Each cell is 20×20 px. Used for MTRG RAM maps (VETO_RAM, TRIG_RAM, SWEEP_RAM). `UpdatePV(forced=True)` iterates all cells. |
| `RMapLineEdit` | `QWidget` | Grid of `RLineEdit` cells (rows × cols) bound to a flat `pvList`. Similar layout to `RMapTwoStateButton`. Each cell is 40×20 px. Used for displaying register arrays across boards/channels. |
| `RRegisterDisplay` | `QWidget` | 16-bit register decoder. Reads one PV and displays each bit as a small colored square. Two modes: `isRTR=True` (RTR status bits) vs `isRTR=False` (MTRG status bits). Bit labels differ: MTRG has "Trig Veto"; RTR has "CPLD 1/2/4/8", "R Lock", "Fast Str". Read-only (no write). Used in the MTRG top panel for `MISC_STAT` display. |

**Naming convention:** `R`-prefix = register-bound widget (reads/writes a PV). `G`-prefix (in `custom_QClasses.py`) = generic widget (no PV binding).

**`RRegisterDisplay` bit layout** (bit 0 = LSB):

| Bit | MTRG label | RTR label |
|-----|-----------|----------|
| 0 | NIM in B | NIM in B |
| 1 | NIM in A | NIM in A |
| 2 | TS roll | R Lock |
| 3 | Fast Str | Fast Str |
| 4–7 | rsvd | CPLD 1/2/4/8 |
| 8–11 | L init State 1/2/4/8 | L init State 1/2/4/8 |
| 12 | Trig Veto | 0 |
| 13 | 0 | 0 |
| 14 | All Lock | All Lock |
| 15 | Lock Err | Lock Err |

---

## `commander.py` — Main Window

> **Full reference:** [`ANLDAQ_commander.md`](ANLDAQ_commander.md) (split 2026-04-23 for size).

Key facts:
- Main entry point — launches the full PyQt6 GUI; checks `SYSTEM` env var; auto-sources `EPICS_para.sh` if needed
- Run control: `caput Online_CS_StartStop Start/Stop`; 2 s delay before `StartAcquisition()`; 3 s delay for SoftIOC; 8 s between repeat-mode runs
- Duration timer combo: Infinity / 1 min / 5 min / 30 min / 1 hr / 2 hr / repeat variants
- SoftIOC auto-spawn: `caget Online_CS_StartStop` ×3; spawns softIOC in `gnome-terminal` if unreachable
- Terminal server: `telnet <TERMINAL_SERVER> <2000+N>` (IOC-1–6 → server 1; IOC-7+ → server 2)
- Script runner: reads `scripts/enableScriptList.txt`; `.py` → `python3`, others → `bash`
- Run timestamp CSV: `<expFolder>/RunTimestamp.csv`, format: `Timestamp, RunID, Event, Comment`

---

## `gui_GS.py` — Per-Detector GS Window

_Source: `ANLDAQ/gui/gui_GS.py` (209 lines). Code-read 2026-04-26._

**`GSWindow`** — `QMainWindow` providing a per-detector control/status panel for Gammasphere detectors.

### Construction
- Accepts `det_id` (integer, e.g. 1–110), `available_det_ids` (list of valid detector IDs), and `cb_pv` (list of `class_PV` objects from commander).
- Lays out a **ComboBox** at the top (items labeled `GS001`..`GS110`) for switching between detectors.
- Two `QGroupBox` columns: **Info / Status** (left) and **Control** (right), each using `QGridLayout`.
- `QTimer` fires every **1000 ms** to refresh all PV widgets (`_UpdateWidgets`). ✅ verified 2026-04-26 — `gui_GS.py:L62-64`

### `_BuildContent(det_id)`
Builds both columns for the currently selected detector using PV name prefix `GS{det_id:03d}`.

**Info / Status column (read/write integers):**
| Label | PV suffix |
|-------|----------|
| Ge ID | `_Ge_ID` |
| SlopeBox ID | `_SlopeBox_ID` |
| Ge Prefix | `_Ge_Prefix` |
| VME Index | `_VME_Index` |
| Dig Index | `_Dig_Index` |
| Dig Channel | `_Dig_Channel` |

All rendered as `RLineEdit` with `decimalPlaces=0` (integer display). ✅ verified 2026-04-26 — `gui_GS.py:L121-126,L132-133`

**Info / Status column (read-only enum/status):**
| Label | PV suffix |
|-------|----------|
| Ge HV | `_SlopeBoxGe_HV_On` |
| Temp | `_SlopeBoxTempHigh` |
| BGO HV | `_SlopeBoxBGO_HV_On` |
| BGO Interlock | `_SlopeBoxBGOInterlock` |

All rendered as `RComboBox` with `SetReadONLY(True)`. ✅ verified 2026-04-26 — `gui_GS.py:L139-142,L147,L149`

**Info / Status column (read-only voltages):**
| Label | PV suffix |
|-------|----------|
| BGO 400V | `_Conv_BGO400` |
| BGO 450V | `_Conv_BGO450` |
| 24V | `_Conv_24V` |
| +12V | `_Conv_plus12V` |
| -12V | `_Conv_minus12V` |
| 5V | `_Conv_5V` |

All rendered as `RLineEdit` with `decimalPlaces=2` (float display), read-only. ✅ verified 2026-04-26 — `gui_GS.py:L155-160,L165,L167-168`

**Control column (read/write enums):**
| Label | PV suffix |
|-------|----------|
| Scan Control | `_Slopebox_Scan_control` |
| Ge HV Ctrl | `_GE_HV_CTRL` |
| BGO HV Ctrl | `_BGO_HV_CTRL` |

All rendered as `RComboBox` (writable). ✅ verified 2026-04-26 — `gui_GS.py:L177-179,L185`

### Detector Switching
- `SwitchTo(det_id)` → updates combo → triggers `_OnDetChanged` → calls `_BuildContent`. ✅ verified 2026-04-26 — `gui_GS.py:L59-60,L72-93`
- On switch: removes existing PV callbacks (`pv.RemoveCallback()`), clears lists, rebuilds content, re-subscribes callbacks. ✅ verified 2026-04-26 — `gui_GS.py:L84-97`
- `_FindPV(pv_name)` scans `cb_pv` list by name; returns `None` gracefully if PV not found (widget still renders, just with no live data). ✅ verified 2026-04-26 — `gui_GS.py:L66-70`

### Lifecycle
- **`showEvent`:** subscribes PV callbacks on first show (`_subscribed` flag prevents double-subscribe). ✅ verified 2026-04-26 — `gui_GS.py:L190-195`
- **`closeEvent`:** stops timer, removes all PV callbacks. ✅ verified 2026-04-26 — `gui_GS.py:L197-203`

---

## `gui_RAM.py` — RAM Visualizer Window

_Source: `ANLDAQ/gui/gui_RAM.py` (30 lines). Code-read 2026-04-27._ ✅ verified 2026-04-27 — line count confirmed

- `RAMWindow(ram_name, pvList)` — simple `QMainWindow` displaying a 32×32 two-state button grid. ✅ verified 2026-04-27 — `gui_RAM.py:L7`
- Uses `RMapTwoStateButton(pvList, rows=32, cols=32)` to visualize VETO_RAM / TRIG_RAM / SWEEP_RAM contents. ✅ verified 2026-04-27 — `gui_RAM.py:L21`
- `QTimer` fires every 500 ms, calling `mapTable.UpdatePV()` to refresh the grid. ✅ verified 2026-04-27 — `gui_RAM.py:L24-28` (`timer.start(500)`, `OnTimer` → `mapTable.UpdatePV()`)
- Opened from `BoardPVWindow.OnRamChanged()` when a RAM type is selected in the board window's dropdown.

---

## `gui_Det.py` — Detector / CollectorBox Monitor Window

_Source: `ANLDAQ/gui/gui_Det.py` (358 lines). Code-read 2026-04-27._

### Detector Geometry Constants

Defined at module level (importable standalone): ✅ verified 2026-04-27 — `gui_Det.py:L25-37`

| Constant | Value | Description |
|----------|-------|-------------|
| `DET_NE` | `range(1, 60, 2)` → [1,3,…,59] | North-East, 30 detectors, CB203 |
| `DET_SE` | `range(2, 61, 2)` → [2,4,…,60] | South-East, 30 detectors, CB201 |
| `DET_NW` | `range(61, 110, 2)` → [61,63,…,109] | North-West, 25 detectors, CB204 |
| `DET_SW` | `range(62, 111, 2)` → [62,64,…,110] | South-West, 25 detectors, CB202 |
| `CB_GROUP` | `{201: ("South-East", DET_SE), …}` | CollectorBox ID → (label, det_list) |

**Total: 110 detectors.** Always 30 slots per CB group; NW/SW pad 5 empty slots at the end. ✅ verified 2026-04-27 — `gui_Det.py:L3-22` (header comment)

### DetWindow Class

- `DetWindow(cb_pv, cb_det_list)` — top-level detector monitor window.
  - `cb_pv`: all CollectorBox PV objects (preloaded).
  - `cb_det_list`: list of `MOD<NNN>` and `GS<NNN>` prefixes for available detectors.
- Layout: top combo (CollectorBox selector, placeholder — `_OnCBSelected` has TODO) + `QTabWidget`.
- **Temperature tab:** 2×2 grid of 4 `QGroupBox` panels (NE/SE/NW/SW), each 30 cells in 3 columns × 10 rows (column-major).
  - Each cell: `MOD<NNN>` ID button + `RLineEdit` (DV_TEMP, 2 dp) + `RLineEdit` (DV_TEMP.HIGH, RW, EPICS `.HIGH` subfield, created manually as `PV()`) + `RTwoStateButton` (DV_EN).
  - ID button click → `_OpenGSWindow(det_id)`. Shift+click → new `GSWindow`; plain click → reuses/reactivates one shared `self._gs_window`.
  - Absent detectors (None slots) render as disabled dummy cells.
- **HV tab:** same 2×2 layout; cells show: ID label + `RLineEdit` (GS<NNN>_GE_HV_DEMAND_VOLTS, 2 dp) + `RLineEdit` (GS<NNN>_Conv_GeHV, 0 dp, read-only) + `RLineEdit` (MOD<NNN>_DS_GEHV, read-only).
- **CA subscription:** lazy — `showEvent` subscribes on first show. `closeEvent` removes callbacks, closes child GSWindows.
- **Timer:** 1000 ms, calls `_UpdateWidgets()` (skips if not visible).

---

## `gui_Board.py` — Generic Board PV Window

_Source: `ANLDAQ/gui/gui_Board.py` (432 lines). Code-read 2026-04-27._

`BoardPVWindow(board_name, board, channelNo=-1)` — generic fallback window for any board type. Used for boards without a specialized window.

### PV Classification

PVs are sorted into special groups by prefix; unmatched PVs fall into a generic 40-row column wrap:

| Group prefix | Widget type | Notes |
|---|---|---|
| `XMAP_`, `YMAP_` | `RMapTwoStateButton` 32×32 | Mapping RAM |
| `DISCRIMINATOR_DELAY` | `RMapLineEdit` | Delay map |
| `LOCK`, `DEN`, `REN`, `SYNC`, `RPwr`, `TPwr`, `SLiL`, `SLoL`, `ILM`, `LINK`, `GATED_THROTTLE`, `RAW_THROTTLE` | `RMapTwoStateButton` row (per type) | Link/Control group; LOCK/ILM/RAW_THROTTLE have inverted color ✅ verified 2026-04-29 — `gui_Board.py:L256-258` (`SetInvertStateColor(True)` for LOCK/ILM/RAW_THROTTLE) |
| `Diag_`, `LOCK_COUNT` | `RMapLineEdit` | Diagnostics (inline with Link group) |
| `FIFOReset` | `RMapTwoStateButton` column | FIFO Reset group |
| `VETO_RAM`, `TRIG_RAM`, `SWEEP_RAM` | `RAMWindow` popup on demand | RAM dropdown |
| `LINK_L/R/U_PROPAGATE` | `RMapTwoStateButton` row | MTRG-specific link propagation |
| `EN_NIM/RAM/REMTRIG/SOFTWARE/THROTTLE_VETO_` | `RMapTwoStateButton` row | Veto enables |
| `reg_MISC_STAT` | `RRegisterDisplay(pv, False)` | MTRG status register |
| `reg_MISC_STAT_REG` | `RRegisterDisplay(pv, True)` | RTR status register |

- **Channel selector:** if `board.NumChannels > 0` and `channelNo < 0`, combo opens child `BoardPVWindow` for a specific channel.
- **RAM selector:** opens `RAMWindow` popup; reuses existing window if open.
- **PV classification prefix lists verified** ✅ 2026-04-29 — `gui_Board.py:L55-83`: `map_pv=[XMAP_/YMAP_/DISCRIMINATOR_DELAY]`, `link_pv` (12 entries), `diag_pv=[Diag_/LOCK_COUNT]`, `fifoReset_pv=[FIFOReset]`, `ram_pv=[VETO_RAM/TRIG_RAM/SWEEP_RAM]`, `mtrg_link_pv=[LINK_L/R/U_PROPAGATE]`, `veto_pv` (5 entries EN_*_VETO_)
- **`closeEvent`:** hides (not destroys) window — keeps PV subscriptions alive. ✅ verified 2026-04-27 — `gui_Board.py:L354-357` (`hide()` + `event.ignore()`)
- **`UpdatePVs`:** 500 ms timer; only runs when `isActiveWindow() and isVisible()`. ✅ verified 2026-04-27 — `gui_Board.py:L347,L359-361`

---

## `gui_DIG.py` — Digitizer Board Window

_Source: `ANLDAQ/gui/gui_DIG.py` (370 lines). Code-read 2026-04-27._

`DIGWindow(board_name, board)` — specialized window for digitizer (DIG) boards.

### Layout (9 panel groups in a 5×3 grid)

| Panel | Grid pos | Contents |
|-------|----------|----------|
| Board Info/Status | [0,0] ×2r | 22 PVs: code_revision, code_date, VME rev, serial_num, timestamps, geo_addr, board_id, fw_type, LED state, power/volt, 3× temp sensors, misc logic status, SD config, VME gp ctrl, ext disc src/mode, TS error count |
| SerDes Status/Control | [0,1] ×2r | 10 PVs: serdes_lock, sm_locked, sm_lost_lock_flag, sd_rx/tx_pwr, sd_local/line_loopback_en, sd_pem, sd_sync, sd_sm_stringent_lock |
| FIFO Status/Control | [0,2] | 12 PVs: master_fifo_reset, fifo_a/b_empty, fifo_a/b/fulla/fullb/almost_full, ini_fifo_prog_flag, fifo_depth (hex), int_FIFO_PROG_ERR/FLG |
| Throttle Control | [1,2] | 4 PVs: rj45_throttle_mode, lfsr_rate_sel, FIFO_Prog_Thresh, lfsr_seed |
| Channel Triggers/Controls | [2,0] ×3r | 10-channel grid: enable/threshold/disc_count/ahit_count/downsample; "All Ch." threshold setter; counter_mode combo; "Open Channel" button |
| Board Control | [2,1] ×3r | 14 PVs: master_logic_enable, CS_Ena, trigger_mux_select, cfd_mode, win_comp_min/max, veto_enable, clk_select, sd_sm_lost_lock_flag_rst, ext_disc_ts_sel, reg_downsample_holdoff, diag_mux_control, DIAG_WAVE_SEL, EXT_DISC_REQ |
| ADC Status | [2,2] | 5 PVs: adc_ph_shift_overflow, adc_dcm_clock_stopped/reset/lock/ctrl_status |
| Acquisition Status | [3,2] | 5 PVs: acq_ph_shift_overflow, acq_dcm_clock_stopped/reset/lock/ctrl_status |
| Phase Status | [4,2] | 5 PVs: ph_checking, ph_hunting_down/up, ph_failure, ph_success |

- **`SetAllChThreshold`:** sets `led_threshold` for all 10 channels.
- **`OpenChannelWindow`:** creates/reuses one `CHWindow` (all-channels view).
- **`closeEvent`:** stops timer, closes `CHWindow`, calls `board.UnsubscribeChannels()`. ✅ verified 2026-04-27 — `gui_DIG.py:L296-301`
- **Forced update mode:** when `isACQRunning=True` (set externally by commander), `disc_count`/`ahit_count`/`led_threshold`/`channel_enable` are force-refreshed every 500 ms tick. ✅ verified 2026-04-27 — `gui_DIG.py:L364-370`

---

## `gui_CH.py` — Channel Settings Window

_Source: `ANLDAQ/gui/gui_CH.py` (396 lines). Code-read 2026-04-27._

Three classes plus `CHWindow`:

### `ChTabTemplate` (base)
- Holds `board`, `pvWidgetList`, `QTimer`.
- `FindPV` / `FindChannelPV(ch, pv_name)` helpers.
- `UpdatePVs(forced)` — delegates to all widget `UpdatePV(forced)` calls.

### `ChannelTab` — single-channel detail view
- Channel selector combo (0–9); on change, rewires all `pvWidget.pv` pointers to new channel.
- **General Settings:** channel_enable, trigger_polarity, pileup_mode, cfd_esum_mode, CFD_fraction, preamp_reset_delay_en/delay, downsample_factor, enable_dec_pause, trig_ts_mode, Early_pre_m_sel, MultiplexWordSelect, reg_channel_control (hex).
- **Ext. Discr.:** ext_disc_sel, ext_disc_src.
- **Window Settings:** led_threshold, k0/k/d/d3/m/p1/p2_window, raw_data_delay/length + `RSetButton` for `load_delays`.
- **Status:** disc_count, ahit_count, accepted_event_count, dropped_event_count, counter_reset + mode selectors.
- Timer: 500 ms.

### `SettingsTabTemplate` — all-channels grid view
- Rows = channels, columns = PV parameters; wrapped in `QScrollArea`.
- Bottom "All" row sets a value for all channels via `_setAllChannels(pvName, value)`.
- `forceUpdate=True` → always force-refreshes (used for Status tab).
- Timer: 500 ms.

### `CHWindow` — top-level channel window

| Tab | Class | PV groups |
|-----|-------|----------|
| Channel | `ChannelTab` | Per-channel single detail view |
| Window Settings | `SettingsTabTemplate` | led_threshold, k0/k/d/d3/m/p1/p2_window, raw_data_delay/length |
| General Settings | `SettingsTabTemplate` | channel_enable, polarity, pileup, CFD, preamp_reset, downsample, dec_pause, trig_ts_mode, Early_pre_m_sel, MultiplexWordSelect, reg_channel_control |
| Ext. Discr. | `SettingsTabTemplate` | ext_disc_sel, ext_disc_src |
| Status | `SettingsTabTemplate` (forceUpdate=True) | channel_enable, disc_count, ahit_count, accepted_event_count, dropped_event_count, counter_reset |

- Tab switch triggers `UpdatePVs(forced=True)` to immediately refresh new tab.
- `closeEvent` stops all tab timers.

---

## `gui_LinkSys.py` — Link System Configuration Window

_Source: `ANLDAQ/gui/gui_LinkSys.py` (295 lines). Code-read 2026-04-27._

`LinkSysWindow(MTRG, RTR_list, DIG_list)` — GUI front-end for the `LinkSys` 5-stage link initialization procedure.

### Layout

- **MTRG Link Map:** 11-row table (links A/B/C/D/E/F/G/H/L/R/U). Each row: link ID + Type combo (RTR names + MASKED/PIXIE/DFMA/DUB/DXA) + Propagate (0/1; only L/R/U enabled).
- **RTR Link Map:** checkbox matrix — rows = RTRs, columns = 11 link IDs. Checked = RTR participates on that link.
- **Settings:** Error Checks checkbox + MTRG Clock Source (local/external) + DIG Clock (0: AUX / 1: SERDES / 2: Oscillator / 3: SERDES). ✅ verified 2026-04-27 — `gui_LinkSys.py:L175`
- **Status label + Run/Cancel buttons.**

### `LinkSysWorker(QThread)`
- Background thread running `link_sys.LinkSys` Stage1–Stage5.
- Emits `stageUpdate(str)` after each stage; `finished(bool, str)` on done/error.
- Catches `AttributeError`, `IndexError`, `ValueError`, `RuntimeError`.

### Config Persistence (`gui/linkMap.json`)
- Saved on every "Run" click (MTRG map, RTR map, settings).
- Loaded on window init; silently ignores missing file or JSON errors.

### Workflow
1. Set MTRG link map (type + propagate per link) and RTR link map (checkbox per RTR×link).
2. Click "Run LinkSys" → config saved → `LinkSysWorker` runs Stage1–5.
3. Status label shows progress; turns green (success) or red (failure).

---

## `gui_MTRG.py` — Master Trigger Board Window

_Source: `ANLDAQ/gui/gui_MTRG.py` (1386 lines). Code-read 2026-04-27._

**Purpose:** PyQt6 window for monitoring and controlling a single MTRG (Master Trigger) board. Opened per-board from `commander.py`. Uses a 500 ms QTimer for PV refresh. Inherits base tab infrastructure from the `templateTab` base class defined at the top of this same file.

### `templateTab` (base class, L15)
Shared base for all MTRG/RTR tab widgets:
- Holds `board`, `pvWidgetList`, and a 500 ms `QTimer` (not auto-started — each tab is started/stopped by the parent window's `showEvent`/`closeEvent`). ✅ verified 2026-04-27 — `gui_MTRG.py:L15-33` (no `start()` in `__init__`; `start(500)` called in `showEvent` L1364-1366)
- `FindPV(name)` → delegates to `board.FindPV(name)`. ✅ verified 2026-04-27 — `gui_MTRG.py:L26-27`
- `UpdatePVs(forced=False)` — iterates `pvWidgetList`; skips if tab not visible. ✅ verified 2026-04-27 — `gui_MTRG.py:L29-33`

### `MTRGWindow` (L1160)
Top-level `QMainWindow`. Header (always visible) shows:
- **Board Info / Status** group: `RRegisterDisplay` for `reg_MISC_STAT` (bit-field decoder, flag=`False`); code revision/date/timestamps A/B/C (hex); `ClkSrc` toggle; `IMP_SYNC` toggle; `CS_Ena` readout toggle; `FifoNum` combo. ✅ verified 2026-04-27 — `gui_MTRG.py:L1224`
- **Trigger Rate Counters** group: 8× raw rate (HIGH+LOW pairs, `reg_RAW_TRIG_RATE_COUNTER_N_HIGH/LOW`), 8× accepted rate (`reg_TRIG_RATE_COUNTER_N_HIGH/LOW`), `Trigger_rate_counter_mode` toggle, `CLEAR_RATE_COUNTERS` set-button; diagnostic counters (8 `reg_Diagnostic*` PVs: Man/Aux, SumX, SumY, SumXY, CPLD, LinkL Locks, NIM1, NIM2), `CLEAR_DIAG_COUNTERS` set-button.

**5-tab QTabWidget** (tab switch forces `UpdatePVs(True)`):

| Tab | Class | Contents |
|-----|-------|----------|
| Trigger/Veto Control | `triggerControlTab` (L36) | 8 trigger algorithms (EN_MAN_AUX/EN_SUM_X/Y/XY/EN_ALGO5/EN_LINK_L/R/EN_MYRIAD_LINK_U) each with NIM/throttle/RAM veto enables per-algorithm, prescale enable+factor; global veto controls (ENBL_NIM_VETO/ENBL_THROTTLE_VETO/EN_RAM_VETO); software veto; Mon 7 veto; EN_NIM_AUX/EN_TRIG_RAM_AUX toggles; X/Y-threshold fields; ALGO_5_SELECT and LINK_U_IS_TRIGGER_TYPE, MYR_TRIGGER_TYPE_SELECT controls |
| Wheel RAM | `wheelRAMTab` (L278) | AUX I/O direction (A/B nibble-pair direction bits, SSI serial vs. parallel mode, SSI gear-ratio and offset); target-wheel encoder controls; opens embedded `RAMWindow` widget showing Trigger RAM lookup table |
| LINK Control | `linkControlTab` (L484) | All SERDES links A–H + L/R/U: LOCK/DEN/REN/SYNC/RPwr/TPwr/SLiL/SLoL/ILM/XLM/YLM per-link map (RMapTwoStateButton, inverted color for ILM/LOCK/XLM/YLM); LRU sub-group (Drv/Rec/Sync enable per L/R/U link, LinkL_DCbal); loopback registers (reg_SERDES_LOCAL_LE/LINE_LE); LVDS pre-emphasis groups ABCD/EFG/HLRU (PrE_0/1/2 toggle + PEABCD/PEEFG/PEHLRU combo); LOCK_RETRY/LOCK_ACK/RESET_LINK_INIT/STRINGENT_LOCK/SM_LOST_LOCK_RESET controls |
| Trigger/CPLD map | `CPLDControlTab` (L756) | Remote Master Logic L/R/U: remote TS offset, remote dig offset, local coincidence mask (binary display), local trig delay; CPLD trigger type map |
| Other Control | `otherControlTab` (L974) | NIM 1/2 output source (combo + sub-select, enable delay + delay count); NIM throttle select; discriminator bit delay enable; overlap delay and assertion delay (20 ns units); ENABLE_VETO toggle; throttle filter time + time range combo; minimum throttle width to MTRG trigger |

**Timer lifecycle:** 500 ms timer started in `showEvent` for MTRGWindow and all 5 tabs; stopped in `closeEvent` for all. ✅ verified 2026-04-27 — `gui_MTRG.py:L1362-1372`

---

## `gui_RTR.py` — Router Trigger Board Window

_Source: `ANLDAQ/gui/gui_RTR.py` (542 lines). Code-read 2026-04-27._

**Purpose:** PyQt6 window for monitoring and controlling a single RTRG (Router Trigger) board. Reuses the `templateTab` base class imported from `gui_MTRG`. Opened per-board from `commander.py`. 500 ms QTimer refresh.

### `RTRWindow` (L265)
Top-level `QMainWindow`. Header (always visible):
- **Board Info / Status** group: `RRegisterDisplay` for `reg_MISC_STAT_REG` (bit-field decoder, flag=`True` = RTRG variant); code revision/date/timestamps A/B/C (hex); `ClkSrc` toggle. ✅ verified 2026-04-27 — `gui_RTR.py:L310`
- **LED Controls** sub-group: `LEDControl` combo; LED4–LED12 individual toggles (9 LEDs).
- **Diagnostic Counters** group: 8 `reg_Diagnostic*` PVs (Type0–4 triggers, Router Lock Count, Link L S/D Lock, Throttle Count); `DIAG_THROTTLE_TYPE` combo; `CLEAR_DIAG_COUNTERS` set-button.
- **Other Controls** group: NIM 1/2 source combos; NIM throttle select (`NIM_THROTTLE_SELECT`); discriminator delay enable (`ENBL_DISCBIT_DELAY`); overlap delay (20 ns units, `OVERLAP_DELAY`); assertion delay (`ASSERTION_DELAY`); enable veto toggle; throttle filter time; throttle time range combo; minimum throttle width to MTRG trig (`THROTTLE_WIDTH`).

**2-tab QTabWidget:**

| Tab | Class | Contents |
|-----|-------|----------|
| LINK Control | `rtrlinkControlTab` (L13) | SerDes links: LOCK/DEN/REN/SYNC/RPwr/TPwr/SLiL/SLoL/ILM/LINK/GATED_THROTTLE/RAW_THROTTLE per-link map (inverted color for ILM/LOCK/XLM/YLM); LRU sub-group (LOCK_RETRY, LOCK_ACK, RESET_LINK_INIT, STRINGENT_LOCK, SM_LOST_LOCK_RESET, LinkL_DCbal); loopback registers (reg_SERDES_LOCAL_LE/LINE_LE); LVDS pre-emphasis ABCD/EFG/HLRU (PrE_0/1/2 + PEABCD/PEEFG/PEHLRU) |
| X/Y Map | `rtrXYMapTab` (L221) | XMAP_* and YMAP_* per-link bit maps (RMapTwoStateButton); DISCRIMINATOR_DELAY per-link map (RMapLineEdit); X_SELECT and Y_SELECT combos for global X/Y source |

**Timer lifecycle:** 500 ms timer for RTRWindow + both tabs, started/stopped in showEvent/closeEvent. ✅ verified 2026-04-27 — `gui_RTR.py:L513-523`

---

## `aux.py` — Utility Helpers

_Source: `ANLDAQ/gui/aux.py` (7 lines). Code-read 2026-04-27._

Two utility functions shared by `gui_RTR.py` and `gui_MTRG.py`:
- **`natural_key(s)`** — natural sort key (splits string on digit runs; sorts e.g. LINK2 before LINK10). ✅ verified 2026-04-27 — `aux.py:L4`
- **`make_pattern_list(prefix_list)`** — given a list of PV name prefixes, returns compiled `re.Pattern` objects matching `^<prefix>_[A-Za-z]$` (single letter suffix). Used to sort per-link PVs (e.g. LOCK_A/LOCK_B…) into ordered lists for `RMapTwoStateButton` layout. ✅ verified 2026-04-27 — `aux.py:L7` (exact regex: `rf'^{prefix}_[A-Za-z]$'`)

---

## `gui_SYS.py` — System Status Tab Library

_Source: `ANLDAQ/gui/gui_SYS.py` (589 lines). Code-read 2026-04-27._

Provides five `QWidget` tab classes instantiated by `commander.py` into the commander window's bottom `QTabWidget`. There is **no standalone window class** — all tabs inherit `sysTemplateTab` and are embedded directly in commander.

### `sysTemplateTab` — Base Class (L14–44)

- Constructor accepts `MTRG : Board`, `RTR_list`, `DIG_list`, `DAQ_list` (any can be `None`).
- `pvWidgetList` — flat list of all PV-bound widgets; iterated by `UpdatePVs()`.
- `UpdatePVs(forced=False)` — skips update when tab is not visible (`self.isVisible()`).
- `FindPV(pv_name, board, isDAQ=False)` — searches `board.Board_PV[]`. DAQ boards: strips leading segment before `_`; others: matches last `:\u2026` suffix.

### `sysTimestampReadOutTab` — Tab: "Timestamp" (L47–173)

Commander's first tab. Two group boxes side by side:

**Timestamp group:** 48-bit timestamps (3 × hex `RLineEdit`) + clock source (`RTwoStateButton`) for MTRG, every RTR, every DIG.
- MTRG PVs: `reg_TIMESTAMP_A/B/C`, `ClkSrc`
- RTR PVs: `reg_TIMESTAMP_A/B/C`, `ClkSrc`
- DIG PVs: `live_timestamp_msb`, `live_timestamp_lsb`, `clk_select`
- Top-right: `IMP_SYNC` toggle (`RTwoStateButton`).

**Readout group:** `CS_Ena` toggle for MTRG and each DIG. MTRG FIFO mode via `FifoNum` (`RComboBox`). Timer: 500 ms.

### `sysLinktab` — Tab: "Link Status" (L175–311)

Four group boxes:

1. **Link Status** — `RRegisterDisplay` for `reg_MISC_STAT` (MTRG, `isRTR=False`) and `reg_MISC_STAT_REG` (each RTR, `isRTR=True`).
2. **Link Lock Status** — `RMapTwoStateButton` grid (1 row × 11 cols, inverted colors). PVs: `LOCK_A/B/C/D/E/F/G/H/L/R/U` for MTRG and each RTR.
3. **Input Link Mask** — Same layout with `ILM_A/B/C/D/E/F/G/H/L/R/U`. Inverted colors.
4. **Link L Control** — Per-board `RTwoStateButton` widgets.
   - MTRG: `LOCK_RETRY`, `LOCK_ACK`, `RESET_LINK_INIT`, `LINK_L_STRINGENT`, `LINK_R_STRINGENT`, `LINK_U_STRINGENT` (no Reset Lock Lost PV).
   - RTR: `LOCK_RETRY`, `LOCK_ACK`, `RESET_LINK_INIT`, `STRINGENT_LOCK`, `SM_LOST_LOCK_RESET`.

Timer: 500 ms.

### `sysTCPTab` — Tab: "TCP Transfer" (L315–352)

One group box: per-DAQ-IOC row showing `CV_BuffersAvail`, `CV_NumSendBuffers`, `CV_SendRate` (found via `FindPV(..., isDAQ=True)`). Timer: 500 ms.

### `sysCodeRevisionTab` — Tab: "Code Revision" (L355–418)

Scrollable grid: code revision + date in hex for all boards.
- MTRG: `reg_CODE_REVISION`, `reg_CODE_DATE`
- RTR: `Code_Revision`, `CODE_DATE`
- DIG: `regin_code_revision`, `code_date`

Timer: 500 ms.

### `globalSettingTab` — Tab: "Global Settings" (L421–589)

**DGS-only** — body is empty unless `$SYSTEM=DGS`.

**Channel Parameters group** — 20-row × 4-col write-only grid (columns: Ge Center/BGO/Ge Side/Aux). Submitting calls `SetDetTypePV(pv_name, det_type, value)`:
- `GeC`/`BGO` → all `MDIG*` boards, channels 5–9 / 0–4
- `GeS`/`Aux` → all `SDIG*` boards, channels 5–9 / 0–4

The 20 per-channel PVs: `channel_enable`, `trigger_polarity`, `k0_window`, `k_window`, `d_window`, `m_window`, `d3_window`, `led_threshold`, `raw_data_length`, `raw_data_delay`, `p1_window`, `p2_window`, `preamp_reset_delay`, `CFD_fraction`, `disc_width`, `coarse_disc_thresh`, `pileup_mode`, `preamp_reset_delay_en`, `P2_mode`, `downsample_factor`

**Digitizer Settings group** — 15-row write-only grid applied to all DIG boards via `SetAllBoardsPV()`. PVs: `cfd_mode`, `trigger_mux_select`, `win_comp_min`, `win_comp_max`, `peak_sensitivity`, `holdoff_time`, `stop_ho_at_peak` (yellow), `counter_mode`, `diag_mux_control`, `DIAG_WAVE_SEL`, `rj45_throttle_mode`, `FIFO_Prog_Thresh`, `lfsr_rate_sel`, `lfsr_seed`, `load_delays` (green).

Warning label on both groups: "Write-only: values here are for setting all channels of a detector type at once. They do not reflect the actual current values of individual channels."

### Commander Integration (commander.py L305–317)

Tabs instantiated once at startup, added in order: Timestamp → Link Status → TCP Transfer → Code Revision → Global Settings. `tabWidget.currentChanged` forces `UpdatePVs(True)` on tab switch. Timers stopped in `closeEvent`.

---

## `Guceiver/` — GUI Live Receiver & Online Monitor

_Source: `ANLDAQ/gui/Guceiver/` (7 files, ~2,236 lines). Code-read 2026-04-27._

**Purpose:** PyQt6 + Matplotlib GUI that connects directly to one IOC's TCP server (port 9001) and displays live waveforms, energy spectra, raw event data, and TAC-II timing. **Online monitor / debugging tool** — not the production file-writing receiver ([`tcpReceiverMT`](ANLDAQ_tcpReceiver.md)). See also: [`guceiver.md`](guceiver.md) for the standalone Guceiver reference.

### `Guceiver.py` — Main Window (331 lines)

**Board list:** Reads `user_package_data` PV from each DIG board via `epics.caget` to build `board_id_list = [(display_text, board_id, bd_name), ...]`.

**IOC selection:** `QComboBox` populated from `$IOC_IP` env var (space-separated IPs). Each entry: `"IOC-N  ip:9001"` with IP as `currentData()`.

**Status bar** (500 ms `QTimer`): Run Time, Total Bytes Received, Data Rate [Byte/s], Total events.

**Four tabs** (one plot timer active at a time — switched by `on_tab_changed`):
1. **Waveform** — `WaveformTab`: live ADC trace; `fillWaveformArray=True`
2. **Spectrum** — `SpectrumTab`: energy histogram; `fillEnergyArray=True`
3. **Data** — `dataTab`: raw event field table; `fillDataArray=True`
4. **TAC-II** — `tacTab`: TAC timing display; `fillTACArray=True` (see [`tac2.md`](tac2.md) for TAC-II hardware details)

**Start/Stop:**
- Start: `caput Online_CS_SaveData Save` + `caput Online_CS_StartStop Start` → connect TCP → start active tab's plot timer
- Stop: `caput Online_CS_StartStop Stop` + `caput Online_CS_SaveData No Save` → stop all timers → close socket
- Parent commander's `btn_startRun` disabled during monitoring; re-enabled on stop.
- End-of-run: channel_id `0xD` in non-7/8 header → `daq_stopped` signal → `stop_receiver()`

**Thread model:** `Receiver` lives in `QThread` (`data_thread`). Communication via flags + `QMutex` (`data_mutex`) only.

### `class_Receiver.py` — TCP Receiver (265 lines)

**Wire protocol** (same as `psNet.h`):
- Sends `struct.pack(">I", 1)` per poll
- Receives 16-byte reply: `(reply_type, record_size, status, num_record)` big-endian
- Receives `record_size × num_record` bytes; converts to `uint32` list

**Decode loop:**
- `0xAAAAAAAA` → DIG (`isDIG=True`); `0x0000AAAA` → TAC-II (`isDIG=False`)
- DIG: `payloadMaxIndex` from word[1] bits 26:16; `header_type` from word[3] bits 19:16. Types 7/8 only. On completion: `dig.decode_data(payload, decodeWaveForm=True)`; append to array if dig_index + channel_index match.
- TAC-II: fixed 16-word; `tac.decode(payload)`; append to `TACArray`
- Non-7/8 header: `channel_id==0xD` → stop; else drop.
- Ring-buffers: `waveformArray`/`dataArray`/`TACArray` max 100; `energyArray` unbounded (spectrum).

**Energy:** `(POST_RISE_ENERGY - PRE_RISE_ENERGY) / M_windows`

**Socket:** SOCK_STREAM TCP, 2 s connect timeout, 5 s recv timeout. Retry loop with `QApplication.processEvents()`.

### `class_DIG.py` — Python DIG Decoder

Python reimplementation of `Aux/class_DIG.h`. Fields: `USER_DEF` (board index), `CH_ID`, `PRE_RISE_ENERGY`, `POST_RISE_ENERGY`, `waveform[]` (14-bit ADC samples). See [`data_structures.md`](data_structures.md) for the canonical DIG event packet layout and [`ANLDAQ_tcpReceiver_Aux.md`](ANLDAQ_tcpReceiver_Aux.md) for the C++ equivalent.

### `class_TAC.py` — Python TAC-II Decoder (110 lines)

Python reimplementation of `Aux/class_TDC.h`. Decodes 16-word TAC-II packets. See [`tac2.md`](tac2.md) for TAC-II hardware and firmware context.

### Display Tab Classes

`class_waveTab.py` (308L), `class_spectrumTab.py`, `class_dataTab.py`, `class_tacTab.py` (233L) — each embeds Matplotlib `FigureCanvas`. Common pattern:
- `plot_timer` (500 ms) → `update_plot()`
- `combo_dig_index` / `spinBox_channel_index` — board/channel filter
- `pause_update_button` — freeze display without stopping acquisition
- Reads from receiver arrays under `receiver.data_mutex`

`SpectrumTab` additionally has `spinbox_M_windows` (M-window divisor for energy).

### Notes

- Launched from commander as a child window; `parent()` back-reference used for button coordination.
- **Does not write data to disk** — online monitoring only.
- Port 9001 hardcoded.

_Source: `ANLDAQ/gui/Guceiver/` (code-read 2026-04-27)_

---

## Cross-References

| File | Relationship |
|------|--------------|
| [`ANLDAQ.md`](ANLDAQ.md) | Parent ANLDAQ overview; this file split from it |
| [`ANLDAQ_GUI_windows.md`](ANLDAQ_GUI_windows.md) | Per-window GUI documentation (uses classes defined here) |
| [`ANLDAQ_gui_sys.md`](ANLDAQ_gui_sys.md) | gui_SYS.py window detail |
| [`ANLDAQ_commander.md`](ANLDAQ_commander.md) | Commander run control GUI (top-level window) |
| [`link_sys_analysis.md`](link_sys_analysis.md) | link_sys.py LinkSys internals (called by gui_LinkSys.py) |
| [`DIG_firmware_expert.md`](DIG_firmware_expert.md) | Digitizer firmware internals — channel FIFO, throttle, ADC details |
| [`tac2.md`](tac2.md) | TAC-II hardware and firmware (decoded by class_TAC.py) |
| [`guceiver.md`](guceiver.md) | Standalone Guceiver reference |
| [`data_structures.md`](data_structures.md) | DIG event packet format (decoded by class_DIG.py) |
| [`ANLDAQ_tcpReceiver.md`](ANLDAQ_tcpReceiver.md) | tcpReceiverMT — production data-writing receiver |
| [`collectorbox_devicesupport.md`](collectorbox_devicesupport.md) | CollectorBox EPICS device support (PV sources for gui_GS/gui_Det) |

---

*Created: 2026-04-25 | Last reviewed: 2026-04-27 (gui_SYS.py + Guceiver/ added)*
