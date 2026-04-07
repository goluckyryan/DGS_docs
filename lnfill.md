# lnfill — Liquid Nitrogen Filling System

## What It Is

Automated control system for filling germanium detector **dewars** with liquid nitrogen (LN). Ported from DGS1 (Scientific Linux 6 / Python 2) to Debian 13 / Python 3. Controls valves, monitors dewar fill level, runs scheduled fills, and sends alerts via Discord.

---

## Physical System

**4 LN Tanks:**
- Tank A, B, C, D
- Tank B pressurizes Tanks A and D
- Tanks A and D supply LN to the manifolds

**4 Manifolds:**
- 2 per side (upper + lower)
- Each manifold: **28 solenoid valves** + LED sensors ✅ verified 2026-04-06 — `DetValve.py:L25` (`valve # 1-28`) + `DetMan.py:L134` ("the 28 detector valves")
- Each valve+LED pair → one detector dewar
- Fill detection: LN flowing past LED sensor changes LED resistance → system detects "full"

**Max 4 dewars filling simultaneously per manifold** (16 total at once across 4 manifolds) ✅ verified 2026-04-06 — `DetMan.py:L253` (`while len(Man)>0 and i<4`)

---

## Computers in the System

| Computer | OS | IP | Role |
|----------|----|----|------|
| ln2con | Fedora 12 | 192.168.203.148 | Boot host for IOC; runs logrotate |
| pi5 | Debian 13 | 192.168.203.58 | Main fill control (runs LNFill_App.py) | ✅ verified 2026-04-07 — LNFill_ping_cron.sh:L19 + README.md:L25 |
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
| `SaveTemp.sh` | Records temperatures every 10 min; pushes to InfluxDB on DCS2 | ✅ verified 2026-04-07 — README.md:L107,L112 |
| `LNFill_ping_cron.sh` | Health check: pings all hosts, reports to InfluxDB + Discord (runs on DCS2, every 12 min) |
| `LNFill_pi5_check.sh` | Checks LNFill_App.py is running at 7:15 and 19:15 (runs on DCS2) ✅ verified 2026-04-06 — `lnfill/README.md:L131` (`15 7,19 * * *`) |
| `EPICS_para.sh` | Sets EPICS environment variables |
| `DetMan.py` | Detector manager |
| `DetValve.py` | Valve control |
| `TankMan.py` | Tank manager |
| `LNValve.py` | LN valve abstraction |
| `pvlock.py` | PV locking utility |
| `pv_cache.py` | PV caching |
| `LNFill_Stop.py` | Emergency stop — close all valves |
| `LNFill_closeValves.py` | Close valves utility |
| `LNFill_check.sh` | Fill status check |
| `WriteDiscordMessage.py` | Sends Discord notifications |
| `gefilltime2.dat` | Historical fill time data |
| `templog/` | Temperature log directory |
| `AddPress.sh` | Add pressure script |
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
- SSHes into pi5, checks if `LNFill_App.py` is running
- On failure: Discord alert to anomaly channel

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
