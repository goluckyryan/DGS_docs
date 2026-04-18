# lnfill — Liquid Nitrogen Filling System

## What It Is

Automated control system for filling germanium detector **dewars** with liquid nitrogen (LN). Ported from DGS1 (Scientific Linux 6 / Python 2) to Debian 13 / Python 3. Controls valves, monitors dewar fill level, runs scheduled fills, and sends alerts via Discord.

---

## Physical System

**4 LN Tanks:**
- Tank A, B, C, D
- Tank B pressurizes Tanks A and D ✅ verified 2026-04-07 — `lnfill/README.md:L14`
- Tanks A and D supply LN to the manifolds ✅ verified 2026-04-07 — `lnfill/README.md:L14`

**4 Manifolds:**
- 2 per side (upper + lower)
- Each manifold: **28 solenoid valves** + LED sensors ✅ verified 2026-04-06 — `DetValve.py:L25` (`valve # 1-28`) + `DetMan.py:L134` ("the 28 detector valves")
- Each valve+LED pair → one detector dewar
- Fill detection: LN flowing past LED sensor changes LED resistance → system detects "full" ✅ verified 2026-04-07 — `lnfill/README.md:L19`

**Max 4 dewars filling simultaneously per manifold** (16 total at once across 4 manifolds) ✅ verified 2026-04-06 — `DetMan.py:L253` (`while len(Man)>0 and i<4`)

---

## Computers in the System

| Computer | OS | IP | Role |
|----------|----|----|------|
| ln2con | Fedora 12 | 192.168.203.148 | Boot host for IOC; runs logrotate |
| pi5 | Debian 13 | 192.168.203.58 | Main fill control (runs LNFill_App.py) ✅ verified 2026-04-07 — LNFill_ping_cron.sh:L19 + README.md:L25 |
| lnfill | IOC | 192.168.203.121 | EPICS IOC for valve/sensor hardware |
| dcs2 | — | DCS2.onenet | Runs ping health checks + pi5 health check crons |

> **pi5 in this context is the LN fill control Pi**, separate from the general pi5-dgs (Ryan's setup Pi).

---

## Key Files

| File | Role |
|------|------|
| `LNFill_App.py` | Main fill control app — manages all 4 manifolds concurrently |
| `LNFill_cron.sh` | Scheduled full-system fill (6am + 6pm daily) ✅ verified 2026-04-18 — live pi5-lnFill crontab (`00 06,18 * * *`); README.md was stale (said 07,19) |
| `LNFill_Auto_EFill_cron.sh` | Auto emergency fill — runs every 15 min, fills warm detectors ✅ verified 2026-04-06 — `lnfill/README.md:L106,L111` (`*/15 * * * *`) |
| `SaveTemp.sh` | Records temperatures every 10 min; pushes to InfluxDB on DCS2 ✅ verified 2026-04-07 — README.md:L107,L112 |
| `LNFill_ping_cron.sh` | Health check: pings all hosts, reports to InfluxDB + Discord (runs on DCS2, every 12 min) ✅ verified 2026-04-07 — `lnfill/README.md:L116,L122` (`*/12 * * * *` on `dcsu@DCS2.onenet`) |
| `LNFill_pi5_check.sh` | Checks LNFill_App.py is running at 6:15 and 18:15 (runs on DCS2) ✅ verified 2026-04-18 — live DCS2 crontab (`15 6,18 * * *`); README.md was stale (said 7,19) |
| `EPICS_para.sh` | Sets EPICS environment variables |
| `DetMan.py` | Detector manager |
| `DetValve.py` | Valve control |
| `TankMan.py` | **Tank manifold manager** — controls LN supply tanks. Two tank manifolds: TMan1 (feeds manifolds A+B, tanks T1/T2/T3) and TMan2 (feeds manifolds C+D, tanks T4/T5/T6). T1→feeds A, T2→feeds B, T3=spare; T4→feeds C, T5→feeds D, T6=spare. ✅ verified 2026-04-12 — `TankMan.py:L19` Each manifold has 1 supply vent valve + 3 × (feed valve + vent valve) per tank. `FillTanks()`: (1) open+cool supply vent (waits for `Cold` sensor or timeout=500s); (2) opens up to **2 tanks simultaneously** (feed+vent per tank), waits until sensor goes Cold or timeout (default 3000s each); T3/T6 (spare) opened if T1/T2 or T4/T5 finish early (`i<2 and Topen[2]==None` check) ✅ verified 2026-04-12 — `TankMan.py:L164-175`; (3) closes all. `CloseAllValves()` called on init and in `LNFill_Stop.py`. |
| `LNValve.py` | **Valve abstraction base class** — wraps a single EPICS-controlled solenoid valve with sensor monitoring. 7 valve types: `SPLY` (supply vent), `TNKF`/`TNKV` (tank feed/vent), `MANF`/`MANS`/`MANV` (manifold feed/spare/vent), `DET` (detector). PV naming pattern: `LNS{n}` (supply), `LNT{n}` (tank), `LNM{n}` (manifold), with suffixes `_VV:EN` (valve enable), `_FV:VM` (valve state), `_SM:SUB.D/.E` (fill time sub-record), `_TM:BT/.AT` (sensor before/after). Default max open times: SPLY=300s, TNKF/TNKV=3600s, MANF/MANS=2500s, MANV=500s, DET=600s. Default min open times: SPLY=100s, TNKF/TNKV=300s, MANF/MANS/MANV=50s, DET=50s. ✅ verified 2026-04-13 — `LNValve.py:L41-42` (`mxotdef` + `mnotdef` dicts) Key methods: `Open()`, `Close()`, `GetStatSens()` (returns "Warm"/"Cold"/"Fault"), `GetState()`, `SetPVOpenTime()`, `getOpenTime()`. Uses `pv_lock` (from `pvlock.py`) + `get_pv()` (from `pv_cache.py`) for thread-safe CA access. |
| `pvlock.py` | Shared `threading.Lock()` (`pv_lock`) for serializing EPICS PV access across modules ✅ verified 2026-04-17 — `pvlock.py:L4` (`pv_lock = threading.Lock()`) |
| `pv_cache.py` | Thread-safe PV object cache (double-checked locking pattern); `get_pv(name)` returns a reused `epics.PV` instance, creating it once per name to avoid redundant CA connections ✅ verified 2026-04-17 — `pv_cache.py:L9-16` (first check outside lock, then acquire lock, then check again inside = double-checked locking) |
| `LNFill_Stop.py` | Emergency stop: kills running `LNFill_App.py` processes (SIGKILL), then closes all manifold + tank valves via `DetMan.CloseAllValves()` + `TankMan.CloseAllValves()`, writes log + error file |
| `LNFill_closeValves.py` | Closes all 4 manifold detector valves (manifolds 1–4) without killing `LNFill_App.py` — safer than Stop for mid-run valve reset; runs from `/home/dgs/lnFill/` with aarch64 EPICS libs hard-coded ✅ verified 2026-04-17 — `LNFill_closeValves.py:L6-8,11` (`PYEPICS_LIBCA=/usr/lib/aarch64-linux-gnu/libca.so`; `BASE_DIR = "/home/dgs/lnFill"`) |
| `LNFill_check.sh` | Fill status check |
| `WriteDiscordMessage.py` | Sends Discord notifications |
| `gefilltime2.dat` | **Per-detector LN2 fill time bounds** — 113 lines, format: `GS_ID, min_time_sec, max_time_sec`. Covers GS 1–110 + detectors 201 (DUO?) and 501. Loaded at startup by `LNFill_App.py:L333-344` into `geminfilltime[]` / `gemaxfilltime[]` arrays (default 151s min / 575s max if file absent ✅ verified 2026-04-12 — `LNFill_App.py:L30-31`). Example: GS 1–109 mostly 139–419s; GS 108=100–419s; GS 110=120–419s. Values set from 2020-03-28 fill session (per file header). Passed to each `DetMan` via `setDetMinFillTimes()` / `setDetMaxFillTimes()`. **Vacuum health indicator:** fill time trending shorter = vacuum degrading = detector warming faster = possible vacuum leak. See QUEUE.md LN2 fill time monitoring task. |
| `templog/` | Temperature log directory |
| `AddPress.sh` | **Spare tank pressure management** — monitors tank pressures during a fill; opens/closes spare tank fill valves (LNT3, LNT6) to keep both tank stations pressurized. Runs for up to 2,200s; exits when fill completes or timeout. See details below. |
| `setTNF.sh` | **Set Time of Next Fill** — writes `LN_TTNF:XC` PV with the next scheduled fill time. Called by `LNFill_cron.sh` after each fill. Logic: if current hour >10 → next fill is 12h earlier (morning→evening or evening→morning); if arg given, uses that hour directly. Format: `HH:01`. ✅ verified 2026-04-13 — `setTNF.sh:L7-29` |
| `epics_cron.sh` | **EPICS environment for cron** — sets `EPICS_BASE`, `PYEPICS_USE_SYSTEM_LIBS`, `PYEPICS_LIBCA/COM`, and `LD_LIBRARY_PATH` for aarch64 Pi. Sourced by cron jobs that need pyepics. ✅ verified 2026-04-13 — `epics_cron.sh:L2-6` |
| `clean.sh` | **Log file cleanup** — deletes fill log files under 80 bytes (aborted/empty runs) and removes stray `core.*` dump files from `~/` and `logs/`. ✅ verified 2026-04-13 — `clean.sh:L4-8` |

---

## LNFill_App.py — Fill Types

| Arg | Mode | Description |
|-----|------|-------------|
| F | Full fill | Fill all manifolds + all tanks |
| M | Monitor fill | Check temperatures, build warm detector list, fill warm ones |
| L | List fill | Fill specific detectors by GS ID list |
| T/A/B/C/D | Selective | Fill selected manifolds or specific tank |

**Pre-fill setup (in `LNFill_cron.sh` and `LNFill_Auto_EFill_cron.sh`):**
- Both cron scripts explicitly **disable `LNH1-20_FV:EN`** via `caput` before calling `LNFill_App.py`. This is a hardcoded valve disable applied before every fill (both scheduled and emergency). `LNH1-20` maps to manifold 1, position 20 (GS detector 20, fill bounds 139–419 s per `gefilltime2.dat`). Reason not documented in code — likely a known-bad valve or stuck hose. ✅ verified 2026-04-18 — `LNFill_cron.sh:L14`, `LNFill_Auto_EFill_cron.sh:L13`
- `LNFill_cron.sh` also: runs `setTNF.sh` (set next fill time), `clean.sh` (delete empty logs + core.* files), and launches `AddPress.sh` in background before calling `LNFill_App.py F`. After fill ends, it calls `source LNFill_check.sh` (not a subshell — runs in same process) to check fill duration. If the output log file is empty at the end, posts anomaly alert to Discord: `'{outfile} is empty at the end of the LNFill_cron.sh. Something wrong with the fill.'` ✅ verified 2026-04-18 — `LNFill_cron.sh:L13-48`
- `LNFill_Auto_EFill_cron.sh` (M-mode) is simpler: disable LNH1-20, run `LNFill_App.py M`, post the fill log file to Discord webhook if it exists. No check script, no AddPress. ✅ verified 2026-04-18 — `LNFill_Auto_EFill_cron.sh:L13-20`

**Flow summary:**
1. Check for existing LNFill_App.py instance (abort or kill old one) — see priority rules below
2. Abort if less than 30 min to next scheduled fill (for non-F modes) — reads `LN_TTNF:XC` PV ✅ verified 2026-04-17 — `LNFill_App.py:L195-219`
3. Check fill status = Ready (abort if `LN_MODE:XC` ≠ `Ready`) ✅ verified 2026-04-17 — `LNFill_App.py:L221-229`
4. For M-mode: run `CheckTemps()` first; if no warm detectors, exit without killing any other instance ✅ verified 2026-04-17 — `LNFill_App.py:L238-244`
5. Decode fill type → build target dewar list
6. Spawn one thread per manifold (1–4), fill up to 4 dewars each concurrently
7. Wait for all manifold threads to finish
8. Close all manifold valves
9. If filling tanks: spawn tank fill threads, wait, close tank valves
10. Write fill statistics to log
11. If mode=M: wait 15 min for temps to stabilize (so M-mode cron doesn't re-trigger too quickly) ✅ verified 2026-04-17 — `LNFill_App.py:L531-532`
12. Push fill data to InfluxDB + send Discord notification

**Instance priority / kill logic (`kill_filtered_instances()`):** ✅ verified 2026-04-17 — `LNFill_App.py:L57-142`

When a new `LNFill_App.py` starts, it scans all running instances and applies these rules:

| New fill type | Existing instance arg | Action |
|---------------|-----------------------|--------|
| F | any non-F | Kill old; close valves (safety) |
| F | F | Abort new (another full fill already running) |
| T | F or T | Abort new |
| T | anything else | Kill old; close valves |
| M | F, T, or M | Abort new |
| M | anything else | Kill old; close valves |
| L/A/B/C/D | anything | Abort new |

The `closeValves` flag is set when killing an existing instance that may have had valves open — `DetMan.CloseAllValves()` is called before the new fill begins.

**`CheckTemps()` — M-mode warm detector detection:** ✅ verified 2026-04-17 — `LNFill_App.py:L625-695`

Reads `MOD{NNN}_DV_TEMP`, `MOD{NNN}_DV_TEMP.HIGH`, and `MOD{NNN}_DV_EN` for GS 1–110. Limits:
- `hardLimit = 0.0` K above HIGH alarm → **fill immediately** ✅ verified 2026-04-18 — `LNFill_App.py:L244`
- `softLimit = 3.0` K below HIGH alarm → **watch (templist)** — added to fill list only if ≥1 detector already in hard-limit list ✅ verified 2026-04-18 — `LNFill_App.py:L245`
- `maxLimit = 100.0` K — upper guard against sensor spikes / warm/unconnected detectors ✅ verified 2026-04-18 — `LNFill_App.py:L246`

During M-fill startup, `CheckTemps()` runs *before* the instance kill check — so if no detector is warm, the new M instance exits without disturbing any existing fill.

**`makePVlist()` — PV enumeration:** Generates all 168 manifold control PVs (`LNH{1-4}-{01-28}` × 6 suffixes: `_SM:SUB.E/D`, `_FV:EN`, `_FV:VM`, `_TM:BT`, `_TM:AT`) plus 220 detector PVs (`MOD{001-110}_DV_TEMP` + `MOD{001-110}_DV_EN`). Used for prefetching/caching PV connections at startup. ✅ verified 2026-04-17 — `LNFill_App.py:L549-562`

---

## Cron Jobs

### On pi5 (`/home/dgs/lnFill/`)

```sh
00 06,18 * * *   LNFill_cron.sh                  # Full fill at 6am + 6pm CDT ✅ verified 2026-04-18 — live pi5-lnFill crontab
*/15 * * * *     LNFill_Auto_EFill_cron.sh        # Emergency fill every 15 min
*/10 * * * *     SaveTemp.sh                      # Record temps every 10 min
```

### On DCS2 (`dcsu@DCS2.onenet`, `/home/phy/dcsu/lnFill/`) ✅ verified 2026-04-16 — live `crontab -l` on DCS2

```sh
*/12 * * * *     LNFill_ping_cron.sh              # Ping all hosts every 12 min (ACTIVE)
15 6,18 * * *    LNFill_pi5_check.sh              # Check LNFill_App.py is running @ 1:15am/pm CDT (ACTIVE)
```

> **Note (2026-04-17):** `LNFill_cron.sh`, `LNFill_Auto_EFill_cron.sh`, and `SaveTemp.sh` are **commented out** in the DCS2 crontab. They run from **pi5-lnFill** (192.168.203.58) directly. ✅ verified 2026-04-18 — `pi5-lnFill` crontab (via `DCS2.onenet` SSH hop): `LNFill_cron.sh` runs at 00 06,18 daily (old 16:44 entry commented out), `LNFill_Auto_EFill_cron.sh` runs every 15 min, `SaveTemp.sh` runs every 10 min — all in `/home/dgs/lnFill/`.

---

## Health Monitoring

### Ping Check (`LNFill_ping_cron.sh`, on DCS2)
- Pings 8 hosts: .148 (ln2con), .58 (pi5), .121 (lnfill IOC), .154 (piserver1), .88 (gs-cne), .149 (gs-cnw), .42 (gs-cse), .26 (gs-csw) ✅ verified 2026-04-09 — `LNFill_ping_cron.sh:L10` + DNS lookup from DCS2
- For pi5: uses **SSH** instead of ping (catches OS-broken-but-network-up failures) ✅ verified 2026-04-13 — `LNFill_ping_cron.sh:L23-24` (`if [ "${ip}" = "${pi5_ip}" ]; then # SSH check`)
- When SSH succeeds: also records `mem_available_mb` → Grafana trend ✅ verified 2026-04-09 — `LNFill_ping_cron.sh:L24-26`
- On SSH failure: Discord alert to anomaly channel — exact message: `"⚠️ pi5-lnFill (<ip>) is unreachable via SSH at <date>"` ✅ verified 2026-04-11 — `LNFill_ping_cron.sh:L34-36`

### Pi5 Health Check (`LNFill_pi5_check.sh`, on DCS2)
- Runs at **1:15 AM and 1:15 PM CDT** (`15 6,18 * * *` UTC) ✅ verified 2026-04-16 — `dcsu@DCS2` crontab
- SSHes into `pi5-lnFill`, checks if `LNFill_App.py` is running via `pgrep -f LNFill_App.py` ✅ verified 2026-04-16 — `LNFill_pi5_check.sh:L11-12`
- On failure: Discord alert to anomaly channel (`discord_anomaly.WebHook`) — exact message: `"⚠️ pi5-lnFill: LNFill_App.py is NOT running at <date>"` ✅ verified 2026-04-16 — `LNFill_pi5_check.sh:L15-17`

### Fill Duration Alert (`LNFill_check.sh`, on DCS2)
- **Not currently in crontab** — must be invoked manually or via another mechanism ✅ verified 2026-04-17 — `dcsu@DCS2` crontab only contains `LNFill_pi5_check.sh`
- Reads `LNFill_cron.log`, finds last Fill Begin/End pair, computes duration in minutes ✅ verified 2026-04-17 — `LNFill_check.sh:L11-40`
- If duration < threshold (default 10 min), sends Discord alert to anomaly channel: `"Last Fill Duration Alert: <X> minutes @ <begin_ts>"` ✅ verified 2026-04-17 — `LNFill_check.sh:L79`
- Threshold is configurable via first argument: `LNFill_check.sh <THRESH_MIN>` ✅ verified 2026-04-17 — `LNFill_check.sh:L17` (`THRESH_MIN=${1:-10}`)
- Uses `discord_anomaly.WebHook` for alerts ✅ verified 2026-04-16 — `LNFill_check.sh` (full script read from DCS2)

---

## AddPress.sh — Spare Tank Pressure Manager

_Source: `lnfill/AddPress.sh` v2.3 (M. Oberling, 2024-06-28)_

Runs alongside a fill to keep tank station pressures high by opening spare (T3) tank fill valves when advantageous. Monitors **TS1** (manifolds A+B) and **TS2** (manifolds C+D) independently.

**PVs monitored:**
- `LNP1-01_PR:AP` / `LNP2-01_PR:AP` — external supply pressure (TS1/TS2)
- `LNP1-02..04_PR:AP` / `LNP2-02..04_PR:AP` — tank 1–3 pressures per station
- `LNM1_FV:EN` – `LNM4_FV:EN` — manifold fill valve state (Open/Auto)
- `LNT3_FV:EN` / `LNT6_FV:EN` — spare tank fill valves (controlled by this script)

**Logic (per tank station):**
1. Only active if at least one manifold valve is Open (i.e., fill is in progress)
2. Opens spare fill valve if: ext pressure − tank pressure ≥ 3 PSI AND tank pressure < 32 PSI
3. Closes spare fill valve if: differential ≤ −1 PSI OR tank pressure ≥ 32 PSI
4. Cross-station coordination: if both spare valves are open, closes one if that station is already ahead by ≥2–3 PSI
5. Valve holdoff timers prevent rapid cycling: 120s after max-pressure close, 60s after differential close
6. Adaptive sleep: starts at 1s, grows to 30s max; resets to 0s when a valve opens
7. Exits when all manifold valves close or `MAX_RUN_TIME` (2,200s) reached; leaves spare valves in Auto

**Known failed sensors (hardcoded v2.3):** `PRESS_EXT2_FAIL=1`, `PRESS_TS2_T2_FAIL=1`, `PRESS_TS2_T3_FAIL=1` ✅ verified 2026-04-13 — `AddPress.sh:L49,L54,L55`

**Gauge calibration (v2.3):** `PRESS_TS1_T3_CAL=+2` (reads 2 PSI low); all others = 0 ✅ verified 2026-04-13 — `AddPress.sh:L42` (`PRESS_TS1_T3_CAL=2`)

Fallback: if all pressure gauges for a station fail, assumes 28 PSI (early in run) or 20 PSI (after 400s).

---

## Communications

### InfluxDB (on DCS2.onenet)
- `SaveTemp.sh` + `templog/StoreDetTemps.py` push temperature data (measurement: `Temperature,gsid=NNN,en=0/1 value=<K>`) to InfluxDB db `HPGeTemp` at `http://192.168.203.56:8181/api/v3/write_lp` ✅ verified 2026-04-13 — `SaveTemp.sh:L7` + `templog/StoreDetTemps.py:L51-57`
- `LNFill_App.py` writes only coarse fill events to InfluxDB: `Fill,type=<X> value=1` at start and `Fill,type=<X> value=0` at end (via `WriteInflux()`) ✅ verified 2026-04-13 — `LNFill_App.py:L259,L532`. **Per-detector fill durations are NOT written to InfluxDB** — they exist only in text log files (`logs/fill_YYYYMMDD_HHMM.log`). This is the key gap for the LN2 fill time monitoring task (QUEUE.md).
- `WriteDiscordMessage.py:WriteInflux()` sends line protocol via curl to `HPGeTemp` db ✅ verified 2026-04-13 — `WriteDiscordMessage.py:L23`
- `LNFill_ping_cron.sh` writes `ping_status` metrics for hosts (ln2con, pi5, lnfill IOC, GS collector servers) ✅ verified 2026-04-13 — `LNFill_ping_cron.sh:L11` (`url=http://192.168.203.56:8181/...`)
- Needs `influx.token` file: `export INFLUXDB_WRITE_TOKEN=<token>`
- Token in elog: https://elog.phy.anl.gov/GS+maintenance/39

#### Fill Duration Data Flow (for LN2 fill time monitoring task)
Per-detector fill duration (seconds) is computed inside `DetMan.py:buildFillLog()` as `filltime = int(self.detMan[i].getOpenTime())` ✅ verified 2026-04-13 — `DetMan.py:L331`. This value is logged to `logs/fill_*.log` in a format like:
```
DET: A-12   35342   247  19.2K  16.6K  Cold   Cold    OK   247
```
(columns: hose-slot, GSID, fill_time_s, temp_before, temp_after, sensor_before, sensor_after, status, LED_cold_time) ✅ verified 2026-04-13 — `DetMan.py:L338-351` (format string field order confirms hose[0]-hose[1], GSID, filltime, temps[0]/[1], sens[0]/[1], stat, ledT)

To implement fill duration trending, two approaches:
1. **Log-parser script:** parse existing `logs/fill_*.log` files, extract per-GSID fill durations + timestamps → push to InfluxDB. Historical data available from existing logs.
2. **Code injection:** add `WriteInflux(f'FillDuration,gsid={gsid} value={filltime}')` in `DetMan.py:buildFillLog()` after L352 — writes per-detector fill duration in real time going forward.

### Discord Webhooks
Two webhook files required:
- `discord.WebHook` → fill logs and general notifications: `export WEBHOOK=<url>`
- `discord_anomaly.WebHook` → SSH failure, fill not running, short fills: `export anomalyWebHook=<url>`
- Webhook URLs in elog: https://elog.phy.anl.gov/GS+maintenance/45

---

## ln2con

- **Role:** Boot host for lnfill IOC
- **Login:** `gamop` user
- **Log files:** `~/lnfill_log2/`
  - `ln.inits` — IOC init state
  - `ln.state` — IOC state
  - `ln_log` — fill logs
- **Logrotate:** runs weekly (Monday noon) to archive/clean logs

---

## pi5 Setup (Debian 13)

```sh
# Install EPICS
sudo apt install epics-base python3-pip python3-dev
pip3 install --break-system-packages pyepics

# Fix libca symlinks
sudo ln -s /usr/lib/aarch64-linux-gnu/libca.so.4.14.4 /usr/lib/aarch64-linux-gnu/libca.so
sudo ln -s /usr/lib/aarch64-linux-gnu/libCom.so.3.23.1 /usr/lib/aarch64-linux-gnu/libCom.so

# In .bashrc
source ~/lnFill/EPICS_para.sh
export PYEPICS_LIBCA=/usr/lib/aarch64-linux-gnu/libca.so
export PYEPICS_LIBCOM=/usr/lib/aarch64-linux-gnu/libCom.so
export LD_LIBRARY_PATH=/usr/lib/aarch64-linux-gnu:$LD_LIBRARY_PATH
```

**Persistent journal logging** (for crash forensics):
```sh
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo systemctl restart systemd-journald
# After crash: journalctl -b -1
```

---

## Connections to Other Subsystems

- **EPICS** — uses EPICS Channel Access to control valves via lnfill IOC (192.168.203.121)
- **collectorboxpi/** — similar Pi-based EPICS infrastructure but for detector HV, not LN
- **DCS2.onenet** — external monitoring host (InfluxDB, Grafana, ping health checks)

---

## Operations & Troubleshooting (from [wiki: LN System](https://wiki.anl.gov/gsdaq/LN_system))

### Accessing the LN Fill System Remotely

```sh
ssh -X gamop@sonata   # (or ssh -X gamop@ln2con, where findhose works)
ssh -X dgs@dgs1
lnmain                # launch LN EPICS control pages
```
> Account on sonata (green gateway machine) required — contact Ken Teh if needed.

### Viewing the lnfill IOC Console

```sh
# On ln2con:
cu -s 9600 -l /dev/ttyS0
```

### When to Reboot the lnfill IOC

- **Valve overtime:** reboot IOC so the dialer alarm re-engages
- **Temperature alarm only:** reboot not necessary

### Manual Fill of One Detector (Remote)

Used when a detector warms up before the next scheduled fill, or after fixing a clogged bayonet:

1. Open the valve for the desired tank
2. Open the associated exhaust valve for the line
3. Wait for temperature at exhaust valve to bottom out (liquid is present)
4. Return exhaust valve to **auto mode**
5. Open the valve to the target detector
6. Wait for temperature at detector return to bottom out
7. Return detector valve to **auto mode**
8. Return tank valve to **auto mode**

### Overtime Troubleshooting

- Clean bayonet
- Check for leaks in line
- After fixing: do a manual fill, then **reboot the IOC**

### Undertime Troubleshooting

- Fill manually after tank fill (or check for bad sensor)
- Reboot lnfill if needed

### Blown Fuse on AB Module

If multiple detectors show overtime on the same manifold → likely a blown fuse on an AB I/O module:
1. Look for red 'fuse' indicator on the module
2. Replace fuse and locate the bad valve
3. Switch to another fill line and disable the bad valve that caused the short

### Finding the Fill Line for a Detector (`findhose`)

On `ln2con`, run:
```sh
findhose
```
Returns the fill line associated with any warming detector.

### Setting Temperature Thresholds (`set_hilo_lim`)

```sh
# On ln2con, as gamop:
cd /home/gamop/lnfill_logs

# View current thresholds for detector 107:
set_hilo_lim 107
# MOD107_DV_TEMP.HIHI 91
# MOD107_DV_TEMP.HIGH 86
# MOD107_DV_TEMP       78.4
# MOD107_DV_TEMP.LOW  66
# MOD107_DV_TEMP.LOLO 61

# Set thresholds centered on 78K (±10 HIGH/LOW, ±20 HIHI/LOLO):
set_hilo_lim 107 78
```

> Set limits when detectors are cold and stable.

### Fill Timing Notes

- Do **not** start a full GS fill between 8 PM and midnight while GT is present
- If a detector starts warming < 1 hour before next scheduled fill → leave it alone and wait; manual fill at that point risks undertime problems

---

## DetMan.py — FillManifold() Internals

_Source: `lnfill/DetMan.py` — verified 2026-04-18_

`DetMan` manages one detector manifold (ManID 1–4). Instantiated by `LNFill_App.py` with one thread per manifold.

### Constructor
- Creates `LNValve` objects: `MFeed` (main feed), `MSpare` (spare feed), `MVent` (vent) — max open times from `timeout_valves[3]` arg (defaults: 3000s/3000s/400s)
- Creates `DetValve` objects for all 28 detector positions (`detMan[1..28]`)
- Calls `CloseAllValves()` on init (safety: ensures all valves closed before any fill)

**ManID > 4 — Spare-Feed Swap (ManD workaround):** ✅ verified 2026-04-18 — `DetMan.py:L65-72`
- If `ManID` arg is 5–8 (i.e., `ManID > 4`), the constructor remaps `ManID = ManID - 4` and **swaps MFeed/MSpare roles**: `MANS` (spare) becomes MFeed and `MANF` (main) becomes MSpare.
- This is a documented "temporary modification for Man D" — allows manifold D to use the spare line as its primary feed.
- Manifolds 5–8 do NOT exist in the normal GS configuration; this is only used in special test/workaround scenarios.

### FillManifold() — 4-Phase Fill Loop

`FillManifold(fill_list)` runs continuously until all detectors are filled (or timeout). `fill_list[0]` is `'A'` (all enabled) or `'L'` (list of GS IDs).

**Max fill time guard:** `max_fill_time = ((ndet-1)/4 + 2) * 500` seconds — scales with number of detectors. If exceeded, aborts and closes MFeed. ✅ verified 2026-04-18 — `DetMan.py:L219`

**4 phases (state machine in `while filling` loop):**

| Phase | Condition | Action |
|-------|-----------|--------|
| 1 — Vent | `vent_open=True` | Wait for MVent sensor to go Cold OR max open time. If Cold → close vent, move to phase 2. If `getOpenTime() > vent_fail_time` (600s) before Cold → abort with error ("Vent Line displaced at HUB"). |
| 2 — Init | `init_det=True` (once) | Pop up to 4 detectors from `Man` list, open each valve, set LED cold time sentinel. |
| 3 — Fill | `nfil > 0` | Poll all 4 fill slots every 3s. For each: if past `minOpenTime` AND (sensor Cold OR past `maxOpenTime`) → close valve, record fill status, open next detector from `Man`. |
| 4 — Done | `nfil == 0` | All detectors filled — close MFeed, set `filling=False`. |

**Fill status classification (set per detector at close):** ✅ verified 2026-04-18 — `DetMan.py:L278-285`
- `OVERTIME` — valve open time exceeded `maxOpenTime` (detector may not be full; vacuum/hose issue)
- `EXTENDED` — fill time within 5s above `minOpenTime` (borderline fast fill)
- `NORMAL` — filled normally between min and max times

**LED Cold Time:** At open, each detector's `LEDColdTime` is set to `maxOpenTime` (sentinel = "not yet cold"). First time sensor goes Cold, `setLEDColdTime(openTime)` records when LN first reached the sensor. Logged in fill log as the last column. Useful for detecting hose leaks (LED goes cold early = LN leaking out before reaching full dewar). ✅ verified 2026-04-18 — `DetMan.py:L258-260`

**Fill log output format (per detector):**
```
DET: A-12   35342   247  19.2K  16.6K  Cold   Cold    NORMAL   247
```
Columns: hose-slot, GSID, fill_time_s, temp_before, temp_after, sensor_before, sensor_after, status, LED_cold_time_s ✅ verified 2026-04-13 — `DetMan.py:L338-351`

---
*Source: `DGS_tools_pack/lnfill/` (git repo on gitlab.phy.anl.gov/dgs-tools-pack). Created: 2026-04-05. Operations section added from [wiki: LN System](https://wiki.anl.gov/gsdaq/LN_system): 2026-04-06. DetMan.py FillManifold section added: 2026-04-18.*

## Cross-References

- `knowledgeBase/influxdb_grafana.md` — InfluxDB/Grafana on DCS2; HPGeTemp database written by `SaveTemp.sh` / `StoreDetTemps.py`
- `knowledgeBase/collectorboxpi.md` — Raspberry Pi soft IOC for HV control; parallel Pi infrastructure
- `knowledgeBase/hardware_architecture.md` — LN2 system role in detector cooling chain
- `knowledgeBase/troubleshooting.md` — LN2-related operational issues
