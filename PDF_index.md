# DGS PDF Document Index

Stability: C2 - Active / semi-stable

_Root: `/home/dgsspark/DGS_tools_pack/DGS_docs/`_
_Indexed: 2026-04-05_
_Last organized: 2026-04-22_

---

## Table of Contents

- [Firmware / Digitizer](#firmware--digitizer)
- [Firmware / Master Trigger](#firmware--master-trigger)
- [Links (SERDES Protocol Specs)](#links-serdes-protocol-specs)
- [Modules](#modules)
- [Cabling](#cabling)
- [Datasheets (Digitizer Hardware Components)](#datasheets-digitizer-hardware-components)
- [SBX Extension / Preamp Interface](#sbx-extension--preamp-interface)
- [Historical System Documentation (SVN Archive)](#historical-system-documentation-svn-archive)
- [Priority Queue (Suggested Order)](#priority-queue-suggested-order)
- [Cross-References](#cross-references)

---

## Firmware / Digitizer

| Pages | Size | File | Notes |
|-------|------|------|-------|
| 72 | 2.4MB | `Firmware/Digitizer/ANL Digitizer Firmware for Experts.pdf` | ✅ **Documented** → [DIG_firmware_expert.md](DIG_firmware_expert.md) |
| — | — | `Non-Expert ANL Firmware - ANL version.pdf` | **Not in local docs** (wiki only). Shorter intro for scientists — recommended first read before Experts doc. |
| 58 | 1.6MB | `Firmware/Digitizer/ANL Firmware for LBL Digitizer.pdf` | ANL firmware on LBL hardware variant — **superseded** by Non-Expert + Experts docs |
| 242 | 1.5MB | `Firmware/Digitizer/Initial Draft - Digitizer Registers.pdf` | Full register map for the digitizer |
| 36 | 547KB | `Firmware/Digitizer/LED-only ANL digitizer firmware.pdf` | Older/simpler leading-edge-only build |
| 1 | 120KB | `Firmware/Digitizer/Understanding_FPGAs.pdf` | Background FPGA concepts (single diagram) |
| 1 | 58KB | `Firmware/Digitizer/Waveform Extension.pdf` | Pileup/waveform extension details (single diagram) |

---

## Firmware / Master Trigger

| Pages | Size | File | Notes |
|-------|------|------|-------|
| 8 | 240KB | `Firmware/Master_Trigger/CPLD_sum_logic.pdf` | Fast CPLD strobe multiplicity logic (~1 µs latency) |
| 16 | 69KB | `Firmware/GRETA Algorithm Architecture-v3 1b (2).pdf` | GRETA algorithm background |
| **DGS Master Trigger** | | | |
| 53 | 170KB | `Firmware/Master_Trigger/DGS Master Trigger/subdoc/mem map.pdf` | DGS MTRG register memory map |
| 10 | 125KB | `Firmware/Master_Trigger/DGS Master Trigger/The Cheat Sheet for Diagnostic FIFOs and Counters in the Master Trigger.pdf` | Diagnostic FIFO/counter reference |
| 1 | 178KB | `Firmware/Master_Trigger/DGS Master Trigger/TDC/design_block_diagram.pdf` | TAC-II TDC block diagram |
| 1–2 ea | ~400KB | `Firmware/Master_Trigger/DGS Master Trigger/TDC/Documentation/*.pdf` | TDC sub-block docs (data_compressor, data_synchronizer, local_pos_finder, offset_flagger, pipeline_unit, timestamp_counter, top_pipeline_unit, trig_ack_processor, vernier_phase_fifo) |
| **DSSD Master Trigger** | | | |
| 24 | 236KB | `Firmware/Master_Trigger/DSSD Master Trigger/20120806 DSSD trig command link.pdf` | DSSD-specific trigger command link |
| 71 | 198KB | `Firmware/Master_Trigger/DSSD Master Trigger/DSSD Master Trigger Only Registers.pdf` | DSSD MTRG registers |
| 53 | 170KB | `Firmware/Master_Trigger/DSSD Master Trigger/subdoc/mem map.pdf` | DSSD MTRG memory map |
| 10 | 125KB | `Firmware/Master_Trigger/DSSD Master Trigger/The Cheat Sheet...` | DSSD diagnostic FIFO/counter reference |
| **GRETINA Master Trigger** | | | |
| 81 | 208KB | `Firmware/Master_Trigger/GRETINA Master Trigger/Master Trigger Registers Master Document.pdf` | GRETINA MTRG full register doc |
| 53 | 170KB | `Firmware/Master_Trigger/GRETINA Master Trigger/subdoc/mem map.pdf` | GRETINA MTRG memory map |
| 10 | 125KB | `Firmware/Master_Trigger/GRETINA Master Trigger/The Cheat Sheet...` | GRETINA diagnostic FIFO/counter reference |

---

## Links (SERDES Protocol Specs)

| Pages | Size | File | Notes |
|-------|------|------|-------|
| 13 | 114KB | `Links/Router to Master Trigger Data Link/20080415 router2mt data link.pdf` | RTRG→MTRG data link format |
| **Trigger Data Link** (Digitizer → Router) | | | |
| 13 | 135KB | `Links/Trigger Data Link/20060526 trig_input_link_spec.pdf` | v1 |
| 12 | 120KB | `Links/Trigger Data Link/20060709 trig_input_link_spec.pdf` | v2 |
| 12 | 121KB | `Links/Trigger Data Link/20060804 trig_input_link_spec.pdf` | v3 (latest) |
| **Trigger Timing and Control Link** (MTRG → Digitizer command stream) | | | |
| 20 | 181KB | `Links/Trigger Timing and Control Link/20060526 trigger command link.pdf` | v1 |
| 20 | 168KB | `Links/Trigger Timing and Control Link/20060707 trig command link.pdf` | v2 |
| 20 | 168KB | `Links/Trigger Timing and Control Link/20060804 trig command link.pdf` | v3 |
| 20 | 193KB | `Links/Trigger Timing and Control Link/20080711 trig command link.pdf` | v4 |
| 39 | 385KB | `Links/Trigger Timing and Control Link/20131203 trig command link.pdf` | v5 |
| 40 | 860KB | `Links/Trigger Timing and Control Link/20160418 trig command link.pdf` | **v6 (latest)** — ✅ **Documented** → [ttcl.md](ttcl.md) |

---

## Modules

| Pages | Size | File | Notes |
|-------|------|------|-------|
| 53 | 1.1MB | `Modules/Digitizer-Specification-RevA-v2 0.pdf` | Hardware spec for the digitizer module |
| 26 | 488KB | `Modules/DGS trigger system firmware user guide.pdf` | System-level trigger operation guide |
| — | — | `Firmware/Master_Trigger/Interfacing Digital Gammasphere with other detectors and systems.docx` | Interfacing DGS with external detector systems (local .docx, not PDF) |
| — | — | `TripletPulseNotesTake2.doc` | Demos Digitizer Tester: effects of integration time + readout length on complex waveforms (local .doc, wiki ref: TripletPulseNotesTake2.pdf) |
| 25 | 500KB | `Modules/Trigger user manual 20140901.pdf` | Trigger user manual (2014) |
| 2 | 145KB | `Modules/Trigger user manual extract.pdf` | Short extract from trigger manual |
| 14 | 247KB | `Modules/MYRIAD_Module_Specification.pdf` | MYRIAD module spec |
| 7 | 211KB | `Modules/myriad.pdf` | MYRIAD (shorter doc) |
| 5 | 70KB | `Modules/Specification for a Digitizer Front Panel Interconnect.pdf` | Front bus/AUX I/O interconnect spec |
| 9 | 237KB | `Modules/TAC.pdf` | TAC (Time-to-Amplitude Converter) module |

---

## Cabling

| Pages | Size | File | Notes |
|-------|------|------|-------|
| 1 | 385KB | `cabling/BGO_Ring.pdf` | BGO ring cabling diagram |
| 1 | 287KB | `cabling/brenda_detector_emulation.pdf` | Detector emulation setup |
| 1 | 18KB | `cabling/digitizer lemo.pdf` | Digitizer LEMO connector pinout |
| 1 | 416KB | `cabling/ksm cable quote.pdf` | Cable procurement quote |
| 1 | 34KB | `cabling/Majorana.pdf` | Majorana detector cabling reference |

---

## Datasheets (Digitizer Hardware Components)

| Pages | Size | File | Notes |
|-------|------|------|-------|
| 72 | 1.1MB | `Datasheets/digitizer/28F128 flash.pdf` | Flash memory (config storage) |
| 22 | 942KB | `Datasheets/digitizer/31Y334-Schematic-10ChanDigitizer-Rev4.1.pdf` | 10-channel digitizer schematic Rev4.1 |
| 22 | 974KB | `Datasheets/digitizer/31Y334-Schematic-10ChanDigitizer-Rev4.2.pdf` | 10-channel digitizer schematic Rev4.2 |
| 24 | 770KB | `Datasheets/digitizer/AD6645.pdf` | ADC chip (14-bit, 80/105 MSPS) |
| 12 | 257KB | `Datasheets/digitizer/ds92lv010a.pdf` | LVDS SERDES transceiver |
| 46 | 382KB | `Datasheets/digitizer/IDT_72V2103-72V2113_DST_20100601.pdf` | IDT FIFO chip (external FIFO memory) |
| 22 | 1001KB | `Datasheets/digitizer/sn74lvc1g125.pdf` | Single bus buffer gate |
| 16 | 460KB | `Datasheets/digitizer/sn74lvt162245a.pdf` | 16-bit bus transceiver |

---

## SBX Extension / Preamp Interface

| Pages | Size | File | Notes |
|-------|------|------|-------|
| 10 | 830KB | `SBX_Extension_Docs/20210121 preamp reset killer results.pdf` | PRK test results (directly relevant to DIG firmware PRK section) |
| 5 | 401KB | `SBX_Extension_Docs/LabNotes/20210415.pdf` | Lab notes April 2021 |
| 5 | 848KB | `SBX_Extension_Docs/Pick_Off Board Final Schematic/pickoff_20190702_asbuilt.pdf` | SBX pickoff board schematic (as-built) |
| 14 | 173KB | `SBX_Extension_Docs/Pick_Off Board Final Schematic/power_board.pdf` | SBX power board schematic |
| 17 | 1.8MB | `SBX_Extension_Docs/Replacement of VXI and pickoff card system.pdf` | VXI→SBX replacement system description |
| 5 | 581KB | `SBX_Extension_Docs/SBX Pickoff ECO Checklist.pdf` | ECO checklist for pickoff card |
| 20 | 872KB | `SBX_Extension_Docs/Serial Control and Monitoring Link Specification.pdf` | Serial control/monitoring link for SBX |

---

## Historical System Documentation (SVN Archive)

_Root: `DGS_tools_pack/DGS_SVN/dgs/Documentation/Formal/`_

| Size | File | Notes |
|------|------|-------|
| 76KB | `Unsorted Docs/dgs_systemdocs.pdf` | **DGS High-Level System Documentation** (T. Madden, APS-XSD, Nov 2011): Block diagram description of DGS — detector array → digitizers → routers → master trigger → VME IOC → sender → receiver → data merge. Defines core terminology (Sender, Receiver, Merger, Gammaware). Historical — predates current PyQt6 GUI and GEBSort; still accurate for basic architecture. |
| — | `Unsorted Docs/dgsSender.pptx` | **dgsSender presentation** — covers the MiniSender / outLoop design inside the VME IOC. Companion to `howTheSenderWorks.docx`. |
| — | `Unsorted Docs/DCBAL.doc` | **DC Balance documentation** — describes the DC balance logic on MTRG/RTRG SERDES links (needed with the fiber expander as of July 2022; see `EN_RTR_DCBAL` PV and `LinkL_DCbal` in `link_sys.py`). |
| — | `Unsorted Docs/pv list digitizer.txt` | **Legacy VisualDCT PV list** — old `Dig$(DB)_` PV naming convention (e.g., `Dig$(DB)_CS_TrigDly`, `Dig$(DB)_CS_NoiseWin`); register addresses in hex (0x0, 0x8, 0xC, …). Pre-dates current `MOD###_DIG_` naming. Useful for cross-referencing old data/docs against modern PV names. |

### Historical Software Documentation

_Root: `DGS_tools_pack/DGS_SVN/dgs/Documentation/Formal/Software/`_

| Size | File | Notes |
|------|------|-------|
| 847KB | `DigitalGammasphereSW.pdf` | **DGS Software Development overview** (T. Madden, ANL): asyn driver approach, EDM→CSS GUI migration, auto-generation of EPICS DBs + screens + st.cmd from register spreadsheets, Python EPICS PV class. Historical — this is the origin of the current ANLDAQ/IOC approach. |
| 274KB | `asynDebugDriverDOCS.pdf` | asyn debug driver documentation — likely documents the `asynDebug.template` record type (removed from current `ioc/db/` but present in `DB_backup_20240205`) |
| 358KB | `DebuggingEPICSChannelAccess.pdf` | Guide to debugging EPICS Channel Access issues |
| 86KB | `pythonEPICS.pdf` | Python EPICS guide (likely pyepics intro for DGS developers) |
| 146KB | `HowCarlware-TimwareWorks.docx` | Explains Carlware/Timware — the legacy multi-crate data merging and time-sorting software used before GEBSort/ANLDAQ. Historical reference for understanding pre-DGS2 data flow. |
| 207KB | `howTheSenderWorks.docx` | Documents the MiniSender (IOC-side TCP data sender) internals — how data flows from VME FIFOs through the IOC outLoop to the TCP receiver. Predecessor or companion to `ANLDAQ/tcpReceiver/`. |
| 3.0MB | `GammaWare.docx` | GammaWare analysis framework documentation — large, likely covers the full gamma-ray analysis pipeline (histogramming, coincidences, calibration). Related to ROOT/GEBSort era. |
| 722KB | `DigitalGammasphereSW.pptx` | Presentation version of DGS software overview (same content as PDF, slide format). |
| 48KB | `epicsPvs.pptx` | Slide deck on DGS EPICS PV organization (likely covers naming conventions, record types, DB structure). |
| — | `IOC code design notes 20210914.pptx` | IOC code design notes from Sept 14 2021 — covers asyn driver architecture, outLoop/inLoop design. Companion to IOC_Code_Design wiki page. |

---

## Priority Queue (Suggested Order)

1. ✅ `ANL Digitizer Firmware for Experts.pdf` — done → [DIG_firmware_expert.md](DIG_firmware_expert.md)
2. ✅ `Links/Trigger Timing and Control Link/20160418 trig command link.pdf` — done → [ttcl.md](ttcl.md)
3. `Firmware/Master_Trigger/DGS Master Trigger/subdoc/mem map.pdf` — DGS MTRG register map
4. `Modules/DGS trigger system firmware user guide.pdf` — system-level trigger guide
5. `Firmware/Digitizer/Initial Draft - Digitizer Registers.pdf` — digitizer register map (242 pg, large)
6. `Links/Trigger Data Link/20060804 trig_input_link_spec.pdf` — DIG→RTRG data format
7. `Links/Router to Master Trigger Data Link/20080415 router2mt data link.pdf` — RTRG→MTRG link

## Cross-References

- [reference_index.md](reference_index.md) — Hardware drawings index (schematics, PCB docs); complements this PDF index
- [DIG_firmware_expert.md](DIG_firmware_expert.md) — Distilled from "ANL Digitizer Firmware for Experts.pdf"
- [ttcl.md](ttcl.md) — Distilled from "20160418 trig command link.pdf"
- [deep_fpga_MTRG_MAIN.md](deep_fpga_MTRG_MAIN.md) — MTRG firmware; related to MTRG register map PDFs
- [deep_fpga_DIG.md](deep_fpga_DIG.md) — DIG firmware; related to digitizer register PDFs
