# Pole-Zero Correction — DGS HPGe Detectors

_Source: `dgs_analysis/working/pz_from_parquet.py`, `armory/gray_apps/src/Fitter/grayfit/pole_zero_fitter.py`, `working/README.md`_
_Reference paper: Begley, Zhu, Carpenter et al., NIM A 1040 (2022) 167113 — "Algorithms of pulse shape analysis for Gammasphere under high count rate conditions"_

---

## What Is Pole-Zero?

HPGe preamplifiers produce an exponential tail after each gamma-ray hit — the output decays as `e^(-t/RC)` where RC is the preamplifier time constant. The digitizer FPGA computes two trapezoidal energy sums:

- **S1** (`sum1`) — integral over the **pre-rise** window (baseline trapezoid, before signal)
- **S2** (`sum2`) — integral over the **post-rise** window (signal trapezoid, after peak)

The simple energy without correction:
```
E = sum2 - sum1
```

But the preamp exponential tail causes `sum2` to pick up a contribution from the slowly decaying baseline — creating a **correlation between S1 and S2** that depends on inter-event time and baseline history. The S1 vs S2 scatter plot shows a **tilted distribution** instead of flat.

An **ideal detector** has an infinite preamp time constant (no exponential tail). S1 and S2 are then truly independent — the scatter is flat and horizontal, and energy resolution is maximized.

**The PZ-corrected energy (SZ_1 algorithm, from `dgs_decode_lib.cpp`):**
```
E = sum2 - sum1 × PZ - base × (1 − PZ)
```
Where:
- `base` = running exponential average of `sum1` (long-term baseline level)
- `PZ ∈ (0,1)` = pole-zero coefficient that cancels the exponential tail contribution

At the correct PZ, S1 and S2 become **uncorrelated** — scatter is flat, recovering the ideal case.

| PZ value | Effect |
|----------|--------|
| PZ = 1 | No correction → reduces to `E = sum2 - sum1` (algo 0) |
| PZ = 0.88–0.99 | Typical operating range |
| PZ → 0 | Over-correction |

**Bad PZ → energy-dependent bias → degraded resolution and peak broadening.**

### Formal derivation (from NIM A 1040 (2022) 167113)

A valid γ-ray signal `v(t)` = charge collecting function `V(t)` convolved with exponential decay `e^(-λt)`. Pole-zero deconvolution recovers the staircase:

```
V(t) = v(t) + λ ∫ v(t)dt
```

The γ-ray energy with trapezoidal shaping time M (recursive algorithm, Eq. 1 in paper):
```
E = (1/M) Σ_{j=i}^{i+M} [ v(j+M+K) - v(j) + λ Σ_{l=j}^{j+M+K-1} v(l) ]
```
Where `i` = trigger time, `M` = shaping (integration) time, `K` = flat-top time.

The **directive algorithm** (SZ_1/SZ_2 in code) avoids ballistic deficit by using pre-rise region as the baseline prediction:
```
E = (1/M) Σ_{j=i}^{i+M} [ v(j+M+K) - v(j)·e^(-λ(M+K)) ] - (1-e^(-λ(M+K)))·v_dc
```
In discrete form (code): `PZ = e^(-λ×dt)` per sample, so `λ = -ln(PZ)/dt`.

**Key insight:** `sum1` ≈ pre-rise integral, `sum2` ≈ post-rise integral. The directive algorithm replaces the exponential prediction with:
```
E = sum2 - sum1×PZ - base×(1-PZ)
```
where `base` estimates `v_dc` (the long-term DC offset = decayed baseline level).

---

## Physical Derivation of the Correction Term

_Why does `g×(sum1 - base)` correct for the previous pulse tail?_

**Setup:** Let g = 1 - PZ. The energy formula becomes:
```
E = sum2 - sum1 + g×(sum1 - base)
```

**Signal model:** Current pulse at t=0, previous pulse at t=-t0, amplitude V0, decay constant k (µs). Signal before trigger:
```
v(t) = baseline_DC + V0·exp(-(t+t0)/k)
```

**Computing sum1** (average over pre-rise window width M, from t=-M to t=0):
```
sum1 = (1/M) ∫_{-M}^{0} v(t) dt
     = baseline_DC + (V0/M) ∫_{-M}^{0} exp(-(t+t0)/k) dt
     = baseline_DC + (V0·k/M)·exp(-t0/k)·[exp(M/k) - 1]
```

**Computing base:** Long-run average of sum1 taken between pulses (when V0·exp(-t0/k) ≈ 0):
```
base ≈ baseline_DC
```

**Therefore:**
```
sum1 - base = (V0·k/M)·exp(-t0/k)·[exp(M/k) - 1]
            = V0·exp(-t0/k)·f(M,k)
```
where `f(M,k) = (k/M)·(exp(M/k)-1)` depends only on hardware constants.

**So `sum1 - base` is directly proportional to the previous pulse tail amplitude `V0·exp(-t0/k)`.** This is the physical basis for the correction.

**Matching Ryan's exact formula:** The correction `g×(sum1-base)` equals Ryan's `V0·exp(-t0/k)·2·sinh(d/k)` when:
```
g × (k/M)·(exp(M/k)-1) = 2·sinh(d/k)
```
This is satisfied by the empirically fitted PZ value — the fitting implicitly encodes the relationship between the sum1 window (M), sum2 window (d≈K/2), and decay constant k.

**The baseline drift problem (preamp resets):** If the baseline is not stable (sawtooth: ramps up between resets, jumps down at reset), then `base ≠ baseline_DC` at the current event. The error is:
```
error = g × (baseline_DC_now - base)
```
The code's exponential moving average of sum1 lags the actual DC level through resets, introducing a systematic bias. Events near resets are most affected. This is a fundamental limitation of the statistical PZ correction vs. an event-by-event correction.

**Time constant:** `GS###_GeCenterTimeConstant` PV sets k per detector (selectable: 5.0–52.0 µs in 14 steps). The nominal PZ follows:
```
PZ = exp(-dt/k)   where dt = 10 ns (100 MHz clock)
```
For k = 10 µs: PZ ≈ 0.999. Empirically fitted PZ should be close to this nominal value; deviations indicate component tolerance or temperature effects.

---

## Summary: Three Levels of PZ Correction

### Level 1 — Approximation (SZ_1, low/medium rate)
```
E = sum2 - sum1·PZ_eff - base×(1 - PZ_eff)
```
- `base` = slow exponential moving average of sum1
- Works well when count rate is low enough for base to track DC
- Fails near preamp resets and at high rates

### Level 2 — Exact (SZ_2, high rate)
Same formula, but `base` is solved **algebraically per event** using the FPGA sampled baseline (`sb`):
```
base = [(sum1 + sb)·(1-pz3) - sb·(1-pz2)] / [(MM+msample)·(1-pz3) - msample·(1-pz2)]
```
where `pz2 = PZ^(msample/MM)`, `pz3 = PZ^((MM+msample)/MM)`, `pz4 = PZ^((MM+KK)/MM) = PZ_eff`.

- Event-by-event, rate-independent
- No assumption about inter-event spacing
- Recommended at high count rates (>10 kHz per crystal)

### Level 3 — Ryan's exact formula (proposed, tested, not in production)

All quantities available in the DIG event packet:
- **V0** = `LAST_POST_RISE_M_SUM` (previous pulse amplitude, stored in current packet)
- **t0** = `EVENT_TIMESTAMP - LAST_DISC_TIMESTAMP` (time since previous pulse)
- **k** = `GeCenterTimeConstant` PV (hardware RC, per detector)
- **d, M** = KK, MM window parameters

```
E = sum2 - sum1 + V0·exp(-t0/k)·2·sinh(d/k)
```

Analytically exact single-pulse tail correction. No base tracking, no rate limitation, no approximation on the instantaneous DC.

**⚠️ Experimental result:** J.T. Anderson (JTA) told Ryan that this formula was tested but the results were **not better than SZ_2**. Likely reasons:
- Real preamps have multi-pole responses — single exponential is an approximation
- `LAST_POST_RISE_M_SUM` carries its own PZ correction error from the previous event
- Pileup from pulses before the immediately preceding one is not corrected
- `LAST_DISC_TIMESTAMP` only covers the nearest previous discriminator, not all contributing pulses

SZ_2 with the FPGA sampled baseline remains the production algorithm.

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

### Why the S1 vs S2 Scatter Encodes PZ

**The locus:** For a given gamma-ray energy E and decay constant k, varying the previous pulse tail (different V0, t0) traces a line in the S1 vs S2 scatter plot.

From the derivation above:
```
sum1 = base + (sum1 - base)                      ← tail at sum1 window
sum2 = base + E_true + (sum1 - base)·exp(-dt/k)   ← E + tail decayed by dt samples
```

This is linear in sum1 with slope:
```
slope = exp(-dt/k) = PZ^dt
```
where `dt` = number of samples between the center of sum1 and sum2 windows ≈ KK + MM (the digitizer K and M window register values).

**Correct PZ:** corrected energy `E = sum2 - sum1·PZ - base·(1-PZ)` is independent of sum1 → scatter is flat.
**Wrong PZ:** residual tilt remains → positive or negative correlation with sum1.

**Extraction from slope:**
```
slope = PZ^dt
→ PZ = slope^(1/dt)   where dt ≈ KK + MM samples
```

**Consistency check:** PZ from the scatter slope should match the nominal value from the hardware setting:
```
PZ_nominal = exp(-dt_sample / k)
           = exp(-10ns / GeCenterTimeConstant)
```
Deviations indicate actual RC differs from the nominal slope box setting (component tolerance, temperature).

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
