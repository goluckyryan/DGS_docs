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
_Source: `DGS_tools_pack/ANLDAQ/collectorBox/rsyncDB.sh` (15 lines). Code-read 2026-04-23._ ✅ verified 2026-04-23

One-shot sync script to download the live CollectorBox DB and cmd files from the running IOC host:
- Requires `ANLDAQ_DIR` environment variable (set by `EPICS_para.sh`); exits with error if absent ✅ verified 2026-04-23 — `rsyncDB.sh:L3-6`
- Target host: **192.168.203.42** (CollectorBox IOC host); user: `dgs`; key: `~/.ssh/id_ed25519` ✅ verified 2026-04-23 — `rsyncDB.sh:L11-14`
- Downloads: `db/*.db` and `iocBoot/iocCollectorApp/*.cmd` from `/shared/EPICS/CollectorBox_RevA/` on the remote ✅ verified 2026-04-23 — `rsyncDB.sh:L13-14`
- Run after deploying a new CollectorBox firmware/IOC to refresh the local templates before re-running `findAllPV.py`

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
3. Auto-compiles `tcpReceiverMT` via `make tcpReceiverMT` in `tcpReceiver/` — skips if already up to date
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

*Created: 2026-04-25 (split from ANLDAQ.md)*
