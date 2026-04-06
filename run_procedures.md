# Typical DGS Run Procedures

**Source:** https://wiki.anl.gov/gsdaq/Typical_DGS_run_procedures  
**Note:** This page describes legacy (pre-ANLDAQ) procedures using GEBSort. Current experiment (exp2008_Chiara) uses `start_run.sh`/`stop_run.sh` from ANLDAQ. Cross-check with `expMemory_2008_Chiara.md` for current setup.

---

## Directory Structure

Standard experiment directory layout:

```
gsfmannn/          ← experiment directory (nnn = run number)
├── dfmadata/
├── dgsdata/
├── xadata/
├── GEBSort/
├── LOG_FILES/
├── Merged/
└── ROOT_FILES/
```

### Setup (Option 1 — preferred)

```bash
cd /dk/fs2/dgs
tar -zxvf dgs_template.tgz
mv template gsfmannn    # replace nnn with run number

cd gsfmannn
(cd GEBSort; git pull && make -B)
(cd trackMain; git pull && make -B)
```

### Setup (Option 2 — manual clone)

```bash
git clone https://gitlab.phy.anl.gov/tlauritsen/trackMain.git
(cd trackMain; make -B)
git clone https://gitlab.phy.anl.gov/tlauritsen/GEBSort.git
(cd GEBSort; make -B)

# Link crmat files (needed by GEBSort, kept in trackMain)
cd GEBSort
ln -sf ../trackMain/GANIL_AGATA_crmat.dat GANIL_AGATA_crmat.dat
ln -sf ../trackMain/GSI_AGATA_crmat.dat GSI_AGATA_crmat.dat
ln -sf ../trackMain/crmat.LINUX crmat.LINUX
```

---

## Run Control

```bash
# Start run (in data directory)
start_run.sh 123

# Stop run
stop_run.sh

# Merge data files from run 123
gebmerge.sh 123
# → merged file in Merged/, log in LOG_FILES/
# Note: run gebmerge on a different machine to avoid disrupting receivers
```

---

## GEBSort Configuration (`GEBSort.chat`)

Key lines to check/modify before sorting:

```
bin_dgs
beta 0.0
dgs_MM 350          ← M window value (in 10 ns units); 350 = 3.50 µs
dgs_PZ dgs_pz.cal   ← pole-zero calibration file
dgs_ecal dgs_ehi.cal ← energy calibration file
```

### Tape station / beta decay (optional):
```
decay_station_bg -10 40          # beta-gamma coincidence window
decay_station_ggdt 20            # max time between gammas in 2D matrices
decay_station_gt1 10 617         # gg decay time gates
decay_station_gt2 620 892
decay_station_gt3 1000 3000
```

---

## Sorting

```bash
cd GEBSort
gebsort.sh 123
# → ROOT_FILES/run123.root

# View results
rootn.exe
dload("../ROOT_FILES/run123.root")
```

---

## Calibrations (GEBSort_nogeb / bin_dgs)

### Step 1 — Generate PZ (Pole-Zero) spectra

Enable in `bin_dgs.c` and recompile:
```c
#define ALL2DS 1
```

Sort a **²⁰⁷Bi source** run, then extract PZ spectra:
```bash
GEBSort_nogeb ....
rootn.exe
dload("bi.root")
.x get_pz.cc
```

### Step 2 — Calculate PZ values

```bash
dgs_pz 350 141 dgs_pz.cal 1.003
```

Arguments:
- `350` — M value (in 10 ns units = 3.50 µs; from `caput GLBL:DIG:m_window`)
- `141` — K value (in 10 ns units); calculated as sum of all K+D windows:
  - K = k_window + d_window + k0_window + d3_window + D2_fixed(0.15 µs)
  - Example: 0.20 + 0.06 + 0.80 + 0.20 + 0.15 = 1.41 µs = **141 in 10 ns units**
  - Note: D values are included in K per S. Zhu convention (6/25/18); D2=0.15 µs fixed (not user-settable, per JTA 6/26/18)
- `dgs_pz.cal` — output calibration file
- `1.003` — PZ fudge factor (FF); determined from energy vs baseline spectra

Output: `d_pz.cmd` — use in `gf3` to check PZ spectra. Bad detectors can be set to average PZ value by editing `dgs_pz.cal` manually.

### Step 3 — Energy calibration

```bash
# Remove old energy cal (reset to defaults: offset=0, gain=1)
rm dgs_ehi.cal

# Re-sort with new PZ values
gebsort.sh 123

# Extract clean uncalibrated ehi spectra
rootn.exe
dload("run123.root")
.x get_ecln.cc

# Run energy calibration (source options: 207Bi, 88Y, 60Co)
dgs_ecal dgs_ehi.cal 207Bi 600 1.0
# 600 = lowest channel to search (avoids noise/x-rays)
# 1.0 = calibration factor (1 keV/ch in this case)
```

Final sort uses both `dgs_pz.cal` (PZ) and `dgs_ehi.cal` (gain/offset).

> ⚠️ `dgs_pz` and `dgs_ecal` can be fooled by noise — inspect spectra and manually fix outliers in the `.cal` files.

---

## map.dat — Detector Map

Before sorting, verify `map.dat` is current and matches the array configuration. This file maps DAQ channel IDs to GS holes and detector types.

Location: `DGS_tools_pack/dgs_analysis/armory/parquet_pysort/map.dat`

See `gammasphere_geometry.md` for the GS hole geometry.

---

## GitLab Repos Used

| Repo | URL |
|------|-----|
| GEBSort | `https://gitlab.phy.anl.gov/tlauritsen/GEBSort.git` |
| trackMain | `https://gitlab.phy.anl.gov/tlauritsen/trackMain.git` |

---

*Created: 2026-04-05 from [wiki: Typical DGS Run Procedures](https://wiki.anl.gov/gsdaq/Typical_DGS_run_procedures)*
