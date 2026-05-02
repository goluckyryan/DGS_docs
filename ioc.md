# ioc — EPICS IOC Configuration & Firmware Repository

Stability: C2 - Active / semi-stable

## Table of Contents

1. [What It Is](#what-it-is)
   - [Historical Origin (Carlware)](#historical-origin-carlware)
2. [Folder Layout](#folder-layout)
3. [EPICS CA Port Map (from cdCommands)](#epics-ca-port-map-from-cdcommands)
4. [Firmware Files (firmware/)](#firmware-files-firmware)
   - [Board Type Encoding — code_revision\[11:8\]](#board-type-encoding--code_revision118)
5. [Boot Script Details](#boot-script-details)
   - [vme66.cmd (DuoGe — CRATE=66)](#vme66cmd-duoge--crate66)
6. [Boot Scripts](#boot-scripts)
   - [Production GS Crate Slot Map (VME01–VME12)](#production-gs-crate-slot-map-vme01vme12)
7. [iocArray — Production GS Crate Boot Scripts](#iocarray--production-gs-crate-boot-scripts)
   - [Standard DIG Crate Boot Sequence (vme01.cmd–vme11.cmd)](#standard-dig-crate-boot-sequence-vme01cmdvme11cmd)
   - [Trigger Crate Boot Sequence (vme32.cmd)](#trigger-crate-boot-sequence-vme32cmd)
   - [RunProtect.asf — Access Security File](#runprotectasf--access-security-file)
   - [Global.substitutions](#globalsubstitutions)
   - [reload* Scripts — Per-Board FPGA Reset via VME Memory Write](#reload-scripts--per-board-fpga-reset-via-vme-memory-write)
   - [Test.rsub / Test.substitutions](#testrsub--testsubstitutions)
8. [FTP Server Setup (IOC Host)](#ftp-server-setup-ioc-host)
9. [Current Firmware Versions](#current-firmware-versions)
10. [IOC Setup (Example: DuoGe / tangerine)](#ioc-setup-example-duoge--tangerine)
    - [VxWorks Boot Parameters (set on MVME5500 boot prompt)](#vxworks-boot-parameters-set-on-mvme5500-boot-prompt)
    - [Prerequisites on Host](#prerequisites-on-host)
11. [Key Concepts](#key-concepts)
    - [munch file (gretDet.munch)](#munch-file-gretdetmunch)
    - [EPICS PVs](#epics-pvs)
    - [DB Templates (ioc/db/)](#db-templates-iocdb)
    - [MTrigUser.template — Key PV Groups](#mtrigusertemplatekey-pv-groups)
    - [Boot script flow](#boot-script-flow)
12. [findAllPV.py — PV Discovery Tool](#findallpvpy--pv-discovery-tool)
13. [Connections to Other Subsystems](#connections-to-other-subsystems)
14. [IOC Connections: Ethernet vs Terminal Server](#ioc-connections-ethernet-vs-terminal-server)
    - [1. Ethernet (Data + EPICS)](#1-ethernet-data--epics)
    - [2. Terminal Server (Console/Shell access)](#2-terminal-server-consoleshell-access)
    - [Terminal Server Assignments](#terminal-server-assignments)
15. [Operational Notes](#operational-notes)
16. [Cross-References](#cross-references)
17. [Full Build & Deployment Procedure (Gammasphere System)](#full-build--deployment-procedure-gammasphere-system)
    - [Shared File System — /global Mapping](#shared-file-system--global-mapping)
    - [Boot Host for Gammasphere VME IOCs](#boot-host-for-gammasphere-vme-iocs)
    - [Build Sequence (VME IOC Binary)](#build-sequence-vme-ioc-binary)
    - [Deploying Updated EPICS Databases](#deploying-updated-epics-databases)
    - [Deploying Updated Soft IOC Database](#deploying-updated-soft-ioc-database)
    - [Spreadsheet → Boot Full Sequence](#spreadsheet--boot-full-sequence)

## What It Is

The **IOC (Input/Output Controller) repository** — contains all files needed to bring up a DGS VME crate:
- EPICS database files (PV definitions)
- Boot scripts (VxWorks startup commands)
- Firmware binaries (DIG, RTRG, MTRG)
- The VxWorks image and munch binary

Uses **Git LFS** for large binaries (`.munch`, `.bin`, `vxWorks`).

> **Current commit target:** TAC2 + Trigger Hold-Off configuration

### Historical Origin (Carlware)

The DGS IOC software originated from **Carlware** — the LBL software written for the GRETINA detector system. Tim Madden (APS-XSD) adapted it for DGS VME crates. All crates run the same VxWorks image; only the per-crate `st.cmd` files differ.

Old template names (historical, pre-Git migration, from `DGS_SVN/dgs/Documentation/Formal/Software/HowCarlware-TimwareWorks.docx`):

| Old Name | Current Equivalent | Notes |
|---|---|---|
| `dgsDigRegisters.template` | `MDigRegisters.template` / `SDigRegisters.template` | Engineering VME registers per dig board |
| `dgsDigUser.template` | `MDigUser.template` / `SDigUser.template` | User PVs per dig board |
| `daqSegment.template` | `daqSegment2.template` | Per-board IOC control PVs: `CS_Ena` (software enable) + `FifoNum` (FIFO select, 16 options: MONFIFOs 1–8, MAIN DATA FIFO, CHAN A–H FIFOs) ✅ verified 2026-04-15 — `ioc/db/daqSegment2.template` (2 records: bo + mbbo) |
| `dgsMTrigRegisters.template` | `MTrigRegisters.template` | MTRG engineering VME registers |
| `dgsRTrigRegisters.template` | `RTrigRegisters.template` | RTRG engineering VME registers |
| `dgsMTrigUser.template` | `MTrigUser.template` | MTRG user controls |
| `dgsRTrigUser.template` | `RTrigUser.template` | RTRG user controls per router |
| `gretVME.template` | — | Per-card VME board PVs (merged into board templates) |
| `link.template` | _(in trigger templates)_ | SERDES link PVs for routers/triggers |
| `asynDebug.template` | removed in Git version | Backdoor VME access via PVs (FPGA flash, register R/W) |
| `dgsGlobals_DGS_GLBL.db` | `dgsGlobals_DGS_VMExx.db` | Global PV fanout DB (per crate in current version) |

> Note: Global PV fanout — a single global DB on the trigger crate fans settings to all digitizer crates + channels via channel fanout records. Each `dgsGlobals_DGS_VMExx.db` instantiates per-board and per-channel records for that crate.

---

## Folder Layout

| Folder | Contents |
|--------|----------|
| `bin/vxWorks-ppc604_long/` | IOC munch file (`gretDet.munch` — loaded on MVME5500) ✅ verified 2026-04-06 — `ioc/bin/vxWorks-ppc604_long/gretDet.munch` |
| `boot/` | IOC boot scripts (VxWorks `.cmd` files) |
| `db/` | EPICS database files (PV record definitions) |
| `dbd/` | EPICS database definition files |
| `firmware/` | DIG, RTRG, MTRG firmware `.bin` files |
| `mvme5500/` | VxWorks boot image |

---

## EPICS CA Port Map (from cdCommands)

| System | CA Server Port | CA Repeater Port | Notes |
|--------|---------------|------------------|-------|
| DGS | 5064 | 5065 | ✅ verified 2026-04-27 — `ANLDAQ/EPICS_env.sh:L45-46` + `ioc/boot/cdCommands:L11-12` |
| DFMA | 5068 | 5069 | ✅ verified 2026-04-27 — `ANLDAQ/EPICS_env.sh:L5` (comment) |
| Xarray | 5072 | 5073 | ✅ verified 2026-04-27 — `ANLDAQ/EPICS_env.sh:L23-24` |
| SlopeBox | 5074 | 5075 | ✅ verified 2026-04-27 — `ANLDAQ/EPICS_env.sh:L36-37` + `ioc/boot/vme99.cmd:L21` |
| DUB | 5078 | 5079 | ✅ verified 2026-04-27 — `ANLDAQ/EPICS_env.sh:L8` (comment) |
| DuoGe | 5080 | 5081 | ✅ verified 2026-04-27 — `ANLDAQ/EPICS_env.sh:L16-17` (DUO=5080/5081 conditional) |

NTP server: `192.168.203.56` ✅ verified 2026-04-06 — `ioc/boot/cdCommands:L23` | Timezone: CDT (UTC-6, `EPICS_TS_MIN_WEST=360`) ✅ verified 2026-04-06 — `ioc/boot/vme99.cmd:L52`

**EPICS CA tuning parameters (set in `cdCommands` and `vme99.cmd`):**

| Parameter | Value | EPICS Default | Purpose |
|-----------|-------|---------------|--------|
| `EPICS_CA_CONN_TMO` | **40 s** | 30 s | Connection timeout — gives IOCs extra 10 s to reconnect after a network hiccup before CA declares the link broken |
| `EPICS_CA_BEACON_PERIOD` | **2 s** | 15 s | How often the IOC broadcasts its heartbeat beacon — shorter period means faster re-discovery by clients after an IOC restart |

✅ verified 2026-04-26 — `ioc/boot/cdCommands:L17-18`; `ioc/boot/vme99.cmd:L19-20`

---

## Firmware Files (firmware/)

| File | Board | Description |
|------|-------|-------------|
| `BUS_LEFT.bin` | DIG Main FPGA | **Master digitizer** firmware (MDIG1, slot 3, board 0) — clock sourced from SERDES link; drives the inter-DIG front bus clock to slave ✅ verified 2026-04-08 — `uploadFW.cmd:L17` (board 0=MDIG1) + `Digitizer.vhd:L354-355` ("SERDES is external clock source in master digitizer") |
| `BUS_RIGHT.bin` | DIG Main FPGA | **Slave digitizer** firmware (MDIG2, slot 4, board 1) — clock sourced from front bus (driven by master DIG); sends throttle/lock status back to master via SDATA ✅ verified 2026-04-08 — `uploadFW.cmd:L22` (board 1=MDIG2) + `Digitizer.vhd:L968-970` (slave SDATA signals) |
| `trigger_top.bin` | MTRG Main FPGA | Master trigger ✅ verified 2026-04-20 — `ioc/firmware/uploadFW.cmd:L28` (`ProgramFlash(5, 0, "trigger_top.bin")` — board 5 = MTRG) |
| `V4747_mod_router_top.bin` | RTRG Main FPGA | Router ✅ verified 2026-04-20 — `ioc/firmware/uploadFW.cmd:L23` uses `router_top.bin`; git repo stores it as `V4747_mod_router_top.bin` — the `V4747` prefix comes from the Virtex-4 part number (`xc4vlx80`). On the actual tangerine system, the file is placed in `/global/ioc/FW_Maint/` as `router_top.bin`. |
| `DIG_VME_FPGA_20220729.mcs` | DIG VME FPGA | VME interface (Jul 2022) |
| `MTRG_VME_FPGA_20250711.mcs` | MTRG VME FPGA | VME interface (Jul 2025) |

_✅ All firmware filenames verified 2026-04-06 — `ls ioc/firmware/`_

### Board Type Encoding — `code_revision[11:8]`

The IOC reads `code_revision` from each FPGA and decodes bits \[11:8\] as the **board type**. Exposed via `DAQC$(CRATE)_BoardType0..N` mbbi PVs in `daqCrate.template`. ✅ verified 2026-04-08 — `ioc/db/daqCrate.template:L14-31`

| Type code | Board identity | Format of full `code_revision` word |
|-----------|---------------|--------------------------------------|
| 1 | GRETINA Router Trigger | — |
| 2 | GRETINA Master Trigger | — |
| 3 | LBNL Digitizer | arbitrary placeholder (JTA) |
| 4 | DGS Master Trigger | — |
| 5 | Unknown | — |
| 6 | DGS Router Trigger | — |
| 7 | Unknown | — |
| 8 | MyRIAD | arbitrary placeholder (JTA) |
| 9–11 | Unknown | — |
| 0xC (12) | **ANL Master Digitizer** | low 16 bits = `4XYZ` (4=DIG, X=master/slave, Y=major rev, Z=minor rev) |
| 0xD (13) | **ANL Slave Digitizer** | low 16 bits = `4XYZ` |
| 0xE (14) | Majorana Master Digitizer | low 16 bits = `FXYZ` (F=Majorana, X=master/slave, Y=major, Z=minor) |
| 0xF (15) | Majorana Slave Digitizer | low 16 bits = `FXYZ` |

DGS production boards are type **0xC** (master DIG) and **0xD** (slave DIG). Types 1/2/4/6 are trigger boards (GRETINA/DGS MTRG/RTRG). Types 14–15 are Majorana experiment digitizers — not used in DGS/Gammasphere.

**Firmware upload sequence** (`uploadFW.cmd`):
1. `ProgramFlash(board#, 0, "file.bin")` — writes firmware to flash
2. `ConfigureFlash(board#, 0)` — loads flash into FPGA
3. Run `ConfigureFlash` again for all boards to confirm

**Reconfigure all DIG FPGAs at once (DGS convenience PV):**
- `GLBL:DIG:config_main_fpga` — write 1 to reconfigure ALL digitizer FPGAs in DGS simultaneously
- Per-crate: `VME01:DIG1:config_main_fpga` (write 1) — reconfigures a single board
- Trigger crate: requires a **power cycle** (no remote reconfig PV)
- Source: [wiki Updating Firmware in Digitizers and Triggers](https://wiki.anl.gov/gsdaq/Updating_Firmware_in_Digitizers_and_Triggers) ✅ visited 2026-04-18

**Current DGS firmware flash procedure (2024):**
- Documented in `Flash_Maintenance_Instructions_20240222.odt` (linked from wiki)
- URL: https://wiki.anl.gov/wiki_gsdaq/images/d/d7/Flash_Maintenance_Instructions_20240222.odt
- Supersedes the old Java `fpgasender.jar` approach (still works for DFMA/DUB/DXA legacy systems)

**Old Java-based flash procedure (legacy DFMA/DUB/DXA):**
- Log into `dgs` account on `dgs1`; firmware `.bin` in `/Digitizer/MAIN_FPGA/Work11_DGS`
- 4 flavors: `MSTR_digitizer`, `SLAVE_digitizer`, `trigger_top`, `router_top`
- Run: `java -classpath jca-2.3.5.jar:caj-1.1.9.jar:fpgasender.jar plotControl`
- API: `epics.sendFpga(firmware, retfile, crateNum, boardNum, erase, program, verify)`
- Example DGS master dig crate 1 board 0: `epics.sendFpga(digware, fn1, 1, 0, 1,1,0)` (erase+program, no verify)
- Trigger crate (crate 0 board 0): `epics.sendFpga(mastware, fn1, 0, 0, 1,1,1)`
- Routers (crate 0 boards 1–3): `epics.sendFpga(routware, fn1, 0, 1, 1,1,1)` etc.
- ⚠️ Wrong bin filename = "yellow fever" (board hangs); double-check filenames before flashing
- `asynRecords.txt` defines active crates — edit if PVs are missing during setup
- Source: wiki + `DGS_SVN/dgs/how_to_fw.txt`

---

## Boot Script Details

### vme66.cmd (DuoGe — CRATE=66)

Crate layout (vme66 = the crate with MTRG, used as reference):
```
Slot 1: IOC (MVME5500)
Slot 3: MDIG1 (Board #0)
Slot 4: MDIG2 (Board #1)
Slot 6: RTR1  (Board #4)
Slot 7: MTRG  (Board #5)
```
✅ verified 2026-04-07 — `ioc/boot/vme66.cmd:L133-140`: `asynDigitizerConfig("MDIG1",0,3)`, `asynDigitizerConfig("MDIG2",1,4)`, `asynTrigRouterConfig1("RTR1",4,6)`, `asynTrigMasterConfig1("MTRG",5,7)`.

Note: `vme66.cmd:L132` has a stale comment saying "Slot #2" but the actual config call correctly uses slot 3 — comment error in source, not a firmware issue.

Boot sequence:
1. Load `cdCommands` + `nfsCommands` (paths + NFS auth)
2. `ld < gretDet.munch` (loads IOC binary)
3. `dbLoadDatabase("dbd/gretDet.dbd")` (register PV types)
4. `dbLoadRecords(...)` for each board: Register PVs, User PVs, VME FPGA PVs
5. `InitializeDaqBoardStructure()`
6. `asynDigitizerConfig("MDIG1",0,3)` / `asynDigitizerConfig("MDIG2",1,4)`
7. `asynTrigRouterConfig1("RTR1",4,6)` / `asynTrigMasterConfig1("MTRG",5,7)`
8. `iocInit()` — starts EPICS IOC
9. `setupFIFOReader()`
10. `dbpf` — set user_package_data per board (MDIG1=170, MDIG2=171, MTRG=172)
11. `seq &inLoop` / `seq &outLoop` / `seq &MiniSender` — start DAQ state machines

**`inLoop` B-parameter syntax:** `seq &inLoop,"CRATE=NN,B0=MDIG1,B1=MDIG2,...,B5=MTRG,B6=X"` ✅ verified 2026-04-07 — `ioc/boot/vme66.cmd:L180-190`

**`inLoop` state machine** (`inLoop.st`, 969 lines, SNL): ✅ verified 2026-04-12 — source read

| State | Purpose |
|-------|---------|
| `INIT` | Priority=190; sets `InloopIsRunning=0`; calls `SetupBoardAddresses()`; waits for `AcqRun` PV; on START → `INITIAL_FIFO_CLEAR`; times out after 10s if no start |
| `INITIAL_FIFO_CLEAR` | Per-board FIFO clear: MTRG → `ClearTrigFIFO()`; DIGs → `ClearDigMstrLogicEnable()` + `ClearDigFIFO()` + `CalcDigMaxEventsPerRead()`; RTRGs require no clear. If 0 boards enabled → `IDLE_ERROR` |
| `IDLE_ERROR` | No boards enabled — waits for `AcqRun` to go away, then returns to `INIT` |
| `ENABLE_DIGITIZERS` | Enables channel select (`CS_Ena`) on all boards; arms the digitizer logic; → `SCAN_FOR_DATA` |
| `SCAN_FOR_DATA` | Main readout loop: iterates boards round-robin; calls `transferDigFifoData()` / `DigitizerTypeFHeader()` for each board with data; pushes raw events to `rawEvtQ`; monitors FIFO fullness flags; → `SCAN_DELAY` between passes |
| `SCAN_DELAY` | Throttle: waits `ScanDelay` seconds (default 0.01s, dynamically adjusted) before next scan pass |
| `DISABLE_COLLECTION` | On stop signal: disables `CS_Ena` on all boards; → `DRAIN_REMAINING_DATA` |
| `DRAIN_REMAINING_DATA` | Drains remaining FIFO data after run stop; sends end-of-run marker; → `INIT` |
- B0–B6 are board name labels for inLoop board iteration (or `X` = empty position)
- `inLoop` uses these to form PV names like `MDIG1_CS_Ena` for readout control (e.g. `B3=X` → PV `X_CS_Ena`)
- BN is **not** a direct physical slot index; it is an ordered list of boards that inLoop iterates. The physical slot for each board is set separately by `asynDigitizerConfig(portName, boardNum, slot)`. BN provides only the board name string used to construct PV names.
  - ⚠️ corrected 2026-04-19 — previous claim ("BN = VME slot 0-indexed from slot 1") was wrong. vme66.cmd: MDIG1 is B0 but physically in slot 3; vme99.cmd: MDIG1 is B0 and physically in slot 2. No consistent slot formula. ✅ verified 2026-04-19 — `ioc/boot/vme66.cmd:L133` (slot 3) vs `L190` (B0=MDIG1); `ioc/boot/vme99.cmd:L147` (slot 2) vs `L200` (B0=MDIG1)
- Example: `B0=MDIG1,B1=MDIG2,B2=X,B3=X,B4=X,B5=MTRG,B6=X` → inLoop iterates 7 board positions; MDIG1 and MDIG2 are readable, MTRG is readable, B2–B4,B6 are dummies (X)

**User package data formula (VME01–12 production crates):** `[(crate# - 1) × 4] + 101 + board#` ✅ verified 2026-04-07 — `ioc/boot/vme66.cmd:L162-164` (comment)
- Board# restricted to {0,1,2,3} for digitizers → VME01: 101–104, VME02: 105–108, … VME12: 145–148
- Master trigger always = 150 ✅ verified 2026-04-07 — `vme66.cmd:L168` (comment)
- Routers: RTR1=151, RTR2=152, etc. — but as of 2023-03-31, Routers have no register to store package data ✅ verified 2026-04-07 — `vme66.cmd:L170`
- **Exception — VME66 (DuoGe):** uses manually assigned values: MDIG1=170, MDIG2=171, MTRG=172 ✅ verified 2026-04-08 — `vme66.cmd:L173-175`
- **Exception — VME99 (test stand):** uses manually assigned values: MDIG1=160, MDIG2=161, MTRG=162 ✅ verified 2026-04-08 — `vme99.cmd:L184-186`
- The formula only applies to the 12 production DGS VME crates; special crates use fixed assigned values outside the 101–148 range

**DB naming convention:**
- Register-level: `VME<CRATE>:<BOARD>:<register>` (e.g. `VME66:MDIG1:reg_led_threshold`)
- BOARD = portName = first arg of asynXxxConfig()

---

## Boot Scripts

Located in `boot/`:
- `vme01.cmd`–`vme12.cmd` — Production GS crates; see table below
- `vme66.cmd` — DuoGe crate (CRATE=66); uses `cdCommands` ⚠️ corrected 2026-04-27: the `cdCommands` in the local repo (ioc/boot/cdCommands:L11-12) sets **DGS ports 5064/5065**, not 5080/5081. DuoGe ports 5080/5081 are listed only in a comment and in `ANLDAQ/EPICS_env.sh:L16-17`. The actual DuoGe boot host (DGS1) likely has a separate cdCommands that sets 5080/5081; the repo cdCommands reflects DGS production. ✅ `vme66.cmd:L14` (< cdCommands) confirmed — `ANLDAQ/EPICS_env.sh:L16-17` (DuoGe=5080/5081)
- `vme99.cmd` — GRETINA lab test stand (CRATE=99); uses `cdCommandsLab` (CA port 5074/5075 G-wing) ✅ verified 2026-04-20 — `ioc/boot/vme99.cmd:L18,L21,L27` (G-wing port 5074/5075 comment + putenv + < cdCommandsLab)
- `cdCommands` — paths + EPICS CA env (sets 5064/5065 DGS in repo; DuoGe boot host may have variant with 5080/5081)
- `nfsCommands` — NFS mount: `nfsAuthUnixSet("fs.gam", 6000, 10, 0, 0)` ✅ verified 2026-04-20 — `ioc/boot/nfsCommands:L1`

### Production GS Crate Slot Map (VME01–VME12)

✅ verified 2026-04-24 — all entries extracted directly from `ioc/boot/vme01.cmd`–`vme12.cmd` (slot comments + `asynDigitizerConfig`/`asynTrigRouterConfig1`/`asynTrigMasterConfig1` lines)

| Crate | Slot 1 | Slot 2 | Slot 3 | Slot 4 | Slot 5 | Slot 6 | Slot 7 | DIG count | Trigger |
|-------|--------|--------|--------|--------|--------|--------|--------|-----------|---------|
| VME01 | IOC | MDIG1 | SDIG1 | MDIG2 | SDIG2 | empty | empty | 4 | — |
| VME02 | IOC | MDIG1 | SDIG1 | MDIG2 | SDIG2 | empty | empty | 4 | — |
| VME03 | IOC | MDIG1 | SDIG1 | MDIG2 | SDIG2 | empty | RTR1 | 4 | RTRG RTR1 (board# 4, slot 7) |
| VME04 | IOC | MDIG1 | SDIG1 | MDIG2 | SDIG2 | empty | empty | 4 | — |
| VME05 | IOC | MDIG1 | SDIG1 | MDIG2 | SDIG2 | empty | empty | 4 | — |
| VME06 | IOC | MDIG1 | SDIG1 | empty | empty | empty | RTR2 | 2 | RTRG RTR2 (board# 4, slot 7) |
| VME07 | IOC | MDIG1 | SDIG1¹ | MDIG2 | SDIG2¹ | empty | empty | 4 | — |
| VME08 | IOC | MDIG1 | SDIG1 | MDIG2 | SDIG2 | empty | empty | 4 | — |
| VME09 | IOC | MDIG1 | SDIG1 | MDIG2 | SDIG2 | empty | RTR3 | 4 | RTRG RTR3 (board# 4, slot 7) |
| VME10 | IOC | MDIG1 | SDIG1 | empty | MTRG | Fiber Exp. | empty | 2 | MTRG (board# 4, slot 5) |
| VME11 | IOC | MDIG1 | SDIG1 | MDIG2 | SDIG2 | empty | empty | 4 | — |
| VME12 | IOC | MDIG1 | SDIG1 | MDIG2 | SDIG2 | empty | RTR4 | 4 | RTRG RTR4 (board# 4, slot 7) |

¹ VME07 comment says SDIG1 and SDIG2 are "Removed to reduce load" but they are still initialized in `asynDigitizerConfig` and `inLoop` B-params — likely physically absent but software slots kept for symmetry.

**Summary:** 11 production crates with digitizers; VME10 houses the MTRG; VME03/06/09/12 each house one RTRG. Total DIG boards: 4×10 + 2×1 + 2×1 = 44 DIGs (matches `SYSTEM_DEFINES.sh` in trig_setup scripts). 4 RTRGs total. 1 MTRG.

**User package data:** `[(crate# − 1) × 4] + 101 + board#` where board# ∈ {0,1,2,3} → VME01: 101–104, VME02: 105–108, … VME11: 141–144, VME12: 145–148. MTRG = 150; RTR1–4 = 151–154 (planned, not yet implemented as of 2023-03-31). ✅ verified 2026-04-24 — `vme01.cmd` + `ioc/boot/vme10.cmd` + comment in all `.cmd` files.

**Key differences between vme66 and vme99:**
- vme66: loads `daqCrate.template` + NFS globals commented out; `cdCommands` (DuoGe port) ✅ verified 2026-04-20 — `vme66.cmd:L98` (daqCrate.template); `vme66.cmd:L105` (dgsGlobals commented out)
- vme99: loads `daqCrate.template` + `dgsGlobals_DGS_VME99.db`; `cdCommandsLab` (G-wing/test port) ✅ verified 2026-04-20 — `vme99.cmd:L123,L130`
- vme99: two MDIG boards both use `MDigRegisters/User` (master-type DB); vme66: MDIG1=master, MDIG2=slave (`SDigRegisters/User`) ✅ verified 2026-04-20 — `vme66.cmd:L59-60` (MDigRegisters+SDigRegisters); `vme99.cmd:L85-86` (both MDigRegisters)
- Both: `asynDebug.template` line present but commented out ✅ verified 2026-04-20 — `vme66.cmd:L84` + `vme99.cmd:L109` (both `#dbLoadRecords("db/asynDebug.template"...)`)
- Regular VME01–12 (Gammasphere) boot scripts live on NFS at `/global/ioc/boot/` — not in this git repo
- **PV dump on startup:** Both `vme66.cmd` and `vme99.cmd` now end with `dbl > "vme<NN>_db.txt"` — dumps the full PV list to a text file at IOC startup ✅ verified 2026-04-17 — `ioc` commit `4eb1eb0`
- **`bootFiles.txt`** currently points to `boot/vme66.cmd` (changed from `vme99.cmd`) ✅ verified 2026-04-17 — `ioc` commit `4eb1eb0`

---

## iocArray — Production GS Crate Boot Scripts

_Source: `vxworks/dgsIoc/iocBoot/iocArray/` (code-read 2026-04-27)_

The `iocArray/` directory contains the production boot scripts used by all 12 standard DGS VME crates plus the dedicated trigger crate (VME32). Unlike the `ioc/boot/` files (used for DuoGe/test stand), these are the scripts loaded at NFS path `/dk/fs2/dgs/global_sandbox/devel/gretTop/9-22/dgsIoc/iocBoot/iocArray/` on the GS production hosts.

### Standard DIG Crate Boot Sequence (vme01.cmd–vme11.cmd)

All 11 DIG crates follow the same pattern:

1. `cd` to iocArray NFS path
2. `< cdCommands` + `< ../nfsCommands` (EPICS CA env + NFS auth)
3. `ld < gretDet2018.munch` — load IOC binary (note: uses `gretDet2018.munch`, not `gretDet.munch`)
4. `dbLoadDatabase("dbd/gretDet.dbd", ...)` + `gretDet_registerRecordDeviceDriver`
5. Load 4 boards' register + user templates: `MDigRegisters.template`, `SDigRegisters.template`, `MDigUser.template`, `SDigUser.template` (P=VMExx:, R=MDIGn:)
6. Load `gretVME.template` for each of 4 digitizer slots (DB=xx_1..4, DC=0..3)
7. Load `daqSegment2.template` for each slot (DN=xx_1..4, DC=0..3)
8. Load crate-level PVs: `daqCrate.template` (DN=crate#) + `onMon.template` (DN=crate#)
9. Load `asynDebug.template` (P=VMExx:, R=DBG:)
10. Load `db/Globals_VMExx.db` — per-crate global PVs
11. AutoSave/Restore: `set_savefile_path(... "vmexx")` + `set_pass1_restoreFile("vmexx.sav")`
12. `asynDigitizerConfig("MDIG1",0,3)` + `asynDigitizerConfig("SDIG1",1,4)` + `asynDigitizerConfig("MDIG2",2,5)` + `asynDigitizerConfig("SDIG2",3,6)` — all slots 3–6
13. `asynDebugConfig("DBG",0)`
14. `asSetFilename("../../db/RunProtect.asf")` — load Access Security rules
15. `iocInit()`
16. `create_monitor_set("vmexx.req", 30, "")` — autosave monitor (30s interval)
17. `setupFIFOReader()`
18. `seq &inLoop` for each of 4 boards (with `PVAcqEna`, `PVMLE`, `PVRun` params)
19. `dbpf "VMExx:MDIGn:user_package_data", "NNN"` — tag each board with global board ID
20. `seq &TrigCon, "CN=xx"` + `seq &BuildSend, "CN=xx,priority=5"`

✅ verified 2026-04-27 — `iocArray/vme01.cmd` (full read) + `iocArray/vme10.cmd` (cross-check)

**Key difference from vme66/vme99:** Uses `gretDet2018.munch` (not `gretDet.munch`). All 4 slots are always 3/4/5/6. No commented-out asynDebug (it's always loaded).

### Trigger Crate Boot Sequence (vme32.cmd)

VME32 is the dedicated trigger crate (MTRG + 3 RTRGs):

- Loads `MTrigRegisters.template` + `MTrigUser.template` for MTRG
- Loads `RTrigRegisters.template` + `RTrigUser.template` for RTR1/RTR2/RTR3
- Loads `gretVME.template` for 4 slots (for legacy compatibility)
- **Does NOT** load `daqSegment2.template` (no digitizers)
- Loads `daqCrate.template` (DN=32) + `onMon.template` (DN=32)
- Loads `Globals_GLBL.db` (not `Globals_VME32.db`)
- `asynTrigMasterConfig1("MTRG",0,3)` + `asynTrigRouterConfig1("RTR1",1,4)` + `RTR2(2,5)` + `RTR3(3,6)`
- **No** `setupFIFOReader()` (commented out)
- **No** `seq &inLoop` or `seq &BuildSend` (all commented out with placeholder notes)
- `asSetFilename("../../db/RunProtect.asf")` — same RunProtect as DIG crates

✅ verified 2026-04-27 — `iocArray/vme32.cmd` (full read)

### RunProtect.asf — Access Security File

_Source: `dgsIoc/tcDetApp/Db/RunProtect.asf` (11 lines)_

Defines two Access Security Groups (ASGs):

| ASG | INPA PV | Read | Write condition |
|-----|---------|------|------------------|
| `DEFAULT` | — | always | always |
| `RUNPROTECT` | `Online_CS_StartStop` | always | only when `A=0` (i.e. run is **stopped**) |

`RUNPROTECT` prevents writes to protected PVs while a run is active (`Online_CS_StartStop != 0`). Records that use this ASG can only be written to when the DAQ is not running. ✅ verified 2026-04-27 — `RunProtect.asf` (full read)

### Global.substitutions

_Source: `dgsIoc/tcDetApp/Db/Global.substitutions` (9 lines)_

Loads a single database:
- `gretGlobal.db` with `{P=DAQG}` — loads global DAQ control PVs (fanout records) for the entire system
- Commented-out: `daqGlobal.template` (now in nodeIoc)
- Comment in file: "Master DAQ control records (one copy necessary for entire system, all crates)" — this substitutions file is loaded exactly once in the system (on the control node, not every crate)

✅ verified 2026-04-27 — `Global.substitutions` (full read)

### reload* Scripts — Per-Board FPGA Reset via VME Memory Write

_Source: `iocArray/reload0`–`reload4`, `reloadMainFPGA` (2–16 lines each)_

Each `reloadN` script pulses a specific VME address (SRAM-mapped FPGA control register) to toggle a reload bit:
- Toggle sequence: write `0` → write `1` at the target address
- Written using VxWorks `m` (memory modify) command with count=4 (word access)

| Script | VME Address | Board |
|--------|-------------|-------|
| `reload0` | `0xe8100900` | Board 0 (slot 1 — IOC?) |
| `reload1` | `0xe8200900` | Board 1 (slot 2) |
| `reload2` | `0xe8300900` | Board 2 (slot 3) |
| `reload3` | `0xe8400900` | Board 3 (slot 4) |
| `reload4` | `0xe8500900` | Board 4 (slot 5) |
| `reloadMainFPGA` | All 4: 0xe8200900–0xe8500900 | Reload all 4 DIG FPGAs at once |

`fix`: chains `< reload1 < reload2 < reload3 < reload4` then calls `reboot` — full crate FPGA reload + reboot recovery script.

✅ verified 2026-04-27 — `devGVME.c:L128` (`base = slot << 20`): the N nibble is the **VME slot number** directly. The VME A32 bus address for a board in slot N is `0xN00000`; the MVME5500 maps A32 VME space starting at `0xe8000000` in local memory, so local address = `0xe8000000 + (slot << 20)` = `0xe8N00000`. Register `0x0900` is `fpga_ctrl_reg` (`devGVME.c:L371`, `VME_FPGA_ANL/Source/register_block.vhd:L330-331`). Writing bit 0 low→high triggers `cnfg_reset_controller` FSM restart, which reloads the Main FPGA from flash.

✅ verified 2026-04-27 — `reload0`–`reload4`, `reloadMainFPGA`, `fix` (all read)

### Test.rsub / Test.substitutions

_Source: `dgsIoc/tcDetApp/Db/Test.rsub` (95 lines), `Test.substitutions` (141 lines)_

Autosave request-file templates (`.rsub` = substitution input for `msi`/`makeRequestFile`):
- `Test.rsub`: defines which DB files contribute autosave PVs (gretCrystal.req, gretBoard.req, daqBoard.req, and optional gretVME.req/daqSegment2.req — both commented out)
- `Test.substitutions`: the substitution values that instantiate `Test.rsub` for a specific crate (board DB numbers, detector numbers, etc.)
- These are used as build-time inputs to `msi` (macro substitution tool) to generate `.req` files: `tcDetApp/Db/Makefile` rule `%.req : ../%.rsub` runs `msi -S$< -o$@` ✅ verified 2026-04-27 — `tcDetApp/Db/Makefile:L49-50` (`%.req : ../%.rsub` + `msi -S$< -o$@`)
- Each `vmeXX.cmd` boot script calls `create_monitor_set("vmeXX.req",30,"")` to load the corresponding `.req` file into the autosave monitor at 30-second intervals ✅ verified 2026-04-27 — `iocBoot/iocArray/vme01.cmd:L107`
- **Note:** `vme*.cmd` scripts contain the comment `# req files are now generated from python code` — this refers to `ANLDAQ/ioc/findAllPV.py` (which processes `dbLoadRecords()` lines from startup scripts → `All_PV.json`). The `.req` file writer from JSON is **not present** in this repo; the Python toolchain likely lives on the production host only. `Test.rsub` is the legacy Makefile-based path; the current production path uses `findAllPV.py` output. ✅ verified 2026-04-27 — `vme01.cmd:L106` (comment); `ANLDAQ/ioc/findAllPV.py` (full text; no `.req` file writer present)

---

## FTP Server Setup (IOC Host)

The IOC boot host (e.g., `tangerine`) needs an FTP server so VxWorks IOCs can fetch boot files. Recommended: **vsftpd** (very-secure FTP server). For Ubuntu 24:

```sh
sudo apt install vsftpd
# Edit /etc/vsftpd.conf: set write_enable=YES
sudo systemctl restart vsftpd.service
sudo systemctl enable vsftpd.service
```

FTP credentials are stored separately (see `boot/` README — password not displayed in README.md). ✅ verified 2026-04-17 — `ioc` commit `155a3b6` (README.md update)

---

## Current Firmware Versions

| Board | VME ID | Date | Revision | Comment |
|-------|--------|------|----------|---------|
| MTRG | 0xf13 | 0x1022 | 0x04A8 | TAC2 + Trigger Hold-Off |
| RTRG | 0xf13 | 0x0414 | 0x260E | Old but working |
| MDIG | 0xf13 | 20250704 | 0x4CD8 | — |
| SDIG | 0xf13 | 20250704 | 0x4CD8 | — |

(MDIG = Master Digitizer, SDIG = Slave Digitizer)

---

## IOC Setup (Example: DuoGe / tangerine)

Host: **tangerine** (192.168.203.78), Rocky Linux 8

### VxWorks Boot Parameters (set on MVME5500 boot prompt)

```
boot device          : gei
unit number          : 0
processor number     : 0
host name            : tangerine
file name            : /global/ioc/mvme5500/vxWorks
inet on ethernet (e) : 192.168.203.81:ffffff00
host inet (h)        : 192.168.203.78
user (u)             : dgs
ftp password (pw)    : <password>
flags (f)            : 0x0
target name (tn)     : vme66
startup script (s)   : /global/ioc/boot/vme66.cmd
```

### Prerequisites on Host

- FTP server must be enabled on the host (serves VxWorks image and IOC files)
- Files served from `/global/ioc/` on the host
- **Ubuntu 24 FTP setup (vsftpd):** ✅ verified 2026-04-08 — `ioc/README.md` (commit 155a3b6)
  ```sh
  sudo apt install vsftpd
  # Edit /etc/vsftpd.conf: set write_enable=YES
  sudo systemctl restart vsftpd.service
  sudo systemctl enable vsftpd.service
  ```

---

## Key Concepts

### munch file (`gretDet.munch`)
The single binary loaded on each MVME5500 — output of the `vxworks/` build pipeline. Contains the full IOC application: EPICS runtime + DGS drivers + state machines. See `vxworks.md` for how it's built.

### EPICS PVs
Database files in `db/` define all Process Variables exposed by the IOC. These PVs are how the ANLDAQ GUI (and any `caget`/`caput` tool) reads and writes board registers.

### DB Templates (ioc/db/)

All templates use `dbLoadRecords("template", "MACRO=val,...")` in the boot scripts.

| Template | Board | Purpose |
|----------|-------|---------|
| `MDigRegisters.template` | Master Digitizer | Low-level register R/W PVs (`asynUInt32Digital`) |
| `MDigUser.template` | Master Digitizer | User-friendly PVs (mbbo/mbbi, bi/bo, calc) |
| `SDigRegisters.template` | Slave Digitizer | Same as MDigRegisters for slave boards |
| `SDigUser.template` | Slave Digitizer | Same as MDigUser for slave boards |
| `MTrigRegisters.template` | MTRG | Master trigger register PVs |
| `MTrigUser.template` | MTRG | User-friendly MTRG PVs |
| `RTrigRegisters.template` | RTRG | Router trigger register PVs |
| `RTrigUser.template` | RTRG | User-friendly RTRG PVs |
| `MDigRegistersVME.template` | DIG VME FPGA | VME FPGA status: power OK, voltage rails, temp sensors, clock select, `vme_code_revision_RBV`, `SERIAL_NUMBER_RBV` (used on crates 66 and 99) |
| `MDigUserVME.template` | DIG VME FPGA | User-facing VME FPGA PVs: power status bits, `serial_num_RBV`, `vme_code_revision_RBV`, `clk_select` |
| `SDigRegistersVME.template` | Slave DIG VME FPGA | Same as MDigRegistersVME for slave boards |
| `SDigUserVME.template` | Slave DIG VME FPGA | Same as MDigUserVME for slave boards |
| `daqCrate.template` | Per-crate | Crate-level: InLoop counters, BoardType0–N mbbi (per slot), inLoop_Running bi, dummy X slot PVs |
| `daqSegment2.template` | Per-board | Per-board: `CS_Ena` bo (readout enable) + `FifoNum` mbbo (FIFO select) |

**`daqSegment2.template` — FifoNum encoding** (used by `inLoop` to select which FIFO to drain per board):

| Value | FIFO name | Notes |
|-------|-----------|-------|
| 0–5 | MONFIFO 1–6 | Monitor FIFOs (diagnostic data) |
| 6 | MAIN DATA FIFO | **Default** — primary data FIFO |
| 7 | MONFIFO 8 | Monitor FIFO 8 |
| 8–15 | CHAN A–H FIFO | Per-channel FIFOs |

Default `FifoNum=6` (MAIN DATA FIFO) is set via `DOL=6`/`PINI=YES` at IOC boot. `CS_Ena` defaults to `Disable` (0); must be explicitly enabled per board. ✅ verified 2026-04-08 — `ioc/db/daqSegment2.template` + `ioc/db/daqCrate.template`

**`daqCrate.template` — InLoop PVs:**
- `DAQC$(CRATE)_CV_InLoop1` — MB/s read rate
- `DAQC$(CRATE)_CV_InLoop2` — Type F buffer raw count
- `DAQC$(CRATE)_CV_InLoop3` — Number of VME transfers
- `DAQC$(CRATE)_CV_InLoop4` — Result of last transfer
- `DAQC$(CRATE):inLoop_Running` — bi (inter-process signal: inLoop→outLoop)

**Also in `daqCrate.template`** (less-documented PVs):
- `DAQC$(CRATE)_CV_SenderRunning` — bi (MiniSender running status: Stopped/Running)
- `DAQC$(CRATE)_OL_HeaderSummaryEnable` — ao (outLoop header summary enable, default 0)
- `DAQC$(CRATE)_OL_HeaderSummaryPrescale` — ao (outLoop header summary prescale, default 4096)
- `DAQC$(CRATE)_OL_HeaderSummaryEventPrescale` — ao (outLoop per-event prescale, default 256)
- `VME$(CRATE):X:CS_Ena` — dummy bo for unoccupied VME slots (always Disable=0); `inLoop` references this for all missing boards so it doesn't need special-case logic for empty slots
✅ verified 2026-04-09 — `ioc/db/daqCrate.template:L380-430`

**Note:** The four `*VME.template` files were added after Feb 2024 (not in `DB_backup_20240205`). They expose VME FPGA internals for crates 66 (DuoGe) and 99 (test stand).

### MTrigUser.template — Key PV Groups

`MTrigUser.template` is 70,386 lines — the largest template by far. The bulk is repetitive `COINC_TRIG_MASK_*` and per-bit RAM records. Key functional PV groups:

| PV Group | Records | Purpose |
|----------|---------|--------|
| `ClkSrc` | mbbo/mbbi | Clock source select (local / link L / link R / link U) |
| `SLOW_CLOCK_SEL` | mbbo/mbbi | Slow clock rate: 10MHz down to 1Hz (7 choices) |
| `EN_SUM_X/Y/XY`, `EN_MAN_AUX`, `EN_ALGO5`, `EN_LINK_L/R/U`, `EN_MYRIAD_LINK_U` | bo/bi | Per-algorithm trigger enables |
| `ALGO_5_SELECT` | bo/bi | Algo 5 mode: coincidence trigger (1) vs fast-strobe CPLD (0) |
| `LINK_L/R/U_IS_TRIGGER_TYPE` | bo/bi | Link mode select: 0=MyRIAD/normal, 1=remote timestamp |
| `LINK_L_CMD_FORMAT` | mbbo/mbbi | Link L command format: DGS / GRETINA / RTR |
| `ILM_A..H/L/R/U` | bo/bi | Input link mask (1=masked, excluded from trigger sums) |
| `XLM_A..H`, `YLM_A..H` | bo/bi | X/Y sum link mask (1=excluded from X/Y multiplicity sum) |
| `LINK_L/R/U_PROPAGATE_F3..F7` | bo/bi | Propagate trigger frames F3–F7 out link L/R/U |
| `EN_NIM_VETO_A..H`, `EN_RAM_VETO_A..H`, `EN_THROTTLE_VETO_A..H` | bo/bi | Per-algorithm veto enables |
| `ENBL_NIM_VETO`, `ENBL_MON7_VETO`, `ENBL_THROTTLE_VETO`, `EN_RAM_VETO` | bo/bi | Global veto enables |
| `SOFTWARE_VETO` | bo/bi | Software-controlled veto |
| `TRIG_MON_SEL` | mbbo/mbbi | TDC trigger monitor source: AUX/NIM, SumX, SumY, SumXY, CPLD, RemMstr(L/R), MyRIAD(U) |
| `AuxTrig_Width` | longout/longin | AUX/NIM trigger width in clocks |
| `AUXPolaritySelect` | bo/bi | AUX input polarity |
| `EN_NIM1_DELAY`, `EN_NIM2_DELAY` | bo/bi | Enable programmable delay on NIM inputs 1/2 |
| `VETO_RAM_ADDR_SRC` | mbbo/mbbi | Veto RAM address source: AUX I/O / 1024µs / DecadeRate / Manual / LinkL-U Det |
| `TRIG_RAM_ADDR_SRC`, `SWEEP_RAM_ADDR_SRC` | mbbo/mbbi | Trigger/sweep RAM address sources |
| `VETO_RAM_PA/PB/PC/PD_B0..B15` | bo/bi | Veto RAM port A-D bit-wise pattern (64 PVs) |
| `TRIG_RAM_*`, `SWEEP_RAM_*` | bo/bi | Trigger/sweep RAM bit patterns |
| `COINC_TRIG_MASK_A1..A8/B1..B7` | bo/bi | Coincidence trigger mask: which detectors participate in each of 14 coincidence groups. **28 records total** (14 × bo + bi RBV) — one per group, not per detector. ✅ verified 2026-04-15 — `MTrigUser.template` grep: 28 COINC_TRIG_MASK records |
| `SSI_InputSelect` | mbbo/mbbi | SSI clock input pin pair (16 options across J34–J37/P101/ECL) |
| `SSI_BIT_RANGE` | mbbo/mbbi | SSI bit range (9..0 through 15..6) |
| `LEDControl` | mbbo/mbbi | Front panel LED source: Link Status / Trig Status / Manual |
| `CLEAR_RATE_COUNTERS`, `CLEAR_DIAG_COUNTERS`, `CLEAR_ENCODER_CNTR` | bo/bi | Counter clears |
| `ALL_CHANNEL_RESET` | bo/bi | Reset all channels |
| `LRUCtl00..10` | bo/bi | Link L/R/U control bits: DEN (data enable), REN (receive enable), Sync |
| `TPwr_*/RPwr_*`, `SLoL_*`, `SLiL_*` | bo/bi | Per-link transmit/receive power, line/local loopback |
| `EN_RTR_DCBAL`, `EN_LINKL/R/U_TX_DCBAL` | bo/bi | DC balance enable for router/link outputs |
| `A_3_0_DIR`, `A_7_4_DIR`, `B_3_0_DIR`, `B_7_4_DIR` | bo/bi | Aux I/O port A/B direction bits |
| `CFC1..7` | mbbo/mbbi | Command FIFO channel selects |
| `CFIFO1..8_FORCE` | bo/bi | Force FIFO entries |
| `CF0..7_F12RESET` | bo/bi | Frame 12 reset per command FIFO channel |

**Size breakdown** (corrected ✅ verified 2026-04-15 — `grep -c "^record("` on `ioc/db/MTrigUser.template`):
- Total: **7,056 records** in this template
- `VETO_RAM_*` records: **2,050** PVs (per-bit veto RAM patterns)
- `TRIG_RAM_*` records: **2,052** PVs (per-bit trigger RAM patterns)
- `SWEEP_RAM_*` records: **2,050** PVs (per-bit sweep RAM patterns)
- RAM records total: **6,152** — these dominate the file
- `COINC_TRIG_MASK_*` records: **28** PVs (14 coincidence groups × bo + bi RBV) — _not_ ~7,000 as previously estimated
- Other functional PVs: **876** (all non-RAM, non-COINC_TRIG_MASK records)

_Source: `ioc/db/MTrigUser.template` (70,386 lines) — explored 2026-04-08_

### Boot script flow
1. VxWorks loads from FTP
2. NFS mounts set up (`nfsCommands`)
3. `vme66.cmd` runs: loads munch, loads DBs, loads firmware, starts IOC

---

## findAllPV.py — PV Discovery Tool

`ioc/findAllPV.py` generates `All_PV.json` — the master PV list consumed by the ANLDAQ GUI.

**How it works:**
1. Parses `dbLoadRecords("template.db", "MACRO=val,...")` lines from IOC startup scripts
2. Reads each `.template` / `.db` file with macro substitution applied
3. Extracts all `record(type, "PV_name")` declarations
4. Detects whether an `_RBV` readback exists for each PV
5. Outputs `All_PV.json`: a list of `[PV_name, {"Type": "IN"|"OUT", "RBV": "Exist"|"None"}]`

**Skip list** (files excluded from PV scan):
- `db/asynDebug.template` — debug records not needed in GUI
- `db/dgsGlobals_DGS_VME99.db` — test stand globals (added 2026-02-26, commit 184570b) ✅ verified 2026-04-17 — `ioc/findAllPV.py` diff

**Output:** `ioc/All_PV.json` — 13,887 entries (as of 2026-04-06)

**When to regenerate:** after changing IOC boot scripts, adding/removing DB templates, or modifying PV names

```bash
cd ioc && python3 findAllPV.py
```

> **Note:** This satisfies the ANLDAQ GUI's PV discovery need. The open QUEUE task ("Auto-generate EPICS DB + PV list from FPGA register map") aims to go further upstream — generating the `.template` DB files themselves from the FPGA register map, so `findAllPV.py` can then derive `All_PV.json` automatically.

**Boot-time PV dump:** As of 2026-02-26 (commit 4eb1eb0), `vme66.cmd` and `vme99.cmd` both end with `dbl > "vme66_db.txt"` / `dbl > "vme99_db.txt"` — VxWorks `dbl` command dumps all loaded PV names to a text file on the IOC at startup. Useful for offline PV discovery without a running CA gateway. `bootFiles.txt` reverted to point at `boot/vme66.cmd` (from `vme99.cmd`) as the primary startup script reference. ✅ verified 2026-04-17 — `ioc/boot/vme66.cmd` + `ioc/boot/vme99.cmd` diff commit 4eb1eb0

---

## Connections to Other Subsystems

- **vxworks/** — provides the `gretDet.munch` binary stored in `bin/`
- **fpga/** — provides the `.bin` firmware files stored in `firmware/`
- **ANLDAQ** — reads/writes the EPICS PVs defined in `db/` and `dbd/`; uses `findAllPV.py`-generated `All_PV.json` for PV discovery
- **collectorboxpi/** — parallel IOC for collector box hardware (different platform)
- **EPICS_asyn.md** — asyn driver internals: caput/caget flow diagrams, port concept, asynUInt32Digital
- **collectorbox_devicesupport.md** — EPICS device support for collector box (SPI, CAMAC_IO)
- **VME_registers.md** — complete VME register address map for all DGS FPGA boards
- **DIG_firmware_expert.md** — DIG firmware expert guide; complements the DB template PV definitions with firmware-level detail

---

## IOC Connections: Ethernet vs Terminal Server

The VME IOC has **two separate physical connections**:

### 1. Ethernet (Data + EPICS)
- Used for: EPICS Channel Access (PV read/write), TCP data stream (port 9001 for `tcpReceiverMT`) ✅ verified 2026-04-08 — `SendReceiveSupport.c:L120` (`#define SERVER_PORT 9001`) + `tcpReceiverMT.cpp:L55` (`cfg.port = 9001`)
- Address = `IOC_IP` in `EPICS_para.sh`
- Example: DuoGe vme66 = **192.168.203.81**
- This is the connection used by ANLDAQ GUI, pyepics scripts, and the data receiver

### 2. Terminal Server (Console/Shell access)
- Used for: VxWorks shell access via telnet (for IOC config, debugging, `dbl`, etc.)
- The VME crate's **serial console port** is connected to a **terminal server** (RS-232 → Ethernet converter)
- Terminal servers come in 4, 6, or 12 port variants on the onenet network
- Address = `TERMINAL_SERVER` in `EPICS_para.sh`

**Terminal server port formula** (from `ANLDAQ/gui/commander.py`): ✅ verified 2026-04-17 — `ANLDAQ/gui/commander.py:L828` (`port = 2000 + id`) + `L839` (`telnet {terminal_server} {port}`)
```
telnet <TERMINAL_SERVER_IP> <2000 + IOC_id>
```
Where `IOC_id` = index of the IOC in the DAQ list (1-based).

### Terminal Server Assignments

| System | Terminal Server IP | IOC Ethernet IP | Telnet Port |
|--------|-------------------|-----------------|-------------|
| DuoGe (DUO) | 192.168.203.54 | 192.168.203.81 (vme66) | 2000 + id | ✅ verified 2026-04-17 — `ANLDAQ/EPICS_para.sh:L18-19` |
| X-Array (DXA) | 192.168.203.47 | 192.168.203.212, .213 | 2000 + id | ✅ verified 2026-04-17 — `ANLDAQ/EPICS_para.sh:L27-28` |
| SlopeBox (test) | 192.168.203.139 (ts99) | vme99 | 2003 (id=3) | ✅ verified 2026-04-17 — `ANLDAQ/EPICS_para.sh:L38-39` |
| DGS crates 1–6 | 192.168.203.186 (gs-ts-south, even GS IDs) | .141–.145, .177–.183 | 2000 + id | ✅ verified 2026-04-17 — `ANLDAQ/EPICS_para.sh:L47` |
| DGS crates 7–12 | 192.168.203.91 (gs-ts-north, odd GS IDs) | (same pool) | 2000 + id | ✅ verified 2026-04-17 — `ANLDAQ/EPICS_para.sh:L47` |

**Example — connect to DuoGe IOC console:**
```sh
telnet 192.168.203.54 2001    # id=1 for vme66
```

**Example — ANLDAQ GUI opens console:**
The GUI button "IOC-1" runs:
```sh
gnome-terminal -- bash -c "telnet 192.168.203.54 2001; exec bash"
```

> ⚠️ Note: the port number depends on which physical RS-232 port on the terminal server the VME crate is connected to. Verify with the lab's terminal server cabling documentation or by checking which port the GUI uses.

---

## Operational Notes

- Git LFS required: `git lfs pull` after clone to get firmware/munch/vxWorks binaries
- Firmware version table is the ground truth for what IOC configuration is active
- Multiple VME crates share the same munch binary but have per-crate boot scripts (`vme66.cmd`, `vme99.cmd`, etc.)
- After updating firmware: update the `ioc/firmware/` binaries, update the README version table, update boot scripts if needed


## Cross-References

- [`vxworks.md`](vxworks.md) — VxWorks cross-compilation: build pipeline, munch process, directory structure
- [`vxworks_migration.md`](vxworks_migration.md) — Migration from Solaris/con6 to Ubuntu 24; all path/source fixes
- [`EPICS.md`](EPICS.md) — EPICS record types, CA tools, Python integration
- [`EPICS_asyn.md`](EPICS_asyn.md) — asyn driver: caput/caget flow, port concept, worker threads
- [`EPICS_implementation_tools.md`](EPICS_implementation_tools.md) — DBGEN macro/substitution workflow: how `make_Substitutions.pl` + `gen_db.pl` generate new template files for new hardware
- [`EPICS_DB_templates.md`](EPICS_DB_templates.md) — All 8 IOC `.template` files documented: PV naming scheme, per-board instantiation, record counts
- [`EPICS_RTrig_templates.md`](EPICS_RTrig_templates.md) — RTrig EPICS DB templates deep-dive: RTrigRegisters + RTrigUser complete PV inventory (created 2026-04-25)
- [`trig_setup_scripts.md`](trig_setup_scripts.md) — 5-stage trigger setup scripts (trig_setup_Stage1–5.sh); system bring-up from cold
- [`snapshot_pv.md`](snapshot_pv.md) — PV snapshot utility (`dumpPVs.py` / `putPVs.py`) invoked by `start_run.sh`
- [`guceiver.md`](guceiver.md) — Guceiver live diagnostic GUI (waveform/spectrum/TAC-II tabs); companion to the ANLDAQ GUI
- [`VME_registers.md`](VME_registers.md) — Complete VME register addresses extracted from asyn driver source
- [`IOC_cmd.md`](IOC_cmd.md) — IOC shell commands available in DGS VxWorks IOC
- [`fpga.md`](fpga.md) — FPGA firmware overview; firmware revisions that must match IOC boot scripts
- [`vxworks_state_machines.md`](vxworks_state_machines.md) — DAQ runtime state machines (`inLoop.st`, `outLoop.st`, `MiniSender.st`); buffer pool; trigger driver summary
- [`vxworks_fifo_readout.md`](vxworks_fifo_readout.md) — DMA buffer architecture, trigger FIFO readout, Type-F synthetic headers, FIFO index map
- [`vxworks_trigger_drivers.md`](vxworks_trigger_drivers.md) — Deep-dive into trigger asyn drivers (`asynTrigCommonDriver`, `asynTrigMasterDriver`, `asynTrigRouterDriver`); firmware type code table
- [`troubleshooting.md`](troubleshooting.md) — IOC connectivity issues, SYNC bit gotcha
- [`DGS_setup_guide.md`](DGS_setup_guide.md) — DuoGe commissioning walkthrough (living doc): full step-by-step setup from IOC boot through data taking for the 2-detector DuoGe system on tangerine/vme66
- [`nfs_layout.md`](nfs_layout.md) — NFS-hosted IOC software tree on `vol2/global_32/ioc/` (boot scripts, py_scripts, dgsReceiver, dgsSoftIOC, fastEventContructor, FW_Maint, EDM screens)

---

## Full Build & Deployment Procedure (Gammasphere System)

*Source: wiki.anl.gov/gsdaq/Building_the_Entire_System (fetched 2026-04-25)*

### Shared File System — /global Mapping

Machine DGS1 and other machines share a file server. `/global` maps to different locations depending on OS:

| Machine type | /global maps to |
|---|---|
| 64-bit OS (e.g., Rocky Linux) | `/dk/fs2/dgs/global_64` |
| Scientific Linux 6 (e.g., DGS1) | `/dk/fs2/dgs/global_32` |
| CON6 (Solaris build machine) | `/dk/fs2/dgs/global_sandbox` |

⚠️ **DO NOT TOUCH `/dk/fs2/dgs/global_sandbox` directly.** CON6 uses it as its `/global`. 

### Boot Host for Gammasphere VME IOCs

- Machine **DGS1** is the boot host for VME01–VME12 (Gammasphere)
- VxWorks OS image + boot scripts for all VME processors located at `/global/ioc/` on DGS1
- Boot sub-folders: `bin/`, `boot/`, `db/`, `dbd/`, `dgsSoftIOC/`, `epics/`, `FW_Maint/` (deprecated), `gui/`

### Build Sequence (VME IOC Binary)

1. `ssh dgs@dgs1`
2. `cd /dk/fs2/dgs/global_sandbox/devel/dgsDrivers/dgsDriverApp/src`
3. `./Export_SVN_ParameterFiles_from_dgs1.sh` — pulls latest asynParams.c/.h from SVN
4. `ssh dgs@con6`
5. `cd /global/devel/dgsDrivers && make clean && make` — **ZERO errors/warnings required**
6. `cd ../dgsIoc && make clean && make` — **ZERO errors/warnings required**
7. Log out of con6; log back into dgs1
8. `cd /global/ioc/bin/vxWorks-ppc604_long && ./CopyNewMunch.sh` — deploys new munch binary

### Deploying Updated EPICS Databases

```sh
ssh dgs@dgs1
cd /global/ioc/db
./Export_SVN_Databases.sh
# Then: reboot all VME IOCs + restart Soft IOC to load new databases
```

### Deploying Updated Soft IOC Database

```sh
cd /global/ioc/dgsSoftIOC/db
./Export_SVN_SoftIOC_Database.sh
```

### Spreadsheet → Boot Full Sequence

1. Update the code-generating spreadsheet on Windows PC (SVN repo: `psg/CodeGeneratingSpreadsheetsGeneric`)
2. Commit all outputs under `SS_output/` to SVN
3. Export `asynParams.c` / `asynParams.h` to CON6 via the export script
4. Compile on CON6 (dgsDrivers → dgsIoc); run CopyNewMunch.sh on DGS1
5. Export `Registers.template`, `User.template`, `VMExx.db` files: run `Export_SVN_Databases.sh` on DGS1
6. Export `JustGlobals.db`: run `Export_SVN_SoftIOC_Database.sh` on DGS1
7. Reboot all VME IOCs + restart Soft IOC
