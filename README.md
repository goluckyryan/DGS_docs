# DGS Knowledge Base

This repository contains technical documentation for the **Digital Gamma-ray Spectrometer (DGS)** system at **Argonne National Laboratory (ANL)**. It is maintained by the DGS AI assistant (General DGS) and serves as a structured, searchable knowledge base for system architecture, firmware, hardware, EPICS control, and operations.

**Official wiki:** https://wiki.anl.gov/gsdaq (may be outdated — cross-check with source code when in doubt)  
**Contact:** Ryan Tang (`ttang@anl.gov`) — tech lead / architect

---

## Contents

### System Overview

| File | Description |
|------|-------------|
| [overview.md](overview.md) | Full system architecture, hardware inventory, IP map, subsystem map |
| [hardware_architecture.md](hardware_architecture.md) | Hardware breakdown: DuoGe vs DGS, signal chain, collector box architecture |
| [hardware_drawings.md](hardware_drawings.md) | Index of schematics, datasheets, and PDF documentation |
| [wiki_gsdaq.md](wiki_gsdaq.md) | Index of the ANL GS DAQ wiki with key facts and known inaccuracies |

### DAQ & Data Acquisition

| File | Description |
|------|-------------|
| [ANLDAQ.md](ANLDAQ.md) | DAQ GUI (PyQt6) + TCP data receiver; EPICS CA config per system; data flow; TCP protocol proof |
| [dgs_analysis.md](dgs_analysis.md) | Post-experiment analysis: fastEventConstructor (ROOT), parquet_pysort |
| [run_procedures.md](run_procedures.md) | Typical DGS run procedures: directory setup, GEBSort, PZ/energy calibration workflow |
| [troubleshooting.md](troubleshooting.md) | DGS troubleshooting: IOC connectivity, SYNC bit gotcha, FIFO issues, timestamp sync errors |
| [expMemory_2008_Chiara.md](expMemory_2008_Chiara.md) | Experiment log: exp2008_Chiara (active); run data locations, cleanup log, monitoring |

### FPGA Firmware

| File | Description |
|------|-------------|
| [fpga.md](fpga.md) | FPGA firmware overview: DIG, RTRG, MTRG hierarchy |
| [raw_fpga_DIG.md](raw_fpga_DIG.md) | DIG firmware: Spartan-3, ADC pipeline, discriminators, event packet format, pole-zero correction |
| [DIG_firmware_expert.md](DIG_firmware_expert.md) | DIG firmware expert guide: all modes, trigger_mux_select (IntAcptAll/ExtTTL/ExtTTCL/Diag), aux I/O |
| [raw_fpga_RTRG.md](raw_fpga_RTRG.md) | RTRG firmware: Virtex-4, multiplicity aggregation, throttle, VME register map |
| [raw_fpga_MTRG.md](raw_fpga_MTRG.md) | MTRG overview: 3 devices (Main FPGA, VME FPGA, CPLD) |
| [raw_fpga_MTRG_MAIN.md](raw_fpga_MTRG_MAIN.md) | MTRG Main FPGA: trigger algorithms, 20-frame command structure, TAC-II TDC, VME map, RF→NIM IN 2 |
| [raw_fpga_MTRG_VIVADO.md](raw_fpga_MTRG_VIVADO.md) | MTRG Vivado port: Kintex UltraScale XCK060 |
| [raw_fpga_MTRG_VME.md](raw_fpga_MTRG_VME.md) | MTRG VME FPGA: Spartan-3, A32/D32 slave, FPGA config manager |
| [raw_fpga_MTRG_CPLD.md](raw_fpga_MTRG_CPLD.md) | MTRG CPLD: XC9500XL, fast strobe multiplicity logic (~1 µs latency) |
| [raw_fpga_building.md](raw_fpga_building.md) | Build toolchain: ISE 14.7 / Vivado 2018.3, Ubuntu 24.04 Docker approach |
| [link_sys_analysis.md](link_sys_analysis.md) | link_sys.py: 5-stage SERDES link init + timestamp sync sequence |
| [ttcl.md](ttcl.md) | TTCL spec (v2.1): word/frame/cycle format, all 20 frames, selective propagation |
| [register_maps.md](register_maps.md) | EPICS register maps for DIG/RTRG/MTRG |

### Hardware & Connectors

| File | Description |
|------|-------------|
| [digitizer_connectors.md](digitizer_connectors.md) | Digitizer connector pinouts: RJ45 (TTCL/SER/DES), 36-pin Aux I/O (ExtTTL = AUX_DIN[10] pins 32/33) |
| [MTRG_connectors.md](MTRG_connectors.md) | MTRG/RTRG connector pinouts: 125-pin SERDES (links A–H/L/R/U), NIM I/O (NIM IN 1=aux trig, NIM IN 2=TDC/RF), CPLD ribbons |

### EPICS IOC

| File | Description |
|------|-------------|
| [ioc.md](ioc.md) | EPICS IOC config, boot scripts, firmware versions, MVME5500 setup |
| [vxworks.md](vxworks.md) | VxWorks cross-compilation summary |
| [vxworks_README.md](vxworks_README.md) | Full README: build pipeline, directory structure, munch process, glossary |
| [vxworks_migration.md](vxworks_migration.md) | Migration notes from con6 (Solaris) to Ubuntu 24 |

### Collector Box & Gammasphere

| File | Description |
|------|-------------|
| [collectorboxpi.md](collectorboxpi.md) | Collector box soft IOC on Raspberry Pi; HV control; PXE boot setup |
| [collectorbox_PVs.md](collectorbox_PVs.md) | CollectorBox PV list: 1,437 records/detector; GS/MOD/VME_GS/Ge_ID numbering explained |
| [collectorbox_devicesupport.md](collectorbox_devicesupport.md) | EPICS device support internals: SPI driver, CAMAC_IO link, conversion coefficients |
| [gammasphere_geometry.md](gammasphere_geometry.md) | Gammasphere array geometry: 110 GS holes, 17 rings, θ angles per hole, full hole→angle map |

### Process Variables (PVs)

| File | Description |
|------|-------------|
| [DGS_PVs.md](DGS_PVs.md) | DGS system PV list |
| [DUO_PVs.md](DUO_PVs.md) | DuoGe system PV list |
| [Xarray_PVs.md](Xarray_PVs.md) | X-Array system PV list |
| [teststand_PVs.md](teststand_PVs.md) | Test stand PV list |

### Liquid Nitrogen & Support

| File | Description |
|------|-------------|
| [lnfill.md](lnfill.md) | Liquid nitrogen filling system; valves, tanks, cron jobs, Discord alerts |
| [myriad.md](myriad.md) | MγRIAD module: aux detector interface, NIM I/O pinout, ECL connectors, TTCL link, DGS usage |
| [digitizer_tester.md](digitizer_tester.md) | Digitizer Tester: dual 200 MHz 16-bit DAC, analog switch matrix (10ch), TTCL link, waveform generation |
| [preamp_reset_readme.md](preamp_reset_readme.md) | Preamplifier reset handling |
| [sbx.md](sbx.md) | Slope Box Extension (SBX): signal conversion, BGO pattern, pickoff card, GS_ID dongle, HV map |

### Legacy & Reference

| File | Description |
|------|-------------|
| [DGS_SVN.md](DGS_SVN.md) | Legacy SVN archive; historical reference |
| [PDF_index.md](PDF_index.md) | Index of all PDF documentation in DGS_tools_pack |

---

## Key Facts

- **System:** Digital Gamma-ray Spectrometer at ATLAS accelerator, ANL
- **Detectors:** 110 HPGe (Gammasphere) + BGO Compton shields
- **Channels:** Up to 640 (1 MTRG × 8 RTRG × 8 DIG × 10 ch)
- **Trigger cycle:** 2 µs (20 frames × 5 words at 50 MHz)
- **End-to-end latency:** ~1.3–1.5 µs
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

*Maintained by General DGS (AI assistant). Last updated: 2026-04-05.*
