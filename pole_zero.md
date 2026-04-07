# Pole-Zero Correction — DGS HPGe Detectors

_Source: `dgs_analysis/working/pz_from_parquet.py`, `armory/gray_apps/src/Fitter/grayfit/pole_zero_fitter.py`, `working/README.md`_

---

## What Is Pole-Zero?

HPGe preamplifiers produce an exponential tail after each gamma-ray hit — the output decays as `e^(-t/RC)` where RC is the preamplifier time constant. The digitizer FPGA computes two trapezoidal energy sums:

- **S1** (`sum1`) — integral over the **pre-rise** window (baseline region, before signal peak)
- **S2** (`sum2`) — integral over the **post-rise** window (signal region, after peak)

In an **ideal detector** (infinite preamp time constant, no exponential tail): `S2` and `S1` are independent of each other and the S1 vs S2 scatter plot shows a flat horizontal distribution. In practice, the preamp tail couples S1 and S2: `S2 = PZ × S1` where `PZ ∈ (0,1)` is the **pole-zero coefficient** encoding the RC decay. A well-corrected detector has S2 uncorrelated with S1 (no tilt in the scatter plot), recovering the ideal case.

When PZ is **wrong**, the S1 vs S2 scatter plot shows a tilted correlation. When PZ is **correct**, the scatter collapses to a horizontal line (energy-independent ratio).

**Physics:** The trapezoidal filter subtracts a delayed version of the signal. PZ corrects for the exponential preamp tail so the flat-top of the trapezoid is truly flat. Bad PZ → ballistic deficit → degraded energy resolution.

---

## PZ Coefficient Range and Meaning

- Typical range: **0.88 – 0.99** (scan this range to find optimum)
- PZ = 1.0 → no correction (infinite RC time constant)
- PZ = 0.0 → over-correction
- Each crystal has its own PZ value (stored per `gsid` in `dgs_pz.cal`)

---

## Calibration Files

### dgs_pz.cal format
```
# gsid  pz_value
7    0.9190
8    0.9107
42   0.9453
...
```
One line per crystal. Used by `decode.py` via `PQDecode.chat: dgs-pz-cal`.

### dgs_gain.cal format
```
# gsid  gain  offset
7    0.123  -1.45
8    0.124  -1.50
...
```
Energy calibration: `E_keV = gain × e_raw + offset`.

---

## Method 1: From Parquet (Recommended)

Uses `sum1`/`sum2` columns already in the decoded parquet — no ROOT needed.

### Step 1 — Decode only (skip event builder)
```bash
./working/RunParquet --decode-only ~/ANLDAQ/tcpReceiver/expInfo.sh <run_number>
# Output: $expFolder/Parquet/decode/exp2008_003_dgs.parquet
```

### Step 2 — Extract PZ constants
```bash
python working/pz_from_parquet.py \
    $expFolder/Parquet/decode/exp2008_003_dgs.parquet \
    --output working/dgs_pz.cal \
    --method chi2 \
    --pz-min 0.88 --pz-max 0.99 \
    --pz-step 0.0005
```

| Option | Default | Description |
|--------|---------|-------------|
| `--output` | `dgs_pz.cal` | Output calibration file |
| `--method` | `chi2` | Algorithm (see below) |
| `--pz-min/max` | 0.88 / 0.99 | Scan range |
| `--pz-step` | 0.0005 | Coarse scan step |
| `--s1-bins/s2-bins` | 512 | 2D histogram bins per axis |
| `--tid N ...` | all | Process only specific crystals |
| `--quiet` | off | Suppress per-crystal progress |

### Extraction Methods

| Method | Description | Best for |
|--------|-------------|----------|
| `chi2` | Scans PZ range; minimizes χ² of S1/S2 scatter off the ideal line | **Default — most robust** |
| `peakmatch` | Finds S2/S1 ridge, matches slope to PZ | Structured background |
| `pca` | PCA on S1/S2 scatter; principal axis slope = PZ | Fastest |
| `ridge` | Aligns Co-60 ridge (1173/1332 keV) | Best with clear Co-60 peaks |

See `armory/gray_apps/polezero_parameters.md` for full parameter reference.

---

## Method 2: From ROOT (Alternative)

### Step 1 — Produce S1/S2 histograms
```bash
cd armory/fastEventContructor
root 'analyzer_pz_cal.cpp("yourfile.root")'
```

### Step 2 — Extract via GrayCAL GUI
```bash
cd armory/gray_apps
./run_graycal.sh
```
1. Load ROOT file, select S1/S2 histogram in sidebar
2. Open **Fit → Pole-Zero Extraction**
3. Set scan range (typically 0.93–0.99, step 0.0005)
4. Choose method, click **Run Extraction**
5. Click **Save Calibration** → writes `gsid  pz_value` per line

---

## Using PZ in the Pipeline

### PQDecode.chat configuration
```
dgs-pz-cal  working/dgs_pz.cal    # pole-zero constants
dgs-ehi-cal working/dgs_gain.cal  # energy gain/offset
dgs-MM      350                    # trapezoid M window (register units)
dgs-KK      131                    # trapezoid K window (register units)
dgs-beta    0.025                  # decay constant for SZ algo
dgs-algo    1                      # 0=algo0, 1=SZ_1, 2=SZ_2
```

**Energy algorithms:**
| Value | Name | Description |
|-------|------|-------------|
| 0 | algo0 | Simple trapezoidal (no PZ correction) |
| 1 | SZ_1 | Standard pole-zero corrected energy | 
| 2 | SZ_2 | High-rate variant |

### decode.py output columns
After PZ correction, `decode.py` writes:
- `e_raw` — raw energy (before gain calibration)
- `e_cal` — calibrated energy in keV (`e_cal = gain × e_raw + offset`)
- `e_dc` — Doppler-corrected energy (if recoil velocity provided)
- `sum1`, `sum2` — raw S1/S2 sums (for diagnostics)
- `CSflag` — Compton suppression flag (0 = suppressed by BGO, 1 = clean)

---

## Quick Reference

```bash
# Full workflow for a new experiment:

# 1. Decode run (no event building)
./working/RunParquet --decode-only expInfo.sh 3

# 2. Extract PZ constants
python working/pz_from_parquet.py \
    $expFolder/Parquet/decode/exp2008_003_dgs.parquet \
    --output working/dgs_pz.cal

# 3. Extract energy calibration (Eu-152 or similar)
python working/gain_from_parquet.py \
    $expFolder/Parquet/decode/exp2008_003_dgs.parquet \
    --output working/dgs_gain.cal \
    --source armory/gray_apps/data/isotopes/sou-files/Euautocal.json

# 4. Edit PQDecode.chat to point to new cal files
# 5. Run full pipeline
./working/RunParquet expInfo.sh 3

# 6. Inspect with parquetCLI
./working/parquetCLI $expFolder/Parquet/events/exp2008_003_events.parquet
```

---

## Related Files

| File | Description |
|------|-------------|
| `working/pz_from_parquet.py` | PZ extraction from parquet |
| `working/pz_from_evtparquet.py` | PZ extraction from event-level parquet |
| `working/gain_from_parquet.py` | Energy calibration from parquet |
| `working/PQDecode.chat` | Main decode config (edit per experiment) |
| `armory/fastEventContructor/analyzer_pz_cal.cpp` | ROOT-based S1/S2 histogram producer |
| `armory/gray_apps/src/Fitter/grayfit/pole_zero_fitter.py` | Core PZ algorithm |
| `armory/gray_apps/polezero_parameters.md` | Full parameter reference |
| `dgs/run_procedures.md` | Full run procedure including calibration workflow |
| `dgs/DIG_firmware_expert.md` | DIG firmware trapezoid filter details (M/K/D windows) |

---
*Created: 2026-04-06. Source: `dgs_analysis/working/` + `armory/gray_apps/`*
