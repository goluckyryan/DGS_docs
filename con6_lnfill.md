# con6 & lnfill Build Environment

Stability: C2 - Active / semi-stable

_Source: `DGS_tools_pack/ln2con/Con6_Inventory.md`, `Plan.md`, `Con6_Retirement_Plan.md`_
_Documented: 2026-04-19 by General DGS_

---

## Overview

**con6** (`192.168.203.136`) is a **Sun Blade 100 running Solaris 10 (SunOS 5.10)** — the original analog Gammasphere software development server, predating DGS. Most content is from the **analog DAQ era** (pre-DGS).

Still critical today:
- **CVS source repository** for the lnfill IOC (the only copy of the C source for `lnfiller.vx`)
- **VxWorks 5.1.1 68040 cross-compiler** (Solaris/SPARC native — the ONLY toolchain that can rebuild `lnfiller.vx`)
- **Production lnfill tree** (`Rev6-01-04/`) mirrored from ln2con

⚠️ **con6 must stay alive to rebuild the lnfill IOC binary.** If con6 fails and the source/toolchain are not archived, `lnfiller.vx` cannot be recompiled. ✅ verified 2026-04-23 — `ln2con/Con6_Inventory.md` (CVS `lnfill/` source + `vxw_5.1.1/solaris.68k/bin/cc68k` toolchain); `ln2con/Plan.md` (rebuild path uses con6 CVS + `cc68k`, notes Solaris/SPARC-native toolchain)

✅ verified 2026-04-18 — direct SSH to `dgs@192.168.203.136` + `Con6_Inventory.md` survey

---

## con6 Access

```bash
sshpass -p 'gam$hippie' ssh \
  -o KexAlgorithms=+diffie-hellman-group1-sha1 \
  -o HostKeyAlgorithms=+ssh-rsa \
  -o PubkeyAcceptedKeyTypes=+ssh-rsa \
  dgs@192.168.203.136
```

Password: `gam$hippie` (same as all DGS systems).

---

## Key Directories on con6

### CVS Repository: `/home/gam/repository/`

| Module | Contents | Importance |
|--------|----------|------------|
| `lnfill/` | LN2 fill IOC — complete C source + EPICS DB + startup scripts | **CRITICAL** — needed to rebuild lnfiller.vx |
| `gam/` | Core hardware library source (`sysAnalysisBd.vx` + `sysAnLEDsupport.vx`) | **CRITICAL** — IOC loads these at startup |
| `daq/` | Old analog GS DAQ (gate, sender, taper, eff) | Historic — superseded by DGS |
| `ddl/` | DDL compiler | Historic |
| `notes/` | Historical design notes (authors: akb/ann/ehh/rab/moog) | Valuable archive |

### `lnfill/src/` Source Files

| File | Description |
|------|-------------|
| `dfill_sub.c,v` | **Detector fill subroutine** — `dfill_sub_init()` sets hose→detID mapping |
| `setup.c,v` | System setup — reads `ln.inits`, initializes hardware |
| `fill.st,v` | EPICS state sequencer program |
| `tfill_sub.c,v` | Tank fill subroutine |
| `valve_sub.c,v` | Valve control |
| `ln_dbaccess.c,v` | Database access |
| `history.c,v` | Fill history logging |
| `hardware.h,v` | Allen-Bradley 1771 hardware definitions |
| `lndefs.h,v` | LN system definitions |
| `filler_globals.h,v` | Global variables |
| `temp_sub.c,v` | Temperature subroutine |
| `tempmon_subs.c,v` | Temperature monitoring |

### Production Trees: `/home/gam/prod/`

| Directory | Contents |
|-----------|----------|
| `lnfill/Rev6-01-04/` | **Active lnfill production tree** — same tree as on ln2con |
| `lnfill/Rev6-01-01/` to `Rev6-01-03/` | Older archived revisions |
| `lnfill/dbase/` | Hose→detID database management (SQL scripts, `ln.inits`, `add_hole.awk`) |
| `vxWorks/vxw_5.1.1/` | **68040 cross-compiler toolchain** (critical — see below) |
| `kernels/` | VxWorks kernel images for MVME166/167/NI CPU030 (analog era) |
| `daq/ver10`–`ver26` | Old analog GS DAQ production trees (archived) |
| `ana/radware/` | RadWare analysis suite (analog era) |
| `epics/R3.12-LBL.1/` | EPICS R3.12 LBL (full source + EDM, medm, alh, stripTool) |
| `epics/R3.11.2/` | EPICS R3.11.2 (older) |

---

## VxWorks 68040 Cross-Compiler

**Location on con6:** `/home/gam/prod/vxWorks/vxw_5.1.1/solaris.68k/bin/`

| Binary | Role |
|--------|------|
| `cc68k` | **68040 C compiler** (main compiler binary) |
| `as68k` | Assembler |
| `ld68k` | Linker |
| `ar68k` | Archiver |
| `nm68k` | Symbol lister |
| `objdump68k` | Object dumper |
| `strip68k` | Strip symbols |

VxWorks headers: `/home/gam/prod/vxWorks/vxw_5.1.1/h/` (needed for cross-compilation)

⚠️ These are **Solaris/SPARC native binaries** — they do NOT run on Linux x86_64 or ARM64.

**Makefile flags used (from CVS):**
```makefile
COMPILE = $(APPL)/vw/`arch`.68k/bin/cc68k
CFLAGS  = -B$(APPL)/vw/`arch`.68k/lib/gcc- -c -O -W -DV5_vxWorks -DCPU=MC68040 -DvxWorks $(INCLUDES)
```

✅ verified 2026-04-18 — `Con6_Inventory.md`; Con6_Retirement_Plan.md:Makefile section

---

## Disk Usage on con6

| Mount | Size | Used | Notes |
|-------|------|------|-------|
| `/` (sda0) | 27 GB | 34% | OS + main software |
| `/export/home/root10` (sda2) | 131 GB | **93%** | Nearly full — large data/software store |
| `/export/home/root3` (sda7) | 4.4 GB | 36% | |
| `/opt` (NFS from dgs1) | 1.9 TB | 5% | NFS mounted from dgs1 |

✅ verified 2026-04-18 — `Con6_Inventory.md:Disk Usage section`

---

## Archiving Priority (before con6 retirement)

| Priority | Item | Action |
|----------|------|--------|
| 🔴 CRITICAL | `repository/lnfill/` | CVS checkout + tar to spark-ca9f |
| 🔴 CRITICAL | `repository/gam/` | CVS checkout + tar (sysAnalysisBd source) |
| 🔴 CRITICAL | `prod/vxWorks/vxw_5.1.1/h/` | Copy VxWorks headers (needed for recompile) |
| 🟡 HIGH | `prod/lnfill/` | Full tree + dbase/ |
| 🟡 HIGH | `prod/kernels/` | All VxWorks kernel images |
| ⚪ LOW | `repository/daq/` | Analog DAQ source — pre-DGS, no longer operational |
| ⚪ LOW | `prod/daq/ver26/` | Analog DAQ binaries — historic archive only |
| ⚪ LOW | `prod/ana/radware/` | Analog era analysis — available elsewhere |

---

## ln2con — NFS Boot Host for LN2 IOC

**Host:** `192.168.203.148`, user `dgs`, password `gam$hippie`  
**SSH flags required:** `-o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedKeyTypes=+ssh-rsa`  
**OS:** Fedora 12  

ln2con is the **NFS boot host** for the lnfill VxWorks IOC (`192.168.203.121`, Motorola MVME167) ✅ verified 2026-04-21 — `ln2con/Plan.md:L10,L172` (inet on ethernet: 192.168.203.121; MVME167):
- Exports `/export/home/lncon` via NFS to the VxWorks IOC ✅ verified 2026-04-21 — `ln2con/Plan.md:L134` (`/export/home/lncon  192.168.203.121(rw,no_root_squash,sync)`)
- Hosts the active IOC boot tree at `/home/lncon/prod/lnfill/Rev6-01-04/`
- Hosts fill log archive and log processing tools

**⚠️ If ln2con dies, the lnfill IOC cannot reboot.** VxWorks needs the NFS mount to load its software. Detectors will not be auto-filled until ln2con is restored.

### Archived Files in DGS_tools_pack

Local archive at `DGS_tools_pack/ln2con/` (key files only):

| File | Source on ln2con | Description |
|------|-----------------|-------------|
| `startup.cmd` | `Rev6-01-04/startup.cmd` | Active VxWorks IOC startup script ✅ verified 2026-04-21 — `ln2con/startup.cmd:L18-30` (loads from `targetmv167/` dirs, confirming MVME167 CPU) |
| `startup.prod` | `Rev6-01-04/startup.prod` | Production copy of startup |
| `rtdb/gamln.db` | `Rev6-01-04/rtdb/gamln.db` | EPICS records (23,078 lines, 1357 records) ✅ verified 2026-04-21 — `wc -l` = 23,078; `grep -c 'record('` = 1,357 |
| `rtdb/tempmon.db` | `Rev6-01-04/rtdb/tempmon.db` | Temperature monitoring EPICS records |
| `local/vx_mounts` | `local/vx_mounts` | VxWorks NFS mount script |
| `local/resource.def.R312` | `local/resource.def.R312` | EPICS IOC resource definitions (CA timeouts, log server) |
| `lnlogrotate.conf` | `log/lnlogrotate.conf` | Logrotate config for fill logs |
| `ln.inits.snapshot.20260418` | `log/ln.inits` | Hose→detID mapping snapshot (2026-04-18) |
| `tools/lnlogs` | `tools/lnlogs` | Log browsing script |
| `tools/process_logs` | `tools/process_logs` | Log processing script |

**Not archived (need root access to copy):**
- `vx68040/lnfiller.vx` — compiled VxWorks fill application (68040 binary) ← **CRITICAL if rebuild needed**
- `targetmv167/iocCore`, `drvSup`, `recSup`, `devSup`, `seq` — EPICS core object files
- `lib/sysAnalysisBd.vx`, `lib/sysAnLEDsupport.vx` — hardware support libraries

---

## con6 Retirement Plan Summary

**Goal:** Retire con6 by migrating the lnfill build environment to a modern Linux x86_64 machine (Ryan's workstation, `192.168.203.75`).

**Key blocker:** `cc68k` is a Solaris/SPARC binary — cannot run on Linux. Need a Linux-native 68040 cross-compiler.

**Migration options (in priority order):**

| Option | Description | Feasibility |
|--------|-------------|-------------|
| A | **Same cross-compiler as DAQ VxWorks build** ← *check `ioc/` Makefile* | Most likely — already proven on Linux |
| B | `gcc-m68k-linux-gnu` (Ubuntu package) | Available but targets Linux ABI, not VxWorks — may need adjustments |
| C | Wind River commercial toolchain | Most accurate but least accessible |
| D | Build GCC m68k-vxworks from source | Possible, complex |

**Immediate action:** Archive the CVS source from con6 before it fails. See `Con6_Retirement_Plan.md` for exact commands.

---

## Cross-References

- `lnfill.md` — LN2 fill system: valves, fill types, cron jobs, Discord alerts, ops procedures
- `lnfill_ioc.md` — Deep internals: ln2con IOC boot tree, hose→detID mapping, InfluxDB data flow, DetMan.py state machine
- `hardware_architecture.md` — System overview including ln2con/lnfill IOC in the network map
- `DGS_tools_pack/ln2con/Plan.md` — Full ln2con recovery plan with archived file list
- `DGS_tools_pack/ln2con/Con6_Inventory.md` — Complete con6 directory survey
- `DGS_tools_pack/ln2con/Con6_Retirement_Plan.md` — Step-by-step migration plan
