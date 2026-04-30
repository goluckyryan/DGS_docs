# Collector Box FPGA Firmware — ControlStripe & CtlFanout

Stability: C3 - Structural / stable

**Source repos:** `~/FPGA_svn2git/ControlStripe_git/` and `~/FPGA_svn2git/CtlFanout_git/`  
**Origin:** ANL PSG SVN (`NewBlackBox/Firmware/ControlStripe/` and `NewBlackBox/Firmware/CtlFanout/`), migrated 2026-04-17  
**Tool:** Xilinx ISE 14.7  
**Authors:** jta (John T. Anderson), moberling (Michael Oberling)

**See also:** [`collectorboxpi.md`](collectorboxpi.md), [`collectorbox_devicesupport.md`](collectorbox_devicesupport.md), [`collectorbox_PVs.md`](collectorbox_PVs.md)

---

## Table of Contents

- [System Context](#system-context)
- [ControlStripe FPGA](#controlstripe-fpga)
  - [Target Device](#target-device)
  - [Role & Purpose](#role--purpose)
  - [Port Summary](#port-summary)
  - [Clock Architecture](#clock-architecture)
  - [DEVSEL / SPI Routing](#devsel--spi-routing)
  - [Internal Registers](#internal-registers)
  - [PCAL6416A LED Controller](#pcal6416a-led-controller)
  - [Sub-Components](#sub-components)
  - [Code Revision](#code-revision)
- [CtlFanout FPGA](#ctlfanout-fpga)
  - [Target Device](#target-device-1)
  - [Role & Purpose](#role--purpose-1)
  - [Port Summary](#port-summary-1)
  - [ADS1158 ADC Scanner](#ads1158-adc-scanner)
  - [SPI CE/MISO Fan-out](#spi-cemiso-fan-out)
  - [Sub-Components](#sub-components-1)
- [Shared Library Files](#shared-library-files)

---

## System Context

Each NewBlackBox **collector box chassis** contains two FPGA boards on the motherboard:

| Board | FPGA | Function |
|-------|------|----------|
| **CtlFanout** | Spartan-6 XC6SLX4 | RPi SPI gateway; ADS1158 ADC scanning; CE/MISO fan-out to all other FPGAs | ✅ verified 2026-04-23 — `CtlFanout_git/Source/Top.vhd:L9` (Target Devices: XC6SLX4-2TQG144)
| **ControlStripe** (×6) | Spartan-3 XC3S400 (revised from XC6SLX4 in 2021 due to supply chain) | Per-stripe power/relay/clock/sync control; SPI passthrough to 5 SlopeBox FPGAs | ✅ verified 2026-04-23 — `ControlStripe_git/Source/Top.vhd:L9` (original XC6SLX4-2TQG144, revised XC3S400-4TQG144C Oct 2021)

The **CtlFanout** sits between the Raspberry Pi and the rest of the board. It decodes the 5-bit DEVSEL bus, fans out CE signals selectively, muxes MISO back to the Pi, and scans the ADS1158 temperature/voltage ADCs.

The **ControlStripe** (one per stripe group of up to 5 SlopeBoxes) handles per-stripe power control, clock distribution (LVDS 50 MHz to SBXs), SYNC propagation, ground-check relay, and LED status controller.

---

## ControlStripe FPGA

### Target Device

| Item | Value |
|------|-------|
| Original target | XC6SLX4-2TQG144 | ✅ verified 2026-04-23 — `ControlStripe_git/Source/Top.vhd:L9`
| Revised target | XC3S400-4TQG144C (Oct 2021, supply chain) | ✅ verified 2026-04-23 — `ControlStripe_git/Source/CtlStripe_Spartan3.ucf:L3-4`
| Package | TQG144 |
| Tool | ISE 14.7 |
| Code date | `0x2301` (Jan 2023) | ✅ verified 2026-04-23 — `ControlStripe_git/Source/Top.vhd:L194`
| Code revision | `0x2018` (changed from 2017→2018 on 2023-01-13) | ✅ verified 2026-04-23 — `ControlStripe_git/Source/Top.vhd:L195`

### Role & Purpose

Each ControlStripe board manages one "stripe" — a group of up to 5 SlopeBox modules. It:
- Distributes **LVDS 50 MHz clock** (selectable: internal oscillator or TTCL trigger clock from Electric Honeycomb) to up to 5 SBX slots
- Routes **SYNC** pulses from the Honeycomb trigger decoder to SBX slots
- Controls **48 V power enable relays** (one per SBX slot) — active-low logic; polarity inverted per STRIPE_ID parity
- Controls **ground-check relay** and **coax communication relay**
- Passes SPI transactions through to SBX FPGAs via the board-wide SPI bus
- Drives the **PCAL6416A** I2C GPIO expander (only odd-numbered stripes: IDs 1, 3, 5) for LED status display

### Port Summary

| Port | Direction | Description |
|------|-----------|-------------|
| `STRIPE_ID[0:2]` | in | 3-bit stripe ID (board-wired); decoded to STRIPE_CODE ✅ verified 2026-04-29 — `ControlStripe_git/Source/Top.vhd:L37` (`STRIPE_ID : in std_logic_vector(0 to 2)`) |
| `OSC_CLOCK_1/2` | in | Free-running 50 MHz oscillator clocks |
| `TRIG_CLK_P/N` | in | LVDS trigger clock from Honeycomb/trigger decoder |
| `TRIG_SYNC` | in | Sync pulse from Honeycomb/trigger decoder |
| `PI_SCK/CE/MISO/MOSI` | in/out | SPI interface from CtlFanout (board-wide fanout) |
| `PI_DEVSEL[0:4]` | in | 5-bit device select driven directly by RPi ✅ verified 2026-04-29 — `ControlStripe_git/Source/Top.vhd:L50` (`PI_DEVSEL : in std_logic_vector(0 to 4); -- Driven directly by RPi`) |
| `CLK_50MHZ_P/N[1:5]` | out | LVDS 50 MHz clock outputs to 5 SBX slots |
| `COLL_CE/SCK/SDI/SDO[1:5]` | inout | SPI signals to/from 5 SBX slots |
| `SYNC[1:5]` | inout | SYNC pulse outputs to 5 SBX slots |
| `ENBL_GND_CHK[5:1]` | out | Enable ground check per slot |
| `GND_TEST_RELAY_EN[5:1]` | out | Ground test relay enable (active-low) |
| `PWR_48V_EN[5:1]` | out | 48 V power enable per slot (active-low, polarity depends on STRIPE_ID parity) |
| `COAX_COMM_RELAY_DRV1/2` | out | Coax communication relay drivers (active-low) |
| `NPRESENT` | in | Board present sense |
| `LED_SCL/SDA` | out/inout | I2C to PCAL6416A LED controller (odd stripes only) |

### Clock Architecture

The `stripe_dcm_mux` sub-component selects between:
- **Source 0 (DCM_INPUT_SEL=0):** `OSC_CLOCK_1` → DCM U1 → 50 MHz / 100 MHz oscillator-derived clocks
- **Source 1 (DCM_INPUT_SEL=1):** `TRIG_CLK_P/N` (LVDS, from Honeycomb) → DCM U2 → trigger-locked 50 MHz

Selection logic:
```
DCM_INPUT_SEL = forced (via STRIPE_CTL_REG[15:14]) OR
                '0' if TRIG DCM not locked (fall back to oscillator) OR
                '1' if TRIG DCM locked and not forced otherwise
```
✅ verified 2026-04-29 — `ControlStripe_git/Source/Top.vhd:L363-369` (STRIPE_CTL_REG[14]=force-enable, STRIPE_CTL_REG[15]=source-select; '0' when DCM_TRIG_LOCKED='0'; '1' when locked and reg[15]='1').

The 50 MHz clock to each SBX slot can be **gated off per slot** (`SBX_CLOCK_OUT_DISABLE[i]`) and globally disabled (`GLOBAL_CLOCK_OUT_DISABLE`) if the selected DCM is not locked. Clock outputs use OBUFTDS (LVDS_25, with P/N **swapped** from standard convention — N drives P pin and vice versa in the UCF). ✅ verified 2026-04-29 — `ControlStripe_git/Source/Top.vhd:L309` (comment: "Note that N and P signals are swapped"); L316-317 (O=>CLK_50MHZ_N, OB=>CLK_50MHZ_P confirms inversion).

### DEVSEL / SPI Routing

The Pi drives a 5-bit `PI_DEVSEL[0:4]` bus to all FPGAs. The ControlStripe decodes it using `STRIPE_CODE` (from `STRIPE_ID`) to determine if a transaction targets **this stripe's SBXs** or this FPGA itself:

| DEVSEL | STRIPE_ID target |
|--------|-----------------|
| 00000 | Globals (this FPGA if DEVSEL=00000) |
| 00001–00101 | Stripe 1 (SBX slots 1–5) |
| 00110–01010 | Stripe 2 (SBX slots 6–10) |
| 01011–01111 | Stripe 3 (SBX slots 11–15) |
| 10000–10100 | Stripe 4 (SBX slots 16–20) |
| 10101–11001 | Stripe 5 (SBX slots 21–25) |
| 11010–11110 | Stripe 6 (SBX slots 26–30) |
| 11111 | All stripes (broadcast) |

Each stripe FPGA passes SPI to its local SBX subset when the DEVSEL falls within its range. DEVSEL is latched on PI_CE with glitch protection (2-clock filter on CE, pipelined DEVSEL sampling).

### Internal Registers

All registers are 16-bit, addressed via the SPI PI_TRANSACTOR interface.

**DEVSEL=00000 (global) register addresses:** ✅ verified 2026-04-23 — `Top.vhd:L601-610`

| Address | Register | Description |
|---------|----------|-------------|
| 40 | `RELAY_CTL_REG` | Relay and ground-check control |
| 41 | `STRIPE_CTL_REG` | Stripe control (clock source, slot enables, SYNC mode, coax relays) |
| 43 | `PULSED_CONTROL_REG` | Self-clearing pulse register (reset logic, force DCM reset) |
| 44 | `LED_CTL_REG_0` | LED controller config word 0 (odd stripes only) |
| 45 | `LED_CTL_REG_1` | LED controller config word 1 (odd stripes only) |
| 48 | *(SBX global write trigger)* | Writing this address arms the global write to all SBX slots |

**DEVSEL=11111 (per-stripe) addresses** (offset per stripe: stripe 1 = +64, stripe 2 = +72, …, stride 8):

| Offset | Register |
|--------|----------|
| 0 | `RELAY_CTL_REG` |
| 1 | `STRIPE_CTL_REG` |
| 2 | `TRISTATE_CTL_REG` |
| 3 | `PULSED_CONTROL_REG` |
| 4 | `LED_CTL_REG_0` (odd stripes) / `SANDBOX_REG_0` (even stripes) |
| 5 | `LED_CTL_REG_1` (odd stripes) / `SANDBOX_REG_1` (even stripes) |

**`RELAY_CTL_REG` bit map:**

| Bits | Function |
|------|----------|
| [4:0] | `ENBL_GND_CHK[5:1]` — enable ground check per slot |
| [9:5] | `GND_TEST_RELAY_EN[5:1]` — ground test relay enable (active-low output) |
| [14:10] | `PWR_48V_EN` — 48 V power per slot (active-low; bit order flips based on STRIPE_ID parity) |

**`STRIPE_CTL_REG` bit map:**

| Bits | Function |
|------|----------|
| [4:0] | Per-slot enable (1=active): gates both 50 MHz clock output AND SYNC output to each SBX slot (bit 0=slot1, …, bit 4=slot5); bit=0 → clock disabled + SYNC driven low ✅ verified 2026-04-23 — `Top.vhd:L270,376-380` |
| [5:7] | *(reserved)* |
| [8] | COAX_COMM_RELAY_DRV2 (active-low output: bit=1 → relay de-energized; bit=0 → relay energized) ✅ verified 2026-04-23 — `Top.vhd:L273` |
| [9] | COAX_COMM_RELAY_DRV1 (active-low output) ✅ verified 2026-04-23 — `Top.vhd:L274` |
| [10] | SYNC source select: 0=use SYNC static value (bit 11), 1=route TRIG_SYNC to SYNC outputs ✅ verified 2026-04-23 — `Top.vhd:L270` |
| [11] | SYNC static value when bit[10]=0 (0=drive SYNC low, 1=drive SYNC high) ✅ verified 2026-04-23 — `Top.vhd:L270` |
| [12] | Force GLOBAL_CLOCK_OUT_DISABLE override enable (1=use bit[13]) ✅ verified 2026-04-23 — `Top.vhd:L371` |
| [13] | Forced GLOBAL_CLOCK_OUT_DISABLE value (inverted: 1=clocks **enabled**/not disabled, 0=clocks disabled) ✅ verified 2026-04-23 — `Top.vhd:L371` (`not STRIPE_CTL_REG(13)`) |
| [14] | Force DCM_INPUT_SEL override enable (1=use bit[15]) ✅ verified 2026-04-23 — `Top.vhd:L365` |
| [15] | Forced DCM_INPUT_SEL value (inverted in forced mode: 1=oscillator/DCM_INPUT_SEL=0, 0=trigger clock/DCM_INPUT_SEL=1; non-forced: 1+TRIG_LOCKED=trigger clock) ✅ verified 2026-04-23 — `Top.vhd:L365-368` |

**`STRIPE_STATUS_REG` (read-only):**

| Bits | Function |
|------|----------|
| [10:8] | STRIPE_CODE |
| [12] | DCM_INPUT_SEL (current) |
| [13] | GLOBAL_CLOCK_OUT_DISABLE (current) |
| [14] | DCM_TRIG_LOCKED |
| [15] | DCM_OSC_LOCKED |

**Read-only registers (same address offsets):**

| Address (per-stripe) | Register |
|---------------------|----------|
| +6 (read) | `STRIPE_STATUS_REG` |
| +7 (read) | `CODE_DATE_REG` = 0x2301 |
| +8 (read) | `CODE_REVISION_REG` = 0x2018 |

### PCAL6416A LED Controller

**Only odd stripes (STRIPE_CODE 001, 011, 101)** drive the PCAL6416A I2C GPIO expander for LED control. Even stripes hold SCL/SDA high (idle).

The `pcal6416a_controller` (`Source/pcal6416a_controller.vhd`, author M. Oberling, created Oct 2014) implements:
- I2C master at SCL = clock_in/8 (adjustable)
- I2C address: `0b0100000` (0x20 — standard PCAL6416A address with A2:A0=000) ✅ verified 2026-04-23 — `ControlStripe_git/Source/pcal6416a_controller.vhd:L57` (`slave_address := "0100000"`)
- 3 trigger modes: full config+pin write/read, pin write/read only, read-only
- 32-bit `config_data_out` → drives I/O direction and polarity registers
- 16-bit `pin_data_out` → drives output data register
- Returns `config_data_in` and `pin_data_in` on read

Controlled via `LED_CTL_REG_0` (32-bit config split over two 16-bit registers) and `LED_CTL_REG_1` (pin output data, 16-bit).

### Sub-Components

| File | Description |
|------|-------------|
| `Top.vhd` | Top-level entity (988 lines) |
| `stripe_dcm_mux.vhd` | DCM clock mux — selects between oscillator and trigger clock sources |
| `pcal6416a_controller.vhd` | I2C master controller for PCAL6416A GPIO expander (LED driver) |
| `PI_TRANSACTOR.vhd` | SPI register transactor — RPi SPI → internal register bus |
| `function_lib.vhd` | General VHDL function library (shared across NewBlackBox boards) |
| `type_lib.vhd` | Type definitions (shared) |
| `xilinx_lib.vhd` | Xilinx primitive wrappers (shared) |
| `SlopeBoxPickoffPkg.vhd` | SlopeBox pickoff package (shared) |

Note: `SERIAL_CTL_MACH` (referenced in Top.vhd as `entity work.SERIAL_CTL_MACH`) handles the core SPI receive state machine; it is defined within `PI_TRANSACTOR.vhd`. ✅ verified 2026-04-23 — `PI_TRANSACTOR.vhd:L47` (`entity SERIAL_CTL_MACH is`)

---

## CtlFanout FPGA

### Target Device

| Item | Value |
|------|-------|
| Target | XC6SLX4-2TQG144 (Spartan-6) | ✅ verified 2026-04-23 — `CtlFanout_git/Source/Top.vhd:L9`
| Package | TQG144 |
| Tool | ISE 14.7 |

### Role & Purpose

The **CtlFanout** is the central SPI gateway for the entire collector box. The Raspberry Pi communicates exclusively through this FPGA, which then:
- Routes **CE signals** selectively to the correct downstream FPGA or ADC based on DEVSEL decoding
- Multiplexes **MISO** back to the Pi from the selected device
- Fans out **SCK** and **SDO (MOSI)** board-wide
- Scans all **ADS1158 multi-channel ADCs** for temperature/voltage monitoring across 6 stripe groups
- Monitors **alarm relay** state from external power supply
- Provides 2 **NIM outputs** and 1 **NIM input** for diagnostics

### Port Summary

| Port Group | Signals | Description |
|-----------|---------|-------------|
| Clocks | `TRIG_CLK_P/N`, `REFCLK1`, `REFCLK2` | LVDS trigger clock + dual reference oscillators (connected to different GCLK pins for routing flexibility) |
| RPi SPI | `PI_GPIO_16/18/19/20/21` | SPI1: CE2, CE0, MISO, MOSI, SCLK |
| DEVSEL | `PI_DEVSEL0`–`4` | 5-bit device select bus (Pi GPIO 13,23,24,25,26) |
| CE outputs | `PI_CE1`–`PI_CE15` | Fan-out CE to 6 stripe FPGAs + BGO pattern FPGA + 6 ADS1158 ADC chips | ✅ verified 2026-04-23 — `CtlFanout_git/Source/Top.vhd:L112-126` (PI_CE1–PI_CE15 port declarations)
| MISO inputs | `PI_MISO1`–`PI_MISO15` | MISO from downstream devices |
| SPI fanout | `SDO_TO_BOARD`, `SCK_TO_BOARD` | Board-wide MOSI and SCK |
| ADC control | `MON_ADC_START1`–`6`, `MON_ADC_DRDY1`–`6`, `MON_ADC_RESET` | ADS1158 start pulses, data-ready inputs, reset |
| Alarm relay | `RLY1_COM`, `RLY2_COM`, `RLY_CABLE_UNPLUGGED` | External PSU relay status |
| Diagnostics | `CTL_FPGA_LED`, `NIM_OUT1/2`, `NIM_IN_1` | LED, NIM I/O |

### CE Fan-out Decoding

The CtlFanout decodes the 5-bit DEVSEL to route `PI_CE` to exactly one target:

| PI_CE output | Destination |
|-------------|-------------|
| PI_CE1–6 | ControlStripe FPGAs 1–6 (one per stripe group) |
| PI_CE7 | BGO Pattern FPGA |
| PI_CE8–9 | Reserved (future "new black box" amplifier) |
| PI_CE10 | ADC U32 (Stripe 1 ADS1158) |
| PI_CE11 | ADC U92 (Stripe 3 ADS1158) |
| PI_CE12 | ADC U93 (Stripe 5 ADS1158) |
| PI_CE13 | ADC U82 (Stripe 2 ADS1158) |
| PI_CE14 | ADC U94 (Stripe 4 ADS1158) |
| PI_CE15 | ADC U95 (Stripe 6 ADS1158) |

✅ verified 2026-04-23 — `CtlFanout_git/Source/Top.vhd:L112-126` (port comments); `L657-716` (COLLECTOR_CE fan-out logic)

The MISO mux (`MISO_REDIRECT_TABLE.vhd`) selects which of `PI_MISO1`–`15` to drive back to `PI_GPIO_19` (Pi MISO) based on DEVSEL.

### ADS1158 ADC Scanner

The collector box uses **6 ADS1158** multi-channel ADCs (one per stripe group) to monitor per-slot temperatures and voltages. The `ADS1158_STARTUP_ROM` (`ADC_SCANNER.vhd`) provides:

- **ROM-driven scan sequencer:** Reads a 32-bit-wide ROM table at startup; upper 16 bits = opcode, lower 16 bits = data. Commands the ADS1158 via the `ADS1158_transactor.vhd` SPI master.
- **Auto-scan loop:** After initialization, continuously cycles through all configured channels.
- **DPRAM output:** Stores ADC readings in a 1K×16-bit dual-port BRAM accessible by the Pi.
- **4 MHz SLOW_CLOCK:** Scanner operates at 4 MHz rate (derived from 100 MHz via divide-by-25 counter).

The `ADS1158_transactor.vhd` is the low-level SPI transactor to the ADC chip.

Clock architecture mirrors ControlStripe: selects between `REFCLK1` (oscillator) and `TRIG_CLK_P/N` (trigger clock) via a DCM mux.

### Sub-Components

| File | Description |
|------|-------------|
| `Top.vhd` | Top-level entity (~1537 lines) |
| `CtlFanout_pkg.vhd` | Package definitions |
| `ADC_SCANNER.vhd` | ADS1158 startup ROM + auto-scan sequencer |
| `ADS1158_transactor.vhd` | ADS1158 SPI transactor (low-level) |
| `MISO_REDIRECT_TABLE.vhd` | SPI MISO multiplexer based on DEVSEL |
| `PI_TRANSACTOR.vhd` | RPi SPI register transactor (same as ControlStripe) |
| `ADS1158_testbench.vhd`, `ADS1158_testbench2.vhd` | Testbenches |
| `Top_testbench.vhd` | Top-level testbench |
| `backup.vhd` | Legacy working backup |

---

## Shared Library Files

Three VHDL library files are shared across ControlStripe (and potentially other NewBlackBox boards):

| File | Description |
|------|-------------|
| `function_lib.vhd` | General utility functions (`to_std_logic_vector`, etc.) |
| `type_lib.vhd` | Common type definitions |
| `xilinx_lib.vhd` | Wrappers for Xilinx primitives (IBUFG, BUFG, DCM, OBUFTDS, etc.) |

---

*Source: `~/FPGA_svn2git/ControlStripe_git/` and `~/FPGA_svn2git/CtlFanout_git/`. Created 2026-04-23.*

---

## Cross-References

- `knowledgeBase/collector_fpga.md` — Collector box FPGA firmware (Git repo, current): CtrlFPGA and StripeFPGA architecture
- `knowledgeBase/collectorboxpi.md` — Collector Box Raspberry Pi IOC; interfaces with CtlFanout (ADS1158 ADC) and ControlStripe (SPI registers) via SPI1
- `knowledgeBase/collectorbox_devicesupport.md` — EPICS device support for collector box PVs; routes CA writes to FPGA registers
- `knowledgeBase/hardware_architecture.md` — Gammasphere hardware overview; collector box role in signal chain
- `knowledgeBase/sbxPi_ioc.md` — SBX Pi IOC; shares same `PickoffLocalSerial` / SPI framework as CtlFanout
- `knowledgeBase/gammasphere_geometry.md` — 110 GS detector holes and collector box quadrant assignments (SE/SW/NE/NW)
