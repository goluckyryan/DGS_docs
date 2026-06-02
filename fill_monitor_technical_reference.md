# Fill Monitor — Technical Reference

Complete algorithm documentation for the Gammasphere LN2 fill monitor system.
For high-level usage and integration, see [USER_GUIDE.md](USER_GUIDE.md).

## Contents

- [System Overview](#system-overview)
- [Physical Layout — LN2 Tank Stations](#physical-layout--ln2-tank-stations)
- [LED Sensor Infrastructure](#led-sensor-infrastructure)
- [Fill Sequence](#fill-sequence)
- [Components](#components)
- [Channel pre-warm](#channel-pre-warm)
- [Data Flow](#data-flow)
- [Input: Fill Log Format](#input-fill-log-format)
- [Classifier](#classifier)
- [Adjuster](#adjuster)
- [Reporter](#reporter)
- [Pressure Snapshot (check-press)](#pressure-snapshot-check-press)
- [Tank Monitoring (monitor-tanks)](#tank-monitoring-monitor-tanks)
- [Pressure Management (add-press)](#pressure-management-add-press)
- [Discord Alert Reference](#discord-alert-reference)
- [State File](#state-file)
- [Configuration](#configuration)
- [Directory Structure](#directory-structure)
- [InfluxDB Integration](#influxdb-integration)
- [Dependencies](#dependencies)
- [Potential Integrations](#potential-integrations)
- [Simulation and Plotting](#simulation-and-plotting)

---

## System Overview

The fill monitor is a Python package (`fill_monitor/`) that provides
automated monitoring, adjustment, and alerting for the Gammasphere LN2
fill system. It has three runtime contexts:

**Post-fill pipeline** (`python3 -m fill_monitor adjust`) — runs
sequentially after each fill (06:00 and 18:00 daily, plus emergency
M-fills every 15 minutes). Classifies anomalies, adjusts per-detector
fill times, writes weekly logs, and dispatches Discord alerts.

```
python3 -m fill_monitor adjust --logfile <fill_log>
  └→ fill_classifier  →  JSON state
  └→ fill_adjuster   →  gefilltime2.dat
  └→ fill_reporter   →  fillmon_YYYYMMDD.log + Discord
```

**Concurrent with fill — pressure management** (`python3 -m fill_monitor add-press`) —
runs alongside `LNFill_App.py` during each fill. Maintains tank pressure
(adaptive ext fill valve selection), monitors manifold temperatures,
performs automatic failover on empty tanks, logs to InfluxDB.

**Concurrent with fill — monitoring only** (`python3 -m fill_monitor monitor-tanks`) —
runs alongside `LNFill_App.py` during auto-fills (cron). Monitors manifold
temperatures and performs failover, but does NOT control ext fill valves.
Uses `--parent-pid $$` for lifecycle detection (no process name scanning).
Uses `--hard-timeout` as a safety net.  Three-phase startup:
  Phase 1: verify parent PID alive (instant) or skip if no `--parent-pid`.
  Phase 2a: wait for manifold valves to open (2s poll, 45s timeout).
            No EPICS pressure reads, no logging.  Exits silently if
            parent dies or timeout expires before any valve opens.
  Phase 2b: active monitoring (30s poll, 2s when warm detection active).
            Pressure logging starts here (first InfluxDB data point).
No fill = no log file, no InfluxDB data.
Replaces the old `add-press --no-pressure-mgmt` mode, which had a
termination bug (ran indefinitely).

```
python3 -m fill_monitor add-press
  └→ reads EPICS PVs → writes ext fill valve (adaptively selected)
  └→ monitors manifold temps → failover on Warm alarm
  └→ writes AddPress.txt → InfluxDB
  └→ writes JSON state + Discord
```

**Hourly snapshot** (`python3 -m fill_monitor check-press`) — reads all
8 pressure gauges and pushes to InfluxDB for Grafana trending.

```
python3 -m fill_monitor check-press
  └→ reads EPICS PVs → check_pressure.txt → InfluxDB
```

---

## Physical Layout — LN2 Tank Stations

The Gammasphere LN2 fill system consists of two identical tank stations,
four manifolds, an external LN2 supply, and 110 detector fill hoses.
All valve and sensor I/O is handled by Allen-Bradley (AB) PLC modules
via EPICS Channel Access.  The fill control software (`LNFill_App.py`)
and fill monitor (`fill_monitor` package) run on separate machines but
access the same EPICS PVs.

### Tank Stations

Each tank station has three 715-liter LN2 storage tanks and feeds
two manifolds.  Two external LN2 supply lines (one per station)
deliver bulk LN2 from a central dewar.

| Station | Tanks | Manifolds | External Supply |
|---------|-------|-----------|------------------|
| TS1 | Tank 1, Tank 2, Tank 3 (spare) | A, B | Supply S1 |
| TS2 | Tank 4, Tank 5, Tank 6 (spare) | C, D | Supply S2 |

Each tank has:
- **Feed valve** (`LNT{n}_FV:EN`) — controls flow between the tank
  and the manifold or external supply line.  Used during tank refill
  to admit LN2 from the external supply into the tank.
- **Vent valve** (`LNT{n}_VV:EN`) — vents gas from the tank during
  refill to allow liquid to flow in.
- **Temperature sensor** (`LNT{n}_TM:BT`) — reads Cold/Warm to
  detect when LN2 is flowing.

### Manifolds

Each manifold distributes LN2 from its feeding tank to up to 28
detector hoses.  Every manifold has three valve types:

| Valve Type | PV Pattern | Purpose |
|------------|------------|----------|
| Feed valve (main) | `LNM{n}_FV:EN` | Controls LN2 flow from the main tank to the manifold. Stays open for the full manifold fill duration (typically 18–25 min). |
| Feed valve (spare) | `LNM{n}A_FV:EN` | Controls LN2 flow from the spare tank to the manifold. Normally closed; opened by failover or ManID≥5 configuration. |
| Vent valve | `LNM{n}_VV:EN` | Primes/chills the manifold feed lines before detector filling begins. Open for ~3–5 min at fill start, then closed. |
| Temperature sensor | `LNM{n}A_TM:BT` | Reads Cold (LN2 flowing) or Warm (gas or no flow). Used for empty tank detection. |

The main-to-manifold feed path:

```
Tank 1 ──→ LNM1_FV:EN ──→ Manifold A ──→ LNH1-{01..28}_FV:EN ──→ Detectors
Tank 2 ──→ LNM2_FV:EN ──→ Manifold B ──→ LNH2-{01..28}_FV:EN ──→ Detectors
Tank 3 ──→ LNM1A_FV:EN ─→ Manifold A (spare path)
       └─→ LNM2A_FV:EN ─→ Manifold B (spare path)

Tank 4 ──→ LNM3_FV:EN ──→ Manifold C ──→ LNH3-{01..28}_FV:EN ──→ Detectors
Tank 5 ──→ LNM4_FV:EN ──→ Manifold D ──→ LNH4-{01..28}_FV:EN ──→ Detectors
Tank 6 ──→ LNM3A_FV:EN ─→ Manifold C (spare path)
       └─→ LNM4A_FV:EN ─→ Manifold D (spare path)
```

### External Supply

Two supply lines deliver LN2 from the bulk storage dewar:

| Supply | PV Prefix | Feeds | Pressure Gauge |
|--------|-----------|-------|----------------|
| S1 | `LNS1_` | TS1 (Tanks 1–3) | `LNP1-01_PR:AP` (Ext1) |
| S2 | `LNS2_` | TS2 (Tanks 4–6) | `LNP2-01_PR:AP` (Ext2) |

Each supply line has a vent valve (`LNS{n}_VV:EN`) used to prime/chill
the supply line before tank refilling begins.  Supply S2's priming vent
valve has been non-functional since late 2023.

### Pressure Gauges

Eight pressure transducers monitor the system:

| Gauge | PV | Location | Range |
|-------|-----|----------|-------|
| Ext1 | `LNP1-01_PR:AP` | External supply, TS1 side | 0–90 psi |
| Ext2 | `LNP2-01_PR:AP` | External supply, TS2 side | 0–90 psi |
| Tank1 | `LNP1-02_PR:AP` | Tank Station 1, Tank 1 | 0–45 psi |
| Tank2 | `LNP1-03_PR:AP` | Tank Station 1, Tank 2 | 0–45 psi |
| Tank3 | `LNP1-04_PR:AP` | Tank Station 1, Tank 3 (spare) | 0–45 psi |
| Tank4 | `LNP2-02_PR:AP` | Tank Station 2, Tank 4 | 0–45 psi |
| Tank5 | `LNP2-03_PR:AP` | Tank Station 2, Tank 5 | 0–45 psi |
| Tank6 | `LNP2-04_PR:AP` | Tank Station 2, Tank 6 (spare) | 0–45 psi |

Known failed gauges (as of 2026-05): Ext2, Tank5, Tank6.

### Detector Hoses

Each manifold serves up to 28 detector hose positions.  Each hose has:
- **Fill valve** (`LNH{m}-{nn}_FV:EN`) — controls LN2 flow to the
  detector dewar.
- **Overflow LED sensor** (`LNH{m}-{nn}_TM:BT` / `_TM:AT`) — detects
  when the dewar is full (reads Cold when LN2 overflows into the
  return line).  See [LED Sensor Infrastructure](#led-sensor-infrastructure)
  for sensor placement, EPICS interface, and voltage polarity
  (low V = warm, high V = cold, > 5.86 V = open fault).

Not all 28 positions are populated.  Empty positions have their
valves set to Disabled.  Currently ~70 detectors are active.

### GN Purgers

Two gaseous nitrogen purger circuits (G1, G2) are defined in the
`ln.inits` configuration but are **not physically installed** in the
current system.  They appear in the configuration files as legacy
entries and are not managed by the fill monitor.

### PV Naming Convention

All PVs follow a consistent pattern:

| Prefix | Device |
|--------|--------|
| `LNS{n}` | Supply line (n=1,2) |
| `LNT{n}` | Tank (n=1–6) |
| `LNM{n}` | Manifold main (n=1–4) |
| `LNM{n}A` | Manifold spare (n=1–4) |
| `LNH{m}-{nn}` | Detector hose (m=1–4 manifold, nn=01–28 position) |
| `LNP{s}-{nn}` | Pressure gauge (s=1,2 station, nn=01–04 gauge) |

Suffix conventions:

| Suffix | Purpose | Values |
|--------|---------|--------|
| `_FV:EN` | Feed valve enable/command | Open, Auto, Disabled |
| `_FV:VM` | Feed valve monitor (readback) | Open, Closed |
| `_VV:EN` | Vent valve enable/command | Open, Auto, Disabled |
| `_VV:VM` | Vent valve monitor (readback) | Open, Closed |
| `_TM:BT` | Temperature sensor (binary) | Warm, Cold |
| `_TM:AT` | Temperature sensor (analog voltage) | 0–6V |
| `_SM:SUB.E` | Fill timer (seconds, feed valves) | float |
| `_SM:SUB.D` | Fill timer (seconds, vent valves) | float |
| `_PR:AP` | Pressure (analog) | psi (float) |

**Note on `_FV:EN` vs `_FV:VM`:** The `_FV:EN` PV is both a command
and a readback — writing `Open` opens the valve, writing `Auto`
closes it, but reading `_FV:EN` returns the current physical valve
state (not the last command).  `_FV:VM` is a separate read-only
status register.  The fill monitor uses `_FV:EN` exclusively for
both reads and writes.

### ln2con Configuration

The real-time fill control system runs on `ln2con` (a VxWorks-era
controller, now CentOS with OpenSSH 5.3).  Two configuration files
define the system layout:

**`ln.inits`** — Device definitions:
```
#  type    name   id  station  supplier  max_customers  min_time  max_time  ...
Supply   S1      1   2         2         20             1200      5.00      ...
Tank     T1      1   1         1         3              3500      5.20      ...
Manifold M1      1   1         4         60             400       5.00      ...
Manifold M1A     5   0         1         100            400       5.00      ...
Detector H1-01  73   1         0         30             500       5.40      ...
```

Key fields: `id` is the device number, `station` maps to TS1/TS2,
`supplier` links devices in the supply chain (supply→tank→manifold→detector).
Manifold spare entries (M1A–M4A) have `id` 5–8; when `LNFill_App.py`
creates a `DetMan` with ManID ≥ 5, it swaps the main and spare feed
valves (spare becomes the primary fill path).

**`ln.state`** — Runtime state including fill times, sensor readings,
and the "Other" devices:
```
Other T1_FV      1    ...   — Tank 1 feed valve (external supply → tank)
Other T2_FV      2    ...   — Tank 2 feed valve
...
Other M1_FV      7    ...   — Manifold 1 feed valve (tank → manifold)
Other M2_FV      8    ...   — Manifold 2 feed valve
...
```

The "Other" category holds valves that are tracked for logging but
not directly managed by the fill sequence timers.

---

## LED Sensor Infrastructure

LN2-level detection throughout the fill system is implemented with
EPICS-monitored LED overflow sensors at every vent point.  Each
sensor combines an LED and a photodetector to read the optical
properties of the medium passing the sensor — warm gas vs. cold
liquid nitrogen.  The output is an analog voltage that EPICS smooths
and exposes as both an analog PV (`_TM:AT`) and a binary enum
(`_TM:BT`, Warm/Cold).

LED sensors confirm priming, dewar fill completion, and tank-level
overflow at the various points in the LN2 plumbing.  The classifier,
adjuster, and tank-monitor subsystems all consume LED data.

### Voltage Polarity

| State | Voltage range | What it means |
|-------|---------------|----------------|
| WARM | ~1.8 V | Sensor surrounded by warm gas (no LN2 present) |
| COLD | ~5.0–5.7 V | Sensor immersed in liquid LN2 |
| OPEN/FAULT | > 5.86 V | Sensor disconnected, dead, or open cable (false "cold") |
| SHORT/FAULT | < 1.5 V | Sensor shorted (false "warm", never registers cold) |

Note the polarity: **low voltage = warm, high voltage = cold**.
This is the opposite of an intuitive "temperature" reading.  A
healthy detector cycles between ~1.8 V (warm, between fills) and
~5.0–5.7 V (cold, during/just after a fill).  Voltages above the
OPEN threshold or below the SHORT threshold indicate sensor or
cable faults rather than thermal state.

### Physical Topology

The LN2 path runs from tank station → long feed line →
distribution manifolds at the array → individual detectors.
A vent path collects boil-off and overflow back to the tank
station and out of the building.  Each vent point in the system
has an associated LED sensor whose Cold reading confirms that
upstream plumbing is fully primed with liquid LN2.

```
                ┌──────────────────────────────────────────────┐
                │              Tank Station (TS1 or TS2)       │
                │                                              │
                │    [Main Tank]              [Spare Tank]     │
                │         │                        │           │
                │     [Main MFV]              [Spare MFV]      │
                │         └────────────┬───────────┘           │
                │                      ↓ [LNMnA_TM:AT]         │
                │      ┌── feed line ──┘   (tank-station LED)  │
                │      │                                       │
                │      │              Tank (x3) and EXT Vents  │
                │      │                       ↓  ↓  ↓  ↓      │
                │      │                  ┌────┴──┴──┴──┴────┐ │   to main vent
                │      │                  │ vent collection  │─┼─→ outside building
                │      │                  └─────────┬────────┘ │
                │      │                            ↑          │
                │      │                            │          │
                └──────┼────────────────────────────┼──────────┘
                       │                            ↑
                       │                      Vent Collection←──── 2 vent lines from
                       │                            ↑              Manifold B
                       │ long feed line             │
                       │                            │ 2 long vent lines
                       │                            │ from Manifold A
                       │                            └──────┐ 
                       ↓                                   │ 
       ┌───────────────┼──────────────────────────────┐    │
       │    Manifold A (paired manifolds at array)    │    │
       │               │                              │    │
       │   ┌─── Supply Manifold ───────────────────┐  │    │
       │   │  ┌───┬───┬───┬─── ... ───┬───┬─────┐  │  │    │
       │   │  │SV1│SV2│SV3│           │SV28│Vent│  │  │    │
       │   │  │   │   │   │           │    │ VV │  │  │    │
       │   │  └─┬─┴─┬─┴─┬─┴─── ... ───┴─┬──┴──┬─┘  │  │    │
       │   └────┼───┼───┼───────────────┼─────┼────┘  │    │
       │        │   │   │               │     │       │    │
       │        │ (detector feed lines) │  (vent      │    │
       │        │                       │   valve     │    │
       │        ↓   ↓   ↓               ↓   line)     │    │
       │   [Inj→Dewar→returns: pressure-driven during │    │
       │    fill, always open to atmosphere on vent   │    │
       │    side]                                     │    │
       │        │   │   │               │     │       │    │
       │   ┌────┼───┼───┼───────────────┼─────┼─────┐ │    │
       │   │    ↓   ↓   ↓               ↓     ↓     │ │    │
       │   │   LED LED LED ...         LED   LED    │ │    │
       │   │   1   2   3               28   (mani.) │ │    │
       │   │ [LNHm-vv_TM:AT]              [LNMn_TM] │ │    │
       │   │    ↓   ↓   ↓               ↓     ↓     │ │    │
       │   │    └───┴───┴────── ... ────┴─────┘     │ │    │
       │   │         Vent Manifold (collective)     │ │    │
       │   │                  ↓                     │ │    │
       │   │           2 vent lines ────────────────┼─┼────┘
       │   │           back to tank station         │ │
       │   └────────────────────────────────────────┘ │
       └──────────────────────────────────────────────┘
```

Each manifold pair (M1/A through M4/D) consists of two physically
separate manifolds at the array:

- **Supply manifold:** 28 detector supply solenoid valves
  (`LNH{m}-{nn}_FV:EN`) + 1 manifold vent valve (`LNM{n}_VV:EN`,
  "position 29") used for priming.  No LEDs on the supply side.
- **Vent manifold (return collective):** 29 LEDs total — one at
  each detector return-line junction (28) and one at the manifold
  vent valve's connection (1).  Vent collective drains via 2
  long vent lines per manifold back to the tank station, then
  through the tank station vent collection out of the building.

The 4 long vent lines from the two manifold pairs (2 per manifold)
merge at a junction point outside the tank station, then enter the
station as a single combined line.  Inside the station, that combined
vent joins the three tank vents and the EXT supply vent at the vent
collection, which then exits to the main vent line out of the
building.  Each tank station handles 4 manifold vent lines, for 8
across the system.  Only one manifold pair (Manifold A) is shown
in detail in the diagram above; Manifolds B (same tank station) and
C/D (other tank station) have identical structure.

The tank station also has its own LEDs for tank-level and external
supply line monitoring (see [LED PV Map](#led-pv-map) below).

### Operational Sequence

**Priming (start of fill):** Manifold vent valve (`LNM{n}_VV`)
OPEN, all 28 detector supply SVs CLOSED.  LN2 flows
tank → MFV → feed line → supply manifold → manifold vent valve →
vent manifold.  When the manifold LED (`LNM{n}_TM:AT`) reads Cold,
the feed line is fully primed with liquid LN2 and detector fills
may begin.  Operating detectors before the line is primed will
blow gas through the dewars and waste LN2.

**Detector fill:** Manifold vent valve CLOSED, supply SV for the
active detector OPEN.  Tank pressure pushes LN2 through:
tank → MFV → feed line → supply manifold → detector supply SV →
detector feed line → injector → dewar.  Boil-off gas and overflow
LN2 exit the dewar under fill pressure: dewar → detector return
line → vent manifold (at that detector's LED point) → vent line →
tank station → main vent → outside.  When liquid LN2 (not just
vapor) reaches the detector LED, voltage rises above the Cold
threshold and `LNFill_App` closes the supply solenoid.

**Idle (not filling):** All valves return to Auto.  Dewars remain
open to ambient through the return lines and vent manifold.  No
gravity drain — the system equalizes to ambient pressure via the
always-open vent path.

### LED PV Map

The system has 128 LED PVs in total.  Each is exposed as both an
analog voltage (`_TM:AT`) and a binary enum (`_TM:BT` with values
Warm/Cold/Fault/"No Data").

| PV pattern | Count | Location | Cold reading means |
|------------|-------|----------|---------------------|
| `LNT{n}_TM:AT` (n=1–6) | 6 | Tank vent valve | Tank is full (overflow detected) |
| `LNS{n}_TM:AT` (n=1,2) | 2 | External supply line vent valve | External feed line from outside building is primed all the way to tank fill valves |
| `LNM{n}A_TM:AT` (n=1–4) | 4 | Inside tank station, at junction where Main+Spare feeds combine into the manifold feed line | LN2 flowing out of tank station into feed line |
| `LNM{n}_TM:AT` (n=1–4) | 4 | Distribution manifold vent valve | Manifold feed line primed (cold LN2 reached the distribution manifold) |
| `LNH{m}-{vv}_TM:AT` (m=1–4, vv=01–28) | 112 | Vent manifold, one per detector at the return-line junction | Detector dewar is full (LN2 in return line) |

Each vent point in the LN2 path has an LED.  The naming convention
makes the role explicit: the PV prefix identifies the device, the
LED PV is co-located with that device's vent valve, and a Cold
reading confirms the plumbing upstream of that vent valve is fully
primed.

### Companion Valves

LED sensors are paired with the valves that control flow at the
same point.  Both feed valves and vent valves use the same `_EN`
(command/state) and `_VM` (monitor readback) conventions.

| Valve PV | Role |
|----------|------|
| `LNT{n}_FV:EN/VM` | Tank Fill Valve (external supply → tank, n=1–6) |
| `LNT{n}_VV:EN/VM` | Tank Vent Valve (n=1–6) |
| `LNS{n}_VV:EN/VM` | External supply line Vent Valve (n=1,2) |
| `LNM{n}_FV:EN/VM` | Manifold Main Feed Valve / MFV (tank → manifold, n=1–4) |
| `LNM{n}A_FV:EN/VM` | Manifold Spare Feed Valve (n=1–4) |
| `LNM{n}_VV:EN/VM` | Manifold Vent Valve / priming valve at distribution manifold (n=1–4) |
| `LNH{m}-{vv}_FV:EN/VM` | Detector supply solenoid (m=1–4, vv=01–28) |

See [PV Naming Convention](#pv-naming-convention) for the full
prefix/suffix table.

### EPICS Interface

LED sensors are exposed via the standard EPICS `ai` (analog input)
record type.  The IOC at `lnfill.onenet:5064` handles record
processing and provides the smoothed values consumed by fill
monitor.

**Update rate:** 1 Hz.  Records are processed by the IOC every
second.  Verified empirically via `caget` timestamps.

**Records per LED:**
- `_TM:AT` — analog voltage, smoothed by SMOO (see below).
- `_TM:BT` — binary enum, derived from `_TM:AT` via thresholds.
  Values: `No Data`, `Fault`, `Warm`, `Cold`.

**Cold/Warm thresholds:** Defined per-device in the `ln.inits`
configuration file on `ln2con` (`/home/gamop/lnfill_logs/ln.inits`,
column `temperature_threshold`).  The IOC compares the smoothed
`_TM:AT` value to the device's threshold and sets `_TM:BT`
accordingly.  Thresholds are NOT defined in Python — fill_monitor
reads the `_TM:BT` enum directly.  See `ln.inits` for the file
format (columnar; the file header documents each column).
Detector default is currently 5.40V with per-detector 0.10V
adjustments documented in the file header for problem detectors.

**Smoothing:** Each LED `_TM:AT` record has `SMOO=0.9` configured.
SMOO is the standard EPICS `ai`-record smoothing field — it
implements a first-order IIR (infinite impulse response) low-pass
filter on the raw ADC reading:

```
VAL = VAL * SMOO + (1 - SMOO) * NewData
```

SMOO ranges from 0 to 1 (0 = no smoothing, 1 = infinite smoothing,
i.e. value never updates).  The equivalent continuous-time time
constant is:

```
τ = −T / ln(SMOO)
```

where T is the record scan period.  At the configured SMOO=0.9
and T=1s, τ ≈ 9.5s.  The filter slows the LED voltage response
to abrupt physical or electrical changes.

Reference: https://epics.anl.gov/base/R7-0/8-docs/aiRecord.html

### SMOO and Open-Sensor Behavior

Because SMOO low-pass filters the LED voltage, an LED that
electrically fails open does NOT jump instantly to the rail.
Instead, the smoothed voltage climbs monotonically from its
previous value (typically the warm baseline, ~1.8V) toward the
open rail (~5.9V) over several time constants.

The path of the smoothed voltage during this transit crosses the
Cold threshold (somewhere in the 5.0–5.7V range, depending on
`ln.inits`) on the way up.  When that happens, the IOC's `_TM:BT`
enum briefly reports `Cold` even though the sensor is faulted
and no actual LN2 is present.  `LNFill_App` interprets this as
the dewar reaching its fill state and records a `cold_time`.

Consequences:
- An open-LED detector typically shows a short, *consistent*
  `cold_time` value every fill (set by the SMOO time constant
  and the threshold position).  This consistency, combined with
  `ovf_end > 5.86V` and `ovf_begin > 5.86V`, is the classifier's
  evidence for an OPEN fault.
- The system reports cold *crossings*, not open states directly.
  A sensor that reads cold at fill end may either be genuinely
  cold (`ovf_end ~5.0–5.7V`) or faulted open (`ovf_end > 5.86V`).
  Polarity-aware classification distinguishes the two.

This filter behavior is also why the open-sensor voltage looks
like a slow ramp in `--log-led` traces, never an instantaneous
step.

### Distinguishing Failure Modes

LED voltage patterns across many fills are informative beyond a
single fill's BAD/OK verdict.  Common signatures:

| Pattern | Likely cause |
|---------|--------------|
| Constant rail (~5.90V), zero variation | Cable or sensor fully open |
| Brief partial response during fill, returns to rail | Degraded sensor still responding to some LN2 contact |
| Wild oscillations between Warm and Cold ranges | Intermittent cable contact OR intermittent LN2 flow |
| Increasing `fill_time` / `min_time` over weeks | Injector icing, progressive |
| Sudden change after stable history | Real fault (newly broken) |

OPEN-style failures (constant rail, partial-response) are sensor
or cable problems and require physical repair.  Flow-related
patterns (oscillations, increasing fill time) indicate icing or
manifold issues and are caught by other parts of the system
(weekly min-time delta, clamped min-times, OVERTIME events).

### OVERTIME — When the LED Never Reads Cold

When a detector's overflow LED never reads Cold within the
configured `max_time`, `LNFill_App` records the fill as
OVERTIME.  About 120 of 121 OVERTIME events are caused by
**frozen injectors** (ice buildup in the supply line restricting
LN2 flow), NOT broken hoses or sensors.

Icing accumulates slowly over weeks.  The fill monitor's existing
weekly-report signals serve as the icing alert — no separate
detection is needed:
- **Clamped Min-Times** — detector hit the 360s clamp.
- **OVERTIME events** — listed in weekly log.
- **Large positive delta in weekly Min-Time Summary** — e.g. +50s
  in a week from repeated temp_override firing.  Strong icing
  signature, though other causes exist (recent reinstall, neighbor
  effects).

Operators typically get a few more fills after clamping/large-delta
before warmup is required.

---

## Fill Sequence

A complete fill cycle has three phases: manifold priming, detector
filling, and tank refilling.

### Phase 1 — Manifold Priming (~3–5 minutes)

When a fill starts, `LNFill_App.py` opens the manifold feed valve
(`LNM{n}_FV:EN → Open`) and the manifold vent valve
(`LNM{n}_VV:EN → Open`) simultaneously for each manifold being filled.

The vent valve allows warm gas to escape from the manifold feed lines
while cold LN2 from the tank chills them down.  The manifold
temperature sensor (`LNM{n}A_TM:BT`) transitions from Warm to Cold
as liquid reaches the manifold.  The vent valve closes when either:
- The temperature sensor reads Cold (priming complete), or
- The vent timeout expires (300s default at AGFA)

If the vent valve stays open beyond 600s (vent_fail_time), the fill
for that manifold is aborted — the vent line may be displaced.

The feed valve remains open — it does NOT close when the vent closes.

**ln_log entries during priming:**
```
260524 06:00:18 MANIFOLD M1  AUTO to OPEN.       ← vent valve opens
260524 06:00:18 Other MANIFOLD M1_FV  AUTO to OPEN.  ← feed valve opens
260524 06:03:25 MANIFOLD M4  OPEN to AUTO.        ← vent valve closes (Cold or timeout)
```

### Phase 2 — Detector Filling (~15–25 minutes)

Once the vent valve closes (manifold is primed), detector filling
begins.  Up to 4 detectors are filled simultaneously per manifold.
`LNFill_App.py` opens detector valves (`LNH{m}-{nn}_FV:EN → Open`)
and monitors each detector's overflow LED sensor:

- **NORMAL**: LED goes Cold after min_time → valve closes
- **EXTENDED**: LED goes Cold before min_time → valve stays open
  until min_time expires
- **OVERTIME**: LED never goes Cold by max_time → valve forced closed

As each detector completes, the next one in the queue opens.  The
manifold feed valve stays open throughout this phase.

When all detectors on a manifold are complete, the manifold feed
valve closes:
```
260524 06:22:46 Other MANIFOLD M1_FV  OPEN to AUTO.  ← feed valve closes
```

After all manifold threads complete, `LNFill_App.py` calls
`CloseAllValves()` to ensure everything is closed (safety net).

### Phase 3 — Tank Refilling (~20–30 minutes)

After detector filling, the depleted tanks are refilled from the
external supply.  `LNFill_App.py` spawns `TankMan` threads for each
tank station:

1. **Supply priming** — Opens the supply vent valve (`LNS{n}_VV:EN`)
   to chill the supply line.  The temperature sensor
   (`LNS{n}_TM:BT`) transitions Warm→Cold when liquid reaches the
   station.  TS2's supply vent valve has been non-functional since
   late 2023.

2. **Tank filling** — Opens each tank's feed valve
   (`LNT{n}_FV:EN → Open`) and vent valve (`LNT{n}_VV:EN → Open`).
   LN2 flows from the external supply into the tank; the vent valve
   releases displaced gas.  The vent valve's temperature sensor
   detects when the tank is full (Cold = liquid overflowing into
   vent line).  Typical fill times: 1300–1900s per tank.

3. **Spare tank** — Tank 3 (TS1) and Tank 6 (TS2) are refilled last,
   with shorter timeout (400–500s for spare vs 3500–4000s for mains).

**ln_log entries during tank refill:**
```
260524 06:23:48 Other TANK T4_FV  AUTO to OPEN.   ← tank 4 feed valve opens
260524 06:23:49 Other TANK T5_FV  AUTO to OPEN.   ← tank 5 feed valve opens
260524 06:46:44 Other TANK T4_FV  OPEN to AUTO.   ← tank 4 full
260524 06:46:45 Other TANK T6_FV  AUTO to OPEN.   ← spare tank 6 starts
260524 06:55:46 Other TANK T5_FV  OPEN to AUTO.   ← tank 5 full (last)
```

### Concurrent Operations

During detector filling (Phase 2), one of two background processes
runs alongside `LNFill_App.py`, depending on the fill type:

**Scheduled fills (F-fills, 06:00 and 18:00 via `LNFill_cron.sh`):**

`add-press` (`fill_add_press.py`) runs in the background.  add-press
is built on top of the monitoring layer (`fill_tank_monitor.py`) and
provides the full set of concurrent functionality:
- Ext fill valve pressure management (opens/closes `LNT{n}_FV:EN`)
- Manifold warm detection via `ManifoldTempTracker`
- Automatic feed failover via `ManifoldFailover`
- Low tank pressure alerts via `LowPressureTracker`
- Pressure gauge logging to InfluxDB
- Adaptive ext fill valve selection (deconfliction)
- Lifecycle signaling (`addpress_started`/`addpress_finished`)

**Auto fills (M-fills, every 15 min via `LNFill_Auto_EFill_cron.sh`):**

`monitor-tanks` (`fill_tank_monitor.py`) runs in the background.  It
provides manifold monitoring and failover but does NOT manage ext
fill valve pressure:
- Manifold warm detection via `ManifoldTempTracker`
- Automatic feed failover via `ManifoldFailover` (feed switching
  active, but no ext fill valve management)
- Low tank pressure alerts via `LowPressureTracker`
- Pressure gauge logging to InfluxDB
- Three-phase lifecycle: PID check → wait for manifold valves →
  active monitoring
- No lifecycle signaling (check-press not affected)

add-press and monitor-tanks never run simultaneously — concurrency
guards ensure add-press terminates any running monitor-tanks instance
on startup, and monitor-tanks yields to any existing add-press.

**Valve write policy:**

| Process | Writes to | Never writes to |
|---------|-----------|------------------|
| add-press | `LNT{n}_FV:EN` (ext fill) | `LNM*_FV:EN` (manifold feed) except during failover |
| monitor-tanks | `LNM*_FV:EN` / `LNM*A_FV:EN` during failover only | `LNT{n}_FV:EN` (ext fill) |
| LNFill_App | `LNM*_FV:EN`, `LNM*A_FV:EN`, `LNH*_FV:EN` (all fill valves) | `LNT{n}_FV:EN` (ext fill) |

Neither add-press nor monitor-tanks interferes with detector filling.
LNFill_App does not re-open a manifold feed valve once closed mid-fill,
so failover's `Auto` write is safe.  LNFill_App only closes spare feed
valves in its final `CloseAllValves()` after all manifold threads complete.

### Ext Fill Valve Closeout Timing

When a tank station finishes filling (all manifold feed valves on
that station close), `step()` closes the ext fill valve for that
station on the next iteration.  `_interruptible_sleep()` detects
the filling→not-filling transition within ~1 second via cached PV
reads, waking the main loop immediately.  This ensures the ext
fill valve is closed well before LNFill_App's TankMan begins the
tank refill phase (~46s for main tanks, ~20 min for spare).

**Note:** Add-press's pressure management activity during the
detector fill phase pre-chills the ext feed line with cold N2 gas
and possibly LN2 flow.  This means LNFill_App.py's feed line
priming phase (prior to tank fill) can complete nearly
instantaneously — the line is already cold and possibly
liquid-filled rather than requiring a full cooldown from ambient.

### Typical Timeline (F-fill)

```
06:00:00  LNFill_cron.sh starts
06:00:09  add-press starts (background)
06:00:10  LNFill_App.py initiates F-fill
06:00:18  Manifold valves open (feed + vent simultaneously)
06:03–05  Vent valves close (manifolds primed, ~3–5 min)
06:03–05  Detector filling begins (up to 4 per manifold)
06:18–23  Manifold feed valves close (all detectors complete)
06:23–24  add-press exits (manifolds closed)
06:23–24  Tank refill begins (T1–T5 from external supply)
06:24     addpress_finished written → check-press high-rate sampling
06:46–55  Tank refill complete
06:55     LNFill_App.py exits (total runtime ~55 min)
```

### ln_log Entry Types

The `ln_log` on `ln2con` records all valve transitions.  Entry types:

| ln_log prefix | Device | Valve | PV |
|---------------|--------|-------|-----|
| `MANIFOLD M{n}` | Manifold | Vent valve | `LNM{n}_VV:EN` |
| `Other MANIFOLD M{n}_FV` | Manifold | Feed valve | `LNM{n}_FV:EN` |
| `DETECTOR H{m}-{nn}` | Detector | Fill valve | `LNH{m}-{nn}_FV:EN` |
| `TANK T{n}` | Tank | Vent valve temp sensor | `LNT{n}_TM:BT` |
| `Other TANK T{n}_FV` | Tank | Feed valve | `LNT{n}_FV:EN` |
| `SUPPLY S{n}` | Supply | Vent valve temp sensor | `LNS{n}_TM:BT` |

The "Other" prefix denotes devices tracked in the `ln.state` Other
section.  These are physical valves that operate alongside (but
independently from) the primary device.  For example, `MANIFOLD M1`
(vent valve) and `Other MANIFOLD M1_FV` (feed valve) open
simultaneously at fill start, but the vent valve closes after 3–5
minutes while the feed valve stays open for 18–25 minutes.

### AB Module Address Map

Valve I/O is distributed across Allen-Bradley PLC modules.  The
`ln_log` records the AB address for each transition as
`#L0 A{a} C{c} S{s}` (adapter, card, slot):

| AB Address | Devices |
|------------|----------|
| A1 | Temperature sensors (supply, tank) |
| A2 | Vent valves (manifold, detector valves) |
| A3 C1 | TS1 valves: manifold feed (M1_FV, M2_FV), tank feed (T1–T3_FV) |
| A3 C3 | TS2 valves: manifold feed (M3_FV, M4_FV), tank feed (T4–T6_FV) |

---

## Components

Nine Python modules make up the package. Each has a distinct
responsibility and communicates with others through the shared JSON
state file. The `fill_interfaces` module provides shared I/O
infrastructure (EPICS, Discord, InfluxDB) imported by all other modules.

### fill_classifier.py

Parses fill logs, classifies per-detector events, computes temperature
means. Owns: history, known_detectors, last_fill_time,
classifier weekly_log_sections sub-keys (including detector_changes),
current_fill_alerts, pending_weekly_summary.
`parse_fill_log()` extracts fill type, timestamp, per-detector data,
and completion status (checks for "Total App Runtime" footer).
For bad temperature readbacks, stores both `temp_begin` and `temp_end`
so the reporter can determine severity (transient vs persistent).

Also provides `check_missing_log()` [ML-1] and `check_fill_completeness()`
[IL-2/IL-3] as standalone functions. These are called by both `adjust`
and `report` commands before any adjustment or reporting. `check_missing_log`
fires when the log file doesn't exist or was unparseable (parsed_log is None);
suppressible via `--no-missing-log-alert` (used by AUTO cron for M-fills
that often produce no log).

Does NOT touch gefilltime2.dat. Not sim-aware. Has no knowledge of
Discord, InfluxDB, or any output mechanism — never imports the reporter.
Only appends structured data to JSON; all output belongs to the reporter.

### fill_adjuster.py

All min_time adjustment logic, tuning parameters, and per-detector state
management. Owns: temp_adj, consec_*, last_temp_trigger, last_decay_time,
last_fill_floor, clamp_first_seen, adjuster weekly_log_sections sub-keys.
Owns gefilltime2.dat. Sim-aware (fill_open computation, cold_time proxy).
Also owns: missing fill log detection [ML-1] (via check_missing_log from
classifier — fires before check_fill_completeness), incomplete fill log
detection [IL-2/IL-3] (checks for completion footer), per-fill clamp
ceiling alerts (writes to current_fill_alerts['clamped']).

### fill_reporter.py

Sole owner of all output: weekly log files and Discord alerts. Two modes:
classifier-only (anomaly sections) vs full (all 10 sections). Reads
structured data from JSON, dispatches, and clears consumed keys:

- `current_fill_alerts` → per-fill Discord (W1-W4, W6, N2, N4), cleared after send
- IL alerts fired by `check_fill_completeness` (called by both adjust and report commands)
- `pending_weekly_summary` → weekly Discord (W5, N3), cleared after send
- `weekly_log_sections` → weekly log file (accumulates all week)

All Discord dispatched via `fill_interfaces` wrappers
(which delegate to `WriteDiscordMessage.py`).

### fill_plotter.py

Per-detector fill history plots. Two modes: archive (reads min_time
history CSVs) or simulation replay (drives full adjuster through all
historical logs).

### fill_prefill_check.py

Pre-fill temperature safety check (V2.1). Before a scheduled fill,
reads live detector temperatures from EPICS, compares against historical
mean from state file, and temporarily bumps min_time in gefilltime2.dat
for detectors running hot. The bump is relative to the detector's last
fill time (not current min_time). Saves the original min_times to
`prefill_backup` in the JSON state so the post-fill adjuster can use
them as its baseline. Sends a Discord notice (N-channel) for each bump.
Does not charge budget or modify any adjuster state — the post-fill
adjuster handles the full correction.

Gates on valve state (Auto), not DV_EN. Reads `LNH{m}-{v}_FV:EN` PVs
alongside manifold assignment PVs during the hose mapping phase. Only
detectors whose valve is in AUTO state are processed for temp bumps or
hose reassignment — this matches LNFill_App's gating behavior.
Also classifies every detector against the 12-row
**Prefill detector state matrix** (PFM-1..5 + PFM-OPEN); see
USER_GUIDE.md “Prefill check — detector state matrix” for the
full table.  Each non-silent matrix row dispatches one per-detector
message to the routed channel (#anomaly for RED / YELLOW /
anomalous-info; #system-messages for system-info).

The classifier also extracts manifold vent-valve data (MAN: lines with
valve_type='Vent-Valve') and stores it in current_fill_alerts for the
reporter to check:
  - [VV-1/VV-2] ovf_end voltage thresholds (warm/marginal priming)
  - [VV-3] Bad LED (ovf > 5.86V at start/end/both, same threshold as detectors)
  - [VV-4] Short priming (< 60s, sensor may be bad)

### BrokenLineDetector (in fill_tank_monitor.py)

Broken manifold feed line detection (SAFETY).  Merged into
fill_tank_monitor.py alongside ManifoldTempTracker, ManifoldFailover,
and LowPressureTracker.

Uses a **4-of-6 sliding window** of recent detectors per manifold
(in fill order) to detect broken/detached feed lines.  When 4 or more
of the last 6 eligible detectors show no LN2 flow (LED < 2.0V after
180s open), the feed valve is closed and a RED alert [BL-1] fires.

**Algorithm:**

1. Monitors all 28 detector valve states (LNH{m}-{v}_FV:VM) each poll.
   Tracks fill duration internally via time.time() (no SM:SUB.D reads).

2. When a valve has been open >= 180s, its LED overflow voltage
   (LNH{m}-{v}_TM:AT) is read lazily and a verdict assigned:
   LED >= 2.0V = flow, LED < 2.0V = no-flow.

3. Maintains a history of the last ~12 eligible detectors per manifold
   (fill time >= 180s).  Active detectors (valve open) have live
   verdicts that update each poll cycle.  Finalized detectors (valve
   closed) have locked verdicts.

4. On every poll cycle, rebuilds the sliding window: walks history
   (most recent first), picks the first 6 with valid LED voltage
   (1.60V–5.86V).  Detectors with invalid LED are excluded entirely.

5. Counts no-flow verdicts in the window.  If count >= 4, fires
   immediately — does not wait for valve close.

**Window eligibility:**
- Fill time >= 180s (either still open past 180s, or closed after >= 180s)
- Valid LED voltage (1.60V–5.86V at current reading for active,
  at close time for finalized)
- Detectors with < 180s fill or invalid LED are invisible (no slot consumed)

**Dynamic window behavior:**
- Active detectors can flip verdict (flow ↔ no-flow) each poll cycle
- Detectors can drop out of window if LED goes invalid mid-fill
- Older history detectors slide back in to fill vacancies
- Detectors that dropped out can re-enter if LED returns to valid range

The manifold cold sensor is upstream of a potential break point and stays
cold — it cannot detect a broken feed line.  The only observable is that
detector LED sensors show no cooling (voltage stays at room temp ~1.8V).

PV reads per iteration: 28 FV:VM (valve states, pre-warmed at fill
detection) + 0-N TM:AT (LED voltage, lazy, only for valves open > 180s).
No SM:SUB.D reads — fill time tracked internally.

**Active.**  A confirmed broken-feedline verdict closes the manifold's
main feed valve (``LNM{x}_FV:EN → 'Auto'``, i.e. lets the AB
controller put it in auto/closed state) and sends the RED Discord
alert via ``run_monitoring()``'s ``caput_fn=_caput`` and
``discord_fn=send_discord_anomaly`` wiring
(``fill_tank_monitor.py:559-571``).

Kill switch: ``--no-feedline-check``.  When set, no detector LED
or valve PV reads happen and no alerts fire (the detector is
skipped entirely — ``bl_detector`` is constructed but never updated).
Use this as the rollback path if BL-1 misbehaves; no code change
required.

**Failover-cooldown suppression.**  ``ManifoldFailover`` and
``BrokenLineDetector`` are wired together by
``failover.attach_bl_detector(bl_detector)`` in both
``run_monitor_tanks`` and ``run_add_press``.  When
``ManifoldFailover._switch_feed`` (main↔spare) or ``_close_feed``
(both feeds exhausted) successfully completes, it calls
``bl_detector.suppress(manifold_id)``, which sets
``BrokenLineDetector._suppress_until[manifold_id]`` to
``time.time() + BL_FAILOVER_SUPPRESS_SECONDS`` (240s).  While
suppressed, ``BrokenLineDetector.update(manifold_id, ...)`` early-
returns ``False`` without rebuilding the sliding window or reading
LED voltages, and logs one stderr line per poll:

.. code-block:: text

   [BL-SUPPRESS] Manifold A: skipped (failover cooldown, Ns)

Scope is **per-manifold**, matching ``ManifoldFailover``'s
per-manifold decision matrix.  Failover on Manifold A does not
suppress B even though both are on TS1 — TS1/TS2 grouping in
``ManifoldFailover`` only applies to ext fill valve selection.
History is left intact so post-mortem debug can still see the
window state that existed at failover time.  The link is null-safe:
when ``--no-feedline-check`` leaves ``bl_detector = None``,
``attach_bl_detector(None)`` (or skipping the attach call entirely)
stores ``None`` in ``_bl_detector`` and the ``suppress()`` hook
becomes a silent no-op.

Why 240s and not 180s (which would match the
``NO_FLOW_CHECK_TIME`` eligibility threshold): a detector whose
valve was already open longer than 180s at the moment of failover
is already eligible for the window by the normal rule.  An extra
60s past the next eligibility crossing gives the just-opened
target feed time to wet the line and let cold gas reach the LED
before BL-1 starts counting that detector.  Why not pre-emptively
suppress on Warm detection (before failover): a Warm event can
fire without an actual feed switch (e.g.
``failover_enabled=False``, target valve Disabled returning FO-1 /
FO-3 RED alert).  In those cases the feed line is genuinely
steady-state and BL-1's verdict is still valid — only suppress
when a feed switch actually happens.

### BrokenPrimingDetector (in fill_tank_monitor.py) — BL-2 + BL-3

Complementary to ``BrokenLineDetector`` (BL-1).  BL-1 catches
broken feed lines DURING detector filling (4-of-6 sliding window,
12-18 min latency).  This detector catches them AT THE END OF
PRIMING by sampling the manifold vent LED at vent close.  See


**Two output tiers from a single trigger:**

* **BL-2 (RED, broken line)** — ``max_vent_led < BL2_BROKEN_LINE_THRESHOLD``
  (2.0V) at vent close.  Fires RED Discord and (when activated)
  closes the manifold's main feed valve.  Initially deployed
  INERT: Discord still fires but with a ``[BL-2 IN DEVELOPMENT —
  NO ACTION TAKEN]`` prefix and no caput.

* **BL-3 (YELLOW, unprimed)** —
  ``BL2_BROKEN_LINE_THRESHOLD ≤ max_vent_led < BL3_UNPRIMED_THRESHOLD``
  (2.0–3.0V).  Fires YELLOW Discord only.  Never closes valves,
  never tagged with ``[BL-N]`` in operator-facing text.

* **Silent** — ``max_vent_led ≥ BL3_UNPRIMED_THRESHOLD`` (3.0V).
  Healthy prime; post-fill VV-1 / VV-2 cover that range with any
  needed informational reporting.

BL-2 and BL-3 share one detector instance, one state machine, one
allowed run per fill cycle (mutual exclusion at the dispatch site).

**Algorithm (per manifold, one-shot per fill):**

*Arm condition:* the manifold vent valve (``LNM{n}_VV:VM``)
transitions ``Auto`` → ``Open`` AND the TS manifold LED
(``LNM{n}A_TM:BT``) reads ``'Cold'`` at that moment.  The
TS-LED-Cold gate ensures the detector only judges fills where LN2
was actually flowing out of the tank station; empty-tank failures
stay under ``ManifoldFailover``'s jurisdiction.  Only the FIRST
``Auto`` → ``Open`` per fill arms; expert-operator re-opens are
ignored.

*Single trigger — vent valve close:* when the vent valve transitions
``Open`` → not-Open, evaluate the maximum vent LED voltage observed
during the armed window and dispatch to the appropriate tier per
the table above.  No during-priming check.  Historical vent-LED
data shows the LED holds at ~1.85V baseline for the first 80% of
priming and only ramps in the final 20%, so any during-priming
threshold would either fire falsely on healthy slow primes or be
so loose it adds no early-warning value.  Vent close is the only
physically meaningful evaluation point.

*Disarm conditions* (any one ends evaluation for that manifold's
fill cycle, no re-arm until next fill):

- Vent valve closes naturally (also triggers the tier evaluation
  above)
- Any detector valve on the manifold opens (= detector-fill phase
  started; BL-1 takes over)
- TS manifold LED transitions ``'Cold'`` → ``'Warm'`` (= tank
  emptied; ``ManifoldFailover`` taking action)
- ``ManifoldFailover._switch_feed`` or ``_close_feed`` succeeds
  for this manifold (redundant safety net — the failover code
  calls ``bl2_detector.suppress(manifold_id)`` explicitly)

**Critically:** if the detector is disarmed (``finalized=True``)
AT the moment of vent close, NEITHER tier fires regardless of
``max_vent_led``.  The vent-close trigger short-circuits via the
early-return on ``s['finalized']``.  The manifold is locked out
for the rest of the fill cycle (no
re-monitoring after vent close, no re-arm if vent re-opens).

*Late-start scenarios:*

- If vent valve is observed already ``Open`` at first poll, arm
  normally on this observation (no eligibility timer needed; no
  during-priming trigger to misfire).
- If a detector valve is observed already ``Open`` before the
  detector arms, disarm for that fill cycle (BL-1 covers).

**Action on verdict:**

* BL-2 RED, activated (future): close ``LNM{n}_FV:EN → 'Auto'``
  via ``caput_fn``, send RED Discord without prefix.
* BL-2 RED, INERT (current deployment): no caput; send RED Discord
  with ``[BL-2 IN DEVELOPMENT — NO ACTION TAKEN]`` prefix.  Selected
  by passing ``caput_fn=None`` from the ``run_monitoring`` wiring
  — no separate CLI flag.
* BL-3 YELLOW: no caput ever; send YELLOW Discord with no ``[BL-N]``
  prefix.  Active in both inert and activated phases of BL-2.

**Polling rate:** uses the existing main loop poll (30s normal, 5s
when warm-manifold tracker is active).  No new thread.  Vent-LED
reads are lazy — only fetched when the detector is armed for that
manifold, to avoid extra PV traffic in the common unarmed state.

**Wiring with ManifoldFailover:** ``ManifoldFailover`` has a
sibling setter ``attach_bl2_detector(bl2_detector)`` mirroring
the BL-1 ``attach_bl_detector``.  Both ``_switch_feed`` and
``_close_feed`` success paths call both ``bl.suppress(manifold_id)``
and ``bl2.suppress(manifold_id)`` (null-safe).  Primary BL-2/BL-3
disarm comes from the in-detector TS-LED-Cold-to-Warm rule above;
the failover-suppress() call is the redundant safety net per TODO
#38 for cases where the TS LED warm-up lags the detector's next
poll.

**No wall-clock suppression mechanism** (unlike BL-1's
``BL_FAILOVER_SUPPRESS_SECONDS = 240``).  The once-per-fill design
+ multi-source disarm rules cover the failover race without a
wall-clock timer.

**Kill switch:** the existing ``--no-feedline-check`` flag covers
this detector too.  When set, ``bl2_detector`` stays ``None``
throughout ``run_monitor_tanks`` / ``run_add_press``; no vent-LED
reads, no alerts, ``attach_bl2_detector(None)`` leaves
``ManifoldFailover._bl2_detector`` at ``None`` and the
``suppress()`` calls become silent no-ops.

**Promoting BL-2 out of inert mode (future change):**
in ``run_monitoring``, change the BL-2 dispatch call from
``caput_fn=None`` to ``caput_fn=_caput``.  One-line diff.  BL-3
behaviour is unchanged.  The dev-banner prefix vanishes from RED
Discord messages and the feed valve closes when RED fires.

### fill_led_logger.py

Hosts both the CSV LED voltage logger (diagnostic, opt-in) and
the **live LED fault detector** (always-on by default, opt-out
via ``--no-led-check``).  A single 1Hz daemon thread spawned by
add-press or monitor-tanks polls all monitored LEDs and runs
both concerns from the same poll loop.  Pre-warms PV connections
at thread start (parallel via libca).

**CSV logger modes** (controlled by ``--log-led``/``--log-full``):

- ``--log-led`` (default): CSV logs the 112 detector LEDs only at
  1Hz.  Back-compat with the original logger.  ~12 MB per 30-min
  fill.
- ``--log-full``: superset of ``--log-led``.  CSV also logs 14
  extra LEDs (4 distribution-manifold vent ``LNM{n}_TM:AT``, 4
  TS-manifold ``LNM{n}A_TM:AT``, 6 tank-vent ``LNT{n}_TM:AT``) and
  emits valve transition event rows for 136 valves (112 detector
  ``FV:VM``, 4 manifold vent ``VV:VM``, 8 manifold feed ``FV:VM``,
  6 tank feed ``FV:VM``, 6 tank vent ``VV:VM``).  Required input
  for BL-2 priming analysis.  ~14 MB per 30-min fill plus a few
  KB of event rows.

**Live fault detector** (``LiveLedMonitor`` class):

Polls 120 LEDs every 1s regardless of CSV mode — 112 detector
(``LNH{m}-{v}_TM:AT``), 4 manifold-vent (``LNM{n}_TM:AT``), 4 TS
(``LNM{n}A_TM:AT``) — and their companion valve state PVs.
Runs the locked B-2 fault state machine on each sample and
accumulates fault entries in an in-memory buffer.  At clean exit
(stop_event set normally), flushes the buffer to JSON state
(``current_fill_alerts['led_faults']`` + ``live_led_active=True``)
via an atomic tmp+rename write.  If the monitor crashes mid-fill,
the buffer is discarded with no JSON side effect — the classifier's
log-based fallback then runs uncontested.

Fault rules (8 fault types, see ``classify_voltage_sample``):

- ``SUSPECT``: first polling sample of the run, 2.4V < V ≤ 5.86V
  (one-shot per process lifetime).
- ``START_HIGH`` / ``START_LOW`` / ``START_SUSPECT``: valve
  Auto→Open sample.  START_SUSPECT only fires in (4.0V, 5.86V];
  START_HIGH supersedes it when V > 5.86V.
- ``END_HIGH`` / ``END_LOW``: valve Open→Auto sample, classified
  from the LAST voltage observed while the valve was Open.  The
  Open→Closed transition ALWAYS disarms HIGH/LOW regardless of
  whether END_* fired.
- ``HIGH`` / ``LOW``: continuous polling, valve CLOSED only.
  Schmitt-trigger state machine armed at startup; rearms when V
  crosses back through the hysteresis zone (5.86 - 0.4 = 5.46V
  for HIGH; 1.6 + 0.4 = 2.0V for LOW).  See
  ``LED_FAULT_HYSTERESIS`` docstring for the rationale.

The ``--no-led-check`` flag disables only the live detector; the
CSV logger continues to run if its own flag is set.  The two
concerns are independent.

Fault entries (one per triggered check per LED per fill cycle)
carry: ``pv_id`` (e.g. ``LNH1-04``, ``LNM2``), ``fault_type``,
``source='live'``, ``fill_id`` (timestamp string for the current
fill cycle), plus ``gsid`` + ``hose`` for detector LEDs.

**CSV format (4 columns, schema preserved across both modes):**

.. code-block:: text

    timestamp,manifold,valve,led_v

- Detector LED voltage (both modes):
  ``ts, m, vv, voltage``  where ``m=1..4``, ``vv=01..28``.
- New non-detector LEDs (``--log-full`` only):
  ``ts, M_VENT, n, voltage``  (manifold vent LED, ``n=1..4``)
  ``ts, TS,     n, voltage``  (TS manifold LED, ``n=1..4``)
  ``ts, TANK,   n, voltage``  (tank vent LED, ``n=1..6``)
- Valve transition event rows (``--log-full`` only):
  ``ts, category, instance, EVENT=<state>``  where the led_v
  column carries the literal ``EVENT=`` prefix followed by the
  new valve state.  Categories: detector valves reuse the manifold
  number ``1..4``; ``M_VENT`` (manifold vent valves, n=1..4),
  ``M_FEED`` (manifold main feed valves), ``M_SPARE`` (manifold
  spare feed valves), ``T_FEED`` (tank feed valves, n=1..6),
  ``T_VENT`` (tank vent valves, n=1..6).  Event rows fire only on
  state change (not per poll), keeping event volume bounded by
  transition count.

**Output path:** ``logs/fill_monitor/led_log_<mode>_YYYYMMDD_HHMMSS.csv``
where ``<mode>`` is ``led`` or ``full``.

Independent of BrokenLineDetector — works with or without
``--no-feedline-check``.

The pre-fill uses a more aggressive formula than the post-fill adjuster:
2× proportional (20-60s vs 10-30s) plus a 30s flat boost, yielding
50-90s total adjustment. No cold_time clamp is applied.  This is
intentionally aggressive: the small amount of LN2 wasted by
overshooting during a scheduled fill pales in comparison to the amount
needed to cool down the entire delivery system for an extra mid-day
auto fill just to service one underfilled detector.  The bump is
temporary (restored by the adjuster after the fill).

### fill_flush_history.py

Reset tracking history for a single detector. Clears history,
temp budget, consecutive counters, holdoff timestamps, and clamp
tracking from the state JSON. Does not modify gefilltime2.dat.
Use when a detector's history is invalidated by a known change
(dewar swap, sensor replacement, hose reconfiguration) or to clear
a false alarm from a sensor glitch.

### fill_adjuster_sim.py

Library for simulation replay. Drives classifier→adjuster→reporter
per log. Used internally by the plotter's simulation mode.

### fill_add_press.py

Ext fill valve pressure management during fills.  Runs concurrently
with `LNFill_App.py` during scheduled fills (LNFill_cron.sh).  Opens
ext fill valves to maintain tank pressure, monitors manifold temps,
performs automatic failover.  Imports monitoring classes from
fill_tank_monitor.py.  See [Pressure Management](#pressure-management-add-press).

### fill_tank_monitor.py

Manifold monitoring and failover during auto-fills (M fills).  Provides
the `monitor-tanks` command (replaces `add-press --no-pressure-mgmt`).
Also serves as the shared monitoring library — ManifoldTempTracker,
ManifoldFailover, LowPressureTracker, pressure fallback cascade, valve
PV access, and Discord alert helpers.  Imported by fill_add_press.py.
See [Tank Monitoring](#tank-monitoring-monitor-tanks).

### fill_interfaces.py

Shared I/O infrastructure.  Single bootstrap point for EPICS
(pyepics env vars, PV cache, caget_many), Discord (WriteDiscordMessage wrappers),
InfluxDB (push_fill.sh wrapper, INFLUX_TXT_DIR), and common utilities
(date_str).  All external dependency setup lives here.

Provides `caget_num(pv)` and `caget_str(pv)` with automatic dispatch:

- **Single PV (str):** reads via pyepics built-in PV cache
  (`epics.get_pv()`).  First read pays ~30ms connection cost;
  subsequent reads return the monitor-cached value in <0.01ms
  (memory lookup, no network round-trip).  Best for long-running
  loops (add-press, monitor-tanks) where the PV cache stays warm
  between iterations.

- **List of PVs:** reads via `epics.caget_many`.  All connections
  happen in parallel — 332 PVs complete in ~180ms cold start.  No
  caching; every call pays the full connection cost (~25ms warm).
  Best for one-shot short-lived processes (check-press, prefill-check,
  SaveTemp) where cold start cost dominates.

For long-running callers, pre-warm the PV cache at startup via
``caget_cache_init`` (defined in `fill_interfaces.py`, which
imports Ryan's ``pv_lock`` from `pvlock.py` for serialisation):

    connected, failed = caget_cache_init(all_my_pvs)

This fires CA-search broadcasts in parallel via libca's background
thread, then waits up to ``timeout`` seconds for each PV to connect.
Measured worst case on the LNFill IOC: 543 ms for 262 PVs cold,
~67 ms for the same set warm (IOC channel table remembered the
connections).  See § "Channel pre-warm" below for the architectural
rationale, the 2026-06-01 incident that drove it, and the four
integration sites.

A bare ``[get_pv(name) for name in all_my_pvs]`` is **not** sufficient
on its own: ``get_pv()`` only registers the PV with pyepics' dict
cache and queues a CA-search with libca's background thread.  The
first real ``caget`` on each PV still pays the cold-channel-open
cost (and can hit the 500 ms ``caget`` timeout cliff under load).
``caget_cache_init`` calls ``wait_for_connection`` on each PV to
actually complete the warmup.

Also provides `caput()`, `get_pv()`, `has_pyepics()`,
`send_discord_operational()`, `send_discord_anomaly()`, `push_influx()`,
and `date_str()`.

No domain logic.  No JSON state access.  No fill-specific knowledge.

### fill_constants.py

Pure physical / algorithmic constants — the values that govern WHAT
the system does (LED voltage thresholds, temperature limits, fill
timing, GSID range, manifold geometry).  Zero side effects, zero
infrastructure dependencies, safe to import anywhere.  Companion to
`fill_interfaces.py`, which owns the HOW (EPICS, Discord, InfluxDB,
paths).

Manifold geometry constants (`NUM_MANIFOLDS`,
`NUM_VALVES_PER_MANIFOLD`, `MANIFOLD_LABELS`) describe the four
fill manifolds (A, B, C, D), each with 28 hose positions, used by
the prefill batched LN IOC read and by any future plumbing‐shape
consumer.

Manifold-key constants are the single source of truth for the
internal ``'man_a'`` / ``'man_b'`` / ``'man_c'`` / ``'man_d'``
identifiers used as dict keys throughout the codebase:

- ``MANIFOLD_KEYS = ('man_a', 'man_b', 'man_c', 'man_d')`` — the
  canonical tuple, iterate this instead of hardcoding the literal.
- ``MANIFOLD_KEY_TO_LABEL = {'man_a': 'A', ...}`` — internal key to
  operator-facing letter.  Use for any log line or alert message.
  Defined as an explicit dict (not derived from ``MANIFOLD_LABELS``)
  to avoid the ``MANIFOLD_LABELS[1] == 'A'`` index-1 off-by-one
  trap from the 1-based ``'Z'`` placeholder at index 0.
- ``MANIFOLD_KEY_TO_NUM = {'man_a': 1, ...}`` — internal key to
  PV-name manifold number.  Use when constructing
  ``LNH{n}-{vv}_*`` style PV names.
- ``STATION_KEYS = ('TS1', 'TS2')`` — the analogous canonical
  tuple for the two Transfer Stations.

GSID range constants (`GSID_MIN = 1`, `GSID_MAX = 110`) replace
every bare `1..110` / `range(1, 111)` in the codebase.

### fill_check_pressure.py

Gauge reading foundation and hourly pressure snapshot.  Single source
of truth for gauge definitions, calibration, failure flags, and
InfluxDB logging.  Imported by fill_tank_monitor.py.

`read_all_gauges(batch=False)` reads all non-failed gauges with
calibration and range validation.  `batch=True` uses `caget_many`
for one-shot reads (check-press command); `batch=False` (default)
uses the pyepics PV cache for warm-cache reads (add-press/tank-monitor loops).

Also provides the `check-press` command for adaptive-rate snapshots
(see [Pressure Snapshot](#pressure-snapshot-check-press)) and the
``_check_ioc_health()`` function which runs on every cron invocation
regardless of rate gating.  If all non-failed gauges return None,
a Discord anomaly [CP-1] fires.  This detects IOC/network outages
between fills when add-press and monitor-tanks are not running.

---

## Channel pre-warm

Long-running commands (`add-press`, `monitor-tanks`) and the LED
logger thread auto-pre-warm their EPICS channels at startup so the
first iteration of their main loop reads from pyepics' warm monitor
cache (sub-millisecond) instead of paying the cold-channel-open
cost on each PV the first time it's read.  The mechanism is
``caget_cache_init`` defined in `fill_interfaces.py`,
which holds Ryan's ``pv_lock`` (from `pvlock.py`) during the
prewarm so any fill_monitor + LNFill_App same-process callers
coordinate through one mutex.

### Background: the 2026-06-01 06:00 incident

On 2026-06-01 the 06:00 cron-fired `add-press` tripped CP-1's
5-strike abort.  Investigation via on-pi5 timing probe showed:

- `read_valve_states()` does 10 sequential single-PV `caget_str`
  calls (4 main feed + 4 spare feed + 2 ext-fill).
- Each `caget_str` has a 0.5 s timeout.
- Against a cold IOC channel (libca background thread still
  processing the CA-search/connect handshake), individual
  `caget` round-trips land in the 200-500 ms range.
- On 2026-06-01 06:00, 5 of the 10 cagets landed at exactly
  500 ms (timeout cliff).  The 5-strike CP-1 watchdog tripped
  and add-press exited via the [PV-2] abort path at 06:00:26.

A second run-through 90 minutes later (no IOC state change
on the IOC side, same cron context, no code changes) showed the
same cold-start pattern: total `read_valve_states()` wall time
1544 ms vs <1 ms post-warmup.  Cold-channel-open is not random
IOC variance — it is the *normal* per-PV cost when libca is
asked to open and read in one shot, and the 0.5 s `caget` timeout
is tight enough that some PVs fail under any concurrent libca
load from the LED logger thread.

### The fix

One-shot pre-warm before the timing-sensitive loop runs:

1. Call ``epics.get_pv(name)`` for every PV in the catalogue.
   This is instant — it registers the PV with pyepics' dict
   cache and hands a CA-search task to libca's background
   thread.
2. Call ``pv.wait_for_connection(timeout=5.0)`` on each.
   Sequential in Python, but libca opens channels **in parallel**
   in the background, so total wall time is `max(connect_time)`
   across the list, not `sum(connect_time)`.
3. After ``caget_cache_init`` returns, every subsequent ``caget``
   on those PVs hits pyepics' warm monitor cache in <0.01 ms
   (memory lookup, no network round-trip).

### Measured performance (LNFill IOC)

| Catalogue | PVs | Cold (first call) | Warm (post-clear, IOC remembers) |
|---|---:|---:|---:|
| Gauges (`read_all_gauges`) | 5 | 113 ms | 1.3 ms |
| Valve states (`read_valve_states`) | 14 | 62 ms | 2.6 ms |
| LED full + LiveLed | 262 | 544 ms | 67 ms |

Default ``caget_cache_init`` timeout is 5.0 s — ~9× headroom over
the measured worst case for the largest catalogue.

### Integration sites

Four sites in fill_monitor wire through ``caget_cache_init``:

1. **`fill_led_logger._led_logger_loop` startup** — pre-warms the
   union of CSV LED PVs, CSV valve PVs (full mode), and LiveLed
   PVs (typically ~146-262 PVs depending on mode).  Replaces an
   earlier ``for pv in to_prewarm: get_pv(pv)`` loop that only
   registered the PVs and didn't actually wait for connection.

2. **`fill_tank_monitor.prewarm_detector_valve_pvs`** — pre-warms
   all 112 detector valve state PVs (LNH{1-4}-{01..28}_FV:VM) at
   fill detection time.  Called once per monitor session before
   the BrokenLineDetector starts polling.

3. **`fill_tank_monitor.read_valve_states` auto-prewarm** — on
   the first call per process, pre-warms the full **14-PV
   failover-complete pool**: 4 main feed + 4 spare feed (always
   the same PVs) + 6 ext-fill candidates (all of
   `TANK_EXT_FILL_PV.values()`, since ManifoldFailover can swap
   `PV_NAMES['ts1_fill']` / `['ts2_fill']` to any of the 6 at
   runtime).  A module-level flag (`_VALVE_STATE_PVS_PREWARMED`)
   gates this to one-shot per process.  Pre-warming only the
   2 currently-active ext-fill PVs would re-introduce the 500 ms
   cliff on the very first post-failover read — the whole 6-PV
   pool must be warm so failover is invisible to the timing.

4. **`fill_check_pressure.read_all_gauges(batch=False)` auto-prewarm**
   — on the first call per process, pre-warms all non-failed
   gauges from `GAUGE_KEY_MAP`.  Gated by `_GAUGE_PVS_PREWARMED`.
   Only the `batch=False` branch is gated; `batch=True` is left
   un-pre-warmed (see architectural exceptions below).

### Lock: shared with Ryan's pv_lock

``caget_cache_init`` is defined in `fill_interfaces.py` and uses
Ryan's ``pv_lock`` (imported from `pvlock.py`, introduced
2026-01-15 in `LNFill_App.py` for the same libca-thread-safety
reasons).  Holding the same lock means if anything ever runs both
fill_monitor code AND LNFill_App-style code in the same process,
the two codebases coordinate through one mutex.  The lock is held
only for the duration of the pre-warm sequence; steady-state
``caget_str`` / ``caget_num`` / ``caput`` / ``get_pv`` calls run
lock-free as they always did.

### Architectural exceptions (DO NOT convert)

Two patterns intentionally do NOT route through
``caget_cache_init``.  These are not bugs.  If a future change
is tempted to "fix" them, that's wrong:

1. **`read_detector_led_voltage`** (BL-1 lazy LED voltage reads,
   `fill_tank_monitor.py` ~line 312) — single-PV cold reads on
   demand only, fired by BrokenLineDetector only after a valve
   has been open >= 180 s.  Pre-warming all 112 ``TM:AT`` PVs to
   save 1-2 reads is wasteful: 99% of valves never reach the
   180 s threshold, the cold-open cost is paid at most a handful
   of times per fill, and the BL-1 path already tolerates per-PV
   timeouts gracefully (returns None on failure).

2. **`caget_many` one-shot paths** — `read_all_gauges(batch=True)`,
   `fill_prefill_check.py`'s 4 bulk LN IOC reads, etc.  These use
   `epics.caget_many` which opens throwaway channels in parallel
   and discards them on return.  A prior pre-warm contributes
   nothing because `caget_many` never consults pyepics' PV cache
   — it always does fresh network I/O.

Both exceptions are cross-referenced in the cache-miss regression
test (`test_bench/test_caget_cache_init.py` group CMR*) so any
future change that accidentally routes either one through
``caget_cache_init`` will fail loud.

### Failure mode and operator visibility

If the pre-warm fails partially (some PVs don't connect within
5 s), each integration site writes a single stderr line of the
shape:

    [WARN] <site>-pre-warm failed to connect N of M PVs (first: [...])

The command continues regardless.  Subsequent per-PV cagets on
the failed PVs hit their own 0.5 s timeouts naturally and the
existing failure-handling paths (PV-1 / PV-2 abort, gauge None
result, BL-1 fallback) take over.  The pre-warm is strictly
best-effort; it never changes the failure semantics of the path
it fronts — only the timing-window in which they can occur.

---

## Data Flow

All components communicate through the shared JSON state file
(`logs/fill_monitor/fill_monitor_state.json`). Each component owns
non-overlapping keys and uses read-modify-write to preserve other
components' data.

**Key design rules:**

- The classifier only appends structured data to JSON and never imports
  the reporter or any output library.
- The reporter is the sole consumer of alert data. After dispatching,
  it clears consumed keys (`current_fill_alerts`, `pending_weekly_summary`)
  so alerts are never re-sent.
- The classifier owns weekly rollover detection (Monday 00:00). On
  rollover, it snapshots the outgoing week's data to
  `pending_weekly_summary`, then clears `weekly_log_sections` for the
  new week.
- The adjuster enriches classifier entries (temp_overrides) in place
  via read-modify-write on the JSON file.

**Idempotency:** The classifier checks `last_classified` (filename) and
skips state updates if the same file is classified again. The reporter
clears alert keys after dispatch, so a second `report()` call is a no-op.

---

## Input: Fill Log Format

All post-fill processing begins with a fill log produced by
`LNFill_App.py`. Understanding this format is essential for the
classifier and adjuster sections that follow.

### Header

Two formats exist:

```
LNFill_App.py F fill initiated on 2026-05-12 at 06:00:07     (current)
Initiating Fill from LNFill_All.py on 2026-01-29 at 16:29:18  (legacy)
```

Fill types: F (full scheduled), M (temperature-monitored emergency),
L (list/subset), T (tank-only, no detector data).

### Detector lines

```
                       OPEN         TEMP         OVERFLOW               COLD
    VALVE     GSID     TIME     BEGIN  END    BEGIN   END     STATUS    TIME
DET: A- 1      73      210      88.1  88.4     1.87  5.65    NORMAL      210
DET: A- 2      95      153      90.5  90.5     1.89  5.78    EXTENDED     96
DET: D- 3      52      421      91.7  91.6     1.80  2.98    OVERTIME    419
```

| Field | Description |
|-------|-------------|
| VALVE | Physical hose label (e.g., "A- 1") |
| GSID | Detector ID (1-110 valid; < 1 or > 110 invalid) |
| OPEN_TIME | Total seconds valve was open |
| TEMP_BEGIN | Detector temperature at fill start (K), or `--` |
| TEMP_END | Temperature at fill end (K), or `--` |
| OVF_BEGIN | Overflow LED voltage at fill start |
| OVF_END | Overflow LED voltage at fill end |
| STATUS | NORMAL, EXTENDED, or OVERTIME |
| COLD_TIME | Seconds until overflow LED went cold |

### Fill outcomes

- **NORMAL** — LED went cold after min_time. Valve closed promptly. Ideal.
- **EXTENDED** — LED went cold before min_time. Valve stayed open until
  min_time expired. Dewar filled faster than expected.
- **OVERTIME** — LED never went cold by max_time. Valve forced closed.
  Dewar may not be full, or sensor malfunctioning.

---

## Classifier

The classifier (`fill_classifier.py`) parses a fill log, validates
GSIDs, detects hose remappings, logs anomalies to weekly_log_sections,
computes temperature means, and writes per-fill alert data to JSON.

### Per-fill validation

1. **GSID validation** — GSIDs < 1 or > 110 counted as invalid (GSID_MIN/GSID_MAX in `fill_constants.py`). Duplicate
   GSIDs within the same fill detected.
2. **Hose→GSID change detection** — Each hose label checked against
   the stored mapping. Reassignments logged.
3. **Detector configuration change detection** — Compares the current
   fill's detectors against `known_detectors` (a dict mapping GSID →
   hose, persisted in JSON state). Three event types:
   - **New detector:** GSID appears for the first time (any fill type).
   - **Removed detector:** GSID absent from an F-fill. Only F-fills
     trigger removal because only F-fills see all installed detectors.
   - **Moved detector:** Existing GSID on a different hose than
     previously recorded (any fill type).

   Any fill type can add or update entries in `known_detectors`. Only
   F-fills replace the entire dict (enabling removal detection).
   Unresolved duplicate GSIDs are excluded from all detector change
   detection and preserved in `known_detectors` on F-fill replacement.

   Events are written to `weekly_log_sections.detector_changes` and
   `current_fill_alerts`. The classifier also snapshots
   `prev_known_detectors` for the adjuster’s reconfiguration detection.

### Per-detector classification

For each detector in the fill log:

1. **Bad temperature** — If temp_begin is invalid (None, ≤60K, or
   >150K), logged to `bad_temp` weekly section.  Also records
   `temp_end` and `temp_end_valid` so the reporter can determine
   severity: valid end = info (transient), invalid end = warning
   (sensor fault or warm detector).
2. **LED voltage faults** — Detector overflow LED faults at fill
   start and end (ovf_begin, ovf_end), logged to the
   `led_faults` weekly section.  Fault types: `START_HIGH` /
   `END_HIGH` (>5.86V — open circuit / failed open), `START_LOW`
   / `END_LOW` (<1.6V — short / failed low), `START_SUSPECT`
   (>4.0V but ≤5.86V at start — not warming between fills,
   M1-13-class).

   **Source-selection gate:** if the live LED monitor (B-2
   `fill_led_logger`) ran cleanly for this fill cycle, it set
   `current_fill_alerts['live_led_active'] = True` and flushed
   buffered fault entries to `current_fill_alerts['led_faults']`.
   In that case, the classifier skips this log-based fallback
   block entirely and instead promotes the live entries into
   `weekly_log_sections['led_faults']`.  Otherwise (live crashed,
   didn't run, or was disabled via `--no-led-check`), the log-
   based path emits entries as described above, each tagged
   `source='log'`.  Live entries carry `source='live'`; the two
   are mutually exclusive per fill cycle (never both).

   `START_HIGH`/`START_LOW` are logged but never used to reject a
   detector reading — detectors often start high (cold from a
   recent fill) and cool during fill.  Only `END_HIGH`/`END_LOW`
   influence the adjuster's hard-fallback decision.
3. **OVERTIME** — Status is OVERTIME: logged to `overtime`.
4. **Temperature mean** — Computed from prior fills (current fill
   excluded). Uses the lesser of two methods:
   - Method A: nearest (then longest) consecutive run with ≤1K spread,
     within last 10 fills (≤18h gaps). Skips outlier spikes.
   - Method B: consecutive recent fills, each ≤18h apart
   Requires ≥2 qualifying fills. Result stored in classification output
   for the adjuster to use.
5. **Temperature delta** — If temp_begin is valid and delta ≥1K from
   mean, logged to `temp_overrides` (adjuster enriches later with
   adjustment details).

### Hose-GSID tracking

Each physical hose maps to a detector GSID. The classifier tracks this
mapping across fills via `known_detectors` (GSID → hose, used for
compatibility) and `known_detectors` (the primary detector tracking
dict). When a hose switches detectors, the adjuster resets all
per-detector state (consecutive counters, holdoffs, clamp, history).

### Weekly rollover

The classifier detects Monday 00:00 (Chicago time) boundaries. On
rollover:
1. Snapshots the outgoing week's `weekly_log_sections` + `week_start`
   to `pending_weekly_summary` in JSON.
2. Clears all weekly_log_sections for the new week.

### Per-fill alerts

After classification, the classifier writes `current_fill_alerts` to
JSON — a snapshot of only THIS fill's events (filtered by timestamp).
The reporter reads this for Discord dispatch. See
[Discord Alert Reference](#discord-alert-reference).

---

## Adjuster

The adjuster (`fill_adjuster.py`) owns all min_time adjustment logic.
It reads the classifier's output (per-detector classifications, parsed
log data, temperature means) and applies rules to nudge each detector's
minimum fill time toward an optimal value.

### Incomplete fill log detection

Before processing, the adjuster checks whether the fill log is complete.
A fill log is **complete** if it contains the `Total App Runtime = NNN
seconds` footer written by `LNFill_App.py` at the end of a normal fill
cycle.

A fill log is **incomplete** if the footer is missing.  This indicates
the fill was aborted or `LNFill_App.py` crashed before finishing.
Incomplete logs fire a 🔴 red alert to #anomaly regardless of fill type:

- **Partial fill** (has detector data, no footer): some detectors were
  filled but the fill didn't complete normally.  The adjuster still
  processes the available detector data.
- **Empty fill** (no detector data, no footer): the fill failed to start
  or was aborted before any detectors were filled.

M-fills that find no warm detectors exit without creating a log file.
The cron script's `if [ -f "$outfile1" ]` guard prevents the adjuster
from being called in that case.  Any log file that reaches the adjuster
was created by a fill that at least started.

### Processing order

For each detector, the adjuster executes these steps after the
classifier has finished:

1. **Initialize defaults** — New GSIDs get DEFAULT_MIN_TIME (150) and
   DEFAULT_MAX_TIME (419).
2. **Compute base clamp** — `clamp_max = min(360, max_time × 0.9)`.
3. **Cold time clamp tightening** — If cold_time ≥ 80s AND ovf_end
   < 5.86V AND cold_time + 30 < current min_time, the clamp condition
   is armed. After 12h of continuous eligibility, the clamp tightens to
   `min(base_clamp, cold_time + 30)`. Any non-eligible fill resets the
   confirmation window.
4. **Temperature override check** — Using the classifier's pre-computed
   mean, if temp_begin is valid and delta ≥ 1K from mean:
   - delta ≥ 3K → raw_adj = +30s
   - 1K ≤ delta < 3K → +10s to +30s (proportional)
   - Compute budget_adj = min(raw_adj, remaining_budget)
   - Compute fill floor: floor_adj = max(0, fill_time + 10 - old_min)
   - temp_adjustment = max(budget_adj, floor_adj)
5. **Apply adjustment** — Exactly one path executes:
   - **Path A** — Temp adjustment available → apply, charge budget
   - **Path B** — Temp triggered, no adjustment needed → skip nominal
   - **Path C** — OVERTIME, no temp trigger → no adjustment
   - **Path D** — NORMAL/EXTENDED, no temp trigger → nominal rule
6. **Record fill result** — Append to fill_results.
7. **Update weekly tracking** — min_time start/end/lo/hi.

### Effective time

The algorithm compares an **effective time** against the target
(`min + TARGET_OFFSET`). The effective time is determined by the
reliability of the cold_time reading:

- **Valid cold_time** — Used directly when
  `cold_time ≥ 80% of (min + 20)`. In simulation, if cold_time < sim's
  adjusted min_time, fill_open is used instead.
- **Soft fallback** — cold_time failed the 80% threshold but ≥ 80s AND
  ovf_end < 5.86V. Uses fill duration. Full decay formula and 2x
  eligibility.
- **Hard fallback** — cold_time < 80s, OR ovf_end ≥ 5.86V, OR
  cold_time is None/0. Uses fill duration. Limited to -1 decay, no 2x
  eligibility.

**START overflow voltage (ovf_begin) is NEVER used to reject a sensor
reading.** Only END overflow voltage causes hard fallback.

### Nominal rule (±1/±2)

Applied to NORMAL and EXTENDED fills when no temperature override is
active:

```
target = min + TARGET_OFFSET (20)

if effective > target:           +1  (or +2 if 2x up active)
elif effective < min + 10:       proportional: adj = floor((min+10 - eff)/10) × (-1) - 2
                                 (multiplied by 2x down_speed if active)
elif effective < target:         -1  (or -2 if 2x down active)
else:                            no change

Hard fallbacks: limited to -1 maximum downward adjustment.
```

### 2x acceleration

**Downward:** After 4+ consecutive eligible fills (not hard fallback)
with effective_time < min + 10, all downward adjustments double. Hard
fallbacks break consecutivity. Counter resets on any temp event. During
post-temp holdoff, consec_below is frozen at 0.

**Upward:** After 8+ consecutive fills (ALL fills count, including hard
fallbacks) with effective_time > min + 30, the +1 doubles to +2.
Upward counter is NOT reset or frozen by temp events.

### Temperature override

| Delta from mean | Adjustment |
|-----------------|------------|
| ≥ 3K | +30s |
| 1K – 3K | +10s to +30s (proportional) |
| < 1K | No override, use nominal rule |

**Fill-time floor:** min_time raised to at least fill_time + 10. Can
exceed budget cap. Constraints: not OVERTIME, 12h cooldown,
`FILL_FLOOR_RECENCY_HOURS` (13h) recency, no hose change.

**Budget:** 30s capacity, regenerates 5s per 12h (72h full recovery).
Floor excess is charged; budget saturates at 0. If min_time <
CLAMP_ABSOLUTE_MIN (75), the portion up to the floor is free.

**When triggered (even if budget-blocked):** fill floor can still
increase min_time. Nominal rule skipped, consec_below resets.

**Invalid temperatures:** None, ≤60K, >150K — excluded entirely.

### OVERTIME

Only temperature override applies. No nominal adjustment.

### Clamp

```
min ≤ min(360, max_fill_time × 0.9, cold_time + 30)
```

The cold_time + 30 clamp requires: cold_time ≥ 80s, ovf_end < 5.86V,
cold_time + 30 < current min_time, AND 12h continuous eligibility.

### Post-temperature holdoff

After any temp event, downward adjustments limited to -1 per 12h for
5 days (120h). consec_below frozen at 0. consec_above unaffected.

### Reconfiguration detection

The adjuster uses `prev_known_detectors` (a snapshot from the
classifier) to detect when a detector needs a state reset. A reset
fires when:
- The GSID has no previous hose (`prev_hose is None` — new detector), or
- The GSID’s hose changed (`prev_hose != hose` — moved detector).

The reset fires exactly once, on the fill where the change is detected.

Fill-floor recency is gated by `FILL_FLOOR_RECENCY_HOURS` (13h),
which replaced the broader `TEMP_MEAN_MAX_GAP_HOURS` (18h) gate
previously used for this purpose.

Resets all per-detector state: consecutive counters, holdoffs, clamp,
and history.

### Worked examples

#### Example 1: Normal fill — increase

Detector with `min_time = 200`, `max_time = 419`, NORMAL fill.

```
cold_time  = 230
target     = min + 20 = 220
threshold  = target × 0.8 = 176

cold_time (230) ≥ threshold (176) → valid, effective_time = 230
effective (230) > target (220) → adjustment = +1
new min_time = 201
```

#### Example 2: Short fill — soft fallback with proportional decay

Detector with `min_time = 200`, EXTENDED fill, LED went cold early.

```
cold_time  = 150
threshold  = (200 + 20) × 0.8 = 176

cold_time (150) < threshold (176) → fallback triggered
cold_time (150) ≥ HARD_FALLBACK_FLOOR (80) → not hard
ovf_end = 3.2V < 5.86V → sensor valid at end
→ SOFT FALLBACK: effective_time = open_time = 200

effective (200) < min+10 (210) → proportional decay zone
gap = 210 - 200 = 10
adj = floor(10/10) × (-1) - 2 = -3
new min_time = 197  (or 194 if 2x down active)
```

#### Example 3: Temperature override

Detector with `min_time = 180`, mean temp = 90.0K, this fill
temp = 95.0K, fill_time = 195s. Budget has 30s remaining.

```
delta = 95.0 - 90.0 = 5.0K → raw_adj = +30s
budget_adj = min(30, 30) = 30
fill floor = 195 + 10 = 205 → floor_adj = 205 - 180 = 25
temp_adjustment = max(30, 25) = 30  (budget wins)

new min_time = min(180 + 30, clamp_max) = 210
Budget charged 30s → 0s remaining. Full recovery in 72h.
```

#### Example 3b: Fill floor exceeds budget

Same detector, budget = 5s, fill_time = 220s.

```
budget_adj = min(30, 5) = 5
fill floor = 220 + 10 = 230 → floor_adj = 230 - 180 = 50
temp_adjustment = max(5, 50) = 50  (floor wins, bypasses budget cap)

new min_time = 230. Budget charged 50s → saturates at 0.
```

#### Example 4: Hard fallback — limited decay

Detector with `min_time = 250`, overflow sensor stuck open.

```
cold_time = 3 → HARD FALLBACK (< 80s)
effective_time = open_time = 250

Would normally get proportional decay, BUT hard fallback → limited to -1
new min_time = 249
consec_below reset to 0. No 2x eligibility.
```

#### Example 5: Overtime with temperature override

Detector with `min_time = 300`, `max_time = 419`, OVERTIME.
Temp delta = 4K.

```
OVERTIME → no nominal adjustment
delta = 4.0K ≥ 3.0K → raw_adj = +30s
clamp_max = min(360, 419 × 0.9) = 360
new min_time = min(300 + 30, 360) = 330
```

### fill_results data structure

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

### Tunable parameters

All defined as class constants on `FillAdjuster`:

| Parameter | Default | Description |
|-----------|---------|-------------|
| TARGET_OFFSET | 20 | Target fill time = min + this |
| PROP_DECAY_THRESHOLD | 10 | Below min + this → proportional decay |
| UP_ACCEL_THRESHOLD | 30 | Above min + this → counts toward 2x up |
| COLD_REJECT_FRACTION | 0.8 | Cold time < target × this → fallback |
| HARD_FALLBACK_FLOOR | 80 | Cold time < this → always hard fallback |
| LED_OPEN_FAULT_THRESHOLD | 5.86 | Overflow voltage above this = sensor open (centralized in `fill_constants.py`; exposed as class attribute on FillAdjuster and BrokenLineDetector under the same canonical name) |
| LED_SHORT_FAULT_THRESHOLD | 1.6 | Overflow voltage below this = sensor shorted/bad cable (exposed as class attribute on BrokenLineDetector under the same canonical name) |
| LED_NO_FLOW_THRESHOLD | 2.0 | Overflow voltage below this after 180s = no LN2 flow at detector (exposed as class attribute on BrokenLineDetector under the same canonical name) |
| CLAMP_ABSOLUTE_MAX | 360 | Absolute max min_time (seconds) |
| CLAMP_ABSOLUTE_MIN | 75 | Absolute min min_time (seconds) |
| CLAMP_MAX_TIME_FRACTION | 0.9 | Clamp at max_fill_time × this |
| CLAMP_COLD_TIME_MARGIN | 30 | Clamp at cold_time + this |
| CLAMP_CONFIRM_HOURS | 12 | Continuous eligible period for cold clamp |
| TEMP_OVERRIDE_DELTA_HIGH | 3.0 | delta ≥ this → max adjustment |
| TEMP_OVERRIDE_DELTA_LOW | 1.0 | delta ≥ this → proportional adjustment |
| TEMP_OVERRIDE_MAX_ADJ | 30 | Max single temp adjustment (adjuster, seconds) |
| TEMP_OVERRIDE_MIN_ADJ | 10 | Min temp adjustment at DELTA_LOW (adjuster) |
| PREFILL_TEMP_MAX_ADJ | 60 | Pre-fill max proportional (2× adjuster) |
| PREFILL_TEMP_MIN_ADJ | 20 | Pre-fill min proportional (2× adjuster) |
| PREFILL_TEMP_FLAT_ADJ | 30 | Pre-fill flat boost added to proportional |
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
| FILL_FLOOR_RECENCY_HOURS | 13 | Max hours since last fill for fill-floor to fire |
| DEFAULT_MIN_TIME | 150 | Default for missing GSIDs |
| DEFAULT_MAX_TIME | 419 | Default max fill time |

---

## Reporter

The reporter (`fill_reporter.py`) formats the weekly monitoring log and
dispatches Discord alerts. It reads everything from the JSON state file
and is completely self-sufficient — no parameters passed from the
classifier or adjuster at call time.

### Weekly log

`logs/fill_monitor/fillmon_YYYYMMDD.log` — rolls over Monday 00:00
Central. The log accumulates across all fills during the week.

**Sections (in order):**

1. **High (Open) LED Voltage (>5.86V)** — Per-detector START/END/BOTH
   counts. START = dewar may have been full already. END = sensor didn’t
   detect fill. BOTH = sensor open throughout. Chronic BOTH suggests
   sensor failure or wiring fault.
2. **OVERTIME** — Each timeout with timestamp, GSID, open_time, ovf_end.
   Low ovf_end (2-3V) = genuine incomplete fill. High ovf_end (>5V) =
   likely sensor issue.
3. **Invalid/Extreme Temperature** — None, ≤60K, >150K. Latest per GSID.
   Persistent entries indicate hardware fault.
4. **Invalid/Duplicate GSIDs** — Out-of-range (< 1 or > 110) with hose and
   counts. Duplicate GSIDs within single fill = configuration error.
5. **Temperature Override Adjustments** — Sorted by GSID. With adjuster:
   shows old→new min_time, adjustment, delta. Without adjuster: shows
   temp delta from mean only. Indicates detectors warming faster than
   normal (cooling issue, dewar leak, or injector icing).
6. **Clamped Min-Times** (adjuster only) — GSIDs where adjustment was
   limited by the clamp (360s / max_time×0.9 / cold_time+30). Flags
   detectors approaching their operational limits.
7. **Detector Configuration Changes** — New detectors, removed
   detectors (F-fills only), and moved detectors. Each entry records
   {ts, type, gsid, hose, from_hose, to_hose}. Replaces the former
   "Hose-GSID Mapping Changes" section (N1). Hose-GSID mapping
   changes are still recorded in JSON state for audit but no longer
   rendered in the weekly log.
8. **Low Tank Pressure Events** — From add-press. Pressure drops below
   thresholds (20/18/15/12/9/6/3 psi) during fills.
9. **Warm Manifold Events** — From add-press. Manifold temperature
   sensor went Warm during fill, indicating possible empty tank.
10. **Min-Time Summary** (adjuster only) — Start/end/min/max/range/delta
    per detector for the week. Only changed detectors listed.

Two reporting modes:
- **Classifier-only** (no adjustment): sections 1-5, 7
- **Classifier + Adjuster** (full): all 10 sections

### Sample output

```
Fill Monitor Weekly Log — Week of 2026-05-05
==============================================================================

--- High (Open) LED Voltage Events (>5.86V) ---
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

---

## Pressure Snapshot (check-press)

The `check-press` subcommand (`fill_check_pressure.py`, V3.0) reads all
8 tank and supply pressure gauges, validates each reading (numeric,
within range), writes InfluxDB line protocol, and pushes via
`push_fill.sh`.

Uses adaptive sampling rate coordinated with AddPress via the shared
JSON state file (read-only — AddPress owns the keys):

- **AddPress running** (`addpress_started` > `addpress_finished`):
  SKIP — avoid interfering with active pressure management.
- **High-rate** (`addpress_finished` < 60 min ago): RUN — 5-minute
  sampling to capture tank pressure during the refill that follows
  each detector fill.
- **Normal** (top of hour): RUN — standard hourly snapshot.
- **Otherwise**: SKIP.

Samples are always aligned to wall-clock 5-minute boundaries
(cron schedule). After the 60-minute high-rate window expires,
sampling reverts to hourly at :00 on the next cron tick.

**Gauges:**
- Ext1/Ext2 (LNP1-01, LNP2-01): external supply, range -5 to 90 psi
- Tank1-Tank6 (LNP1-02..LNP2-04): tank pressures, range -5 to 45 psi

**Output:** `influx_txt/check_pressure.txt` (overwritten each run).

**Flags:** `--no-influxdb` writes the file but skips push.
`--force` bypasses all gating (addpress check, high-rate window, hourly
schedule) and samples immediately.

**Cron:** See [README.md](../README.md) for the authoritative crontab
entries. Runs every 5 minutes via cron; script self-gates.

---

## Tank Monitoring (monitor-tanks)

The tank monitoring layer (`fill_tank_monitor.py`, V1.0) provides the
shared monitoring infrastructure used by both `monitor-tanks` and
`add-press`, plus the `monitor-tanks` CLI command.

**This command is only intended to run alongside `LNFill_App.py`**
(launched by the same cron script).  It is not a standalone tool.
In current production, it is used only with M-fills (auto-fills via
`LNFill_Auto_EFill_cron.sh`).  F-fills use `add-press` which provides
both pressure management and monitoring.

### Architecture

Middle layer of a three-layer design:

```
fill_check_pressure.py   (Layer 1 — gauge reads, InfluxDB logging)
      ↑ imports from
fill_tank_monitor.py     (Layer 2 — monitoring classes + monitor-tanks command)
      ↑ imports from
fill_add_press.py        (Layer 3 — ext fill valve pressure control)
```

Layer 2 serves dual purpose: it provides the `monitor-tanks` CLI command
AND is the shared library imported by `add-press` for its monitoring
functionality.

### Pressure gauge cascade

Each station tries Tank1 → Tank2 → Tank3 gauges with calibration
offsets (from `PRESS_CAL` in check_pressure).  Failed gauges
(`PRESS_FAIL`) are skipped; if all fail, defaults to 28 psi (≤400s)
or 20 psi (>400s).  External pressure: Ext1 → Ext2 fallback →
default 28 psi.

Gauge reads use `read_all_gauges()` from check_pressure.  In
monitoring mode (default), individual reads use the pyepics PV cache (warm,
<1ms per gauge).  In one-shot mode (`batch=True`), reads use
`caget_many` for fast cold-start bulk reads.

### Pressure logging

Every iteration snapshots all valid gauges to an InfluxDB line-protocol
file.  Each command writes to its own file:
- `influx_txt/AddPress.txt` — add-press (from startup)
- `influx_txt/MonitorTanks.txt` — monitor-tanks (only after manifold open)
- `influx_txt/check_pressure.txt` — check-press (one-shot snapshot)

Known-failed sensors skipped; invalid readings warned once per run.
File bulk-pushed to InfluxDB via `push_fill.sh` at end of run.

### Warning deduplication

First gauge check reports all bad sensors upfront.  Subsequent warnings
for same device+reason suppressed with "(further warnings suppressed)"
note.  Dedup state reset at the start of each run via `clear_warned()`.

### Low tank pressure alerts

During fills, monitors tank station pressures and fires alerts when
thresholds are crossed.  Each threshold fires once per station per fill.

- **20 psi** — info (#anomaly channel)
- **18, 15, 12, 9, 6, 3 psi** — warning (#anomaly channel)

Pressure checks only run while manifold feed valves are open (fill in
progress).  Both main (`LNM*_FV:EN`) and spare (`LNM*A_FV:EN`) feed
valves are checked — fills can run on either (LNFill_App ManID≥5
configuration, or after failover switches to spare).  Thresholds use
calibrated float pressure (no integer truncation).  Events logged to
`weekly_log_sections.low_pressure`.

### Warm manifold detection

Monitors manifold temperature sensors (`LNM1A_TM:BT` through
`LNM4A_TM:BT`).  A manifold reads 'Cold' when LN2 is flowing and
'Warm' when it isn't — a Cold→Warm transition during a fill may
indicate the feeding tank is empty.

Alert conditions (once per manifold per fill):
- **Case 1:** Was Cold for ≥2 min, then sustained Warm for ≥10s.
- **Case 2:** Sustained Warm for ≥180s after fill valve opened
  (catches tanks that were already empty at fill start).

While any manifold is in Warm state, the sleep interval is capped at
2s instead of the normal poll interval.  Events logged to

During normal poll intervals (30s), cached manifold temperature PVs
are checked every second (interruptible sleep).  If a manifold
transitions to Warm while its feed valve is open, the main loop wakes
immediately instead of waiting for the full sleep.  This reduces
worst-case Cold→Warm detection from 30s to ~1s.

After a successful failover, when the manifold temperature transitions
back to Cold on the new feed, a ✅ confirmation message ([FO-6]) is
sent to #anomaly, confirming that liquid is flowing from the new tank.
`weekly_log_sections.warm_manifold`.

### Manifold failover

Bidirectional feed switching when a Warm alarm fires.  Each manifold
(A, B, C, D) is tracked independently — ManA's failover state does
not affect ManB's decisions.

Decision matrix:

| Current Feed | _failed_over | Action |
|---|---|---|
| main | False | Switch main→spare (normal failover) |
| main | True | Close feed (came from spare→main, both tried) |
| spare | False | Switch spare→main (spare was initial feed) |
| spare | True | Close feed (came from main→spare, both tried) |

**Failover sequence (main→spare or spare→main):**
1. Check target feed valve — if Disabled, abort (🔴 alert).
2. Close current feed valve (set to Auto, not Disable).
3. Open target feed valve.
4. Switch ext fill valve to the empty tank (if pressure mgmt active).
5. Reset warm tracker state for second-alarm detection.
6. Send 🔴 red Discord alert.

**Both feeds exhausted (shutdown):**
1. Close feed valve.  Better to stop than blow LN2 out of dewars.
2. Mark manifold as exhausted (no further failover attempts).
3. Send 🔴 red Discord alert.

### Disabled valve policy

Disabled is an administrative state set by an operator.  The code
NEVER writes to a Disabled valve.  Before every `_caput`, the current
state is read; if Disabled, the write is skipped.  When a required
valve is Disabled:
- Feed valve Disabled → failover cannot proceed, alert only.
- Ext fill Disabled → try another tank, alert if none available.
- Already Disabled → skip close (already effectively closed).

### Concurrency

On startup, add-press and monitor-tanks check for other running
instances via `psutil.process_iter()`.  Priority rules:

| Existing instance | New instance | Result |
|---|---|---|
| Full add-press | Full add-press | New aborts |
| Full add-press | monitor-tanks | New aborts |
| monitor-tanks | Full add-press | Existing terminated, new continues |
| monitor-tanks | monitor-tanks | New aborts |
| Old --no-pressure-mgmt | Full add-press | Existing terminated, new continues |
| Old --no-pressure-mgmt | monitor-tanks | New aborts |

### Valve PV timeout handling

If valve PV reads (manifold states, spare feed states, ext fill states)
fail 5 consecutive times, the command aborts with a ⚠️ yellow Discord
warning.  This catches EPICS IOC or network failures.  The counter
resets to 0 on any successful valve read cycle.

Temp PV timeouts are handled individually per manifold — a failed temp
read skips warm detection for that manifold but doesn't affect pressure
control, feed state tracking, or termination logic.

### Discord alert summary

| Condition | Icon | Channel |
|---|---|---|
| Tank empty — failover to alternate feed | 🔴 | #anomaly |
| Both feeds exhausted — manifold shutdown | 🔴 | #anomaly |
| Feed valve Disabled — cannot failover | 🔴 | #anomaly |
| Anomalous exit — manifolds open at termination | 🔴 | #anomaly |
| Ext fill valve deconfliction (add-press only) | ℹ️ | #anomaly |
| Ext fill Disabled — pressure mgmt degraded (add-press only) | ⚠️ | #anomaly |
| Warm manifold detected | ⚠️ | #anomaly |
| Valve PV reads failed 5 times | ⚠️ | #anomaly |
| Non-critical timeout — manifolds closed (add-press only) | ℹ️ | #anomaly |
| Low pressure 20 psi | ℹ️ | #anomaly |
| Low pressure 18-3 psi | ⚠️ | #anomaly |

### monitor-tanks command

**Three-phase startup:**

1. **Phase 1** — Parent PID check (with `--parent-pid`).  Instant
   `os.kill(pid, 0)` check.  If parent already exited, exits immediately.
2. **Phase 2a** — Wait for manifold valves to open (2s poll, 45s timeout).
   No pressure reads, no logging.  Exits if parent dies or timeout.
3. **Phase 2b** — Active monitoring (30s poll, 2s during warm detection).
   First pressure reading taken here.  No fill = no log, no InfluxDB.

**Termination:**

| Condition | Action |
|---|---|
| Parent PID gone + manifolds closed | Normal exit |
| Parent PID gone + manifolds open | 🔴 Red alert + exit |
| Manifolds closed (no `--parent-pid`) | Normal exit |
| Hard timeout (`--hard-timeout`) | Exit, 🔴 alert if manifolds open |
| 5 consecutive LN IOC valve PV failures | ⚠️ Yellow alert + exit |

---

## Pressure Management (add-press)

The `add-press` subcommand (`fill_add_press.py`, V3.0) adds ext fill
valve pressure control on top of the monitoring layer.  It imports all
monitoring classes from `fill_tank_monitor.py` and adds the
`AddPressController` state machine for valve open/close decisions.

Runs concurrently with `LNFill_App.py` during scheduled fills
(F-fills via `LNFill_cron.sh`).

### Physical purpose

During each fill, detectors draw LN2 from the local tanks, causing
pressure to drop.  The add-press script maintains tank pressure by
opening an ext fill valve to inject nitrogen from the external supply
into an unused tank.  The valve is adaptively selected — always
targeting a tank NOT currently feeding any manifold.  Because the
tanks are not vented during fills, the external feed line does not
fully chill down — mostly gas flows, which is effective at maintaining
pressure.

### Control logic

- Opens ext fill valve when ext−tank pressure ≥ 3 psi (`VALVE_OPEN_PRESS`)
- Closes when differential ≤ −1 psi (`VALVE_CLOSE_PRESS`) or tank ≥
  32 psi (`MAX_TANK_PRESS`)
- Holdoff timers prevent valve cycling: 120s after max-pressure close,
  60s after differential close
- Cross-station balancing: closes ext fill if own pressure exceeds the
  other station by `PRESS_DIFF_OFF_THRESH` (only when both open)
- Sleep ramp: 1→30s between iterations, resets to 1s on valve open
- Starts logging immediately (pre-fill pressure baseline is useful data)
- `--parent-pid`: exits when parent gone OR all manifold feed valves
  closed (both main AND spare, after MIN_RUN_TIME).  Whichever
  comes first.  Closes ext fill valves on exit.
- Fallback (no `--parent-pid`): exits after MIN_RUN_TIME (45s) +
  all manifold feed valves closed (both main AND spare), or hard
  timeout (2200s)
- Non-critical hard timeout (manifolds closed): sends info message
  to #anomaly indicating exit condition didn't trigger properly
- Anomalous hard timeout (manifolds open): sends red alert to #anomaly

### Operating modes

**Normal** (default): Full pressure management + monitoring + failover.

**--no-failover**: Pressure management active.  Warm detection fires
alerts but does not switch feeds.  Ext fill deconfliction still active.

For monitoring without pressure control, use the `monitor-tanks` command.

### Differences from monitor-tanks

| | add-press | monitor-tanks |
|---|---|---|
| Ext fill valve control | ✅ | ❌ |
| Ext fill deconfliction | ✅ | ❌ |
| Lifecycle signals | ✅ addpress_started/finished | ❌ |
| Startup | Immediate | Three-phase (wait for valves) |
| Logging | From startup | Only after manifold open |
| Poll interval | 1→30s ramp (resets on valve open) | 30s fixed (2s warm) |
| Intended use | F-fills (scheduled) | M-fills (auto) |

### Verification

The `verify` subcommand parses a recorded `AddPress.log`, feeds each
line through `AddPressController.step()`, and compares predicted valve
states against actual logged states.  Known limitation: manifold valve
state must be inferred from pressure transitions (no ln_log access),
causing expected mismatches at manifold boundaries.

---

## Discord Alert Reference

The fill monitor dispatches Discord alerts from multiple components.
Two channels are used:

- `WriteDiscordMessage(msg)` → **#system-messages** (`discord.WebHook`)
- `WriteAnomalyMessage(msg)` → **#anomaly** (`discord_anomaly.WebHook`)

Both functions live in `WriteDiscordMessage.py` (repo root, shared with
LNFill_App.py) and auto-prefix the hostname (e.g., `[pi5-lnFill]`).

Wrapper functions in `fill_interfaces.py`:
- `send_discord_operational(msg)` → `WriteDiscordMessage(msg)`
- `send_discord_anomaly(msg)` → `WriteAnomalyMessage(msg)`

Both wrappers append one line to `DISCORD_LOG_FILE`
(`cron_logs/discord_log.txt`) via the private `_log_discord(channel, msg)`
helper before calling the network sender.  Format:
`{timestamp}  {channel:11}  {msg-with-\n-escaped}\n`.
The local log is an audit trail / debugging aid; a write failure
is swallowed with a one-time stderr warning so the Discord send
always runs.  Suppressed alerts never reach the wrappers and
therefore never appear in the log.  See USER_GUIDE.md § Local
Discord Message Log.  Monthly rotation via `archive_cron_log.sh`
(archives to `cron_logs/archive/discord_log_YYYYMM.txt`).

All alert-sending components accept `--no-discord` to suppress output.
`--test` on add-press and monitor-tanks implies `--no-discord` and
`--no-influxdb`.

### Alert dispatch architecture

| Dispatch path | Components | Batching |
|---------------|-----------|----------|
| Reporter `discord_per_fill()` | adjust, report | Warnings batched → #anomaly; notices batched → #system-messages |
| Reporter `discord_weekly()` | report (Monday rollover) | W5 → #anomaly; N3 → #system-messages |
| Classifier `check_fill_completeness()` | adjust, report | ML-1, IL-2, IL-3 sent individually → #anomaly |
| Pre-fill check `pre_fill_adjust_check()` | pre-fill-adjust-check | PFM-1..5 per-detector immediate (no batching); PFM-OPEN one per affected manifold; VS-4, GF-1, N5, PF-1 batched by type; PF-2, PF-3, GF-2 sent individually |
| Tank monitor main loop | add-press, monitor-tanks | FO, W-WM, EX, PV sent individually → #anomaly |
| `LowPressureTracker.check()` | add-press, monitor-tanks | N-LP, W-LP sent individually → #anomaly |
| `BrokenLineDetector` | add-press, monitor-tanks | BL-1 sent individually → #anomaly (active; suppressed only by `--no-feedline-check`) |
| `BrokenPrimingDetector` | add-press, monitor-tanks | Single vent-close trigger, two-tier output: BL-2 RED (`<2.0V`) and BL-3 YELLOW (`2.0–3.0V`) both sent individually → #anomaly.  BL-2 currently INERT (Discord with `[BL-2 IN DEVELOPMENT — NO ACTION TAKEN]` prefix, no caput); BL-3 always live (no caput ever).  Suppressed entirely only by `--no-feedline-check`. |
| `_check_ioc_health()` | check-press | CP-1 sent individually → #anomaly |

---

### Infrastructure & IOC Health

#### PV-1 — Valve PV Read Failures (monitor-tanks)

| Field | Value |
|-------|-------|
| **Severity** | ⚠️ warn |
| **Channel** | #anomaly |
| **Source** | `fill_tank_monitor.py`, main loop (~line 2449) |
| **Trigger** | `consecutive_failures >= MAX_CONSECUTIVE_PV_FAILURES` (5) on `read_valve_states()` |
| **Dedup** | Fires once then process aborts |

`read_valve_states()` reads manifold feed (`LNMx_FV:EN`), spare feed
(`LNMxA_FV:EN`), and ext fill (`LNTx_FV:EN`) PVs from the LN IOC.
Each successful read resets the counter to 0.  After 5 consecutive
failures, monitor-tanks sends the alert and sets `done = True`,
exiting the main loop.

#### PV-2 — Valve PV Read Failures (add-press)

| Field | Value |
|-------|-------|
| **Severity** | ⚠️ warn |
| **Channel** | #anomaly |
| **Source** | `fill_add_press.py`, main loop (~line 1115) |
| **Trigger** | `consecutive_failures >= MAX_CONSECUTIVE_PV_FAILURES` (5) on `read_valve_states()` |
| **Dedup** | Fires once then process aborts |

Identical logic to PV-1 but in the add-press main loop.  Same PVs,
same threshold, same abort behavior.  Code comment: `[PV-2]`.

**PV-1 and PV-2 produce identical messages** — they differ only in
which process fires them.  The USER_GUIDE merges their presentation.

#### CP-1 — IOC Health Check Failure

| Field | Value |
|-------|-------|
| **Severity** | ⚠️ warn |
| **Channel** | #anomaly |
| **Source** | `fill_check_pressure.py`, `_check_ioc_health()` (~line 503) |
| **Trigger** | All non-failed gauges return `None` from `read_all_gauges(batch=True)` |
| **Dedup** | Fires every invocation where all reads fail (no one-time flag) |

Runs on **every** cron invocation of check-press (every 5 minutes),
regardless of the adaptive rate gating that controls InfluxDB logging.
Gauges marked in `PRESS_FAIL` are excluded — they always return None
and don't indicate an IOC problem.

**Interaction:** CP-1 detects IOC outages between fills.  During fills,
PV-1/PV-2 detect the same problem via valve PV reads.

#### PF-2 — Pre-Fill LN IOC PV Read Failure Abort

| Field | Value |
|-------|-------|
| **Severity** | 🔴 red |
| **Channel** | #anomaly |
| **Source** | `fill_prefill_check.py`, `_build_live_hose_mapping()` |
| **Trigger** | Batched `caget_many` read of all 224 LN IOC PVs (112 `SM:SUB.E` + 112 `FV:EN`) returns >= `MAX_CONSECUTIVE_PV_FAILURES` (5) `None` values in a single batch |
| **Dedup** | Naturally one-shot — the whole batch is a single read; function returns early after the alert |
| **Format** | `🔴 LN IOC PV reads: {N} failed in one batch — IOC may be down. Aborting prefill.` |

The LN IOC PVs (manifold assignment `SM:SUB.E` and valve state `FV:EN`)
are critical for pre-fill decisions.  Without knowing which detector
is on which hose and whether its valve is AUTO, fill times cannot be
safely adjusted.  The pre-fill check aborts entirely.

**Interaction:** The fill proceeds without any pre-fill adjustments.
The post-fill adjuster will still run and apply normal corrections.

#### PF-3 — Pre-Fill Collector IOC PV Failure Notice

| Field | Value |
|-------|-------|
| **Severity** | ⚠️ warn |
| **Channel** | #anomaly |
| **Source** | `fill_prefill_check.py`, `_read_collector_batch()` + caller dispatch |
| **Trigger** | Batched `caget_many` read of `MODxxx_DV_EN` + `MODxxx_DV_TEMP` for the active GSIDs returns >= `MAX_CONSECUTIVE_PV_FAILURES` (5) `None` values in a single batch |
| **Dedup** | Naturally one-shot — the whole Collector batch is a single read |
| **Format** | `⚠️ Collector IOC PV reads: {N} failed in one batch — IOC may be down. Skipping matrix classification for affected detectors.` |

The Collector IOC is **separate** from the LN IOC.  Failures here do
NOT abort the pre-fill check.  Instead:
- DV_EN unavailable → matrix classification (PFM-1..5) skipped for the
  affected detectors (classifier returns None when ``enabled is None``)
- DV_TEMP unavailable → temperature bump (PF-1) skipped for the
  affected detectors

Hose reassignment (N5) still works because it depends on LN IOC PVs,
not Collector IOC PVs.  Sparse per-PV `None`s below the batch
threshold do NOT fire PF-3 — the individual VS/temp-bump checks
for those detectors are silently skipped.

#### Prefill batching optimization

The pre-fill check uses 4 batched `caget_many` reads up front (LN IOC
SM:SUB.E + FV:EN for all 112 positions, Collector IOC DV_EN + DV_TEMP
for every active GSID) instead of 224–448 sequential single PV reads.

Cold timing went from ~6–12 seconds of sequential single reads to
~750ms–1s of batched parallel I/O (typical 180–250ms per batch).
The per-PV default timeout (0.5s) stays as-is for long-running
processes; batched callers use `CAGET_MANY_TIMEOUT` (1.0s) for a
defensive margin on cold network paths.

---

### Pre-Fill Checks

#### PF-1 — Pre-Fill Temperature Bump

| Field | Value |
|-------|-------|
| **Severity** | ℹ️ info |
| **Channel** | #anomaly |
| **Source** | `fill_prefill_check.py`, per-GSID loop (~line 640) |
| **Trigger** | `temp_begin` valid and ≥1K above historical mean; detector unchanged (not new/moved) |
| **Dedup** | Batched: all PF-1 notices collected in `pf1_notices[]`, sent as one message |

The pre-fill bump formula is intentionally aggressive: `2× proportional
(20-60s) + 30s flat boost = 50-90s total adjustment`.  No cold_time
clamp is applied.  This overshoots slightly to avoid the much larger
cost of an extra mid-day auto-fill to service one underfilled detector.

The bump is temporary — stored in `prefill_backup` in JSON state.
The post-fill adjuster reads `prefill_backup` as the `old_min` baseline,
restores the original value, and applies the full correction with
budget tracking.  `prefill_backup` is always cleared by the adjuster.

**Interaction:** Mutually exclusive with N5 for the same GSID (new/moved
detectors take the hose reassignment path, not the temp bump path).

#### PFM matrix — prefill detector state classifier

Pure classifier (`_classify_prefill_state` in `fill_prefill_check.py`)
over the four-dimensional detector state `(DV_EN, hose, temp, valve)`.
Returns one of 5 message IDs (or None for silent / unclassifiable
cases).  Per-detector immediate dispatch from `run()` — no batching
at this layer, one Discord per affected detector.  Open valve is
handled by the orthogonal `_detect_open_valves_per_manifold` helper
(see PFM-OPEN below).

| Field | Value |
|-------|-------|
| **IDs** | PFM-1 (cold won't fill, RED), PFM-2 (warm will fill, YELLOW), PFM-3 (cold unmonitored, YELLOW), PFM-4 (warm monitored won't fill, anomalous info), PFM-5 (warm in array won't fill, system info) |
| **Source** | `fill_prefill_check.py::_classify_prefill_state()`; dispatch in `run()` per-detector loop |
| **Channel routing** | RED / YELLOW / anomalous-info → `#anomaly`; system-info → `#system-messages` |
| **Dedup** | None at this layer; one message per detector per prefill run.  Repeats every prefill until operator action resolves the underlying state. |
| **Skipped when** | `enabled is None` (DV_EN read failed — see PF-3); `dv_temp is None` OR temperature below `TEMP_MIN_VALID` (sensor-inactive sentinel, no classification). |

See USER_GUIDE.md “Prefill check — detector state matrix” for the
canonical 12-row table with message text and operator actions per ID.

**State → row → ID summary** (12 distinct cases):

| State                                          | Row | ID | Severity | Channel       |
|------------------------------------------------|----:|:---:|:--------:|:--------------|
| DV_EN=1 + hose + Cold + Auto                   |   1 | —     | silent    | (no message)  |
| DV_EN=1 + hose + Cold + Disable                |   2 | PFM-1 | RED       | #anomaly      |
| DV_EN=1 + hose + Warm + Auto                   |   3 | PFM-2 | YELLOW    | #anomaly      |
| DV_EN=1 + hose + Warm + Disable                |   4 | PFM-4 | info      | #anomaly      |
| DV_EN=1 + no hose + Cold                       |   5 | PFM-1 | RED       | #anomaly      |
| DV_EN=1 + no hose + Warm                       |   6 | PFM-4 | info      | #anomaly      |
| DV_EN=0 + hose + Cold + Auto                   |   7 | PFM-3 | YELLOW    | #anomaly      |
| DV_EN=0 + hose + Cold + Disable                |   8 | PFM-1 | RED       | #anomaly      |
| DV_EN=0 + hose + Warm + Auto                   |   9 | PFM-2 | YELLOW    | #anomaly      |
| DV_EN=0 + hose + Warm + Disable                |  10 | PFM-5 | info      | #system-messages  |
| DV_EN=0 + no hose + Cold                       |  11 | PFM-1 | RED       | #anomaly      |
| DV_EN=0 + no hose + Warm                       |  12 | PFM-5 | info      | #system-messages  |

**Notes on the classifier design:**
- **Open valve as Auto**: the matrix treats `valve_state == 'Open'`
  identically to `'Auto'`.  The orthogonal `PFM-OPEN` dispatch fires
  one manifold-level message for the actual Open valves, suppressing
  the per-detector Open noise.  The matrix continues to evaluate the
  underlying configuration question (would this fill correctly if not
  for the concurrent fill?) regardless.
- **Severity ↔ channel**: a single mapping rule at the dispatch site
  (`if verdict['channel'] == 'operational': send_discord_operational(...)
  else: send_discord_anomaly(...)`).  Adding a new severity tier or
  rerouting an existing one is one change to the classifier dict and
  zero changes elsewhere.
- **No matrix-row constants**: row IDs 1–12 are inline ints in the
  classifier return value (`verdict['row']`); used only by tests for
  diagnostic assertions.  Adding the matrix rows to `fill_constants.py`
  was considered and rejected — the rows are not referenced from
  multiple files.

#### PFM-OPEN — Concurrent M-fill manifold-level dispatch

| Field | Value |
|-------|-------|
| **Severity** | ⚠️ warn |
| **Channel** | #anomaly |
| **Source** | `fill_prefill_check.py::_detect_open_valves_per_manifold()`; dispatch in `run()` before the per-detector matrix loop |
| **Trigger** | One or more detector valves on a manifold read `Open` at prefill time |
| **Dedup** | One message per affected manifold per prefill run (not per Open detector).  Per-detector PFM-1..5 messages still fire for non-Open detectors. |
| **Format** | `⚠️ [PFM-OPEN] Manifold {letter}: prefill check ran while {N} valve{s} {is\|are} Open (positions {csv}) — possible concurrent M-fill. Per-detector states may be transient; review after current fill completes.` |

The most likely cause of an Open detector valve at prefill time is
a concurrent M-fill on the same manifold (operator started an M-fill
shortly before the prefill check ran).  Detector states sampled
during an M-fill are transient and not reliable inputs to the matrix
classifier, so we surface the situation at the manifold level and
let the operator decide whether to review per-detector states after
the current fill completes.

**Implementation:** `_detect_open_valves_per_manifold` returns a
dict `{manifold_letter: [sorted hose labels]}`; the dispatch loop in
`run()` formats and sends one message per manifold key.

#### VS-4 — Wrong-Side Hose Assignment

| Field | Value |
|-------|-------|
| **Severity** | ⚠️ warn |
| **Channel** | #anomaly |
| **Source** | `fill_prefill_check.py`, per-GSID loop |
| **Trigger** | `live_hose is not None` AND `valve_state == 'Auto'` AND `MANIFOLD_TO_TS[hose_letter] != _gsid_expected_ts(gsid)` |
| **Dedup** | Batched: all VS-4 notices collected in `vs4_notices[]`, sent as one message |
| **Format** | `⚠️ GS{gsid:03d}: assigned to hose {hose} (Manifold {X}, served by {TSn}) but detector is physically on {TSother}. Check hose mapping.` |

**TS / manifold mapping** (constants in `fill_constants.py`):
- `MANIFOLD_TO_TS = {'A': 'TS1', 'B': 'TS1', 'C': 'TS2', 'D': 'TS2'}`
- `_gsid_expected_ts(gsid)`: returns `'TS1'` if odd, `'TS2'` if even.

**Skipped when:** valve is not in AUTO (a mis-mapping on Disabled
is not immediately operational).  GSID out of range is also skipped
(it would never enter `live_hose_map` in the first place).

#### GF-1 — Missing GSID in gefilltime2.dat

| Field | Value |
|-------|-------|
| **Severity** | ℹ️ info |
| **Channel** | #anomaly |
| **Source** | `fill_prefill_check.py` (~line 530) and `fill_adjuster.py` (~line 621) |
| **Trigger** | GSID present in fill (or live EPICS mapping) but absent from gefilltime2.dat |
| **Dedup** | Batched in pre-fill; per-GSID in adjuster. Both fire to #anomaly |

Initial value priority: (1) `hose_min_time[hose]` — previous occupant's
fill time for this physical hose, (2) detector's own fill history from
JSON state, (3) `DEFAULT_MIN_TIME` (150s).

**Interaction:** Mutually exclusive with N5 for the same GSID.  If GF-1
adds the GSID with hose-based time, N5 won't also fire because the
min_time already matches hose history.

`flush-history` does NOT modify gefilltime2.dat — only the adjuster
and pre-fill check can add entries.

#### GF-2 — Missing gefilltime2.dat

| Field | Value |
|-------|-------|
| **Severity** | ⚠️ warn (pre-fill) / ℹ️ info (adjuster) |
| **Channel** | #anomaly |
| **Source** | `fill_prefill_check.py` (~line 393) and `fill_adjuster.py` (~line 496) |
| **Trigger** | `os.path.isfile(filltimes_path)` returns False |
| **Dedup** | One alert per invocation (pre-fill OR adjuster, not both in same run) |

**Pre-fill (⚠️):** Cannot operate without min_times.  Aborts, fill
proceeds with LNFill_App defaults.

**Adjuster (ℹ️):** Recovers from `min_time_history/` archive CSVs.
If no archive exists, creates with defaults.  Normal operation resumes.

---

### Fill Monitoring

These alerts fire during active fills from add-press or monitor-tanks.
Both share the same monitoring classes (`ManifoldTempTracker`,
`LowPressureTracker`, `ManifoldFailover`, `BrokenLineDetector`) via
`fill_tank_monitor.py`.

#### BL-1 — Broken Manifold Feed Line

| Field | Value |
|-------|-------|
| **Severity** | 🔴 red |
| **Channel** | #anomaly |
| **Source** | `fill_tank_monitor.py`, `BrokenLineDetector` class (~line 553+) |
| **Trigger** | ≥4 of last 6 eligible detectors on same manifold show no LN2 flow |
| **Automatic action** | Closes manifold feed valve (`LNMx_FV:EN → Auto`) |
| **Currently** | **Active** — `caput_fn=_caput`, `discord_fn=send_discord_anomaly` in `run_monitoring()`.  Kill switch: `--no-feedline-check` skips all BL-1 PV reads + alerts.  Suppressed for 240s per manifold after every successful tank failover (see *Failover-cooldown suppression* in the `BrokenLineDetector` section). |

**Algorithm detail:**

1. Monitors 28 valve states (`LNH{m}-{v}_FV:VM`) each poll cycle.
   Tracks fill duration via `time.time()` (no SM:SUB.D reads).

2. When valve open ≥180s, reads LED overflow voltage (`LNH{m}-{v}_TM:AT`).
   Verdict: LED ≥2.0V = flow, LED <2.0V = no-flow.

3. Maintains ~12 recent eligible detectors per manifold.  Active
   detectors (valve open) have live verdicts that update each poll.
   Finalized detectors (valve closed) have locked verdicts.

4. Sliding window: walks history most-recent-first, picks first 6 with
   valid LED (1.60V–5.86V).  Invalid LED → excluded (no slot consumed).

5. If ≥4 no-flow in window → fires immediately (doesn't wait for close).

**Eligibility:** fill time ≥180s AND valid LED voltage (1.60–5.86V).
Detectors with <180s fill or invalid LED are invisible to the window.

**Dynamic behavior:** Active detectors can flip verdict each poll.
Detectors can drop out if LED goes invalid, older history slides in.

**Activation:** Remove `caput_fn=None` and `discord_fn=None` overrides
in `run_monitoring()`.  Disabled entirely by `--no-feedline-check`.

#### BL-2 — Broken Feed Line at End of Priming (RED)

| Field | Value |
|-------|-------|
| **Severity** | 🔴 red |
| **Channel** | #anomaly |
| **Source** | `fill_tank_monitor.py`, `BrokenPrimingDetector` class |
| **Trigger** | At vent valve close: `max_vent_led < BL2_BROKEN_LINE_THRESHOLD` (2.0V) |
| **Automatic action** | Closes manifold main feed valve (`LNM{n}_FV:EN → Auto`) when activated |
| **Currently** | **INERT** — wired in `run_monitoring()` with `caput_fn=None`, `discord_fn=send_discord_anomaly`.  RED Discord fires with `[BL-2 IN DEVELOPMENT — NO ACTION TAKEN]` prefix; no caput.  Promotion to active = change `caput_fn=None` to `caput_fn=_caput` at the wiring site.  Suppressed entirely only by `--no-feedline-check`. |
| **Threshold rationale** | Set from 6 years of historical vent-close voltages: in clean recent operation only ~6 F-fill events/year fall below 2.0V and every one is a genuine broken-line or facility-wide LN2 supply failure (cross-confirmed by detector OVERTIME fills on the same manifold). |

See the `BrokenPrimingDetector` section above for the full algorithm,
arm/disarm rules, late-start handling, and once-per-fill mutual
exclusion with BL-3.

#### BL-3 — Manifold Not Primed (YELLOW)

| Field | Value |
|-------|-------|
| **Severity** | ⚠️ warn |
| **Channel** | #anomaly |
| **Source** | `fill_tank_monitor.py`, `BrokenPrimingDetector` class (shared with BL-2) |
| **Trigger** | At vent valve close: `BL2_BROKEN_LINE_THRESHOLD ≤ max_vent_led < BL3_UNPRIMED_THRESHOLD` (2.0–3.0V) |
| **Automatic action** | NONE — BL-3 is informational only; never closes valves regardless of `caput_fn`. |
| **Currently** | Live from day 1.  No inert/active distinction.  Suppressed entirely only by `--no-feedline-check`. |
| **User-facing tag** | NONE — Discord message does NOT carry `[BL-3]` (or any `[BL-N]`) tag.  The internal stderr log line uses `[BL-3]` for code-side tracing only. |
| **Threshold rationale** | Set from F-fill historical data: ~10 events/year at <3.0V (≈1/month).  Most are not broken-line failures but indicate slow / weak priming that may degrade detector fill performance.  Operator gets the YELLOW notice; fill is allowed to proceed. |

BL-3 fires from the same `BrokenPrimingDetector` evaluation that
produces BL-2; the two tiers are mutually exclusive per manifold per
fill (one detector instance, one allowed run, dispatch routes to RED
or YELLOW or silent based on `max_vent_led`).

#### N-LP — Low Tank Pressure (info)

| Field | Value |
|-------|-------|
| **Severity** | ℹ️ info |
| **Channel** | #anomaly |
| **Source** | `fill_tank_monitor.py`, `LowPressureTracker.check()` (~line 958) |
| **Trigger** | `pressure < LOW_PRESS_INFO_THRESHOLD` (20 psi) |
| **Dedup** | `(station, threshold)` tuple in `self._fired` set — once per station per fill |

Pressure checks only run while manifold feed valves are open.
Uses calibrated float pressure from the gauge cascade (no integer
truncation).

#### W-LP — Low Tank Pressure (warning)

| Field | Value |
|-------|-------|
| **Severity** | ⚠️ warn |
| **Channel** | #anomaly |
| **Source** | `fill_tank_monitor.py`, `LowPressureTracker.check()` (~line 973) |
| **Trigger** | `pressure < thresh` for each `LOW_PRESS_WARN_THRESHOLDS` (18, 15, 12, 9, 6, 3 psi) |
| **Dedup** | Same `(station, threshold)` set — each threshold fires once per station per fill |

Multiple thresholds can fire for the same station as pressure drops
(e.g., 18, then 15, then 12).

#### W-WM — Warm Manifold Detected

| Field | Value |
|-------|-------|
| **Severity** | ⚠️ warn |
| **Channel** | #anomaly |
| **Source** | `fill_tank_monitor.py`, `ManifoldTempTracker` |
| **Trigger** | Case 1: Cold ≥2 min then sustained Warm ≥10s. Case 2: Sustained Warm ≥180s after feed valve opened |
| **Dedup** | Once per manifold per fill |

Monitors `LNMxA_TM:BT` PVs ('Warm'/'Cold').  While any manifold is in
Warm state, sleep interval is capped at 2s (normally 30s).  Cached PVs
are checked every second during interruptible sleep for sub-second
Cold→Warm detection.

**Interaction:** W-WM triggers the failover decision logic (FO-1
through FO-5).  FO-6 confirms recovery.

#### FO-1 — Failover: Spare Feed Disabled

| Field | Value |
|-------|-------|
| **Severity** | 🔴 red |
| **Channel** | #anomaly |
| **Source** | `fill_tank_monitor.py`, `ManifoldFailover._do_failover()` (~line 1670) |
| **Trigger** | Main tank Warm + spare feed valve state is 'Disable' |
| **Dedup** | Once per failover attempt per manifold |

#### FO-2 — Failover: Main → Spare

| Field | Value |
|-------|-------|
| **Severity** | 🔴 red |
| **Channel** | #anomaly |
| **Source** | `fill_tank_monitor.py`, `ManifoldFailover._do_failover()` (~line 1723) |
| **Trigger** | Main tank Warm + spare feed available + `_failed_over == False` |

**Failover sequence:** (1) check target valve not Disabled, (2) close
current feed (set to Auto, not Disable), (3) open target feed,
(4) switch ext fill to empty tank, (5) reset warm tracker, (6) send alert.

After failover, ext fill pressure management moves to the main tank.
Add-press closes the ext fill valve within ~1s of manifold close.

#### FO-3 — Failover: Main Feed Disabled (spare-first)

| Field | Value |
|-------|-------|
| **Severity** | 🔴 red |
| **Channel** | #anomaly |
| **Source** | `fill_tank_monitor.py`, `ManifoldFailover._do_failover()` (~line 1670) |
| **Trigger** | Spare tank Warm (ManID≥5 config) + main feed valve Disabled |

Same as FO-1 but for spare-first fills.

#### FO-4 — Failover: Spare → Main

| Field | Value |
|-------|-------|
| **Severity** | 🔴 red |
| **Channel** | #anomaly |
| **Source** | `fill_tank_monitor.py`, `ManifoldFailover._do_failover()` (~line 1723) |
| **Trigger** | Spare tank Warm + main feed available + either initial spare-first or after FO-2 |

After failover, ext fill moves to the spare tank.

#### FO-5 — Both Feeds Exhausted

| Field | Value |
|-------|-------|
| **Severity** | 🔴 red |
| **Channel** | #anomaly |
| **Source** | `fill_tank_monitor.py`, `ManifoldFailover._do_failover()` (~line 1778) |
| **Trigger** | `_failed_over == True` (already tried alternate) + current feed Warm |

Closes feed valve.  Manifold is marked exhausted — no further failover
attempts.  Detectors on this manifold will not receive LN2.

#### FO-6 — Failover Confirmed

| Field | Value |
|-------|-------|
| **Severity** | ✅ info |
| **Channel** | #anomaly |
| **Source** | `fill_tank_monitor.py`, `ManifoldTempTracker` (~line 1095) |
| **Trigger** | Manifold temp transitions to Cold after a successful FO-2 or FO-4 |
| **Dedup** | Once per failover event |

Confirms liquid nitrogen is flowing from the new feed.  Resolves the
preceding red alert.

#### AD-1 through AD-5 — Ext Fill Valve Adaptation

| Field | Value |
|-------|-------|
| **Source** | `fill_tank_monitor.py`, `_resolve_ext_fill_valve()` (~lines 1853–1909) |
| **Channel** | #anomaly |
| **Trigger** | Ext fill valve selection changed due to tank in use or valve Disabled |

| ID | Severity | Condition |
|----|----------|-----------|
| AD-1 | ℹ️ info | Ext fill tank now in use as manifold feed — switched to alternate |
| AD-3 | ⚠️ warn | Ext fill valve Disabled — cannot manage pressure |
| AD-4 | ⚠️ warn | No available ext fill valve on station — pressure mgmt unavailable |
| AD-5 | ⚠️ warn | Default spare ext fill Disabled at startup — switched to alternate |

**Disabled valve policy:** The code NEVER writes to a Disabled valve.
Before every `_caput`, current state is read; if Disabled, write is
skipped.

#### EX-1 — Monitor-tanks Hard Timeout

| Field | Value |
|-------|-------|
| **Severity** | 🔴 red |
| **Channel** | #anomaly |
| **Source** | `fill_tank_monitor.py`, main loop (~line 2509) |
| **Trigger** | `run_time >= MAX_RUN_TIME` with manifold valves still open |

#### EX-2 — Parent Process Terminated

| Field | Value |
|-------|-------|
| **Severity** | 🔴 red |
| **Channel** | #anomaly |
| **Source** | `fill_tank_monitor.py`, main loop (~line 2538) |
| **Trigger** | `os.getppid()` changed (parent cron script died) with manifolds open |

#### EX-3 — AddPress Exiting (manifolds open)

| Field | Value |
|-------|-------|
| **Severity** | 🔴 red |
| **Channel** | #anomaly |
| **Source** | `fill_add_press.py`, exit handler (~line 1216) |
| **Trigger** | `ctrl.exit_manifolds_open == True` at add-press exit |

Re-reads valve states at exit to report which manifolds are open.
If the read fails, reports 'unknown'.

#### I-TO — Non-Critical Timeout

| Field | Value |
|-------|-------|
| **Severity** | ℹ️ info |
| **Channel** | #anomaly |
| **Source** | `fill_add_press.py`, exit handler (~line 1246) |
| **Trigger** | `run_time >= MAX_RUN_TIME` (2200s) AND all manifolds closed |
| **Code comment** | `[TO-1]` (code uses TO-1, docs use I-TO) |

Not an operational emergency — the fill completed normally but
add-press's exit condition didn't trigger promptly.

---

### Post-Fill Analysis

These alerts fire after a fill completes, during the `adjust` or
`report` pipeline.  The classifier writes per-fill alert data to
`current_fill_alerts` in JSON state; the reporter reads it, dispatches,
and clears the key so alerts are never re-sent.

**Batching:** The reporter collects all warnings into one #anomaly
message with header `Fill (YYYY-MM-DD HH:MM) — N warning(s):` and
all notices into one #system-messages message.  ML-1 and IL-2/IL-3
are dispatched separately by `check_fill_completeness()` before the
reporter runs.

#### W1 — OVERTIME

| Field | Value |
|-------|-------|
| **Severity** | ⚠️ warn |
| **Channel** | #anomaly (batched) |
| **Source** | Classifier: `fill_classifier.py`, `_classify_detector()`. Reporter: `fill_reporter.py`, `discord_per_fill()` |
| **Trigger** | `fill_time >= max_time` (detector hit the safety cutoff) |

The message includes end overflow voltage.  If `ovf < 5.6V`, the LED
was working but the fill was genuinely too short — "fill status
uncertain" is appended.  If `ovf >= 5.6V`, the LED sensor may be
faulty (always reads high).

**Interaction:** Overtime detectors still get min_time adjustments.
The adjuster applies the OVERTIME path: `new_min = min(max_time,
effective + 10)` (extends by 10s toward max_time).

#### W2 — Invalid/Duplicate GSIDs

| Field | Value |
|-------|-------|
| **Severity** | ⚠️ warn |
| **Channel** | #anomaly (batched) |
| **Source** | Classifier: `fill_classifier.py`, `_check_gsids()`. Reporter: `fill_reporter.py`, `discord_per_fill()` |
| **Trigger** | GSID < 1 or GSID > 110 (out of range, GSID_MIN/GSID_MAX in `fill_constants.py`) or same GSID on multiple hoses in one fill |

Unresolved duplicate GSIDs are excluded from all detector change
detection (N4) and are preserved in `known_detectors` during F-fill
replacement.

#### ML-1 — Missing or Unparseable Fill Log

| Field | Value |
|-------|-------|
| **Severity** | 🔴 red |
| **Channel** | #anomaly (sent individually, not batched) |
| **Source** | `fill_classifier.py`, `_check_log_exists()` (~line 950) |
| **Trigger** | `os.path.isfile(logfile)` returns False, OR file has no parseable timestamp header |

Two message variants: "Fill log missing" (file doesn't exist) vs
"Fill log empty/unparseable" (file exists but unreadable).

**Suppression:** `--no-missing-log-alert` flag (used in AUTO cron for
M-fills where most runs find no warm detectors and exit without
creating a log).

#### IL-2 — Incomplete Fill Log (Partial)

| Field | Value |
|-------|-------|
| **Severity** | 🔴 red |
| **Channel** | #anomaly (sent individually) |
| **Source** | `fill_classifier.py`, `check_fill_completeness()` (~line 1013) |
| **Trigger** | `parsed_log['complete'] == False` AND `parsed_log['detectors']` is non-empty |

Fill has detector data but missing "Total App Runtime" footer —
LNFill_App started filling but crashed or was killed.

#### IL-3 — Incomplete Fill Log (Empty)

| Field | Value |
|-------|-------|
| **Severity** | 🔴 red |
| **Channel** | #anomaly (sent individually) |
| **Source** | `fill_classifier.py`, `check_fill_completeness()` (~line 1020) |
| **Trigger** | `parsed_log['complete'] == False` AND `parsed_log['detectors']` is empty |

Fill started (log file created) but never reached detector filling —
all manifolds likely aborted during priming.

#### W3-a — Bad Temperature Readback (None start)

| Field | Value |
|-------|-------|
| **Severity** | ⚠️ warn |
| **Channel** | #anomaly (batched) |
| **Source** | Classifier: `fill_classifier.py`, `_classify_detector()`. Reporter: `fill_reporter.py`, `discord_per_fill()` |
| **Trigger** | `temp_begin is None` |

Always a warning regardless of `temp_end`.  Usually indicates the hose
position is not connected to a working temperature sensor, or the GSID
doesn't map to a real detector.

#### W3-b — Bad Temperature Readback (end reading ok)

| Field | Value |
|-------|-------|
| **Severity** | ℹ️ info |
| **Channel** | #system-messages (batched with notices) |
| **Source** | Same as W3-a |
| **Trigger** | `temp_begin` anomalous (non-None, ≤60K or >150K) AND `temp_end` valid (60–150K) |

Transient readback glitch.  The adjuster uses the valid end temperature.

#### W3-c — Warm Detector

| Field | Value |
|-------|-------|
| **Severity** | ⚠️ warn |
| **Channel** | #anomaly (batched) |
| **Source** | Same as W3-a |
| **Trigger** | `temp_begin` anomalous AND `temp_end` also invalid AND both >150K and <320K |

Both readings are high-side anomalous — the detector is physically
warmer than it should be.

#### W3-d — Sensor Fault (Abnormal Temperature)

| Field | Value |
|-------|-------|
| **Severity** | ⚠️ warn |
| **Channel** | #anomaly (batched) |
| **Source** | Same as W3-a |
| **Trigger** | `temp_begin` anomalous AND `temp_end` also invalid AND NOT both in >150K/<320K range |

Covers low-side anomalies (≤60K), extreme values (≥320K), and None
end readings.  Indicates persistent sensor/wiring failure.

**Temperature classification summary:**

| `temp_begin` | `temp_end` | Alert | Severity |
|--------------|-----------|-------|----------|
| None | any | W3-a | ⚠️ warn |
| ≤60K or >150K | 60–150K (valid) | W3-b | ℹ️ info |
| >150K, <320K | >150K, <320K | W3-c | ⚠️ warn |
| anomalous | anomalous (other) | W3-d | ⚠️ warn |
| valid | anomalous | (not reported) | — |

#### N2-a — Temperature Adjustment Applied

| Field | Value |
|-------|-------|
| **Severity** | ℹ️ info |
| **Channel** | #system-messages (batched) |
| **Source** | Adjuster: `fill_adjuster.py`, temperature override path. Reporter: `discord_per_fill()` enriches with min_before/min_after/adj from `weekly_log_sections` |
| **Trigger** | `temp_begin` valid AND ≥1K above historical mean (computed from `history[gsid]`, excluding current fill) |

The reporter matches `current_fill_alerts.temp_overrides` to the
adjuster-enriched `weekly_log_sections.temp_overrides` by gsid+ts,
so the Discord message includes specific values.

Bounded by temperature budget: 30s max, regenerates 5s per 12h
(full recovery in 72h).  Also bounded by fill-time floor:
`min_time >= fill_time + 10s`.

#### N2-b — Temperature Adjustment (Budget Exhausted)

| Field | Value |
|-------|-------|
| **Severity** | ℹ️ info |
| **Channel** | #system-messages (batched) |
| **Source** | Same as N2-a |
| **Trigger** | Same temperature condition as N2-a but `temp_adj` budget is 0 |

The fill-time floor may still have increased min_time independently.

#### W6 — Clamp Ceiling Reached

| Field | Value |
|-------|-------|
| **Severity** | ⚠️ warn |
| **Channel** | #anomaly (batched) |
| **Source** | Adjuster: `fill_adjuster.py`, clamp path. Reporter: `discord_per_fill()` |
| **Trigger** | `effective_min >= upper_clamp` where `upper_clamp = min(360, max_time × 0.9)` |

Per-fill alert (not weekly).  The adjuster wanted to increase min_time
further but the clamp prevented it.

#### N5 — Hose-Based Min Fill Time Reassignment

| Field | Value |
|-------|-------|
| **Severity** | ℹ️ info |
| **Channel** | #system-messages (batched in pre-fill; per-fill in adjuster) |
| **Source** | `fill_prefill_check.py` (~line 748) and `fill_adjuster.py` |
| **Trigger** | New or moved detector + `hose_min_time[hose]` exists and differs from current min_time |

**Interaction:** Mutually exclusive with GF-1 for the same GSID.
If a GSID was missing from gefilltime2.dat and added via GF-1, N5
does not fire because GF-1 already set the value from hose history.

#### N4 — Detector Configuration Changes

| Field | Value |
|-------|-------|
| **Severity** | ℹ️ info |
| **Channel** | #system-messages (batched) |
| **Source** | Classifier: `fill_classifier.py`, `_update_known_detectors()`. Reporter: `discord_per_fill()` |

| Sub-ID | Trigger |
|--------|---------|
| N4-a | GSID appears for the first time in any fill type |
| N4-b | Known GSID absent from F-fill (only F-fills trigger removal — they see all detectors) |
| N4-c | Known GSID on a different hose than previously recorded |

N4 replaces the former N1 (hose-centric).  `known_detectors` dict
tracks `{gsid_int: hose_str}`.  Unresolved duplicate GSIDs are excluded.

#### VV-1 — Marginal Vent-Valve Overflow

| Field | Value |
|-------|-------|
| **Severity** | ℹ️ info |
| **Channel** | #system-messages (batched with notices) |
| **Source** | Classifier: `fill_classifier.py`, vent-valve processing (~line 847). Reporter: `fill_reporter.py`, `discord_per_fill()` (~line 783) |
| **Trigger** | Vent-valve overflow voltage at end of fill: `4.00V ≤ ovf < 5.00V` |

Manifold was not fully primed at start of detector filling.  Marginal.

#### VV-2 — Warm Vent-Valve Overflow

| Field | Value |
|-------|-------|
| **Severity** | ⚠️ warn |
| **Channel** | #system-messages (batched with notices) |
| **Source** | Same as VV-1 |
| **Trigger** | Vent-valve overflow voltage at end of fill: `ovf < 4.00V` |

Manifold was not primed — warm delivery means longer detector fills
and potential OVERTIME outcomes.

**Note:** VV-1 and VV-2 are sent as notices (#system-messages), not
warnings (#anomaly), despite VV-2 having ⚠️ severity in the message.

#### VV-3 — Vent-Valve Bad LED

| Field | Value |
|-------|-------|
| **Severity** | ⚠️ warn |
| **Channel** | #system-messages (batched with notices) |
| **Source** | Classifier: vent-valve processing (~line 866). Reporter: `discord_per_fill()` (~line 792) |
| **Trigger** | Vent-valve overflow voltage > `LED_OPEN_FAULT_THRESHOLD` (5.86V) at start, end, or both |

Reports as BAD START, BAD END, or BAD BOTH.  Same threshold used for
detector overflow sensors (faulty/disconnected sensor).

#### VV-4 — Short Vent-Valve Priming

| Field | Value |
|-------|-------|
| **Severity** | ⚠️ warn |
| **Channel** | #anomaly (batched with warnings) |
| **Source** | Classifier: vent-valve processing (~line 877). Reporter: `discord_per_fill()` (~line 810) |
| **Trigger** | Vent-valve priming time < 60s |

Suspiciously fast — suggests priming overflow sensor is bad (falsely
reading cold).  The manifold may not actually be primed.

**Note:** VV-4 goes to #anomaly (warnings list), unlike VV-1/2/3 which
go to #system-messages (notices list).

---

### Weekly Summary

#### W5 — Weekly Faulty LN2 Sensors

| Field | Value |
|-------|-------|
| **Severity** | ⚠️ warn |
| **Channel** | #anomaly |
| **Source** | Reporter: `fill_reporter.py`, `discord_weekly()` (~line 860) |
| **Trigger** | Detector with ≥7 `led_faults` events in `pending_weekly_summary` (out of ~14 fills/week) |
| **Data source** | Classifier snapshots `led_faults` to `pending_weekly_summary` on Monday rollover |

Events are `led_faults` entries from the classifier (both live
and log sources merged).  The reporter applies the
`LED_FAULT_RULES` policy filter at render time so non-detector
LED entries are scoped per LED class (manifold-vent drops
HIGH/LOW; TS / tank / supply drop all faults).  Eight
possible fault types (five from the log-based path: `START_HIGH`,
`START_LOW`, `END_HIGH`, `END_LOW`, `START_SUSPECT`; plus three
from the live mid-fill polling path: `HIGH`, `LOW`, `SUSPECT`).
Aggregated by (hose, gsid) with fault type breakdown.  The weekly
Discord summary lists per-detector counts of each fault type
present that week.

The ≥7 threshold means ≥50% of fills had a faulty reading — this is a
chronic sensor problem, not a one-time glitch.  The adjuster uses hard
fallback mode for these detectors (limited to -1 downward adjustment
per fill).

#### N3 — Weekly Large Min-Time Changes

| Field | Value |
|-------|-------|
| **Severity** | ℹ️ info |
| **Channel** | #system-messages |
| **Source** | Reporter: `fill_reporter.py`, `discord_weekly()` (~line 880) |
| **Trigger** | `|min_time_end - min_time_start| > 60s` for any GSID during the past week |
| **Data source** | `min_time_start` and `min_time_end` from `pending_weekly_summary` |

Large increases usually follow temperature events; large decreases
follow a period of consistently short fills after sensor repair or
detector cooldown.

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

`--test` on add-press and monitor-tanks implies `--no-discord` and
`--no-influxdb`.

---

## State File

`logs/fill_monitor/fill_monitor_state.json` — shared by all components.

### Classifier-owned keys

| Key | Description |
|-----|-------------|
| history | Per-detector fill history (up to 20 entries) for temp mean |
| known_detectors | Dict {gsid_int: hose_str} tracking currently installed detectors. Any fill type can add/update entries. Only F-fills replace the entire dict (enabling removal detection). Unresolved duplicate GSIDs are excluded and preserved on F-fill replacement. |
| hose_min_time | Dict {hose_str: min_time_int} — last known min_time per hose. Updated by the adjuster after every fill. Used by pre-fill check and adjuster for hose-based reassignment when a detector moves to a different hose. |
| last_fill_time | Per-GSID timestamp of most recent fill |
| current_week_start | Monday 00:00 of current log week |
| last_classified | Last classified filename (idempotency) |
| last_fill_ts | ISO timestamp of most recently classified fill |
| current_fill_alerts | Per-fill alert data (current fill only, cleared by reporter) |
| pending_weekly_summary | Outgoing week snapshot (cleared by reporter) |

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

### Pre-fill-adjust-check-owned keys

| Key | Description |
|-----|-------------|
| prefill_backup | Original min_times before pre-fill bump {gsid: min_time}. Read by adjuster as old_min baseline, always cleared by adjuster after use. |

### AddPress-owned keys

| Key | Description |
|-----|-------------|
| addpress_started | ISO timestamp written on AddPress entry (tells check-press to hold off). Not written by monitor-tanks. |
| addpress_finished | ISO timestamp written on AddPress exit (triggers check-press high-rate sampling). Not written by monitor-tanks. |

### weekly_log_sections (split ownership)

| Sub-key | Owner |
|---------|-------|
| led_faults (open / short / suspect LED fault events from live + log paths), overtime, bad_temp, invalid_gsids, duplicate_gsids, hose_changes (JSON audit only), gsid_hose, detector_changes | Classifier |
| temp_overrides | Both (classifier writes partial, adjuster enriches) |
| clamped, min_time_start, min_time_end, min_time_lo, min_time_hi | Adjuster |
| low_pressure, warm_manifold | AddPress |

---

## Configuration

### gefilltime2.dat

```
# Ge Fill times (GS ID / min time / max time)
1,139,419
42,201,419
```

CSV: GSID, min_time (seconds), max_time (seconds). Read and written
exclusively by the adjuster.

**Missing file recovery:** If gefilltime2.dat does not exist, the
adjuster recovers last known min_times from min_time_history archive
CSVs. Falls back to DEFAULT_MIN_TIME (150) for GSIDs not in any
archive.

### Min-time history CSV

`logs/fill_monitor/min_time_history/min_time_YYYYMM.csv` — monthly,
written by adjuster after each fill.

```csv
timestamp,gsid,hose,min_time,fallback
2026-05-13 06:00,42,D- 7,201,
2026-05-13 06:00,56,C-24,221,hard
```

Used by plotter in archive mode and by adjuster for recovery.

---

## Directory Structure

```
lnFill/
├── logs/                              ← all log output
│   ├── fill_YYYYMMDD_HHMM.log         ← per-fill detector logs (LNFill_App.py)
│   ├── fill_YYYYMMDD_HHMM.error.log   ← per-fill error logs
│   └── fill_monitor/                  ← fill monitor output
│       ├── fill_monitor_state.json    ← shared state
│       ├── fillmon_YYYYMMDD.log       ← weekly monitoring logs
│       └── min_time_history/
│           └── min_time_YYYYMM.csv    ← monthly min_time archive
├── cron_logs/                         ← cron stdout/stderr captures
│   ├── LNFill_cron.log                ← main fill cron output
│   ├── LNFill_Auto_EFill_cron.log     ← emergency fill cron output
│   ├── AddPress.log                   ← add-press output
│   ├── check_pressure.log             ← hourly pressure snapshot
│   ├── SaveTemp.log                   ← temperature logging output
│   └── archive/                       ← monthly rotated logs
│       └── AddPress_YYYYMM.log        ← etc.
└── influx_txt/                        ← InfluxDB line protocol files
    ├── AddPress.txt                   ← pressure during fills
    ├── check_pressure.txt             ← hourly pressure snapshot
    ├── fillTime.txt                   ← per-detector fill durations (parse_fill_log.py)
    ├── temp.txt                       ← detector temperatures (SaveTemp.py)
    └── temp_ping.txt                  ← host ping status (LNFill_ping_cron.sh)
```

---

## InfluxDB Integration

All pressure and fill data is pushed to InfluxDB (v3) for Grafana
trending. Scripts write InfluxDB line protocol to files in `influx_txt/`,
then `push_fill.sh` POSTs each file via HTTP.

### Push pipeline

```
script → writes influx_txt/<name>.txt → push_fill.sh → HTTP POST → InfluxDB
```

Endpoint: `http://192.168.203.56:8181/api/v3/write_lp?db=HPGeTemp`
(dcs2, port 8181, database HPGeTemp).

Each file is overwritten (not appended) at the start of each run.
`push_fill.sh` is a shared bash helper that handles the POST and
error reporting.

### Data sources

| File | Writer | Frequency | Content |
|------|--------|-----------|----------|
| `AddPress.txt` | fill_add_press.py | Every fill (concurrent) | All 8 pressure gauges, every iteration (~1-30s) |
| `check_pressure.txt` | fill_check_pressure.py | Hourly (cron) | All 8 pressure gauges, single snapshot |
| `fillTime.txt` | parse_fill_log.py | Every fill | Per-detector fill durations |
| `temp.txt` | SaveTemp.py | Every 10 min | Detector temperatures |
| `temp_ping.txt` | LNFill_ping_cron.sh | Periodic | Host ping status |

### Suppression

`--no-influxdb` flag on `add-press` and `check-press` writes the line
protocol file but skips the `push_fill.sh` call. `--test` on `add-press`
implies `--no-influxdb`.

---

## Dependencies

- Python 3.9+ (uses `zoneinfo`)
- matplotlib (plotting only)
- add-press live mode requires pyepics (via fill_interfaces).
  The verify subcommand does not require pyepics.

pyepics operates in preemptive callback mode by default: a background
CA thread processes network responses and fires monitor callbacks
automatically.  PV objects are thread-safe in this mode — each thread
automatically attaches to the main CA context.  No external lock is
required for concurrent PV reads or writes.

The legacy `pv_cache.py` and `pvlock.py` modules (used by
`LNFill_App.py`, `SaveTemp.py`, `LNValve.py`) duplicate the built-in
`epics.get_pv()` cache and add an unnecessary threading lock.
`fill_monitor` does not use them — it uses `epics.get_pv()` directly
via `fill_interfaces.py`.

---

## Potential Integrations

Several standalone scripts share data with fill_monitor. Evaluation:

### parse_fill_log.py

Parses fill log for InfluxDB push. Classifier already has all the data.
**Recommendation:** Leave as-is — low gain, adds GeID dependency.

### LNFill_check.sh

Parses cron log (not fill log) for fill duration alerts.
**Recommendation:** Leave as-is — different domain (cron orchestration).

### SaveTemp.py

Reads detector temperatures every 10 minutes for InfluxDB.
**Recommendation:** Leave as-is — independent, no functional benefit.

---

## Simulation and Plotting

### Production mode

The adjuster processes a single fill log and writes updated min_times
back to gefilltime2.dat. State persists in JSON between cron
invocations. Weekly log sections accumulate across fills within the week.

```bash
python3 -m fill_monitor adjust --logfile logs/fill_20260512_0600.log
```

### Simulation replay (via plotter)

Accessed through `fill_monitor plot` (default mode). Replays the full
adjuster through all historical logs — one fresh classifier+adjuster
per fill, state via JSON each cycle, exactly like production. Sim
output goes to a temp directory (auto-deleted unless `--keep-sim`).

In simulation, fill duration = `max(min_time, cold_time) + rand(0,3)`.
Bad cold_times (< 80s with valid ovf_end) are proxied with open_time.

```bash
python3 -m fill_monitor plot --all --outdir plots/ \
    --logdir logs/ --filltimes gefilltime2.dat --keep-sim
```

### Archive mode (--archive, recommended for production)

Reads min_time history from CSVs + fill data from logs. No simulation.

```bash
python3 -m fill_monitor plot --archive --all --outdir plots/ --logdir logs/
```

### Plot legend

- **Green dots**: Effective time (NORMAL fills)
- **Orange dots**: Effective time (EXTENDED fills)
- **Red dots**: Effective time (OVERTIME fills)
- **Hollow circles**: Historical fill time (sim mode only)
- **Red X**: Hard fallback (sensor invalid)
- **Purple X**: Soft fallback (cold_time below 80% threshold)
- **Blue line**: Min time
- **Dashed blue line**: Target (min + 20)
