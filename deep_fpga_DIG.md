# DIG — Digitizer Firmware

Stability: C3 - Structural / stable

## Table of Contents

- [Target Devices](#target-devices)
- [Memory Resources](#memory-resources)
  - [Internal BRAM](#internal-bram)
  - [External Memory](#external-memory)
- [Role](#role)
- [Repository Layout](#repository-layout)
- [Build Branches](#build-branches)
- [Source Files (DGS Branch — Production)](#source-files-dgs-branch--production)
  - [Top Level](#top-level)
  - [Per-Channel Signal Processing](#per-channel-signal-processing-10-instances)
  - [Cross-Channel & Readout](#cross-channel--readout)
  - [Router Interface (SERDES)](#router-interface-serdes)
  - [Front Bus (Partner Digitizer Interface)](#front-bus-partner-digitizer-interface)
  - [Event Data Aggregation & Readout](#event-data-aggregation--readout)
  - [VME Register Interface](#vme-register-interface)
- [Architecture](#architecture)
  - [Signal Flow](#signal-flow)
  - [SERDES TX Format (DIG → Router)](#serdes-tx-format-dig--router)
  - [SERDES RX Frame Handling (Router → DIG)](#serdes-rx-frame-handling-router--dig)
  - [Clock Domains](#clock-domains)
  - [ADC Interface](#adc-interface)
  - [External Discriminator Modes (per channel)](#external-discriminator-modes-per-channel)
  - [Event Packet Format](#event-packet-format) → **[deep_fpga_DIG_eventpacket.md](deep_fpga_DIG_eventpacket.md)**
- [Per-Channel Signal Processing, VME FPGA & See Also](#per-channel-signal-processing-vme-fpga--see-also) → **[deep_fpga_DIG_channel.md](deep_fpga_DIG_channel.md)**
- [Channel_Readout_Controller.vhd — Readout Arbitration & Timing State Machine](#channel_readout_controllervhd--readout-arbitration--timing-state-machine)
- [Cross-References](#cross-references)

## Target Devices

| Device | Part | Package | Speed | Tool | Role |
|--------|------|---------|-------|------|------|
| Main FPGA | xc3s5000 (Spartan-3) | fg900 | -5 | ISE 14.7 | Signal processing, trigger interface, event readout ✅ verified 2026-04-06 — `DIG/MAIN_FPGA/BuildBranches/DGS/Source/LEFT_RIGHT.ucf:L9` |
| VME FPGA | xc3s400 (Spartan-3) | fg320 | -5 | ISE 14.7 | VME slave, main FPGA configuration ✅ verified 2026-04-17 — `DIG/VME_FPGA_ANL/Work11/vme_A32_D32.xise:L2` (`ise_version="14.7"`) + `Digitizer_VME.prm:L1` (PROMGEN O.87xd = ISE 14.x series) |

## Memory Resources

### Internal BRAM

The XC3S5000 contains **104 BRAM blocks** (~1.9 Mb total), each configurable as ✅ verified 2026-04-06 — `DIG/MAIN_FPGA/BuildBranches/SumOverRise/Work/DIGITIZER.syr:L21666` (87/104 BRAMs used in that build)
1024 × 18-bit dual-port RAM. **87 RAMB16 blocks used (83%)** in the SumOverRise build ✅ verified 2026-04-21 — `DIGITIZER.syr:L21666` (`Number of BRAMs: 87 out of 104`). The breakdown of explicitly instantiated primitives:

| Use | Blocks | Details |
|-----|--------|---------|
| Signal delay chains | 50 | 5 × `DP_BRAM_RWA_RB_1Kx18` per channel × 10 channels (P2, M×2, trigger delay×2) |
| Accepted event FIFO | ~4 | `fifo_36x1025_sepclk_pfiport_fwft` — 1025-entry, 36-bit, dual-clock, per channel (10 total, packed by ISE) ✅ verified 2026-04-15 — `Channel_Readout_Controller.vhd:L138` `cACPTD_EVENT_FIFO_DEPTH=1025`; instantiation at `Channel_Readout_Mach.vhd:L486` |
| Event header FIFO | ~2 | `fifo_36x514_comclk_pfiport_fwft` — 513-entry (not 512), 36-bit, common clock ✅ verified 2026-04-15 — `Channel_Readout_Controller.vhd:L138` comment: "the fifo_36x514_comclk_pfiport_fwft is actually 513 elements deep" |
| Register shadow BRAM | 1 | `Register_Logic.vhd` — VME register backing store |

Shorter delay stages (K, D, D3) and the PEHQ (16-entry, 324-bit) use **SRL16/SRL32
shift registers** (LUT-based), not BRAM, as their depths are ≤128 entries.

### External Memory

Two external memory components are present, on separate FPGAs:

**IDT 7007 — Asynchronous FIFO** (main digitizer FPGA)

The primary event data buffer between the 100 MHz acquisition domain and the 50 MHz
VME readout domain. All 10 channels drain their internal accepted-event FIFOs through
this single chip.

| Property | Value |
|----------|-------|
| Part | IDT 7007 |
| Data width | 36-bit ✅ verified 2026-04-08 — `Fifo.vhd:L51` (`ext_fifo_data: out std_logic_vector(35 downto 0)`) |
| Clocking | Asynchronous dual-port: WCLK = CLK100, RCLK = CLK50 ✅ verified 2026-04-08 — `Fifo.vhd:L29-30` (`clk200`, `clk100`, `fifo_read_clk`) |
| Control | `WEN_N`, `REN_N`, `OE_N`, `MRS_N` (master reset), `PRS_N` (partial reset) ✅ verified 2026-04-08 — `Fifo.vhd:L40-45` (`ext_fifo_master_reset`, `ext_fifo_partial_reset`, `ext_fifo_noe`, `ext_fifo_nread`, `ext_fifo_write`) |
| Depth | Up to 262,143 words (19-bit depth counter; max = 0x3FFFF) ✅ verified 2026-04-16 — `Fifo.vhd:L91` (`signal FIFO_DEPTH : std_logic_vector(18 downto 0); --max depth is 0x3FFFF (18 bits)`) |
| Status flags | Empty × 2, Full × 2, Half-Full, Prog. Almost-Empty, Prog. Almost-Full ✅ verified 2026-04-08 — `Fifo.vhd:L34-37` |
| Prog. full threshold | Programmable in 1/16ths of FIFO depth (`FIFO_ARB_THRESH` — 4-bit input) ✅ verified 2026-04-16 — `Fifo.vhd:L62` (`--depth in 16th of FIFO at which "prog full" occurs`) |
| VHDL interface | `Fifo.vhd` (`COMP_FIFO` entity) ✅ verified 2026-04-08 — `Fifo.vhd:L27` |

**Flash Memory** (VME FPGA)

Non-volatile storage for the main FPGA bitstream. At power-up, the VME FPGA reads
the bitstream from flash and configures the main FPGA via a serial configuration bus.
New bitstreams can be written remotely over VME.

| Property | Value |
|----------|-------|
| Bus width | 16-bit data, 24-bit address (up to 16 MB) | ✅ verified 2026-04-19 — `external_bus_controller.vhd:L48-50` (`external_address: out std_logic_vector(23 downto 0)`; `external_data_in/out: std_logic_vector(15 downto 0)`) |
| Chip enables | 3 lines (`CE[2:0]`) — supports up to 3 flash devices | ✅ verified 2026-04-19 — `external_bus_controller.vhd:L56` (`flash_CE: out std_logic_vector(2 downto 0)`) |
| Control | `WE_N`, `OE_N`, `VPEN` (program voltage enable), `RESET` | ✅ verified 2026-04-19 — `external_bus_controller.vhd:L53-55` (`flash_WE`, `flash_OE`, `flash_reset`); `register_block.vhd:L46` (`flash_vpen_out: out std_logic`) |
| VHDL interface | `external_bus_controller.vhd`, `configuration_controller.vhd` | ✅ verified 2026-04-19 — both files present in `FPGA/DIG/VME_FPGA_ANL/Source/` |
| Config bus to main FPGA | `FPGA_cclk_out`, `FPGA_serial_din`, `FPGA_program_out`, `FPGA_done_in` | ✅ verified 2026-04-19 — `A32_D32_VME_pinlock.ucf:L226-230` (all 4 signals assigned to physical pins R1/R2/T1/T2); `top_tb.vhd:L59-62` (entity port declarations) |

**Memory architecture overview:**

```
Main FPGA (XC3S5000)                       VME FPGA (XC3S400)
─────────────────────────────              ────────────────────
10× BRAM delay chains (50 blocks)          Flash memory (16-bit, 24-bit addr)
10× acptd_event_fifo (BRAM, 1K×36)            │
Event_Header_FIFO    (BRAM, 513×36)        Bitstream load at power-up
        │                                      │
        └──────────→ IDT 7007 (36-bit) ←───────┘ (shared VME bus)
                          │
                          └──→ VME A32/D32 readout
```

## Role

The Digitizer is a 10-channel waveform digitizer for germanium detector readout. It:
1. Digitizes 10 detector channels via 14-bit ADCs running at 100 MHz ✅ verified 2026-04-12 — Digitizer.vhd:L59 (`ADC_DATA_PINS: in Array_9_0_slv_13_0`, i.e. 10ch × 14-bit) + L57 ("100 MHz ADC clock") + L876 (`CLK => clk100`)
2. Runs per-channel signal processing — delay chains, threshold/CFD discriminators, pileup rejection, energy integration
3. Sends discriminator hit patterns to the Router via SERDES link
4. Receives trigger decisions back from the Router
5. Packs accepted event data (timestamps, energies, waveforms) into a 36-bit external FIFO for VME readout
6. Supports a front bus ribbon cable to share discriminator bits with an adjacent partner digitizer

## Repository Layout

```
DIG/
├── MAIN_FPGA/
│   └── BuildBranches/
│       ├── DGS/                    # Production branch (full signal processing)
│       ├── Majorana/               # Simplified variant for MAJORANA experiment
│       ├── DGS_QUAD_M_SUMS/        # Quad M-sum energy capture variant
│       ├── DGS_TAG_20180607_TWEAK/ # Dated archive tag
│       ├── DoubleSampleTag/        # Double ADC sampling variant
│       ├── DGSBramTest/            # BRAM delay chain test variant
│       └── SumOverRise/            # Energy summed over rise time variant
├── VME_FPGA_ANL/                   # VME interface and configuration FPGA
├── Sims/                           # Simulation testbenches
│   ├── Decimator/
│   ├── Filter/
│   └── VME_TB/
├── ChipScope/                      # ChipScope debug projects
├── Datasheets/                     # ADC, clock, FPGA, I/O buffer datasheets
└── Walter_Release_MDIG_6194/       # Tagged release binaries
```

Each branch under `BuildBranches/` has the same internal layout:
```
<Branch>/
├── Source/     # VHDL source files
├── Cores/      # ISE IP cores (FIFOs, BRAM, ILA)
└── Work/       # ISE project file (.xise) and build outputs (.bit, .bin)
```

## Compile-Time Build Options (Generics)

_Source: `DIG/MAIN_FPGA/Build options for the digitizer.docx` (JTA, 2016-05-27)_

These ISE project generics/parameters control what is compiled into the firmware:

| Generic | Type | Default | Description |
|---------|------|---------|-------------|
| `SLAVE_MODE` | bool | **0** (false) | Clock distribution: 0=master (use DS92LV18 clock), 1=slave (use front bus cable clock). **Always 0 for DGS production.** Code revision reads `0x4Cnn` if master, `0x4Dnn` if slave. |
| `EXPANDED_T_BUFFER` | bool | — | **Defunct (hard-coded true since 2015).** All builds have 20 µs T buffer. |
| `RUN_EXT_FIFO_AT_100MHZ` | bool | 1 | Use 100 MHz clock for external FIFO logic. Always 1 in practice; will be deprecated. |
| `INCLUDE_ILA` | bool | 0 | Enable ChipScope ILA (internal logic analyzer). Uses ~10 BRAMs, risk of timing issues. Dev/debug only. |
| `DIAG_MUX_SIZE` | int | 2 | DAC diagnostic output: 0=DAC disabled (1-2% slice savings), 1=DAC on (X waveform only), 2=DAC on with full per-channel mux. |
| `MAJORANA_MODE_FLAG` | bool | 0 | Majorana experiment mode: disables triple-filter discriminator, uses running pre/post-rise sums instead. Fixed M-buffer size set by `MAJORANA_M_SIZE`. **Off for DGS production.** |
| `MAJORANA_M_SIZE` | int | — | (Only when `MAJORANA_MODE_FLAG=1`) Fixed M-buffer size: 0=128, 1=256, 2=512, 3=1024 samples. |

**Maximal build** (for resource estimation): `SLAVE_MODE=0, RUN_EXT_FIFO_AT_100MHZ=1, INCLUDE_ILA=1, DIAG_MUX_SIZE=2, MAJORANA_MODE_FLAG=0, MAJORANA_M_SIZE=3`

---

## Build Branches

All branches target `xc3s5000-fg900-5`.

| Branch | Description | Key Differences |
|--------|-------------|-----------------|
| **DGS** | Production | Full pipeline: triple filters, CFD, coarse discriminators, diagnostic waveform mux |
| **Majorana** | MAJORANA experiment | Simplified: no CFD, no coarse discriminators, no triple filters |
| **DGS_QUAD_M_SUMS** | Quad energy capture | 4 M-sum windows instead of standard count ✅ verified 2026-04-21 — `BuildBranches/DGS_QUAD_M_SUMS/Source/Digitizer.vhd:L597` ("20220717: quad 'M' sum modification, with dual early sum capture") |
| **DoubleSampleTag** | PEHQ rework + readout refactor | Name is misleading — **no double-sampling logic found**. Primary changes (2020–2023): (1) PEHQ width made parametric (`PEHQ_WIDTH` generic; was hardcoded 340-bit); (2) PEHQ overflow/underflow protection added (2023-10-01: `a /= "1110"` guard on write, `a /= "1111"` guard on read), `PEHQ_overflow`/`PEHQ_underflow` output flags added; (3) PEHQ reset changed from asynchronous to synchronous (2023-07-25); (4) `event_packer.vhd` reset logic refactored from async to inline synchronous per-state; uses older ISE UCF constraint files (`MSTR_digitizer.ucf`, `SLAVE_digitizer.ucf`) instead of XDC. Branch appears to be a long-running engineering development branch rather than a feature branch. ✅ verified 2026-04-21 — `BuildBranches/DoubleSampleTag/Source/pehq.vhd` (PEHQ_WIDTH generic, overflow/underflow flags, sync reset); `event_packer.vhd` diff vs DGS branch |
| **SumOverRise** | Adjustable pre-rise sum timing | Adds `EARLY_PRE_M_SEL` generic to `jta_channel`: bit[8] of `reg_d3_window` selects whether the early pre-rise M-sum uses pre-rise or post-rise M+K0 as coarse delay ✅ verified 2026-04-21 — `BuildBranches/SumOverRise/Source/Digitizer.vhd:L1167` (`EARLY_PRE_M_SEL => reg_d3_window(i)(8)`) |
| **DGSBramTest** | BRAM-based energy accumulation | Uses `SUM_ACROSS_BUFFER` (`ESUM.vhd`) for BRAM-based rolling energy integration (P2, M1, M2, D sums) vs. the production DGS state-machine approach (`ESUM_STATES`). Both branches use SRL delays for P1/K/D; distinction is in energy sum implementation. ✅ verified 2026-04-21 — `BuildBranches/DGSBramTest/Source/jta_channel.vhd:L1346-1398` (5× `SUM_ACROSS_BUFFER` instances); `ESUM.vhd:L1-4` (`SUM_ACROSS_BUFFER` = "summing logic across a RAM used as a delay buffer") |

**Project files (ISE 14.7):**
- DGS: `DGS/Work/BUS_LEFT.xise` / `BUS_RIGHT.xise`
- Others: `<Branch>/Work/<name>.xise`

## Source Files (DGS Branch — Production)

**Location:** `MAIN_FPGA/BuildBranches/DGS/Source/`

### Top Level
| File | Lines | Description |
|------|-------|-------------|
| `Digitizer.vhd` | 2,391 | Top-level entity (`DIGITIZER`) — instantiates all submodules |
| `Digitizer_pkg.vhd` | — | Package with shared type definitions |
| `trigger_data_types.vhd` | — | Array type definitions |
| `trigger_comp_defs.vhd` | — | Component declarations |
| `trigger_top_comp_defs.vhd` | — | Top-level component declarations |

### Per-Channel Signal Processing (10 instances)
| File | Description |
|------|-------------|
| `jta_channel.vhd` | Per-channel pipeline: delay chain (P1, P2, M, K, D, D3 stages), threshold and CFD discriminators, pileup detection, peak finding, energy integration (PRE_RISE, POST_RISE, P2, baseline), Pending Event Queue (PEQ) |
| `chan_trigger_control.vhd` | **Trigger Rondel — PEQ state machine** (entity `trigger_rondel`). 16-entry circular buffer of pending events (discriminator timestamps + status bits); 5-machine arbiter (Filler, Remover, Searcher, Vetoer, Check); BYPASS mode for internal triggering; veto gating; throttle control. **Fully documented in `deep_fpga_DIG_channel.md` → § Trigger Rondel.** ✅ verified 2026-04-24 — `chan_trigger_control.vhd` (1,191 lines, DGS branch) |
| `thresh_disc.vhd` | Leading-edge threshold discriminator |
| `cfd_disc.vhd` | Constant Fraction Discriminator |
| `coarse_disc_count.vhd` | Coarse discriminator with count |
| `coarse_thresh_disc.vhd` | **Simplified coarse threshold discriminator** (`thresh_disc_mach`). Stripped-down version of `thresh_disc.vhd` (2023-07-24 JTA) for use as coarse trigger only. Accepts 15-bit THRESH_DISC_PROMPT/DELAYED inputs, subtracts them, applies `DISCRIMINATOR_THRESHOLD` comparison with polarity control (pos/neg/both). State machine: `IDLE → INITIAL_FALSE_EDGE → WAIT_EDGE → PULSE → WAIT_DELAY`. Includes 9-bit holdoff counter (5.12 µs max). Outputs `DISC_FLAG_OUT`, `DISC_POLARITY`, `DISCBIT_HOLDOFF_RELEASE`, `DISC_MONITOR_OUT[13:0]`. `LOCAL_THROTTLE` (M. Oberling, 2014) suppresses discriminator during excessive decimation readout. ✅ verified 2026-04-18 — `coarse_thresh_disc.vhd:L3-8` (simplified 2023-07-24), `L30-57` (entity ports), `L87-88` (9-bit holdoff, 5.12 µs), `L120-155` (edge detection + threshold comparison) |
| `baseline_tracker.vhd` | Running baseline estimation |
| `pileup_processor.vhd` | Pileup detection and rejection/acceptance logic (8-state FSM, 4-bit pileup counter, REJ/ACC halves). **Fully documented in `deep_fpga_DIG_modules.md` → § pileup_processor.** |
| `triple_filter.vhd` | Triple moving-average filter |
| `single_filter.vhd` | Single-stage moving-average filter |
| `decimator.vhd` | Waveform decimation for readout |
| `filtered_subtraction.vhd` | **Gaussian-filtered difference for LED/CFD discriminator.** Implements a 4-stage cascaded 1-2-1 FIR filter (Pascal's triangle) on the sample-pair difference `X(n) - X(n-d)` (LED mode) or `X(n) - X(n-k)` (CFD mode). Four cascaded stages give a 1-8-28-56-70-56-28-8-1 coefficient set (normalization factor 256). Each stage uses a divide-by-4 bit shift to keep output at 17 bits. Controlled by `CFD_MODE` generic. Output is `FINAL_DIFFERENCE[16:0]`. ✅ verified 2026-04-18 — `filtered_subtraction.vhd:L12-52` (filter math derivation), `L83-143` (RTL: SUB_D/SUB_K + FILTER1 + FILTER_BLOCK generate loop) |
| `pehq.vhd` | Pending Event History Queue |

### Cross-Channel & Readout
| File | Description |
|------|-------------|
| `Timestamp_Generator.vhd` | 48-bit free-running timestamp counter; synchronized to SERDES SYNC frames from Router |
| `Trigger_Mux.vhd` | Multiplexes discriminator bits from all 10 channels; routes trigger signals back to channels |
| `Channel_Readout_Mach.vhd` | Per-channel readout state machine; handles decimation and event header generation |
| `Channel_Readout_Controller.vhd` | Arbitrates readout access across all 10 channels |
| `Channel_FIFO_Readout_Mach.vhd` | Per-channel FIFO readout |
| `event_data_fifo.vhd` | Per-channel event data FIFO |
| `Event_Header_FIFO.vhd` | Event header management FIFO |

### Router Interface (SERDES)
| File | Description |
|------|-------------|
| `SERDES_TX_Mach_DGS.vhd` | Packs 10-channel discriminator bits into 18-bit SERDES frames; sends to Router. **Fully documented in `deep_fpga_DIG_modules.md` → § SERDES_TX_Mach_DGS.** |
| `SERDES_RX_Mach.vhd` | Receives and decodes 20-frame command structure from Router (Sync, Trigger, Veto, Cal, Capture frames). **Fully documented in `deep_fpga_DIG_modules.md` → § SERDES_RX_Mach.** 1,252 lines; CLK50; 5-stage prelock, 20-frame FSM; F1=Sync/ISY (48-bit TS), F2=debug/monitor data, F3–F10=8 trigger decision frames (TRIG_FLAG+TRIG_TIMESTAMP), F11–F14=spare/stripped, F15=async commands (CAL/RESET/EXT_DISC+latch mode), F16=synchronous capture command, F17=AUX detector, F18–F19=spare, F20=EOC (always checked); per-frame VETO_EVENT[9:0] on every 5th word. |
| `disparity_lookup.vhd` | DC balance disparity lookup table |

### Front Bus (Partner Digitizer Interface)
| File | Description |
|------|-------------|
| `Front_Bus.vhd` | Bidirectional discriminator bit sharing via ribbon cable to adjacent digitizer; `FRONT_BUS_LEFT` generic selects sender (TRUE) or receiver (FALSE) role ✅ verified 2026-04-14 — `Digitizer.vhd:L48` ("if set, this digitizer SENDS discriminator bits, if clear it RECEIVES them") |

### Event Data Aggregation & Readout
| File | Description |
|------|-------------|
| `event_packer.vhd` | "Accordion" FIFO controller: injects event headers then transfers waveform sample pairs into `acptd_event_fifo`; manages decimator enable/pause via timing-mark-driven logic. **Fully documented in `deep_fpga_DIG_modules.md` → § event_packer.** |
| `Fifo.vhd` | External dual-clock FIFO interface (IDT 7007, 36-bit); bridges CLK100 write domain to CLK50 VME read domain |
| `CLOCK_MANAGER.vhd` | Differential ADC/DAC clock output generation |
| `DCM_CONTROLLER.vhd` | DCM lock/unlock/reset management |

### VME Register Interface
| File | Description |
|------|-------------|
| `Lvme.vhd` | A32/D32 VME interface handler; address decoding, CS decode, FIFO control |
| `Registers.vhd` | VME-accessible register definitions |
| `Register_Logic.vhd` | Register read/write logic with shadow BRAM backing |

## Architecture

### Signal Flow

```
Router
  │  SERDES RX (18-bit DC-balanced, 50 MHz)
  ▼
SERDES_RX_Mach (Sync / Trigger / Veto / Cal / Capture)
  │
  ├───────────────────────────────────┐
  ▼                                   ▼
Timestamp_Generator (48-bit)    Trigger & Veto flags
  │  SYNC_TIMESTAMP                   │  channel enable / veto
  │                                   │
  └──────────────┬────────────────────┘
                 │
ADC_DATA[13:0] × 10 (14-bit, 100 MHz)
                 │
                 ▼
┌────────────────────────────────────────────────┐
│       Per-Channel Pipeline ×10                 │
│              jta_channel.vhd                   │
│                                                │
│  Delay chain (P1 → P2 → M → K → D → D3)       │
│       │                                        │
│  Filters (triple / single moving average)      │
│       │                                        │
│  Discriminators (LED threshold / CFD zero-X)   │
│       │                                        │
│  Pileup processor                              │
│       │                                        │
│  Pending Event Queue (16 entries)              │
│       │  energy sums + timestamp               │
│  Channel Readout Machine                       │
│       │                                        │
└───────┼────────────────────────────────────────┘
        │
        ├──────────────────────────────────────────►  SERDES_TX_Mach_DGS
        │   COARSE_DISC[9:5] + ACCEPTED_HITS[9:0]     │  SERDES TX (18-bit, 50 MHz)
        │                                             ▼
        │                                           Router
        │
        ├──────────────────────────────────────────►  Partner Digitizer
        │   FBUS_10BIT[9:0] (ribbon cable, bidir)  ◄──
        │
        ▼
Event Packer
  │  36-bit event packets
  ▼
External FIFO — IDT 7007 (36-bit, CLK100 write / CLK50 read)
  │
  ▼
LVME (VME Interface, A32/D32)
  │
  ▼
Host Computer
```

### SERDES TX Format (DIG → Router)

The DIG sends one 16-bit word per 50 MHz clock continuously — there is no frame structure upstream. The DC-balance wrapper pads the 16-bit word to 18 bits (`'0' & data & '1'`), making the physical link 18 bits wide. ✅ verified 2026-04-14 — `Digitizer.vhd:L2078` (`unbalanced_serdes_tx_data <= '0' & serdes_tx_data & '1'`); `dc_balance_mach.vhd:L32-33` (18-bit in/out). Source: `SERDES_TX_Mach_DGS.vhd`.

| Bits (wire) | Bits (data) | Field | Description |
|-------------|-------------|-------|-------------|
| 17:16 | — | DC balance | CG/POL appended by `disparity_lookup.vhd`; stripped by Router |
| 15 | 15 | SYNC_FLAG | Echo of SERDES_SYNC_FLAG (pulses high on cycle the DIG received Frame 1) |
| 14:10 | 14:10 | COARSE_DISC[9:5] | Coarse discriminator flags for channels 5–9 (pre-stretched, passed directly) |
| 9:0 | 9:0 | ACCEPTED_HITS[9:0] | Stretched accepted-hit one-shots for all 10 channels; pulse width set by `reg_disc_width` |

Note: COARSE_DISC[4:0] (channels 0–4) are **not** transmitted upstream; the Router counts multiplicity from ACCEPTED_HITS.

✅ verified 2026-04-08 — `SERDES_TX_Mach_DGS.vhd:L137-139` (20230809 tag): `TX_DATA_OUT(15)<=SERDES_SYNC_FLAG`, `TX_DATA_OUT(14:10)<=COARSE_DISC_FLAGS(9:5)`, `TX_DATA_OUT(9:0)<=DISC_TO_TRIG_ONE_SHOT`

---

### SERDES RX Frame Handling (Router → DIG)

The full 20-frame command protocol is defined in [deep_fpga_MTRG_MAIN.md — Command Frame Timing](deep_fpga_MTRG_MAIN.md#command-frame-timing). Below is a DIG-centric summary of how `SERDES_RX_Mach.vhd` responds to each frame in real time.

#### Timing of DIG responses

Each frame is 100 ns (5 words × 20 ns). The DIG's state machine clocks through the 5 words sequentially and takes action as data arrives — it does not buffer the frame. Key timing points:

- **Most signals are asserted at Word 5** (the last word, ~80 ns after the frame starts). For example, `SERDES_SYNC_FLAG`, `TRIG_FLAG`, and `SYNC_CAPTURE_FLAG` all fire on the 5th word clock.
- **Data words are latched incrementally** — e.g. the 48-bit timestamp is assembled across Words 2, 3, 4 and is complete by Word 4, one clock before the flag fires on Word 5.
- **VETO is updated 20× per cycle** — bits [9:0] of Word 5 of *every* frame carry the per-channel veto mask, so the DIG receives a fresh veto decision every 100 ns throughout the cycle.
- **Frames 3–10 are the critical sequence** — the PEQ Searcher waits for `TRIG_FLAG` + `TRIG_TIMESTAMP` from these 8 frames to accept or reject each pending event. Each frame is one independent trigger slot; unused slots carry null (0xAAAA).
- **Some Frame 15 actions outlast the frame** — `CAL_INJECT` sets a one-shot flag that fires a calibration pulse independently; the pulse duration is not bounded by the 100 ns frame.

#### DIG Actions per Frame

The table below reads as a timeline: Frame 1 arrives at t = 0, Frame 2 at t = 100 ns, and so on through Frame 20 at t = 1900 ns, completing the 2 µs cycle.

| Frame | t (ns) | Name | DIG response |
|-------|--------|------|--------------|
| 1 | 0 | Sync | Latch 48-bit SYNC_TIMESTAMP (Words 2–4); assert SERDES_SYNC_FLAG / ISYNC_FLAG at Word 5 |
| 2 | 100 | Detector Status | Latch TRIG_MON_DET_DATA (Word 1), TRIG_MON_XTRA_DATA (Word 2) |
| 3 | 200 | Trigger Decision | Latch TRIG_TIMESTAMP (Words 2–4); assert TRIG_FLAG at Word 5 if non-null |
| 4 | 300 | Trigger Decision | Same as Frame 3 — second independent trigger slot |
| 5–10 | 400–900 | Trigger Decision ×6 | Same — slots 3–8; PEQ Searcher evaluates each TRIG_FLAG as it arrives |
| 11 | 1000 | Spare | No action; words checked for lock integrity (expect 0xAAAA) |
| 12 | 1100 | Router Command | DIG sees null (Router stripped MTRG content) |
| 13 | 1200 | GRETINA Slow Data | No action (GRETINA compatibility; fixed pattern checked) |
| 14 | 1300 | Router Command | DIG sees null (same as Frame 12) |
| 15 | 1400 | Async Command | Decode command byte at Word 1; assert CAL_INJECT / FRONT_END_RESET / EXTERNAL_DISC etc. |
| 16 | 1500 | Sync Capture | Latch capture TS (Words 2–3), length (Word 4), FIFO delay (Word 5); assert SYNC_CAPTURE_FLAG |
| 17 | 1600 | Auxiliary Detector | Not decoded by DIG |
| 18–19 | 1700–1800 | Spare | No action; lock check |
| 20 | 1900 | End-of-Cycle | Check fixed pattern; relock if mismatch |

For the full word-by-word bit layout of all 20 frames, see [deep_fpga_MTRG_MAIN.md — Command Frame Timing](deep_fpga_MTRG_MAIN.md#command-frame-timing).

#### DIG-specific decoding notes

- **Frames 3–10 (Trigger Decision):** `TRIG_TYPE` is an accumulating 3-bit bitmap across all 8 frames, so multiple simultaneous trigger types within one cycle are all recorded.
- **Frame 15 (Async Command):** Module selection uses a two-stage AND/OR mask against `reg_user_package_data` to address specific digitizer boards on a shared link.
- **Frame 16 (Sync Capture):** `SYNC_CAPTURE_FLAG` is asserted on Word 5; `SYNC_CAPTURE_TS`, `SYNC_CAPTURE_LENGTH`, and `SYNC_CAPTURE_FIFO_DELAY` are held until the next capture command.
- **Frames 12 and 14:** Always arrive as null (`0xAAAA`) because the Router strips the MTRG content before forwarding.

### Clock Domains

| Clock | Frequency | Source | Used For |
|-------|-----------|--------|----------|
| CLK100 | 100 MHz | DCM ×2 from CLK50 | All channel pipelines, event packing, ADC/DAC output | ✅ verified 2026-04-09 — `LEFT_RIGHT.ucf:L47,L114` (ACQ_DCM_2X_BUFG = "100MHz main digitizer logic clock")
| CLK50 | 50 MHz | Oscillator or FBUS_CLK via DCM | SERDES TX/RX, VME readout, timestamp | ✅ verified 2026-04-09 — `LEFT_RIGHT.ucf:L21-22` (CLK50_OSC PERIOD=20ns)
| CLK200 | 200 MHz | DCM ×4 from CLK50 | Optional high-speed logic | ✅ verified 2026-04-09 — `LEFT_RIGHT.ucf:L121` (ADC_DCM uses ACQ_DCM_2X_BUFG=100MHz as input → generates 200MHz)
| ADC_CLK_P/N | 100 MHz | CLK100 (differential) | ADC chip clock | ✅ verified 2026-04-09 — `LEFT_RIGHT.ucf:L49` (ADC_DCM_CLOCK_2X_BUFG = "200MHz source of 100MHz to external FADCs")
| SERDES_TX_CLK50 | 50 MHz | CLK50 | SERDES transmit clock to Router |

### ADC Interface

- 10 channels, 14-bit resolution, 100 MHz sampling ✅ verified 2026-04-17 — `Digitizer.vhd:L57-61` (`ADC_CLK_P/N` comment says "100 MHz ADC clock"; `ADC_DATA_PINS: Array_9_0_slv_13_0` = 10×14-bit)
- Differential clock output: `ADC_CLK_P/N` ✅ verified 2026-04-17 — `Digitizer.vhd:L57-58`
- Per-channel data ready: `ADC_DRDY_PINS[9:0]` ✅ verified 2026-04-17 — `Digitizer.vhd:L60`
- Per-channel overflow: `ADC_OVR[9:0]` ✅ verified 2026-04-17 — `Digitizer.vhd:L61`
- Data captured in IOBs and re-latched for pipeline alignment ✅ verified 2026-04-18 — `Digitizer.vhd:L463-474` (`attribute iob of adc_data_iobl_0..9 : signal is "TRUE"`; also `attribute keep` to prevent SRL inference per comment "needed for IOB constraint")

### External Discriminator Modes (per channel)

Two registers work together to configure external discriminator behaviour. Both are 32-bit registers packed with per-channel fields.

#### `reg_external_disc_src` — Address `0x008` ✅ verified 2026-04-17 — `Digitizer.vhd:L1904` (`reg_external_disc_src => ... -- Address = 0x008`)

Selects **what** the external discriminator source is for each channel. 3 bits per channel, packed as channel N = bits `[(N×3+2):(N×3)]`. ✅ verified 2026-04-17 — `Digitizer.vhd:L968` (`reg_bit_slices(i) <= reg_external_disc_src((i*3)+2 downto (i*3))`)

| 3-bit value | Source |
|-------------|--------|
| 0 (`000`) | Disabled (channel 9 cannot slave to itself) |
| 1 (`001`) | Remote discriminator bit from front bus |
| 2 (`010`) | BGO pattern discriminator from front bus |
| 3 (`011`) | Timestamp-related trigger (edge selected by `reg_external_disc_mode[29:27]`) |
| 4 (`100`) | VME-requested discriminator (`VME_EXT_DISC_REQUEST`) |
| 5 (`101`) | BGO sum or coarse discriminator from front bus |
| 6 (`110`) | Router Frame 15 external discriminator (`EXT_DISC_CHAN_SELECT`) |
| 7 (`111`) | Any-other-channel-fired (ANDed with `OTHER_CHANNEL_MAPS`) |

Bit packing example: to set channel 0 = mode 6 and channel 1 = mode 1, write `0x008` = `0b...001_110` = `0x0E`.

#### `reg_external_disc_mode` — Address `0x420` ✅ verified 2026-04-17 — `Digitizer.vhd:L1930` (`reg_external_disc_mode => ... -- Address = 0x420`)

Selects **how** each channel uses its external discriminator source. 2 bits per channel, packed as channel N = bits `[(N×2+1):(N×2)]`. ✅ verified 2026-04-17 — `Digitizer.vhd:L1191` (`EXTERNAL_DISC_MODE => reg_external_disc_mode( ((i*2)+1) downto (i*2) )`)

| 2-bit value | Behaviour |
|-------------|-----------|
| `00` | Normal — external discriminator ignored; pileup timing check active |
| `01` | OR — channel fires if internal OR external discriminator fires |
| `10` | AND — channel fires only if internal AND external both fire |
| `11` | External only — channel fires on external discriminator alone; pileup timing check bypassed |

Bits [29:27] of `reg_external_disc_mode` additionally select which timestamp edge flag is used when `reg_external_disc_src` = 3 (timestamp mode). ✅ verified 2026-04-17 — `Digitizer.vhd:L1001` (`external_disc_flag(i) <= ts_edge_flags(conv_integer(reg_external_disc_mode(29 downto 27)))`)

### Event Packet Format

→ Full detail in **[deep_fpga_DIG_eventpacket.md](deep_fpga_DIG_eventpacket.md)** — LED header layout, CFD differences, waveform samples, pole-zero correction, integration timelines.

**Summary:** When a trigger is accepted, the channel readout machine assembles a 14-word packet and writes it to the per-channel `acptd_event_fifo`. After arbitration across all 10 channels, data flows through the IDT 7007 external FIFO to VME readout. Each packet: words 0–13 header (LED or CFD format) + optional waveform samples (`reg_raw_data_length` words).

| Format | HEADER_TYPE | Words 4–7 | Extra fields |
|--------|-------------|-----------|--------------|
| LED | `0100` | TS_OF_LAST_EVENT[47:0], PU flags, TRIG_MON | Standard |
| CFD | `0101` | CFD_SAMPLE[0/1/2], TS_OF_LAST_EVENT[29:0] | CVD_VALID, TSM_FLAG |

### Channel_Readout_Controller.vhd — Readout Arbitration & Timing State Machine

_Source: `FPGA/DIG/MAIN_FPGA/BuildBranches/DGS/Source/Channel_Readout_Controller.vhd` (697 lines). Fully read 2026-04-20._

One instance per channel (10 total). Manages the **go/no-go decision** and **waveform position timing** for each accepted event before it enters the `acptd_event_fifo`. Author: Michael Oberling.

**Key constants:**
- `cHEADER_SIZE = 28` — number of header words injected ahead of waveform ✅ verified 2026-04-20 — `Channel_Readout_Controller.vhd:L136`
- `cREPORTED_HEADER_SIZE = 26` (= cHEADER_SIZE − 2) — per GRETINA's packet length definition ✅ verified 2026-04-20 — `Channel_Readout_Controller.vhd:L137`
- `cT_BUFFER_SIZE = 2048` — size of the T waveform delay buffer (20.48 µs at 100 MHz) ✅ verified 2026-04-20 — `Channel_Readout_Controller.vhd:L138`
- `cACPTD_EVENT_FIFO_DEPTH = 1025` — accepted event FIFO depth ✅ verified 2026-04-20 — `Channel_Readout_Controller.vhd:L139`

**Readout timing pipeline (3 stages):**

1. **Stage 1** — Latch `READOUT_WINDOW` (forced even), `READOUT_PRETRIGGER`, and compute `WAVEFORM_DELAY = M + T = INTERNAL_READOUT_OFFSET + cT_BUFFER_SIZE`.
2. **Stage 2** — Compute `WAVEFORM_WINDOW_WIDTH = READOUT_WINDOW − cREPORTED_HEADER_SIZE` (or 0 if window < header); `PACKET_LENGTH = READOUT_WINDOW`; `WAVEFORM_WINDOW_OFFSET = WAVEFORM_DELAY − READOUT_PRETRIGGER`.
3. **Stage 3** — Apply CFD or LED `TRIGGER_OFFSET_CALIBRATION` (LED=17 clock ticks, generic constant) to produce `WAVEFORM_WINDOW_OFFSET`. Also sets `ACPTD_EVENT_FIFO_FULL_THRESH` = FIFO depth − `PACKET_LENGTH/2 − 1` (uses `EMPTY` flag instead if packet is very large).

**Timestamp comparison (rollover-safe):** `READOUT_START_TIMESTAMP = event_timestamp + WAVEFORM_WINDOW_OFFSET`. `EVENT_READOUT_READY_FLAG` fires when `CURRENT_TIMESTAMP − READOUT_START_TIMESTAMP ≥ 0x800000` (i.e., time has come for readout to start). `OFFSET_COMPARISON` is set if the event was late (timestamp already passed). ✅ verified 2026-04-20 — `Channel_Readout_Controller.vhd:L371-401`

**State machine (`READOUT_CONTROL_STATE`, 7 states):** ✅ verified 2026-04-20 — `Channel_Readout_Controller.vhd:L144-150`

| State | Description |
|-------|-------------|
| `WAIT_FOR_EVENT_FIFO_DELAY` | 1-clock pipeline flush after processing an event; returns to WAIT_FOR_EVENT_FIFO |
| `WAIT_FOR_EVENT_FIFO` | Polls `xxNEXT_EVENT_DATA_READY`; routes to NORMAL or EXTENDED wait based on pileup flag |
| `NORMAL_EVENT_WAIT` | Waits for PEQ decision (accept/reject) + `EVENT_READOUT_READY_FLAG`; drops rejected events |
| `EXTENDED_EVENT_WAIT` | Uses **previous** event's decision for pileup extensions; drops if `PILEUP_EXTEND_ENABLE=0` or if `DROP_TRAIN` is set |
| `EVENT_READ_TRIG` | Final go/no-go based on `EVENT_EXTEND_MODE` (see below); triggers `READ_EVENT` or drops |
| `DROP_THIS_EVENT` | Drops event (no counter increment); returns to WAIT_FOR_EVENT_FIFO_DELAY |
| `DROP_AND_COUNT_THIS_EVENT` | Drops and increments `DROPPED_EVENT_COUNT`; sets `DROP_TRAIN` to drop rest of pileup train |

**`EVENT_EXTEND_MODE` (2-bit register, applied in `EVENT_READ_TRIG` state):**

| Mode | Name | Behavior |
|------|------|----------|
| `00` | Drop offset | Offset events → DROP_AND_COUNT; non-offset events → read if FIFO ready |
| `01` | Full extension | Read full waveform regardless of offset; drop only if FIFO full |
| `10` | Truncation | Waveform truncated by `SHIFTED_EVENT_OFFSET_TIME`; reads only remaining portion |
| `11` | Headers only | Offset events → headers only (no waveform); non-offset → normal readout |

✅ verified 2026-04-20 — `Channel_Readout_Controller.vhd:L564-650`

**`WAVEFORM_VETO`:** Set in `NORMAL_EVENT_WAIT` when `PILEUP_WAVEFORM_ONLY=1` but event is not a pileup event. Causes `EVENT_READ_TRIG` to set `LAST_EVENT_OFFSET_TIME = WAVEFORM_WINDOW_WIDTH` → zero waveform samples, header only.

**`DROP_TRAIN` signal (added 2014-06-25):** Once any event in a pileup train is dropped (FIFO full or offset), `DROP_TRAIN` is asserted to force all subsequent pileup extensions to also be dropped. Prevents partial pileup train readout, which would corrupt event correlation. ✅ verified 2026-04-20 — `Channel_Readout_Controller.vhd:L217-228`

**Dropped event counter:** Counts only modes #3 (offset event in mode 00) and #4 (FIFO full). Rejected events (#1) and disabled pileup extensions (#2) are dropped silently. Uses `sync_capture_counter` instance with rate/accumulate mode switchable via `DROPPED_EVENT_COUNTER_MODE`. ✅ verified 2026-04-20 — `Channel_Readout_Controller.vhd:L214-216,L358-368`

**`EVENT_DECISION_DROPPED` diagnostic:** Set permanently if the PEQ tries to register a decision while the 5-bit decision FIFO is full. Non-recoverable — requires reset. ✅ verified 2026-04-20 — `Channel_Readout_Controller.vhd:L238-248`

## Per-Channel Signal Processing, VME FPGA & See Also

→ Continued in **[deep_fpga_DIG_channel.md](deep_fpga_DIG_channel.md)** — LED/CFD discriminator modes, delay chain, CFD zero-crossing, pileup, VME FPGA, IP cores.

→ Additional module analysis in **[deep_fpga_DIG_modules.md](deep_fpga_DIG_modules.md)** — `SERDES_TX_Mach_DGS.vhd`, `event_packer.vhd`, `pileup_processor.vhd`, `SERDES_RX_Mach.vhd`.

---
*Source: `DGS_tools_pack/raw_FPGA/Dig*/` — VHDL source. PDF: `ANL Digitizer Firmware for Experts.pdf`. Created: 2026-04-05.*

## Cross-References

- `knowledgeBase/DIG_firmware_expert.md` — Expert guide: all readout modes, data format, timing, trigger_mux_select
- `knowledgeBase/preamp_reset_readme.md` — Preamp reset (PRK) handling: detection thresholds, CHANNEL_KILLED gate, PARST timestamp
- `knowledgeBase/ttcl.md` — TTCL spec: trigger command frames received and acted on by DIG
- `knowledgeBase/data_structures.md` — GEB binary format: DIG event packet layout
- `knowledgeBase/connectors.md` — DIG connector pinouts: RJ45 SERDES, 36-pin Aux I/O
- `knowledgeBase/VME_registers.md` — DIG VME register addresses from asyn driver source
- `knowledgeBase/fpga.md` — FPGA system overview: DIG role in 3-tier trigger hierarchy

*Created: 2026-04-07 | Last reviewed: 2026-04-24*
