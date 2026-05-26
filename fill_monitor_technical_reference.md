# Fill Monitor — Technical Reference

Complete algorithm documentation for the Gammasphere LN2 fill monitor system.
For high-level usage and integration, see [USER_GUIDE.md](USER_GUIDE.md).

## Contents

- [Architecture](#architecture)
- [Decision Flow](#decision-flow)
- [Adjustment Rules](#adjustment-rules)
- [Worked Examples](#worked-examples)
- [Fill Log Format](#fill-log-format)
- [Hose-GSID Tracking](#hose-gsid-tracking)
- [fill_results Data Structure](#fill_results-data-structure)
- [Weekly Log Format](#weekly-log-format)
- [State File](#state-file)
- [Configuration — gefilltime2.dat](#configuration--gefilltime2dat)
- [Tunable Parameters](#tunable-parameters)
- [Simulation vs Production](#simulation-vs-production)
- [Min-Time History CSV](#min-time-history-csv)
- [Plotting](#plotting)
- [Component Details](#component-details)
- [Dependencies](#dependencies)

## Architecture

The fill monitor is split into components that communicate through a shared
JSON state file (`log_mon/fill_monitor_state.json`):

```
fill log → fill_classifier → JSON → fill_adjuster → gefilltime2.dat
                                  ↓
                             fill_reporter → fillmon_YYYYMMDD.log
```

Each component owns non-overlapping keys in the JSON file and uses
read-modify-write to preserve other components' data. See
[Component Details](#component-details) for per-file documentation.

## Decision Flow

Each fill log is processed in a strict order. The classifier handles steps
1-6, the adjuster handles steps 7-12:

### 1. Per-Fill Validation (classifier)

1. **GSID validation**: GSIDs > 120 are counted as invalid. Duplicate GSIDs
   within the same fill are detected.
2. **Hose→GSID change detection**: Each hose label is checked against the
   previous mapping. Reassignments are logged.

### 2. Per-Detector Processing

For each detector in the fill log:

1. **Bad temperature logging** (classifier) — If `temp_begin` is invalid
   (None, ≤ 0K, or > 150K), logged to weekly section. Happens for ALL
   GSIDs including > 120.

2. **GSID filter** — GSIDs < 1 or > 120 are skipped for adjustment.

3. **Initialize defaults** (adjuster) — New GSIDs get DEFAULT_MIN_TIME (150)
   and DEFAULT_MAX_TIME (419).

4. **Compute base clamp** (adjuster) — `clamp_max = min(360, max_time × 0.9)`.

5. **High voltage logging** (classifier) — Overflow voltage > 5.85V at
   START, END, or BOTH. Note: START voltage is logged but NEVER used to
   reject a sensor reading — detectors often start high and cool during fill.

6. **OVERTIME logging** (classifier) — Status is OVERTIME.

7. **Cold time clamp tightening** (adjuster) — If cold_time ≥ 80s AND
   ovf_end < 5.85V AND cold_time + 30 < current min_time, the clamp
   condition is armed. After 12h of continuous eligibility, the clamp
   tightens to min(base_clamp, cold_time + 30). Any non-eligible fill
   resets the confirmation window.

8. **Temperature override check** (adjuster, using classifier's mean) —
   If temp_begin is valid and delta ≥ 1K from mean:
   - delta ≥ 3K → raw_adj = +30s
   - 1K ≤ delta < 3K → raw_adj = +10s to +30s (linear)
   - Compute budget_adj = min(raw_adj, remaining_budget)
   - Compute fill floor: floor_adj = max(0, fill_time + 10 - old_min)
   - temp_adjustment = max(budget_adj, floor_adj)

9. **Apply adjustment** (adjuster) — Exactly one path executes:
   - **Path A** — Temperature adjustment available → apply, charge budget
   - **Path B** — Temperature triggered, no adjustment needed → skip nominal
   - **Path C** — OVERTIME, no temp trigger → no adjustment
   - **Path D** — NORMAL/EXTENDED, no temp trigger → nominal rule

10. **Update temperature history** (classifier) — Append current fill to
    history AFTER the temp check, so current fill is excluded from its
    own mean.

11. **Record fill result** (adjuster) — Append to fill_results.

12. **Update weekly tracking** (adjuster) — min_time start/end/lo/hi.

## Adjustment Rules

### Effective Time

The algorithm compares an **effective time** against the target:

- **Valid cold_time**: Used directly when `cold_time ≥ 80% of (min + 20)`.
  In simulation, if cold_time < sim's adjusted min_time, fill_open is used.
- **Soft fallback**: cold_time failed 80% threshold but ≥ 80s AND ovf_end
  < 5.85V. Uses fill duration. Full decay formula and 2x eligibility.
- **Hard fallback**: cold_time < 80s, OR ovf_end ≥ 5.85V, OR cold_time
  is None/0. Uses fill duration. Limited to -1 decay, no 2x eligibility.

**START overflow voltage (ovf_begin) is NEVER used to reject a sensor
reading.** Only END overflow voltage causes hard fallback.

### Nominal Rule (±1/±2)

Applied to NORMAL and EXTENDED fills when no temperature override is active:

```
target = min + TARGET_OFFSET (20)

if effective > target:           +1  (or +2 if 2x up active)
elif effective < min + 10:       proportional: adj = floor((min+10 - eff)/10) * (-1) - 2
                                 (multiplied by 2x down_speed if active)
elif effective < target:         -1  (or -2 if 2x down active)
else:                            no change

Hard fallbacks: limited to -1 maximum downward adjustment.
```

### 2x Acceleration

**Downward:** After 4+ consecutive eligible fills (not hard fallback) with
effective_time < min + 10, all downward adjustments double. Hard fallbacks
break consecutivity. Counter resets on any temp event. During post-temp
holdoff, consec_below is frozen at 0.

**Upward:** After 8+ consecutive fills (ALL fills count, including hard
fallbacks) with effective_time > min + 30, the +1 doubles to +2. Upward
counter is NOT reset or frozen by temp events.

### Temperature Override

| Delta from mean | Adjustment |
|-----------------|------------|
| ≥ 3K | +30s |
| 1K – 3K | +10s to +30s (proportional) |
| < 1K | No override, use nominal rule |

**Fill-time floor:** min_time raised to at least fill_time + 10. Can exceed
budget cap. Constraints: not OVERTIME, 12h cooldown, 18h recency, no hose
change.

**Budget:** 30s capacity, regenerates 5s per 12h (72h full recovery). Floor
excess is charged; budget saturates at 0. If min_time < CLAMP_ABSOLUTE_MIN
(75), the portion up to the floor is free.

**When triggered (even if budget-blocked):** fill floor can still increase
min_time. Nominal rule skipped, consec_below resets.

**Temperature mean** (classifier-exclusive): lesser of Method A (up to 10
fills, ≤ 1K range, ≤ 18h gaps) and Method B (consecutive fills ≤ 18h apart).
Current fill excluded. Requires ≥ 2 qualifying fills.

**Invalid temperatures:** None, ≤ 0K, > 150K — excluded entirely.

### OVERTIME

Only temperature override applies. No nominal adjustment.

### Clamp

```
min ≤ min(360, max_fill_time × 0.9, cold_time + 30)
```

The cold_time + 30 clamp requires: cold_time ≥ 80s, ovf_end < 5.85V,
cold_time + 30 < current min_time, AND 12h continuous eligibility.

### Post-Temperature Holdoff

After any temp event, downward adjustments limited to -1 per 12h for
5 days (120h). consec_below frozen at 0. consec_above unaffected.

### Reconfiguration Detection

Gap > 48h between fills for the same detector, or hose→GSID swap, resets
all per-detector state (consecutive counters, holdoffs, clamp, history).

## Worked Examples

### Example 1: Normal Fill — Increase

Detector with `min_time = 200`, `max_time = 419`, NORMAL fill.

```
cold_time  = 230
target     = min + 20 = 220
threshold  = target × 0.8 = 176

cold_time (230) ≥ threshold (176) → valid, effective_time = 230
effective (230) > target (220) → adjustment = +1
new min_time = 201
```

### Example 2: Short Fill — Soft Fallback with Proportional Decay

Detector with `min_time = 200`, EXTENDED fill, LED went cold early.

```
cold_time  = 150
threshold  = (200 + 20) × 0.8 = 176

cold_time (150) < threshold (176) → fallback triggered
cold_time (150) ≥ HARD_FALLBACK_FLOOR (80) → not hard
ovf_end = 3.2V < 5.85V → sensor valid at end
→ SOFT FALLBACK: effective_time = open_time = 200

effective (200) < min+10 (210) → proportional decay zone
gap = 210 - 200 = 10
adj = floor(10/10) × (-1) - 2 = -3
new min_time = 197  (or 194 if 2x down active)
```

### Example 3: Temperature Override

Detector with `min_time = 180`, mean temp = 90.0K, this fill temp = 95.0K,
fill_time = 195s. Budget has 30s remaining.

```
delta = 95.0 - 90.0 = 5.0K → raw_adj = +30s
budget_adj = min(30, 30) = 30
fill floor = 195 + 10 = 205 → floor_adj = 205 - 180 = 25
temp_adjustment = max(30, 25) = 30  (budget wins)

new min_time = min(180 + 30, clamp_max) = 210
Budget charged 30s → 0s remaining. Full recovery in 72h.
```

### Example 3b: Fill Floor Exceeds Budget

Same detector, budget = 5s, fill_time = 220s.

```
budget_adj = min(30, 5) = 5
fill floor = 220 + 10 = 230 → floor_adj = 230 - 180 = 50
temp_adjustment = max(5, 50) = 50  (floor wins, bypasses budget cap)

new min_time = 230. Budget charged 50s → saturates at 0.
```

### Example 4: Hard Fallback — Limited Decay

Detector with `min_time = 250`, overflow sensor stuck open.

```
cold_time = 3 → HARD FALLBACK (< 80s)
effective_time = open_time = 250

Would normally get proportional decay, BUT hard fallback → limited to -1
new min_time = 249
consec_below reset to 0. No 2x eligibility.
```

### Example 5: Overtime with Temperature Override

Detector with `min_time = 300`, `max_time = 419`, OVERTIME. Temp delta = 4K.

```
OVERTIME → no nominal adjustment
delta = 4.0K ≥ 3.0K → raw_adj = +30s
clamp_max = min(360, 419 × 0.9) = 360
new min_time = min(300 + 30, 360) = 330
```

## Fill Log Format

Produced by `LNFill_App.py`. Two header formats:

```
LNFill_App.py F fill initiated on 2026-05-12 at 06:00:07     (new format)
Initiating Fill from LNFill_All.py on 2026-01-29 at 16:29:18  (old format)
```

Fill types: F (full), M (temperature-monitored), L (list), T (tank-only).

```
                       OPEN         TEMP         OVERFLOW               COLD
    VALVE     GSID     TIME     BEGIN  END    BEGIN   END     STATUS    TIME
DET: A- 1      73      210      88.1  88.4     1.87  5.65    NORMAL      210
DET: A- 2      95      153      90.5  90.5     1.89  5.78    EXTENDED     96
DET: D- 3      52      421      91.7  91.6     1.80  2.98    OVERTIME    419
```

## Hose-GSID Tracking

Each physical hose (e.g., "A- 1") maps to a detector GSID. The classifier
tracks this mapping across fills. When a hose switches detectors, all
per-detector state is reset (consecutive counters, holdoffs, clamp, history).

## fill_results Data Structure

Per-detector per-fill records accumulated by the adjuster:

| Field | Type | Description |
|-------|------|-------------|
| gsid | int | Detector ID |
| hose | str | Valve/hose label |
| timestamp | datetime | Fill timestamp (Chicago timezone) |
| open_time | int | Total seconds valve was open |
| fill_open | int | Sim fill duration (= open_time in production) |
| cold_time | int | Seconds until LED went cold |
| effective_time | int | Time used by algorithm |
| is_fallback | bool | cold_time rejected, fill duration used |
| is_hard_fallback | bool | Hard fallback (limited to -1, no 2x) |
| status | str | NORMAL, EXTENDED, or OVERTIME |
| min_before | int | min_time before adjustment |
| min_after | int | min_time after adjustment |

## Weekly Log Format

`log_mon/fillmon_YYYYMMDD.log` — rolls over Monday 00:00 Central.

### Sections

1. **High (Open) LED Voltage (>5.85V)** — Per-detector START/END/BOTH counts.
   START high = dewar may have been full. END high = sensor didn't detect fill.
   BOTH = sensor open throughout. Chronic BOTH suggests sensor failure.

2. **OVERTIME** — Each timeout with timestamp, GSID, open_time, ovf_end.
   Low ovf_end (2-3V) = genuine incomplete fill. High ovf_end (>5V) = sensor.

3. **Invalid/Extreme Temperature** — None, ≤ 0K, > 150K. Latest per GSID.
   Persistent entries suggest hardware fault.

4. **Invalid/Duplicate GSIDs** — Out-of-range (>120) with counts. Duplicates
   within single fill with timestamp.

5. **Temperature Override Adjustments** — Sorted by GSID. With adjuster:
   shows old→new min_time, adjustment, delta. Without adjuster: shows
   temp delta from mean only.

6. **Clamped Min-Times** (adjuster only) — GSIDs hitting the clamp limit.

7. **Hose-GSID Mapping Changes** — Hose, old→new GSID with timestamp.

8. **Min-Time Summary** (adjuster only) — Start/end/min/max/range/delta
   per detector for the week.

### Sample Output

```
Fill Monitor Weekly Log — Week of 2026-05-05
==============================================================================

--- High (Open) LED Voltage Events (>5.85V) ---
  Total events: 14  (START=3, END=8, BOTH=3)
   Hose    GSID   START     END    BOTH   Total
   A- 5      18       1       2       0       3
   B-12      42       0       3       1       4

--- OVERTIME Detectors ---
  [2026-05-06 06:01]  D- 3  GSID  52  open_time=421s  ovf_end=2.98

--- Temperature Override Adjustments ---
  [2026-05-05 18:01]  A- 1  GSID  18  min: 195 → 225  temp_override (+30s, delta=3.2K)

--- Min-Time Summary ---
   Hose    GSID   Start     End     Min     Max   Range   Delta
   A- 1      18     195     228     195     228      33     +33
   D- 3      52     275     310     275     310      35     +35

  (104 detectors unchanged)
```

## State File

`log_mon/fill_monitor_state.json` — shared by all components.

### Classifier-owned keys

| Key | Description |
|-----|-------------|
| history | Per-detector fill history (up to 20 entries) for temp mean |
| hose_gsid_map | Current hose→GSID mapping |
| last_fill_time | Per-GSID timestamp of most recent fill |
| current_week_start | Monday 00:00 of current log week |
| last_classified | Last classified filename (idempotency) |

### Adjuster-owned keys

| Key | Description |
|-----|-------------|
| temp_adj | Temperature budget entries [{ts, amount}] |
| consec_below | Consecutive fills below min+10 counter |
| consec_above | Consecutive fills above min+30 counter |
| last_temp_trigger | Timestamp of last temp event |
| last_decay_time | Timestamp of last holdoff decay |
| last_fill_floor | Timestamp of last fill-floor fire |
| clamp_first_seen | Timestamp of first clamp-eligible fill |

### weekly_log_sections (split ownership)

| Sub-key | Owner |
|---------|-------|
| high_voltage, overtime, bad_temp, invalid_gsids, duplicate_gsids, hose_changes, gsid_hose | Classifier |
| temp_overrides | Both (classifier writes partial, adjuster enriches) |
| clamped, min_time_start, min_time_end, min_time_lo, min_time_hi | Adjuster |

## Configuration — gefilltime2.dat

```
# Ge Fill times (GS ID / min time / max time)
1,139,419
42,201,419
```

CSV: GSID, min_time (seconds), max_time (seconds). Read and written
exclusively by the adjuster.

**Missing file recovery:** If gefilltime2.dat does not exist, the adjuster
recovers last known min_times from min_time_history archive CSVs. Falls back
to DEFAULT_MIN_TIME (150) for GSIDs not in the archive.

## Tunable Parameters

All defined as class constants on `FillAdjuster`:

| Parameter | Default | Description |
|-----------|---------|-------------|
| TARGET_OFFSET | 20 | Target fill time = min + this |
| PROP_DECAY_THRESHOLD | 10 | Below min + this → proportional decay |
| UP_ACCEL_THRESHOLD | 30 | Above min + this → counts toward 2x up |
| COLD_REJECT_FRACTION | 0.8 | Cold time < target × this → fallback |
| HARD_FALLBACK_FLOOR | 80 | Cold time < this → always hard fallback |
| OVF_VOLTAGE_THRESHOLD | 5.85 | Overflow voltage above this = sensor open |
| CLAMP_ABSOLUTE_MAX | 360 | Absolute max min_time (seconds) |
| CLAMP_ABSOLUTE_MIN | 75 | Absolute min min_time (seconds) |
| CLAMP_MAX_TIME_FRACTION | 0.9 | Clamp at max_fill_time × this |
| CLAMP_COLD_TIME_MARGIN | 30 | Clamp at cold_time + this |
| CLAMP_CONFIRM_HOURS | 12 | Continuous eligible period for cold clamp |
| TEMP_OVERRIDE_DELTA_HIGH | 3.0 | delta ≥ this → max adjustment |
| TEMP_OVERRIDE_DELTA_LOW | 1.0 | delta ≥ this → proportional adjustment |
| TEMP_OVERRIDE_MAX_ADJ | 30 | Max single temp adjustment (seconds) |
| TEMP_OVERRIDE_MIN_ADJ | 10 | Min temp adjustment at DELTA_LOW |
| TEMP_OVERRIDE_FILL_MARGIN | 10 | Floor: min_time ≥ fill_time + this |
| TEMP_ADJ_BUDGET_CAP | 30 | Max budget capacity (seconds) |
| TEMP_ADJ_REGEN_RATE | 5 | Budget regen seconds per interval |
| TEMP_ADJ_REGEN_INTERVAL | 12 | Hours per regen step |
| TEMP_MEAN_MAX_GAP_HOURS | 18 | Max gap between fills for mean calc |
| TEMP_MEAN_MAX_ENTRIES | 10 | Max fills for method (a) of mean calc |
| TEMP_MEAN_MIN_ENTRIES | 2 | Min entries for valid mean |
| TEMP_MEAN_RANGE_TOLERANCE | 1.0 | Max temp range for method (a) (K) |
| CONSEC_BELOW_THRESHOLD | 4 | Consecutive below for 2x down |
| CONSEC_ABOVE_THRESHOLD | 8 | Consecutive above for 2x up |
| TEMP_DECAY_LIMIT_HOURS | 120 | Holdoff duration (5 days) |
| TEMP_DECAY_LIMIT_INTERVAL | 12 | Max one -1 per this many hours |
| RECONFIG_GAP_HOURS | 48 | Gap triggering state reset |
| DEFAULT_MIN_TIME | 150 | Default for missing GSIDs |
| DEFAULT_MAX_TIME | 419 | Default max fill time |

## Simulation vs Production

### Production Mode

The adjuster processes a single fill log and writes updated min_times back
to gefilltime2.dat. State persists in the JSON file between cron invocations.
Weekly log sections accumulate across fills within the same week.

```bash
python3 -m fill_monitor adjust --logfile logs/fill_20260512_0600.log
```

### Simulation Replay (via plotter)

Accessed through `fill_monitor plot` (default mode). Replays the full adjuster
through all historical logs — one fresh classifier+adjuster per fill, state
via JSON each cycle, exactly like production. Sim output goes to a temp
directory (auto-deleted unless `--keep-sim`).

```bash
python3 -m fill_monitor plot --all --outdir plots/ \
    --logdir logs/ --filltimes gefilltime2.dat --keep-sim
```

In simulation, fill duration = `max(min_time, cold_time) + rand(0,3)`.
Bad cold_times (< 80s with valid ovf_end) are proxied with open_time.

## Min-Time History CSV

`log_mon/min_time_history/min_time_YYYYMM.csv` — monthly, written by adjuster.

```csv
timestamp,gsid,hose,min_time,fallback
2026-05-13 06:00,42,D- 7,201,
2026-05-13 06:00,56,C-24,221,hard
```

Used by plotter in archive mode and by adjuster for recovery when
gefilltime2.dat is missing.

## Plotting

### Archive Mode (--archive, recommended for production)

Reads min_time history from CSVs + fill data from logs. No simulation.

```bash
python3 -m fill_monitor plot --archive --all --outdir plots/ --logdir logs/
```

### Simulation Replay (default)

Replays adjuster through all logs. Requires starting gefilltime2.dat.

```bash
python3 -m fill_monitor plot --all --outdir plots/ \
    --logdir logs/ --filltimes gefilltime2.dat
```

### Plot Legend

- **Green dots**: Effective time (NORMAL fills)
- **Orange dots**: Effective time (EXTENDED fills)
- **Red dots**: Effective time (OVERTIME fills)
- **Hollow circles**: Historical fill time (sim mode only, where sim deviated)
- **Red X**: Hard fallback (sensor invalid)
- **Purple X**: Soft fallback (cold_time below 80% threshold)
- **Blue line**: Min time
- **Dashed blue line**: Target (min + 20)

## Component Details

### fill_classifier.py
Parses fill logs, classifies events, computes temperature mean (exclusive).
Owns: history, hose_gsid_map, last_fill_time, classifier wls sub-keys.
Does NOT touch gefilltime2.dat. Not sim-aware.

### fill_adjuster.py
All adjustment logic, tuning parameters, state management.
Owns: temp_adj, consec_*, last_temp_trigger, last_decay_time, last_fill_floor,
clamp_first_seen, adjuster wls sub-keys. Owns gefilltime2.dat.
Sim-aware (fill_open computation, cold_time proxy).

### fill_reporter.py
Formats weekly log from JSON. Never writes to JSON (except weekly reset).
Two modes: classifier-only (anomaly sections) vs full (all 8 sections).

### fill_plotter.py
Per-detector plots. Two modes: archive (CSV) vs simulation replay.

### fill_adjuster_sim.py
Library for simulation replay. Drives classifier→adjuster→reporter per log.
Used internally by the plotter's simulation mode.

### add-press subcommand (fill_add_press.py, V2.5)
Tank pressure management during fills. Counteracts pressure drop by opening
the spare tank's external supply fill valve to inject nitrogen from the
external supply line into the spare tank. Because the supply line never
fully chills down between fills, what flows in is predominantly gas rather
than liquid — but this is sufficient to maintain tank pressure while the
main tanks are being depleted by the detector fill.
Uses pyepics via `pv_cache`/`pvlock`.

Invoked as `python3 -m fill_monitor add-press`, runs concurrently with
`LNFill_App.py` during fills. Only writes to `LNT3_FV:EN` (TS1 spare) and
`LNT6_FV:EN` (TS2 spare).

**Control logic** (`AddPressController`):
- Opens spare tank fill valve when ext−ts pressure ≥ `VALVE_OPEN_PRESS` (3 psi)
- Closes when differential ≤ `VALVE_CLOSE_PRESS` (−1 psi) or tank ≥ `MAX_TANK_PRESS` (32 psi)
- Holdoff timers prevent valve cycling: 120s after max-pressure close, 60s after differential close
- Cross-station balancing: station closes spare if its pressure exceeds the other by `PRESS_DIFF_OFF_THRESH` (only when both spares open)
- Sleep ramp: 0→30s between iterations, resets to 0 on valve open
- Done when: runtime ≥ 2200s, or all manifold valves closed after 240s minimum

**Pressure gauge cascade** (bash-identical):
- Each station tries Tank1 → Tank2 → Tank3 gauges with calibration offsets
- Failed gauges (`PRESS_FAIL=1`) are skipped; if all fail, defaults to 28 psi (≤400s) or 20 psi (>400s)
- External pressure: Ext1 → Ext2 fallback → default 28 psi

**Pressure logging** (V2.4+):
- Every iteration snapshots all 8 gauges to `influx_txt/AddPress.txt` (InfluxDB line protocol)
- Known-failed sensors skipped (no wasted caget timeout); invalid readings logged as `[WARN]` once
- File bulk-pushed to InfluxDB via `push_fill.sh` at end of run

**Warning deduplication** (V2.5):
- Initial gauge check at startup reports all bad sensors upfront
- Subsequent warnings for same device+reason suppressed with "(further warnings suppressed)" note

**Verification** (`verify` subcommand):
- Parses recorded AddPress.log, feeds each line through `AddPressController.step()`
- Compares predicted valve states against actual logged states
- Known limitation: manifold valve state must be inferred from pressure transitions (no ln_log access), causing expected mismatches at manifold boundaries

### check-press subcommand (fill_check_pressure.py, V1.0)
Hourly pressure gauge snapshot. Reads all 8 tank and supply pressure
gauges, validates each reading (numeric, in range), writes to InfluxDB
line protocol, and pushes via push_fill.sh. Replaces check_pressure.sh.

Gauges:
- Ext1/Ext2 (LNP1-01_PR:AP, LNP2-01_PR:AP): external supply, range -5 to 90 psi
- Tank1-Tank6 (LNP1-02..LNP2-04_PR:AP): tank pressures, range -5 to 45 psi

Output: `influx_txt/check_pressure.txt` (overwritten each run).
`--no-influxdb` flag writes the file but skips the InfluxDB push.

Cron:
```
07 * * * * python3 -m fill_monitor check-press >> cron_logs/check_pressure.log 2>&1
```

## Discord Alert Integration (2026-05-20)

The reporter dispatches Discord alerts via two functions in
`WriteDiscordMessage.py` (repo root, shared with LNFill_App.py):

- `WriteDiscordMessage(msg)` — #system-messages channel (`discord.WebHook`)
- `WriteAnomalyMessage(msg)` — #anomaly channel (`discord_anomaly.WebHook`)

Both auto-prefix the hostname (e.g., `[pi5-lnFill]`).

### Per-fill alerts (`discord_per_fill`)

Called by the reporter's `report()` after writing the weekly log file.
The reporter is the natural home for per-fill alerts because it runs
after every fill (both full and M-fills) and has access to the complete
`weekly_log_sections` including any adjuster-enriched data (temp override
details, clamp info). Reads the current `weekly_log_sections` and builds
batched messages:

- **W1 OVERTIME** — from `overtime` list. Overflow <5.6V adds "fill status uncertain."
- **W2 Invalid/Duplicate GSIDs** — from `invalid_gsids` and `duplicate_gsids`. Includes hose labels.
- **W3 No temp readback** — from `bad_temp` where temp is None.
- **W4 Abnormal temp** — from `bad_temp` entries with non-None temperature
  (all are outside the valid 60–150K range by definition).
- **N1 Hose changes** — from `hose_changes` list.
- **N2 Temp adjustments** — from `temp_overrides` (adjuster-enriched entries only).

Warnings (W1–W4) are batched into one anomaly-channel message per fill.
Notices (N1–N2) are batched into one operational-channel message per fill.

### Weekly alerts (`discord_weekly`)

Called by the classifier's `_ensure_week()` on Monday rollover, BEFORE
resetting `weekly_log_sections`. Receives the previous week's accumulated data.

The weekly summary is sent from the classifier rather than the reporter
because the classifier owns the weekly rollover logic (`_ensure_week()`)
and is the only component that knows when a new week has started. On
rollover, the classifier must clear `weekly_log_sections` to start the
new week — so the Discord summary must fire before that reset. The
reporter cannot do this because it runs after the classifier and would
only see the already-cleared data. The `discord_weekly()` method itself
lives on the `FillReporter` class (keeping all Discord formatting in one
place), but it is called by the classifier at rollover time.

- **W5 Chronic high voltage** — overflow LED sensors with ≥12 events out of
  ~14 fills/week. Indicates a failed or intermittent LED sensor or wiring
  that needs repair.
- **N3 Big movers** — |min_time delta| > 60s. Sent to #system-messages channel.
- **W6 Clamp ceiling** — min_time reached upper clamp limit (360s or max_time×0.9).

### Low tank pressure alerts (add-press)

During fills, the add-press subcommand monitors tank station pressures and fires
alerts when thresholds are crossed. Each threshold fires once per station
per fill (no repeat alerts for sustained low pressure).

- **20 psi** — info (N-level, #system-messages channel)
- **18, 15, 12, 9, 6, 3 psi** — warning (W-level, #anomaly channel)

Pressure checks only run while manifold valves are open (fill in progress).
Thresholds use the same integer-truncated, calibrated pressure values as
the valve control logic.

All threshold crossings are logged to `weekly_log_sections.low_pressure`
in the fill monitor JSON state file. The classifier clears this key on
weekly rollover. The reporter formats them as section 8 ("Low Tank
Pressure Events") in the weekly log.

### Warm manifold detection (add-press)

Monitors manifold temperature sensors (`LNM1A_TM:BT` through `LNM4A_TM:BT`)
each iteration. A manifold reads 'Cold' when LN2 is flowing through it and
'Warm' when it isn't — a Cold→Warm transition during a fill may indicate
the feeding tank is empty.

Alert conditions (once per manifold per fill):
- Was Cold for ≥2 min (`COLD_QUALIFY_TIME`), then sustained Warm for ≥20s
  (`WARM_TRIGGER_SHORT`)
- Sustained Warm for ≥180s (`WARM_TRIGGER_LONG`) after manifold fill valve
  opened, regardless of prior Cold duration

At alert time, reads spare feed valve PVs (`LNM{n}A_FV:EN`) to determine
if the spare tank (Tank 3 or 6) is also feeding the manifold. Reports all
active tanks in the alert message.

While any manifold is in Warm state, the sleep interval is capped at 5s
(`FAST_POLL_INTERVAL`) instead of the normal 0→30s ramp. Fast polling
continues until the manifold goes Cold.

Events logged to `weekly_log_sections.warm_manifold` in the JSON state
file. The reporter formats them as section 9 ("Warm Manifold Events") in
the weekly log.

PV reads added (all read-only, not in original AddPress.sh):
- `LNM1A_TM:BT` .. `LNM4A_TM:BT` — temp sensors (every iteration)
- `LNM1A_FV:EN` .. `LNM4A_FV:EN` — spare feed valves (at alert time only)

### Suppression

`--no-discord` flag on `adjust`, `report`, or `add-press` subcommand
suppresses all Discord messages. The `--test` flag on `add-press` implies
`--no-discord` and `--no-influxdb`.

## Dependencies

- Python 3.9+ (uses `zoneinfo`)
- matplotlib (for plotting only)
- add-press live mode requires pyepics + pv_cache/pvlock (shared with LNFill_App.py).
  The verify subcommand does not require pyepics (log parsing only).
