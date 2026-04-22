# DGS Knowledge Base

This repository contains technical documentation for the **Digital Gamma-ray Spectrometer (DGS)** system at **Argonne National Laboratory (ANL)**. It is maintained by the DGS AI assistant (General DGS) and serves as a structured, searchable knowledge base for system architecture, firmware, hardware, EPICS control, and operations.

**Official wiki:** https://wiki.anl.gov/gsdaq (may be outdated — cross-check with source code when in doubt)  
**Contact:** Ryan Tang (`ttang@anl.gov`) — tech lead / architect

---

## Contents

### System Overview

| File | Description |
|------|-------------|
| [overview_DGS.md](overview_DGS.md) | Full DGS (Gammasphere) system architecture, hardware inventory, IP map, subsystem map |
| [overview_SmallSystem.md](overview_SmallSystem.md) | DuoGe (DUO) and X-Array (DXA) small system overviews |
| [hardware_architecture.md](hardware_architecture.md) | Hardware breakdown: DuoGe vs DGS, signal chain, collector box architecture |
| [reference_index.md](reference_index.md) | Hardware drawings index: register map CSVs (DIG/MTRG/RTRG), schematics, PCB docs, datasheet locations |
| [wiki_gsdaq.md](wiki_gsdaq.md) | Index of the ANL GS DAQ wiki with key facts and known inaccuracies |

### DAQ & Data Acquisition

| File | Description |
|------|-------------|
| [ANLDAQ.md](ANLDAQ.md) | DAQ GUI (PyQt6) overview: EPICS CA config, VxWorks data pipeline (inLoop/outLoop/MiniSender), softIOC (JustGlobals.db, dgsSupport.db), class_PV/Board, findAllPV, commander.py |
| [ANLDAQ_tcpReceiver.md](ANLDAQ_tcpReceiver.md) | tcpReceiver deep-dive: 3 binaries, TCP protocol proof, data flow, GEB header, class_DIG.h/class_TDC.h decoders, run control scripts, legacy receivers, packet consistency table |
| [ANLDAQ_GUI_windows.md](ANLDAQ_GUI_windows.md) | GUI window reference: gui_MTRG (5 tabs), gui_Det (collector box map), gui_scalar, gui_RTR, gui_Board (generic PV table), gui_CH (per-channel 5-tab), gui_RAM, gui_SYS, gui_LinkSys, gui_DataTaking |
| [trig_setup_scripts.md](trig_setup_scripts.md) | 5-stage trigger setup scripts (trig_setup_Stage1–5.sh): full step-by-step MTRG→RTRG→DIG link initialization, SYSTEM_DEFINES.sh GS topology (4 RTRGs, 44 DIGs, MTRG in VME10), DC balance/fiber notes, algorithm reference |
| [guceiver.md](guceiver.md) | Guceiver: live diagnostic GUI (waveform, spectrum, TAC-II, raw data) — connects to IOC TCP:9001 |
| [dgs_analysis.md](dgs_analysis.md) | Post-experiment analysis pipeline: EventBuilder variants (Q, PQ — k-way merge, parallel, double-buffered), parquet_pysort, gray_apps summary, parquetCLI, gain_from_parquet.py, pz_from_parquet.py, RunParquet, ProcessRUN, GEB data format |
| [dgs_analysis_grayapps.md](dgs_analysis_grayapps.md) | gray_apps full reference: GrayCAL (HPGe energy calibration GUI, core modules, polezero_dialog), GrayMAN (multi-peak spectrum analysis), grayfit (AutoFitter, FittingRunner, PeakFinder, pole_zero_fitter, FitResult hierarchy) — split from dgs_analysis.md |
| [dgs_analysis_root_scripts.md](dgs_analysis_root_scripts.md) | ROOT analysis scripts (fastEventConstructor): analyzer.cpp (γ-γ coincidence), analyzer_tac.cpp (TAC-II), analyzer_trace.cpp (waveform/PSD), analyzer_pz_cal.cpp (PZ from traces), Cali_e.C (energy calibration, 110 dets), checkTACFile.cpp (TAC binary validator), findMapping.sh/findGS.sh (GS channel map tools) — split from dgs_analysis.md |
| [gebsort.md](gebsort.md) | GEBSort: event builder/sorter, GEBMerge, DGS calibration workflow (find_MK, fwhm_onepeak, dgs_ecal), GEBSort.chat config |
| [data_structures.md](data_structures.md) | Binary data structures: GEBHeader, DIG event payload (all 13 words, all HEADER_TYPE modes), TAC-II TDC, UniqueID convention, full event flow |
| [run_procedures.md](run_procedures.md) | Typical DGS run procedures: directory setup, GEBSort, PZ/energy calibration workflow |
| [pole_zero.md](pole_zero.md) | Pole-zero correction: physics, PZ coefficient, pz_from_parquet.py workflow, GrayCAL method, PQDecode.chat config |
| [troubleshooting.md](troubleshooting.md) | DGS troubleshooting: IOC connectivity, SYNC bit gotcha, FIFO/PV name corrections, timestamp sync errors, BGO channel suppression by preamp-reset blanking, InfluxDB/pi5-lnFill temperature stale recovery |
| [expMemory_2008_Chiara.md](expMemory_2008_Chiara.md) | Experiment log: exp2008_Chiara (active); run data locations, cleanup log, monitoring |

### FPGA Firmware

| File | Description |
|------|-------------|
| [fpga.md](fpga.md) | FPGA firmware overview: DIG/RTRG/MTRG hierarchy, signal flow, end-to-end timing, build types, auxiliary firmware (FPGA/others/) |
| [deep_fpga_DIG.md](deep_fpga_DIG.md) | DIG firmware: Spartan-3, ADC pipeline, SERDES, discriminators, architecture overview |
| [deep_fpga_DIG_channel.md](deep_fpga_DIG_channel.md) | DIG per-channel signal processing: LED/CFD discriminator modes, delay chain, pileup, VME FPGA, IP cores (split from deep_fpga_DIG.md) |
| [deep_fpga_DIG_eventpacket.md](deep_fpga_DIG_eventpacket.md) | DIG event packet format: LED/CFD header layout, split field reconstruction, pole-zero correction, waveform samples, integration timelines (split from deep_fpga_DIG.md) |
| [DIG_firmware_expert.md](DIG_firmware_expert.md) | DIG firmware expert guide: all modes, trigger_mux_select (IntAcptAll/ExtTTL/ExtTTCL/Diag), aux I/O |
| [deep_fpga_RTRG.md](deep_fpga_RTRG.md) | RTRG firmware: Virtex-4, multiplicity aggregation, throttle, VME register map |
| [260E_trigger_scheme.md](260E_trigger_scheme.md) | RTRG 0x260E + MTRG trigger scheme deep-dive: `chan_in.vhd` (serial reception, 18-bit SERDES word, 640 ns DPRAM delay alignment, X/Y plane maps), `disc_mach.vhd` (clean/dirty/BGO-only classification), `router_data_path.vhd` (Link-L multiplicity aggregation), MTRG `mt_input_channel.vhd`, `eight_mt_channel.vhd`, `sum_hits_X.vhd`, `calc_total_sum.vhd`, `top.vhd` trigger decision; full end-to-end timing with DUO example and Mermaid diagram |
| [deep_fpga_MTRG.md](deep_fpga_MTRG.md) | MTRG overview: 3 devices (Main FPGA, VME FPGA, CPLD) |
| [deep_fpga_MTRG_MAIN.md](deep_fpga_MTRG_MAIN.md) | MTRG Main FPGA: trigger algorithms, 20-frame command structure, TAC-II TDC, VME map, RF→NIM IN 2 |
| [tac2.md](tac2.md) | TAC-II TDC in MTRG: vernier interpolation (~30–50 ps), 250 MHz 4-phase clock, 64-tap delay lines, data collection state machines, 350 ns pipeline delay |
| [deep_fpga_MTRG_VIVADO.md](deep_fpga_MTRG_VIVADO.md) | MTRG Vivado port: Kintex UltraScale XCK060 |
| [deep_fpga_MTRG_VME.md](deep_fpga_MTRG_VME.md) | MTRG VME FPGA: Spartan-3, A32/D32 slave, FPGA config manager |
| [deep_fpga_MTRG_CPLD.md](deep_fpga_MTRG_CPLD.md) | MTRG CPLD: XC9500XL, fast strobe multiplicity logic (~1 µs latency) |
| [deep_fpga_building.md](deep_fpga_building.md) | Build toolchain: ISE 14.7 / Vivado 2018.3, Ubuntu 24.04 Docker approach |
| [link_sys_analysis.md](link_sys_analysis.md) | link_sys.py: 5-stage SERDES link init + timestamp sync sequence |
| [ttcl.md](ttcl.md) | TTCL spec (v2.1): word/frame/cycle format, all 20 frames, selective propagation |


### Hardware & Connectors

| File | Description |
|------|-------------|
| [connectors.md](connectors.md) | All connector pinouts: DIG RJ45 (TTCL/SER/DES), 36-pin Aux I/O (ExtTTL = AUX_DIN[10] pins 32/33); MTRG/RTRG 125-pin SERDES (links A–H/L/R/U), NIM I/O (NIM IN 1=aux trig, NIM IN 2=TDC/RF), CPLD ribbons. ASCII pin diagrams included. |

### EPICS IOC

| File | Description |
|------|-------------|
| [ioc.md](ioc.md) | EPICS IOC config, boot scripts, firmware versions, MVME5500 setup |
| [IOC_cmd.md](IOC_cmd.md) | Full IOC shell command reference: DGS custom (ProgramFlash, VMERead32, asynDigitizerConfig…), EPICS 3.14 (dbl, dbpr, casr…), asyn 4.17; safety classification; terminal server map |
| [VME_registers.md](VME_registers.md) | Complete VME register address map for DIG, MTRG, and RTRG main FPGAs + VME FPGA + flash; address patterns, bit-field notes, IOC shell usage examples |
| [EPICS.md](EPICS.md) | EPICS primer: record types, tools, Python integration for DGS |
| [EPICS_asyn.md](EPICS_asyn.md) | asyn driver support: caput flow diagram, port concept, worker threads, bulk writes, passive hardware callbacks |
| [vxworks.md](vxworks.md) | VxWorks cross-compilation: build pipeline, directory structure, munch process, glossary, legacy `devGData.c`, Port 9010 FIFO grabber design, `MergedAsynDigParams.c` DIG param registration (222 params, all groups documented), `FlashMaintenance.c`, `profile.c` |
| [vxworks_fifo_readout.md](vxworks_fifo_readout.md) | DMA buffer architecture, trigger FIFO readout (`readTrigFIFO.c`, `CheckAndReadTrigger`), Type-F synthetic headers (trigger + digitizer), FIFO index map, DMA chunking |
| [vxworks_migration.md](vxworks_migration.md) | Migration notes from con6 (Solaris) to Ubuntu 24 |

### Collector Box & Gammasphere

| File | Description |
|------|-------------|
| [collector_fpga.md](collector_fpga.md) | Collector box FPGA firmware: CtrlFPGA (housekeeping/monitoring), StripeFPGA (relay/stripe/LED), pickoff card FPGAs (SBX Interface + Extension) |
| [collectorboxpi.md](collectorboxpi.md) | Collector box soft IOC on Raspberry Pi; HV control; PXE boot setup |
| [collectorbox_PVs.md](collectorbox_PVs.md) | CollectorBox PV list: 1,431 records/detector; GS/MOD/VME_GS/Ge_ID numbering explained |
| [collectorbox_devicesupport.md](collectorbox_devicesupport.md) | EPICS device support internals: SPI driver, CAMAC_IO link, conversion coefficients |
| [gammasphere_geometry.md](gammasphere_geometry.md) | Gammasphere array geometry: 110 GS holes, 17 rings, θ angles per hole, full hole→angle map |
| [n126_target_wheel.md](n126_target_wheel.md) | N=126 target wheel encoder interface: FPGA (Spartan-6), Raspberry Pi EPICS IOC, DRV8824 stepper, L6203 DC motor, quadrature encoder, 8-ch DAC, I²C power board and LED display (auxiliary experiment device, not in main DAQ trigger chain) |

### Process Variables (PVs)

| File | Description |
|------|-------------|
| [DGS_PVs.md](DGS_PVs.md) | DGS system PV list |
| [DUO_PVs.md](DUO_PVs.md) | DuoGe system PV list |
| [Xarray_PVs.md](Xarray_PVs.md) | X-Array system PV list |
| [teststand_PVs.md](teststand_PVs.md) | Test stand PV list |

### Detector Interface Hardware

| File | Description |
|------|-------------|
| [sbx.md](sbx.md) | Slope Box + SBX: HV generation, signal conditioning, BGO pattern/sum, pickoff card (FPGA-based: programmable gain/offset/BGO discrimination/background scanning), GS_ID dongle, Pi IOC (SPI/GPIO), I2C opcode engine |
| [sbxPi_ioc.md](sbxPi_ioc.md) | SBX Pi standalone IOC (PickoffApp_RevC): direct SPI1 to Pickoff FPGA, CAMAC_IO device support, global mailboxes, I2C command FIFO protocol, HV ramp logic, DB file descriptions, CA port 5080 |
| [slope_box_interface.md](slope_box_interface.md) | SlopeBox EPICS IOC software (SVN): PickoffApp device support, CAMAC_IO link type repurposing for SPI/GPIO, transaction format, why asyn was rejected, BGO counter scripts, original BASIC control programs (1997) |
| [myriad.md](myriad.md) | MγRIAD module: aux detector interface, NIM I/O pinout, ECL connectors, TTCL link, DGS usage |
| [digitizer_tester.md](digitizer_tester.md) | Digitizer Tester: dual 200 MHz 16-bit DAC, analog switch matrix (10ch), TTCL link, waveform generation |
| [preamp_reset_readme.md](preamp_reset_readme.md) | Preamplifier reset handling: ADC threshold detection (LOLO/HIHI), PREAMP_DELAY kill state, BGO veto gate, PARST timestamp, Frame 15 remote reset |

### VHDL Module Analysis (`vhdl/` subdirectory)

Detailed plain-English summaries of key FPGA VHDL source files. Generated 2026-04-15 from primary source.

| File | Module | Description |
|------|--------|-------------|
| [vhdl/RTRG_chan_in.md](vhdl/RTRG_chan_in.md) | `chan_in.vhd` | RTRG: serial SERDES input receiver, 18-bit word decoding, 640 ns DPRAM delay alignment, discriminator bit extraction |
| [vhdl/RTRG_disc_mach.md](vhdl/RTRG_disc_mach.md) | `disc_mach.vhd` | RTRG: discriminator classifier (clean/dirty/BGO-only), event tagging logic |
| [vhdl/RTRG_overlap_mach.md](vhdl/RTRG_overlap_mach.md) | `overlap_mach.vhd` | RTRG: trigger overlap and hold-off state machine |
| [vhdl/RTRG_router_data_path.md](vhdl/RTRG_router_data_path.md) | `router_data_path.vhd` | RTRG: Link-L multiplicity aggregation, data forwarding to MTRG |
| [vhdl/RTRG_top.md](vhdl/RTRG_top.md) | `TOP.VHD` | RTRG top-level: all sub-block wiring, port map, SERDES link management |
| [vhdl/MTRG_top.md](vhdl/MTRG_top.md) | `top.vhd` | MTRG top-level: 8 Router aggregation, trigger decision distribution, NIM I/O, CPLD bus |
| [vhdl/MTRG_eight_mt_channel.md](vhdl/MTRG_eight_mt_channel.md) | `eight_mt_channel.vhd` | MTRG: instantiates 8 `mt_input_channel` blocks, one per Router link |
| [vhdl/MTRG_mt_input_channel.md](vhdl/MTRG_mt_input_channel.md) | `mt_input_channel.vhd` | MTRG: per-Router input channel: SERDES receiver, hit extraction, multiplicity contribution |
| [vhdl/MTRG_sum_hits_X.md](vhdl/MTRG_sum_hits_X.md) | `sum_hits_X.vhd` | MTRG: summing hit counts across X-plane (north/south hemisphere aggregation) |
| [vhdl/MTRG_calc_total_sum.md](vhdl/MTRG_calc_total_sum.md) | `calc_total_sum.vhd` | MTRG: final multiplicity sum and trigger decision comparator |
| [vhdl/MTRG_MYRIAD_RCV_MACH.md](vhdl/MTRG_MYRIAD_RCV_MACH.md) | `MYRIAD_RCV_MACH.vhd` | MTRG: MγRIAD receiver state machine — locks onto 5-word SERDES frame from Link U, extracts NIM/ECL/FERA states and raw/gated trigger bits |
| [vhdl/MTRG_MYRIAD_TRIGGER.md](vhdl/MTRG_MYRIAD_TRIGGER.md) | `MYRIAD_TRIGGER.vhd` | MTRG: MγRIAD trigger algorithm — programmable delay line, optional coincidence with other algorithms, selectable timestamp mode, subtypes 0x78/0x79 |
| [vhdl/MTRG_mstr_mach.md](vhdl/MTRG_mstr_mach.md) | `mstr_mach.vhd` | MTRG: Master State Machine — continuously emits the 20-frame TTCL command cycle (SYNC, trigger decisions, Frame 12/13/14/15/16 control), local and remote-master modes, FIFO pipelining, monitor FIFO controls |
| [vhdl/MTRG_local_trig_coinc.md](vhdl/MTRG_local_trig_coinc.md) | `local_trig_coinc.vhd` | MTRG: Local-vs-local coincidence trigger algorithm — dual-mask OR selects any two algorithm acks, overlap_mach enforces coincidence window, feeds trig_algo_support for standard FIFO/prescale/holdoff handling |
| [vhdl/PROGRESS.md](vhdl/PROGRESS.md) | — | Checklist of VHDL files summarized (RTRG + MTRG) |

### Liquid Nitrogen

| File | Description |
|------|-------------|
| [lnfill.md](lnfill.md) | LN filling system: valves, tanks, fill schedule, cron jobs, Discord alerts, ops procedures (overtime, manual fill, findhose) |
| [lnfill_ioc.md](lnfill_ioc.md) | LN fill system deep internals: InfluxDB data flow, hose→detID mapping table, ln2con IOC boot tree, DetMan.py FillManifold() state machine |
| [con6_lnfill.md](con6_lnfill.md) | con6 (Solaris 10, 192.168.203.136): CVS source repo + 68040 cross-compiler for lnfill IOC; ln2con NFS boot host; archiving priority and retirement migration plan |

### Utilities & Operations

| File | Description |
|------|-------------|
| [nfs_layout.md](nfs_layout.md) | NFS mount layout on DCS2: vol2–vol5, fs1/vol2, fs2/vol3, piserver; full directory inventory (experiment data, IOC py_scripts, gamln.db PV structure, legacy lnfill, EDM screens, GEBSort binaries, sbx2022tuning); collector box PXE MAC map |
| [utility_scripts.md](utility_scripts.md) | BGO HV tuning scripts (NS_scripts/slopebox_scripts), PV discovery scripts, ANLDAQ GUI helper scripts (basic_settings_LED.py, terminals); data0 space monitor (deleted — does not exist) |
| [snapshot_pv.md](snapshot_pv.md) | snapshot_pv repo: PV snapshot & watchdog utilities (Python/pyepics) |
| [influxdb_grafana.md](influxdb_grafana.md) | InfluxDB 3 + Grafana monitoring on DCS2 (192.168.203.56) |


### Legacy & Reference

| File | Description |
|------|-------------|
| [DGS_SVN.md](DGS_SVN.md) | Legacy SVN archive: per-directory inventory (daq_system_tags, Data_Generator, salvaged_notes, psg, SlopeBoxExtension, VXI_database, etc.); historical reference |
| [PDF_index.md](PDF_index.md) | Index of all PDF documentation in DGS_tools_pack |

---

## Key Facts

- **System:** Digital Gamma-ray Spectrometer at ATLAS accelerator, ANL
- **Detectors:** 110 HPGe (Gammasphere) + BGO Compton shields
- **Channels:** Up to 640 (1 MTRG × 8 RTRG × 8 DIG × 10 ch)
- **Trigger cycle:** 2 µs (20 frames × 5 words at 50 MHz)
- **End-to-end latency:** ~2–4 µs (1–2 trigger cycles; DIG→RTRG→MTRG→decision→RTRG→DIG)
- **Data transport:** TCP, port 9001 per IOC (`SOCK_STREAM`)
- **EPICS CA ports:** DGS 5064/5065 · DXA 5072/5073 · DUO 5080/5081
- **IOC crates:** 12 VME (192.168.203.141–145, 177–183)
- **Admin host:** spark-ca9f / DGX Spark (192.168.203.132) — replaced pi5-dgs as General DGS host 2026-04-15
- **Data host:** DCS2 (`dcsu@DCS2.onenet`)

---

## Source Verification Policy

All information in this knowledge base is verified against primary sources before being documented. We do **not** rely solely on wiki pages or informal descriptions. Sources checked include:

- **Source code** — VxWorks C/C++ (IOC drivers, sender/receiver), Python (ANLDAQ GUI, run control), VHDL (FPGA firmware)
- **FPGA firmware** — Xilinx ISE/Vivado project files, `.vhd` source, synthesis constraints
- **Hardware schematics & drawings** — digitizer PCB schematics (Rev 4.1/4.2), trigger module layouts
- **EPICS database templates** — `.db`, `.template` files defining PV names, types, and enumerations
- **Official PDF documentation** — ANL firmware expert guides, TTCL spec, trigger user manual, digitizer spec
- **SVN/GitLab history** — legacy code and historical versions for cross-checking

Where the wiki (`wiki.anl.gov/gsdaq`) contradicts the source code, the source code is treated as ground truth. Known wiki inaccuracies are flagged inline (see `wiki_gsdaq.md`).

---

*Maintained by General DGS (AI assistant). Last updated: 2026-04-22. MEMORY.md Knowledge Base Index replaced by this file as the single source of truth.*


