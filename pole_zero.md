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
| $P_z$ | `pz1` | Per-sample pole-zero coefficient: $P_z = e^{-dt/k}$, $dt=10\,$ns |
| $P'_{z}$ | `pz4` | Effective PZ over full window gap: $P_z^{K+M}$ |
| $g$ | | $g = 1 - P_z$ |
| $V_0$ | `LAST_POST_RISE_M_SUM` | Amplitude of the immediately preceding pulse | ✅ verified 2026-04-14 — `class_DIG.h:L63` (field name + comment: "Word 11(31:24) AND Word 12(31:24) AND Word 13(31:24)")
| $t_0$ | derived from `LAST_DISC_TIMESTAMP` | Time since the immediately preceding pulse | ✅ verified 2026-04-14 — `class_DIG.h:L87` ("LED = 48 bit value, CFD = 30 bit value")
| $\text{sb}$ | `SAMPLED_BASELINE` | FPGA-sampled baseline at trigger time (used by SZ\_2) — latched from `RUNNING_T1_SUM` (running S1 accumulator) at discriminator fire; 24-bit field, PEHQ bits 347:324; appears in event header Word 6 bits 23:0 ✅ verified 2026-04-09 — `jta_channel.vhd:L1937,L1796` (20230809 tag) |
| $dt$ | | Sample period = 10 ns (100 MHz clock) ✅ verified 2026-04-07 — `Digitizer.vhd:L57` ("100 MHz ADC clock, differential pair") + `LEFT_RIGHT.ucf:L114` (ACQ_DCM_2X_BUFG = 100MHz) |

---

## What Is Pole-Zero?

HPGe preamplifiers produce an exponential tail after each gamma-ray hit — the output decays as `e^(-t/RC)` where RC is the preamplifier time constant. The digitizer FPGA computes two trapezoidal energy sums:

- **S1** (`sum1`) — integral over the **pre-rise** window (baseline trapezoid, before signal)
- **S2** (`sum2`) — integral over the **post-rise** window (signal trapezoid, after peak)

The simple energy without correction:


$$E = S_2 - S_1$$

But the preamp exponential tail causes $S_2$ to pick up a contribution from the slowly decaying baseline — creating a **correlation between $S_1$ and $S_2$** that depends on inter-event time and baseline history. The $S_1$ vs $S_2$ scatter plot shows a **tilted distribution** instead of flat.

An **ideal detector** has an infinite preamp time constant (no exponential tail). $S_1$ and $S_2$ are then truly independent — the scatter is flat and horizontal, and energy resolution is maximized.

**The PZ-corrected energy (SZ_1 algorithm):**

$$E = S_2 - S_1 \cdot P_z - b\,(1 - P_z)$$

Where $b$ is a running exponential average of $S_1$ (long-term baseline estimate) and $P_z \in (0,1)$ is the pole-zero coefficient. ✅ verified 2026-04-13 — `dgs_decode_lib.cpp:L463` (`e_raw = sum2_n - sum1_n * pz1 - base * (1.0 - pz1)`)

At the correct PZ, $S_1$ and $S_2$ become **uncorrelated** — scatter is flat, recovering the ideal case.

| PZ value | Effect |
|----------|--------|
| $P_z = 1$ | No correction → reduces to $E = S_2 - S_1$ (algo 0) |
| $P_z = 0.88$–0.99 | Typical operating range ✅ verified 2026-04-08 — `pz_from_parquet.py:L54-55` (CLI defaults: `--pz-min 0.88 --pz-max 0.99`); `PZParams` internal default is 0.930–0.990 (`pole_zero_fitter.py:L168-169`) |
| $P_z \to 0$ | Over-correction |

**Bad PZ → energy-dependent bias → degraded resolution and peak broadening.**

### Formal derivation (from NIM A 1040 (2022) 167113)

A valid $\gamma$-ray signal $v(t)$ = charge collecting function $V(t)$ convolved with exponential decay $e^{-\lambda t}$. Pole-zero deconvolution recovers the staircase:

$$V(t) = v(t) + \lambda \int_0^t v(t')\,dt'$$

The $\gamma$-ray energy with trapezoidal shaping time $M$ (recursive algorithm, Eq. 1 in paper):

$$E = \frac{1}{M} \sum_{j=i}^{i+M} \left[ v(j+M+K) - v(j) + \lambda \sum_{l=j}^{j+M+K-1} v(l) \right]$$

Where $i$ is trigger sample, $M$ is integration time, $K$ is flat-top gap.

The **directive algorithm** (SZ\_1 or SZ\_2) avoids ballistic deficit:

$$E = \frac{1}{M} \sum_{j=i}^{i+M} \left[ v(j+M+K) - v(j) e^{-\lambda (M+K) } \right] - \left(1 - e^{-\lambda (M+K) }\right) v_{dc}$$

In discrete form: $P_z = e^{-\lambda dt}$ per sample, so $\lambda = -\ln(P_z)/dt$.

**Key insight:** $S_1 \approx$ pre-rise integral, $S_2 \approx$ post-rise integral. The directive algorithm becomes:

$$E = S_{2} - S_{1} \cdot P_{z} - b (1 - P_{z})$$

where $b$ estimates $v_{dc}$ (the long-term DC offset).

---

## Physical Derivation of the Correction Term

_Why does `g×(sum1 - base)` correct for the previous pulse tail?_

Let $g = 1 - P_z$. The energy formula becomes:

$$E = S_2 - S_1 + g (S_1 - b)$$

**Signal model:** Current pulse at $t=0$, previous pulse at $t=-t_0$, amplitude $V_0$, decay constant $k$ (µs):

$$v(t) = b_{\text{DC}} + V_0\,e^{-(t+t_0)/k}$$

**Computing $S_1$** (average over pre-rise window $[-M, 0]$):

$$S_1 = \frac{1}{M}\int_{-M}^{0} v(t)\,dt = b_{\text{DC}} + \frac{V_0 k}{M}\,e^{-t_0/k}\left(e^{M/k}-1\right)$$

**Computing $b$:** Long-run average between pulses (when $V_0 e^{-t_0/k} \approx 0$):

$$b \approx b_{\text{DC}}$$

**Therefore:**

$$S_1 - b = V_0\,e^{-t_0/k}\cdot f(M,k), \qquad f(M,k) = \frac{k}{M}\left(e^{M/k}-1\right)$$

$S_1 - b$ is **directly proportional to the previous pulse tail amplitude** $V_0 e^{-t_0/k}$. This is the physical basis for the correction.

**Matching Ryan’s exact formula:** The correction $g\,(S_1 - b)$ equals $V_0 e^{-t_0/k}\cdot 2\sinh(d/k)$ when:

$$g \cdot \frac{k}{M}\left(e^{M/k}-1\right) = 2\sinh \left(\frac{d}{k}\right)$$

This is satisfied by the empirically fitted PZ value — the fitting implicitly encodes $M$, $K$, and $k$ together.

**Baseline drift problem (preamp resets):** If the baseline is not stable (sawtooth pattern), then $b \neq b_{\text{DC}}$ at the current event. The error is:

$$\epsilon = g\,(b_{\text{DC,now}} - b)$$

The exponential moving average lags through resets, introducing systematic bias. Events near resets are most affected.

**Time constant:** `GS###_GeCenterTimeConstant` PV sets $k$ per detector (16 steps, mbbo record, values 0–15: 52.0, 33.6, 20.6, 16.7, 12.8, 11.2, 9.2, 8.3, 7.7, 7.2, 6.2, 5.9, 5.3, 5.0, 4.6, 4.4 µs). ✅ verified 2026-04-08 — `collectorboxpi/CollectorBox_RevA/db/Pickoff.db:L517-555` (ZRST=52.0us→FFST=4.4us, 16 enum values). The nominal PZ follows:

$$P_z = e^{-dt/k}, \qquad dt = 10\,\text{ns (100 MHz clock)}$$

For $k = 10\,\mu\text{s}$: $P_z \approx 0.999$. Deviations of fitted PZ from nominal indicate component tolerance or temperature effects.

---

## Summary: Three Levels of PZ Correction

### Level 1 — Approximation (SZ\_1, low/medium rate)

$$E = S_2 - S_1 \cdot P'_z - b (1 - P'_z)$$

- $b$ = slow exponential moving average of $S_1$
- Works at low rate; fails near preamp resets and at high rates

### Level 2 — Event-by-event (SZ\_2, high rate)

Same formula, but $b$ is solved **algebraically per event** from the FPGA sampled baseline $\text{sb}$:

$$b = \frac{(S_1 + \text{sb})(1-p_3) - \text{sb}\,(1-p_2)}{(M+m_s)(1-p_3) - m_s(1-p_2)}$$

where $p_2 = P_z^{m_s/M}$, $p_3 = P_z^{(M+m_s)/M}$, $P'_z = P_z^{(M+K)/M}$, and $m_s$ is the peak sample offset.

- Event-by-event, rate-independent
- Recommended at high count rates (>10 kHz per crystal)

### Level 3 — Ryan’s exact formula (proposed, tested, not in production)

All quantities available in the DIG event packet ($V_0$ = `LAST_POST_RISE_M_SUM`, $t_0$ from `LAST_DISC_TIMESTAMP`):

$$E = S_2 - S_1 + V_0\,e^{-t_0/k}\cdot 2\sinh \left(\frac{d}{k}\right)$$

Analytically exact single-pulse tail correction, no base tracking needed.

**⚠️ Experimental result:** J.T. Anderson (JTA) told Ryan that this formula was tested but the results were **not better than SZ\_2**. Likely reasons:
- Real preamps have multi-pole responses — single exponential is an approximation
- $V_0$ = `LAST_POST_RISE_M_SUM` carries its own PZ error from the previous event
- Pulses before the immediately preceding one are not corrected
- `LAST_DISC_TIMESTAMP` covers only the nearest previous discriminator

SZ\_2 with the FPGA sampled baseline remains the production algorithm.

### Level 4 — Waveform-based correction (proposed, not yet implemented)

**Key observations:**

1. $S_1/M$ is the mean of $v(t)$ over the pre-rise window — approximately the signal at the **midpoint** of that window. Similarly $S_2/M$ at the midpoint of the post-rise window. $P_2$ provides a third sample point.
2. Together, $S_1$, $S_2$, $P_2$ (and $\text{sb}$) give 3–4 effective "trace samples" at known time offsets. Two exponential decay amplitudes can be fitted to these points, recovering $E$ without any assumption about $b$.
3. **Decimated trace:** The DIG firmware implements true **block-averaging decimation** (not subsampling), verified in `decimator.vhd` (M. Oberling):
   - `downsample_factor` PV selects 1×/2×/4×/8×/16×/32×/64×/128× ✅ verified 2026-04-09 — `decimator.vhd:L37` (3-bit `dec_factor` port, values 0–7 = factors 1×–128×); PV name `VME$(CRATE):$(BOARD):downsample_factor0–9` ✅ `MDigUser.template:L10137-10380`
   - Accumulates N consecutive ADC samples, outputs 16-bit average
   - Block-aligned to the readout window start (`dec_enable` driven by `pending_read_event`); alignment to trigger set by `readout_pretrigger`
   - **`dec_pause` feature (added 2016-03-04):** switches between full-rate and decimated within the same event — read the rising edge at 100 MHz (precise timing), then switch to 8× decimation for the tail (80 ns/sample, low cost) ✅ verified 2026-04-09 — `decimator.vhd:L36` (`dec_pause` port) + `L189` comment (`20160304`)
   - Example: 8× decimation → 80 ns/point; 8 points covers 640 ns of exponential tail → sufficient for offline $k$ fitting and full trapezoidal filter

**Trade-off:** Even 8 trace words per event is significant at high rates. Recommend a calibration run with trace mode on a subset of crystals to: (1) validate $k$ vs `GeCenterTimeConstant`, (2) compare resolution vs SZ\_2, (3) test `dec_pause` for rise-time integrity.

---

## PZ Coefficient Range and Meaning

- Typical range: **0.88 – 0.99** (CLI scan range) ✅ verified 2026-04-08 — `pz_from_parquet.py:L54-55`; `PZParams` internal defaults are 0.930–0.990 (`pole_zero_fitter.py:L168-169`)
- PZ = 1.0 → no correction (infinite RC time constant)
- PZ = 0.0 → over-correction
- Each crystal has its own PZ value (stored per `gsid` in `dgs_pz.cal`)

---

## Calibration Files

### dgs_pz.cal format
```
  7  0.919000
  8  0.910700
 42  0.945300
...
```
Format: `det_id  pz_value` (6 decimal places, right-aligned 3-char det_id, no header). ✅ verified 2026-04-09 — `pole_zero_fitter.py:L793` (`f"{d:3d}  {pz_map[d]:.6f}\n"`). No comment header is written. One line per crystal. Used by `decode.py` via `PQDecode.chat: dgs-pz-cal`.

### dgs_gain.cal format
```
# gsid  gain  offset
  7  0.123000  -1.4500
  8  0.124000  -1.5000
...
```
Energy calibration: `E_keV = gain × e_raw + offset`. ✅ verified 2026-04-09 — `gain_from_parquet.py:L66-71` (header comment `# gsid  gain  offset` is written; format: `{gsid:3d}  {gain:.6f}  {offset:.4f}`).

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

$$S_1 = b + (S_1 - b)$$

$$S_2 = b + E + (S_1 - b)\cdot e^{-\Delta t/k} = b + E + (S_1-b)\cdot P_z^{\Delta t}$$

where $\Delta t$ = number of samples from the center of $S_1$ to the center of $S_2$ $\approx K + M$.

This is **linear in $S_1$** with slope:

$$\text{slope} = e^{-\Delta t/k} = P_z^{\Delta t}$$

**Correct PZ:** $E = S_2 - S_1\cdot P_z^{\Delta t} - b\,(1-P_z^{\Delta t})$ is independent of $S_1$ → scatter is flat.
**Wrong PZ:** residual tilt remains.

**Extraction from slope:**

$$P_z = \text{slope}^{1/\Delta t}, \qquad \Delta t \approx K + M \text{ (samples)}$$

The `pca` method in `pz_from_parquet.py` measures this slope directly via principal component analysis.

**Consistency check:** the extracted PZ should satisfy:

$$P_z = e^{-dt/k} = e^{-10\,\text{ns}\,/\,k_{\mu\text{s}}}$$

Deviations indicate the actual RC differs from the nominal slope box setting.

### Extraction Methods

| Method | Description | Best for |
|--------|-------------|----------|
| `chi2` | Scans PZ range; minimizes χ² of S1/S2 scatter off the ideal line | **Default — most robust** |
| `peakmatch` | Finds S2/S1 ridge, matches slope to PZ | Structured background |
| `pca` | PCA on S1/S2 scatter; principal axis slope = PZ | Fastest |
| `ridge` | Aligns Co-60 ridge (1173/1332 keV) | Best with clear Co-60 peaks |

**`PZParams` defaults** (`chi2` method — `pole_zero_fitter.py`):

| Parameter | Default | Description |
|-----------|---------|-------------|
| `pz_min` | 0.930 | Coarse scan lower bound | ✅ verified 2026-04-12 — `pole_zero_fitter.py:L168`
| `pz_max` | 0.990 | Coarse scan upper bound | ✅ verified 2026-04-12 — `pole_zero_fitter.py:L169`
| `pz_step` | 0.0005 | Coarse scan step size | ✅ verified 2026-04-12 — `pole_zero_fitter.py:L170`
| `s1_ref_width` | 10.0 | Half-width around V_dc for reference gate | ✅ verified 2026-04-18 — `pole_zero_fitter.py:L173`
| `s1_hi_min` | 300.0 | Min offset above V_dc for high-baseline gate | ✅ verified 2026-04-18 — `pole_zero_fitter.py:L174`
| `e_bins` | 8192 | Energy histogram bins for χ² evaluation | ✅ verified 2026-04-18 — `pole_zero_fitter.py:L177`
| `e_min/max` | 0–8192 | Energy axis range | ✅ verified 2026-04-18 — `pole_zero_fitter.py:L178-179`
| `do_refine` | True | Run fine-scan refinement after coarse scan | ✅ verified 2026-04-12 — `pole_zero_fitter.py:L186`
| `refine_pz_halfwidth` | 0.002 | Fine scan ± half-range around coarse best | ✅ verified 2026-04-12 — `pole_zero_fitter.py:L187`
| `refine_pz_step` | 0.0001 | Fine scan step size | ✅ verified 2026-04-12 — `pole_zero_fitter.py:L188`
| `peak_search_emin/max` | 50–8192 | Energy range for peak finder in refinement | ✅ verified 2026-04-14 — `pole_zero_fitter.py:L191-192`

Note: `armory/gray_apps/polezero_parameters.md` referenced here does **not exist** — the above is extracted directly from `pole_zero_fitter.py:PZParams` (2026-04-08).

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
| 0 | algo0 | Simple trapezoidal (no PZ correction): `e_raw = sum2 - sum1` |
| 1 | SZ_1 | Standard PZ-corrected energy (exponential baseline averaging) |
| 2 | SZ_2 | High-rate variant (SAMPLED_BASELINE extrapolation) |

**SZ_1 implementation** (`dgs_decode_lib.cpp:L454`) ✅ verified 2026-04-12:
- Baseline update: `base = base × (1 - α) + S1_norm × α` where **α = `BASE_ALPHA` = 0.01**
- Update only when inter-event time `dtev ≥ 250` (= 2.5 µs at 10 ns/tick) — avoids pile-up contaminating baseline
- Energy: `e_raw = S2_norm - S1_norm × pz1 - base × (1 - pz1)` (requires `base > 10.0` to be valid)
- `pz1` = per-sample PZ coefficient from `.cal` file; `S1_norm = sum1 / M`, `S2_norm = sum2 / M`

**SZ_2 implementation** (`dgs_decode_lib.cpp:L469`) ✅ verified 2026-04-12:
- Uses SAMPLED_BASELINE (`sb`) from FPGA header field, plus window parameters:
  - `pz2 = pz1^((M + msample) / M)`, `pz3 = pz1^(msample / M)`, `pz4 = pz1^((M + K) / M)`
  - Default `msample = 8.0` (samples before trigger used as baseline reference)
- Two timing regions (thresholds in 10 ns ticks, defaults: `t1=50` → 500 ns, `t2=20` → 200 ns):
  - `dtev ≥ t2` (≥200 ns): compute fresh base from sum1 + sb using pz2/pz3 ratio formula
  - `t2 > dtev > t1`: extrapolate from last known base using linear factor
- Energy: `e_raw = S2_norm - S1_norm × pz4 - base × (1 - pz4)`
- CLI: `--dgs-SZ-t1 50.0 --dgs-SZ-t2 20.0 --dgs-msample 8.0`

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

## DetResult — Output Dataclass

`estimate_pz_from_histogram()` returns a `DetResult` dataclass (`pole_zero_fitter.py:L252`). Key fields:

| Field | Type | Description |
|-------|------|-------------|
| `det` | int | Crystal/detector ID |
| `vdc` | float | DC baseline level (V_dc) used for S1 gating |
| `pz_coarse` | float | Best PZ from coarse grid scan |
| `pz_refined` | float | Best PZ after fine-scan refinement (use this) |
| `axis_choice` | str | Which 2D axis was treated as S1 (auto-detected): `"native"` or `"transposed"` |
| `chi2_curve` | ndarray | χ² vs PZ grid (diagnostic) |
| `pz_grid` | ndarray | PZ values tested (matches chi2_curve) |
| `eref` / `ehi` / `eall` | ndarray | 1D energy spectra at reference, high-S1, and all events |
| `e_edges` | ndarray | Histogram bin edges for eref/ehi/eall |
| `ridge_diag` | dict | Ridge-tracking diagnostics (method=ridge only) |
| `peakmatch_diag` | dict | Peak-matching diagnostics (method=peakmatch only) |
| `pz_split_low/high/delta` | float | Stability cross-check: PZ from low vs high S1 halves (peakmatch only) |
| `evs1_values` | ndarray | Corrected energy vs S1 (2D diagnostic) |

`write_pz_cal(path, pz_map)` takes `{det_id: pz_refined}` and writes the `.cal` file. ✅ verified 2026-04-13 — `pole_zero_fitter.py:L252-292`

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
| `armory/gray_apps/polezero_parameters.md` | Full parameter reference — **⚠️ does not exist** (parameters extracted directly from `pole_zero_fitter.py:PZParams`) |
| `knowledgeBase/run_procedures.md` | Full run procedure including calibration workflow |
| `knowledgeBase/DIG_firmware_expert.md` | DIG firmware trapezoid filter details (M/K/D windows) |

---
*Created: 2026-04-06. Source: `dgs_analysis/working/` + `armory/gray_apps/`*

## Cross-References

- `knowledgeBase/dgs_analysis.md` — pz_from_parquet.py, pz_from_evtparquet.py, gain_from_parquet.py; full analysis pipeline
- `knowledgeBase/run_procedures.md` — K value formula; pole-zero in the DGS run workflow (GEBSort → pz_from_parquet)
- `knowledgeBase/gebsort.md` — GEBSort: accepts dgs_pz.cal for corrected energy output
- `knowledgeBase/DIG_firmware_expert.md` — DIG firmware: S1/S2 accumulator design; PREAMP_RESET_DELAY register
- `knowledgeBase/preamp_reset_readme.md` — Preamplifier reset handling; why PZ correction is needed
- `knowledgeBase/data_structures.md` — DIG event header: sum1/sum2/e_raw field locations in binary data
