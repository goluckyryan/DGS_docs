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
| `tcpReceiver/` | C++ multi-threaded TCP data receiver submodule (port 9001, `SOCK_STREAM`) |

### GUI Modules

| File | Lines | Function |
|------|-------|----------|
| `commander.py` | 852 | Main window: run control, board buttons, settings persistence |
| `gui_DIG.py` | 374 | Digitizer board config window (per-channel + per-board PVs) |
| `gui_MTRG.py` | 1425 | Master trigger board window (largest GUI module) |
| `gui_RTR.py` | — | Router trigger board window |
| `gui_DataTaking.py` | 227 | IOC config dialog + live run status window |
| `gui_SYS.py` | — | System tabs: timestamps, link status, TCP rates, code revision |
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
`All_PV.json` is ~113k lines for a full Gammasphere config (all VME crates × all boards × all channels).

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
All lists sorted by PV name. Called once at startup in `commander.py`.

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
1. Checks `SYSTEM` env var — if not set, re-execs itself via `bash -c "source EPICS_para.sh && exec python3 ..."`
2. Calls `GeneratePVLists('../ioc/All_PV.json')` to build all PV lists
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
- Start: `caput Online_CS_StartStop Start` + `caput Online_CS_SaveData Save` → spawns `tcpReceiverMT`
- Stop: `caput Online_CS_StartStop Stop` → wait → `kill_IOC.sh`
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

| System | CA Server Port | CA Repeater Port | Terminal Server | IOC IPs |
|--------|---------------|-----------------|----------------|---------|
| DGS | 5064 | 5065 | 192.168.203.186, 192.168.203.91 | .141–.145, .177–.183 (12 VMEs) |
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
- `MstrLogicReg[10]` — VME addresses of master logic status registers (one per board)
- `FIFOStatusReg[10]` — VME addresses of FIFO status registers (one per board)
- `RawDataLengthReg[10][10]` — configurable raw data length per channel per board (added 2023-04-10)
- `DigitizerCalcEventSize[10][10]` — computed event size per channel per board
- `DataRate[board]`, `DataTotal[board]`, `DataLost[board]` — per-board statistics
- `TotalBuffers_Written`, `TotalBuffers_Lost` — global buffer statistics

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

## tcpReceiver — Detailed Notes (updated 2026-04-04)

### Three Executables

| Binary | Source | Mode |
|--------|--------|------|
| `tcpReceiver` | `tcpReceiver.cpp` | Single-threaded, one IOC |
| `tcpReceiverMT` | `tcpReceiverMT.cpp` | Multi-threaded, N IOCs from config file |
| `tcpReceiverUDP` | `tcpReceiverUDP.cpp` | Multi-threaded + UDP forwarding (online analysis) |

### TCP Protocol — Verified from Source Code

The IOC↔receiver connection is **TCP (`SOCK_STREAM`)** on **port 9001**. This has been verified across all code generations:

**IOC side (VxWorks — `SendReceiveSupport.c`):**
```c
#define SERVER_PORT 9001
SocketForRequests = socket(AF_INET, SOCK_STREAM, 0);  // TCP
bind(SocketForRequests, ...);
listen(SocketForRequests, 10);
ReadWriteSocket = accept(SocketForRequests, ...);
// Options set: TCP_NODELAY, TCP_MAXSEG
```
- IOC is the **TCP server** — binds, listens, accepts incoming connections

**Receiver side (ANLDAQ — `receiver.h`):**
```c
netSocket = socket(AF_INET, SOCK_STREAM, 0);  // TCP
connect(netSocket, &server_addr, ...);         // connects to IOC:9001
send(netSocket, &request, ...);                // sends request
recv(netSocket, reply, ...);                   // receives data
```
- Receiver is the **TCP client** — connects to IOC IP:9001

**Old receiver (SVN `gtReceiver6.c` and `dgsReceiver.cpp`):**
```c
instance->recSock = socket(AF_INET, SOCK_STREAM, 0);  // TCP
connect(instance->recSock, &instance->adr_srvr, ...); // TCP client
```
- Same TCP pattern going back to the oldest known version

> ⚠️ **The wiki (`/gsdaq/DAQ_system`) says "UDP packets" — this is incorrect.** The wiki description (written by J. Anderson, March 2023) is misleading. `SOCK_STREAM` = TCP by definition; UDP would use `SOCK_DGRAM` + no `listen()`/`accept()`. `tcpReceiverUDP` is a separate experimental variant, not the production receiver.

### Data Flow

```
IOC (VME crate, port 9001)
    │  TCP request/reply protocol
    ▼
IOCReceiver::GetData()
    │  4-word header: [type, recordSize, status, numRecords]
    │  then recv() loop until all bytes received
    ▼
IOCReceiver::WriteData()
    │  parse word by word:
    ├─ 0xAAAAAAAA → DIG packet
    │     decode board_id, ch_id, packet_length
    │     prepend GEB header (type, length, timestamp)
    │     write to per-channel file: runName_NNN_BBBB_C
    │
    ├─ 0xAAAA0000 → TRIG packet
    │     16 words raw → repack into 10-word DIG-like payload
    │     write to file: board=99, ch=A (0xA)
    │
    └─ ch_id special codes:
         0xD = Run is Done → exit
         0xE = Empty packet
         0xF = FIFO overflow/underflow
    ▼
OutFile (per board+channel)
    - auto-splits at 2 GB
    - sets file read-only on close
    - thread-safe via printMutex
```

### Key Constants (`constant.h`)

- Max file size: **2 GB**
- Default receive buffer: **1M words = 4 MB**
- TRIG raw packet: **16 words** → repacked to **10 words**

### GEB Header (`#define ENABLE_GEB_HEADER`)

Each event is prepended with a 16-byte GEB header before writing:
```
int32_t  type       ← GEB_ID from expInfo.sh (e.g. 14)
int32_t  length     ← packet length in bytes
uint64_t timestamp  ← 48-bit event timestamp from DIG header
```
This is the format consumed by downstream analysis (fastEventConstructor / parquet_pysort).

### `class_DIG.h` — DIG Hit Decoder

Decodes the full DIG event packet header (words 0–13 + trace):
- **Header type 7** = LED mode (software convention; FPGA hardware uses type 4)
- **Header type 8** = CFD mode (software convention; FPGA hardware uses type 5)
- Fields: `EVENT_TIMESTAMP`, `PRE/POST_RISE_ENERGY`, `SAMPLED_BASELINE`, `PEAK_TIMESTAMP`, `PILEUP_FLAG`, CFD samples (0/1/2), vernier timestamps, trace waveform

### `class_TDC.h` — TAC-II Hit Decoder

Decodes the MTRG TDC/TAC-II packet (10 words after repacking):
- `timestampTrig` — MTRG 48-bit trigger timestamp (×10 ns)
- `coarseTS` — 16-bit coarse TDC counter
- Four-phase 4 ns counters (0°/90°/180°/270°) + vernier AB/CD (6 bits each, ~50 ps/step)
- `CalTAC_simple()` — computes average phase timestamp in ns with ~50 ps resolution
- Trash data detection: specific counter pattern `0x1006/1005/1004/1003`

### Run Control Scripts

**`start_run.sh`:**
1. Sources `expInfo.sh` (experiment name, folder, GEB_ID, run number)
2. Increments `NEXT_RUN` in expInfo.sh
3. Creates `dataFolder/expName_RRR/`
4. Runs `~/snapshot_pv/dumpPVs.py` → **snapshot_pv lives on dcs2.onenet, not pi5-dgs**
5. Appends to `RunTimestamp.txt`
6. Posts to ELOG (`elog.phy.anl.gov:443`)
7. Launches `tcpReceiver` (one gnome-terminal per IP) or `tcpReceiverMT` (one process)
8. `caput Online_CS_StartStop Start` + `caput Online_CS_SaveData Save`

**`stop_run.sh`:**
1. `caput Online_CS_StartStop Stop`
2. Wait 10 s for IOC to flush
3. `caput Online_CS_SaveData No Save`
4. Wait 5 s → `kill_IOC.sh`
5. Posts to ELOG with run duration + data size
6. Launches `RunParquet` (parquet sort pipeline) in a new terminal

**`sync_exp_data.sh`** (new in latest pull):
- Monitors `dataFolder` + `expInfo.sh` with `inotifywait` (falls back to polling every 20 s)
- Rsyncs data to `nfsFolder` (NFS mount) on new run or file change
- Uses `--append` during active run (binary files are append-only)
- Final sync on Ctrl+C

**`run_control_gui.py`** — Standalone Tkinter run control GUI for `dgs4`:
- Runs on `dgs4` (shebang: `/home/dgs/.conda/envs/py3tk/bin/python3`); SSHes to `dcsu@dcs2.onenet` using `~/.ssh/id_rsa`
- Reads experiment info (name, next run number, folder) via `expInfo.sh` on dcs2 at startup + on demand
- **Start Run**: streams `start_run.sh` output line-by-line via SSH Popen; maps output substrings to friendly status messages (e.g. "Taking PV Snapshot" → "Taking PV snapshot...", "is running" → "DAQ started!")
- **Stop Run**: similarly streams `stop_run.sh`; includes Parquet sort + elog post status messages
- Live displays: wall clock, elapsed run timer, data folder size (polled every 10 s via `du -sh` on dcs2), recent run log (tails `RunTimestamp.txt`)
- State machine: `idle → starting → running → stopping → idle`; buttons enable/disable accordingly
- ANSI escape codes stripped from SSH output before display
- Script dir on dcs2: `/home/phy/dcsu/ANLDAQ/tcpReceiver/`

### Packet Consistency: Receiver vs FPGA Firmware

The receiver and `class_DIG.h` are fully consistent with the FPGA DIG packet format (`Event_Header_FIFO.vhd`, `event_packer.vhd`):

| Field | FPGA Word/Bits | `class_DIG.h` extraction | Match |
|-------|---------------|--------------------------|-------|
| Magic `0xAAAAAAAA` | Word 0 | DIG packet delimiter | ✅ |
| `CH_ID[3:0]` | W1[3:0] | `header[1] & 0x0000000F` | ✅ |
| `UserDef[11:0]` | W1[15:4] | `(header[1] & 0x0000FFF0) >> 4` | ✅ |
| `PacketLen[10:0]` | W1[26:16] | `(header[1] & 0x07FF0000) >> 16` | ✅ |
| `GeoAddr[4:0]` | W1[31:27] | `(header[1] & 0xF8000000) >> 27` | ✅ |
| `TIMESTAMP[47:0]` | W2[31:0] + W3[15:0] | `W2 \| (W3[15:0] << 32)` | ✅ |
| `HEADER_TYPE[3:0]` | W3[19:16] | `(header[3] & 0x000F0000) >> 16` | ✅ |
| `EVT_TYPE[2:0]` | W3[25:23] | `(header[3] & 0x03800000) >> 23` | ✅ |
| `HDR_LEN[5:0]` | W3[31:26] | `(header[3] & 0xFC000000) >> 26` | ✅ |
| Word 4 flags | W4 various bits | all correctly bit-extracted | ✅ |
| `SAMPLED_BASELINE` | W6[23:0] | `header[6] & 0x00FFFFFF` | ✅ |
| `PRE_RISE_SUM` | W8[23:0] | `header[8] & 0x00FFFFFF` | ✅ |
| `POST_RISE_SUM` | W9[15:0]<<8 \| W8[31:24] | same reconstruction | ✅ |
| Waveform (W14+) | 2 × 14-bit samples/word | `word & 0x3FFF` + `(word>>16) & 0x3FFF` | ✅ |

**Note on header type encoding:**
- FPGA hardware: LED = `0100` (4), CFD = `0101` (5)
- Software/IOC convention: LED = 7, CFD = 8
- The IOC device support re-encodes the type before sending over TCP.

**TRIG packet:** MTRG sends 16 words raw (`0xAAAA0000` header). Receiver repacks to 10 words. `class_TDC.h::FillTDC()` correctly decodes trigger timestamp, trigger type, wheel, multiplicity, coarse TDC, trigger bitmask, four 4 ns phase counters, and vernier AB/CD — all consistent with the TAC-II data path in the MTRG Main FPGA firmware.

---

## GUI: Master Trigger Window (`gui_MTRG.py`)

`MTRGWindow` — opened by clicking an MTRG board button in DGS Commander. The largest GUI module (1,425 lines). Uses a tab widget with 5 tabs, plus a top-level board info panel.

_Source: `ANLDAQ/gui/gui_MTRG.py` (code-read 2026-04-06)_

### Tab Structure

| Tab | Class | Contents |
|-----|-------|----------|
| **Trigger/Veto Control** | `triggerControlTab` | Trigger mode, multiplicity thresholds, veto masks, NIM I/O settings |
| **Wheel RAM** | `wheelRAMTab` | Lookup table editor for the trigger wheel RAM (determines accept/reject per pattern) |
| **LINK Control** | `linkControlTab` | Per-link enable/disable, SERDES lock status, SYNC bit controls |
| **Trigger/CPLD map** | `CPLDControlTab` | CPLD threshold map: maps 40-pin ribbon connections to CPLD fast-strobe channels |
| **Other Control** | `otherControlTab` | Miscellaneous MTRG register controls |

### Top-Level Board Info Panel

Always visible above the tabs. Shows:
- **MISC_STAT** register — decoded bit display (`RRegisterDisplay`): NIM IN 1/2 state + misc status
- **Code Revision** / **Code Date** — hex readback (`reg_CODE_REVISION`, `reg_CODE_DATE`)
- **Timestamp A/B/C** — hex readback
- **Clock Source** — toggle button (`ClkSrc`): internal vs external
- **Imp Sync** — imposing-sync status
- **Diagnostic counters** — 8 counters: Man/Aux Trigs, Sum X Trigs, Sum Y Trigs, Sum XY Trigs, CPLD Trigs, Link L Locks, NIM 1 Trigs, NIM 2 Trigs
- **Raw trigger rate counters** — `reg_RAW_TRIG_RATE_COUNTER_N_HIGH/LOW` per algorithm
- **Accepted trigger rate counters** — `reg_TRIG_RATE_COUNTER_N_HIGH/LOW` per algorithm

### `templateTab` Base Class

All 5 tabs inherit from `templateTab`:
- Holds a `board` reference + `pvWidgetList`
- `QTimer` calls `UpdatePVs()` periodically — only updates if the tab is **visible** (skips hidden tabs to reduce CA traffic)
- `FindPV(name)` — searches `board.Board_PV` by the last `:` segment of the PV name

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

The window has two tabs:
- **DV Monitor tab** — 2×2 grid of group boxes (NE/SE/NW/SW), one cell per detector showing live PV values
- **HV tab** — high voltage status per detector

## GUI: Scalar Window (`gui_scalar.py`)

`ScalarWindow` — "Scalar - All Digitizers" window. Shows all digitizer boards in a scrollable grid of GroupBoxes. Per channel per board, displays (live PV, timer-updated):
- `led_threshold` — LED threshold setting
- `disc_count` — discriminator fire count
- `ahit_count` — accepted hit count

_Source: `gui_scalar.py` commit `0f3f2df` 2026-04-06 (code-verified)_

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
| `MT_VME_LEADER` | `VME10` | MTRG crate |
| `MT_USE_LINK_CLK` | `0` | Local clock (not remote/link clock) |
| `DIG_CLOCK_SEL` | `1` | Digitizer clock source = SERDES |
| `PROPAGATE_TRIG_FROM_DUB/DFMA/DXA` | `0` | No remote trigger propagation |
| `PERFORM_ERROR_CHECKS` | `0` | Error checking disabled |

**Router config** (`LIST_OF_ROUTERS`): 4 RTRGs, each using links A–F (6 Router channels) + Link L back to MTRG:
```
VME03:RTR1  A B C D E F X X  L X X
VME06:RTR2  A B C D E F X X  L X X
VME09:RTR3  A B C D E F X X  L X X
VME12:RTR4  A B C D E F X X  L X X
```

**MTRG link map** (`MT_LINK_MAP`): Links A–D → RTR1–RTR4; E–H and L/R/U masked:
```
RTR1 → Link A    RTR2 → Link B    RTR3 → Link C    RTR4 → Link D
E, F, G, H, L, R, U → MASKED
```

**Digitizer config** (`LIST_OF_DIGITIZERS`): 12 crates; VME06 and VME10 have only 2 DIGs, rest have 4:
```
VME01–05, 07–09, 11–12: MDIG1 MDIG2 MDIG3 MDIG4  (4 boards each)
VME06, VME10:            MDIG1 MDIG2               (2 boards each)
```
Total: 10×4 + 2×2 = **44 digitizer boards × 10 ch = 440 channels** ✅ verified 2026-04-07 — `SYSTEM_DEFINES.sh`

Key details:
- All stages read a `SYSTEM_DEFINES.sh` file (passed as arg 1) that defines `MT_VME_LEADER`, `LIST_OF_ROUTERS`, etc.
- Stage 1 sets `ClkSrc` and all `LINK_L_PROPAGATE_Fx` registers on MTRG
- Stage 2 loops over `LIST_OF_ROUTERS` with bash substring extraction (`${RTR:6:3}`) to parse VME address
- Stage 5 clears SYNC bits (the gotcha described in `troubleshooting.md`) — this is the final step that lets real data flow
- `basic_settings_LED.py` — sets all DIG channels to **LED mode** with hardcoded defaults: threshold=300, `IntAcptAll` trigger, `p1=0.07µs, p2=0.05µs, m=2.5µs, k0=0.5µs, k=0.5µs, d=0.16µs`, RiseEdge polarity, raw_data_delay=0.5µs, raw_data_length=0.32µs. Targets CH 5–9 on VME66 MDIG1/MDIG2 (test stand config). Uses `epics.caput` with wait=True. Ends with `Online_CS_StartStop=Stop`, `Online_CS_SaveData=No Save`.
- `enableScriptList.txt` — lists which scripts are enabled in the GUI

**Important EPICS write caveat** (from `Serdes_Linkup.sh` comments):
- Writing to a **whole-register PV** (e.g. `VME10:MTRG:reg_INPUT_LINK_MASK`) updates that PV and its `_RBV`, but does **NOT** update the "breakout" bit PVs (e.g. `ILM_A` through `ILM_H`)
- Writing to a **breakout PV** updates the register and the whole-reg `_RBV`, but does **NOT** update the whole-reg PV itself
- **Scripts must always use breakout PVs** (not whole-reg PVs) to match what the GUI displays

_Source: `ANLDAQ/gui/scripts/trig_setup_Stage*.sh` + `Serdes_Linkup.sh` (code-verified 2026-04-06)_

---

## softIOC — Global Broadcast PV System (`JustGlobals.db`)

_Source: `ANLDAQ/EPICS/softIOC/db/JustGlobals.db` (14,248 lines, auto-generated)_

The `ANLDAQ/EPICS/softIOC` is a lightweight EPICS soft IOC that runs alongside the GUI. Its primary purpose is hosting `JustGlobals.db` — a **broadcast PV layer** that lets operators set any DIG parameter across all 12 VME crates simultaneously with a single caput.

### How It Works

Each top-level `GLBL:DIG:<preset>_<param>` PV fans out through a chain of `dfanout` records (`GLBL:DIG:F00:`, `F01:`, ...) until reaching `VMExx:GLBL:<param>` on every crate. Writing one PV → all 12 crates updated. The DB contains ~690 `dfanout` records and 69+ top-level broadcast PVs.

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

## Notes


- `EPICS_para.sh` auto-patches `EPICS/softIOC/configure/RELEASE` with the correct `EPICS_BASE` path — never edit that file manually
- PV list (`ioc/All_PV.json`) must be regenerated after any IOC boot script or DB file changes
- The GUI will attempt to auto-source `EPICS_para.sh` if it wasn't sourced manually

---

## See Also

- `dgs/ioc.md` — EPICS IOC boot scripts, DB files, PV definitions
- `dgs/vxworks.md` — VxWorks build pipeline (produces the firmware ANLDAQ talks to)
- `dgs/fpga.md` — DIG/RTRG/MTRG firmware overview
- `dgs/dgs_analysis.md` — Downstream analysis (fastEventConstructor, parquet_pysort) consuming `tcpReceiverMT` output
- `dgs/snapshot_pv.md` — PV snapshot utility (`dumpPVs.py` / `putPVs.py`) invoked by `start_run.sh`
- `dgs/ttcl.md` — TTCL trigger timing (feeds the MTRG TAC-II data decoded in `class_TDC.h`)
- `dgs/DIG_firmware_expert.md` — DIG firmware details; confirms packet format matched by `class_DIG.h`
- `dgs/EPICS_asyn.md` — asyn driver internals: caput/caget flow, port concept, asynUInt32Digital
- `dgs/collectorbox_devicesupport.md` — collector box EPICS device support (SPI driver, CAMAC_IO link)
