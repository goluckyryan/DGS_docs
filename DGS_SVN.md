# DGS_SVN — Legacy SVN Repository Mirror

## What It Is

A mirror/checkout of the DGS legacy **SVN (Subversion)** repository. This is the historical source of DGS code before the migration to Git.

Contents:
- `dgs/` — Main DGS SVN tree (large, many subdirectories)
- `findFile.sh` — Utility script to search within the SVN tree

---

## Contents of `dgs/`

The SVN tree contains a broad historical archive of DGS development:

| Directory | Contents |
|-----------|----------|
| `17pc030-GretinaTRGT` | Gretina trigger target hardware |
| `17pc031-HeliosPreampPower` | Helios preamp power supply |
| `20180921`, `20230818_edm` | Dated snapshots / EDM displays |
| `con6_20220728` | con6 (Solaris host) snapshot |
| `con6_EPICS_base` | EPICS base from con6 era |
| `con6_work` | con6 working files |
| `daq_system_tags` | **DAQ system version snapshots** — full-system tagged releases, each containing firmware binaries and/or IOC + EDM config trees. Tags: `DFMA_20220711`, `DFMA_20220712` (July 2022 DFMA experiment: DIG=6194, MTRG=5542, RTRG=5814 or 4747); `DFMA_20220729`, `DFMA_20221021` (fuller tags with `firmware/`, `ioc/`, `edm/`, `softIOC/` dirs: DIG=6194, MTRG=5542, RTRG=4747, Digitizer_VME=3963, Trigger_VME=4485); `DUB_20211101` (DUBALL experiment: DIG=6194, MTRG=5542, RTRG=4747); `DXA_20220720` (X-Array experiment: same firmware); `SL6_DGS_20220923` (SL6 = DFMA+DXA combined? — contains all 5 firmware types with `XXXX` placeholder names, suggesting it was a template snapshot). Useful for: recovering exact firmware sets used in past experiments, cross-referencing with run data. |
| `Data_Generator` | **Trigger-chain test injector** — FPGA board that injects programmable discriminator bit patterns into the trigger chain (simulating digitizer hits) via SERDES links. Two firmware variants: `DSSD_MAIN_FPGA/` and `GRETINA_MAIN_FPGA/` (same VHDL, different target system). Contains a CPLD and main FPGA. Key features: 8 SERDES output channels, each with up to 1K discriminator bit patterns loaded via VME RAM tables (16-bit: bits[15:6]=pattern, bits[5:0]=assertion time in clocks); 9th table drives 2 NIM outputs synchronously. Patterns triggered by NIM pulse or Frame12 command (bit15=start now, bit14=start at timestamp rollover). Throttle output can be time-controlled. CPLD drives throttle request lines; main FPGA drives discriminator bits (reversed relative to normal Router pinout). Noted connection quirk: cable swaps SERDES_IN↔OUT and discriminator↔throttle pins. See `DataGenOperationNotes.txt`. |
| `DB_backup_20240205` | **EPICS DB snapshot (Feb 5, 2024)** — templates and per-crate `.db` files as they existed before the Git migration. Templates: `asynDebug.template`, `daqCrate.template`, `daqSegment2.template`, `MDigRegisters.template`, `MDigUser.template`, `MTrigRegisters.template`, `MTrigUser.template` (+ `.TMP` draft), `RTrigRegisters.template`, `RTrigUser.template`, `SDigRegisters.template`, `SDigUser.template`, `RunProtect.asf`. Per-crate `.db` files: `dgsGlobals_DGS_VME01.db` through `VME12.db` + `VME99.db`. **Notable difference vs current `ioc/db/`:** VME-variant templates (`MDigRegistersVME`, `MDigUserVME`, `SDigRegistersVME`, `SDigUserVME`) did NOT exist in Feb 2024 — added later for crates 66 and 99 to expose VME FPGA status PVs (power/voltage/temp sensors, clock select, `vme_code_revision_RBV`, `SERIAL_NUMBER_RBV`). `asynDebug.template` removed from ioc/db/ in Git version. |
| `Detector_Repair` | Detector repair records/tools |
| `devel8`, `dgs_devel8` | Development branch 8 |
| `devel_tracker` | Development tracking |
| `DGS1_clean_folders` | DGS1 (old system) clean directory tree |
| `DGS1_total_backup` | Full DGS1 backup |
| `dgsext` | DGS extensions |
| `DGSFiberExpander` | **VME Fiber Adapter / Fiber Expander** — PCB #3174 (ANL part `21pc032`, Rev A, Sept 2021). Adapts VME backplane fiber connections to front-panel SFP/fiber connectors. Contains OrCAD schematics, Allegro layout, Gerber fab package, BOM, DRC, and front panel drawings. Vendor quote from Q74173A1 / QT-2873. Likely used to extend SERDES fiber links between crates or VME chassis. |
| `dgsSoftIOC` | Soft IOC (pre-Git version) |
| `ArdisiaDocuments.zip` | Ardisia documentation |
| `DigitizerFanout` | **Digitizer Fanout PCB** (Aug 2022): RJ45-based LVDS signal fanout board. BOM includes dual LVDS receivers (SN65LVDT9637B ×3), LVDS drivers (FIN1001M5X ×2), clock buffers (NB6N11SMNG ×2), OR/NOR logic gates, RJ45 jacks (Amphenol RJHSE5387 ×2 + vertical male plugs ×4), RJ11 connector ×1. Purpose: fan out SERDES/LVDS differential signals across RJ45 links — likely used to split digitizer trigger/data links to multiple destinations. OrCAD schematic + Allegro layout + BOM + DRC included. |
| `Digitizer_Tester` | Digitizer Tester module (generates test waveforms) |
| `Documentation` | General DGS documentation |
| `edmhelp`, `edm-master`, `20230818_edm` | Legacy EDM screen utilities and dated snapshots |
| `screens/` | **Legacy EDM control screens** (141 `.edl` files): full DGS operator GUI before PyQt migration — includes `BigSummary.edl`, `CAGMrunControl.edl`, `BGSrunControl.edl`, IOC status, BGO rates, CLO (collector) channel/board/global screens, trigger lock, digitizer enable, analog GS, aux I/O control, acq status. Historical reference; current GUI is PyQt6 (`ANLDAQ/`). |
| `sbxscreens/` | **SBX EDM screens + commissioning scripts**: 20 `.edl` files (CollectorBox, PowerBoard, PickoffCtl, PickoffStatus, PreampData, SlopeBox_CTL, SlopeBoxrunControl, SlopeBoxSelect, RAM_Buffer, dgsTrigCntrl, etc.) + `PV.list` (201-line doc of CAMAC I/O record types: bi/bo, mbbi/mbbo, ai/ao with ANDmask, shift-factor, read-modify-write semantics) + `Std_Test.sh` (SBX cross-test setup script by JTA 2021-04-02: sets ScanMode=3, Ge/BGO HV off, DAC offsets=150, Ge threshold=600, then BGO HV0-13 all to 180, enables HV, sets `PARST_AutoClampDwell=10000`) + `D0_Std_Test.sh` / `D99_Std_Test.sh` (same for detector 0 and 99). Nominal BGO HV: **180 DAC units** per tube (from Std_Test.sh). |
| `FromT` | **Hardware files from Todd (trigger system history)**: contains PCB design packages, bitfiles, schematics, and legacy trigger docs. Key items:<br>• `06pc057d/` — PCB #06pc057 (likely early MTRG/RTRG carrier or adapter board); Gerber, BOM, placement files<br>• `12pc006a FLIC-FTK/` — FLIC-FTK (Fiber Link Interface Card — Front-end Trigger module, 2012); Gerbers, front panel DWG, BOM<br>• `12pc007b-FLIC-RTM/` — FLIC-RTM (Rear Transition Module for FLIC); Gerbers, front panel DWG<br>• `13pc003a Myriad/` — MγRIAD PCB (2013); Gerbers + Allegro board files (also see `dgs/myriad.md`)<br>• `17pc001a-ProtoDUNE-SSP/` — ProtoDUNE SSP (Signal Shaping Processor) hardware from 2017; not DGS-specific<br>• `Scarlet Trigger/` — **Pre-DGS multi-crate CAMAC trigger system** (2005) by John Weizeorick: PC + CAMAC-based busy/trigger fanout over RS485/RJ45; master→slave architecture; FPGA bitfiles (`m1trg.bit`, `strg.bit`) for Mesa 5i20 I/O card + accompanying VHD source; manual included. Historical predecessor to the DGS MTRG/RTRG FPGA trigger.<br>• `sspCABLE/` — SSP cable pinout CSVs<br>• `PrintCapture` software + `AUTORUN.INF` — Windows oscilloscope print-capture utility (not DGS-specific)<br>• Various `.DAT`/`.IVX`/`.LBL` files — likely OrCAD/Cadence netlist exports for other PCB designs |
| `GRETNA_CPLD_CHECK` | GRETNA CPLD check utilities |
| `gtReceiver` | Legacy C++ data receiver (pre-ANLDAQ); `dgsReceiver/dgsReceiver.cpp` v6.57 by M. Oberling. Writes GEB format (types 14=DGS, 15=DGSTRIG). Also contains `rcvr_merge`, `rcvr_unmerge`, `rcvr_data_scrubber` utilities. Full GEB type table now in `dgs_analysis.md`. |
| `HELIOS_Preamp_data` | HELIOS preamp data |
| `LBL_Digitizer` | Lawrence Berkeley National Lab digitizer firmware images — binary bitfiles (`chip_top_2.00_009D.bin`, `chip_top_2.00_006e.bin`) and VME MCS file (`vme-30-41.mcs`) from Dec 2014. Historical reference; ANL developed its own digitizer firmware independently. |
| `Miscellany` | Miscellaneous files — contains `MSRDPCLI.EXE` / `MSRDPCLI(1).EXE` (Windows RDP client), `tightvnc-1.3.10-setup.exe` (TightVNC installer). Likely dropped here for remote-access convenience; no DGS-specific content. |
| `MyRIAD` | MγRIAD module files |
| `NewBlackBox` | New black box hardware — contains `RaspberryPi/PreEpicsCode/`: pre-EPICS collector box SPI code (bcm2835 library, `extract_Eprom_b.c`, `Scan_DVI_Power.c`, GPIO/SPI tests); predecessor to current `collectorboxpi/` EPICS soft IOC |
| `NS_scripts` | BGO tuning + PV extraction scripts (see `dgs/utility_scripts.md`): `BGO_tune_v2.py`, `extract_PV.py`, `GS_nums.py`, PV list txt files |
| `OrCAD` | **OrCAD component library**: `MBO_LIB.OLB` — Cadence OrCAD compound document library file (part symbols/schematic components). Likely used across multiple PCB designs in this SVN archive. |
| `Schematics` | **PCB schematics directory**: two designs — `PaddleCard/` (SFP fiber paddle card: OrCAD `TRIG_PADDLE_SFP.DSN`/`.opj` + Allegro layout; likely the SFP fiber interface board between trigger crates) and `Trigger_IO_Adapter/` (trigger I/O adapter: `TRIGGER_IO_ADPATER.DSN`/`.opj`, `AUX_IO_Pin_Mapping.xlsx` spreadsheet documenting aux I/O pin assignments, OrCAD library + parts BOM). |
| `SchematicTagsForTodd` | Empty directory — was likely used to tag schematic versions for Todd (trigger hardware lead); no files present in this SVN snapshot. |
| `Patch_Panel` | **Signal patch panel PCBs**: OrCAD schematics + Allegro layout for two boards — PCB `11pc015` (GS input patch, Rev A) and PCB `11pc016` (front+rear panel, Rev A); `PropagationSheet.xlsx` (signal routing/propagation reference); `Pictures/` (board photos). These are the analog signal patch panels routing detector signals from the Gammasphere array to the digitizer inputs. |
| `psg` | **PSG — Code-Generating Spreadsheet system**: reads CSV register map files and auto-generates EPICS `.db` records + CSS/EDM screens. Contains `CodeGeneratingSpreadsheetGeneric/Projects/DGS_Rtrig/fromdgs1.txt` (8,392-line generated EPICS DB output for the Router Trigger — all `asynUInt32Digital` mbbo/mbbi/ao/ai/bi/bo records). Config in `salvaged_notes/dgs_config.txt` references CSV sources: `DGSMasterTriggerRegisterMap.csv`, `DGSRouterTriggerRegisterMap.csv`, `MasterDigitizerRegisterMap.csv`. The CSV files and generator Python script itself are not present in this SVN snapshot. Historical precursor to the `Auto-generate EPICS DB` task (see QUEUE.md). |
| `salvaged_notes` | Recovered notes from old systems |
| `sbxl_osc` | **Early Pi-based SBX/pickoff IOC dev notes**: `diagdump.txt` (EPICS boot trace for `PickoffApp_RevC` on Pi, CA ports 5074/5075 for test stand), `EPICS boot trace 20201028.txt` + `EPICS_init_trace_20200911` (IOC startup logs), `Breakdown of globals.ods` (spreadsheet of global variables), `NonEpics_SPI/` (direct SPI access without EPICS), bcm2835-1.63 library, `NotesOnVPNandSVN.txt` (VPN/SVN setup for Pi). Predecessor to current `collectorboxpi/` repo. |
| `sbxscreens` | SBX EDM screens + commissioning scripts (see main table above) |
| `SlopeBoxExtension` | SBX hardware + firmware + GS_ID dongle (→ `sbx.md`) |
| `SlopeBoxInterface` | Slope box interface hardware: schematics, PCB layout, pickoff card design, power budget lab notes, RPi interface. `Documentation/LabNotes/SlopeBoxCurrentDraw.txt` (2018-05-07, JTA) measured per-rail current draw (no detector connected): +12V=105mA (1.26W), -12V=113mA (1.36W), +24V=21mA (0.5W), +5V=81mA (0.4W) → total ~3.5W idle, ~5.7W with HV on. Estimated total per-detector (slope box + Pickoff Rev B + RPi): ~22W → requires 802.3at PoE (30W/port); 802.3af (15.4W/port) insufficient. Power supply recommendation: CUI PYB30-Q48-T512 converter. |
| `slopebox_scripts` | BGO HV sweep + counter averaging scripts: `BGO_Sweep_test` (sweeps `GS000_BGO_HV0..13` from 0→250 DAC in steps, averages `GS000_BGO[1-7]Sum_counter`), `caget_avg` (shell: reads PV N times, averages numeric value), `Avg_all_BGO_count` (reads all 8 BGO counter PVs via `caget_avg`). Run from `/global/devel/scripts/`. Legacy — see `utility_scripts.md` for current Python equivalents. |
| `TrigToTrig` | Trigger-to-trigger interconnect PCB (PCB#1446, Rev A, June 2021): GRETINA Trigger Module SERDES I/O; OrCAD schematics + BOM + DRC; Allegro layout; fab package `21KB001` |
| `VXI_database` | **Legacy analog Gammasphere EPICS database** (6 files, ~192K lines total, ~2,233 records each): `resm1.db`–`resm6.db` — pre-DGS VXI-based DAQ EPICS records for the original analog Gammasphere system. PV prefix conventions: `MOD###_` (module/digitizer), `BGO###_` (BGO detector), `VXI###_` (VXI crate channel). Record types: mbbo/mbbi, ai/ao, bi/bo, sub, waveform. Key PV groups: `BGO###_CS_PAT[1-7]` (BGO coincidence pattern bits), `BGO###_CS_DB[1-7]` (delay buffer settings), `BGO###_CS_100KV` (HV control), `BGO###_DS/DV_LRE/TDC` (leading-edge/TDC status/value), `VXI###_DV_GE_CFD/PUD/SIDE` (Ge CFD, pole-up/down, side settings), `MOD###_DV_EN` (module enable), `MOD###_HV_CON` (HV control sub record). Historical reference only — predates DGS digitizers. |
| `signoise.run/` | **Signal integrity simulation run directory** — SIS/Cadence SigXplorer project stub (`cases.cfg` + empty `case1/`). Likely a placeholder or leftover from PCB signal integrity analysis on one of the DGS PCBs. No simulation data present. |
| `VxWorksDocs/` | **VxWorks/Tornado 2 reference documentation archive**: `hostLib.html` (host library API), `msgQLib.html` + `msgQShow.html` + `msgQSmLib.html` (message queue APIs), `sockLib.html` (socket library); `Tornado-getStart.pdf` (Getting Started guide), `Tornado-Guide.pdf` (Tornado 2 IDE guide), `Tornado-Releasenotes.pdf` (release notes), `Vx-Progr-Guide1.pdf` (VxWorks Programmer's Guide). Useful when reading IOC/VxWorks source code — documents the APIs used in `inLoop.st`, `SendReceiveSupport.c`, etc. |

---

## `salvaged_notes/` — Key Historical Documents

Recovered notes and config files from old systems. Explored 2026-04-06.

| File | Contents |
|------|----------|
| `DGS_systemdef.txt` | VME crate to board assignment map: VME01-12, each with MDIG1/SDIG1/MDIG2/SDIG2 pairs (VME06 and VME10 have only MDIG1/SDIG1 — 2 digitizers instead of 4). Total: 10 crates × 4 DIGs + 2 crates × 2 DIGs = **44 digitizer boards × 10 ch = 440 channels** (matches ~110 GS holes × 4 signals: 1 Ge + 3 BGO) |
| `dgs_config.txt` | Python config for DB/screen generation scripts. Defines CSV register map sources (`DGSMasterTriggerRegisterMap.csv`, `DGSRouterTriggerRegisterMap.csv`, `MasterDigitizerRegisterMap.csv`), VME crate/slot assignments (MTRG in crate 32 slot 3; VME01-12 with MDIG/SDIG in slots 3-6), and EPICS runtime path (`/global/devel`). Historical precursor to current IOC config. |
| `FW_MAINT_README.txt` | Firmware flashing note: refers to old Java-based EPICS client (`flashdgs.js`, `runJEpics`) on con6 -- historical, superseded by current Python/EPICS approach |
| `how_to_fw.txt` / `how_to_fw2.txt` | **2013 firmware flashing session logs** (terminal recordings from `dgs1` Solaris host). Shows `epics.sendFpga(digware, fn1, crate, slot, ...)` JavaScript API sending 1.66 MB bitfiles via EPICS to `VMExx:DBG:fpga_erase/program/verify_RBV` PVs. Also shows `checkfpgas.py` output (2013-08-30) confirming all VME01–VME11 DIG boards at rev 0xB.0 (date 0x20130808), except VME08:DIG3 = 0xFFFFFFFF (bad/unresponsive board). **Key historical fact:** PV naming in 2013 was `VME01:DIG1:code_date_RBV` / `mjr_code_revision_RBV` / `min_code_revision_RBV` — the `DIG1`/`DIG3` prefix vs current `MDIG1`/`SDIG1` prefix. Also: VME32 MTRG/RTRG PVs (CODE_DATE, CODE_REVISION) were not yet connected in 2013. |
| `NotesOnInloop.txt` | Detailed analysis (JTA) of `inLoop.st` state machine PV hooks and queue usage: `SCAN_FOR_DATA` calls `transferDigFifoData()` and `DigitizerTypeFHeader()` with type codes (0=empty, 1=end-of-run, 2=error). Confirms `TypeOfBoard` (board_type) field is populated but not used by receiver. |
| `20180924_notes.txt` | Architecture notes on FIFO readout hierarchy: `inLoop.st` -> `checkDigFIFO()` (in `readDigFIFO.c`) -> `serviceOneBufferDig()`. Documents `rawEvt` struct fields (`board`, `len`, `refcount`, `curptr`, `bptr`, `owner`); queue sizes: `RAW_Q_SIZE=400`, `RAW_BUF_SIZE=512KB`. |
| `jta_notes_20200501.txt` | JTA notes on VxWorks build environment: `munch` files, EPICS base paths (`/global/devel/base`, `/global/devel/supTop`), EPICS base versions 3.14.10 and 3.14.12.1. Historical context for vxWorks migration. |
| `ChangeNotes_20220801.txt` | **Key design decision (Aug 2022):** Eliminated requirement that trigger data must look like digitizer data. Updated `SendReceiveSupport.c` and receiver (`dgsReceiver`). Non-digitizer data dumped to `<run_name>_DIAG` file. Type `0xFF` = unspecified (backwards compat); type `0x00` = normal digitizer. Trigger data has GEB type 15=DGSTRIG. Authors: M. Oberling + J. Anderson. |
| `WikiExtract.txt` | Firmware flashing procedure (historical): log into dgs1, get `*.bin` from `/Digitizer/MAIN_FPGA/Work11_DGS` (4 flavors: MSTR_digitizer, SLAVE_digitizer, trigger_top, router_top), copy to `/home/dgs/tmadden/DGSDigFirmware`, then run `java -classpath jca-2.3.5.jar:caj-1.1.9.jar:fpgasender.jar plotControl`. Historical Java-based flashing tool (`fpgasender.jar`), superseded by current Python/EPICS approach. |
| `DecodingTheMakefile.txt` | Notes on EPICS makefile traversal in the con6 build environment |

---

## `findFile.sh`

A utility to locate files within the large SVN tree by name pattern.

```sh
./findFile.sh <pattern>
```

---

## Usage / Status

This is primarily an **archive** and **reference** — not actively developed. The active development has migrated to Git repositories (fpga/, ioc/, vxworks/, etc.).

Useful for:
- Finding historical implementations
- Recovering old configuration files
- Understanding the DGS1 → DGS2 migration
- Looking up Solaris/con6-era build environments

---

## Connections to Other Subsystems

- **vxworks/** — `migration.md` references the con6/Solaris-to-Ubuntu migration this SVN contains
- **fpga/** — FPGA firmware may have early versions in the SVN tree
- **ioc/** — pre-Git IOC files are in `dgsSoftIOC/`
- **sbx.md** — `SlopeBoxExtension/` explored: GS_ID dongle, BGO HV map, pickoff card notes
- **ANLDAQ.md** — `gtReceiver/` is the legacy receiver; current receiver is in ANLDAQ git repo
