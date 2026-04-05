# ANLDAQ — Front-End DAQ GUI & Data Receiver

## What It Is

ANLDAQ is the **front-end operator interface** for the DGS (and other ANL detector) DAQ systems. It provides:
- A **PyQt6 GUI** (`commander.py`) for configuring and controlling detector electronics (DIG digitizers, RTRG routers, MTRG master trigger) via EPICS
- A **multi-threaded C++ TCP receiver** (`tcpReceiverMT`) that collects raw binary data from VME IOCs during runs via TCP (port 9001)

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

| File | Function |
|------|----------|
| `gui_DIG.py` | Digitizer board configuration window |
| `gui_RTR.py` | Router trigger board window |
| `gui_MTRG.py` | Master trigger board window |
| `gui_DataTaking.py` | Run control (start/stop, folder, name, duration) |
| `gui_SYS.py` | System status (timestamps, link status, TCP rates) |
| `class_Board.py` | Base board abstraction |
| `class_PV.py` | EPICS PV abstraction |
| `json2pv.py` | Converts `All_PV.json` to PV objects |

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
| DGS | 5064 | 5065 | 192.168.203.186, 192.168.203.91 | .141–.145, .177–.183 (12 VMEs) | ✅ verified 2026-04-05 — `ANLDAQ/EPICS_para.sh:45-46` |
| DFMA | 5068 | 5069 | — | — |
| DXA | 5072 | 5073 | 192.168.203.47 | .212, .213 |
| SlopeBox | 5074 | 5075 | 192.168.203.139 | — |
| DUB | 5078 | 5079 | — | — |
| DUO | 5080 | 5081 | 192.168.203.54 | 192.168.203.81 |

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

**`run_control_gui.py`** (new in latest pull):
- Tkinter GUI running on `dgs4`, SSHes to `dcsu@dcs2.onenet`
- Start/Stop buttons, live output, run timer, data size display, recent run log
- SSH streams `start_run.sh` / `stop_run.sh` output in real time
- Auto-refreshes every 15 s during a run

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

## Notes

- `EPICS_para.sh` auto-patches `EPICS/softIOC/configure/RELEASE` with the correct `EPICS_BASE` path — never edit that file manually
- PV list (`ioc/All_PV.json`) must be regenerated after any IOC boot script or DB file changes
- The GUI will attempt to auto-source `EPICS_para.sh` if it wasn't sourced manually
