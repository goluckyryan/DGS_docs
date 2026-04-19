# DGS Utility Scripts

Collection of utility/analysis scripts found in the DGS SVN and tools pack.

> ⚠️ **Legacy caveat:** Most scripts here come from the SVN archive (`DGS_SVN/`) — treat them as historical reference.
> - **PV names may be outdated** — the active system PV list is in `collectorbox_PVs.md`; always verify against live EPICS before using any PV from these scripts.
> - **Methods may no longer be needed** — some workflows (BGO tuning, PV extraction) may have been superseded by newer tooling or procedures.
> - **SVN = legacy** — active development is in Git repos (`ioc/`, `ANLDAQ/`, `collectorboxpi/`, etc.)

---

## NS_scripts (SVN: `DGS_SVN/dgs/NS_scripts/`)

Scripts for BGO HV tuning and PV monitoring. Authored by NS (Nishiki Sensharma based on wiki edit history).

### `BGO_tune_v2.py` — BGO HV Scan & Tuning

Scans BGO bias voltage from 0–250V in steps and measures BGO counter rates to find optimal HV operating point.

**HV scan range:** `[0, 25, 50, 75, 90, 100, 110, ..., 250]` V

**Key PVs used:**
- `GS{N}_BGO_HV0` through `GS{N}_BGO_HV13` — **14 BGO HV channels** total (HV0,2,4,6,8,10,12 = even; HV1,3,5,7,9,11,13 = odd) ✅ verified 2026-04-09 — `slopebox_scripts/BGO_Sweep_test` (caput GS000_BGO_HV0..HV13)
- `GS{N}_BGO1_counter` through `GS{N}_BGO7_counter` — BGO rate counters

**Operation:** For each HV step, sets HV via `caput`, waits 1 s, reads counter rates via `caget` (10 samples averaged), plots adjusted count rate vs HV.

**Dependencies:** Python, `matplotlib` (GTKAgg backend), `numpy`, EPICS `caput`/`caget` in PATH. ✅ verified 2026-04-09 — `NS_scripts/BGO_tune_v2.py:L5-7` (`matplotlib.use('GTKAgg')`, numpy import); HV sweep values: `[0,25,50,...,250]` (21 steps) ✅ verified 2026-04-09 — `BGO_tune_v2.py:L11`; HV PVs: `GS000_BGO_HV0`–`GS000_BGO_HV13` (7 even + 7 odd) ✅ verified 2026-04-09 — `BGO_tune_v2.py:L12-13`

**Tuning algorithm (detailed):**
1. Set BGO comparator threshold to 15 (noise rejection)
2. Verify preconditions: `GS000_Slopebox_Scan_control` == `read/write`, `GS000_SlopeBoxBGOInterlock` == `Closed`, `GS000_Conv_BGO400` ≥ 400 V, `GS000_Conv_BGO450` ≥ 450 V
3. Sweep odd HV tubes (BGO_HV1,3,5,7,9,11,13) while even are 0 — log average count rate per step
4. Sweep even HV tubes (BGO_HV0,2,4,6,8,10,12) while odd are 0 — log average count rate
5. Find lowest-gain tube (peak count rate across sweep) — excludes any BGO with <50% of max peak rate
6. Use `numpy.interp` to find HV → target count rate matching lowest-gain tube
7. Iterative readjustment: if residual (normalized deviation from target) >3%, proportionally adjust HV until converged
8. BGO backplug (BGO7/HV12,13): set separately to 10% of average of other 6 BGO tubes
9. Save target HVs to `Target_HV_GS000.txt` and plot final count rates + residuals

---

### `extract_PV.py` — Discover Active Ge Detectors + Generate camonitor Script

1. Polls `GS{N}_SBX_Present` for GS holes 1–110 to find active detectors
2. Builds a list of all digitizer channel PVs for active detectors
3. Generates a shell script to run `camonitor` on all relevant PVs

**PV format used:** `GS001_SBX_Present`, `GS010_SBX_Present`, `GS100_SBX_Present` (zero-padded to 3 digits) ✅ verified 2026-04-09 — `NS_scripts/extract_PV.py:L8-22` (loops GS_numbers 1–110, formats with `GS00{k}`, `GS0{k}`, `GS{k}`); generates `PV_list.sh` with `camonitor` of `VMExx:MDIGy:disc_count{ch}_RBV` for ch 5–9 ✅ verified 2026-04-09 — `extract_PV.py:L28-50`

---

### `extract_PV_post_processing.py` — Post-run PV Extraction

Reads detector configuration from `Pre_EPICS_Collector` scan files (not live EPICS), extracts enabled Ge detector list, and generates PV lists for post-run analysis.

---

### `GS_nums.py` — GS Hole Number Utilities

Helper module with GS hole numbering utilities used by other scripts.

---

### `PV_list.sh` / `PV_even_list.txt` / `PV_odd_list.txt`

Pre-computed PV lists for even/odd GS holes (south/north hemispheres).

### `average_values_even.txt` / `average_values_odd.txt`

Stored average BGO tuning values per hemisphere.

---

## Data0 Space Monitor (Cron)

A cron job that ran hourly checking `/mnt/data0` free space on DCS2. Previously hosted on pi5-dgs.

- **Threshold:** 300 GB free ⚠️ unverified — no script found in ANLDAQ, lnfill repos, DCS2 crontab (dcsu), or spark-ca9f crontab (2026-04-18 verified). Threshold likely from old pi5-dgs crontab no longer in version control. Ask Ryan to confirm.
- **Action:** Discord alert to #dgsclaw if below threshold
- **Current status (2026-04-05 09:00 CDT):** 396 GB free (78% used) — runs are accumulating
- **Migration status (checked 2026-04-18):** No crontab active on spark-ca9f (DGX Spark, current General DGS host) and not on DCS2. This cron job has **not** been migrated and appears to have no surviving source — ask Ryan if it should be set up on spark-ca9f.

---

## slopebox_scripts (SVN: `DGS_SVN/dgs/slopebox_scripts/`)

Legacy bash scripts for BGO counter averaging and sweeping:

- **`caget_avg`** — reads a PV N times (1 s apart), returns average. Usage: `./caget_avg <PV> <N_samples>` ✅ verified 2026-04-10 — `slopebox_scripts/caget_avg:L9-15` (loops N times, strips PV name at char 25, strips non-digits, accumulates)
- **`Avg_all_BGO_count`** — runs `caget_avg` on all 7 BGO counters + BGO sum for one detector
  - PVs: `GS000_BGO1_counter` … `GS000_BGO7_counter`, `GS000_BGOSum_counter`
- **`BGO_Sweep_test`** — BGO HV sweep: iterates DAC values 0–250 (in steps: 0,25,50,...250) across 14 HV channels (`GS000_BGO_HV0`–`GS000_BGO_HV13`), calls `Avg_all_BGO_count` at each step to log BGO counter rates. Sweeps odd tubes first, then even. PV pattern: `GS000_BGO_HV{0..13}`, `GS000_BGO{1..7}_counter`, `GS000_BGOSum_counter`. ✅ verified 2026-04-10 — `slopebox_scripts/BGO_Sweep_test:L28` (`for HVSET in 0 25 50 75 90 100 110 120 130 140 150 160 170 180 190 200 210 220 230 240 250`); odd tubes swept first (L30-37), then even (L50-57)
- **`BGO_counter_sweep.ods`** — spreadsheet for sweep analysis

---

## VXI_database (SVN: `DGS_SVN/dgs/VXI_database/`)

Legacy EPICS database files (`resm1.db` – `resm6.db`) from the pre-upgrade VXI system. Uses same `MOD{N}` PV prefix as the current system — confirms naming continuity across the upgrade. Historical reference only; replaced by SBX + Collector Box system.

---

## Modernization Notes

> ⚠️ The NS_scripts and slopebox_scripts use shell `caput`/`caget` subprocess calls — one process fork per PV read. This is slow and fragile.

**Recommended: rewrite in [pyepics](https://pyepics.github.io/pyepics/)**
- Persistent CA connections — no subprocess overhead
- `epics.caget(pv)` / `epics.caput(pv, val)` — cleaner syntax
- `epics.caget_many([pv1, pv2, ...])` — parallel batch reads
- Native async monitoring via `epics.PV(name, callback=fn)` (replaces `camonitor` shell scripts)
- For `BGO_tune_v2.py`: could read all 7 BGO counters in one batch call instead of 7 sequential subprocesses

*Note added 2026-04-05 per Ryan.*

---

---

## ANLDAQ GUI Helper Scripts (`ANLDAQ/gui/scripts/`)

Two scripts listed in `enableScriptList.txt` are selectable from the ANLDAQ GUI scripts combo box:

### `basic_settings_LED.py` — Apply Default LED Mode to All DIGs

Sets all digitizer channels to **LED (Leading-Edge Discriminator) mode** with hardcoded baseline settings. Intended as a quick "reset to known-good teststand defaults" before a run.

**Source:** `ANLDAQ/gui/scripts/basic_settings_LED.py` (78 lines) ✅ verified 2026-04-17 — file read directly

**Targets:** VME66, MDIG1+MDIG2, CH 5–9 (test stand config — **not** the full Gammasphere 440-ch setup)

**Key settings applied:**

| Parameter | Value | Meaning |
|---|---|---|
| `cfd_mode` | `LED_Mode` | Leading-edge discriminator (no CFD) |
| `led_threshold{ch}` | 300 | Fixed LED threshold |
| `trigger_polarity{ch}` | `RiseEdge` | Rising-edge trigger |
| `raw_data_delay{ch}` | 0.5 µs | Pre-trigger delay |
| `raw_data_length{ch}` | 0.32 µs | Waveform capture window |
| `p1_window{ch}` | 0.07 µs | Peaking time window 1 |
| `p2_window{ch}` | 0.05 µs | Peaking time window 2 |
| `m_window{ch}` | 2.5 µs | Main gate window |
| `k0_window{ch}` | 0.5 µs | Pre-gate k0 |
| `k_window{ch}` | 0.5 µs | Gate k |
| `d_window{ch}` | 0.16 µs | Delay window |
| `CS_Ena` | `Enable` | Enable coincidence sorting |
| `veto_enable` | 0 | Disable veto |
| `trigger_mux_select` | `IntAcptAll` | Accept all internal triggers |
| `Online_CS_StartStop` | `Stop` | Ensure run is stopped |
| `Online_CS_SaveData` | `No Save` | No data saved (safe default) |

**Operation:**
1. For each board: `master_logic_enable=Reset` → set all parameters → `master_fifo_reset=reset→run`
2. After all boards: set `Online_CS_StartStop=Stop` and `Online_CS_SaveData=No Save`
3. Uses `epics.caput(pv, val, wait=True, timeout=5.0)` — synchronous, waits for IOC acknowledgement

**Exception list for DIG count per VME:** VME06 and VME10 have only MDIG1 (not MDIG2) — matches the Gammasphere crate layout (2 shorter crates). All other VMEs default to MDIG1+MDIG2.

---

### `terminals` — Terminal Server / SoftIOC Spawner

A bash helper script that opens `gnome-terminal` windows connecting to VME IOC consoles or spawning the softIOC. Called by `commander.py` when the user selects "Open Terminal" in the GUI.

**Source:** `ANLDAQ/gui/scripts/terminals` (55 lines, no `.sh` extension) ✅ verified 2026-04-17 — file read directly

**Usage:** `terminals <arg>`

| Arg | Effect |
|---|---|
| `S` | Spawn a new SoftIOC in a gnome-terminal (checks if already running first via `ps ax \| grep SoftIOC`) |
| `1`–`N` | Open a gnome-terminal → `telnet $TERMINAL_SERVER $((2000 + N))` (e.g., `1` → port 2001) |

**Environment variables required:**
- `$TERMINAL_SERVER` — hostname/IP of the terminal server (set in `EPICS_para.sh`; for DGS: 192.168.203.186 / .91)
- `$ANLDAQ_DIR` — path to ANLDAQ repo root
- `$EPICS_HOST_ARCH` — EPICS architecture string (e.g. `linux-x86_64`)

**SoftIOC path:** `$ANLDAQ_DIR/EPICS/softIOC/iocBoot/iocdgsSoftIOC/dgsSoftIoc.cmd`

**Note:** This script runs the gnome-terminal windows locally (on the machine running the ANLDAQ GUI), not remotely. Telnet provides a serial console to the VxWorks IOC boot prompt over the terminal server's serial port.

---

### `enableScriptList.txt` — Enabled Script Registry

Two-line file listing the scripts that appear in the ANLDAQ GUI's "Enable Script" combo box:
```
basic_settings_LED.py
Serdes_Linkup.sh
```
`Serdes_Linkup.sh` is documented in `trig_setup_scripts.md`. The GUI reads this file to populate the dropdown; only listed scripts are selectable. ✅ verified 2026-04-17 — `enableScriptList.txt` read directly

---

*Source: `DGS_tools_pack/DGS_SVN/dgs/NS_scripts/`. Created: 2026-04-05. Updated: 2026-04-17 (cron migration status check). Updated: 2026-04-17 (added ANLDAQ GUI helper scripts: basic_settings_LED.py, terminals, enableScriptList.txt).*

## Cross-References

- `knowledgeBase/DGS_SVN.md` — SVN archive context; NS_scripts and slopebox_scripts source
- `knowledgeBase/collectorbox_PVs.md` — Current authoritative PV list (replaces outdated SVN PV references)
- `knowledgeBase/snapshot_pv.md` — Modern Python/pyepics PV snapshot utilities (supersedes these legacy scripts)
- `knowledgeBase/gammasphere_geometry.md` — GS hole → GS_ID numbering used by BGO tuning scripts
- `knowledgeBase/sbx.md` — Slope box hardware; BGO HV channels addressed by slopebox_scripts
- `knowledgeBase/trig_setup_scripts.md` — `Serdes_Linkup.sh` script (listed in `enableScriptList.txt`; documented there)
- `knowledgeBase/expMemory_2008_Chiara.md` — current experiment data locations
