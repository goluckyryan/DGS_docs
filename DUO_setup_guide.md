# DUO Setup Guide — DuoGe Commissioning Walkthrough

Stability: C1 - Active / Living Document

> 🔗 **Related:** [overview_DGS.md](overview_DGS.md) — system overview | [ioc.md](ioc.md) — IOC details | [fpga.md](fpga.md) — firmware | [collectorboxpi_commissioning.md](collectorboxpi_commissioning.md) — SBX/collector commissioning

**Source:** Live exploration of `tangerine` (192.168.203.78), `/global/ioc/`, ANLDAQ source, KB cross-references  
**Last updated:** 2026-04-30 (25 passes — all 9 sections + appendix complete. New in pass 25: Resolved BGO bias supply voltage question — ~450V and ~400V confirmed as nominal internal SBX rails, derived from `SlopeBox.db` alarm limits. Remaining open item: physical signal chain SBX/cable details (requires Ryan).

---

## Table of Contents

1. [Hardware Connections](#1-hardware-connections)
2. [IOC Setup](#2-ioc-setup)
3. [Firmware Programming](#3-firmware-programming)
4. [EPICS PV Verification](#4-epics-pv-verification)
5. [HV Nominal Settings](#5-hv-nominal-settings)
6. [EDM Screens](#6-edm-screens)
7. [Trigger Scheme Setup](#7-trigger-scheme-setup)
8. [tcpReceiver / Data Acquisition Startup](#8-tcpreceiver--data-acquisition-startup)
9. [Data Analysis](#9-data-analysis)

---

## 1. Hardware Connections

### DuoGe Minimal System Overview

DuoGe is a **2-Germanium detector** minimal DGS system. The hardware lives in a single VME crate (CRATE=66, target name `vme66`) housed at ANL, served by **tangerine** (192.168.203.78, Rocky Linux 8).

### VME Crate Slot Map (vme66)

From `/global/ioc/boot/vme66.cmd`:

| Slot | Board  | EPICS Name | Board# | Notes                        |
|------|--------|------------ |--------|------------------------------|
| 1    | IOC    | —           | —      | MVME5500 CPU                 |
| 3    | MDIG1  | `VME66:MDIG1` | 0   | Primary digitizer            |
| 4    | MDIG2  | `VME66:MDIG2` | 1   | Secondary digitizer (SDig)   |
| 6    | RTR1   | `VME66:RTR1`  | 4   | Router trigger               |
| 7    | MTRG   | `VME66:MTRG`  | 5   | Master trigger               |

> ℹ️ Slots 2, 5, 8+ are empty in this configuration.

### Network

| Host       | IP               | Role                           |
|------------|------------------|-------------------------------|
| tangerine  | 192.168.203.78   | Linux host, FTP server, EDM   |
| vme66      | 192.168.203.81   | VME IOC (MVME5500, vxWorks)   |

- **Subnet mask:** `ffffff00` (255.255.255.0)
- vme66 FTP boots from tangerine; tangerine must have **vsftpd** (or equivalent) running and `/global/ioc/` exported

### Signal Chain (SBX / Pickoff / Collector)

❓ QUESTION FOR RYAN: What is the physical signal path from the Ge detectors to MDIG1/MDIG2 inputs for DuoGe? (SBX model, cable type, Pickoff Tee arrangement)

✅ **BGO shields confirmed present:** `RunTimestamp.txt` records "BGO veto" runs (haha_1002/1003), and `SBX_Det_Ctl.edl` monitors `GS$(DetNbr)_SlopeBoxBGO_HV_On` — BGO suppression shields are part of the DuoGe hardware. ✅ verified 2026-04-30 — `/global/tacReceiver/data/RunTimestamp.txt` + `/global/edm/screens/SBX/SBX_Det_Ctl.edl`

### MDIG1 Channel Type Assignments (DuoGe)

The standard DGS channel assignment (from `dgsGlobals_DGS_VME99.db`) maps signal types to MDIG1 channel ranges:

| Channels | Signal Type | Notes |
|----------|-------------|-------|
| ch 0–4   | **BGOs** (Bi-silicate scintillator, summed shield) | `BGOs_ext_disc_src` fanout |
| ch 5–9   | **GeC** (Ge Center segment) | `GeC_ext_disc_src` fanout |

For SDIG1 (MDIG2 in DuoGe): ch 0–4 = BGOp, ch 5–9 = GeS (Ge Side segment).

**DuoGe conclusion:** The active channels on MDIG1 are:
- **ch 0, 1 = BGO shield signals** (two BGO channels)
- **ch 5, 6 = Ge Center (HPGe) signals** (two Ge detectors)

This explains why `basic_settings_DGS.sh` only configures ch 5–9 (Ge CFD parameters), while `start_run.sh` sets a higher LED threshold (200) for ch 0/1/5/6. ✅ verified 2026-04-30 — `dgsGlobals_DGS_VME99.db:L19-46` (BGOs=ch0-4, GeC=ch5-9)

---

## 2. IOC Setup

### Prerequisites

- tangerine must be running an **FTP server** on port 21, serving `/global/ioc/` to vme66.
  > ⚠️ **vsftpd CURRENTLY FAILED:** As of 2026-01-15, `vsftpd.service` on tangerine is in `failed (exit-code)` state and has been since January 15, 2026. The service is *enabled* but not running. This means vme66 **cannot boot** until vsftpd is repaired or restarted. ✅ verified 2026-04-30 — `systemctl status vsftpd` on tangerine.  
  ❓ QUESTION FOR RYAN: Can you restart vsftpd on tangerine (requires sudo)? `sudo systemctl restart vsftpd` — if that fails, the config at `/etc/vsftpd/vsftpd.conf` needs to be inspected.
- NFS auth set in `/global/ioc/boot/nfsCommands`:  
  `nfsAuthUnixSet("fs.gam", 6000, 10, 0, 0)` — authenticates with NFS as uid 6000 (group 10)

### vme66 Boot Parameters

Set via vxWorks bootrom (serial console or telnet to bootROM):

```
boot device          : gei
unit number          : 0
processor number     : 0
host name            : tangerine
file name            : /global/ioc/mvme5500/vxWorks
inet on ethernet (e) : 192.168.203.81:ffffff00
host inet (h)        : 192.168.203.78
user (u)             : dgs
ftp password (pw)    : <stored in secrets>
flags (f)            : 0x0
target name (tn)     : vme66
startup script (s)   : /global/ioc/boot/vme66.cmd
```

### Startup Script: `/global/ioc/boot/vme66.cmd`

Sequence on boot:

1. `cd /global/ioc/boot` — set working directory
2. `< cdCommands` — set environment variables (CA port, NTP, timezone, paths)
3. `< nfsCommands` — NFS auth
4. `ld < gretDet_DGS.munch` — load IOC binary from `/global/ioc/bin/vxWorks-ppc604_long/`
5. `dbLoadDatabase("dbd/gretDet.dbd")` — load database definitions
6. Load EPICS records (see below)
7. `iocInit()` — start IOC
8. `setupFIFOReader()` — initialize readout FIFO queue
9. `dbpf` — set `user_package_data` PVs (170, 171, 172)
10. `seq &inLoop, &outLoop, &MiniSender` — start DAQ sequencer tasks

### EPICS CA Port

DuoGe uses **non-default CA ports** to avoid conflicts with other DGS systems:

```
EPICS_CA_SERVER_PORT = 5080
EPICS_CA_REPEATER_PORT = 5081
```

All `caget`/`caput` clients must set `EPICS_CA_SERVER_PORT=5080`.

Path to `caget` on tangerine: `/global/base/base-3.14.12.8/bin/linux-x86_64/caget`

**EPICS CA address list (DuoGe):**
```bash
export EPICS_CA_ADDR_LIST="192.168.203.81 192.168.203.78 192.168.203.164 192.168.203.158"
export EPICS_CA_AUTO_ADDR_LIST=NO
# .81 = vme66 (VME IOC), .78 = tangerine (softIOC), .164 and .158 = additional DuoGe nodes
```
> ✅ verified 2026-04-30 — `/global/EPICS_env.sh` SYSTEM=DUO block

**TERMINAL_SERVER:** `192.168.203.54` (duts.onenet) — VME terminal server for vme66 serial console access. ✅ verified 2026-04-30 — `/global/EPICS_env.sh:L22`

### Multi-System CA Port Reference

All DGS systems share the same network but use different CA ports to coexist. From `/global/EPICS_env.sh`:

| System   | Name                  | CA Server Port | CA Repeater Port |
|----------|-----------------------|----------------|------------------|
| DGS      | Gammasphere           | 5064           | 5065             |
| DFMA     | DGS-FRIB Mini-Array   | 5068           | 5069             |
| DXA      | X-array               | 5072           | 5073             |
| slopebox | Test stand            | 5074           | 5075             |
| DUB      | EuroBall              | 5078           | 5079             |
| **DUO**  | **DuoGe**             | **5080**       | **5081**         |

> ✅ verified 2026-04-30 — `/global/EPICS_env.sh` (all system port assignments)

The active system is selected by `export SYSTEM="DUO"` at the top of `EPICS_env.sh`. Source this file before any EPICS commands on tangerine:
```bash
source /global/EPICS_env.sh   # SYSTEM=DUO is the default on tangerine
```

### EPICS Databases Loaded (CRATE=66)

| Template                   | BOARD | Purpose                           |
|----------------------------|-------|-----------------------------------|
| `MTrigRegisters.template`  | MTRG  | Master trigger register-level PVs |
| `RTrigRegisters.template`  | RTR1  | Router trigger register-level PVs |
| `MDigRegisters.template`   | MDIG1 | Digitizer register-level PVs      |
| `SDigRegisters.template`   | MDIG2 | Digitizer register-level PVs      |
| `MTrigUser.template`       | MTRG  | Master trigger field-level PVs    |
| `RTrigUser.template`       | RTR1  | Router trigger field-level PVs    |
| `MDigUser.template`        | MDIG1 | Digitizer field-level PVs         |
| `SDigUser.template`        | MDIG2 | Digitizer field-level PVs         |
| `MDigRegistersVME.template`| MDIG1 | VME FPGA register PVs             |
| `SDigRegistersVME.template`| MDIG2 | VME FPGA register PVs             |
| `MDigUserVME.template`     | MDIG1 | VME FPGA field-level PVs          |
| `SDigUserVME.template`     | MDIG2 | VME FPGA field-level PVs          |
| `daqSegment2.template`     | MTRG, MDIG1, MDIG2 | Readout enable PVs |
| `daqCrate.template`        | (CRATE=66 only) | Per-crate DAQ PVs        |

### inLoop Board Map

```
CRATE=66, B0=MDIG1, B1=MDIG2, B2=X, B3=X, B4=X, B5=MTRG, B6=X
```

Note: B indices correspond to **board numbers** (not slots). B5=MTRG at board#5 = slot 7.

### user_package_data Assignment

| Board | PV                          | Value |
|-------|-----------------------------|-------|
| MDIG1 | `VME66:MDIG1:user_package_data` | 170 |
| MDIG2 | `VME66:MDIG2:user_package_data` | 171 |
| MTRG  | `VME66:MTRG:USER_PACKAGE_DATA`  | 172 |

### Full-System user_package_data Numbering Reference

For reference, the formula `[(VME# - 1) * 4] + 101 + Board#` applies to the full Gammasphere system:

| VME Crate | Boot Script              | MDIG1 | MDIG2 | MDIG3 | MDIG4 |
|-----------|--------------------------|-------|-------|-------|-------|
| VME01     | `vme01.4MDIG.cmd`        | 101   | 102   | 103   | 104   |
| VME02     | `vme02.4MDIG.cmd`        | 105   | 106   | 107   | 108   |
| VME32     | `vme32.trigger.cmd` (trigger crate) | — | — | — | — |
| VME32:MTRG| `dbpf MTRG USER_PACKAGE_DATA` | 99 | — | — | — |
| VME66     | `vme66.cmd` (DuoGe)      | 170   | 171   | — | — |
| VME66:MTRG| embedded in vme66.cmd    | 172   | — | — | — |

> ✅ verified 2026-04-30 — `~/DGS_system/ioc/boot/vme01.4MDIG.cmd` (dbpf 101–104), `vme02.4MDIG.cmd` (105–108), `vme32.trigger.cmd` (dbpf MTRG=99), `/global/ioc/boot/vme66.cmd` (dbpf 170/171/172). Slot map for VME32: MTRG at slot 3 (`asynTrigMasterConfig1("MTRG",0,3)`), RTR1 at slot 4 (`asynTrigRouterConfig1("RTR1",1,4)`).

> ℹ️ **MDIG2 in vme66 is an SDIG:** Despite being named `MDIG2`, it is loaded with `SDigRegisters.template` + `SDigUser.template` (not `MDigRegisters`/`MDigUser`). This is consistent with DGS architecture where the second board in each Ge detector pair is a Slave Digitizer. The full-system scripts use SDIG1/SDIG2 naming explicitly; vme66.cmd reuses the MDIG2 name but loads SDIG templates. ✅ verified 2026-04-30 — `/global/ioc/boot/vme66.cmd:L41-43` (`SDigRegisters.template`/`SDigUser.template` with BOARD=MDIG2).

### dgsSoftIOC — Linux Soft IOC

In addition to the vxWorks VME IOC, DuoGe runs a **Linux-based soft IOC** (`dgsSoftIOC`) on tangerine. This is a separate EPICS IOC that serves global fanout PVs and run control records.

**Source:** `~/DGS_system/softIOC/`  
**Boot script:** `iocBoot/iocdgsSoftIOC/dgsSoftIoc.cmd` (last edited MBO 20220711)

**Databases loaded:**

| Database         | Purpose |
|------------------|---------|
| `JustGlobals.db` | Global fanout PVs — distributes `master_fifo_reset` and signal-type discriminator source (`ext_disc_src`) across all VME crates via `dfanout` chains |
| `dgsSupport.db`  | Run control PVs: `Online_CS_StartStop` (Run/Stop), `Online_CS_SaveData` (Save/No Save), `Setup_Script_State` (8-state enum; last edit 20230415), `ScriptStage`, and trigger rate calc PVs (`VME10:MTRG:TIMESTAMP_RBV` for full-GS MTRG crate) |

**Key run control PVs (served by softIOC):**

| PV                     | Type | Values          | Description |
|------------------------|------|-----------------|-------------|
| `Online_CS_StartStop`  | bo   | 0=Stop, 1=Start | Main run/stop button (big red button in dgscommander) |
| `Online_CS_SaveData`   | bo   | 0=No Save, 1=Save | Data save enable |
| `Setup_Script_State`   | mbbo | 0=UNKNOWN, 1=TRIG OK, 2=DIG OK, 3=OTHER, 4=TRIG ERROR, 5=DIG ERROR, 6=OTHER ERROR, 7=SCRIPT RUNNING | Setup script state machine |
| `ScriptStage`          | ao   | integer         | Progress indicator during long setup scripts |

> ✅ verified 2026-04-30 — `~/DGS_system/softIOC/iocBoot/iocdgsSoftIOC/dgsSoftIoc.cmd`, `db/JustGlobals.db`, `db/dgsSupport.db`

> ⚠️ **Two softIOC trees exist on tangerine.** `~/DGS_system/softIOC/` is the DuoGe production softIOC (boot script: `dgsSoftIoc.cmd`, last edited MBO 20220711; `dgsSupport.db` last edited JTA 20230415; loads `JustGlobals.db` + `dgsSupport.db`). `/global/softIOC/` is a separate older tree whose `st.cmd` is a generic EPICS template (`dbLoadTemplate "db/userHost.substitutions"`) and does **not** load `JustGlobals.db` or `dgsSupport.db`; its `dgsSupport.db` is an older GRETINA-origin version with `DigFIFOSpeed`, `Data_CS_Dir`, `Data_CS_FileName`, and `VME32:MTRG` trigger rate calcs (vs VME10 in the production version). Use `~/DGS_system/softIOC/` for DuoGe. ✅ verified 2026-04-30 — `/global/softIOC/iocBoot/iocdgsSoftIOC/st.cmd`, `db/dgsSupport.db` (both trees inspected).

---

## 3. Firmware Programming

### Current Working Firmware (from `/global/ioc/README.md`)

This IOC repo is tagged for **TAC2 + Trigger Hold OFF**:

| Board | VME ID | Date Code | Revision | Comment                  |
|-------|--------|-----------|----------|--------------------------|
| MTRG  | 0xf13  | 0x1022    | 0x04A8   | TAC2 + Trigger Hold OFF  |
| RTRG  | 0xf13  | 0x0414    | 0x260E   | old but working          |
| MDIG  | 0xf13  | 20250704  | 0x4CD8   |                          |
| SDIG  | 0xf13  | 20250704  | 0x4CD8   |                          |

### Firmware Files in `/global/ioc/firmware/`

| File                          | Purpose                     |
|-------------------------------|-----------------------------|
| `BUS_LEFT.bin`                | DIG bus FPGA (left)         |
| `BUS_RIGHT.bin`               | DIG bus FPGA (right)        |
| `DIG_VME_FPGA_20220729.mcs`   | DIG VME FPGA bitfile        |
| `MTRG_VME_FPGA_20250711.mcs`  | MTRG VME FPGA bitfile       |
| `trigger_top.bin`             | MTRG trigger FPGA           |
| `V4747_mod_router_top.bin`    | RTRG (router) FPGA          |
| `uploadFW.cmd`                | vxWorks script to upload FW |

### Verifying Firmware Versions

After IOC boots, check:
```
caget VME66:MTRG:VME_CODE_REVISION   # expect ~0x04A8 (MTRG)
caget VME66:MDIG1:VME_CODE_REVISION  # expect ~0x4CD8 (MDIG)
caget VME66:RTR1:VME_CODE_REVISION   # expect ~0x260E (RTRG)
```

### Firmware Upload Procedure (`uploadFW.cmd`)

`uploadFW.cmd` is a vxWorks shell script run from the IOC console to re-flash firmware via the `ProgramFlash` / `ConfigureFlash` driver calls. It targets the flash chips on each board. Slot map for DuoGe:

| Slot | Board# | Device |
|------|--------|--------|
| 1    | —      | IOC    |
| 3    | 0      | MDIG1  |
| 4    | 1      | MDIG2  |
| 6    | 4      | RTRG   |
| 7    | 5      | MTRG   |

The script looks for FW files in `/global/ioc/FW_Maint/` (note: distinct from `/global/ioc/firmware/`). After flashing, it calls `ConfigureFlash` twice per board to ensure configuration. ✅ verified 2026-04-30 — `/global/ioc/firmware/uploadFW.cmd`

> ❓ QUESTION FOR RYAN: Is `uploadFW.cmd` the standard procedure for re-flashing firmware on DuoGe, or is a separate JTAG programmer used?

---

## 4. EPICS PV Verification

### CA Environment (set before caget)

```bash
source /global/EPICS_env.sh   # recommended: sets SYSTEM=DUO, port 5080, full addr list
# Or manually:
export EPICS_CA_SERVER_PORT=5080
export EPICS_CA_ADDR_LIST="192.168.203.81 192.168.203.78 192.168.203.164 192.168.203.158"
export EPICS_CA_AUTO_ADDR_LIST=NO
CAGET=/global/base/base-3.14.12.8/bin/linux-x86_64/caget
```

### Key PVs to Check After IOC Boot

| PV                              | Expected Value     | Notes                              |
|---------------------------------|--------------------|------------------------------------|
| `VME66:MTRG:VME_CODE_REVISION`  | 0x04A8             | MTRG firmware version              |
| `VME66:MDIG1:VME_CODE_REVISION` | 0x4CD8             | MDIG1 firmware version             |
| `VME66:MDIG2:VME_CODE_REVISION` | 0x4CD8             | MDIG2 firmware version             |
| `VME66:MTRG:USER_PACKAGE_DATA`  | 172                | Set by dbpf at startup             |
| `VME66:MDIG1:user_package_data` | 170                | Set by dbpf at startup             |
| `VME66:MDIG2:user_package_data` | 171                | Set by dbpf at startup             |
| `VME66:MDIG1:CS_Ena`            | 1 (enabled)        | Readout enable for MDIG1           |
| `VME66:MDIG2:CS_Ena`            | 1 (enabled)        | Readout enable for MDIG2           |
| `VME66:MTRG:CS_Ena`             | 1 (enabled)        | Readout enable for MTRG            |

> ⚠️ NOTE: As of 2026-04-30, the vme66 IOC is **not responding to CA** queries. The VME crate at 192.168.203.81 pings OK (ICMP responding), but CA on port 5080 times out — confirmed both on 2026-04-29 and 2026-04-30. Root cause is likely the vsftpd failure (see Section 2): vme66 cannot FTP-boot its IOC binary from tangerine while vsftpd is down. All PV expected values above are from KB/source — not live-verified. ✅ verified 2026-04-30 — ping 192.168.203.81 OK; `caget VME66:MDIG1:ID` timed out (CA port 5080).

❓ QUESTION FOR RYAN: Is vme66 normally left booted continuously, or does it need to be manually started?

---

## 5. HV Nominal Settings

### HV Architecture — SBX (Slope Box)

DGS uses **no external HV controller** (no CAEN/ISEG/WIENER). High voltage for both the Ge detectors and BGO shields is **generated internally by the SBX (Slope Box / Pickoff board)** and controlled via EPICS PVs served by the SBX IOC.

There is no `/global/hv/` directory or HV IOC on tangerine. HV is managed per-detector via `GS$(DetNbr)` PVs. ✅ verified 2026-04-30 — `find /global/ -name '*hv*'` returned nothing; `SBX_Det_Ctl.edl` + `dgsDetOverviewCharts.edl` confirm SBX-internal HV.

### HV EPICS PVs (per detector GS`NNN`)

| PV                                  | Meaning                                     |
|-------------------------------------|---------------------------------------------|
| `GS$(DetNbr)_SlopeBoxGe_HV_On`     | Ge HV enable status (readback)              |
| `GS$(DetNbr)_SlopeBoxBGO_HV_On`    | BGO HV enable status (readback)             |
| `GS$(DetNbr)_Conv_GeHV`            | Ge HV readback voltage [V]                  |
| `GS$(DetNbr)_Conv_BGO400`          | BGO bias supply ~400V readback [V]          |
| `GS$(DetNbr)_Conv_BGO450`          | BGO bias supply ~450V readback [V]          |
| `GS$(DetNbr)_SBX_Present`          | SBX present flag (used as vis condition)    |
| `GS$(DetNbr)_Conv_plus12V`         | +12 V supply readback                       |
| `GS$(DetNbr)_Conv_minus12V`        | −12 V supply readback                       |
| `GS$(DetNbr)_Conv_5V`              | +5 V supply readback                        |
| `GS$(DetNbr)_Conv_24V`             | 24 V supply readback                        |
| `GS$(DetNbr)_Conv_Temp`            | SBX temperature readback                   |
| `GS$(DetNbr)_SlopeBoxBGOInterlock` | BGO interlock status                        |

> Source: `/global/edm/screens/SBX/SBX_Det_Ctl.edl` and `/global/edm/screens/Detectors/dgsDetOverviewCharts.edl`

### Nominal HV Values

**DuoGe GS detector numbers — RESOLVED:**

✅ **Active detector pair: GS033 and GS091** — confirmed from `/global/edm/scripts/setup_HV.sh` which explicitly names the physical detectors:
- **GS033 = Ge14** (mounted in SBX-H3)
- **GS091 = Ge67** (mounted in SBX-CC)

This is confirmed by all live setup scripts: `setup_HV.sh`, `setup_sbx.sh`, `setup_dig.sh`. ✅ verified 2026-04-30 — `/global/edm/scripts/setup_HV.sh` (comment line: "SBX-H3/GS033=Ge14 and SBX-CC/GS91=Ge67")

> ⚠️ **Note:** The analysis software `~/fastEventContructor/analyzer.cpp` and `analyzer_tac.cpp` reference GS062 and GS070 — these are **stale** from an earlier run configuration and do not reflect the current DuoGe hardware. The analyzer would need to be updated with detX=33, detY=91 (or equivalent) for current data.

### Ge HV Nominal Values

From `/global/edm/scripts/setup_HV.sh`: **Ge HV demand = 3500 V** for both detectors (both GS033 and GS091 set to same value).

```bash
caput GS033_GE_HV_ABSMAX 3500
caput GS033_GE_HV_DEMAND_VOLTS 3500
caput GS091_GE_HV_ABSMAX 3500
caput GS091_GE_HV_DEMAND_VOLTS 3500
```

Also: `setup_HV.sh` sets `GS033_GeCenterGain 3.4MeV` and `GS091_GeCenterGain 3.4MeV` — but `setup_sbx.sh` (run afterwards) overrides this to **4.3 MeV** (final effective value). See SBX Configuration table below.

### BGO HV Per-PMT Values

From `/global/edm/scripts/setup_sbx.sh` — individual PMT HV settings (14 PMTs per detector, channels 0–13):

**GS033 BGO HV (V):**
| PMT | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 |
|-----|---|---|---|---|---|---|---|---|---|---|----|----|----|----|----|
| HV  | 120 | 0 | 150 | 160 | 120 | 150 | 115 | 135 | 100 | 105 | 100 | 100 | 140 | 140 |

> ⚠️ PMT1 HV = 0 V (disabled or dead PMT)

**GS091 BGO HV (V):**
| PMT | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 |
|-----|---|---|---|---|---|---|---|---|---|---|----|----|----|----|----|
| HV  | 120 | 125 | 145 | 145 | 175 | 175 | 180 | 170 | 145 | 145 | 120 | 115 | 130 | 140 |

> Source: `/global/edm/scripts/setup_sbx.sh` ✅ verified 2026-04-30

### SBX Configuration (from setup_sbx.sh)

| Parameter | Value |
|-----------|-------|
| `GeCenterGain` | 4.3 MeV |
| `BGOSumAttenuation` | X1 |
| `BGOpSelect` | Fix20MeV |
| `GeCenterTimeConstant` | 52.0 µs |
| `GeCenter_DCOffset` | 140 |
| `BGOsum_DCOffset` | 150 |
| `BGOpattern_DCOffset` | 150 |
| `Ge_Threshold` | 700 |
| `BGO_Threshold` | 30 |
| `PARST_ClampVoltage` | 200 |
| `PARST_AutoClampDwell` | 2000 |
| `PA_RST_WidthEdgeSel` | Neg2Neg |
| `PA_RST_WidthEnbl` | continuous |
| `GeSideInputSelect` | Fix8MeV (both segmented & non-segmented) |

### BGO Bias Supply Voltages

The SBX generates two internal BGO bias supplies, monitored via:
- `GS$(DetNbr)_Conv_BGO450` — ~**450V** nominal
- `GS$(DetNbr)_Conv_BGO400` — ~**400V** nominal

Nominal values are deducible from alarm limits in `SlopeBox.db`:

| PV Suffix    | LOLO  | LOW    | HIGH   | HIHI  | Nominal |
|--------------|-------|--------|--------|-------|---------|
| `Conv_BGO450`| 405 V | 427.5 V| 477.5 V| 495 V | ~452 V  |
| `Conv_BGO400`| 360 V | 380 V  | 420 V  | 440 V | ~400 V  |

> ✅ derived 2026-04-30 from `sbxPi/PickoffApp_RevC/db/SlopeBox.db` alarm fields (LOLO/LOW/HIGH/HIHI)  
> These are **internal SBX supply rails**, not user-settable. Normal operation expects both PVs in the LOW–HIGH band.

---

## 6. EDM Screens

### EDM Launch (from tangerine)

```bash
/global/edm/extensions/dgs1/src/edm/edmMain/O.linux-x86_64/edm \
  -m SYS=DUO,VM=66,VR=66,V1=66,V2=67 \
  -x runControl
```

Macros: `SYS=DUO`, `VM=66` (master VME crate), `VR=66` (router crate), `V1=66`, `V2=67`

> ℹ️ As of 2026-04-29, EDM is actively running in 3 persistent sessions on tangerine (PIDs 1106137, 1850920, 2084439).

### Available EDM Screens

**Global screens** (`/global/edm/screens/`):

| Screen                    | Purpose                                  |
|---------------------------|------------------------------------------|
| `runControl.edl`          | Top-level run control                    |
| `genBoard.edl`            | Generic board view                       |
| `genGlobal.edl`           | Global system overview                   |
| `genIOC.edl`              | IOC status                               |
| `gen_linkControl.edl`     | Link control                             |
| `genRouter.edl`           | Router trigger control                   |
| `dgsTrigCntrl.edl`        | DGS trigger control                      |
| `dgs_XYMAP.edl`           | XY detector map                          |
| `DigEnable.edl`           | Digitizer enable controls                |
| `DigitizerLinkStatus.edl` | Digitizer link status                    |
| `DigParamContrl.edl`      | Digitizer parameter control              |
| `AuxIO_Ctl.edl`           | Auxiliary I/O control                    |
| `CounterRates.edl`        | Rate counters                            |
| `eventRates2.edl`         | Event rate display                       |
| `iocStatus.edl`           | IOC status monitor                       |
| `NewTrigSummaryAndControl.edl` | Trigger summary and control         |
| `ThrottleControl.edl`     | DAQ throttle control                     |
| `Trace.edl`               | Trace display                            |
| `genVMEstats_4M_0S.edl`   | VME stats (4 MDig, 0 SDig config)        |
| `X_LiveTS2.edl`           | Live timestamp display                   |

**Trigger subscreens** (`/global/edm/screens/Trigger/`):

| Screen          | Purpose                   |
|-----------------|---------------------------|
| `MstrCPLD.edl`  | Master trigger CPLD view  |
| `RtrCPLD.edl`   | Router trigger CPLD view  |
| `WheelMap.edl`  | Trigger wheel map         |

**Detector screens** (`/global/edm/screens/Detectors/`):

| Screen                   | Purpose                  |
|--------------------------|--------------------------|
| `dgsDetOverviewCharts.edl` | Detector overview charts |

**Local screens** (`~/DGS_system/edm/screens/`):

| Screen                | Purpose                      |
|-----------------------|------------------------------|
| `runControl.edl`      | Local run control            |
| `trigCntrl.edl`       | Trigger control              |
| `genRouter.edl`       | Router control               |
| `trigWheelMap.edl`    | Trigger wheel map            |
| `trigWheelRAMCntrl.edl` | Trigger wheel RAM control  |
| `gen_linkControl.edl` | Link control                 |

---

## 7. Trigger Scheme Setup

### DuoGe Trigger Architecture

- **MTRG** (Master Trigger, slot 7): Collects coincidence decisions, generates global trigger
- **RTR1** (Router Trigger, slot 6): Routes local trigger signals from detectors to MTRG

### Digitizer Settings (from `/global/edm/scripts/setup_dig.sh`)

DuoGe runs in **LED_Mode** (not CFD mode). The actual DuoGe digitizer setup script is `/global/edm/scripts/setup_dig.sh` — applied to both MDIG1 and MDIG2 on VME66. All 10 channels (0–9) are initialized to Reset; specific channels are enabled later by `setup_custom.sh`.

> ⚠️ `/global/tacReceiver/basic_settings_DGS.sh` is **NOT** the DuoGe script — it targets VME01–VME12 (full 110-detector Gammasphere). Do not confuse it with `setup_dig.sh`.

**Global digitizer settings (applied to MDIG1 and MDIG2):**

| Parameter | PV suffix | Value | Notes |
|-----------|-----------|-------|-------|
| Trigger mux | `trigger_mux_select` | ExtTTCL | External TTCL trigger |
| CFD mode | `cfd_mode` | LED_Mode | Leading-edge discriminator |
| Peak sensitivity | `peak_sensitivity` | 3 | |
| Coincidence window min | `win_comp_min` | -6.30 | Overridden to -7.60 by setup_custom.sh |
| Coincidence window max | `win_comp_max` | -5.50 | Overridden to -6.60 by setup_custom.sh |
| Holdoff time | `holdoff_time` | 1.50 µs | Overridden to 1.2 by setup_custom.sh |
| Veto enable | `veto_enable` | Disabled | Overridden to Enabled by setup_custom.sh |

**Per-channel settings (all 10 channels, then overridden per-channel by setup_custom.sh):**

| Parameter | PV suffix | Value | Notes |
|-----------|-----------|-------|-------|
| Channel enable | `channel_enable{CH}` | Reset | All off; setup_custom.sh enables 0,1,5,6 |
| Trigger TS mode | `trig_ts_mode{CH}` | TrigMsg | |
| p1 window | `p1_window{CH}` | 0.07 µs | Peaking time 1 |
| p2 window | `p2_window{CH}` | 0.05 µs | Peaking time 2 |
| m window | `m_window{CH}` | 3.5 µs | Measurement window |
| k0 window | `k0_window{CH}` | 0.5 µs | LED mode (CFD mode uses 0.56) |
| k window | `k_window{CH}` | 0.5 µs | LED mode (CFD mode uses 0.16) |
| d window | `d_window{CH}` | 0.10 µs | LED mode (CFD mode uses 0.1) |
| d3 window | `d3_window{CH}` | 0.2 µs | |
| LED threshold | `led_threshold{CH}` | 50 | Per-channel overrides in setup_custom.sh |
| Raw data delay | `raw_data_delay{CH}` | 0.25 µs | |
| Raw data length | `raw_data_length{CH}` | 0.32 µs | Trace length |
| Downsample factor | `downsample_factor{CH}` | 1x | |
| Coarse threshold | `coarse_threshold{CH}` | 100 | |
| Trigger polarity | `trigger_polarity{CH}` | RiseEdge | Overridden per-channel by setup_custom.sh |
| Pileup mode | `pileup_mode{CH}` | Accept | GeC ch5/6 set to Reject by setup_custom.sh |
| Preamp reset delay en | `preamp_reset_delay_en{CH}` | Enabled | |
| Preamp reset delay | `preamp_reset_delay{CH}` | 45 | |
| Disc width | `disc_width{CH}` | 0.15 µs | |

✅ verified 2026-04-30 — `/global/edm/scripts/setup_dig.sh`

> ℹ️ `setup_dig.sh` resets all channels, then `setup_custom.sh` selectively enables ch0, ch1 (BGO) and ch5, ch6 (GeC), sets per-channel trigger polarity and pileup modes, and overrides win_comp/holdoff values.

### Run-Start Digitizer Enable Sequence (from `/global/receiver/start_run.sh`)

The `start_run.sh` script in `/global/receiver/` additionally sets LED thresholds for specific channels before starting:

```bash
caput VME66:MDIG1:led_threshold0 200
caput VME66:MDIG1:led_threshold1 200
caput VME66:MDIG1:led_threshold5 200
caput VME66:MDIG1:led_threshold6 200
```

Then enables master logic and readout:

```bash
caput VME66:MDIG1:master_logic_enable Enable
caput VME66:MDIG2:master_logic_enable Enable
caput VME66:MDIG1:CS_Ena Enable
caput VME66:MDIG2:CS_Ena Enable
```

### MTRG Multiplicity / Coincidence Settings

The MTRG trigger source mode is controlled by the `FS_trig_source` PV. Available modes (from `MTrigUser.template`):

| Value | Mode          | Description                                      |
|-------|---------------|--------------------------------------------------|
| 0     | fast or       | Any fast signal                                  |
| 1     | fast mult     | Fast multiplicity threshold                      |
| 2     | discbit or    | Discriminator bit OR                             |
| 3     | (AorB)AND(CorD) | Quadrant coincidence                           |
| 4     | A or B        |                                                  |
| 5     | C or D        |                                                  |
| 6     | 0             |                                                  |
| 7     | 1             |                                                  |

The multiplicity **threshold** is set via `VME66:MTRG:Threshold` (maps to `reg_FS_MULT_THRESH`); read back via `VME66:MTRG:Threshold_RBV`. Current multiplicity sum is read via `VME66:MTRG:Mult_sum_RBV` (maps to `reg_CPLD_MULT`).

Clean/dirty source options (from `RTrigUser.template`):

| Value | Description |
|-------|-------------|
| 1     | CLEAN       |
| 2     | DIRTY       |
| 4     | MODULE      |
| 0     | DISCBITS    |

**RESOLVED (2026-04-30):** `setup_custom.sh` confirms:
- **Multiplicity threshold:** `reg_SUM_OF_X_THRESH = 1` → any single Ge detector hit triggers (singles mode, not coincidence)
- **Clean/dirty source:** X_SELECT=CLEAN (GeC channels in X-plane XMAP), Y_SELECT=DISCBITS (from `setup_trig.sh`, not overridden)

> ✅ verified 2026-04-30 — `/global/edm/scripts/setup_custom.sh`

### Trigger System Initialization — SERDES Linkup

Before a run, the trigger boards must be linked up via SERDES. This is done by running `Serdes_Linkup.sh` from `/global/edm/scripts/`. It sources `VME_SYSTEM_DEFINES.sh` to know the system topology, then runs 5 sequential stages:

| Stage | Script | Purpose |
|-------|--------|---------|
| 1 | `trig_setup_Stage1.sh` | Set MTRG to drive SYNC pattern out all links |
| 2 | `trig_setup_Stage2.sh` | Initialize routers (local clock), receive SYNC from MTRG |
| 3 | `trig_setup_Stage3.sh` | Check SERDES lock status on all active links |
| 4 | `trig_setup_Stage4.sh` | Set digitizers to send SYNC to router, verify router lock |
| 5 | `trig_setup_Stage5.sh` | Flip all boards from SYNC to live data mode |

Each stage exits on failure with a message. ✅ verified 2026-04-30 — `/global/edm/scripts/Serdes_Linkup.sh`

> ⚠️ NOTE: `Serdes_Linkup.sh` removes any Link L F1 propagation during setup. Cross-system clock/triggering (if applicable) must be re-established afterward.

### DuoGe System Topology (`VME_SYSTEM_DEFINES.sh`)

`/global/edm/scripts/VME_SYSTEM_DEFINES.sh` defines the DuoGe system for all setup scripts:

```
MT_VME_LEADER = VME66
LIST_OF_DIGITIZERS: VME66 → MDIG1, MDIG2
LIST_OF_ROUTERS: VME66:RTR1 links A(active), B(unused), C(active), D-H(unused), L(active), R/U(unused)
LIST_OF_DETECTOR_GS_NUMBERS: (empty — DuoGe doesn't use the standard full-GS numbering scheme)
DIG_CLOCK_SEL = 0 (Serdes clock)
MT_USE_LINK_CLK = 0 (local clock)
PROPAGATE_TRIG_FROM_DUB/DFMA/DXA = 0 (standalone system)
```

✅ verified 2026-04-30 — `/global/edm/scripts/VME_SYSTEM_DEFINES.sh`

### Router and MTRG Parameters (`setup_trig.sh`)

After SERDES linkup, `/global/edm/scripts/setup_trig.sh` configures the trigger routing and coincidence logic:

**Clean/Dirty Router Settings:**

| Parameter | PV | Value | Notes |
|-----------|-----|-------|-------|
| Overlap window | `VME66:RTR1:OVERLAP_DELAY` | 25 | 25 × 20 ns = 500 ns clean/dirty window |
| Assertion delay | `VME66:RTR1:ASSERTION_DELAY` | 20 | 20 × 20 ns = 400 ns |
| X-plane source | `VME66:RTR1:X_SELECT` | CLEAN | Router reports clean multiplicity |
| Y-plane source | `VME66:RTR1:Y_SELECT` | DISCBITS | Router reports individual discriminator bits |
| Veto enable | `VME66:RTR1:ENABLE_VETO` | VETO ON:1 | Router vetoes dirty channel pairs |

**Throttle Settings:**

| Parameter | PV | Value | Notes |
|-----------|-----|-------|-------|
| Filter time | `VME66:RTR1:THROTTLE_FILTER_TIME` | 4 | × step = 81.92 µs |
| Time range | `VME66:RTR1:THROTTLE_TIME_RANGE` | `* 20.48 us` | Step size |
| Throttle width | `VME66:RTR1:THROTTLE_WIDTH` | 100 | 100 ticks = 2 µs |
| MTRG throttle veto | `VME66:MTRG:ENBL_THROTTLE_VETO` | 1 | All trigger types (A–H) throttle-gated |

**MTRG Trigger Enables (DuoGe defaults — all OFF in setup_trig.sh):**

```
EN_MAN_AUX=off, EN_SUM_X=off, EN_SUM_Y=off, EN_SUM_XY=off
EN_ALGO5=off, EN_LINK_L=off, EN_LINK_R=off, EN_MYRIAD_LINK_U=off
```

**DuoGe Run Enables — set by `/global/edm/scripts/setup_custom.sh`** (run after `setup_trig.sh`):

Only **EN_SUM_X** is enabled for DuoGe runs:

```bash
caput VME66:MTRG:EN_SUM_X 1           # Enable BGO sum-X trigger
caput VME66:MTRG:reg_SUM_OF_X_THRESH 1  # Threshold = 1 (any BGO hit triggers)
caput VME66:RTR1:X_SELECT CLEAN       # X source = clean multiplicity
```

All other trigger sources (EN_SUM_Y, EN_SUM_XY, EN_MAN_AUX, EN_ALGO5, links) remain **OFF**.

> ✅ RESOLVED — `/global/edm/scripts/setup_custom.sh` (confirmed 2026-04-30)

**Router XMAP assignments (from setup_custom.sh):**

| Map entry | Value | Meaning |
|-----------|-------|---------|
| XMAP_A_0 | 0 | Ch0 (BGO) excluded from X-plane |
| XMAP_A_1 | 0 | Ch1 (BGO) excluded from X-plane |
| XMAP_A_5 | 1 | Ch5 (GeC) included in X-plane |
| XMAP_A_6 | 1 | Ch6 (GeC) included in X-plane |
| XMAP_C_0 | 0 | Ch0 excluded from C-plane |
| XMAP_C_1 | 0 | Ch1 excluded from C-plane |
| XMAP_C_5 | 0 | Ch5 excluded from C-plane |
| XMAP_C_6 | 0 | Ch6 excluded from C-plane |
| OVERLAP_DELAY | 50 | 50 × 20 ns = 1 µs (overrides setup_trig.sh value of 25) |
| ASSERTION_DELAY | 30 | 30 × 20 ns = 600 ns (overrides setup_trig.sh value of 20) |

> ℹ️ Note: BGO channels (ch0/ch1) are excluded from the X-plane XMAP even though X_SELECT=CLEAN and EN_SUM_X=1. The "clean" multiplicity counts only GeC channels (ch5/ch6). BGO signals provide the DISCBITS for Y-plane routing but do **not** directly trigger MTRG in DuoGe.

**Digitizer settings (from setup_custom.sh):**

| PV | Value | Notes |
|----|-------|-------|
| `VME66:MDIG1:win_comp_min` | -7.60 | LED mode coincidence window min |
| `VME66:MDIG1:win_comp_max` | -6.60 | LED mode coincidence window max |
| `VME66:MDIG1:led_threshold0` | 120 | BGO ch0 LED threshold |
| `VME66:MDIG1:led_threshold1` | 120 | BGO ch1 LED threshold |
| `VME66:MDIG1:led_threshold5` | 50 | GeC ch5 LED threshold |
| `VME66:MDIG1:led_threshold6` | 50 | GeC ch6 LED threshold |
| `VME66:MDIG1:trigger_polarity0` | FallEdge | BGO ch0 triggers on falling edge |
| `VME66:MDIG1:trigger_polarity1` | FallEdge | BGO ch1 triggers on falling edge |
| `VME66:MDIG1:trigger_polarity5` | RiseEdge | GeC ch5 triggers on rising edge |
| `VME66:MDIG1:trigger_polarity6` | RiseEdge | GeC ch6 triggers on rising edge |
| `VME66:MDIG1:pileup_mode5` | Reject | Pileup rejection for GeC ch5 |
| `VME66:MDIG1:pileup_mode6` | Reject | Pileup rejection for GeC ch6 |
| `VME66:MDIG1:veto_enable` | Enabled | Veto logic enabled |
| `VME66:MDIG1:peak_sensitivity` | 3 | Peak finding sensitivity |
| `VME66:MDIG1:holdoff_time` | 1.2 | Holdoff time (µs) |

> ℹ️ MDIG2 has identical settings. BGO channels use falling-edge polarity (inverted signal from SBX); GeC channels use rising-edge.

✅ verified 2026-04-30 — `/global/edm/scripts/setup_custom.sh`

**NIM Input/Output:**

| PV | Value | Notes |
|----|-------|-------|
| `VME66:MTRG:EN_NIM1_DELAY` | Y | NIM1 delay enabled |
| `VME66:MTRG:reg_NIM1_DELAY` | 505 | Changed to 505 on Run 6+ |
| `VME66:MTRG:NIMSrc1` | AnyTrig | NIM out 1 = any trigger |
| `VME66:MTRG:NIMSrc2` | AnyTrig | NIM out 2 = any trigger |

✅ verified 2026-04-30 — `/global/edm/scripts/setup_trig.sh`

### user_package_data and Data Sorting

```
MDIG1 → 170   MDIG2 → 171   MTRG → 172
```

Formula reference (for future expansion): `[(VME# - 1) * 4] + 101 + Board#` for digitizers; `150` for master trigger in full DGS. DuoGe uses 170–172 (non-standard). **Explanation:** The formula would give `(66-1)*4 + 101 = 361` for VME66, which is out of range for a 2-crate mini-system. DuoGe uses 170–172 as a deliberately chosen non-formula assignment for this standalone system — the values are arbitrary but consistent (set via `dbpf` in `vme66.cmd`). ✅ verified 2026-04-29 — `/global/ioc/boot/vme66.cmd:L162-167` (comment explains formula; dbpf values confirm 170/171/172 hardcoded for DuoGe).

---

## 8. tcpReceiver / Data Acquisition Startup

### Receiver Programs

Two receiver programs are available on tangerine for DuoGe:

**A) `dgsReceiver`** — Located in `/global/receiver/`

Used by the older `/global/receiver/start_run.sh` script. Launch:
```bash
./dgsReceiver "192.168.203.81" "<output_dir>" "gtd01" "2000000000" "27"
```
Arguments: `<vme66 IP>` `<output folder>` `<run name>` `<max bytes>` `<GEB ID>`

**B) `tcp_Receiver`** — Located in `/global/tacReceiver/`

Used by the newer `/global/tacReceiver/start_run.sh`. Launch:
```bash
./tcp_Receiver 192.168.203.81 9001 <GEB_ID> <output_path/basename>
```
Arguments: `<vme66 IP>` `<port 9001>` `<GEB_ID>` `<output basename>`

GEB_ID for CFD mode: **14** (set in `/global/tacReceiver/start_run.sh`)

> ℹ️ Both receivers connect to the MiniSender port on vme66 (192.168.203.81).

### Run Data Storage

- **tacReceiver** data: `data/duo_NNN/` (e.g., `data/duo_001/` for run 001) relative to `/global/tacReceiver/` ✅ verified 2026-04-29 — confirmed from `start_run.sh:L5` (`expName="duo"`) and live data directory listing (duo_001–duo_021 present)
- **`haha_NNN` data:** also lives in `/global/tacReceiver/data/haha_NNNN/` — these are the older commissioning/test runs (haha_1000–haha_1021) using `expName="haha"`. **Both `duo_` and `haha_` runs use the same tacReceiver system and same hardware** — `haha` was the naming used before `duo` was adopted. ✅ verified 2026-04-30 — `ls /global/tacReceiver/data/`
- **`/global/receiver/`** — legacy dgsReceiver directory; `start_run.sh` there uses `expName="haha"` and GEB_ID=27, but stores to `/global/receiver/data/` (separate from tacReceiver data). Currently unused.
- **Receiver log:** `RunTimestamp.txt` in the data folder — timestamped run comments
- Run number is zero-padded 3 digits: `printf "%03d" $RUN`

### GEB_ID Values

| Receiver       | GEB_ID | Notes                                               |
|----------------|--------|-----------------------------------------------------|
| tacReceiver    | 14     | CFD mode; set in `/global/tacReceiver/start_run.sh` ✅ verified 2026-04-29 |
| dgsReceiver    | 27     | Older script; set in `/global/receiver/start_run.sh` |

### Active Data Channels (Observed)

From actual DuoGe run data (duo_001 through duo_021), files are named `duo_NNN_000_<boardID>_<ch>`:

| Board ID | EPICS Name   | Active Channels Observed | Notes                          |
|----------|--------------|--------------------------|--------------------------------|
| 0170     | VME66:MDIG1  | 0, 1, 5, 6               | All DuoGe data is from MDIG1 only ✅ verified 2026-04-29 |
| 0171     | VME66:MDIG2  | *(none observed)*        | No MDIG2 data files in any duo run |
| 0172     | VME66:MTRG   | *(none observed)*        | MTRG not producing separate data files |

> ℹ️ Channels 0,1 and 5,6 on MDIG1 correspond to the 2 Ge detectors. The older `/global/receiver/start_run.sh` explicitly overrides `led_threshold0/1/5/6` to 200 — confirming these 4 channels are the intentional active channels. ✅ verified 2026-04-30 — `/global/receiver/start_run.sh:L30-34` (`caput VME66:MDIG1:led_threshold0/1/5/6 200`).

> ✅ **Channel type confirmed:** `dgsGlobals_DGS_VME99.db` establishes the standard assignment: **ch0–4 = BGOs, ch5–9 = GeC**. For DuoGe: **ch0/1 = BGO shield signals, ch5/6 = Ge Center (HPGe) signals.** This explains why `basic_settings_DGS.sh` configures only ch5–9 (Ge CFD parameters) and `start_run.sh` applies a high LED threshold (200) to all four active channels (0/1/5/6). ✅ verified 2026-04-30 — `dgsGlobals_DGS_VME99.db:L19-46` (BGOs_ext_disc_src → ch0-4; GeC_ext_disc_src → ch5-9)

> ✅ BGO shields confirmed: `RunTimestamp.txt` in `/global/tacReceiver/data/` explicitly records "BGO veto" for runs haha_1002 and haha_1003. DuoGe is **not** bare Ge only — BGO suppression shields are in use. ✅ verified 2026-04-30 — `/global/tacReceiver/data/RunTimestamp.txt:L3-4`.

> ℹ️ haha_001 (earliest observed run) has data from **both** MDIG1 (0170) and MDIG2 (0171), channels 0,1,5,6. Later runs (haha_1000+, duo_NNN) have data only from MDIG1 (0170). MDIG2 was previously active but is currently idle.

> ℹ️ **TAC-II timing:** DuoGe uses a TAC-II (Time-to-Amplitude Converter, 2nd gen) for sub-nanosecond timing between the two detectors. The `tcp_Receiver` in `/global/tacReceiver/` captures both DIG energy packets (GEB_ID=14, CFD mode) and TAC-II timing packets from the same data stream. Analysis in `Aux/script.cpp` correlates DIG hits with TAC_hits for time-of-flight / coincidence timing. This is **distinct** from the older `dgsReceiver` (GEB_ID=27) which does not handle TAC packets.

> ✅ **ch0/1 vs ch5/6 resolved:** ch0/1 = BGO shield signals; ch5/6 = HPGe (Ge Center) signals. See Section 1 for full explanation.

### Full DAQ Startup Sequence

#### Prerequisites
1. tangerine FTP server running (vsftpd), serving `/global/ioc/`  
   ⚠️ vsftpd has been in `failed` state since 2026-01-15 — needs sudo to fix before vme66 can boot
2. vme66 IOC booted and loaded (check ping 192.168.203.81, CA on port 5080)

#### Pre-Run Settings
```bash
# Source EPICS environment (sets SYSTEM=DUO, port 5080, full CA addr list)
source /global/EPICS_env.sh
# EPICS_CA_ADDR_LIST is now set to all 4 DuoGe nodes by EPICS_env.sh

# Configure digitizer parameters via EDM setup script (DuoGe-specific)
/global/edm/scripts/setup_dig.sh
# NOTE: Do NOT use basic_settings_DGS.sh for DuoGe — see warning below
```

> ⚠️ **`/global/tacReceiver/basic_settings_DGS.sh` is NOT a DuoGe script.** It loops over `VME01` through `VME12` (full Gammasphere crate range) and programs channels 5–9 on each. It was placed in `/global/tacReceiver/` alongside DuoGe scripts but targets the full 110-detector Gammasphere system — running it against a DuoGe EPICS CA environment would spray 'channel not found' warnings and do nothing useful. The correct DuoGe digitizer setup is `/global/edm/scripts/setup_dig.sh`, which is VME66-specific. ✅ verified 2026-04-30 — `/global/tacReceiver/basic_settings_DGS.sh:L5` (`for vme in {01..12}`) — loops over all 12 full-GS VME crates.

#### Start a Run (tacReceiver method)
```bash
cd /global/tacReceiver
./start_run.sh <RUN_NUMBER> [COMMENT] [WAIT_SECONDS]
# Example: ./start_run.sh 1 "test run"
```
This script:
1. Creates `data/duo_NNN/` folder
2. Logs comment + timestamp to `RunTimestamp.txt`
3. Launches `tcp_Receiver` in an xterm (connects to 192.168.203.81:9001)
4. Waits 5 sec for receiver to connect
5. Enables MDIG1 and MDIG2 master logic + readout + FIFO reset
6. Calls `caput Online_CS_StartStop Start` and `caput Online_CS_SaveData Save`
7. If `WAIT_SECONDS > 5`: auto-stops after timeout; else runs until manually stopped

#### Stop a Run
```bash
caput Online_CS_StartStop Stop
sleep 10
caput Online_CS_SaveData No Save
# Or use: ./simpleStartStop.sh 0
```

#### Simple Start/Stop (no xterm)
```bash
./simpleStartStop.sh 1   # Start: SaveData=Save, StartStop=Start
./simpleStartStop.sh 0   # Stop: StartStop=Stop, sleep 5, SaveData=No Save
```

**MiniSender is a TCP server on vme66 (port 9001).** It calls `bind()` + `listen()` + `accept()` — the receiver on tangerine connects **to** vme66:9001 (pull model). `tcp_Receiver` initiates the connection; vme66 does not push to tangerine. ✅ verified 2026-04-29 — `SendReceiveSupport.c:L120` (`#define SERVER_PORT 9001`), `L184` (`bind`), `L196` (`listen`), `L220` (`accept`); `MiniSender.st:L106` (`InitRequestSocket()`), `L133` (`when (ConnectionAccepted > 0)`)

> ⚠️ **`~/DGS_system/receiver/` is NOT the DuoGe receiver.** The scripts in `~/DGS_system/receiver/` (`start_run.sh`, `basic_settings.sh`, `generateTrigger.sh`, `kill_IOC.sh`, `simpleStartStop.sh`) are for the **full Gammasphere system** — they connect to 12 VME crates (IPList: `.141`–`.183`), use `VME10:MTRG`, and use `expName="haha"` with output to `/global/receiver/data/`. The `generateTrigger.sh` there uses `VME99:MTRG:MANUAL_TRIGGER` (a legacy/test prefix not seen in any vme66.cmd). The DuoGe DAQ scripts are in **`/global/tacReceiver/`**, and `simpleStartStop.sh` (in `~/DGS_system/receiver/`) does use the softIOC `Online_CS_*` PVs — those are shared between both systems. ✅ verified 2026-04-30 — `~/DGS_system/receiver/start_run.sh` IPList+`VME10:MTRG`; `generateTrigger.sh` `VME99:MTRG`; `/global/tacReceiver/start_run.sh` confirmed DuoGe-specific (CRATE=66, VME66:, expName="duo").

> ℹ️ **`/global/tacReceiver/basic_settings.sh`** (note: not `basic_settings_DGS.sh`) is a **stale DuoGe single-channel test script** still using the old `VME99:` prefix (the crate's original test-stand naming — `vme66.cmd` header comments say "IOC99 (GRETINA lab test stand)"; the crate was renamed to CRATE=66 / VME66: for production DuoGe use). `basic_settings.sh` programs only a single channel (CH=7) with CFD or LED parameters and is not the current production setup script — use `basic_settings_DGS.sh` instead. ✅ verified 2026-04-30 — `/global/tacReceiver/basic_settings.sh` uses `VME99:MTRG` and `VME99:MDIG1`; `/global/ioc/boot/vme66.cmd:L1` says "IOC99 (GRETINA lab test stand)"; all production DB loads use `CRATE=66`.

> ℹ️ **`/global/tacReceiver/stop_run.sh`** header comment reads "for use on the test stand (VME99, connected to detector GS000, hosted by machine 'slopebox')" — this is historical context from when DuoGe was tested on the slopebox test stand before being moved to tangerine. The script itself is functional (`caput Online_CS_StartStop Stop`, 10 sec wait, `caput Online_CS_SaveData No Save`) and is the standard way to end a run. The receiver shuts itself down on receipt of the type-F (end-of-run) header from each IOC. ✅ verified 2026-04-30 — `/global/tacReceiver/stop_run.sh`.

> ℹ️ **`/global/tacReceiver/legacy/`** contains: older `dgsReceiver.cpp` (same Oberling receiver), `dgsReceiver_Ryan.cpp` (Ryan's fork), `psNet.h`, and `PyReceiver.py` (Python receiver prototype connecting to slopebox `192.168.203.211:9001`, uses `class_DIG` from ANLDAQ for packet parsing). These are archived/development versions — not used in production DuoGe runs. ✅ verified 2026-04-30 — `ls /global/tacReceiver/legacy/`; `PyReceiver.py:L9` (SERVER_IP=192.168.203.211).

> ℹ️ **`/global/tacReceiver/copy2Slopebox.sh`** rsyncs `tcp_Receiver.cpp`, `Makefile`, and `constant.h` to `dgs@slopebox:/global/ioc/dgsReceiver/` — used when syncing receiver source to the slopebox for compilation/testing there. ✅ verified 2026-04-30 — `/global/tacReceiver/copy2Slopebox.sh`.

> ℹ️ **`/global/tacReceiver/kill_IOC.sh`** — kills all processes listed in `pidList.txt` (in the same directory) using `xargs kill -9`. This is the DuoGe-specific receiver kill script — it does NOT loop over a full GS IP list (that commented-out code at the top is legacy). Use this to cleanly kill a stuck `tcp_Receiver` process. `pidList.txt` contains the PIDs of the most recent receiver launch. ✅ verified 2026-04-30 — `/global/tacReceiver/kill_IOC.sh` + `pidList.txt`.

> ℹ️ **`/global/tacReceiver/basic_settings_TACII.sh`** — a TAC-II test stand setup script using **`VME10:MTRG`** prefix (full Gammasphere VME crate 10, NOT DuoGe VME66). It enables MTRG CS, SYSMON, MDIG1 master logic, and sets TRIG_MON_SEL to `SumY`. The script is named for TAC-II (Time-to-Amplitude Converter Mark II) and reflects settings for the full GS environment — do NOT use for DuoGe/vme66. ✅ verified 2026-04-30 — `/global/tacReceiver/basic_settings_TACII.sh` (VME10:MTRG throughout).

> ℹ️ **`/global/tacReceiver/Guceiver/`** — PyQt6 live-display GUI (`Guceiver.py`) for monitoring live waveforms, energy spectra, TAC data, and data rates from an IOC port 9001. The IOC dropdown includes: slopebox (192.168.203.211), dgs8-IOC1/2 (192.168.203.212/213), dgs-IOC1–12 (192.168.203.141–145, 177–183). **vme66 (192.168.203.81) is NOT in the dropdown** — would need to be added manually. This is consistent with Guceiver being a full-GS monitoring tool being repurposed for DuoGe context. Python deps: PyQt6, matplotlib, numpy (see `requirements.txt`). ✅ verified 2026-04-30 — `/global/tacReceiver/Guceiver/Guceiver.py:L34-51`.

> ℹ️ **`/global/tag/DXA_20220720/`** — archived firmware tag directory for the DXA experiment (2022-07-20 snapshot). Contains firmware binaries: `master_digitizer_6194.bin`, `master_trigger_5542.bin`, `router_trigger_4747.bin`, and MCS files (`Digitizer_VME_3963.mcs`, `Trigger_VME_4485.mcs`), plus a `PVs/` subdirectory with DB templates for that era. This is a historical reference snapshot, not the current DuoGe firmware. ✅ verified 2026-04-30 — `ls /global/tag/DXA_20220720/`.

> ℹ️ **Run history:** RunTimestamp.txt in `/global/tacReceiver/data/` shows continuous DuoGe commissioning runs since 2026-04-13. Most recent runs as of 2026-04-29: Run019 (207Bi+multi-source, clean doubles, 16h, 16.7μs decay time), Run020 (207Bi calibration, 5min), Run021 (207Bi 12in away, 2h, 1mm Pb, 52μs decay time). Runs are typically 1–24h; dual-source configurations use 207Bi + 133Ba/152Eu/154Eu/137Cs/60Co/226Ra. ✅ verified 2026-04-30 — `/global/tacReceiver/data/RunTimestamp.txt`.

---

## 9. Data Analysis

### Output Data Format

Data is written as raw binary GEB-format files by `tcp_Receiver` or `dgsReceiver`. GEB ID 14 is used in CFD mode. Files are named `duo_NNN` (tacReceiver) or `haha_NNN` (legacy receiver).

### Receiver Data Scrubbing / Merging

The `/global/receiver/` directory contains:
- `rcvr_data_scrubber` — post-processing / scrubber tool
- `rcvr_merge` — merges multi-receiver output files

### fastEventContructor — Primary Analysis Tool

The primary analysis suite for DuoGe data lives at `~/fastEventContructor/` on tangerine (note: "Contructor" — typo is in the actual directory name). It is a C++/ROOT pipeline for event building and analysis.

#### Pipeline Overview

```
Raw GEB binary files (duo_NNN / haha_NNN)
        ↓
  EventBuilder variant (e.g., EventBuilder_Q or EventBuilder_PQ)
        ↓
  ROOT TTree file (root_data/TAC2_NNN.root)
        ↓
  analyzer.cpp (ROOT macro — histograms, energy spectra, time differences)
```

#### EventBuilder Variants

| Binary              | Description                                         |
|---------------------|-----------------------------------------------------|
| `EventBuilder`      | Original; per-DigiID parallel scan + priority queue |
| `EventBuilder_S`    | Adds scan pre-pass for hit count/timestamp range    |
| `EventBuilder_Q`    | Async double-buffered k-way merge; best for modest data |
| `EventBuilder_PQ`   | Parallel k-way merge with sector partitioning; fastest (12 threads, 4 writers: ~14 s for 16 GB) |
| `EventBuilder_X`    | Same as PQ but outputs Apache Parquet (requires Arrow/Parquet libs) |

All variants (except X) output CERN ROOT TTrees. All accept optional trace saving.

#### Build

```bash
cd ~/fastEventContructor
make                    # build all variants
make EventBuilder_Q     # build only Q
make EventBuilder_PQ    # build only PQ
make EventBuilder_X     # build only X (requires libarrow-dev / PyArrow)
```

#### Usage

```bash
# EventBuilder_Q
./EventBuilder_Q [outfile] [timeWindow_ns] [useTrigTS] [saveTrace] [nWorkers] [file1] [file2] ...

# EventBuilder_PQ (fastest)
./EventBuilder_PQ [outfile] [timeWindow_ns] [useTrigTS] [saveTrace] [nMerge] [nWriters] [file1] [file2] ...

# Example: build all files for run 021, 1000 ns window, use trigger timestamp
./EventBuilder_Q tac2_021.root 1000 1 0 20 $(ls -1 data/TAC2_021/* | grep -v _T)
```

#### ProcessRUN Script

The `ProcessRUN` script automates build + analysis for a named experiment:

```bash
cd ~/fastEventContructor
./ProcessRUN <run_number> [BUILD=1] [ANALYSIS=1]
# Example: build and analyze TAC2 run 21
./ProcessRUN 21
# Skip EventBuilder, run analysis only:
./ProcessRUN 21 0 1
```

- `expName` is hardcoded as `TAC2` in the script — may need editing for DuoGe `duo_NNN` data.
- `timeWin=1000` (ns), `useTrigTS=1` by default.
- Output: `root_data/TAC2_NNN_1.root`

#### Analysis Step (analyzer.cpp)

Run inside ROOT:

```bash
root 'analyzer.cpp("root_data/TAC2_021_1.root")'
```

Produces:
- Per-detector energy spectra (`he001`–`he110`)
- Energy vs Detector ID 2D histogram (`heID`)
- Gamma multiplicity histogram (`hMultiHits`)
- Energy-Energy coincidence matrix (`hEE`, `hGG`)
- Time difference spectra (`hTDiff`, `hGTimeDiff`)
- Reads a **channel map file** via `LoadChannelMapFromFile()` (defined in `misc.h:L13`) — maps `(boardID, channel)` pairs to GS detector numbers
  - File: `GS_channel_map.txt` (must be in the **working directory** when the analyzer/EventBuilder binary is run)
  - Generated by: `findMapping.sh` — queries EPICS PVs (`GSxxx_VME_Index`, `GSxxx_Dig_Index`, `GSxxx_Dig_Channel`, `GSxxx_Ge_ID`, `VMExx:MDIGx:user_package_data_RBV`) for detectors GS001–GS110 and writes the map
  - Format: `detID VME digIdx chID boardID GS_True` (2 header lines skipped)
  - **BGO handling:** `chID + 5` = HPGe channel; `chID` (original) = BGO channel; BGO stored as negative detID in `channelMap`
  - `VMEDIGtoBoard[VME][digIdx]` = boardID (i.e., `user_package_data` value, e.g. 170, 171, 172)
  - If `GS_channel_map.txt` is not found, analyzer prints a warning and continues with empty map
  - A companion `GS_energy_cal.txt` is loaded by `LoadEnergyCalFromFile()` for energy calibration (slope + intercept per detector)

#### Helper Scripts in ~/fastEventContructor/

| Script | Purpose |
|--------|---------|
| `findMapping.sh` | Queries live EPICS PVs (GS001–GS110) to generate `GS_channel_map.txt` |
| `findGS.sh` | Lookup a specific GS number in `GS_channel_map.txt` — prints VME, DIG, Channel, BoardID for that detector |
| `readHexFile.sh` | Hex dump of a raw binary data file |
| `script.sh` | Scratch/history file — commented-out EventBuilder invocations for various past runs (testD, TAC2, haha, etc.) — **not a production script**, just a developer scratch pad |

> ℹ️ **`findGS.sh`** takes a GS number (e.g. `./findGS.sh 62`) and looks it up in `GS_channel_map.txt`, printing VME/DIG/Channel/BoardID. Must be run from the `~/fastEventContructor/` directory (where `GS_channel_map.txt` lives). Normalizes the input to 3 digits with leading zeros. ✅ verified 2026-04-30 — `~/fastEventContructor/findGS.sh`.

> ℹ️ **`script.sh`** is Ryan's scratch pad — a list of commented-out (and a few uncommented) EventBuilder and ROOT analyzer invocations for various test runs (`testD_003`, `testD_004`, `testD_005`, `TAC2_021`, `TAC2_022`, `TAC2_044`, `TAC2_204`, `haha_002`). The only active (non-commented) line runs `EventBuilder_S` on `testD_005` data. This file is useful as a reference for how Ryan invokes the analysis pipeline but should not be run as-is. ✅ verified 2026-04-30 — `~/fastEventContructor/script.sh`.

#### GUI Receiver (Guceiver)

A PyQt6 live-display GUI is available at `/global/tacReceiver/Guceiver/Guceiver.py`. It connects directly to an IOC port (9001) and displays live waveforms, energy spectra, TAC data, and data rates. Supports slopebox (192.168.203.211), dgs8 crates, and dgs-IOC1–12. Not configured for vme66 by default — would require adding a `vme66 192.168.203.81:9001` entry.

#### Receiver Utilities

- `rcvr_data_scrubber` — post-process / scrub receiver output
- `rcvr_merge` — merge multi-receiver output files
- `readHexFile.sh` — read hex dump of binary data files
- `checkTACFile.cpp` — verify TAC file integrity

> **DuoGe expName:** The tacReceiver script uses `expName="duo"` — data files are `duo_NNN` in `/global/tacReceiver/data/`. Older commissioning runs used `expName="haha"` — also in `/global/tacReceiver/data/haha_NNNN/`. The `ProcessRUN` script uses `TAC2` hardcoded — **this needs editing for DuoGe runs** (change `TAC2` → `duo` or `haha` depending on the run). ✅ verified 2026-04-29 — from `/global/tacReceiver/start_run.sh:L5`; `haha` and `duo` both confirmed in `/global/tacReceiver/data/` directory listing ✅ verified 2026-04-30.

**Channel map answer (from source):** The channel map file is `GS_channel_map.txt`, generated by running `findMapping.sh` in `~/fastEventContructor/` (queries live EPICS PVs for GS001–GS110). No pre-generated file exists on tangerine — it is created on-demand when the IOC is running. The analyzer must be run from the directory containing `GS_channel_map.txt`. (`misc.h:L13-43`)

> ⚠️ IMPORTANT: `findMapping.sh` requires `GSxxx_VME_Index`, `GSxxx_Dig_Index`, `GSxxx_Dig_Channel`, and `GSxxx_Ge_ID` PVs to be live. These come from `dgsGlobals_DGS_VME99.db`, but in `vme66.cmd` this load is **commented out** (`##dbLoadRecords("db/dgsGlobals_DGS_VME99.db","CRATE=66")`). Until that DB is loaded (or an equivalent), `findMapping.sh` will produce all `-1` entries and `GS_channel_map.txt` will be empty/invalid. ✅ verified 2026-04-30 — `/global/ioc/boot/vme66.cmd:L106`.
>
> ⚠️ ROOT CAUSE — PV prefix mismatch: `dgsGlobals_DGS_VME99.db` has **1164 hardcoded `VME99:` PV names** — the prefix is not a `$(CRATE)` macro. Passing `CRATE=66` to `dbLoadRecords` does nothing to fix these; all PVs would appear as `VME99:xxx`, not `VME66:xxx`. This is why the DB is commented out for DuoGe. To enable globals for DuoGe, a new `dgsGlobals_DGS_VME66.db` would be needed (either copied + sed-substituted, or regenerated with macros). For reference, full GS has `dgsGlobals_DGS_VME01.db` and `dgsGlobals_DGS_VME02.db` in `~/DGS_system/ioc/db/`. ✅ verified 2026-04-30 — `dgsGlobals_DGS_VME99.db` (grep: 1164 × `VME99:`, 0 × `VME66:`); `find /global /home/dgs ~/DGS_system -name 'dgsGlobals*'`.

**Raw data ID field:** In GEB binary data, the `id` field encodes `boardID * 100 + channel` (e.g., MDIG1 ch5 → id=17005). ✅ verified 2026-04-30 — `README.md` in `~/fastEventContructor/` ("id: boardID×100 + channel").

> ⚠️ **DuoGe GS detector numbers — CONFLICT:** The live setup scripts on tangerine (`setup_HV.sh`, `setup_sbx.sh`, `setup_dig.sh` in `/global/edm/scripts/`) configure **GS033 and GS091**, while the analysis software (`analyzer.cpp:L32-33`) hardcodes `detX=70` (GS070) and `detY=62` (GS062). These may represent different run configurations or the analyzer may be stale. `findMapping.sh` would query live EPICS PVs to generate the actual current channel map. Ask Ryan which detector pair is currently installed.

> Cross-reference: [dgs_analysis.md](dgs_analysis.md) — full DGS analysis tools (fastEventConstructor, parquet_pysort, gray_apps)

---

## Appendix: Quick-Start Checklist

```
[ ] 1. Verify tangerine FTP server is running (⚠️ vsftpd failed since 2026-01-15; `sudo systemctl restart vsftpd`)
[ ] 2. Power on VME crate; vme66 should auto-boot from tangerine
[ ] 3. Check vme66 boots (serial console or ping 192.168.203.81)
[ ] 4. Verify IOC loaded: caget VME66:MTRG:VME_CODE_REVISION
[ ] 5. Launch EDM: edm -m SYS=DUO,VM=66,VR=66,V1=66,V2=67 -x runControl
[ ] 6. Run /global/edm/scripts/setup_dig.sh (digitizer params: LED_Mode, ch enables, CFD windows, preamp reset)
[ ] 7. Run /global/edm/scripts/setup_sbx.sh (SBX gain, DC offsets, BGO HV per PMT, preamp reset monitor)
[ ] 8. Run /global/edm/scripts/setup_HV.sh on (Ge HV to 3500V, BGO HV_CTRL enable)
[ ] 9. Configure trigger thresholds via EDM NewTrigSummaryAndControl.edl
[ ] 10. NOTE: basic_settings_DGS.sh in /global/tacReceiver/ is for FULL GS (VME01–12) — NOT DuoGe; setup_dig.sh (step 6) already handles DuoGe CFD params
[ ] 11. Start tcp_Receiver: ./start_run.sh <N> "comment" [wait_sec]
[ ] 12. Begin run via runControl EDM screen (or caput Online_CS_StartStop Start)
```

---

## See Also

- [overview_DGS.md](overview_DGS.md) — DGS/DuoGe system overview: machine table, network map, VME crate layout
- [ioc.md](ioc.md) — IOC details: boot scripts, startup sequence, DB templates, VxWorks shell
- [fpga.md](fpga.md) — FPGA firmware overview: BUILD_TYPE map, firmware versions, programming flow
- [collectorboxpi_commissioning.md](collectorboxpi_commissioning.md) — Collector box Pi commissioning steps
- [run_procedures.md](run_procedures.md) — Standard run procedures for DGS experiments
- [trig_setup_scripts.md](trig_setup_scripts.md) — Trigger setup scripts (`basic_settings_DGS.sh` and related)
- [DUO_PVs.md](DUO_PVs.md) — DuoGe system PV list
- [dgs_analysis.md](dgs_analysis.md) — Full DGS analysis tools (fastEventConstructor, parquet_pysort, gray_apps)
- [ANLDAQ_tcpReceiver.md](ANLDAQ_tcpReceiver.md) — tcpReceiver / dgsReceiver documentation
- [ANLDAQ_commander.md](ANLDAQ_commander.md) — ANLDAQ run control GUI (commander.py)
- [ANLDAQ_gui_internals.md](ANLDAQ_gui_internals.md) — ANLDAQ GUI internals: PV widgets, class_Board, gui_DataTaking, gui_GS, gui_MTRG, Guceiver classes
- [overview_SmallSystem.md](overview_SmallSystem.md) — DuoGe and X-Array small system architecture overview
