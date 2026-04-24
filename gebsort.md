# GEBSort — GRETINA/DGS Event Builder and Sorter

Stability: C2 - Active / semi-stable

_Source: `DGS_tools_pack/gebsort/` (cloned from `https://gitlab.phy.anl.gov/tlauritsen/gebsort`)_
_Author: T. Lauritsen (ANL). Used by DGS, GRETINA, X-Array, DUO, and other detector systems._

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

**Note (2025-11-12):** Code comments indicate the "supposedly better code" for baseline tracking was reverted to the standard form; the reason is unknown and flagged for investigation.

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

---

## GEBMerge

Merges multiple single-crate GEB data files into one timestamp-sorted stream:
```bash
GEBMerge -input file1.geb file2.geb ... -output merged.gtd_000 -chat GEBMerge.chat
```
Configuration in `GEBMerge.chat`. Output format is standard GEB binary.

**Note:** For DGS, the `ANLDAQ/tcpReceiverMT` writes per-IOC files which must be merged before GEBSort can build coincidence events.

---

## Calibration Files

| File | Format | Description |
|------|--------|-------------|
| `dgs_pz.cal` | `gsid  pz_value` | Pole-zero coefficient per crystal |
| `dgs_ehi.cal` | `gsid  gain  offset` | Energy calibration: E_keV = gain×e_raw + offset |
| `dgs_factor.dat` | `gsid  factor` | SZ energy extrapolation factor |
| `map.dat` | `gsid  dig_id  channel` | DGS detector → digitizer channel map |

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

## Related

- `knowledgeBase/run_procedures.md` — GEBSort workflow in context of DGS runs
- `knowledgeBase/dgs_analysis.md` — parquet_pysort (modern alternative)
- `knowledgeBase/pole_zero.md` — PZ correction physics and algorithms
- `knowledgeBase/data_structures.md` — GEB binary format
- `knowledgeBase/tac2.md` — TAC-II TDC (sorted by `bin_tac2`)

---
*Created: 2026-04-07. Source: `DGS_tools_pack/gebsort/` README + source files.*

## Cross-References

- `knowledgeBase/run_procedures.md` — Full DGS run workflow: where GEBSort fits in (PZ cal → energy cal → sort)
- `knowledgeBase/dgs_analysis.md` — Modern alternative: fastEventConstructor (ROOT) + parquet_pysort pipeline
- `knowledgeBase/data_structures.md` — GEB binary format: the input data format GEBSort reads
- `knowledgeBase/pole_zero.md` — PZ correction theory; dgs_pz.cal consumed by GEBSort's bin_dgs
- `knowledgeBase/ANLDAQ.md` — tcpReceiverMT produces the raw GEB files that GEBSort processes
- `knowledgeBase/nfs_layout.md` — NFS paths where experiment data and GEBSort binaries live (vol4/dgs_testing/)
