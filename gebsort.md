# GEBSort — GRETINA/DGS Event Builder and Sorter

Stability: C2 - Active / semi-stable

_Source: `DGS_tools_pack/gebsort/` (cloned from `https://gitlab.phy.anl.gov/tlauritsen/gebsort`)_
_Author: T. Lauritsen (ANL). Used by DGS, GRETINA, X-Array, DUO, and other detector systems._

---

## Table of Contents

- [Overview](#overview)
- [Repository Structure](#repository-structure)
- [Configuration: GEBSort.chat](#configuration-gebsortchat)
  - [Basic Parameters](#basic-parameters)
  - [Detector Modules](#detector-modules-enable-by-uncommenting)
  - [DGS-Specific Parameters](#dgs-specific-parameters)
- [DGS Calibration Workflow](#dgs-calibration-workflow-from-readmebin_dgs)
  - [Step 0 — Setup](#step-0--setup)
  - [Step 1 — Find M and K parameters](#step-1--find-m-and-k-parameters)
  - [Step 2 — Generate detector map](#step-2--generate-detector-map)
  - [Step 3 — Pole-Zero calibration](#step-3--pole-zero-calibration-pz-scan)
  - [Step 4 — SZ energy factor](#step-4--sz-energy-factor)
  - [Step 5 — Energy calibration](#step-5--energy-calibration)
  - [Step 6 — Deploy calibration files](#step-6--deploy-calibration-files)
  - [Step 7 — Final sort](#step-7--final-sort)
- [Utility Programs — Internal Details](#utility-programs--internal-details)
  - [SZ_basic_PZ](#sz_basic_pz)
  - [SZ_factor](#sz_factor)
- [Energy Algorithm `.stub` Files](#energy-algorithm-stub-files)
  - [SZ_1 Algorithm (Positive Polarity)](#sz_1-algorithm-positive-polarity)
  - [SZ_1_neg (Negative Polarity)](#sz_1_neg-negative-polarity)
  - [bin_dgs_GE.c and bin_dgs_AUX.c](#bin_dgs_gec-and-bin_dgs_auxc)
  - [bin_angcor_DGS.c — Angular Correlation Sorter](#bin_angcor_dgsc--angular-correlation-sorter)
- [GEBMerge, gtReceiver, GEBClient, dmpdata](#gebmerge-gtreceiver-gebclient-dmpdata)
- [jta.c — DGSEvDecompose_v3 Event Decoder](#jtac--dgsevdecompose_v3-event-decoder)
- [Calibration Files](#calibration-files)
- [bin_dgs.c — Main DGS Sort Function Internals](#bin_dgsc--main-dgs-sort-function-internals)
- [Comparison with parquet_pysort](#comparison-with-parquet_pysort)
- [GRETINA-Specific and Additional Detector Sorters](#gretina-specific-and-additional-detector-sorters)
- [Cross-References](#cross-references)
- [Minor Utility Programs and Support Files](gebsort_utilities.md) _(split to `gebsort_utilities.md`)_

---

## Overview

**GEBSort** is the primary offline analysis framework for GEB-format nuclear physics data. It:
- Reads merged GEB binary files (from `GEBMerge`)
- Builds coincidence events (time-windowed event building)
- Calibrates and sorts data into ROOT histograms and spectra
- Is highly modular: separate `bin_*.c` files handle each detector type

**Key programs:**

| Program | Description |
|---------|-------------|
| `GEBSort` / `GEBSort_nogeb` | Main sorter (with/without live GEB data stream) |
| `GEBMerge` | Merges multiple GEB data streams by timestamp ✅ verified 2026-04-10 — `GEBMerge.c:L1237,L1632` ("find lowest time stamp of the candidates we have") |
| `GEBFilter` | Filters GEB events by type/condition |
| `GEBCrop` | Crops/trims GEB files |
| `GEBSplit` | Splits GEB files |
| `GEBHeader` | Reads/prints GEB headers |
| `find_MK` | Reads live EPICS PVs to compute recommended `dgs_MM` and `dgs_KK` values ✅ verified 2026-04-10 — `find_MK.c:L63-94` (reads `GLBL:DIG:GeC_{d,k,k0,d3,m}_window` via CA; K = d+k+k0+d3+0.15us hidden fixed; outputs in 10 ns units) |
| `mk_dgs_map` | Generates the DGS detector map file |
| `fwhm_onepeak` | Finds best PZ value by minimizing peak FWHM — scans PZ 0.70→1.0 in 0.01 steps, fits parabola to find vertex ✅ verified 2026-04-14 — `fwhm_onepeak.c:L56,L103-125` |
| `SZ_factor` | Extracts the SZ energy extrapolation factor |
| `dgs_ecal` | Automatic energy calibration from known source (207Bi, 88Y, 60Co) — linear fit (gain/offset) per detector ✅ verified 2026-04-14 — `dgs_ecal.c:L49,L64-78` |

---

## Repository Structure

```
gebsort/
├── GEBSort.cxx          Main sorter (C++, ROOT)
├── GEBSort.h            Header
├── GEBSort.chat         Configuration file (all parameters)
├── GEBMerge.c           Timestamp-merge of GEB streams
├── GEBMerge.chat        GEBMerge config
├── GEBFilter.c          Event filter
├── GEBCrop.c            File cropper
├── GEBSplit.c           File splitter
├── GEBHeader.c          Header reader
├── GEBClient.c/h        Live data stream client (VxWorks/GEB)
├── GEBLink.h            GEB data structure definitions
├── bin_dgs.c            DGS digitizer sort module
├── bin_dgs_GE.c         DGS Ge-specific sorting
├── bin_dgs_AUX.c        DGS auxiliary data
├── bin_mode1.c          GRETINA mode1 sort
├── bin_mode2.c          GRETINA mode2 sort
├── bin_tac2.c           TAC-II timing sort
├── bin_angcor_DGS.c     DGS angular correlation
├── bin_angdis.c         Angular distribution
├── bin_dfma.c           DFMA detector sort
├── bin_XA.c             X-Array sort
├── bin_dub.c            DuoGe sort
├── bin_linpol.c         Linear polarization
├── find_MK.c            M/K parameter optimizer
├── fwhm_onepeak.c       PZ fwhm optimizer
├── SZ_factor.c          SZ factor extractor
├── dgs_ecal.c           Energy calibration utility
├── mk_dgs_map.c         DGS detector map generator
├── Makefile             Linux build
├── Makefile.Darwin      macOS build
├── README               Setup instructions
├── README.bin_dgs       Full DGS calibration workflow
├── README.GEBCrop       GEBCrop usage
└── GEBSort.chat         Main config (all parameters)
```

---

## Configuration: GEBSort.chat

All runtime parameters are set in `GEBSort.chat`. Key sections:

### Basic Parameters
```
nevents       2000000000   # Max events to process
maxDataTime   86400        # Max data time (seconds)
timewin       800          # Coincidence window (ticks, 1 tick = 10 ns) ✅ verified 2026-04-08 — bin_dgs.c:L833 ("1024*10ns"), L645 ("10nsec units")
beta          0.00         # Recoil velocity β for Doppler correction
```

### Detector Modules (enable by uncommenting)
```
bin_mode1                  # GRETINA mode1 data
bin_mode2                  # GRETINA mode2 data
;bin_tac2                  # TAC-II timing (commented = disabled)
;bin_dgs                   # DGS digitizer data
;bin_dub                   # DuoGe
;bin_XA                    # X-Array
;bin_dfma                  # DFMA
;bin_angcor_DGS            # DGS angular correlations
```

### DGS-Specific Parameters
```
dgs_algo    1              # Energy algorithm: 0=simple, 1=SZ_1, 2=SZ_2 ✅ verified 2026-04-17 — `GEBSort.chat:L218` (value=1, SZ_1). Note: `2` (SZ_2) requires additional dgs_factor + t1/t2 params.
dgs_MM      350            # Trapezoid M window (samples) — example value from README.bin_dgs:L11 (commented out); **current GEBSort.chat uses 200** ✅ verified 2026-04-10 — `GEBSort.chat:L219` (dgs_MM=200). Use `find_MK` to determine optimal value for each experiment.
dgs_KK      141            # Trapezoid K window (samples) ✅ verified 2026-04-08 — README.bin_dgs:L12; current GEBSort.chat also uses 141 ✅ verified 2026-04-10 — `GEBSort.chat:L220`
dgs_PZ      dgs_pz.cal    # Pole-zero calibration file
dgs_ecal    dgs_ehi.cal   # Energy gain/offset calibration file
dgs_factor  dgs_factor.dat # SZ energy factor file
dgs_SZ_t1   50             # SZ_2 transition time 1 (samples) ✅ verified 2026-04-17 — `GEBSort.chat:L221`
dgs_SZ_t2   20             # SZ_2 transition time 2 (samples) ✅ verified 2026-04-17 — `GEBSort.chat:L222`
```

---

## DGS Calibration Workflow (from `README.bin_dgs`)

Full calibration uses a source run (typically $^{207}$Bi) and an in-beam run:

### Step 0 — Setup
```bash
rm GEBSORT; ln -s ../GEBSort GEBSORT
rm GTDATA; ln -s /path/to/merged/data GTDATA
```

### Step 1 — Find M and K parameters
```bash
GEBSORT/find_MK
```
Reads the data and finds optimal trapezoid shaping parameters. Outputs recommended `dgs_MM` and `dgs_KK` values for `GEBSort.chat`.

### Step 2 — Generate detector map
```bash
GEBSORT/mk_dgs_map map.dat
```
Creates the DGS detector map file mapping GS holes to digitizer channels.

### Step 3 — Pole-Zero calibration (PZ scan)
```bash
# Scan PZ values 0.87–0.98 on source + in-beam runs
lambda_scan.sh 0.87
lambda_scan.sh 0.88
# ... (one per value)
lambda_scan.sh 0.98

# Find best PZ per crystal (minimizes FWHM, sign change in skewness)
GEBSORT/fwhm_onepeak 0.01 | grep diff | grep -v agree | awk '{print $3,$9}' > dgs_pz.cal
```
Output: `dgs_pz.cal` — one line per crystal: `gsid  pz_value`

### Step 4 — SZ energy factor
```bash
./go_one.sh GEBMerged_run002.gtd_000
GEBSORT/SZ_factor test.root factor dgs_factor.dat
```
Output: `dgs_factor.dat` — per-crystal energy extrapolation correction factor.

### Step 5 — Energy calibration
```bash
rm dgs_ehi.cal
./go_one.sh GEBMerged_run019.gtd_000
root -b -l ./get_eraw.C
GEBSORT/dgs_ecal dgs_ehi.cal 207Bi 900 1.0
```
Output: `dgs_ehi.cal` — per-crystal gain + offset in keV.

### Step 6 — Deploy calibration files
```bash
cp dgs_pz.cal ../GEBSort
cp dgs_factor.dat ../GEBSort
cp dgs_ehi.cal ../GEBSort
```

### Step 7 — Final sort
```bash
./go_one.sh GEBMerged_runXXX.gtd_000
# Or directly:
GEBSORT/GEBSort_nogeb \
  -input GEBMerged_runXXX.gtd_000 \
  -rootfile output.root \
  -chat GEBSort.chat > GEBSort.log
```

---

## Utility Programs — Internal Details

### SZ_basic_PZ

_Source: `gebsort/SZ_basic_PZ.c`_ ✅ verified 2026-04-17 — `SZ_basic_PZ.c:L20-56`

Simple standalone program that computes the **baseline pole-zero coefficient** for all 110 GS detectors:

```bash
SZ_basic_PZ decay_constant MM fudge_factor
# e.g.: SZ_basic_PZ 3251.342285 330 1.003
```

Output: 110 lines of `gsid  pz_value`, one per GS hole.

**Formula:** `pz = fudge_factor × exp(-MM / decay_constant)`
- `decay_constant` (1/λ) in 10 ns units — the preamplifier RC decay constant
- `MM` — trapezoid M window in samples (10 ns units)
- `fudge_factor` — small empirical correction (e.g. 1.003)

This gives a **single uniform PZ value** for all detectors as a starting point. The `fwhm_onepeak` tool then refines per-crystal values by minimizing peak width.

### SZ_factor

_Source: `gebsort/SZ_factor.c`_ ✅ verified 2026-04-17 — `SZ_factor.c:L1-142`

Extracts the **per-crystal SZ energy extrapolation factor** from a 2D ROOT histogram:

```bash
SZ_factor rootfile 2DspectrumName outputFile
# e.g.: GEBSORT/SZ_factor test.root factor dgs_factor.dat
```

**How it works:**
1. Opens the ROOT file and finds the named 2D histogram (x = GS detector ID, y = factor value)
2. For each detector (x-bin), computes the **weighted mean** of all y-bins with >5 counts
3. Writes `gsid  mean_factor` to `outputFile`

**Input spectrum `factor`** is produced by `bin_dgs` during sorting: it plots the ratio of the SZ energy to the simple energy, showing the systematic correction needed per detector.

Output: `dgs_factor.dat` — one line per crystal, read by GEBSort at runtime to scale SZ_2 energies.

---

## Energy Algorithm `.stub` Files

_Source: `gebsort/*.stub`_

The three energy algorithms (selected by `dgs_algo` in `GEBSort.chat`) are implemented as **C code stubs** that are `#include`d into `bin_dgs.c` at compile time. This keeps the main sort file identical for all algorithm variants.

| File | Algorithm | Used by |
|------|-----------|--------|
| `SZ_1.stub` | SZ method 1 (2021) — positive signal polarity | `bin_dgs.c`, `bin_dgs_GE.c` |
| `SZ_1_neg.stub` | SZ method 1 — **negative** signal polarity (sign-flipped energy formula) | `bin_dgs_AUX.c` |
| `SZ_2.stub` | SZ method 2 (2021) — two-segment PZ with t1/t2 transition times | `bin_dgs.c` (when `dgs_algo=2`) |
| `SZ_0_3456.stub` | Simple/legacy energy algorithm | `bin_dgs.c` (when `dgs_algo=0`) |
| `GEBMerge_TS_manip.stub` | Timestamp manipulation for GEBMerge | `GEBMerge.c` |

### SZ_1 Algorithm (Positive Polarity)

Implements the SZ energy method from Sept 2021 (T. Lauritsen emails):

```c
pz1 = powf(PZ[gsid], (Pars.dgs_MM + Pars.dgs_KK) / (double)Pars.dgs_MM);
sum1 = DGSEvent[i].sum1 / Pars.dgs_MM;  // normalized pre-rise sum
sum2 = DGSEvent[i].sum2 / Pars.dgs_MM;  // normalized post-rise sum

// Update baseline when inter-event gap >= 450 us:
if (d1 >= 450) base[gsid] = sum1;

// Energy (positive polarity):
Energy = sum2 - sum1 * pz1 - base[gsid] * (1 - pz1);
```

Base tracking: uses stored `DTlast[gsid]` (timestamp of previous event) to compute inter-event time in µs. LED mode uses raw timestamp difference; CFD mode masks to 30-bit counter and handles rollover.

**Note (2025-11-12):** ✅ verified 2026-04-26 — `SZ_1.stub:L19-47`. The source comment says "switched back to the standard way" but the `if (0)` guard makes the standard path (using `DGSEvent[i].dtev`) **dead code**. The active code is the else-branch DTlast path (the "supposedly better" approach). Both paths share `if (d1 >= 450) base[gsid] = sum1;`. The reason the DTlast code is preferred but the comment says otherwise is unknown; flagged for investigation.

### SZ_1_neg (Negative Polarity)

Identical to `SZ_1.stub` except the energy formula is **sign-flipped** for detectors with negative-going signals (auxiliary/AUX-type detectors):

```c
// Negative polarity:
Energy = sum1 * pz1 - sum2 + base[gsid] * (1 - pz1);  // signs reversed vs SZ_1
```

Used by `bin_dgs_AUX.c` for auxiliary detector channels.

### bin_dgs_GE.c and bin_dgs_AUX.c

`bin_dgs_GE.c` is **byte-for-byte identical** to `bin_dgs.c` (diff produces no output). ✅ verified 2026-04-24 — `diff bin_dgs.c bin_dgs_GE.c` → empty.

`bin_dgs_AUX.c` differs from `bin_dgs.c` in 5 places:

| Location | `bin_dgs.c` | `bin_dgs_AUX.c` |
|----------|------------|----------------|
| Event type filter | `DGSEvent[i].tpe == GE` | `DGSEvent[i].tpe == AUX` |
| `sum2` condition for energy | `sum1 > 0 && sum1 < sum2` | `sum1 > 0` (no upper bound) |
| Energy stub | `#include "SZ_1.stub"` | `#include "SZ_1_neg.stub"` |
| Event loop filter (×3) | `tpe == GE` | `tpe == AUX` |

**Purpose:** `bin_dgs_AUX.c` sorts events from **auxiliary detector channels** (BGO, ancillary detectors) rather than Ge crystals, using the negative-polarity energy formula appropriate for those signal types. ✅ verified 2026-04-24 — `diff bin_dgs.c bin_dgs_AUX.c`.

### bin_angcor_DGS.c — Angular Correlation Sorter

**File:** `gebsort/bin_angcor_DGS.c` (489 lines, C)  
**Purpose:** Fills 3D angular-correlation (angcor) ROOT spectra from DGS coincidence events. Requires `bin_dgs` to be active (for Doppler-corrected, energy-calibrated hits); exits with error otherwise.

**Outputs (ROOT histograms):**

| Name | Type | Description |
|------|------|-------------|
| `Gsangdiff` | `TH1D` (180 bins, 0–180°) | Distribution of inter-detector angles for all 110×110 GS detector pairs ✅ verified 2026-04-27 — `bin_angcor_DGS.c:L31` (declaration); `L43` (`#define NBINS 180`); `sup_angcor_DGS:L143` (`mkTH1D(…NBINS,0,180)`) |
| `Angcor_cube` | `TH3F` (E1 × E2 × angle, 36 angle bins 0–180°) | γ-γ angular correlation cube: filled with `(E_i, E_j, θ_ij)` for all prompt coincident Ge pairs ✅ verified 2026-04-27 — `bin_angcor_DGS.c:L278-280` (`mkTH3F(…Pars.GGMAX,1,Pars.GGMAX,Pars.GGMAX,1,Pars.GGMAX,36,0,180)`) |
| `Angcor_cube_oo` | `TH3F` (same shape) | Background angular correlation cube: filled with `(E_current, E_old, θ)` using a 15-deep event mixing queue ✅ verified 2026-04-27 — `bin_angcor_DGS.c:L43` (`#define LOQ 15`); `L281-282` (same `mkTH3F` shape as `Angcor_cube`) |
| `SMAP_DGS` | `TH2F` (361×181 bins, azimuth × polar) | Schematic detector map: plots where hits land in (azimuth, polar) space, smeared by ≤5° random disk ✅ verified 2026-04-27 — `bin_angcor_DGS.c:L145` (`mkTH2F(…361,0,360,181,0,180)`) |

**Geometry:** Polar (`angtheta[]`) and azimuthal (`angphi[]`) angles from `gsang.h` (origin unknown, attributed to I.Y. Lee). Unit vectors for all 110 detectors are computed in `sup_angcor_DGS()`. All 5,995 pair-wise inter-detector angles (one way: k < l) are pre-computed at startup and stored in `angdif[k][l]` (degrees, symmetric). The computation uses both dot-product and explicit cos formula and asserts agreement to < 0.0001° (cross-check). Angles also written to `GS_ancor_angles.txt` (sort/wc comments embedded in printf). ✅ verified 2026-04-27 — `bin_angcor_DGS.c:L258` (printf: `"110*109/2 = 5995"`); `L225` (`assert(fabs(d1-d2) < 0.0001)`); `L231` (`fopen("GS_ancor_angles.txt","w")`); `L197-208` (dot-product loop) + `L211-226` (explicit cos cross-check)

**Event filter** (in `bin_angcor_DGS()`): selects hits where `tpe==GE`, `flag==0`, `Pars.enabled[tid]`, `ehi > 0`, `ehi < GGMAX`, `1 ≤ tid < MAX_GES`. Events with < 2 surviving Ge hits are skipped. ✅ verified 2026-04-27 — `bin_angcor_DGS.c:L349-357` (6-deep nested `if` filter building `ee[]/id[]`); `L362` (`if (nn < 2) return (0)`)

**Prompt fill** (`Angcor_cube`): For every pair (k, l) with k < l, fills `(ee[k], ee[l], angdif[k][l])` **and** `(ee[l], ee[k], angdif[l][k])` — symmetric fill. ✅ verified 2026-04-27 — `bin_angcor_DGS.c:L373-378` (pair loop `k<l`; two `->Fill` calls with swapped energy order)

**Background mixing** (`Angcor_cube_oo`): Uses a FIFO queue of depth `LOQ=15` events. The current event is correlated against all 15 previous events. After fill, the queue shifts down and the current event is stored at position 0. (A note in the code mentions this could be improved with a circular buffer.) ✅ verified 2026-04-27 — `bin_angcor_DGS.c:L43` (`#define LOQ 15`); `L431-436` (triple loop m/k/l, `if(nn_oo[m]>0)` guard); `L455-470` (shift register: `while(l>0)` copies slots up; `nn_oo[0]=nn` stores current); `L462` (circular-buffer comment)

**SMAP fill** (limited to first 1,000,000 events): Each hit is projected to `(sX, sY)` in a SMAP-style azimuth/polar space using `sX = π + (azi−π)·sin(pol)`, then smeared by a random disk of radius ≤ 5° to simulate finite detector size. ✅ verified 2026-04-27 — `bin_angcor_DGS.c:L386` (`if(Pars.CurEvNo <= 1000000)`); `L389-391` (`sX=M_PI+(azi-M_PI)*sin(pol); sX/=RAD2DEG`); `L406-411` (rejection sampling `rr>5` loop)

**Detector hit-count monitor** (`angdis_hitp[tid]`): Tracks how many events each detector contributes. Exit prints per-detector hit counts and flags detectors deviating more than ±15% from the mean as "too low" / "too high". ✅ verified 2026-04-27 — `bin_angcor_DGS.c:L80-96` (`exit_angcor_DGS()`: `if(d1<85.0) printf(" too low"); if(d1>115.0) printf(" too high")` where `d1=100.0*angdis_hitp[i]/mm`)

**Required `.chat` settings:**
```
bin_dgs          # must be enabled (angcor depends on Doppler-corrected energies)
bin_angcor_DGS   # this module
```
GGMAX is set via GEBSort.chat and controls the energy axis range of the cubes. The code explicitly checks that `GGMAX² × 36 × sizeof(float) < 1073741822 bytes` (ROOT 1 GB TH3 limit).

*Source: `gebsort/bin_angcor_DGS.c` (code-read 2026-04-27)*

---


## GEBMerge, gtReceiver, GEBClient, dmpdata

Documented in: **[gebsort_merge_receive.md](gebsort_merge_receive.md)**

Covers: GEBMerge (N-way timestamp-merge), gtReceiver4 (DAQ-side GEB writer), GEBClient (VxWorks IOC → GEB sender), dmpdata (GEB file dump utility).

---


## jta.c — DGSEvDecompose_v3 Event Decoder

_Source: `gebsort/jta.c`_ (649 lines, JTA)

Contains the primary GEBSort-side digitizer event decoder: `DGSEvDecompose_v3()`. Called from `bin_dgs.c` to decompose raw GEB payloads into `DGSEVENT` structs.

### Key operations
1. **Byte-swap:** swaps all 32-bit words in the payload from big-endian (VxWorks/VME) to little-endian (host). Standard 4-byte reversal.
2. **Generic header decode** (word 0 and word 2):
   - `chan_id` = `ev[0] & 0xF` (bits 3:0)
   - `board_id` = `(ev[0] >> 4) & 0xFFF` = USER_DEF field
   - `id` = `board_id * 10 + chan_id`
   - `packet_length`, `geo_addr` from word 0
   - `header_type`, `event_type`, `header_length` from word 2
3. **48-bit timestamp:** `ev[1]` (lower 32) | `(ev[2] & 0xFFFF) << 32` (upper 16)
4. **Per-header-type decode:** switch on `header_type` (0–8, 15) → fill `DGSEVENT` fields
5. **Header tracking:** `dgsHeaderID[header_type]++` per event (global counter)

### Known bug
`pileup_count` extraction uses `(word5 & 0x00FFC000) >> 24` which always produces 0 (10-bit field at bits 14–23 shifted right 24 instead of 14). Bug is preserved in all downstream analysis code for compatibility. ✅ verified 2026-04-18 — `jta.c:L553`

### Also in jta.c
- Full set of `CLR_BITn` / `SET_BITn` / `BITn_MASK` constants and `EXTRACT_BITn` / `READ_MOD_WRITE_BITn` macros (shared across gebsort .c files)
- ⚠️ Bug in `EXTRACT_BIT29`: `>> 22` instead of `>> 29` — `jta.c:L177` ✅ verified 2026-04-25

---

## Calibration Files

| File | Format | Description |
|------|--------|-------------|
| `dgs_pz.cal` | `gsid  pz_value` | Pole-zero coefficient per crystal |
| `dgs_ehi.cal` | `gsid  gain  offset` | Energy calibration: E_keV = gain×e_raw + offset |
| `dgs_factor.dat` | `gsid  factor` | SZ energy extrapolation factor |
| `map.dat` | `gsid  dig_id  channel` | DGS detector → digitizer channel map |

---

## bin_dgs.c — Main DGS Sort Function Internals

Source: `DGS_tools_pack/gebsort/bin_dgs.c` (1681 lines, ROOT + C)

`bin_dgs.c` is the primary GEBSort binning module for DGS Ge/BGO data. It defines three entry-point functions called by the GEBSort framework:

| Function | Purpose |
|----------|---------|
| `sup_dgs()` | Setup: read map file, create ROOT histograms, load calibration files, init arrays |
| `bin_dgs(GEB_EVENT *GEB_event)` | Per-coincidence-event processing; the hot path |
| `exit_dgs()` | Finalization: print statistics, dump `d2.cmd`, print baselines/rates |

### Compile-time flags

| `#define` | Default | Effect |
|-----------|---------|--------|
| `ALL2DS` | 0 | Enable extra 2D per-detector histograms (`e2_e1vse1`, `SZe2_e1vse1`, etc.) |
| `TRACE` | 0 | Enable per-detector trace matrices (`NTRACES=200`) |
| `SZ_EXTRA` | 1 | Enable SZ diagnostic spectra (`vdcXsum1_NNN`, `eXsum1_NNN`, `spbaseXbase_NNN`, etc.) |

### map.dat format

`sup_dgs()` reads `map.dat` (required; exits if missing). Each line:
```
NNNN  MM  KK  label
  |    |   |
  |    |   +--- id number for that type (e.g. detector 1..110 for GE)
  |    +------- type id: GE=1, BGO=2, SIDE=3, AUX=4, DSSD, FP, XARRAY, CHICO2, SSD, CLOVER, SPARE, SIBOX
  +------------ channel label = digitizer_id*10 + channel (0–9)
```
Duplicate channel labels are detected and cause a fatal error. Channel type dispatched to `tlkup[]` and `tid[]` arrays (size `NCHANNELS`).

### Calibration files loaded in `sup_dgs()` / `getcal()`

| File | `Pars` field | Default if missing |
|------|-------------|--------------------|
| `dgs_ecalfn` (energy cal) | `Pars.dgs_ecalfn` | gain=1.0, offset=0.0 |
| `dgs_PZfn` (pole-zero) | `Pars.dgs_PZfn` | PZ[i]=1.0 for all i |
| `dgs_factorfn` (SZ factor) | `Pars.dgs_factorfn` | factor[i]=0.0 for all i |
| `last_baseline.txt` | (optional) | start from 0 if absent |
| `baseline_limits.txt` | (optional) | hibaselim=4000, lobaselim=0 per det |

Energy cal file format: `ge_id  offset  gain` (3 fields; note: offset read as `c`, gain as `d`).
PZ file format: `ge_id  pz_value`.
Factor file format: `ge_id  factor_value`.

### ROOT histograms created in `sup_dgs()`

| Name | Type | Description |
|------|------|-------------|
| `EvntCounter` | TH1D 14400 bins | Events/sec for up to 4 hours |
| `GErate` | TH2F 14400×NGE | Ge detector rate vs time |
| `BGOrate` | TH2F 14400×NGE | BGO rate vs time |
| `EhiRaw` | TH2F LENSP×NGE | Raw energy (after gain cal) per det |
| `EhiRawRaw` | TH2F LENSP×NGE | Raw energy before any cal |
| `EhiCln` | TH2F LENSP×NGE | Clean (BGO-unsuppressed) energy per det |
| `EhiCln_nodop` | TH2F LENSP×NGE | Clean energy without Doppler correction |
| `EhiDrty` | TH2F LENSP×NGE | Dirty (BGO-suppressed) energy per det |
| `GeBGO_DT` | TH2F 400×NGE | Ge–BGO time difference (±200 10ns units) |
| `pzraw` | TH2F NGE×2000 | PZ diagnostic (0–2.0 range) |
| `gg` | TH2F GGMAX×GGMAX | Gamma-gamma coincidence matrix |
| `baseXid` | TH2F 4000×NGE | SZ baseline vs detector id |
| `dtev` | TH2F NGE×10000 | Time since last event per det (μs, 0–10000) |
| `dgs_sumtraceXge1` | TH2F NGE×4000 | Summed traces (raw) vs geid |
| `dgs_sumtraceXge2` | TH2F NGE×4000 | Summed traces (base-subtracted) vs geid |
| `factor` | TH2F NGE×1000 | SZ2 extrapolation factor |
| `eXdtev` | TH2F 1500×500 | Energy vs dtev |
| `tt1` / `tt2` | TH2F | CFD timing: pair time diff (Bi-207 calibration) |
| `rate_dgs_sec` | TH2F NGE×RATELEN | Rate vs time in seconds |

Decay-mode histograms (created only when `Pars.have_decay_data` is set):
- `BetaCounter`, `GeBeta_DT`, `GeBeta_ID`, `GeTape_DT`
- `gg_decay1/2/3` and `gg_decay1/2/3_beta` — three time windows, with/without coincident beta

SZ_EXTRA per-detector histograms (compiled-in when `SZ_EXTRA=1`):
- `vdcXsum1_NNN`, `vdcXsum1_NNN_a/b` — SZ baseline tracking plots
- `baseXbase_NNN` — SZ vs sampled baseline
- `eXsum1_NNN` — energy vs sum1
- `eXdtev_det_NNN` — energy vs dtev per det
- `sum1Xdtev_NNN` — sum1 vs dtev
- Global: `sampled_baselineXgid`, `sampled_baselinesum1Xgid`, `cor_factorXgid`

### bin_dgs() per-event processing flow

1. **Decompose GEB event** — loop `GEB_event->mult` hits; filter `GEB_TYPE_DGS` type; call `DGSEvDecompose_v3()` (from `jta.c`) into `DGSEvent[ng]`; count gammas in `ng`.

2. **CFD sub-clock interpolation** — if `cfd_valid_flag`, extract 3 CFD samples (`cfd_sample_0/1/2`) as signed 14-bit values (shift left 2, interpret as `signed short`), then find zero-crossing by linear interpolation between adjacent 10ns samples to get `event_cfd_timestamp` with sub-ns precision. (JTA's alternative sign-extend method also present but `#if 0`'d out.) ✅ verified 2026-04-26 — `bin_dgs.c:L887-931` (`cfd_valid_flag` check at L887; L895-899 shift-left-2 + signed-short cast; `#if 0` alt method at L913-931)

3. **Rate tracking** — `EvTimeStam0` captures first-event timestamp; `RelEvT = (TS - TS0) / 100000000` maps to seconds.

4. **Per-Ge-hit inner loop** (`for i=0..ng`):
   - Guard: `gsid` range check (1..NGE); prints warning and sets gsid=0 if out of range.
   - **Summed trace** accumulation: first `NAVETRACE=1000` traces per detector are averaged into `dgs_sumtraceXge1/ge2`.
   - **dtev (time-since-last-event)** computation:
     - LED timing (header_type odd): `dtev = event_timestamp − last_disc_timestamp` (47-bit full TS)
     - CFD timing (header_type even): both timestamps masked to 30 bits (`& 0x3fffffff`); if result ≤ 0, add `0x40000000` to handle rollover
     - Convert to μs: divide by `MicroSECOND = 100` (1 tick = 10 ns, 100 ticks = 1 μs) ✅ verified 2026-04-26 — `bin_dgs.c:L832` (`MicroSECOND = 100`)
   - **Energy extraction** via `#include`d stub (selected by `Pars.dgs_algo`):
     - Guard: `sum1 > 0 && sum1 < sum2` required; otherwise Energy=0, bad counters incremented
     - `case 0`: `#include "SZ_0_3456.stub"` — legacy M/K trapezoid
     - `case 1`: `#include "SZ_1.stub"` — SZ 2021 method 1
     - `case 2`: `#include "SZ_2.stub"` — SZ 2021 method 2 (requires t1>t2>0)
   - **Gain calibration**: `Energy = Energy * ehigain[gsid] + ehioffset[gsid]`
   - **Good/bad energy counting**: good if `10.0 < Energy < 4000.0`
   - **Doppler correction**: `Energy *= (1 - β·cos θ) / √(1-β²)` using `angtheta[gsid-1]` (degrees→radians); applies only if `Pars.beta != 0` ✅ verified 2026-04-26 — `bin_dgs.c:L1197-1203` (`d1 = angtheta[gsid-1]/57.29577951`; `Energy *= (1 - Pars.beta * cos(d1)) / sqrt(1 - Pars.beta * Pars.beta)`)
   - **Compton suppression**: inner j-loop scans all hits for matching BGO (`tpe==BGO && tid==gsid`); if `|Ge_TS - BGO_TS| ≤ 50` (500 ns), sets `DGSEvent[i].flag = 1` (dirty) ✅ verified 2026-04-26 — `bin_dgs.c:L1244-1256` (`abs(tdiff) <= 50`; flag=1)
   - **BGO ehi**: `ehi = sum2 - sum1` (simple difference, no baseline correction)
   - **Decay/tape logic** (if `have_decay_data`): detects TAPE_MOVED (type 10), EBIS_CLOCK (type 11), BETA_FIRED (type 12) from map-assigned special channels ✅ verified 2026-04-26 — `bin_dgs.c:L30-32` (`#define TAPE_MOVED 10`, `EBIS_CLOCK 11`, `BETA_FIRED 12`)

5. **Binning loop** (`for i=0..ng` second pass):
   - Fill `EhiRawRaw`, `EhiRaw`, `EhiCln`/`EhiCln_nodop` (if flag==0), `EhiDrty` (if flag==1)
   - Beta-gated histograms: `GeBeta_DT`, `GeBeta_ID`, `GeTape_DT` with configurable time windows

6. **gg matrix loop** (nested i/j, i<j, both GE+clean+energy>0):
   - Fills `gg` symmetrically (`[i][j]` and `[j][i]`)
   - Decay mode: also fills `gg_decay1/2/3` (configurable tape-relative time windows) and `_beta` variants

7. **CFD timing diagnostics** (Bi-207 calibration check):
   - If exactly 2 clean gammas and one is near 569 or 1063 keV: fills `tt1` (pair index × dt) and `tt2` (vs reference detector 9)

### exit_dgs() statistics

- Per-header-ID hit counts (types 0–19) with percentage
- Per-detector good/bad energy fraction
- Per-detector last baseline
- Per-detector mean dtev and rate from the `dtev` histogram (writes `d2.cmd` with ROOT commands)
- Bad sum1/sum2 rates per detector

### Key global arrays

| Variable | Size | Description |
|----------|------|-------------|
| `tlkup[]` | NCHANNELS | Map: channel label → detector type (GE/BGO/etc.) |
| `tid[]` | NCHANNELS | Map: channel label → detector id |
| `ehigain[]` | NGE+1 | Energy calibration gain per detector |
| `ehioffset[]` | NGE+1 | Energy calibration offset per detector |
| `PZ[]` | NGE+1 | Pole-zero correction per detector |
| `factor[]` | NGE | SZ2 extrapolation factor per detector |
| `ave_base[]` | NGE+1 | Running average baseline (persisted to `last_baseline.txt`) |
| `base[]` | NGE+1 | Current baseline per detector |
| `DTlast[]` | NGE | Timestamp of last event per detector (for dtev) |
| `baselast[]` | NGE | Last baseline per detector |
| `sum1last[]` | NGE | Last sum1 per detector |
| `nn_all[]` | NGE+1 | Total hits per detector |
| `nn_badsum1[]` | NGE+1 | Hits with sum1 < 1 |
| `nn_badsum12[]` | NGE+1 | Hits with sum2 < sum1 |
| `ngood_e[]` | NGE+1 | Good energies (10–4000 keV) per detector |
| `nbad_e[]` | NGE+1 | Bad energies per detector |

---

## Comparison with parquet_pysort

| Feature | GEBSort | parquet_pysort |
|---------|---------|----------------|
| Language | C + ROOT | Python (PyArrow, C++ lib) |
| Output | ROOT `.root` histograms + spectra | Parquet columnar files |
| Energy algo | SZ_1, SZ_2 (via `dgs_algo`) | SZ_1, SZ_2 (via `dgs-algo`) |
| Event building | Time-window coincidence | Time-window coincidence |
| Calibration | Offline scripts (`fwhm_onepeak`, `dgs_ecal`) | `pz_from_parquet.py`, `gain_from_parquet.py` |
| Interactive analysis | GammaWare, ROOT scripts | `parquetCLI` REPL |
| Rate | Slower (single-threaded sort) | Faster (parallel decode + sort) |
| Use case | Traditional workflow, angular correlations, TAC | Modern workflow, Parquet ecosystem |

Both read the same GEB binary files. For DGS exp2008_Chiara, parquet_pysort is the active workflow.

---

## GRETINA-Specific and Additional Detector Sorters

Documented in: **[gebsort_additional_sorters.md](gebsort_additional_sorters.md)**

Covers: `bin_angcor_GT`, `bin_DCO_GT`, `bin_g4sim`, `bin_gtcal`, `bin_ft`, `bin_final`, `bin_tac2`, `bin_dub`, `bin_XA`, `bin_dfma`, `bin_angdis`, `bin_ndc`, `bin_mux`, `bin_s800`, `bin_linpol`, `bin_mode3`.

---
## Cross-References

### Sub-files (split from this document)
- [gebsort_merge_receive.md](gebsort_merge_receive.md) — GEBMerge, gtReceiver, GEBClient, dmpdata detail
- [gebsort_additional_sorters.md](gebsort_additional_sorters.md) — GRETINA-specific + additional detector sorters (bin_tac2, bin_XA, bin_dfma, etc.)

### Related knowledge base files
- [run_procedures.md](run_procedures.md) — Full DGS run workflow: where GEBSort fits in (PZ cal → energy cal → sort)
- [dgs_analysis.md](dgs_analysis.md) — Modern alternative: fastEventConstructor (ROOT) + parquet_pysort pipeline
- [data_structures.md](data_structures.md) — GEB binary format: the input data format GEBSort reads
- [pole_zero.md](pole_zero.md) — PZ correction theory; dgs_pz.cal consumed by GEBSort's bin_dgs
- [tac2.md](tac2.md) — TAC-II TDC format: sorted by `bin_tac2`; TDC event packet structure
- [ANLDAQ.md](ANLDAQ.md) — tcpReceiverMT produces the raw GEB files that GEBSort/GEBMerge processes
- [ANLDAQ_tcpReceiver.md](ANLDAQ_tcpReceiver.md) — tcpReceiverMT deep-dive: packet parsing, FIFO readout, GEB event assembly; contrast with gtReceiver in gebsort_merge_receive.md
- [guceiver.md](guceiver.md) — Guceiver live receiver GUI: live waveform/spectrum display from the same TCP data stream
- [nfs_layout.md](nfs_layout.md) — NFS paths where experiment data and GEBSort binaries live (vol4/dgs_testing/)
- [gebsort_utilities.md](gebsort_utilities.md) — Minor utility programs and support files (listTS, GTPrint, dtbtev, DataExtract, GEBCrop, GEBFilter, GEBSplit, GEBHeader, utils, spe_fun, tlutil, tlutil2, trig_fun, 2d_fun, etc.)

---

## Minor Utility Programs and Support Files

_Moved to [`gebsort_utilities.md`](gebsort_utilities.md) (2026-04-27 — parent file exceeded 500-line threshold)._

See [gebsort_utilities.md](gebsort_utilities.md) for all minor utility and support file documentation.

---
*Created: 2026-04-07. Source: `DGS_tools_pack/gebsort/` README + source files.*
*Split: 2026-04-27 — utility programs moved to `gebsort_utilities.md`.*
