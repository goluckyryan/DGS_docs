# ANLDAQ Commander — DGS Run Control GUI

Stability: C2 - Active / semi-stable

**Source:** `DGS_tools_pack/ANLDAQ/commander.py` (862 lines) ✅ verified 2026-04-23 — `commander.py` (line count confirmed)  
**Date documented:** 2026-04-23  
**See also:** [`ANLDAQ_GUI_windows.md`](ANLDAQ_GUI_windows.md) — board-specific GUI windows; [`ANLDAQ.md`](ANLDAQ.md) — ANLDAQ system overview; [`ANLDAQ_tcpReceiver.md`](ANLDAQ_tcpReceiver.md) — TCP data receiver; [`run_procedures.md`](run_procedures.md) — operator run procedures

---

## Table of Contents

1. [Purpose](#purpose)
2. [Startup & Environment](#startup--environment)
3. [PV & Board Initialization](#pv--board-initialization)
4. [Main Window Layout](#main-window-layout)
5. [Run Control Flow](#run-control-flow)
6. [Acquisition PV Control](#acquisition-pv-control)
7. [Duration & Repeat Mode](#duration--repeat-mode)
8. [Board Sub-Windows](#board-sub-windows)
9. [Other GUI Panels](#other-gui-panels)
10. [SoftIOC Auto-Spawn](#softioc-auto-spawn)
11. [IOC Terminal Access](#ioc-terminal-access)
12. [Settings Persistence](#settings-persistence)
13. [Script Runner](#script-runner)
14. [RunTimestamp Log](#runtimestamp-log)

---

## Purpose

`commander.py` is the **top-level run control GUI** for the DGS system. It serves as the master operator interface for:
- Starting and stopping data acquisition runs
- Configuring and monitoring the IOC TCP receiver
- Opening board-specific control windows (DIG, RTR, MTRG)
- Controlling the SoftIOC and IOC terminal sessions
- Launching utility panels ([scalar monitor](ANLDAQ_gui_sys.md), [link system](link_sys_analysis.md), live trace monitor via [Guceiver](guceiver.md))

Launched by sourcing `EPICS_para.sh` (sets `SYSTEM`, `IOC_IP`, `ANLDAQ_DIR`, `EPICS_HOST_ARCH`, `TERMINAL_SERVER`, etc.) then running `python3 commander.py`. If `SYSTEM` is not set when invoked directly, the script re-execs itself under bash after sourcing `EPICS_para.sh`.

---

## Startup & Environment

**Required environment variables** (set by `EPICS_para.sh`):

| Variable | Purpose |
|----------|---------|
| `SYSTEM` | System type: `"DGS"`, `"SlopeBox"`, etc. |
| `IOC_IP` | Space-separated list of IOC host IP addresses |
| `ANLDAQ_DIR` | Root path of the ANLDAQ installation |
| `EPICS_HOST_ARCH` | EPICS host architecture string (e.g. `linux-aarch64`) |
| `TERMINAL_SERVER` | Space-separated telnet terminal server host(s) |

The script **changes its working directory to `script_dir`** (the directory containing `commander.py`) on startup, so all relative paths are anchored there. ✅ verified 2026-04-23 — `commander.py:L5-6` (`script_dir = os.path.dirname(os.path.realpath(__file__)); os.chdir(script_dir)`)

---

## PV & Board Initialization

On startup, `commander.py` builds all board objects from the JSON PV lists:

```python
from json2pv import GeneratePVLists
DIG_CHANNEL_PV, DIG_BOARD_PV, RTR_BOARD_PV, MTRG_BOARD_PV,
DIG_BOARD_LIST, RTR_BOARD_LIST, MTRG_BOARD_LIST, DAQ_PV, DAQ_LIST =
  GeneratePVLists('../ioc/All_PV.json')
```

CollectorBox PVs loaded optionally (DGS-only, fails gracefully) — see [`collectorbox_PVs.md`](collectorbox_PVs.md) for the full PV list:
```python
from cb_json2pv import LoadCollectorBoxPVs
CB_PV, CB_DET_LIST = LoadCollectorBoxPVs('../collectorBox/CollectorBox_PV.json')
```

Board objects (`class_Board.Board`) are instantiated for all DIGs, RTRs, MTRG, and DAQ nodes. `ALLBOARD` is the concatenated list used for the generic board picker.

---

## Main Window Layout

The main window (`MainWindow`, `QMainWindow`, 750×200 px) uses a `QGridLayout` with four major sections stacked vertically: ✅ verified 2026-04-23 — `commander.py:L122` (`self.resize(750, 200)`)

### Data Taking GroupBox
- **Exp Name** — experiment name text field (persisted to `settings.json`) ✅ verified 2026-04-27 — `commander.py:L360,L371` (`s.get("expName", ...)` in LoadSettings; `"expName": self.expName_edit.text()` in SaveSettings)
- **Run ID** — read-only display of the current run counter ✅ verified 2026-04-27 — `commander.py:L167` (`self.lbl_runID.setReadOnly(True)`)
- **Exp Folder** — path to the experiment data directory (Browse button)
- **Start Run** button — green/red toggle; arms the run sequence
- **Duration** combo — `Infinity`, `1 min`, `5 min`, `30 min`, `1 hr`, `2 hr`, `1 hr repeat`, `2 hr repeat` ✅ verified 2026-04-23 — `commander.py:L193-195`
- **Edit IOC Config** button — opens `IOCConfigDialog` to configure TCP receiver IPs/ports

### Acquisition Controls GroupBox
- `Online_CS_StartStop` — two-state EPICS PV button (`Start`/`Stop`)
- `Online_CS_SaveData` — two-state EPICS PV button (`Save`/`No Save`)
- **Live trace/data monitor** — opens the Guceiver live data window

### Board Selection GroupBox
- **Master Trigger Board** button → opens `MTRGWindow`
- **RTR Board** combo → opens `RTRWindow` for the selected RTR
- **DIG Board** combo → opens `DIGWindow` for the selected DIG (lazy CA subscription on first open) ✅ verified 2026-04-27 — `commander.py:L653` (`DIG_List[id].SubscribeChannels()  # lazy: subscribe CA only when window first opens`)

### Others GroupBox
- **Link System** button → opens `LinkSysWindow`
- **Script** combo → runs `.py` or `.sh` scripts from `scripts/` (list from `enableScriptList.txt`)
- **Open Terminal** combo → opens gnome-terminal with telnet to IOC terminal server
- **Scalar** button → opens `ScalarWindow`
- **SBX/CollectorBox** button → opens `DetWindow` (DGS SYSTEM only) ✅ verified 2026-04-27 — `commander.py:L293-295` (`btn_det.setEnabled(os.environ.get("SYSTEM") == "DGS")`); `L101` (`from gui_Det import DetWindow`); `L669-675` (`OpenDetWindow` creates `DetWindow(CB_PV, CB_DET_LIST)`)

### Tab Widget (bottom)
Five system-level tabs refreshed on tab switch + by 500ms `QTimer`: ✅ verified 2026-04-27 — `commander.py:L317` (`currentChanged.connect(lambda _: self.tabWidget.currentWidget().UpdatePVs(True))`); `L350` (`self.timer.start(500)`)
- **Timestamp** — `sysTimestampReadOutTab` (MTRG + all RTR/DIG/DAQ timestamps)
- **Link Status** — `sysLinktab` (MTRG + RTR link health)
- **TCP Transfer** — `sysTCPTab` (DAQ node TCP state)
- **Code Revision** — `sysCodeRevisionTab` (firmware revisions across boards)
- **Global Settings** — `globalSettingTab` (MTRG + RTR + DIG global settings)

### Generic Board Picker
A combo at the bottom allows opening any board's `BoardPVWindow` (raw PV view). Selection auto-resets to index 0 after opening.

---

## Run Control Flow

```
Operator clicks "Start Run"
  → prompt for comment (unless auto-repeat)
  → increment runCounter
  → create RunStatusWindow (opens TCP receiver, monitors run)
  → log "start" to RunTimestamp.csv in expFolder
  → QTimer.singleShot(2000ms) → StartAcquisition()    ✅ verified 2026-04-23 — `commander.py:L458`
      → epics.caput("Online_CS_SaveData", "Save")    ✅ verified 2026-04-23 — `commander.py:L489`
      → epics.caput("Online_CS_StartStop", "Start")   ✅ verified 2026-04-23 — `commander.py:L490`
  → SetupDurationTimer(duration)

Operator clicks "Stop" (or duration expires)
  → RunStatusWindow.StopRun(comment)
      → epics.caput("Online_CS_StartStop", "Stop")    ✅ verified 2026-04-23 — `commander.py:L495`
      → QTimer.singleShot(3000ms) → epics.caput("Online_CS_SaveData", "No Save")  ✅ verified 2026-04-23 — `commander.py:L496`
  → log "stop" to RunTimestamp.csv
  → if repeat mode: QTimer.singleShot(8000ms) → _StartRepeatRun()  ✅ verified 2026-04-23 — `commander.py:L521`
```

**Key sequencing:**
- 2-second delay before `StartAcquisition()` — allows `RunStatusWindow` TCP receiver time to connect
- 3-second delay between `StartStop=Stop` and `SaveData=No Save` — allows last data to flush
- 8-second delay before repeating a run — allows `RunStatusWindow` to close cleanly

---

## Acquisition PV Control

Two global EPICS PVs control the acquisition state machine:

| PV | Values | Meaning |
|----|--------|---------|
| `Online_CS_StartStop` | `Start` / `Stop` | Start or stop the DAQ collection |
| `Online_CS_SaveData` | `Save` / `No Save` | Enable/disable data file writing |

These PVs live in the SoftIOC (`Online_CS_*` namespace). The `RTwoStateButton` widgets poll them at 500ms intervals.

---

## Duration & Repeat Mode

```python
DURATION_MAP = {
  "1 min": 60, "5 min": 300, "30 min": 1800,
  "1 hr": 3600, "2 hr": 7200,
  "1 hr repeat": 3600, "2 hr repeat": 7200,
}
```
✅ verified 2026-04-23 — `commander.py:L463-466`

- **Infinity** — no timer, run until manually stopped
- **Finite durations** — `QTimer.singleShot` fires `OnDurationExpired()` after the set time
- **Repeat modes** — `"1 hr repeat"` / `"2 hr repeat"` stop the run and restart it automatically after 8 seconds. Auto-repeat uses a fixed comment `"auto repeat run for 1 hr"` (no user prompt). ✅ verified 2026-04-23 — `commander.py:L521,L529` (`QTimer.singleShot(8000, self._StartRepeatRun)`, `autoComment = f"auto repeat run for {duration.replace(' repeat', '')}"`)

---

## Board Sub-Windows

Sub-windows are lazy-initialized (created on first open, re-raised if already open):

| Window | Class | Opened by |
|--------|-------|-----------|
| MTRG | `MTRGWindow` | "Master Trigger Board" button |
| RTR (×N) | `RTRWindow` | RTR combo selection |
| DIG (×N) | `DIGWindow` | DIG combo selection (also calls `SubscribeChannels()`) |
| Generic board | `BoardPVWindow` | Generic board picker combo |
| Link System | `LinkSysWindow` | "Link System" button |
| Scalar | `ScalarWindow` | "Scalar" button |
| SBX/CollectorBox | `DetWindow` | "SBX/CollectorBox" button (DGS only) |
| Live monitor | Guceiver `GUI` | "Live trace/data monitor" button |

`DIGWindow` defers CA channel subscription until first open (`DIG_List[id].SubscribeChannels()`).

---

## Other GUI Panels

### Live Trace Monitor (Guceiver)
The [Guceiver](guceiver.md) live data monitor is embedded from `ANLDAQ/gui/Guceiver/`. It is injected with `IOC_IP` entries from the environment so operators don't need to retype IPs. GUI class: `from Guceiver import GUI as GuceiverGUI`.

### Scalar Window
`ScalarWindow(DIG_BOARD_LIST, DIG_List, CB_DET_LIST)` — positioned to the **left** of the main window.

### Link System Window
`LinkSysWindow(MTRG, RTR_List, DIG_List)` — positioned **50px to the right** of the main window.

---

## SoftIOC Auto-Spawn

See [`EPICS_softIOC.md`](EPICS_softIOC.md) for the SoftIOC DB layout and PV namespace. On startup, `CheckACQCanStart()` runs:
1. Attempts `epics.caget("Online_CS_StartStop", timeout=5.0)` up to **3 times** with 2-second retries. ✅ verified 2026-04-25 — `commander.py:L756-762` (`for attempt in range(3)`, `timeout=5.0`, `time.sleep(2)`)
2. If the PV is unreachable, calls `OpenSoftIOC()`.

`OpenSoftIOC()`:
- Checks `ps ax` for an existing SoftIOC process (looks for "SoftIOC" + "bin" + trailing ".cmd"). ✅ verified 2026-04-25 — `commander.py:L768-779` (`ps_out` scan for "SoftIOC", "bin", `first[-3:] == "cmd"` or `".cmd" in first`)
- If not running, spawns a gnome-terminal with:
  ```
  cd $ANLDAQ_DIR/EPICS/softIOC/iocBoot/iocdgsSoftIOC
  ../../bin/$EPICS_HOST_ARCH/dgsSoftIOC dgsSoftIoc.cmd
  ``` ✅ verified 2026-04-25 — `commander.py:L791-799`
- If spawn fails (e.g. no gnome-terminal), the `ACQStartStop` and `ACQSaveData` buttons are disabled. ✅ verified 2026-04-25 — `commander.py:L802-803` (`self.ACQStartStop.setEnabled(False); self.ACQSaveData.setEnabled(False)`)

---

## IOC Terminal Access

"Open Terminal" combo opens gnome-terminal sessions telnet'd to the terminal server:

```
telnet $TERMINAL_SERVER <port>
```

Port formula: `2000 + IOC_id` (e.g., IOC-1 → port 2001). ✅ verified 2026-04-23 — `commander.py:L838` (`port = 2000 + id`)

**DGS system split:** for IOC id ≤ 6, uses `TERMINAL_SERVER.split()[0]` (first server); for id > 6, uses `TERMINAL_SERVER.split()[1]` (second server). This reflects two terminal servers covering the two VME crate racks. ✅ verified 2026-04-23 — `commander.py:L833-836`

**SlopeBox system:** always uses IOC id = 3 regardless of selection. ✅ verified 2026-04-25 — `commander.py:L829-830` (`if system == "SlopeBox": id = 3`)

---

## Settings Persistence

Settings are stored in `ANLDAQ/gui/settings.json`:

```json
{
  "expFolder": "/path/to/data",
  "expName": "Experiment_Name",
  "runCounter": 42,
  "durationIndex": 0
}
```

- Loaded on startup via `LoadSettings()`
- Saved on `Start Run` (to persist the run counter increment immediately)
- Saved on window close via `closeEvent()`
- Also saved when the user edits and presses Enter in `expName_edit` or `expFolder_edit`

---

## Script Runner

The "Others" panel includes a script combo that reads `scripts/enableScriptList.txt` for a list of available scripts (one filename per line, `#` lines skipped). Scripts can be `.py` (run with `python3`) or `.sh` (run with `bash`). Output is piped to the console. Scripts run in a `QProcess` in the `scripts/` directory. ✅ verified 2026-04-25 — `commander.py:L268-274` (`enableScriptList.txt` path, `not name.startswith("#")` filter)

---

## RunTimestamp Log

Every run start and stop is logged to `<expFolder>/RunTimestamp.csv`:

```
Timestamp, RunID, Event, Comment
2026-04-23 10:00:00, 1, start, "first test run"
2026-04-23 10:30:00, 1, stop, "single 30 min run stopped"
```

The file is created with header if it doesn't exist. ✅ verified 2026-04-25 — `commander.py:L402-408` (`needHeader = not os.path.isfile(csvFile)`, writes `"Timestamp, RunID, Event, Comment\n"` on first create) Run comments are entered by the operator at run start (except for auto-repeat runs which use a fixed comment). ✅ verified 2026-04-25 — `commander.py:L445,L501` (start/stop LogRunTimestamp calls)

---

## Cross-References

- [`ANLDAQ.md`](ANLDAQ.md) — ANLDAQ system overview: tcpReceiverMT, IOC sender, production DAQ pipeline
- [`ANLDAQ_GUI_windows.md`](ANLDAQ_GUI_windows.md) — Board-specific GUI windows launched from the commander (DIG, MTRG, RTRG, collector)
- [`ANLDAQ_tcpReceiver.md`](ANLDAQ_tcpReceiver.md) — TCP data receiver (tcpReceiverMT) that commander starts/stops
- [`run_procedures.md`](run_procedures.md) — Operator run procedures: full context for how commander fits into a run
- [`trig_setup_scripts.md`](trig_setup_scripts.md) — Stage 1–5 trigger initialization scripts invoked via the Script Runner
- [`utility_scripts.md`](utility_scripts.md) — Helper scripts listed in `enableScriptList.txt` (basic_settings_LED.py, Serdes_Linkup.sh)
- [`guceiver.md`](guceiver.md) — Guceiver diagnostic GUI: optionally embedded as child widget inside commander
- [`ioc.md`](ioc.md) — IOC configuration; PVs that commander reads/writes originate from VME IOC DB templates
- [`snapshot_pv.md`](snapshot_pv.md) — PV snapshot utilities called at run start; commander triggers dumpPVs.py
- [`EPICS_softIOC.md`](EPICS_softIOC.md) — SoftIOC DB layout and PV namespace for `Online_CS_*` PVs; auto-spawned by commander at startup
- [`collectorbox_PVs.md`](collectorbox_PVs.md) — CollectorBox PV list loaded via `cb_json2pv` at startup (DGS SYSTEM only)

*Source: `DGS_tools_pack/ANLDAQ/commander.py`. Created: 2026-04-23.*
