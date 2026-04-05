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
- `GS{N}_BGO_HV0` through `GS{N}_BGO_HV12` — 7 BGO HV channels (even/odd split)
- `GS{N}_BGO1_counter` through `GS{N}_BGO7_counter` — BGO rate counters

**Operation:** For each HV step, sets HV via `caput`, waits 1 s, reads counter rates via `caget` (10 samples averaged), plots adjusted count rate vs HV.

**Dependencies:** Python, `matplotlib` (GTKAgg backend), `numpy`, EPICS `caput`/`caget` in PATH.

---

### `extract_PV.py` — Discover Active Ge Detectors + Generate camonitor Script

1. Polls `GS{N}_SBX_Present` for GS holes 1–110 to find active detectors
2. Builds a list of all digitizer channel PVs for active detectors
3. Generates a shell script to run `camonitor` on all relevant PVs

**PV format used:** `GS001_SBX_Present`, `GS010_SBX_Present`, `GS100_SBX_Present` (zero-padded to 3 digits)

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

A cron job runs hourly on pi5-dgs checking `/mnt/data0` free space on DCS2.

- **Threshold:** 300 GB free
- **Action:** Discord alert to #dgsclaw if below threshold
- **Current status (2026-04-05 09:00 CDT):** 396 GB free (78% used) — runs are accumulating

---

## slopebox_scripts (SVN: `DGS_SVN/dgs/slopebox_scripts/`)

Legacy bash scripts for BGO counter averaging and sweeping:

- **`caget_avg`** — reads a PV N times (1 s apart), returns average. Usage: `./caget_avg <PV> <N_samples>`
- **`Avg_all_BGO_count`** — runs `caget_avg` on all 7 BGO counters + BGO sum for one detector
  - PVs: `GS000_BGO1_counter` … `GS000_BGO7_counter`, `GS000_BGOSum_counter`
- **`BGO_Sweep_test`** — BGO sweep test script
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

## Related Files
- `collectorbox_PVs.md` — full PV list including `GS{N}_SBX_Present`, BGO HV PVs
- `expMemory_2008_Chiara.md` — current experiment data locations
- `gammasphere_geometry.md` — GS hole numbering

---

*Source: `DGS_tools_pack/DGS_SVN/dgs/NS_scripts/`. Created: 2026-04-05.*
