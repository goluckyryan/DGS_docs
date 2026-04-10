# ioc — EPICS IOC Configuration & Firmware Repository

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
| `daqSegment.template` | `daqSegment2.template` | Per-channel user PVs (10 per board) |
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
| `BUS_LEFT.bin` | DIG Main FPGA | **Master digitizer** firmware (MDIG1, slot 3, board 0) — clock sourced from SERDES link; drives the inter-DIG front bus clock to slave ✅ verified 2026-04-08 — `uploadFW.cmd:L17` (board 0=MDIG1) + `Digitizer.vhd:L354-355` ("SERDES is external clock source in master digitizer") |
| `BUS_RIGHT.bin` | DIG Main FPGA | **Slave digitizer** firmware (MDIG2, slot 4, board 1) — clock sourced from front bus (driven by master DIG); sends throttle/lock status back to master via SDATA ✅ verified 2026-04-08 — `uploadFW.cmd:L22` (board 1=MDIG2) + `Digitizer.vhd:L968-970` (slave SDATA signals) |
| `trigger_top.bin` | MTRG Main FPGA | Master trigger |
| `V4747_mod_router_top.bin` | RTRG Main FPGA | Router |
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
- B0–B6 map slot numbers to board names (or `X` = empty slot)
- `inLoop` uses these to form PV names like `MDIG1_CS_Ena` for readout control (e.g. `B3=X` → PV `X_CS_Ena`)
- The slot index in `BN=` must match the physical VME slot (0-indexed from slot 1)
- Example: `B0=MDIG1,B1=MDIG2,B2=X,B3=X,B4=X,B5=MTRG,B6=X` → MDIG1 in slot 1, MDIG2 in slot 2, MTRG in slot 6

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
| `COINC_TRIG_MASK_A1..A7/B1..B7` (×110 detectors) | bo/bi | Coincidence trigger mask: which detectors participate in each of the 14 coincidence groups. **This is the bulk of the template — ~7,000 unique PVs (14 groups × ~110 det × 2 for RBV).** |
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

**Size breakdown** (approximate):
- `COINC_TRIG_MASK_*` records: ~7,000 PVs (dominates file size)
- Per-bit RAM records (`VETO_RAM_*`, `TRIG_RAM_*`, `SWEEP_RAM_*`): ~200 PVs
- Other functional PVs: ~350 unique names

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

**Terminal server port formula** (from `ANLDAQ/gui/commander.py`):
```
telnet <TERMINAL_SERVER_IP> <2000 + IOC_id>
```
Where `IOC_id` = index of the IOC in the DAQ list (1-based).

### Terminal Server Assignments

| System | Terminal Server IP | IOC Ethernet IP | Telnet Port |
|--------|-------------------|-----------------|-------------|
| DuoGe (DUO) | 192.168.203.54 | 192.168.203.81 (vme66) | 2000 + id |
| X-Array (DXA) | 192.168.203.47 | 192.168.203.212, .213 | 2000 + id |
| SlopeBox (test) | 192.168.203.139 (ts99) | vme99 | 2003 (id=3) |
| DGS crates 1–6 | 192.168.203.186 (gs-ts-south, even GS IDs) | .141–.145, .177–.183 | 2000 + id |
| DGS crates 7–12 | 192.168.203.91 (gs-ts-north, odd GS IDs) | (same pool) | 2000 + id |

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
