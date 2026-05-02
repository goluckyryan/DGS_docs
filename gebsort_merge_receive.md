# GEBSort — Data Receivers and Merge Tools

Stability: C2 - Active / semi-stable

_Split from `gebsort.md` 2026-04-26. Source: `DGS_tools_pack/gebsort/`._

This file covers the GEB data pipeline tools: **GEBMerge** (timestamp-merge of multi-crate streams),
**gtReceiver** (DAQ-side data receiver), **GEBClient** (VxWorks IOC → GEB sender), and
**dmpdata** (GEB file dump tool). For the core sorting framework and bin_dgs, see `gebsort.md`.

---

## Table of Contents

- [GEBMerge](#gebmerge)
  - [Command-line syntax](#command-line-syntax)
  - [Algorithm](#algorithm)
  - [Chat file parameters (GEBMerge.chat)](#chat-file-parameters-gebmergechat)
  - [Notes](#notes)
- [gtReceiver — DAQ-Side Data Receiver](#gtreceiver--daq-side-data-receiver)
  - [Versions](#versions)
  - [Build](#build)
  - [Usage](#usage)
  - [What gtReceiver4 does](#what-gtreceiver4-does)
  - [Packet Field Decoding](#packet-field-decoding)
  - [GEB Header Format (written by gtReceiver)](#geb-header-format-written-by-gtreceiver)
  - [Relationship to production tcpReceiverMT](#relationship-to-production-tcpreceivermt)
  - [Online (interactive) mode](#online-interactive-mode)
- [GEBClient — EPICS IOC → GEB Data Sender](#gebclient--epics-ioc--geb-data-sender)
  - [Struct](#struct)
  - [API](#api)
  - [Protocol](#protocol)
  - [Endianness](#endianness)
  - [Where it is used](#where-it-is-used)
- [dmpdata — GEB File Dump Tool](#dmpdata--geb-file-dump-tool)
  - [Usage](#usage-1)
  - [What it prints](#what-it-prints)
  - [Summary at end](#summary-at-end)
- [Cross-References](#cross-references)

---

## GEBMerge

_Source: `gebsort/GEBMerge.c` (1,803 lines). Code-read 2026-04-26._

Merges multiple single-crate GEB data files into one timestamp-sorted stream.

### Command-line syntax
```bash
GEBMerge  <chatfile>  <outfile>  <file1>  <file2>  ...
# e.g.:
GEBMerge gtmerge.chat merged.gtd  t1.gtd  t2.gtd  t3.gtd  t4.gtd
```
✅ verified 2026-04-26 — `GEBMerge.c:L665-669` (usage string printed when `argc==1`)

- **Argument 1:** Chat file (configuration)
- **Argument 2:** Output file
- **Arguments 3+:** One or more input `.gtd` files (one per IOC/crate)

### Algorithm

1. Opens all input files simultaneously. Checks `ulimit -n` (file descriptor limit); exits with instructions to raise it if insufficient. ✅ verified 2026-04-26 — `GEBMerge.c:L873-890`
2. Reads one event from each input file into a **pool** of `nPoolEvents = nfiles` events.
3. Repeatedly selects the event with the **lowest timestamp** from the pool, writes it to output, and replaces it by reading the next event from its source file — classic N-way timestamp-sort merge. ✅ verified 2026-04-26 — `GEBMerge.c:L1237` (comment: "find lowest time stamp of the candidates we have"), `L1632`
4. Continues until all input files are exhausted.

**Note:** For DGS, `ANLDAQ/tcpReceiverMT` writes per-IOC files; these must be merged via GEBMerge before GEBSort can build coincidence events.

### Chat file parameters (GEBMerge.chat)

| Parameter | Description |
|---|---|
| `maxNoEvents N` | Stop after N events |
| `tsmask 0xHEX` | Timestamp mask (default = `LONG_MAX` = no mask) |
| `reportinterval N` | Print progress every N events |
| `chunksiz N` | Read buffer chunk size (multiplied x8 internally) ✅ verified 2026-04-26 — `GEBMerge.c:L769-770` (`control.chunksiz*=8`) |
| `bigbufsize N` | Internal merge-sort buffer size in events ✅ verified 2026-04-26 — `GEBMerge.c:L773-775` |
| `wosize P` | Write-out size as % of bigbufsize (float; `r1/100.0 * size`) ✅ verified 2026-04-26 — `GEBMerge.c:L790-792` |
| `nprint N` | Print detail for first N events ✅ verified 2026-04-26 — `GEBMerge.c:L782-786` |
| `startTS lo hi` | Process only events with TS in [lo, hi] range ✅ verified 2026-04-26 — `GEBMerge.c:L797-802` |
| `TSlistelen N lo hi` | Timestamp list filter controls ✅ verified 2026-04-26 — `GEBMerge.c:L804-808` |
| `dts_min N` | Minimum DTS between consecutive events before flagging ✅ verified 2026-04-26 — `GEBMerge.c:L809-813` |
| `dts_max N` | Maximum DTS before flagging ✅ verified 2026-04-26 — `GEBMerge.c:L815-819` |
| `dtsfabort N` | DTS forward abort threshold (default **5**; skips ahead) ✅ verified 2026-04-26 — `GEBMerge.c:L685,L821-825` |
| `dtsbabort N` | DTS backward abort threshold (default **5**; backward jumps) ✅ verified 2026-04-26 — `GEBMerge.c:L686,L826-830` |
| `waitfordata N` | Wait N seconds for more data (live/streaming mode) ✅ verified 2026-04-26 — `GEBMerge.c:L831-834` |
| `TSoffset gebType chan_range offset` | Apply timestamp offset `offset` (integer, 10 ns ticks) to all channels in `chan_range` for events of GEB type `gebType`. `chan_range` is a comma/range string like `1-5,7` (parsed by `str_decomp`); applies to DGS/DFMA and BANL88 types (ioff=0 or 1 header offset). ✅ verified 2026-04-26 — `GEBMerge.c:L836-844`, `str_decomp.c:L14-95` |
| `#` or `;` | Comment line — skipped |

✅ verified 2026-04-26 — `GEBMerge.c:L720-858` (chatfile parse loop; each `strstr()` branch matches a parameter)

### Notes
- Supports both raw (`USEZLIB==0`) and gzip-compressed (`USEZLIB==1`) input files. ✅ verified 2026-04-26 — `GEBMerge.c:L49-55` (conditional `off_t inData[]` vs `gzFile zFile[]`)
- Max simultaneous input files bounded by `MAXCOINEV` (compile-time constant) and OS file-descriptor limit (`ulimit -n`). ✅ verified 2026-04-26 — `GEBMerge.c:L900` (`assert(nPoolEvents < MAXCOINEV)`)
- Output format is standard GEB binary (GEBDATA header + payload per event).

---

## gtReceiver — DAQ-Side Data Receiver

_Source: [wiki.anl.gov/gsdaq/Receivers/GEBMerge/GEBsort](https://wiki.anl.gov/gsdaq/Receivers/GEBMerge/GEBsort) + `https://gitlab.phy.anl.gov/tlauritsen/gtreceiver.git`_

**gtReceiver** is Tim Lauritsen's DAQ-side receiver that collects data from IOCs and writes GEB-formatted files for each digitizer. It is separate from the production `ANLDAQ/tcpReceiverMT`.

### Versions

| Version | Description |
|---------|-------------|
| `gtReceiver3` | Old — writes without GEB headers. **Do not use.** ✅ verified 2026-04-25 — `gtReceiver3.c` has no GEBDATA/GEB_TYPE references; raw write only |
| `gtReceiver4` | Current recommended version. Writes GEB header + payload per digitizer, one file per channel. |
| `gtReceiver5` | Like gtReceiver4 but buffers and sorts timestamps before writing (allows running IOC in 'copy' mode). **Reported to have a bug — do not use.** |

### Build
```bash
git clone https://gitlab.phy.anl.gov/tlauritsen/gtreceiver.git
make -B
```

### Usage
```
gtReceiver4 <ioc_name> <output_file> <max_size_bytes> <GEB_type_id>
```
Example:
```bash
gtReceiver4 ioc1 data_run_001.gtd 2000000000 14
```
- `ioc1` — IP/hostname of the IOC to receive from
- `data_run_001.gtd` — base filename (actual output is **split per digitizer channel**)
- `2000000000` — max file size (2 GB) before rolling to new file
- `14` — GEB type ID (see http://gretina.lbl.gov/tools-etc/gebheaders for full list)

### What gtReceiver4 does
1. Connects to IOC TCP stream
2. Extracts from each packet: `board_id`, `packet_length`, `timestamp`
3. Constructs a **16-byte GEB header** and prepends it to the payload
4. Writes one file per digitizer (split by `board_id`)

**Data file naming:** one file per board_id; filename = `<base>_<board_id_4digits>`. ✅ verified 2026-04-25 — `gtReceiver4.c:L783` (`sprintf(str, "%s_%4.4i", fn, board_id)`)

**File roll:** when a file hits the max size limit, it is closed and a new one is opened.

### Packet Field Decoding
```c
int tmp = hdr[0];                                 // save before shift
board_id   = ((hdr[0] >>= 4) & 0xfff);           // bits 4..15 ✅ verified 2026-04-25 — gtReceiver4.c:L765
packet_len = ((tmp >>= 16) & 0x000007ff);         // bits 16..26 of ORIGINAL hdr[0] ✅ verified 2026-04-25 — gtReceiver4.c:L760,L767
packet_len = packet_len * 4;                      // convert word count → bytes
Geb.timestamp  = (unsigned long long int) hdr[1]; // ✅ verified 2026-04-25 — gtReceiver4.c:L751
ulli1          = (unsigned long long int) (hdr[2] & 0x0000ffff);
ulli1          = (ulli1 << 32);
Geb.timestamp += ulli1;                           // ✅ verified 2026-04-25 — gtReceiver4.c:L751-754
```
⚠️ **Note:** `packet_len` is derived from `tmp` (original `hdr[0]`), **not** from `hdr[0]` after the board_id shift. The two extractions must use a saved copy because `>>=` modifies `hdr[0]` in place.
_Note: in full IOC data (which includes `0xAAAAAAAA` SOE word), hdr[0/1/2] maps to packet word 1/2/3 (0-indexed after the SOE word)._

`board_id` is also called the **USER PACKET DATA** field and uniquely identifies the digitizer.

### GEB Header Format (written by gtReceiver)
```c
struct gebData {
    int            type;       // GEB type ID
    int            length;     // payload length in bytes
    long long      timestamp;  // event timestamp
};
```
✅ verified 2026-04-25 — `gtReceiver4.c:L47-53` (exact struct layout: `int type`, `int length`, `long long timestamp` = 16 bytes total)
The receiver does **not** interpret the payload data — it just wraps it. Interpretation is left to GEBSort.

### Relationship to production tcpReceiverMT
`gtReceiver4` is an **alternative** to `ANLDAQ/tcpReceiverMT`. Both write GEB-format files compatible with GEBMerge/GEBSort. The production DGS DAQ uses `tcpReceiverMT`. `gtReceiver4` is used when running from Tim Lauritsen's tools or in certain test/analysis setups.

### Online (interactive) mode
Run one `gtReceiver4` per IOC simultaneously with `GEBMerge` + `GEBSort` using `waitfordata 300` in chat files to stream data in real-time:
```bash
# Start display root session:
rootn.exe
.L GSUtil.cc++
sdummyload(200000000)   # Returns map start address, e.g. 0x9ef6e000

# Start GEBSort with mapfile:
./GEBSort_nogeb \
  -input disk merged_data.gtd \
  -mapfile c1.map 200000000 0x9ef6e000 \
  -chat GEBSort.chat

# In display session: sload("c1.map"); update(); [display]; update(); ...

# Stop at end of run:
pkill -9 GEBmerge
pkill -9 GEBSort
```
_If beam is absent >5 minutes, receivers must be restarted (or a new run started)._

---

## GEBClient — EPICS IOC → GEB Data Sender

_Source: `gebsort/GEBClient.c`, `gebsort/GEBClient.h`_ (215 lines)

`GEBClient` is a small C library that provides an EPICS-aware TCP connection from inside a VxWorks IOC to a running GEB server (e.g. `GEBMerge`). It uses `epicsMutex` for thread-safe access across IOC threads.

**`GEBLink.h`** (30 lines) — shared header defining the in-memory GEB struct and GRETINA-origin constants: ✅ verified 2026-04-26 — `gebsort/GEBLink.h:L1-30`
- `struct GEBData` — in-memory event: `int type`, `int length`, `long long timestamp`, `void *payload`, `short refCount`, `short refIndex`, `struct GEBData *next` (linked list for TrackIF.c)
- `GEB_HEADER_BYTES 16` — on-disk/wire GEB header size (matches `data_structures.md`: `int type + int length + uint64_t timestamp = 4+4+8 = 16`)
- `GEB_PORT 9005` — **GRETINA convention only**; DGS uses **port 9001** for all VME crate TCP connections
- GRETINA GEB types: `KEEPALIVE=0, DECOMP=1, RAW=2, TRACK=3, BGS=4, S800=5` — DGS uses types 14 (`GEB_TYPE_DGS`) and 15 (`GEB_TYPE_DGSTRIG`), defined separately in `GEBSort.h`
- TAP protocol codes: `TAP_DATA=0, TAP_ACK=1, TAP_TIMEOUT=2, TAP_NOT_FOUND=4, TAP_NOT_RUNNING=8, TAP_ERROR=16` — used by GRETINA `gretTap`; not active in DGS

### Struct
```c
struct gebClient {
   epicsMutexId connectionMutex;
   int outSock;      // TCP socket fd (0 = disconnected)
   int bigendian;    // 1 if host is big-endian
   struct GEBData zeromsg;  // pre-built keepalive (zero) message
};
```

### API

| Function | Return | Description |
|----------|--------|-------------|
| `GEBClientInit()` | `struct gebClient *` | Allocate + init client struct; create mutex; detect endianness via `htonl(1)==1` |
| `setGEBClient(i, addr, port)` | int | Resolve hostname, open TCP socket, connect; reads 4-byte reject word from server; returns 0 on success, 1–6 on error |
| `closeGEBClient(i)` | void | Close socket under mutex lock |
| `checkGEBClient(i)` | int | Keepalive probe: 0=OK, 1=soft error (buffers full), 2=hard error (disconnected), 3=uninitialized |
| `sendGEBData(i, outmsg)` | int | Send one GEB event; returns 0=success, 1=soft (retry), 2=hard (reconnect needed) |

### Protocol
1. On connect: server sends a 4-byte `reject` word. Nonzero = server not ready.
2. Each `sendGEBData` call: first reads a 4-byte ack from the previous send (if `reject` → server buffers full → send keepalive instead).
3. Payload is sent as: GEB header bytes (endian-swapped if big-endian host) followed by raw payload bytes (NOT swapped — payload is caller's responsibility).
4. A null `outmsg` sends a keepalive (zeroed GEB header only).

### Endianness
VxWorks PowerPC (VME) is **big-endian**. The GEB protocol is little-endian. `GEBClient` byte-swaps the header using `swab()` when `bigendian=1`, but leaves the payload untouched — the caller must pre-swap payload data if needed. ✅ verified 2026-04-25 — `GEBClient.c:L181-194` (header: `swab(outmsg, &outdata[0], ...)` when bigendian; payload: `bcopy` only, no swap)

### Where it is used
`GEBClient` was historically used within VxWorks IOC device support to forward digitizer data directly to GEBMerge while the IOC is running. In the modern DGS, `tcpReceiverMT` (running on a Linux host) handles data collection instead, making `GEBClient` a legacy path.

---

## dmpdata — GEB File Dump Tool

_Source: `gebsort/dmpdata.c`_ (161 lines)

Simple command-line utility to dump GEB binary file contents in human-readable format.

### Usage
```bash
dmpdata <infile> <N_events> <dt_threshold>
```
- `N_events` — number of events to print (stops after this many)
- `dt_threshold` — print a separator line when timestamp gap > dt (in 10 ns ticks)

### What it prints
For each event:
- Event number, GEB type (as string via `get_GEB_Type_str()`), type int
- 48-bit timestamp (raw), delta from previous event in ticks + microseconds
- Payload size in bytes
- If `GEB_TYPE_DECOMP`: decoded `CRYS_INTPTS` structure (GRETINA crystal interaction points)
- If `GEB_TYPE_TRACK`: decoded `TRACKED_GAMMA_HIT` structure
- Backward-in-time events flagged with `+-+-+-+-+-+` separator line

### Summary at end
- Total events read
- Count of backward-in-time events (with percentage)
- Histogram of GEB type IDs seen
- `sizeof(TRACKED_GAMMA_HIT)` (divided by 32 and 64 for sanity check)

**Note:** `dmpdata` is primarily a diagnostic for GRETINA-format data (CRYS_INTPTS, TRACKED_GAMMA_HIT). For DGS-format data (GEB type 14), the payload is not decoded — only the GEB header is printed.

---

## GEB File Manipulation Utilities

_Source: `gebsort/GEBCrop.c` (141 L), `GEBSplit.c` (144 L), `GEBFilter.c` (767 L), `GEBHeader.c` (30 L). Code-read 2026-04-26._

These standalone executables operate on GEB binary data files (stream of GEB header + payload records). They are compiled as part of the gebsort tree but are not GEBSort sorters — they manipulate raw GEB streams offline.

---

### `GEBHeader.c` — GEB Header Inspector (30 L)

Single utility function `printDgsHeader(DGSHEADER dgsHeader)`.  Prints the header ID field; if `id == 0xaaaaaaaa`, warns that the file has no valid header (old data). Called from other tools that read DGSHEADER-prefixed files. Not a standalone binary.

---

### `GEBCrop.c` — GRETINA Mode-2 File Cropper (141 L)

Reads a GEB binary file, crops GRETINA mode-2 (`GEB_TYPE_DECOMP`) payloads to the actual number of interaction points, and writes the result to a new file.

```bash
GEBCrop <infile> <outfile>
```

**How it works:**
- Reads GEB header + payload pairs in a loop
- For `GEB_TYPE_DECOMP` events: casts payload to `CRYS_INTPTS`, reads `ptinp->num` (number of interactions), recomputes `plz = DCRHDRLEN + num × sizeof(DCR_INTPTS)`, and writes back only that many bytes (smaller payload)
- Other event types pass through unchanged
- First 10 events print debug info: original and cropped payload size

**Purpose:** GRETINA mode-2 data may be padded to `MAX_INTPTS` slots; cropping reduces file size by stripping unused interaction slots.

**Output summary:** prints `sizeof(CRYS_INTPTS)`, `sizeof(DCR_INTPTS)`, `MAX_INTPTS`, `sizeof(GEBDATA)`, and total events read.

---

### `GEBSplit.c` — GEB File Splitter by Type (144 L)

Splits a GEB binary file into two output files: one for GRETINA mode-2 data and one for GRETINA module-29 data.

```bash
GEBSplit <indata> <mode2_out> <bank88_out>
```

**How it works:**
- Reads each GEB record (header + payload via `PAYLOAD` buffer — `char p[MAXDATASIZE]`)
- Routes `GEB_TYPE_DECOMP` events → output file 1 (`mode2`)
- Routes `GEB_TYPE_GT_MOD29` events → output file 2 (`bank88`)
- Any other type triggers an error message and exits
- Prints running totals: events read, written to file 1, written to file 2

**Note:** binary name in the help string is `GEBSplit` but the executable usage line says `GEBSplit indata mode2 bank88` — the `bank88` label is legacy naming for module-29 bank 88 data from GRETINA.

---

### `GEBFilter.c` — GEB Stream Filter / Transform (767 L)

The most powerful GEB utility: reads a GEB binary file, applies configurable filters and transforms (via a chat file), and writes the filtered/transformed result to a new output file.

```bash
GEBFilter <chatfile> <infile> <outfile>
```

**Chat-file parameters:**

| Option | Effect |
|---|---|
| `nevents <N>` | Stop after N events |
| `waitusec <N>` | Sleep N µs between output events (for online simulation) |
| `vetocube <file>` | Load veto-cube file and reject bad interaction points |
| `addT0` | Convert GEB TS from 10 ns → 1 ns units; for mode2 adds `cip->t0` (nearest ns) |
| `GT2AGG4 <file>` | Convert GRETINA tracked data to AGATA GEANT4 ASCII format |
| `xyz_smear <dx> <dy> <dz>` | Smear x/y/z interaction positions by ±(dx, dy, dz) mm (uniform random) |
| `fix104 <dt>` | Offset timestamps of detector 104 by `dt` ticks (per-detector TS correction) |
| `echo` | Echo each chat line as it is processed |
| `#` or `;` | Comment lines (skipped) |

**Veto-cube filtering:**
- Veto cube is a 3D per-interaction-point lookup loaded from binary file; each voxel is 0 (good) or non-zero (bad)
- Bad interaction points are removed from `CRYS_INTPTS.intpts[]`; energy from removed points is redistributed proportionally to surviving points
- If all interaction points are vetoed (`cip->num == 0`), the event is dropped from output

**GT2AGG4 mode:**
- Converts GRETINA crystal-local coordinates to world coordinates using `crmat.LINUX` (binary rotation matrix file, `MAXDETPOS+1 × MAXCRYSTALNO+1 × 4 × 4` floats)
- Reads AGATA rotation/translation data from `GANIL_AGATA_crmat.dat` (180 entries, 4 lines each)
- For each interaction point: normalizes crystal coordinates, applies `crmat` rotation, writes AGATA G4 ASCII format: `<AG_crystal_no> <e> <x> <y> <z> 1`
- Finds nearest AGATA crystal by dot-product angle comparison against 180 AGATA detector vectors
- Events in GT2AGG4 mode are **not** written to the normal binary output

**addT0 mode:**
- Multiplies all TS ×10 (converts from 10 ns to 1 ns units)
- For `GEB_TYPE_DECOMP`: additionally adds `(long long int)(10.0 × cip->t0 + 0.5)` (nearest 1 ns)

**Fix104 mode:**
- Specifically targets `detectornumber == 104` in GRETINA (crystal_id decoding: `holeNum = (crystal_id & 0xfffc) >> 2`, `crystalNumber = crystal_id & 0x0003`, `detectornumber = 4×holeNum + crystalNumber`)
- Adds `dt` ticks to both `ptgd->timestamp` and `cip->timestamp` for affected events
- Was used for a known timestamp offset bug in experiment 2137

**Output summary:** events read/written (and veto statistics if vetocube was active).

---

## `dtbtev.c` — GEB Timing Diagnostics Tool (309 L)

Standalone diagnostic tool. Reads a GEB binary file and produces three kinds of timing spectra per GEB type.

```bash
dtbtev <infile>
```

**Three spectrum types (per GEB type 0–MAX_GEB_TYPE, where type 0 = any type):**

| Spectrum file | Content | Units |
|---|---|---|
| `dtbtev_NN.spe` | Histogram of time-between-events (delta TS) | 10 ns ticks (0–16000) |
| `ts_NN.spe` | TS value as function of event number (first 16000 events) | raw TS units |
| `evtime_NN.spe` | Event rate vs elapsed time (binned every CTFAC=100 ticks = 1 µs) | 1 µs per bin |

- Spectra are only written if the total count for that type is > 0
- Prints per-type rates (Hz) and total counts at end
- Hardcoded `TSSEARCH=1`: also prints a warning line if certain specific timestamps are found (Darek sweeper data special search — hardcoded TS values for debugging)
- Elapsed time computed as `(last_TS - first_TS) / 100,000,000` seconds (assuming 10 ns tick)
- Uses `wr_spe()` to write spectra in GEBSort SPE binary format

**Purpose:** quick sanity check on GEB data integrity — exponential dtbtev confirms Poisson-distributed source; flat evtime confirms stable rate; TS monotonically increasing confirms correct merge order.

---

## Cross-References

- [gebsort.md](gebsort.md) — Core GEBSort: GEBSort framework, bin_dgs, jta, calibration workflow
- [gebsort_additional_sorters.md](gebsort_additional_sorters.md) — Non-DGS sorters (GRETINA, TAC-II, DFMA, etc.)
- [ANLDAQ_tcpReceiver.md](ANLDAQ_tcpReceiver.md) — tcpReceiverMT deep-dive (production DGS receiver; contrast with gtReceiver here)
- [data_structures.md](data_structures.md) — GEB binary format: what GEBMerge reads and writes

---
*Created: 2026-04-26. Split from gebsort.md. Source: `DGS_tools_pack/gebsort/`.*
