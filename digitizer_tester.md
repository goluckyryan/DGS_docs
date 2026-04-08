# Digitizer Tester

**Purpose:** Test module for DGS/GRETINA digitizer boards. Generates arbitrary analog waveforms to exercise digitizers without real detector signals.  
**Source:** `DGS_SVN/dgs/Digitizer_Tester/`  
**FPGA:** Xilinx Virtex-4 `XC4VLX40-10FFG1148C` (same family as MTRG/RTRG) ✅ verified 2026-04-07 — `Dig_Tester.ucf:L1` (`# FPGA = XC4VLX40-10FFG1148C`)

---

## Function

- Generates arbitrary test waveforms via dual **16-bit DACs** (AD9747, up to 200 MHz) ✅ verified 2026-04-07 — `DAC_SPI.vhd:L1` (AD9747); `Dig_Tester_pkg.vhd` (clock_freq_sel: 00=50MHz, 01=100MHz, 11=200MHz)
- Drives waveforms to up to 10 output channels via an **analog switch matrix**
- Connects to TTCL (DGS/GRETINA trigger system) via **RJ45** — can sync to master timestamp or run asynchronously
- Provides 2 **NIM outputs** for triggering/synchronization
- VME register interface for waveform programming and control

---

## Hardware

### DACs
- **2× AD9747 dual 16-bit DAC** ✅ verified 2026-04-07 — `DAC_SPI.vhd` header
- Clock selectable: 50 / 100 / 200 MHz (register `clock_freq_sel[1:0]`: `00`=50, `01`=100, `10`=50, `11`=200) ✅ verified 2026-04-07 — `Dig_Tester_pkg.vhd`
- ⚠️ Code note: "previous comments say wavx_cs_trigx outputs don't route at 200 MHz" — re-enabled 2019-08-14 for noise logic checking. ✅ verified 2026-04-08 — `Dig_Tester.vhd:L450-451` (comment + re-enable note)
- Waveform memory is 18-bit wide internally; output truncated to 16-bit for DAC (`Waveform_Reader.vhd: dac_data_out(15:0)`) ✅ verified 2026-04-07 — `Waveform_Reader.vhd:L4-5`
- SPI clock max 40 MHz per AD9747 datasheet ✅ verified 2026-04-07 — `DAC_SPI.vhd` comment
- Clock sources selectable: external, 50 MHz local oscillator, or SYS_CLK × 20/7
- DAC outputs driven via SPI interface from FPGA (`DAC_SPI.vhd`)

### Analog Switch Matrix (`Analog_Switch_MUX.vhd`)
- Routes any DAC output (DAC0, DAC1, DAC0+DAC1 sum, DAC0−DAC1 diff) to any of 10 output channels
- MUX controlled via I2C (`MUX_SCL`, `MUX_SDA`)
- Signal amplitudes and connection scheme compatible with LBL digitizer module

### SERDES (TTCL link)
- RJ45 connector — **not Ethernet**; same as digitizer, Cat5e to trigger module
- `SERDES_RX_Mach.vhd` — receives TTCL from Master Trigger or Router
- `SERDES_TX_MACH.vhd` — transmits data back to trigger
- Can lock to master timestamp or run asynchronously
- SERDES lock/lost-lock flags, sync timestamp capture

### NIM I/O
- 2× NIM outputs (`NIM_OUT[1:0]`) — triggering and synchronization

### Key VHDL Modules
| File | Function |
|------|---------|
| `Dig_Tester.vhd` | Top-level entity |
| `Dig_Tester_pkg.vhd` | Package: all component declarations + signal definitions |
| `Waveform.vhd` | Waveform memory + sequencer |
| `Waveform_Reader.vhd` | Reads waveform data from VME-loaded buffer |
| `DAC_SPI.vhd` | SPI interface to DAC chips |
| `Analog_Switch_MUX.vhd` | Analog switch matrix control |
| `SERDES_TX_MACH.vhd` | TTCL transmit state machine |
| `Timestamp_Generator.vhd` | Local timestamp counter |
| `TS_Mask_Trigger.vhd` | Timestamp-based trigger generation |
| `Phase_Shifter.vhd` | Clock phase shifting |
| `Pulse_Stretcher.vhd` | NIM pulse width control |
| `DCM_Controller.vhd` | Digital Clock Manager control |
| `Registers.vhd` | VME register map |
| `Local_Vme.vhd` | Local VME bus interface |

---

## Operation Modes

1. **Synchronous to TTCL** — locks to Master Trigger clock; generates waveforms at timestamp-specific times
2. **Asynchronous** — runs independently from local oscillator; useful for standalone testing

---

## Connection to DGS System

- Connects to **any link (A–H, L, R, U)** of Master Trigger or Router
- Typically used during commissioning and debugging to:
  - Verify cabling between trigger and digitizers
  - Test digitizer firmware response to known waveforms
  - Exercise the DAQ data path end-to-end without beam

---

## Related Files
- `connectors.md` — RJ45 pinout (same connector as digitizer)
- `connectors.md` — trigger module links the Digitizer Tester connects to
- `DIG_firmware_expert.md` — digitizer firmware being tested
- `DGS_SVN.md` — `Digitizer_Tester/` in SVN tree

---

*Created: 2026-04-05 (from SVN Digitizer_Tester VHDL source)*
