# SlopeBox Extension (SBX) — Control FPGA Firmware

Stability: C3 - Structural / stable

**Source:** `~/FPGA_svn2git/MotherBoard_git/CtrlFPGA/Source/`  
**Date analyzed:** 2026-04-24  
**Author:** John T. Anderson (ANL), created 2020-10-05  
**Related KB files:** [slope_box_interface.md](slope_box_interface.md), [fpga.md](fpga.md)

---

## Table of Contents

1. [Target Device & Tools](#target-device--tools)
2. [Source Files](#source-files)
3. [Overview / Role](#overview--role)
4. [Clock Architecture](#clock-architecture)
5. [SPI Communication Interface (PI_TRANSACTOR)](#spi-communication-interface-pi_transactor)
6. [Address Map & Register File](#address-map--register-file)
7. [Dual-Port RAM (DPRAM)](#dual-port-ram-dpram)
8. [I2C Subsystem (I2C_template)](#i2c-subsystem-i2c_template)
9. [I2C Scanner Machines](#i2c-scanner-machines)
10. [BGO Discriminator Interface](#bgo-discriminator-interface)
11. [Slope Box Serial Interface](#slope-box-serial-interface)
12. [Analog Control Outputs](#analog-control-outputs)
13. [Preamp Reset / Clamp](#preamp-reset--clamp)
14. [BGO Pattern Collector Interface (DVI cable)](#bgo-pattern-collector-interface-dvi-cable)
15. [Timestamp & Sync](#timestamp--sync)
16. [Raspberry Pi / Fake-Pi Presence Detection](#raspberry-pi--fake-pi-presence-detection)
17. [Fan Control & Power Monitor](#fan-control--power-monitor)
18. [Firmware Version Registers](#firmware-version-registers)
19. [Chipscope / ILA Debug](#chipscope--ila-debug)
20. [Address Decode (LOOK_UP_TABLE1)](#address-decode-look_up_table1)

---

## Target Device & Tools

| Item | Value |
|------|-------|
| FPGA | Xilinx Spartan-6 XC6SLX9-2TQG144 |
| Tool | ISE 14.7 |
| Entity | `SlopeBoxInt` |
| Architecture | `Behav` |
| Generic | `globalPI_SPI_PORT` (1 = normal; 0 = use RPi GPIO SPI0 pins) |

---

## Source Files

| File | Entity / Purpose |
|------|-----------------|
| `SlopeBoxInt_TopLevel_RevC.vhd` | Top-level (4,654 lines) — all logic and port definitions |
| `PI_TRANSACTOR.vhd` | `SERIAL_CTL_MACH` — 24-bit SPI state machine, dual-port Pi + Collector interface (1,131 lines) |
| `I2C_template.vhd` | Generic I2C transactor, FIFO-driven, command-word based (1,056 lines) |
| `LOOK_UP_TABLE1.VHD` | Address-to-machine one-hot decoder (245 lines) |

---

## Overview / Role

The SBX Control FPGA sits on the Motherboard (Pickoff card) and interfaces between the Raspberry Pi (or Collector Box serial port) and all analog/digital hardware on the slope box assembly. Its key functions:

- **SPI host interface** — receives 24-bit read/write commands from the Raspberry Pi (SPI0) or from the Collector Box over a DVI serial link
- **Address decode & routing** — maps 7-bit addresses (0x00–0x7F) to 16 one-hot enables selecting subsidiary state machines via `LOOK_UP_TABLE1`
- **Analog control** — drives SPI and I2C interfaces to:
  - GeCenter time constant switch (TAU)
  - GeCenter gain switch (GAIN)
  - GeSide function select switch (SIDE)
  - BGO Sum gain switch (BGO_GAIN)
  - BGO Pattern selector (BGO_MUX)
  - Octet DAC for DC offsets & thresholds (OFFSETANDTHRESHOLD)
- **Slope box interface** — serial protocol (SLOPEBOXSERDATACLK / IN / OUT) for ADC data readback (8 channels) and configuration writes; periodic scan at ~1 Hz
- **I2C subsystem** — three independent I2C buses (power board, preamp, dongle EEPROM) each with their own transactor + scanner machine
- **BGO discriminator readout** — monitors 7 individual BGO disc bits + 1 BGO sum disc bit; drives two differential outputs (BGOP_A, BGOP_B) to the collector box via DDR OSERDES
- **Preamp reset clamp** — monitors PREAMPRSTMON comparator, drives GeCenterClampEn analog switch during reset events
- **Timestamp** — 48-bit counter synchronized to TTCL trigger clock from collector box; monitors clock presence and falls back to local oscillator if trigger clock disappears
- **Diagnostic** — 3 Chipscope ILA blocks + 1 VIO for in-system debugging; VIO can substitute for Pi in bench testing

---

## Clock Architecture

| Signal | Frequency | Source |
|--------|-----------|--------|
| `OSC_CLOCK` | ~50 MHz | Free-running oscillator on board |
| `BUFG_OSC_CLOCK` | ~50 MHz | `BUFG` of OSC_CLOCK |
| `TRIGCLK_PLUS/MINUS` | 50 MHz | Differential clock from Collector Box (same as TTCL) |
| `BUFG_CLK_FROM_TRIG` | 50 MHz | `IBUFGDS` + `BUFG` of differential trigger clock |
| `CLK_50MHz` | 50 MHz | PLL output (from either OSC or trigger clock via mux) |
| `CLK_100MHz` | 100 MHz | PLL output |
| `CLK_200MHz` | 200 MHz | PLL output — used only by OSERDES DDR output |
| `CLK_10MHz` | 10 MHz | PLL output |
| `CLK_4MHz` | 4 MHz | PLL output — used for slope box scan cycle timing |
| `CLK_50MHz_INV` | 50 MHz inverted | PLL output — for OSERDES |
| `BUFPLL_IOCLK` / `SERDESSTROBE` | — | BUFPLL output for OSERDES DDR |

**Clock selection:** A `BUFGMUX` selects between the oscillator and the trigger clock as PLL input. The FPGA monitors `BUFG_CLK_FROM_TRIG` with a 20-bit counter latched every millisecond; `TRIG_CLK_PRESENT` goes high when the count is non-zero. If the trigger clock disappears, the FPGA falls back to the oscillator.

**Reset:** `POWER_UP_PLL_DLY_CNT` counts 0x2FAF080 = 1 second at 50 MHz before releasing the PLL reset. Software can also trigger a soft-boot via `PULSED_CONTROL_REG`.

---

## SPI Communication Interface (PI_TRANSACTOR)

Entity: `SERIAL_CTL_MACH` (file: `PI_TRANSACTOR.vhd`)

**Transaction format (24 bits):**

| Bit(s) | Field |
|--------|-------|
| 23 | R/W: 1=read, 0=write |
| 22:16 | Address[6:0] (7-bit, 0x00–0x7F) |
| 15:0 | Data (for write; for read, returned on MISO) |
✅ verified 2026-04-24 — PI_TRANSACTOR.vhd:L6-11 (header: "Bit 23: Read(1) or write(0); Bits 22-16: Object Address[6:0]; Bits 15-00: Data to write if write transaction"; L627: `RW_FLAG <= DATA_RCV_SHIFT_REG(23)`; L629: `INT_LATCHED_ADDR <= DATA_RCV_SHIFT_REG(22 downto 16)`)

- Data is asserted on the **falling edge** of SCK; latched on the **rising edge**
- During the address phase (bits 22:16), the FPGA asserts **8-bit status data** (`FPGA_STATUS`) on MISO
- The pre-fetch latch is loaded **between the last address rising edge and the first data falling edge** — so read data must be pre-fetched or immediately available
- Transactions are always simultaneous read+write; the R/W bit only controls what data is presented on MISO during the data phase
✅ verified 2026-04-24 — PI_TRANSACTOR.vhd:L9-15 (header: "Data is asserted on the FALLING edge. Data into FPGA is latched on RISING edge. During address sequence FPGA asserts status data on MISO. During data sequence: asserts zeroes for write, requested data for read.")

**Two SPI sources:**
1. **Raspberry Pi** — GPIO 7–11 (SPI0: CE1, CE0, MISO, MOSI, SCLK); active when `PIPRESENCESENSE = '1'`
2. **Collector Box** — DVI serial link (BUFSERCOMMCLK / BUFSERDATTOPICKOFF / FPGASERDATTOCOLLECTOR); active when `TRIG_CLK_PRESENT = '1'`

A mux selects which source drives the state machine depending on trigger clock presence.

**VIO back-door:** `VIO_DEMAND_CONTROL` can substitute for the Pi in bench testing (Chipscope VIO).

---

## Address Map & Register File

The FPGA has 128 addresses (7-bit). Selected key registers:

| Address | Name | Default | Notes |
|---------|------|---------|-------|
| 0x00 | `PULSED_CONTROL_REG` | 0x0000 | Write-only; bit triggers soft-boot |
| 0x01 | `FPGA_CTL_REG` | 0x7A03 | General control; bit[8]=PREAMP_I2C_OE |
| 0x02 | `PA_RESET_COUNT` | 0x0000 | Preamp reset event count |
| 0x03 | `BGOP_MUX_CTL_REG` | 0x7100 | BGO pattern mux control; default holds at 7 |
| 0x04 | `GE_CENTER_TAU_ADDR` | — | GeCenter time constant (output group 1) |
| 0x05 | `GE_CENTER_GAIN_ADDR` | — | GeCenter gain (output group 2) |
| 0x06 | `GE_SIDE_SEL_ADDR` | — | GeSide function select (output group 3) |
| 0x07 | `BGO_SUM_GAIN_ADDR` | — | BGO sum attenuation (output group 4) |
| 0x08 | `BGO_PATTERN_SEL_ADDR` | — | BGO pattern selector (output group 5) |
| 0x09–0x0E | `GE_CENTER_OFFSET` through `BGO_DISCBIT_THRESHOLD` | — | Octet DAC control (output group 6) |
| 0x11 | `I2C_SPEED_CONTROL_REG` | 0x0909 | I2C clock divider for all three buses |
| 0x12 | `CYCLE_DELAY_REG` | 0x003D | Upper 16 bits of 4MHz cycle count between slope box reads (~1 Hz) |
| 0x14 | `MISC_CTL_STAT_REG` | 0x0000 | Misc status/control; bit[6]=PREAMP_FIFO_EMPTY; bit[2]=PREAMP_FIFO_FULL |
| 0x1C | `SLOPE_BOX_ID_REG` | 0x0000 | Slope box identifier |
| 0x1D–0x24 | `LAST_SLOPEBOX_ADC_VAL[0:7]` | 0x0000 | Last readback from slope box ADC channels 0–7 |
| 0x36 | `TRIG_CLK_MON_COUNTER` | — | 20-bit latched trigger clock frequency counter |
| 0x37 | `TRACE_FIFO_RD_PORT` | — | Read port for trace FIFO |
| 0x3D–0x44 | `BGO_DISCBIT_COUNTER[1:8]` | 0x0000 | Per-channel BGO discriminator event counters |
| 0x45–0x52 | `BGO_DAC_DEMAND[0:13]` | — | BGO DAC threshold demands (output group 13); read from DPRAM |
| 0x56 | `MISC_CTL2_REG` | 0x0075 | Miscellaneous control 2 |
| 0x7B | `DONGLE_WRITE_PORT` | — | Write port for dongle EEPROM I2C command FIFO |
| 0x7C | `PREAMP_WRITE_PORT` | — | Write port for preamp I2C command FIFO |
| 0x7D | `POWER_BOARD_WRITE_PORT` | — | Write port for power board I2C command FIFO |
| 0x7E | `CODE_DATE_REG` | 0x0311 | Firmware date code (read-only constant) |
| 0x7F | `CODE_REVISION_REG` | 0x0060 | Firmware revision (read-only; RevB = 0x0020) |

Many analog control registers (output groups 1–6, 13) are **write-only from the Pi** — their values are stored in DPRAM and read back from there.

---

## Dual-Port RAM (DPRAM)

- **Size:** 1024 × 16-bit (10-bit address, 16-bit data)
- **Usage:** Backing store for write-only configuration registers (DAC demands, analog switch settings, slope box read results, I2C read data from scanner machines)
- **Port A:** Control interface (Pi read path — prefetch reads)
- **Port B:** Scanner machines write results into DPRAM; multiple scanner machines arbitrate access via a cross-domain FIFO
- **Bank:** `DPRAM_BANK[2:0]` combined with `LATCHED_PI_ADDRESS[6:0]` forms the full 10-bit address — allows banking to access more than 128 DPRAM locations from 7-bit Pi address space

---

## I2C Subsystem (I2C_template)

Entity: instantiated 3× (power board, preamp, dongle). File: `I2C_template.vhd`

**Command word format (16-bit FIFO entry):**

| Bit | Field | Description |
|-----|-------|-------------|
| 15 | DONE | Transmit byte, then STOP |
| 14 | RPTS | Repeated Start after ACK |
| 13 | NACK | Skip ACK check (don't check SDA low in 9th clock) |
| 12 | READ | Sample SDA into read latch |
| 11 | SAVE | Save read data to DPRAM |
| 10 | EXTD | Extended (16-bit) transaction |
| 9 | LOOP | Loop back to start of FIFO |
| 8 | MACK | Master sends ACK (for read sequences) |
| 7:0 | DATA | Byte to assert on SDA |

**ACK4_CTL (bits 15:14) encoding:**
- 00 → transmit + ACK + continue
- 01 → transmit + ACK + Repeated Start + continue
- 10 → transmit + ACK + STOP
- 11 → transmit + ACK + STOP + loop (restart)

**Speed:** Controlled by `I2C_SPEED_CONTROL_REG` (default 0x0909 — separate dividers for upper/lower nibbles).

**Three I2C buses:**

| Bus | Purpose | Signals | Error flag |
|-----|---------|---------|------------|
| Power board | Power converter monitoring (temperature, fan, status) | `FPGAPOWERCONVERTERSDA/SCL` | `POWER_I2C_ACK_ERR` |
| Preamp | Detector preamp configuration | `FPGA_PREAMPSDA/SCL` | `PREAMP_I2C_ACK_ERR` |
| Dongle | Hole-ID EEPROM identification | `DONGLE_SDA/SCL` | `DONGLE_I2C_ACK_ERR` |

**Note:** Preamp "secret special mode" — the SCL line can be repurposed as a raw clock (not I2C). This is noted in the comments at line ~971 of the top-level and applies to special preamp initialization sequences.

---

## I2C Scanner Machines

Each I2C bus has an associated **scanner machine** that auto-scans at startup (and periodically), playing back a ROM-stored command sequence without Pi intervention. Three scanners: `PA` (preamp), `PWR` (power), `DONGLE` (dongle EEPROM).

- Each scanner has a 9-bit ROM address pointer (`INITIAL_PA/PWR/DONGLE_ROM_ADDRESS`)
- Results are written to DPRAM via a 26-bit FIFO (intercepting path)
- The arbiter (scan arbiter machine) sequences which scanner gets bus access at any given time
- `SCANNER_GENERAL_STATUS` (0x26) shows combined scanner running/abort/error state
- Error addresses stored in `PA/PWR/DONGLE_SCAN_ROM_ERR_ADDR` registers (0x32–0x34)
- `FPGA_STATUS[2]` = scanner running (scanner state 1–10 active)
- `FPGA_STATUS[0]` = any I2C ACK error (OR of all three)

---

## BGO Discriminator Interface

**Inputs:**
- `BGOPATTERNDISCBIT[7:1]` — 7 individual BGO disc bits
- `BGOSUMDISCBIT` — BGO sum discriminator output

**Masking:** `BGO_DISCBIT_CTL_REG` provides per-channel masks → `MASKED_BGO_DISCBIT[7:0]`

**Multiplicity:** `MASKED_BGO_MULTIPLICITY[2:0]` — 3-bit count of active BGO channels; compared against threshold to drive `BGO_ABOVE_THRESH`

**Per-channel counters:** `BGO_DISCBIT_COUNTER[1:8]` (addresses 0x3D–0x44) — 16-bit event counters per channel

**Differential outputs to collector box (DDR OSERDES):**
- `BGOP_A_PLUS/MINUS` — carries `BGO_GROUP_A_DATA`
- `BGOP_B_PLUS/MINUS` — carries `BGO_GROUP_B_DATA`
- `BGO_DISCBIT_CTL_REG(9)` selects which group gets MASKED_BGO_DISCBIT vs. Sum/Thresh/Mult
- DDR at 100 MHz with CLK_200MHz for OSERDES; BGOP_DATA0 = mux bit 0

**Analog mux control:**
- `BGOMUXSEL[2:0]` and `BGOMUXENBL` — parallel control to BGO Pattern analog mux
- `BGOP_MUX_SDI/CLK/CS` — serial control to BGO Pattern collector mux switch

---

## Slope Box Serial Interface

The slope box communicates via a 3-wire serial interface:
- `SLOPEBOXSERDATACLK` (output — clock driven by FPGA)
- `SLOPEBOXSERDATAIN` (output — data to slope box)
- `SLOPEBOXSERDATAOUT` (input — data from slope box)

**Scan cycle:**
- Periodic scan at ~1 Hz (controlled by `CYCLE_DELAY_REG` = 0x003D at upper 16 bits × 4 MHz ticks)
- Reads 8 ADC channels per scan → `LAST_SLOPEBOX_ADC_VAL[0:7]`
- Results stored via `SLOPEBOX_DOMAIN_CROSS_FIFO` to handle clock domain crossing (FPGA 50 MHz vs slope box clock)
- `SLOPE_BOX_DATA_FLAGS[7:0]` — one-hot flags indicating which ADC channel just completed a read
- `LATCHED_CHANNEL_OF_DATA[2:0]` — channel index of last completed read

**FIFO command buffer:** Pi can write commands to `SLOPEBOX_CMD_BUF_FIFO` (24-bit entries) to override or inject transactions outside the normal scan cycle.

---

## Analog Control Outputs

All analog control uses SPI (serial) or parallel outputs:

| Signal Group | Interface | Target |
|-------------|-----------|--------|
| TAU (`TAU_SCLK/SDI/CS`) | SPI | GeCenter time constant analog switch |
| GAIN (`GAIN_SDI/SCLK/RESET/CS`) | SPI | GeCenter gain control analog switch |
| SIDE (`SIDE_SDI/CLK/CS`) | SPI | GeSide function select analog switch |
| BGO_GAIN (`BGO_GAIN_SDI/SCLK/RESET/CS`) | SPI | BGO Sum attenuation control analog switch |
| OFFSET (`OFFSETANDTHRESHOLDSDI/SCK/CS/CLR`) | SPI | Octet DAC for DC offsets and discriminator thresholds |
| `BGOMUXSEL[2:0]`, `BGOMUXENBL` | Parallel | BGO Pattern analog mux |
| `BGOP_MUX_SDI/CLK/CS` | SPI | BGO Pattern collector mux switch |

Internal versions of these signals (prefixed `x`) are used for ILA monitoring; the external pins are driven from these.

---

## Preamp Reset / Clamp

- `PREAMPRSTMON` — input from comparator monitoring the GeCenter signal; goes high during a spontaneous preamp reset event
- `GeCenterClampEn` — output; activates `TMUX6119DCNR` analog switch to clamp the GeCenter line during preamp reset
- `PA_RESET_COUNT` (0x02) — 16-bit counter of preamp reset events
- `PARST_SWITCH_COUNT` (0x38) — default 0x1388 (5000 counts = 100 µs at 50 MHz) — duration of clamp activation
- `PARST_CLAMP_DC_VAL` (0x10) — DC value to assert during clamp

---

## BGO Pattern Collector Interface (DVI cable)

A DVI cable carries the BGO Pattern data to the collector module:
- **Primary:** `BGOP_A/B_PLUS/MINUS` — DDR differential OSERDES outputs at 100 Mbit/s
- **Serial config link:** `BUFSERDATTOPICKOFF` / `FPGASERDATTOCOLLECTOR` / `BUFSERCOMMCLK` — bidirectional serial link that **replaces** the Raspberry Pi SPI interface when the collector is driving the pickoff (when `TRIG_CLK_PRESENT = '1'`)
- `BUFSYNCFROMCOLLECTOR` — timing synchronization (Imperative Sync) from collector box
- `FPGACOLLECTOR_SPI_CE` — chip enable from collector for serial transactions

---

## Timestamp & Sync

- `TIMESTAMP[47:0]` — 48-bit counter running at trigger clock rate (50 MHz = 20 ns per tick)
- `Imperative_Sync` — from `BUFSYNCFROMCOLLECTOR`; resets timestamp counter for global synchronization
- `Internal_Sync` — software-triggered sync event
- `SYNC_CHK_STATE` FSM (4 states: LOCAL_CLOCK, WAIT_FIRST_EDGE, MONITOR, ERROR) — validates sync period consistency; `LATCHED_SYNC_ERR` / `SYNC_ERROR` flags
- `SYNC_PERIOD_COUNT[7:0]` — measured sync interval in clock ticks
- `TIMESTAMP_LOW` (0x3C) — lower 16 bits of timestamp (readable by Pi)

---

## Raspberry Pi / Fake-Pi Presence Detection

`PIPRESENCESENSE` — read from a voltage divider on the Pi power rail:
- `'1'` = Pi present → FPGA tri-states Pi-conflicting pins
- `'0'` = Pi absent (or "fake Pi" board installed)

When `PIPRESENCESENSE = '0'`, FPGA can drive:
- `DONGLE_TP` (IOBUF, T=PIPRESENCESENSE)
- `FAKEPI_GREEN_LED`, `FAKEPI_RED_LED`, `DONGLE_LED` (OBUFT, T=PIPRESENCESENSE)
- `DONGLE_SDA`, `DONGLE_SCL`, `DONGLE_LOOP_OUT`

The "Fake Pi" board + "Dongle" board combination allows bench testing without a Raspberry Pi.

---

## Fan Control & Power Monitor

The power board (I2C bus) provides:
- `PwrLocalTempMSB/LSB`, `PwrPickoffTempMSB/LSB`, `PwrTempExtended` — temperature readings
- `PwrStatus` — power converter status
- `PwrFanSpeedRaw` — raw fan tachometer reading
- `FanControlStat1/2` — fan control state machine status
- These are all written to DPRAM by the power board scanner machine and readable by the Pi via DPRAM bank

---

## Firmware Version Registers

| Register | Address | Value | Meaning |
|---------|---------|-------|---------|
| `CODE_DATE_REG` | 0x7E | 0x0311 | Firmware date code (03=March, 11=11th? 2021?) |
| `CODE_REVISION_REG` | 0x7F | 0x0060 | Revision; RevB started at 0x0020; current 0x0060 = RevC |
✅ verified 2026-04-24 — SlopeBoxInt_TopLevel_RevC.vhd:L627-628 (`CODE_DATE_REG := X"0311"`; `CODE_REVISION_REG := X"0060"` — comment: "revision B started with 0x0020"); L350-351 (addr constants 0x7E/0x7F); L2090-2091 (prefetch returns constants for these addresses)

---

## Chipscope / ILA Debug

Three Chipscope ILA blocks (`CONTROL0/1/2`, `TRIG0[33:0]`):
- `ILA_SEL_CODE[4:1]` — 4-bit code selecting which signals to route to ILA trigger inputs
- `WIDE_ILA_CLK` — separate clock for wide ILA
- `DIAG_*` signals — internal diagnostic copies of serial signals, state machines, and error flags connected to ILA trigger inputs

VIO block provides `VIO_DEMAND_CONTROL` to substitute for Pi in bench mode.

---

## Address Decode (LOOK_UP_TABLE1)

Entity: `Look_UP_TABLE1` (file: `LOOK_UP_TABLE1.VHD`)

Maps 7-bit Pi address → 16-bit one-hot `MACH_ENABLE[15:0]`. Each bit enables one subsidiary state machine. Address 0x00–0x7F covers all registers; unrecognized addresses produce `MACH_ENABLE = 0x0000` (no machine responds).

`DIAG_DEV_ADDR[3:0]` — diagnostic 4-bit device code output for ILA monitoring.

---

## See Also

- [sbx.md](sbx.md) — SBX hardware overview: slope box role, HV generation, BGO pattern/sum, GS_ID dongle, I2C opcode format, SVN location
- [slope_box_interface.md](slope_box_interface.md) — SBX hardware overview, connector pinout, analog chain
- [fpga.md](fpga.md) — FPGA firmware index across all DGS boards
- [hardware_architecture.md](hardware_architecture.md) — system-level hardware map
- [deep_fpga_DIG.md](deep_fpga_DIG.md) — Digitizer FPGA (comparison: different role, Spartan-3)
- [deep_fpga_MTRG_MAIN.md](deep_fpga_MTRG_MAIN.md) — Master Trigger FPGA
