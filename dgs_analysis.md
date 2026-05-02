# dgs_analysis — DGS Analysis Code Collection

Stability: C2 - Active / semi-stable

## Table of Contents
- [What It Is](#what-it-is)
- [Repository](#repository)
- [armory/ — Reusable Tools](#armory--reusable-tools)
  - [fastEventConstructor](#fasteventconstructor)
  - [parquet_pysort — Python/C++ Parquet Pipeline](#parquet_pysort--pythonc-parquet-pipeline)
  - [gray_apps](#gray_apps)
  - [`misc.h` — Shared Utility Functions](#misch--shared-utility-functions)
  - [`EventBuilder_XR` — Parallel k-way Merge → ROOT TTree Output](#eventbuilder_xr--parallel-k-way-merge--root-ttree-output)
- [working/ — Experiment-Specific Scripts & Calibration](#working--experiment-specific-scripts--calibration)
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
| `EventBuilder_XR` | Same engine as PQ/X, **ROOT TTree output**, optional GSL trace fitting + PZ correction | When ROOT output needed without Arrow/Parquet build dependency |

**EventBuilder_S details:**
- **Parallel scan pre-pass**: all input files are scanned in parallel threads (one thread per file) via `BinaryReader::Scan(true)`; reports hit counts, file sizes, and global timestamp range (earliest + latest) ✅ verified 2026-04-17 — `EventBuilder_S.cpp:L342-360`
- **Single-threaded k-way merge**: priority queue over all groups (files grouped by DigID); initial batch loaded from each group, then merge loop reads next hits as needed
- **Trace analysis option** (`nWorkers` arg): when `nWorkers > 0`, each event's waveform traces are fit to a **logistic sigmoid model**: `Yi = A / (1 + exp(-(t - T0) / riseTime)) + B` using the GNU Scientific Library (GSL) nonlinear least-squares (`gsl_multifit_nlinear_trust`). 4 parameters per trace: A (amplitude), T0 (timing offset), riseTime (pulse rise time), B (baseline). Up to 100 iterations, convergence at `1e-5`. Chi² residual also stored. ✅ verified 2026-04-17 — `EventBuilder_S.cpp:L202-263`
- **Multithreaded trace fitting**: when `nWorkers > 1`, events with traces are dispatched to a worker thread pool via a condition-variable queue (`dataQueue`); an ordered output map (`outputMap` keyed by evID) ensures in-order ROOT tree writes. When `nWorkers == 1`, trace fitting runs single-threaded inline.
- **ROOT output**: same branch layout as EventBuilder_Q (evID, NumHits, id, detID, pre/post_rise_energy, eventTS, trigTS); trace branches (`traceCount`, `traceDetID`, `traceLen`, `trace[][]`) added when `saveTrace=1`; `tracePara[traceCount][4]` and `traceChi2[traceCount]` added when `nWorkers > 0` ✅ verified 2026-04-17 — `EventBuilder_S.cpp:L403-420`
- **Usage**: `./EventBuilder_S [outfile] [timeWindow] [useTrigTS] [saveTrace] [nWorkers] [file1] [file2] ...` — note `nWorkers` is the 5th arg (vs `nWorkers` in Q), but semantics differ: in S it controls trace analysis thread count
- **Benchmark**: 659.7s for 17.4 GB / 228M hits (TAC2_054, 30 workers) — much slower than PQ (32.2s). EventBuilder_S output verified to match PQ: 26,656,402 events ✅ verified 2026-04-12 — `README.md`

**EventBuilder_Q optimizations:**
- Batch pre-decoding (not per-hit) ✅ verified 2026-04-26 — `EventBuilder_Q.cpp:L95-99` (`GroupReader: pre-decodes batches, async pre-fill, no double decode`)
- ReadPool: async pre-fills next batch while current is consumed (double buffering) ✅ verified 2026-04-26 — `EventBuilder_Q.cpp:L99-124` (ReadPool class); `L190,202` (startBackFill on every decode)
- Lightweight merge heap: `{timestamp, groupIndex}` only (16 bytes vs full DIG struct) ✅ verified 2026-04-26 — `EventBuilder_Q.cpp:L228-237` (struct MergeEntry); `L595` (comment: `16 bytes: {timestamp, groupIdx}`)
- 4 pipelined ROOT writers (round-robin) ✅ verified 2026-04-26 — `EventBuilder_Q.cpp:L37` (`#define N_WRITERS 4`); `L500` (`ch = item->evID % N_WRITERS` — round-robin assignment)

**EventBuilder_PQ optimizations (extends Q):**
- **QuickBounds phase**: reads only the first+last GEB header of each file to get per-file timestamp bounds and compute hit count from `fileSize / blockSize`. Sector seeking uses binary search on the file (fixed block size → any hit N is at byte offset `N × blockSize`). Falls back to full scan for files with inconsistent payload sizes. ✅ verified 2026-04-26 — `BinaryReader.h:L48,L271-302` (QuickBounds: first+last GEB header read; `totalNumHits = fileSize/blockSize`); `L538-558` (binary search on file using blockSize); `L292` (payload mismatch → fall back to `Scan(true)`)
- Sector partitioning: divides time span into N sectors with ghost regions (width = `timeWindow`) at boundaries to avoid splitting coincidence events ✅ verified 2026-04-26 — `EventBuilder_PQ.cpp:L3,8` (file header comments); `L726-751` (sector boundary computation)
- N parallel merge threads × M pipelined writers per thread (N×M partial files, merged at end) ✅ verified 2026-04-26 — `EventBuilder_PQ.cpp:L552,763-775` (sector threads launched; `nWriters` writers per sector)
- Per-sector double-buffered ReadPool ✅ verified 2026-04-26 — `EventBuilder_PQ.cpp:L771,782` (`ReadPool sectorPool(1)` per sector thread)
- Hits whose first event falls in a ghost region are discarded to prevent duplicates across sector boundaries ✅ verified 2026-04-26 — `EventBuilder_PQ.cpp:L916-917` (`firstTS < bounds.startTS || firstTS >= bounds.endTS` → discard); `L1001,1088` (discard counter reported)

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
- ⚠️ GrayMAN `peak_detection.py` is a known placeholder — use GrayCAL's `AutoFitter` for production ✅ verified 2026-04-26 — `gray_apps/src/GrayMAN/core/peak_detection.py`: `find_peak_centroids` defined twice (second overwrites first), body comment "Need to replace this with something more realistic", uses naive `np.argmax` with no actual fitting; not integrated into any GUI caller
- `Euautocal.json` (16 152Eu lines, 121.8–1408.0 keV) is the default calibration source

> **Full details:** → [`dgs_analysis_grayapps.md`](dgs_analysis_grayapps.md)

_Source: `dgs_analysis/armory/gray_apps/src/Fitter/` — explored 2026-04-07_

### `misc.h` — Shared Utility Functions

_Source: `dgs_analysis/armory/fastEventContructor/misc.h` (303 lines, code-read 2026-04-23)_

Header-only file included by `EventBuilder`, `EventBuilder_X`, `EventBuilder_XR`, and the ROOT analyzer scripts. Defines global state (channel/energy/PZ maps) and the complete pole-zero correction pipeline.

#### Global State

| Variable | Type | Description |
|----------|------|-------------|
| `channelMap` | `map<ushort, map<ushort, short>>` | `boardID → channelID → detID` (positive=HPGe, negative=BGO) |
| `VMEDIGtoBoard` | `map<ushort, map<ushort, ushort>>` | `VME → digID → boardID` |
| `numberValidDetectors` | `ushort` | Total valid HPGe detectors loaded |
| `energyCalSlope` / `energyCalIntercept` | `map<ushort, float>` | Per-detector energy calibration |
| `pzCal` | `map<short, float>` | Per-detID PZ constant (from `dgs_pz.cal`) |
| `pzCorrectExp` | `map<ushort, float>` | Per-detector exponential decay term `e^(-λ·M)` (from `GS_pz_correct_exp.txt`) |
| `pzCorrectFactor` | `map<ushort, float>` | Per-detector slope from Vdc vs. S1 plot (from `GS_pz_correct_factor.txt`) |
| `pre_rise_energy_last` | `map<ushort, uint>` | Last S1 (pre-rise energy) per detector — used by Algo 2 high-rate extrapolation |
| `vdc_last` | `map<ushort, float>` | Last computed Vdc per detector — used by both Algo 1 and Algo 2 |
| `timestamp_last` | `map<ushort, ullong>` | Last event timestamp per detector — used by Algo 1 |

#### Loader Functions

| Function | Input File | Description |
|----------|-----------|-------------|
| `LoadChannelMapFromFile()` | `GS_channel_map.txt` | Loads `boardID/chID → detID` map; skips 2 header rows; `chID+5` for HPGe, `chID` for BGO (negative detID) |
| `LoadEnergyCalFromFile()` | `GS_energy_cal.txt` | Per-detector `intercept slope` lines → `energyCalSlope`/`energyCalIntercept` |
| `LoadPZCalFromFile()` | `dgs_pz.cal` | Per-detector PZ constant; skips `#`/`/`/blank lines; returns `false` if file absent (e_raw = S2−S1) |
| `LoadPZCorrectionFromFile()` | `GS_pz_correct_exp.txt` + `GS_pz_correct_factor.txt` | Loads per-detector exp + factor maps for Algo 2 |

#### Lookup Functions

| Function | Description |
|----------|-------------|
| `FindVMEDIGFromBoardID(boardID)` | Reverse lookup: `boardID → (VME, digID)` pair |
| `FindBoardIDFromDetID(detID)` | Returns all `boardID`s associated with a `detID` |
| `FindVMEDIGCHFromDetID(detID)` | Returns flat `[VME, DIG, CH, ...]` list for all hits mapping to `detID` |

#### Pole-Zero Correction Functions

**`CalculateVdcAlgo1(pre_rise_energy, event_timestamp, detID) → float`**
- Simple time-gap approach: if `Δt ≥ 450 µs` since last event for this detector, assume signal has fully decayed; compute `vdc = S1 / M` where `M=350` (sample time in ticks). If `Δt < 450 µs`, reuse cached `vdc_last[detID]`.
- Updates `timestamp_last[detID]` and `vdc_last[detID]` on each call.
- **Weakness:** coarse — uses only inter-event timing, no baseline sample. ✅ verified 2026-04-23 — `misc.h:L212-236`

**`CalculateVdcAlgo2(pre_rise_energy, baseline, deltaT, detID) → float`**  _(default, `PZ_ALGO=2`)_
- Two-regime approach using sampled baseline (`SAMPLED_BASELINE` from DIG) and `deltaT` (time since last discriminator in µs):
  - **Normal rate** (`Δt ≥ 20 µs`): computes `vdc` analytically using `pzExp` (`e^(-λ·M)`) and `pzFactor` from the calibration files. Full formula: `vdc = ((baseline + S1)·(1-pzExp^(m/M)) - baseline·(1-pzExp^((M+m)/M))) / ((M+m)·(1-pzExp^(m/M)) - m·(1-pzExp^((M+m)/M)))` where `M=350` (pre-rise sample time), `m=1024` (baseline sample time).
  - **High rate** (`Δt < 20 µs`): extrapolates: `vdc = vdc_last + pzFactor × (S1 - S1_last)` using cached values.
  - Saves `vdc_last`/`S1_last` when `20 µs ≤ Δt ≤ 50 µs` (for future high-rate extrapolation).
- ✅ verified 2026-04-23 — `misc.h:L238-273`

**`PoleZeroCorrection(pre_rise_energy, post_rise_energy, vdc, detID) → float`**
- Applies the correction once `vdc` is known: `energy = (1/M)·(S2 - S1·pzExp^((M+K)/M)) - (1-pzExp^((M+K)/M))·vdc` where `M=350`, `K=141` (rise time), `pzExp = pzCorrectExp[detID]`.
- Returns `0` if `vdc ≤ 10` (unphysical baseline; correction skipped).
- ✅ verified 2026-04-23 — `misc.h:L276-303`

> **Which algorithm is active?** Controlled by `#define PZ_ALGO` in each EventBuilder: currently `PZ_ALGO=2` (Algo2) in `EventBuilder_X.cpp`, `EventBuilder_XR.cpp`, and `EventBuilder.cpp`. `PZ_ALGO=1` selects Algo1; any other value uses raw `SAMPLED_BASELINE` directly as Vdc.

### `EventBuilder_XR` — Parallel k-way Merge → ROOT TTree Output

_Source: `dgs_analysis/armory/fastEventContructor/EventBuilder_XR.cpp` (960 lines, code-read 2026-04-23)_

Same parallel-sector engine as `EventBuilder_X` (QuickBounds pre-scan, ghost regions, double-buffered `ReadPool` per sector), but outputs a **ROOT TTree** instead of Apache Parquet. No Arrow/Parquet build dependency required.

**TTree branches (one entry per event):**

| Branch | Type | Description |
|--------|------|-------------|
| `evID` | `UInt_t` | Event ID (sequential per sector, with per-sector base offset) |
| `NumHits` | `UInt_t` | Number of hits in this event |
| `id[NumHits]` | `UShort_t[]` | Raw digitizer ID (`boardID*100 + channel`) |
| `detID[NumHits]` | `Short_t[]` | Mapped detector ID (999=trigger, negative=BGO, 0=unmapped) |
| `sum1[NumHits]` | `UInt_t[]` | Pre-rise trapezoidal sum (S1), full 24-bit |
| `sum2[NumHits]` | `UInt_t[]` | Post-rise trapezoidal sum (S2), full 24-bit |
| `e_raw[NumHits]` | `Float_t[]` | PZ-corrected energy from `CalculateVdcAlgo2 + PoleZeroCorrection` (ADC counts) |
| `baseline[NumHits]` | `UInt_t[]` | `SAMPLED_BASELINE` from DIG packet |
| `eventTS[NumHits]` | `ULong64_t[]` | `EVENT_TIMESTAMP` in ticks |
| `trigTS[NumHits]` | `ULong64_t[]` | `TS_OF_TRIGGER_FULL` in ticks |
| `deltaT[NumHits]` | `Double_t[]` | Time since last discriminator (µs); `(EVENT_TIMESTAMP - LAST_DISC_TIMESTAMP) / 100` |
| `traceCount` | `UShort_t` | Number of traces saved (when `saveTrace=1`) |
| `traceDetID[traceCount]` | `UShort_t[]` | Det ID per trace |
| `traceLen[traceCount]` | `UShort_t[]` | Length of each trace |
| `trace[traceCount][1250]` | `UShort_t[][]` | Waveform samples (max 1250 per trace) |
| `tracePara[traceCount][4]` | `Float_t[][]` | GSL sigmoid fit: [A, T0, riseTime, B] |
| `traceChi2[traceCount]` | `Float_t[]` | Chi² residual of sigmoid fit |

**Key differences vs `EventBuilder_X` (Parquet):**
- ROOT TTree output (requires ROOT; no Arrow/Parquet)
- Includes `e_raw` (PZ-corrected energy, `PZ_ALGO=2`) + raw `baseline` + `deltaT` — richer per-hit data than Parquet schema
- Sector output files written to `/dev/shm` (same as X), merged with `TFileMerger` at end
- Optional GSL sigmoid trace fitting (same model as EventBuilder_S/X)

**Usage:** `./EventBuilder_XR [outfile] [timeWindow] [useTrigTS] [saveTrace] [nMerge] [file-1] [file-2] ...`
- `timeWindow` — coincidence window in ticks (10 ns each)
- `useTrigTS` — 0=use EVENT_TIMESTAMP, 1=use TS_OF_TRIGGER_FULL for coincidence
- `saveTrace` — 0=no traces, 1=save waveform traces
- `nMerge` — number of parallel sector threads

**When to use:** When the EventBuilder_X parallel-sector speed is needed but downstream tools require ROOT TTree format (not Parquet), and the build environment doesn't have Arrow installed. ✅ verified 2026-04-23 — `EventBuilder_XR.cpp:L1-30,L497-535,L787-797`

---

## working/ — Experiment-Specific Scripts & Calibration

> **Split to separate file** (2026-04-25) — see [`dgs_analysis_working.md`](dgs_analysis_working.md) for full documentation of:
> - `RunParquet` (legacy Python pipeline)
> - `parquetCLI` (interactive Parquet REPL with plotting, gating, fitting, scripting)
> - `gain_from_parquet.py` (energy calibration from parquet hits)
> - `pz_from_parquet.py` / `pz_from_evtparquet.py` (pole-zero calibration)
> - `DownloadRaw.sh` (NFS data download via rsync)
> - `ProcessRUN` (primary pipeline — drives EventBuilder_X, Parquet output)
> - `BenchmarkTAC2_021.sh` (performance comparison script)

All paths driven by `expInfo.sh` from `~/ANLDAQ/tcpReceiver/expInfo.sh`. Primary workflow (Apr 2026): `ProcessRUN` → `EventBuilder_X` (Parquet). `RunParquet` is legacy.

<!--  working/ section moved to dgs_analysis_working.md 2026-04-25  -->

> **Full working/ documentation:** → [`dgs_analysis_working.md`](dgs_analysis_working.md)

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

## Cross-References

| Topic | File |
|-------|------|
| GEBSort full reference (all programs, GEBSort.chat, find_MK, fwhm_onepeak, dgs_ecal) | [gebsort.md](gebsort.md) |
| Typical DGS run procedures (GEBSort workflow + modern ANLDAQ workflow) | [run_procedures.md](run_procedures.md) |
| Pole-zero correction theory + `pz_from_parquet.py` detail | [pole_zero.md](pole_zero.md) |
| GEB binary data format + GEBHeader struct | [data_structures.md](data_structures.md) |
| Gammasphere geometry (GS hole → θ/φ, map.dat context) | [gammasphere_geometry.md](gammasphere_geometry.md) |
| DIG firmware readout modes (source of raw GEB payloads) | [DIG_firmware_expert.md](DIG_firmware_expert.md) |
| ROOT analysis scripts (analyzer_*.cpp, Cali_e, checkTACFile, findMapping/findGS) | [dgs_analysis_root_scripts.md](dgs_analysis_root_scripts.md) |
| gray_apps toolkit (GrayCAL, GrayMAN, grayfit) | [dgs_analysis_grayapps.md](dgs_analysis_grayapps.md) |
| working/ scripts (parquetCLI, ProcessRUN, RunParquet, gain/pz_from_parquet, DownloadRaw, Benchmark) | [dgs_analysis_working.md](dgs_analysis_working.md) |

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

*Source: `DGS_tools_pack/dgs_analysis/` + `DGS_tools_pack/gebsort/`. Updated: 2026-04-18 (ROOT scripts split to dgs_analysis_root_scripts.md). MD organization 2026-04-24: fixed heading levels for misc.h + EventBuilder_XR; ToC de-duplicated. MD organization 2026-04-25: working/ section split to dgs_analysis_working.md (475 → 475 lines after split).*
