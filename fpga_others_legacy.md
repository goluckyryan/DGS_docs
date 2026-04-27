# FPGA/others/ — Legacy & Ancestor Digitizer Designs

Stability: C3 - Structural / stable

**Source:** `DGS_tools_pack/FPGA/others/`
**Last documented:** 2026-04-27

**See also:**
- [`fpga.md`](fpga.md) — current DGS DIG/MTRG/RTRG FPGA overview
- [`deep_fpga_SBX_CtrlFPGA.md`](deep_fpga_SBX_CtrlFPGA.md) — current SBX FPGA
- [`trig_sys_sim.md`](trig_sys_sim.md) — MTRG trigger system VHDL testbench

---

## Table of Contents

- [Directory Layout](#directory-layout)
- [LBL_Digitizer — LBNL GRETINA Ancestor Design](#lbl_digitizer--lbnl-gretina-ancestor-design)
  - [Origin and Context](#origin-and-context)
  - [Architecture Overview (chip_top.vhd)](#architecture-overview-chip_topvhd)
  - [Signal Interfaces](#signal-interfaces)
  - [Key Sub-Modules](#key-sub-modules)
  - [JTA Simplification Work (jta_temp_branch/)](#jta-simplification-work-jta_temp_branch)
  - [VME Address Decode (gammasphere functional notes)](#vme-address-decode-gammasphere-functional-notes)
  - [Synthesis Results (Spartan-3 5000)](#synthesis-results-spartan-3-5000)
- [Majorana_Digitizer — ANL HEP Digital Gammasphere Design](#majorana_digitizer--anl-hep-digital-gammasphere-design)
  - [Origin and Context](#origin-and-context-1)
  - [Top-Level Entity (Digitizer.vhd)](#top-level-entity-digitizervhd)
  - [Package Types (Digitizer_pkg.vhd)](#package-types-digitizer_pkgvhd)
  - [Key Sub-Modules (Source/)](#key-sub-modules-source)
  - [DGS vs Majorana Branch Differences](#dgs-vs-majorana-branch-differences)
  - [Firmware Constant](#firmware-constant)
- [Cross-References](#cross-references)

---

## Directory Layout

```
FPGA/others/
├── LBL_Digitizer/           # LBNL GRETINA digitizer VHDL (ancestor design, 2006–2011)
│   ├── MAIN FPGA/
│   │   ├── jta_temp_branch/ # JTA's working branch (simplification pass, 2011)
│   │   ├── 20110930/        # Snapshot as of 2011-09-30
│   │   ├── LBL_20100804/    # Earlier LBL snapshot
│   │   ├── original_greta14bitse/  # Original 14-bit GRETINA design
│   │   └── bins/            # Compiled .bin firmware images
│   ├── VME FGPA/            # VME FPGA VHDL + .mcs files
│   └── *.pdf                # Documentation: GRETA Algorithm Architecture, VHDL Modules description
└── Majorana_Digitizer/      # ANL HEP Digital Gammasphere digitizer (production ancestor)
    └── MAIN_FPGA/
        ├── Source/          # VHDL source (50 files)
        ├── Cores/           # Xilinx IP cores
        ├── SimulationConfigs/  # ISim config files
        └── Work/            # ISim simulation work library
```

---

## LBL_Digitizer — LBNL GRETINA Ancestor Design

### Origin and Context

Original design by **Vincent Riot**, Lawrence Berkeley National Laboratory (LBNL), for the GRETINA detector system. Headers note "LBNL PROPRIETARY." Created circa 2006 for the GRETINA experiment; later adapted by JTA (Ryan Tang) as the starting point for the DGS digitizer.

The design is for a **14-bit, 10-channel digitizer** running on a **Spartan-3 5000 (XC3S5000-FG900-5)** FPGA at 50 MHz base clock (100 MHz ADC clock). The board interfaces to a custom VME crate via a daughter VME FPGA that decodes bus cycles and drives the main FPGA.

The `jta_temp_branch/` directory represents JTA's working copy of the VHDL with a simplification pass begun in 2011, focusing on removing hand-coded counter components and replacing them with idiomatic VHDL `+1` incrementers.

### Architecture Overview (chip_top.vhd)

Top entity: `Chip_TOP` with ~20 port groups:

| Port Group | Description |
|-----------|-------------|
| `ADCData[0–9]` (14-bit) + `ADCMidOver[0–9]` | 10× 14-bit ADC inputs + overflow flags |
| `CKL100_OUT_[0,1]` | 100 MHz ADC clock outputs (derived from 50 MHz) |
| `FIFO_*` (36-bit data bus) | External FIFO interface: WEN_N, WCLK, REN_N, OE_N, RT_N, MRS_N, PRS_N, RCLK_N |
| `AUX_DIN/DOUT` (11-bit) | Auxiliary I/O with per-direction enable signals (EN_AUX2A–2F TX/RX) |
| `LED_OUT` (26-bit) | LED indicator outputs |
| `TEST_OUT` (16-bit), `TEST_CLK` | Test points |
| `SERIAL_NUMBER_N` (12-bit) | Board serial number (active low) |
| `DAC_Data0` (8-bit), `DAC_CLK_P/N` | DAC output (differential clock) |
| `LVME_Data` (32-bit, inout), `LVME_ADDR` (32-bit) | VME local data bus |
| `DMM_CS_IN` (3-bit), `DMM_RNW_IN`, `DMM_STRB_IN`, `DMM_LACK_OUT` | VME control from VME FPGA |
| `CLK50_IN`, `AUX_CLK50_IN` | 50 MHz system clocks |
| `SD_TX/RX` (18-bit), SerDes control signals | Serializer/Deserializer (SERDES) for trigger uplink |
| `RJ_AUX[0,1] DIN/DOUT/DIR` | RJ45 auxiliary serial links |
| `FB_*` | Master Front Bus (10-bit data, 8-bit address, bidirectional) |
| `P2_IO` (24-bit) | VME P2 connector I/O |

### Signal Interfaces

- **VME interface:** Not direct VME — a **VME FPGA daughter** handles bus decoding and drives the main FPGA via `DMM_CS_IN/RNW_IN/STRB_IN/LACK_OUT` plus local data/address buses. Internal signal names use `PROGAddrsig`/`PROGDatasig` convention.
- **PROGFlagsig[19:0]:** Internal sub-decode flags, loaded from VME address bits[19:2]. Key mappings (from jta_notes.txt):
  - Bits 15, 14, 10:0 → TenChannel module (bits 14/15 = FIFO control, 10:0 = per-channel)
  - Bit 13 → MasterLogicModule (VME 0x500)
  - Bit 12 → FrontBusSlave (VME 0x480)
  - Bit 11 → DACControl (VME 0x400)
  - Bit 16 → SelfTrigger (VME 0x860)
  - Addresses 0x0000–0x00FF hit all channels
- **Internal data bus:** pseudo-tristate implemented in FPGA fabric

### Key Sub-Modules

| Module | File | Description |
|--------|------|-------------|
| `chip` | `chip.vhd` | Mid-level wrapper connecting chip_top pads to Ten_Channel + VMEControl |
| `Ten_Channel` | `Ten_Channel.vhd` | Instantiates 10 `jta_channel` processing channels |
| `jta_channel` | `jta_channel.vhd` | Single-channel processing: ADC capture, filter, discriminator, energy integration |
| `VMEControl` | `VMEControl.vhd` | VME register decode and control logic |
| `MasterLogicModule` | `MasterLogicModule.vhd` | Cross-channel trigger/coincidence logic |
| `MasterLogicProcessingCore` | `MasterLogicProcessingCore.vhd` | Core of master logic |
| `MasterLogicRegisters` | `MasterLogicRegisters.vhd` | Registers for master logic |
| `FIFOMachine` | `FIFOMachine.vhd` | External FIFO write state machine |
| `FIFOInterface` | `FIFOInterface.vhd` | FIFO port interfacing |
| `SerDes` | `SerDes.vhd` | SERDES wrapper |
| `SD_TX/RX_SM` | `SD_TX_SM.vhd`, `SD_RX_SM.vhd` | SERDES TX/RX state machines |
| `DcBalance_b` | `DcBalance_b.vhd` | DC balance encoder (disparity lookup table) |
| `cfd` / `cfdprocess` | `cfd.vhd`, `cfdprocess.vhd` | Constant fraction discriminator |
| `TrapzFil` | `TrapzFil.vhd` | Trapezoidal energy filter |
| `DiffFilt` | `DiffFilt.vhd` | Differentiation filter |
| `SnapShotMem` | `SnapShotMem.vhd` | Waveform snapshot memory |
| `TSMemoryBlock` | `TSMemoryBlock.vhd` | Timestamp memory block |
| `FrontBusSlaveLowLevel` | `FrontBusSlaveLowLevel.vhd` | Front bus slave low-level protocol |
| `GaussFilter` | `gaufilt1.vhd`, `gaufilt2.vhd` | Gaussian shaping filters |
| `packetmachine` | `packetmachine.vhd` | Event packet assembly |
| `timingmachine` | `timingmachine.vhd` | Event timing logic |
| `statemachine` | `statemachine.vhd` | Main channel state machine |
| `SelfTrigger` | `SelfTrigger.vhd` | Self-trigger mode logic |

### JTA Simplification Work (jta_temp_branch/)

JTA's simplification pass (2011) is documented in `jta_notes.txt` inside the branch:

1. **Counter simplification:** Removed 8 hand-coded counter VHDL files (`COUNTER48`, `COUNTER12`, `COUNTER11`, `COUNTER10`, `COUNTER9`, `COUNTER7`, `COUNTER6`) — all were just `x = x + 1`. Replaced with VHDL `+1` expressions directly in the using modules. Required adding `IEEE.std_logic_arith` and `IEEE.std_logic_unsigned` library imports.

2. **Result:** Slice count dropped from 24,464 → 23,796 (1 LUT save). Flip-flop count nearly identical (29,943 → 29,971). Maximum frequency unchanged: **54.844 MHz** on Spartan-3. ✅ verified 2026-04-27 — `jta_notes.txt:L246` (24,464 slices before), `jta_notes.txt:L458` (23,796 after), `jta_notes.txt:L266` (54.844 MHz).

3. **Clock domain crossing bug flagged:** In `WaitCounter.vhd`: `RawLengthsig` and `SlidingWaitsig` are loaded by a `CLK50`-clocked process, but compared asynchronously against `Increment` (a `CLK`-clocked counter). Flagged as poor practice in JTA's notes.

4. **`disparity_lookup.vhd`:** Renamed from `disparitycounter.vhd` — JTA noted it is a two-level asynchronous lookup table, not a counter.

5. **VME register defaults** (from `VMEControl.vhd` reset block):
   - `ExternalWindow = 0x190`, `PileupWindow = 0x400`, `NoiseWindow = 0x040`
   - `ExternalTriggerSlidingLength = 0x190`
   - `CollectionTime = CollectionTimeLR = 0x1C2`, `IntegrationTime = 0x1C2`, `IntegrationTimeLR = 0x040`
   - `CC_LEDDriverEnable = 0x200`
   - `AUX_IO_WRITE = AUX_IO_CONFIG = 0x555` (enable read, disable write)
   - `ADC_CONFIGreg = 0x3`

### VME Address Decode (gammasphere functional notes)

From `gammasphere functinoal notes.txt`:
- ADC discriminator delays need ~700–1000 ns ("k" parameter) — requires **Block RAM** (not distributed RAM) for the discriminator delay buffer. Noted at a 9 AM meeting with the GITMO team (2011 era).

### Synthesis Results (Spartan-3 5000)

| Resource | Used | Available | % |
|----------|------|-----------|---|
| Slices | 23,796 | 33,280 | 71% |
| Slice FFs | 29,971 | 66,560 | 45% |
| 4-input LUTs | 35,187 | 66,560 | 52% |
| BRAMs | 63 | 104 | 60% |
| MULT18X18 | 22 | 104 | 21% |
| GCLKs | 7 | 8 | 87% |
| DCMs | 3 | 4 | 75% |
| IOBs | 488 | 633 | 77% |
| Max frequency | 54.844 MHz | — | — |

---

## Majorana_Digitizer — ANL HEP Digital Gammasphere Design

### Origin and Context

Designed by the **High Energy Physics Division, Argonne National Laboratory**, for the **Physics Division, ANL** (Gammasphere). Same FPGA target: **XC3S5000-5F900C** (Spartan-3). This is the direct architectural predecessor to the production DGS digitizer firmware found in `FPGA/DIG/`.

**Branched 2015-08-31** into two versions: Digital Gammasphere (main trunk) and Majorana (a branch). ✅ verified 2026-04-27 — `Digitizer.vhd:L12` comment: "20150831: Separation of design into two branches, Digital Gammasphere (main trunk) and Majorana (the branch)." The `Majorana_Digitizer/` folder contains the Majorana branch. Key Majorana differences vs DGS: ✅ verified 2026-04-27 — `Digitizer.vhd:L13-17` lists all 5 changes verbatim.
- CFD removed
- CFD_MODE control flag removed
- Triple-filter removed
- Threshold discriminator runs from full 24-bit M1/M2 sums (not filtered)
- Threshold is 24-bit (not 12-bit)

The main DGS trunk diverged from this branch to retain the CFD and triple-filter.

### Top-Level Entity (Digitizer.vhd)

Entity: `DIGITIZER` with two generics:

| Generic | Description |
|---------|-------------|
| `SLAVE_MODE : boolean` | If true: front bus direction reversed; internal cross-channel event veto disabled | ✅ verified 2026-04-27 — `Digitizer.vhd:L41-43` |
| `RUN_EXT_FIFO_AT_100MHZ : boolean` | Controls clock speed of external FIFO in discriminator | ✅ verified 2026-04-27 — `Digitizer.vhd:L44` |

**Key port groups vs LBL design:**

| Port Group | Description |
|-----------|-------------|
| `ADC_DATA_PINS[9:0]` (14-bit array) | 10× 14-bit ADC inputs |
| `ADC_DRDY_PINS[9:0]` | Data-ready signals per channel |
| `ADC_OVR[9:0]` | ADC overrange flags |
| `ADC_CLK_N/P` | Differential 100 MHz ADC clock output |
| `SERDES_*` | SERDES (replaced SD_* in LBL design) |
| `FBUS_*` | Front Bus (Master/Slave data, direction, WOR, LVDS) |
| `FIFO_*` (36-bit) | External FIFO interface (same as LBL) |
| `LVME_*` | VME interface signals |
| `LED_GRN/RED` (13-bit each) | LED outputs (separate green/red vs single in LBL) |
| `TEST_POINT[15:0]` (inout) | Test points |
| `VME_P2_IO[23:0]` (inout) | VME P2 connector |
| `SPARE_N/P[1:0]` | Front-end input connector spares |
| `RJ_AUX_DIN/DOUT/DIR[1:0]` | RJ45 auxiliary serial |

**Top-level constant:** `C_FIRMWARE = X"0001"` — firmware revision identifier. ✅ verified 2026-04-27 — `Digitizer.vhd:L131`: `constant C_FIRMWARE : std_logic_vector(15 downto 0) := X"0001";`

### Package Types (Digitizer_pkg.vhd)

Key record types defined in `Digitizer_pkg`:

**`tEVENT_DATA`** — per-channel event record:

| Field | Width | Description |
|-------|-------|-------------|
| `PILEUP_FLAG` | 1 | Pileup state flag |
| `SYNC_ERROR_FLAG` | 1 | Set for 2 µs after sync error |
| `PEAK_VALID_FLAG` | 1 | Peak timestamp valid |
| `EXTERNAL_DISC_FLAG` | 1 | External trigger generated this event |
| `PRE_RISE_ENTER_SAMPLE` | 14 | Sample near waveform edge |
| `PRE_RISE_LEAVE_SAMPLE` | 14 | Sample well before edge |
| `PRE_RISE_ENERGY` | 24 | Sum of samples across pre-rise buffer |
| `POST_RISE_SAMPLE` | 14 | Sample near peak in post-rise buffer |
| `LAST_POST_RISE_SAMPLE` | 14 | Previous event's post-rise sample |
| `POST_RISE_ENERGY` | 24 | Sum of samples across post-rise buffer |
| `PEAK_SAMPLE` | 14 | Filtered ADC value at peak |
| `BASE_SAMPLE` | 14 | Filtered ADC value at base of rise |
| `TS_OF_DISCBIT` | 48 | Timestamp of discriminator hit |
| `TS_OF_LAST_DISCBIT` | 48 | Timestamp of previous discriminator hit |
| `LAST_PEAK_SAMPLE` | 14 | Previous event's peak sample |
| `TS_OF_PEAK` | 16 | Timestamp of signal peak |
| `CFD_VALID_FLAG` | 1 | CFD valid (DGS trunk only) |
| `TSM_FLAG` | 1 | Upper timestamp bits match current/last |
| `CFD_SAMPLE[2:0]` | 14×3 | CFD samples (DGS trunk only) |

**`tCHANNEL_SLOW_DATA`** — "slow" (diagnostic) data per channel:

| Field | Width | Description |
|-------|-------|-------------|
| `PRE_SUM` | 24 | Pre-event sum |
| `POST_SUM` | 24 | Post-event sum |
| `DELTA_T1` | 16 | Primary event rise time |
| `DELTA_T2` | 16 | Time from primary peak to pileup event start |
| `DELTA_T3` | 16 | Pileup event rise time |
| `DELTA_A` | 16 | Event amplitude |
| `SIMPLE_PILEUP` | 1 | Single-event pileup status |
| `MULTI_PILEUP` | 1 | Multiple-event pileup status |
| `BGO_LATE` | 1 | BGO signal was late |
| `CHANNEL_ERROR` | 1 | Channel error |

**`tREGISTER_CONFIG`** — register descriptor for generic register infrastructure:
- `address` (12-bit), `reset_value` (32-bit), `read_mask` (32-bit), `write_mask` (32-bit), `fan_in_group` (integer)
- Used with `shadow_registers.vhd` and `Register_Logic.vhd` for parameterized VME register maps.

### Key Sub-Modules (Source/)

| File | Description |
|------|-------------|
| `Digitizer.vhd` | Top-level entity |
| `Digitizer_pkg.vhd` | Shared type/constant/function package |
| `Registers.vhd` | VME register map instantiation |
| `Register_Logic.vhd` | Generic register read/write logic |
| `shadow_registers.vhd` | Shadow register implementation (BRAM-based) |
| `GenericReg.vhd` | Generic register cell |
| `jta_channel.vhd` | Single channel processing (shared with LBL design) |
| `jta_bram_dlybuf.vhd` | BRAM-based discriminator delay buffer |
| `jta_dpram_template.vhd` | DPRAM instantiation template |
| `Channel_Readout_Mach.vhd` | Channel FIFO readout state machine |
| `Channel_Readout_Controller.vhd` | Readout controller |
| `Channel_FIFO_Readout_Mach.vhd` | Channel FIFO readout machine |
| `Channel_FIFO_Readout_Mach_Rework_WIP.vhd` | WIP rework of above |
| `chan_trigger_control.vhd` | Per-channel trigger control |
| `baseline_tracker.vhd` | Adaptive baseline tracking |
| `basic_capture_counter.vhd` | Simple capture counter |
| `cfd_disc.vhd` | CFD discriminator (DGS trunk — removed in Majorana branch) |
| `thresh_disc.vhd` | Threshold discriminator |
| `disc_led.vhd` | Discriminator LED output |
| `triple_filter.vhd` | Triple moving-average filter (DGS trunk — removed in Majorana) |
| `single_filter.vhd` | Single moving-average filter |
| `filtered_subtraction.vhd` | Filtered subtraction |
| `decimator.vhd` | Signal decimator |
| `pehq.vhd` | Pulse energy/height quantifier |
| `pileup_processor.vhd` | Pileup detection/processing |
| `Trigger_Mux.vhd` | Trigger multiplexer |
| `event_packer.vhd` | Event packet packer |
| `event_data_fifo.vhd` | Internal event data FIFO |
| `Event_Header_FIFO.vhd` | Event header FIFO |
| `Flag_Queue.vhd` | Trigger flag queue |
| `Timestamp_Generator.vhd` | 48-bit event timestamp |
| `sync_capture_controller.vhd` | Sync/capture controller |
| `sync_capture_counter.vhd` | Sync capture counter |
| `SERDES_RX_Mach.vhd` | SERDES RX state machine |
| `SERDES_TX_Mach_DGS.vhd` | SERDES TX state machine (DGS-specific) |
| `Phase_Hunter.vhd` + `Phase_Hunter_SerDes.vhd` | SERDES phase alignment |
| `CLOCK_MANAGER.vhd` | DCM/clock management |
| `DCM_CONTROLLER.vhd` | DCM controller |
| `dc_balance_mach.vhd` | DC balance encoder state machine |
| `disparity_lookup.vhd` | Disparity lookup table (shared with LBL design) |
| `dp_srl_template.vhd` | SRL delay template |
| `Front_Bus.vhd` | Front Bus module |
| `Lvme.vhd` | Local VME interface |
| `Fifo.vhd` | External FIFO module |
| `mult17x17_u.vhd` | 17×17 unsigned multiplier |
| `ILA.vhd` | ChipScope ILA debug core wrapper |

### DGS vs Majorana Branch Differences

| Feature | DGS Main Trunk | Majorana Branch |
|---------|----------------|-----------------|
| CFD | Present | Removed |
| Triple filter | Present | Removed |
| Threshold discriminator input | Filtered signal | Full 24-bit M1/M2 sums |
| Threshold width | 12-bit | 24-bit |
| BGO veto | Enabled | Extended events enabled (TODO item) |

### Firmware Constant

- `C_FIRMWARE = X"0001"` — the production DGS digitizer uses type code `0x7` (DIG type) in the ANL firmware revision scheme; the `0x0001` constant here is simply a revision number, not the type code.

---

## Cross-References

- [`fpga.md`](fpga.md) — current production DIG/MTRG/RTRG FPGA overview and BUILD_TYPE codes
- [`trig_sys_sim.md`](trig_sys_sim.md) — VHDL testbench in `FPGA/others/Trig_sys_sim/`
- [`data_structures.md`](data_structures.md) — DGS binary event packet format (descends from LBL GRETINA packet)
- [`ANLDAQ_tcpReceiver_Aux.md`](ANLDAQ_tcpReceiver_Aux.md) — `class_DIG.h` decodes the event packets produced by the current DIG firmware
- [`myriad.md`](myriad.md) — MyRIAD GITMO module (interfaces to SERDES_TX_Mach_DGS equivalent)
