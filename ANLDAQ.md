# ANLDAQ — Front-End DAQ GUI & Data Receiver

## What It Is

ANLDAQ is the **front-end operator interface** for the DGS (and other ANL detector) DAQ systems. It provides:
- A **PyQt6 GUI** (`commander.py`) for configuring and controlling detector electronics (DIG digitizers, RTRG routers, MTRG master trigger) via EPICS
- A **multi-threaded C++ TCP receiver** (`tcpReceiverMT`) that collects raw binary data from VME IOCs during runs via TCP (port 9001) ✅ verified 2026-04-06 — `tcpReceiverMT.cpp:L55` (`cfg.port = 9001`) + `gui_DataTaking.py:L8,L104` (spawn)

Branches exist for multiple experiments: `master` (SlopeBox/DUO teststand), `DGS`, `X-array`.

---

## Key Files & Roles

| File/Dir | Role |
|----------|------|
| `commander.py` | Main entry point — launches the full GUI |
| `EPICS_para.sh` | Sets all EPICS environment variables; source this before anything |
| `gui/` | PyQt6 GUI source: board windows, run control, live monitoring |
| `gui/Guceiver` | Live data monitor (matplotlib) for waveforms and spectra |
| `gui/scripts/` | Bash/Python scripts runnable from GUI combo box |
| `EPICS/base-7.0` | EPICS base submodule |
| `EPICS/softIOC` | Soft IOC submodule (for GUI-side PV serving) |
| `ioc/` | IOC submodule (boot scripts, DB files, `findAllPV.py`) |
| `tcpReceiver/` | C++ multi-threaded TCP data receiver submodule (port 9001 ✅ verified 2026-04-08 — `SendReceiveSupport.c:L120` + `tcpReceiverMT.cpp:L55`, `SOCK_STREAM` ✅ verified 2026-04-08 — `SendReceiveSupport.c:L134`) |

### GUI Modules

| File | Lines | Function |
|------|-------|----------|
| `commander.py` | 852 | Main window: run control, board buttons, settings persistence | ✅ verified 2026-04-14 — `wc -l commander.py`=852 |
| `gui_DIG.py` | 374 | Digitizer board config window (per-channel + per-board PVs) |
| `gui_MTRG.py` | 1425 | Master trigger board window (largest GUI module) |
| `gui_RTR.py` | 550 | Router trigger board window (2 tabs: LINK Control, X/Y Map) |
| `gui_DataTaking.py` | 227 | IOC config dialog + live run status window |
| `gui_SYS.py` | 427 | System tabs: timestamps, link status, TCP rates, code revision (see GUI section below) |
| `gui_Board.py` | — | Generic board PV window (table of all PVs for a board) |
| `gui_LinkSys.py` | — | Link system window (`link_sys.py` launcher) |
| `gui_scalar.py` | — | Scalar/rate monitor window |
| `gui_Det.py` | — | Detector view window |
| `class_Board.py` | 73 | Board abstraction (see below) |
| `class_PV.py` | — | EPICS PV abstraction (see below) |
| `class_PVWidgets.py` | — | PV-bound Qt widgets (see below) |
| `custom_QClasses.py` | — | Custom Qt base classes (see below) |
| `json2pv.py` | — | Parses `All_PV.json` → PV objects (see below) |
| `aux.py` | 7 | Minimal helpers |
| `Guceiver/` | — | Live waveform/spectrum monitor (matplotlib) |
| `scripts/` | — | Shell/Python scripts launchable from GUI combo box |

---

## GUI Internals

### `class_PV.py` — EPICS PV Abstraction

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

### `class_Board.py` — Board Abstraction

Represents one VME board (DIG, RTRG, MTRG, DAQ):

- `BD_name` — board prefix (e.g. `MDIG0101`, `RTR001`, `MTRG001`)
- `Board_PV[]` — list of board-level `PV` objects (subscribed immediately on creation)
- `CH_PV[ch][pv_idx]` — 2D array of per-channel `PV` objects (10 ch for DIG)
  - Channel PVs are **not subscribed at creation** — only subscribed via `SubscribeChannels()` when a channel window opens, and unsubscribed via `UnsubscribeChannels()` when it closes
  - This keeps CA connections minimal when boards aren't being inspected
- Board name prefix is stamped onto each PV: `f"{BD_name}:{pv.name}{ch_idx}"` for channels, `f"{BD_name}:{pv.name}"` for board PVs

---

### `findAllPV.py` — PV List Generator (ioc/)
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

### `json2pv.py` — PV List Parser

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

### `gui_DataTaking.py` — Run Control & Data Taking

_Source: `ANLDAQ/gui/gui_DataTaking.py` (227 lines). Code-verified 2026-04-09._

Two classes:

**`IOCConfigDialog`** — Modal dialog for editing the IOC connection list before starting a run.
- Text format: one IOC per line: `IP  Port  DataType` (port defaults to 9001, DataType defaults to 8 if omitted; `#` = comment)
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
5. `QTimer` fires every 200 ms to read stdout via `select.select` (non-blocking)
6. ANSI color codes stripped (`re.sub(r'\033\[[0-9;]*m', '', line)`) before display
7. Parses `====== X.XXX Mbytes | ...` lines to update live total-size label

Stop sequence:
1. User clicks **Stop Run** → prompts for a stop comment (`QInputDialog.getText`)
2. Calls `parent.StopAcquisition(comment)` (stops EPICS acquisition, lets IOC flush data)
3. Waits **5 seconds** (`QTimer.singleShot(5000, ...)`) then sends **SIGTERM** to `tcpReceiverMT`
4. Process exit codes: 0 = finished cleanly, `-SIGTERM` = stopped by user, other = error

Button color scheme: running=green, stopped=orange, finished=blue, error=red.

---

### `custom_QClasses.py` — Custom Qt Base Widgets

| Class | Inherits | Purpose |
|---|---|---|
| `GLabel` | `QLabel` | Right-aligned label (default) |
| `GLineEdit` | `QLineEdit` | Text turns **blue** on edit, black on Enter — visual dirty indicator |
| `GTwoStateButton` | `QPushButton` | Toggles between two text/color states; `setState(bool)` + `stateChanged` signal; supports inverted color logic |
| `GFlagDisplay` | `QWidget` | Label + disabled colored square; green=True, grey=False; tooltip shows pass/fail message |
| `GArrow` | `QWidget` | Custom-painted directional arrow with configurable length, color, angle |

---

### `commander.py` — Main Window

Startup sequence:
1. Checks `SYSTEM` env var — if not set, re-execs itself via `bash -c "source EPICS_para.sh && exec python3 ..."` ✅ verified 2026-04-14 — `commander.py:L8-10`
2. Calls `GeneratePVLists('../ioc/All_PV.json')` to build all PV lists ✅ verified 2026-04-14 — `commander.py:L21,L23`
3. Creates `Board` objects for all DIG, RTR, MTRG, DAQ boards
4. Launches PyQt6 `MainWindow`

`MainWindow` layout:
- **Data Taking groupbox** — Exp Name, Run ID, Exp Folder (Browse), Start/Stop Run button, Duration combo
- **Board buttons** — one button per DIG/RTR/MTRG board → opens respective window
- **System tabs** — via `gui_SYS.py`: timestamps, links, TCP, code revision
- **Settings** — persisted to `settings.json` (exp name, folder, run counter, IOC config, duration index)
- **CollectorBox PVs** — optionally loaded via `cb_json2pv.py` + `CollectorBox_PV.json` (graceful fallback if absent) ✅ verified 2026-04-07 — `commander.py:L93-97`: `try: from cb_json2pv import LoadCollectorBoxPVs ... except Exception: CB_PV, CB_DET_LIST = [], []`
- **Guceiver** — live monitor launched from GUI; path added to `sys.path` at startup

Run control:
- Start: `caput Online_CS_StartStop Start` + `caput Online_CS_SaveData Save` → spawns `tcpReceiverMT` ✅ verified 2026-04-08 — `start_run.sh:L212-213,L163-164`
- Stop: `caput Online_CS_StartStop Stop` → wait → `kill_IOC.sh` ✅ verified 2026-04-08 — `start_run.sh:L220`
- Run ID auto-increments; saved to `settings.json`

---

## Architecture

```
VME IOC (VxWorks / MVME5500)
    |  EPICS Channel Access (PV read/write)    |  TCP port 9001 (GEB binary data)
    v                                           v
commander.py (PyQt6 GUI)               tcpReceiverMT (C++ server)
    |                                           |
    +-- DIG / RTR / MTRG board windows          +-- Binary run files (timestamped)
    +-- System status tabs                      +-- Guceiver (live matplotlib plots)
    +-- Run control (start/stop)
```

---

## EPICS System Configurations (from EPICS_para.sh)

`EPICS_HOST_ARCH` is auto-detected: `export EPICS_HOST_ARCH="linux-$(uname -m)"` (e.g. `linux-aarch64` on Pi5, `linux-x86_64` on x86). ✅ verified 2026-04-13 — `EPICS_para.sh:L1` (commit 6af0b88, 2026-04-02)


| System | CA Server Port | CA Repeater Port | Terminal Server | IOC IPs |
|--------|---------------|-----------------|----------------|---------|
| DGS | 5064 | 5065 | 192.168.203.186, 192.168.203.91 ✅ verified 2026-04-14 — `EPICS_para.sh:L47` (`TERMINAL_SERVER`) | .141–.145, .177–.183 (12 VMEs) |
| DFMA | 5068 | 5069 | — | — |
| DXA | 5072 | 5073 | 192.168.203.47 | .212, .213 |
| SlopeBox | 5074 | 5075 | 192.168.203.139 | — |
| DUB | 5078 | 5079 | — | — |
| DUO | 5080 | 5081 | 192.168.203.54 | 192.168.203.81 |

✅ DGS ports (5064/5065) and IOC IPs verified 2026-04-05 — `ANLDAQ/EPICS_para.sh:L45-46`

**DGS VME IOC IPs:** 192.168.203.141–145, 177–183 (12 crates total)

---

## How to Use

### Setup

```sh
# 1. Set system target (edit top of file)
# export SYSTEM="DGS"
source EPICS_para.sh

# 2. Build EPICS base (first time only)
cd EPICS/base-7.0 && make && cd ../..

# 3. Build softIOC (first time only)
cd EPICS/softIOC && make && cd ../..

# 4. Generate PV list
cd ioc && ./findAllPV.py && cd ..

# 5. Launch GUI
./gui/commander.py
```

### Build tcpReceiver

```sh
cd tcpReceiver && make
# Produces: tcpReceiverMT (TCP) and tcpReceiverUDP
```

### Running a DAQ Run

1. Click **Edit IOC Config** → set IOC IP, port (default 9001), data type (default 8)
2. Enter experiment folder and run name
3. Set run duration
4. Click **Start Run** → spawns `tcpReceiverMT`
5. Monitor with **Guceiver** (live waveform/spectrum view)
6. Click **Stop Run** → data saved to `{expFolder}/{expName}_{runID:03d}/`

---

## Dependencies

- Python: `python3-pyqt6`, `python3-pyepics`
- C++: standard `make`, `g++`
- EPICS base 7.0 (submodule)
- IOC repo (submodule) — provides `All_PV.json`
- tcpReceiver repo (submodule) — provides the data receiver binary

---

## VxWorks IOC Data Pipeline (inLoop / outLoop / MiniSender)

**Source:** `DGS_tools_pack/vxworks/dgsDrivers/dgsDriverApp/src/`  
Authors: John Anderson (inLoop), Michael Oberling (outLoop)

The VME IOC runs three cooperating state machines that read digitizer and trigger FIFOs over the VME bus and hand off data to the TCP sender:

```
Digitizer FIFO (VME bus)
    │
    ▼  inLoop (inLoopSupport.c)
    │  └─ CheckAndReadDigitizer() / CheckAndReadTrigger()
    │  └─ Reads raw events from DIG/TRIG FIFOs via VME A32/D32
    │  └─ Prepends GEB-style header, pushes into shared memory queue
    │
    ▼  outLoop (outLoopSupport.c)
    │  └─ Dequeues buffers from shared memory
    │  └─ Validates event headers, checks timestamps, tracks data rates
    │  └─ Handles per-channel raw data length (configurable since 2023-04-10)
    │  └─ Optionally builds histograms (HISTO_ENABLE) or captures traces (TRACE_ENABLE)
    │  └─ Hands validated buffers to MiniSender queue
    │
    ▼  MiniSender (SendReceiveSupport.c)
       └─ TCP server on port 9001 (SOCK_STREAM)
       └─ Accepts connection from tcpReceiverMT
       └─ Responds to requests with data buffers via send()
```

### Key data structures

**inLoopSupport.c:** ✅ verified 2026-04-08 — `inLoopSupport.c:L31-49`
- `MstrLogicReg[10]` — VME addresses of master logic status registers (one per board)
- `FIFOStatusReg[10]` — VME addresses of FIFO status registers (one per board)
- `RawDataLengthReg[10][10]` — configurable raw data length per channel per board (added 2023-04-10)
- `DigitizerCalcEventSize[10][10]` — computed event size (in 32-bit words) per channel per board; derived from `RawDataLengthReg` ÷ 2 + 1 (for the `0xAAAAAAAA` sync word)
- `MinimumCalcEventSize[10]`, `MaximumCalcEventSize[10]` — smallest/largest event size per board (added 2023-04-10)

**outLoopSupport.c:** ✅ verified 2026-04-08 — `outLoopSupport.c:L61-62`
- `TotalBuffers_Written`, `TotalBuffers_Lost` — global buffer statistics (exposed via `GetTotalBuffers_Written()` / `GetTotalBuffers_Lost()`)
- `DataRate[board]`, `DataTotal[board]`, `DataLost[board]` — per-board statistics

### Functions (inLoop)
- `SetupBoardAddresses()` — maps VME addresses for all boards in a crate
- `CheckAndReadDigitizer()` — polls DIG FIFO, reads events if available
- `CheckAndReadTrigger()` — polls TRIG FIFO, reads trigger records
- `SendEndOfRun()` — signals end-of-run marker into queue
- `CalcDigMaxEventsPerRead()` — computes max events per VME read cycle
- `ResetAndReEnableDig()` — resets and re-enables digitizer

### outLoop statistics tracking
- Prescaled debug output (1 in 0x101 buffers prints status, 1 in 0x4001 prints headers)
- Per-board error counts + error data (4 error types per board)
- Per-channel last timestamp tracking for sequence checking

### Queue system (VxWorks pointers, not data copies)
_Source: [IOC Code Design](https://wiki.anl.gov/gsdaq/IOC_Code_Design) (wiki)_

Three pointer queues — buffers are never copied, only pointers move:
- **qFree** → inLoop grabs pointer, fills buffer with VME data → moves pointer to **qWritten**
- **qWritten** → outLoop checks buffer, validates → moves pointer to **qSender**
- **qSender** → minisender grabs pointer, sends data to gtReceiver → returns pointer to **qFree**

Key flow-control PVs:
- `Online_CS_StartStop`, `Online_CS_SaveData` — monitored by all three state machines
- `DAQBx_*_CS_Ena` — per-board enable, monitored by inLoop
- `VMEx:MDIG1_CV_Running` — inLoop↔outLoop communication PV
- `DAQCx_CV_InLoop1/2` — inLoop MB/s and type-F header rate
- `DAQCV_CV_BuffersAvail`, `DAQCx_CV_NumSendBuffers` — qFree and qSender depths
- `DAQCx_CV_OutLoop1,2,3,4` — lost buffers per digitizer
- `DAQCx_OL_DataRate0..3` — read rates in kB/s per digitizer
- `DAQCx_OL_BufLostPercent` — percent of buffers lost
- `DAQCx_CV_SendRate` — sender data rate in kB/s

### Digitizer FIFO depth: live vs event-bound
- **Live depth**: exact current count of words in FIFO
- **Event-bound depth**: updates only when a full event is in the FIFO (lags live by ≤ 1 event)
- inLoop uses **event-bound depth** → every buffer read is guaranteed to contain only complete events (no need to stitch partial events across reads)

---

## Connections to Other Subsystems

- **ioc/** — provides EPICS PV definitions (db/dbd) and boot scripts used by ANLDAQ
- **vxworks/** — the firmware running on VME crates that ANLDAQ talks to over EPICS CA
- **fpga/** — firmware that runs inside the DIG/RTRG/MTRG boards ANLDAQ configures
- **dgs_analysis/** — consumes the binary data files written by `tcpReceiverMT`

---

## tcpReceiver — Detailed Notes

> **Full reference moved to [`ANLDAQ_tcpReceiver.md`](ANLDAQ_tcpReceiver.md)** (split 2026-04-16 for size).

Key facts:
- Three binaries: `tcpReceiver` (single-threaded), `tcpReceiverMT` (production multi-threaded), `tcpReceiverUDP` (experimental UDP forwarding)
- IOC is the **TCP server** on port 9001 (`SOCK_STREAM`); receiver is the TCP client
- DIG packets: `0xAAAAAAAA` magic; TRIG packets: `0xAAAA0000` (16 words → repacked to 10)
- Max file size: 2 GB; auto-split. Default buffer: 1M words (4 MB).
- Run control: `expInfo.sh` → `start_run.sh` / `stop_run.sh` / `sync_exp_data.sh`
- `run_control_gui.py` — standalone Tkinter GUI for `dgs4` (SSHes to dcs2)
- `basic_settings_DGS.sh` — sets all DIG channels to known-good CFD config
- Legacy receivers in `legacy/`: `dgsReceiver.cpp` (MBO v6.57) + Ryan's fork

> ⚠️ Wiki says "UDP packets" — incorrect. `SOCK_STREAM` = TCP. See `ANLDAQ_tcpReceiver.md` for details.

> **Full details, packet consistency table, `class_DIG.h`/`class_TDC.h` decode docs:** → [`ANLDAQ_tcpReceiver.md`](ANLDAQ_tcpReceiver.md)

## GUI Windows — Detailed Reference

> **Full reference moved to [`ANLDAQ_GUI_windows.md`](ANLDAQ_GUI_windows.md)** (split 2026-04-16 for size).

Windows covered:
- **`gui_MTRG.py`** (`MTRGWindow`, 1,425 lines) — 5 tabs: Trigger/Veto, Wheel RAM, LINK Control, CPLD map, Other
- **`gui_Det.py`** (`DetWindow`) — 110 detectors by collector box group (NE/SE/NW/SW); DV Monitor + HV tabs
- **`gui_scalar.py`** (`ScalarWindow`) — all DIG boards; per-channel threshold, disc count, accepted hit count
- **`gui_RTR.py`** (`RTRWindow`, 550 lines) — 2 tabs: LINK Control (SERDES grid), X/Y Map
- **`gui_RAM.py`** (`RAMWindow`, 34 lines) — 32×32 grid for VETO/TRIG/SWEEP RAM visualization
- **`gui_SYS.py`** (427 lines) — Timestamps, Link Status, TCP Transfer, Code Revision, Global Settings tabs
- **`gui_LinkSys.py`** (295 lines) — wrapper for `link_sys.py`; MTRG+RTR link map GUI, 5-stage runner
- **Trigger Setup Scripts** (`gui/scripts/`) — 5-stage `trig_setup_Stage*.sh` + `SYSTEM_DEFINES.sh` topology

> **Full tab structure, PV lists, SYSTEM_DEFINES.sh topology, DC balance notes:** → [`ANLDAQ_GUI_windows.md`](ANLDAQ_GUI_windows.md)


---

## softIOC — Global Broadcast PV System (`JustGlobals.db`)

_Source: `ANLDAQ/EPICS/softIOC/db/JustGlobals.db` (14,248 lines, auto-generated)_ ✅ verified 2026-04-10 — `wc -l JustGlobals.db`

The `ANLDAQ/EPICS/softIOC` is a lightweight EPICS soft IOC that runs alongside the GUI. Its primary purpose is hosting `JustGlobals.db` — a **broadcast PV layer** that lets operators set any DIG parameter across all 12 VME crates simultaneously with a single caput.

### How It Works

Each top-level `GLBL:DIG:<preset>_<param>` PV fans out through a chain of `dfanout` records (`GLBL:DIG:F00:`, `F01:`, ...) until reaching `VMExx:GLBL:<param>` on every crate. Writing one PV → all 12 crates updated. The DB contains **2,124 `dfanout` records** (all records are dfanout type) and **177 top-level broadcast PVs** matching `GLBL:DIG:[A-Z]...`. ✅ verified 2026-04-11 — `grep -c "record(dfanout" JustGlobals.db` = 2124; `grep -c "record(.*GLBL:DIG:[A-Z]" JustGlobals.db` = 177. File: 14,248 lines.

### Preset Profiles

The current DB has **per-detector-type presets** in addition to the flat `GLBL:DIG:` namespace:

| Preset prefix | Detector type |
|--------------|---------------|
| `BGOp` | BGO **primary** channel |
| `BGOs` | BGO **secondary** channel |
| `GeS` | Ge crystal (standard) |
| `GeC` | Ge crystal (CFD mode) |

Full PV name pattern: `GLBL:DIG:<NN>:<preset>_<param>` where `<NN>` is a fanout stage index.

### Broadcast PV Categories

| Category prefix | Examples |
|----------------|----------|
| `master_*` | `master_fifo_reset`, `master_counter_reset`, `master_logic_enable` |
| `sd_*` | SERDES control: `sd_sync`, `sd_tx_pwr`, `sd_rx_pwr`, `sd_line_loopback_en` |
| `trigger_*` | `trigger_mux_select`, `trigger_polarity` |
| `disc_*` | Discriminator: `disc_width`, `disc_count_mode`, `coarse_threshold` |
| `pileup_*` | `pileup_mode`, `pileup_extension_mode`, `pileup_waveform_only_mode` |
| `preamp_reset_*` | `preamp_reset_delay`, `preamp_reset_delay_en` |
| `event_*` | `event_extension_mode`, `event_count_mode` |
| `win_*` | Window params: `d_window`, `k_window`, `m_window`, `p1_window`, `p2_window` |
| `channel_enable` | Enable/disable all channels at once |
| `counter_reset` | Reset all counters |

> **Note:** The SVN archive (2022) had 5,195-line version with simpler flat namespace. Current version (14,248 lines) adds per-preset profiles for BGO/Ge channel types.

---

## softIOC — Support PVs (`dgsSupport.db`)

_Source: `ANLDAQ/EPICS/softIOC/db/dgsSupport.db` (235 lines, hand-crafted — last edit 2025-03-23 JTA/Ryan)_ ✅ verified 2026-04-10 — `wc -l dgsSupport.db`

Companion to `JustGlobals.db`. Contains **hand-crafted PVs** that are not auto-generated from the register map — glue records for run control, setup state, and computed readbacks.

### Run Control PVs

| PV | Type | Function |
|----|------|----------|
| `RunNum` | `longout` | Current run ID number. Added by Ryan 2025-03-23. Default=0, PINI=YES. |
| `Online_CS_StartStop` | `bo` | Main Run/Stop button (`Stop`=0, `Start`=1). Monitored by all three IOC state machines. |
| `Online_CS_SaveData` | `bo` | Data save toggle (`No Save`=0, `Save`=1). The green button below Run/Stop. |
| `Setup_Script_State` | `mbbo` | Setup script status indicator (0=UNKNOWN, 1=TRIG OK, 2=DIG OK, 3=OTHER, 4=TRIG ERROR, 5=DIG ERROR, 6=OTHER ERROR, 7=SCRIPT RUNNING). |
| `ScriptStage` | `ao` | Stage counter displayed during long scripts (written by the script to show progress). |

### How `Online_CS_StartStop` Triggers All IOCs

`Online_CS_StartStop` is a plain `bo` record in the softIOC — **no `OUT` link, no forward links, no fan-out**. It is simply a global CA-visible flag. The actual run start/stop mechanism is driven by **SNL (State Notation Language) sequencer programs** running inside each VME IOC.

Each DIG IOC runs an instance of `inLoop` (source: `DGS_SVN/dgs/20180921/inLoop.st`), which monitors the PV via CA:

```
DECLMON(short, AcqRun, Online_CS_StartStop)  // CA monitor — reacts on any change
DECLMON(short, AcqEna, {PVAcqEna})           // per-board enable (e.g. DAQB1_1_CS_Ena)
```

**State machine flow (`inLoop`):**

```
[setup]  wait for AcqRun==1 AND AcqEna==1
           → drain FIFO (clearDigFIFO)
           → set MLE=1 (Master Logic Enable on that DIG board)
         → [run]

[run]    continuously poll FIFO (checkDigFIFO + serviceOneBuffer)
           → if AcqRun==0: clear MLE=0
         → [waitfordone]

[waitfordone]  drain remaining data → [setup]
```

**Key point:** Every VME DIG IOC runs its own `inLoop` instance, all CA-monitoring the same `Online_CS_StartStop` PV simultaneously. When `caput Online_CS_StartStop Start` is issued:
1. All DIG IOC sequencers detect `AcqRun=1` in parallel
2. Each independently drains its FIFO, enables its MLE, and begins readout
3. No central coordinator — the broadcast PV *is* the synchronization primitive

The trigger IOC (`inLoopTrig.st`) uses the same pattern to control the MTRG/RTRG side.

✅ verified 2026-04-10 — `DGS_SVN/dgs/20180921/inLoop.st:L50,L110-190` + `ANLDAQ/EPICS/softIOC/db/dgsSupport.db:L20-27`

### MTRG Computed Readbacks (VME10)

The MTRG firmware exposes trigger rate counters and the 48-bit timestamp as **split 16-bit registers** (HIGH + LOW). `dgsSupport.db` reassembles them using `calcout` records with `CALC="(B<<16)+A"` at 1-second scan:

- `VME10:MTRG:TIMESTAMP_RBV` — 32-bit assembled timestamp from `TIMESTAMP_A/B/C_RBV` (0.1 second scan; uses only B and C: `(B<<16)+C`)
- `VME10:MTRG:TRIG_RATE_COUNTER_1–8_RBV` — accepted trigger rate counters 1–8
- `VME10:MTRG:RAW_TRIG_RATE_COUNTER_1–8_RBV` — raw (pre-prescale) trigger rate counters 1–8

All 17 `calcout` records are hardcoded to `VME10` (the MTRG crate in the standard DGS configuration).

> **Note:** `dgsSupport.db` and `JustGlobals.db` are both loaded by the softIOC at boot. Together they provide the full soft-IOC PV surface that the ANLDAQ GUI and IOC state machines depend on.

---

## Notes


- `EPICS_para.sh` auto-patches `EPICS/softIOC/configure/RELEASE` with the correct `EPICS_BASE` path — never edit that file manually
- PV list (`ioc/All_PV.json`) must be regenerated after any IOC boot script or DB file changes
- The GUI will attempt to auto-source `EPICS_para.sh` if it wasn't sourced manually

---

## See Also

- `knowledgeBase/ioc.md` — EPICS IOC boot scripts, DB files, PV definitions
- `knowledgeBase/vxworks.md` — VxWorks build pipeline (produces the firmware ANLDAQ talks to)
- `knowledgeBase/fpga.md` — DIG/RTRG/MTRG firmware overview
- `knowledgeBase/dgs_analysis.md` — Downstream analysis (fastEventConstructor, parquet_pysort) consuming `tcpReceiverMT` output
- `knowledgeBase/snapshot_pv.md` — PV snapshot utility (`dumpPVs.py` / `putPVs.py`) invoked by `start_run.sh`
- `knowledgeBase/ttcl.md` — TTCL trigger timing (feeds the MTRG TAC-II data decoded in `class_TDC.h`)
- `knowledgeBase/DIG_firmware_expert.md` — DIG firmware details; confirms packet format matched by `class_DIG.h`
- `knowledgeBase/EPICS_asyn.md` — asyn driver internals: caput/caget flow, port concept, asynUInt32Digital
- `knowledgeBase/collectorbox_devicesupport.md` — collector box EPICS device support (SPI driver, CAMAC_IO link)
- `knowledgeBase/guceiver.md` — Guceiver live diagnostic GUI (waveform/spectrum/TAC-II tabs); companion to the main ANLDAQ GUI
