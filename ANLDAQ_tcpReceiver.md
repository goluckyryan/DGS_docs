# ANLDAQ — tcpReceiver Detailed Notes

_Split from `ANLDAQ.md` on 2026-04-16. Source: `DGS_tools_pack/ANLDAQ/tcpReceiver/`._

**See also:** [`ANLDAQ.md`](ANLDAQ.md) — parent overview + run control + GUI internals

---

## tcpReceiver — Detailed Notes (updated 2026-04-04)

### Three Executables

| Binary | Source | Mode |
|--------|--------|------|
| `tcpReceiver` | `tcpReceiver.cpp` | Single-threaded, one IOC |
| `tcpReceiverMT` | `tcpReceiverMT.cpp` | Multi-threaded, N IOCs from config file |
| `tcpReceiverUDP` | `tcpReceiverUDP.cpp` | Multi-threaded TCP receiver + UDP forwarding for online analysis. Extends `IOCReceiver`; adds a `UDPSender` per IOC that drains a **4 MB lock-free SPSC ring buffer** and packs data into UDP datagrams (max 1,472 bytes — 1500 MTU − 28 header). Ports: `UDP_BASE_PORT + ioc_index` starting at **12300**. Mode: `--raw` forwards raw TCP bytes; default forwards GEB header + payload. 64-byte fixed header precedes data in each UDP frame. Not the production receiver (`tcpReceiverMT` is); used for online monitoring (e.g. feeding a DAMM/online sort). |✅ verified 2026-04-15 — `tcpReceiverUDP.cpp:L12,L96,L199` (ring buf 4 MB, port 12300, max payload 1472) |

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

- Max file size: **2 GB** ✅ verified 2026-04-12 — `tcpReceiver/constant.h:L7` (`MAX_FILE_SIZE_BYTE = 1024LL*1024*1024*2`)
- Default receive buffer: **1M words = 4 MB** ✅ verified 2026-04-12 — `tcpReceiver/constant.h:L8` (`DEFAULT_DATA_SIZE = 1000000`)
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

**`expInfo.sh` — Experiment Configuration File** *(gitignored, created per experiment)*

All run control scripts source this file. Template (from `start_run.sh` defaults ✅ verified 2026-04-09):
```bash
expName="myExp"                          # Short experiment name (used in filenames)
expFolder="/mnt/data0/exp000000"         # Root folder for this experiment
dataFolder="${expFolder}/data"           # Where run subfolders are created
GEB_ID=14                               # GEB data type (14=DGS, 15=DGSTRIG)
NEXT_RUN=1                              # Auto-incremented by start_run.sh
```
Create a symlink in `dgs_analysis/working/` for use with `ProcessRUN`:
```bash
ln -s ~/ANLDAQ/tcpReceiver/expInfo.sh ~/dgs_analysis/working/expInfo.sh
```

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

**`Aux/` — Offline timing analysis tools** (development/debugging, not used in normal DAQ):
- `script.cpp` (775 lines) — ROOT analysis script correlating TAC-II TDC timestamps with DIG timestamps; studies vernier interpolation precision and phase alignment. Histograms: `hTrigDig` (trigger vs DIG timing), `hPhaseDigVernier` (CFD phase vs vernier value), `hTOF`/`hTOF2` (time-of-flight using zero-crossing vs avg). Implements `ZeroCrossing()` (quadratic + linear interpolation). Builds an output ROOT TTree with TAC/DIG timing pairs.
- `script_LED.cpp` (147 lines) — Same but for LED (leading-edge) mode traces.
- `class_DIG.h` / `class_TDC.h` / `reader.h` — shared decode classes used by both scripts (same API as `fastEventConstructor` but standalone).
- `downloadData.sh` / `readHexFile.sh` — utility scripts for pulling raw hex data from IOC.

**`run_control_gui.py`** — Standalone Tkinter run control GUI for `dgs4`:
- Runs on `dgs4` (shebang: `/home/dgs/.conda/envs/py3tk/bin/python3`); SSHes to `dcsu@dcs2.onenet` using `~/.ssh/id_rsa`
- Reads experiment info (name, next run number, folder) via `expInfo.sh` on dcs2 at startup + on demand
- **Start Run**: streams `start_run.sh` output line-by-line via SSH Popen; maps output substrings to friendly status messages (e.g. "Taking PV Snapshot" → "Taking PV snapshot...", "is running" → "DAQ started!")
- **Stop Run**: similarly streams `stop_run.sh`; includes Parquet sort + elog post status messages
- Live displays: wall clock, elapsed run timer, data folder size (polled every 10 s via `du -sh` on dcs2), recent run log (tails `RunTimestamp.txt`)
- State machine: `idle → starting → running → stopping → idle`; buttons enable/disable accordingly
- ANSI escape codes stripped from SSH output before display
- Script dir on dcs2: `/home/phy/dcsu/ANLDAQ/tcpReceiver/`

**`basic_settings_DGS.sh`** — Sets all DIG channels (CH 5–9) on VME01–VME12 to a known-good starting configuration. Default mode: **CFD** ("Mike CFD values" by M. Carpenter). Key parameters:
- CFD: p1=0.07µs, p2=0.05µs, m=3.5µs, k0=0.56µs, k=0.2µs, d=0.06µs, d3=0.2µs, CFD_fraction=25
- LED: p1=0.07µs, p2=0.05µs, m=2.5µs, k0=0.5µs, k=0.5µs, d=0.16µs
- Both modes: threshold=30, RiseEdge polarity, raw_data_delay=0.5µs, raw_data_length=0.32µs
- Respects VME06/VME10 exception (MDIG1 only; 2 DIGs instead of 4)
- Ends with `Online_CS_StartStop=Stop`, `Online_CS_SaveData=No Save`
- **Note:** `trigger_mux_select` is commented out — trigger mode must be set separately

**`basic_settings_TACII.sh`** — Quick setup script for TAC-II teststand (VME10 only): enables MTRG + MDIG1, clears all vetoes, selects `SumY` trigger monitor, enables `EN_MAN_AUX` trigger (manual/auxiliary). Momentarily pulses `SOFTWARE_VETO` on then off. Used for TAC-II commissioning/testing on the single-crate teststand.

**`simpleStartStop.sh`** — Minimal start/stop wrapper:
```bash
./simpleStartStop.sh 1   # Start: caput SaveData Save; caput StartStop Start
./simpleStartStop.sh 0   # Stop:  caput StartStop Stop; sleep 5; caput SaveData No Save
```
Lightweight alternative to `start_run.sh` / `stop_run.sh` for quick tests without ELOG or parquet sort.

**`copy2Slopebox.sh`** — One-liner: `rsync -av tcpReceiver.cpp Makefile constant.h dgs@slopebox:/global/ioc/dgsReceiver/.` — copies TCP receiver source to the `slopebox` host's IOC receiver directory. Used to deploy receiver code updates to the slopebox machine. ✅ verified 2026-04-12 — `ANLDAQ/tcpReceiver/copy2Slopebox.sh`

**`legacy/` — Older single-IOC receivers** (pre-MT era, kept for reference):
- `dgsReceiver.cpp` (v6.57, 2,633 lines, by Michael Oberling) — original single-threaded receiver, connects to a single IOC at `argv[1]:9001`. Supports cross-platform compile (Linux + Windows/CodeBlocks with `ws2_32`). Build configuration controlled by `#define` switches at top of file:
  - `WRITEGTFORMAT` — wraps data in GEB event header (replaces `0xAAAAAAAA` SOE word with GEB header)
  - `FILE_PER_CHANNEL` (default on) — one output file per digitizer channel; alternatively one-per-DIG or one-per-IOC (`SINGLE_FILE`)
  - `FOLDER_PER_RUN` — creates `runName/` subdirectory per run
  - `DUMP_UNKNOWN_DATA_TO_DISK` + `DEBUG_OUTPUT_FILE` — write unrecognized packets and trigger debug data to separate files
  - `SINGLESHOT`/`FULL_FILE_MODE` — exit automatically when output file(s) reach a size limit
  - SOE constants: `DIG_SOE=0xAAAAAAAA`, `TRIG_SOE=0xAAAA0000`, `MAXCHID=16`, `MAXBOARDID=4095`, `INBUFSIZE=64KB`
  - GEB type IDs hard-coded from `torben`'s list: `GEB_TYPE_DGS=14`, `GEB_TYPE_DGSTRIG=15`, plus others (DECOMP=1, RAW=2, TRACK=3, S800=5/9, GODDESS=19, XA=23, etc.)
  - Output file naming: `runName/runName.prefix_NNN` (per channel) or `runName.prefix_NNN` (flat)
  - Trigger (TRIG) packets are reformatted from 16 raw words → 12 words before saving
- `dgsReceiver_Ryan.cpp` (v6.57 fork, 1,567 lines) — Ryan's experimental fork (2025-05-21). Differences vs MBO's version:
  - Linux-only (Windows `#ifdef` blocks removed)
  - Adds `const bool SAVE_TYPE_F = false;` — stub for controlling type-F header save behavior (TODO: not yet implemented)
  - Otherwise identical build configuration and protocol
- `dgsReceiver.h` (67 lines) — C API header for the classic receiver library: `initReceiver()`, `getReceiverData()`, `getEvent()`, `stopReceiver()`. Defines `SERVER_PORT=9001`, `INBUFSIZE=64KB`, `EVENT_MARKER=0xAAAAAAAA`
- `psNet.h` (68 lines) — Shared network protocol layer header (server + client). Defines `evtServerRetStruct` (4 words: type, recLen, status, recs), `reqPacket`, `incoming`. Constants: `CLIENT_REQUEST_EVENTS=1`, `SERVER_NORMAL_RETURN=2`, `SERVER_SENDER_OFF=3`, `SERVER_SUMMARY=4`, `INSUFF_DATA=5`, `TARGSIZE=7168`, `SENDER_PORT=1101` (UDP sender), `SERVER_PORT=9001`. The 4-word reply header used by `dgsReceiver.h` is the `evtServerRetStruct` struct from this file.
- `PyReceiver.py` (141 lines, by Ryan) — Python proof-of-concept receiver connecting to a single IOC at `argv[1]:argv[2]`. Sends a 4-byte big-endian request (`struct.pack(">I", 1)`), receives the 16-byte `evtServerRetStruct` reply, then recv-loops for all data bytes. Decodes events using `class_DIG` (from `Aux/class_DIG.h`). Handles `SERVER_SUMMARY` (type 4) and `INSUFF_DATA` (type 5) reply types. Validates header type and detects end-of-run via channel_id `0xD` in non-7/8 header packets. Useful for debugging single-IOC data streams from Python without compiling C++.

**`udp_testing/` — UDP experimental infrastructure** (not production-used):
- `udpReceiver.cpp` — C++ UDP receiver variant for testing UDP-based data delivery
- `iocSimulator.py` — Python script that simulates an IOC's TCP server (port 9001) for receiver testing without live VME hardware
- `test_udp.sh` — shell script to exercise the UDP receiver pipeline
- These files support development/testing of `tcpReceiverUDP.cpp` (the multi-threaded variant that forwards data via UDP to an online analysis process in addition to writing to disk)

_Source: `ANLDAQ/tcpReceiver/legacy/` + `udp_testing/` (code-read 2026-04-13)_

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


## See Also

- `knowledgeBase/ANLDAQ.md` — parent overview, VxWorks pipeline, EPICS config
- `knowledgeBase/data_structures.md` — GEB header format + DIG event packet layout
- `knowledgeBase/dgs_analysis.md` — downstream analysis consuming tcpReceiverMT output
