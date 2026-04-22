# dgs_analysis — DGS Analysis Code Collection

## Table of Contents
- [What It Is](#what-it-is)
- [Repository](#repository)
- [armory/ — Reusable Tools](#armory--reusable-tools)
  - [fastEventConstructor](#fasteventconstructor)
  - [parquet_pysort — Python/C++ Parquet Pipeline](#parquet_pysort--pythonc-parquet-pipeline)
  - [gray_apps](#gray_apps)
- [working/ — Experiment-Specific Scripts & Calibration](#working--experiment-specific-scripts--calibration)
  - [RunParquet (legacy)](#runparquet-legacy--superseded-by-processrun)
  - [parquetCLI](#parquetcli)
  - [gain_from_parquet.py](#gain_from_parquetpy)
  - [pz_from_parquet.py](#pz_from_parquetpy)
  - [pz_from_evtparquet.py](#pz_from_evtparquetpy)
  - [DownloadRaw.sh](#downloadrawsh)
  - [ProcessRUN (primary pipeline)](#processrun-primary-pipeline--apr-2026)
  - [BenchmarkTAC2_021.sh](#benchmarktac2_021sh)
- [Data Format: GEB](#data-format-geb-gretina-event-builder)
- [Connections to Other Subsystems](#connections-to-other-subsystems)
- [EventBuilder_PQ Benchmark Results](#eventbuilder_pq-benchmark-results)
- [Notes](#notes)
- [Cross-References](#cross-references)
- [analyzer_*.cpp — ROOT Analysis Scripts](#analyzercpp--root-analysis-scripts)

---

## What It Is

A collection of **experiment-independent analysis code** for processing DGS raw data. Structured as:
- `armory/` — reusable, experiment-independent tools (event builders, decoders, etc.)
- `working/` — experiment-specific files (calibration files, etc.)

---

## Repository

```sh
git clone --recursive https://gitlab.phy.anl.gov/dgs-tools-pack/dgs_analysis.git
# or via SSH:
git clone --recursive git@gitlab.phy.anl.gov:dgs-tools-pack/dgs_analysis.git
```

---

## armory/ — Reusable Tools

### fastEventConstructor

**Purpose:** Reads raw DGS binary files and produces CERN ROOT TTrees (event-built data).

**Data format:** Raw DGS binary files contain **GEB (GRETINA Event Builder) format** records — each record has a `GEBHeader` + payload. The `Hit` class stores these, decoding the payload into a `DIG` struct.

**Important fields per hit:**
- `UniqueID` — `DigID * 100 + channel` (parsed from filename; `DigID` = last 3-4 digits of filename, `channel` = last component) ✅ verified 2026-04-06 — `BinaryReader.h:L33` (`GetUniqueID() { return DigID * 100 + channel; }`). Note: doc previously said `board_id * 10 + channel_id` — corrected.
- `pre_rise_energy` — ADC sum accumulator before discriminator rise; Word 8[23:0] ✅ verified 2026-04-07 — `class_DIG.h:L52,L369`
- `post_rise_energy` — ADC sum after rise; spans Word 8[31:24] + Word 9[15:0] (32-bit total) ✅ verified 2026-04-07 — `class_DIG.h:L53,L370`
- `early_pre_rise_energy` — earlier pre-rise window sum; Word 12[23:0] — used for pole-zero correction ✅ verified 2026-04-07 — `class_DIG.h:L65`
- `timestamp` — timing

**EventBuilder Variants:**

| Variant | Key Feature | Use Case |
|---------|-------------|----------|
| `EventBuilder` | Original; priority queue, parallel file scan; compile-time switch between ROOT and **flat Parquet** output via `OUTPUT_TYPE` macro (`ParquetOutput.h`) | Legacy / general use |
| `EventBuilder_S` | Parallel scan pre-pass + single-threaded k-way merge + **optional multithreaded GSL sigmoid trace fitting** | When trace shape analysis (PSD) needed; also used as reference for PQ verification |
| `EventBuilder_Q` | Async double-buffered k-way merge; batch pre-decoding; pipelined ROOT writers | Fast I/O, large datasets |
| `EventBuilder_PQ` | Parallel k-way merge with sector partitioning; N parallel threads | Maximum throughput, multi-core |
| `EventBuilder_X` | Same engine as PQ, **Parquet output** (no ROOT dependency) | **Primary pipeline** — used by `ProcessRUN` |

**EventBuilder_S details:**
- **Parallel scan pre-pass**: all input files are scanned in parallel threads (one thread per file) via `BinaryReader::Scan(true)`; reports hit counts, file sizes, and global timestamp range (earliest + latest) ✅ verified 2026-04-17 — `EventBuilder_S.cpp:L342-360`
- **Single-threaded k-way merge**: priority queue over all groups (files grouped by DigID); initial batch loaded from each group, then merge loop reads next hits as needed
- **Trace analysis option** (`nWorkers` arg): when `nWorkers > 0`, each event's waveform traces are fit to a **logistic sigmoid model**: `Yi = A / (1 + exp(-(t - T0) / riseTime)) + B` using the GNU Scientific Library (GSL) nonlinear least-squares (`gsl_multifit_nlinear_trust`). 4 parameters per trace: A (amplitude), T0 (timing offset), riseTime (pulse rise time), B (baseline). Up to 100 iterations, convergence at `1e-5`. Chi² residual also stored. ✅ verified 2026-04-17 — `EventBuilder_S.cpp:L202-263`
- **Multithreaded trace fitting**: when `nWorkers > 1`, events with traces are dispatched to a worker thread pool via a condition-variable queue (`dataQueue`); an ordered output map (`outputMap` keyed by evID) ensures in-order ROOT tree writes. When `nWorkers == 1`, trace fitting runs single-threaded inline.
- **ROOT output**: same branch layout as EventBuilder_Q (evID, NumHits, id, detID, pre/post_rise_energy, eventTS, trigTS); trace branches (`traceCount`, `traceDetID`, `traceLen`, `trace[][]`) added when `saveTrace=1`; `tracePara[traceCount][4]` and `traceChi2[traceCount]` added when `nWorkers > 0` ✅ verified 2026-04-17 — `EventBuilder_S.cpp:L403-420`
- **Usage**: `./EventBuilder_S [outfile] [timeWindow] [useTrigTS] [saveTrace] [nWorkers] [file1] [file2] ...` — note `nWorkers` is the 5th arg (vs `nWorkers` in Q), but semantics differ: in S it controls trace analysis thread count
- **Benchmark**: 659.7s for 17.4 GB / 228M hits (TAC2_054, 30 workers) — much slower than PQ (32.2s). EventBuilder_S output verified to match PQ: 26,656,402 events ✅ verified 2026-04-12 — `README.md`

**EventBuilder_Q optimizations:**
- Batch pre-decoding (not per-hit)
- ReadPool: async pre-fills next batch while current is consumed (double buffering)
- Lightweight merge heap: `{timestamp, groupIndex}` only (16 bytes vs full DIG struct)
- 4 pipelined ROOT writers (round-robin)

**EventBuilder_PQ optimizations (extends Q):**
- **QuickBounds phase**: reads only the first+last GEB header of each file to get per-file timestamp bounds and compute hit count from `fileSize / blockSize`. Sector seeking uses binary search on the file (fixed block size → any hit N is at byte offset `N × blockSize`). Falls back to full scan for files with inconsistent payload sizes.
- Sector partitioning: divides time span into N sectors with ghost regions (width = `timeWindow`) at boundaries to avoid splitting coincidence events
- N parallel merge threads × M pipelined writers per thread (N×M partial files, merged at end)
- Per-sector double-buffered ReadPool
- Hits whose first event falls in a ghost region are discarded to prevent duplicates across sector boundaries

```sh
# Build
make                    # all variants
make EventBuilder_Q     # single variant
make EventBuilder_PQ

# Usage
./EventBuilder_Q [outfile] [timeWindow] [useTrigTS] [saveTrace] [nWorkers] [file1] [file2] ...
```

**Original `EventBuilder` — `ParquetOutput.h` flat Parquet schema:**
The original `EventBuilder.cpp` includes `ParquetOutput.h` and has a compile-time switch `#define OUTPUT_TYPE 1` (0=ROOT, 1=Parquet). When Parquet output is selected, it uses a **flat schema** (one row per hit, not nested lists), implemented via the `BatchBuilder` class in `ParquetOutput.h`. Flat schema columns: ✅ verified 2026-04-18 — `ParquetOutput.h:L50-64, EventBuilder.cpp:L11,39`

| Column | Arrow Type | Description |
|--------|-----------|-------------|
| `event_id` | uint64 | Sequential event ID |
| `NumHits` | uint32 | Hits in this event (repeated per hit row) |
| `id` | uint16 | Raw digitizer ID (boardID×10 + channelID) |
| `detID` | uint16 | Mapped detector ID |
| `pre_rise_energy` | uint32 | Trapezoidal sum S1 |
| `post_rise_energy` | uint32 | Trapezoidal sum S2 |
| `energy` | int64 | Computed energy (S2 − S1, integer) |
| `eventTS` | uint64 | Event timestamp (10 ns ticks) |
| `trigTS` | uint64 | Trigger timestamp (10 ns ticks) |
| `deltaT` | float64 | Time difference (computed) |
| `vdc` | float64 | Velocity/Doppler correction factor |
| `hitBGO` | bool | True if BGO hit |

> **Note:** This flat schema differs from `EventBuilder_X`'s nested list schema (which uses `list<uint16>` per column, `e_raw` as float32 with PZ correction, and writes to `/dev/shm` before merge). The original `EventBuilder` Parquet is simpler and row-oriented. `EventBuilder_X` is preferred for production runs (via `ProcessRUN`).

SNAPPY compression, PLAIN encoding. Author: Scott Carmichael (2026-02-24).

### parquet_pysort — Python/C++ Parquet Pipeline

**Purpose:** Converts raw GRETINA GEB binary data into Apache Parquet for analysis. 3-stage pipeline.

**Pipeline stages:**

| Stage | Script | Input → Output |
|-------|--------|----------------|
| 1 | `make_filemap_dgs.py` | Run dir + `map.dat` → `<exp>_<run>_fileMap.dat` (board/chan → tid/tpe mapping) |
| 2 | `decode.py` + C++ lib | Filemap + raw GEB files → `<exp>_<run>_dgs.parquet` (hit-level, timestamp-sorted) |
| 3 | `event_builder.py` | Hits parquet → `<exp>_<run>_events.parquet` (coincidence events) |

**Decode output schema** (`_dgs.parquet`):
- `tid`, `header_ts`, `trigger_ts`, `sum1`, `sum2`
- `e_raw`, `e_cal`, `e_dc` (Doppler-corrected)
- `CSflag` (Compton suppression), `pileup_count`

**Events output schema** (`_events.parquet`):
- `event_id`, `gs_mult`, `gs_hitid`
- `gs_ts`, `gs_cryid`
- `gs_eraw`, `gs_ecal`, `gs_edc`, `gs_flag`

**C++ decode operations** (in `dgs_decode_lib.cpp`):
- Pole-zero correction
- Energy algorithms: algo 0 / SZ_1 / SZ_2
- Gain + offset calibration
- Doppler correction
- Compton suppression flag

**Event builder:** timestamp coincidence window `(ts − first_ts) < timewin` ✅ verified 2026-04-20 — `dgs_decode_lib.cpp:L676`; N parallel threads with boundary merge and global `event_id` assignment.

**Key files:**
| File | Role |
|------|------|
| `decode.py` | Stage 2 decoder (drives C++ lib) |
| `dgs_decode_lib.cpp` | C++ shared lib — pole-zero, energy, calibration |
| `geb_format.py` | GEB header/payload format definitions |
| `event_builder.py` | Stage 3 coincidence event builder |
| `make_filemap_dgs.py` | **Stage 1 filemap builder** (227 lines, by Youngju Cho). Reads raw GEB files in a run dir; for each DGS event verifies all events map to exactly one `(tpe, tid)` pair; writes `<exp>_<run>_fileMap.dat` (format: `tid  tpe  suffix`). DGS event ID extracted via `_dgs_id()`: `board_id = ((w0 >> 4) & 0xFFF)`, `chan_id = (w0 & 0xF)` → `id = board_id * 10 + chan_id` (big-endian first word). Supported detector types (`TPE_NAME`, from GTMerge.h): 0=NOTHING, 1=GE, 2=BGO, 3=SIDE, 4=AUX, 5=DSSD, 6=FP, 7=XARRAY, 8=CHICO2, 9=SSD, 10=CLOVER, 11=SPARE, 12=SIBOX, 14=DUBDET, 15=XIA. Sort rank: GE=0, BGO=1, SIDE=2, AUX=3. Also handles GEB_TYPE_GT_MOD29 events (skipped in filemap). ✅ verified 2026-04-15 — `make_filemap_dgs.py:L28-50` (TPE_NAME dict + _dgs_id formula) |
| `read_parquet.py` | **PyArrow parquet inspector** (450 lines, by Youngju Cho). CLI: `--info` (metadata only, instant), `--columns col1 col2`, `--where "geb_type==8" "timestamp>8e11"`, `--head N`, `--tail N`, `--rows M N` (row slice), `--hex N` (print N words as hex), `--to-csv out.csv`. No pandas dependency — pure PyArrow. |
| `PQDecode.chat` | Decode config: algo, MM, KK, threads, etc. |
| `PQMerge.chat` | Merge config |
| `dgs_gain.cal` | Gain calibration (format: `# gsid  gain  offset` header + one line per crystal) |
| `dgs_pz.cal` | Pole-zero calibration (format: no header, `{det:3d}  {pz:.6f}` per line) |
| `angtheta.dat` | **Theta angles per GS hole** — 110 lines, one angle per line (degrees). Matches Gammasphere geometry order. Used for Doppler correction. Sample: hole 1–4 = 17.3°, hole 5 = 31.7°, etc. |
| `filemap_dgs.dat` | **DAQ-id → GS-hole + channel-type mapping**. Format: `{GS_hole} {type} {DAQ_id}_{ch}`. Types: GE, BGO, SIDE, AUX. Example: `7 GE 0133_8` = GS hole 7, Ge channel, DAQ board 0133 ch 8. Used by `make_filemap_dgs.py` to build per-run filemaps. ✅ verified 2026-04-14 — `filemap_dgs.dat` lines 1-4 confirm format and example |
| `map.dat` | DAQ id → tid/tpe mapping |

**Build C++ lib:**
```bash
g++ -O2 -std=c++17 -shared -fPIC -o libdgs.so dgs_decode_lib.cpp
```
✅ verified 2026-04-15 — `parquet_pysort/README.md:L61,L144`

**In practice:** use `working/ProcessRUN` (C++ EventBuilder, primary) or `working/RunParquet` (Python parquet_pysort, legacy) — both driven from `expInfo.sh`.

**Ultimate goal:** Roaring bitmap index — for each energy bin, a roaring bitmap stores the set of `event_id`s containing a hit, enabling rapid energy-gated coincidence queries without scanning the full dataset.

**Prototype:** `armory/fastEventContructor/roaring_bitmap_tests.py` (by S. Carmichael, 2026-02-25) — proof-of-concept using `pyroaring` + PyArrow. Two approaches tested:
1. Load entire table with `pq.read_table()` + pandas, iterate rows to populate bitmaps — simple but memory-heavy
2. Stream via `pyarrow.dataset` scanner with `batch_size=10^7` (respects row-group boundaries) — memory-efficient

Benchmark result: ~15 s to build bitmaps for 10^6 events; ~1–2 s to retrieve events matching a gamma-gamma gate (`event_id in g1_union`). Energy bins: 4000 bins spanning 0–8000 ADC (bin_width = 2.0). Gate query: union of per-bin bitmaps in the gate range → filter parquet with `(event_id, “in”, g1_union)`. Dependency: `pip install pyroaring`. ✅ verified 2026-04-20 — `roaring_bitmap_tests.py`

**Threading details:**
- `decode.py`: `ThreadPoolExecutor`, one worker per tid; GE/BGO files submitted as sub-tasks for overlapping I/O. Requires **Python 3.14.3t (free-threaded / no-GIL)** for true parallelism. ✅ verified 2026-04-14 — `parquet_pysort/CLAUDE.md:L63` ("Python 3.14.3t (free-threaded) — No-GIL build"); `README.md:L145` ("Python 3.12+ — free-threaded build (3.14t) recommended")
- `event_builder.py`: Reads all input into one Arrow table, splits into N chunks, calls C++ `build_events()` per chunk in parallel. Column renames (`header_ts→gs_ts`, etc.) are zero-copy Arrow references — no `.to_pylist()`. Optional `--sort-by gs_ecal|gs_edc` sorts output before writing (default: no sort). ✅ verified 2026-04-18 — `parquet_pysort/CLAUDE.md` event_builder description.
- `decode.py --write-threads N`: Output split into `_000.parquet`, `_001.parquet`, … — feed multiple files to `event_builder.py`.

**Algo notes:**
- Algo 0: simple `sum2/MM - sum1/MM` (no pole-zero, no baseline).
- Algo 1 (SZ_1, low-rate): Baseline via exponential avg (`BASE_ALPHA=0.01` ✅ verified 2026-04-20 — `dgs_decode_lib.cpp:L35`, updated only when `dtev ≥ 250 µs`). Energy = `sum2/MM - sum1/MM * pz1 - base*(1-pz1)` where `pz1 = PZ^(1/MM)`. Both require `base > 10.0` for nonzero energy.
- Algo 2 (SZ_2, high-rate): Uses `pz4 = PZ^((MM+KK)/MM)`. Two regimes based on `dtev` vs `dgs_SZ_t1`/`dgs_SZ_t2`: `≥ t2` computes baseline from `sampled_baseline` (FPGA-sampled, 24-bit, `MSAMPLE=1024` = 10.24 µs window) using PZ decay formula; `< t2` extrapolates from pre-learned `baselast`/`sum1last` factor. Energy formula same as SZ_1 but uses `pz4`. Requires `base > 10.0`.
- `dtev`: computed from firmware `last_disc_timestamp` (time since last discriminator trigger), with wrap-around correction. Matches `lastdisc_dt_ticks()` in `bin_dgs.c`.
- `sum2` field extraction in `jta.c` is header-type-dependent: types 0/1/3/5 use `>> 28` (4-bit); types 4/6/7/8 use `>> 24` (8-bit).
- `pileup_count` extraction in `jta.c` has a known bit-shift bug (`(word5 & 0x00FFC000) >> 24`) that always produces 0; replicated as-is for compatibility. ✅ verified 2026-04-18 — `gebsort/jta.c:L553` (bug confirmed: 10-bit field at bits 14–23, shifted right 24 → always 0); `dgs_analysis/armory/parquet_pysort/dgs_decode_lib.cpp:L174` (comment explicitly says "replicates jta.c >> 24 exactly")

**GEBSort reference:** `GEBSort.cxx:GEBGetEv()` — coincidence grouping: `while ((TS - curTS) < dTS)`. Default `dTS=500` (= 5 µs at 10 ns/tick). ✅ verified 2026-04-09 — `GEBSort.cxx:L2502` (`Pars.dTS = 500`). `jta.c:DGSEvDecompose_v3()` parses payloads (big-endian swap, 48-bit timestamp from words 1+2, `trigger_timestamp` only in header types 7/8).

_Source: `dgs_analysis/armory/parquet_pysort/CLAUDE.md` + `README.md` — updated 2026-04-07_

### gray_apps

> **Full reference moved to [`dgs_analysis_grayapps.md`](dgs_analysis_grayapps.md)** (split 2026-04-16 for size).

**GrayCAL** — HPGe energy calibration GUI (M.P. Carpenter). **GrayMAN** — multi-peak spectrum analysis GUI. **grayfit** — shared fitting library (AutoFitter, FittingRunner, PeakFinder, pole_zero_fitter). See the split file for full module-level documentation.

Key facts:
- GrayCAL requires Python 3.13+; install via `conda env create -f environment.yml` or `./setup_venv.sh`
- Run: `graycal` (installed) or `python main.py`
- GrayMAN uses matplotlib; GrayCAL uses Plotly
- `grayfit` provides `AutoFitter`, `FittingRunner`, `PeakFinder`, `CalibrationPoints`, `pole_zero_fitter`
- ⚠️ GrayMAN `peak_detection.py` is a known placeholder — use GrayCAL's `AutoFitter` for production
- `Euautocal.json` (16 152Eu lines, 121.8–1408.0 keV) is the default calibration source

> **Full details:** → [`dgs_analysis_grayapps.md`](dgs_analysis_grayapps.md)

_Source: `dgs_analysis/armory/gray_apps/src/Fitter/` — explored 2026-04-07_

---

## working/ — Experiment-Specific Scripts & Calibration

Holds experiment-specific scripts and calibration files. All paths driven by `expInfo.sh` from `~/ANLDAQ/tcpReceiver/expInfo.sh`.

*Updated 2026-04-07 from git pull (commits up to 0100567)*

### RunParquet *(legacy — superseded by ProcessRUN)*

> **As of Apr 2026, `ProcessRUN` (C++ EventBuilder) is the primary pipeline.** `RunParquet` drove the Python `parquet_pysort` pipeline (`make_filemap_dgs.py → decode.py → event_builder.py`). It still exists but `ProcessRUN` is faster and preferred.

```bash
./working/RunParquet [--decode-only] <expInfo.sh> <run_number> [TIMEWIN] [THREADS]
```

| Arg | Default | Description |
|-----|---------|-------------|
| `--decode-only` | off | Stages 1+2 only, skip event builder (for pole-zero prep) |
| `TIMEWIN` | 1000 | Coincidence window in ticks ✅ verified 2026-04-09 — `RunParquet:L67` (`TIMEWIN="${2:-1000}"`) — note: script header comment still says 800 (stale) |
| `THREADS` | 78 | Threads for decode + event builder |

**Output:**
- `$expFolder/Parquet/decode/$expName_NNN_dgs.parquet` — timestamp-sorted hits
- `$expFolder/Parquet/events/$expName_NNN_events.parquet` — coincidence events

**Parquet schema:**

| File | Column | Type | Description |
|------|--------|------|-------------|
| `_dgs.parquet` | `tid` | int64 | Crystal ID |
| | `header_ts` | int64 | GEB header timestamp (10 ns/tick) |
| | `trigger_ts` | int64 | DGS payload trigger timestamp (0 if unavailable) |
| | `sum1` | int64 | Trapezoid baseline sum (ADC counts) |
| | `sum2` | int64 | Trapezoid peak sum (ADC counts) |
| | `e_raw` | float64 | Raw energy before gain/offset cal (ADC channels) |
| | `e_cal` | float64 | Calibrated energy: `e_raw × gain + offset` (keV) |
| | `e_dc` | float64 | Doppler-corrected energy (keV) |
| | `CSflag` | int8 | Compton suppression: 1=BGO-vetoed, 0=clean |
| | `pileup_count` | int32 | Pileup count from firmware (typically 0 due to known bit-shift bug) |
| `_events.parquet` | `event_id` | int64 | Global event index (0-based) |
| | `gs_mult` | int64 | Gamma-ray multiplicity (hits per event) |
| | `gs_hitid` | int64 | Hit index within event (0-based) |
| | `gs_ts` | int64 | Hit timestamp (DAQ ticks) |
| | `gs_cryid` | int64 | Crystal ID |
| | `gs_eraw` | float64 | Raw energy (ADC channels) |
| | `gs_ecal` | float64 | Calibrated energy (keV) |
| | `gs_edc` | float64 | Doppler-corrected energy (keV) |
| | `gs_flag` | int8 | Compton suppression flag |

_Source: `parquet_pysort/README.md` ✅ verified 2026-04-09_

### parquetCLI

Interactive REPL for exploring `_dgs.parquet` (hit-level) or `_events.parquet` (event-level) files. Columns discovered dynamically at load time.

```bash
./working/parquetCLI <file.parquet>
./working/parquetCLI <file.parquet> --script working/script.py
```

**`pq_api.py`** — IDE type-stub only; never import directly. Provides typed signatures for all REPL functions.

#### Commands

| Command | Description |
|---------|-------------|
| `ls` | List columns, types, descriptions |
| `ls hist` | List stored histograms |
| `ls cal` | List loaded calibration objects |
| `info` | File metadata (rows, size, schema) |
| `lsID` | List all crystal IDs with hit counts |
| `print <col> [N] [S] [G(<gate>)]` | Print N rows starting at row S, optionally gated |
| `loadParquet <file> [file2 ...]` | Load or chain parquet files |
| `unloadParquet` | Free loaded file(s) |
| `loadCal <file.cal>` | Load gain/offset cal → callable `cal<gsid>` objects |
| `loadScript <script>` | Run a script: `.py` = full Python; otherwise custom CLI syntax (see Scripting below) |
| `rm <name>` | Delete stored object |
| `newWindow` | Next plot in a new window |
| `set <VAR> <value>` | Set a script variable; use `{VAR}` in subsequent commands (scripting) |
| `saveParquet <out.parquet> G(<gate>) [CS] [CAL(col, file.cal)]` | **Filter and save**: apply gate (required), optional CS flag, optional energy calibration → write surviving rows/events to new Parquet ✅ verified 2026-04-20 — `dgs_analysis/working/parquetCLI:L743-833` |

#### Plotting

| Command | Description |
|---------|-------------|
| `plot1D [CS] <col\|expr> [bw [xmin xmax]] [G(<gate>)]` | 1D histogram |
| `plot2D [col1 col2] [bw]` | 2D histogram (default: sum1 vs sum2) |
| `plotGG [CS] <col> [bw [xmin xmax]] [G(<gate>)]` | Gamma-gamma coincidence 2D (event-level only) — builds all pairwise hit combinations per event into a symmetric 2D matrix ✅ verified 2026-04-18 — `dgs_analysis/working/README.md:L76-78` |
| `plot <name> [name ...]` | Display stored histogram(s) |
| `overlay <name> [name ...]` | Overlay multiple 1D histograms |

`CS` flag = Compton suppression (drops hits with matching negative-ID veto hit in same event).

Store a histogram with `>`:
```
dgs> plot1D e_cal 1 0 4000 > spec
dgs> plot1D CS e_cal 1 0 4000 G(tid==6) > spec_cs
```

#### Gate syntax

Gates filter inline via `G(<expr>)` — combine with `&&` / `||`:
```
G(tid==6)                  single crystal
G(tid==6&&CSflag==0)       AND
G(e_cal>=500&&e_cal<1500)  energy window
```

#### Formula columns

Any Python expression using column names, numpy functions, and loaded calibrations:
```
dgs> plot1D "sum2-sum1" 1 0 5000
dgs> plot1D "e_cal*1.05" 1 0 4000 G(tid==6)
dgs> plot1D "cal88(e_raw)" 1 0 4000    # apply loaded calibration
```

Available: `sqrt`, `abs`, `log`, `log10`, `exp`, `sin`, `cos`, `np.*`

#### Fitting and calibration

```
dgs> fit spec 5                 # find+fit peaks above 5% of max
dgs> cal spec eu152             # auto-calibrate against 152Eu source
dgs> cal spec 10 207Bi Mx      # 10% threshold, anchor on highest-energy peak
```

Available sources: `am241`, `am243`, `ba133`, `co56`, `eu152`, `na24`, `se75`, `ta182`, `y88`, `207Bi`, etc.

```
dgs> loadCal working/dgs_gain.cal    # load per-crystal gain/offset
dgs> cal88 > dgs_gain.cal 88         # write cal to file (new)
dgs> cal89 >> dgs_gain.cal 89        # append to existing file
```

#### Histogram arithmetic

```
dgs> h1 + h2 > h3
dgs> h1 - h2 > hdiff
dgs> h1 * h2 > hprod
dgs> h1 / h2 > hratio
```

#### Saving / exporting

```
dgs> spec > spec.png          # save figure (png, pdf, svg, ...)
dgs> saveParquet out.parquet G(tid==6) CS CAL(e_raw, dgs_gain.cal)
# Note: G(<gate>) is required for saveParquet. CS applies CSflag==0 filter.
# CAL(col, file.cal) applies per-crystal gain/offset to <col>, adds e_cal column.
```

#### Scripting

**Custom syntax** (`.txt`): supports `set`, `for`/`endfor`, `if`/`endif`, `break`, `continue`:
```
set TIDS 6 7 8 9
for TID in {TIDS}
    plot1D e_cal 1 0 4000 G(tid=={TID}) > spec_{TID}
endfor
```

**Python scripts** (`.py`): full Python with access to the CLI session. Example `working/script.py` demonstrates a calibration-then-sort workflow:

```python
# Pass 1 (isCal=True): load run 001, fit 207Bi peaks per crystal, write dgs.cal
cmd("loadParquet Parquet/exp2008_001_0.parquet")
tids = lsID()
for tid in tids:
    cmd(f"plot1D e_raw/350 1 1000 6000 G(detID=={tid}) > h{tid}")
    cmd(f"cal h{tid} 207Bi > cal{tid}")
    cmd(f"cal{tid} > dgs.cal {tid}")   # first write; >> for subsequent

# Pass 2 (isCal=False): load run 005, apply cal, plot crystal 13
cmd("loadParquet Parquet/exp2008_005_0.parquet")
cmd("loadCal dgs.cal")
for tid in tids:
    cmd(f"plot1D cal{tid}(e_raw/350) 0.1 100 1000 G(detID=={tid}) > h{tid}")
cmd("plot h13")
```

Note `e_raw/350` — divides raw sum by trapezoid gap (350 samples) to get per-sample energy before applying the gain cal function.

**Pipe via stdin:** `./working/parquetCLI data.parquet < commands.txt`

#### Scripting — `if`/`endif` condition syntax
`if <cond>` / `endif` conditions in custom scripts support:
- `exists <name>` — true if a stored object named `<name>` exists
- `not exists <name>`
- `{VAR} == value`, `{VAR} != value`, `{VAR} > value`, etc.
✅ verified 2026-04-18 — `dgs_analysis/working/README.md:L214-217`

#### Interactive features
- Tab completion (commands, columns, filenames)
- **Argument hints** — press Tab at the start of each argument to see what's expected ✅ verified 2026-04-18 — `dgs_analysis/working/README.md:L229`
- Persistent history (`parquetCLI.history`)
- Rectangle zoom: left-drag to zoom, right-click to reset

_Source: `dgs_analysis/working/README.md` commit b609604 (2026-04-07)_

### gain_from_parquet.py

Extracts **energy gain/offset calibration** from a `_dgs.parquet` file. For each crystal (`tid`), builds a 1D `e_raw` spectrum, auto-fits gamma peaks using the GrayCAL `AutoFitter`, matches to a known calibration source, and performs a linear fit: `E_keV = gain * e_raw + offset`. Writes `dgs_gain.cal`.

```bash
python working/gain_from_parquet.py <dgs.parquet> [options]
  --output FILE      Output cal file          (default: dgs_gain.cal)
  --source FILE      JSON isotope source      (default: Euautocal.json = 152Eu)
  --e-bins N         e_raw histogram bins     (default: 4096)
  --e-max N          Upper bound for e_raw    (default: 99.5th percentile)
  --threshold F      Peak sigma threshold     (default: 3.0)
  --tid N [N ...]    Process only these crystal IDs (default: all)
  --quiet            Suppress per-crystal output
```

**Pipeline per crystal:**
1. Histogram `e_raw` (skips crystals with <500 hits) ✅ verified 2026-04-08 — `gain_from_parquet.py:L182`
2. `AutoFitter` finds + fits peaks (SNIP background, 20 iterations, min_fwhm=3 channels) ✅ verified 2026-04-08 — `gain_from_parquet.py:L91-94`
3. `CalibrationPoints.add_points_from_fit()` extracts peak centroids
4. `cal_pts.match_w_source()` matches to known isotope gamma lines
5. `cal_pts.energy_calibration()` → linear fit → `gain`, `offset`
6. Skips crystals with <2 matched peaks or failed fits

**Output format** (`dgs_gain.cal`):
```
# gsid  gain  offset
  1  0.123456  -12.3456
  2  0.124001   -9.8765
...
```

Uses `gray_apps/data/isotopes/sou-files/Euautocal.json` (152Eu) as default source.

### pz_from_parquet.py

Extracts **pole-zero calibration** from a decoded hit-level `_dgs.parquet` file. Reads `sum1` (S1 baseline trapezoid integral), `sum2` (S2 signal trapezoid integral), and `tid` (crystal ID). For each crystal, builds a 2D S1/S2 histogram then calls `pole_zero_fitter.estimate_pz_from_histogram()` to find the PZ coefficient. Writes `dgs_pz.cal` in GEBSort format.

Use with `--decode-only` RunParquet output (hit-level, not event-level):
```bash
python working/pz_from_parquet.py $expFolder/Parquet/decode/exp2008_003_dgs.parquet \
    --output working/dgs_pz.cal \
    --method chi2   # chi2 | peakmatch | pca | ridge
    --pz-min 0.88 --pz-max 0.99 --pz-step 0.0005
    --s1-bins 512 --s2-bins 512
```

**Pipeline per crystal:**
1. Read `sum1`, `sum2`, `tid` columns from parquet
2. Build 2D numpy histogram (S1 × S2, 512×512 default bins)
3. `estimate_pz_from_histogram()` → `DetResult` dataclass with `.pz` coefficient
4. `write_pz_cal(path, pz_map)` → `dgs_pz.cal` (format: `det  pz`, one line per crystal)

For event-level parquet (list columns), use `pz_from_evtparquet.py` instead.

### pz_from_evtparquet.py *(added Apr 2026)*

Extract pole-zero constants from **event-level** `.parquet` files (where `detID`, `sum1`, `sum2` are `list<>` columns rather than flat columns). Flattens lists to rows then runs the same 2D-histogram + PZ fitting pipeline per crystal. ✅ verified 2026-04-17 — `pz_from_evtparquet.py:L1-30` (docstring + argparse confirm interface)

```bash
python working/pz_from_evtparquet.py <file.parquet> [options]
  --output FILE      Output cal file (default: dgs_pz.cal)
  --method METHOD    chi2 | peakmatch | pca | ridge  (default: chi2)
  --pz-min/max/step  PZ scan bounds (default: 0.88–0.99, step 0.0005)
  --s1-bins/s2-bins  Histogram bins (default: 512 each)
  --detID N ...      Process only these crystal IDs (default: all)
  --quiet            Suppress per-crystal progress
```

### DownloadRaw.sh *(added Apr 2026)*

Copies raw GEB run data from NFS to local `expFolder/data/` via rsync. ✅ verified 2026-04-17 — `DownloadRaw.sh:L1-30` (header comments + nfsFolder check at L59 confirmed)

```bash
./working/DownloadRaw.sh [--dry-run] <expInfo.sh> <run_number> [run_number ...]
# e.g.: ./working/DownloadRaw.sh expInfo.sh 3 5 7
```

Requires `nfsFolder` defined in `expInfo.sh` (root of NFS data mount). Supports `--dry-run` to preview without copying.

### ProcessRUN *(primary pipeline — Apr 2026)*

Higher-level run processing wrapper. Drives **EventBuilder_X** (Parquet output, no ROOT dependency). Pole-zero and ROOT analysis steps are currently commented out in the script.

```bash
./working/ProcessRUN [expInfo.sh] <run_number> [BUILD] [ANALYSIS]
  expInfo.sh : experiment config (default: expInfo.sh in script dir)
  run_number : integer run number (e.g. 3 → 003)
  BUILD      : 1=build if stale (default), 0=skip, -1=force rebuild
  ANALYSIS   : 1=run ROOT analyzer (default), 0=skip
```

Sources `expInfo.sh` (from arg or script dir) for `expName`, `expFolder`, `dataFolder`. **Recommended setup:** create a symlink so the default path works without passing the arg every time:
```bash
ln -s ~/ANLDAQ/tcpReceiver/expInfo.sh working/expInfo.sh
```

**Key settings (hardcoded in script):**
- `timeWin=1000` ticks coincidence window ✅ verified 2026-04-09 — `ProcessRUN:L39`
- `useTrigTS=0`, `saveTrace=0`, `nMerge=40` threads ✅ verified 2026-04-09 — `ProcessRUN:L40-42`
- Output: `${expFolder}/Parquet/${expName}_${RUN}_0.parquet` (filename = `_${useTrigTS}.parquet`; `0` because useTrigTS=0) ✅ verified 2026-04-09 — `ProcessRUN:L66`
- Input: `${expFolder}/data/${expName}_${RUN}/${expName}_${RUN}*` (excludes `.geb` and `.txt`) ✅ verified 2026-04-09 — `ProcessRUN:L65`

**EventBuilder_X** — the builder used by ProcessRUN. Identical parallel k-way merge engine as EventBuilder_PQ, but writes **Apache Parquet** output instead of ROOT (no ROOT dependency required). Output schema (nested, one row = one coincidence event):

| Column | Type | Description |
|--------|------|-------------|
| `evID` | uint32 | Event index |
| `NumHits` | uint16 | Hits in this event |
| `id` | list\<uint16\> | Raw digitizer id (boardID×100 + channel) |
| `detID` | list\<int16\> | Mapped detector id (999=trigger, negative=BGO) |
| `sum1` | list\<uint32\> | Pre-rise trapezoidal sum S1 (24-bit) |
| `sum2` | list\<uint32\> | Post-rise trapezoidal sum S2 (24-bit) |
| `e_raw` | list\<float32\> | PZ-corrected energy: S2 − pz×S1 |
| `eventTS` | list\<uint64\> | EVENT_TIMESTAMP in ticks |
| `trigTS` | list\<uint64\> | TS_OF_TRIGGER_FULL in ticks |
| `trace*` | list (optional) | Waveform + GSL fit when saveTrace=1 |

_Source: `fastEventContructor/EventBuilder_X.cpp` header comment. Verified 2026-04-08._

### BenchmarkTAC2_021.sh

Performance comparison script for the three event-building approaches:
```bash
./working/BenchmarkTAC2_021.sh [THREADS]
```
- Runs `EventBuilder_Q`, `EventBuilder_PQ`, and `parquet_pysort` against the `TAC2_021` dataset
- Reports wall time + output size for each; prints Q/PQS, PQ/PQS, and Q/PQ speedup ratios
- Default: 8 threads; uses `TIMEWIN=1000` (ticks) for fair apples-to-apples comparison
- Requires ROOT environment (`/opt/root/bin/thisroot.sh`) and Python venv (`.venv/`)
- Output log: `working/benchmark_output/benchmark_<timestamp>.log`

---

## Data Format: GEB (GRETINA Event Builder)

Raw DGS binary files use the GEB format:
- **GEBHeader** — contains type, length, timestamp
- **Payload** — digitizer data (DIG struct): board_id, channel_id, energies, timestamp, waveform trace (optional)

`TRASH_DATA` markers in files are skipped via inline `if (HEADER_TYPE == TRASH_DATA) continue` logic. `TRASH_DATA = 666` is a sentinel value set when `DecodeData()` fails to parse a valid header. ✅ verified 2026-04-09 — `class_Hit.h:L28` (`#define TRASH_DATA 666`), `class_Hit.h:L79` (set on decode failure), `EventBuilder_Q.cpp:L163` (skip check)

### GEB Type Codes
_Source: `DGS_SVN/dgs/gtReceiver/dgsReceiver/dgsReceiver.cpp` — list from Torben (GRETINA), as of 2021-12-07_

| Code | GEB Type | Notes |
|------|----------|-------|
| 1 | `GEB_TYPE_DECOMP` | Decomposed GRETINA |
| 2 | `GEB_TYPE_RAW` | Raw GRETINA |
| 3 | `GEB_TYPE_TRACK` | Tracked |
| 4 | `GEB_TYPE_BGS` | BGS detector |
| 5 | `GEB_TYPE_S800_RAW` | S800 spectrometer raw |
| 6 | `GEB_TYPE_NSCLnonevent` | NSCL non-event |
| 7 | `GEB_TYPE_GT_SCALER` | GRETINA scaler |
| 8 | `GEB_TYPE_GT_MOD29` | GRETINA Module 29 |
| 9 | `GEB_TYPE_S800PHYSDATA` | S800 physics data |
| 10 | `GEB_TYPE_NSCLNONEVTS` | NSCL non-events |
| 11 | `GEB_TYPE_G4SIM` | Geant4 simulation |
| 12 | `GEB_TYPE_CHICO` | CHICO detector |
| **14** | **`GEB_TYPE_DGS`** | **DGS digitizer data** |
| **15** | **`GEB_TYPE_DGSTRIG`** | **DGS trigger data** |
| 16 | `GEB_TYPE_DFMA` | DFMA (Digital Fast Multiplicity Array) |
| 17 | `GEB_TYPE_PHOSWICH` | Phoswich detector |
| 18 | `GEB_TYPE_PHOSWICHAUX` | Phoswich aux |
| 19 | `GEB_TYPE_GODDESS` | GODDESS array |
| 20 | `GEB_TYPE_LABR` | LaBr3 detector |
| 21 | `GEB_TYPE_LENDA` | LENDA neutron detector |
| 22 | `GEB_TYPE_GODDESSAUX` | GODDESS aux |
| 23 | `GEB_TYPE_XA` | X-Array |
| 24 | `MAX_GEB_TYPE` | |

Note: type 13 is absent (unassigned). DGS uses types 14 (digitizer hits) and 15 (trigger data).

---

## Connections to Other Subsystems

- **ANLDAQ** — `tcpReceiverMT` writes the raw binary files that `dgs_analysis` reads
- **ioc/** — the IOC munch file determines what data fields are available in the payload
- **fpga/** — DIG firmware determines signal processing (energy, timestamp, trace format)

---

## EventBuilder_PQ Benchmark Results
_Source: `fastEventContructor/README.md`. NVMe Kingston SFYRD4000G (~7 GB/s rated)._

### TAC2_054 — Large run (17.4 GB, 157 files, ~228M hits)

| Builder | Config | Scan | Merge | Total | Read Rate | Write Rate |
|---------|--------|------|-------|-------|-----------|------------|
| EventBuilder_S | 30 workers | — | — | 659.7s | — | — |
| EventBuilder_PQ | 12 merge, 4 writers | 0.3s | 23.7s | **32.2s** | 735 MB/s | 127 MB/s |

EventBuilder_PQ verified against EventBuilder_S: both produce 26,656,402 events. ✅ verified 2026-04-12 — `fastEventContructor/README.md`

### TAC2_021 — (16 GB, 158 files, ~60M hits, nWriters=4)

**Top-level comparison:**

| Builder | Config | Wall time | Output |
|---------|--------|-----------|--------|
| EventBuilder_Q | 4 workers | 58.0s | 734M ROOT |
| EventBuilder_PQ (no ReadPool) | 12 merge, 4 writers | 37.4s | 734M ROOT |
| EventBuilder_PQ (full Scan + ReadPool) | 12 merge, 4 writers | 19.0s | 734M ROOT |
| EventBuilder_PQ (QuickBounds + ReadPool) | 12 merge, 4 writers | **13.6s** | 734M ROOT |

✅ verified 2026-04-11 — `fastEventContructor/README.md` (TAC2_021 benchmark table). QuickBounds variant added; replaces old full-scan pre-pass with per-file header sampling for faster sector partitioning.

**EventBuilder_PQ thread scaling (with ReadPool + optimizations, nWriters=4):**

| nMerge | Scan | Merge | Total | Read Rate | Write Rate |
|--------|------|-------|-------|-----------|------------|
| 1 | 8.0s | 30.2s | 40.1s | 517 MB/s | 25 MB/s |
| 2 | 7.8s | 17.0s | 27.0s | 919 MB/s | 44 MB/s |
| 4 | 7.9s | 10.9s | 21.4s | 1433 MB/s | 69 MB/s |
| 8 | 7.9s | 8.8s | **19.2s** | 1774 MB/s | 86 MB/s |
| 12 | 7.9s | 8.5s | **19.0s** | 1837 MB/s | 89 MB/s |
| 16 | 7.7s | 8.9s | 19.1s | 1754 MB/s | 85 MB/s |
| 24 | 7.6s | 10.7s | 20.6s | 1460 MB/s | 71 MB/s |

**Sweet spot: 8–12 merge threads** (~1.8 GB/s read, ~89 MB/s write). Beyond 12 threads, NVMe saturates and gains plateau.

Key improvements with ReadPool (vs without):
- ~3× faster merge phase (26s → 9s at 16 threads)
- ~3× higher read rate (590 MB/s → 1.8 GB/s)
- Double-buffered pre-fetch eliminates I/O wait in merge loop
- Conditional trace arrays reduce per-event memory from ~163 KB to ~11 KB

---

## Notes

- Save traces (`saveTrace=true`) allocates ~153 KB per event — keep disabled for large runs unless needed
- Parquet pipeline is an alternative to ROOT for Python-native workflows
- `angtheta.dat` and `map.dat` are geometry files mapping detector IDs to physical positions

---

## Cross-References

| Topic | File |
|-------|------|
| GEBSort full reference (all programs, GEBSort.chat, find_MK, fwhm_onepeak, dgs_ecal) | `knowledgeBase/gebsort.md` |
| Typical DGS run procedures (GEBSort workflow + modern ANLDAQ workflow) | `knowledgeBase/run_procedures.md` |
| Pole-zero correction theory + `pz_from_parquet.py` detail | `knowledgeBase/pole_zero.md` |
| GEB binary data format + GEBHeader struct | `knowledgeBase/data_structures.md` |
| Gammasphere geometry (GS hole → θ/φ, map.dat context) | `knowledgeBase/gammasphere_geometry.md` |
| DIG firmware readout modes (source of raw GEB payloads) | `knowledgeBase/DIG_firmware_expert.md` |
| ROOT analysis scripts (analyzer_*.cpp, Cali_e, checkTACFile, findMapping/findGS) | `knowledgeBase/dgs_analysis_root_scripts.md` |
| gray_apps toolkit (GrayCAL, GrayMAN, grayfit) | `knowledgeBase/dgs_analysis_grayapps.md` |

---

## analyzer_*.cpp — ROOT Analysis Scripts

> **Split to separate file** (2026-04-18) — see [`dgs_analysis_root_scripts.md`](dgs_analysis_root_scripts.md) for full documentation of:
> - `analyzer.cpp` — standard γ-γ coincidence analysis
> - `analyzer_tac.cpp` — TAC-II coincidence analysis
> - `analyzer_trace.cpp` — waveform trace / PSD analysis
> - `analyzer_pz_cal.cpp` — pole-zero calibration from traces
> - `analyzer_script.cpp` — trace shape ROOT script
> - `Cali_e.C` — interactive energy calibration (110 detectors)
> - `checkTACFile.cpp` — TAC-II binary file validator
> - `findMapping.sh` / `findGS.sh` — GS channel map tools

<!--  original section removed 2026-04-18; see dgs_analysis_root_scripts.md  -->

*Source: `DGS_tools_pack/dgs_analysis/` + `DGS_tools_pack/gebsort/`. Updated: 2026-04-18 (ROOT scripts split to dgs_analysis_root_scripts.md).*
