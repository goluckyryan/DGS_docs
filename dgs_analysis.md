# dgs_analysis — DGS Analysis Code Collection

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
| `EventBuilder` | Original; priority queue, parallel file scan, no intermediate files | General use |
| `EventBuilder_S` | Pre-scan pass for timestamp ranges; single-threaded k-way merge | When timestamp bounds needed |
| `EventBuilder_Q` | Async double-buffered k-way merge; batch pre-decoding; pipelined ROOT writers | Fast I/O, large datasets |
| `EventBuilder_PQ` | Parallel k-way merge with sector partitioning; N parallel threads | Maximum throughput, multi-core |

**EventBuilder_Q optimizations:**
- Batch pre-decoding (not per-hit)
- ReadPool: async pre-fills next batch while current is consumed (double buffering)
- Lightweight merge heap: `{timestamp, groupIndex}` only (16 bytes vs full DIG struct)
- 4 pipelined ROOT writers (round-robin)

**EventBuilder_PQ optimizations (extends Q):**
- Pre-scan phase: reads only GEB headers to get per-file timestamp bounds + seek index
- Sector partitioning: divides time span into N sectors with ghost regions at boundaries
- N parallel merge threads × M pipelined writers per thread
- Per-sector double-buffered ReadPool

```sh
# Build
make                    # all variants
make EventBuilder_Q     # single variant
make EventBuilder_PQ

# Usage
./EventBuilder_Q [outfile] [timeWindow] [useTrigTS] [saveTrace] [nWorkers] [file1] [file2] ...
```

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

**Event builder:** timestamp coincidence window `(ts − first_ts) < timewin`; N parallel threads with boundary merge and global `event_id` assignment.

**Key files:**
| File | Role |
|------|------|
| `decode.py` | Stage 2 decoder (drives C++ lib) |
| `dgs_decode_lib.cpp` | C++ shared lib — pole-zero, energy, calibration |
| `geb_format.py` | GEB header/payload format definitions |
| `event_builder.py` | Stage 3 coincidence event builder |
| `make_filemap_dgs.py` | Stage 1 filemap builder |
| `read_parquet.py` | Inspect output (`--info`, `--head`, `--where`, `--to-csv`) |
| `PQDecode.chat` | Decode config: algo, MM, KK, threads, etc. |
| `PQMerge.chat` | Merge config |
| `dgs_gain.cal` | Gain calibration |
| `dgs_pz.cal` | Pole-zero calibration |
| `angtheta.dat` | Angle/theta lookup (for Doppler correction) |
| `map.dat` | DAQ id → tid/tpe mapping |

**Build C++ lib:**
```bash
g++ -O2 -std=c++17 -shared -fPIC -o libdgs.so dgs_decode_lib.cpp
```

**In practice:** use `working/ProcessRUN` (C++ EventBuilder, primary) or `working/RunParquet` (Python parquet_pysort, legacy) — both driven from `expInfo.sh`.

**Ultimate goal:** Roaring bitmap index — for each energy bin, a roaring bitmap stores the set of `event_id`s containing a hit, enabling rapid energy-gated coincidence queries without scanning the full dataset.

**Threading details:**
- `decode.py`: `ThreadPoolExecutor`, one worker per tid; GE/BGO files submitted as sub-tasks for overlapping I/O. Requires **Python 3.14.3t (free-threaded / no-GIL)** for true parallelism.
- `event_builder.py`: Reads all input into one Arrow table, splits into N chunks, calls C++ `build_events()` per chunk in parallel. Column renames (`header_ts→gs_ts`, etc.) are zero-copy Arrow references — no `.to_pylist()`.
- `decode.py --write-threads N`: Output split into `_000.parquet`, `_001.parquet`, … — feed multiple files to `event_builder.py`.

**Algo notes:**
- Algo 1 (SZ_1, low-rate): Baseline via exponential avg (`alpha=0.01`), only updated when `dtev ≥ 250 µs`.
- Algo 2 (SZ_2, high-rate): Two regimes based on `dtev` vs `dgs_SZ_t1`/`dgs_SZ_t2`; extrapolates baseline from pre-learned factor when `dtev < dgs_SZ_t2`. Both require `base > 10.0` for nonzero energy.
- `pileup_count` extraction in `jta.c` has a known bit-shift bug that always produces 0; replicated as-is for compatibility.

**GEBSort reference:** `GEBSort.cxx:GEBGetEv()` — coincidence grouping: `while ((TS - curTS) < dTS)`. Default `dTS=500`. `jta.c:DGSEvDecompose_v3()` parses payloads (big-endian swap, 48-bit timestamp from words 1+2, `trigger_timestamp` only in header types 7/8).

_Source: `dgs_analysis/armory/parquet_pysort/CLAUDE.md` + `README.md` — updated 2026-04-07_

### gray_apps

**GrayCAL** — Python GUI toolkit for gamma-ray energy calibration of HPGe detectors. Written by M.P. Carpenter (mpcarp19). Requires Python 3.13+.

**Status:** Active development; not for external distribution (see README_GrayCAL.md).

**Install:**
```bash
conda env create -f environment.yml && conda activate grayapps
# or
./setup_venv.sh && pip install -e .
```

**Run:** `graycal` (after install) or `python main.py`

**Structure (62 Python files):**

| Path | Contents |
|------|----------|
| `src/GrayCAL/core/` | Core logic: `SpectrumData`, `BatchFit`, `CalibrationPoints`, `IsotopeGammaData`, `SyntheticSpectrumGenerator`, `FileReader`, `radware_eff` |
| `src/GrayCAL/gui/` | PyQt/Plotly GUI: `SpectrumViewer`, `FitsViewer`, `CalibrationViewer`, `autofit_dialog`, `batchfit_dialog`, `polezero_dialog`, `calpts_window`, etc. |
| `src/GrayCAL/utils/` | Settings, file I/O, utilities |
| `data/isotopes/` | Known gamma-ray source data (JSON format) |

**Features:**
- Load/manage radioactive source data (JSON)
- Visualize HPGe gamma-ray spectra interactively (Plotly-based)
- Automated peak fitting (`autofit`, `batchfit`)
- Pole-zero correction dialog
- Calibration point management and energy calibration
- Synthetic spectrum generation (random or isotope-based) for testing
- Radware efficiency (`radware_eff.py`) support

**Planned:** Isotopic source identification, efficiency calibration, Doppler-broadened peak fitting.

#### `radware_eff.py` — Detector Efficiency Curve

Implements the RadWare detector efficiency function. Two-piece log-linear model covering low- and high-energy regions with a smooth turnover.

- **`eff_function(x, *par)`**: Evaluates log-efficiency at energies `x` (keV). Fixed pivot points: p7=100 keV, p8=1000 keV. Two linear regions (`f1 = p0 + p1*ln(E/100)`, `f2 = p3 + p4*ln(E/1000)`) joined by `turnover_function()`. Free parameters: `par[0..4]` (slopes/intercepts) + `par[6]` (turnover sharpness). Returns efficiency (not log).
- **`turnover_function(f, r, g)`**: Smooth interpolation between the two linear regions: `logy = f / (r^g + 1)^(1/g)`.
- Fit via `scipy.optimize.curve_fit`; author: S. Carmichael (2025-11-08).

#### `IsotopeGammaData` — Source Line Data

Loads JSON calibration source files from `data/isotopes/sou-files/`. JSON format: `{"isotope": "...", "lines": [{"E_gamma": ..., "dE": ..., "RI": ..., "dRI": ...}, ...]}`.

Available sources and key lines:

| File | Isotope | Lines | E range (keV) |
|------|---------|-------|---------------|
| `Euautocal.json` | 152Eu (auto-cal) | 16 | 121.8 – 1408.0 |
| `eu152.sou` / `Calib.json` | 152Eu (full) | — | — |
| `co56.sou` | 56Co | — | — |
| `ba133.sou` | 133Ba | — | — |
| `am241.sou` | 241Am | — | — |
| `na24.sou` | 24Na | — | — |
| `y88.sou` | 88Y | — | — |
| `ta182.sou` | 182Ta | — | — |
| `se75.sou` | 75Se | — | — |
| `am243.sou` | 243Am | — | — |

`Euautocal.json` is the default for `gain_from_parquet.py` — 16 strong 152Eu lines from 121.8 to 1408.0 keV covering the full Ge dynamic range.

_Source: `dgs_analysis/armory/gray_apps/` — explored 2026-04-06; radware_eff + isotope data explored 2026-04-08_

### GrayMAN — Gamma-Ray MultiPeak Analyzer Network

**Full name:** GrayMAN (Gamma-Ray MultiPeak Analyzer Network)

A separate PyQt6 GUI application for multi-peak gamma-ray spectrum analysis. Uses matplotlib (not Plotly) for rendering. Distinct from GrayCAL — focuses on peak detection, fitting, and analysis rather than calibration workflow.

**Structure:**

| Path | Contents |
|------|----------|
| `src/GrayMAN/core/` | `SpectrumData`, `fitting`, `peak_detection`, `snip`, `spectrum_model` |
| `src/GrayMAN/gui/` | `GammaSpectrumAnalyzer` (main window), `canvas` (matplotlib), `matrix_viewer`, `dialogs` |
| `src/GrayMAN/utils/` | Utilities |

**Features:**
- Interactive matplotlib-based spectrum viewer with sidebar + command window
- Multi-peak fitting and automatic peak detection (`core/fitting.py`, `core/peak_detection.py`)
- SNIP background subtraction (`core/snip.py`)
- Matrix/2D spectrum viewer (`gui/matrix_viewer.py`)

**Run:** `python src/GrayMAN/main.py`

_Source: `dgs_analysis/armory/gray_apps/src/GrayMAN/` — explored 2026-04-06_

### grayfit — Shared Fitting Library

**Location:** `src/Fitter/grayfit/`  
**Purpose:** Shared gamma-ray spectrum fitting library, importable by both GrayCAL and GrayMAN via `from grayfit import ...`. Packaged with `pyproject.toml` and installable via `pip install -e src/Fitter/`.  
**Author:** M.P. Carpenter (Aug 2025)

| Module | Role |
|--------|------|
| `gamma_spectrum_fitter.py` | `GammaSpectrumFitter` — main peak fitting class |
| `FitResult.py` | `FitResult` dataclass — structured fit output |
| `fitting_runner.py` | `FittingRunner` — orchestrates batch/auto fitting runs |
| `ModelEvaluator.py` | `evaluate_model2`, `evaluate_modelN` — evaluate peak models |
| `peak_finder.py` | `PeakFinder` — automatic peak detection in spectra |
| `auto_fitter.py` | **`AutoFitter`** — automatic peak finding + fitting without user guidance. Supports 1D and 2D spectra. Pipeline: (1) SNIP background estimation (`BackgroundFitter`, `snip_1d_mod` or `snip_2d_mod`); (2) threshold-based peak detection (`PeakFinder`) with residual = (data−bg)/√bg; (3) local FWHM estimation per region; (4) region expansion by `fwhm_multiplier`×FWHM on both sides; (5) N-D region merging. Key params: `threshold` (σ above bg, default 1), `snip_iterations` (default 20), `min_fwhm_channels` (default 3), `fwhm_multiplier` (default 2). Returns `(regions, background, residual)` from `identify_regions()`; then `fit_all_regions()` returns `FitSessionCollection`. |
| `background_fitter.py` | Background estimation / subtraction |
| `pole_zero_fitter.py` | **PZ coefficient extraction** from 2D S1/S2 histograms (1,303+ lines). Refactored from `pz_from_S1S2_current_v6.py` to work on NumPy arrays (no ROOT I/O). Public API: `estimate_pz_from_histogram(h2, s1_edges, s2_edges, params)` → `DetResult` dataclass; `write_pz_cal(path, pz_map)` / `read_pz_cal(path)` → GEBSort-style "det  pz" calibration files. |
| `visualization.py` | Plotting helpers for spectra and fit results |
| `utils.py` | Shared utilities |
| `debug_utils.py` | Debug/diagnostic helpers |

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
| `TIMEWIN` | 1000 | Coincidence window in ticks |
| `THREADS` | 78 | Threads for decode + event builder |

**Output:**
- `$expFolder/Parquet/decode/$expName_NNN_dgs.parquet` — timestamp-sorted hits
- `$expFolder/Parquet/events/$expName_NNN_events.parquet` — coincidence events

**Parquet schema:**

| File | Key Columns |
|------|-------------|
| `_dgs.parquet` | `tid`, `header_ts`, `trigger_ts`, `sum1`, `sum2`, `e_raw`, `e_cal`, `e_dc`, `CSflag`, `pileup_count` |
| `_events.parquet` | `event_id`, `gs_mult`, `gs_hitid`, `gs_ts`, `gs_cryid`, `gs_eraw`, `gs_ecal`, `gs_edc`, `gs_flag` |

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
| `rm <name>` | Delete stored object |
| `newWindow` | Next plot in a new window |

#### Plotting

| Command | Description |
|---------|-------------|
| `plot1D [CS] <col\|expr> [bw [xmin xmax]] [G(<gate>)]` | 1D histogram |
| `plot2D [col1 col2] [bw]` | 2D histogram (default: sum1 vs sum2) |
| `plotGG [CS] <col> [bw [xmin xmax]] [G(<gate>)]` | Gamma-gamma coincidence 2D (event-level only) |
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
```

#### Scripting

**Custom syntax** (`.txt`): supports `set`, `for`/`endfor`, `if`/`endif`, `break`, `continue`:
```
set TIDS 6 7 8 9
for TID in {TIDS}
    plot1D e_cal 1 0 4000 G(tid=={TID}) > spec_{TID}
endfor
```

**Python scripts** (`.py`): full Python with access to the CLI session.

**Pipe via stdin:** `./working/parquetCLI data.parquet < commands.txt`

#### Interactive features
- Tab completion (commands, columns, filenames)
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

### pz_from_evtparquet.py *(new Apr 2026)*

Extract pole-zero constants from **event-level** `.parquet` files (where `detID`, `sum1`, `sum2` are `list<>` columns rather than flat columns). Flattens lists to rows then runs the same 2D-histogram + PZ fitting pipeline per crystal.

```bash
python working/pz_from_evtparquet.py <file.parquet> [options]
  --output FILE      Output cal file (default: dgs_pz.cal)
  --method METHOD    chi2 | peakmatch | pca | ridge  (default: chi2)
  --pz-min/max/step  PZ scan bounds (default: 0.88–0.99, step 0.0005)
  --s1-bins/s2-bins  Histogram bins (default: 512 each)
  --detID N ...      Process only these crystal IDs (default: all)
  --quiet            Suppress per-crystal progress
```

### DownloadRaw.sh *(new Apr 2026)*

Copies raw GEB run data from NFS to local `expFolder/data/` via rsync.

```bash
./working/DownloadRaw.sh [--dry-run] <expInfo.sh> <run_number> [run_number ...]
# e.g.: ./working/DownloadRaw.sh expInfo.sh 3 5 7
```

Requires `nfsFolder` defined in `expInfo.sh` (root of NFS data mount). Supports `--dry-run` to preview without copying.

### ProcessRUN *(updated Apr 2026)*

Higher-level run processing wrapper — event building + pole-zero extraction + analysis for one run.

```bash
./working/ProcessRUN [expInfo.sh] <run_number> [BUILD] [ANALYSIS]
  BUILD     : 1=build if stale (default), 0=skip, -1=force rebuild
  ANALYSIS  : 1=run ROOT analyzer (default), 0=skip
```

Sources `expInfo.sh` (from arg or script dir) for `expName`, `expFolder`, `dataFolder`.

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

`TRASH_DATA` markers in files are skipped via `skipTrash()` cursor logic.

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
_Source: `fastEventContructor/README.md`. Test dataset: TAC2_021, 16 GB, 158 files, ~60M hits. NVMe Kingston SFYRD4000G (~7 GB/s rated). nWriters=4._

**Top-level comparison:**

| Builder | Config | Wall time | Output |
|---------|--------|-----------|--------|
| EventBuilder_Q | 4 workers | 58.0s | 734M ROOT |
| EventBuilder_PQ (no ReadPool) | 12 merge, 4 writers | 37.4s | 734M ROOT |
| EventBuilder_PQ (with ReadPool) | 12 merge, 4 writers | 19.0s | 734M ROOT |

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
| GEBSort full reference (all programs, GEBSort.chat, find_MK, fwhm_onepeak, dgs_ecal) | `dgs/gebsort.md` |
| Typical DGS run procedures (GEBSort workflow + modern ANLDAQ workflow) | `dgs/run_procedures.md` |
| Pole-zero correction theory + `pz_from_parquet.py` detail | `dgs/pole_zero.md` |
| GEB binary data format + GEBHeader struct | `dgs/data_structures.md` |
| Gammasphere geometry (GS hole → θ/φ, map.dat context) | `dgs/gammasphere_geometry.md` |
| DIG firmware readout modes (source of raw GEB payloads) | `dgs/DIG_firmware_expert.md` |

---

*Source: `DGS_tools_pack/dgs_analysis/` + `DGS_tools_pack/gebsort/`. Updated: 2026-04-07.*
