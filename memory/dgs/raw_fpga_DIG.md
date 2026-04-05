# DIG — Digitizer Firmware

## Table of Contents

- [Target Devices](#target-devices)
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
- [Memory Resources](#memory-resources)
  - [Internal BRAM](#internal-bram)
  - [External Memory](#external-memory)
- [Architecture](#architecture)
  - [Signal Flow](#signal-flow)
  - [SERDES TX Format (to Router)](#serdes-tx-format-to-router)
  - [SERDES RX Frame Types (from Router)](#serdes-rx-frame-types-from-router)
  - [Clock Domains](#clock-domains)
  - [ADC Interface](#adc-interface)
  - [External Discriminator Modes](#external-discriminator-modes-per-channel)
  - [Event Packet Format](#event-packet-format)
- [Per-Channel Signal Processing: LED and CFD Modes](#per-channel-signal-processing-led-and-cfd-modes)
  - [Common Signal Path — Delay Chain and Filtering](#common-signal-path--delay-chain-and-filtering)
  - [LED Mode — Leading-Edge Threshold Discriminator](#led-mode--leading-edge-threshold-discriminator)
  - [CFD Mode — Constant Fraction Discriminator](#cfd-mode--constant-fraction-discriminator)
  - [Mode Selection](#mode-selection)
  - [After Discrimination — PEQ and Energy Integration](#after-discrimination--peq-and-energy-integration)
  - [Pileup Detection](#pileup-detection)
  - [VME Registers for Discriminator Configuration](#vme-registers-for-discriminator-configuration)
- [VME FPGA](#vme-fpga)
- [Main FPGA Bitfiles](#main-fpga-bitfiles)
- [IP Cores](#ip-cores)

## Target Devices

| Device | Part | Package | Speed | Tool | Role |
|--------|------|---------|-------|------|------|
| Main FPGA | xc3s5000 (Spartan-3) | fg900 | -5 | ISE 14.7 | Signal processing, trigger interface, event readout |
| VME FPGA | xc3s400 (Spartan-3) | fg320 | -5 | ISE 13.4 | VME slave, main FPGA configuration |

## Memory Resources

### Internal BRAM

The XC3S5000 contains **104 BRAM blocks** (~1.9 Mb total), each configurable as
1024 × 18-bit dual-port RAM. Approximately 54–56 blocks are used (~52%):

| Use | Blocks | Details |
|-----|--------|---------|
| Signal delay chains | 50 | 5 × `DP_BRAM_RWA_RB_1Kx18` per channel × 10 channels (P2, M×2, trigger delay×2) |
| Accepted event FIFO | ~4 | `fifo_36x1025_sepclk_pfiport_fwft` — 1024-entry, 36-bit, dual-clock, per channel (10 total, packed by ISE) |
| Event header FIFO | ~2 | `fifo_36x514_comclk_pfiport_fwft` — 512-entry, 36-bit, common clock |
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
| Data width | 36-bit |
| Clocking | Asynchronous dual-port: WCLK = CLK100, RCLK = CLK50 |
| Control | `WEN_N`, `REN_N`, `OE_N`, `MRS_N` (master reset), `PRS_N` (partial reset) |
| Status flags | Empty × 2, Full × 2, Half-Full, Prog. Almost-Empty, Prog. Almost-Full |
| VHDL interface | `Fifo.vhd` (`COMP_FIFO` entity) |

**Flash Memory** (VME FPGA)

Non-volatile storage for the main FPGA bitstream. At power-up, the VME FPGA reads
the bitstream from flash and configures the main FPGA via a serial configuration bus.
New bitstreams can be written remotely over VME.

| Property | Value |
|----------|-------|
| Bus width | 16-bit data, 24-bit address (up to 16 MB) |
| Chip enables | 3 lines (`CE[2:0]`) — supports up to 3 flash devices |
| Control | `WE_N`, `OE_N`, `VPEN` (program voltage enable), `RESET` |
| VHDL interface | `external_bus_controller.vhd`, `configuration_controller.vhd` |
| Config bus to main FPGA | `FPGA_cclk_out`, `FPGA_serial_din`, `FPGA_program_out`, `FPGA_done_in` |

**Memory architecture overview:**

```
Main FPGA (XC3S5000)                       VME FPGA (XC3S400)
─────────────────────────────              ────────────────────
10× BRAM delay chains (50 blocks)          Flash memory (16-bit, 24-bit addr)
10× acptd_event_fifo (BRAM, 1K×36)            │
Event_Header_FIFO    (BRAM, 512×36)        Bitstream load at power-up
        │                                      │
        └──────────→ IDT 7007 (36-bit) ←───────┘ (shared VME bus)
                          │
                          └──→ VME A32/D32 readout
```

## Role

The Digitizer is a 10-channel waveform digitizer for germanium detector readout. It:
1. Digitizes 10 detector channels via 14-bit ADCs running at 100 MHz
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

## Build Branches

All branches target `xc3s5000-fg900-5`.

| Branch | Description | Key Differences |
|--------|-------------|-----------------|
| **DGS** | Production | Full pipeline: triple filters, CFD, coarse discriminators, diagnostic waveform mux |
| **Majorana** | MAJORANA experiment | Simplified: no CFD, no coarse discriminators, no triple filters |
| **DGS_QUAD_M_SUMS** | Quad energy capture | 4 M-sum windows instead of standard count |
| **DoubleSampleTag** | Double sampling | Samples ADC twice per clock cycle |
| **SumOverRise** | Rise-time energy | Energy sum calculated over rise time instead of fixed windows |
| **DGSBramTest** | BRAM validation | BRAM-based delay chains instead of SRL |

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
| `thresh_disc.vhd` | Leading-edge threshold discriminator |
| `cfd_disc.vhd` | Constant Fraction Discriminator |
| `coarse_disc_count.vhd` | Coarse discriminator with count |
| `baseline_tracker.vhd` | Running baseline estimation |
| `pileup_processor.vhd` | Pileup detection and rejection logic |
| `triple_filter.vhd` | Triple moving-average filter |
| `single_filter.vhd` | Single-stage moving-average filter |
| `decimator.vhd` | Waveform decimation for readout |
| `filtered_subtraction.vhd` | Filtered baseline subtraction |
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
| `SERDES_TX_Mach_DGS.vhd` | Packs 10-channel discriminator bits into 18-bit SERDES frames; sends to Router |
| `SERDES_RX_Mach.vhd` | Receives and decodes 20-frame command structure from Router (Sync, Trigger, Veto, Cal, Capture frames) |
| `disparity_lookup.vhd` | DC balance disparity lookup table |

### Front Bus (Partner Digitizer Interface)
| File | Description |
|------|-------------|
| `Front_Bus.vhd` | Bidirectional discriminator bit sharing via ribbon cable to adjacent digitizer; `FRONT_BUS_LEFT` generic selects sender (TRUE) or receiver (FALSE) role |

### Event Data Aggregation & Readout
| File | Description |
|------|-------------|
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

The DIG sends one 16-bit word per 50 MHz clock continuously — there is no frame structure upstream. The DC-balance wrapper adds 2 bits (CG/POL), making the physical link 18 bits wide. Source: `SERDES_TX_Mach_DGS.vhd`.

| Bits (wire) | Bits (data) | Field | Description |
|-------------|-------------|-------|-------------|
| 17:16 | — | DC balance | CG/POL appended by `disparity_lookup.vhd`; stripped by Router |
| 15 | 15 | SYNC_FLAG | Echo of SERDES_SYNC_FLAG (pulses high on cycle the DIG received Frame 1) |
| 14:10 | 14:10 | COARSE_DISC[9:5] | Coarse discriminator flags for channels 5–9 (pre-stretched, passed directly) |
| 9:0 | 9:0 | ACCEPTED_HITS[9:0] | Stretched accepted-hit one-shots for all 10 channels; pulse width set by `reg_disc_width` |

Note: COARSE_DISC[4:0] (channels 0–4) are **not** transmitted upstream; the Router counts multiplicity from ACCEPTED_HITS.

---

### SERDES RX Frame Handling (Router → DIG)

The full 20-frame command protocol is defined in [MTRG/MAIN_FPGA.md — Command Frame Timing](../MTRG/MAIN_FPGA.md#command-frame-timing). Below is a DIG-centric summary of how `SERDES_RX_Mach.vhd` responds to each frame in real time.

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

For the full word-by-word bit layout of all 20 frames, see [MTRG/MAIN_FPGA.md — Command Frame Timing](../MTRG/MAIN_FPGA.md#command-frame-timing).

#### DIG-specific decoding notes

- **Frames 3–10 (Trigger Decision):** `TRIG_TYPE` is an accumulating 3-bit bitmap across all 8 frames, so multiple simultaneous trigger types within one cycle are all recorded.
- **Frame 15 (Async Command):** Module selection uses a two-stage AND/OR mask against `reg_user_package_data` to address specific digitizer boards on a shared link.
- **Frame 16 (Sync Capture):** `SYNC_CAPTURE_FLAG` is asserted on Word 5; `SYNC_CAPTURE_TS`, `SYNC_CAPTURE_LENGTH`, and `SYNC_CAPTURE_FIFO_DELAY` are held until the next capture command.
- **Frames 12 and 14:** Always arrive as null (`0xAAAA`) because the Router strips the MTRG content before forwarding.

### Clock Domains

| Clock | Frequency | Source | Used For |
|-------|-----------|--------|----------|
| CLK100 | 100 MHz | DCM ×2 from CLK50 | All channel pipelines, event packing, ADC/DAC output |
| CLK50 | 50 MHz | Oscillator or FBUS_CLK via DCM | SERDES TX/RX, VME readout, timestamp |
| CLK200 | 200 MHz | DCM ×4 | Optional high-speed logic |
| ADC_CLK_P/N | 100 MHz | CLK100 (differential) | ADC chip clock |
| SERDES_TX_CLK50 | 50 MHz | CLK50 | SERDES transmit clock to Router |

### ADC Interface

- 10 channels, 14-bit resolution, 100 MHz sampling
- Differential clock output: `ADC_CLK_P/N`
- Per-channel data ready: `ADC_DRDY_PINS[9:0]`
- Per-channel overflow: `ADC_OVR[9:0]`
- Data captured in IOBs and re-latched for pipeline alignment

### External Discriminator Modes (per channel)

Two registers work together to configure external discriminator behaviour. Both are 32-bit registers packed with per-channel fields.

#### `reg_external_disc_src` — Address `0x008`

Selects **what** the external discriminator source is for each channel. 3 bits per channel, packed as channel N = bits `[(N×3+2):(N×3)]`.

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

#### `reg_external_disc_mode` — Address `0x420`

Selects **how** each channel uses its external discriminator source. 2 bits per channel, packed as channel N = bits `[(N×2+1):(N×2)]`.

| 2-bit value | Behaviour |
|-------------|-----------|
| `00` | Normal — external discriminator ignored; pileup timing check active |
| `01` | OR — channel fires if internal OR external discriminator fires |
| `10` | AND — channel fires only if internal AND external both fire |
| `11` | External only — channel fires on external discriminator alone; pileup timing check bypassed |

Bits [29:27] of `reg_external_disc_mode` additionally select which timestamp edge flag is used when `reg_external_disc_src` = 3 (timestamp mode).

### Event Packet Format

When a trigger is accepted, the channel readout machine assembles a packet and writes it into the per-channel `acptd_event_fifo` (36-bit, `Channel_Readout_Mach.vhd`). The `channel_FIFO_readout_mach` arbitrates across all 10 channels and feeds the external IDT 7007 FIFO. LVME reads the FIFO over VME (A32/D32, 32-bit words).

Each packet consists of a **14-word header** (words 0–13) followed by zero or more **waveform sample words**. Source: `Event_Header_FIFO.vhd`, `event_packer.vhd`.

#### 36-bit FIFO word framing (internal)

| Bits | Field | Description |
|------|-------|-------------|
| 35:32 | EOP | `0001` = last word of packet; `0000` = normal |
| 31:0 | Data | 32-bit header word or waveform sample pair |

Bits [35:32] are used internally for packet framing and are not forwarded to VME — the host sees 32-bit words only.

#### LED Header (14 words, words 0–13)

```
Bit layout:  31 30 29 28 27 26 25 24 23 22 21 20 19 18 17 16 15 14 13 12 11 10 09 08 07 06 05 04 03 02 01 00
Word  0:  |                              FIXED 0xAAAAAAAA (event delimiter)                               |
Word  1:  |  GeoAddr[4:0]  | PacketLen[10:0] (filled at readout) |     UserDef[11:0]     |  CH_ID[3:0]   |
Word  2:  |                              TIMESTAMP_OF_EVENT[31:0]                                         |
Word  3:  | HDR_LEN[5:0] | EVT_TYPE[2:0] | 0 |TM|PM| HEADER_TYPE[3:0] |  TIMESTAMP_OF_EVENT[47:32]       |
Word  4:  |     TS_OF_LAST_EVENT[15:0]    |PU|PO|GE|SE| 0 |OF|PV|ED| 0 |VF|WF|PE|       "00000"          |
Word  5:  |                              TS_OF_LAST_EVENT[47:16]                                          |
Word  6:  |  "0000"  | PU_CNT[3:0] |                    SAMPLED_BASELINE[23:0]                           |
Word  7:  |              TRIG_MON_DET_DATA[15:0]              |       TRIG_MON_XTRA_DATA[15:0]            |
Word  8:  | POST_RISE_SUM[7:0]  |                       PRE_RISE_SUM[23:0]                               |
Word  9:  |          TS_OF_PEAK[15:0]             |            POST_RISE_SUM[23:8]                        |
Word 10:  |       TS_OF_TRIGGER[15:0] (filled at readout)     |CP|P2|          P2_SUM[13:0]              |
Word 11:  |  PREV_POST[23:16]  |                       MPX_FIELD[23:0]                                   |
Word 12:  |  PREV_POST[15:8]   |                     EARLY_PRE_RISE_SUM[23:0]                            |
Word 13:  |  PREV_POST[7:0]    |  TS_OF_COARSE[9:0]  |CF|PC|PT|ST|          P2_SUM[23:14]               |
```

Note: `MPX_FIELD[23:0]` in Word 11 is the multiplexed field — when `CP` (Word 10 bit 15) = 0 it holds a 2nd early pre-rise energy; when CP = 1 it holds `TS_OF_LAST_PREAMP_RESET[23:0]`.

**Single-word or already-contiguous fields:**

| Field | Location | Description |
|-------|----------|-------------|
| GeoAddr[4:0] | W1[31:27] | Board geographic address (VME slot ID) |
| PacketLen[10:0] | W1[26:16] | Total packet length in 32-bit words (filled at readout) |
| UserDef[11:0] | W1[15:4] | User tag from `reg_user_package_data` |
| CH_ID[3:0] | W1[3:0] | Channel number (0–9) |
| HDR_LEN[5:0] | W3[31:26] | Header length constant = 28 |
| EVT_TYPE[2:0] | W3[25:23] | Event type (filled at readout) |
| TM | W3[21] | `TRIG_TS_MODE`: 0 = use arrival TS; 1 = use trigger-mux TS |
| PM | W3[20] | `PEQ_BYPASS`: 1 = pending event queue bypassed |
| HEADER_TYPE[3:0] | W3[19:16] | Format: `0100` = LED; `0101` = CFD |
| SAMPLED_BASELINE[23:0] | W6[23:0] | Baseline estimate latched at event time (ADC counts × M) |
| TRIG_MON_DET_DATA[15:0] | W7[31:16] | Detector trigger monitor data from Frame 2 |
| TRIG_MON_XTRA_DATA[15:0] | W7[15:0] | Extra trigger monitor data from Frame 2 |
| PRE_RISE_SUM[23:0] | W8[23:0] | Pre-peak energy integral (M samples) — see Energies |
| TS_OF_PEAK[15:0] | W9[31:16] | Lower 16 bits of 48-bit timestamp at pulse peak |
| TS_OF_TRIGGER[15:0] | W10[31:16] | Lower 16 bits of 48-bit timestamp when trigger arrived |
| TS_OF_COARSE[9:0] | W13[23:14] | Coarse discriminator timestamp (10-bit) |
| PU_CNT[3:0] | W6[27:24] | Number of simultaneous pileup events |

**Status flag bits (Word 4):**

| Bit | Abbrev | Meaning |
|-----|--------|---------|
| 15 | PU | Pileup flag: another event was in-flight when this one fired |
| 14 | PO | Pileup-waveform-only mode active |
| 13 | GE | `TS_OF_COARSE` extension bit 1 (replaced GENERAL_ERROR in 2023) |
| 12 | SE | `TS_OF_COARSE` extension bit 0 (replaced SYNC_ERROR in 2023) |
| 10 | OF | Offset flag (filled at readout) |
| 9 | PV | Peak valid: peak-finding algorithm found a clean peak |
| 8 | ED | External discriminator flag: event was triggered externally |
| 6 | VF | Veto flag (filled at readout) |
| 5 | WF | Write-flags mode: 1 = header-only, no waveform data written |
| 4 | PE | `EARLY_PRE_M_SEL`: which early pre-rise window was used |

**Status flag bits (Word 13):**

| Bit | Abbrev | Meaning |
|-----|--------|---------|
| 13 | CF | `COARSE_FIRED`: coarse discriminator fired on this event |
| 12 | PC | `PARST_TSM`: preamp reset timestamp matched |
| 10 | ST | `SECOND_THRESH`: second (higher) threshold of thresh_disc satisfied |

**Word 10 control bits:**

| Bit | Abbrev | Meaning |
|-----|--------|---------|
| 15 | CP | `CAPTURE_PARST_TS`: 1 = MPX_FIELD (Word 11[23:0]) holds `TS_OF_LAST_PREAMP_RESET` |
| 14 | P2 | `P2_MODE`: P2 sum integration mode |

#### Split field reconstruction

Several values are spread across non-contiguous bit fields. Reconstruction (all values are unsigned unless noted):

```
TIMESTAMP_OF_EVENT[47:0]   = (Word3[15:0]  << 32) | Word2[31:0]
TS_OF_LAST_EVENT[47:0]     = (Word5[31:0]  << 16) | Word4[31:16]      (LED only)
POST_RISE_SUM[23:0]        = (Word9[15:0]  <<  8) | Word8[31:24]
P2_SUM[23:0]               = (Word13[9:0]  << 14) | Word10[13:0]
PREV_POST[23:0]            = (Word11[31:24]<< 16) | (Word12[31:24]<< 8) | Word13[31:24]
```

For CFD mode (Words 4–7 differ):

```
TS_OF_LAST_EVENT[29:0]     = (Word5[13:0]  << 16) | Word4[31:16]      (CFD, 30-bit only)
TRIG_MON_DET_DATA[15:0]    = (Word4[3:0]   << 12) | (Word5[31:30]<<10) | (Word5[15:14]<<8) | Word6[31:24]
PILEUP_COUNT[3:0]          = (Word7[31:30] <<  2) | Word7[15:14]      (CFD, split around CFD samples)
```

#### Signal and integration timeline

The vertical axis is time (one tick = 10 ns at 100 MHz). The horizontal axis shows the ADC signal amplitude from a charge-sensitive germanium preamplifier: a flat baseline, a fast step rise (~100–200 ns charge collection), a flat plateau, and a slow exponential decay tail (preamp RC feedback). Energy sums and timestamps are annotated on the right.

```
  Time  Amplitude (ADC)         Integration windows         Timestamps
  ▼     0────────────────────   ─────────────────────────   ──────────────────────────
        │
        │  baseline ─────────   ┐
        │  baseline ─────────   │ EARLY_PRE_RISE_SUM         ← TS_OF_LAST_PREAMP_RESET
        │  baseline ─────────   ┘   (optional early baseline;   [23:0] (MPX_FIELD, CP=1)
        │  baseline ─────────       captured at coarse disc      anywhere in the past)
        │
        │  baseline ─────────   ┐
        │  baseline ─────────   │
        │  baseline ─────────   │ PRE_RISE_SUM
        │  baseline ─────────   │ (M samples = reg_m_window × 10 ns)
        │  baseline ─────────   │ baseline reference; subtract from POST_RISE for energy
        │  baseline ─────────   ┘
        │ /
        │/ rising edge ───────────────────────────────────   ← TIMESTAMP_OF_EVENT [47:0]
        /  (~100–200 ns                                          LED: threshold crossing
        │   charge collection)                                   CFD: sign-flip clock
        │\                                                        (±10 ns, see CFD interp.)
        │ ─── plateau ───────   ┐
        │ ─── plateau ───────   │                            ← TS_OF_PEAK [15:0]
        │ ─── plateau ───────   │ POST_RISE_SUM                  (lower 16 bits; at peak
        │ ─── slow decay ────   │ (M samples = reg_m_window × 10 ns)  of filtered signal)
        │ ─── slow decay ────   │ signal + baseline
        │ ─── slow decay ────   │ net energy ≈ POST_RISE − PRE_RISE
        │ ─── slow decay ────   ┘
        │ ─── tail ──────────   ┐
        │ ─── tail ──────────   │ P2_SUM
        │ ─── tail ──────────   ┘ (reg_p2_window × 10 ns; tail/pileup characterization)
        │  baseline ─────────
        ┊
        ┊  (2–20 µs gap)        ─────────────────────────   ← TS_OF_TRIGGER [15:0]
        ┊                                                        (lower 16 bits; trigger
        ┊                                                        accept/reject from Router)
```

**Notes on timestamps not on the main pulse timeline:**

- `TIMESTAMP_OF_EVENT [47:0]` is the full 48-bit counter value (10 ns/tick). All other `TS_OF_*` fields are the **lower N bits** only — you reconstruct absolute time by combining with the upper bits of `TIMESTAMP_OF_EVENT`.
- `TS_OF_LAST_EVENT [47:0]` (LED) — the previous discriminator fire on this channel. Precedes the current event; see two-event timeline below. `ΔT = TIMESTAMP_OF_EVENT − TS_OF_LAST_EVENT` gives inter-event interval and drives decay-tail correction of baseline windows.
- `TS_OF_COARSE [11:0]` — coarse (BGO/NaI scintillator) discriminator timestamp. Asynchronous; can fall before or after `TIMESTAMP_OF_EVENT`. Coincidence gates check `|TS_OF_COARSE − TIMESTAMP_OF_EVENT| < window`.
- `TS_OF_LAST_PREAMP_RESET [23:0]` (MPX_FIELD when CP=1) — time of last Ge preamp reset (inhibit pulse). Typically µs to ms in the past; used to assess whether the preamp was still recovering at event time.
- `EARLY_PRE_RISE_SUM` — captured at the coarse discriminator time, not at the main discriminator time. It provides a second, earlier baseline window and is useful when the preamp baseline is drifting or when standard PRE_RISE might catch the tail of a preceding pulse.

#### Two-event timeline — inter-event decay correction

`TS_OF_LAST_EVENT` and `PREV_POST` belong to the **previous event**. The preamp RC
tail from that event persists into the current event's baseline windows, biasing
`PRE_RISE_SUM` and `POST_RISE_SUM`. With known τ the packet supplies all inputs
needed to remove this bias.

```
  Time  Amplitude (ADC)         Integration windows         Timestamps / notes
  ▼     0────────────────────   ─────────────────────────   ──────────────────────────

  ── PREVIOUS EVENT ──────────────────────────────────────────────────────────────────
        │  baseline ─────────
        │ /
        │/ prev rising edge ──────────────────────────────   ← TS_OF_LAST_EVENT [47:0]
        /  (~100–200 ns)                                         (stored in current packet)
        │ ─── plateau ───────   ┐
        │ ─── slow decay ────   │ PREV_POST                      step height A_prev
        │ ─── slow decay ────   │ (POST_RISE_SUM of prev event;  ≈ PREV_POST/M − V_base
        │ ─── slow decay ────   ┘  M samples from prev peak)
        │ ─── tail ──────────   ┐
        │ ─── tail ──────────   │ (prev P2, not stored)
        │ ─── tail ──────────   ┘
        │
        ┊ ← ΔT = TIMESTAMP_OF_EVENT − TS_OF_LAST_EVENT ──────────────────────────────
        ┊   exponential tail A_prev · exp(−t/τ) contaminates baseline windows below
        │
  ── CURRENT EVENT ───────────────────────────────────────────────────────────────────
        │  "baseline"────────   ┐                           ← TS_OF_LAST_PREAMP_RESET
        │  "baseline"────────   │ EARLY_PRE_RISE_SUM            [23:0] (MPX_FIELD, CP=1)
        │  "baseline"────────   ┘ = V_base·M + tail_e           anywhere in the past
        │  "baseline"────────
        │
        │  "baseline"────────   ┐
        │  "baseline"────────   │ PRE_RISE_SUM
        │  "baseline"────────   │ = V_base·M + tail_p
        │  "baseline"────────   │ (tail_p < tail_e: more time elapsed)
        │  "baseline"────────   ┘
        │ /
        │/ rising edge ───────────────────────────────────   ← TIMESTAMP_OF_EVENT [47:0]
        /  (~100–200 ns)
        │ ─── plateau ───────   ┐
        │ ─── plateau ───────   │                            ← TS_OF_PEAK [15:0]
        │ ─── plateau ───────   │ POST_RISE_SUM
        │ ─── slow decay ────   │ = (V_base + A_curr)·M + tail_q
        │ ─── slow decay ────   │ (tail_q < tail_p: more time elapsed)
        │ ─── slow decay ────   ┘
        │ ─── tail ──────────   ┐
        │ ─── tail ──────────   │ P2_SUM
        │ ─── tail ──────────   ┘ (reg_p2_window × 10 ns)
        │  baseline ─────────
        ┊
        ┊  (2–20 µs gap)                                     ← TS_OF_TRIGGER [15:0]
```

`tail_e`, `tail_p`, `tail_q` = `A_prev · exp(−t/τ) · M` evaluated at the midpoint
of each window, where `t` is measured from `TS_OF_LAST_EVENT`.

**Packet inputs for decay-tail correction:**

| Input | Derived from | Notes |
|-------|-------------|-------|
| `ΔT` | `TIMESTAMP_OF_EVENT − TS_OF_LAST_EVENT` | Full 48-bit, 10 ns/tick |
| `A_prev` (prev step height) | `PREV_POST/M − SAMPLED_BASELINE` | Needs `reg_m_window` from run config |
| POST_RISE start offset | `TS_OF_PEAK[15:0] − TIMESTAMP_OF_EVENT[15:0]` | Variable; accounts for pulse rise time |
| `τ` | Detector calibration | Not in packet |

**Alternative — two-window baseline solve.** With τ known, EARLY_PRE and PRE provide
two measurements of the same decaying tail at different times, giving two equations
in two unknowns (V_base and A_prev), without needing PREV_POST:

```
EARLY_PRE / M = V_base + A_prev · exp(−t_e / τ)
PRE       / M = V_base + A_prev · exp(−t_p / τ)
```

The time offsets `t_e` and `t_p` (from `TS_OF_LAST_EVENT` to each window's midpoint)
are fixed for a run and come from register settings (coarse disc timing, `reg_k_window`).

---

#### Measured quantities

**Energies** — all are 24-bit unsigned sums of ADC counts:

| Quantity | Words | Physical meaning |
|----------|-------|-----------------|
| `PRE_RISE_SUM` | W8[23:0] | Integral of M samples ending at the pulse onset. Represents baseline × M. Used as the baseline reference for energy subtraction. |
| `POST_RISE_SUM` | W8[31:24] + W9[15:0] | Integral of the signal region from just before the peak onwards, for a fixed window. This is the primary energy measurement. Net energy ≈ POST_RISE_SUM − PRE_RISE_SUM. |
| `P2_SUM` | W10[13:0] + W13[9:0] | Integral of the pulse tail after POST_RISE, for `reg_p2_window` cycles. Used for pileup characterization and decay-tail correction. |
| `EARLY_PRE_RISE_SUM` | W12[23:0] | A second, earlier baseline integral window (before M). Available when `EARLY_PRE_M_SEL` = 1. Useful for double-pulse baseline isolation. |
| `SAMPLED_BASELINE` | W6[23:0] | Running baseline estimate from `baseline_tracker.vhd` latched at event time. Tracks DC level on a ~10 µs timescale. |
| `PREV_POST` | W11[31:24]+W12[31:24]+W13[31:24] | The POST_RISE_SUM of the immediately preceding event on this channel. Available in pileup-recording mode (`PILEUP_DISABLE = 1`) for pileup correction. |

Typical net energy computation:
```
Energy = POST_RISE_SUM - PRE_RISE_SUM
```
For precise spectroscopy (baseline drift correction):
```
Energy = POST_RISE_SUM - SAMPLED_BASELINE × (POST_RISE window length in samples)
```

**Timestamps** — the 48-bit counter runs at 100 MHz (10 ns/tick), synchronized to MTRG SYNC frames:

| Quantity | Words | Physical meaning |
|----------|-------|-----------------|
| `TIMESTAMP_OF_EVENT[47:0]` | W2 + W3[15:0] | Full 48-bit event time. In LED mode: leading-edge threshold crossing. In CFD mode: the clock tick when the zero-crossing sign flip was detected (see CFD interpolation below). |
| `TS_OF_LAST_EVENT[47:0]` | W4[31:16] + W5 | Full 48-bit timestamp of the previous event on this channel (LED). Difference `TIMESTAMP_OF_EVENT − TS_OF_LAST_EVENT` gives the inter-event interval for dead-time and pileup analysis. |
| `TS_OF_PEAK[15:0]` | W9[31:16] | Lower 16 bits of the timestamp at the pulse peak (the sample where the peak-finding algorithm detected the maximum). `TS_OF_PEAK − TIMESTAMP_OF_EVENT[15:0]` ≈ signal rise time. |
| `TS_OF_TRIGGER[15:0]` | W10[31:16] | Lower 16 bits of the timestamp when the trigger message arrived from the Router (~2–20 µs after the event, within the TRIG_DELAY window). |
| `TS_OF_COARSE[9:0]` + GE/SE | W13[23:14] + W4[13:12] | 12-bit coarse discriminator timestamp (TS_OF_COARSE is 10 bits; GE and SE extend it by 2 more). Marks when the coarse (BGO/NaI) discriminator fired, for Ge–BGO coincidence gating. |
| `TS_OF_LAST_PREAMP_RESET[23:0]` | W11[23:0] (MPX_FIELD, when CP=1) | Lower 24 bits of the timestamp of the most recent preamp reset pulse. Used to measure and correct for decay-tail artifacts in Ge detectors after a reset. |

#### Pole-zero correction

The Ge charge-sensitive preamplifier output decays exponentially after each pulse
with a characteristic time τ (the feedback RC constant). This tail biases the energy
sums of subsequent events. Correcting for it requires two steps: baseline
reconstruction and sum correction.

**Signal model**

After a pulse of step height A at time t₀ (full charge collection), the preamp output is:

```
V(t) = V_base + A · exp(−(t − t₀) / τ)     for t ≥ t₀
```

With multiple events, tails from all prior pulses accumulate on V_base. In practice
only the immediately preceding event contributes significantly, and the packet
provides the fields needed for that correction.

**Step 1 — Reconstruct the tail amplitude**

Two equivalent approaches, both computable from the packet:

*Approach A — PREV_POST + ΔT:*
```
A_prev = PREV_POST / M − SAMPLED_BASELINE          (previous pulse step height)
tail_0 = A_prev · exp(−ΔT / τ)                    (tail level at TIMESTAMP_OF_EVENT)

where ΔT = TIMESTAMP_OF_EVENT − TS_OF_LAST_EVENT  (full 48-bit difference, 10 ns/tick)
```

*Approach B — two-window baseline solve (PREV_POST not required):*

EARLY_PRE and PRE measure the same decaying tail at two different, known times,
giving two equations in two unknowns (V_base, A_prev):
```
EARLY_PRE / M = V_base + A_prev · exp(−t_e / τ)
PRE       / M = V_base + A_prev · exp(−t_p / τ)
```
Subtracting eliminates V_base; solving gives A_prev directly. The offsets t_e and
t_p (from TS_OF_LAST_EVENT to each window midpoint) come from register settings
(coarse disc timing, `reg_k_window`) — fixed for a run, not per-event.

**Step 2 — Correct PRE_RISE_SUM and POST_RISE_SUM**

The previous pulse tail contributes a bias to each integration window. The exact
integral of the tail over an M-sample window starting at time t_start (measured
from TS_OF_LAST_EVENT) is:

```
tail_in_window(t_start) = A_prev · exp(−t_start / τ) · τ · (1 − exp(−M / τ))
```

For M << τ (typical: M ~ 2 µs, τ ~ 50–500 µs for Ge) this simplifies to:
```
tail_in_window(t_start) ≈ A_prev · exp(−t_start / τ) · M
```

The two window start times (measured from TS_OF_LAST_EVENT):
```
t_p0 = ΔT − M                                           (start of PRE_RISE window)
t_q0 = ΔT + (TS_OF_PEAK[15:0] − TIMESTAMP_OF_EVENT[15:0])  (start of POST_RISE window)
```

`TS_OF_PEAK` supplies the variable POST_RISE start offset, which depends on the
pulse rise time and varies event-by-event.

Corrected sums:
```
PRE_corrected  = PRE_RISE_SUM  − tail_in_window(t_p0)
POST_corrected = POST_RISE_SUM − tail_in_window(t_q0)
```

**Corrected net energy**

```
E_corrected = POST_corrected − PRE_corrected
            = (POST_RISE_SUM − PRE_RISE_SUM)
              − A_prev · τ · (1 − exp(−M/τ)) · [exp(−t_q0/τ) − exp(−t_p0/τ)]
```

For M << τ:
```
E_corrected ≈ (POST_RISE_SUM − PRE_RISE_SUM)
              − A_prev · M · [exp(−t_q0/τ) − exp(−t_p0/τ)]
```

The second term is the pole-zero correction. Since t_q0 > t_p0, exp(−t_q0/τ) <
exp(−t_p0/τ), so the correction is negative — it reduces the apparent energy,
removing the tail that was artificially elevating PRE_RISE relative to POST_RISE.

**Within-pulse decay correction (systematic)**

Independently of inter-pulse tails, the current pulse itself decays during the
M-sample POST_RISE window, so POST_RISE_SUM underestimates A_curr · M:

```
POST_RISE_SUM(from current pulse) = A_curr · τ · (1 − exp(−M/τ)) ≈ A_curr · M · (1 − M/(2τ))
```

Dividing by the scale factor recovers the true energy. This is a run-constant
multiplicative correction that depends only on M and τ:
```
scale = (τ / M) · (1 − exp(−M / τ))
E_true = E_corrected / scale
```

**Inputs summary**

| Quantity | Source | Comment |
|----------|--------|---------|
| `ΔT` | `TIMESTAMP_OF_EVENT − TS_OF_LAST_EVENT` | Full 48-bit, 10 ns/tick |
| `A_prev` | `PREV_POST/M − SAMPLED_BASELINE` (A) or two-window solve (B) | Step height of previous pulse |
| POST_RISE start | `TS_OF_PEAK[15:0] − TIMESTAMP_OF_EVENT[15:0]` | Variable rise-time offset, per-event |
| `M` | `reg_m_window` from run config | Samples per integration window (cycles × 10 ns) |
| `τ` | Detector calibration | Not stored in packet |

**Limitations**

- Only the immediately preceding event is correctable via PREV_POST and
  TS_OF_LAST_EVENT. At high rates or with long τ, tails from earlier events
  accumulate; SAMPLED_BASELINE partially absorbs the DC drift but lags by ~10 µs.
- After a preamp reset the tail structure is disrupted; `TS_OF_LAST_PREAMP_RESET`
  (MPX_FIELD, CP=1) identifies affected events for flagging or exclusion.

---

#### CFD Header differences (words 4–7 only)

In CFD mode (`HEADER_TYPE = 0101`), words 0–3 and 8–13 are identical to LED. Words 4–7 carry different fields:

```
Word  4:  |     TS_OF_LAST_EVENT[15:0]    |PU|PO|GE|SE|CV|OF|PV|ED|TF|VF|WF|PE| TDD[15:12]             |
Word  5:  | TDD[11:10] |  CFD_SAMPLE(0)[13:0]   | TDD[9:8] |   TS_OF_LAST_EVENT[29:16]                  |
Word  6:  |    TDD[7:0]    |                        SAMPLED_BASELINE[23:0]                               |
Word  7:  | PUC[3:2] |  CFD_SAMPLE(2)[13:0]  | PUC[1:0] |         CFD_SAMPLE(1)[13:0]                   |
```

Additional CFD status bits in Word 4:

| Bit | Abbrev | Meaning |
|-----|--------|---------|
| 11 | CV | `CFD_VALID_FLAG`: the zero-crossing was clean and valid |
| 7 | TF | `TSM_FLAG`: upper bits of timestamp match previous event (pileup proximity check) |

Note: In CFD mode `TS_OF_LAST_EVENT` is only 30 bits (not 48). `TRIG_MON_DET_DATA` replaces the `PU_CNT + SAMPLED_BASELINE` slot in Words 4/6; SAMPLED_BASELINE is still in W6[23:0]. `PILEUP_COUNT` is split across Word 7 flanking the two CFD samples. See reconstruction formulas above.

#### CFD zero-crossing interpolation

The three CFD samples are 14-bit signed values of `LOCAL_DIFFERENCE = CFD_SUBTRACTION − LOCAL_ZERO`, the shifted CFD waveform. They are captured at the sign-flip clock:

| Sample | Clock relative to sign flip | Sign | Description |
|--------|-----------------------------|------|-------------|
| `CFD_SAMPLE(1)` | T − 1 (10 ns before) | **positive** | Last sample above zero |
| `CFD_SAMPLE(0)` | T = `TIMESTAMP_OF_EVENT` | **negative** | First sample below zero |
| `CFD_SAMPLE(2)` | T − 2 (20 ns before) | positive | Earlier sample (quadratic correction) |

`TIMESTAMP_OF_EVENT` is latched at the same clock as `CFD_SAMPLE(0)`, so the actual zero-crossing lies somewhere in the 10 ns interval `[T−1, T]`.

**Linear interpolation** (sufficient for most purposes):

```
S1 = CFD_SAMPLE(1)   (positive, 14-bit signed)
S0 = CFD_SAMPLE(0)   (negative, 14-bit signed)

sub_sample_fraction  = S1 / (S1 - S0)        -- value in [0, 1]

t_zero = (TIMESTAMP_OF_EVENT - 1 + sub_sample_fraction) × 10 ns
```

Equivalently, as a correction referenced directly to `TIMESTAMP_OF_EVENT`:
```
correction = -S0 / (S1 - S0) × 10 ns         -- value in (-10 ns, 0]
t_zero = TIMESTAMP_OF_EVENT × 10 ns + correction
```

**Quadratic interpolation** using all three samples (for best timing resolution):

The three points at t = T−2, T−1, T with values S2, S1, S0 define a parabola. The zero crossing of the parabola:
```
a = (S0 - 2×S1 + S2) / 2
b = (4×S1 - S0 - 3×S2) / 2
c = S2
root = (-b - sqrt(b²- 4×a×c)) / (2×a)     -- root in [0, 2], relative to T-2
t_zero = (TIMESTAMP_OF_EVENT - 2 + root) × 10 ns
```

In practice the linear formula is used; quadratic only matters when the signal slope changes rapidly at the crossing (very asymmetric pulse shapes).

The `CFD_VALID_FLAG` (CV bit) confirms that the zero-crossing logic completed normally within the holdoff window. If CV = 0, the samples may be unreliable and the event should be discarded from timing analysis.

#### Waveform samples (after header)

After word 13, zero or more waveform words follow. The number of samples is controlled by `reg_raw_data_length`.

| Bits | Field | Description |
|------|-------|-------------|
| 31:16 | Sample N | ADC sample (16-bit, sign-extended from 14-bit ADC) |
| 15:0 | Sample N+1 | Next ADC sample |

Two consecutive ADC samples (100 MHz, 10 ns/sample) are packed into each 32-bit word. The waveform window is positioned relative to the discriminator fire by `reg_raw_data_window`. Decimation (1×, 2×, 4×, …, 128×) is applied by `decimator.vhd` and automatically pauses around the pulse rise time to preserve full-rate timing accuracy near the discriminator crossing.

If `reg_raw_data_length = 0`, no waveform words are written and the packet ends after word 13.

---

## Per-Channel Signal Processing: LED and CFD Modes

Each of the 10 channels runs an identical 100 MHz pipeline implemented in `jta_channel.vhd`. The pipeline has two discriminator modes selectable per channel: **LED** (Leading-Edge Discriminator, threshold-based) and **CFD** (Constant Fraction Discriminator, zero-crossing-based). Both modes share the same upstream delay chain and filters; they differ in how they derive the discriminator signal and fire the event timestamp.

### Common Signal Path — Delay Chain and Filtering

The raw 14-bit ADC sample passes through a series of programmable delay buffers before reaching the discriminators. All delays are in 100 MHz clock cycles (10 ns steps).

```
ADC_DATA[13:0]  (14-bit, 100 MHz)
    │
    ▼ P1 delay  (reg_p1_window, default 1 cycle)
    │
    ▼ P2 delay  (reg_p2_window, default 2 cycles)
    │
    ▼ M delay   (reg_m_window[9:0], default 200 cycles = 2 µs)
    │   X_M  ← pre-event buffer; holds signal before the pulse arrives
    │
    ▼ K0 delay  (lower bits of reg_k_window)
    │   X_M_K0
    │
    ▼ K delay   (upper bits of reg_k_window, default ~100 cycles)
    │   X_M_K0_K
    │
    ▼ D delay   (reg_d_window[6:0], default 10 cycles)
    │   X_M_K0_K_D
    │
    ▼ D3 delay  (reg_d3_window[6:0], default 23 cycles)
    │   X_M_K0_K_D_D3  ← used for baseline tracking input
    │
    ▼ TRIPLE_FILTER  (triple_filter.vhd)
    │   Cascaded moving-average filter: 3× (1-2-1) stages
    │   Smooths the signal for cleaner threshold comparison
    │   Produces two taps: PROMPT (at K0) and DELAYED (at K0+K)
    │
    ▼ Baseline subtraction
        FILTERED_SIGNAL − BASELINE_VALUE  →  discriminator inputs
```

**Triple filter:** Each stage is a (1-2-1) moving average. Three cascaded stages produce an effective kernel of [1,8,28,56,70,56,28,8,1] / 256, reducing high-frequency noise without significantly broadening the pulse.

**Baseline tracker** (`baseline_tracker.vhd`): Estimates the DC baseline by accumulating a running difference `X(n) − X(n−T)` over a 1024-sample (10.24 µs) window. It holds off updates for a programmable time after every discriminator fire (`reg_baseline_delay`) to avoid pulling the baseline onto the pulse tail.

---

### LED Mode — Leading-Edge Threshold Discriminator

In LED mode (`CFD_MODE = '0'`), the discriminator fires as soon as the filtered, baseline-subtracted signal crosses a fixed threshold. This gives a coarse timestamp tied to the signal's leading edge.

```
THRESH_DISC_PROMPT  = triple_filter output at tap X_M_K0_K  (earlier)
THRESH_DISC_DELAYED = triple_filter output at tap X_M_K0_K_D (D cycles later)

Both taps − BASELINE_VALUE
    │
    ▼ thresh_disc.vhd
    Compare THRESH_DISC_DELAYED > DISCRIMINATOR_THRESHOLD
    ─── AND ───
    Compare THRESH_DISC_PROMPT  > DISCRIMINATOR_THRESHOLD
    │
    ▼
THRESH_DISC_FLAG  (one-shot pulse)
    │
    └─→ Opens PEQ entry, starts energy integration, latches 48-bit timestamp
```

The two-tap comparison (PROMPT and DELAYED both above threshold) acts as a simple coincidence filter that suppresses single-sample noise spikes. The threshold value is set by `reg_led_threshold`.

**Timing:** The discriminator flag is asserted approximately 5 clock cycles (50 ns) after the signal crosses threshold, accounting for filter pipeline latency.

---

### CFD Mode — Constant Fraction Discriminator

In CFD mode (`CFD_MODE = '1'`), the discriminator fires at the zero-crossing of `(fraction × prompt_signal) − delayed_signal`. Because the zero-crossing position on the pulse shape is independent of amplitude, CFD gives significantly better timing resolution than LED for pulses of varying heights.

```
CFD_PROMPT  = triple_filter tap at X_M_K0_K    (same as LED prompt)
CFD_DELAYED = triple_filter tap at X_M_K0_K_D  (D cycles later)

Step 1 — Pre-trigger (thresh_disc.vhd fires first as a gate):
    THRESH_DISC_FLAG fires on leading edge (same as LED but used only as a gate)
    → triggers CFD_SAMPLE_ZERO: latches LOCAL_ZERO = current CFD_SUBTRACTION value
    → after K cycles, asserts CFD_PRE_TRIGGER

Step 2 — Fraction multiply (MULT17×17, 34-bit result):
    FRACTIONAL_PROMPT = CFD_PROMPT × CFD_FRACTION >> 13
    (CFD_FRACTION register encodes the fraction as N/8192;
     e.g. reg_cfd_fraction = 0x0C00 ≈ 75% of full scale)

Step 3 — CFD subtraction:
    CFD_SUBTRACTION = FRACTIONAL_PROMPT − CFD_DELAYED

Step 4 — Zero-crossing detection (cfd_disc.vhd):
    LOCAL_DIFFERENCE = CFD_SUBTRACTION − LOCAL_ZERO
    Track sign of LOCAL_DIFFERENCE each clock cycle
    When sign flips → CFD_DISC_FLAG asserted, 48-bit timestamp latched
    Three CFD_SAMPLES captured around the crossing for interpolation
```

The zero-crossing tracks the point on the pulse where `fraction × amplitude = delayed_amplitude`, which moves in time but not in amplitude — giving the amplitude-independent timestamp.

**Key difference from LED:** The timestamp latched in CFD mode is the zero-crossing time, not the threshold-crossing time. This typically improves coincidence timing resolution from ~10 ns (LED) to ~2–3 ns (CFD) for germanium detectors.

---

### Mode Selection

| Register | Address (Ch 0) | Bit | Function |
|----------|---------------|-----|----------|
| `reg_channel_control` | `0x040` | `CFD_MODE` | `0` = LED, `1` = CFD |

Channels 1–9 use addresses `0x044` through `0x064` (4-byte spacing). The `CFD_MODE` bit is distributed as four copies (`xCFD_MODE[3:0]`, with KEEP attribute) inside `jta_channel.vhd` to avoid long-path timing issues.

---

### After Discrimination — PEQ and Energy Integration

When a discriminator fires (LED or CFD), the channel opens a slot in the **Pending Event Queue (PEQ)** — a 16-deep FIFO. The event remains pending until the trigger decision arrives from the Router (~2–4 µs later, within the ~20 µs TRIG_DELAY window). During that time, three energy sums are accumulated:

```
Discriminator fire
    │
    ├─ Latch 48-bit timestamp
    ├─ Open PEQ entry
    │
    ├─→ PRE_RISE integration
    │     Accumulates M cycles of samples before the pulse peak
    │     Duration: reg_m_window[9:0] clock cycles
    │
    ├─→ POST_RISE integration
    │     Accumulates samples from peak onwards
    │     Starts at PEAK_FLAG (peak-finding algorithm in thresh_disc.vhd)
    │
    └─→ P2 integration  (tail sum)
          Accumulates after POST_RISE for additional baseline/tail correction
          Duration: reg_p2_window[9:0] clock cycles

Trigger decision arrives (~2–4 µs later):
    Accepted → pack (timestamp + PRE_RISE + POST_RISE + P2 + pileup flags)
               into 36-bit external FIFO for VME readout
    Rejected → discard PEQ entry silently
```

In CFD mode with `CFD_ESUM_MODE = '1'`, the energy integration start is deferred to `THRESH_DISC_FLAG_DELAYED` (the LED crossing) rather than the CFD zero-crossing, so energy always integrates the same portion of the pulse regardless of discriminator mode.

---

### Pileup Detection

The **pileup processor** (`pileup_processor.vhd`) tracks how many events are in-flight (discriminator fired but not yet readout-complete). It uses a 4-bit counter and an 8-state machine:

```
States: IDLE → ONE_HIT → MANY_HIT → OVERFLOW
        (each with ACC or REJ variant)

Counter increments: on each THRESH_DISC_FLAG
Counter decrements: on PILE_RELEASE_DLYD (end-of-event holdoff pulse)

PILEUP_DISABLE register:
    0 → reject second and subsequent pileup hits (standard spectroscopy)
    1 → accept all hits (pileup recording mode)

Outputs per event:
    ACCEPTED_HIT   — first hit (or any hit in accept-all mode)
    EXTENDED_EVENT — subsequent pileup hits (accept-all mode only)
    PILEUP_FLAG    — level: counter > 0
    OVERFLOW_FLAG  — counter saturated at 15
    PU_TOO_SHORT   — pileup interval shorter than retrigger holdoff; event invalid
```

The holdoff time (`reg_holdoff_control[8:0]`) controls both the minimum inter-event spacing and the peak-finding window; it is shared between the threshold discriminator and the pileup counter.

---

### VME Registers for Discriminator Configuration

All addresses are per-channel. Channel 0 uses the base address shown; channels 1–9 add `4 × channel_number`.

**Discriminator mode and thresholds:**

| Register | Ch 0 Address | Bits | Description |
|----------|-------------|------|-------------|
| `reg_channel_control` | `0x040` | `CFD_MODE` bit | `0` = LED, `1` = CFD |
| `reg_led_threshold` | `0x080` | `[13:0]` | Threshold in ADC counts (both LED and CFD pre-gate) |
| `reg_cfd_fraction` | `0x0C0` | `[12:0]` | CFD fraction encoded as N/8192 (e.g. `0x0C00` ≈ 75%) |
| `reg_external_disc_mode` | `0x420` | 2 bits/ch | `00`=normal, `01`=OR with external, `10`=AND, `11`=external only |

**Delay chain:**

| Register | Ch 0 Address | Bits | Description |
|----------|-------------|------|-------------|
| `reg_p1_window` | `0x300` | `[3:0]` | P1 delay (cycles) |
| `reg_p2_window` | `0x404` | `[9:0]` | P2 delay and tail-sum window (cycles) |
| `reg_m_window` | `0x200` | `[9:0]` | M delay = pre-event buffer depth (cycles) |
| `reg_k_window` | `0x1C0` | `[13:0]` | K0+K combined delay (cycles) |
| `reg_d_window` | `0x180` | `[6:0]` | D delay — sets CFD fraction delay (cycles) |
| `reg_d3_window` | `0x240` | `[6:0]` | D3 delay — baseline tracker input offset (cycles) |

**Baseline tracking:**

| Register | Ch 0 Address | Bits | Description |
|----------|-------------|------|-------------|
| `reg_baseline_start` | `0x2C0` | `[13:0]` | Initial baseline value (ADC counts) |
| `reg_baseline_delay` | `0x418` | `[7:0]` | Holdoff after disc fire before resuming tracking (× 10.24 µs) |
| `reg_baseline_delay` | `0x418` | `[10:8]` | Baseline update step size (tracking speed) |

**Holdoff and peak finding:**

| Register | Address | Bits | Description |
|----------|---------|------|-------------|
| `reg_holdoff_control` | `0x414` | `[8:0]` | Retrigger holdoff duration (cycles × 10 ns) |
| `reg_holdoff_control` | `0x414` | `[11:9]` | Peak sensitivity (controls peak-finding rate) |
| `reg_disc_width` | `0x280` | `[7:0]` | Discriminator output pulse width (cycles) |

**Waveform capture:**

| Register | Ch 0 Address | Bits | Description |
|----------|-------------|------|-------------|
| `reg_raw_data_length` | `0x100` | `[9:0]` | Number of waveform samples to capture |
| `reg_raw_data_window` | `0x140` | `[10:0]` | Capture window relative to discriminator fire (samples) |

**Diagnostic counters (read-only):**

| Register | Ch 0 Address | Description |
|----------|-------------|-------------|
| `reg_disc_count` | `0x7C0` | Total discriminator fires |
| `reg_accepted_event_count` | `0x740` | Events accepted by trigger |
| `reg_dropped_event_count` | `0x700` | Events dropped (FIFO full or vetoed) |
| `reg_ahit_count` | `0x780` | Accepted-hit pulses from pileup processor |

---

## VME FPGA

**Location:** `VME_FPGA_ANL/`

| Field | Value |
|-------|-------|
| Part | xc3s400 (Spartan-3) |
| Package | fg320 |
| Speed Grade | -5 |
| Tool | ISE 13.4 |
| Project | `VME_FPGA_ANL/Work11/vme_A32_D32.xise` |
| Top Entity | `vme_top` |

Same architecture as the MTRG VME FPGA: acts as A32/D32 VME slave, programs the main FPGA from external flash, and bridges host VME commands to the main FPGA.

### Source Files
| File | Description |
|------|-------------|
| `TOP.VHD` | Top-level entity (`vme_top`) |
| `vme_addr_decode.vhd` | VME address space decoder |
| `external_bus_controller.vhd` | Flash/FPGA bus multiplexer |
| `configuration_controller.vhd` | FPGA programming sequencer |
| `register_block.vhd` | Status and control registers |
| `register_block_FlashHi.vhd` | Upper flash address register block |

### Bitfiles
| File | Description |
|------|-------------|
| `Work11/vme_top.bit` | Standard VME FPGA bitfile |
| `Work11/vme_top_usehi.bit` | Variant using upper flash address |
| `Work11/20230928.mcs` | MCS flash image (Sept 2023) |
| `Work11/20230928_usehi.mcs` | MCS flash image, upper flash variant |

### Clock Select Register (`clk_select`)

**VME address:** `0x0910` bits[1:0] in `register_block.vhd`

Controls which clock the digitizer uses as its system clock. The two bits (`sysclk_sel[1:0]`) drive physical output pins `sysclk_sel0_out` (B9) and `sysclk_sel1_out` (B10) off-chip to a hardware clock mux on the digitizer PCB.

**Important:** the register bits are **inverted** before the output pins (`sysclk_sel0_out <= NOT sysclk_sel0`) to match the original LBL digitizer board design. EPICS values are correct end-to-end — the inversion is transparent to software.

Default at reset: `sysclk_sel0=1, sysclk_sel1=0` → OSC (local oscillator).

| `clk_select` EPICS value | `sel[1:0]` | Meaning |
|---|---|---|
| 0 | `00` | S/D — SERDES derived (link clock from Router) |
| 1 | `01` | **OSC** — local on-board oscillator (default) |
| 2 | `10` | S/D — same as 0 |
| 3 | `11` | AUX — auxiliary clock input |

**EPICS PV:** `VME$(CRATE):$(BOARD):clk_select` (mbbo, `MDigUserVME.template` / `SDigUserVME.template`)

**Usage in `link_sys.py`:**
- Stage 4A: `clk_select=1` (OSC) — initialize DIGs on independent local clock first
- Stage 4E: `clk_select=0` (S/D) — switch DIGs to Router-derived link clock for full timestamp sync

## Main FPGA Bitfiles

| Branch | Bitfile | Description |
|--------|---------|-------------|
| DGS | `DGS/Work/BUS_LEFT.bit` | Production — front bus sender role |
| DGS | `DGS/Work/BUS_RIGHT.bit` | Production — front bus receiver role |
| Majorana | `Majorana/Work/digitizer.bit` | MAJORANA experiment variant |
| DGS_QUAD_M_SUMS | `DGS_QUAD_M_SUMS/Work/FB_SENDER.bit` / `FB_RCVR.bit` | Quad M-sum, sender/receiver |
| SumOverRise | `SumOverRise/Work/FB_SENDER.bit` / `FB_RCVR.bit` | Sum-over-rise, sender/receiver |
| DGSBramTest | `DGSBramTest/Work/MSTR_digitizer.bit` | BRAM test |
| — | `tag_4975_mod_fifo_digitizer.bit` (root) | Tagged release build |
| — | `Walter_Release_MDIG_6194/MSTR_digitizer-6194.bin` | Release candidate v6194 |

Note: DGS branch produces two bitfiles (`BUS_LEFT` / `BUS_RIGHT`) because the `FRONT_BUS_LEFT` generic changes the front bus direction — two digitizer modules are paired, one sender and one receiver.

## IP Cores

Located in each branch's `Cores/` directory:

| Core | Description |
|------|-------------|
| `chipscope_icon` | ChipScope controller |
| `chipscope_ila` | ChipScope logic analyzer |
| `fifo_16x1023_async` | 16-bit, 1K deep async FIFO |
| `fifo_16x64K_async` | 16-bit, 64K deep async FIFO |
| `BRAM_1024X16_REGSHADOW` | Block RAM register shadow |

---
*Source: `DGS_tools_pack/raw_FPGA/Dig*/` — VHDL source. PDF: `ANL Digitizer Firmware for Experts.pdf`. Created: 2026-04-05.*
