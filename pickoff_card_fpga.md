# Pickoff Card FPGA — SBX Extension Revision C

Stability: C3 - Structural / stable

**Source:** `DGS_tools_pack/FPGA/collectorBox/PickoffCard_SBX_Extension/Revision_C/Source/`  
**Top-level file:** `SlopeboxInt_TopLevel_RevC.vhd`  
**Target device:** Xilinx Spartan-6 `XC6SLX9-2TQG144` ✅ verified 2026-04-25 — `SlopeboxInt_TopLevel_RevC.vhd:L12`  
**Tool:** Xilinx ISE 14.7 ✅ verified 2026-04-25 — `SlopeboxInt_TopLevel_RevC.vhd:L13`  
**Author:** John T. Anderson (ANL), created 2020-10-05 ✅ verified 2026-04-25 — `SlopeboxInt_TopLevel_RevC.vhd:L8`  
**Firmware version:** Code date `0x0914` (Sept 2014?), revision `0x0182` ✅ verified 2026-04-25 — `SlopeboxInt_TopLevel_RevC.vhd:L619-620`  
**See also:** [`sbx.md`](sbx.md), [`slope_box_interface.md`](slope_box_interface.md), [`sbxPi_ioc.md`](sbxPi_ioc.md)

---

## Overview

The **Pickoff Card FPGA** (SBX Extension, Revision C) is the central digital controller on the Slope Box Extension (SBX) board. It sits between the Raspberry Pi (or Collector Box) and all analog/digital hardware on the SBX/slope box. Revision C is the current production version.

Key responsibilities:
- Route SPI transactions from Pi or Collector Box to analog switches, DACs, and slope box serial bus
- Control GeCenter gain (ADGS5412 analog switch) and BGO sum gain (ADGS5412)
- Control GeSide function selection, BGO pattern mux, GeCenter tau (time constant) switch
- Drive slope box serial data bus (Ge/BGO gain + offset programming)
- Monitor BGO discriminator bits (7 pattern + 1 sum) and serialize to Collector Box at 400 MHz
- Manage preamp reset clamp (GeCenterClampEn, TMUX6119DCNR analog switch)
- I2C scanning of preamp, power board, and dongle via dedicated FIFOs and state machines
- Monitor trigger clock from Collector Box; can fall back to local oscillator
- Optional "fake Pi" mode when a dongle board replaces the Raspberry Pi

---

## Hardware Interfaces

### Clock Architecture
- **Primary:** Differential trigger clock (`TRIGCLK_PLUS/MINUS`) from Collector Box — used when `UseTrigClock` (FPGA_CTL_REG[10]) = 1
- **Fallback:** Local free-running `OSC_CLOCK` oscillator — used at power-up and when trig clock absent
- **PLL outputs:** 50 MHz, 100 MHz, 200 MHz (OSERDES only), 50 MHz inverted, 10 MHz, 4 MHz ✅ verified 2026-04-25 — `SlopeboxInt_TopLevel_RevC.vhd:L376-383`
- Clock switchover triggers a soft reboot to allow PLL to relock ✅ verified 2026-04-25 — `SlopeboxInt_TopLevel_RevC.vhd:L395-410`

### Raspberry Pi / Collector Box Interface
- **SPI port select:** Generic `globalPI_SPI_PORT` (normally 1 = SPI1; 0 = SPI0 via GPIO) ✅ verified 2026-04-25 — `SlopeboxInt_TopLevel_RevC.vhd:L30`
- **DEVSEL lines:** `PI_GPIO_13` (DEVSEL0), `PI_GPIO_23` (DEVSEL1), `PI_GPIO_24` (DEVSEL2), `PI_GPIO_25` (DEVSEL3), `PI_GPIO_26` (DEVSEL4) — used to select target device in collector mode ✅ verified 2026-04-25 — `SlopeboxInt_TopLevel_RevC.vhd:L148-162`
- **Pi presence detect:** `PI_GPIO_4` / `PIPRESENCESENSE` — resistor divider; high = Pi present, low = fake Pi dongle ✅ verified 2026-04-25 — `SlopeboxInt_TopLevel_RevC.vhd:L129,L166`
- **Fake Pi mode:** When `PIPRESENCESENSE` = low, FPGA drives `DONGLE_TP`, `FAKEPI_GREEN_LED`, `FAKEPI_RED_LED`, `DONGLE_SDA`, `DONGLE_SCL`, `DONGLE_LED` ✅ verified 2026-04-25 — `SlopeboxInt_TopLevel_RevC.vhd:L127-167`

### BGO Discriminator Output to Collector Box
- 7 BGO pattern discriminator bits + 1 BGO sum bit, serialized differentially to collector ✅ verified 2026-04-25 — `SlopeboxInt_TopLevel_RevC.vhd:L2113-2120`
- Uses **8:1 cascaded OSERDES** pair → 400 MHz serial stream on `BGOP_A` and `BGOP_B` differential pairs ✅ verified 2026-04-25 — `SlopeboxInt_TopLevel_RevC.vhd:L2117-2122`
- Bit 0 of the 8-bit frame = sync marker (constant '1', or BGOsum discriminator when sync locked) ✅ verified 2026-04-25 — `SlopeboxInt_TopLevel_RevC.vhd:L2128-2132`
- BGO mask register (`BGO_DISCBIT_CTL_REG[7:0]`) gates each bit independently ✅ verified 2026-04-25 — `SlopeboxInt_TopLevel_RevC.vhd:L2069`

### Preamp Reset Clamp
- `GeCenterClampEn` output drives `TMUX6119DCNR` analog switch to clamp GeCenter line during preamp reset ✅ verified 2026-04-25 — `SlopeboxInt_TopLevel_RevC.vhd:L102`
- **Clamp duration:** `PARST_SWITCH_COUNT[14:0]` × 2 × 10 ns at 100 MHz; default `PARST_SW_DELAY_INIT` from register (bit 15 = enable) ✅ verified 2026-04-25 — `SlopeboxInt_TopLevel_RevC.vhd:L2088-2089`

---

## Register Map (SPI Address Space, 7-bit address)

All registers are 16-bit wide. Read/write controlled by SPI bit 23 (0=write, 1=read).

| Addr (hex) | Addr (bin) | Name | R/W | Default | Notes |
|-----------|-----------|------|-----|---------|-------|
| 0x00 | 0000000 | PULSED_CONTROL_REG | W | 0x0000 | Auto-clears; one-shot control bits |
| 0x01 | 0000001 | FPGA_CTL_REG | R/W | 0x7A03 | Main control register |
| 0x02 | 0000010 | PA_RESET_COUNT | R | 0x0000 | Preamp reset event counter |
| 0x03 | 0000011 | BGOP_MUX_CTL_REG | R/W | — | BGO pattern mux control |
| 0x04 | 0000100 | GE_CENTER_TAU_ADDR | W | — | GeCenter time constant (analog switch) |
| 0x05 | 0000101 | GE_CENTER_GAIN_ADDR | W | — | GeCenter gain (ADGS5412) |
| 0x06 | 0000110 | GE_SIDE_SEL_ADDR | W | — | GeSide function select |
| 0x07 | 0000111 | BGO_SUM_GAIN_ADDR | W | — | BGO sum gain (ADGS5412) |
| 0x08 | 0001000 | BGO_PATTERN_SEL_ADDR | W | — | BGO pattern mux select |
| 0x09 | 0001001 | GE_CENTER_OFFSET_ADDR | W | — | GeCenter DC offset (DAC) |
| 0x0A | 0001010 | GE_SIDE_OFFSET_ADDR | W | — | GeSide offset (DAC) |
| 0x0B | 0001011 | BGO_SUM_OFFSET_ADDR | W | — | BGO sum offset (DAC) |
| 0x0C | 0001100 | BGO_PATTERN_OFFSET_ADDR | W | — | BGO pattern offset (DAC) |
| 0x0D | 0001101 | GE_RESET_THRESHOLD_ADDR | W | — | GeCenter reset threshold (DAC) |
| 0x0E | 0001110 | BGO_DISCBIT_THRESHOLD_ADDR | W | — | BGO discriminator bit threshold (DAC) |
| 0x0F | 0001111 | GE_SIDEB_OFFSET_ADDR | W | — | GeSide-B offset (DAC) |
| 0x10 | 0010000 | PARST_CLAMP_DC_VAL_ADDR | W | — | Preamp reset clamp DC level |
| 0x11 | 0010001 | I2C_SPEED_CONTROL_REG | R/W | 0x0909 | I2C speed: [15:8]=power/dongle delay, [7:0]=preamp delay |
| 0x12 | 0010010 | CYCLE_DELAY_REG | R/W | 0x003D | Cycle delay (61 decimal = 61 µs?) |
| 0x13 | 0010011 | BGO_DISCBIT_CTL_REG | R/W | 0x0000 | BGO bit mask [7:0], collector select [9:8], ILA sel [11:10] |
| 0x14 | 0010100 | MISC_CTL_STAT_REG | R/W | 0x0000 | Status bits [7:0] (read), control bits [14:8] (write) |
| 0x15 | 0010101 | PA_RST_WIDTH_REG | R/W | 0x0000 | Preamp reset pulse width |
| 0x16 | 0010110 | PA_RST_MAX_REG | R/W | 0x4C4B | Max preamp reset count |
| 0x17 | 0010111 | SCAN_ROM_MATCH_REG | R/W | 0x0000 | ROM scan match value [8:0] |
| 0x18 | 0011000 | ILA_CONTROL_REG | R/W | 0x0000 | ILA debug control |
| 0x19 | 0011001 | INITIAL_PA_ROM_ADDRESS | R/W | — | Preamp scanner start ROM address [8:0] |
| 0x1C | 0011100 | SLOPE_BOX_ID_REG | R/W | 0x0000 | [7:0]=SlopeBox ID (R), [9:8]=scan ctrl, [14]=data delay, [15]=clk delay |
| 0x1D | 0011101 | LAST_SLOPEBOX_ADC_VAL0 | R | — | Last slope box ADC scan value, ch 0 |
| 0x1E | 0011110 | LAST_SLOPEBOX_ADC_VAL1 | R | — | Last slope box ADC scan value, ch 1 |
| 0x1F | 0011111 | LAST_SLOPEBOX_ADC_VAL2 | R | — | Last slope box ADC scan value, ch 2 |
| 0x20 | 0100000 | LAST_SLOPEBOX_ADC_VAL3 | R | — | Last slope box ADC scan value, ch 3 |
| 0x21 | 0100001 | LAST_SLOPEBOX_ADC_VAL4 | R | — | Last slope box ADC scan value, ch 4 |
| 0x22 | 0100010 | LAST_SLOPEBOX_ADC_VAL5 | R | — | Last slope box ADC scan value, ch 5 |
| 0x23 | 0100011 | LAST_SLOPEBOX_ADC_VAL6 | R | — | Last slope box ADC scan value, ch 6 |
| 0x24 | 0100100 | LAST_SLOPEBOX_ADC_VAL7 | R | — | Last slope box ADC scan value, ch 7 |
| 0x26 | 0100110 | SCANNER_GENERAL_STATUS | R | — | General I2C scanner status |
| 0x27 | 0100111 | INITIAL_PWR_ROM_ADDRESS | R/W | — | Power board scanner start address [8:0] |
| 0x28 | 0101000 | INITIAL_DONGLE_ROM_ADDRESS | R/W | — | Dongle scanner start address [8:0] |
| 0x2A | 0101010 | PREAMP_GO_COUNT | R | 0x0000 | Preamp I2C transaction count |
| 0x2B | 0101011 | PWR_GO_COUNT | R | 0x0000 | Power board I2C transaction count |
| 0x2C | 0101100 | DONGLE_GO_COUNT | R | 0x0000 | Dongle I2C transaction count |
| 0x2E | 0101110 | PREAMP_ACK_ERR_COUNT | R | — | Preamp I2C ACK error count |
| 0x2F | 0101111 | POWER_ACK_ERR_COUNT | R | — | Power board ACK error count |
| 0x30 | 0110000 | DONGLE_ACK_ERR_COUNT | R | — | Dongle ACK error count |
| 0x32 | 0110010 | PA_SCAN_ROM_ERR_ADDR | R | — | Last preamp ROM scan error address |
| 0x33 | 0110011 | PWR_SCAN_ROM_ERR_ADDR | R | — | Last power ROM scan error address |
| 0x34 | 0110100 | DONGLE_SCAN_ROM_ERR_ADDR | R | — | Last dongle ROM scan error address |
| 0x36 | 0110110 | TRIG_CLK_MON_COUNTER | R | — | Trigger clock monitor counter |
| 0x37 | 0110111 | TRACE_FIFO_RD_PORT | R | — | Trace FIFO read port |
| 0x38 | 0111000 | PARST_SWITCH_COUNT | R/W | — | [15]=clamp enable, [14:0]=clamp duration (×2×10 ns) |
| 0x3C | 0111100 | TIMESTAMP_LOW_REG | R | — | Low 16 bits of trigger timestamp |
| 0x3D | 0111101 | BGO_DISCBIT_COUNTER1 | R | — | BGO discbit 1 hit counter |
| 0x3E | 0111110 | BGO_DISCBIT_COUNTER2 | R | — | BGO discbit 2 hit counter |
| 0x3F | 0111111 | BGO_DISCBIT_COUNTER3 | R | — | BGO discbit 3 hit counter |
| 0x40 | 1000000 | BGO_DISCBIT_COUNTER4 | R | — | BGO discbit 4 hit counter |
| 0x41 | 1000001 | BGO_DISCBIT_COUNTER5 | R | — | BGO discbit 5 hit counter |
| 0x42 | 1000010 | BGO_DISCBIT_COUNTER6 | R | — | BGO discbit 6 hit counter |
| 0x43 | 1000011 | BGO_DISCBIT_COUNTER7 | R | — | BGO discbit 7 hit counter |
| 0x44 | 1000100 | BGO_DISCBIT_COUNTER8 | R | — | BGO discbit 8 (sum) hit counter |
| 0x45–0x4D | 1000101–1001101 | BGO_DAC_DEMAND0–10 | W | — | BGO discriminator DAC demand values (per BGO detector, 0–13) |
| 0x53 | 1010011 | SLOPE_BOX_HV_CTL | W | — | Slope box HV control |
| 0x54 | 1010100 | SLOPE_BOX_GEHV_DEMAND | W | — | Ge HV demand |
| 0x55 | 1010101 | TRACE_FIFO_DATA_COUNT | R | — | Trace FIFO word count |
| 0x56 | 1010110 | MISC_CTL2_REG | R/W | 0x0075 | [2:0]=PreampQIRate, [3]=Enable_Preamp_QI, [4]=FakePiGreenLED, [5]=FakePiRedLED, [6]=DongleLED, [12]=ENABLE_WATCHDOG, [15]=DisablePARSTRecognition |
| 0x57 | 1010111 | CenterFETBiasHi | R | — | GeCenter FET bias voltage MSB |
| 0x58 | 1011000 | CenterFETBiasLo | R | — | GeCenter FET bias voltage LSB |
| 0x59 | 1011001 | SideFETBiasHi | R | — | GeSide FET bias voltage MSB |
| 0x5A | 1011010 | SideFETBiasLo | R | — | GeSide FET bias voltage LSB |
| 0x5B | 1011011 | CenterFETcurrentHi | R | — | GeCenter FET current MSB |
| 0x5C | 1011100 | CenterFETcurrentLo | R | — | GeCenter FET current LSB |
| 0x5D | 1011101 | SideFETcurrentHi | R | — | GeSide FET current MSB |
| 0x5E | 1011110 | SideFETcurrentLo | R | — | GeSide FET current LSB |
| 0x5F | 1011111 | CenterFETVDSHi | R | — | GeCenter FET VDS voltage MSB |
| 0x60 | 1100000 | CenterFETVDSLo | R | — | GeCenter FET VDS voltage LSB |
| 0x61 | 1100001 | SideB_OffsetHi | R | — | GeSide-B offset MSB |
| 0x62 | 1100010 | SideB_OffsetLo | R | — | GeSide-B offset LSB |
| 0x63 | 1100011 | SideFETVDSHi | R | — | GeSide FET VDS voltage MSB |
| 0x64 | 1100100 | SideFETVDSLo | R | — | GeSide FET VDS voltage LSB |
| 0x65 | 1100101 | SideA_OffsetHi | R | — | GeSide-A offset MSB |
| 0x66 | 1100110 | SideA_OffsetLo | R | — | GeSide-A offset LSB |
| 0x67 | 1100111 | ADC1X_OffsetA_GainHi | R | — | 1× ADC channel A offset/gain MSB |
| 0x68 | 1101000 | ADC1X_OffsetA_GainLo | R | — | 1× ADC channel A offset/gain LSB |
| 0x69 | 1101001 | ADC1X_OffsetB_GainHi | R | — | 1× ADC channel B offset/gain MSB |
| 0x6A | 1101010 | ADC1X_OffsetB_GainLo | R | — | 1× ADC channel B offset/gain LSB |
| 0x6B | 1101011 | ADC4X_OffsetA_GainHi | R | — | 4× ADC channel A offset/gain MSB |
| 0x6C | 1101100 | ADC4X_OffsetA_GainLo | R | — | 4× ADC channel A offset/gain LSB |
| 0x6D | 1101101 | ADC4X_OffsetB_GainHi | R | — | 4× ADC channel B offset/gain MSB |
| 0x6E | 1101110 | ADC4X_OffsetB_GainLo | R | — | 4× ADC channel B offset/gain LSB |
| 0x6F | 1101111 | TempSensorTemp | R | — | On-board temperature sensor reading |
| 0x70 | 1110000 | HumidityRawData | R | — | On-board humidity sensor raw data |
| 0x71 | 1110001 | PreampPCBTemp | R | — | Preamp PCB temperature (from preamp I2C) |
| 0x72 | 1110010 | PreampSecretValue | R | — | Preamp ID/secret value |
| 0x73 | 1110011 | PwrLocalTempMSB | R | — | Power board local temperature MSB |
| 0x74 | 1110100 | PwrPickoffTempMSB | R | — | Power board pickoff temperature MSB |
| 0x75 | 1110101 | PwrTempExtended | R | — | Power board temperature extended data |
| 0x76 | 1110110 | PwrStatus | R | — | Power board status register |
| 0x77 | 1110111 | PwrFanSpeedRaw | R | — | Power board fan speed raw count |
| 0x78 | 1111000 | FanControlStat1 | R | — | Fan control status 1 |
| 0x79 | 1111001 | FanControlStat2 | R | — | Fan control status 2 |
| 0x7B | 1111011 | DONGLE_WRITE_PORT | W | — | Dongle I2C FIFO write port |
| 0x7C | 1111100 | PREAMP_WRITE_PORT | W | — | Preamp I2C FIFO write port |
| 0x7D | 1111101 | POWER_BOARD_WRITE_PORT | W | — | Power board I2C FIFO write port |
| 0x7E | 1111110 | CODE_DATE_REG | R | 0x0914 | Firmware code date |
| 0x7F | 1111111 | CODE_REVISION_REG | R | 0x0182 | Firmware code revision; writing sets DPRAM bank [2:0] |

All addresses verified 2026-04-25 — `SlopeboxInt_TopLevel_RevC.vhd:L228-345` (constants block).

---

## Key Control Register Bit Maps

### PULSED_CONTROL_REG (0x00) — Write-only, auto-clears
| Bit | Name | Action |
|-----|------|--------|
| 15 | SoftBootComms | Soft reboot communications subsystem |
| 13 | ClearSyncError | Clear sync error flag |
| 12 | ResetAllI2C | Reset all I2C machines |
| 11 | ResetDongleScanner | Reset dongle scanner |
| 10 | ResetTraceFIFO | Reset trace FIFO |
| 9 | ResetPreampScanner | Reset preamp I2C scanner |
| 8 | ResetCounters | Reset all counters |
| 7 | DONGLE_FIFO_DATA_READY | Signal dongle FIFO has data |
| 6 | PA_FIFO_DATA_READY | Signal preamp FIFO has data |
| 5 | PWR_FIFO_DATA_READY | Signal power board FIFO has data |
| 4 | MeasPARSTGo | Trigger preamp reset measurement |
| 3 | ResetPowerScanner | Reset power board I2C scanner |
| 2 | SoftBootFPGA | Soft reboot FPGA |
| 1 | ResetPickoffDAC | Reset pickoff DAC |
| 0 | ResetSlopeBoxScan | Reset slope box scanner |

✅ verified 2026-04-25 — `SlopeboxInt_TopLevel_RevC.vhd:L2032-2047`

### FPGA_CTL_REG (0x01) — Default 0x7A03
| Bits | Name | Notes |
|------|------|-------|
| 15 | int_PowerConverterOE | Power converter output enable |
| 14 | I2CTransactorReset | Reset I2C transactor |
| 13 | DongleI2CFifoReset | Reset dongle I2C FIFO |
| 12 | PreampI2CFifoReset | Reset preamp I2C FIFO |
| 11 | PowerBoardI2CFifoReset | Reset power board I2C FIFO |
| 10 | UseTrigClock | 1=use trigger clock from collector; 0=use local oscillator |
| 9 | ResetAllScanMachines | Reset all I2C scan state machines |
| 8 | int_Preamp_I2C_OE | Preamp I2C output enable |
| 7 | int_PowerConverterEN | Power converter enable |
| 6:5 | PARST_EdgeSel | Preamp reset edge select |
| 4 | PARSTContinuousMode | Continuous preamp reset mode |
| 3 | PARSTCounterMode | Preamp reset counter mode |
| 2 | BGOCounterMode | BGO discriminator counter mode |
| 1 | int_BGOsum_GainReset | BGO sum gain reset |
| 0 | int_GeC_GainReset | GeCenter gain reset |

✅ verified 2026-04-25 — `SlopeboxInt_TopLevel_RevC.vhd:L2048-2063`

### MISC_CTL_STAT_REG (0x14)
| Bits | Direction | Name |
|------|-----------|------|
| 0 | R | DONGLE_COMMAND_FIFO_FULL |
| 1 | R | PWR_COMMAND_FIFO_FULL |
| 2 | R | PREAMP_COMMAND_FIFO_FULL |
| 3 | R | xDONGLE_LOOP_OUT |
| 4 | R | PIPRESENCESENSE (Pi present?) |
| 5 | R | DONGLE_COMMAND_FIFO_EMPTY |
| 6 | R | PREAMP_COMMAND_FIFO_EMPTY |
| 7 | R | PWR_COMMAND_FIFO_EMPTY |
| 11:8 | R/W | Digital_side_marker_select |
| 14:12 | R/W | BGO_multiplicity_threshold |

✅ verified 2026-04-25 — `SlopeboxInt_TopLevel_RevC.vhd:L2069-2078`

---

## Analog Peripheral Controllers

| Component | VHDL Module | Description |
|-----------|-------------|-------------|
| ADGS5412 analog switch | `ADGS5412_CONTROLLER.vhd` | Controls GeCenter gain and BGO sum gain; SPI at 1/2 × 50 MHz |
| LTC1660 DAC | `LTC1660_CONTROLLER.vhd` | DC offset and threshold DAC; SPI at 12.5 MHz |
| Slope box serial | `SLOPEBOX_SM1_CONTROLLER.vhd` | Serial bus to slope box module (Ge/BGO gain, offset) |
| BGO pattern mux | Direct FPGA GPIO | 3-bit MUX select (`BGOMUXSEL2:0`) + enable |
| Preamp I2C | `PI_TRANSACTOR.vhd` | I2C transactor via dedicated FIFO + scanner ROM |
| Power board I2C | `PI_TRANSACTOR.vhd` | I2C transactor via dedicated FIFO + scanner ROM |
| Dongle I2C | `PI_TRANSACTOR.vhd` | I2C transactor via dedicated FIFO + scanner ROM |

✅ verified 2026-04-25 — `SlopeBoxPickoffPkg.vhd:L26-96` (component declarations)

---

## Notes

- Default `FPGA_CTL_REG` = `0x7A03` = `0111_1010_0000_0011b` — at startup: `UseTrigClock=1`, `ResetAllScanMachines=1`, `Preamp_I2C_OE=1`, `PowerConverterEN=1`, `PowerConverterOE=1`, `BGOsum_GainReset=0`, `GeC_GainReset=1` ✅ verified 2026-04-25 — signal default, `SlopeboxInt_TopLevel_RevC.vhd:L503`
- Default `MISC_CTL2_REG` = `0x0075` = watchdog enabled (`[12]=0`), `PreampQIRate[2:0]=5`, `Enable_Preamp_QI=1`, `FakePiGreenLED=1`, `FakePiRedLED=1` ✅ verified 2026-04-25 — `SlopeboxInt_TopLevel_RevC.vhd:L579`
- Rev C was supposed to be the same as Rev B at time of design (per `ReadMe.txt`)
- 3 startup ROMs: preamp (`I2C_STARTUP_ROM` / `INITIAL_PA_ROM_ADDRESS`), power board, and dongle — each scanned independently after power-on via scanner state machines
