# N=126 Target Wheel — Encoder Interface System

Stability: C3 - Structural / stable

**Source:** `DGS_tools_pack/DGS_SVN/psg/N126_target_wheel/` (SVN archive)  
**FPGA firmware:** `DGS_SVN/psg/N126_target_wheel/Firmware/source/` (ISE 14.7, Xilinx XC6SLX9-2TQG144)  
**EPICS IOC (Pi):** `DGS_SVN/psg/N126_target_wheel/RaspberryPi/N126App_RevA/`  
**Code-gen spreadsheet output:** `DGS_SVN/psg/CodeGeneratingSpreadsheetGeneric/Projects/N126_Encoder/SS_output/`  
**Schematic:** `n126_target_wheel.pdf` (OrCAD), PCB: `23pc017-N126TargetWheel.brd`  
**Specs:** `Encoder interface spec.pdf` / `Encoder interface schematic review notes.pdf`  

---

## Purpose

The N=126 target wheel is an **auxiliary nuclear physics device** used with Gammasphere at ANL. It physically rotates a thin nuclear target to expose fresh material to the beam (preventing damage/burnout). The encoder interface board:

- Controls a **DRV8824 stepper motor driver** (for indexed positioning)
- Controls an **L6203 DC motor driver** (for continuous rotation)
- Reads a **quadrature shaft encoder** (A/B/index signals) for position feedback
- Provides **8-channel DAC output** (LTC1660-style, octal) for thresholds/offsets
- Monitors motor current via 4 analog **comparator inputs**
- Interfaces a **numeric LED display** via I²C
- Connects to a **power board** via I²C (I2C_STARTUP_ROM ROM-scripted initialization)
- Provides **BNC coax outputs** for beam sweep control
- All controlled via **EPICS IOC on a Raspberry Pi** (SPI bus to FPGA)

---

## FPGA

**Chip:** Xilinx Spartan-6 **XC6SLX9-2TQG144** ✅ verified 2026-04-21 — `N126.ucf:L9`  
**Toolchain:** Xilinx ISE 14.7  
**EPICS device type:** `PickoffLocalSerial` (same SPI framework as Collector Box / Slope Box IOC)  
**Transaction format:** 24-bit SPI (1-bit direction + 7-bit address + 16-bit data), GPIO for address extension ✅ verified 2026-04-21 — `N126Support.c:L47,50`

### Firmware Source Files

| File | Function |
|------|----------|
| `SS_top.vhd` | Top-level entity `EncoderInterfaceTop` — all port wiring ✅ verified 2026-04-21 — `SS_top.vhd:L26` |
| `n126_pkg.vhd` | Component declarations package |
| `quadrature_decoder.vhd` | Quadrature A/B/index decoder (Digikey/Scott Larson, 2017); debounce, direction, position, sync signals |
| `I2C_STARTUP_ROM.vhd` | ROM-scripted I²C startup sequence for power board init |
| `SCANNER_ROM.vhd` | ROM-driven scanner machine for I²C power board |
| `PI_TRANSACTOR.vhd` | Pi SPI transactor: receives SPI transactions from Raspberry Pi, routes to register banks |
| `LTC1660_controller.vhd` | Octal DAC (LTC1660) SPI controller |
| `sync_capture_controller.vhd` | Sync/capture state machine controller |
| `sync_capture_counter.vhd` | Counter for sync capture |
| `LOOK_UP_TABLE1.VHD` | Address-to-machine-enable lookup table (MACH_ENABLE 16-bit one-hot vector) |
| `I2C_template.vhd` | Generic I²C bus master template |
| `init_prom_generator.xlsx` | Spreadsheet to generate ROM init data |

### Top-Level Ports (key groups)

```
Raspberry Pi GPIO (SPI0 and SPI1) — PI_GPIO_2 through PI_GPIO_27
Stepper motor:   DRV8824_STEP, DIR, NENBL, DECAY, MODE0/1/2, nRESET, nSLEEP, nFAULT, nHOME
DC motor:        DC_MOT_ENBL, DC_MOT_IN1, DC_MOT_IN2; limit switches LIMIT1/LIMIT2
Encoder:         FPGA_ENCODER_A, FPGA_ENCODER_B, FPGA_ENCODER_IDX
Current monitors: COMPARATOR_OUT1–4 (MOTOR_A_PLUS/MINUS, MOTOR_B_PLUS/MINUS IMON)
DAC:             OFFSETANDTHRESHOLDSCK/SDI/CS/CLR
Power board:     PowerConverterEN, PowerConverterSCL, PowerConverterSDA (I²C); PI_5V_DISABLE
LED display:     FPGA_DISP_SCL, FPGA_DISP_SDA (I²C)
Relay:           RELAY1, RELAY2 (solid state relay PRLY_4_1)
BNC outputs:     BNC_DRIVE1, BNC_DRIVE2 (beam sweep)
LEDs/test pts:   SFP_LED1–3, FPGAOPT1–3
50 MHz oscillator: OSC_CLOCK
Pi presence sense: PIPRESENCESENSE
```

---

## EPICS IOC (Raspberry Pi)

**Framework:** Same `PickoffLocalSerial` DTYP as Collector Box / Slope Box IOCs  
**Protocol:** SPI 24-bit transactions (1 dir + 7 addr + 16 data); GPIO extension bits for routing

### EPICS Source Files (`N126App/src/`)

| File | Function |
|------|----------|
| `N126Support.c` | Core device support init, SPI transaction framework |
| `N126Support.h` | Global data structure (`PickoffGlobDataStructure`), I²C flags |
| `N126Support_AI.c` / `_AO.c` | Analog input/output record device support |
| `N126Support_BI.c` / `_BO.c` | Binary input/output record device support |
| `N126Support_MBBI.c` / `_MBBO.c` | Multi-bit binary input/output |
| `N126Ctl_AI.c` / `_AO.c` / `_BI.c` / `_BO.c` / `_MBBI.c` | Control-path device support |
| `N126Display_AI.c` | LED display readback device support |
| `N126Main.cpp` | Standard EPICS iocsh main (Marty Kraimer template) |
| `spi.c` / `spi.h` | Low-level SPI bit-bang driver |
| `bcm2835.c` / `bcm2835.h` | Broadcom BCM2835 GPIO library (Raspberry Pi peripheral access) |
| `NonEPICS_SPI_lib.c` / `.h` | Non-EPICS SPI utility library (testing/diagnostics) |
| `SevenSegment.c` / `.h` | 7-segment LED display driver (I²C-based LED backpack) |
| `initTrace.c` | Initialization trace/log utility |
| `i2c_test.c` | I²C bus test utility |

**Why not asyn?** The `N126Support.c` header explicitly explains: asyn assumes byte-aligned serial transactions, but the DGS SPI protocol uses arbitrary-length transactions (not necessarily multiples of 8 bits). Full-duplex SPI (simultaneous MOSI/MISO) is also required and unsupported by asyn. This is the same rationale as the Slope Box / Collector Box IOCs. ✅ verified 2026-04-21 — `N126Support.c:L26-35`

---

## EPICS PV Register Map

All PVs use `DTYP = PickoffLocalSerial`. Addresses map to FPGA register indices.

### Control Registers (Write)

| PV | Addr | Function |
|----|------|----------|
| `FPGA_CTL_REG` | 1 | FPGA control — resets all I²C transactor state machines |
| `DRV8824_MANUAL_CTL_REG` | 2 | Manual stepper motor driver control |
| `DC_MOTOR_DRV_REG` | 3 | DC motor driver control |
| `INITIAL_PWR_ROM_ADDRESS` | 8 | ROM start address for power board I²C scanner |
| `PWR_SCAN_CTL_REG` | 12 | Power scanner control (select alternate ROM program before reset) |
| `MISC_CTL2_REG` | 14 | Miscellaneous control 2 |
| `PWR_I2C_MANUAL` | 15 | Manual access to power board I²C bus |
| `DAC_CHAN0` – `DAC_CHAN7` | 16–23 | 8-channel DAC output (LTC1660 octal DAC for offsets/thresholds) |
| `PowerBrdWritePort` | 122/123/125 | Power board write port / self-test controls |
| `IDX_DEBOUNCE_REG` | 124 | Index signal debounce time (how long to debounce) |

### Status Registers (Read)

| PV | Addr | Function |
|----|------|----------|
| `ENCODER_POS_REG` | 4 | Full encoder position register (16-bit) — polled 1 Hz |
| `ROTATION_CNT_REG` | 5 | Rotation count (16-bit) — total full rotations |
| `RPM_COUNT_REG` | 6 | RPM count (16-bit) — latched RPM value |
| `SCANNER_GENERAL_STATUS` | 7 | General power scanner status bits |
| `PWR_GO_COUNT` | 9 | Number of completed power I²C transactions |
| `POWER_ACK_ERR_COUNT` | 10 | Power board I²C acknowledgment error count |
| `PWR_SCAN_ROM_ERR` | 11 | ROM error address from last failed power scan |
| `MISC_STAT_REG` | 13 | Miscellaneous status |
| `RPS_COUNT_REG` | 121 | Revolutions-per-second count |
| `Code_Date` | 126 | Firmware date (polled 1 Hz) |
| `Code_Revision` | 127 | Firmware revision (polled 1 Hz) |

### Bitgroup PVs (sub-field access via mask)

| PV | Register | Bits | Function |
|----|----------|------|----------|
| `PowerBoardEnable` | `FPGA_CTL_REG` (1) | 3:0 | Power board enable |
| `ENCODER_POSITION` | `ENCODER_POS_REG` (4) | 9:0 | 10-bit encoder position |
| `ROTATION_COUNT` | `ROTATION_CNT_REG` (5) | 15:0 | Full rotation counter |
| `LATCHED_RPM_COUNT` | `RPM_COUNT_REG` (6) | 15:0 | Latched RPM |
| `PWR_ScanStrtAddr` | `INITIAL_PWR_ROM_ADDRESS` (8) | 8:0 | ROM start address |
| `Power_I2C_delay` | `INITIAL_PWR_ROM_ADDRESS` (8) | 15:9 | I²C transaction delay |
| `PWR_Scan_Err_Addr` | `PWR_SCAN_ROM_ERR` (11) | 8:0 | ROM address at error |

---

## Quadrature Decoder (`quadrature_decoder.vhd`)

Decodes A/B/index quadrature encoder signals:
- Inputs: `a`, `b` (quadrature signals), `x` (index/zero pulse)
- Configurable debounce: `debounce_time_same_dir` and `debounce_time_diff_dir` (in clock cycles) — allows different debounce windows for same-direction vs direction-change events ✅ verified 2026-04-21 — `quadrature_decoder.vhd:L36-37`
- Outputs: `position` (13-bit), `direction`, `cycle_syncd`, `position_error`, `position_syncd`, `position_update`, `direction_update` ✅ verified 2026-04-21 — `quadrature_decoder.vhd:L38` (`12 downto 0`)
- `set_origin` input: active-low synchronous clear of position counter (re-home) ✅ verified 2026-04-21 — `quadrature_decoder.vhd:L35`
- Source: Digikey/Scott Larson open-source quadrature decoder, 2017, adapted for this project ✅ verified 2026-04-21 — `quadrature_decoder.vhd:L6-18` (Digi-Key license header + Scott Larson v1.0 2017-09-07)

---

## Stepper Motor — DRV8824

Texas Instruments DRV8824 stepper motor driver IC. FPGA controls:
- `DRV8824_STEP` — step pulse
- `DRV8824_DIR` — direction
- `DRV8824_NENBL` — enable (active low)
- `DRV8824_DECAY` — decay mode select
- `DRV8824_MODE0/1/2` — microstepping resolution
- `DRV8824_nRESET` — reset (active low)
- `DRV8824_nSLEEP` — sleep mode (active low)
- `DRV8824_nFAULT` — fault flag (input from driver IC)
- `DRV8824_nHOME` — home position signal

---

## DC Motor — L6203

Used for **continuous wheel rotation**. FPGA drives:
- `DC_MOT_ENBL` — motor enable
- `DC_MOT_IN1`, `DC_MOT_IN2` — direction/braking
- `LIMIT1`, `LIMIT2` — limit switch inputs (from wheel frame)

---

## I²C Power Board

A separate power board is initialized via ROM-scripted I²C (`I2C_STARTUP_ROM.vhd`):
- ROM data: generated by `init_prom_generator.xlsx`
- Machine runs at 4 MHz clock; RETIMED_CMD_FIFO_WE uses 100 MHz clock
- Supports NACK handling, READ, SAVE, EXTEND, LOOP operations
- Power board I²C is also accessible manually via `PWR_I2C_MANUAL` PV

---

## LED Display

Numeric 7-segment LED display (Adafruit I²C backpack, HT16K33 driver):
- Connected via `FPGA_DISP_SCL` / `FPGA_DISP_SDA`
- Driven by `SevenSegment.c` (BCM2835 I²C through Pi GPIO)
- Shows current target wheel position

---

## Relationship to Other DGS Systems

- Uses same **SPI/EPICS framework** (`PickoffLocalSerial`, `NonEPICS_SPI_lib`) as Collector Box Pi IOC and Slope Box Pi IOC
- **Not part of the main DGS DAQ trigger chain** — auxiliary experiment hardware
- **No TTCL link** — standalone Pi EPICS IOC; not connected to DGS Master Trigger
- **Location in SVN:** `psg/` (not `dgs/`) — part of Paul Gruss's physics group hardware, not core DGS firmware

---

## Related Files

- `knowledgeBase/collectorboxpi.md` — similar Pi EPICS IOC pattern
- `knowledgeBase/slope_box_interface.md` — same `PickoffLocalSerial` framework
- `knowledgeBase/sbx.md` — Slope Box hardware context
- `knowledgeBase/myriad.md` — another auxiliary detector interface

---

*Created: 2026-04-21 from SVN source exploration (`psg/N126_target_wheel/`). No prior knowledge base entry existed for this system.*
