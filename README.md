# DGS Knowledge Base

Stability: C2 - Active / semi-stable

This repository contains technical documentation for the **Digital Gamma-ray Spectrometer (DGS)** system at **Argonne National Laboratory (ANL)**. It is maintained by the DGS AI assistant (General DGS) and serves as a structured, searchable knowledge base for system architecture, firmware, hardware, EPICS control, and operations.

**Official wiki:** https://wiki.anl.gov/gsdaq (may be outdated — cross-check with source code when in doubt)  
**Contact:** Ryan Tang (`ttang@anl.gov`) — tech lead / architect

---

## 📁 Guides by Audience

| Folder | Purpose |
|--------|---------|
| [`operator_guides/`](operator_guides/README.md) | Step-by-step procedures for operators and physicists — what to set and in what order |
| [`engineer_guides/`](engineer_guides/README.md) | Deep technical references for hardware, firmware, and software internals |
| [`proposals/`](proposals/README.md) | PAC-approved experiment proposals (PDF + summary MD per experiment) |

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
| [analog_gammasphere.md](analog_gammasphere.md) | **Legacy** Analog Gammasphere DAQ: startup procedure, c1.cmd format, GSSort online monitoring, VXI processor boot params (dgs6, niCpu030-t, 192.168.203.170–173), CES event builder, vs-DGS comparison |

### DAQ & Data Acquisition

| File | Description |
|------|-------------|
| [ANLDAQ.md](ANLDAQ.md) | DAQ GUI (PyQt6) overview: EPICS CA config, VxWorks data pipeline (inLoop/outLoop/MiniSender), softIOC (JustGlobals.db, dgsSupport.db), class_PV/Board, findAllPV, commander.py |
| [ANLDAQ_tcpReceiver.md](ANLDAQ_tcpReceiver.md) | tcpReceiver deep-dive: 3 binaries, TCP protocol proof, data flow, GEB header, class_DIG.h/class_TDC.h decoders, run control scripts, legacy receivers, packet consistency table |
| [ANLDAQ_tcpReceiver_Aux.md](ANLDAQ_tcpReceiver_Aux.md) | tcpReceiver/Aux offline ROOT analysis framework: `class_DIG.h` (DIG event decoder), `class_TDC.h` (TAC-II vernier timing, TDC200MHz, 50 ps resolution), `reader.h` (binary file reader, GEB/raw modes), `script.cpp` (DIG+TAC correlation, CFD zero-crossing), `script_LED.cpp` (LED-mode variant) |
| [ANLDAQ_GUI_windows.md](ANLDAQ_GUI_windows.md) | GUI window reference: gui_MTRG (5 tabs), gui_Det (collector box map), gui_scalar, gui_RTR, gui_Board (generic PV table), gui_CH (per-channel 5-tab), gui_RAM, gui_SYS, gui_LinkSys, gui_DataTaking |
| [ANLDAQ_commander.md](ANLDAQ_commander.md) | commander.py deep-dive: top-level run control GUI, startup/env, board init, run start/stop flow, duration/repeat modes, SoftIOC auto-spawn, IOC terminal access, script runner, RunTimestamp CSV log |
| [ANLDAQ_gui_internals.md](ANLDAQ_gui_internals.md) | ANLDAQ GUI component internals: class_PV (ref-counted CA subscriptions), class_Board (lazy channel subscribe), findAllPV/json2pv PV pipeline, CollectorBox PV utilities, gui_DataTaking run lifecycle, custom_QClasses base widgets, gui_scalar scalar monitor, class_PVWidgets (RLineEdit/RTwoStateButton/RComboBox/RRegisterDisplay); gui_MTRG.py (MTRGWindow 5-tab: trigger/veto/link/CPLD/other controls, rate counters, templateTab base); gui_RTR.py (RTRWindow 2-tab: link control + X/Y map, diagnostics, LED controls); aux.py (natural_key + make_pattern_list) |
| [ANLDAQ_gui_sys.md](ANLDAQ_gui_sys.md) | gui_SYS.py deep-dive: 5 tab classes (sysTimestampReadOutTab, sysLinktab, sysTCPTab, sysCodeRevisionTab, globalSettingTab), per-board PV name conventions (MTRG/RTRG/DIG), global channel parameter mass-set (20 params, Ge/BGO/GeS/Aux detector types), board settings (15 params, all-boards write), TCP buffer monitoring |
| [trig_setup_scripts.md](trig_setup_scripts.md) | 5-stage trigger setup scripts (trig_setup_Stage1–5.sh): full step-by-step MTRG→RTRG→DIG link initialization, SYSTEM_DEFINES.sh GS topology (4 RTRGs, 44 DIGs, MTRG in VME10), DC balance/fiber notes, algorithm reference |
| [guceiver.md](guceiver.md) | Guceiver: live diagnostic GUI (waveform, spectrum, TAC-II, raw data) — connects to IOC TCP:9001 |
| [dgs_analysis.md](dgs_analysis.md) | Post-experiment analysis pipeline: EventBuilder variants (S, Q, PQ, X, XR — k-way merge, parallel, ROOT/Parquet output), `misc.h` shared utility functions (channel/energy/PZ maps, Algo1/Algo2 pole-zero correction), parquet_pysort, gray_apps summary, parquetCLI, gain_from_parquet.py, pz_from_parquet.py, RunParquet, ProcessRUN, GEB data format |
| [dgs_analysis_grayapps.md](dgs_analysis_grayapps.md) | gray_apps full reference: GrayCAL (HPGe energy calibration GUI, core modules, polezero_dialog), GrayMAN (multi-peak spectrum analysis), grayfit (AutoFitter, FittingRunner, PeakFinder, pole_zero_fitter, FitResult hierarchy) — split from dgs_analysis.md |
| [dgs_analysis_root_scripts.md](dgs_analysis_root_scripts.md) | ROOT analysis scripts (fastEventConstructor): analyzer.cpp (γ-γ coincidence), analyzer_tac.cpp (TAC-II), analyzer_trace.cpp (waveform/PSD), analyzer_pz_cal.cpp (PZ from traces), Cali_e.C (energy calibration, 110 dets), checkTACFile.cpp (TAC binary validator), findMapping.sh/findGS.sh (GS channel map tools) — split from dgs_analysis.md |
| [dgs_analysis_working.md](dgs_analysis_working.md) | dgs_analysis/working/ experiment-specific tools: `parquetCLI` (interactive Parquet REPL — plotting, gating, calibration, scripting), `ProcessRUN` (primary pipeline via EventBuilder_X), `RunParquet` (legacy Python pipeline), `gain_from_parquet.py`, `pz_from_parquet.py`, `pz_from_evtparquet.py`, `DownloadRaw.sh`, `BenchmarkTAC2_021.sh` — split from dgs_analysis.md 2026-04-25 |
| [gebsort.md](gebsort.md) | GEBSort core: event builder/sorter overview, repo structure, GEBSort.chat config, DGS calibration workflow (find_MK, fwhm_onepeak, dgs_ecal, SZ_factor, SZ_basic_PZ), SZ_1/SZ_2 stub algorithms, bin_dgs/bin_dgs_AUX internals, jta.c/DGSEvDecompose_v3 (byte-swap, all header types 0–8/15), calibration files. Sub-files: gebsort_merge_receive.md, gebsort_additional_sorters.md, gebsort_utilities.md |
| [gebsort_merge_receive.md](gebsort_merge_receive.md) | GEBMerge (N-way timestamp merge, full chat param table), gtReceiver3/4/5 (alternative DAQ receiver, packet decoding, GEB header format), GEBClient (VxWorks→GEB TCP sender library, endianness), dmpdata (GEB file dump tool) |
| [gebsort_additional_sorters.md](gebsort_additional_sorters.md) | GEBSort non-DGS sorters: GRETINA (bin_angcor_GT, bin_DCO_GT, bin_g4sim, bin_gtcal, bin_ft, bin_final); additional detectors (bin_tac2, bin_dub, bin_XA, bin_dfma, bin_angdis, bin_ndc, bin_mux, bin_s800, bin_linpol, bin_mode3) |
| [gebsort_utilities.md](gebsort_utilities.md) | GEBSort minor utility programs and support files: listTS, GTPrint, dtbtev, DataExtract (header types 0–F), dgs_ecal2, gretTapClient, GF_veto_cube, pairProd, findAngle/findCAngle/findVector, time_stamp, temp_ge (legacy LN2 CA monitor), trig_fun (3D vector math), tlutil2 (peak-find/math), 2d_fun (2D matrix I/O), GEBCrop, GEBFilter, GEBSplit, GEBHeader, utils (24-bit int conversions), spe_fun (Radford .spe I/O), tlutil (math lib v1) |
| [data_structures.md](data_structures.md) | Binary data structures: GEBHeader, DIG event payload (all 13 words, all HEADER_TYPE modes), TAC-II TDC, UniqueID convention, full event flow |
| [run_procedures.md](run_procedures.md) | Typical DGS run procedures: directory setup, GEBSort, PZ/energy calibration workflow |
| [pole_zero.md](pole_zero.md) | Pole-zero correction: physics, PZ coefficient, pz_from_parquet.py workflow, GrayCAL method, PQDecode.chat config |
| [troubleshooting.md](troubleshooting.md) | DGS troubleshooting: IOC connectivity, SYNC bit gotcha, FIFO/PV name corrections, timestamp sync errors, BGO channel suppression by preamp-reset blanking, InfluxDB/pi5-lnFill temperature stale recovery |
| [expMemory_2008_Chiara.md](expMemory_2008_Chiara.md) | Experiment log: exp2008_Chiara (active); run data locations, cleanup log, monitoring |

### FPGA Firmware

| File | Description |
|------|-------------|
| [fpga.md](fpga.md) | FPGA firmware overview: DIG/RTRG/MTRG hierarchy, signal flow, end-to-end timing, build types, auxiliary firmware (FPGA/others/) |
| [trig_sys_sim.md](trig_sys_sim.md) | Trigger system VHDL testbench (`FPGA/others/Trig_sys_sim/`): 2× DGS MTRG (BUILD_TYPE=4) wired via fake SERDES; VME bus procedures (`bus_write`/`bus_read`), stimulus sequence, VME register address constants table, BUILD_TYPE firmware code map |
| [fpga_others_legacy.md](fpga_others_legacy.md) | Legacy/ancestor digitizer VHDL designs in `FPGA/others/`: LBL_Digitizer (LBNL GRETINA 14-bit 10-ch Spartan-3, Vincent Riot, 2006; JTA simplification pass 2011, counter component removal, clock-domain crossing bug, VME address decode, 54 MHz) and Majorana_Digitizer (ANL HEP Digital Gammasphere ancestor, XC3S5000, 2015 DGS/Majorana split, tEVENT_DATA/tCHANNEL_SLOW_DATA record types, 50 VHDL modules, CFD/triple-filter present in DGS trunk removed in Majorana branch) |
| [deep_fpga_DIG.md](deep_fpga_DIG.md) | DIG firmware: Spartan-3, ADC pipeline, SERDES, discriminators, architecture overview |
| [deep_fpga_DIG_channel.md](deep_fpga_DIG_channel.md) | DIG per-channel signal processing: LED/CFD discriminator modes, delay chain, pileup, VME FPGA, IP cores (split from deep_fpga_DIG.md) |
| [deep_fpga_DIG_eventpacket.md](deep_fpga_DIG_eventpacket.md) | DIG event packet format: LED/CFD header layout, split field reconstruction, pole-zero correction, waveform samples, integration timelines (split from deep_fpga_DIG.md) |
| [deep_fpga_DIG_modules.md](deep_fpga_DIG_modules.md) | DIG module deep-dive (Part 1): SERDES_TX_Mach_DGS, event_packer, pileup_processor, SERDES_RX_Mach (20-frame FSM), Timestamp_Generator, Trigger_Mux, Channel_Readout_Controller, Channel_Readout_Mach |
| [deep_fpga_DIG_modules2.md](deep_fpga_DIG_modules2.md) | DIG module deep-dive (Part 2): dc_balance_mach, disparity_lookup, event_data_fifo, decimator, Event_Header_FIFO (LED/CFD formats + late injection), Channel_FIFO_Readout_Mach (7-state FSM, 36×1025 FWFT FIFO), Lvme VME FSM, Registers (199-entry 0x000–0x848 register map) |
| [DIG_firmware_expert.md](DIG_firmware_expert.md) | DIG firmware expert guide: all modes, trigger_mux_select (IntAcptAll/ExtTTL/ExtTTCL/Diag), aux I/O |
| [deep_fpga_RTRG.md](deep_fpga_RTRG.md) | RTRG firmware: Virtex-4, multiplicity aggregation, throttle, VME register map |
| [vhdl/RTRG_support_modules.md](vhdl/RTRG_support_modules.md) | RTRG support VHDL modules: `overlap_mach.vhd` (HPGe/BGO coincidence FSM, 4-state, up to 2.56 µs window ✅ verified 2026-04-25), `throttle_monos.vhd` (simple 2 µs DIG throttle stretcher + ANY_THROTTLE monostable), `throttle_limiters.vhd` (enhanced throttle with 3-state hold-off limiter FSM, programmable delay 20.48 µs–hours via cascaded timers, NIM2 mux), `channel_resets.vhd` (per-channel pipeline reset with clock-domain crossing per SERDES RCLK), `Plane_bit_count.vhd` (10→4 bit combinatorial popcount LUT for plane multiplicity) |
| [260E_trigger_scheme.md](260E_trigger_scheme.md) | **RTRG side** of the 0x260E trigger chain (sections 1–4): `chan_in.vhd` (serial reception, 18-bit SERDES word, 640 ns DPRAM delay alignment, X/Y plane maps), `disc_mach.vhd` (clean/dirty/BGO-only classification), `router_data_path.vhd` (Link-L multiplicity aggregation), `TOP.VHD` (top-level integration, veto insertion, clock arch) |
| [260E_MTRG_scheme.md](260E_MTRG_scheme.md) | **MTRG side** of the 0x260E trigger chain (split from above 2026-04-26): `mt_input_channel.vhd`, `eight_mt_channel.vhd`, `sum_hits_X.vhd`, `calc_total_sum.vhd`, `top.vhd` trigger decision; full end-to-end timing with DUO example and Mermaid diagram |
| [deep_fpga_MTRG.md](deep_fpga_MTRG.md) | MTRG overview: 3 devices (Main FPGA, VME FPGA, CPLD) |
| [deep_fpga_MTRG_MAIN.md](deep_fpga_MTRG_MAIN.md) | MTRG Main FPGA: trigger algorithms, 20-frame command structure, TAC-II TDC, VME map, RF→NIM IN 2 |
| [tac2.md](tac2.md) | TAC-II TDC in MTRG: vernier interpolation (~30–50 ps), 250 MHz 4-phase clock, 64-tap delay lines, data collection state machines, 350 ns pipeline delay |
| [deep_fpga_MTRG_VIVADO.md](deep_fpga_MTRG_VIVADO.md) | MTRG Vivado port: Kintex UltraScale XCK060 |
| [deep_fpga_MTRG_VME.md](deep_fpga_MTRG_VME.md) | MTRG VME FPGA: Spartan-3, A32/D32 slave, FPGA config manager |
| [deep_fpga_MTRG_CPLD.md](deep_fpga_MTRG_CPLD.md) | MTRG CPLD: XC9500XL, fast strobe multiplicity logic (~1 µs latency) |
| [deep_fpga_building.md](deep_fpga_building.md) | Build toolchain: ISE 14.7 / Vivado 2018.3, Ubuntu 24.04 Docker approach |
| [link_sys_analysis.md](link_sys_analysis.md) | link_sys.py: 5-stage SERDES link init + timestamp sync sequence |
| [multi_system_linking.md](multi_system_linking.md) | Cross-system clock + trigger sharing (timing monarch, DEN/REN/SYNC, jitter budget, 6-hop limit, step-by-step recipe) |
| [ttcl.md](ttcl.md) | TTCL spec (v2.1): overview, physical/electrical specs, word/frame/cycle format, protocol layers, initialization (sections 1–6, 8) |
| [ttcl_frame_spec.md](ttcl_frame_spec.md) | TTCL section 7: per-frame wire-format details for all 20 frames (Sync, Trigger Decision, System Capture, etc.) |


### Hardware & Connectors

| File | Description |
|------|-------------|
| [connectors.md](connectors.md) | All connector pinouts: DIG RJ45 (TTCL/SER/DES), 36-pin Aux I/O (ExtTTL = AUX_DIN[10] pins 32/33); MTRG/RTRG 125-pin SERDES (links A–H/L/R/U), NIM I/O (NIM IN 1=aux trig, NIM IN 2=TDC/RF), CPLD ribbons. ASCII pin diagrams included. |

### EPICS IOC

| File | Description |
|------|-------------|
| [ioc.md](ioc.md) | EPICS IOC config, boot scripts, firmware versions, MVME5500 setup; VME01–12 production crate slot map (44 DIGs, 4 RTRGs, 1 MTRG), user-package-data formula, FTP/NFS server setup, CA port map, Carlware/Timware historical origin |
| [IOC_cmd.md](IOC_cmd.md) | Full IOC shell command reference: DGS custom (ProgramFlash, VMERead32, asynDigitizerConfig…), EPICS 3.14 (dbl, dbpr, casr…), asyn 4.17; safety classification; terminal server map |
| [VME_registers.md](VME_registers.md) | Complete VME register address map for DIG, MTRG, and RTRG main FPGAs + VME FPGA + flash; address patterns, bit-field notes, IOC shell usage examples |
| [EPICS_DB_templates.md](EPICS_DB_templates.md) | EPICS DB templates: all 8 `.template` files (daqCrate, daqSegment2, MDigUser/SDigUser, RTrigUser, MTrigUser, Registers variants), PV naming scheme, record counts, per-crate board instantiation, board-type and FIFO encoding tables |
| [EPICS_softIOC.md](EPICS_softIOC.md) | dgsSoftIOC (DFMA Linux IOC): JustGlobals.db fanout architecture — 177 GLBL:DIG:F00:\<param\> PV trees, 12-crate dfanout chain, global + per-detector-type parameter tables (BGOs/BGOp/GeC/GeS), PREC=3 facts, boot script |
| [EPICS_RTrig_templates.md](EPICS_RTrig_templates.md) | RTrig EPICS DB templates deep-dive: complete raw register inventory (RTrigRegisters.template), all user-facing PV groups (RTrigUser.template) — NIM/LED, LVDS pre-emphasis, channel FIFO modes, X/Y plane mapping, 80-PV XMAP/YMAP arrays, discriminator delays, throttle, SERDES power/loopback, FIFO resets, status readbacks |
| [EPICS.md](EPICS.md) | EPICS primer: record types, tools, Python integration for DGS |
| [EPICS_asyn.md](EPICS_asyn.md) | asyn driver support: caput flow diagram, port concept, worker threads, bulk writes, passive hardware callbacks |
| [EPICS_implementation_tools.md](EPICS_implementation_tools.md) | DGS/DFMA EPICS implementation tools: Excel→Python PV database generation workflow, GammaWare (LabWindows/CVI, WDGS Windows 7 laptop, JTAG/Chipscope), Carlware EDM GUI (`dgscommander`), CSS auto-generated GUIs, remote access via Sonata (`rdesktop wdgs`, `ssh dgs@dgs1`), VME peek/poke via IOC console (`d`/`m`/`reboot`) |
| [vxworks.md](vxworks.md) | VxWorks cross-compilation: build pipeline, directory structure, munch process, glossary, legacy `devGData.c`, `MergedAsynDigParams.c` DIG param registration (222 params), `FlashMaintenance.c`, `profile.c`, `equalSub.c`, `restoreSub.c` |
| [vxworks_state_machines.md](vxworks_state_machines.md) | DAQ runtime state machines: `inLoop.st` (FIFO readout, board enable), `outLoop.st` (data validation, buffer routing), `MiniSender.st` (TCP send port 9001), Port 9010 FIFO grabber plan; trigger board drivers (`asynTrigCommonDriver` base, `asynTrigRouterDriver` RTRG 188 params, `asynTrigMasterDriver` MTRG 369 params); `vmeDriverMutex` cross-driver VME bus lock; `QueueManagement.c` three-queue buffer pool |
| [vxworks_trigger_drivers.md](vxworks_trigger_drivers.md) | Trigger asyn driver deep-dive: `asynTrigCommonDriver` (base class, 1-second polling thread, VME mutex, `0xaaaa0000` sub-field mask, `address_list[]` param-to-VME-offset map), `asynTrigMasterDriver` (MTRG, 369 params, card init from reg 0x15C), `asynTrigRouterDriver` (RTRG, 188 params), firmware type code table (ftype 0–F), `asynMTrigParams.c`/`asynRTrigParams.c` textual-include parameter registration |
| [vxworks_fifo_readout.md](vxworks_fifo_readout.md) | DMA buffer architecture, trigger FIFO readout (`readTrigFIFO.c`, `CheckAndReadTrigger`), Type-F synthetic headers (trigger + digitizer), FIFO index map, DMA chunking |
| [vxworks_vme_devlayer.md](vxworks_vme_devlayer.md) | VME hardware abstraction layer: `devGVME.c` (board init, `VMERead32`/`VMEWrite32`, flash Verify/Erase/Program/Download/Configure via IOCShell), `devGData.c` (EPICS `ai`/`ao`/`bo` device support for board state), `DGS_DEFS.h` (all key constants, `daqBoard`/`daqRegister`/`rawEvt` structs, board type codes 0–15, buffer ownership enum, outLoop globals) |
| [vxworks_utility_modules.md](vxworks_utility_modules.md) | VxWorks utility modules: `profile.c` (multi-counter CPU profiler with task-switch hooks, prescale, calibration overhead subtraction), `asynDebugDriver.cpp` (generic VME peek/poke asyn driver, global debug level vars), `FlashMaintenance.c` (VME FPGA config/flash register address constants 0x0900–0x098C), `equalSub.c` (EPICS equality sub-routine for up to 12 inputs with precision scaling), `restoreSub.c` (async EPICS PV restore from save file via `fdbrestore`), `MergedAsynDigParams.c` (DIG asyn parameter registration — 380 params in 7 groups), `QueueManagement.c` (three-queue event buffer pool: RAW_Q 200 × 1 MB, OUTPUT_Q, FREE_Q; sentinel guards 0x87654321/0x12345678) |
| [vxworks_migration.md](vxworks_migration.md) | Migration notes from con6 (Solaris) to Ubuntu 24 |

### Collector Box & Gammasphere

| File | Description |
|------|-------------|
| [collector_fpga.md](collector_fpga.md) | Collector box FPGA firmware: CtrlFPGA (housekeeping/monitoring), StripeFPGA (relay/stripe/LED), pickoff card FPGAs (SBX Interface + Extension) |
| [collector_ctrlFPGA_registers.md](collector_ctrlFPGA_registers.md) | Collector Box CtrlFPGA full register interface: 141-register layout, control bit assignments, per-stripe DVI/48V/ADC monitoring, BGO FPGA voltage monitors |
| [collector_stripeFPGA_registers.md](collector_stripeFPGA_registers.md) | Collector Box StripeFPGA register map: 48 registers × 6 stripes (relay_control/irly/grly/prly, stripe_control/clock/sync/crly, tristate_control/SPI/clock-sync, stripe_status/DCM-lock/stripe-id, LED pulser, sandbox, code_revision) |
| [collector_box_fpga.md](collector_box_fpga.md) | NewBlackBox motherboard FPGAs (PSG SVN origin): ControlStripe (Spartan-3 XC3S400, per-stripe 48V power/clock/relay/SYNC/LED control, 6 per chassis, PCAL6416A I2C GPIO) and CtlFanout (Spartan-6 XC6SLX4, RPi SPI gateway, ADS1158 ADC scanning, CE/MISO 15-port fan-out) |
| [collectorboxpi.md](collectorboxpi.md) | Collector box EPICS soft IOC on Raspberry Pi (aarch64/Debian 13): PXE boot (fs2.onenet), pi0–pi3 MAC map, st_201–204.cmd generation (GenerateCmdFile.py), HV PVs, db/ templates (18 types), SPI hardware layer, systemd service, Discord integration, GS hole→pi assignments |
| [collectorboxpi_commissioning.md](collectorboxpi_commissioning.md) | Pre_EPICS_Collector commissioning utilities: Add_Remove_Detectors.sh, 15-executable inventory (Scan_DVI_*, TurnOnAllConnected, Dump_Preamp_EEPROM, Write_to_EEPROM, SPI_rw), Src/ C library internals (NonEPICS_SPI_lib.c, NonEPICS_Collector_lib.c, DPRAM_access.c, Non_EPICS_Globvars.c), ADC conversion arrays, relay control bit map |
| [DUO_setup_guide.md](DUO_setup_guide.md) | DuoGe (DUO) commissioning walkthrough (living doc): VME slot map (CRATE=66), network (tangerine/vme66), IOC boot params, vme66.cmd sequence, CA port 5080, DB templates loaded, firmware versions (TAC2+HoldOFF), EDM screen inventory, trigger scheme, receiver startup, open questions for Ryan |
| [collectorbox_PVs.md](collectorbox_PVs.md) | CollectorBox PV list: 1,431 records/detector; GS/MOD/VME_GS/Ge_ID numbering explained |
| [collectorbox_devicesupport.md](collectorbox_devicesupport.md) | EPICS device support internals: SPI driver, CAMAC_IO link, conversion coefficients (core architecture) |
| [collectorbox_devicesupport_advanced.md](collectorbox_devicesupport_advanced.md) | Advanced DTYP modules: CollectorI2C, CollectorStep, CollectorDPRSupport, CollectorCalc AI/BI, CollectorCtl_Waveform, CollectorADC |
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
| [PickoffPi_ioc.md](PickoffPi_ioc.md) | PickoffPi standalone IOC (PickoffApp_RevC): direct SPI1 to Pickoff FPGA, CAMAC_IO device support, global mailboxes, I2C command FIFO protocol, HV ramp logic, DB file descriptions, CA port 5080 |
| [slope_box_interface.md](slope_box_interface.md) | SlopeBox EPICS IOC software (SVN): PickoffApp device support, CAMAC_IO link type repurposing for SPI/GPIO, transaction format, why asyn was rejected, BGO counter scripts, original BASIC control programs (1997) |
| [deep_fpga_SBX_CtrlFPGA.md](deep_fpga_SBX_CtrlFPGA.md) | SBX Motherboard Control FPGA deep dive (Spartan-6 XC6SLX9, ISE 14.7): 24-bit SPI interface (Pi/Collector Box → FPGA), 7-bit address decode → one-hot MACH_ENABLE, 128-addr register file + 1024×16-bit DPRAM, 3× I2C buses (power board/preamp/dongle), BGO disc counters + DDR OSERDES, 3-wire slope box serial, analog switch SPI, preamp reset clamp, 48-bit TS sync FSM, firmware RevC v0x0060 (date 0x0311) |
| [pickoff_card_fpga.md](pickoff_card_fpga.md) | Pickoff Card FPGA (SBX Extension RevC, Spartan-6 XC6SLX9, ISE 14.7): full 128-register SPI map, PULSED_CONTROL/FPGA_CTL/MISC_CTL_STAT bit maps, analog peripherals (ADGS5412 gain switch, LTC1660 DAC, slope box serial), BGO 8:1 OSERDES serializer to collector, 3× I2C channels (preamp/power/dongle) via FIFOs + scanner ROMs, preamp reset clamp (TMUX6119DCNR), fake-Pi dongle mode, clock architecture (trig clock ↔ local oscillator), firmware code 0x0182/0x0914 |
| [myriad.md](myriad.md) | MyRIAD trigger expansion module (VME, Spartan-3 XC3S1000): dual FPGA (MAIN+VME), NIM/ECL I/O, 64-bit carry-chain TDC, SERDES (DS92LV18) + TTCL receive, aux detector coincidence logic, dual external FIFOs, GITMO legacy GS adapter, register map, hardware status/pulsed-ctrl/SERDES-cfg bit maps |
| [digitizer_tester.md](digitizer_tester.md) | Digitizer Tester: dual 200 MHz 16-bit DAC, analog switch matrix (10ch), TTCL link, waveform generation |
| [majorana_digitizer.md](majorana_digitizer.md) | Majorana Digitizer firmware branch (Spartan-3 XC3S5000, 2015-08-31 split): discriminator changes (CFD removed ✅; triple-filter present, 14-bit threshold — NOT 24-bit as comments suggest), baseline tracker, front bus, SERDES, external FIFO, full file inventory, comparison to DGS trunk |
| [XIA_1SFP.md](XIA_1SFP.md) | XIA 1-SFP Interface FPGA (Spartan-6 XC6SLX9, ISE 14.7): bridges XIA Pixie digitizers to DGS/GRETINA TTCL trigger; SERDES_RX_Mach (DGS_MASTER format), 48-bit timestamp sync, 5-state delayed-trigger FSM (fires NIM out at TRIG_TS + configurable offset), SPI host interface, clock fallback (oscillator↔trigger-clock), Si5324 VCXO jitter cleaner, DS92LV18 LVDS SerDes; receive-only TTCL client (no trigger messages generated) |
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
| [vhdl/MTRG_trig_algo_support.md](vhdl/MTRG_trig_algo_support.md) | `trig_algo_support.vhd` | MTRG: **Shared base component for all trigger algorithms** — dual FIFO (algo + monitor), event counter, prescaler, holdoff (2025-10-22), throttle request; implements enable/veto/prescale decision matrix |
| [vhdl/MTRG_support_modules.md](vhdl/MTRG_support_modules.md) | `timestamp.vhd`, `data_compressor.vhd`, `link_tx_block.vhd`, `remote_trig_support.vhd`, `trig_mon_collect.vhd`, `trigger_data_types.vhd` | MTRG support/infrastructure: 48-bit timestamp counter, TDC vernier compressor, DC-balanced SERDES fan-out (11 links), cross-system Link R trigger algorithm, trigger monitor FIFO collector, shared VHDL type definitions |
| [vhdl/MTRG_registers.md](vhdl/MTRG_registers.md) | `registers.vhd` | MTRG VME register map: ~120 R/W + R/O registers (CS 0x0000–0x08FC, 0x1000–0x3FFC), 3 lookup RAMs (VETO/TRIG/SWEEP), 8+8 monitor FIFOs (Mon1-8 + Chan1-8), VME FSM state machine, rate counters |
| [vhdl/MTRG_SERDES_RX_Mach.md](vhdl/MTRG_SERDES_RX_Mach.md) | `SERDES_RX_Mach_R2.vhd` | MTRG: SERDES reception state machine for inter-trigger links (L/R/U) — 20-frame FSM, 5-word prelock sequence, all frame decoders (F1 Sync/ISY, F3-F10 triggers, F11-F19 cmd/spare, F20 EOC), VETO_EVENT, LINK_IS_L generic, propagation control |
| [vhdl/MTRG_AUX_IO.md](vhdl/MTRG_AUX_IO.md) | `AUX_IO.VHD` | MTRG: AUX port mux (8-bit bidirectional AUX_A/B), NIM output mux (4 modes each), target wheel encoder interface — parallel FILTER FSM (debounce) + SSI SLIDE FSM (2 µs stepped) + 7-state SSI serial receiver, polarity inversion, BEAM_SWEEP_OUT 4 modes |
| [vhdl/MTRG_pos_finder.md](vhdl/MTRG_pos_finder.md) | `pos_finder.vhd` | MTRG TDC: thermometer-code edge position lookup — 11/12-bit slice → 4-bit position + valid flag; 2048/4096-entry ROM, 1-cycle pipeline; instantiated inside `vernier_pos_finder` |
| [vhdl/MTRG_sum_hits_XY.md](vhdl/MTRG_sum_hits_XY.md) | `sum_hits_XY.vhd` | MTRG: XY coincidence trigger — fires when both X and Y global sums simultaneously exceed VME-configurable thresholds; 2-state FSM + trig_algo_support |
| [vhdl/MTRG_comp_defs.md](vhdl/MTRG_comp_defs.md) | `trigger_comp_defs.vhd`, `trigger_top_comp_defs.vhd` | MTRG component declaration packages — all sub-design and top-level component port lists; includes `tdc_chain_cont` (4-phase 250 MHz TDC chain) port documentation |
| [vhdl/MTRG_tdc_chain_cont.md](vhdl/MTRG_tdc_chain_cont.md) | `tdc_chain_cont.vhd` | MTRG TDC chain controller — 4× carry-chain TDC units (250 MHz 4-phase), fine counters, trigger ACK resampling + accumulation (toggle-phase 50→100 MHz), WANT_NEXT_TDC latch, 5-state autosample FSM, 80→20-bit FIFO repacking, 8-word TDC event packet format, TDC_FIFO_DATA_READY, dual ILA debug blocks |
| [vhdl/MTRG_Generated_top.md](vhdl/MTRG_Generated_top.md) | `Generated_top.vhd` | MTRG top-level structural integration (6,286 L) — all 24 component instances (U1 timestamp, U2 mstr_mach, U3 link_tx_block, U4 link_init, U5 LINK_LRU_RX, U10 eight_mt_channel, TL1-8 algo slots, TRIGGER_COLLECTION, U20 registers, LINK L/R/U receivers, TDC1, TRIG_MON_COLLECTOR); trigger algo slot map; veto system (SYSTEM_VETO_STATE[15:0]); 8 monitor FIFO assignments; clock infrastructure; firmware type codes |
| [vhdl/MTRG_GITMO.md](vhdl/MTRG_GITMO.md) | `GITMO_RCV_MACH.vhd`, `GITMO_TRIGGER.vhd` | MTRG: GITMO (Gammasphere In-Test-Mode Operation) legacy VXI interface — 2-stage prelock (SERDES lock + 0xA5 frame marker), 5-word frame format (common bits: CLK10/TRIG0/RDY_BSY/EOE/TOKEN_BUSY/RUN; NIM/ECL/FERA/frame-counter rotation), 5-state GITMO_TRIGGER FSM (trigger type 0x56, 20 ns configurable delay); NIM_TRIG+TOKEN_RCVD removed 2012-01-28 |
| [vhdl/MTRG_link_init_and_input_pipeline.md](vhdl/MTRG_link_init_and_input_pipeline.md) | `link_init.vhd`, `mstr_trigger_input_pipeline.vhd` | MTRG: SERDES link initializer (7-state FSM: INIT→EN_SERDES→SYNC→WAIT_FOR_LOCK→ALL_LOCKED_UP→ACKED→ERROR; LINK_MASK, LOCK_ACK, RETRY); per-Router data unpacker (mt_pipeline: 3-word preamble + 8×12-word DIG blocks/cycle; 132-bit TEMP_REG; crystal/channel ID, hit pattern, timestamp, CC energy, throttle bits, CHAN_MASK extraction) |
| [vhdl/PROGRESS.md](vhdl/PROGRESS.md) | — | Checklist of VHDL files summarized (RTRG + MTRG) |

### Liquid Nitrogen

| File | Description |
|------|-------------|
| [lnfill.md](lnfill.md) | LN filling system: valves, tanks, fill schedule, cron jobs, Discord alerts, ops procedures (overtime, manual fill, findhose) |
| [lnfill_ioc.md](lnfill_ioc.md) | LN fill system deep internals: InfluxDB data flow, hose→detID mapping table, ln2con IOC boot tree, DetMan.py FillManifold() state machine |
| [con6_lnfill.md](con6_lnfill.md) | con6 (Solaris 10, 192.168.203.136): CVS source repo + 68040 cross-compiler for lnfill IOC; ln2con NFS boot host; archiving priority and retirement migration plan |
| [fill_monitor_user_guide.md](fill_monitor_user_guide.md) | fill_monitor user guide: adjust/report/plot/add-press/check-press subcommands, cron integration, Discord alerts (W1–W6, N1–N3), weekly log sections, flag reference |
| [fill_monitor_technical_reference.md](fill_monitor_technical_reference.md) | fill_monitor technical reference: classifier/adjuster/reporter architecture, decision flow, adjustment rules, state file, gefilltime2.dat format, tunable parameters, worked examples |

| [gs_model.md](gs_model.md) | GS_model 3D detector viewer: Three.js sphere/flat/ring views, server.py API (`/api/temps/epics`, `/api/temps/influx`, `/api/mapping`), geometry/mapping/hose-map JSON files, coordinate system |

### Utilities & Operations

| File | Description |
|------|-------------|
| [nfs_layout.md](nfs_layout.md) | NFS mount layout on DCS2: vol2–vol5, fs1/vol2, fs2/vol3, piserver; full directory inventory (experiment data, IOC py_scripts, gamln.db PV structure, legacy lnfill, EDM screens, GEBSort binaries, sbx2022tuning); collector box PXE MAC map |
| [utility_scripts.md](utility_scripts.md) | BGO HV tuning scripts (NS_scripts/slopebox_scripts), PV discovery scripts, ANLDAQ GUI helper scripts (basic_settings_LED.py, terminals); data0 space monitor (deleted — does not exist) |
| [snapshot_pv.md](snapshot_pv.md) | snapshot_pv repo: PV snapshot & watchdog utilities (Python/pyepics); `copy_pvs.sh` (SCP from gs-cse/csw/cne/cnw + collector201); `EPICS_env.sh` (port map: DGS/DUO/DXA/slopebox/DUB) |
| [influxdb_grafana.md](influxdb_grafana.md) | InfluxDB 3 + Grafana monitoring on DCS2 (192.168.203.56) |
| [esata_removable_disks.md](esata_removable_disks.md) | eSATA/USB removable disk procedures on GS/DGS Linux machines: mount, unmount, format (fdisk + mkfs.ext4), e2label, sudo config for gamuser/dgs |


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

## Stability Classes

Knowledge Base files should carry a stability label near the top of the file:

- `Stability: C1 - Operational / volatile`
- `Stability: C2 - Active / semi-stable`
- `Stability: C3 - Structural / stable`

Meaning:

- **C1 — Operational / volatile**
  - Facts likely to change day-to-day or week-to-week
  - Examples: current machine state, temporary workarounds, live operational status, recent outages, procedures actively being adjusted

- **C2 — Active / semi-stable**
  - Facts likely to change month-to-month
  - Examples: current scripts, deployed workflows, active system roles, implementation details that may drift over time

- **C3 — Structural / stable**
  - Facts expected to remain valid for 6+ months
  - Examples: hardware architecture, wiring, geometry, protocols, register maps, packet/data formats

How to use these:

- Stability is a **relevance hint**, not a truth ranking.
- For questions about **current status / latest state / what is happening now**, C1 is often most relevant.
- For questions about **current implementation or workflow**, C2 is often most relevant.
- For questions about **architecture, mapping, protocol, or long-term design facts**, C3 is often most relevant.
- If a C1 operational note conflicts with a C3 structural fact, surface the conflict explicitly instead of silently preferring one.

Important:

- **Stability is separate from verification.**
- A C3 fact can still be wrong.
- A C1 fact can still be the most relevant answer.
- Keep verification tags, sources, and review dates separate from the stability class.

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

*Maintained by General DGS (AI assistant). Last updated: 2026-04-27. MEMORY.md Knowledge Base Index replaced by this file as the single source of truth.*


