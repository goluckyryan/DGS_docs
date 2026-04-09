# GEBSort — GRETINA/DGS Event Builder and Sorter

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
| `GEBMerge` | Merges multiple GEB data streams by timestamp |
| `GEBFilter` | Filters GEB events by type/condition |
| `GEBCrop` | Crops/trims GEB files |
| `GEBSplit` | Splits GEB files |
| `GEBHeader` | Reads/prints GEB headers |
| `find_MK` | Finds optimal M and K trapezoid parameters from data |
| `mk_dgs_map` | Generates the DGS detector map file |
| `fwhm_onepeak` | Finds best PZ value by minimizing peak FWHM |
| `SZ_factor` | Extracts the SZ energy extrapolation factor |
| `dgs_ecal` | Automatic energy calibration from known source |

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
dgs_algo    2              # Energy algorithm: 0=simple, 1=SZ_1, 2=SZ_2
dgs_MM      350            # Trapezoid M window (samples) ✅ verified 2026-04-08 — README.bin_dgs:L11
dgs_KK      141            # Trapezoid K window (samples) ✅ verified 2026-04-08 — README.bin_dgs:L12
dgs_PZ      dgs_pz.cal    # Pole-zero calibration file
dgs_ecal    dgs_ehi.cal   # Energy gain/offset calibration file
dgs_factor  dgs_factor.dat # SZ energy factor file
dgs_SZ_t1   50             # SZ_2 transition time 1 (samples)
dgs_SZ_t2   20             # SZ_2 transition time 2 (samples)
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

- `dgs/run_procedures.md` — GEBSort workflow in context of DGS runs
- `dgs/dgs_analysis.md` — parquet_pysort (modern alternative)
- `dgs/pole_zero.md` — PZ correction physics and algorithms
- `dgs/data_structures.md` — GEB binary format
- `dgs/tac2.md` — TAC-II TDC (sorted by `bin_tac2`)

---
*Created: 2026-04-07. Source: `DGS_tools_pack/gebsort/` README + source files.*
