# Pole-Zero Correction — DGS HPGe Detectors

_Source: `dgs_analysis/working/pz_from_parquet.py`, `armory/gray_apps/src/Fitter/grayfit/pole_zero_fitter.py`, `working/README.md`_
_Reference paper: Begley, Zhu, Carpenter et al., NIM A 1040 (2022) 167113 — "Algorithms of pulse shape analysis for Gammasphere under high count rate conditions"_

---

## Symbol Table

| Symbol | Code variable | Meaning |
|--------|--------------|--------|
| $S_1$ | `sum1` | Pre-rise trapezoidal sum (baseline window, width $M$, ends at trigger) |
| $S_2$ | `sum2` | Post-rise trapezoidal sum (signal window, width $M$, starts after gap $K$) |
| $E$ | `e_raw` | True $\gamma$-ray energy (what we want) |
| $b$ | `base` | Estimated instantaneous baseline DC level |
| $k$ | `GeCenterTimeConstant` | Preamp RC decay constant (µs) |
| $M$ | `dgs-MM` / `m_window` | Trapezoid integration window width (samples) |
| $K$ | `dgs-KK` / `k_window` | Trapezoid flat-top gap between $S_1$ and $S_2$ windows (samples) |
| $\text{PZ}$ | `pz1` | Per-sample pole-zero coefficient: $\text{PZ} = e^{-dt/k}$, $dt=10\,$ns |
| $\text{PZ}_{\text{eff}}$ | `pz4` | Effective PZ over full window gap: $\text{PZ}^{K+M}$ |
| $g$ | | $g = 1 - \text{PZ}$ |
| $V_0$ | `LAST_POST_RISE_M_SUM` | Amplitude of the immediately preceding pulse |
| $t_0$ | derived from `LAST_DISC_TIMESTAMP` | Time since the immediately preceding pulse |
| $\text{sb}$ | `SAMPLED_BASELINE` | FPGA-sampled baseline at trigger time (used by SZ\_2) |
| $dt$ | | Sample period = 10 ns (100 MHz clock) |

---

## What Is Pole-Zero?

HPGe preamplifiers produce an exponential tail after each gamma-ray hit — the output decays as `e^(-t/RC)` where RC is the preamplifier time constant. The digitizer FPGA computes two trapezoidal energy sums:

- **S1** (`sum1`) — integral over the **pre-rise** window (baseline trapezoid, before signal)
- **S2** (`sum2`) — integral over the **post-rise** window (signal trapezoid, after peak)

The simple energy without correction:

```math
E = S_2 - S_1
```

But the preamp exponential tail causes $S_2$ to pick up a contribution from the slowly decaying baseline — creating a **correlation between $S_1$ and $S_2$** that depends on inter-event time and baseline history. The $S_1$ vs $S_2$ scatter plot shows a **tilted distribution** instead of flat.

An **ideal detector** has an infinite preamp time constant (no exponential tail). $S_1$ and $S_2$ are then truly independent — the scatter is flat and horizontal, and energy resolution is maximized.

**The PZ-corrected energy (SZ_1 algorithm):**

```math
E = S_2 - S_1 \cdot \text{PZ} - b\,(1 - \text{PZ})
```

Where $b$ is a running exponential average of $S_1$ (long-term baseline estimate) and $\text{PZ} \in (0,1)$ is the pole-zero coefficient.

At the correct PZ, $S_1$ and $S_2$ become **uncorrelated** — scatter is flat, recovering the ideal case.

| PZ value | Effect |
|----------|--------|
| $\text{PZ} = 1$ | No correction → reduces to $E = S_2 - S_1$ (algo 0) |
| $\text{PZ} = 0.88$–0.99 | Typical operating range |
| $\text{PZ} \to 0$ | Over-correction |

**Bad PZ → energy-dependent bias → degraded resolution and peak broadening.**

### Formal derivation (from NIM A 1040 (2022) 167113)

A valid $\gamma$-ray signal $v(t)$ = charge collecting function $V(t)$ convolved with exponential decay $e^{-\lambda t}$. Pole-zero deconvolution recovers the staircase:

```math
V(t) = v(t) + \lambda \int_0^t v(t')\,dt'
```

The $\gamma$-ray energy with trapezoidal shaping time $M$ (recursive algorithm, Eq. 1 in paper):

```math
E = \frac{1}{M} \sum_{j=i}^{i+M} \left[ v(j+M+K) - v(j) + \lambda \sum_{l=j}^{j+M+K-1} v(l) \right]
```

Where $i$ = trigger sample, $M$ = integration time, $K$ = flat-top gap.

The **directive algorithm** (SZ_1/SZ_2) avoids ballistic deficit:

```math
E = \frac{1}{M} \sum_{j=i}^{i+M} \left[ v(j+M+K) - v(j)\,e^{-\lambda(M+K)} \right] - \left(1 - e^{-\lambda(M+K)}\right) v_{\text{dc}}
```

In discrete form: $\text{PZ} = e^{-\lambda\,dt}$ per sample, so $\lambda = -\ln(\text{PZ})/dt$.

**Key insight:** $S_1 \approx$ pre-rise integral, $S_2 \approx$ post-rise integral. The directive algorithm becomes:

```math
E = S_2 - S_1\cdot\text{PZ} - b\,(1 - \text{PZ})
```

where $b$ estimates $v_{\text{dc}}$ (the long-term DC offset).

---

## Physical Derivation of the Correction Term

_Why does `g×(sum1 - base)` correct for the previous pulse tail?_

Let $g = 1 - \text{PZ}$. The energy formula becomes:

```math
E = S_2 - S_1 + g\,(S_1 - b)
```

**Signal model:** Current pulse at $t=0$, previous pulse at $t=-t_0$, amplitude $V_0$, decay constant $k$ (µs):

```math
v(t) = b_{\text{DC}} + V_0\,e^{-(t+t_0)/k}
```

**Computing $S_1$** (average over pre-rise window $[-M, 0]$):

```math
S_1 = \frac{1}{M}\int_{-M}^{0} v(t)\,dt = b_{\text{DC}} + \frac{V_0 k}{M}\,e^{-t_0/k}\left(e^{M/k}-1\right)
```

**Computing $b$:** Long-run average between pulses (when $V_0 e^{-t_0/k} \approx 0$):

```math
b \approx b_{\text{DC}}
```

**Therefore:**

```math
S_1 - b = V_0\,e^{-t_0/k}\cdot f(M,k), \qquad f(M,k) = \frac{k}{M}\left(e^{M/k}-1\right)
```

$S_1 - b$ is **directly proportional to the previous pulse tail amplitude** $V_0 e^{-t_0/k}$. This is the physical basis for the correction.

**Matching Ryan’s exact formula:** The correction $g\,(S_1 - b)$ equals $V_0 e^{-t_0/k}\cdot 2\sinh(d/k)$ when:

```math
g \cdot \frac{k}{M}\left(e^{M/k}-1\right) = 2\sinh(d/k)
```

This is satisfied by the empirically fitted PZ value — the fitting implicitly encodes $M$, $K$, and $k$ together.

**Baseline drift problem (preamp resets):** If the baseline is not stable (sawtooth pattern), then $b \neq b_{\text{DC}}$ at the current event. The error is:

```math
\epsilon = g\,(b_{\text{DC,now}} - b)
```

The exponential moving average lags through resets, introducing systematic bias. Events near resets are most affected.

**Time constant:** `GS###_GeCenterTimeConstant` PV sets $k$ per detector (selectable: 5.0–52.0 µs in 14 steps). The nominal PZ follows:

```math
\text{PZ} = e^{-dt/k}, \qquad dt = 10\,\text{ns (100 MHz clock)}
```

For $k = 10\,\mu\text{s}$: $\text{PZ} \approx 0.999$. Deviations of fitted PZ from nominal indicate component tolerance or temperature effects.

---

## Summary: Three Levels of PZ Correction

### Level 1 — Approximation (SZ\_1, low/medium rate)

```math
E = S_2 - S_1\cdot\text{PZ}_{\text{eff}} - b\,(1 - \text{PZ}_{\text{eff}})
```

- $b$ = slow exponential moving average of $S_1$
- Works at low rate; fails near preamp resets and at high rates

### Level 2 — Event-by-event (SZ\_2, high rate)

Same formula, but $b$ is solved **algebraically per event** from the FPGA sampled baseline $\text{sb}$:

```math
b = \frac{(S_1 + \text{sb})(1-p_3) - \text{sb}\,(1-p_2)}{(M+m_s)(1-p_3) - m_s(1-p_2)}
```

where $p_2 = \text{PZ}^{m_s/M}$, $p_3 = \text{PZ}^{(M+m_s)/M}$, $\text{PZ}_{\text{eff}} = \text{PZ}^{(M+K)/M}$, and $m_s$ is the peak sample offset.

- Event-by-event, rate-independent
- Recommended at high count rates (>10 kHz per crystal)

### Level 3 — Ryan’s exact formula (proposed, tested, not in production)

All quantities available in the DIG event packet ($V_0$ = `LAST_POST_RISE_M_SUM`, $t_0$ from `LAST_DISC_TIMESTAMP`):

```math
E = S_2 - S_1 + V_0\,e^{-t_0/k}\cdot 2\sinh(d/k)
```

Analytically exact single-pulse tail correction, no base tracking needed.

**⚠️ Experimental result:** J.T. Anderson (JTA) told Ryan that this formula was tested but the results were **not better than SZ\_2**. Likely reasons:
- Real preamps have multi-pole responses — single exponential is an approximation
- $V_0$ = `LAST_POST_RISE_M_SUM` carries its own PZ error from the previous event
- Pulses before the immediately preceding one are not corrected
- $\text{LAST\_DISC\_TIMESTAMP}$ covers only the nearest previous discriminator

SZ\_2 with the FPGA sampled baseline remains the production algorithm.

### Level 4 — Waveform-based correction (proposed, not yet implemented)

**Key observations:**

1. $S_1/M$ is the mean of $v(t)$ over the pre-rise window — approximately the signal at the **midpoint** of that window. Similarly $S_2/M$ at the midpoint of the post-rise window. $P_2$ provides a third sample point.
2. Together, $S_1$, $S_2$, $P_2$ (and $\text{sb}$) give 3–4 effective "trace samples" at known time offsets. Two exponential decay amplitudes can be fitted to these points, recovering $E$ without any assumption about $b$.
3. **Decimated trace:** The DIG firmware implements true **block-averaging decimation** (not subsampling), verified in `decimator.vhd` (M. Oberling):
   - `downsample_factor` PV selects 1×/2×/4×/8×/16×/32×/64×/128×
   - Accumulates N consecutive ADC samples, outputs 16-bit average
   - Block-aligned to the readout window start (`dec_enable` driven by `pending_read_event`); alignment to trigger set by `readout_pretrigger`
   - **`dec_pause` feature (added 2016):** switches between full-rate and decimated within the same event — read the rising edge at 100 MHz (precise timing), then switch to 8× decimation for the tail (80 ns/sample, low cost)
   - Example: 8× decimation → 80 ns/point; 8 points covers 640 ns of exponential tail → sufficient for offline $k$ fitting and full trapezoidal filter

**Trade-off:** Even 8 trace words per event is significant at high rates. Recommend a calibration run with trace mode on a subset of crystals to: (1) validate $k$ vs `GeCenterTimeConstant`, (2) compare resolution vs SZ\_2, (3) test `dec_pause` for rise-time integrity.

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

```math
S_1 = b + (S_1 - b)
```
```math
S_2 = b + E + (S_1 - b)\cdot e^{-\Delta t/k} = b + E + (S_1-b)\cdot\text{PZ}^{\Delta t}
```

where $\Delta t$ = number of samples from the center of $S_1$ to the center of $S_2$ $\approx K + M$.

This is **linear in $S_1$** with slope:

```math
\text{slope} = e^{-\Delta t/k} = \text{PZ}^{\Delta t}
```

**Correct PZ:** $E = S_2 - S_1\cdot\text{PZ}^{\Delta t} - b\,(1-\text{PZ}^{\Delta t})$ is independent of $S_1$ → scatter is flat.
**Wrong PZ:** residual tilt remains.

**Extraction from slope:**

```math
\text{PZ} = \text{slope}^{1/\Delta t}, \qquad \Delta t \approx K + M \text{ (samples)}
```

The `pca` method in `pz_from_parquet.py` measures this slope directly via principal component analysis.

**Consistency check:** the extracted PZ should satisfy:

```math
\text{PZ} = e^{-dt/k} = e^{-10\,\text{ns}\,/\,k_{\mu\text{s}}}
```

Deviations indicate the actual RC differs from the nominal slope box setting.

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
