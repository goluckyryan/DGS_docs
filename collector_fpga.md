# Collector Box FPGA Firmware
Stability: C3 - Structural / stable
_Source: `DGS_tools_pack/collector_FPGA/` (local repo)_
_Explored: 2026-04-06_

## Table of Contents

- [Overview](#overview)
- [CtrlFPGA — Control & Housekeeping FPGA](#ctrlfpga--control--housekeeping-fpga)
  - [Register Map Summary](#register-map-summary)
  - [ADC Hardware Calibration Constants](#adc-hardware-calibration-constants)
  - [EPICS PV Pattern](#epics-pv-pattern)
  - [Control Register Bit Definitions](#control-register-bit-definitions)
  - [Per-Slot CtrlFPGA Register Addresses (Exact)](#per-slot-ctrlfpga-register-addresses-exact)
- [StripeFPGA — Stripe & Relay Control FPGA](#stripefpga--stripe--relay-control-fpga)
  - [Register Map Summary](#register-map-summary-1)
- [PickoffCard SBX Interface / Extension](#pickoffcard-sbx-interface--extension)
- [Relationship to collectorboxpi](#relationship-to-collectorboxpi)
- [Cross-References](#cross-references)

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

### Control Register Bit Definitions
_Source: `FPGA/collectorBox/CollectorBox_CtrlFPGA/CtrlFPGA_registers_B.vhd` — code-read 2026-04-25_

**`ctl_pulsed_control` (W, addr 0) — write-pulsed, self-clearing:**
| Bit | Name | Function |
|-----|------|----------|
| 1 | `ctl_reset_startup_rom` | Reset the startup ROM scanner |
| 2 | `ctl_master_reset` | Full FPGA master reset |
| 3 | `ctl_serial_reset` | Reset SPI serial interface |
| 6 | `ctl_reset_cmd_fifo` | Reset command FIFO |
| 7 | `ctl_transactor_go` | Trigger ADC transactor cycle |

**`FPGA_CTL_REG` (R/W, addr 1) — persistent control bits:**
| Bit | Name | Function |
|-----|------|----------|
| 0 | `MON_ADC_RESET` | Reset monitoring ADC |
| 1 | `CTL_FPGA_LED` | CtrlFPGA status LED |
| 2 | `NIM_OUT1` | NIM output 1 (front-panel NIM signal) |
| 3 | `NIM_OUT2` | NIM output 2 (front-panel NIM signal) |
| 4 | `ADC_scanner_reset` | Reset ADC scanner state machine |
| 6 | `ADC_transactor_fifo_rst` | Reset ADC transactor FIFO |
| 7 | `ADC_transactor_reset` | Reset ADC transactor |
| 15:8 | *(unnamed)* | Passed through to FPGA logic |

Default power-up value: `0xFFDF` (all bits set except bit 5). ✅ verified 2026-04-25 — `CtrlFPGA_registers_A.vhd:L3` (reset mask `X"FFDF"` = bits 0–4,6–15 active; bit 5 unused)

**`ctl_ila_control` (W, addr 2) — ILA (Integrated Logic Analyzer) debug:**
| Bits | Name | Function |
|------|------|----------|
| 1:0 | `ctl_ila_subsel` | ILA sub-select (which internal bus to capture) |
| 7:4 | `ctl_ila_sel_code` | ILA source selection code (which module to instrument) |
| 13 | `ctl_ila_clock_sel` | ILA clock source selection |

Default power-up value: `0x20F3`. ✅ verified 2026-04-25 — `CtrlFPGA_registers_A.vhd:L4`

**`ctl_dpram_bank_sel` (R/W, shares addr 14 with `ctl_code_revision`) — DPRAM bank:**
| Bits | Name | Function |
|------|------|----------|
| 2:0 | `ctl_dpram_bank` | Selects DPRAM bank (0–7) for banked register readback |

Default: `0x0007`. ✅ verified 2026-04-25 — `CtrlFPGA_registers_A.vhd:L15`

**BGO FPGA power rail register scanner slots** (monitoring-only, read from `ctl_bank_readback` at addr 0 after bank select):
| Signal | Scanner Slot | Description |
|--------|-------------|-------------|
| `BGO_FPGA_33V` | 131 | BGO FPGA 3.3V supply |
| `BGO_FPGA_25V` | 132 | BGO FPGA 2.5V supply |
| `BGO_FPGA_12V` | 136 | BGO FPGA 1.2V supply |

✅ verified 2026-04-25 — `CtrlFPGA_registers_A.vhd:L121-L136` (scanner slot assignments for BGO_FPGA rails)

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

Per-slot registers at addresses 64–111 (8 registers × 6 slots, base 64 + slot_offset×8): ✅ verified 2026-04-23 — `collectorboxpi/Pre_EPICS_Collector/SPI_Address.md:L190-L239`, `FPGA/collectorBox/CollectorBox_StripeFPGA/StrpFPGA_register_instantiation.vhd:L6-L23,L60-L83` (register names, slot bases, LED/sandbox asymmetry, reserved/code_revision mapping)

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

### Bit-Level Register Definitions

Derived from `StrpFPGA_registers.vhd` (generated by JTA's Excel VBA code generator). ✅ verified 2026-04-25 — `FPGA/collectorBox/CollectorBox_StripeFPGA/StrpFPGA_registers.vhd` full entity + architecture read

**`relay_control_sN`** — R/W, mask `0x7FFF` (bits[14:0]), reset `0x00000000` ✅ verified 2026-04-25 — `StrpFPGA_registers.vhd:L130-L135` (mask=0x7FFF; irly/grly/prly bits confirmed L187-L201)
| Bits | Name | Function |
|------|------|----------|
| [4:0] | `irly_sN_1..5` | Individual current injection relay — one per SBX slot detector (5 total) |
| [9:5] | `grly_sN_1..5` | Group relay bits — one per SBX slot detector (5 total) |
| [14:10] | `prly_sN_1..5` | P-relay (preamp/power relay) — one per SBX slot detector (5 total) |

All three relay types control physical relays used in the ground test current injection circuit. 5 detectors per stripe slot → 15 relay control bits.

**`stripe_control_sN`** — R/W, mask `0xFF1F` (bits[15:8] + bits[4:0]), reset `0x00000000` ✅ verified 2026-04-25 — `StrpFPGA_registers.vhd:L136-L141,L277-L289` (mask=0xFF1F; clock_and_sync bits[4:0], crly_earth[8], crly_signal[9], trig_sync[10], manual_sync[11], clock overrides[12-15])
| Bits | Name | Function |
|------|------|----------|
| [4:0] | `clock_and_sync_enable_sN_1..5` | Enable clock & sync distribution per SBX port (one per detector) |
| [8] | `crly_earth_sN` | Capacitive relay — earth connection |
| [9] | `crly_signal_sN` | Capacitive relay — signal connection |
| [10] | `trig_sync_enable_sN` | Enable trigger sync for this stripe |
| [11] | `manual_sync_sN` | Force manual sync assertion |
| [12] | `clock_output_enable_override_sN` | Override clock output enable state |
| [13] | `manual_clock_output_enable_state_sN` | Manual clock output enable state (when override=1) |
| [14] | `clock_source_override_sN` | Override clock source selection |
| [15] | `manual_clock_source_select_sN` | Manual clock source (when override=1) |

**`tristate_control_sN`** — R/W, mask `0x03FF` (bits[9:0]), reset `0x00000000`
| Bits | Name | Function |
|------|------|----------|
| [4:0] | `sbx_spi_tristate_sN_1..5` | Tristate the SPI bus to each SBX port (one per detector) |
| [9:5] | `sbx_clock_sync_tristate_sN_1..5` | Tristate clock/sync lines to each SBX port (one per detector) |

**`stripe_status_sN`** — read-only, mask `0xF700` (bits[15:8] + bits[10:8] = bits[15:8]), reset `0x00000000`
| Bits | Name | Function |
|------|------|----------|
| [10:8] | `stripe_id_sN` | 3-bit hardware stripe ID (wired from board) |
| [12] | `sbx_clock_source_sN` | Current clock source being used |
| [13] | `sbx_clock_enable_sN` | Current clock enable state |
| [14] | `dcm_trig_locked_sN` | DCM (trigger clock) PLL locked status |
| [15] | `dcm_osc_locked_sN` | DCM (oscillator clock) PLL locked status |

**`led_control_sN_1`** (odd slots S1/S3/S5 only) — R/W, mask `0xFFFF`, reset `0x00000000`
- Controls LEDs for the **pair** of slots (odd + next even). Each LED = 2 bits (intensity/mode).
- Bits[15:10] = `led_sN_3`, `led_sN_4`, `led_sN_5` (3 detector LEDs from odd slot)
- Bits[9:0] = `led_sN+1_1` through `led_sN+1_5` (5 detector LEDs from the paired even slot)
- Even slots (S2/S4/S6): this register position is a **sandbox** (writes accepted but no hardware function)

**`led_control_sN_2`** (odd slots S1/S3/S5 only) — R/W, mask `0xFFFF`, reset `0x00000000`
- Bits[3:0] = `led_sN_2`, `led_sN_1` (remaining 2 detector LEDs from odd slot)
- Bits[5:4] = `led_tip_sN_sN+1` — pair tip LED (2-bit)
- Bits[7:6] = `led_stat_sN_sN+1` — pair status LED (2-bit)
- Bits[15:8] = 4 unused 2-bit LED fields (currently unconnected)
- Even slots: this register position is also a **sandbox**

**`reserved_sN`** — read-only (write mask = 0), reset `0x00002208`
- Described as "nominally CODE_DATE but may be repurposed"
- Default value `0x2208` decodes as date `22/08` = August 2022 (month/day or year/week convention)

**`code_revision_sN`** — read-only (write mask = 0), reset `0x00000010` (= revision 16 decimal)
- Returns current FPGA firmware build revision

---

## PickoffCard SBX Interface / Extension

Two pickoff card variants for the Slope Box (SBX) interface:

| Directory | Description |
|-----------|-------------|
| `PickoffCard_SBX_Interface/` | Standard SBX pickoff card (Revision A + **Revision B**) |
| `PickoffCard_SBX_Extension/` | SBX Extension pickoff card (Revisions A–C + Prototype + Tags) |

**SBX Extension Revision C** (`PickoffCard_SBX_Extension/Revision_C/Source/`): ✅ verified 2026-04-22 — `RefTop.vhd` entity + `SlopeBoxPickoffPkg.vhd` component declarations read in full
- Target device: **Spartan-6 XC6SLX9-2TQG144** (larger than Rev A/B which use XC6SLX4), toolchain ISE 14.7, created 2020-10-05 (entity `SlopeBoxInt`, generic `globalPI_SPI_PORT`)
- Note: Revision C README states it is "supposed to be the same as revision B, at least at the time of design" — same register map, same interfaces.

Key source files and architecture:
- `RefTop.vhd` — top-level entity (`SlopeBoxInt`); same port list as Revision B (TrigClk diff pair, OSC, BGO disc bits, parallel BGO mux select, 5 SPI buses for analog switches, Pi SPI, DVI serial collector link)
- `SlopeBoxPickoffPkg.vhd` — package with all component declarations; defines the full module hierarchy
- **`SERIAL_CTL_MACH`** — central Pi SPI controller. Dual-port: accepts SPI from Raspberry Pi (SPI0: SCK/MOSI/MISO/CE) *or* DVI serial from the collector box (`BufSerCommClk`/`BufSerDatToPickoff`). `TRIG_CLK_PRESENT` flag selects which port is active. VIO backdoor available for lab debug (7-bit address, 16-bit data, RW, DO_TRANSACTION). Generates `MACH_ENABLEs[15:0]` (one-hot) to dispatch to subsidiary state machines, watchdog timers, `PREFETCH_DATA` from RAM, and `LATCHED_ADDRESS[6:0]` + `LATCHED_DATA_RECEIVED[15:0]` outputs.
- **`LOOK_UP_TABLE1`** — 7-bit Pi address → 4-bit `DIAG_DEV_ADDR` + 16-bit one-hot `MACH_ENABLE` decoder; routes each Pi address to the correct subsidiary controller
- **`ADGS5412_CONTROLLER`** — controls the ADGS5412 analog switch (BGO gain/function select) via SPI at 50 MHz (1/2 of CLK_100MHZ). 16-bit serial transaction: bit[0]=R/W* (always WRITE=0), bits[7:1]=internal register address `0b0000001` (switch control register), bits[15:8]=switch control data from Pi. State machine: IDLE→ASSERT_CS→ASSERT_DATA→SCK_ON↔SCK_OFF (×16 bits)→DEASSERT_CS→IDLE. Triggered by rising edge of `MACH_ENABLE`.
- **`SlopeBoxScanner`** — 4 MHz scanner for slope box ADC readout. `SCAN_CONTROL[1:0]`: 00=no action, 01=scan only, 10=command only, 11=scan+command. Command buffer FIFO (24-bit) holds Pi write commands; `Scanner2Arbiter_FIFO` (26-bit) crosses domains; `FIFO_FWFT_1Kx16` delivers results. `SLOPE_BOX_ID[7:0]` = ID read from ADC scan; `CYCLE_DELAY[15:0]` sets scan rate (counts of 4 MHz clock between cycles). `DOMAIN_CROSS_FIFO_EMPTY` flag. `DIAG_SCANNER_STATE[6:0]` = bits[6:3] scan state + bits[2:0] transaction state.
- **`RAM_BUFFER`** — dual-clock dual-port RAM with two operating modes:
  - **Histogram mode** (MODE=0): A-side writes from slope box scanner (4 MHz domain); B-side reads from Pi. `HISTO_STOP_VAL[15:0]` = histogram stop threshold. `USER_PRESCALE[7:0]` = writes to skip between samples. Stops when `BUFFER_FULL` (data count maxed or address maxed).
  - **Waveform recorder mode** (MODE=1): same dual-port structure but records time-ordered samples. `DATA_AVERAGE[15:0]` = average of all data (recorder mode) or index of peak (histogram mode).
  - Status outputs: `LAST_A_DATA[15:0]`, `MAX_A_VAL[15:0]`, `MIN_A_VAL[15:0]`, `DATA_COUNT[15:0]`, `BUFFER_ACTIVE`, `BUFFER_FULL`. `SHIFT_CTL[2:0]` controls write behavior.
- `LTC1660_controller.vhd` — DAC controller (LTC1660: 8-ch 10-bit DAC) ✅ verified 2026-04-16 — `LTC1660_controller.vhd:L127-134` (DAC A–H = 8 ch, 10-bit data `LATCHED_PI_DATA(9 downto 0)`)
- `I2C_STARTUP_ROM.vhd` / `Scanner_ROMs.vhd` — I2C startup ROM sequencer (Pickoff_Startup_ROM: 12-bit address, 4-bit data out) and scanner ROM tables
- `sync_capture_controller.vhd` / `sync_capture_counter.vhd` — synchronous capture logic (120 + 93 lines)
- `mult_count.vhd` — multiplied counter utility
- `SlopeBoxScan.vhd` (813 lines) — main slope box scanning implementation
- **FIFOs**: `Command_buffer_fifo` (24-bit, dual-clock), `Scanner2Arbiter_FIFO` (26-bit, dual-clock), `FIFO_FWFT_1Kx16` (16-bit FWFT 1K), `Single_bit_FIFO` (1-bit dual-clock)
- **ChipScope**: Multiple ILA variants (1024×17, 2048×17, 4096×17, 16384×17, 8192×34, 4K×89), VIO variants (7-bit addr, 1RW, 1GO, 16/24-bit in/out), ICON2/3

**SBX Interface Revision B** (`PickoffCard_SBX_Interface/Revision_B/Source/SlopeBoxInt_TopLevel_RevB.vhd`): ✅ verified 2026-04-19 — top-level entity + architecture signals read in full
- Target device: **Spartan-6 XC6SLX4-2TQG144** (same as Revision A), toolchain ISE 14.7, created 2018-06-22
- Firmware: `CODE_REVISION = 0x0011`, `CODE_DATE = 0x0414`
- Clocks: PLL generates 50/100/200 MHz + inverted 50 MHz, 10 MHz, and 4 MHz from TrigClk or free-running OSC. 4 MHz used as slope box scan clock (default cycle = 4,000,000 counts = 1 s)
- **New in Revision B vs prototype/Revision A:**
  - `BGOP_MUX_SDI/SDO/CLK/CS` — BGO Pattern analog mux now SPI-controlled (was parallel-select in prototype)
  - `FPGAPOWERCONVERTERSDA/SCL/EN/OE` — I2C interface to power converter board
  - `FPGA_PREAMPSCL/SDA`, `PREAMP_I2C_OE`, `PreampRstMon` — I2C interface to preamplifier + reset monitor input
  - `BufSerCommClk`, `BufSerDatToPickoff`, `FPGASERDATTOCOLLECTOR`, `BUFSYNCFROMCOLLECTOR`, `FPGACOLLECTOR_SPI_CE` — new DVI serial interface to BGO Pattern Collectors (replaces Pi-only path)
- **Track-and-hold control** (`TRACK_HOLD_CTL_REG`, added 2020-03-08): controls timing of external T&H switch during preamp resets; delay counter (`[7:0]`) + assert counter (`[7:0]`)
- **Pi SPI interface**: SPI0 (GPIO 7–11: CE1_N, CE0_N, MISO, MOSI, SCLK); GPIO 4 = Pi presence detect; all other GPIOs routed as `inout`
- **ChipScope**: ILA + VIO + ICON present for debug; VIO can substitute for Pi (address/data/RW/DO_TRANSACTION signals)
- Slope box scanner: scans 8 ADC channels via serial; result latched per channel; `SLOPE_BOX_DATA_FLAGS` (one-hot per channel); data buffered in dual-port RAM (A side = slope box writes, B side = Pi reads)
- Internal registers: `FPGA_CTL_REG` (0x0004 PU = ENABLE_SCAN on), `MISC_CTL_REG` (0x00FF PU), `BGOP_MUX_CTL_REG`, `TRACK_HOLD_CTL_REG` (0x80FF PU), `BUFFER_MODE_REG` (0x0021 PU)

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
- `knowledgeBase/collector_box_fpga.md` — ControlStripe + CtlFanout FPGAs (PSG SVN origin, sister codebase): per-stripe 48V relay/clock/SYNC control (Spartan-3) and RPi SPI gateway + ADS1158 ADC scanning (Spartan-6)
- `knowledgeBase/sbx.md` — Slope Box Extension hardware, BGO HV, GS_ID dongle
- `knowledgeBase/deep_fpga_SBX_CtrlFPGA.md` — Deep VHDL analysis of SBX Motherboard Control FPGA (entity SlopeBoxInt, Rev C, Spartan-6 XC6SLX9): 24-bit SPI, register file, I2C buses, BGO DDR outputs, analog switch control, preamp clamp, timestamp, firmware version registers

---

_Last reviewed: 2026-04-24_
