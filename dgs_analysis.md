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
| `EventBuilder_X` | Same engine as PQ, **Parquet output** (no ROOT dependency) | **Primary pipeline** — used by `ProcessRUN` |

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

**In practice:** use `working/ProcessRUN` (C++ EventBuilder, primary) or `working/RunParquet` (Python parquet_pysort, legacy) — both driven from `expInfo.sh`.

**Ultimate goal:** Roaring bitmap index — for each energy bin, a roaring bitmap stores the set of `event_id`s containing a hit, enabling rapid energy-gated coincidence queries without scanning the full dataset.

**Threading details:**
- `decode.py`: `ThreadPoolExecutor`, one worker per tid; GE/BGO files submitted as sub-tasks for overlapping I/O. Requires **Python 3.14.3t (free-threaded / no-GIL)** for true parallelism. ✅ verified 2026-04-14 — `parquet_pysort/CLAUDE.md:L63` ("Python 3.14.3t (free-threaded) — No-GIL build"); `README.md:L145` ("Python 3.12+ — free-threaded build (3.14t) recommended")
- `event_builder.py`: Reads all input into one Arrow table, splits into N chunks, calls C++ `build_events()` per chunk in parallel. Column renames (`header_ts→gs_ts`, etc.) are zero-copy Arrow references — no `.to_pylist()`.
- `decode.py --write-threads N`: Output split into `_000.parquet`, `_001.parquet`, … — feed multiple files to `event_builder.py`.

**Algo notes:**
- Algo 0: simple `sum2/MM - sum1/MM` (no pole-zero, no baseline).
- Algo 1 (SZ_1, low-rate): Baseline via exponential avg (`BASE_ALPHA=0.01`, updated only when `dtev ≥ 250 µs`). Energy = `sum2/MM - sum1/MM * pz1 - base*(1-pz1)` where `pz1 = PZ^(1/MM)`. Both require `base > 10.0` for nonzero energy.
- Algo 2 (SZ_2, high-rate): Uses `pz4 = PZ^((MM+KK)/MM)`. Two regimes based on `dtev` vs `dgs_SZ_t1`/`dgs_SZ_t2`: `≥ t2` computes baseline from `sampled_baseline` (FPGA-sampled, 24-bit, `MSAMPLE=1024` = 10.24 µs window) using PZ decay formula; `< t2` extrapolates from pre-learned `baselast`/`sum1last` factor. Energy formula same as SZ_1 but uses `pz4`. Requires `base > 10.0`.
- `dtev`: computed from firmware `last_disc_timestamp` (time since last discriminator trigger), with wrap-around correction. Matches `lastdisc_dt_ticks()` in `bin_dgs.c`.
- `sum2` field extraction in `jta.c` is header-type-dependent: types 0/1/3/5 use `>> 28` (4-bit); types 4/6/7/8 use `>> 24` (8-bit).
- `pileup_count` extraction in `jta.c` has a known bit-shift bug (`(word5 & 0x00FFC000) >> 24`) that always produces 0; replicated as-is for compatibility.

**GEBSort reference:** `GEBSort.cxx:GEBGetEv()` — coincidence grouping: `while ((TS - curTS) < dTS)`. Default `dTS=500` (= 5 µs at 10 ns/tick). ✅ verified 2026-04-09 — `GEBSort.cxx:L2502` (`Pars.dTS = 500`). `jta.c:DGSEvDecompose_v3()` parses payloads (big-endian swap, 48-bit timestamp from words 1+2, `trigger_timestamp` only in header types 7/8).

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
| `eu152.sou` | 152Eu (full) | 22 | 121.8 – 1408.0 |
| `eu_autocal.sou` | 152Eu (autocal subset) | 16 | 121.8 – 1408.0 |
| `co56.sou` | 56Co | 14 | 846.8 – 3451.2 |
| `ba133.sou` | 133Ba | 9 | 53.2 – 383.9 |
| `am241.sou` | 241Am | 4 | 26.3 – 59.5 |
| `am243.sou` | 243Am | 11 | 43.5 – 334.3 |
| `na24.sou` | 24Na | 6 | 511.0 – 2754.0 |
| `y88.sou` | 88Y | 2 | 898.0 – 1836.1 |
| `ta182.sou` | 182Ta | 19 | 31.7 – 1231.0 |
| `se75.sou` | 75Se | 9 | 66.1 – 400.7 |
| `calib.sou` | mixed (calib) | 14 | 121.8 – 1836.1 |

✅ verified 2026-04-13 — line counts and E ranges from `data/isotopes/sou-files/*.sou` (awk on energy column)

`Euautocal.json` is the default for `gain_from_parquet.py` — 16 strong 152Eu lines from 121.8 to 1408.0 keV covering the full Ge dynamic range.

#### `SpectrumData.py` — Unified Spectrum Container

Central data object for GrayCAL (author: M.P. Carpenter, 2025-09-04). Stores a raw spectrum plus derived results; used by GUI tabs, fitters, and serialization.

**Supports:** 1D spectra (energies + counts arrays) and N-D spectra (counts ndarray + tuple of energy arrays per axis).

**Key attributes:**
- `energies`, `counts` — core data (numpy arrays)
- `background_counts`, `residual_counts` — SNIP background + (counts − background)
- `model_counts` — full model overlay (background + fitted peaks)
- `peakfind_centroids` — peak-finder output centroids
- `fit_limits` — fit window bounds
- `fit_results` — list of `FitResult` objects
- `fit_session_collection` — full `AutoFitter` session output

**I/O:** `to_json()` / `from_json()` and HDF5 (`h5py`) export/import for session persistence.

**Plotting:** Plotly-based `plot()` method returns `go.Figure`; used by GUI `SpectrumViewer`.

_Source: `gray_apps/src/GrayCAL/core/SpectrumData.py` (720 lines) ✅ verified 2026-04-13 — wc -l confirms 720_

#### `CalibrationPoints.py` — Peak-to-Source Matching & Energy Calibration

The key workhorse for `gain_from_parquet.py`. Holds fitted peaks and performs the full calibration pipeline. Author: S. Carmichael (2025-11-04).

**Key methods:**
- `add_points_from_fit(fit_res, isotope_name, binwidth)` — extracts peak centroids (µ, µ_err), counts, widths from `AutoFitter` results into internal lists
- `match_w_source(isotope, source_data, active_en)` — matches fitted centroids to known gamma lines using `select_match_lines()` (selects most-intense lines spanning ≥50% of energy range, then scales to match fit count)
- `energy_calibration()` — linear fit `E_keV = gain × channel + offset` via `scipy.optimize.curve_fit`; stores in `self.en_par = [gain, offset]` + `self.en_red_chi2`
- `efficiency_calibration()` — optional RadWare efficiency fit per detector; stores in `self.eff_par`

**Helper: `select_match_lines(active_en, active_ri)`** — selects strongest source lines that span ≥50% of total energy range (starts at RI > 90% of max, walks down to 10% until ≥3 lines span enough range). Used to choose which literature lines to attempt matching against.

**Key attributes:** `num_pts`, `centers[]`, `centers_unc[]`, `gamma_en[]` (literature values), `matched[]` (bool per peak), `en_par` ([gain, offset] after calibration), `en_red_chi2`, `eff_par`.

_Source: `gray_apps/src/GrayCAL/core/CalibrationPoints.py` (756 lines) ✅ verified 2026-04-13 — wc -l confirms 756_

#### `SyntheticSpectrumGenerator.py` — Synthetic Gamma-Ray Spectrum Generator

Generates synthetic gamma-ray spectra for testing/validation of the fitting and calibration pipeline. Written by M.P. Carpenter (mpcarp).

**Energy-dependent resolution:** `sigma(E) = sqrt(a + b×E + c×E²)` with defaults a=1, b=5e-3, c=1e-6 (keV units). Models realistic HPGe peak broadening vs energy.

**Key methods:**
- `generate_random_peaks(start, stop, n_peaks, min_distance, min_amp, max_amp)` — random spectrum with N peaks, minimum separation enforced; uses `evaluate_model2` (GF3-style: Gaussian + skew tail + step background)
- `generate_isotope_spectrum(isotope_data, efficiency_curve, ...)` — physics-based spectrum from `IsotopeGammaData` + `radware_eff` efficiency; peaks scaled by relative intensity × efficiency at that energy
- `set_tail(r, beta, step_height)` — configures asymmetric left-tail parameters (r=tail fraction, beta=decay constant, step_height=Compton step)
- `add_background(bg_type, ...)` — adds flat/linear/exponential background
- `plot()` — matplotlib visualization

**Used for:** Unit testing `AutoFitter`, validating `CalibrationPoints.match_w_source()` with known peak positions, and demonstrating the full GrayCAL pipeline without real data.

_Source: `gray_apps/src/GrayCAL/core/SyntheticSpectrumGenerator.py` (458 lines) ✅ verified 2026-04-13 — wc -l confirms 458_

#### `spectrum_types.py` — Spectrum Dataclass

Defines the `Spectrum` dataclass: `E_bins` (numpy array of energy bin edges in keV), `counts` (numpy array), `label` (optional string), `metadata` (dict). Serialization methods: `save_to_json/load_from_json`, `save_to_npz/load_from_npz`, `save_to_hdf5/load_from_hdf5`. Used as the shared spectrum container across GrayCAL and GrayMAN.

#### `FileReader.py` — Multi-format Spectrum File Reader

Unified file reader for GrayCAL GUI (134 lines, by S. Carmichael). Supports `.root`, `.txt`, and `.spe` files.

- **Constructor:** Takes `filepath`; auto-detects format from extension; for ROOT files, scans and lists all TH1 and TH2 histogram names via `uproot`.
- **`get_hist_data(histname)`**: Returns `(centers, counts)` arrays. ROOT: uses `uproot` + `hist.to_numpy()`; TXT: `np.loadtxt` (2-column: energy, counts); SPE: delegates to `spedata.rdspe()` (counts only, centers = 0-indexed).
- **`get_spectrum_data()`**: Iterates all known 1D+2D histogram names, returns list of `SpectrumData` objects.
- Used by GrayCAL GUI when user opens a spectrum file for viewing/calibration.

#### `spedata.py` — RadWare `.spe` File Reader/Writer

Legacy Python class for reading and writing RadWare `.spe` binary spectrum files (gf3/GEBSort format). Uses Fortran-style record framing (4-byte size prefix + suffix around each record). Header: 8-char name + 4 ints (idim=nbins, 1, 1, 1). Spectrum: `idim` 32-bit floats.

- `rdspe()` — reads `.spe` file, returns spectrum as tuple of floats; sets `self.spec`, `self.idim`, `self.name`
- `wrspe(spename, spec)` — writes spectrum to `.spe` file with given name (padded to 8 chars) and float array

Useful for importing/exporting spectra to/from RadWare tools (gf3, levit8r, etc.) and GEBSort.

#### `BatchFit.py` — Batch Calibration Container

Holds calibration results from multiple histograms (e.g. all detector IDs in one run). Stores:
- `ids` — list of detector IDs processed
- `cal_spectra` — dict of `SpectrumData` per ID
- `cal_collections` — dict of `FitSessionCollection` per ID
- `cal_pts_dict` — dict of `CalibrationPoints` per ID

Key methods:
- `get_cal_centroids()` — returns per-gamma-line dict of `{energy_str: ([ids], [cal_en], [cal_en_res])}`; only for matched/calibrated detectors
- `gen_rand_factors(num)` — test utility: generates random gain factors per ID (uniform center 0.5–1.5, σ=0.01)

Author: Scott Carmichael (2025-12-11). Used by `BatchfitDialog` GUI (Tools → BatchFit).

_Source: `dgs_analysis/armory/gray_apps/` — explored 2026-04-06; radware_eff + isotope data explored 2026-04-08; spectrum_types + spedata explored 2026-04-11; BatchFit explored 2026-04-13_

#### GrayCAL GUI — `main_window.py`

**Entry point:** `gray_apps/src/GrayCAL/gui/main_window.py` (1,200+ lines, PyQt6). `MainWindow(QMainWindow)` is the top-level app window.

**Layout:**
- **Left sidebar** — `QTreeView`-based tree with file nodes, isotope nodes, spectrum nodes, fit result groups, fit session items, and fit result items. Role constants: `KIND_FILE_ROOT`, `KIND_SPECTRUM_ITEM`, `KIND_FIT_SESSION_ITEM`, `KIND_FIT_RESULT_ITEM`, etc.
- **Right panel** — tabbed: spectrum viewer (Plotly or matplotlib), fits viewer, calibration viewer, data tab (isotope line table)
- **Menu bar** — File→Open (ROOT/SPE), View, Tools (AutoFit, BatchFit, PoleZero, BackgroundFit, SpectrumGeneration)

**Key workflows:**
1. Open spectrum file → `FileReader` (ROOT/txt/SPE) → `SpectrumData` objects → sidebar tree
2. Select isotope → `IsotopeGammaData` loads JSON → active gamma lines displayed in data tab
3. AutoFit → `AutofitDialog` → `AutoFitter` → fit results displayed in sidebar + `FitsViewer`
4. Match peaks → `on_match_button_click()` → `CalibrationPoints.match_w_source()` → calibration points table
5. Calibrate → `CalibrationPoints.energy_calibration()` → gain/offset → `CalibrationViewer`
6. BatchFit → `BatchfitDialog` → `BatchFit` core → bulk calibration across many spectra

**Dialogs:** `AutofitDialog`, `BatchfitDialog`, `PoleZeroDialog`, `BackgroundFitDialog`, `SpectrumGenerationDialog`, `PlotControlDialog`, `CalPtsWindow` (calibration points editor), `PeakRangeDialog`.

**File formats supported:** ROOT (via uproot), `.spe` (RadWare/Maestro text format), `.txt` (channel→counts). SPE loaded via `spedata` + `_load_spe_to_spectrumdata()` adapter.

_Source: `gray_apps/src/GrayCAL/gui/main_window.py` (explored 2026-04-11)_

#### `polezero_dialog.py` — Pole-Zero Extraction GUI Dialog

`PoleZeroDialog(QDialog)` — PyQt6 dialog for configuring and running PZ extraction from a 2D S1/S2 histogram loaded in GrayCAL. Wraps `estimate_pz_from_histogram()` from `pole_zero_fitter.py`.

**Parameter groups (shown in scrollable dialog):**
- **Scan Parameters** — `PZParams` coarse scan settings (pz_min, pz_max, pz_step) + gate settings
- **Refinement** — fine-scan settings from `PZParams`
- **Method selector** — chi2 / peakmatch / pca / ridge (combo box)
- **PCA Parameters** — `PCAParams` (hidden unless method=pca)
- **PeakMatch Params** — `PeakMatchParams` (hidden unless method=peakmatch)
- **Ridge Params** — `RidgeParams` (hidden unless method=ridge)
- **Axis Mode / Misc** — 2D histogram axis options

**Workflow:**
1. User selects a 2D spectrum (S1 vs S2 histogram) in the GrayCAL sidebar
2. Opens via Tools → PoleZero menu
3. Configures method + params in dialog
4. Clicks Run → `run_extraction()` calls `estimate_pz_from_histogram(h2, s1_edges, s2_edges, params)` → returns `DetResult` with `.pz` coefficient
5. Result shown in dialog; Save button calls `write_pz_cal(path, pz_map)` → `.cal` file

**Key detail:** Dialog reads the current spectrum directly from parent `MainWindow`; validates it's a 2D histogram before allowing run. Stores last `DetResult` in `self._last_result` for save.

_Source: `gray_apps/src/GrayCAL/gui/polezero_dialog.py` (494 lines, explored 2026-04-12)_

### GrayMAN — Gamma-Ray MultiPeak Analyzer Network

**Full name:** GrayMAN (Gamma-Ray MultiPeak Analyzer Network)

A separate PyQt6 GUI application for multi-peak gamma-ray spectrum analysis. Uses matplotlib (not Plotly) for rendering. Distinct from GrayCAL — focuses on peak detection, fitting, and analysis rather than calibration workflow.

**Structure:**

| Path | Contents |
|------|----------|
| `src/GrayMAN/core/` | `SpectrumData`, `fitting`, `peak_detection`, `snip`, `spectrum_model` |
| `src/GrayMAN/gui/` | `GammaSpectrumAnalyzer` (main window), `canvas` (matplotlib), `matrix_viewer`, `dialogs` |
| `src/GrayMAN/utils/` | Utilities |

**Features (updated 2026-04-14, main_window.py = 1,289 lines):**
- PyQt6 `GammaSpectrumAnalyzer` main window: horizontal splitter (sidebar left, canvas+command right)
- **3 tabs:** "1D Spectrum" (main matplotlib view + toolbar), "Fitter View" (dedicated fitting canvas), "2D Coincidence Matrix" (`MatrixViewer` widget)
- Sidebar: spectrum type selector, background subtraction controls, peak finding/fitting buttons, display settings
- Command window: text-based command interface (`execute_command()`)
- Dialogs: `open_generate_spectrum_dialog`, `open_background_subtraction_dialog`, `open_find_peaks_dialog`, `open_set_fitting_limits_dialog`, `open_fit_peaks_dialog`, display settings
- SNIP background subtraction (`core/snip.py`)
- Matrix/2D spectrum viewer (`gui/matrix_viewer.py`, 135 lines)

**Core module notes (explored 2026-04-11):**
- **`snip.py`** — SNIP background estimation: iterative windowed min-filter with optional LLS (Log-Log-Sqrt) transform for stabilisation; `m` iterations controls smoothness. Returns `(y_corrected, background)`. Clean, complete implementation.
- **`peak_detection.py`** — `find_peak_centroids()` searches fixed-separation peaks in a window around expected positions (simple local-max approach). Has an in-code `TODO: "Need to replace this with something more realistic"` — **placeholder, not production-ready**. `set_fitting_limits()` finds regions exceeding `threshold_factor × noise_level` above background, merges gaps ≤ `max_gap`, extends by ±5σ each side.
- **`spectrum_model.py`** — Gaussian peak model utilities for fitting.
- **`fitting.py`** — Multi-peak fitting using the spectrum model.

> ⚠️ GrayMAN is less mature than GrayCAL — `peak_detection.py` has a known placeholder. Use GrayCAL's `AutoFitter` (from the `Fitter/grayfit/` library) for production peak finding. ✅ verified 2026-04-13 — `GrayMAN/core/peak_detection.py:L2` (`pass`) + L5 (`"""Need to replace this with something more realistic"""`); function is duplicated (stub then basic impl).

**Run:** `python src/GrayMAN/main.py`

_Source: `dgs_analysis/armory/gray_apps/src/GrayMAN/` — explored 2026-04-06; core internals verified 2026-04-11_

### grayfit — Shared Fitting Library

**Location:** `src/Fitter/grayfit/`  
**Purpose:** Shared gamma-ray spectrum fitting library, importable by both GrayCAL and GrayMAN via `from grayfit import ...`. Packaged with `pyproject.toml` and installable via `pip install -e src/Fitter/`.  
**Author:** M.P. Carpenter (Aug 2025)

| Module | Role |
|--------|------|
| `gamma_spectrum_fitter.py` | `GammaSpectrumFitter` — main peak fitting class |
| `FitResult.py` | **Three-level fit result hierarchy:** `FitResult` → `FitSession` → `FitSessionCollection`. `FitResult`: one fit iteration — holds `params` (list of dicts with name/value/fixed/shared_id), `cov`, `chi2`, `chi2_r`, `aic`, `bic`, `residual_rms`, `fit_region`, `model_type`, `sigma_mode`. `FitSession`: one fit region (e.g. one peak cluster), stores multiple `FitResult` iterations; `get_best_fit()` picks by `best_result_iteration` if set, else lowest AIC → chi2\_r → chi2 → residual\_rms. `FitSessionCollection`: container for all sessions from one spectrum; iterates over sessions. Round-trip JSON-serializable (numpy scalars/arrays auto-converted). |
| `fitting_runner.py` | **`FittingRunner`** — iterative peak-finding + fitting engine. Takes a `GammaSpectrumFitter` + `PeakFinder` and runs a loop: (1) detect highest peak (≥10σ above background); (2) fit all current peaks; (3) check residuals for additional peaks; (4) add peaks one at a time and re-fit; (5) stop when `residual_rms < residual_rms_threshold` (default 0.01), `chi2_r` converges (improvement < `min_improvement`=1e-3), or `max_iterations` (default 10) / `max_peaks` (default 30) hit. Supports `model_type='gaussian'` or `'gf3'`; `sigma_mode='independent'` (per-peak σ), `'dependent'` (shared σ), or `'fixed'` (σ from calibration function). Selects best result across iterations by AIC. Used by `AutoFitter.fit_all_regions()`. |
| `ModelEvaluator.py` | `evaluate_model2`, `evaluate_modelN` — evaluate peak models |
| `peak_finder.py` | **`PeakFinder`** — heuristic peak detection using `scipy.signal.find_peaks`. Constructor params: `min_height` (min peak height, default None), `min_distance` (min channel separation, default 1), `min_sigma` (min peak width in σ, default 1.0). Key methods: `suggest_peaks_src(x, y_residual, height_thresh, y_err, existing_mus, max_peaks)` — finds peaks in residual (or normalized residual if `y_err` provided); returns list of `(peak_idx, mu)` tuples sorted by prominence; skips peaks near `existing_mus`. `suggest_peak(x, y_residual, existing_mus)` — returns single best new peak not overlapping existing ones. `score_peak(height, width, prominence, residual_rms)` — heuristic quality score: 0.6×(h/rms) + 0.3×(w/σ) + 0.1×(prom/rms); also computes dynamic height threshold. Used by `FittingRunner` in iterative add-one-peak loop. |
| `auto_fitter.py` | **`AutoFitter`** — automatic peak finding + fitting without user guidance. Supports 1D and 2D spectra. Pipeline: (1) SNIP background estimation (`BackgroundFitter`, `snip_1d_mod` or `snip_2d_mod`); (2) threshold-based peak detection (`PeakFinder`) with residual = (data−bg)/√bg; (3) local FWHM estimation per region; (4) region expansion by `fwhm_multiplier`×FWHM on both sides; (5) N-D region merging. Key params: `threshold` (σ above bg, default 1), `snip_iterations` (default 20), `min_fwhm_channels` (default 3), `fwhm_multiplier` (default 2). Threshold mask: `residual > threshold * sqrt(max(bg,1)) + 10` (fixed +10 floor prevents noise triggers near zero). Returns `(regions, background, residual)` from `identify_regions()`; then `fit_all_regions()` returns `FitSessionCollection`. |
| `background_fitter.py` | Background estimation / subtraction |
| `pole_zero_fitter.py` | **PZ coefficient extraction** from 2D S1/S2 histograms (1,342 lines). Refactored from `pz_from_S1S2_current_v6.py` to work on NumPy arrays (no ROOT I/O). Public API: `estimate_pz_from_histogram(h2, s1_edges, s2_edges, params)` → `DetResult` dataclass; `write_pz_cal(path, pz_map)` / `read_pz_cal(path)` → GEBSort-style "det  pz" calibration files. **Four methods:** `chi2` (default — coarse+refine grid scan minimizing χ² of S2 energy spectrum), `peakmatch` (peak-shape matching across S1 slices), `pca` (PCA/orthogonal regression on S1/S2 point cloud), `ridge` (ridge tracking across S1 slices). **`PZParams` defaults:** `pz_min=0.930, pz_max=0.990, pz_step=0.0005`; refine halfwidth=0.002, step=0.0001; `e_bins=8192`. Note: `pz_from_parquet.py` overrides to `pz_min=0.88`. |
| `visualization.py` | Plotting helpers for spectra and fit results. **`FitVisualizer.display_fit(fit_result, x, y)`** — 2-panel matplotlib figure (data+fit top, residuals bottom). **`display_fit_components()`** / **`display_fit_components_2()`** — show individual peak components (Gaussians + background) as separate curves. **`draw_fit_on_axes()`** — draw onto caller-provided axes (no new figure). **`PlotStyle`** dataclass controls labels/linewidths/alpha. All functions use `evaluate_model` / `evaluate_model2` from `ModelEvaluator.py`. Author: mpcarp (M.P. Carpenter). |
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

---

*Source: `DGS_tools_pack/dgs_analysis/` + `DGS_tools_pack/gebsort/`. Updated: 2026-04-07.*
