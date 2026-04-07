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
- `350` — M value (in 10 ns units = 3.50 µs; from `caput GLBL:DIG:m_window`) ✅ verified 2026-04-06 — `MDigUser.template`: all window PVs use `ESLO=0.010` (µs/count), so raw register = EGU/0.01; 3.5 µs → 350 counts
- `141` — K value (in 10 ns units); calculated as sum of all K+D windows: ✅ verified 2026-04-06 — k_window, k0_window, d_window, d3_window all use ESLO=0.010 in MDigUser.template
  - K = k_window + d_window + k0_window + d3_window + D2_fixed
  - Example: 0.20 + 0.06 + 0.80 + 0.20 + 0.15 = 1.41 µs = **141 in 10 ns units** *(note: example uses 0.15 µs for D2 — see note below)*
  - Note: D values are included in K per S. Zhu convention (6/25/18); D2 is firmware-internal (not user-settable via EPICS, per JTA 6/26/18). Register name is **reg_d3_window** (addr 0x240–0x264, confusingly named 'd3' in firmware but represents algorithm's 'd2'). Production default = **23 clocks = 0.23 µs** (at 100 MHz). Simulation testbenches use 21–22 clocks. The 0.15 µs figure in the original K-value example may be a specific calibration run's value, not the firmware default. ✅ verified 2026-04-06 — `DIG/MAIN_FPGA/BuildBranches/DGS/Source/Registers.vhd:L186` (`to_std_logic_vector(23,32)`) + `jta_channel.vhd:L59` (comment "delay factor 'd2'" on reg_d3_window port)
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

Before sorting, verify `map.dat` is current and matches the array configuration. This file maps DAQ channel IDs (`board_id × 10 + chan_id`) to crystal IDs (`tid`) and detector types (GE/BGO/SIDE/AUX).

**Note:** `map.dat` is a **per-experiment input file** — not a committed file in the repo. It must be provided by the user and passed via `--map-file map.dat` to `make_filemap_dgs.py`. There is no static copy in the repo. ✅ verified 2026-04-07 — `make_filemap_dgs.py`: `default=Path("map.dat")` + CLAUDE.md: "DAQ channel ID → crystal ID mapping (columns: id, type, tid)"

Columns: `id` (DAQ channel), `type` (GE/BGO/SIDE/AUX), `tid` (crystal ID). Loaded by C++ as `tlkup[]`/`tid[]` arrays.

See `gammasphere_geometry.md` for the GS hole geometry.

---

## GitLab Repos Used

| Repo | URL |
|------|-----|
| GEBSort | `https://gitlab.phy.anl.gov/tlauritsen/GEBSort.git` |
| trackMain | `https://gitlab.phy.anl.gov/tlauritsen/trackMain.git` |

---

## Modern Workflow (exp2008_Chiara / ANLDAQ era)

The current experiment uses a Python+Parquet pipeline instead of GEBSort. Key differences:

| Step | Legacy (GEBSort) | Modern (ANLDAQ) |
|------|-----------------|------------------|
| Run control | `gcdaq`/`bgscdaq` + `gebsort.sh` | `start_run.sh` / `stop_run.sh` |
| Data format | GEB binary files | GEB binary → Parquet via `RunParquet` |
| Event building | GEBSort C++ | `fastEventConstructor` (C++/ROOT) |
| PZ calibration | `dgs_pz` binary | `pz_from_parquet.py` or `pz_from_evtparquet.py` |
| Energy calibration | `dgs_ecal` binary | `gain_from_parquet.py` (AutoFitter + GrayCAL) |
| Output | ROOT TTrees (GEBSort format) | ROOT TTrees (EventBuilder format) |

### Modern Run Flow Summary

```bash
# 1. Download raw GEB data from NFS
bash working/DownloadRaw.sh

# 2. Decode GEB → Parquet (hit-level)
./working/RunParquet --decode-only <run_files> --output expFolder/Parquet/decode/

# 3. PZ calibration from hit-level parquet
python working/pz_from_parquet.py expFolder/Parquet/decode/exp_003_dgs.parquet \
    --output working/dgs_pz.cal

# 4. Energy calibration (152Eu source)
python working/gain_from_parquet.py expFolder/Parquet/decode/exp_003_dgs.parquet \
    --output working/dgs_gain.cal

# 5. Full decode + event build → Parquet (event-level)
./working/RunParquet <run_files> --pz-cal dgs_pz.cal --gain-cal dgs_gain.cal \
    --output expFolder/Parquet/events/

# 6. Build ROOT events (parallel k-way merge)
./armory/fastEventContructor/EventBuilder_PQ \
    out.root <timeWindow_ns> 0 0 12 4 <parquet_files...>
```

See `dgs/dgs_analysis.md` for full details on each step.

---

*Created: 2026-04-05 from [wiki: Typical DGS Run Procedures](https://wiki.anl.gov/gsdaq/Typical_DGS_run_procedures)*
*Updated: 2026-04-07 — added Modern Workflow section*
