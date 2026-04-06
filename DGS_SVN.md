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
| `daq_system_tags` | DAQ system version tags |
| `Data_Generator` | **Trigger-chain test injector** — FPGA board that injects programmable discriminator bit patterns into the trigger chain (simulating digitizer hits) via SERDES links. Two firmware variants: `DSSD_MAIN_FPGA/` and `GRETINA_MAIN_FPGA/` (same VHDL, different target system). Contains a CPLD and main FPGA. Key features: 8 SERDES output channels, each with up to 1K discriminator bit patterns loaded via VME RAM tables (16-bit: bits[15:6]=pattern, bits[5:0]=assertion time in clocks); 9th table drives 2 NIM outputs synchronously. Patterns triggered by NIM pulse or Frame12 command (bit15=start now, bit14=start at timestamp rollover). Throttle output can be time-controlled. CPLD drives throttle request lines; main FPGA drives discriminator bits (reversed relative to normal Router pinout). Noted connection quirk: cable swaps SERDES_IN↔OUT and discriminator↔throttle pins. See `DataGenOperationNotes.txt`. |
| `DB_backup_20240205` | Database backup (Feb 2024) |
| `Detector_Repair` | Detector repair records/tools |
| `devel8`, `dgs_devel8` | Development branch 8 |
| `devel_tracker` | Development tracking |
| `DGS1_clean_folders` | DGS1 (old system) clean directory tree |
| `DGS1_total_backup` | Full DGS1 backup |
| `dgsext` | DGS extensions |
| `DGSFiberExpander` | **VME Fiber Adapter / Fiber Expander** — PCB #3174 (ANL part `21pc032`, Rev A, Sept 2021). Adapts VME backplane fiber connections to front-panel SFP/fiber connectors. Contains OrCAD schematics, Allegro layout, Gerber fab package, BOM, DRC, and front panel drawings. Vendor quote from Q74173A1 / QT-2873. Likely used to extend SERDES fiber links between crates or VME chassis. |
| `dgsSoftIOC` | Soft IOC (pre-Git version) |
| `ArdisiaDocuments.zip` | Ardisia documentation |
| `DigitizerFanout` | Digitizer fanout hardware |
| `Digitizer_Tester` | Digitizer Tester module (generates test waveforms) |
| `Documentation` | General DGS documentation |
| `edmhelp`, `edm-master`, `sbxscreens`, `20230818_edm` | EDM screen files |
| `FromT` | Trigger hardware files from Todd (trigger card schematics etc.) |
| `GRETNA_CPLD_CHECK` | GRETNA CPLD check utilities |
| `gtReceiver` | Legacy C++ data receiver (pre-ANLDAQ); `dgsReceiver/dgsReceiver.cpp` v6.57 by M. Oberling. Writes GEB format (types 14=DGS, 15=DGSTRIG). Also contains `rcvr_merge`, `rcvr_unmerge`, `rcvr_data_scrubber` utilities. Full GEB type table now in `dgs_analysis.md`. |
| `HELIOS_Preamp_data` | HELIOS preamp data |
| `LBL_Digitizer` | Lawrence Berkeley National Lab digitizer firmware images — binary bitfiles (`chip_top_2.00_009D.bin`, `chip_top_2.00_006e.bin`) and VME MCS file (`vme-30-41.mcs`) from Dec 2014. Historical reference; ANL developed its own digitizer firmware independently. |
| `Miscellany` | Miscellaneous files |
| `MyRIAD` | MγRIAD module files |
| `NewBlackBox` | New black box hardware — contains `RaspberryPi/PreEpicsCode/`: pre-EPICS collector box SPI code (bcm2835 library, `extract_Eprom_b.c`, `Scan_DVI_Power.c`, GPIO/SPI tests); predecessor to current `collectorboxpi/` EPICS soft IOC |
| `NS_scripts` | BGO tuning + PV extraction scripts (see `dgs/utility_scripts.md`): `BGO_tune_v2.py`, `extract_PV.py`, `GS_nums.py`, PV list txt files |
| `OrCAD`, `Schematics`, `SchematicTagsForTodd` | Schematic files |
| `Patch_Panel` | Patch panel hardware |
| `psg` | PSG (pulse/signal generator?) |
| `salvaged_notes` | Recovered notes from old systems |
| `sbxl_osc`, `sbxscreens` | SBX oscilloscope + screen files |
| `SlopeBoxExtension` | SBX hardware + firmware + GS_ID dongle (→ `sbx.md`) |
| `SlopeBoxInterface` | Slope box interface files |
| `slopebox_scripts` | Slope box utility scripts |
| `TrigToTrig` | Trigger-to-trigger interconnect PCB (PCB#1446, Rev A, June 2021): GRETINA Trigger Module SERDES I/O; OrCAD schematics + BOM + DRC; Allegro layout; fab package `21KB001` |
| `VXI_database` | Legacy VXI system database |
| `VxWorksDocs` | VxWorks/Tornado 2 reference documentation archive: `hostLib.html`, `msgQLib.html`, `sockLib.html`; Tornado Getting Started + Guide + Release Notes PDFs; VxWorks Programmer's Guide PDF |

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
