# MγRIAD - Multipurpose γ-Ray Interface to Auxiliary Detectors

**Full name:** Multipurpose γ-Ray Interface to Auxiliary Detectors
**Source:** `DGS_SVN/dgs/MyRIAD/Documentation/MyRIAD Abridged User Notes.pdf` (v0.0, 2018-04-27, J. Anderson) + `MyRIAD User Manaual.pdf` (v1.2, March 2015, J. Anderson - full register map in Section 3)
**Also:** `DGS_SVN/dgs/MyRIAD/Documentation/MYRIAD_Module_Specification.pdf`
**Wiki PDFs:** `https://wiki.anl.gov/wiki_gsdaq/images/4/40/MyRIAD_User_Manaual.pdf`

Stability: C3 - Structural / stable

---

## Purpose

The MγRIAD bridges auxiliary detectors (NIM/ECL-based) to the DGS/GRETINA TTCL trigger system. It:
- Receives the TTCL timestamp from the Master Trigger via RJ45 (Cat5e, same as digitizer)
- Propagates DGS/GRETINA timestamps to auxiliary VME-based DAQs
- Sends trigger messages back to the Master Trigger when the auxiliary detector fires
- Provides local coincidence logic between local detector and auxiliary trigger
- Interfaces legacy FERA ADC systems via ECL

Connected to **link U** of the Master Trigger. ✅ verified 2026-04-07 - `top.vhd:L3562` (`RECEIVED_MYRIAD_DATA => LAT_LINKU_RX`).

---

## Front Panel Connectors

### RJ45 - TTCL Link
- **Not Ethernet** - Cat5e cable to DGS/GRETINA trigger module only ✅ verified 2026-04-17 - `MyRIAD User Manaual.pdf` p.2
- Carries TTCL (Trigger Timing and Control Link) - same protocol as digitizer RJ45 ✅ verified 2026-04-17 - `MyRIAD User Manaual.pdf` p.2: "SerDes interface compatible with Trigger Timing and Control Link specification" + "The RJ-45 connector is NOT ETHERNET. This connector uses Cat5e cable"
- LEDs on connector indicate SERDES link lock state

#### MγRIAD → MTRG SERDES Data Frame Format
_Source: `MYRIAD_RCV_MACH.vhd` header comment ✅ verified 2026-04-07_

MγRIAD sends a **5-word repeating frame** over SERDES (16-bit words):

| Word | Bits 15:13 | Bit 12 | Bit 11 | Bits 10:8 | Bits 7:0 |
|------|-----------|--------|--------|-----------|----------|
| All  | bit15=10MHz flag, bits14:13=00 | Raw trigger (NIM or ECL input) | Gated trigger (coincidence logic output) | 3-bit ordinal counter (sync check) | word-specific (below) |
| 00 | - | - | - | - | `0xAD` (part of reset sync `0x0BAD`) |
| 01 | - | - | - | - | NIM input states (8 bits) |
| 02 | - | - | - | - | ECL input states |
| 03 | - | - | - | - | FERA control states |
| 04 | - | - | - | - | 8-bit frame counter (sync check) |

MTRG receiver (`MYRIAD_RCV_MACH`) locks onto this stream and extracts NIM/ECL state, raw trigger, gated trigger, and 10 MHz flag. Reset detected by `0x0BAD` signature in word 00.

### JTAG
- Direct FPGA access via Xilinx JTAG programmer

### ECL CTL Header (10-pin, 2 differential inputs + 3 differential outputs)
- Pinout compatible with FERA ADC system cables ✅ verified 2026-04-06 - MyRIAD Abridged User Notes.pdf p.3
- Firmware-defined; current DGS build uses ECL CTL outputs as diagnostics:
  - **FERA FULL** pair → copy of multiplexed 50 MHz FPGA clock (locked = TTCL sync OK) ✅ verified 2026-04-06 - MyRIAD Abridged User Notes.pdf p.3
  - **FERA ACK** pair → copy of **NIM input 7** (0-based index, i.e. the GP Counter NIM input) ✅ verified 2026-04-16 - `MyRIAD.vhd:L1926` (`FERA_ACK_pin <= NIM_IN_PIN(7)`) ⚠️ PDF (p.3) says "NIM input 1" - VHDL is authoritative; PDF may refer to older firmware
  - **FERA OVF** pair → driven by `CLOCK_50MHZ` via DDR output flip-flop (OFDDRRSE), producing a 50 MHz square wave - used for jitter comparison with `MAIN_FPGA_MACH_CLK_1_pin` ✅ verified 2026-04-16 - `MyRIAD.vhd:L1929-1940` (`CLOCK_50_DRIVER_DDR` OFDDRRSE, MBO comment 2022-08-10: "changed to CLOCK_50MHz for jitter comparison")
  - **FERA WSI** pair (input) → alternate source for local trigger signal; if `GATING_REG` bit is set, rising edge of `FERA_WSI_IO_pin` is used as trigger input instead of NIM In 0 ✅ verified 2026-04-16 - `MyRIAD.vhd:L1062-1064` (`if FERA_WSI_PIPE(1)='0' and FERA_WSI_PIPE(0)='1'` → local trigger)
  - **FERA VETO** pair (input) → readable via `ECL_STATUS_B[0]` status register; no active trigger function in current firmware ✅ verified 2026-04-16 - `MyRIAD.vhd:L1976` (`ECL_STATUS_B <= "00000000000000" & FERA_WSI_IO_pin & FERA_VETO_pin`)

### ECL I/O Header (16 differential ECL signals)
- Default: 16 receiver inputs (100 Ω termination per differential pair)
- Assembly positions allow installing driver chips instead (reconfigurable)
- All signals available to firmware

### NIM I/O - 8 inputs, 4 outputs
Layout: two groups of 4 inputs (I), two groups of 2 outputs (O). ✅ verified 2026-04-06 - MyRIAD Abridged User Notes.pdf p.5 Fig.1

#### NIM Input Functions
| Input | Function |
|-------|---------|
| **NIM In 0** (upper left) | **Local system trigger input** - latches timestamp on each edge; minimum pulse width **100 ns** ✅ verified 2026-04-08 - MYRIAD_Module_Specification.pdf ("100ns wide should be used") |
| **NIM In 1** | Local coincidence input - starts coincidence timer after NIM In 0 edge; asserts coincidence if NIM In 1 fires before timeout |
| NIM In 2-7 | General purpose - counted only (no trigger function as of 2018-04-27) |

- All NIM inputs connected to 16-bit edge counters (sampled at 100 MHz) ✅ verified 2026-04-06 - MyRIAD Abridged User Notes.pdf p.5
- All NIM input states regularly sent to Master Trigger over SERDES ✅ verified 2026-04-06 - MyRIAD Abridged User Notes.pdf p.5

#### NIM Output Functions

| Output | Function |
|--------|----------|
| **NIM Out 0** | Echoed copy of NIM In 0 (fixed, not configurable) ✅ verified 2026-04-08 - MYRIAD_Module_Specification.pdf §3.5 |
| **NIM Out 1** | Buffered copy of NIM In 1 (fixed, not configurable) ✅ verified 2026-04-08 - MYRIAD_Module_Specification.pdf §3.5 |
| **NIM Out 2** | Copy of 'sync flag' from DGS/Gretina master trigger over SERDES; pulses every 2 μs - useful for timing verification ✅ verified 2026-04-08 - MYRIAD_Module_Specification.pdf §3.5 |
| **NIM Out 3** | Coincidence logic output (fires when NIM In 1 occurs within coincidence window after starting trigger) ✅ verified 2026-04-08 - MYRIAD_Module_Specification.pdf §3.4 |

> Note: GATING_REG (0x0702) bits 0-1 select the coincidence *starting trigger* source (bit 0 = NIM In 0, bit 1 = SERDES trigger from master). The NIM Out 0/1 outputs are hardwired echoes - they are not muxed by GATING_REG. Prior doc claiming bits 3:2 and 6:4 controlled NIM Out 0/1 selection was incorrect.

---

## Front Panel LEDs (3×3 array)

```
P1  P2  V4
F3  F5  S6
S7  S8  S9
```

| LED | Label | Meaning |
|-----|-------|---------|
| 1 | P | +5V VME power present (blue) |
| 2 | P | DC-DC converter subsidiary voltages OK (green) |
| 4 | V | VME access activity (flashes on each VME access) |
| 3 | F | FPGA configuration in progress (blinks) |
| 5 | F | Firmware-specific main FPGA indicator (currently unused) |
| 6 | S | Blinks when internal coincidence logic satisfied |
| 7 | S | Blinks on NIM input 1 leading edges |
| 8 | S | Blinks on local detector TRIGGER IN edges (NIM In 0 or ECL FERA WSI, firmware-selectable) |
| 9 | S | Blinks on NIM input 7 leading edges |

---

## DGS Usage

In DGS, MγRIAD is connected to **link U** of the MTRG: ✅ verified 2026-04-07 - `MTRG/top.vhd:L3562`
- Receives TTCL timestamps → propagates to auxiliary VME DAQs
- Sends auxiliary detector trigger messages back to MTRG
- Local NIM input 0 = aux detector trigger (e.g. ancillary detector, tape station) ✅ verified 2026-04-17 - `MyRIAD.vhd:L393,L1057-1058` (`AUX_DETECTOR_TRIG` driven by `NIM_IN_PIPE(0)` rising edge)
- Local NIM input 1 = coincidence gate signal ✅ verified 2026-04-17 - `MyRIAD.vhd:L1692` (ILA comment: "coincidence logic 2nd NIM input"; L1464: `NIM_IN_PIPE(1)` rising edge → TRIG_COINC_STATE)

Coincidence timer is programmable via register - window defines valid auxiliary trigger window relative to NIM In 0.

---

## VME Register Map
_Source: MyRIAD User Manual v1.2, 2015/2018, J. Anderson - Section 3.2/3.3_

All registers are 16-bit, A16/D16. Addresses in hex.

| Address | Mode | Register | Function |
|---------|------|----------|----------|
| 0x0000 | R | `board_id` | Board address + firmware info |
| 0x0004 | RW | `fifo_status` | FIFO status |
| 0x0020 | R | `hardware_status` | DCM status information |
| 0x040C | W | `pulsed_control` | Write-only self-clearing: resets |
| 0x040E | RW | `fifo_control` | FIFO operational modes |
| 0x0410 | RW | `Capture_time` | SerDes capture time (FIFO test) |
| 0x0600 | R | `code_revision` | Firmware revision |
| 0x0604 | R | `code_date` | Compilation date (MMDD) |
| 0x0606 | R | `code_year` | Compilation year (YYYY) |
| 0x0700 | R | `NIM_input_status` | Current state of all NIM inputs ✅ verified 2026-04-19 - `registers.vhd:L257` (`when X"0700" => VME_DATA_OUT <= NIM_STATUS`) |
| 0x0702 | RW | `GATING_REG` | Coincidence starting trigger select: bit 0 = use NIM In 0 as starting trigger; bit 1 = use SERDES trigger from master as starting trigger. (NIM Out 0/1 are hardwired echoes, not controlled by this register.) ✅ verified 2026-04-08 - MYRIAD_Module_Specification.pdf §3.4 |
| 0x0704 | R | `ECL_input_status_A` | Current state of ECL data inputs ✅ verified 2026-04-19 - `registers.vhd:L260` (`when X"0704" => VME_DATA_OUT <= ECL_STATUS_A`) |
| 0x0706 | R | `ECL_input_status_B` | Current state of ECL control inputs ✅ verified 2026-04-19 - `registers.vhd:L261` (`when X"0706" => VME_DATA_OUT <= ECL_STATUS_B`) |
| 0x0708 | R | `LATCHED_TIMESTAMP_A` | Timestamp bits 47:32 (latched on NIM In 0) ✅ verified 2026-04-19 - `registers.vhd:L262` (`when X"0708" => VME_DATA_OUT <= REG_708 -- RELATCHED_TIMESTAMP(47 downto 32)`) |
| 0x070A | R | `LATCHED_TIMESTAMP_B` | Timestamp bits 31:16 ✅ verified 2026-04-19 - `registers.vhd:L263` |
| 0x070C | R | `LATCHED_TIMESTAMP_C` | Timestamp bits 15:0 ✅ verified 2026-04-19 - `registers.vhd:L264` |
| 0x070E | RW | `SerDes_COMMAND_FORMAT` | Select DGS or GRETINA command format ✅ verified 2026-04-19 - `registers.vhd:L265` (`when X"070E" => VME_DATA_OUT <= REG_70E_IN -- SERDES_COMMAND_FORMAT`) |
| 0x0710 | RW | `Coincidence_window_delay` | Delay value for coincidence trigger window (writable). Default=0x0100. ✅ verified 2026-04-20 - `registers.vhd:L265,L394` (both read and write cases present; `xREG_710` default `X"0100"` at L379) |
| 0x0712 | RW | `coincidence_window_width` | Gate width for coincidence trigger (writable). Default=0x0110. ✅ verified 2026-04-20 - `registers.vhd:L266,L395` (read+write; `xREG_712` default `X"0110"` at L380) |
| 0x0714 | R | `LIVE_TIMESTAMP_A` | Running timestamp bits 47:32 ✅ verified 2026-04-20 - `registers.vhd:L267` (`VME_DATA_OUT <= REG_714 -- SYSTEM_TIMESTAMP(47 downto 32)`) |
| 0x0716 | R | `LIVE_TIMESTAMP_B` | Running timestamp bits 31:16 ✅ verified 2026-04-20 - `registers.vhd:L268` |
| 0x0718 | R | `LIVE_TIMESTAMP_C` | Running timestamp bits 15:0 ✅ verified 2026-04-20 - `registers.vhd:L269` |
| 0x071A | RW | *(reserved)* | **Reserved - do not read or write.** ⚠️ Previously mislabeled as `TIMESTAMP_ERROR_CTRL` - corrected 2026-04-20. ✅ verified 2026-04-20 - `registers.vhd:L270` (comment: "reserved, do not read or write") |
| 0x071E | R | `TIMESTAMP_ERROR_CNT_A` | Timestamp error counter bits 31:16 (errors from SERDES) ✅ verified 2026-04-20 - `registers.vhd:L272` (`REG_TS_ERR_COUNT(31 downto 16)`) |
| 0x0720 | R | `TIMESTAMP_ERROR_CNT_B` | Timestamp error counter bits 15:0 ✅ verified 2026-04-20 - `registers.vhd:L273` (`REG_TS_ERR_COUNT(15 downto 0)`) |
| 0x0722 | RW | `TTCL_TIME_OFFSET` | Master trigger re-issue offset control ✅ verified 2026-04-20 - `registers.vhd:L274,L398` (read + writable; comment: "TTCL_TIME_OFFSET register") |
| 0x0724 | R | `MISSED_TRIG_COUNT` | Counter of missed re-issued trigger messages ✅ verified 2026-04-20 - `registers.vhd:L275` (`std_logic_vector(REG_724) -- MISSED_TRIG_COUNT`) |
| 0x0726 | R | `DLYD_TRIG_ERR_COUNT` | Counter of re-issued trigger errors ✅ verified 2026-04-20 - `registers.vhd:L276` (`std_logic_vector(REG_726) -- DLYD_TRIG_ERR_COUNT`) |
| 0x0728 | RW | `PROPAGATION_CONTROL` | Controls SerDes command processing ✅ verified 2026-04-20 - `registers.vhd:L277,L399` (read + write) |
| 0x07EC | R | `FIFO_COUNTER` | Number of triggers stored in FIFO ✅ verified 2026-04-19 - `registers.vhd:L280` (`when X"07EC" => VME_DATA_OUT <= std_logic_vector(FIFO_COUNTER)`) |
| 0x07F0 | R | `TRIG_COUNTER` | Total triggers received ✅ verified 2026-04-19 - `registers.vhd:L281` (`when X"07F0"`); **KB previously said 0x07EE - corrected** |
| 0x07F2-0x07Fe | R | `USER_COUNTER_0-6` | Edge counters for NIM inputs 0-6 ✅ verified 2026-04-19 - `MyRIAD.vhd:L910` (`USER_COUNTERs reads back at 0x07F2 - 0x0800`; 8 total NIM input counters) |
| 0x0800 | R | `USER_COUNTER_7` | Edge counter for NIM input 7 ✅ verified 2026-04-19 - `MyRIAD.vhd:L910` (0x0800 = last of 8 USER_COUNTERs driven by `NIM_IN` rising edges, L997-1005) |
| 0x0848 | RW | `sd_config` | SerDes configuration register ✅ verified 2026-04-20 - `registers.vhd:L295,L401` (read: `xREG_848 -- SERDES configuration register`; write case at L401) |
| 0x0860-0x0866 | R | TDC vernier data | 4 × 16-bit TDC vernier words (bits 63:48, 47:32, 31:16, 15:0) ✅ verified 2026-04-19 - `registers.vhd:L296-299` (`TDC_VERNIER(63 downto 48/47 downto 32/31 downto 16/15 downto 0)`) |
| 0x0900 | RW | `fpga_ctrl_reg` | Main FPGA configuration control |
| 0x0902 | R | `vme_status` | VME FPGA status |
| 0x0904 | R | `vme_aux_status` | VME FPGA auxiliary status |
| 0x0906 | RW | `config_req_ack` | Configuration request/ack (write triggers FPGA reconfiguration; read back = ack) ✅ verified 2026-04-20 - `register_block.vhd:L243` (comment: "0x0906 (configuration request/ack)") |
| 0x0908 | RW | `flash_vpen` | Flash write-protect enable |
| 0x090A | RW | `config_start_low` | FPGA config start address (low) |
| 0x090C | RW | `config_start_high` | FPGA config start address (high) - only bits [7:0] used ✅ verified 2026-04-20 - `register_block.vhd:L181,L252` (read/write cases both `config_start_high`; note: **KB previously had 0x090C and 0x0910 swapped**) |
| 0x090E | RW | `config_stop_low` | FPGA config stop address (low) |
| 0x0910 | RW | `config_stop_high` | FPGA config stop address (high) - only bits [7:0] used; default=0x0007 (stop at 0x70000) ✅ verified 2026-04-20 - `register_block.vhd:L185,L256` (read/write cases both `config_stop_high`; L134 default `X"0007"`) |
| 0x0918 | RW | `vme_sandbox1` | VME sandbox register 1 (debug) |
| 0x091A | RW | `vme_sandbox2` | VME sandbox register 2 (debug) |
| 0x091C | RW | `vme_sandbox3` | VME sandbox register 3 (debug) |
| 0x091E | RW | `vme_sandbox4` | VME sandbox register 4 (debug) - R/W, default=0x3333 ✅ verified 2026-04-20 - `register_block.vhd:L195,L265` (both read and write cases; L118 default `X"3333"`) |
| 0x1000 | R | `fifo` | FIFO read port - each read returns one 16-bit word from the trigger FIFO |

_Note: Above 0x0902 entries sourced from `MYRIAD_Module_Specification.pdf` (v1.0, 2014). Some may differ from User Manual v1.2 (2015)._

### Coincidence Logic
State machine: fires when "starting trigger" occurs (NIM In 0 if `GATING_REG[0]`=1, or SERDES trigger if `GATING_REG[1]`=1). After `Coincidence_window_delay` × 20 ns, opens a window of width `coincidence_window_width`. If NIM In 1 fires during window → NIM Out 3 asserts. Timeout without match → return to idle.

---

---

## FPGA Firmware Architecture
_Source: `DGS_tools_pack/FPGA/others/MyRIAD/MAIN_FPGA/Source/MyRIAD.vhd` (explored 2026-04-08)_

**Main FPGA chip:** Xilinx Spartan-3 **XC3S1000-FG456** ✅ verified 2026-04-19 - `FPGA/others/MyRIAD/MAIN_FPGA/Work13/MyRIAD.xise`: Device=xc3s1000, Package=fg456, Family=Spartan3

**VME FPGA chip:** Xilinx Spartan-3 **XC3S400-FG320** ✅ verified 2026-04-19 - `FPGA/others/MyRIAD/VME_FPGA/Work13.4/Work13.4.xise`: Device=xc3s400, Package=fg320, Family=Spartan3

### Generic Parameter
```vhdl
COMMAND_LINE_COMMAND_FORMAT : integer
  -- 0 = DGS Master format
  -- 1 = DGS Router format
  -- 2 = GRETINA Master format
```
Controls how the SERDES RX machine interprets incoming commands - same hardware supports all three modes via a compile-time generic.

### Key Internal Modules

| Module/Instance | File | Function |
|----------------|------|----------|
| `MAIN_DCM_CTRL` | `DCM_CONTROLLER.vhd` | Manages Xilinx DCM - generates 50/100/250 MHz clocks from oscillator or SERDES recovered clock; clock source selectable via `CLOCK_SEL_pin` |
| `INST_SERDES_RX_Mach` | `SERDES_RX_Mach_R2.vhd` | Receives 18-bit SERDES data; DC-balance decoding; command frame parsing per `COMMAND_FORMAT` |
| `TX_INST` | `SERDES_TX_MACH.vhd` | Transmits 18-bit SERDES words at 100 MHz; DC-balanced via disparity lookup |
| `U20` | `registers.vhd` | VME-accessible register file (16-bit, A16/D16); all registers in MYRIAD's VME map above |
| `NIM_PIPES_BLK` | `MyRIAD.vhd` (inline generate loop × 8) | Fixed 3-stage shift-register pipelines for each NIM input; samples at 100 MHz, detects rising edge via `NIM_IN_PIPE(i)(1:0) = "01"`. **Note:** `NIM_Delay.vhd` exists in Source/ (a programmable circular-buffer delay, 2 inputs) but is **not instantiated** in `MyRIAD.vhd` — it appears to be from the Digitizer Tester project header. ✅ verified 2026-04-23 - `MyRIAD.vhd:L985-995` (inline generate, no component instantiation) |
| `FIFO_TIMESTAMP_MACH` | inline process | Writes trigger timestamps + NIM/ECL states into dual external FIFOs (A and B, 18-bit wide each = 16-bit data + 2 flag bits) |
| `TRIGGER_COINC_PROC` | `MyRIAD.vhd` (inline process) | Coincidence logic: waits for `LATCH_TRIG_FLAG` (aux detector trigger), counts down `AUX_DETECTOR_DLY_CNT`; if NIM In 1 rises within that window → `COINC_TRIG_FLAG` asserts (NIM Out 3). **Note:** `mstr_mach.vhd` exists in Source/ but is a separate Satellite Master state machine (TTCL command generator), **not** the coincidence logic. It is not instantiated in `MyRIAD.vhd`. ✅ verified 2026-04-23 - `MyRIAD.vhd:L1427-1470` (`TRIGGER_COINC_PROC`, `TRIG_COINC_STATE` FSM) |
| `Phase_Hunter_SerDes` | `Phase_Hunter_SerDes.vhd` | Aligns SERDES receive clock phase for proper data sampling |

### TDC
A TDC block (`tdc_unit2`) exists in `Source/` but is **commented out** in `MyRIAD.vhd` - it was tested but not integrated into the production firmware. The `TDC_LOOPBACK_OUT_pin` is hard-wired to `'0'`. Only `TDC_VERNIER_OUT` signal is declared; actual TDC is in MTRG Main FPGA, not here.

### FIFO Architecture
Dual external FIFOs (A and B) hold trigger records for VME readout:
- **FIFO A** → VME data bits [15:0] (lower word) ✅ verified 2026-04-17 - `MyRIAD.vhd:L37` ("Bits (15:0) of FIFO 'A' drive directly out VME data bits 15:0")
- **FIFO B** → VME data bits [31:16] (upper word) ✅ verified 2026-04-17 - `MyRIAD.vhd:L73` ("Bits (15:0) of FIFO 'B' drive directly out VME data bits 31:16")
- 18-bit write port: bits[15:0] = data, bit16 = FIFOFLAG0/2 (control), bit17 = FIFOFLAG1/3 (control) ✅ verified 2026-04-17 - `MyRIAD.vhd:L36,L72` (header comments)
- Clock: 100 MHz write; VME-synchronous read ✅ verified 2026-04-17 - `MyRIAD.vhd:L466,L1536` (FIFODATA_IOB process clocked on CLOCK_100MHZ)
- Reset via `PULSED_CTRL[5]` (self-clearing pulsed control register bit) ✅ verified 2026-04-17 - `MyRIAD.vhd:L1625` (comment: "hit PULSED_CTRL(5)"), L1817/1839 (MRS driven by `not PULSED_CTRL(5)`)

### Git Location
`DGS_tools_pack/FPGA/others/MyRIAD/MAIN_FPGA/Source/` (ISE project, Spartan-3)

---

## ANLDAQ / IOC Integration
_Source: `ANLDAQ/gui/gui_MTRG.py`, `ANLDAQ/gui/link_sys.py`, `ANLDAQ/ioc/db/MTrigUser.template` (explored 2026-04-23)_

The Master Trigger IOC and GUI expose MγRIAD as a distinct Link U / overlap-control path:

- `EN_MYRIAD_LINK_U` is the trigger-mask enable bit for the MγRIAD/Link U source in the MTRG trigger mask table. ✅ verified 2026-04-23 - `MTrigUser.template:L33605-L33613`
- `link_sys.py` explicitly clears `EN_MYRIAD_LINK_U` during trigger reset/setup, so scripted trigger reconfiguration treats MyRIAD as a standard selectable trigger source. ✅ verified 2026-04-23 - `link_sys.py:L105-L113`
- `gui_MTRG.py` provides a `MYR_TRIGGER_TYPE_SELECT` two-state button labeled **Myriad Raw / Myriad Gated**, confirming the GUI can choose whether MTRG uses raw or coincidence-gated MγRIAD trigger information. ✅ verified 2026-04-23 - `gui_MTRG.py:L199-L206`
- `MYRIAD_OVERLAP_DELAY` and the associated overlap-enable mask live in `reg_MYRIAD_OVERLAP_CTL`, showing that MTRG can require time overlap between the MγRIAD path and selected trigger algorithms. ✅ verified 2026-04-23 - `MTrigUser.template:L69907-L69919`

## Related Files
- `connectors.md` - links R and U on MTRG where MγRIAD connects
- `ttcl.md` - TTCL protocol MγRIAD uses for trigger communication
- `wiki_gsdaq.md` - MγRIAD mentioned in DAQ system overview

## SVN Location
`DGS_tools_pack/DGS_SVN/dgs/MyRIAD/`

---

## Firmware Revision (from Source)
_Source: `DGS_tools_pack/FPGA/others/MyRIAD/MAIN_FPGA/Source/MyRIAD_pkg.vhd` - verified 2026-04-12_

| Constant | Value | Meaning |
|----------|-------|---------|
| `cCODE_REVISION` | `0x0B16` | PCB rev=0 (no proto suffix), firmware type=B (MyRIAD expansion), major=1, minor=6 |
| `cCODE_DATE_MMDD` | `0x0810` | August 10 |
| `cCODE_DATE_YEAR` | `0x2022` | 2022 |

**Firmware type code B** = "MyRIAD Trigger expansion module" - same numeric encoding as all other DGS/GRETINA firmware (see `MyRIAD_pkg.vhd:L35-51` for full table). This is the production firmware as of 2022-08-10.

---

## GITMO_TOP.vhd - Legacy Module in Same FPGA Repo

`FPGA/others/MyRIAD/MAIN_FPGA/Source/GITMO_TOP.vhd` (795 lines) is a **separate legacy module** stored in the same repo, not the current MγRIAD firmware. It is **not built or deployed** on the current DGS/Gammasphere setup.

**GITMO = Gammasphere Interface to Trigger MOdule** - author: John Anderson (ANL).

**Purpose:** Collect clock and trigger information from the **Gammasphere Master Trigger crate** (VXI-bus based), pack into a data stream, and transmit over SERDES to control the Digital Gammasphere (DGS) Master Trigger. This was the bridge between the legacy analog Gammasphere trigger hardware and the digital DGS system during the transition period.

**Key hardware interfaces:**
- `TRIG0_FROM_VXI_pin` - 'early' trigger from Gammasphere VXI backplane
- `VXI_RDY_BSY_IN_T_pin` - VXI Ready/Busy bus signal
- `TTLTRIG_pin` - TTL trigger inputs (2 bits)
- `NIM_IN_pin`, `ECL_IN_pin` - NIM/ECL front panel inputs
- `FERA_*` - FERA ADC control interface (same as MγRIAD)
- SERDES I/O to/from DGS MTRG (same DS92LV18 chip as MγRIAD)

**Clocks:** 10 MHz VXI clock → DCM → 50 MHz logic clock; ICS502 PLL generates SERDES TCLK.

**Architecture:** `mstr_mach` sub-component (data generator) collects NIM/ECL/VXI trigger inputs and formats a SERDES command stream → `COMMAND_OUT` → sent to DGS MTRG Link L.

> **Relationship to current system:** The MTRG's `GITMO_TRIGGER.vhd` algorithm (Link L input) and `GITMO_RCV_MACH.vhd` receiver were originally designed to receive data from this GITMO_TOP module. Current DGS uses Link L for remote-master or GRETINA interconnects, not the physical GITMO hardware. The GITMO_TOP.vhd is preserved as historical reference. ✅ verified 2026-04-17 - `GITMO_TOP.vhd` header comment + port declarations

---

*Created: 2026-04-05 (from SVN MyRIAD Abridged User Notes PDF). FPGA firmware section added 2026-04-08. Firmware revision verified from source 2026-04-12. GITMO_TOP section added 2026-04-17.*

## Cross-References

- `knowledgeBase/connectors.md` - MTRG front-panel connector pinouts; MγRIAD connects to Link U (NIM I/O, ECL CTL, ECL I/O, RJ45 TTCL)
- `knowledgeBase/deep_fpga_MTRG_MAIN.md` - MTRG Main FPGA: Link U SERDES reception, `RECEIVED_MYRIAD_DATA` signal
- `knowledgeBase/ttcl.md` - TTCL spec; MγRIAD receives timestamps and trigger decisions from MTRG over SERDES
- `knowledgeBase/fpga.md` - 3-tier trigger hierarchy; MγRIAD as auxiliary trigger input via Link U
