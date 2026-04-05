# DGS Hardware Drawings Index

_Created 2026-04-04. All paths relative to `/home/ryan/DGS_tools_pack/`_

---

## Digitizer (DIG)

| Document | File | Location |
|----------|------|----------|
| Schematic Rev 4.1 | `31Y334-Schematic-10ChanDigitizer-Rev4.1.pdf` | `DGS_docs/DGS_System_Documentation/Datasheets/digitizer/` |
| Schematic Rev 4.2 | `31Y334-Schematic-10ChanDigitizer-Rev4.2.pdf` | `DGS_docs/DGS_System_Documentation/Datasheets/digitizer/` |
| Firmware doc (expert) | `ANL Digitizer Firmware for Experts.pdf` | `DGS_docs/DGS_System_Documentation/Firmware/Digitizer/` |
| Firmware doc (LBL version) | `ANL Firmware for LBL Digitizer.pdf` | `DGS_docs/DGS_System_Documentation/Firmware/Digitizer/` |
| Register map (master) | `Master Digitizer Register Map.xls` | `DGS_docs/DGS_System_Documentation/Firmware/Digitizer/` |
| Register map (slave) | `Slave Digitizer Register Map.xls` | `DGS_docs/DGS_System_Documentation/Firmware/Digitizer/` |
| Register map w/ VBA db generator | `Dig reg map with vba.xls` | `DGS_docs/DGS_System_Documentation/Firmware/Digitizer/` |
| Register map CSV | `MDRM.csv` | `DGS_docs/RegisterMaps/` |
| Header format doc | `Header Format.docx` | `DGS_docs/DGS_System_Documentation/Firmware/Digitizer/` |
| Header format (current) | `Digitizer Header Formats in Digital Gammasphere.docx` | `DGS_docs/DGS_System_Documentation/System/` |
| Module specification | `Digitizer-Specification-RevA-v2 0.pdf` | `DGS_docs/DGS_System_Documentation/Modules/` |
| ADC datasheet | `AD6645.pdf` | `DGS_docs/DGS_System_Documentation/Datasheets/digitizer/` |
| FIFO datasheet | `IDT_72V2103-72V2113_DST_20100601.pdf` | `DGS_docs/DGS_System_Documentation/Datasheets/digitizer/` |

---

## Master Trigger (MTRG)

| Document | File | Location |
|----------|------|----------|
| Register map (XLS) | `DGSMasterTriggerRegisterMap.xls` | `DGS_docs/RegisterMaps/` |
| Register map (CSV) | `DGSMasterTriggerRegisterMap.csv` | `DGS_docs/RegisterMaps/` |
| GRETINA register master doc | `Master Trigger Registers Master Document.pdf` | `DGS_docs/DGS_System_Documentation/Firmware/Master_Trigger/GRETINA Master Trigger/` |
| DSSD register doc | `DSSD Master Trigger Only Registers.pdf` | `DGS_docs/DGS_System_Documentation/Firmware/Master_Trigger/DSSD Master Trigger/` |
| Trigger user manual | `Trigger user manual 20140901.pdf` | `DGS_docs/DGS_System_Documentation/Modules/` |
| Trigger system firmware user guide | `DGS trigger system firmware user guide.pdf` | `DGS_docs/DGS_System_Documentation/Modules/` |
| CPLD sum logic | `CPLD_sum_logic.pdf` | `DGS_docs/DGS_System_Documentation/Firmware/Master_Trigger/` |
| Cheat sheet (diagnostic FIFOs) | `The Cheat Sheet for Diagnostic FIFOs and Counters in the Master Trigger.pdf` | `DGS_docs/DGS_System_Documentation/Firmware/Master_Trigger/DGS Master Trigger/` |
| TAC/TDC design | `TAC.docx` / `TAC.pdf` | `DGS_docs/DGS_System_Documentation/Firmware/Master_Trigger/` |
| TDC block diagram | `design_block_diagram.pdf` | `DGS_docs/DGS_System_Documentation/Firmware/Master_Trigger/DGS Master Trigger/TDC/` |
| TDC sub-design docs | Various `.docx`/`.pdf` | `DGS_docs/DGS_System_Documentation/Firmware/Master_Trigger/DGS Master Trigger/TDC/Documentation/` |
| Interfacing with other detectors | `Interfacing Digital Gammasphere with other detectors and systems.docx` | `DGS_docs/DGS_System_Documentation/Firmware/Master_Trigger/` |

---

## Router (RTRG)

| Document | File | Location |
|----------|------|----------|
| Register map (XLS) | `DGSRouterTriggerRegisterMap.xls` | `DGS_docs/RegisterMaps/` |
| Register map (CSV) | `DGSRouterTriggerRegisterMap.csv` | `DGS_docs/RegisterMaps/` |
| MISC_CTL register doc | `MISC_CTL.docx` | `DGS_docs/DGS_System_Documentation/Firmware/Router/` |

---

## Data Links (DIG ↔ RTRG ↔ MTRG)

| Document | File | Location |
|----------|------|----------|
| Trigger command link (latest 2016) | `20160418 trig command link.pdf` | `DGS_docs/DGS_System_Documentation/Links/Trigger Timing and Control Link/` |
| Trigger command link (2013) | `20131203 trig command link.pdf` | same |
| Trigger input link spec | `20060804 trig_input_link_spec.pdf` | `DGS_docs/DGS_System_Documentation/Links/Trigger Data Link/` |
| Router to Master Trigger link | `20080415 router2mt data link.pdf` | `DGS_docs/DGS_System_Documentation/Links/Router to Master Trigger Data Link/` |
| Frame 2/12/14/16/17 review | `20180226 review of Frames 2_12_14_16_17.docx` | `DGS_docs/DGS_System_Documentation/Links/Trigger Timing and Control Link/` |

---

## MyRIAD (Auxiliary Detector)

| Document | File | Location |
|----------|------|----------|
| Schematic | `myriad.pdf` | `DGS_SVN/dgs/MyRIAD/Schematics/` |
| Module specification | `MYRIAD_Module_Specification.pdf` | `DGS_SVN/dgs/MyRIAD/Documentation/` |
| User manual | `MyRIAD User Manaual.pdf` | `DGS_SVN/dgs/MyRIAD/Documentation/` |
| Abridged user notes | `MyRIAD Abridged User Notes.pdf` | `DGS_SVN/dgs/MyRIAD/Documentation/` |
| Memory map (XLS) | `MyRIAD_memory_map.xls` | `DGS_SVN/dgs/MyRIAD/` |
| Register map (CSV) | via `DGS_docs/DGS_System_Documentation/Firmware/MyRIAD/MyRIAD memory map.xls` | `DGS_docs/DGS_System_Documentation/Firmware/MyRIAD/` |

---

## SlopeBox / Pickoff Card (SBX)

| Document | File | Location |
|----------|------|----------|
| Pickoff card schematic (as-built) | `pickoff_20190702_asbuilt.pdf` | `DGS_SVN/dgs/SlopeBoxInterface/Documentation/Pick_Off Board Final Schematic/` |
| Pickoff card power board schematic | `power_board.pdf` | same |
| Pickoff card PCB layout (PICKOFF_01NOV18) | `PICKOFF_01NOV18.pdf` | `DGS_SVN/dgs/SlopeBoxInterface/PickoffCard/Layout/Aimtron/AimtronFinalFileSet/ANL_Pick_Off_Board_Database/` |
| Pickoff card assembly drawing | `Assembly Searchable.pdf` | `DGS_SVN/dgs/SlopeBoxInterface/PickoffCard/Layout/Aimtron/AimtronFinalFileSet/ANL_Pick_Off_Board_Assembly/` |
| Pickoff card Rev C schematic | `pickoff_c_Artwork.pdf` | `DGS_SVN/dgs/SlopeBoxExtension/PickoffCard/Layout/RevC/SlopeBoxExtensionPickoff_FAB_OUTPUT_02102021/` |
| Pickoff card Rev C fab drawing | `pickoff_c_fab.pdf` | same |
| Power module schematic | `RPie_Power_Module.pdf` | `DGS_SVN/dgs/SlopeBoxExtension/PowerConverter/Schematic/` |
| DVI Breakout schematic | `DVI_Breakout_Schematic.pdf` | `DGS_SVN/dgs/SlopeBoxExtension/DVIBreakout/RFQ Package - PCB Fab & Assembly/` |
| FPGA pin list (as-built) | `PICKOFF_20190702_ASBUILT_fpga_pin_list.xlsx` | `DGS_SVN/dgs/SlopeBoxExtension/PickoffCard/Schematic/Prototype/` |
| Preamp register map | `Preamp Registers, Mapping, and Scanning V2.xlsx` | `DGS_docs/SBX_Extension_Docs/` |
| VXI system EPICS PVs | `VXI system EPICS PVs.xlsx` | `DGS_docs/SBX_Extension_Docs/` |
| Replacement of VXI and pickoff card | `Replacement of VXI and pickoff card system.pdf` | `DGS_SVN/dgs/SlopeBoxInterface/Documentation/` |

---

## CollectorBox FPGAs

| Document | File | Location |
|----------|------|----------|
| Control FPGA register VHDL | `CtrlFPGA_registers*.vhd`, `CtrlFPGAPackage*.vhd` | `DGS_tools_pack/collector_FPGA/CollectorBox_CtrlFPGA/` |
| Stripe FPGA register VHDL | `StrpFPGA_registers*.vhd`, `StrpFPGAPackage*.vhd` | `DGS_tools_pack/collector_FPGA/CollectorBox_StripeFPGA/` |
| Control FPGA EPICS `.db` | `CtrlFPGA.db`, `CtrlFPGA_reg.db` | `DGS_tools_pack/collectorboxpi/CollectorBox_RevA/db/` |
| Stripe FPGA EPICS `.db` | `StrpFPGA.db`, `StrpFPGA_reg.db` | same |

---

## MTRG Trigger Card (06pc057)

| Document | File | Location |
|----------|------|----------|
| Hardware designs | Various schematics + BOM | `DGS_SVN/dgs/FromT/06pc057d/` |

---

## General System

| Document | File | Location |
|----------|------|----------|
| DGS crate mapping | `DGS crate mapping.xlsx` | `DGS_docs/DGS_System_Documentation/System/` |
| System inventory (2014) | `20140415_Digital DAQ System Inventory.xls` | `DGS_docs/DGS_System_Documentation/` |
| Cabling — BGO rings | `BGO_Ring.pdf` | `DGS_docs/DGS_System_Documentation/cabling/` |
| Raspberry Pi instructions | `Instructions for running Raspberry Pi.docx` | `DGS_docs/DGS_System_Documentation/` |
| AUX IO pin mapping (MTRG front panel) | `AUX_IO_Pin_Mapping.xlsx` | `DGS_SVN/dgs/Schematics/Trigger_IO_Adapter/` |
