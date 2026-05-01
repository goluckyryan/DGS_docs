# ANLDAQ — tcpReceiver Detailed Notes

Stability: C2 - Active / semi-stable

_Split from `ANLDAQ.md` on 2026-04-16. Source: `DGS_tools_pack/ANLDAQ/tcpReceiver/`._

**See also:** [`ANLDAQ.md`](ANLDAQ.md) — parent overview + run control + GUI internals

---

## Table of Contents

- [tcpReceiver — Detailed Notes](#tcpreceiver--detailed-notes-updated-2026-04-04)
  - [Three Executables](#three-executables)
  - [TCP Protocol](#tcp-protocol--verified-from-source-code)
  - [Data Flow](#data-flow)
  - [Key Constants](#key-constants-constanth)
  - [Output File Naming](#output-file-naming-outfilenewfile)
  - [TRIG Packet Repacking](#trig-packet-repacking-16-raw-words--10-words)
  - [tcpReceiverMT — Multi-Thread In-Place Display](#tcpreceivermt--multi-thread-in-place-display-mtmode)
  - [GEB Header](#geb-header-define-enable_geb_header)
  - [class_DIG.h — DIG Hit Decoder](#class_digh--dig-hit-decoder)
  - [IOCReceiver — Extensibility](#iocreceiver--extensibility-virtual-hooks)
  - [tcpReceiverUDP — RingBuffer and UDPSender](#tcpreceiverудp--ringbuffer-and-udpsender-internals)
  - [class_TDC.h — TAC-II Hit Decoder](#class_tdch--tac-ii-hit-decoder)
  - [Run Control Scripts](#run-control-scripts)
  - [Packet Consistency: Receiver vs FPGA Firmware](#packet-consistency-receiver-vs-fpga-firmware)
- [Legacy Receiver Code (`tcpReceiver/legacy/`)](#legacy-receiver-code-tcpreceiverlowlegacy)
- [UDP Testing Tools (`tcpReceiver/udp_testing/`)](#udp-testing-tools-tcpreceiverudp_testing)
- [Run Control Scripts — start_run.sh, stop_run.sh, run_control_gui.py](#run-control-scripts--start_runsh-stop_runsh-run_control_guipy)
  - [Overview](#overview)
  - [expInfo.sh — Experiment Configuration](#expinfosh--experiment-configuration)
  - [start_run.sh — Run Start Sequence](#start_runsh--run-start-sequence)
  - [stop_run.sh — Run Stop Sequence](#stop_runsh--run-stop-sequence)
  - [run_control_gui.py — Tkinter GUI](#run_control_guipy--tkinter-gui)
- [See Also](#see-also)

---

## tcpReceiver — Detailed Notes (updated 2026-04-04)

### Three Executables

| Binary | Source | Mode |
|--------|--------|------|
| `tcpReceiver` | `tcpReceiver.cpp` | Single-threaded, one IOC — thin wrapper: parses 4 args (IP, port, dataType, file_prefix), builds one `IOCConfig`, instantiates one `IOCReceiver`, calls `.Run()`. Installs SIGTERM+SIGINT handlers via `stopRequested` atomic flag. ✅ verified 2026-04-27 — `tcpReceiver.cpp:L1-30` |
| `tcpReceiverMT` | `tcpReceiverMT.cpp` | Multi-threaded, N IOCs from config file |
| `tcpReceiverUDP` | `tcpReceiverUDP.cpp` | Multi-threaded TCP receiver + UDP forwarding for online analysis. Extends `IOCReceiver`; adds a `UDPSender` per IOC that drains a **4 MB lock-free SPSC ring buffer** and packs data into UDP datagrams (max 1,472 bytes — 1500 MTU − 28 header). Ports: `UDP_BASE_PORT + ioc_index` starting at **12300**. Mode flags: `-r` forwards raw TCP bytes (no GEB header); `-g` (default) forwards GEB header + payload. Each UDP frame contains one event record (no separate fixed header — GEB header is the first 16 bytes when in `-g` mode). Note: source code comment mentioning "64-byte fixed header" is aspirational/stale — actual `OnRecord()` pushes `processedData` directly (16-byte GEB + payload). Not the production receiver (`tcpReceiverMT` is); used for online monitoring (e.g. feeding a DAMM/online sort). |✅ verified 2026-04-23 — `tcpReceiverUDP.cpp:L235-238` (`-r`/`-g` flags), `L203-212` (`OnRecord` pushes processedData/rawData directly) |

### TCP Protocol — Verified from Source Code

The IOC↔receiver connection is **TCP (`SOCK_STREAM`)** on **port 9001**. This has been verified across all code generations:

**IOC side ([VxWorks](vxworks_state_machines.md) — `SendReceiveSupport.c`):**
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

> ⚠️ **The wiki (`/gsdaq/DAQ_system`) says "UDP packets" — this is incorrect.** The wiki description (written by J. Anderson, March 2023) is misleading. `SOCK_STREAM` = TCP by definition; UDP would use `SOCK_DGRAM` + no `listen()`/`accept()`. `tcpReceiverUDP` is a separate experimental variant, not the production receiver. ✅ verified 2026-04-23 — `vxworks/dgsDrivers/dgsDriverApp/src/SendReceiveSupport.c:L120,L134,L196,L220`; `ANLDAQ/tcpReceiver/receiver.h:L261,L267`

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
    │     SOE overwritten to 0xAAAAAAAA (DIG format!)
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
- TRIG raw packet: **16 words** → repacked to **10 words** ✅ verified 2026-04-16 — `tcpReceiver/constant.h:L4-5` (`TRIG_DATA_SIZE=16`, `TRIG_PACKET_LENGTH=10`)

### Output File Naming (`OutFile::NewFile()`)

Files are named `<runName>_<NNN>_<BBBB>_<C>` where:
- `NNN` — rolling file counter (starts at 0; auto-increments when file hits 2 GB limit) ✅ verified 2026-04-25 — `receiver.h:L83-98`
- `BBBB` — `board_id` zero-padded to 4 digits
- `C` — `ch_id` as a single hex digit (0–9 or A–F)
- **Exception:** if `ch_id == 10` (decimal, i.e. `0xA`; the trigger channel), the suffix is `_T` instead of `_A` ✅ verified 2026-04-25 — `receiver.h:L83-87`

Examples:
- DIG channel 5 on board 141: `myRun_000_0141_5`
- Trigger (TRIG) channel (board 99): `myRun_000_0099_T`

File split: when `fileSize > MAX_FILE_SIZE_BYTE` (2 GB), the current file is closed, `count` is incremented, and a new file is opened — output is `myRun_001_0141_5`, etc.

File tracking: files are indexed in `outFileMap` by key `board_id * 100 + ch_id`. Files are set **read-only** (`chmod 444`) on close. ✅ verified 2026-04-25 — `receiver.h:L139-147`

### TRIG Packet Repacking (16 raw words → 10 words)

The raw TRIG packet (`0xAAAA0000` header, 16 words after `ntohl`) is repacked into a 10-word DIG-compatible payload before saving. Word mapping ✅ verified 2026-04-25 — `receiver.h:L448-466`:

| Repacked Word | Content | Source Raw Words |
|--------------|---------|------------------|
| `payload[0]` | `0xAAAAAAAA` (SOE — replaces `0xAAAA0000`) | hardcoded |
| `payload[1]` | `ch_id=0xA \| board_id(99)<<4 \| TRIG_PACKET_LENGTH(10)<<16` | hardcoded |
| `payload[2]` | `header[4] \| header[3]<<16` | raw W4, W3 |
| `payload[3]` | `header[2] \| header_type(0xE)<<16 \| 3<<26` | raw W2 |
| `payload[4]` | `(header[1]<<16) + header[5]` | raw W1, W5 |
| `payload[5]` | `(header[6]<<16) + header[7]` | raw W6, W7 |
| `payload[6]` | `(header[8]<<16) + header[9]` | raw W8, W9 |
| `payload[7]` | `(header[10]<<16) + header[11]` | raw W10, W11 |
| `payload[8]` | `(header[12]<<16) + header[13]` | raw W12, W13 |
| `payload[9]` | `(header[14]<<16) + header[15]` | raw W14, W15 |

The repacked payload mimics DIG format: `payload[0]` starts with `0xAAAAAAAA`, and `payload[3]` embeds `header_type=0xE` and `3<<26` (HDR_LEN=3). The fake board_id is **99**, ch_id is **0xA** (10), which is why TRIG files appear as `_0099_T`.

### `tcpReceiverMT` — Multi-Thread In-Place Display (`mtMode`)

When `mtMode=true` (set by `tcpReceiverMT` for all threads), each `IOCReceiver` thread uses ANSI cursor escape codes to update its status on a **fixed terminal line** instead of scrolling:
```
\033[<N>A    ← move cursor up N lines (to this thread's reserved line)
\r\033[K    ← carriage return + erase line
... status line ...
\033[<N>B    ← move cursor back down N lines
\r            ← return to base
```
N = `totalThreads - threadIdx` — so thread 0 is the bottom-most, thread N-1 is the top. Each thread has exactly one reserved display line, pre-printed at startup by `tcpReceiverMT` main as `[<IP>] (connecting...)`. The per-line format is:
```
[%-15s] %7.3f MB | %d B | %ld s | empty: %d
```
(IP, total file MB, bytes-received-since-last-print, elapsed seconds, specialCount)

All cursor movements and status writes are protected by `printMutex`. ✅ verified 2026-04-25 — `receiver.h:L581-589`, `tcpReceiverMT.cpp:L68-71`

### GEB Header (`#define ENABLE_GEB_HEADER`)

Each event is prepended with a 16-byte GEB header before writing:
```
int32_t  type       ← GEB_ID from expInfo.sh (e.g. 14)
int32_t  length     ← packet length in bytes
uint64_t timestamp  ← 48-bit event timestamp from DIG header
```
This is the format consumed by downstream analysis ([fastEventConstructor / parquet_pysort](dgs_analysis.md)).

### `class_DIG.h` — DIG Hit Decoder

Decodes the full DIG event packet header (words 0–13 + optional trace words):
- **Header type 7** = LED mode — value comes directly from FPGA (Word 3 bits 19:16); types 7 & 8 adopted in Aug 2021, replacing prior types 5 & 6 (which replaced 3 & 4, which replaced 1 & 2 pre-May 2015) ✅ verified 2026-04-19 — `tcpReceiver/Aux/class_DIG.h:L248`, `knowledgeBase/DIG_firmware_expert.md:L289`
- **Header type 8** = CFD mode — same FPGA-assigned value; both extracted from raw hardware packet, not software-remapped
- Fields: `EVENT_TIMESTAMP`, `PRE/POST_RISE_ENERGY`, `SAMPLED_BASELINE`, `PEAK_TIMESTAMP`, `PILEUP_FLAG`, CFD samples (0/1/2), vernier timestamps, trace waveform
- **Trace/waveform extraction** (added commit 570092e): `DecodeHeader_7_8(raw, dataLen)` accepts optional `dataLen > 14` to extract waveform samples from words 14+. Each 32-bit word encodes two 14-bit unsigned samples: bits 13:0 = even sample, bits 29:16 = odd sample. Stored in `std::vector<unsigned short> trace`. `PrintTrace()` dumps all samples with index, decimal, and hex. ✅ verified 2026-04-18 — `tcpReceiver/Aux/class_DIG.h:L89-90,L225-232,L359-363` (commit 570092e)

### `IOCReceiver` — Extensibility (Virtual Hooks)

`IOCReceiver` provides three virtual hooks for subclasses (used by `tcpReceiverUDP`):
- `OnRecord(processedData, processedLen, rawData, rawLen)` — called after every successfully written event (DIG or TRIG). For DIG: `processedData` = GEB header + DIG payload; `rawData` = original TCP word (including `0xAAAAAAAA` SOE). For TRIG: `processedData` = repacked 10-word payload; `rawData` = original 16 raw TRIG words.
- `OnRunStart()` — called after TCP connection is established, before the data loop begins.
- `OnRunEnd()` — called after `TypeD_RunIsDone` or `stopRequested`, before files are closed.

`tcpReceiverUDP` overrides `OnRecord()` to push each event into its per-IOC `UDPSender` ring buffer for live forwarding to an online sort process. ✅ verified 2026-04-17 — `receiver.h:L234-238` (virtual hooks), `tcpReceiverUDP.cpp` (override)

### `tcpReceiverUDP` — `RingBuffer` and `UDPSender` Internals

**`RingBuffer` (SPSC, 4 MB, lock-free):** ✅ verified 2026-04-26 — `tcpReceiverUDP.cpp:L14-79`
- Fixed 4 MB (`4 * 1024 * 1024` bytes) in-class `char buffer[CAPACITY]`
- `head` (atomic, producer-side) and `tail` (atomic, consumer-side) — true lock-free SPSC with `memory_order_relaxed`/`memory_order_acquire`/`memory_order_release`
- `Push(data, len)`: writes 4-byte little-endian length prefix then `len` bytes; wraps modulo CAPACITY; rejects if not enough free space (returns `false`)
- `Pop(out, maxLen)`: reads 4-byte length, copies data; if `len > maxLen` the record is **silently discarded** (tail advanced past the data) — no error reported
- Both `Push`/`Pop` handle circular wrap byte-by-byte

**`UDPSender` — packing and flushing:** ✅ verified 2026-04-26 — `tcpReceiverUDP.cpp:L82-200`
- One `UDPSender` per IOC thread; socket is `SOCK_DGRAM`; dest = `udpDestIP:UDP_BASE_PORT+ioc_index`
- Internal `sendBuf[1472]` + `sendBufLen` accumulate records greedily until the buffer would overflow:
  - If `sendBufLen + len <= 1472`: `memcpy` record into `sendBuf`, advance `sendBufLen`
  - Else: `Flush()` (send current buffer), start new buffer with this record
- Records larger than 1,472 bytes are silently dropped in `Push()` (`len > MAX_UDP_PAYLOAD` check)
- `SenderLoop()` runs until `running=false` AND ring is empty:
  - Drains ring into UDP frames; if no data was popped: `Flush()` + `usleep(100)` (100 µs idle wait)
  - Final `Flush()` after loop to send any partial buffer
- Thread lifecycle: `Start()` sets `running=true` and launches `senderThread`; `Stop()` sets `running=false` + joins
- `OnRunStart()` calls `udpSender.Start()`; `OnRunEnd()` calls `udpSender.Stop()` — guarantees ring is fully drained before files close

**Key implication:** multiple event records can be packed into a single UDP datagram (bin-packed up to 1,472 bytes). The receiver side must re-split them by reading the packed stream — there is no per-record delimiter in the UDP payload; records are simply concatenated.

### `class_TDC.h` — TAC-II Hit Decoder

Decodes the MTRG [TAC-II](tac2.md) packet (10 words after repacking):
- `timestampTrig` — MTRG 48-bit trigger timestamp (×10 ns) ✅ verified 2026-04-22 — `class_TDC.h:L189,L207` (`timestampTrig = (((uint64_t) data[2] & 0xFFFF) << 32) + data[1]; ... timestampTrig *= 10;`)
- `coarseTS` — 16-bit coarse TDC counter ✅ verified 2026-04-22 — `class_TDC.h:L111,L194` (`uint16_t coarseTS`; `coarseTS = data[5] >> 16;`)
- Four-phase 4 ns counters (0°/90°/180°/270°) + vernier AB/CD (6 bits each, ~50 ps/step) ✅ verified 2026-04-22 — `class_TDC.h:L52` (`vernierAB & 0x3F` = 6 bits); `class_TDC.h:L228` (`0.05 * tdcData.vernier[i]` = 50 ps/step)
- `CalTAC_simple()` — computes average phase timestamp in ns with ~50 ps resolution ✅ verified 2026-04-22 — `class_TDC.h:L214-255` (averages valid phaseTime[i] values using 50 ps/LSB vernier)
- Trash data detection: specific counter pattern `0x1006/1005/1004/1003` ✅ verified 2026-04-22 — `class_TDC.h:L199-202` (exact `== 0x1006/0x1005/0x1004/0x1003` comparison; sets `trashData = true`)

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
Create a symlink in [`dgs_analysis`](dgs_analysis.md) `working/` for use with `ProcessRUN`:
```bash
ln -s ~/ANLDAQ/tcpReceiver/expInfo.sh ~/dgs_analysis/working/expInfo.sh
```

**`start_run.sh`:**
1. Sources `expInfo.sh` (experiment name, folder, GEB_ID, run number)
2. Increments `NEXT_RUN` in expInfo.sh
3. Creates `dataFolder/expName_RRR/`
4. Runs `~/snapshot_pv/dumpPVs.py` → **[snapshot_pv](snapshot_pv.md) lives on dcs2.onenet, not pi5-dgs**
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

**`sync_exp_data.sh`** — Live data rsync daemon (sources `expInfo.sh` for paths):
- Reads `expName`, `dataFolder`, `nfsFolder` from `expInfo.sh` at startup
- Monitors `dataFolder` + `expInfo.sh` with `inotifywait` (falls back to polling every `MIN_SYNC_GAP=20` s)
- **New-run detection**: polls `NEXT_RUN` field in `expInfo.sh`; on increment, triggers immediate rsync of that run's subfolder
- **Debounced event-driven sync**: on inotifywait event, only syncs if ≥20 s since last sync
- **Active-run rsync**: scoped to current run subfolder with `rsync -aO --no-perms --append` (append-only since `tcpReceiverMT` never rewrites data)
- **Boundary/cleanup sync**: full `dataFolder/` → `nfsFolder/data/` when no specific run is active
- **Staleness sentinel**: touches `nfsFolder/data/.last_sync` after each periodic sync to track freshness
- **RunTimestamp.txt**: always synced alongside data so NFS has the latest run log
- Final sync on Ctrl+C / SIGTERM before exit ✅ verified 2026-04-21 — `ANLDAQ/tcpReceiver/sync_exp_data.sh` (full script review)

**`Aux/` — Offline timing analysis tools** (development/debugging, not used in normal DAQ):
- `script.cpp` (775 lines) — ROOT analysis script correlating TAC-II TDC timestamps with DIG timestamps; studies vernier interpolation precision and phase alignment. Histograms: `hTrigDig` (trigger vs DIG timing), `hPhaseDigVernier` (CFD phase vs vernier value), `hTOF`/`hTOF2` (time-of-flight using zero-crossing vs avg). Implements `ZeroCrossing()` (quadratic + linear interpolation). Builds an output ROOT TTree with TAC/DIG timing pairs.
- `script_LED.cpp` (147 lines) — Same but for LED (leading-edge) mode traces.
- `class_DIG.h` / `class_TDC.h` — shared decode classes used by both scripts (same API as `fastEventConstructor` but standalone).
- **`reader.h`** (255 lines) — `class Reader`: standalone offline file reader for DGS binary data files. Wraps `FILE*` with block-level navigation; holds one `TDC_Hit` (TAC mode) and one `DIG_Hit` (DIG mode) instance that are reused across calls. Key design:
  - **Constructor**: `Reader(fileName, isTAC=true, isGEB=false)` — opens file, `fseek`s to end to record `inFileSize`, rewinds.
  - **`ReadNextBlock(fastRead, debug)`**: reads one trigger packet at a time. Expects `0xAAAAAAAA` as the first word (sync marker). If `isGEB=true`, reads and discards a 4-word GEB header first, then forces `firstWord=0xAAAAAAAA`. In TAC mode, `packageLen = TRIG_PACKET_LENGTH` (constant from `constant.h`). In DIG mode, peeks at word 2 bits 26:16 (`packageLen = ((ntohl(word2) >> 16) & 0x3FF) + 1`) then rewinds 1 word. In `fastRead` mode, `fseek` skips payload without decoding. In full mode: TAC → `TDC_Hit::FillTDC()` + `CalTAC_simple()`; DIG → `ntohl()` each word + `DIG_Hit::DecodeHeader_7_8()`. Returns `0` on success, `-1` on sync error, `-10` on EOF.
  - **`ScanNumBlock()`**: fast-reads the entire file in `fastRead=true` mode, records each block's file offset in `blockPos[]`, sets `totNumBlock`. Call once before `ReadBlock(index)`.
  - **`ReadBlock(index)`**: seeks to `blockPos[index]` and calls `ReadNextBlock()` — enables random-access into any block by index after a `ScanNumBlock()` pass.
  - **`PrintPayLoad()`**: dumps raw `payload[]` vector (populated only in TAC mode) as hex.
  - DIG data: each word is `ntohl()`-converted on read; TAC data: stored as-is from `fread` (no byte-swap).
- **`downloadData.sh`** — `rsync`s a run's binary data files from `slopebox:/global/ioc/dgsReceiver/data/<prefix>*` into `../data/`. Takes one argument (file prefix/run ID). Commented-out lines show prior test variants (`haha*`, `XXXX*`). Uses `slopebox` as the NFS/rsync source hostname.
- **`readHexFile.sh`** — Dumps the first N 32-bit words of a binary file as hex, one word per line with a 6-digit decimal index. Usage: `readHexFile.sh <binary_file> <num_words>`. Internally: `hexdump -n (N×4) -v -e '1/4 "%08X\n"' file | awk '{printf "%06d: 0x%s\n", NR-1, $1}'`. Useful for inspecting raw packet headers.

**`run_control_gui.py`** — Standalone Tkinter run control GUI for `dgs4`:
- Runs on `dgs4` (shebang: `/home/dgs/.conda/envs/py3tk/bin/python3`); SSHes to `dcsu@dcs2.onenet` using `~/.ssh/id_rsa`
- Reads experiment info (name, next run number, folder) via `expInfo.sh` on dcs2 at startup + on demand
- **Start Run**: streams `start_run.sh` output line-by-line via SSH Popen; maps output substrings to friendly status messages (e.g. "Taking PV Snapshot" → "Taking PV snapshot...", "is running" → "DAQ started!")
- **Stop Run**: similarly streams `stop_run.sh`; includes Parquet sort + elog post status messages
- Live displays: wall clock, elapsed run timer, data folder size (polled every 15 s via `du -sh` on dcs2), recent run log (tails `RunTimestamp.txt`) ✅ verified 2026-04-17 — `run_control_gui.py:L257` (`self.after(15000, self._auto_refresh)`)
- State machine: `idle → starting → running → stopping → idle`; buttons enable/disable accordingly
- Blank comment auto-expands to `no comment`; otherwise comment text is passed through after `strip()` ✅ verified 2026-04-22 — `run_control_gui.py:L259-260`
- Data-size polling targets `<dataFolder>/<expName>_<NEXT_RUN-1 padded to 3 digits>/` every 15 s during the run ✅ verified 2026-04-22 — `run_control_gui.py:L241-256`
- ANSI escape codes stripped from SSH output before display ✅ verified 2026-04-22 — `run_control_gui.py:L16,L44`
- Script dir on dcs2: `/home/phy/dcsu/ANLDAQ/tcpReceiver/`

**`basic_settings_DGS.sh`** — Sets all DIG channels (CH 5–9) on VME01–VME12 to a known-good starting configuration. Default mode: **CFD** ("Mike CFD values" by M. Carpenter). Key parameters:
- CFD: p1=0.07µs, p2=0.05µs, m=3.5µs, k0=0.56µs, k=0.2µs, d=0.06µs, d3=0.2µs, CFD_fraction=25
- LED: p1=0.07µs, p2=0.05µs, m=2.5µs, k0=0.5µs, k=0.5µs, d=0.16µs
- Both modes: threshold=30, RiseEdge polarity, raw_data_delay=0.5µs, raw_data_length=0.32µs
- Respects VME06/VME10 exception (MDIG1 only; 2 DIGs instead of 4)
- Ends with `Online_CS_StartStop=Stop`, `Online_CS_SaveData=No Save`
- **Note:** `trigger_mux_select` is commented out — trigger mode must be set separately

**`basic_settings.sh`** — Single-crate teststand setup script (VME99, MDIG1, CH=7 only). The teststand analogue of `basic_settings_DGS.sh` for benchtop/single-digitizer work. Key differences from the production DGS version:
- Targets `VME99` (test crate alias) and `MDIG1` only — no loop over multiple crates
- CH=7 hardcoded (single channel)
- MTRG settings included: `IMP_SYNC` pulse, `SOFTWARE_VETO on`, `ENBL_MON7_VETO on`, `SUM_OF_Y/X_THRESH=0`, `EN_NIM1_DELAY=N`, `EN_SUM_X on`, `CS_Ena=Enable`, `FifoNum=MAIN DATA FIFO`, `SYSMON_ENABLE=ON`, `TRIG_MON_SEL=${trigger}` (default: `SumX`), FIFO reset pulse
- CFD parameters: p1=0.07µs, p2=0.05µs, m=2.5µs, k0=0.56µs, k=0.16µs (fixed), d=0.1µs (fixed), CFD_fraction=50 — **note:** different CFD values from the DGS production version (m=2.5 not 3.5; fraction=50 not 25)
- LED parameters: p1=0.07µs, p2=0.05µs, m=2.5µs, k0=0.5µs, k=0.5µs, d=0.16µs (identical to DGS version)
- Both modes: threshold=30, RiseEdge polarity, raw_data_delay=0.5µs, raw_data_length=4.0µs (larger than DGS 0.32µs — longer trace for diagnostic use)
- Sets `trigger_mux_select IntAcptAll` explicitly (DGS version has this commented out)
- Ends with `CS_Ena=Enable`, `veto_enable=0`, FIFO reset
- Comment at top: `#todo, NIM or RAM` (trigger mode placeholder)
✅ verified 2026-04-27 — `ANLDAQ/tcpReceiver/basic_settings.sh:L1-76`

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
  - Trigger (TRIG) packets are reformatted from 16 raw words → **10 words** before saving (the internal `reformatted_hdr[]` array is 12 words but only 10 are written, starting from `reformatted_hdr[1]`; `packet_length_in_words=10` at L1657) ✅ verified 2026-04-16 — `legacy/dgsReceiver.cpp:L126,L1657,L2079-2100`
- `dgsReceiver_Ryan.cpp` (v6.57 fork, 1,567 lines) — Ryan's experimental fork (2025-05-21). Differences vs MBO's version:
  - Linux-only (Windows `#ifdef` blocks removed)
  - Adds `const bool SAVE_TYPE_F = false;` — stub for controlling type-F header save behavior (TODO: not yet implemented in this fork; the feature was never ported to production `dgsReceiver.cpp`) ✅ verified 2026-04-20 — `legacy/dgsReceiver_Ryan.cpp:L49`; grep of production `tcpReceiver/*.cpp` confirms no `SAVE_TYPE_F` in production code
  - Otherwise identical build configuration and protocol
- `dgsReceiver.h` (67 lines) — C API header for the classic receiver library: `initReceiver()`, `getReceiverData()`, `getEvent()`, `stopReceiver()`. Defines `SERVER_PORT=9001`, `INBUFSIZE=64KB`, `EVENT_MARKER=0xAAAAAAAA`
- `psNet.h` (68 lines) — Shared network protocol layer header (server + client). Defines `evtServerRetStruct` (4 words: type, recLen, status, recs), `reqPacket`, `incoming`. Constants: `CLIENT_REQUEST_EVENTS=1`, `SERVER_NORMAL_RETURN=2`, `SERVER_SENDER_OFF=3`, `SERVER_SUMMARY=4`, `INSUFF_DATA=5`, `TARGSIZE=7168`, `SENDER_PORT=1101` (UDP sender), `SERVER_PORT=9001`. The 4-word reply header used by `dgsReceiver.h` is the `evtServerRetStruct` struct from this file.
- `PyReceiver.py` (141 lines, by Ryan) — Python proof-of-concept receiver connecting to a single IOC at `argv[1]:argv[2]`. Sends a 4-byte big-endian request (`struct.pack(">I", 1)`), receives the 16-byte `evtServerRetStruct` reply, then recv-loops for all data bytes. Decodes events using `class_DIG` (from `Aux/class_DIG.h`). Handles `SERVER_SUMMARY` (type 4) and `INSUFF_DATA` (type 5) reply types. Validates header type and detects end-of-run via channel_id `0xD` in non-7/8 header packets. Useful for debugging single-IOC data streams from Python without compiling C++.

**`udp_testing/` — UDP experimental infrastructure** (not production-used):
- `udpReceiver.cpp` — C++ UDP receiver variant for testing UDP-based data delivery
- `iocSimulator.py` — Python script simulating a real IOC TCP server (port 9001) for receiver testing without live VME hardware. Generates **real DIG LED (HEADER_TYPE=7) packets** with proper bit-packed fields matching `class_DIG.h` decoding (commit 570092e, 2026-03-24). Wire format: `0xAAAAAAAA` marker + packet_length_in_words data words in network byte order. Usage: `python3 iocSimulator.py [port] [num_batches] [num_trace_words]` (defaults: 9001, 100, 4). Supports optional trace/waveform appended after word 13: configurable number of trace words; each word encodes **two 14-bit unsigned ADC samples** (bits 13:0 = sample N, bits 29:16 = sample N+1). ✅ verified 2026-04-18 — `tcpReceiver/udp_testing/iocSimulator.py` (commit 570092e)
- `test_udp.sh` — shell script to exercise the UDP receiver pipeline (hexdump + ROOT decode step + trace word count arg added in commit 570092e)
- These files support development/testing of `tcpReceiverUDP.cpp` (the multi-threaded variant that forwards data via UDP to an online analysis process in addition to writing to disk)

_Source: `ANLDAQ/tcpReceiver/legacy/` + `udp_testing/` (code-read 2026-04-13; iocSimulator/trace updated 2026-04-18)_

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
- LED = `7` (`0111`), CFD = `8` (`1000`) — encoded **directly by the FPGA** into the packet header (`cHEADER_TYPE_LED = to_unsigned(7,4)`, `cHEADER_TYPE_CFD = to_unsigned(8,4)`) ✅ verified 2026-04-22 — `Event_Header_FIFO.vhd:L101-102,L326,L419`
- There is **no IOC re-encoding** — the FPGA writes 7/8 directly; the IOC DMA-copies raw VME FIFO data to TCP unchanged. The IOC simply DMA-transfers whatever the FPGA put in its event header FIFO directly onto the wire.
- Prior note claiming "FPGA hardware: LED=4, CFD=5" and "IOC re-encodes" was incorrect and has been removed.

**TRIG packet:** MTRG sends 16 words raw (`0xAAAA0000` header). Receiver repacks to 10 words. `class_TDC.h::FillTDC()` correctly decodes trigger timestamp, trigger type, wheel, multiplicity, coarse TDC, trigger bitmask, four 4 ns phase counters, and vernier AB/CD — all consistent with the TAC-II data path in the [MTRG Main FPGA firmware](deep_fpga_MTRG_MAIN.md).

---

## Run Control Scripts — `start_run.sh`, `stop_run.sh`, `run_control_gui.py`

_Source: `ANLDAQ/tcpReceiver/` | Analyzed 2026-04-24_

### Overview

The DAQ run control consists of three components that work together:

| Component | Location | Role |
|-----------|----------|------|
| `start_run.sh` | DCS2: `SCRIPT_DIR` | Shell script: sets up run folder, snapshots PVs, launches receivers, starts EPICS acquisition, posts elog |
| `stop_run.sh` | DCS2: `SCRIPT_DIR` | Shell script: stops acquisition, flushes data, kills receivers, posts elog, launches Parquet sort |
| `run_control_gui.py` | dgs4 | Tkinter GUI that SSHes into DCS2 to run the above scripts, displays friendly status messages |

### `expInfo.sh` — Experiment Configuration

`start_run.sh` sources `expInfo.sh` in the same directory. All experiment-specific config lives there:

| Variable | Purpose |
|----------|---------|
| `expName` | Experiment name (e.g. `myExp`) |
| `expFolder` | Root folder for experiment (e.g. `/mnt/data0/exp000000`) |
| `dataFolder` | Run subfolders go here (`${expFolder}/data`) |
| `nfsFolder` | NFS sync destination (local path) |
| `nfsPath` | NFS network path (e.g. `fs2.onenet:/mnt/vol5/atlasdata/dgs/exp000000`) |
| `GEB_ID` | GEB data type ID (e.g. `14`) |
| `elogLogbook` | ELOG logbook name |
| `NEXT_RUN` | Auto-incremented run counter; updated by `sed -i` each run |

If `expInfo.sh` is missing, `start_run.sh` creates a template and exits.

### `start_run.sh` — Run Start Sequence

1. **Run folder setup:** creates `${dataFolder}/${expName}_NNN/`; increments `NEXT_RUN` in `expInfo.sh` via `sed -i`
2. **Final adjustments block:** commented `caput` commands for MDIG1/MDIG2 enable, master logic enable, FIFO reset — uncomment as needed per experiment
3. **PV snapshot:** calls [`~/snapshot_pv/dumpPVs.py`](snapshot_pv.md) `--all --skip GS085,GS091,GS099,MOD085,MOD091,MOD099 --outdir <runFolder>` (skips 3 known-bad detectors)
4. **Timestamp log:** appends `YYYYMMDD_HHMMSS, RunNNN, <comment>` to `${expFolder}/RunTimestamp.txt`
5. **ELOG post:** posts start entry to `elog.phy.anl.gov:443` logbook, Category=Run
6. **Receiver launch mode** (`USE_MT=false` default):
   - `USE_MT=false` (default): opens one `gnome-terminal` per IP in `IPList[]` running `./tcpReceiver IP 9001 GEB_ID <outpath>`; two-column layout (6 per col), left col x=0, right col x=1400 ✅ verified 2026-04-24 — `start_run.sh:L9,L179,L191`
   - `USE_MT=true`: opens one `gnome-terminal` running `./tcpReceiverMT config.txt <outpath>`; auto-generates `config.txt` from `IPList[]` if missing ✅ verified 2026-04-24 — `start_run.sh:L149,L153-163`
   - PIDs written to `pidList.txt`
7. **sleep 5:** waits for receivers to connect ✅ verified 2026-04-24 — `start_run.sh:L204`
8. **EPICS start:** `caput Online_CS_StartStop Start` then `caput Online_CS_SaveData Save`
9. **Auto-stop:** if `$waitTime` arg > 5 s, sleeps then auto-stops; otherwise unlimited ✅ verified 2026-04-24 — `start_run.sh:L216,L238`

Current `IPList`: `.141 .142 .143 .144 .145 .177 .178 .179 .180 .183 .181 .182` (12 IOCs = 12 VME crates) ✅ verified 2026-04-24 — `ANLDAQ/tcpReceiver/start_run.sh:L12`

### `stop_run.sh` — Run Stop Sequence

1. `caput Online_CS_StartStop Stop`
2. Appends "Stopped" entry to `RunTimestamp.txt`
3. `sleep 10` — waits for IOC to flush data ✅ verified 2026-04-24 — `stop_run.sh:L29`
4. `caput Online_CS_SaveData No Save` ✅ verified 2026-04-24 — `stop_run.sh:L31`
5. `sleep 5` then calls `kill_IOC.sh` to kill receiver processes ✅ verified 2026-04-24 — `stop_run.sh:L34-35`
6. **ELOG post:** calculates duration from `RunTimestamp.txt` start/stop timestamps; posts run size (`du -sh`) and NFS path ✅ verified 2026-04-24 — `stop_run.sh:L40-67`
7. **Parquet sort:** launches `~/DGS_Analysis/working/RunParquet expInfo.sh <run_num>` in a new `gnome-terminal` ✅ verified 2026-04-24 — `stop_run.sh:L82-83`

### `run_control_gui.py` — Tkinter GUI

Runs on **dgs4**, SSHes into **DCS2** (`dcsu@dcs2.onenet`) using identity file `/home/dgs/.ssh/id_rsa`.

**UI layout:**
- Experiment name + next run number (refreshed by sourcing `expInfo.sh` over SSH)
- Comment text entry (base64-encoded before SSH to survive shell quoting)
- Start Run / Stop Run buttons (mutually exclusive; grayed during transitions)
- Output area (dark background, green text) showing script output mapped to friendly messages
- Clock (HH:MM:SS), run timer (green; starts when "is running" seen), data folder size (polled `du -sh` every 15 s)
- Recent runs log panel (tail -15 from `RunTimestamp.txt`)

**Status message mapping (key substrings in script stdout):**

Start: "Final adjustments" → "Preparing final adjustments...", "Taking PV Snapshot" → "Taking PV snapshot...", "Start Run" → "Setting up run folder...", "tcpReceiver" → "Opening receivers...", "tcpReceiverMT" → "Opening receiver (MT)...", "Online_CS_StartStop Start" → "Starting DAQ...", "Online_CS_SaveData Save" → "Saving data...", "is running" → "DAQ started!"

Stop: "Online_CS_StartStop Stop" → "Stopping DAQ...", "flush data" → "Waiting for IOC to flush data...", "Online_CS_SaveData" → "Stopping data save...", "kill receivers" → "Killing receivers...", "DAQ stopped" → "DAQ stopped.", "Parquet Sort" → "Running Parquet sort...", "elog" → "Posting to elog..."

---

## Legacy Receiver Code (`tcpReceiver/legacy/`)

_Source: `ANLDAQ/tcpReceiver/legacy/`. Code-read 2026-04-27._ ✅ verified 2026-04-27

This directory contains **pre-ANLDAQ legacy TCP receiver code** — the original single-threaded receiver that predates the current `tcpReceiverMT`/`tcpReceiverUDP` architecture. Retained for reference.

### Files

| File | Lines | Description |
|------|-------|-------------|
| `dgsReceiver.h` | 67 | C API header for the legacy receiver library |
| `dgsReceiver.cpp` | 2633 | Legacy single-threaded receiver implementation |
| `dgsReceiver_Ryan.cpp` | 1567 | Ryan's variant/fork of the legacy receiver |
| `psNet.h` | 68 | Network utility macros (legacy platform compat) |
| `PyReceiver.py` | 141 | Python 3 receiver client — connects to IOC TCP server, decodes DIG events |

### `dgsReceiver.h` — Legacy C API

Defines the public API for the old receiver library:
- **`initReceiver(server)`** — connect to IOC at dotted-decimal address; port hardcoded to `9001` ✅ verified 2026-04-27 — `dgsReceiver.h:L11` (`#define SERVER_PORT 9001`)
- **`getReceiverData(instance, len, dat)`** — blocking call; returns data for `len` seconds (range 0.1–1.0); fills contiguous buffer
- **`getEvent(instance, buffer)`** — pop single event from received pool; return codes: `0=OK`, `-4=OUTOFEVENTS`, `-5=BUFFERSMALL`
- **`stopReceiver(instance)`** — close socket
- **`EVENT_MARKER`** — `0xAAAAAAAA` sync word (same as current architecture) ✅ verified 2026-04-27 — `dgsReceiver.h:L40`
- **`INBUFSIZE`** — `64 * 1024` bytes
- **Internal `inbuf` struct**: singly-linked list nodes (`next`, `type`, `evtlen`, `recs`, `*pdata`)

### `PyReceiver.py` — Python 3 TCP Receiver Client

Lightweight Python 3 diagnostic receiver. Connects to an IOC TCP server, requests data in a loop, and decodes DIG events using `class_DIG`. ✅ verified 2026-04-27 — full code-read

**Wire protocol understood by PyReceiver:**
- Request: 4-byte big-endian word (`1` = data request)
- Reply header: 4×`uint32` big-endian: `[reply_type, record_size, status, num_record]`
  - `reply_type=4` → `SERVER_SUMMARY` (normal data)
  - `reply_type=5` → `INSUFF_DATA` (no data yet)
- Payload: `record_size × num_record` bytes, big-endian `uint32` words

**DIG event parsing logic:**
- Iterates word-by-word through the flat payload
- Word 1 (index 1): bits `[26:16]` = `PACKET_LENGTH` (packet boundary marker)
- Word 3 (index 3): bits `[19:16]` = `HEADER_TYPE`
  - Types 7 (`LED`) and 8 (`LED+PILEUP`) are valid event types → decoded via `DIG.decode_data()`
  - Any other header type with `channel_id=0xD` → "run done" signal → stops loop
  - Other unrecognized types → skipped
- After collecting `packet_length` words → calls `dig.decode_data(payload)` + `dig.print()`

**Usage:** `python3 PyReceiver.py [IP] [Port]`

---

## UDP Testing Tools (`tcpReceiver/udp_testing/`)

_Source: `ANLDAQ/tcpReceiver/udp_testing/`. Code-read 2026-04-27._ ✅ verified 2026-04-27

Three tools for end-to-end testing of the `tcpReceiverUDP` pipeline **without live hardware**. Together they simulate a full IOC→receiver→analysis chain using local TCP and UDP sockets.

### Files

| File | Lines | Description |
|------|-------|-------------|
| `iocSimulator.py` | 204 | Fake IOC TCP server — sends synthetic DIG LED events |
| `udpReceiver.cpp` | 168 | UDP listener — prints stats for UDP packets from tcpReceiverUDP |
| `test_udp.sh` | 126 | End-to-end test harness orchestrating all three components |

### `iocSimulator.py` — Fake IOC TCP Server

Simulates a DGS VME IOC digitizer board over TCP. Responds to data requests with synthetic DIG LED (HEADER_TYPE=7) events. ✅ verified 2026-04-27

**Usage:** `python3 iocSimulator.py [port] [num_batches] [num_trace_words]`  
**Defaults:** port=9001, num_batches=100, num_trace_words=4

**Wire protocol:**
- Waits for 4-byte request from receiver per batch
- Sends reply header (4×`uint32` big-endian): `[type=4, total_bytes, status=0, num_record=1]`
- Sends DIG LED record: `0xAAAAAAAA` marker (native endian) + 13 data words (network byte order) + optional trace words

**Synthetic event structure (HEADER_TYPE=7 / LED mode):**
- Word 1: `CH_ID[3:0] | USER_DEF[15:4] | PACKET_LENGTH[26:16] | GEO_ADDR[31:27]`
- Word 2: `EVENT_TIMESTAMP` low 32 bits
- Word 3: `TS_HIGH[15:0] | HEADER_TYPE=7[19:16] | EVENT_TYPE[25:23] | HEADER_LENGTH[31:26]`
- Word 4: `flags | LAST_DISC_TIMESTAMP[31:16]`
- Word 5: `LAST_DISC_TIMESTAMP[47:16]` (full 48-bit LED timestamp, upper 32 bits)
- Word 6: `SAMPLED_BASELINE[23:0] | PILEUP_COUNT[27:24]`
- Word 7: `TRIG_MON_XTRA_DATA[15:0] | TRIG_MON_DET_DATA[31:16]`
- Word 8: `PRE_RISE_ENERGY[23:0] | POST_RISE_ENERGY[31:24]` (low byte)
- Word 9: `POST_RISE_ENERGY[15:0]` (bits 23:8) `| PEAK_TIMESTAMP[31:16]`
- Word 10: `P2_SUM[13:0] | P2_MODE[14] | CAPTURE_PARST_TS[15] | TS_OF_TRIGGER[31:16]`
- Word 11: `MULTIPLEX_DATA[23:0] | LAST_POST_RISE_M_SUM[31:24]` (bits 23:16)
- Word 12: `EARLY_PRE_RISE_ENERGY[23:0] | LAST_POST_RISE_M_SUM[31:24]` (bits 15:8)
- Word 13: `P2_SUM_HIGH[9:0] | flags | TS_OF_COARSE[23:14] | LAST_POST_RISE_M_SUM[31:24]` (bits 7:0)
- Words 14+: optional trace — 2 unsigned 14-bit samples per 32-bit word (`bits[13:0]=sample1`, `bits[29:16]=sample2`); Gaussian-shaped synthetic waveform

**Run-done signal:** After all batches, sends a packet with `channel_id=0xD` and `HEADER_TYPE=0` to signal end-of-run.

### `udpReceiver.cpp` — UDP Packet Inspector

Multi-port UDP listener for diagnostic monitoring of `tcpReceiverUDP` output. ✅ verified 2026-04-27

**Usage:** `./udpReceiver [port]` or `./udpReceiver [port] [num_ports]`
- Multi-port mode: spawns one thread per port (`port` to `port + num_ports - 1`)
- Each thread prints received packets as hex words and prints per-second throughput stats
- **UDP payload format**: first 16 words (64 bytes) = fixed header; remaining = trace data
- Max UDP payload: 1472 bytes (MTU-safe)
- Throughput: printed per 1-second interval as MB/s; total packet count + byte count displayed on stop

### `test_udp.sh` — End-to-End Test Harness

Orchestrates all three components for a local self-contained integration test. ✅ verified 2026-04-27

**Test flow:**
1. Builds `tcpReceiverUDP` and `udpReceiver` via `make`
2. Starts `iocSimulator.py` on TCP port 9001 (16 batches, 4 trace words)
3. Starts `udpReceiver` on UDP port 12300
4. Starts `tcpReceiverUDP` with config `127.0.0.1 9001 8` → `tcpReceiverUDP` connects to fake IOC, receives data, writes to `/tmp/test_udp_run/test_000_*`, forwards via UDP to `127.0.0.1:12300`
5. After `tcpReceiverUDP` exits: hexdumps saved data file; runs ROOT macro using `Aux/reader.h` to decode channel-0 file and call `digHit.PrintTrace()`
6. Cleanup: kills all background processes, removes temp files

**Config file format** (line per IOC): `IP  TCP_PORT  num_channels`

---

## See Also

- [`ANLDAQ.md`](ANLDAQ.md) — parent overview, VxWorks pipeline, EPICS config
- [`ANLDAQ_GUI_windows.md`](ANLDAQ_GUI_windows.md) — gui_DataTaking: GUI front-end that spawns and controls tcpReceiverMT
- [`data_structures.md`](data_structures.md) — GEB header format + DIG event packet layout
- [`dgs_analysis.md`](dgs_analysis.md) — downstream analysis consuming tcpReceiverMT output
- [`run_procedures.md`](run_procedures.md) — operator-level run procedures (uses `start_run.sh`/`stop_run.sh` as the key start/stop mechanism)
- [`gebsort.md`](gebsort.md) — GEBSort/GEBMerge: downstream consumers of the raw GEB files tcpReceiverMT writes; also gtReceiver (Tim Lauritsen's alternative test receiver)
- [`ANLDAQ_tcpReceiver_Aux.md`](ANLDAQ_tcpReceiver_Aux.md) — auxiliary receivers: tcpReceiverSingle, DFMA receiver, auxiliary data streams
- [`tac2.md`](tac2.md) — TAC-II TDC hardware details; timing resolution, packet format, calibration

*Created: 2026-04-14 | Last reviewed: 2026-04-27*
