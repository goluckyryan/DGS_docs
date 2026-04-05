# DGS Utility Scripts

Collection of utility/analysis scripts found in the DGS SVN and tools pack.

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

## Related Files
- `collectorbox_PVs.md` — full PV list including `GS{N}_SBX_Present`, BGO HV PVs
- `expMemory_2008_Chiara.md` — current experiment data locations
- `gammasphere_geometry.md` — GS hole numbering

---

*Source: `DGS_tools_pack/DGS_SVN/dgs/NS_scripts/`. Created: 2026-04-05.*
