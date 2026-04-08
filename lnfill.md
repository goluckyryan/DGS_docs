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
| `LNFill_cron.sh` | Scheduled full-system fill (7am + 7pm daily) ✅ verified 2026-04-06 — `lnfill/README.md:L105,L110` (`00 07,19 * * *`) |
| `LNFill_Auto_EFill_cron.sh` | Auto emergency fill — runs every 15 min, fills warm detectors ✅ verified 2026-04-06 — `lnfill/README.md:L106,L111` (`*/15 * * * *`) |
| `SaveTemp.sh` | Records temperatures every 10 min; pushes to InfluxDB on DCS2 ✅ verified 2026-04-07 — README.md:L107,L112 |
| `LNFill_ping_cron.sh` | Health check: pings all hosts, reports to InfluxDB + Discord (runs on DCS2, every 12 min) ✅ verified 2026-04-07 — `lnfill/README.md:L116,L122` (`*/12 * * * *` on `dcsu@DCS2.onenet`) |
| `LNFill_pi5_check.sh` | Checks LNFill_App.py is running at 7:15 and 19:15 (runs on DCS2) ✅ verified 2026-04-06 — `lnfill/README.md:L131` (`15 7,19 * * *`) |
| `EPICS_para.sh` | Sets EPICS environment variables |
| `DetMan.py` | Detector manager |
| `DetValve.py` | Valve control |
| `TankMan.py` | **Tank manifold manager** — controls LN supply tanks. Two tank manifolds: TMan1 (feeds manifolds A+B, tanks T1/T2/T3) and TMan2 (feeds manifolds C+D, tanks T4/T5/T6). T1→feeds A, T2→feeds B, T3=spare; T4→feeds C, T5→feeds D, T6=spare. Each manifold has 1 supply vent valve + 3 × (feed valve + vent valve) per tank. `FillTanks()`: (1) open+cool supply vent (waits for `Cold` sensor or timeout=500s); (2) opens up to **2 tanks simultaneously** (feed+vent per tank), waits until sensor goes Cold or timeout (default 3000s each); (3) closes all. `CloseAllValves()` called on init and in `LNFill_Stop.py`. |
| `LNValve.py` | **Valve abstraction base class** — wraps a single EPICS-controlled solenoid valve with sensor monitoring. 7 valve types: `SPLY` (supply vent), `TNKF`/`TNKV` (tank feed/vent), `MANF`/`MANS`/`MANV` (manifold feed/spare/vent), `DET` (detector). PV naming pattern: `LNS{n}` (supply), `LNT{n}` (tank), `LNM{n}` (manifold), with suffixes `_VV:EN` (valve enable), `_FV:VM` (valve state), `_SM:SUB.D/.E` (fill time sub-record), `_TM:BT/.AT` (sensor before/after). Default max open times: SPLY=300s, TNKF/TNKV=3600s, MANF/MANS=2500s, MANV=500s, DET=600s. Default min open times: SPLY=100s, TNKF/TNKV=300s, MANF/MANS/MANV=50s, DET=50s. Key methods: `Open()`, `Close()`, `GetStatSens()` (returns "Warm"/"Cold"/"Fault"), `GetState()`, `SetPVOpenTime()`, `getOpenTime()`. Uses `pv_lock` (from `pvlock.py`) + `get_pv()` (from `pv_cache.py`) for thread-safe CA access. |
| `pvlock.py` | Shared `threading.Lock()` (`pv_lock`) for serializing EPICS PV access across modules |
| `pv_cache.py` | Thread-safe PV object cache (double-checked locking pattern); `get_pv(name)` returns a reused `epics.PV` instance, creating it once per name to avoid redundant CA connections |
| `LNFill_Stop.py` | Emergency stop: kills running `LNFill_App.py` processes (SIGKILL), then closes all manifold + tank valves via `DetMan.CloseAllValves()` + `TankMan.CloseAllValves()`, writes log + error file |
| `LNFill_closeValves.py` | Closes all 4 manifold detector valves (manifolds 1–4) without killing `LNFill_App.py` — safer than Stop for mid-run valve reset; runs from `/home/dgs/lnFill/` with aarch64 EPICS libs hard-coded |
| `LNFill_check.sh` | Fill status check |
| `WriteDiscordMessage.py` | Sends Discord notifications |
| `gefilltime2.dat` | Historical fill time data |
| `templog/` | Temperature log directory |
| `AddPress.sh` | **Spare tank pressure management** — monitors tank pressures during a fill; opens/closes spare tank fill valves (LNT3, LNT6) to keep both tank stations pressurized. Runs for up to 2,200s; exits when fill completes or timeout. See details below. |
| `setTNF.sh` | Set tank fill script |

---

## LNFill_App.py — Fill Types

| Arg | Mode | Description |
|-----|------|-------------|
| F | Full fill | Fill all manifolds + all tanks |
| M | Monitor fill | Check temperatures, build warm detector list, fill warm ones |
| L | List fill | Fill specific detectors by GS ID list |
| T/A/B/C/D | Selective | Fill selected manifolds or specific tank |

**Flow summary:**
1. Check for existing LNFill_App.py instance (abort or kill old one)
2. Decode fill type → build target dewar list
3. Check fill status = Ready (abort if not)
4. Spawn one thread per manifold (1–4), fill up to 4 dewars each concurrently
5. Wait for all manifold threads to finish
6. Close all manifold valves
7. If filling tanks: spawn tank fill threads, wait, close tank valves
8. Write fill statistics to log
9. If mode=M: wait 15 min for temps to stabilize
10. Push fill data to InfluxDB + send Discord notification

---

## Cron Jobs

### On pi5 (`/home/dgs/lnFill/`)

```sh
00 07,19 * * *   LNFill_cron.sh                  # Full fill at 7am + 7pm
*/15 * * * *     LNFill_Auto_EFill_cron.sh        # Emergency fill every 15 min
*/10 * * * *     SaveTemp.sh                      # Record temps every 10 min
```

### On DCS2 (`dcsu@DCS2.onenet`, `/home/phy/dcsu/lnFill/`)

```sh
*/12 * * * *     LNFill_ping_cron.sh              # Ping all hosts every 12 min
15 7,19 * * *    LNFill_pi5_check.sh              # Check LNFill_App.py is running
```

---

## Health Monitoring

### Ping Check (`LNFill_ping_cron.sh`, on DCS2)
- Pings: ln2con, pi5, lnfill IOC, GS collector servers
- For pi5: uses **SSH** instead of ping (catches OS-broken-but-network-up failures)
- When SSH succeeds: also records `mem_available_mb` → Grafana trend
- On SSH failure: Discord alert to anomaly channel

### Pi5 Health Check (`LNFill_pi5_check.sh`, on DCS2)
- Runs at 7:15 and 19:15 (15 min after scheduled fill starts)
- SSHes into `pi5-lnFill`, checks if `LNFill_App.py` is running via `pgrep`
- On failure: Discord alert to anomaly channel (`discord_anomaly.WebHook`)

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

**Known failed sensors (hardcoded v2.3):** `PRESS_EXT2_FAIL=1`, `PRESS_TS2_T2_FAIL=1`, `PRESS_TS2_T3_FAIL=1`

**Gauge calibration (v2.3):** `PRESS_TS1_T3_CAL=+2` (reads 2 PSI low); all others = 0

Fallback: if all pressure gauges for a station fail, assumes 28 PSI (early in run) or 20 PSI (after 400s).

---

## Communications

### InfluxDB (on DCS2.onenet)
- `SaveTemp.sh` pushes temperature data
- Needs `influx.token` file: `export INFLUXDB_WRITE_TOKEN=<token>`
- Token in elog: https://elog.phy.anl.gov/GS+maintenance/39

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
*Source: `DGS_tools_pack/lnfill/` (git repo on gitlab.phy.anl.gov/dgs-tools-pack). Created: 2026-04-05. Operations section added from [wiki: LN System](https://wiki.anl.gov/gsdaq/LN_system): 2026-04-06.*
