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
| [reference_index.md](reference_index.md) | Index of schematics, datasheets, and PDF documentation |
| [wiki_gsdaq.md](wiki_gsdaq.md) | Index of the ANL GS DAQ wiki with key facts and known inaccuracies |

### DAQ & Data Acquisition

| File | Description |
|------|-------------|
| [ANLDAQ.md](ANLDAQ.md) | DAQ GUI (PyQt6) + TCP data receiver; EPICS CA config per system; data flow; TCP protocol proof; trigger setup scripts (5-stage); softIOC (JustGlobals.db, dgsSupport.db); GUI windows (MTRG, Det, scalar, SYS) |
| [guceiver.md](guceiver.md) | Guceiver: live diagnostic GUI (waveform, spectrum, TAC-II, raw data) — connects to IOC TCP:9001 |
| [dgs_analysis.md](dgs_analysis.md) | Post-experiment analysis: fastEventConstructor (ROOT), parquet_pysort |
| [gebsort.md](gebsort.md) | GEBSort: event builder/sorter, GEBMerge, DGS calibration workflow (find_MK, fwhm_onepeak, dgs_ecal), GEBSort.chat config |
| [data_structures.md](data_structures.md) | Binary data structures: GEBHeader, DIG event payload (all 13 words, all HEADER_TYPE modes), TAC-II TDC, UniqueID convention, full event flow |
| [run_procedures.md](run_procedures.md) | Typical DGS run procedures: directory setup, GEBSort, PZ/energy calibration workflow |
| [pole_zero.md](pole_zero.md) | Pole-zero correction: physics, PZ coefficient, pz_from_parquet.py workflow, GrayCAL method, PQDecode.chat config |
| [troubleshooting.md](troubleshooting.md) | DGS troubleshooting: IOC connectivity, SYNC bit gotcha, FIFO issues, timestamp sync errors |
| [expMemory_2008_Chiara.md](expMemory_2008_Chiara.md) | Experiment log: exp2008_Chiara (active); run data locations, cleanup log, monitoring |

### FPGA Firmware

| File | Description |
|------|-------------|
| [fpga.md](fpga.md) | FPGA firmware overview: DIG/RTRG/MTRG hierarchy, signal flow, end-to-end timing, build types, auxiliary firmware (FPGA/others/) |
| [deep_fpga_DIG.md](deep_fpga_DIG.md) | DIG firmware: Spartan-3, ADC pipeline, discriminators, event packet format, pole-zero correction |
| [deep_fpga_DIG_channel.md](deep_fpga_DIG_channel.md) | DIG per-channel signal processing: LED/CFD discriminator modes, delay chain, pileup, VME FPGA, IP cores (split from deep_fpga_DIG.md) |
| [DIG_firmware_expert.md](DIG_firmware_expert.md) | DIG firmware expert guide: all modes, trigger_mux_select (IntAcptAll/ExtTTL/ExtTTCL/Diag), aux I/O |
| [deep_fpga_RTRG.md](deep_fpga_RTRG.md) | RTRG firmware: Virtex-4, multiplicity aggregation, throttle, VME register map |
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
| [vxworks.md](vxworks.md) | VxWorks cross-compilation: build pipeline, directory structure, munch process, glossary |
| [vxworks_migration.md](vxworks_migration.md) | Migration notes from con6 (Solaris) to Ubuntu 24 |

### Collector Box & Gammasphere

| File | Description |
|------|-------------|
| [collector_fpga.md](collector_fpga.md) | Collector box FPGA firmware: CtrlFPGA (housekeeping/monitoring), StripeFPGA (relay/stripe/LED), pickoff card FPGAs (SBX Interface + Extension) |
| [collectorboxpi.md](collectorboxpi.md) | Collector box soft IOC on Raspberry Pi; HV control; PXE boot setup |
| [collectorbox_PVs.md](collectorbox_PVs.md) | CollectorBox PV list: 1,431 records/detector (887 per-detector + 544 global); GS/MOD/VME_GS/Ge_ID numbering explained |
| [collectorbox_devicesupport.md](collectorbox_devicesupport.md) | EPICS device support internals: SPI driver, CAMAC_IO link, conversion coefficients |
| [gammasphere_geometry.md](gammasphere_geometry.md) | Gammasphere array geometry: 110 GS holes, 17 rings, θ angles per hole, full hole→angle map |

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
| [sbx.md](sbx.md) | Slope Box + SBX: HV generation, signal conditioning, BGO pattern/sum, pickoff card (hardwired routing), GS_ID dongle, Pi IOC (SPI/GPIO); no FPGA |
| [myriad.md](myriad.md) | MγRIAD module: aux detector interface, NIM I/O pinout, ECL connectors, TTCL link, DGS usage |
| [digitizer_tester.md](digitizer_tester.md) | Digitizer Tester: dual 200 MHz 16-bit DAC, analog switch matrix (10ch), TTCL link, waveform generation |
| [preamp_reset_readme.md](preamp_reset_readme.md) | Preamplifier reset handling: ADC threshold detection (LOLO/HIHI), PREAMP_DELAY kill state, BGO veto gate, PARST timestamp, Frame 15 remote reset |

### Liquid Nitrogen

| File | Description |
|------|-------------|
| [lnfill.md](lnfill.md) | LN filling system: valves, tanks, fill schedule, cron jobs, Discord alerts, ops procedures (overtime, manual fill, findhose) |

### Utilities & Operations

| File | Description |
|------|-------------|
| [nfs_layout.md](nfs_layout.md) | NFS mount layout on DCS2: vol2–vol5, fs1/vol2, piserver; directory inventory, IOC py_scripts, collector box PXE MAC map |
| [utility_scripts.md](utility_scripts.md) | BGO HV tuning script, PV discovery scripts, data0 space monitor |
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
- **Admin host:** pi5-dgs (192.168.203.2)
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

*Maintained by General DGS (AI assistant). Last updated: 2026-04-10. MEMORY.md Knowledge Base Index replaced by this file as the single source of truth.*


