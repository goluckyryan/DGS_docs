# Fill Monitor — User Guide

## Overview

The fill monitor automatically adjusts per-detector LN2 minimum fill times
and detects fill anomalies. After each fill cycle, it processes the fill log,
adjusts `gefilltime2.dat`, and updates the weekly monitoring log.

## Components

| Script | Purpose | Inputs | Outputs |
|--------|---------|--------|---------|
| `adjust` | Process fill log: detect anomalies, adjust min fill times, update weekly log | Fill log, gefilltime2.dat | Updated gefilltime2.dat, JSON state, weekly log |
| `report` | Process fill log: detect anomalies, update weekly log (no adjustment) | Fill log | JSON state, weekly log |
| `plot` | Plot per-detector fill history (simulation, testing, diagnostics) | Fill logs or archive CSVs | PNG plots |
| `add-press` | Tank pressure management during fills | EPICS PVs (live) | influx_txt/AddPress.txt, InfluxDB |
| `check-press` | Hourly pressure gauge snapshot | EPICS PVs (live) | influx_txt/check_pressure.txt, InfluxDB |

## Production Usage

### After Each Fill (cron)

The adjuster is the primary entry point. It internally runs the classifier
first, then applies adjustments, then runs the reporter:

```bash
python3 -m fill_monitor adjust --logfile ${outfile1} --filltimes gefilltime2.dat
```

This single command:
1. Parses the fill log (classifier)
2. Classifies anomalies: high voltage, overtime, bad temps, etc. (classifier)
3. Adjusts min_times per detector (adjuster)
4. Updates gefilltime2.dat (adjuster)
5. Writes/updates the weekly monitoring log (reporter)

### Cron Integration

In `LNFill_cron.sh` (full fills):
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

## Plotting

### Archive Mode (recommended for production)

Reads min_time history directly from the monthly CSV logs. No simulation
replay needed — uses the actual min_time values computed in production.

```bash
python3 -m fill_monitor plot --archive --all --outdir plots/ \
    --logdir logs/ --log-mon-dir logs/fill_monitor
```

Requires:
- Fill logs in `--logdir` (for cold_time, open_time, status)
- Min_time history CSVs in `--log-mon-dir/min_time_history/`

### Simulation Replay Mode (default)

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

### Plot Specific Detectors

```bash
python3 -m fill_monitor plot 42 26 18 --logdir logs/ --filltimes gefilltime2.dat
python3 -m fill_monitor plot --archive 42 26 18 --logdir logs/
```

### Weekly Log: Low Tank Pressure

Low pressure events from `fill_add_press.py` are logged to the JSON state
file and appear in section 8 of the weekly log:

```
--- Low Tank Pressure Events ---
  Timestamp             Station  Pressure  Threshold
  2026-05-19 20:45        TS1      15.2     <15 psi
  2026-05-19 20:48        TS2      11.5     <12 psi
```

## Weekly Monitoring Log

The weekly log (`logs/fill_monitor/fillmon_YYYYMMDD.log`) rolls over on Monday 00:00
and contains these sections:

1. **High (Open) LED Voltage** — overflow voltage > 5.85V events
2. **OVERTIME Detectors** — fills that reached max_time
3. **Invalid/Extreme Temperature Readbacks** — bad sensor readings
4. **Invalid/Duplicate GSIDs** — configuration errors
5. **Temperature Override Adjustments** — warm detector events
6. **Clamped Min-Times** — adjustments limited by clamp (adjuster only)
7. **Hose-GSID Mapping Changes** — detector swaps
8. **Min-Time Summary** — per-detector changes for the week (adjuster only)

## File Dependencies

```
fill log → classifier → JSON state → adjuster → gefilltime2.dat
                                   ↓                    ↓
                              reporter → fillmon_YYYYMMDD.log
                                   ↓
                              plotter → PNG plots
```

## AddPress — Tank Pressure Management

During each fill cycle, the detectors draw liquid nitrogen from the local
tanks, causing tank pressure to drop. If pressure falls too low, LN2 flow
to the detectors slows or stops, resulting in incomplete fills.

The `add-press` subcommand runs in the background during every fill to
counteract this pressure drop by opening the spare tank's external supply
fill valve (`LNT3_FV:EN` for TS1, `LNT6_FV:EN` for TS2) to inject
nitrogen from the external supply line into the spare tank. Because the
supply line never fully chills down between fills, what flows in is
predominantly gas rather than liquid — but this is sufficient to maintain
tank pressure while the main tanks are being depleted by the detector fill.

### How It Works

1. Opens spare tank fill valve when external supply pressure exceeds
   local tank pressure by ≥3 psi (`VALVE_OPEN_PRESS`)
2. Closes valve when differential drops to ≤-1 psi (`VALVE_CLOSE_PRESS`)
   or tank reaches 32 psi (`MAX_TANK_PRESS`)
3. Cross-station balancing: if both stations have spare valves open, the
   higher-pressure station closes its spare to share flow
4. Logs all 8 pressure gauges each iteration to InfluxDB
5. Monitors manifold temperature sensors for empty tank detection
6. Bad-sensor warnings printed once at startup, then suppressed

### Usage

```bash
# Production (called by LNFill_cron.sh in background)
python3 -m fill_monitor add-press >> cron_logs/AddPress.log 2>&1 &

# Test without pushing to InfluxDB
python3 -m fill_monitor add-press --no-influxdb

# Full test mode (no InfluxDB, no Discord, log mirrored to test file)
python3 -m fill_monitor add-press --test

# Verify logic against a recorded AddPress.log
python3 -m fill_monitor add-press verify cron_logs/AddPress.log
python3 -m fill_monitor add-press verify cron_logs/AddPress.log --verbose
```

### Cron Integration

In `LNFill_cron.sh`:
```bash
python3 -m fill_monitor add-press >> cron_logs/AddPress.log 2>&1 &
```

Runs concurrently with `LNFill_App.py` — does not interfere with the
detector fill process. Only writes to `LNT3_FV:EN` and `LNT6_FV:EN`
(spare tank external supply valves).

## Check-Pressure — Hourly Pressure Snapshot

Reads all 8 tank and supply pressure gauges, validates readings, writes
InfluxDB line protocol, and pushes to InfluxDB. Replaces `check_pressure.sh`.

```bash
# Production (cron, every hour at :07)
python3 -m fill_monitor check-press >> cron_logs/check_pressure.log 2>&1

# Write file but don't push to InfluxDB
python3 -m fill_monitor check-press --no-influxdb
```

## Command Flag Reference

| Subcommand | Flag | Description |
|------------|------|-------------|
| `adjust` | `--logfile FILE` | Fill log file to process (required) |
| `adjust` | `--filltimes FILE` | Path to gefilltime2.dat (default: gefilltime2.dat) |
| `adjust` | `--log-mon-dir DIR` | Log monitor directory (default: logs/fill_monitor) |
| `adjust` | `--state-file FILE` | State JSON path (default: DIR/fill_monitor_state.json) |
| `adjust` | `--no-discord` | Suppress Discord alert messages |
| `report` | `--logfile FILE` | Fill log file to classify (required) |
| `report` | `--log-mon-dir DIR` | Log monitor directory |
| `report` | `--state-file FILE` | State JSON path |
| `report` | `--no-discord` | Suppress Discord alert messages |
| `plot` | `--archive` | Use archived min_time history CSVs (no simulation) |
| `plot` | `--all` | Plot all detectors |
| `plot` | `GSIDs` | Specific GSID numbers to plot |
| `plot` | `--outdir DIR` | Output directory for plots |
| `plot` | `--logdir DIR` | Fill log directory |
| `plot` | `--filltimes FILE` | Path to gefilltime2.dat (sim mode) |
| `plot` | `--log-mon-dir DIR` | Log monitor directory (archive mode) |
| `plot` | `--keep-sim` | Retain simulation output after plotting |
| `add-press` | `--no-influxdb` | Suppress InfluxDB push at end of run |
| `add-press` | `--no-discord` | Suppress Discord alert messages |
| `add-press` | `--test` | Suppress InfluxDB + Discord, log to test file |
| `add-press` | `--log-mon-dir DIR` | Log monitor directory (for state file) |
| `add-press verify` | `LOGFILE` | AddPress.log file to verify against (required) |
| `add-press verify` | `--verbose` | Print every line during verification |
| `add-press verify` | `--manifold FILE` | Manifold transition log for exact verification |
| `check-press` | `--no-influxdb` | Write line protocol file but skip InfluxDB push |

## Discord Alerts

The fill monitor sends Discord alerts after each fill and on weekly rollover.
Two Discord channels are used:

- **#anomaly** (`discord_anomaly.WebHook`) — Warnings that need attention
- **#system-messages** (`discord.WebHook`) — Informational notices

### Per-Fill Alerts

| ID | Alert | Channel | Condition |
|----|-------|---------|----------|
| W1 | OVERTIME | #anomaly | Fill time limited by max_time. Notes fill status uncertain if overflow <5.6V. |
| W2 | Invalid/Duplicate GSIDs | #anomaly | Out-of-range (>120) or duplicate GSIDs in fill, with hose labels. |
| W3 | No temperature readback | #anomaly | Detector temp is None — check hose mapping. |
| W4 | Abnormal temperature | #anomaly | Temp ≤60K or >150K — sensor fault. |
| W-LP | Low tank pressure | #anomaly | Tank station below 18/15/12/9/6/3 psi during fill (from add-press). |
| W-WM | Warm manifold | #anomaly | Manifold temp sensor went Warm during fill — possible empty tank (from add-press). |
| N1 | Hose reconfiguration | #system-messages | Hose-GSID mapping changed from previous fill. |
| N2 | Temperature adjustment | #system-messages | Detector starting temp above average, min fill time adjusted. |
| N-LP | Low tank pressure (20 psi) | #system-messages | Tank station below 20 psi during fill (from add-press). |

### Weekly Alerts (Monday rollover)

| ID | Alert | Channel | Condition |
|----|-------|---------|----------|
| W5 | Chronic high voltage | #anomaly | Overflow LED sensors with ≥12 of 14 high-voltage events. Failed or intermittent sensor/wiring — needs repair. |
| W6 | Min time at ceiling | #anomaly | Detector min_time reached upper clamp limit. |
| N3 | Big movers | #system-messages | Detectors with \|delta\| > 60s during the week. |

### Suppression

```bash
# Suppress all Discord alerts
python3 -m fill_monitor adjust --logfile ${outfile1} --no-discord
python3 -m fill_monitor report --logfile ${outfile1} --no-discord
python3 -m fill_monitor add-press --no-discord
```

## Configuration

Fill times are stored in `gefilltime2.dat` (CSV: GSID, min_time, max_time).
All algorithm state is persisted in `logs/fill_monitor/fill_monitor_state.json`.
Per-fill min_time history is archived to `logs/fill_monitor/min_time_history/min_time_YYYYMM.csv`.

If `gefilltime2.dat` is lost, the adjuster automatically recovers the last
known min_times from the archive CSVs.
