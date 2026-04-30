# Typical DGS Run Procedures

Stability: C2 - Active / semi-stable

**Source:** https://wiki.anl.gov/gsdaq/Typical_DGS_run_procedures  
**Note:** This page describes legacy (pre-ANLDAQ) procedures using GEBSort. Current experiment (exp2008_Chiara) uses `start_run.sh`/`stop_run.sh` from ANLDAQ. Cross-check with `expMemory_2008_Chiara.md` for current setup.

---

## Table of Contents

- [Directory Structure](#directory-structure)
- [Run Control](#run-control)
- [GEBSort Configuration](#gebsort-configuration-gebsortchat)
- [Sorting](#sorting)
- [Calibrations (GEBSort\_nogeb / bin\_dgs)](#calibrations-gebsort_nogeb--bin_dgs)
- [map.dat — Detector Map](#mapdat--detector-map)
- [GitLab Repos Used](#gitlab-repos-used)
- [Modern Workflow (exp2008\_Chiara / ANLDAQ era)](#modern-workflow-exp2008_chiara--anldaq-era)
- [Cross-References](#cross-references)
- [DGS Commander EDM Screens](#dgs-commander-edm-screens)
- [Shift Operator Guide](#shift-operator-guide-8-hour-shift)

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
start_run.sh "run comment here"   # arg1 = optional comment; run number auto-incremented from expInfo.sh

# Stop run
stop_run.sh

# Merge data files from run 123
gebmerge.sh 123
# → merged file in Merged/, log in LOG_FILES/
# Note: run gebmerge on a different machine to avoid disrupting receivers
```

> ✅ verified 2026-04-14 — `start_run.sh:L49` (`argComment="$1"`); run number comes from `NEXT_RUN` in `expInfo.sh`, not a CLI argument. See `knowledgeBase/ANLDAQ.md` §expInfo.sh for full setup.

---

## GEBSort Configuration (`GEBSort.chat`)

Key lines to check/modify before sorting:

```
bin_dgs
beta 0.0
dgs_MM 350          ← M window value (in 10 ns units); 350 = 3.50 µs ✅ verified 2026-04-17 — `basic_settings_DGS.sh:L46` (`m_window=3.5`) = 3.5 µs = 350 in 10 ns units. Note: `GEBSort.chat:L219` currently shows 200 (different template; must match the m_window used during data taking)
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

> Note: `dgs_pz` here is the **GEBSort standalone binary** (`gebsort/dgs_pz.c`), not the `dgs_PZ` GEBSort.chat keyword. The binary computes PZ coefficients from ROOT histograms; the chat keyword loads the `.cal` file. ✅ verified 2026-04-15 — `gebsort/dgs_pz.c` exists; `GEBSort.chat:L223` uses `dgs_PZ dgs_pz.cal` (keyword, no M/K args).

Arguments:
- `350` — M value (in 10 ns units = 3.50 µs; from `caput GLBL:DIG:m_window`) ✅ verified 2026-04-06 — `MDigUser.template`: all window PVs use `ESLO=0.010` (µs/count), so raw register = EGU/0.01; 3.5 µs → 350 counts
- `141` — K value (in 10 ns units); calculated as sum of all K+D windows: ✅ verified 2026-04-06 — k_window, k0_window, d_window, d3_window all use ESLO=0.010 in MDigUser.template
  - K = k_window + d_window + k0_window + d3_window + D2_fixed
  - Example: 0.20 + 0.06 + 0.80 + 0.20 + 0.15 = 1.41 µs = **141 in 10 ns units** *(note: example uses 0.15 µs for D2 — see note below)*
  - Note: D values are included in K per S. Zhu convention (6/25/18); D2 is firmware-internal (not user-settable via EPICS, per JTA 6/26/18). Register name is **reg_d3_window** (addr 0x240–0x264, confusingly named 'd3' in firmware but represents algorithm's 'd2'). Production default = **23 clocks = 0.23 µs** (at 100 MHz). Simulation testbenches use 21–22 clocks. **The 0.15 µs hidden fixed value** is hardcoded in `find_MK.c:L87` (`r1 += 0.15; printf("hidden = %f us\n", 0.15)`) — this is not a register, it is a constant representing D2 in the sorting algorithm. ✅ verified 2026-04-10 — `gebsort/find_MK.c:L87` confirms 0.15 µs hardcoded; `Registers.vhd:L186` (`to_std_logic_vector(23,32)`) gives firmware default 23 clocks = 0.23 µs ≠ 0.15 µs — the difference is that find_MK uses 0.15 µs as the sorting D2 constant regardless of firmware register value.
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

# 2. Decode GEB → Parquet (hit-level) via RunParquet
#    Signature: RunParquet [--decode-only] [--map-file <file>] <expInfo.sh> <run_number> [TIMEWIN] [THREADS]
#    TIMEWIN default: 1000 ticks (10 µs); THREADS default: 78
#    Note: comment in script header says 800 but actual code default is 1000 (RunParquet:L67)
./working/RunParquet --decode-only ~/ANLDAQ/tcpReceiver/expInfo.sh 3
# ✅ verified 2026-04-14 — RunParquet:L1-23 (usage); L67-68 (TIMEWIN=1000, THREADS=78 actual defaults)

# 3. PZ calibration from hit-level parquet
python working/pz_from_parquet.py expFolder/Parquet/decode/exp2008_003_dgs.parquet \
    --output working/dgs_pz.cal

# 4. Energy calibration (152Eu source)
python working/gain_from_parquet.py expFolder/Parquet/decode/exp2008_003_dgs.parquet \
    --output working/dgs_gain.cal

# 5. Full decode + event build → Parquet (event-level)
./working/RunParquet ~/ANLDAQ/tcpReceiver/expInfo.sh 3 1000 40
# (no --pz-cal/--gain-cal flags; RunParquet does decode+event_builder only;
#  pole-zero and gain are applied separately in analysis, not in RunParquet)

# 6. Build ROOT events (parallel k-way merge) via ProcessRUN (preferred over RunParquet)
./armory/fastEventContructor/EventBuilder_PQ \
    out.root <timeWindow_ns> 0 0 12 4 <parquet_files...>
```

See `knowledgeBase/dgs_analysis.md` for full details on each step.

---

## Cross-References

| Topic | File |
|-------|------|
| Full analysis pipeline (EventBuilder variants, RunParquet, parquetCLI) | `knowledgeBase/dgs_analysis.md` |
| Pole-zero correction theory + `pz_from_parquet.py` | `knowledgeBase/pole_zero.md` |
| Troubleshooting IOC, FIFO, link lock issues | `knowledgeBase/troubleshooting.md` |
| Trigger bring-up (5-stage SERDES link-up scripts) | `knowledgeBase/trig_setup_scripts.md` |
| DAQ GUI (ANLDAQ commander, data-taking tab) | `knowledgeBase/ANLDAQ.md` |
| Run control scripts (`start_run.sh`, `stop_run.sh`, `run_control_gui.py`) deep-dive | `knowledgeBase/ANLDAQ_tcpReceiver.md` §Run Control Scripts |
| DIG firmware — readout modes, data formats | `knowledgeBase/DIG_firmware_expert.md` |
| GEB data format + type codes | `knowledgeBase/data_structures.md`, `knowledgeBase/dgs_analysis.md` § GEB |
| GEBSort full reference (all programs, GEBSort.chat, find_MK, fwhm_onepeak, dgs_ecal) | `knowledgeBase/gebsort.md` |
| Snapshot PV / save+restore settings | `knowledgeBase/snapshot_pv.md` |
| DuoGe commissioning walkthrough (HV setup, IOC boot, trigger, DAQ) | `knowledgeBase/DGS_setup_guide.md` |

---

---

## DGS Commander EDM Screens

**Source:** https://wiki.anl.gov/gsdaq/DGS_Commander_EDM_Screens  
The EDM (Extensible Display Manager) GUI is the primary operator interface for Gammasphere run control. The main screen has 7 sections:

| Section | Purpose |
|---------|---------|
| **Run Control** | Start/Stop data acquisition; Save/NoSave toggle for data writing |
| **Main Controller** | Control/monitoring screens for waveforms, hardware, timing, trigger setup |
| **Main Controller Side Panel** | Extension of Main Controller (additional controls) |
| **VXI Heartbeat / Enabled Detectors** | **Obsolete** — leftover from pre-upgrade DAQ system; no longer functional |
| **Temperatures** | Per-detector temperature monitoring (HPGe cold chain health) |
| **LN Main** | LN2 system control and monitoring |
| **Setup Script State** | Indicator for scripts run via Main Controller |

Note: Much information is duplicated across screens by design — same data presented in different contexts.

---

## Shift Operator Guide (8-Hour Shift)

**Source:** https://wiki.anl.gov/gsdaq/User_Guides_for_Experiments

### Things to Watch During a Run

- **Run Control box** in DGS Main Controller (top right) must show **"Start; Save; Sort"** during a run — if not, data is not being recorded.
- **TCP/IP rates** (center grey boxes) should be changing; **Buffs Avail** ("Buffers" column in ANLDAQ GUI, PV `DAQCX_CV_BuffersAvail`) should stay near **200** ⚠️ Correction: wiki says 400, but pool was reduced from 400 → 200 on 2023-04-12 (JTA). ✅ verified 2026-04-26 — `DGS_DEFS.h:L48` (`#define RAW_Q_SIZE 200 //changed from 400 to 200 20230412 JTA`); `QueueManagement.c:L83` (qFree created with RAW_Q_SIZE); `DGS_DEFS.h:L113` (bypass threshold comment: "changed from 150/400 to scale with # bufs 20230412 JTA").
- Each VME terminal (one per IOC receiver) should update every 15 seconds showing Mb/sec written to disk.

### Start/Stop Runs During Shift

Recommended: start a new run approximately **every hour** (limits data loss if a run is corrupted).

```bash
cd /dgsdata
./stop_run.sh              # stop current run
# Wait 5-10 sec for all Buffs Avail to show 200 in Big Summary (pool = 200 since 2023-04-12)
./start_run.sh ###         # start new run (### = next run number)
```

Log in the logbook/elog: stop/start times, trigger rate, beam current.

Check disk capacity if advised:
```bash
df -h
```

### Quality Control — Histograms

DGS does **not** display live histograms for users. Users must merge, sort, and display in ROOT while data is being collected — see `/gsdaq/Analysis_codes` for details. Contact the person **ON CALL** if data appears nonsensical.

---

### Troubleshooting: VME Crash

**Signs:** TCP/IP rate drops to 0.0; Buffs Avail continuously fall below ~190 without recovering. (Old wiki threshold was 380/400; pool is now 200 — adjust proportionally.)

**Prevention:** Stop the run and start a new one before Buffs Avail reach 0. If they reach 0.0, the VME has crashed and needs a restart. ✅ verified 2026-04-26 — pool = 200 (`DGS_DEFS.h:L48`); Buffs Avail = `getFreeBufCount()` → PV `DAQCX_CV_BuffersAvail` (`outLoop.st:L97,L473`).

**Recovery procedure:**

1. **Stop the current run.**
2. In DGS Main Controller → **Terminals** → select the affected VME (IOC 1–11) → open terminal window.
3. Hit **Return** to get a prompt, then **Ctrl+X** to start the auto-reboot countdown.
   - Reboot takes >1 min; ignore non-fatal warning messages.
   - When prompt reappears, the IOC is back up.
   - If no auto-boot message or no prompt appears → hard reboot required (see below).
4. In DGS Main Controller → **Scripts** → run **"Lock All Setup"** (>1 min).
   - Verify Trigger Summary screen looks correct (accessible under **Trigger** in Main Controller and at the bottom of Big Summary).
5. Scripts → run **"Digitizer Setup"** (>1 min; issues EPICS channel PV commands for digitizer channels).
6. Buffs Avail should return to 200; start a new run.

### Troubleshooting: Power Cycling the DAQ (Hard Reboot)

Required when soft reboot fails.

**If remote PCU is operational:** use [Network Accessible Power Control Units of DGS](/gsdaq/Network_Accessible_Power_Control_Units_of_DGS).

**If physical access to Area 4 is required:**
- Check the monitor above the cage entrance for room status on ARIS 2.0.
- Faraday cup must be **IN** if the area is locked — contact operators at **2-4115** (ANL landline) for assistance.
- Follow the [Sweep Area 4](/gsdaq/Sweep_Area_4) procedure.

**Power cycle steps:**
1. Go to DGS racks (inside the shack or next to the hemisphere).
2. Turn off the crashed VME crate; wait **30 seconds** before turning power back on.
3. Leave the shack and Sweep Area 4.
4. Back in Data Room: reboot the **Trigger IOC** via DGS Main Controller → Terminals → Trigger IOC → Return → Ctrl+X.
5. Run **"Lock All Setup"** then **"Digitizer Setup"** as above.
6. If still failing, contact the person **ON CALL**.

---

*Created: 2026-04-05 from [wiki: Typical DGS Run Procedures](https://wiki.anl.gov/gsdaq/Typical_DGS_run_procedures)*
*Updated: 2026-04-16 — moved verification note outside code fence (formatting fix); RunParquet defaults verified 2026-04-14*
*Updated: 2026-04-20 — added DGS Commander EDM Screens section from wiki*
