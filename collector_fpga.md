# Collector Box FPGA Firmware
_Source: `DGS_tools_pack/collector_FPGA/` (local repo)_
_Explored: 2026-04-06_

---

## Overview

The collector box contains **two FPGAs** plus pickoff card FPGAs for SBX:

| FPGA | Directory | Role |
|------|-----------|------|
| **CtrlFPGA** | `CollectorBox_CtrlFPGA/` | Housekeeping, monitoring, control |
| **StripeFPGA** | `CollectorBox_StripeFPGA/` | Per-slot relay/stripe/LED control |
| **PickoffCard SBX Interface** | `PickoffCard_SBX_Interface/` | Pickoff card for standard SBX — **Spartan-6 XC6SLX4** (TQG144) |
| **PickoffCard SBX Extension** | `PickoffCard_SBX_Extension/` | Pickoff card for SBX Extension (SBX2) — **Spartan-6 XC6SLX4** (TQG144) |

> ✅ verified 2026-04-13 — `PickoffCard_SBX_Extension/Revision_A/Work/SlopeBoxInt.twr:L1` (`xc6slx4,tqg144,C,-2`). Tagged release: `FPGA/Firmware_Tags/SBX/tag_20221020/slopeboxint.bit` (Oct 20, 2022).

The collector box hosts **6 slots (S1–S6)**, each carrying DIGitizer boards.

---

## CtrlFPGA — Control & Housekeeping FPGA

**Key files:**
- `CtrlFPGAinitializer.c` / `.h` — register address map (C struct)
- `CtrlFPGA_registers_A.vhd` / `_B.vhd` / `_C.vhd` — VHDL register definitions
- `CtrlFPGA.db` — EPICS records (`GS${DetNbr}_*` PVs, DTYP=CollectorLocalSerial)
- `CtrlFPGA_reg.db` — Register-level EPICS records
- `ControlGlobals.db` — Global control records

### Register Map Summary

**Control registers (addr 0–8):**

| Addr | Name | Function |
|------|------|----------|
| 0 | `ctl_bank_readback` (R) / `ctl_pulsed_control` (W) | Read: register bank readback; Write: pulsed resets (startup ROM reset, master reset, serial reset, FIFO reset, etc.) |
| 1 | `FPGA_CTL_REG` | Main FPGA control register |
| 2 | `ctl_ila_control` | ILA (Integrated Logic Analyzer) control |
| 3 | `ctl_mask` | Alarm mask |
| 4 | `ctl_alarm_control` | Alarm control |
| 5 | `ctl_mask_misc_control` | Misc mask control |
| 6 | `scanner_INITIAL_ROM_ADDRESS` | Scanner ROM start address |
| 7 | `ADC_transactor_FIFO` | ADC transactor FIFO (user port) |

✅ verified 2026-04-12 — `collector_FPGA/CollectorBox_CtrlFPGA/CtrlFPGAinitializer.h:L2-10` (corrected addresses; previously off by 2)

**Housekeeping registers (addr 123–127):**

| Addr | Name | Function |
|------|------|----------|
| 123 | `ctl_trig_clk_counter` | Trigger clock counter |
| 124 | `pi_gpio_readback_1` | Raspberry Pi GPIO readback 1 |
| 125 | `pi_gpio_readback_2` | Raspberry Pi GPIO readback 2 |
| 126 | `ctl_code_date` | Firmware build date |
| 127 | `ctl_code_revision` / `ctl_dpram_bank_sel` | Firmware revision + DPRAM bank select |

✅ verified 2026-04-12 — `collector_FPGA/CollectorBox_CtrlFPGA/CtrlFPGAinitializer.h:L11-16`

**Per-slot monitoring (S1–S6, addr 129–254):**

Each slot (S1–S6) has a block of registers:

| Register | Function |
|----------|----------|
| `Sx_DVI1–DVI5_GndFault_I` | Ground fault current for each DVI channel (5 per slot) |
| `Sx_DVI1–DVI5_48V_I` | 48V current monitor for each DVI channel |
| `Sx_12V` / `Sx_25V` / `Sx_33V` | Supply voltage monitors (1.2V / 2.5V / 3.3V rails) |
| `Sx_ADC_OFFSET` | ADC offset calibration |
| `Sx_ADC_VCC` | ADC supply voltage |
| `Sx_ADC_TEMP` | ADC temperature |
| `Sx_ADC_GAIN` | ADC gain |
| `Sx_ADC_REF` | ADC reference |

Also monitors `BGO_FPGA_33V`, `BGO_FPGA_25V`, `BGO_FPGA_12V` (BGO board power rails).

### ADC Hardware Calibration Constants
_Source: `DGS_SVN/dgs/NewBlackBox/RaspberryPi/PreEpicsCode/Scan_DVI_Power.c` (pre-EPICS reference code)_

CtrlFPGA uses a 16-bit ADC with 5V span: **13,107 counts/V**. ✅ verified 2026-04-16 — `Scan_DVI_Power.c:L53-54` ("ADC input span is 5V and ADC is a 16 bit device, so we expect that the conversion is nominally 65535/5, or 13,107 counts per volt.")

| Rail | Expected ADC value |
|------|-------------------|
| 3.3V | ≈ 43,353 counts | ✅ verified 2026-04-16 — `Scan_DVI_Power.c:L56`
| 2.5V | ≈ 32,767 counts | ✅ verified 2026-04-16 — `Scan_DVI_Power.c:L57`
| 1.8V | ≈ 23,593 counts | ✅ verified 2026-04-16 — `Scan_DVI_Power.c:L58`
| 1.2V | ≈ 15,728 counts | ✅ verified 2026-04-16 — `Scan_DVI_Power.c:L59`

**48V current monitor (TMCS1101A4B):** 400 mV/A → **5,243 counts/A** ✅ verified 2026-04-16 — `Scan_DVI_Power.c:L61-62` ("TMCS1101A4B conversion ratio is 400mv/A. So 13107cnt/v multiplied by 0.4V/A gives us 5243 counts per Amp.")
- Warning threshold: > 2,700–3,000 counts (~0.5–0.6 A)
- Shutdown threshold: > 5,000 counts (~0.95 A)
- Expected: SBX + detector < 0.6 A under any circumstance

**Ground fault monitor (AD8236, 22 Ω shunt, gain=100):** **28,835 counts/mA** ✅ verified 2026-04-16 — `Scan_DVI_Power.c:L67-69` ("AD8236 measures the voltage drop across a 22 ohm resistor ... 13107cnt/V multiplied by 2.2V/ma gives us 28,835 counts per mA")
- Fitted relationship for relay-open with external detector path: `ADCval ≈ 5e6 × Rdet^−0.731` ✅ verified 2026-04-16 — `Scan_DVI_Power.c:L94`

**ADC scanner update rates (6 ADCs, 1 scanner):** ✅ verified 2026-04-16 — `DGS_SVN/dgs/NewBlackBox/RaspberryPi/PreEpicsCode/Scan_DVI_Power.c:L111-131`
- Fast (DRATE=11, 23,739 sps): 3.29–5.31 ms total update cycle
- Slow (DRATE=00, 1,831 sps): 42.6–69 ms total update cycle

### EPICS PV Pattern

All CtrlFPGA PVs use pattern `GS${DetNbr}_<register_name>` with `DTYP=CollectorLocalSerial`.

Example pulsed control PVs (all `bo` records):
- `GS${DetNbr}_ctl_reset_startup_rom`
- `GS${DetNbr}_ctl_master_reset`
- `GS${DetNbr}_ctl_serial_reset`
- `GS${DetNbr}_ctl_reset_cmd_fifo`

### Per-Slot CtrlFPGA Register Addresses (Exact)
_Source: `collector_FPGA/CollectorBox_CtrlFPGA/TEMP.c`, `GAIN.c`, `OFFSET.c`, `REF.c`, `IMON.c`, `VMON.c`, `GF_I.c`_
_Explored: 2026-04-18_

The following tables give exact FPGA register addresses used by the ADC scanner for each slot.
Note: Slots are addressed in the order S1, S3, S5, S2, S4, S6 (odd slots first).

**ADC Temperature (`ADC_TEMP_ADDR`):** ✅ verified 2026-04-19 — `TEMP.c:L1-8` (exact addresses match)
| Slot | Addr |
|------|------|
| S1 | 147 |
| S3 | 168 |
| S5 | 189 |
| S2 | 210 |
| S4 | 231 |
| S6 | 252 |

**ADC Gain (`ADC_GAIN_ADDR`):** ✅ verified 2026-04-19 — `GAIN.c:L1-8`
| Slot | Addr |
|------|------|
| S1 | 148 |
| S3 | 169 |
| S5 | 190 |
| S2 | 211 |
| S4 | 232 |
| S6 | 253 |

**ADC Offset (`ADC_OFFSET_ADDR`):** ✅ verified 2026-04-19 — `OFFSET.c:L1-8`
| Slot | Addr |
|------|------|
| S1 | 145 |
| S3 | 166 |
| S5 | 187 |
| S2 | 208 |
| S4 | 229 |
| S6 | 250 |

**ADC Reference (`ADC_REF_ADDR`):** ✅ verified 2026-04-19 — `REF.c:L1-8`
| Slot | Addr |
|------|------|
| S1 | 149 |
| S3 | 170 |
| S5 | 191 |
| S2 | 212 |
| S4 | 233 |
| S6 | 254 |

**48V Current Monitor per DVI (`FPGA_IMON_ADDR`, indexed 1–30, 5 DVIs × 6 slots):** ✅ verified 2026-04-19 — `IMON.c:L1-30` (S1=134-138, S3=155-159, S5=176-180, S2=197-201 reversed, S4=218-222 reversed, S6=239-243 reversed)
| Index | Signal | Addr |
|-------|--------|------|
| 1–5 | S1_DVI1–DVI5_48V_I | 134–138 |
| 11–15 | S3_DVI1–DVI5_48V_I | 155–159 |
| 21–25 | S5_DVI1–DVI5_48V_I | 176–180 |
| 10–6 | S2_DVI1–DVI5_48V_I | 197–201 (reversed: DVI1=197, DVI5=201) |
| 15–11 | S4_DVI5–DVI1_48V_I | 218–222 (reversed) |
| 30–26 | S6_DVI5–DVI1_48V_I | 239–243 (reversed) |

**Voltage Monitors (`FPGA_VOLTAGE_ADDR`, 12V/2.5V/3.3V per slot + BGO rails):** ✅ verified 2026-04-19 — `VMON.c:L1-21`
| Slot / Rail | 12V addr | 25V addr | 33V addr |
|-------------|----------|----------|----------|
| S1 | 141 | 142 | 143 |
| S3 | 162 | 163 | 164 |
| S5 | 183 | 184 | 185 |
| S2 | 204 | 205 | 206 |
| S4 | 225 | 226 | 227 |
| S6 | 246 | 247 | 248 |
| BGO_FPGA | 249 | 245 | 244 |

**Ground Fault Current per DVI (`FPGA_GNDFAULT_I_ADDR`, 5 per slot):** ✅ verified 2026-04-19 — `GF_I.c:L1-30`
| Slots | DVI1–DVI5 Addr Range |
|-------|---------------------|
| S1 | 129–133 |
| S3 | 150–154 |
| S5 | 171–175 |
| S2 | 192–196 |
| S4 | 213–217 |
| S6 | 234–238 |

---

## StripeFPGA — Stripe & Relay Control FPGA

**Key files:**
- `StrpFPGAinitializer.c` / `.h` — register address map
- `StrpFPGA.db` (4,620 lines!) — full EPICS DB
- `StripeGlobals.db` — global stripe records

### Register Map Summary

Per-slot registers at addresses 64–111 (8 registers × 6 slots, base 64 + slot_offset×8): ✅ verified 2026-04-18 — `collectorboxpi/Pre_EPICS_Collector/SPI_Address.md` (Stripe FPGA Register Map)

| Offset | Name | Function |
|--------|------|----------|
| +0 | `relay_control_sx` | Ground test current injection relay enable (per channel) |
| +1 | `stripe_control_sx` | Stripe control |
| +2 | `tristate_control_sx` | SBX port tristate controls |
| +3 | `stripe_status_sx` | Stripe status (readback) |
| +4 | `led_control_sx_1` (odd slots S1/S3/S5) or `sandbox_reg_sx_1` | LED control 1 (odd slots only; sandbox stub on even slots) |
| +5 | `led_control_sx_2` / `sandbox_reg_sx_2` | LED control 2 (odd slots) / sandbox |
| +6 | `reserved_sx` | Reserved (nominally CODE_DATE but may be repurposed) |
| +7 | `code_revision_sx` | FPGA firmware build number (read-only) |

**Exact base addresses per slot:**

| Slot | Base addr | relay_control | stripe_control | tristate_control | stripe_status | code_revision |
|------|-----------|--------------|----------------|-----------------|--------------|---------------|
| S1 | 64 | 64 | 65 | 66 | 67 | 71 |
| S2 | 72 | 72 | 73 | 74 | 75 | 79 |
| S3 | 80 | 80 | 81 | 82 | 83 | 87 |
| S4 | 88 | 88 | 89 | 90 | 91 | 95 |
| S5 | 96 | 96 | 97 | 98 | 99 | 103 |
| S6 | 104 | 104 | 105 | 106 | 107 | 111 |

**Relay control** (`irly_sx_N`): Enables ground test current injection per channel per slot. `GS${DetNbr}_irly_s1_1` through `irly_s6_N`.

---

## PickoffCard SBX Interface / Extension

Two pickoff card variants for the Slope Box (SBX) interface:

| Directory | Description |
|-----------|-------------|
| `PickoffCard_SBX_Interface/` | Standard SBX pickoff card (Revision A only) |
| `PickoffCard_SBX_Extension/` | SBX Extension pickoff card (Revisions A–C + Prototype + Tags) |

**SBX Extension Revision C** (`PickoffCard_SBX_Extension/Revision_C/Source/`):
Key source files:
- `RefTop.vhd` — top-level entity
- `PI_TRANSACTOR.vhd` — Pi transactor (SPI communication with Raspberry Pi)
- `ADGS5412_controller.vhd` — analog switch controller
- `LTC1660_controller.vhd` — DAC controller (LTC1660: 8-ch 10-bit DAC) ✅ verified 2026-04-16 — `PickoffCard_SBX_Extension/Revision_C/Source/LTC1660_controller.vhd:L127-134` (DAC A–H = 8 ch, 10-bit data `LATCHED_PI_DATA(9 downto 0)`)
- `I2C_STARTUP_ROM.vhd` — I2C startup ROM sequencer
- `LOOK_UP_TABLE1.VHD` / `Ref_LUT1.VHD` — lookup tables
- `Ref_SlopeBoxScan.vhd` — slope box scanning logic
- `RAM_Buffer.vhd` — RAM buffer

Note: Revision C README states it is "supposed to be the same as revision B, at least at the time of design."

---

## Relationship to collectorboxpi

The Raspberry Pi (collectorboxpi soft IOC) communicates with both FPGAs via SPI:
- Pi GPIO readback registers in CtrlFPGA (`pi_gpio_readback_1/2`) confirm Pi ↔ CtrlFPGA link
- StripeFPGA relay/LED control is driven from the Pi via the EPICS IOC
- PickoffCard `PI_TRANSACTOR.vhd` implements the Pi ↔ pickoff card SPI protocol

See: `knowledgeBase/collectorboxpi.md`, `knowledgeBase/collectorbox_devicesupport.md`, `knowledgeBase/sbx.md`

---

## Cross-References

- `knowledgeBase/collectorboxpi.md` — Raspberry Pi soft IOC, PXE boot, HV control
- `knowledgeBase/collectorbox_PVs.md` — Full PV list (1,431 records/detector ✅ verified 2026-04-16)
- `knowledgeBase/collectorbox_devicesupport.md` — EPICS device support internals, SPI driver, CAMAC_IO
- `knowledgeBase/sbx.md` — Slope Box Extension hardware, BGO HV, GS_ID dongle
