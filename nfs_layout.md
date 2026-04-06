# NFS Layout — DCS2 Mount Points

Explored via SSH as `dcsu@DCS2.onenet`. All mounts are read-only (ro) except vol5 and piserver which are rw.

*Started: 2026-04-05. Incremental — updated each heartbeat.*

---

## Path Structure — Important

The NFS server (`fs2.onenet`, `fs1.onenet`) exports only the **`dgs` subdirectory** of each volume. The full server-side path is:

```
fs2.onenet:/mnt/volX/atlasdata/dgs/
                     ^^^^^^^^^^
                     atlasdata/ contains other subdirs (musics, etc.)
                     — but only /dgs is exported/mounted here
```

DCS2 mounts these exports as:
```
/dgsdata/fs2/volX/   ← maps to fs2.onenet:/mnt/volX/atlasdata/dgs/
/dgsdata/fs1/vol2/   ← maps to fs1.onenet:/mnt/vol2/atlasdata/dgs/
/piserver/           ← maps to fs2.onenet:/mnt/vol1/fs2/nfs/piserver/
```

So when this doc says e.g. `exp2008_Chiara/`, the **full absolute path on DCS2** is:
```
/dgsdata/fs2/vol5/exp2008_Chiara/
```
And the **NFS server-side path** is:
```
fs2.onenet:/mnt/vol5/atlasdata/dgs/exp2008_Chiara/
```

---

## Mount Summary

| Local path on DCS2 | NFS server export | Size | Used | Avail | RW? |
|---|---|---|---|---|---|
| `/dgsdata/fs2/vol2/` | `fs2.onenet:/mnt/vol2/atlasdata/dgs/` | 165T | 159T | 6.5T | ro |
| `/dgsdata/fs2/vol3/` | `fs2.onenet:/mnt/vol3/atlasdata/dgs/` | 219T | 178T | 41T | ro |
| `/dgsdata/fs2/vol4/` | `fs2.onenet:/mnt/vol4/atlasdata/dgs/` | 227T | 178T | 50T | ro |
| `/dgsdata/fs2/vol5/` | `fs2.onenet:/mnt/vol5/atlasdata/dgs/` | 264T | 104T | 160T | **rw** |
| `/dgsdata/fs1/vol2/` | `fs1.onenet:/mnt/vol2/atlasdata/dgs/` | 40T | 17T | 23T | ro |
| `/piserver/` | `fs2.onenet:/mnt/vol1/fs2/nfs/piserver/` | 17T | ~0 | 17T | **rw** |

DCS2 local storage: `/` on NVMe1 (915G, 45% used), `/mnt/data0` on NVMe0 (1.8T, 26% used) — not NFS.

> ⚠️ **Scope:** We only see the `dgs/` subtree. Other data (e.g. `musics/`, other experiment groups) may exist under `atlasdata/` on the server but are not mounted on DCS2 and not accessible from here.

*Source: `ssh dcsu@DCS2.onenet "cat /proc/mounts | grep nfs"` and `df -h` — 2026-04-05*

---

## /dgsdata/fs2/vol5 — Current Experiment Data (rw)
*Source: `ssh dcsu@DCS2.onenet "ls -la /dgsdata/fs2/vol5/"` — 2026-04-05*

Active volume — current experiments and ongoing work.

| Directory | Notes |
|---|---|
| `exp2008_Chiara/` | **Current experiment** (Ryan, dcsu owner) — last modified 2026-04-05 |
| `exp1756_Hoff/` | Sep 2025 |
| `exp1985x_Chowdhury/` | May 2024 |
| `exp2017_Morse/` | May 2024 |
| `exp2026_Mueller-Gatermann2/` | Nov 2025 |
| `exp2031x_Hartley/` | Oct 2024 |
| `exp2045_Rogers/` | Mar 2024 |
| `exp2059_Ackermann/` | Jun 2024 |
| `exp2071_Andreyev/` | Jun 2024 |
| `exp2078_Huang/` | Apr 2024 |
| `exp2092x_Reviol/` | Jul 2024 |
| `exp2202_Mattera/` | Feb 2025 |
| `253no/` | Sep 2025 |
| `ChicoTest/` | Jun 2025 |
| `cmg/` | Feb 2025 |
| `ebss2024/` | Jul 2024 |
| `GT_test_ak/` | Dec 2024 |
| `mpc/` | Apr 2025 — contains `calib/`, `docs/`, `pyepics/pv_dashboard.py` |
| `MSM/` | Feb 2025 (matthew.martin) |

### vol5/mpc/pyepics/pv_dashboard.py ✅ read 2026-04-05
*Source: `ssh dcsu@DCS2.onenet "cat /dgsdata/fs2/vol5/mpc/pyepics/pv_dashboard.py"` — 2026-04-05*
A **PyQt6 + pyepics** EPICS PV health dashboard. Clean modern example of proper pyepics usage.

**Architecture:**
- `PVRow` widget — one row per PV: name, value, connection status, last-update timestamp, optional Put 0/Put 1 buttons
- `epics.PV(..., callback=..., connection_callback=..., auto_monitor=True)` — fully async, no polling
- Thread-safe UI updates via `QObject`/`pyqtSignal` bridge (avoids Qt cross-thread issues)
- `EPIXDashboard` — scrollable container of `PVRow`s with a Refresh button
- Color-coded status: green dot = connected, red = disconnected

**PV list in `main()`:** placeholder PVs only (`IOC:HEARTBEAT`, `DAQ:STATE`, `DET:COUNT_RATE`, etc.) — intended as a template, not DGS-specific yet.

**Key pyepics patterns shown:**
- `epics.PV(pvname, callback=fn, auto_monitor=True)` — subscribe to changes
- `pv.wait_for_connection(timeout=1.0)` — initial connect
- `pv.get()` / `pv.put(value)` — read/write
- Signal bridge for thread-safe Qt updates: `pyqtSignal(dict)` on `QObject`

> ⚠️ **Context:** The active DGS GUI is `ANLDAQ/gui/` (Git repo, PyQt6 + pyepics). It has a much more sophisticated `class_PV.py` (with RBV support, ref-counted subscriptions, read-only flags, states) and `class_PVWidgets.py` (custom widgets). `pv_dashboard.py` appears to be a **standalone prototype/template** by mpc — not integrated into the main ANLDAQ GUI. Useful as a simple reference but superseded by ANLDAQ/gui for real DGS work.

---

## /dgsdata/fs2/vol3 — Older Experiments (ro)
*Source: `ssh dcsu@DCS2.onenet "ls -la /dgsdata/fs2/vol3/"` — 2026-04-05*

Experiments roughly 2019–2025, older isotope studies.

| Directory | Notes |
|---|---|
| `exp1889_Chiara/` | Earlier Chiara experiment (2021) |
| `exp1893/` | Through Feb 2025 |
| `exp1913/` | Through Jul 2025 |
| `exp1917_Hartley/` | Large — 41 entries, through Jan 2026 |
| `ln2con/` | LN2 control VxWorks boot trees (see detail below) |
| `sbx2022tuning/` | SBX tuning data from 2022 |
| `mpc/` | Mar 2025 |
| `global_sl7/` | SL7 global environment |
| + many isotope dirs | `102YDecay`, `125pm`, `190at`, `254No`, `256no`, `Am241`, etc. |

### vol3/ln2con/ — LN2 Control VxWorks Boot Trees
*Source: `ssh dcsu@DCS2.onenet "find /dgsdata/fs2/vol3/ln2con/ -maxdepth 3 | sort"` — 2026-04-05*

Contains two VxWorks boot tree roots for the LN2 filling IOC (`ln2con`, MVME167 CPU):

**`vxboot/`** — Older tree; contains `lnfill/` subtree with versioned revisions:
- `Rev6-01-02/` — VxWorks image + symbol file
- `Rev6-01-03/` — Full boot tree (arReq, ascii, base, config, rtdb, scripts, startup.cmd, targetmv167/, vx68040/)
- `Rev6-01-04/` — Same structure; this is the **last revision in this tree**
- `tools/` — Log processing scripts: `auto_process`, `lnlogs`, `log_cleanup`, `process_logs`
- `log/iocLog.text` — IOC log

**`vxboot_ln2con/`** — Older/alternate tree with broader EPICS environment:
- `lnfill/` — Older revisions (Rev6-01-01 through Rev6-01-03) plus separate `arChan`, `arExp`, `arSet`, `boot/`, `dbase/`, `dl/`, `epics/`, `local/`, `log/`
- `scripts/` — Legacy utility scripts: `adbdiff`, `addlc*`, `brsave`, `btest`, `bumpless`, and many more (Solaris-era tools)
- `ana/`, `daq/`, `epics/`, `epics_R3.7/`, `epics.new/`, `gam/`, `gamop/` — Older global environments
- `R3.11.2ls/`, `R3.12-LBL.1/` — Old EPICS base archives
- `kernels/`, `vxWorks` — VxWorks kernel images

**startup.cmd key facts (Rev6-01-04):**
- Loads: `sysAnalysisBd.vx`, `sysAnLEDsupport.vx` (LED status bar), `iocCore`, `drvSup`, `recSup`, `devSup`, `devLibOpt`, `initHooks.o`, `seq`
- Main image: `vx68040/lnfiller.vx`
- `gamln_auto_det_fill_enabled=1`, `gamln_auto_tank_fill_enabled=1` — both auto-fill modes on at boot
- Loads: `default.dctsdr` + `rtdb/gamln.db` + `rtdb/tempmon.db`
- State save dir: `/vxboot/lnfill/log/ln.state`
- `iocLogDisable=1` — IOC logging disabled in this rev
- CPU: MVME167 (Motorola 68040, shown by `targetmv167/` + `vx68040/` image dirs)

> **Context:** This is the legacy VME-based LN2 IOC boot tree, archived on NFS. The **active** lnfill system lives in `DGS_tools_pack/lnfill/` (Pi-based soft IOC). The VxWorks tree here is historical reference — it shows the system before migration to the Pi.

---

## /dgsdata/fs2/vol2 — Development & Calibration (ro, nearly full at 97%)
*Source: `ssh dcsu@DCS2.onenet "ls -la /dgsdata/fs2/vol2/"` — 2026-04-05*

Development environment, calibration data, global software environments.

| Directory | Notes |
|---|---|
| `global_32/` | Main 32-bit global software environment — actively updated (Apr 2025) |
| `dgscalib/` | Calibration data + GEBSort logs, `gf3`, calib files |
| `dgscalib2/` | Secondary calibration set |
| `dgscalib/bin/`, `calib/`, `data/` | Calibration binaries and data |
| `dgsMapping/` | Feb 2025 — detector mapping files |
| `dgsReceiver/` | Permission denied for dcsu — likely root-owned |
| `GEBSort/` | Empty directory |
| `EPICS_ROOT.tar.gz` | 67 MB — archived EPICS root install |
| `dgs_template_2.tgz` | DGS template archive (2019) |
| `template/` | Template experiment directory |
| `daq_root/` | DAQ ROOT environment |
| `daqtest/` | DAQ test directory (world-writable) |
| + many experiment dirs | Older experiments 2019–2022 |

### vol2/global_32/ioc/ — IOC Software Tree
*Source: `ssh dcsu@DCS2.onenet "ls /dgsdata/fs2/vol2/global_32/ioc/"` — 2026-04-05*
Important: this is the **NFS-hosted IOC software** (separate from the Git `ioc/` repo):

| Path | Contents |
|---|---|
| `boot/` | Full set of `vme01.cmd`–`vme12.cmd` + `vme99.cmd`; `nfsCommands`, `cdCommands`, `DGS_systemdef.txt`; per-crate `*_inLoop.txt`, `*_outLoop.txt`, `*_MiniSender.txt` |
| `bin/` | IOC binaries |
| `db/`, `dbd/` | EPICS database files |
| `py_scripts/` | Python utility scripts (see below) |
| `dgsReceiver/`, `dgsSoftIOC/` | Receiver + soft IOC code |
| `fastEventContructor/` | Fast event constructor (note typo in dir name) |
| `FW_Maint/` | Firmware maintenance tools |
| `gui/` | GUI files |
| `epics/` | EPICS base |
| `SBX_SDCard_Image/` | SBX SD card image |

### vol2/global_32/ioc/py_scripts/ ✅ partially read 2026-04-05
*Source: `ssh dcsu@DCS2.onenet "ls /dgsdata/fs2/vol2/global_32/ioc/py_scripts/"` + file reads — 2026-04-05*
Python utility scripts — pyepics-based, focused on DIG↔RTR connectivity and throttle diagnostics:

| File | Purpose |
|---|---|
| `trace_throttle.py` | **DIG↔RTR cable mapping tool** (see below) ✅ read |
| `trace_throttle2.py` | Likely updated version of above |
| `compare_pvs.py` | Compare MTRG PVs between two `MTrigRegisters.template` snapshots ✅ read |
| `compare_user_pvs.py` | Compare user-facing PVs |
| `test_throtca.py` | Throttle CA interaction test |
| `test_array*.py` | Array PV tests (4 variants) |
| `COLLSCAN.TXT` | Collector scan output |
| `DGS_SYSTEMDEF.TXT` | System definition text |
| `DNG_TO_REGIN_MAP.TXT` | DNG→REGIN mapping table |
| `count_mon.txt` / `count_mon_filtered.txt` | Count monitor output |

#### `compare_pvs.py` — MTRG PV Diff Tool ✅ read 2026-04-05
*Source: `ssh dcsu@DCS2.onenet "cat /dgsdata/fs2/vol2/global_32/ioc/py_scripts/compare_pvs.py"` — 2026-04-05*

Compares `longin`/`longout` PV names between two dated snapshots of `MTrigRegisters.template` to see what PVs were added or removed between MTRG firmware/DB revisions.

- **Compares:** `ioc/db/20250818_MTRG/MTrigRegisters.template` vs `ioc/db/20251022_MTRG/MTrigRegisters.template`
- **Output:** count of old/new PVs, sorted lists of added and removed PV names with record type
- Uses `pathlib` + `re` only — no EPICS required, pure file diff
- **Not integrated into ANLDAQ GUI** — standalone dev utility
- **Cross-reference:** `memory/dgs/ioc.md` documents the `ioc/db/` directory; MTRG templates are in `db/`

---

#### `trace_throttle.py` — DIG↔RTR Connectivity Mapper ✅ read 2026-04-05
*Source: `ssh dcsu@DCS2.onenet "cat /dgsdata/fs2/vol2/global_32/ioc/py_scripts/trace_throttle.py"` — 2026-04-05*

Verifies physical SERDES cable connections between digitizer boards and router boards by toggling throttle bits and reading router status registers via EPICS.

**Purpose:** Maps which DIG output cable (A–H) connects to which RTR input port. Useful after cabling changes or for initial system verification.

**System topology hardcoded:**
- 12 VME crates (VME01–VME12); VME06 and VME10 have only 2 DIGs (not 4)
- 4 RTRs: VME03:RTR1, VME06:RTR2, VME09:RTR3, VME12:RTR4
- 8 cables per RTR: A–H (bits 15:8 of `reg_throttle_status_RBV`)

**Key PVs:**
- `GLBL:DIG:rj45_throttle_mode` — global throttle toggle (all DIGs at once)
- `{crate}:{module}:rj45_throttle_mode` — per-DIG throttle
- `{router}:reg_throttle_status_RBV` — RTR status register; bits [15:8] = cable H..A

**Algorithm:**
1. Toggle global throttle 0→1→0, read all RTRs → identify **unused cables** (bits that don't change)
2. For each DIG: toggle 0→1→0 individually, find which RTR bit changes → record connection
3. Verify MDIG/SDIG pairs map to the same RTR cable (they share a physical SERDES cable)
4. Write `connectivity_map.txt` (tab-separated)

**Notes:**
- Uses `epics.caget`/`epics.caput` (not pyepics `PV` objects) — simpler one-shot style
- Has a bug: `from defaultdict import defaultdict` → should be `from collections import defaultdict`; script likely doesn't run as-is
- `datetime.now()` should be `datetime.datetime.now()` — another bug
- Not integrated into ANLDAQ GUI — standalone diagnostic tool
- Cross-reference: `rj45_throttle_mode` PV is in the DIG firmware (`reg_MISC_CTRL` area); throttle status bits in RTRG `reg_throttle_status`

---

## /dgsdata/fs2/vol4 — Analysis & Experiments 2022–2025 (ro)
*Source: `ssh dcsu@DCS2.onenet "ls -la /dgsdata/fs2/vol4/"` — 2026-04-05*

| Directory | Notes |
|---|---|
| `dgs_testing/` | Sep 2025 — contains `GEBSort/`, `calibration.txt`, `Merged/` dirs |
| `mbo/` | Dec 2025 — 40 entries, active |
| `yjc/` | Feb 2026 — 29 entries, active |
| `ML_AK/` | Jul 2025 — machine learning? |
| `NeutronShell_testing/` | Apr 2024 |
| `dgs20230807/` | Dec 2023 — DGS test dataset |
| + many experiment dirs | 1859–2175, 2019–2025 |

---

## /dgsdata/fs1/vol2 — Legacy / Migrated (ro)
*Source: `ssh dcsu@DCS2.onenet "ls -la /dgsdata/fs1/vol2/"` — 2026-04-05*

Mostly symlinks pointing to `/dk/fs2b/dgs/` (old disk path) with `.OLD` backups alongside. Experiments: `exp1758`, `exp1881`, `gsfma370–372`, `cmg`, `global_sl7`. Historical only.

---

## /piserver — PXE Boot Server for Collector Box Pis (rw)
*Source: `ssh dcsu@DCS2.onenet "ls -la /piserver/ && ls /piserver/tftpboot/ && cat /piserver/tftpboot/README.md"` — 2026-04-05*

NFS root for PXE-booted Raspberry Pi collector boxes.

| Path | Contents |
|---|---|
| `tftpboot/` | Per-MAC boot dirs (7 MACs) + `debian13Boot/` + `README.md` |
| `os/` | OS images: `Debian13/`, `piOS/`, `origos/`, `jta/`, `Raspberry_Pi_OS_Full-2022-01-28/`, `shared/` |
| `home/` | Home dirs for Pi users |

### piserver/tftpboot/README.md (collector box MAC→IP map):
| MAC | IP | Location | Box |
|---|---|---|---|
| b8:27:eb:fc:97:08 | .42 | SE | 201 |
| b8:27:eb:57:19:db | .26 | SW | 202 |
| b8:27:eb:5a:d0:8e | .88 | NE | 203 |
| b8:27:eb:99:19:3f | .149 | NW | 204 |

**3 additional MAC dirs** (not in README — spare/decommissioned, hostname not configured):
- `b8-27-eb-39-f2-ce` — spare Pi (default hostname, no location assigned)
- `b8-27-eb-91-bd-1b` — spare Pi (default hostname, no location assigned)
- `b8-27-eb-df-8c-d6` — spare Pi (default hostname, no location assigned)

> ⚠️ Note: README lists NW box as `b8:27:eb:99:19:3f` but tftpboot dir is `b8-27-eb-99-18-3f` (last byte differs: `3f` vs `3f` same, second-to-last `19` vs `18`). One character discrepancy — README may have a typo. Actual dir = `b8-27-eb-99-18-3f`.

*Source: `ssh dcsu@DCS2.onenet "ls /piserver/tftpboot/"` — 2026-04-06*

---

---

## /dgsdata/fs2/vol2/global_32/ — 32-bit Global Development Environment
*Source: `ssh dcsu@DCS2.onenet "ls -la /dgsdata/fs2/vol2/global_32/"` — 2026-04-05*

The main 32-bit software environment for DGS. Contains IOC builds, VxWorks dev environments, EDM screens, and an AI agent framework experiment.

| Entry | Last modified | Notes |
|---|---|---|
| `devel -> devel6` | symlink | Active development env points to devel6 |
| `devel6/` | Jan 2024 | VxWorks IOC build tree (see below) |
| `devel7_newbsp/` | Feb 2017 | Older BSP experiment |
| `develbuild -> devel_tjm` | symlink | |
| `devel_tjm/` | Feb 2013 | Oldest dev env, TJM's build |
| `edmroot/` | — | EDM screen files |
| `ioc/` | Dec 2024 | NFS-hosted IOC software tree (documented above) |
| `new_mv5500_bsp/` | Apr 2015 | MVME5500 BSP experiment |
| `openclaw_framework/` | Apr 2026 | **AI agent framework experiment** (see below) |
| `results/` | Apr 2026 | Job result JSON files |
| `SBX_SDCard_Image/` | Nov 2024 | SBX SD card image (phyadmin owner) |
| `svn_devel/` | Mar 2019 | SVN checkout of dev environment |
| `tempchk/` | Apr 2023 | Temporary IOC build/test area |
| `tgz/` | — | Archived snapshots: devel3–8, global_30MAR23, etc. |
| `TEST_SYNC_FILE` | Apr 2026 | Empty file used to test NFS sync |
| `dbg_StateRecs` | Mar 2019 | State records debug data (1.5 MB) |

### devel6/ — Active VxWorks IOC Build Tree
*Source: `ssh dcsu@DCS2.onenet "ls /dgsdata/fs2/vol2/global_32/devel6/"` — 2026-04-05*

The current VxWorks cross-compilation environment (symlinked as `devel`):

| Dir | Contents |
|---|---|
| `base/` | EPICS base |
| `boot/` | IOC boot scripts |
| `dgs/` | DGS-specific code |
| `dgs1Top/` | DGS1 legacy top-level |
| `extensions/` | EPICS extensions |
| `gcc/` | Cross-compiler |
| `gretTop/` | GRETINA top-level |
| `gs/` | Gammasphere specific |
| `runcon/` | Run control |
| `support/` | EPICS support modules |
| `supTop/` | Support top-level |
| `svnstat.txt` | SVN status snapshot |
| `synApps/` | SynApps modules |
| `systems/` | System configurations |
| `vxWorks/` | VxWorks kernel |

### edmroot/ — EDM Screen Files
*Source: `ssh dcsu@DCS2.onenet "ls /dgsdata/fs2/vol2/global_32/edmroot/"` — 2026-04-05*

| Dir | Purpose |
|---|---|
| `DGScommander/` | DGS operator screen (EDM-based, pre-ANLDAQ GUI) |
| `DGScommander2/` | Updated version |
| `digcntrl/` | Digitizer control screens |
| `lncntrl/` | LN2 control screens |
| `sbxcntrl/` | SBX (slope box) control screens |

> EDM = EPICS Display Manager. These screens predate the PyQt6 ANLDAQ GUI. Likely still used on the control room workstations alongside ANLDAQ.

### openclaw_framework/ — AI Agent Framework Experiment ⭐
*Source: `ssh dcsu@DCS2.onenet "ls -la /dgsdata/fs2/vol2/global_32/openclaw_framework/"` and file reads — 2026-04-05*

> **Note:** Name is coincidence — unrelated to our OpenClaw assistant.

A job-queue-based AI agent framework set up on DCS2, likely for experimenting with LLM-assisted Gammasphere monitoring. Created early April 2026.

**Directory structure:**
```
openclaw_framework/
├── config/
│   ├── ai/            ← AI provider config
│   └── control/       ← framework control config
│   └── framework.json (empty)
├── jobs/
│   ├── ai/            ← job definitions (JSON + txt prompts)
│   ├── queue/
│   ├── running/
│   ├── pending/
│   ├── done/          ← completed jobs
│   ├── failed/
│   └── scripts/
├── runners/
│   └── dispatcher.py  ← job dispatcher (permission denied for dcsu)
├── results/       ← JSON results per job
├── logs/
├── state/
├── templates/
└── docs/
    ├── Gammasphere_angentic.md   ← AI conversation log about building GS monitoring agent
    ├── OpenClaw_Gammasphere_README.md  ← permission denied
    ├── OpenClaw_Ingestion.md           ← permission denied
    └── checkpoint.md                   ← permission denied
```

**Job format** (from readable job JSON files):
```json
{
  "task": "ask_claude",
  "model": "claude-sonnet-4-6",
  "max_tokens": 1000,
  "temperature": 0.1,
  "system": "You are a careful scientific assistant...",
  "prompt": "Explain gamma-gamma coincidence..."
}
```

**Example jobs seen:**
- `job1.json` — asks Claude about gamma-gamma coincidence level ordering
- `job2.json` — asks Claude about coincidence gating background suppression
- `job_test_claude.json` (done) — similar coincidence prompt, completed
- `epics_test.txt`, `http_test.txt`, `test_prompt.txt` — EPICS and HTTP integration tests

**`Gammasphere_angentic.md`** — a saved chat conversation (likely from a web AI assistant, Apr 4 2026) about:
- Building a local LLM agent on a Mac Mini M4 (24GB) to monitor Gammasphere
- Using PyEPICS to read PVs, Gemma 4 26B MoE as the reasoning core
- Alerting (immediate) + reporting (scheduled shift/daily/run reports)
- Long-term plan: NVIDIA DGX Spark (128GB) for fine-tuning on Gammasphere run history
- Tool-calling agent architecture: `get_detector_status()`, `flag_anomaly()`, `send_alert()`, `generate_report()`

> 💡 **Relevance:** This is **Michael Carpenter's** project — he is the PI / big boss of Gammasphere at ANL. The `openclaw_framework` is his early prototype of an LLM-assisted Gammasphere monitoring system, using a file-based job queue to dispatch Claude API calls. The conversation log (`Gammasphere_angentic.md`) documents him exploring local LLM options (Mac Mini M4 → eventual DGX Spark) and planning a PyEPICS-based agent to monitor all 110 Ge detectors with alerts + shift reports. Worth coordinating with — there may be overlap or collaboration opportunities with the General DGS monitoring work.

---

## global_32 Structure (2026-04-05)
*Source: SSH exploration — 2026-04-05*

**Top-level dirs:** `devel/`, `edmroot/`, `ioc/`, `openclaw_framework/`, `SBX_SDCard_Image/`, `svn_devel/`, `tgz/`, `results/`, `tempchk/`, several old BSP trees (`devel6/`, `devel7_newbsp/`, etc.)

**devel/systems/gs/** — GS-specific live software:
- `lnfill/` — **active** lnfill Python+DB tree (fill logs through 2026-03-30)
- `lnmain`, `lnmain2` — LN main scripts/symlinks
- `dbase/`, `edm/`, `gscommander/`, `gscommander2/`, `resm/`, `tar/`

**devel/systems/** — other systems: `dfma/` (just README), `SBX/` (EProms, GenDB, test), `dub/` (empty), `caribu/`, `clov/`, `mobacq/`

**edmroot/** — EDM screen roots:
- `DGScommander`, `DGScommander2`, `digcntrl`, `lncntrl/` (screens/ + scripts/), `sbxcntrl/`, `null`

**ioc/** subdirs:
- `dgsSoftIOC/` — soft IOC loading JustGlobals.db, Fanout_leader.db, JustGlobals_VME.db, Fanout_leader_VME.db, dgsSupport.db; boot cmd by JTA 2020-06-24
- `FW_Maint/` — firmware binaries: MDIG, MTRG, RTRG, SDIG + BUS_LEFT/RIGHT.bin, router_top.bin; Flash Maintenance Instructions
- `gui/` — dgscommander, dgscommander2, dgscommander2_r8, dgscommander_r8, lnmain_r8, tracecommander, screens/, scripts/, current_status_DGS*.save
- `dgsReceiver/` — standalone DAQ receiver (basic_settings.sh, constant.h, legacy_code/, simpleStartStop.sh)
- `fastEventContructor/` — event builder: EventBuilder.cpp, class_DIG.h, class_Hit.h, BinaryReader.h, PP.cpp
- `py_scripts/` — compare_pvs.py, trace_throttle.py/2, DGS_SYSTEMDEF.TXT, DNG_TO_REGIN_MAP.TXT, COLLSCAN.TXT
- `bin/vxWorks-ppc604_long/` — VxWorks cross-compile output
- `epics/` — EPICS base install

**openclaw_framework/** — Michael Carpenter's LLM framework (config, docs, jobs, logs, results, runners, state, templates, tmp)
**SBX_SDCard_Image/** — single file: `sbxRR.image`
**svn_devel/** — SVN mirror of devel tree

---

## lnfill IOC — Hardware Findings (2026-04-05)

**Physical IOC:** Motorola MVME167 VME SBC, 68040 CPU, VxWorks
**Boot host:** ln2con (Fedora 12, 192.168.203.148) via network
**Console:** `cu -s 9600 -l /dev/ttyS0` on ln2con
**Databases:** `gamln.db` (23,078 lines — valves/manifolds) + `tempmon.db` (429 lines — temps)

**Hardware: Allen-Bradley 1771 I/O rack** (PLC backplane system)
EPICS device support types in gamln.db:
- `AB-16 bit BO/BI` — valve open/close control + readback
- `AB-1771IFE-SE` — AB 1771-IFE analog input (LED sensors / temps)
- `AB-Binary Input` — generic binary input
Link syntax: `#L0 A3 C1 S11 @` = Link / Adapter / Crate / Slot

**Replacement implications:**
- Must either keep the AB 1771 rack (and write a new driver) or replace the hardware too
- PV names (`LNG1-01_FV:VC`, `LNH1-01_SM`, `MOD107_DV_TEMP`…) can be replicated in any new soft IOC
- Live lnfill NFS tree: `/dgsdata/fs2/vol2/global_32/devel/systems/gs/lnfill/`
- Git repo (`DGS_tools_pack/lnfill/`) is the ported Pi-based version (Python/pyepics control layer only, not the IOC itself)

---

## What to Explore Next (Heartbeat G queue)

- [x] `vol2/global_32/ioc/py_scripts/trace_throttle.py` — documented 2026-04-05
- [x] `vol2/global_32/ioc/py_scripts/compare_pvs.py` — documented 2026-04-05
- [x] piserver extra 3 MACs — identified 2026-04-06 (spare/unassigned Pis)
- [x] `trace_throttle2.py` — documented 2026-04-06 (see below)
- [ ] `vol2/dgscalib/` — what's in `bin/` and `calib/`? (partial: dir listed, not read)
- [ ] `vol4/dgs_testing/GEBSort/` — GEBSort build/config?
- [ ] `vol3/sbx2022tuning/` — SBX tuning data
- [ ] `vol2/global_32/edmroot/lncntrl/screens/` — LN EDM screens (grep only, no full reads!)
- [ ] `vol2/global_32/devel/systems/gs/lnfill/gamln.db` — extract unique PV prefixes/record types for documentation

### trace_throttle2.py — Updated Connectivity Mapper ✅ read 2026-04-06
*Source: `ssh dcsu@DCS2.onenet "cat /dgsdata/fs2/vol2/global_32/ioc/py_scripts/trace_throttle2.py"` — 2026-04-06*

Revised version of `trace_throttle.py`. Key improvements over v1:
- Uses `datetime.datetime.now()` correctly (v1 had `datetime.now()` bug)
- MDIG/SDIG pair verification consolidated into `verify_mdig_sdig_pairs()` function
- Output now tab-separated with full columns: Crate, MDIG, SDIG, Router, Cable, MDIG_Worked, SDIG_Worked, Status
- Status values: `OK`, `BOTH_FAILED`, `MDIG_FAILED`, `SDIG_FAILED`, `PAIR_MISMATCH`
- Still Python 2 syntax (`print` statements, `except Exception, e:`) — needs Python 3 port
- `write_results()` writes `connectivity_map.txt` with timestamp header
- Summary prints: total pairs tested, successful, issues
- Resets global throttle to 0 on exit (including error/interrupt)

### vol2/dgscalib/ — Calibration Data (partial)
*Source: `ssh dcsu@DCS2.onenet "ls -la /dgsdata/fs2/vol2/dgscalib/bin/ && ls -la /dgsdata/fs2/vol2/dgscalib/calib/"` — 2026-04-06*

**`bin/`** contains:
- `GEBSort_Runtmp1.log` — 90 KB GEBSort run log (Aug 2021)
- `gebsort.sh` / `gebsort.sh~` — GEBSort launch script (Nov 2019)
- `gsfma373/` — experiment directory
- `map.dat` — detector map file (16 KB, Oct 2019)
- `README` — small README
- `scr.txt` — script output (17 KB)
- `TS.list` — timestamp list (Aug 2021)

**`calib/`** contains:
- `calib.tar` — archived calibration files
- `feb6/` — Feb 6 2020 calibration set
- `fwbase.cc` — C++ base firmware calibration utility
- `.spe` spectrum files — per-detector (ge9, ge13, ge17, ge33, ge34) from Feb 2020
- `get_bgocln.cc`, `get_bgodrty.cc`, `get_bgoraw.cc` — BGO calibration extractors (ROOT macros)
- `get_ecln.cc`, `get_eraw.cc`, `get_pz.cc` — energy/PZ calibration extractors (ROOT macros)

> These are ROOT-based calibration macros for extracting BGO and Ge energy/PZ calibration from `.spe` spectra. Historical reference, likely superseded by current calibration workflows.

---

*Source: SSH exploration of DCS2.onenet as dcsu, 2026-04-05.*
