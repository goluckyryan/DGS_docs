# lnfill — IOC Internals, Communications & Fill Data Flow

Stability: C2 - Active / semi-stable

Deep technical reference for the LN2 fill system's IOC, data flows, and software internals.
For operational overview, cron jobs, and troubleshooting: see [`lnfill.md`](lnfill.md).

---

## Table of Contents

1. [Communications](#communications)
   - [InfluxDB (on DCS2.onenet)](#influxdb-on-dcs2onenet)
   - [Discord Webhooks](#discord-webhooks)
2. [ln2con — LN2 IOC Boot Host](#ln2con--ln2-ioc-boot-host)
3. [LNValve Class Hierarchy — Python Fill Control Architecture](#lnvalve-class-hierarchy--python-fill-control-architecture)
   - [LNValve — Base Valve Object](#lnvalve--base-valve-object)
   - [DetValve — Detector Valve Extension](#detvalve--detector-valve-extension)
   - [TankMan — Tank Manifold Manager](#tankman--tank-manifold-manager)
4. [DetMan.py — FillManifold() Internals](#detmanpy--fillmanifold-internals)
5. [Auxiliary Scripts (lnfill repo)](#auxiliary-scripts-lnfill-repo)
6. [Cross-References](#cross-references)

---

## Communications

### InfluxDB (on DCS2.onenet)
- `SaveTemp.sh` + `templog/StoreDetTemps.py` push two measurements to InfluxDB db `HPGeTemp` at `http://192.168.203.56:8181/api/v3/write_lp` ✅ verified 2026-04-13 — `SaveTemp.sh:L7` + `templog/StoreDetTemps.py:L51-57`:
  - **Detector temps:** `Temperature,gsid=NNN,en=0/1 value=<K>` — all 110 GS detectors, only written if temp > 10 K; values > 520 K are zeroed (treated as disconnected/bad sensor). Tags: `gsid` = zero-padded GS ID (001–110), `en` = `MOD###_DV_EN` (0=disabled, 1=enabled). CA get timeout: 0.15s per channel. ✅ verified 2026-04-18 — `StoreDetTemps.py:L44-55` (`if detTemps[i]>10:` write; `if detTemp[gsid] > 520: detTemp[gsid] = 0`)
  - **Pi5 board temperature:** `pi_Temp value=<celsius>` — measured via `vcgencmd measure_temp` (parsed from `temp=42.8'C` format). Also pushed to `HPGeTemp` db in the same curl batch. ✅ verified 2026-04-18 — `StoreDetTemps.py:L31-33,L39-40` (`getPiTemp()` + `logfile2.write(influx_entry)`)
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

**How detID is obtained — hose-to-detector mapping:**

The detID stored in `_SM:SUB.E` is read via Channel Access at startup by `DetValve.py`:
```python
self.gsid_pv = "LNH" + str(mid) + "-" + str(vid).zfill(2) + "_SM:SUB.E"
self.gsID = int(self.getPVValue(self.gsid_pv))
```

The `.E` field of each `LNHx-xx_SM:SUB` record is **NOT set in `gamln.db`** — it is set at IOC initialization time by the `dfill_sub_init` C function compiled into **`lnfiller.vx`** (the VxWorks application binary, loaded by `startup.cmd`). ✅ verified 2026-04-18 — `gamln.db` has no `field(E,...)` entries; `startup.cmd` loads `vx68040/lnfiller.vx` then calls `iocInit`; `gamln.db` subroutine records use `INAM="dfill_sub_init"`.

**Key implication:** The hose→detID mapping is **embedded in the compiled VxWorks binary** (`lnfiller.vx`). It cannot be changed without recompiling the binary and rebooting the IOC. If a fill hose is physically rerouted to a different detector dewar, the binary must be updated.

**Complete hose→detID mapping** ✅ verified 2026-04-18 — extracted from `dgs@ln2con:/home/lncon/prod/lnfill/log/ln.inits` (IOC state-save file written at each boot by `dfill_sub_init`).

> ✅ **`ln.inits` snapshot confirmed 2026-04-18** — archived at `DGS_tools_pack/ln2con/ln.inits.snapshot.20260418`. **Updated 2026-04-29:** H4-25 corrected from detID=22 to detID=425 (spare/non-standard slot) by Ryan — resolves the H4-25/H4-26 duplicate. H4-26 remains detID=22. Verified live on ln2con 2026-04-29.

Format: `Hx-yy  detID  manifold_num  ...`

| Hose | detID | Hose | detID | Hose | detID | Hose | detID |
|------|-------|------|-------|------|-------|------|-------|
| H1-01 | 73 | H2-01 | 87 | H3-01 | 301 | H4-01 | 82 |
| H1-02 | 502 | H2-02 | 79 | H3-02 | 76 | H4-02 | 72 |
| H1-03 | 105 | H2-03 | 69 | H3-03 | 88 | H4-03 | 52 |
| H1-04 | 504 | H2-04 | 59 | H3-04 | 94 | H4-04 | 404 |
| H1-05 | 101 | H2-05 | 205 | H3-05 | 106 | H4-05 | 40 |
| H1-06 | 61 | H2-06 | 206 | H3-06 | 66 | H4-06 | 110 |
| H1-07 | 103 | H2-07 | 207 | H3-07 | 104 | H4-07 | 42 |
| H1-08 | 508 | H2-08 | 71 | H3-08 | 86 | H4-08 | 96 |
| H1-09 | 509 | H2-09 | 209 | H3-09 | 102 | H4-09 | 409 |
| H1-10 | 81 | H2-10 | 210 | H3-10 | 310 | H4-10 | 30 |
| H1-11 | 511 | H2-11 | 51 | H3-11 | 92 | H4-11 | 411 |
| H1-12 | 75 | H2-12 | 29 | H3-12 | 312 | H4-12 | 412 |
| H1-13 | 93 | H2-13 | 49 | H3-13 | 313 | H4-13 | 413 |
| H1-14 | 89 | H2-14 | 214 | H3-14 | 84 | H4-14 | 414 |
| H1-15 | 65 | H2-15 | 13 | H3-15 | 98 | H4-15 | 24 |
| H1-16 | 516 | H2-16 | 63 | H3-16 | 78 | H4-16 | 416 |
| H1-17 | 517 | H2-17 | 31 | H3-17 | 317 | H4-17 | 417 |
| H1-18 | 107 | H2-18 | 25 | H3-18 | 64 | H4-18 | 26 |
| H1-19 | 57 | H2-19 | 21 | H3-19 | 54 | H4-19 | 419 |
| H1-20 | 520 | H2-20 | 43 | H3-20 | 70 | H4-20 | 420 |
| H1-21 | 41 | H2-21 | 221 | H3-21 | 18 | H4-21 | 421 |
| H1-22 | 77 | H2-22 | 33 | H3-22 | 322 | H4-22 | 68 |
| H1-23 | 523 | H2-23 | 27 | H3-23 | 20 | H4-23 | 80 |
| H1-24 | 83 | H2-24 | 11 | H3-24 | 56 | H4-24 | 38 |
| H1-25 | 19 | H2-25 | 23 | H3-25 | 44 | H4-25 | 425 |
| H1-26 | 526 | H2-26 | 17 | H3-26 | 74 | H4-26 | 22 |
| H1-27 | 55 | H2-27 | 45 | H3-27 | 28 | H4-27 | 36 |
| H1-28 | 37 | H2-28 | 35 | H3-28 | 328 | H4-28 | 428 |

Notes:
- detIDs 5xx, 2xx, 3xx, 4xx = non-standard (spare/special detectors, DUO, or empty slots)
- H4-25 was detID=22 (duplicate of H4-26) — **fixed 2026-04-29**: H4-25 now detID=425 (spare slot). H4-26 remains detID=22. ✅ verified live on ln2con 2026-04-29
- Hx-yy maps directly to EPICS PV `LNHx-yy_SM:SUB.E` and Python `mid=x, vid=yy`
- This mapping is set by `dfill_sub_init` in `lnfiller.vx` at IOC boot; `ln.inits` is the state-save record

**To extract the current mapping:** Two options:
1. **Read ln.inits:** `dgs@ln2con:/home/lncon/prod/lnfill/log/ln.inits` — contains full table (see above)
2. **Parse fill logs:** Each `fill_YYYYMMDD_HHMM.log` in `~/lnFill/logs/` shows `DET: A-12  35342` — hose-slot to detID, one per fill run

**Where `gamln.db` and `lnfiller.vx` live (on ln2con, 192.168.203.148):**
- IOC boot tree: `/home/lncon/prod/lnfill/Rev6-01-04/` ✅ verified 2026-04-18 — `dgs@ln2con` direct SSH
- `startup.cmd` loads: `vx68040/lnfiller.vx` + `rtdb/gamln.db` + `rtdb/tempmon.db`
- Also mirrored on NFS: `/dgsdata/fs2/vol3/ln2con/vxboot/lnfill/Rev6-01-04/`
- **Not in any git or SVN repo** — lives only on ln2con and NFS server
- See `nfs_layout.md` for full `gamln.db` analysis (1357 records, 406 sub records) ✅ verified 2026-04-13

To implement fill duration trending, two approaches:
1. **Log-parser script:** parse existing `logs/fill_*.log` files, extract per-detID fill durations + timestamps → push to InfluxDB as `FillDuration,detid=<detID> value=<seconds>`. Historical data available from 237+ existing logs (back to Jan 2026).
2. **Code injection:** add `WriteInflux(f'FillDuration,detid={detID} value={filltime}')` in `DetMan.py:buildFillLog()` after L352 — writes per-detector fill duration in real time going forward.
- Note: use **detID** (not hose-slot) as the InfluxDB tag — it is the stable Gammasphere detector identifier; hose positions can be rerouted.

### Discord Webhooks
Two webhook files required:
- `discord.WebHook` → fill logs and general notifications: `export WEBHOOK=<url>`
- `discord_anomaly.WebHook` → SSH failure, fill not running, short fills: `export anomalyWebHook=<url>`
- Webhook URLs in elog: https://elog.phy.anl.gov/GS+maintenance/45

---

## ln2con — LN2 IOC Boot Host

✅ verified 2026-04-18 — direct SSH as `dgs@192.168.203.148`

**Hardware:**
- Fedora 12 (Constantine), Linux 2.6.32, i686 — very old, no `ip` command
- IP: 192.168.203.148
- Serves as NFS boot host for the lnfill VxWorks IOC (MVME167, 192.168.203.121)

**Login:**
- `dgs` account ✅ works (password in TOOLS.md / workspace secrets)
- `gamop` account exists but password unknown to General DGS
- SSH requires old key type flags: `-o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedKeyTypes=+ssh-rsa`

**Disk layout:**
- `/` — 77G, 5% used
- `/export/home` — 181G, 3% used (NFS exports to VxWorks IOC)
- `/home` — 20G (local home dirs)
- Key dirs: `/export/home/{dgs,edm,EPICS,gam6_alt,lncon,scripts}`

**NFS export to VxWorks IOC (`lnfill`, 192.168.203.121):**
- VxWorks mounts: `lncon.gam.lbl.gov:/export/home/lncon` → `/home/lncon` (VxWorks side)
- Old hostname `lncon.gam.lbl.gov` still used in `vx_mounts` config

**IOC boot tree location:** `/home/lncon/prod/lnfill/`
| Dir/File | Contents |
|----------|----------|
| `Rev6-01-04/` | **Active** VxWorks boot tree (current production version) |
| `Rev6-01-01/` to `Rev6-01-03/` | Older revisions (archived) |
| `Rev6-01-03.tar.gz` | Compressed archive of Rev6-01-03 |
| `boot/`, `boot02/` | VxWorks boot ROM configs |
| `local/` | Site-specific configs: `vx_mounts`, `resource.def.R312` |
| `log/` | IOC logs (83 MB total): `ln.inits`, `ln_log`, `ln_log_archive/`, `ln.state` |
| `tools/` | Log processing: `auto_process`, `lnlogs`, `log_cleanup`, `process_logs` |
| `scaget/` | Local EPICS CA get utilities |

**`Rev6-01-04/` boot tree structure:**
- `startup.cmd` — VxWorks startup script (loads all modules, DBs, starts fill sequencer)
- `vx68040/lnfiller.vx` — **compiled fill application** (contains `dfill_sub_init` which sets hose→detID mapping)
- `rtdb/gamln.db` — EPICS records (1357 records: valves, manifolds, sensors) ✅ verified 2026-04-21 — `grep -c 'record(' gamln.db` via DCS2 NFS: 1357
- `rtdb/tempmon.db` — Temperature monitoring records (43 records) ✅ verified 2026-04-21 — `grep -c 'record' tempmon.db` via DCS2 NFS: 43 (previously incorrect as 429)
- `default.dctsdr` — VxWorks DCT database (binary)
- `targetmv167/` — MVME167-specific object files (iocCore, drvSup, recSup, devSup, seq)

**`startup.cmd` boot sequence:**
1. Mount NFS filesystem (`vx_mounts`)
2. Load LED support libraries
3. Load EPICS core, drivers, records, device support, sequencer
4. Set state-save/log paths to `/vxboot/lnfill/log/`
5. Load `lnfiller.vx` (main fill application)
6. Set `gamln_auto_det_fill_enabled=1` and `gamln_auto_tank_fill_enabled=1`
7. Call `ln_init()` — initializes fill system, spawns `ln_logger`
8. Load databases: `default.dctsdr`, `gamln.db`, `tempmon.db`
9. `iocInit` — starts IOC (calls `dfill_sub_init` on all sub records → sets detID mapping)
10. `seq &fill` — starts EPICS state sequencer for fill logic

**Log files (on ln2con):**
| File | Contents |
|------|----------|
| `log/ln.inits` | IOC initialization state — contains full hose→detID mapping table |
| `log/ln_log` | Running fill event log (valve open/close events with timestamps) |
| `log/ln_log_archive/` | Weekly rotated logs (date-stamped, back to 2020) |
| `log/ln.state` | IOC state-save (current valve states) |
| `log/LNFillApp/` | Additional app logs |

**Log rotation:** Managed by `lnlogrotate.conf` on ln2con. The actual conf file is at `/home/lncon/prod/lnfill/log/lnlogrotate.conf` (also symlinked from `/export/home/lncon/`). Schedule: `0 12 * * 1` (every Monday at noon) in `gamop`'s crontab. ✅ verified 2026-04-18 — `lnfill/README.md:L84` + live read of conf on ln2con (192.168.203.148). Conf parameters: `rotate 520` (keep up to 520 archives), `size 512k` (rotate when file exceeds 512 KB), `daily` (daily rotation eligible), `dateext` + `dateformat .%Y%m%d` (date-stamped archives), `nocompress` (plain text), `copytruncate` (truncate in-place — safe for live file), `olddir ln_log_archive` (archives go to `ln_log_archive/`), `create 0666` (recreate with world-rw). Target log: `/home/gamop/lnfill_logs/ln_log`.

**hosts file:** Contains the full original onenet host list (many legacy machines: gam0, gam1, gamvxi01-07, gsdaq1-4, gamdas01-18, etc.)

---

## LNValve Class Hierarchy — Python Fill Control Architecture

_Source: `lnfill/LNValve.py`, `lnfill/DetValve.py`, `lnfill/TankMan.py`, `lnfill/pv_cache.py`, `lnfill/pvlock.py` — verified 2026-04-20_

The Pi5 fill control software uses a class hierarchy built around `LNValve`, the base object for any controllable valve in the GS LN system.

### Class Hierarchy

```
LNValve           — base valve: any GS LN valve (supply, tank, manifold, detector)
  └── DetValve    — extends LNValve: adds detector temp PV, gsID lookup, min fill time
```

`DetMan` and `TankMan` are *managers* (not subclasses) — they aggregate multiple valve objects and run fill threads.

---

### LNValve — Base Valve Object (`LNValve.py`, 370 lines)

**Valve types** (the `type` constructor arg selects one of 7 keywords):

| Type | Description | PV prefix example |
|------|-------------|-------------------|
| `SPLY` | Supply vent valve | `LNS1_VV:EN` |
| `TNKF` | Tank feed valve (1–6) | `LNT3_FV:EN` |
| `TNKV` | Tank vent valve (1–6) | `LNT3_VV:EN` |
| `MANF` | Manifold main feed valve (1–4) | `LNM2_FV:EN` |
| `MANS` | Manifold spare feed valve (1–4) | `LNM2A_FV:EN` |
| `MANV` | Manifold vent valve (1–4) | `LNM2_VV:EN` |
| `DET`  | Detector fill valve (manifold 1–4, valve 1–28) | `LNH3-07_FV:EN` |

**PV naming convention** (derived from type + manifold ID + valve ID):

| PV suffix | Role | Notes |
|-----------|------|-------|
| `_FV:EN` | Feed valve enable (Open/Auto/Disable) | Write `Open` to open, `Auto` to close |
| `_VV:EN` | Vent valve enable | Same Open/Auto/Disable states |
| `_FV:VM` | Valve mode readback | Reflects actual hardware state |
| `_VV:VM` | Vent valve mode readback | |
| `_SM:SUB.D` | Fill time written by software (seconds) | Updated at open and close |
| `_SM:SUB.E` | detID (set at IOC boot by `dfill_sub_init`) | For DET type only |
| `_TM:BT` | Overflow sensor voltage (at fill start) | Analog sensor (LN presence) |
| `_TM:AT` | Overflow sensor voltage (at fill end) | `Cold` = LN reached sensor |

**Default timeouts per type** (seconds):

| Type | Max open time | Min open time |
|------|--------------|---------------|
| SPLY | 300 s | 100 s |
| TNKF | 3600 s | 300 s |
| TNKV | 3600 s | 300 s |
| MANF | 2500 s | 50 s |
| MANS | 2500 s | 50 s |
| MANV | 500 s | 50 s |
| DET  | 600 s | 50 s |

**Key methods:**
- `Open()` — checks valve state, records open time (`valvetime[0]`), reads sensor voltage, writes `Open` to `_FV:EN` (or `_VV:EN`)
- `Close()` — reads sensor voltage again, writes `Auto` to valve PV, records close time (`valvetime[1]`), calls `SetPVOpenTime()`
- `GetState()` — reads `_FV:VM` (valve mode readback)
- `SetPVOpenTime()` / `GetPVOpenTime()` — writes/reads fill duration to/from `_SM:SUB.D`
- `GetStatSens()` / `GetVoltsSens()` — reads overflow sensor status / analog voltage from `_TM:BT` / `_TM:AT`
- `getOpenTime()` — returns `valvetime[1] - valvetime[0]` (or elapsed time if still open)
- `setMaxOpenTime()` / `setMinOpenTime()` / `setLEDColdTime()` — configuration setters
- `getFillStatus()` — returns `NORMAL`, `OVERTIME`, or `EXTENDED` (set by DetMan at close)
- `checkValve()` — self-test method: opens for 15s, closes, prints state

**Thread safety:** All CA reads/writes go through `pv_cache.get_pv(name)` with `pv_lock` (a `threading.Lock()` from `pvlock.py`). The `pv_cache.py` module is a simple singleton cache — `PV()` objects are created once and reused for the lifetime of the process, avoiding redundant CA channels.

---

### DetValve — Detector Valve Extension (`DetValve.py`, 152 lines)

Inherits `LNValve` with `type="DET"`. Adds:
- **gsID lookup:** reads `LNHx-yy_SM:SUB.E` at construction time to get the GS detector ID (0–110). If `0 < gsID < 111`, constructs temp PVs; otherwise `temp_pv = None` (empty slot)
- **Temperature PVs:**
  - `MOD###_DV_TEMP` — HPGe detector temperature (Kelvin)
  - `MOD###_DV_EN` — Detector enabled/disabled flag (0/1)
- **minFillTime:** hardcoded 100s default (independent of `LNValve.minopentime`)
- **Override of `Open()`:** checks `DoFill` flag, then delegates to EPICS write via `v_pv` (`LNHx-yy_FV:EN`)
- **Temperature readback:** `DetMan.py` reads `temp_pv` before and after each fill via the DetValve object to populate the fill log `temp_before`/`temp_after` columns

---

### TankMan — Tank Manifold Manager (`TankMan.py`, 246 lines)

Manages one **tank manifold** (TMan 1 or TMan 2). Not a subclass of LNValve; composes valve objects:

**Tank plumbing:**
- **TMan 1** → feeds manifolds A and B (tanks 1–3: T1→A, T2→B, T3→spare)
- **TMan 2** → feeds manifolds C and D (tanks 4–6: T4→C, T5→D, T6→spare)

**Valve objects created:**
- `Supply` — `LNValve("SPLY", ManID)` — supply vent valve; max open 500s (configurable)
- `TFeed[0..2]` — `LNValve("TNKF", j)` for j=1–3 (TMan1) or j=4–6 (TMan2)
- `TVent[0..2]` — `LNValve("TNKV", j)` — paired feed+vent open/close simultaneously
- Default tank timeout: 3000s (configurable via `timeouts[3]` constructor arg)

**Key method:** `FillTanks()` — opens Supply, then opens each TFeed/TVent pair in sequence; waits for sensor Cold or max timeout; closes all; logs result. Called from `LNFill_App.py` after manifold fills complete.

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
Columns: hose-slot, detID, fill_time_s, temp_before, temp_after, sensor_before, sensor_after, status, LED_cold_time_s ✅ verified 2026-04-13 — `DetMan.py:L338-351`
- **hose-slot** (`A-12`) maps to EPICS PV (`LNH1-12`) — physical valve position, can change if cables are rerouted
- **detID** (`35342`) is the stable Gammasphere detector identifier used in physics analysis; use this as the InfluxDB tag for fill duration trending

---

## Auxiliary Scripts (lnfill repo)

_Documented 2026-04-21 — code-verified from `DGS_tools_pack/lnFill/`._

### `AddPress.sh` (V2.3, 2024-06-28)

**Purpose:** Auxiliary pressure management during LN2 fills — keeps dewar tanks pressurized by selectively opening spare (Tank 3) fill valves on each tank station.

**Key logic:**
- Monitors pressure on Tank Station 1 (TS1: VME03 manifolds A+B) and Tank Station 2 (TS2: VME09 manifolds C+D) during an active fill.
- Opens the spare tank fill valve (`LNT3_FV:EN` / `LNT6_FV:EN`) when the external dewar pressure exceeds tank pressure by `VALVE_OPEN_PRESS` (3 PSI) and tank is below `MAX_TANK_PRESS` (32 PSI).
- Closes the spare fill valve when: external pressure falls too low (`VALVE_CLOSE_PRESS` = -1 PSI), tank is at max, or the two stations have diverged in pressure by `TS1/TS2_PRESS_DIFF_OFF_THRESH` (2/3 PSI) — prevents one station from hogging flow at the expense of the other.
- Redundant gauge fallback: if primary pressure gauge is out-of-range or flagged `FAIL`, falls back to gauge 2, then gauge 3, then a hardcoded estimate (28 PSI early in fill, 20 PSI late).
- **Currently failed sensors (hardcoded):** `PRESS_EXT2_FAIL=1` (TS2 external gauge), `PRESS_TS2_T2_FAIL=1` (TS2 tank 2 gauge), `PRESS_TS2_T3_FAIL=1` (TS2 tank 3 gauge).
- Runs for up to `MAX_RUN_TIME=2200 s` (≈37 min); exits early when all manifold fill valves are closed after `MIN_RUN_TIME=240 s`.
- Check interval grows from 1 s up to `MAX_SLEEP_TIME=30 s`; resets to 0 when a valve is opened.
- **Called by:** `LNFill_App.py` during automated detector fills (started as a background subprocess alongside fill sequence).

**Key PVs:**
| PV | Meaning |
|---|---|
| `LNP1-01_PR:AP` | TS1 external dewar pressure |
| `LNP1-02_PR:AP` / `LNP1-03_PR:AP` / `LNP1-04_PR:AP` | TS1 tank 1/2/3 pressure |
| `LNP2-01_PR:AP` | TS2 external dewar pressure |
| `LNP2-02_PR:AP` / `LNP2-03_PR:AP` / `LNP2-04_PR:AP` | TS2 tank 1/2/3 pressure |
| `LNT3_FV:EN` | TS1 spare (Tank 3) fill valve |
| `LNT6_FV:EN` | TS2 spare (Tank 6) fill valve |
| `LNM1_FV:EN` / `LNM2_FV:EN` | TS1 manifold fill valves (A/B) |
| `LNM3_FV:EN` / `LNM4_FV:EN` | TS2 manifold fill valves (C/D) |

---

### `setTNF.sh`

**Purpose:** Sets the EPICS PV `LN_TTNF:XC` ("Time of Next Fill") to a time 12 hours from the current hour.

**Logic:** If current hour is >10 AM, subtracts 12 → sets to 0X:01 (morning time). Otherwise adds 12 → sets to afternoon time. Optionally accepts a specific hour as `$1` to override.

**Called by:** `LNFill_cron.sh` at the end of each scheduled fill to schedule the next fill approximately 12 hours out.

---

### `LNFill_pi5_check.sh`

**Purpose:** Health check — confirms that `LNFill_App.py` is running on `pi5-lnFill`. If not, sends a Discord alert to the `#anomaly` webhook.

**Mechanism:** SSH to `pi5-lnFill` with 5-second connect timeout; runs `pgrep -f LNFill_App.py`. On failure (process not found or SSH unreachable): sends a `curl` POST to `$anomalyWebHook` (loaded from `discord_anomaly.WebHook` file).

**Called by:** DCS2 cron at **06:15 and 18:15 daily** (`15 6,18 * * *`) ✅ verified 2026-04-21 — `dcsu@DCS2.onenet crontab -l`.

---

### `LNFill_check.sh`

**Purpose:** Post-fill duration anomaly detector. Parses `LNFill_cron.log` to find the last fill begin/end timestamps, computes fill duration, and alerts Discord if duration was below threshold.

**Logic:**
1. Finds last `LN2  Fill Ends:` line in log → extracts timestamp (`MM/DD/YYYY-H:MM`).
2. Finds the most recent `LN2  Fill Begin` line before that end line.
3. Parses both timestamps → computes `duration_min`.
4. If `duration_min < THRESH_MIN` (default 10 min): sends alert to `$anomalyWebHook`.
5. A very short fill (<10 min) signals a potential problem — fill aborted early, sensor failure, or cryostat vacuum issue.

**Arguments:** `$1` = optional threshold in minutes (default: 10).

**Alert format:** `"Last Fill Duration Alert: <N> minutes @ <begin_ts>"`

**Called by:** `LNFill_cron.sh` — sourced at L55 after the fill completes (`source LNFill_check.sh`). Runs on pi5-lnFill as part of the normal fill cron cycle. ✅ verified 2026-04-22 — `LNFill_cron.sh:L55`

---

### `archive_cron_log.sh`

**Purpose:** Rotates `LNFill_cron.log` — copies it to `logs/archive/LNFill_cron_YYYYMMDD.log` then truncates the live log to zero. Prevents unbounded growth.

**Called by:** pi5-lnFill crontab — every Sunday at midnight (`0 0 * * 0`). ✅ verified 2026-04-22 — cross-referenced with `knowledgeBase/lnfill.md:L159` (live pi5-lnFill crontab entry)

---

### `epics_cron.sh`

**Purpose:** EPICS environment setup for cron jobs on aarch64 (Pi5). Exports `EPICS_BASE`, `PYEPICS_USE_SYSTEM_LIBS`, `PYEPICS_LIBCA`, `PYEPICS_LIBCOM`, and `LD_LIBRARY_PATH` so that `caget`/`caput`/pyepics work correctly in non-login shell cron context.

**Used by:** All lnfill cron jobs that invoke `caget`/`caput` or `python3 -c "import epics"` — sourced at the top of each cron script.

---

## tempmon.db — VXI Crate Temperature Monitor EPICS Database

_Source: `DGS_tools_pack/ln2con/rtdb/tempmon.db` (429 lines, 43 records) ✅ verified 2026-04-23 — local file read_

`tempmon.db` is loaded at IOC boot alongside `gamln.db`. It implements a **VXI crate heartbeat + HPGe temperature alarm aggregator** used by the analog-era LN2 fill IOC (`ln2con`). It does **not** monitor temperatures directly — it aggregates per-module temperature status PVs that are populated by `lnfiller.vx` subroutine records.

### Record Summary (43 total)

| Group | Records | Count |
|-------|---------|-------|
| `CRATE_ENABLEn` (mbbo) | Enable/disable per-VXI-crate monitoring | 6 |
| `HEARTBEAT_OKn` (sub) | Subroutine: check heartbeat PV for each crate | 6 |
| `HEARTBEAT_STATUSn` (bo) | Readback: DEAD / OK per crate | 6 |
| `CRATE_OKn` (calc) | = HEARTBEAT_OK × CRATE_ENABLE (0 or 1) | 6 |
| `CRATES_OK` (calc) | Sum of all 6 CRATE_OK values | 1 |
| `CRATES_ENABLED` (calc) | Sum of all 6 CRATE_ENABLE values | 1 |
| `CRATES_DISABLED` (calc) | 6 − CRATES_ENABLED | 1 |
| `CRATES_NOTOK` (calc) | CRATES_ENABLED − CRATES_OK | 1 |
| `TEMPLO_STATE` / `TEMPHI_STATE` / `TEMPDO_STATE` / `TEMPMC_STATE` / `CNOTOK_STATE` (mbbo) | Alarm state machines (OK / ALARM_PENDING / ALARM / ACK) | 5 |
| `TEMPMON_HOLDOVER` (ai) | Alarm holdover time in seconds (default 300 s, max 21600 s) | 1 |
| `TOTAL_ENABLED_TEMP_LO/OK/HI` (calc) | Cross-crate module count: enabled modules at LO/OK/HI temp | 3 |
| `TOTAL_DISABLED_TEMP_OK` (calc) | Cross-crate count: disabled modules with temp OK | 1 |
| `TOTAL_ENABLED` / `TOTAL_DISABLED` / `TOTAL_MISCONFIGURED` (calc) | Cross-crate totals | 3 |
| `TEMPMON` (sub) | Main aggregator subroutine (reads all totals, drives alarm states) | 1 |
| `TEMPMON_CTL` (sub) | Periodic control subroutine (SCAN=10s, reads LN_FILL_ID/LN_TANK_FILL_ID) | 1 |

### VXI Crate Heartbeat Logic

Each VXI crate (1–6) has a heartbeat PV (`HEARTBEAT1`–`HEARTBEAT6`) written by the hardware/VME driver running in `lnfiller.vx`. The `HEARTBEAT_OKn` subroutine record (INAM=`hb_sub_init`, SNAM=`hb_sub`) reads `HEARTBEATn.VAL` (INPA) and monitors for change — if the PV stops toggling, the crate is declared DEAD.

- `CRATE_OKn = HEARTBEAT_OKn × CRATE_ENABLEn` — a crate only counts as OK if it's both heartbeating AND enabled
- `CRATES_NOTOK = CRATES_ENABLED − CRATES_OK` — number of enabled crates that are not responding

### Alarm State Machine (5 independent FSMs)

Each alarm type has a 4-state mbbo FSM:

| State | Value | Severity | Meaning |
|-------|-------|----------|---------|
| OK | 0 | NO_ALARM | Normal |
| ALARM_PENDING | 1 | MINOR | Condition first detected (within holdover window) |
| ALARM | 2 | MAJOR | Condition persisted past holdover (default 300 s) |
| ACK | 3 | MINOR | Alarm acknowledged by operator |

The 5 FSMs cover: `TEMPLO_STATE` (module too cold), `TEMPHI_STATE` (module too warm), `TEMPDO_STATE` (disabled module with temp OK — may indicate misconfiguration), `TEMPMC_STATE` (misconfigured modules), `CNOTOK_STATE` (enabled crates not OK).

### Cross-Crate Module Counters

The `TOTAL_*` calc records aggregate module counts across all 6 crates, weighted by `CRATE_OKn` (only counts modules from crates that are alive):
- `TOTAL_ENABLED_TEMP_LO` — sum over crates of (CRATE_OKn × MODULES_ENABLED_TEMP_LOn)
- `TOTAL_ENABLED_TEMP_OK` — same for modules in normal temperature range
- `TOTAL_ENABLED_TEMP_HI` — same for modules above high-temp threshold
- `TOTAL_DISABLED_TEMP_OK` — disabled modules with temp OK (flag for misconfiguration checking)
- `TOTAL_ENABLED` / `TOTAL_DISABLED` / `TOTAL_MISCONFIGURED` — total module counts

The `MODULES_ENABLED_*n` and `MODULES_DISABLED_*n` per-crate PVs are **not in tempmon.db** — they are populated by `lnfiller.vx` (the compiled VxWorks application, sourced from CVS on con6 via `tempmon_subs.c`). ✅ verified 2026-04-23 — `grep -n "MODULES_ENABLED\|HEARTBEAT[1-6]" rtdb/gamln.db` returns no matches; these PVs must originate in compiled C subroutine records within `lnfiller.vx`.

### TEMPMON_CTL — Periodic Control Subroutine

Runs every 10 seconds (SCAN="10 second"). Reads:
- INPA: `LN_FILL_ID:XC.VAL` — current LN fill ID (which detector is being filled)
- INPB: `LN_TANK_FILL_ID:XC.VAL` — current tank fill ID

Purpose: correlate active fills with temperature alarm state to suppress false alarms during fill (detectors warm briefly when LN is first flowing).

---

## Cross-References

- [`lnfill.md`](lnfill.md) — Operational overview, fill types, cron jobs, health monitoring, troubleshooting
- `knowledgeBase/influxdb_grafana.md` — InfluxDB/Grafana on DCS2; HPGeTemp database
- `knowledgeBase/nfs_layout.md` — Full NFS layout; `gamln.db` analysis (1357 records)
- `knowledgeBase/con6_lnfill.md` — con6 (Solaris 10): CVS source repo + 68040 cross-compiler for lnfill IOC

---

*Split from `lnfill.md` 2026-04-20 (F task — MD organization). Original content created 2026-04-05 through 2026-04-18. Auxiliary scripts documented 2026-04-21. tempmon.db structure documented 2026-04-23.*
