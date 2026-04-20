# dgs_analysis — ROOT Analysis Scripts (fastEventConstructor)

> **Split from `dgs_analysis.md`** (2026-04-18). Full pipeline context: see [`dgs_analysis.md`](dgs_analysis.md).

_Source: `dgs_analysis/armory/fastEventContructor/` — five ROOT CINT/compiled scripts. Documented 2026-04-17._

These are **standalone ROOT analysis scripts** that work on event-built ROOT TTrees produced by the EventBuilder variants. They are not part of the automated pipeline — run interactively in ROOT or compiled.

---

## Common Infrastructure

**`analyzer.h`** — shared utilities:
- `MWIN 350.0` — coincidence window constant (match window, units = 10 ns ticks, so 3.5 µs) ✅ verified 2026-04-17 — `analyzer.h:L16` (`#define MWIN 350.;`)
- `ZeroCrossing()` — finds zero crossing of 2–3 (x,y) points using linear interpolation (2 pts) or quadratic fit (3 pts); used for CFD timing from trace samples ✅ verified 2026-04-17 — `analyzer.h:L18-50` (3-pt: quadratic Newton form; 2-pt: linear interp)

**`misc.h`** — `LoadChannelMapFromFile()` — loads the GS hole → detector ID mapping from `angtheta.dat` / `map.dat`

---

## analyzer.cpp — Standard Gamma-Gamma Analysis (347 lines)

Main HPGe coincidence analysis. Reads a ROOT TTree with the standard event-built schema.

**Branches used:** `NumHits`, `detID`, `energy` (long long), `eventTS`, `trigTS`, `CFD_sample_0/1/2`

**Histograms produced:**
| Histogram | Description |
|-----------|-------------|
| `he[110]` | Per-detector energy spectra (110 histograms) |
| `heID` | Energy vs detector ID (2D, 200 bins × 110 dets) |
| `heIDEven/Odd` | Energy vs det ID split south/north hemispheres |
| `hTDiff[110]` | Per-detector time difference spectra |
| `hMultiHits` | Gamma multiplicity distribution (up to 10) |
| `hIDvID` | Det ID vs Det ID for multiplicity-2 events |
| `hEE` | Energy-energy for multiplicity-2 events |
| `hGG` | Energy-energy for two specific detectors (default: 70 vs 62) |
| `hGTimeDiff` | Time difference between those two detectors |
| `hTT1/hTT2` | Timestamp diffs from first hit of events |
| `hTDiffHitOrder` | Time diff vs hit order for det 62 |
| `hTDiffEnergy` | Time diff vs energy for det 62 |

**Energy range:** 100–6000 a.u. (raw units, uncalibrated); **time range:** 0–600 ns.

**CFD timing:** For each hit, uses `ZeroCrossing(CFD_sample_0/1/2, tick_offsets)` to compute sub-tick timing. Offset tick positions are derived from `eventTS` bits.

**Hardcoded detector pair:** `detX=70, detY=62` (Gammasphere hole numbers) for the gated γ-γ analysis. ✅ verified 2026-04-17 — `analyzer.cpp:L32-33`

---

## analyzer_tac.cpp — TAC-II Coincidence Analysis (265 lines)

Analyzes TAC-II timing data in coincidence with HPGe hits. TAC hits are identified by `detID == 999`. ✅ verified 2026-04-17 — `analyzer_tac.cpp:L78`

**Key histograms:**
| Histogram | Description |
|-----------|-------------|
| `htacDiff` | TAC trigTime − det 62 trigTime (10 ns bins, ±10 µs range) |
| `htacDiffID` | TAC time diff vs detector ID (2D) |
| `hTrigTime` | Trigger time distribution (0–10 µs) |
| `heventTStrigTSDiff` | eventTS − trigTS vs detector ID (timing offset map) |
| `hTACMultiplicity` | TAC hit multiplicity per event |
| `hTACMultiVsTimeDiff[7]` | TAC time diff to previous hit, per multiplicity level |
| `heventTimeSpan` | Time span of hits within one coincidence event |

**Event classification:** Each event is counted as one of four categories:
- `withTACcount`: has both GS and TAC hits
- `noTACcount`: GS only
- `onlyTACcount`: TAC only (beam monitor without γ)
- `noTACandGScount`: neither (should be rare)

**Time units:** 10 ns ticks (DIG pipeline runs at 100 MHz → 10 ns per tick; `trigTS` and `eventTS` branches are in these units). ✅ verified 2026-04-20 — `analyzer_tac.cpp:L39-58` (histogram axes consistently labeled `[10 ns]`; `BinaryReader.h:L60` "in 10 ns"; `deep_fpga_DIG_channel.md:L14` "100 MHz clock cycles (10 ns steps)")

---

## analyzer_trace.cpp — Waveform Trace Analysis (324 lines)

Analyzes waveform trace data stored in the ROOT tree (requires `saveTrace=true` during event building).

**Key branches:** `tracePara[hit][param]`, `traceDetID` — trace parameters and associated detector.

`tracePara` columns (GSL fit params, 4 values per hit) — ✅ verified 2026-04-17 — `EventBuilder_X.cpp:L25` + `analyzer_trace.cpp:L113-115`:
- `[0]` = A — amplitude (proportional to energy)
- `[1]` = T0 — timing offset from reference
- `[2]` = riseTime — pulse rise time
- `[3]` = B — baseline

**Purpose:** Characterize pulse shape discrimination (PSD) — separates neutrons from gammas, or identifies pile-up. The `TCutG` graphical cuts in `analyzer_script.cpp` are drawn on `tracePara[0][1]` (T0) vs `tracePara[0][2]` (riseTime) to select "lower tail" and "high tail" events for a specific detector (`traceDetID == 12107`).

---

## analyzer_pz_cal.cpp — Pole-Zero Calibration from Traces (155 lines)

Extracts pole-zero correction parameters from waveform traces. Complements `pz_from_parquet.py` (which uses Parquet output). Works on ROOT TTree trace data.

---

## analyzer_script.cpp — ROOT Script for Trace Shape Analysis (71 lines)

Short ROOT CINT script demonstrating trace parameter analysis:
- Opens `tac2_021_single.root`
- Applies graphical cuts (`TCutG`: `lowerTail`, `highTail`) on `tracePara[0][1]` vs `tracePara[0][2]`
- Plots energy vs trace shape parameters for detector `12107` with/without a 500 a.u. energy cut
- **Purpose:** Inspect tail shape to separate full-energy peaks from Compton scatter or charge-loss events

---

## Cali_e.C — Interactive Energy Calibration ROOT Script (461 lines)

Interactive ROOT macro for energy calibration of all 110 HPGe detectors from a `TTree` (EventBuilder output).

**Key steps:**
1. User sets raw energy range (default 400–3200 ch, 400 bins) at runtime via stdin prompts
2. One `TH1F` histogram per detector (`q[0..109]`), filled from `(post_rise_energy - pre_rise_energy)/350` gated by `detID == i+1`
3. Peak finding via `TSpectrum` — user sets threshold (fraction of tallest peak, default 0.2)
4. **Combination-based peak matching:** For each detector, all combinations of found peaks are tested against known reference gamma lines. R² correlation maximized to find the best-fit (fit energy ↔ ref energy assignment). Handles case where # found peaks ≠ # reference lines by trying both `nX < nY` and `nX > nY` branches.
5. Linear calibration coefficients `a0[det]` (offset, keV) and `a1[det]` (gain, keV/ch) extracted per detector
6. Results displayed on a 10×11 canvas grid (one pad per detector)

**Helper functions:**
- `combination(arr, r)` — generates all r-combinations of arr (used to match found peaks to reference lines)
- `sumMeanVar(data)` — returns {sum, mean, variance} for R² calculation

**Usage:** Load in ROOT, call `Cali_e(tree)` where `tree` is an EventBuilder TTree. Requires `detID`, `pre_rise_energy`, `post_rise_energy` branches. ✅ verified 2026-04-18 — `Cali_e.C:L173-176` (numDet=110), `L233` (energy expression)

---

## checkTACFile.cpp — TAC-II Binary File Validator (ROOT macro)

Diagnostic ROOT script for inspecting raw TAC-II binary data files before event building. Reads via `BinaryReader` + `class_TDC`.

**What it does:**
- Opens a TAC binary file (hardcoded path: `data_slopebox/testD_005/...`, intended to be modified per use)
- Decodes each hit as a `TDC` struct; classifies hits as trash (3 categories) or good:
  - `TACTrash::NoTrigger` — hit has no trigger
  - `TACTrash::TDCoffsetInvalid` — TDC offset is invalid
  - `TACTrash::VernierInvalid` — vernier interpolation invalid
- Tracks time anomalies: TAC time < globalEarliestTime, gap > 1 second, or timestamp going backward
- Reports: file size, total events, good event count, trash/anomaly counts per category
- Plots 3 histograms on one canvas:
  - `htacTimeDiff` — inter-hit time difference [µs, 0–2000]
  - `htrigTimetdcTimeDiff` — TDC timestamp − trig timestamp [ns, 0–200]
  - `htrigTimetacTimeDiff` — avgPhaseTime − trig timestamp [ns, -300–0]

**Purpose:** Verify TAC-II file integrity before event building; identify timing pathologies (gaps, reversals, invalid vernier). ✅ verified 2026-04-18 — `checkTACFile.cpp`

---

## findMapping.sh — Generate GS Channel Map from EPICS (Bash)

Generates `GS_channel_map.txt` by querying live EPICS PVs for all 110 GS detectors.

**For each GS001–GS110, caget-queries:**
- `GS###_VME_Index` — VME crate index
- `GS###_Dig_Index` — DIG board index within crate
- `GS###_Dig_Channel` — channel on DIG board
- `GS###_Ge_ID` — true Ge detector ID
- `VME##:MDIG#:user_package_data_RBV` — board firmware ID

**Output columns:** `GS_number  VME  DIG  Channel  BoardID  GS_True`

**Note:** Requires live EPICS CA access (must be run on dgs4 or with CA env set). `VME_Index == -1` means detector not installed (BoardID + Ge_ID set to -1). ✅ verified 2026-04-18 — `findMapping.sh`

---

## findGS.sh — GS Detector Lookup Utility (Bash)

Lookup tool for the `GS_channel_map.txt` file generated by `findMapping.sh`.

**Usage:** `./findGS.sh <GS_number>` (e.g., `./findGS.sh 015` or `./findGS.sh 15`)

- Normalizes input to 3-digit format with leading zeros
- AWK search skips 2-line header; prints: `GS_number  VME  DIG  Channel  BoardID`
- Exits non-zero if GS number not found

**Purpose:** Quick CLI lookup of which VME crate, DIG board, and channel a given GS detector maps to. Useful during run setup and hardware debugging without EPICS access. ✅ verified 2026-04-18 — `findGS.sh`

---

## readTrace.C — Interactive Waveform Trace Viewer (ROOT)

Interactive ROOT macro for inspecting **waveform traces** saved by `EventBuilder_S` (when `saveTrace=1`). Reads ROOT TTrees with `trace[traceCount][traceLen]` branches.

**Usage:**
```bash
root -l 'readTrace.C("runXXX.root")'
root -l 'readTrace.C("runXXX.root", 6, 9)'    # filter to detID 6–9
root -l 'readTrace.C("runXXX.root", 0, 999, true)'  # print raw ADC values
```

**Arguments:**
- `fileName` — ROOT file from `EventBuilder_S` with saveTrace=1 (or `EventBuilder_Q`)
- `minDetID` / `maxDetID` — filter by detector ID range (default: 0–999000)
- `print` — if true, prints raw `{tick, ADC}` pairs for each trace

**What it does:**
- Opens `tree` (or falls back to `gen_tree`) in the ROOT file
- Reads trace branches: `trace[200][maxTraceLen]`, `traceCount`, `traceLen`, `traceDetID`
- Optionally reads GSL fit results: `tracePara[traceCount][4]` and `traceChi2[traceCount]` (if present)
- For each hit in each event, plots the raw waveform as a `TGraph` (x = tick × 10 ns)
- Overlays a **falling logistic sigmoid fit**: `A / (1 + exp((x−T0)/riseTime)) + B` (TF1 in red)
- If `tracePara` exists, also draws the pre-computed GSL fit result (blue) for comparison
- Interactive navigation: press Enter = next trace, `q` = quit, `w` = back to previous

**Fit parameters printed:**
- `Amp`: signal amplitude (ADC counts, negative for HPGe falling edge)
- `Time`: pulse timing offset T0 (ticks)
- `Rise time (10%–90%)`: `riseTime × 4.29` ticks (conversion factor empirically chosen)
- `baseline`: ADC baseline level
- `chi2` (if pre-computed GSL fit exists)

**Branch layout (from `EventBuilder_S` with saveTrace=1):**
- `traceCount` — number of traces in event
- `traceDetID[traceCount]` — detector ID per trace
- `traceLen[traceCount]` — trace length (samples) per trace
- `trace[traceCount][maxTraceLen]` — raw ADC samples (UShort_t, 10 ns/tick)
- `tracePara[traceCount][4]` — GSL sigmoid fit: [A, T0, riseTime, baseline] (when nWorkers>0)
- `traceChi2[traceCount]` — GSL fit chi² residual

**Note:** `maxTraceLen` is read from the `trace_info` TMacro stored in the ROOT file. The 4.29 rise-time conversion factor converts the logistic σ parameter to a 10%–90% rise-time estimate.

✅ verified 2026-04-20 — `dgs_analysis/armory/fastEventContructor/readTrace.C` (307 lines)

---

## Cross-References

| Topic | File |
|-------|------|
| Full pipeline reference (EventBuilder, parquetCLI, ProcessRUN, gain/pz scripts) | `knowledgeBase/dgs_analysis.md` |
| gray_apps (GrayCAL, GrayMAN, grayfit) | `knowledgeBase/dgs_analysis_grayapps.md` |
| Pole-zero correction theory + `pz_from_parquet.py` detail | `knowledgeBase/pole_zero.md` |
| GEBSort full reference | `knowledgeBase/gebsort.md` |
| GEB binary data format + GEBHeader struct | `knowledgeBase/data_structures.md` |
| Gammasphere geometry (GS hole → θ/φ, map.dat context) | `knowledgeBase/gammasphere_geometry.md` |
| DIG firmware readout modes (source of raw GEB payloads) | `knowledgeBase/DIG_firmware_expert.md` |

---

*Split from `dgs_analysis.md` 2026-04-18. Original content documented 2026-04-17/18.*
