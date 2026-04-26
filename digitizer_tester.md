# Digitizer Tester

Stability: C3 - Structural / stable

**Purpose:** Test module for DGS/GRETINA digitizer boards. Generates arbitrary analog waveforms to exercise digitizers without real detector signals.  
**Source:** `DGS_SVN/dgs/Digitizer_Tester/`  
**FPGA:** Xilinx Virtex-4 `XC4VLX40-10FFG1148C` (same family as MTRG/RTRG) ✅ verified 2026-04-07 — `Dig_Tester.ucf:L1` (`# FPGA = XC4VLX40-10FFG1148C`)

---

## Function

- Generates arbitrary test waveforms via dual **16-bit DACs** (AD9747, up to 200 MHz) ✅ verified 2026-04-07 — `DAC_SPI.vhd:L1` (AD9747); `Dig_Tester_pkg.vhd` (clock_freq_sel: 00=50MHz, 01=100MHz, 11=200MHz)
- Drives waveforms to up to 10 output channels via an **analog switch matrix** (8×10 ADG2108 chips; each channel independently selectable from: Digital discriminator, AUX0, AUX1, DAC0+, DAC0−, DAC1+, DAC1−) ✅ verified 2026-04-11 — `Analog_Switch_MUX.vhd:L3-4` ("8x10 ADG2108") + L283-297 (per-channel source select encoding)
- Connects to TTCL (DGS/GRETINA trigger system) via **RJ45** — can sync to master timestamp or run asynchronously ✅ verified 2026-04-17 — `Dig_Tester.ucf:L196` ("RJ45 Trigger Connector Pins"); `SERDES_RX_Mach.vhd:L35,L43` (SERDES sync timestamp, TRIG_FLAG; async commands via GRETINA frame)
- Provides 2 **NIM outputs** for triggering/synchronization ✅ verified 2026-04-17 — `Dig_Tester.vhd:L62` (`NIM_OUT: out std_logic_vector(1 downto 0)`)
- VME register interface for waveform programming and control

---

## Hardware

### DACs
- **2× AD9747 dual 16-bit DAC** ✅ verified 2026-04-07 — `DAC_SPI.vhd` header
- Clock selectable: 50 / 100 / 200 MHz (register `clock_freq_sel[1:0]`: `00`=50, `01`=100, `10`=50, `11`=200) ✅ verified 2026-04-07 — `Dig_Tester_pkg.vhd`
- ⚠️ Code note: "previous comments say wavx_cs_trigx outputs don't route at 200 MHz" — re-enabled 2019-08-14 for noise logic checking. ✅ verified 2026-04-08 — `Dig_Tester.vhd:L450-451` (comment + re-enable note)
- Waveform memory is 18-bit wide internally; output truncated to 16-bit for DAC (`Waveform_Reader.vhd: dac_data_out(15:0)`) ✅ verified 2026-04-07 — `Waveform_Reader.vhd:L4-5`
- SPI clock max 40 MHz per AD9747 datasheet ✅ verified 2026-04-07 — `DAC_SPI.vhd` comment
- Clock sources selectable: external (SERDES RCLK), 50 MHz local oscillator, or SYS_CLK × 20/7 ✅ verified 2026-04-17 — `Clock_Manager.vhd:L42` (`clock_source: 0b00=external clock, 0b01=50MHz local osc, 0b11=SYS_CLK x 20/7`); SYS_CLK is 16 MHz VME SYSCLK (quadrupled internally before ×20/7 multiply)
- DAC outputs driven via SPI interface from FPGA (`DAC_SPI.vhd`)

### Analog Switch Matrix (`Analog_Switch_MUX.vhd`)
- Routes any DAC output (DAC0, DAC1, DAC0+DAC1 sum, DAC0−DAC1 diff) to any of 10 output channels
- MUX controlled via I2C (`mux_scl`, `mux_sda_in/out`, `mux_sda_dir`, `mux_reset_n`) ✅ verified 2026-04-21 — `Analog_Switch_MUX.vhd:L39-43` (entity port declarations)
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
| `Analog_Switch_MUX.vhd` | Analog switch matrix control — 8×10 ADG2108 chips, 7 source signals per channel, SPI clock ≤769 kHz (1.3 µs min low time ✅ `L134`) |
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

## Operational Notes (from wiki, 2013-02-10 — JTA)

- **Current installation:** Crate 10 of the DFMA system (as of 2013-02-10)
- **Control software:** No page in the DGS Commander EDM screens — must use **GammaWareR2** program on the engineering PC in the data room
- **Power cycle warning:** This board **completely loses its FPGA program** (goes blank) if power is cycled. Known PCB issue. If power was cycled:
  - Use the old beater laptop to re-load the FPGA while board is powered
  - FPGA image: `dig_tester.bit` (on the laptop desktop)
  - Contact John Anderson or Mike Oberling for password and reload sequence
- **LED status check:** Visually verify FPGA loaded state — yellow LEDs at bottom-right:
  - **Illuminated** → FPGA has valid program
  - **Dark** → FPGA lost its mind; reload needed
- **VME access activation:** Normal IOC 10 boot script does **not** map the Digitizer Tester to VME. ✅ verified 2026-04-25 — `ANLDAQ/ioc/boot/vme10.cmd` (no `asynDebugCard` call in boot script; only `asynDebugConfig("DBG",0)` at L115 but no card mapped). To enable engineering PC access, type at the IOC 10 terminal:
  ```
  asynDebugCard(4,7)
  ```
  After this, GammaWareR2 can reach the board. ✅ verified 2026-04-25 — `asynDebugDriver.cpp:L472` (`asynDebugCard(cardno, slot)` exists; maps card #4, slot 7 to `daqBoards[]` for debug register access)

---

*Created: 2026-04-05 (from SVN Digitizer_Tester VHDL source)*

## Cross-References

- `knowledgeBase/DGS_SVN.md` — `Digitizer_Tester/` entry in the SVN archive index
- `knowledgeBase/fpga.md` — FPGA firmware overview; Digitizer Tester injects test signals into the trigger chain
- `knowledgeBase/ttcl.md` — TTCL spec; the Digitizer Tester generates compatible discriminator bit patterns
- `knowledgeBase/connectors.md` — RJ45 pinout (same connector as digitizer); MTRG/RTRG 125-pin SERDES links the Tester connects to
- `knowledgeBase/DIG_firmware_expert.md` — digitizer firmware being tested
- `knowledgeBase/troubleshooting.md` — General DGS troubleshooting guide (no specific Digitizer Tester section; commissioning use is undocumented)
