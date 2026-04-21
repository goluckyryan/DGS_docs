# gray_apps — GrayCAL, GrayMAN, grayfit

> **Split from `dgs_analysis.md`** (2026-04-16). Full context: see [`dgs_analysis.md`](dgs_analysis.md).

`gray_apps` is a Python analysis toolkit in `dgs_analysis/armory/gray_apps/`. It contains three components:
- **GrayCAL** — HPGe energy calibration GUI (M.P. Carpenter)
- **GrayMAN** — multi-peak spectrum analysis GUI
- **grayfit** — shared fitting library used by both GUIs

---

## GrayCAL

**Purpose:** Python GUI toolkit for gamma-ray energy calibration of HPGe detectors. Written by M.P. Carpenter (mpcarp19). Requires Python 3.13+.

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

---

### Core Modules

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

Unified file reader for GrayCAL GUI (134 lines ✅ verified 2026-04-17 — `wc -l`, by S. Carmichael). Supports `.root`, `.txt`, and `.spe` files.

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

Author: Scott Carmichael (2025-12-11 ✅ verified 2026-04-17 — `BatchFit.py` header comment). Used by `BatchfitDialog` GUI (Tools → BatchFit).

_Source: `dgs_analysis/armory/gray_apps/` — explored 2026-04-06; radware_eff + isotope data explored 2026-04-08; spectrum_types + spedata explored 2026-04-11; BatchFit explored 2026-04-13_

---

### GrayCAL GUI — `main_window.py`

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

_Source: `gray_apps/src/GrayCAL/gui/main_window.py` (1,331 lines ✅ verified 2026-04-17 — `wc -l`; previously listed as "1,200+", now confirmed 1,331 — file has grown since initial exploration on 2026-04-11)_

### `polezero_dialog.py` — Pole-Zero Extraction GUI Dialog

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

_Source: `gray_apps/src/GrayCAL/gui/polezero_dialog.py` (494 lines ✅ verified 2026-04-17 — `wc -l`, explored 2026-04-12)_

---

## GrayMAN — Gamma-Ray MultiPeak Analyzer Network

**Full name:** GrayMAN (Gamma-Ray MultiPeak Analyzer Network)

A separate PyQt6 GUI application for multi-peak gamma-ray spectrum analysis. Uses matplotlib (not Plotly) for rendering. Distinct from GrayCAL — focuses on peak detection, fitting, and analysis rather than calibration workflow.

**Structure:**

| Path | Contents |
|------|----------|
| `src/GrayMAN/core/` | `SpectrumData`, `fitting`, `peak_detection`, `snip`, `spectrum_model` |
| `src/GrayMAN/gui/` | `GammaSpectrumAnalyzer` (main window), `canvas` (matplotlib), `matrix_viewer`, `dialogs` |
| `src/GrayMAN/utils/` | Utilities |

**Features (updated 2026-04-14, main_window.py = 1,289 lines ✅ verified 2026-04-17 — `wc -l GrayMAN/gui/main_window.py`):**
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

---

## grayfit — Shared Fitting Library

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
| `auto_fitter.py` | **`AutoFitter`** — automatic peak finding + fitting without user guidance. Supports 1D and 2D spectra. Pipeline: (1) SNIP background estimation (`BackgroundFitter`, `snip_1d_mod` or `snip_2d_mod`); (2) threshold-based peak detection (`PeakFinder`) with residual = (data−bg)/√bg; (3) local FWHM estimation per region; (4) region expansion by `fwhm_multiplier`×FWHM on both sides; (5) N-D region merging. Key params: `threshold` (σ above bg, default 1), `snip_iterations` (default 20), `min_fwhm_channels` (default 3), `fwhm_multiplier` (default 2). Threshold mask: `residual > threshold * sqrt(max(bg,1)) + 10` (fixed +10 floor prevents noise triggers near zero) ✅ verified 2026-04-17 — `auto_fitter.py:L170`. Returns `(regions, background, residual)` from `identify_regions()`; then `fit_all_regions()` returns `FitSessionCollection`. (539 lines ✅ verified 2026-04-17 — `wc -l`) |
| `background_fitter.py` | Background estimation / subtraction |
| `pole_zero_fitter.py` | **PZ coefficient extraction** from 2D S1/S2 histograms (1,342 lines ✅ verified 2026-04-17 — `wc -l`). Refactored from `pz_from_S1S2_current_v6.py` to work on NumPy arrays (no ROOT I/O). Public API: `estimate_pz_from_histogram(h2, s1_edges, s2_edges, params)` → `DetResult` dataclass; `write_pz_cal(path, pz_map)` / `read_pz_cal(path)` → GEBSort-style "det  pz" calibration files. **Four methods:** `chi2` (default — coarse+refine grid scan minimizing χ² of S2 energy spectrum), `peakmatch` (peak-shape matching across S1 slices), `pca` (PCA/orthogonal regression on S1/S2 point cloud), `ridge` (ridge tracking across S1 slices). **`PZParams` defaults:** `pz_min=0.930, pz_max=0.990, pz_step=0.0005`; refine halfwidth=0.002, step=0.0001; `e_bins=8192`. Note: `pz_from_parquet.py` overrides to `pz_min=0.88`. |
| `visualization.py` | Plotting helpers for spectra and fit results. **`FitVisualizer.display_fit(fit_result, x, y)`** — 2-panel matplotlib figure (data+fit top, residuals bottom). **`display_fit_components()`** / **`display_fit_components_2()`** — show individual peak components (Gaussians + background) as separate curves. **`draw_fit_on_axes()`** — draw onto caller-provided axes (no new figure). **`PlotStyle`** dataclass controls labels/linewidths/alpha. All functions use `evaluate_model` / `evaluate_model2` from `ModelEvaluator.py`. Author: mpcarp (M.P. Carpenter). |
| `utils.py` | Shared utilities |
| `debug_utils.py` | Debug/diagnostic helpers |

_Source: `dgs_analysis/armory/gray_apps/src/Fitter/` — explored 2026-04-07_

---

## Cross-References

| Topic | File |
|-------|------|
| dgs_analysis overview (EventBuilders, parquet_pysort, working/ scripts) | [`dgs_analysis.md`](dgs_analysis.md) |
| Pole-zero correction theory + `pz_from_parquet.py` workflow | [`pole_zero.md`](pole_zero.md) |
| GEBSort full reference | [`gebsort.md`](gebsort.md) |
| GEB binary data format | [`data_structures.md`](data_structures.md) |

---

*Split from `dgs_analysis.md` on 2026-04-16. Source: `dgs_analysis/armory/gray_apps/`.*

*Last reviewed: 2026-04-20*
