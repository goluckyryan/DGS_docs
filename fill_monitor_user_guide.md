# Fill Monitor — User Guide

## Contents

- [Overview](#overview)
- [Components](#components)
- [Production Usage](#production-usage)
  - [After Each Fill (cron)](#after-each-fill-cron)
  - [Pre-Fill Temperature Check](#pre-fill-temperature-check-scheduled-fills-only)
  - [Cron Integration](#cron-integration)
  - [Anomaly-Only Monitoring](#anomaly-only-monitoring-no-adjustment)
- [Weekly Monitoring Log](#weekly-monitoring-log)
- [File Dependencies](#file-dependencies)
- [Commands](#commands)
  - [adjust](#adjust)
  - [report](#report)
  - [add-press](#add-press)
  - [monitor-tanks](#monitor-tanks)
  - [check-press](#check-press)
  - [pre-fill-adjust-check](#pre-fill-adjust-check)
  - [flush-history](#flush-history)
  - [plot](#plot)
- [Command Flag Reference](#command-flag-reference)
- [Discord Alerts](#discord-alerts)
  - [Infrastructure & IOC Health](#infrastructure--ioc-health)
  - [Pre-Fill Checks](#pre-fill-checks)
  - [Fill Monitoring Alerts](#fill-monitoring-alerts)
  - [Post-Fill Analysis](#post-fill-analysis)
  - [Weekly Summary](#weekly-summary-monday-rollover)
  - [Suppression](#suppression)
  - [Alert Reference](#alert-reference)
- [Configuration](#configuration)

## Overview

The fill monitor automatically adjusts per-detector LN2 minimum fill times
and detects fill anomalies. After each fill cycle, it processes the fill log,
adjusts `gefilltime2.dat`, and updates the weekly monitoring log.

### LED Voltage Polarity (important)

LED overflow sensor voltages run **inverse** to what intuition might
suggest:

- **Low voltage (~1.8 V)** = WARM (no LN2 at the sensor)
- **High voltage (~5.0–5.7 V)** = COLD (LN2 has reached the sensor)
- **> 5.86 V** = OPEN / sensor fault
- **< 1.5 V** = SHORT / sensor fault

This matters when reading Discord alerts, weekly log entries, and
fill records.  An alert that says "vent valve WARM (3.50V)" means
the vent valve LED is reading **low voltage** because no cold LN2
has reached it yet — not that the voltage itself is "warm."

Full technical detail: see
[TECHNICAL_REFERENCE.md § LED Sensor Infrastructure](TECHNICAL_REFERENCE.md#led-sensor-infrastructure).

## Components

| Script | Purpose | Inputs | Outputs |
|--------|---------|--------|---------|
| `adjust` | Process fill log: detect anomalies, adjust min fill times, update weekly log | Fill log, gefilltime2.dat | Updated gefilltime2.dat, JSON state, weekly log |
| `report` | Process fill log: detect anomalies, update weekly log (no adjustment) | Fill log | JSON state, weekly log |
| `plot` | Plot per-detector fill history (simulation, testing, diagnostics) | Fill logs or archive CSVs | PNG plots |
| `add-press` | Tank pressure management during F-fills (includes full monitoring stack: warm detection, failover, low pressure alerts) | EPICS PVs (live) | influx_txt/AddPress.txt, InfluxDB |
| `monitor-tanks` | Manifold monitoring during auto-fills (warm detection, failover, no pressure control) | EPICS PVs (live) | influx_txt/MonitorTanks.txt, InfluxDB |
| `check-press` | Adaptive-rate pressure gauge snapshot | EPICS PVs (live) | influx_txt/check_pressure.txt, InfluxDB |
| `pre-fill-adjust-check` | Pre-fill hose reassignment + temperature safety bump | EPICS PVs (live), JSON state | Updated gefilltime2.dat, JSON state |
| `flush-history` | Reset tracking history or set fill time | GSID or hose | Updated JSON state, gefilltime2.dat (--min-time-only) |

## Channel pre-warm (cold-start race fix)

Long-running commands (`add-press`, `monitor-tanks`) and the LED logger
thread auto-pre-warm their EPICS channels on startup so the first iteration
of their main loop reads from pyepics' warm monitor cache (sub-millisecond)
instead of paying the cold-channel-open cost.

You don't need to do anything — the pre-warm is automatic.  Operationally,
you will see:

- `add-press` startup takes ≈500 ms longer than the per-iteration cost
  (one-time, before the first pressure read).
- `monitor-tanks` same as above.
- `check-press` and `pre-fill-adjust-check` are unaffected (they use
  `caget_many` batch reads, which are already parallel one-shots).

If the pre-warm fails (IOC unreachable, network partition, etc.), each
affected command writes a single `[WARN] *-pre-warm failed to connect`
line to stderr listing how many PVs failed.  The command continues —
subsequent per-PV cagets hit their own timeouts naturally and the same
failure-handling code paths that existed before this fix take over.

Background: on 2026-06-01 the 06:00 cron-fired add-press tripped CP-1's
5-strike abort because its 10 sequential single-PV cagets in
`read_valve_states()` raced libca's cold-channel-open pipeline and 5
landed above the 0.5 s caget timeout.  Pre-warm eliminates that race.
See TECHNICAL_REFERENCE.md § "Channel pre-warm" for the architectural
details.

## Production Usage

### After Each Fill (cron)

The adjuster is the primary entry point. It internally runs the classifier
first, then applies adjustments, then runs the reporter:

```bash
python3 -m fill_monitor adjust --logfile ${outfile1} --filltimes gefilltime2.dat
```

This single command:
1. Parses the fill log (classifier)
2. Classifies anomalies: open LED voltage, overtime, bad temps, etc. (classifier)
3. Adjusts min_times per detector (adjuster)
4. Updates gefilltime2.dat (adjuster)
5. Writes/updates the weekly monitoring log (reporter)

### Pre-Fill Temperature Check (scheduled fills only)

Before a scheduled fill, the pre-fill check reads live detector
temperatures and temporarily bumps min_time for any detector running
hotter than its historical average. This prevents underfilling when
temperatures are rising between scheduled fills.

The pre-fill uses a more aggressive formula than the post-fill adjuster:
2× proportional (20-60s vs 10-30s) plus a 30s flat boost, yielding
50-90s total adjustment vs the adjuster's 10-30s.  No cold_time clamp
is applied — the small amount of LN2 wasted by overshooting during a
scheduled fill pales in comparison to the amount needed to cool down
the entire delivery system for an extra mid-day auto fill to service
one underfilled detector.  The bump is temporary; the adjuster restores
the original baseline after the fill.

```bash
python3 -m fill_monitor pre-fill-adjust-check
```

The bump is temporary — the post-fill adjuster restores the original
baseline and applies the full adjustment (with budget tracking, floor
calculation, and Discord reporting). Auto fills do NOT run the pre-fill
check; the post-fill adjuster handles them directly.

### Cron Integration

In `LNFill_cron.sh` (full fills):
```bash
python3 -m fill_monitor pre-fill-adjust-check   # temp safety bump (before fill)
python3 -m fill_monitor add-press --parent-pid $$ &  # tank pressure management
${dir}/LNFill_App.py F ${tstamp}                 # fill runs with bumped min_times
...
python3 -m fill_monitor adjust --logfile ${outfile1}  # full post-fill adjustment
```

The `adjust` call reads the original min_times from the JSON backup
created by the pre-fill check, so all adjustments are computed from
the correct baseline.

In `LNFill_cron.sh` (post-fill, unchanged):
```bash
python3 -m fill_monitor adjust --logfile ${outfile1} --filltimes gefilltime2.dat
```

In `LNFill_Auto_EFill_cron.sh` (temperature-monitored fills):
```bash
if [ -f "$outfile1" ]; then
    python3 -m fill_monitor adjust --logfile ${outfile1} --filltimes gefilltime2.dat
fi
```

### Anomaly-Only Monitoring (no adjustment)

To classify and report without adjusting min_times:
```bash
python3 -m fill_monitor report --logfile ${outfile1}
```

The weekly log will contain anomaly sections only (no min-time summary
or clamp sections).

## Weekly Monitoring Log

The weekly log (`logs/fill_monitor/fillmon_YYYYMMDD.log`) rolls over on Monday 00:00
and contains these sections:

1. **LED Voltage Fault Events** — detector overflow LED fault events.  Eight fault types: `START_HIGH` and `END_HIGH` (>5.86V at fill start/end — open-circuit / sensor failed open), `START_LOW` and `END_LOW` (<1.6V at fill start/end — short-circuit / sensor failed low), `START_SUSPECT` (>4.0V at fill start but ≤5.86V — detector not warming between fills), `HIGH`, `LOW`, `SUSPECT` (same thresholds, fired by mid-fill continuous polling from the live LED monitor).  The live monitor runs by default during `add-press` and `monitor-tanks`; pass `--no-led-check` to disable it (the log-based fallback then runs in the classifier as before).
2. **OVERTIME Detectors** — fills that reached max_time
3. **Invalid/Extreme Temperature Readbacks** — bad sensor readings (shows start and end temp; marks `(recovered)` if end temp is valid)
4. **Invalid/Duplicate GSIDs** — configuration errors
5. **Temperature Override Adjustments** — warm detector events
6. **Clamped Min-Times** — adjustments limited by clamp (adjuster only)
7. **Detector Configuration Changes** — new detectors, removed detectors, moved detectors
8. **Low Tank Pressure Events** — from add-press, pressure drops below thresholds
9. **Warm Manifold Events** — from add-press, manifold temp went Warm during fill
10. **Min-Time Summary** — per-detector changes for the week (adjuster only)

## File Dependencies

```
fill log → classifier → JSON state → adjuster → gefilltime2.dat
                                   ↓                    ↓
                              reporter → fillmon_YYYYMMDD.log
                                   ↓
                              plotter → PNG plots
```

## Commands

### adjust

The primary entry point after each fill. Internally runs the classifier
(parses fill log, detects anomalies), then the adjuster (adjusts
per-detector min_times), then the reporter (writes weekly log, sends
Discord alerts).

```bash
# Standard usage (called by LNFill_cron.sh and LNFill_Auto_EFill_cron.sh)
python3 -m fill_monitor adjust --logfile logs/fill_20260522_1800.log

# Specify custom gefilltime2.dat path
python3 -m fill_monitor adjust --logfile ${outfile1} --filltimes gefilltime2.dat

# Use non-standard state directory
python3 -m fill_monitor adjust --logfile ${outfile1} --log-mon-dir /alt/path

# Suppress Discord alerts
python3 -m fill_monitor adjust --logfile ${outfile1} --no-discord
```

What it does:
1. Parses the fill log — extracts per-detector fill times, temperatures,
   overflow voltages, status (NORMAL/EXTENDED/OVERTIME).
2. Classifies anomalies — open LED voltage, overtime, bad temps, invalid
   GSIDs, duplicate GSIDs, hose-GSID mapping changes.
3. Computes temperature mean and delta for each detector. If delta ≥ 1.0K,
   applies a temperature override adjustment to min_time (with budget
   tracking and fill-floor safety minimum).
4. Applies nominal adjustment rule — nudges min_time up or down based
   on effective fill time relative to target (min_time + 20s). Includes
   4-consecutive acceleration (2x speed) and post-temperature holdoff.
5. Writes updated min_times to gefilltime2.dat.
6. Updates weekly monitoring log.
7. Sends per-fill Discord alerts (warnings and notices).
8. If a pre-fill-adjust-check ran before this fill, restores the original
   min_times from `prefill_backup` in the JSON state as the adjustment
   baseline, then clears the backup.

Idempotent: re-running with the same log file skips state updates
(classifier checks `last_classified` filename).

### report

Classifies a fill log and writes the weekly monitoring log without
adjusting min_times. Useful for monitoring fills on a system where
automatic adjustment is not desired.

```bash
python3 -m fill_monitor report --logfile logs/fill_20260522_1800.log

# Suppress Discord
python3 -m fill_monitor report --logfile ${outfile1} --no-discord
```

The weekly log will contain anomaly sections (high voltage, overtime,
bad temps, invalid GSIDs, hose changes, temperature overrides) but
no min-time summary or clamp sections.

### add-press

During each fill cycle, detectors draw liquid nitrogen from the local
tanks, causing tank pressure to drop. The `add-press` subcommand runs
concurrently with `LNFill_App.py` to:

1. **Maintain tank pressure** by opening an ext fill valve to inject
   nitrogen from the external supply into an unused tank. The valve is
   adaptively selected — always targeting a tank NOT currently feeding
   any manifold.
2. **Monitor manifold temperatures** for empty tank detection (Warm/Cold).
3. **Perform automatic failover** when a tank empties mid-fill — closes
   the current feed and opens the alternate feed to keep LN2 flowing.
4. **Log pressures** to InfluxDB and send Discord alerts.

#### Operating Modes

- **Normal** (default): Full pressure management + warm detection + failover.
- **--no-failover**: Pressure management active. Warm detection fires
  alerts but does not switch feeds. Ext fill deconfliction still active.
- For monitoring without pressure control, see the
  [monitor-tanks](#monitor-tanks) command.

#### Usage

```bash
# Production (called by LNFill_cron.sh in background)
python3 -m fill_monitor add-press --parent-pid $$ >> cron_logs/AddPress.log 2>&1 &

# Disable manifold failover (warm detection still fires alerts)
python3 -m fill_monitor add-press --no-failover

# Test without pushing to InfluxDB
python3 -m fill_monitor add-press --no-influxdb

# Full test mode (no InfluxDB, no Discord, log mirrored to test file)
python3 -m fill_monitor add-press --test

# Verify logic against a recorded AddPress.log
python3 -m fill_monitor add-press verify cron_logs/AddPress.log
python3 -m fill_monitor add-press verify cron_logs/AddPress.log --verbose
```

#### Termination

Exits on the first condition met:

- **Manifolds closed:** all manifold feed valves (main AND spare)
  read not-Open, after MIN_RUN_TIME (45s).  Normal exit — fill
  complete, no further pressure management needed.
- **Parent gone** (with `--parent-pid`): parent process exited.
  Normal if manifolds are also closed.  Red alert if manifolds
  are still open (anomalous — cron script crashed or was killed).
- **Hard timeout:** 2200s safety net.  Info message to
  #anomaly if manifolds closed (exit condition issue).
  Red alert to #anomaly if manifolds still open.
- **Valve PV timeout:** 5 consecutive valve read failures =
  yellow warning + exit.

#### Cron Integration

In `LNFill_cron.sh` (full fills):
```bash
python3 -m fill_monitor add-press --parent-pid $$ >> cron_logs/AddPress.log 2>&1 &
```

### monitor-tanks

Runs alongside `LNFill_App.py` during auto-fills (M fills) to provide
manifold monitoring and failover without pressure control.  Replaces
the old `add-press --no-pressure-mgmt` mode.

Does NOT control ext fill valves — only monitors manifold temperatures,
detects empty tanks, and performs automatic feed failover (main→spare
or spare→main).

#### Three-Phase Startup

1. **Phase 1** — Check parent PID alive (with `--parent-pid`) or skip.
2. **Phase 2a** — Wait for manifold valves to open (2s poll, 45s timeout).
   No EPICS pressure reads, no logging.  Exits silently if parent dies
   or timeout expires before any valve opens.
3. **Phase 2b** — Active monitoring.  Pressure logging starts here
   (first InfluxDB data point).  30s poll interval, 2s during warm
   manifold detection.

**No fill = no log file, no InfluxDB data.**  The log file and pressure
readings only begin when a manifold valve is actually detected open.

#### Termination

Exits on the first condition met:

- **Manifolds closed:** all manifold feed valves (main AND spare)
  read not-Open.  Normal exit — fill complete.
- **Parent gone** (with `--parent-pid`): parent process exited.
  Normal if manifolds closed.  Red alert if manifolds still open.
- **Without `--parent-pid`:** manifolds closed = exit
  (Phase 2a already waited for them to open).
- **Hard timeout:** safety net (default 2200s, override with
  `--hard-timeout`).  Red alert if manifolds still open.
- **Valve PV timeout:** 5 consecutive valve read failures = yellow
  warning + exit.

#### Usage

```bash
# Production (called by LNFill_Auto_EFill_cron.sh in background)
python3 -m fill_monitor monitor-tanks --parent-pid $$ --hard-timeout 1500 >> cron_logs/MonitorTanks.log 2>&1 &

# Initial deployment (Discord suppressed until verified working)
python3 -m fill_monitor monitor-tanks --no-discord --parent-pid $$ --hard-timeout 1500 >> cron_logs/MonitorTanks.log 2>&1 &

# Without --parent-pid (fallback: waits 45s for valves, exits when closed)
python3 -m fill_monitor monitor-tanks >> cron_logs/MonitorTanks.log 2>&1 &

# Test mode
python3 -m fill_monitor monitor-tanks --test
```

#### Cron Integration

In `LNFill_Auto_EFill_cron.sh` (auto fills):
```bash
python3 -m fill_monitor monitor-tanks --parent-pid $$ --hard-timeout 1500 >> cron_logs/MonitorTanks.log 2>&1 &
```

### check-press

Reads all 8 tank and supply pressure gauges, validates readings, writes
InfluxDB line protocol, and pushes to InfluxDB. Replaces `check_pressure.sh`.

Uses adaptive sampling rate coordinated with AddPress via the shared
JSON state file:
- **During fills**: holds off while AddPress is running
- **After fills**: samples every 5 minutes for 60 minutes (tank refill monitoring)
- **Normal**: samples once per hour at top of hour

Additionally, an **IOC health check** runs on every cron invocation
regardless of rate gating.  If all non-failed gauges return None
(timeout or error), a Discord anomaly [CP-1] fires to alert the
operator that the LN IOC or network may be down.

```bash
# Production (cron, every 5 min — script self-gates based on state)
# IOC health check runs every invocation; InfluxDB logging is gated
python3 -m fill_monitor check-press >> cron_logs/check_pressure.log 2>&1

# Write file but don't push to InfluxDB
python3 -m fill_monitor check-press --no-influxdb

# Force a sample regardless of schedule
python3 -m fill_monitor check-press --force

# Suppress Discord alerts (IOC health check still runs, just silent)
python3 -m fill_monitor check-press --no-discord
```

### pre-fill-adjust-check

Before each scheduled fill, performs two types of pre-fill adjustment:

1. **Hose reassignment** [N5] — For new or moved detectors, sets
   min_time from the destination hose's history (`hose_min_time`).
   Detected by reading live EPICS manifold assignment PVs
   (`LNH{m}-{v}_SM:SUB.E`) and comparing against `known_detectors`.

2. **Temperature bump** [PF-1] — For existing unchanged detectors,
   reads live temperatures and bumps min_time if warm (≥1.0K above
   mean). Uses a 2× proportional + 30s flat formula (50-90s) — more
   aggressive than the post-fill adjuster's 10-30s.  Overshooting is
   acceptable: the LN2 cost pales vs cooling the delivery system for
   an extra mid-day auto fill.

Each detector gets one action or the other, not both.  New/moved
detectors get hose reassignment (skip temp bump — temperature history
is from the old hose).  Also adds missing GSIDs to gefilltime2.dat
[GF-1] and aborts when a single batched LN IOC read returns
>= MAX_CONSECUTIVE_PV_FAILURES (5) `None` values [PF-2].
Collector IOC batch failures (DV_EN, DV_TEMP) do not abort —
affected detectors are skipped and a one-time notice fires [PF-3].

All adjustments are temporary — the post-fill adjuster restores the
original baseline from `prefill_backup` and applies the full
adjustment.

Auto fills do NOT run the pre-fill check.

```bash
# Production (called by LNFill_cron.sh before the fill)
python3 -m fill_monitor pre-fill-adjust-check

# Suppress Discord
python3 -m fill_monitor pre-fill-adjust-check --no-discord
```

### flush-history

Two modes:

**Default (flush):** Clears all tracking history for a detector from
the JSON state file.  Use when a detector's history has been
invalidated by a known change (dewar swap, sensor replacement, hose
reconfiguration) or polluted by a sensor glitch.  The detector starts
fresh — no mean temperature available until enough new fills rebuild
history (minimum 3 entries).  Does NOT modify gefilltime2.dat.

**`--min-time-only N`:** Sets the detector's fill time in
gefilltime2.dat to N seconds and updates the hose fill time record.
Does NOT clear tracking history or adjuster state.

Accepts either a numeric GSID (1–110) or a hose label (A-1, B14, etc.).

```bash
# Flush tracking history (does NOT change fill times)
python3 -m fill_monitor flush-history 95
python3 -m fill_monitor flush-history A-2

# Set fill time only (does NOT clear history)
python3 -m fill_monitor flush-history 95 --min-time-only 200
python3 -m fill_monitor flush-history B-14 --min-time-only 180
```

### plot

#### Archive Mode (recommended for production)

Reads min_time history directly from the monthly CSV logs. No simulation
replay needed — uses the actual min_time values computed in production.

```bash
python3 -m fill_monitor plot --archive --all --outdir plots/ \
    --logdir logs/ --log-mon-dir logs/fill_monitor
```

Requires:
- Fill logs in `--logdir` (for cold_time, open_time, status)
- Min_time history CSVs in `--log-mon-dir/min_time_history/`

#### Simulation Replay Mode (default)

Replays the full adjuster through all historical logs to generate
fill_results for plotting. Useful for evaluating algorithm changes
or when archive CSVs don't exist.

```bash
python3 -m fill_monitor plot --all --outdir plots/ \
    --logdir logs/ --filltimes gefilltime2.dat
```

Simulation output goes to a temporary directory (auto-deleted after
plotting). Use `--keep-sim` to retain it.

```bash
python3 -m fill_monitor plot --all --outdir plots/ --keep-sim \
    --logdir logs/ --filltimes gefilltime2.dat
```

#### Plot Specific Detectors

```bash
python3 -m fill_monitor plot 42 26 18 --logdir logs/ --filltimes gefilltime2.dat
python3 -m fill_monitor plot --archive 42 26 18 --logdir logs/
```

## Command Flag Reference

| Flag | Commands | Description |
|------|----------|-------------|
| `--logfile FILE` | adjust, report | Fill log file to process (required) |
| `--filltimes FILE` | adjust, plot, pre-fill-adjust-check, flush-history | Path to gefilltime2.dat |
| `--log-mon-dir DIR` | adjust, report, add-press, monitor-tanks, plot, pre-fill-adjust-check, flush-history | Log monitor directory (for state file) |
| `--state-file FILE` | adjust, report, pre-fill-adjust-check, flush-history | State JSON path (default: log-mon-dir/fill_monitor_state.json) |
| `--no-discord` | adjust, report, add-press, monitor-tanks, pre-fill-adjust-check, check-press | Suppress Discord alert messages |
| `--no-influxdb` | add-press, monitor-tanks, check-press | Suppress InfluxDB push at end of run |
| `--no-missing-log-alert` | adjust, report | Suppress RED alert when log file is missing or empty [ML-1]. Use in AUTO cron where most runs produce no log. |
| `--no-failover` | add-press, monitor-tanks | Disable manifold feed switching (warm detection, ext fill deconfliction, and Discord alerts still active) |
| `--no-feedline-check` | add-press, monitor-tanks | Disable broken feed line detection (BL-1 + BL-2).  Skips all detector valve PV reads and vent-LED reads.  Use during maintenance or known empty tank replacement. |
| `--parent-pid PID` | add-press, monitor-tanks | PID of calling cron script for termination detection (usage: `--parent-pid $$`) |
| `--test` | add-press, monitor-tanks | Test mode: suppress InfluxDB + Discord, log to test file |
| `--log-led` | add-press, monitor-tanks | DIAGNOSTIC ONLY — not for routine operation. Enable 1Hz LED voltage logging to CSV. Captures overflow LED voltage curves for all 112 detector LEDs during fill. ~12MB per 30-min fill. |
| `--log-full` | add-press, monitor-tanks | DIAGNOSTIC ONLY — superset of `--log-led`.  Also logs 14 extra LEDs (4 manifold-vent, 4 TS-manifold, 6 tank-vent) and emits valve transition event rows for 136 valves (detector + manifold + tank).  ~14MB per 30-min fill plus a few KB of event rows. Required input for BL-2 priming analysis. |
| `--no-led-check` | add-press, monitor-tanks | Disable the live LED fault detector.  The CSV logger (`--log-led`/`--log-full`) is independent and still runs if its flag is set.  Use only for diagnostic runs where you want pure CSV output without classifier side effects — in production the live detector should always run. |
| `--hard-timeout SEC` | monitor-tanks | Override hard timeout in seconds (default: 2200). Safety net — exits after this many seconds regardless of state. |
| `--force` | check-press | Bypass all gating — sample immediately |
| `--archive` | plot | Use archived min_time history CSVs (no simulation) |
| `--all` | plot | Plot all detectors |
| `GSIDs` | plot | Specific GSID numbers to plot |
| `--outdir DIR` | plot | Output directory for plots |
| `--logdir DIR` | plot | Fill log directory |
| `--keep-sim` | plot | Retain simulation output after plotting |
| `GSID-or-hose` | flush-history | Detector GSID (1-110) or hose label (A-1, B14, etc.) |
| `--min-time-only N` | flush-history | Set fill time to N seconds (no history flush) |
| `LOGFILE` | add-press verify | AddPress.log file to verify against (required) |
| `--manifold FILE` | add-press verify | Manifold transition log (ln_log format) for exact verification |
| `--verbose` | add-press verify | Print every line during verification |

## Discord Alerts

The fill monitor sends Discord alerts from multiple components.  Two
Discord channels are used:

- **#anomaly** (`discord_anomaly.WebHook`) — Warnings that need attention
- **#system-messages** (`discord.WebHook`) — Informational notices

All Discord output can be suppressed per-component with `--no-discord`.
The `--test` flag also suppresses Discord.

**Alert IDs** (e.g. W1, FO-2, PV-1) are unique identifiers for each
Discord message.  The same IDs appear as code comments in the source
files — grep for `[FO-2]` to find the code that generates a specific
alert.

### Local Discord Message Log

Every outgoing Discord message is also appended to a local log file
at `cron_logs/discord_log.txt` before the network send is attempted.
This gives an audit trail / debugging aid that is independent of
Discord's history retention or connectivity.

**Format:** one line per send, three space-separated fields
(channel field padded to 11 columns for column-alignment when
reading with `column -t`):

```
{timestamp}  {channel}      {exact message as sent to discord}
```

Examples:

```
Sun May 31 08:31:08 AM CDT 2026  anomaly      🔴 GS073 (A- 1): DV_EN=1 but valve is 'Disable'. Detector will NOT be filled.
Sun May 31 08:31:09 AM CDT 2026  operational  ℹ️ Manifold A vent 'WARM' (4.50V, open 300s) at start of fill
```

Multi-line messages are collapsed: embedded newlines become the
literal two-character sequence `\n` so each send is exactly one
line (greppable and tailable).

**Suppressed alerts** (`--no-discord`, `--no-feedline-check`,
inert mode, etc.) do NOT appear here.  The log hook lives inside
the `send_discord_*` functions in `fill_interfaces.py`, so only
messages that actually reach the send call get logged.  The log
records what the system tried to send to Discord, not hypothetical
messages that were filtered upstream.

**Failure behaviour:** if the log write fails (disk full,
permission denied, etc.), the Discord send still proceeds.  A
one-time stderr warning is emitted per process; later failures
stay silent to avoid log spam.

**Rotation:** monthly, via `archive_cron_log.sh`.  Rotated copies
land in `cron_logs/archive/discord_log_YYYYMM.txt`.

### Infrastructure & IOC Health

These alerts fire when EPICS IOC or network connectivity is lost.
They can occur at any time — before, during, or between fills.

| ID | Severity | Channel | Component | Example message |
|----|----------|---------|-----------|-----------------|
| [PV-1/PV-2](#pv-1--pv-2--valve-pv-read-failures) | ⚠️ warn | #anomaly | add-press, monitor-tanks | `⚠️ LN IOC PV reads failed 5 consecutive times. IOC or network may be down.` |
| [CP-1](#cp-1--ioc-health-check-failure) | ⚠️ warn | #anomaly | check-press | `⚠️ LN IOC PV reads failed — all gauge reads timed out. IOC or network may be down.` |
| [PF-2](#pf-2--pre-fill-ln-ioc-pv-read-failure-abort) | 🔴 red | #anomaly | pre-fill | `🔴 LN IOC PV reads: 224 failed in one batch — IOC may be down. Aborting prefill.` |
| [PF-3](#pf-3--pre-fill-collector-ioc-pv-failure-notice) | ⚠️ warn | #anomaly | pre-fill | `⚠️ Collector IOC PV reads: 12 failed in one batch — IOC may be down. Skipping matrix classification for affected detectors.` |

- **PV-1/PV-2:** Fired by monitor-tanks (PV-1) or add-press (PV-2)
  when valve state reads fail 5 consecutive times during a fill.
  The process aborts — it cannot safely operate without valve state.
- **CP-1:** Fired by check-press every 5 minutes (regardless of fill
  state) when all pressure gauge reads fail. Detects IOC outages
  between fills.
- **PF-2:** Fired by pre-fill-adjust-check when LN IOC PVs (manifold
  assignment SM:SUB.E or valve state FV:EN) fail. Pre-fill check aborts.
- **PF-3:** Fired by pre-fill-adjust-check when Collector IOC PVs
  (DV_EN or DV_TEMP) fail. Does NOT abort — affected detectors are
  skipped, processing continues.

### Pre-Fill Checks

Fired by **pre-fill-adjust-check** before scheduled F-fills.  Two
distinct categories, each with its own subsection below:

- **Pre-Fill Operational Checks** — messages produced by the fill
  adjustment + IOC health logic itself (temperature bumps, file
  recovery, IOC failures).
- **Pre-Fill Detector Configuration Checks** — per-detector
  classifier verdicts (PFM matrix) and per-detector hose-mapping
  sanity check (VS-4).

#### Pre-Fill Operational Checks

Messages from the adjustment pipeline and IOC-health monitoring —
these fire regardless of detector configuration state.

| ID | Severity | Channel | Component | Example message |
|----|----------|---------|-----------|-----------------|
| [PF-1](#pf-1--pre-fill-temperature-bump) | ℹ️ info | #anomaly | pre-fill | `ℹ️ GS042 (D- 7) temp +3.2K above avg \| min time adj: 190 → 220s (last fill time 234s)` |
| [PF-2](#pf-2--pre-fill-ln-ioc-pv-read-failure-abort) | 🔴 red | #anomaly | pre-fill | `🔴 LN IOC PV reads: 224 failed in one batch — IOC may be down. Aborting prefill.` |
| [PF-3](#pf-3--pre-fill-collector-ioc-pv-failure-notice) | ⚠️ warn | #anomaly | pre-fill | `⚠️ Collector IOC PV reads: 12 failed in one batch — IOC may be down. Skipping matrix classification for affected detectors.` |
| [GF-1](#gf-1--missing-gsid-in-gefilltime2dat) | ℹ️ info | #anomaly | adjust, pre-fill | `ℹ️ GS042 missing from gefilltime2.dat — added with min fill time 200s (from hose A-1 history)` |
| [GF-2](#gf-2--missing-gefilltime2dat) | ⚠️ warn / ℹ️ info | #anomaly | pre-fill, adjust | `⚠️ Pre-fill check: gefilltime2.dat not found` / `ℹ️ gefilltime2.dat was missing — recovered from archive` |

Pre-fill bumps (PF-1) are temporary — the post-fill adjuster restores
the baseline and applies the full correction with budget tracking.

#### Pre-Fill Detector Configuration Checks

Per-detector classifier that validates the (DV_EN, hose, temp, valve)
state of every detector against the 12-row matrix.  See the
[detector state matrix](#prefill-check--detector-state-matrix)
subsection further down for the full row-by-row breakdown.  These
do NOT change fill behaviour — they surface configuration issues
the operator should resolve.  VS-4 is the hose-side-mapping check
and fires independently of the matrix.

| ID | Severity | Channel | Component | Example message |
|----|----------|---------|-----------|-----------------|
| [PFM-1](#prefill-check--detector-state-matrix) | 🔴 red | #anomaly | pre-fill | `🔴 GS073 (A- 1): cold detector with valve Disabled — detector will not be filled. Temp=85.0K, DV_EN=1.` |
| [PFM-2](#prefill-check--detector-state-matrix) | ⚠️ warn | #anomaly | pre-fill | `⚠️ GS073 (A- 1): warm detector will be filled. Temp=290.0K, valve=Auto, DV_EN=1.` |
| [PFM-3](#prefill-check--detector-state-matrix) | ⚠️ warn | #anomaly | pre-fill | `⚠️ GS073 (A- 1): cold detector, not monitored. Detector will be filled but is unmonitored. Temp=85.0K, DV_EN=0, valve=Auto.` |
| [PFM-4](#prefill-check--detector-state-matrix) | ℹ️ info | #anomaly | pre-fill | `ℹ️ GS073 (A- 1): warm detector, monitored, will not be filled. Temp=290.0K, DV_EN=1, valve=Disable.` |
| [PFM-5](#prefill-check--detector-state-matrix) | ℹ️ info | #system-messages | pre-fill | `ℹ️ GS073 (A- 1): warm detector in array, will not be filled. DV_EN=0, valve=Disable, Temp=290.0K.` |
| [PFM-OPEN](#prefill-check--detector-state-matrix) | ⚠️ warn | #anomaly | pre-fill | `⚠️ [PFM-OPEN] Manifold A: prefill check ran while 3 valves are Open (positions 1, 5, 12) — possible concurrent M-fill. Per-detector states may be transient; review after current fill completes.` |
| [VS-4](#vs-4--wrong-side-hose-assignment) | ⚠️ warn | #anomaly | pre-fill | `⚠️ GS073: assigned to hose C- 5 (Manifold C, served by TS2) but detector is physically on TS1. Check hose mapping.` |

### Fill Monitoring Alerts

Fired by **add-press** (during F-fills) or **monitor-tanks** (during
M-fills) while running alongside `LNFill_App.py`.  Both components
share the same monitoring classes (`ManifoldTempTracker`,
`LowPressureTracker`, `ManifoldFailover`, `BrokenLineDetector`)
from `fill_tank_monitor.py`.

#### Safety

| ID | Severity | Channel | Component(s) | Example message |
|----|----------|---------|--------------|-----------------|
| [BL-1](#bl-1--broken-manifold-feed-line) | 🔴 red | #anomaly | add-press, monitor-tanks | `🔴 BROKEN FEED LINE DETECTED: Manifold A — 4 of 6 recent detectors with no LN2 flow. Feed valve closed. Requires immediate attention, manifold cannot fill.` |
| [BL-2](#broken-feed-line-at-end-of-priming--bl-2-red--bl-3-yellow) | 🔴 red | #anomaly | add-press, monitor-tanks | `🔴 [BL-2 IN DEVELOPMENT — NO ACTION TAKEN] BROKEN FEED LINE DETECTED: Manifold A — vent LED 1.85V at vent close (<2.0V). No LN2 reaching manifold. When activated, this would close the main feed valve.` (currently in inert mode; see BL-2/BL-3 detail section) |
| [BL-3](#broken-feed-line-at-end-of-priming--bl-2-red--bl-3-yellow) | ⚠️ warn | #anomaly | add-press, monitor-tanks | `⚠️ Manifold A not primed before start of detector filling — vent LED 2.45V at vent close (<3.0V). Filling performance will be degraded.` |

BL-1 is **active**: a confirmed broken-feedline verdict closes the
manifold's main feed valve (`LNM{x}_FV:EN → Auto`) and sends the
RED Discord notice.  Use `--no-feedline-check` on add-press /
monitor-tanks as the kill switch — it disables every BL detector
PV read and all alerts, no code change required.

#### Low Tank Pressure

Each threshold fires once per station per fill.  Pressure checks only
run while manifold feed valves are open (main or spare).

| ID | Severity | Channel | Component(s) | Example message |
|----|----------|---------|--------------|-----------------|
| [N-LP](#n-lp--low-tank-pressure-20-psi-info) | ℹ️ info | #anomaly | add-press, monitor-tanks | `ℹ️ Tank pressure notice: TS1 at 19.5 psi (below 20 psi)` |
| [W-LP](#w-lp--low-tank-pressure-warning) | ⚠️ warn | #anomaly | add-press, monitor-tanks | `⚠️ Low tank pressure: TS2 at 14.8 psi (below 15 psi threshold)` |

#### Empty Tank Detection & Failover

| ID | Severity | Channel | Component(s) | Example message |
|----|----------|---------|--------------|-----------------|
| [W-WM](#w-wm--warm-manifold-detected) | ⚠️ warn | #anomaly | add-press, monitor-tanks | `⚠️ Empty tank detected: Manifold A (Tank 1) — manifold warm, possible empty tank. Pressure: 15.2 psi` |
| [FO-2](#fo-2--failover-main--spare) | 🔴 red | #anomaly | add-press, monitor-tanks | `🔴 Manifold A tank empty — switched to spare feed (Tank 3). Pressure mgmt switched to Tank 1 ext fill.` |
| [FO-4](#fo-4--failover-spare--main) | 🔴 red | #anomaly | add-press, monitor-tanks | `🔴 Manifold A spare tank empty — switched to main feed (Tank 1). Pressure mgmt switched to Tank 3 ext fill.` |
| [FO-5](#fo-5--both-feeds-exhausted) | 🔴 red | #anomaly | add-press, monitor-tanks | `🔴 Manifold A: Tank 3 also empty — feed closed. Detectors on this manifold will not receive LN2.` |
| [FO-6](#fo-6--failover-confirmed) | ✅ info | #anomaly | add-press, monitor-tanks | `✅ Manifold A — new feed confirmed (temp readback Cold)` |
| [FO-1](#fo-1--failover-spare-feed-disabled) | 🔴 red | #anomaly | add-press, monitor-tanks | `🔴 Manifold A tank empty — spare feed valve DISABLED, cannot failover. Manual intervention required.` |
| [FO-3](#fo-3--failover-main-feed-disabled-spare-first-fill) | 🔴 red | #anomaly | add-press, monitor-tanks | `🔴 Manifold A spare tank empty — main feed valve DISABLED, cannot failover. Manual intervention required.` |

Warm manifold alert conditions (once per manifold per fill):
- Case 1: Was Cold ≥2 min, then sustained Warm ≥10s
- Case 2: Sustained Warm ≥180s after fill valve opened

#### Ext Fill Valve Adaptation (add-press only)

| ID | Severity | Channel | Component | Example message |
|----|----------|---------|-----------|-----------------|
| [AD-1](#ad-1-through-ad-5--ext-fill-valve-adaptation) | ℹ️ info | #anomaly | add-press | `ℹ️ TS1: ext fill tank (Tank 3) now in use as manifold B feed — pressure mgmt switched to Tank 2 ext fill.` |
| [AD-3](#ad-1-through-ad-5--ext-fill-valve-adaptation) | ⚠️ warn | #anomaly | add-press | `⚠️ TS1: Tank 3 ext fill valve DISABLED — cannot manage pressure using this tank.` |
| [AD-4](#ad-1-through-ad-5--ext-fill-valve-adaptation) | ⚠️ warn | #anomaly | add-press | `⚠️ TS1: no available ext fill valve — pressure mgmt unavailable.` |
| [AD-5](#ad-1-through-ad-5--ext-fill-valve-adaptation) | ⚠️ warn | #anomaly | add-press | `⚠️ TS1: spare ext fill valve (Tank 3) DISABLED — pressure mgmt switched to Tank 1 ext fill.` |

#### Process Lifecycle

| ID | Severity | Channel | Component(s) | Example message |
|----|----------|---------|--------------|-----------------|
| [EX-1](#ex-1--monitor-tanks-hard-timeout-manifolds-open) | 🔴 red | #anomaly | monitor-tanks | `🔴 Monitor-tanks hard timeout after 2200s with manifold valve(s) still open (A). Tank status uncertain. Manual check required.` |
| [EX-2](#ex-2--parent-process-terminated-manifolds-open) | 🔴 red | #anomaly | monitor-tanks | `🔴 Parent process terminated while manifold valve(s) still open (B (spare)). Tank status uncertain. Manual check required.` |
| [EX-3](#ex-3--addpress-exiting-manifolds-open) | 🔴 red | #anomaly | add-press | `🔴 AddPress exiting with manifold valve(s) still open (A, C). Tank status uncertain. Manual check required.` |
| [I-TO](#i-to--non-critical-timeout) | ℹ️ info | #anomaly | add-press | `ℹ️ add-press timed out (2200s) — manifolds were closed. Non-critical, but indicates exit condition did not trigger properly.` |

### Post-Fill Analysis

Fired after each fill completes.  The Component column shows which
command(s) can produce each alert: **adjust** (full pipeline) or
**report** (classifier + reporter, no adjustment).

Per-fill warnings are sent as a single batched message to #anomaly with
a header: `Fill (YYYY-MM-DD HH:MM) — N warning(s):`.  Notices are sent
as a separate message to #system-messages.  Per-fill alerts are scoped
to the current fill only — each fill's events are captured in
`current_fill_alerts` in the JSON state file, so previous fills'
warnings are never re-sent.

#### Fill Outcomes

| ID | Severity | Channel | Component | Example message |
|----|----------|---------|-----------|-----------------|
| [W1](#w1--overtime) | ⚠️ warn | #anomaly | adjust, report | `⚠️ OVERTIME: GS52 (D- 3) fill time limited to 421s. End overflow 2.98V — fill status uncertain.` |
| [W2](#w2--invalid--duplicate-gsids) | ⚠️ warn | #anomaly | adjust, report | `⚠️ Invalid/Duplicate GSIDs in fill: Out-of-range: B-10 GSID 210 (×3)` |
| [ML-1](#ml-1--missing-or-unparseable-fill-log) | 🔴 red | #anomaly | adjust, report | `🔴 Fill log missing: fill_20260519_1800.log — expected log file does not exist.` |
| [IL-2](#il-2--incomplete-fill-log-partial) | 🔴 red | #anomaly | adjust, report | `🔴 Incomplete fill log: fill_20260524_0600.log (30 detectors, F-fill) — missing completion footer.` |
| [IL-3](#il-3--incomplete-fill-log-empty) | 🔴 red | #anomaly | adjust, report | `🔴 Incomplete fill log: fill_20260519_1800.log (F-fill) — no detector data and no completion footer.` |

#### Temperature & Sensor Warnings

| ID | Severity | Channel | Component | Example message |
|----|----------|---------|-----------|-----------------|
| [W3-a](#w3-a--bad-temperature-readback-none-start) | ⚠️ warn | #anomaly | adjust, report | `⚠️ Bad temperature readback: A- 4 (GSID 504) start=None, end=None/-- — check hose mapping` |
| [W3-b](#w3-b--bad-temperature-readback-end-reading-ok) | ℹ️ info | #system-messages | adjust, report | `ℹ️ Bad temperature readback: B-19 (GSID 21) start=30.79K, end=87.46K — end reading ok` |
| [W3-c](#w3-c--warm-detector) | ⚠️ warn | #anomaly | adjust, report | `⚠️ Warm detector: A-16 (GSID 99) start=200.0K, end=195.0K — possible warm detector` |
| [W3-d](#w3-d--sensor-fault-abnormal-temperature) | ⚠️ warn | #anomaly | adjust, report | `⚠️ Abnormal temperature: B-19 (GSID 21) start=30.79K, end=None/-- — sensor fault` |

**Temperature classification logic:**
- `temp_begin` is None → always ⚠️ warning, regardless of `temp_end`.
- `temp_begin` anomalous (non-None, ≤60K or >150K), severity depends on `temp_end`:
  - `temp_end` valid (60–150K) → ℹ️ info ("end reading ok")
  - `temp_end` also invalid → ⚠️ warning:
    - Both >150K and <320K → "warm detector" (high-side anomaly only)
    - Otherwise → "sensor fault" (low-side, extreme, or None)
- A bad `temp_end` in isolation (valid start, bad end) is not reported.

#### Fill Time Adjustments

| ID | Severity | Channel | Component | Example message |
|----|----------|---------|-----------|-----------------|
| [N2-a](#n2-a--temperature-adjustment-applied) | ℹ️ info | #system-messages | adjust | `ℹ️ GS95 (A- 2) starting temp 2.8K above average. Min fill time: 135 → 162s (+27s)` |
| [N2-b](#n2-b--temperature-adjustment-budget-exhausted) | ℹ️ info | #system-messages | adjust | `ℹ️ GS95 (A- 2) starting temp 1.2K above average. No adjustment (budget exhausted). Current min fill time: 220s` |
| [W6](#w6--clamp-ceiling-reached) | ⚠️ warn | #anomaly | adjust | `⚠️ GS83 (A-24) min fill time has reached upper limit (259s, clamped at 260s)` |
| [N5](#n5--hose-based-min-fill-time-reassignment) | ℹ️ info | #system-messages | adjust, pre-fill | `ℹ️ GS083 (A-24) min fill time set from hose history: 166 → 200s` |

#### Detector Configuration

| ID | Severity | Channel | Component | Example message |
|----|----------|---------|-----------|-----------------|
| [N4-a](#n4--detector-configuration-changes) | ℹ️ info | #system-messages | adjust, report | `ℹ️ New detector: GS030 on hose A-4` |
| [N4-b](#n4--detector-configuration-changes) | ℹ️ info | #system-messages | adjust, report | `ℹ️ Removed detector: GS073 (was on hose A-1)` |
| [N4-c](#n4--detector-configuration-changes) | ℹ️ info | #system-messages | adjust, report | `ℹ️ Moved detector: GS042 from hose B-3 to hose A-1` |

#### Vent-Valve (Manifold Priming)

| ID | Severity | Channel | Component | Example message |
|----|----------|---------|-----------|-----------------|
| [VV-1](#vv-1--marginal-vent-valve-overflow) | ℹ️ info | #system-messages | report | `ℹ️ Manifold A vent 'WARM' (4.50V, open 300s) at start of fill — manifold was not fully primed for detector fill` |
| [VV-2](#vv-2--warm-vent-valve-overflow) | ⚠️ warn | #system-messages | report | `⚠️ Manifold A vent 'WARM' (3.50V, open 300s) at start of fill — manifold not primed for detector fill, needs more time` |
| [VV-3](#vv-3--vent-valve-bad-led) | ⚠️ warn | #system-messages | report | `⚠️ Manifold A vent-valve LED BAD START (begin=5.90V, end=4.50V)` |
| [VV-4](#vv-4--short-vent-valve-priming) | ⚠️ warn | #anomaly | report | `⚠️ Manifold A vent-valve primed in 45s (< 60s) — priming sensor may be bad` |

### Weekly Summary (Monday rollover)

Fired by the **reporter** when it finds a `pending_weekly_summary`
in the JSON state (written by the classifier on Monday rollover).
The key is cleared after sending.

| ID | Severity | Channel | Component | Example message |
|----|----------|---------|-----------|-----------------|
| [W5](#w5--weekly-faulty-ln2-sensors-monday-rollover) | ⚠️ warn | #anomaly | reporter | `⚠️ Weekly LED voltage — faulty LN2 sensors (May 18–May 24): 39 events across 3 detectors` |
| [N3](#n3--weekly-large-min-time-changes-monday-rollover) | ℹ️ info | #system-messages | reporter | `ℹ️ Weekly large min time changes (May 18–May 24): GS38 (D-24): 280 → 163s (-117s)` |

### Suppression

```bash
# Suppress all Discord alerts for a specific component
python3 -m fill_monitor adjust --logfile ${outfile1} --no-discord
python3 -m fill_monitor report --logfile ${outfile1} --no-discord
python3 -m fill_monitor add-press --no-discord
python3 -m fill_monitor monitor-tanks --no-discord
python3 -m fill_monitor pre-fill-adjust-check --no-discord
python3 -m fill_monitor check-press --no-discord
```

The `--test` flag on add-press and monitor-tanks implies `--no-discord`.

### Alert Reference

Detailed explanation of each alert: what triggers it, what it means,
and what action (if any) is needed.

> **Note:** The "Action" recommendations in this section were generated
> by AI and have not been reviewed for operational accuracy.  Use
> engineering judgment and consult the system documentation before
> acting on any recommendation.

*Infrastructure & IOC Health*

---

#### PV-1 / PV-2 — Valve PV Read Failures

LN IOC PV reads for valve states failed 5 consecutive times.  The LN
IOC or network may be down.  The process (add-press or monitor-tanks)
is aborting because it cannot safely manage pressure or monitor
manifold state without knowing valve states.

These PVs (manifold feed LNMx_FV:EN and ext fill LNTx_FV:EN) come
from the LN IOC that controls the fill valves.

**Action:** Check the LN IOC status and network connectivity to
ln2con.  If the IOC restarted, the next fill cycle should work normally.
If the network is down, the fill system cannot operate.

---

#### CP-1 — IOC Health Check Failure

All non-failed pressure gauges returned None (timeout or read error).
The LN IOC or network path may be down.  This check runs on **every**
cron invocation of check-press (every 5 minutes), regardless of the
adaptive rate gating that controls InfluxDB logging.

Known-failed gauges (marked in PRESS_FAIL) are excluded from this
check — they always return None and don't indicate an IOC problem.
Only gauges that are supposed to be working are considered.

**Action:** Check the LN IOC status and network connectivity.  If only
check-press fires this alert (add-press and monitor-tanks are not
running), the IOC may have gone down between fills.  If add-press also
fails its valve reads, the next fill will abort with [PV-2].

---

#### PF-2 — Pre-Fill LN IOC PV Read Failure Abort

The pre-fill check did one batched read of all 224 LN IOC PVs
(112 manifold assignment SM:SUB.E + 112 valve state FV:EN) and got
back at least `MAX_CONSECUTIVE_PV_FAILURES` (5) `None` values in
that single batch.  These come from the LN IOC that controls the
fill valves.  The check was aborted because without knowing which
detector is on which hose and whether its valve is AUTO, fill times
cannot be safely adjusted.  The fill will proceed without any
pre-fill adjustments.

**Action:** Check the LN IOC status and network connectivity.  The
adjuster will still run after the fill and apply normal adjustments.

---

#### PF-3 — Pre-Fill Collector IOC PV Failure Notice

The pre-fill check did one batched read of all DV_EN + DV_TEMP PVs
for the active GSIDs and got back at least
`MAX_CONSECUTIVE_PV_FAILURES` (5) `None` values in that single
batch.  These come from a different IOC than the LN IOC PVs used
for manifold mapping.

Unlike PF-2, this does **not** abort the pre-fill check.  Instead:
- **DV_EN unavailable** → matrix classification (PFM-1..5)
  are skipped for the affected detectors.  Hose reassignment still
  works (it depends on valve state from the LN IOC, not DV_EN).
- **DV_TEMP unavailable** → temperature bump is skipped for the
  affected detectors.  Each keeps its current min_time.

The whole Collector batch is one read so the notice naturally fires
at most once per pre-fill check run.  Sparse per-PV failures below
the batch threshold do NOT fire PF-3 — they just silently skip
the affected detector's VS/temp-bump checks.

**Action:** Check the Collector IOC status and network connectivity.
Detectors with missing DV_EN/DV_TEMP reads will miss their pre-fill
adjustments but will be corrected by the post-fill adjuster.

---

*Pre-Fill Checks*

---

#### PF-1 — Pre-Fill Temperature Bump

Before a scheduled F-fill, a detector was found running hotter than its
historical average.  The min_time was temporarily increased to prevent
underfilling.

**Action:** None needed.  The bump is temporary — the post-fill
adjuster restores the original baseline and applies the full adjustment
with budget tracking.  If the same detector triggers pre-fill bumps
repeatedly, it may have a chronic temperature issue.

---

#### Prefill check — detector state matrix

The prefill check classifies every detector against a 12-row state
matrix built from four dimensions: **DV_EN** (0 or 1), **hose**
(assigned or not), **temperature** (cold or warm), and **valve**
(Auto or Disable; valve=Open is handled separately by the
orthogonal **PFM-OPEN** dispatch — see below).  When the matrix
classifier picks a non-silent row, one Discord message dispatches
immediately for that detector to the routed channel.  No batching;
one message per affected detector per prefill run.

Severity ↔ channel routing:
- RED, YELLOW, anomalous-info → **#anomaly**
- system-info → **#system-messages**

Matrix:

| # | DV_EN | Hose | Temp | Valve   | ID | Severity | Channel | Message |
|--:|:-----:|:----:|:----:|:-------:|:----:|:--------:|:-------:|---------|
| 1 | 1 | Yes | Cold | Auto    | —        | — silent | — | (no message — healthy fillable monitored cold detector) |
| 2 | 1 | Yes | Cold | Disable | PFM-1 | 🔴 RED      | #anomaly | `🔴 GS073 (A- 1): cold detector with valve Disabled — detector will not be filled. Temp=85.0K, DV_EN=1.` |
| 3 | 1 | Yes | Warm | Auto    | PFM-2 | ⚠️ YELLOW   | #anomaly | `⚠️ GS073 (A- 1): warm detector will be filled. Temp=290.0K, valve=Auto, DV_EN=1.` |
| 4 | 1 | Yes | Warm | Disable | PFM-4 | ℹ️ info     | #anomaly | `ℹ️ GS073 (A- 1): warm detector, monitored, will not be filled. Temp=290.0K, DV_EN=1, valve=Disable.` |
| 5 | 1 | No  | Cold | n/a     | PFM-1 | 🔴 RED      | #anomaly | `🔴 GS073: cold detector with no hose assigned — detector will not be filled. Temp=85.0K, DV_EN=1.` |
| 6 | 1 | No  | Warm | n/a     | PFM-4 | ℹ️ info     | #anomaly | `ℹ️ GS073: warm detector, monitored, will not be filled (no hose). Temp=290.0K, DV_EN=1.` |
| 7 | 0 | Yes | Cold | Auto    | PFM-3 | ⚠️ YELLOW   | #anomaly | `⚠️ GS073 (A- 1): cold detector, not monitored. Detector will be filled but is unmonitored. Temp=85.0K, DV_EN=0, valve=Auto.` |
| 8 | 0 | Yes | Cold | Disable | PFM-1 | 🔴 RED      | #anomaly | `🔴 GS073 (A- 1): cold detector with valve Disabled — detector will not be filled. Temp=85.0K, DV_EN=0.` |
| 9 | 0 | Yes | Warm | Auto    | PFM-2 | ⚠️ YELLOW   | #anomaly | `⚠️ GS073 (A- 1): warm detector will be filled. Temp=290.0K, valve=Auto, DV_EN=0 (not monitored).` |
| 10 | 0 | Yes | Warm | Disable | PFM-5 | ℹ️ info  | #system-messages | `ℹ️ GS073 (A- 1): warm detector in array, will not be filled. DV_EN=0, valve=Disable, Temp=290.0K.` |
| 11 | 0 | No  | Cold | n/a     | PFM-1 | 🔴 RED     | #anomaly | `🔴 GS073: cold detector with no hose assigned — detector will not be filled. Temp=85.0K, DV_EN=0 (not monitored).` |
| 12 | 0 | No  | Warm | n/a     | PFM-5 | ℹ️ info  | #system-messages | `ℹ️ GS073: warm detector in array (no hose), will not be filled. DV_EN=0, Temp=290.0K.` |

**Orthogonal dispatch — PFM-OPEN (concurrent M-fill detection):**
Any valve in the **Open** state at prefill time is suppressed from
the matrix and reported via a manifold-level message instead.  This
reflects the most likely cause of a valve being Open during prefill
(a concurrent M-fill on the same manifold).  One YELLOW message per
affected manifold:

> `⚠️ [PFM-OPEN] Manifold A: prefill check ran while 3 valves are Open (positions 1, 5, 12) — possible concurrent M-fill. Per-detector states may be transient; review after current fill completes.`

The matrix classifier still runs against every detector — it treats
Open as Auto for the configuration question — so the operator gets
both the configuration verdict (matrix row) and the concurrency
notice (PFM-OPEN) when applicable.

**Common operator actions by ID:**
- **PFM-1 (RED, cold won't fill):** Either set the valve to Auto, or
  assign / re-verify the manifold hose mapping (`SM:SUB.E`).  A cold
  detector that won't get filled is the highest-priority issue.
- **PFM-2 (YELLOW, warm will fill):** Verify whether the warm state
  is expected (e.g. first fill after a warm-up).  If unexpected, the
  fill will still proceed — monitor the result.
- **PFM-3 (YELLOW, cold unmonitored):** Set `DV_EN=1` in EPICS so the
  detector's temperature joins post-fill tracking.  The fill itself
  still happens.
- **PFM-4 (info, warm monitored, won't fill):** No action required —
  detector is being watched but is intentionally not in the fill
  rotation.  Anomaly channel for visibility.
- **PFM-5 (info, warm retired):** No action required — expected
  retired-detector state.  Operational channel; safe to ignore.
- **PFM-OPEN (YELLOW, concurrent fill):** Check whether an M-fill is
  running on the affected manifold.  Per-detector states reported in
  the same prefill run may be transient.

---

#### VS-4 — Wrong-Side Hose Assignment

The detector is on the wrong Transfer Station (TS).  Each manifold
serves one half of the Gammasphere array:

- **TS1** serves Manifolds A and B — holds all **odd** GSIDs
- **TS2** serves Manifolds C and D — holds all **even** GSIDs

A detector physically on TS1 cannot be plumbed to a TS2 hose and
vice versa.  If the assignment crosses sides, either the SM:SUB.E
readback is wrong or the detector was physically moved to the wrong
hose by accident.

Only fires for valves in AUTO state — a mis-mapping on a Disabled
valve isn't immediately operational.

**Action:** Verify the physical hose connection.  Either move the
hose, or correct the SM:SUB.E mapping.

---

#### GF-1 — Missing GSID in gefilltime2.dat

A detector appeared in the fill (or was enabled in EPICS) but had no
entry in gefilltime2.dat.  The system added it automatically using the
best available source: (1) hose history from the previous occupant,
(2) the detector's own fill history, or (3) the default (150s).

This typically happens when a new detector is installed.  It can also
indicate an accidental deletion from gefilltime2.dat.

Fired by both the pre-fill check and the adjuster.  The default flush
command (`flush-history`) does NOT modify gefilltime2.dat.

**Action:** Verify the fill time is appropriate for this detector and
hose.  Use `flush-history --min-time-only N` to set a specific value
if the automatic choice is wrong.

---

#### GF-2 — Missing gefilltime2.dat

The gefilltime2.dat file is missing entirely.  Two messages fire at
different stages:

- **Pre-fill check** (⚠️ warn): The pre-fill check cannot operate
  without min_times.  It aborts and the fill proceeds with whatever
  defaults LNFill_App uses internally.
- **Adjuster** (ℹ️ info): The adjuster recovers the file from the
  min_time_history archive (or creates it with defaults if no archive
  exists).  Normal operation resumes from the next fill.

**Action:** Investigate why the file was missing (filesystem issue,
accidental deletion).  The adjuster has recovered it.

---

*Fill Monitoring*

---

#### BL-1 — Broken Manifold Feed Line

**SAFETY ALERT.** Four or more of the last six eligible detectors on
the same manifold show no LN2 flow (LED overflow voltage stays at room
temperature after 180s open).  This indicates the manifold feed line
between the tank station and Gammasphere is broken or detached — LN2 is
dumping into the room at ~32 psi.

The detector uses a **4-of-6 sliding window** of recent detectors (in
fill order per manifold).  This catches cases where one detector in the
middle shows flow (thermal coupling, marginal LED) but the feed line is
actually broken.

**Eligibility rules:** A detector enters the window only when it has
been open for >= 180s (with a live verdict) or has closed after a fill
of >= 180s (with a finalized verdict).  Additionally, its LED voltage
must be in the valid range (1.60V–5.86V).  Detectors with fill times
< 180s or invalid LED voltages are invisible to the window.

Active detectors (valve still open, past 180s) have live verdicts that
update each poll cycle.  When a valve closes, the verdict is locked.
The window is rebuilt every poll cycle from a history of ~12 recent
eligible detectors.  The trigger fires immediately when 4+ no-flow
verdicts are found in the window.

The manifold cold sensor (upstream of the break) stays cold because LN2
is still flowing past it.  The only observable is that detector LED
sensors show no cooling — nothing cold is reaching Gammasphere.

**Automatic action:** The manifold feed valve is closed immediately
(`LNM{x}_FV:EN → Auto`).  LNFill_App will continue trying to fill
detectors on that manifold but with no LN2 flowing, they will time
out harmlessly.

**Manual action:** Inspect the manifold feed line between the tank
station and Gammasphere immediately.  Do not re-open the feed valve
until the line is verified intact.

**Kill switch.**  Add `--no-feedline-check` to the add-press or
monitor-tanks command (or to the cron line) to fully disable BL-1.
With the flag set, no detector valve PV reads happen, no flow/
no-flow tracking runs, and no alerts fire.  This is the rollback
path if BL-1 misbehaves in production — no code change required.

**Failover interaction.**  When a tank failover fires (FO-2, FO-4,
or FO-5) on a manifold, BL-1 is suppressed on that same manifold
for the next 240s.  The newly opened feed valve needs a few seconds
to wet the line and let cold gas reach the detector LEDs again;
without the suppression window, BL-1 could sample LED voltages
during the dry-line gap and falsely close the just-opened spare
feed on top of the empty-tank failure that just triggered failover.
Suppression is per-manifold — a failover on Manifold A does not
suppress BL-1 on Manifold B even though both are on TS1.  No
operator action is needed; the suppression is logged to stderr as
`[BL-SUPPRESS] Manifold A: skipped (failover cooldown, Ns)` each
poll cycle until it expires.

---

#### Broken Feed Line at End of Priming — BL-2 (RED) / BL-3 (YELLOW)

**SAFETY ALERT.**  At the end of the priming phase (when the
manifold vent valve closes), one detector evaluates the highest
vent-LED voltage observed during priming and dispatches one of
three outcomes per manifold per fill:

| Tier | max vent LED at vent close | Severity | Action | Internal marker |
|------|----------------------------|----------|--------|-----------------|
| BL-2 | < 2.0V                     | 🔴 RED   | Close manifold main feed valve + RED Discord | `[BL-2]` |
| BL-3 | 2.0V ≤ max < 3.0V          | ⚠️ YELLOW | Discord notice only — NEVER closes valves | `[BL-3]` |
| (silent) | ≥ 3.0V                | —        | None (handled by post-fill VV-1/VV-2) | n/a |

The internal `[BL-N]` markers appear only in stderr logs and
developer docs.  Operator-facing Discord messages do NOT carry the
tag (RED message says "BROKEN FEED LINE DETECTED"; YELLOW message
says "manifold not primed before start of detector filling").

**BL-2 RED interpretation:** the manifold vent LED never rose above
2.0V — essentially no LN2 reached the vent.  This is a broken feed
line (split, disconnect, blocked valve) or a facility-wide LN2
supply failure.  In current operation BL-2 RED fires ~6 times per
year on F-fills, almost all confirmed against detector OVERTIME
fills on the same manifold.

**BL-3 YELLOW interpretation:** the manifold vent LED reached 2-3V
— some flow got through but not enough to fully prime.  Detector
fill performance may be degraded but the fill is allowed to
proceed.  BL-3 YELLOW fires ~10 times per year on F-fills.

**Initial deployment is INERT for BL-2.**  In the inert phase,
BL-2 RED still sends Discord but with a `[BL-2 IN DEVELOPMENT —
NO ACTION TAKEN]` prefix, and the feed valve does NOT close.
BL-3 YELLOW is live from day 1 (informational only, never needed
inert mode).

Example messages:

*BL-2 RED (inert mode — current deployment):*
```
🔴 [BL-2 IN DEVELOPMENT — NO ACTION TAKEN] BROKEN FEED LINE DETECTED:
Manifold A — vent LED 1.85V at vent close (<2.0V). No LN2 reaching
manifold. When activated, this would close the main feed valve.
```

*BL-2 RED (after activation, future):*
```
🔴 BROKEN FEED LINE DETECTED: Manifold A — vent LED 1.85V at vent
close (<2.0V). No LN2 reaching manifold. Main feed valve closed.
```

*BL-3 YELLOW (always-on):*
```
⚠️ Manifold A not primed before start of detector filling — vent
LED 2.45V at vent close (<3.0V). Filling performance will be
degraded.
```

**Arm condition:** the detector arms on the first vent valve
`Auto→Open` transition per fill per manifold, AND only when the TS
manifold LED (`LNM{n}A_TM:BT`) reads `'Cold'` at that moment.  This
gates evaluation on "LN2 was actually flowing out of the tank
station when priming started" — empty-tank failures don't trigger
BL-2/BL-3 (they're ManifoldFailover's job).

**Disarm conditions** (any one ends evaluation for that manifold's fill):
- Vent valve closes naturally (also triggers the tier evaluation above)
- A detector valve on the manifold opens (= detector-fill phase
  started; BL-1 takes over)
- TS manifold LED goes Cold→Warm (= tank emptied; ManifoldFailover
  taking action)
- A tank failover succeeds for this manifold (redundant safety net
  matching the BL-1 holdoff pattern)

**If the detector is disarmed AT the moment of vent close** (e.g.
failover suppress() fired just before close), NEITHER tier fires
regardless of vent LED voltage.  The manifold is locked out for the
rest of the fill cycle.

**Kill switch.**  BL-2/BL-3 is covered by the same
`--no-feedline-check` flag as BL-1.  When set, this detector is
never constructed; no vent-LED reads, no alerts.

**Once per fill.**  After RED or YELLOW fires (or any disarm
condition triggers), the detector stays finalized for the rest of
that fill cycle.  An expert operator manually re-opening the vent
valve mid-fill is ignored — BL-2/BL-3 only arm on the FIRST vent
transition per fill.

---

#### N-LP — Low Tank Pressure (20 psi, info)

A tank station's pressure dropped below 20 psi during the fill.  This
is an early warning — the fill is still proceeding but pressure is
getting low.

**Action:** Monitor.  The external supply should be maintaining
pressure via the ext fill valve.  If pressure continues to drop, the
tanks may be nearly empty or the external supply is insufficient.

---

#### W-LP — Low Tank Pressure (warning)

Tank pressure dropped below a warning threshold (18, 15, 12, 9, 6, or
3 psi).  Each threshold fires once per station per fill.  Low pressure
means LN2 flow to detectors is reduced — fills may be incomplete.

**Action:** Check the external LN2 supply.  If the bulk dewar is empty,
fills will be short until it's refilled.  If the supply is available but
pressure is still dropping, the ext fill valve may not be opening
(check add-press logs) or the supply line may be blocked.

---

#### W-WM — Warm Manifold Detected

A manifold temperature sensor transitioned from Cold to Warm during an
active fill.  This suggests the feeding tank has run out of liquid
nitrogen — warm gas is now flowing instead of liquid, which can blow
remaining LN2 out of detector dewars.

**Action:** If failover is enabled (default), add-press/monitor-tanks
will automatically switch to the alternate feed (see FO-1 through
FO-5).  If failover is disabled (`--no-failover`), manual intervention
is needed to prevent detector damage.

---

#### FO-1 — Failover: Spare Feed Disabled

The main tank went empty (warm manifold detected) but the spare feed
valve is in Disabled state — automatic failover cannot proceed.

**Action:** Manual intervention required.  Either enable the spare feed
valve on the AB controller, or manually switch the manifold feed.
Detectors on this manifold are receiving warm gas until addressed.

---

#### FO-2 — Failover: Main → Spare

The main tank went empty and the system automatically switched the
manifold to the spare tank feed.  Filling continues via the spare.
Ext fill pressure management has been redirected to replenish the
empty main tank.

**Action:** None immediate — the failover handled it.  After the fill,
verify the empty tank is refilled during the tank refill phase.  If
this happens frequently, the main tank may be undersized or the fill
schedule may need adjustment.

> After this failover, pressure management moves to a main tank.
> Add-press closes the ext fill valve within ~1 second of the station's
> manifolds closing, well before TankMan begins tank refill.

---

#### FO-3 — Failover: Main Feed Disabled (spare-first fill)

The fill was running on the spare tank (ManID≥5 configuration), the
spare went empty, but the main feed valve is Disabled — cannot switch.

**Action:** Same as FO-1.  Manual intervention required.

---

#### FO-4 — Failover: Spare → Main

The fill was running on the spare tank (ManID≥5 configuration or after
a previous FO-2 failover) and the spare went empty.  The system
switched to the main tank feed.

**Action:** Same as FO-2.  Filling continues on the main tank.

> After this failover, ext fill moves to the spare tank.  Add-press
> closes the ext fill valve within ~1 second of the station's
> manifolds closing, well before TankMan begins tank refill.

---

#### FO-5 — Both Feeds Exhausted

Both the main and spare tanks have been tried and both went warm.  The
manifold feed has been closed to prevent blowing LN2 out of detector
dewars with warm gas.  Detectors on this manifold will not receive LN2
for the remainder of this fill.

**Action:** This is a serious condition.  The tanks on this station are
empty.  Check the external LN2 supply and tank refill status.  Affected
detectors will warm up and likely trigger temperature overrides on the
next fill.

---

<a id="fo-6"></a>
#### FO-6 — Failover Confirmed

After a successful failover (FO-2 or FO-4), the manifold temperature
sensor transitions back to Cold, confirming that liquid nitrogen is
flowing from the new feed.  This resolves the preceding red alert.

**Action:** None needed — the failover succeeded and the fill is
continuing normally on the alternate tank.

---

#### AD-1 through AD-5 — Ext Fill Valve Adaptation

The ext fill valve selection has been changed because the original
target tank is now in use as a manifold feed, or a valve is Disabled.

- **AD-1:** Ext fill tank now in use as manifold feed — ext fill
  switched to an unused tank.
- **AD-3:** A tank's ext fill valve is Disabled — cannot manage pressure
  using that tank.
- **AD-4:** No available ext fill valve on this station — all tanks are
  either feeding manifolds or have Disabled ext fill valves.  Pressure
  management is unavailable for this station.
- **AD-5:** The default ext fill valve (spare tank) was Disabled at
  startup — switched to an alternate tank.

**Action for AD-1/5:** Informational — the system adapted
automatically.  No action needed unless this indicates an unexpected
configuration.

**Action for AD-3/4:** Check why ext fill valves are Disabled.  If
intentional (maintenance), pressure will not be managed on that station
during fills — tank pressure may drop.

> After ext fill adaptation, add-press closes the ext fill valve
> within ~1 second of the station's manifolds closing, well before
> TankMan begins tank refill.

---

#### EX-1 — Monitor-tanks Hard Timeout (manifolds open)

Monitor-tanks hit its hard timeout safety net with manifold valves still
open.  This should not happen in normal operation — the fill should
complete and manifolds should close before the timeout.

**Action:** Check if LNFill_App is hung or if the fill is taking
unusually long.  The manifold valves may need to be closed manually.

---

#### EX-2 — Parent Process Terminated (manifolds open)

The parent cron script terminated (crashed or was killed) while manifold
valves are still open.  Monitor-tanks is exiting but the fill may still
be in progress without orchestration.

**Action:** Check if LNFill_App is still running.  If so, it will
continue the fill independently.  If not, manifold valves may need to
be closed manually via the AB controller.

---

#### EX-3 — AddPress Exiting (manifolds open)

AddPress is exiting (parent death or hard timeout) with manifold valves
still open.  Ext fill valves are being closed as a safety measure.
Tank pressure will no longer be managed for the remainder of the fill.

**Action:** Same as EX-2.  Check if the fill is still running.  Tank
pressure may drop without ext fill valve management, potentially
causing incomplete detector fills.

---

#### I-TO — Non-Critical Timeout

AddPress hit the 2200s hard timeout but all manifold valves were already
closed.  The fill completed normally but add-press's exit condition
didn't trigger promptly.  This is not an operational emergency.

**Action:** Review AddPress.log for the exit timing.  This typically
indicates a software issue with the exit condition detection, not a
hardware or fill problem.

---

*Post-Fill Analysis*

---

#### W1 — OVERTIME

A detector's fill valve was forced closed at max_time without the
overflow LED going Cold.  The dewar may not be fully filled.

- **End overflow < 5.6V:** LED was working but never triggered — the
  fill was genuinely too short.  The detector may warm up before the
  next scheduled fill.
- **End overflow > 5.6V:** LED sensor may be faulty (always reads high).
  The detector was likely filled normally but the sensor couldn't
  confirm it.

**Action:** Check the detector's overflow LED sensor and hose
connection.  If this recurs across multiple fills, the sensor or wiring
may need repair (see W5 for weekly chronic LED tracking).

---

#### W2 — Invalid/Duplicate GSIDs

The fill log contains detector GSIDs outside the valid range
(1–110, i.e. `< 1` or `> 110`) or the same GSID appearing multiple
times.  This indicates a configuration error in the hose-to-detector
mapping on the AB controller.  Both ends of the range are caught:
GSIDs below 1 (negative or zero) and above 110.

**Action:** Check the hose mapping in `ln.inits` on ln2con.  An
out-of-range GSID means a hose position is mapped to a nonexistent
detector.  Duplicate GSIDs mean two hoses claim to fill the same
detector — one of them is wrong.

---

#### ML-1 — Missing or Unparseable Fill Log

The fill log file does not exist, or exists but contains no recognizable
fill data (no timestamp header).  This is distinct from IL-2/IL-3 where
the file exists and was parsed but is incomplete.

Two variants:
- **File missing:** The fill never produced a log file.  For scheduled
  F-fills this is abnormal — the cron script always passes a log path.
- **File empty/unparseable:** The file was created but LNFill_App crashed
  before writing any recognizable output.

Suppressed by `--no-missing-log-alert` (used in AUTO cron for M-fills,
where most runs find no warm detectors and exit without creating a log).

**Action:** Check if LNFill_App started.  Review cron logs and the error
log.  If the fill script itself failed to launch, the log won't exist.

---

#### IL-2 — Incomplete Fill Log (Partial)

The fill log has detector data but is missing the "Total App Runtime"
footer.  This means LNFill_App started filling detectors but crashed or
was killed before completing.  Some detectors were filled; others may
not have been.

**Action:** Check why LNFill_App terminated.  Review the error log
(`fill_YYYYMMDD_HHMM.error.log`).  Detectors that were filled will
have their min_times adjusted normally; detectors that were missed will
not appear in the log and won't be adjusted.

---

#### IL-3 — Incomplete Fill Log (Empty)

The fill log was created (fill started) but contains no detector data
and no completion footer.  The fill failed to reach the detector filling
phase — typically all manifolds aborted during the priming/vent phase.

**Action:** Check the error log and the manifold vent valve status.  All
four manifolds failing to prime usually indicates a systemic issue
(supply line empty, all vent valves Disabled, or AB controller
communication failure).

---

#### W3-a — Bad Temperature Readback (None start)

The detector's start temperature read as None — no EPICS readback at
all.  This is always a warning regardless of end temperature.

**Action:** Check the hose mapping.  A None temp usually means the hose
position is not connected to a working temperature sensor, or the GSID
doesn't map to a real detector.

---

#### W3-b — Bad Temperature Readback (end reading ok)

The detector's start temperature was anomalous (≤60K or >150K) but the
end temperature was valid (60–150K).  This is a transient readback
glitch — the detector is fine.

**Action:** None needed.  The start reading was a one-time EPICS
artifact.  The adjuster uses the valid end temperature to confirm the
detector is operating normally.  If this recurs frequently on the same
detector, there may be an intermittent sensor issue.

---

#### W3-c — Warm Detector

Both start and end temperatures are above the valid range (>150K) but
below 320K.  The detector is physically warmer than it should be — it's
not cold enough for normal LN2 fill operation.

**Action:** Investigate why the detector is warm.  Possible causes:
dewar vacuum loss, insufficient LN2 delivery, or the detector was
recently installed and hasn't cooled down yet.

---

#### W3-d — Sensor Fault (Abnormal Temperature)

The start temperature is anomalous and the end temperature is also
invalid (or None).  This indicates a persistent sensor problem — the
temperature readback is unreliable for this detector.

**Action:** Check the temperature sensor and wiring.  Low-side anomalies
(≤60K) and extreme values (≥320K) are sensor faults, not warm detectors.

---

#### N2-a — Temperature Adjustment Applied

A detector started this fill warmer than its historical average (≥1K
above mean).  The adjuster increased its minimum fill time to
compensate — a warmer detector needs more LN2.

**Action:** None needed.  This is normal adaptive behavior.  The
adjustment is bounded by the temperature budget (30s max, regenerates
over 72h) and the fill-time floor (min_time ≥ fill_time + 10s).

---

#### N2-b — Temperature Adjustment (Budget Exhausted)

Same condition as N2-a, but the temperature adjustment budget is
exhausted — no increase was applied.  The fill-time floor may still
have increased min_time if the actual fill time exceeded it.

**Action:** Monitor this detector.  If it consistently runs warm and
the budget is always exhausted, the base min_time may need manual
review.  The budget regenerates 5s per 12h (full recovery in 72h).

---

#### W6 — Clamp Ceiling Reached

A detector's min_time adjustment was limited by the upper clamp
(`min(360, max_time × 0.9)`).  The adjuster wanted to increase
min_time further but the clamp prevented it.

**Action:** Review the detector's max_time in gefilltime2.dat.  If the
max_time is appropriate and the detector genuinely needs a longer fill,
the max_time may need to be increased.  If the detector has a chronic
sensor issue driving false temperature overrides, fix the sensor first.

---

#### N5 — Hose-Based Min Fill Time Reassignment

A new or moved detector had its min fill time set from the destination
hose's history.  Fill times are mostly hose-dependent (line length,
routing, manifold position), so the previous occupant's fill time is a
better starting point than carrying over the old hose's tuned value.

Fired by both the pre-fill check (before the fill) and the adjuster
(after the fill).  The pre-fill check detects moves via live EPICS
manifold PV reads; the adjuster detects them via `known_detectors`.

GF-1 and N5 are mutually exclusive for the same detector — if a GSID
was missing from gefilltime2.dat and added via GF-1, N5 does not also
fire because GF-1 already set the value from hose history.

**Action:** Verify the detector was intentionally moved or installed.
The fill time will continue to be adjusted normally after this fill.

---

#### N4 — Detector Configuration Changes

Three sub-types track changes to the installed detector population
via the `known_detectors` dict (maps GSID → hose):

- **N4-a (New detector):** A GSID appears for the first time in any
  fill type.  Sent to #system-messages.
- **N4-b (Removed detector):** A GSID that was previously known is
  absent from an F-fill.  Only F-fills can trigger removal detection
  (F-fills see all installed detectors).  Sent to #system-messages.
- **N4-c (Moved detector):** An existing GSID appears on a different
  hose than previously recorded.  Sent to #system-messages.

N4 replaces the former N1 (hose reconfiguration) which was hose-centric.
N4 provides detector-centric coverage of the same events. Hose-GSID
mapping changes are still recorded in the JSON state for audit but no
longer appear in the weekly log or Discord.

Unresolved duplicate GSIDs are excluded from all detector change
detection and are preserved in `known_detectors` during F-fill
replacement.

**Action:** Verify the change was intentional.  New and moved detectors
trigger adjuster state resets for the affected GSID.

---

#### VV-1 — Marginal Vent-Valve Overflow

The vent-valve overflow sensor voltage at end of fill is between 4.00V
and 5.00V.  The manifold was not fully primed at the start of detector
filling.  Marginal — worth monitoring but not critical.

---

#### VV-2 — Warm Vent-Valve Overflow

The vent-valve overflow sensor voltage at end of fill is below 4.00V.
The manifold was not fully primed at the start of detector filling.
A warm delivery system means detectors take longer to fill and may
result in OVERTIME outcomes.

---

#### VV-3 — Vent-Valve Bad LED

The vent-valve overflow sensor voltage exceeds 5.86V at the start,
end, or both phases of manifold priming.  This indicates the overflow
sensor is open, disconnected, or failed — same threshold used for
detector overflow sensors.  Reported as BAD START, BAD END, or BAD BOTH.

The operational consequence depends on which phase was bad:

- **BAD START** (begin > 5.86V, end OK): the sensor was bad at the
  start of priming but recovered.  The END reading was valid, so
  the priming verdict for this fill is trustworthy.  The sensor
  still needs replacement before it fails permanently.
- **BAD END** (begin OK, end > 5.86V): the sensor failed *during*
  priming.  The Cold reading that triggered "primed" may have been
  the SMOO transit toward the open rail rather than actual cold LN2
  reaching the vent valve.  **The manifold may not actually be primed.**
  Detectors filling against an unprimed manifold will see warm gas
  blow through their dewars.
- **BAD BOTH** (both > 5.86V): the sensor was unreliable for the
  entire priming cycle.  No trustworthy reading exists.  **Priming
  status is unknown.**  Treat as not primed until verified by other
  means.

**Action:** Inspect the overflow sensor on the affected manifold's
vent valve.  For BAD END and BAD BOTH, also verify the manifold is
actually cold (e.g. by reading `LNM{n}_TM:AT` directly during the
next priming cycle, or by checking detector fill outcomes for that
manifold — widespread OVERTIME or low cold_time across many detectors
on one manifold suggests the manifold went into detector filling
while still warm).  A bad sensor means the priming controller can't
reliably detect when the manifold is cold.

---

#### VV-4 — Short Vent-Valve Priming

The vent valve primed in under 60 seconds.  This is suspiciously fast
and suggests the priming overflow sensor may be bad (falsely reading
cold).  The manifold may not actually be primed.

**Action:** Check the manifold vent-valve overflow sensor.  Verify
the manifold is actually cold before trusting the fill results.

---

*Weekly Summary*

---

#### W5 — Weekly Faulty LN2 Sensors (Monday rollover)

Overflow LED sensors that reported high voltage (>5.86V) on ≥7 or more
fills during the past week.  These are faulty sensors, not one-time
glitches.  The sensor or wiring is failing and needs physical
repair.

**Action:** Schedule sensor/wiring repair for the affected detectors.
The adjuster uses hard fallback mode for these detectors (limited to -1
downward adjustment per fill) to avoid aggressive min_time changes
based on unreliable sensor data.

---

#### N3 — Weekly Large Min-Time Changes (Monday rollover)

Detectors whose min_time changed by more than 60 seconds during the
past week.  Large changes may indicate a detector problem (warm,
sensor drift) or a hose swap that reset tracking.

**Action:** Review the affected detectors.  Large increases usually
follow temperature events; large decreases follow a period of
consistently short fills after a sensor was repaired or a detector
cooled down.


## Configuration

Fill times are stored in `gefilltime2.dat` (CSV: GSID, min_time, max_time).
All algorithm state is persisted in `logs/fill_monitor/fill_monitor_state.json`.
Per-fill min_time history is archived to `logs/fill_monitor/min_time_history/min_time_YYYYMM.csv`.

All subcommands that use the state file accept `--log-mon-dir` (default:
`logs/fill_monitor`). The state file, weekly logs, and min_time history
are all stored under this directory. If you need a non-standard state
file location, use `--state-file` to override — it takes precedence
over the path derived from `--log-mon-dir`.

If `gefilltime2.dat` is lost, the adjuster automatically recovers the last
known min_times from the archive CSVs.
