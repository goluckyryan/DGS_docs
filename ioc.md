# ioc — EPICS IOC Configuration & Firmware Repository

## What It Is

The **IOC (Input/Output Controller) repository** — contains all files needed to bring up a DGS VME crate:
- EPICS database files (PV definitions)
- Boot scripts (VxWorks startup commands)
- Firmware binaries (DIG, RTRG, MTRG)
- The VxWorks image and munch binary

Uses **Git LFS** for large binaries (`.munch`, `.bin`, `vxWorks`).

> **Current commit target:** TAC2 + Trigger Hold-Off configuration

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

| System | CA Server Port | CA Repeater Port |
|--------|---------------|------------------|
| DGS | 5064 | 5065 |
| DFMA | 5068 | 5069 |
| Xarray | 5072 | 5073 |
| SlopeBox | 5074 | 5075 |
| DUB | 5078 | 5079 |
| DuoGe | 5080 | 5081 |

NTP server: `192.168.203.56` ✅ verified 2026-04-06 — `ioc/boot/cdCommands:L23` | Timezone: CDT (UTC-6, `EPICS_TS_MIN_WEST=360`) ✅ verified 2026-04-06 — `ioc/boot/vme99.cmd:L52`

---

## Firmware Files (firmware/)

| File | Board | Description |
|------|-------|-------------|
| `BUS_LEFT.bin` | DIG Main FPGA | Front bus sender role |
| `BUS_RIGHT.bin` | DIG Main FPGA | Front bus receiver role |
| `trigger_top.bin` | MTRG Main FPGA | Master trigger |
| `V4747_mod_router_top.bin` | RTRG Main FPGA | Router |
| `DIG_VME_FPGA_20220729.mcs` | DIG VME FPGA | VME interface (Jul 2022) |
| `MTRG_VME_FPGA_20250711.mcs` | MTRG VME FPGA | VME interface (Jul 2025) |

_✅ All firmware filenames verified 2026-04-06 — `ls ioc/firmware/`_

**Firmware upload sequence** (`uploadFW.cmd`):
1. `ProgramFlash(board#, 0, "file.bin")` — writes firmware to flash
2. `ConfigureFlash(board#, 0)` — loads flash into FPGA
3. Run `ConfigureFlash` again for all boards to confirm

---

## Boot Script Details

### vme66.cmd (DuoGe — CRATE=66)

Crate layout:
```
Slot 1: IOC (MVME5500)
Slot 3: MDIG1 (Board #0)
Slot 4: MDIG2 (Board #1)
Slot 6: RTR1  (Board #4)
Slot 7: MTRG  (Board #5)
```

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

**`inLoop` B-parameter syntax:** `seq &inLoop,"CRATE=NN,B0=MDIG1,B1=MDIG2,...,B5=MTRG,B6=X"`
- B0–B6 map slot numbers to board names (or `X` = empty slot)
- `inLoop` uses these to form PV names like `MDIG1_CS_Ena` for readout control
- The slot index in `BN=` must match the physical VME slot (0-indexed from slot 1)
- Example: `B0=MDIG1,B1=MDIG2,B2=X,B3=X,B4=X,B5=MTRG,B6=X` → MDIG1 in slot 1, MDIG2 in slot 2, MTRG in slot 6

**User package data formula:** `[(crate# - 1) × 4] + 101 + board#`
- Master trigger always = 150
- Routers: RTR1=151, RTR2=152, etc.

**DB naming convention:**
- Register-level: `VME<CRATE>:<BOARD>:<register>` (e.g. `VME66:MDIG1:reg_led_threshold`)
- BOARD = portName = first arg of asynXxxConfig()

---

## Boot Scripts

Located in `boot/`:
- `vme66.cmd` — DuoGe crate (CRATE=66); uses `cdCommands` (CA port 5080/5081 DuoGe)
- `vme99.cmd` — GRETINA lab test stand (CRATE=99); uses `cdCommandsLab` (CA port 5074/5075 G-wing)
- `cdCommands` — paths + EPICS CA env for DuoGe system
- `nfsCommands` — NFS mount: `nfsAuthUnixSet("fs.gam", 6000, 10, 0, 0)`

**Key differences between vme66 and vme99:**
- vme66: loads `daqCrate.template` + NFS globals commented out; `cdCommands` (DuoGe port)
- vme99: loads `daqCrate.template` + `dgsGlobals_DGS_VME99.db`; `cdCommandsLab` (G-wing/test port)
- vme99: two MDIG boards both use `MDigRegisters/User` (master-type DB); vme66: MDIG1=master, MDIG2=slave (`SDigRegisters/User`)
- Both: `asynDebug.template` line present but commented out
- Regular VME01–12 (Gammasphere) boot scripts live on NFS at `/global/ioc/boot/` — not in this git repo

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
| `daqCrate.template` | Per-crate | Crate-level status |
| `daqSegment2.template` | Per-segment | Segment-level DAQ status |

**Note:** The four `*VME.template` files were added after Feb 2024 (not in `DB_backup_20240205`). They expose VME FPGA internals for crates 66 (DuoGe) and 99 (test stand).

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

**Output:** `ioc/All_PV.json` — 13,887 entries (as of 2026-04-06)

**When to regenerate:** after changing IOC boot scripts, adding/removing DB templates, or modifying PV names

```bash
cd ioc && python3 findAllPV.py
```

> **Note:** This satisfies the ANLDAQ GUI's PV discovery need. The open QUEUE task ("Auto-generate EPICS DB + PV list from FPGA register map") aims to go further upstream — generating the `.template` DB files themselves from the FPGA register map, so `findAllPV.py` can then derive `All_PV.json` automatically.

---

## Connections to Other Subsystems

- **vxworks/** — provides the `gretDet.munch` binary stored in `bin/`
- **fpga/** — provides the `.bin` firmware files stored in `firmware/`
- **ANLDAQ** — reads/writes the EPICS PVs defined in `db/` and `dbd/`; uses `findAllPV.py`-generated `All_PV.json` for PV discovery
- **collectorboxpi/** — parallel IOC for collector box hardware (different platform)
- **EPICS_asyn.md** — asyn driver internals: caput/caget flow diagrams, port concept, asynUInt32Digital
- **collectorbox_devicesupport.md** — EPICS device support for collector box (SPI, CAMAC_IO)

---

## Operational Notes

- Git LFS required: `git lfs pull` after clone to get firmware/munch/vxWorks binaries
- Firmware version table is the ground truth for what IOC configuration is active
- Multiple VME crates share the same munch binary but have per-crate boot scripts (`vme66.cmd`, `vme99.cmd`, etc.)
- After updating firmware: update the `ioc/firmware/` binaries, update the README version table, update boot scripts if needed
