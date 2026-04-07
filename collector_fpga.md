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
| **PickoffCard SBX Interface** | `PickoffCard_SBX_Interface/` | Pickoff card for standard SBX |
| **PickoffCard SBX Extension** | `PickoffCard_SBX_Extension/` | Pickoff card for SBX Extension (SBX2) |

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
| 0 | `ctl_bank_readback` | Read-only register bank readback |
| 1 | `ctl_pulsed_control` | Pulsed resets (startup ROM reset, master reset, serial reset, FIFO reset, etc.) |
| 2 | `FPGA_CTL_REG` | Main FPGA control register |
| 3 | `ctl_ila_control` | ILA (Integrated Logic Analyzer) control |
| 4 | `ctl_mask` | Alarm mask |
| 5 | `ctl_alarm_control` | Alarm control |
| 6 | `ctl_mask_misc_control` | Misc mask control |
| 7 | `scanner_INITIAL_ROM_ADDRESS` | Scanner ROM address |
| 8 | `ADC_transactor_FIFO` | ADC transactor FIFO access |

**Housekeeping registers (addr 123–127):**

| Addr | Name | Function |
|------|------|----------|
| 123 | `ctl_trig_clk_counter` | Trigger clock counter |
| 124 | `pi_gpio_readback_1` | Raspberry Pi GPIO readback 1 |
| 125 | `pi_gpio_readback_2` | Raspberry Pi GPIO readback 2 |
| 126 | `ctl_code_date` | Firmware build date |
| 127 | `ctl_code_revision` / `ctl_dpram_bank_sel` | Firmware revision + DPRAM bank select |

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

CtrlFPGA uses a 16-bit ADC with 5V span: **13,107 counts/V**.

| Rail | Expected ADC value |
|------|-------------------|
| 3.3V | ≈ 43,353 counts |
| 2.5V | ≈ 32,767 counts |
| 1.8V | ≈ 23,593 counts |
| 1.2V | ≈ 15,728 counts |

**48V current monitor (TMCS1101A4B):** 400 mV/A → **5,243 counts/A**
- Warning threshold: > 2,700–3,000 counts (~0.5–0.6 A)
- Shutdown threshold: > 5,000 counts (~0.95 A)
- Expected: SBX + detector < 0.6 A under any circumstance

**Ground fault monitor (AD8236, 22 Ω shunt, gain=100):** **28,835 counts/mA**
- Fitted relationship for relay-open with external detector path: `ADCval ≈ 5e6 × Rdet^−0.731`

**ADC scanner update rates (6 ADCs, 1 scanner):**
- Fast (DRATE=11, 23,739 sps): 3.29–5.31 ms total update cycle
- Slow (DRATE=00, 1,831 sps): 42.6–69 ms total update cycle

### EPICS PV Pattern

All CtrlFPGA PVs use pattern `GS${DetNbr}_<register_name>` with `DTYP=CollectorLocalSerial`.

Example pulsed control PVs (all `bo` records):
- `GS${DetNbr}_ctl_reset_startup_rom`
- `GS${DetNbr}_ctl_master_reset`
- `GS${DetNbr}_ctl_serial_reset`
- `GS${DetNbr}_ctl_reset_cmd_fifo`

---

## StripeFPGA — Stripe & Relay Control FPGA

**Key files:**
- `StrpFPGAinitializer.c` / `.h` — register address map
- `StrpFPGA.db` (4,620 lines!) — full EPICS DB
- `StripeGlobals.db` — global stripe records

### Register Map Summary

Per-slot registers at addresses 64–127 (8 registers × 8 addresses per slot):

| Offset | Name | Function |
|--------|------|----------|
| +0 | `relay_control_sx` | Ground test current injection relay enable (per channel) |
| +1 | `stripe_control_sx` | Stripe control |
| +2 | `tristate_control_sx` | Tristate control |
| +3 | `stripe_status_sx` | Stripe status (readback) |
| +4 | `led_control_sx_1` (odd slots S1/S3/S5) or `sandbox_reg_sx_1` | LED control or sandbox register |
| +5 | `led_control_sx_2` / `sandbox_reg_sx_2` | LED control 2 / sandbox |
| +6 | `reserved_sx` | Reserved |
| +7 | (additional) | — |

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
- `LTC1660_controller.vhd` — DAC controller (LTC1660: 8-ch 10-bit DAC)
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

See: `dgs/collectorboxpi.md`, `dgs/collectorbox_devicesupport.md`, `dgs/sbx.md`

---

## Cross-References

- `dgs/collectorboxpi.md` — Raspberry Pi soft IOC, PXE boot, HV control
- `dgs/collectorbox_PVs.md` — Full PV list (1,437 records/detector)
- `dgs/collectorbox_devicesupport.md` — EPICS device support internals, SPI driver, CAMAC_IO
- `dgs/sbx.md` — Slope Box Extension hardware, BGO HV, GS_ID dongle
