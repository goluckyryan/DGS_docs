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

**vol5 sizes (explored 2026-04-13):** exp1756_Hoff 7.8T, exp2026_Mueller-Gatermann2 12T, exp2045_Rogers 18T, exp2008_Chiara 19T. Other experiments (exp1985x, exp2017, exp2031x, exp2059, exp2071, exp2078, exp2092x, exp2202, ebss2024, cmg, ChicoTest, 253no) not yet sized.

### vol5/exp2008_Chiara/ — Active Experiment Structure ✅ explored 2026-04-13

Directory layout:
```
exp2008_Chiara/
├── data/           # Raw GEB run files: exp2008_NNN/exp2008_NNN_000_DDDD_C (per-digitizer)
├── docs/
├── gebsort/        # GEBSort output: PZ scan runs (0.88.root, 0.89.root, etc. + .dgs_pz.cal + .basepj)
├── gebsort_cmg/
├── LOG/
├── LOG_FILES/
├── Merged/
├── ROOT_FILES/
├── RunTimestamp.txt
└── scripts/        # Per-experiment Python tuning scripts (NOT in git repo)
```

**`scripts/` tuning tools** (experiment-specific, written by Ryan/JTA):
| Script | Purpose |
|--------|---------|
| `ge_set_gain.py` | Set GeCenterGain / GeSideInputSelect / BGOpSelect for all 110 GS channels in parallel (ThreadPoolExecutor, MAX_WORKERS=20) |
| `ge_dc_offset_tune.py` | DC offset tuning for Ge channels |
| `ge_threshold_tune.py` | Ge discriminator threshold tuning |
| `sdig_led_threshold.py` | Slave DIG LED threshold adjustment |
| `set_pv.py` / `set_pv.sh` | Generic PV setter |
| `dgs_trace.py` | Waveform trace capture utility |

> These scripts use `subprocess` + `caput`/`caget` CLI or pyepics directly. They are experiment-specific and not versioned in the main repos — copies live on NFS only.

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
| `dgs_testing/` | Sep 2025 — contains `GEBSort/` (208-file source tree), `GEBSort_old/`, `calibration.txt`, `Merged/` (86 `.gtd` files), `Merged_haha/`, `dgsdata/` (receiver binaries + run dirs) |
| `mbo/` | 123 GB — Michael Oberling trigger timing test data; `mult3_times_*.bin` files (10ns/30ns/100ns coincidence window scans with/without holdoff), `gebmerge.sh`/`gebmerge_new.sh` (uses `rcvr_merge` for data merging, not GEBMerge), core dumps (bug investigation) |
| `1930_Favier/` | 5.2 TB — Favier experiment; `dgsdata/` (dgsReceiver binaries + dgs_runNNN dirs), `GEBSort/` + variants (`GEBSort_2019`, `GEBSort_ak`, `GEBSort-JB`, `GEBSort_rc`), `Merged/`, `Merged_decay/`, `dfmadata/`, `dxadata/`, `map_decay.dat` |
| `1984_Recchia/` | 2.4 TB — Recchia experiment; similar structure |
| `dgs20230807/` | 2.9 TB — DGS commissioning/test dataset (Aug 2023); `dgsdata/` (dgs_run001–002), `GEBSort/` + `GEBSort_ak/`, `Merged/` + multi-experiment merged trees, `dgs_pz_M700.cal`, `run_infos/` |
| `yjc/` | **184 GB** — Y.J. Chen analysis area (Feb 2026, active). Contents: `176hg_dgsdata/` + `176hg_dxadata/` + `176hg_dfmadata/` (raw GEB data for 176Hg experiment); `ROOT_FILES_189at/` + `ROOT_FILES_190at/` + `TREE_FILES_*/` (ROOT analysis output for 189,190At experiments); `geb-sort-tac2/` binary; `parquet-sort/` dir; `merge.py`; `Macros/`; compiled ROOT dict (`geb_class.h`, `.so`, `.pcm`, `.d`) |
| `ML_AK/` | Jul 2025 — machine learning? |
| `NeutronShell_testing/` | **654 GB** — Neutron shell testing / commissioning area (Apr 2024). Contains: `GEBSort/` (standard), `GEBSort_VK/` (V. Kumar variant), `.spc` spectra files (0.spc–6.spc), `2023_Nolen.code-workspace` VS Code workspace; `xiadata/` + `XIA_udp_linux/` — XIA Pixie-16 DAQ data (UDP-based readout, different DAQ chain from DGS). The Pixie-16 data coexists with DGS analysis infrastructure. |
| `exp2019_Marin/` | Marin experiment — follows standard structure |
| `exp2026_Mueller-Gatermann/` | **5.3 TB** — Mueller-Gatermann experiment; `dgsdata/` (170 run dirs), `GEBSort/` + `GEBSort_new/` + `GEBSort_xa/`, `Merged/`, `dfmadata/` + `dxadata/` + `plunger/` + `dgsReceiver/` + `dgstests/` ✅ explored 2026-04-13 |
| `exp2027x_Lopez-Caceres/` | Lopez-Caceres experiment (x = extension/supplemental) |
| `exp2051_Heery/` | **8.3 TB** — Heery experiment; `dgsdata/` (170 run dirs), `GEBSort/` + `GEBSort_VK_VK/`, `Merged/` (GEBMerged_*.gtd_000 files), `dfmadata/` + `dubdata/` + `xiadata/` + `plunger/` ✅ explored 2026-04-13 |
| + many other experiment dirs | 1859–2175 range (exp-prefixed) — **Standard pattern:** `dgsdata/dgs_runNNN/` (raw GEB files), `GEBSort*/` (sort code + config), `Merged/` (GEBMerged*.gtd), optional subsystem dirs: `dfmadata/`, `dxadata/`, `dubdata/`, `xiadata/`, `plunger/` |

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
| b8:27:eb:99:18:3f | .149 | NW | 204 | ✅ corrected 2026-04-08 — README has typo (`19`→`18`); tftpboot dir `b8-27-eb-99-18-3f` confirms `18` |

**3 additional MAC dirs** (not in README — spare/decommissioned, hostname not configured):
- `b8-27-eb-39-f2-ce` — spare Pi (default hostname, no location assigned)
- `b8-27-eb-91-bd-1b` — spare Pi (default hostname, no location assigned)
- `b8-27-eb-df-8c-d6` — spare Pi (default hostname, no location assigned)

> ✅ Resolved 2026-04-08: README lists NW box MAC as `b8:27:eb:99:19:3f` but tftpboot dir is `b8-27-eb-99-18-3f` (byte 5: `19` vs `18`). The tftpboot dir contains valid boot content (cmdline.txt, dtb files), confirming the **actual Pi MAC is `b8:27:eb:99:18:3f`** and the README has a typo (`19` should be `18`). The nfs_layout.md MAC table above already shows the corrected value.

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
- Live lnfill NFS tree: `/dgsdata/fs2/vol2/global_32/devel/systems/gs/lnfill/` (active working copy)
- Archived legacy copy: `/dgsdata/fs1/vol2/global_sl7/devel/systems/gs/lnfill/` (older, fewer files — see sl7 section below)
- Git repo (`DGS_tools_pack/lnfill/`) is the ported Pi-based version (Python/pyepics control layer only, not the IOC itself)

**gamln.db PV analysis** ✅ 2026-04-13 — `ssh dcsu@DCS2.onenet "grep -c 'record(' .../gamln.db"`

1,357 EPICS records in `gamln.db`. Record type breakdown:
| Type | Count | Role |
|------|-------|------|
| `mbbo` | 415 | Multi-bit output — valve modes (Auto/Open/Disable), alerts |
| `sub` | 406 | Subroutine — complex fill logic, state machines |
| `bo` | 179 | Binary output — valve open/close control |
| `ai` | 166 | Analog input — sensors (level, pressure, temp, Vcc) |
| `calc` | 140 | Calculated — threshold checks, Vcc monitors |
| `bi` | 24 | Binary input — valve status readbacks |
| `stringout` | 12 | String output |
| `ao` | 8 | Analog output |
| `stringin` | 5 | String input |
| `mbbi/state` | 2 | Multi-bit input / state |

PV prefix hierarchy (from `gamln.db`):
| Prefix | Meaning |
|--------|---------|
| `LNG1`, `LNG2` | GN2 (gaseous nitrogen) valve groups 1/2 |
| `LNH1`–`LNH4` | LN header valves 1–4 |
| `LNM1`–`LNM4`, `LNM1A`–`LNM4A` | LN manifold valves 1–4 (A = alternate) |
| `LNP1`, `LNP2` | LN pressure sensors |
| `LNS1`, `LNS2` | LN supply valves |
| `LNT1`–`LNT6` | LN tank valves 1–6 |
| `LNVC` | LN VCC (sensor power) monitors |
| `LN` | Top-level LN system PVs (fill mode, setpoints, heartbeat) |
| `SETPOINT` | Fill setpoint PV |
| `TEMPHI/LO/MC/DO` | Temperature alarm thresholds |
| `CNOTOK` | Connection-not-OK alarm record |

---

## Detailed Findings — Subsections

_(All originally-queued areas now explored and documented below. Remaining: vol5/exp2059_Ackermann, vol5/exp2071_Andreyev — lower priority.)_

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

### vol4/dgs_testing/ — DGS Testing Area (Sep 2025)
*Source: `ssh dcsu@DCS2.onenet "ls /dgsdata/fs2/vol4/dgs_testing/"` — 2026-04-13*

A working test area used for Sep 2025 DGS testing. Contains:

| Subdirectory | Contents |
|---|---|
| `GEBSort/` | 208-file GEBSort source tree (full `.c`/`.h`/`.o` + `UserChat.h`, `UserDeclare.h`, `UserEv.h`, `UserFunctions.h`, etc.) — full User*.h customization headers present |
| `GEBSort_old/` | Earlier GEBSort version (includes `bgo_threshold_scan*.txt`, `am243.sou`, `AR.snippets`) |
| `calibration.txt` | Energy calibration table: 3-column format (GS hole #, offset, gain, ~0.60–0.62 keV/ch); holes 27–49+ |
| `Merged/` | 86 merged `.gtd` files (`GEBMerged_run_*.gtd_000`, `GEBMerged_haha*.gtd_000`) |
| `Merged_haha/` | Additional merged output (1 file seen: `GEBMerged_haha0.gtd_000`) |
| `dgsdata/` | Receiver binaries + numbered run directories (see below) |

**dgsdata/ — Receiver Executables (built May 2025):**
- `dgsReceive` / `dgsReceiver` — standard receiver (43,952 bytes, May 13 2025)
- `dgsReceiver_mca` — MCA-mode receiver (44,134 bytes, May 13 2025)
- `dgsReceiver_nosave` — no-save mode (39,817 bytes, Feb 5 2025)
- `dgsReceiver_old` — older build (May 8 2025)
- `dgs_run001` through `dgs_run027` (+ runtest, runtrace dirs) — per-run raw `.gtd` files
  - Files named `dgs_run001.gtd01_000_01NN_X` (crate + channel suffix hex-encoded)
- `start_run.sh`, `stop_run.sh`, `dgs_start_run_no_save.sh` — run control shell scripts
- `histogram_daemon`, `histogram_from_binary`, `histogram_from_binary_rt` — live histogram tools (with `.c` sources)
- `mca`, `mca_run44–46` — MCA data dirs
- `live_plot.gnu` — gnuplot live plot script
- `core.18302`, `core.7887` — core dumps (crash artifacts)

> **Cross-reference:** The `dgsReceive`/`dgsReceiver` binaries are standalone C programs (not ANLDAQ). The ANLDAQ Python GUI is the current production receiver; these binaries likely predate it or are test stand variants. The `.gtd` raw file format matches the GEB data format (see `dgs/data_structures.md`).

---

### vol4/mbo/ — Mike Carpenter's Workspace (Dec 2025) ✅ 2026-04-13
*Source: `ssh dcsu@DCS2.onenet "ls -la /dgsdata/fs2/vol4/mbo/"` — 2026-04-13*

Active workspace for Mike Carpenter (PI). Contains custom GEB merge/sort utilities and trigger timing studies.

**Custom C/C++ merge tools:**

| File | Description |
|---|---|
| `rcvr_merge.c` / `rcvr_merge` | GEB merge: reads multiple receiver output files, merges by timestamp, writes split output. |
| `rcvr_merge_new.c` / `rcvr_merge_new` | Revised version: 32 KB read / 2 KB write buffers, up to 1024 input files. Compile: `gcc -O3 rcvr_merge_new.c -o rcvr_merge_new` |
| `rcvr_merge_test.c` / `rcvr_merge_test` | Test variant of rcvr_merge_new. |
| `rcvr_data_scrubber.c` / `rcvr_data_scrubber` | Data cleaning/scrubbing tool. |
| `rcvr_unmerge` | Reverse: split merged GEB file back into per-crate files. |
| `gebmerge.sh` / `gebmerge_new.sh` | Shell wrapper: copies binary to run dir, runs merge with 10 GB max file size. Takes run number as arg. |

**GEB header struct used by `rcvr_merge_new.c`:**
```c
typedef struct {
    int32_t  type;       // GEB packet type
    int32_t  length;     // payload length in bytes
    uint64_t timestamp;  // 64-bit timestamp
} GEB_HEADER;
```
> Note: Uses a 64-bit `timestamp` field vs the standard 6-byte (48-bit) format in `data_structures.md`. May be extended-format or the upper 16 bits are unused padding — worth clarifying.

**Trigger timing analysis (Python):**

`trigger_reply_*.py` scripts analyze trigger timing distributions from binary event files (16-byte fixed-size events, 48-bit LE timestamp at bytes 8–13). Use a sliding window to find coincidences at various time thresholds:

| Script variants | Windows tested |
|---|---|
| `trigger_reply_10ns.py` / `_mid.py` / `_mid_holdoff.py` | 10 ns threshold |
| Same pattern for 30 ns, 50 ns, 100 ns | Multiple thresholds |

**Output data (Dec 2025):**

| File/Dir | Size | Description |
|---|---|---|
| `test.out.0.gtd` | 9.6 GB | Large merged GEB file |
| `mult3_times_*.bin` | 26–323 MB | Multiplicity timing distributions per threshold/holdoff |
| `test_103/`, `test_104/` | EXP1917 runs 103–104 raw `.gtd` files + timing analysis scripts |
| `test_20/`, `test_20_ge/`, `test_20_trig/` | Run 20 data subsets |
| `core.*` | Crash core dumps from Dec 15 2025 |

> **Cross-reference:** `rcvr_merge` tools are NFS-only (not in any Git repo). Trigger holdoff tuning results here are directly relevant to MTRG holdoff register settings. See `dgs/deep_fpga_MTRG_MAIN.md` for trigger holdoff configuration.

---

### vol3/sbx2022tuning/ — Slope Box Tuning Data (2022)
*Source: `ssh dcsu@DCS2.onenet "ls /dgsdata/fs2/vol3/sbx2022tuning/"` — 2026-04-13*

| Subdirectory | Contents |
|---|---|
| `dgsdata/` | Raw DGS data from SBX tuning runs |
| `GEBSort/` | GEBSort source tree used for this tuning campaign |
| `gtreceiver/` | GT (GRETINA) receiver — legacy receiver binary/source |
| `LOG_FILES/` | Run logs |
| `Merged/` | Merged output `.gtd` files |
| `ROOT_FILES/` | ROOT output histograms/trees from SBX tuning |

> **Context:** Used for Slope Box characterization in 2022. `gtreceiver/` suggests this predates or overlaps with the migration from GRETINA-based tools. `GEBSort/` is the event sorter/builder used to process raw data. ROOT_FILES contains analysis output. See `dgs/sbx.md` for SBX hardware documentation.

---

---

### fs1/vol2/global_sl7/devel/systems/gs/lnfill/ — Legacy LNFill Scripts (archived)
*Source: `ssh dcsu@DCS2.onenet "ls /dgsdata/fs1/vol2/global_sl7/devel/systems/gs/lnfill/"` — 2026-04-13*

An **older/archived copy** of the LN2 fill Python scripts on the `fs1` server (Scientific Linux 7 global environment). Fewer files than the active `fs2/vol2/global_32` copy — missing many utilities like `TankMan.py`, `set_hilo_lim.cr6`, etc.

| File | Description |
|---|---|
| `gamln.db` | Same 23k-line EPICS DB as active copy |
| `GeStat.py` / `GeStat.pyc` | Ge detector status class (reads `MODxxx_DV_TEMP`, `MODxxx_DV_EN` PVs via CaChannel) |
| `LNFill.py` | Main fill script — takes detector IDs as args, reads fill mode PV (`LNFILL_MODE:XC`), logs to `LNFill.YYYY.log`; Python 2 |
| `LNFill_monitor.py` / `LNFill_monitor2.py` | Monitoring loop scripts |
| `LNFill_Auto.sh` | Bash loop: runs `LNFill_monitor2.py` every 1200s for 500 iterations |
| `tempmon.db` | Temperature monitor EPICS DB |
| `LNFill_Auto.log`, `LNFill_Auto..log` | Auto-run logs |
| `LNFill.2016.log`–`LNFill.2018.log` | Historical fill logs 2016–2018 |
| `lastFill.log` | Most recent fill record |
| `Warmup_ManD_Feb22.log` | Manual warmup log Feb 2022 |
| `fill_20200819_*.log` | August 2020 fill run logs |
| `sandbox/` | Test/scratch area |

**Key architecture details from `LNFill.py`:**
- Uses `CaChannel` (legacy EPICS Python binding, not pyepics)
- Fill log path was hardcoded to `/global/devel/systems/gs/lnfill/LNFill.2018.log` (old NFS path)
- `GeStat` class reads `MODxxx_DV_TEMP` and `MODxxx_DV_EN` PVs for each detector
- Argument: list of GS detector IDs (integers 1–110+)
- Checks `LNFILL_MODE:XC` PV before filling

> **Context:** This is the **predecessor** to `DGS_tools_pack/lnfill/` (current Pi-based system). Shows the original Python 2 + CaChannel architecture before migration to Python 3 + pyepics. The core IOC (`gamln.db`) is unchanged.

---

---

### vol2/global_32/edmroot/lncntrl/screens/ — LN Control EDM Screens ✅ 2026-04-13
*Source: `ssh dcsu@DCS2.onenet "ls + grep /dgsdata/fs2/vol2/global_32/edmroot/lncntrl/screens/"` — 2026-04-13*

Only 2 EDM screen files (plus backups):

| File | Lines | Purpose |
|------|-------|--------|
| `lnmain.edl` | 463 | LN main operator overview screen |
| `tempmon.edl` | 7658 | Per-detector temperature monitor (all 110 Ge modules) |

**`lnmain.edl` — key PVs:**
- `LN_ATOD:XC` — auto-to-demand fill mode
- `LN_MODE:XC` — LN operating mode
- `LN_ATNF:XC` — auto tank fill
- `LN_FILL_MODE:XC` — fill mode control
- `LN_ALMSTAT:XC` — alarm status
- `LN_REBOOT:XC` — IOC reboot trigger
- LN IOC telnet: `192.168.203.121` (the MVME167 VME SBC)
- Links to sub-screens: `lncontrols.edl`, `lnsettings.edl`, `lnalarm.edl`, `lnmanifold.edl`, `lntanks.edl`, `lnmansum.edl`, `lnmanup.edl`, `lnmandn.edl`, `lnmandnA20.edl`

**`tempmon.edl` — key PVs:**
- `MOD001_DV_TEMP` through `MOD110_DV_TEMP` — temperature per Ge detector (110 total)
- `MOD001_DV_EN` through `MOD110_DV_EN` — enable/disable per detector
- Covers all 110 Gammasphere Ge module slots

> The LN IOC EPICS address is `192.168.203.121` (the MVME167 VME CPU running VxWorks). The EDM screens here are the legacy operator interface — currently the Pi lnfill system handles the fill logic while the IOC continues to manage the hardware directly.

---

### fs1/vol2/global_sl7/ — Legacy SL7 Build Environment ✅ 2026-04-13
*Source: `ssh dcsu@DCS2.onenet "ls /dgsdata/fs1/vol2/global_sl7/"` — 2026-04-13*

Note: This is `fs1.onenet` (not `fs2.onenet`) — a separate 40T volume. Path: `/dgsdata/fs1/vol2/`.

**Top-level experiments on fs1/vol2:**
| Directory | Size | Notes |
|-----------|------|-------|
| `exp1758/` | 7.4T | Older major experiment — 204 merged GTD runs |
| `exp1881/` | 3.2T | 124 merged GTD runs |
| `gsfma370–372/` | — | GSFMA test experiments (includes `dgsdata/` raw dirs) |
| `cmg/` | — | (not accessible — path issue) |
| `global_sl7/` | — | SL7 global build environment |
| `test/` | — | Test data/GEBSort/Merged |

**global_sl7/ subdirs:**
- `devel/` → `devel6/` (symlink) — SL6.8 from dgs1
- `devel6/` — SL6.8 build tree
- `devel6_sl7/` — SL7.6 development build (most current on fs1)
- `devel7_newbsp/` — SL6.8 with new IOC BSP support
- `develbuild/` → `devel_tjm/` (symlink)
- `new_mv5500_bsp/` — MVME5500 BSP experiment
- `README` — documents: `devel6`=SL6.8 from dgs1, `devel6_sl7`=SL7.6 devel, `devel7_newbsp`=SL6.8 new BSP

**devel6_sl7/dgs/ — DGS code:**
- `edm/screens/` — full set of EDM screens (same as fs2 but SL7 variant): `acqControl.edl`, `DGSchannel.edl`, `DGSIOC.edl`, `DGSrunControl.edl`, `Router.edl`, `Trace.edl`, `Trace_all.edl`, `TrigLock.edl`, `Master.edl`, `chargeInj.edl`, `decomps.edl`, `getrigall.edl`, etc.
- `edm/scripts/global_config` — bash script that sets global EPICS PV defaults via `caput`:
  - `CryG_CS_*` global crystal settings (BRE, Pol, PZEna, LEDTh, RDDly, RDLen, CFDDly, CFDThr, CFDFrac, etc.)
  - `DigG_CS_*` global DIG settings (ExtWin, NoiseWin, PileWin, TrigDly, LRCollTime, IntTime, CollTime, BaseRes, AuxIOMux, DCMReset)
  - Per-IOC MuxCon, MLE, DAQB enable settings for all 12 IOCs
- `edm/scripts/trigger_config` — bash sets trigger EPICS PVs via `caput`:
  - `Trig0_CS_ClkSrc local` — master trigger uses local clock
  - LRU control: positions 00,01,02,04,05,06,08,09,10,15 = ON
  - Input link masks (ILM): A,B,C=0; D,E,F,G,H=1; L,R,U=0
  - Transmit power: all cables A–H,L,R,U = 1
- `firmware/` — DIG firmware: `chip_top_1.06_0016.bin`, `chip_top_1.06_0017.bin` (1.06 = older than current)
- `firmware/mt/trigger_top.bin` — MT trigger firmware binary
- `dgscommander`, `dgscommander2` — commander executables (not directories)

**devel6_sl7/gtreceiver/ — GT Receiver Source:**
Full source for the GRETINA/DGS GT receiver program:
- `gtReceiver4.c` / `gtReceiver4.h` — main receiver (v4, last rev 1.13, Sep 19 2013, TL author)
- `gtReceiver2.c`, `gtReceiver3.c` — earlier versions
- `chicoReceiver` — CHICO2 receiver variant (standalone binary)
- `nullRcv.py` — Python null receiver
- `psNet.h`, `receiver.h` — network/receiver headers
- `Makefile.Linux_i386`, `Makefile.Linux_x86_64` — 32/64-bit builds
- Comment: "This code was proudly stoled from Carl Lionberger at LBL"
- Purpose: standalone network data receiver for GRETINA-format data at DGS; connects to multiple servers (MAX_SERVERS=4), receives 7KB packets (PACKBYTES=1024*7), writes GRETINA format (`WRITEGTFORMAT=1`)

**runcon/ — Run Control:**
- `1-1/runcon.py` — wxPython 2.x GUI run control app (circa 2005, GRETINA-era)
  - Manages runs: new run dialog (run number + comment + cabling file), run directory creation
  - Uses `/remote/devel/runcon/connections/*.sgl` cabling files
  - Data paths: `/remote/data/gretina`, `/remote/devel/gretDAQ`
  - This is legacy GRETINA run control, not DGS-specific
- `connections/*.sgl` — signal connection files (2005 era: `05032005.sgl`, `06172005.sgl`, etc.)
- `runs/` — named run config dirs: `coincidence_scan_I/II/III`, `NoiseAnlysis`, `t1–t5`

> **Context:** The `fs1/vol2` volume is legacy storage, mostly matching `fs2/vol2` in structure but older. The EDM screens and `global_config`/`trigger_config` scripts are operational reference — the `caput` values in `global_config` are the baseline DIG crystal/digitizer settings for GS runs. The trigger_config shows the TTCL LRU wiring pattern for the GS run (positions 00,01,02,04,05,06,08,09,10,15 active).

*Source: SSH exploration of DCS2.onenet as dcsu, 2026-04-05 / 2026-04-13.*

---

### fs2/vol5 — Selected Experiment Directories (survey)
*Source: `ssh dcsu@DCS2.onenet "ls /dgsdata/fs2/vol5/"` — 2026-04-14*

Top-level directories on vol5 (18 named experiments + test areas):
`253no`, `ChicoTest`, `cmg`, `ebss2024`, `exp1756_Hoff`, `exp1985x_Chowdhury`, `exp2008_Chiara`, `exp2017_Morse`, `exp2026_Mueller-Gatermann2`, `exp2031x_Hartley`, `exp2045_Rogers`, `exp2059_Ackermann`, `exp2071_Andreyev`, `exp2078_Huang`, `exp2092x_Reviol`, `exp2202_Mattera`, `GT_test_ak`, `mpc`, `MSM`

**exp2045_Rogers** (18 TB total):
- `dgsdata/` (5.2 TB) — 173 raw run dirs (`dgs_run1`–`dgs_run172` + misc)
- `Merged/` (4.9 TB) — 162 GEBMerged files
- `xiadata/` (7.7 TB) — XIA detector data (separate DAQ system)
- `GEBSort*/` — multiple analysis directories (GEBSort_ak, GEBSort_AR, GEBSort_co, GEBSort_DTD, GEBSort_Ni, GEBSort_spb, GEBSort_VK, etc.)
- `Merged_XIA/` — merged DGS+XIA data
- Pattern: large combined DGS+XIA experiment; many parallel GEBSort analysis directories suggest iterative calibration runs by multiple analysts

**exp1756_Hoff** (7.8 TB total):
- `dgsdata/` + `dfmadata/` — **DGS+DFMA combined experiment** (both DAQ systems running simultaneously)
- `Merged/` — merged data
- `GEBSort*/` — multiple analysis dirs including GEBSort_dssd, GEBSort_exp1977, GEBSort_12MeVA, GEBSort_6MeVA
- `scripts/` — experiment scripts
- `start_both_daq.sh` / `stop_both_daq.sh` — scripts to start/stop both DGS and DFMA DAQ simultaneously; evidence of multi-system coordination
- Pattern: multi-system run (DGS+DFMA), DSSD detector data, beam energy variants (12 MeV/A, 6 MeV/A)

## Cross-References

- `dgs/collectorboxpi.md` — Raspberry Pi soft IOC; PXE boot infrastructure served from fs2.onenet piserver NFS
- `dgs/influxdb_grafana.md` — InfluxDB/Grafana on DCS2 (192.168.203.56); same server as NFS mounts
- `dgs/expMemory_2008_Chiara.md` — Active experiment data locations on NFS (vol3/vol4 paths)
- `dgs/lnfill.md` — LN2 fill system; lnfill scripts on vol3, ln2con home on vol3
- `dgs/ANLDAQ.md` — Data acquisition; raw run files land on NFS vol4/vol5
- `dgs/dgs_analysis.md` — Post-analysis; reads from NFS experiment directories
