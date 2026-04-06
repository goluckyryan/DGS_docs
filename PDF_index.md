# DGS PDF Document Index
_Root: `/home/ryan/DGS_tools_pack/DGS_docs/`_
_Indexed: 2026-04-05_

---

## Firmware / Digitizer

| Pages | Size | File | Notes |
|-------|------|------|-------|
| 72 | 2.4MB | `Firmware/Digitizer/ANL Digitizer Firmware for Experts.pdf` | ✅ **Documented** → `memory/dgs/DIG_firmware_expert.md` |
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
| 40 | 860KB | `Links/Trigger Timing and Control Link/20160418 trig command link.pdf` | **v6 (latest)** — 🔄 In progress |

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

## Priority Queue (Suggested Order)

1. ✅ `ANL Digitizer Firmware for Experts.pdf` — done
2. 🔄 `Links/Trigger Timing and Control Link/20160418 trig command link.pdf` — **in progress**
3. `Firmware/Master_Trigger/DGS Master Trigger/subdoc/mem map.pdf` — DGS MTRG register map
4. `Modules/DGS trigger system firmware user guide.pdf` — system-level trigger guide
5. `Firmware/Digitizer/Initial Draft - Digitizer Registers.pdf` — digitizer register map (242 pg, large)
6. `Links/Trigger Data Link/20060804 trig_input_link_spec.pdf` — DIG→RTRG data format
7. `Links/Router to Master Trigger Data Link/20080415 router2mt data link.pdf` — RTRG→MTRG link
