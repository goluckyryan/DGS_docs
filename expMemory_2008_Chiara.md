# Experiment Memory: exp2008_Chiara

## Overview

| Field | Value |
|-------|-------|
| Experiment Name | `exp2008_Chiara` |
| PI | Chiara |
| Status | **Active / Ongoing** |
| Elog Logbook | `2008_Chiara` |
| GEB_ID | 14 |
| Next Run # | >122 (as of 2026-04-14; latest run dir is `exp2008_122` on vol5) |

---

## Data Locations

| Location | Path |
|----------|------|
| **Local data (DCS2)** | `/mnt/data0/exp2008_Chiara/data/` |
| **NFS sync destination (DCS2 local mount)** | `~/ExpData/exp2008_Chiara` → symlink → `/dgsdata/fs2/vol5/exp2008_Chiara` |
| **NFS mount point on DCS2** | `fs2.onenet:/mnt/vol5/atlasdata/dgs` mounted at `/dgsdata/fs2/vol5` (NFS4, rw) |
| **NFS network path** | `fs2.onenet:/mnt/vol5/atlasdata/dgs/exp2008_Chiara` |
| Root data | `/mnt/data0/exp2008_Chiara/root_data/` |
| Parquet data | `/mnt/data0/exp2008_Chiara/Parquet/` |

Run subfolders: `exp2008_001`, `exp2008_002`, ... (runs 1–122 present as of 2026-04-14; latest: `exp2008_122`)

---

## Run Control

- **Run control host:** DCS2 (`dcsu@DCS2.onenet`)
- **ANLDAQ folder:** `/home/phy/dcsu/ANLDAQ/tcpReceiver/`
- **Start run:** `start_run.sh`
- **Stop run:** `stop_run.sh`
- **Simple start/stop:** `simpleStartStop.sh`
- **TCP receiver binary:** `tcpReceiverMT` (multi-threaded)
- **expInfo.sh:** defines experiment variables (name, folders, GEB_ID, next run #)

---

## TCP Receiver Config (`config.txt`)

12 VME IOC crates, all on port 9001, DataType 14:

| IP | VME Crate |
|----|-----------|
| 192.168.203.141 | VME01 |
| 192.168.203.142 | VME02 |
| 192.168.203.143 | VME03 |
| 192.168.203.144 | VME04 |
| 192.168.203.145 | VME05 |
| 192.168.203.177 | VME06 |
| 192.168.203.178 | VME07 |
| 192.168.203.179 | VME08 |
| 192.168.203.180 | VME09 |
| 192.168.203.183 | VME10 |
| 192.168.203.181 | VME11 |
| 192.168.203.182 | VME12 |

---

## Digitizer Settings (`basic_settings_DGS.sh`)

- **Discriminator mode:** CFD (Constant Fraction Discriminator)
- **Channels used:** 5–9 per digitizer
- **Active digitizers per VME crate:** MDIG1 + MDIG2 (except VME06 and VME10: MDIG1 only)

**CFD parameters (Mike's values):**
| Parameter | Value |
|-----------|-------|
| p1_window | 0.07 µs |
| p2_window | 0.05 µs |
| m_window | 3.5 µs |
| k0_window | 0.56 µs |
| k_window | 0.2 µs (fixed) |
| d_window | 0.06 µs (fixed) |
| d3_window | 0.2 µs |
| CFD_fraction | 25 |
| trigger_polarity | RiseEdge |
| led_threshold | 30 |
| raw_data_delay | 0.5 µs |
| raw_data_length | 0.32 µs (trace) |

---

## Trigger Settings (`basic_settings.sh`)

- **Trigger:** `SumX`
- **MTRG VME crate:** VME99 (`VME99:MTRG:*`)
- **Trigger monitor select:** SumX
- **EN_SUM_X:** on
- **SUM_OF_X/Y_THRESH:** 0 (open threshold)
- **NIM1 delay:** disabled
- **MON7 veto:** enabled
- **Software veto:** on (initially)
- **SYSMON:** enabled

**VME99 test stand digitizer (MDIG1, ch 7, CFD):**
| Parameter | Value |
|-----------|-------|
| m_window | 2.5 µs |
| k0_window | 0.56 µs |
| k_window | 0.16 µs |
| d_window | 0.1 µs |
| CFD_fraction | 50 |
| raw_data_length | 4.0 µs |
| trigger_mux_select | IntAcptAll |

---

## DCS2 Storage Overview

| Filesystem | Size | Used | Avail | Use% | Mount |
|-----------|------|------|-------|------|-------|
| `/dev/nvme0n1` | 1.8T | 1.5T | 262G | 85% | `/mnt/data0` ← **local run data** |
| `fs2.onenet:/mnt/vol5/atlasdata/dgs` | 264T | 102T | 162T | 39% | `/dgsdata/fs2/vol5` ← **NFS for exp2008_Chiara** |
| `fs2.onenet:/mnt/vol3/atlasdata/dgs` | 219T | 178T | 41T | 82% | `/dgsdata/fs2/vol3` |
| `fs2.onenet:/mnt/vol2/atlasdata/dgs` | 165T | 159T | 6.5T | **97%** | `/dgsdata/fs2/vol2` ⚠️ nearly full |
| `fs2.onenet:/mnt/vol4/atlasdata/dgs` | 227T | 178T | 50T | 79% | `/dgsdata/fs2/vol4` |

**Why the symlink:** `/mnt/data0` (local NVMe) is 85% full — only 262 GB left. The experiment data is pointed to NFS vol5 (264T, 39% used, 162T free) which has plenty of headroom. The symlink `~/ExpData/exp2008_Chiara → /dgsdata/fs2/vol5/exp2008_Chiara` keeps it accessible as if local while actually writing to NFS.

⚠️ **vol2 is at 97% — nearly full.** Not used for this experiment but worth watching.

---

## Notes

- Data synced to NFS at `fs2.onenet:/mnt/vol5/atlasdata/dgs/exp2008_Chiara` via `sync_exp_data.sh`
- Same ANLDAQ folder structure exists on pi5-dgs (Ryan's Pi) as well as DCS2
- As of 2026-04-14 check: 122 run directories present on vol5 (latest: `exp2008_122`)

---

## data0 Retention Policy

**Always keep on data0 (local NVMe):**
- Run 001 — first run reference
- Run 005 — early reference run
- The **last completed run** — so it's immediately accessible for analysis

All other runs: delete from data0 once confirmed on NFS (spot-check MD5 of a few files).

---

## Cleanup Log

| Date | Action |
|------|--------|
| 2026-04-05 | Deleted runs 017–027 from `/mnt/data0` (local NVMe) — all confirmed synced to NFS vol5. Runs 001, 005, 028 kept on local. data0 freed from 85% → 16% used (1.5T free). |
| 2026-04-05 | Runs 028–034 deleted from data0 after rsync --size-only verified 0 file differences vs NFS. Kept 001, 005, 035. Next run: 036. data0 at 21% (1.4T free). |
| 2026-04-05 | Runs 035–041 + 043 deleted from data0. MD5 spot-checked 4 random files across runs (all matched NFS). Kept 001, 005, 042. data0: 29% used, 1.3T free. |

## Monitoring

⚠️ **Data0 space monitor cron status (as of 2026-04-16):** The hourly data0 space monitor cron (id: `d3285cee-893e-49c4-91df-85e57ace9b07`) that previously ran on pi5-dgs is **NOT active** on spark-ca9f (crontab empty). The data0 filesystem is on DCS2, not spark-ca9f. Confirm with Ryan whether this monitoring should be migrated — either to a cron on DCS2 itself, or to spark-ca9f with an SSH-based check.

---

*Created: 2026-04-05*

## Cross-References

- `knowledgeBase/nfs_layout.md` — Full NFS mount layout on DCS2; vol3/vol4 directory structure
- `knowledgeBase/run_procedures.md` — Typical DGS run workflow; GEBSort calibration and sorting steps
- `knowledgeBase/dgs_analysis.md` — Post-experiment analysis: fastEventConstructor, parquet_pysort
- `knowledgeBase/influxdb_grafana.md` — InfluxDB/Grafana monitoring on DCS2; temperature and health data
