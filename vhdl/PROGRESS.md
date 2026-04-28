# VHDL Summary Progress
Stability: C2 - Active / semi-stable

Quick checklist/index of detailed VHDL module analysis pages under `knowledgeBase/vhdl/`. This file tracks coverage status rather than firmware behavior details.

## RTRG (~/FPGA_svn2git/RTRG_git/MAIN_FPGA/Source/)
- [x] chan_in.vhd → RTRG_chan_in.md
- [x] disc_mach.vhd → RTRG_disc_mach.md
- [x] overlap_mach.vhd → RTRG_overlap_mach.md
- [x] router_data_path.vhd → RTRG_router_data_path.md
- [x] TOP.VHD → RTRG_top.md

## MTRG (~/FPGA_svn2git/MTRG_git/MAIN_FPGA/trunk/Source/)
- [x] top.vhd → MTRG_top.md
- [x] eight_mt_channel.vhd → MTRG_eight_mt_channel.md
- [x] mt_input_channel.vhd → MTRG_mt_input_channel.md  (NOTE: at ~/FPGA_svn2git/MTRG_git/MAIN_FPGA/mt_input_channel.vhd)
- [x] sum_hits_X.vhd → MTRG_sum_hits_X.md
- [x] calc_total_sum.vhd → MTRG_calc_total_sum.md
- [x] MYRIAD_RCV_MACH.vhd → MTRG_MYRIAD_RCV_MACH.md  (tag 20220705)
- [x] MYRIAD_TRIGGER.vhd → MTRG_MYRIAD_TRIGGER.md     (tag 20220705)
- [x] mstr_mach.vhd → MTRG_mstr_mach.md               (Master State Machine, full 20-frame TTCL output)
- [x] local_trig_coinc.vhd → MTRG_local_trig_coinc.md (Local-vs-local trigger coincidence algorithm)
- [x] trig_algo_support.vhd → MTRG_trig_algo_support.md (Generic shared base for all MTRG trigger algorithms: dual FIFO, event counter, prescaler, holdoff, throttle)

## MTRG Support / Infrastructure Modules
- [x] timestamp.vhd → MTRG_support_modules.md (48-bit counter, test rates, slave sync)
- [x] data_compressor.vhd → MTRG_support_modules.md (TDC vernier position extractor, 4× FIFOs)
- [x] link_tx_block.vhd → MTRG_support_modules.md (DC-balanced SERDES fan-out, 11 links)
- [x] remote_trig_support.vhd → MTRG_support_modules.md (Link R / cross-system trigger algorithm)
- [x] trig_mon_collect.vhd → MTRG_support_modules.md (trigger monitor FIFO collector → Mon FIFO 7)
- [x] trigger_data_types.vhd → MTRG_support_modules.md (JTA type definitions, disp_calc, itoa)

## MTRG VME Interface
- [x] registers.vhd → MTRG_registers.md (complete VME register map: ~120 registers, 3 lookup RAMs, 8+8 monitor FIFOs, VME FSM, rate counters)

## Not Yet Analyzed
- [x] AUX_IO.VHD → MTRG_AUX_IO.md (AUX port mux, NIM outputs, target wheel encoder filter/slide FSM, SSI serial encoder receiver)
- [x] GITMO_RCV_MACH.vhd + GITMO_TRIGGER.vhd → MTRG_GITMO.md (GITMO Link L receiver: 5-word SERDES frame, 2-stage prelock, NIM/ECL/FERA/RDY_BSY/EOE decoding; GITMO_TRIGGER: 5-state FSM, 0x56 trigger type, delay countdown; NIM_TRIG/TOKEN_RCVD states removed 2012-01-28 per MPC)
- [x] cpld_trig.vhd — CPLD fast-sum trigger → MTRG_support_modules.md (thin wrapper over trig_algo_support)
- [x] jta_odelay.vhd (entity: tdc_unit_cont) — 64-element carry-chain TDC → MTRG_support_modules.md
- [x] jta_vernier_pos_finder.vhd (entity: vernier_pos_finder) — 5-stage pipelined TDC position decoder → MTRG_support_modules.md
- [x] pos_finder.vhd → MTRG_pos_finder.md (11/12-bit thermometer slice → 4-bit edge position + valid; ROM lookup, 1-cycle pipeline; used by vernier_pos_finder)
- [x] SERDES_RX_Mach_R2.vhd → MTRG_SERDES_RX_Mach.md (full 20-frame FSM: lock/prelock, all frame decoders, VETO_EVENT, sanitized output)
- [x] Generated_top.vhd → MTRG_Generated_top.md (top-level structural glue: all 24 component instances, trigger algo slot map, veto system, monitor FIFO assignments, inline logic inventory, clock infrastructure, firmware type codes)
- [x] trigger_comp_defs.vhd / trigger_top_comp_defs.vhd → MTRG_comp_defs.md (component declaration packages: all sub-design + top-level components; tdc_chain_cont key ports documented)
- [x] sum_hits_XY.vhd → MTRG_sum_hits_XY.md (XY coincidence trigger: fires when both X+Y global sums exceed thresholds simultaneously; 2-state FSM + trig_algo_support)
- [x] tdc_chain_cont.vhd → MTRG_tdc_chain_cont.md (4-phase 250 MHz carry-chain TDC controller; fine counters; trigger ACK resampling + accumulation; 5-state autosample FSM; 2× 80-bit FIFO writes per event → 8 × 20-bit output words; TDC_FIFO_DATA_READY pulse; ILA debug blocks)

## DIG (~/FPGA_svn2git/DGSDIG_git/MAIN_FPGA_ISE11/Source/ + BuildBranches/DGS/Source/)

DIG analysis is split across three knowledge-base files rather than per-module vhdl/ stubs:

- [deep_fpga_DIG.md](../deep_fpga_DIG.md) — top-level architecture, BRAM/memory, build branches, source file table; covers:
  - [x] `Digitizer.vhd` (top-level, 2,391 L) — full instantiation map
  - [x] `Digitizer_pkg.vhd` — shared type definitions
  - [x] `trigger_data_types.vhd` — array type definitions
  - [x] `trigger_comp_defs.vhd` / `trigger_top_comp_defs.vhd` — component declarations
  - [x] `Fifo.vhd` (`COMP_FIFO`) — IDT 7007 36-bit external FIFO wrapper
  - [x] `Front_Bus.vhd` — bidirectional discriminator bit sharing (LEFT/RIGHT generic)
  - [x] `jta_bram_dlybuf.vhd` — BRAM delay buffer
  - [x] `GenericReg.vhd` — generic register
  - [x] `Register_Logic.vhd` — VME register backing store (BRAM shadow)
  - [x] `CLOCK_MANAGER.vhd` / `DCM_CONTROLLER.vhd` — clock synthesis
  - [x] `Phase_Hunter.vhd` / `Phase_Hunter_SerDes.vhd` — SERDES phase alignment
  - [x] `mult17x17_u.vhd` — unsigned 17×17 multiplier
  - [x] `counter.vhd` — generic counter

- [deep_fpga_DIG_channel.md](../deep_fpga_DIG_channel.md) — per-channel signal processing pipeline; covers:
  - [x] `jta_channel.vhd` — full per-channel pipeline (P1/P2/M/K/D/D3 delay stages, CFD, baseline, PEQ, energy integration)
  - [x] `thresh_disc.vhd` — leading-edge threshold discriminator
  - [x] `cfd_disc.vhd` — Constant Fraction Discriminator
  - [x] `coarse_thresh_disc.vhd` (`thresh_disc_mach`) — coarse discriminator (simplified 2023-07-24)
  - [x] `coarse_disc_count.vhd` — coarse discriminator with count
  - [x] `baseline_tracker.vhd` — running baseline estimation
  - [x] `filtered_subtraction.vhd` — cascaded 1-2-1 FIR Gaussian filter for LED/CFD
  - [x] `pehq.vhd` — Pending Event History Queue
  - [x] `triple_filter.vhd` / `single_filter.vhd` — moving-average filters
  - [x] `dp_srl_template.vhd` — SRL delay template
  - [x] `chan_trigger_control.vhd` (`trigger_rondel`, 1,191 L) — 16-entry PEQ circular buffer, 5-machine arbiter (**Trigger Rondel** section)
  - [x] `dirty_event_sense.vhd` — dirty event detection
  - [x] `disc_led.vhd` — LED discriminator output
  - [x] `basic_capture_counter.vhd` — capture counter
  - [x] `chan_reg_lookup.vhd` — per-channel register lookup
  - [x] `Flag_Queue.vhd` — flag queue
  - [x] `jta_dpram_template.vhd` / `jta_dpram_nocnt_template.vhd` — DPRAM templates
  - [x] `shadow_registers.vhd` — register shadow logic
  - [x] `sync_capture_controller.vhd` / `sync_capture_counter.vhd` — sync capture

- [deep_fpga_DIG_modules.md](../deep_fpga_DIG_modules.md) — Part 1: signal chain & SERDES deep-dives; covers:
  - [x] `SERDES_TX_Mach_DGS.vhd` (94 L) — discriminator hit packer, 10-ch stretch + TX word format
  - [x] `event_packer.vhd` (395 L) — accordion FIFO writer, 6-state FSM, timing-mark decimation
  - [x] `pileup_processor.vhd` (359 L) — 8-state dual-half FSM, 4-bit pileup counter
  - [x] `SERDES_RX_Mach.vhd` (1,252 L) — 20-frame Router command receiver, 5-stage prelock
  - [x] `Timestamp_Generator.vhd` (268 L) — 3-state sync FSM, 48-bit TS, ISYNC vs SYNC behavior
  - [x] `Trigger_Mux.vhd` (121 L) — 4-mode TRIGGER_SELECT mux
  - [x] `Channel_Readout_Controller.vhd` (697 L) — 7-state FSM, rollover-safe 24-bit TS compare, 5×34 FIFO
  - [x] `Channel_Readout_Mach.vhd` (491 L) — structural wrapper, PEQ_BYPASS zeroing, write_flags format

### Additional DIG Modules (Analyzed)
- [x] `dc_balance_mach.vhd` — DC balance machine → **deep_fpga_DIG_modules2.md**
- [x] `disparity_lookup.vhd` — 8-bit disparity ROM → **deep_fpga_DIG_modules2.md**
- [x] `Event_Header_FIFO.vhd` — event header FIFO (LED/CFD header formats, 6-state read SM, late injection) → **deep_fpga_DIG_modules2.md**
- [x] `event_data_fifo.vhd` — accordion waveform FIFO (32-bit packing, write_toggle alignment) → **deep_fpga_DIG_modules2.md**
- [x] `decimator.vhd` — waveform decimation / averaging, 3-state FSM, PAUSE mode, dec_factor 0–7, timing marks → **deep_fpga_DIG_modules2.md**
- [x] `Channel_FIFO_Readout_Mach.vhd` — per-channel FIFO readout machine → **deep_fpga_DIG_modules2.md** (7-state FSM; stop-bit bit 32; 36×1025 FWFT collection FIFO; EVENT_BOUNDARY_FLAG)
- [x] `Channel_FIFO_Readout_Mach_Rework_WIP.vhd` — WIP rework → **deep_fpga_DIG_modules2.md** (6-state redesign, never deployed; 36×513 FIFO; no ILA/BUF)
- [x] `Lvme.vhd` — VME interface → **deep_fpga_DIG_modules2.md** (8-state VME FSM; addr decode; DTACK 3-cycle delay for reads)
- [x] `Registers.vhd` — DIG VME register map → **deep_fpga_DIG_modules2.md** (199 entries, 0x000–0x848; per-channel at 100 MHz; board-wide at 50 MHz)
- [x] `dp_srl_template.vhd` — PEHQ SRL delay wrapper → deep_fpga_DIG_modules.md (entity PEHQ; 324-bit wide × 16-deep SRL; 4-bit address counter; wraps PEHQ_SRL_DELAY primitive)

---

**Last reviewed:** 2026-04-27 (DIG section added; pos_finder, sum_hits_XY, trigger_comp_defs, tdc_chain_cont, Generated_top analyzed — all MTRG VHDL files now complete; all DIG VHDL files now complete; deep_fpga_DIG_modules.md split into Part 1 [signal chain & SERDES, 667 lines] and Part 2 [DC balance, FIFOs, VME, 605 lines] — both under 500/600 line target; fpga.md and PROGRESS.md cross-references updated; 2026-04-24 18:57 CDT: GITMO_RCV_MACH.vhd + GITMO_TRIGGER.vhd analyzed → MTRG_GITMO.md)  
**Purpose:** Coverage checklist for per-module VHDL notes under `knowledgeBase/vhdl/`  
**See also:** [fpga.md](../fpga.md), [deep_fpga_RTRG.md](../deep_fpga_RTRG.md), [deep_fpga_MTRG_MAIN.md](../deep_fpga_MTRG_MAIN.md), [deep_fpga_DIG.md](../deep_fpga_DIG.md)
