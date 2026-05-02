# collectorboxpi — Commissioning Utilities (Pre_EPICS_Collector)

Stability: C2 - Active / semi-stable

> 🔗 **Parent:** `collectorboxpi.md` — EPICS soft IOC overview | `sbx.md` — SBX hardware | `collector_fpga.md` — Stripe/Control FPGA firmware

**Source:** `DGS_tools_pack/collectorboxpi/Pre_EPICS_Collector/` and `Src/`  
**Last updated:** 2026-04-26

---

## Table of Contents

1. [Overview](#1-overview)
2. [Add_Remove_Detectors.sh](#2-add_remove_detectorssh-must-run-as-root)
3. [Pre_EPICS_Collector — Program Inventory](#3-pre_epics_collector--program-inventory)
4. [Src/ — C Library Details](#4-src--c-library-details)
   - [NonEPICS_SPI_lib.c](#41-nonepics_spi_libc)
   - [NonEPICS_Collector_lib.c](#42-nonepics_collector_libc)
   - [DPRAM_access.c](#43-dpram_accessc)
   - [Non_EPICS_Globvars.c](#44-non_epics_globvarsc)
5. [Programs/ — Key Program Internals](#5-programs--key-program-internals)
   - [TurnOnAllConnected.c](#51-turnonallconnectedc)
   - [Dump_Preamp_EEPROM.c](#52-dump_preamp_eepromc)
   - [Write_to_EEPROM.c](#53-write_to_eepromc)
   - [ALL_power_OFF.c](#54-all_power_offc)
   - [SPI_rw.c](#55-spi_rwc)
   - [Test_Port_Comms.c](#56-test_port_commsc)
   - [Dump_EEPROMs.c](#57-dump_eepromsck)
   - [Scan_Collector_FPGAs.c](#58-scan_collector_fpgasc)
   - [Write_to_DPRAM.c](#59-write_to_dpramc)
   - [Scan_DVI_Power.c / Scan_DVI.c](#510-scan_dvi_powerc--scan_dvic)
   - [spi_with_b_mbo_debug.c](#511-spi_with_b_mbo_debugc)
   - [Scan_DVI_Comms.c](#512-scan_dvi_commsc)
   - [Scan_DVI_Comms_No_Reg_Writes.c](#513-scan_dvi_comms_no_reg_writesc)
   - [Scan_DVI_Grounding.c](#514-scan_dvi_groundingc)
   - [Scan_DVI_Power_with_SBID.c](#515-scan_dvi_power_with_sbidc)
6. [Cross-References](#6-cross-references)

---

## 1. Overview

The `Pre_EPICS_Collector/` directory contains standalone C programs that run on the Raspberry Pi **before EPICS is active**. They communicate directly with Stripe and Control FPGAs via BCM2835 SPI. Must run as root.

**Normal 4-step commissioning sequence:**
1. `ALL_power_OFF` — cut all SBX power
2. `Scan_Collector_FPGAs` → `SCAN_OUTPUT_1_COLL_<BOX>.txt`
3. `Scan_DVI_Power` → `SCAN_OUTPUT_2_POWER_<BOX>.txt`
4. `Scan_DVI_Comms` → `SCAN_OUTPUT_3_COMM_<BOX>.txt` ← this feeds `GenerateCmdFile.py` (see [collectorboxpi.md](collectorboxpi.md) for IOC database generation)

---

## 2. `Add_Remove_Detectors.sh` (must run as root)

The top-level script in `collectorboxpi/` that manages the detector add/remove lifecycle:

> ⚠️ **Known bug (2026-04-18):** Line 11 sets `EPIC_dir=/share/EPICS` (missing trailing `d`) — the `cd $EPIC_dir` at L12 silently fails. The script continues because all subsequent paths are hardcoded absolute (`/shared/EPICS/Pre_EPICS_Collector`), so the actual scan and IOC restart steps still work correctly. The failing `cd` is a no-op bug. ✅ verified 2026-04-18 — `Add_Remove_Detectors.sh:L10-12` vs working paths at L24

Full sequence executed:
1. `systemctl stop softIOC.service` — stop the EPICS IOC
2. `cd /shared/EPICS/Pre_EPICS_Collector` (via hardcoded absolute path — the earlier `cd $EPIC_dir` fails silently)
3. `./ALL_power_OFF` — cut power to all SBXs on all stripes
4. `sleep 5` — allow power to fully discharge
5. `./Scan_DVI_Power` — scan 48V power state per DVI cable; exit code 0 = all OK, exit 148 (= C exit 404 mod 256) = cables not usable
6. `./Scan_DVI_Comms` — scan DVI communications; reads SBID from [pickoff card](pickoff_card_fpga.md) address; generates `SCAN_OUTPUT_3_COMM_<BOX>.txt`
7. `./GenerateCmdFile.py` — re-generate `st_20x.cmd` and `softIOC_<N>_settings.req` from scan output (see [collectorboxpi.md](collectorboxpi.md))
8. `systemctl start softIOC.service` — restart IOC with new configuration

Note: `Scan_Collector_FPGAs` is optional (commented out) — use when Stripe FPGA reachability is in question.

---

## 3. `Pre_EPICS_Collector/` — Program Inventory

**Full program inventory (15 executables):**

| Executable | Purpose |
|---|---|
| `ALL_power_OFF` | Turns off all power to all SBXs on all stripes |
| `Dump_EEPROMs` | Reads and dumps preamp EEPROM contents to screen |
| `Dump_Preamp_EEPROM` | Reads preamp and dongle EEPROM data for a specific cable |
| `Scan_Collector_FPGAs` | Tests communication with all Stripe FPGAs; generates SCAN_OUTPUT_1 |
| `Scan_DVI` | Scans DVI power with per-stripe enable; reads [Slope Box](sbx.md) ID and Dongle ID |
| `Scan_DVI_Comms` | Scans DVI comms; reads SBID from [pickoff card](pickoff_card_fpga.md) address; generates SCAN_OUTPUT_3 |
| `Scan_DVI_Comms_No_Reg_Writes` | Same as above but without writing to FPGA registers |
| `Scan_DVI_Grounding` | Scans DVI grounding / ground fault status |
| `Scan_DVI_Power` | Scans 48V power state per DVI cable; generates SCAN_OUTPUT_2 |
| `Scan_DVI_Power_with_SBID` | Same as above + reads [Slope Box](sbx.md) IDs |
| `Test_Port_Comms` | Interactive: select channel, enable power, do SPI I/O — for debugging |
| `TurnOnAllConnected` | Turns on power to all connected cables from a turn-on data file |
| `Write_to_DPRAM` | Reads a data file and writes it to the DPRAM of a specified SBX |
| `Write_to_EEPROM` | Reads address/data file and writes to 24AA002 EEPROM (byte or page mode) |
| `spi_with_b_mbo_debug` | Low-level SPI debug utility for verifying raw SPI transfers |
| `SPI_rw` | Interactive SPI read/write for testing arbitrary register access. Usage: `sudo ./SPI_rw <r|w> <devsel> <addr> <data>` (devsel 0–31, addr 0–1023, data 16-bit). See `collectorboxpi/Pre_EPICS_Collector/SPI_Address.md` for full register map and examples. ✅ verified 2026-04-18 — `SPI_Address.md:L1-16` |

**SPI architecture (brief):**
- Uses **SPI1** (Aux SPI), not SPI0; enabled via `dtoverlay=spi1-1cs` in `/boot/config.txt`
- 24-bit transactions: bit 23 = R/W, bits 22:16 = 7-bit register address, bits 15:0 = 16-bit data
- **5-bit DEVSEL bus** on GPIO 13/23/24/25/26 (binary device-select multiplexer, 0–31 devices)
- Banked addressing for FPGAs with >127 registers: bank# written to addr 127, then real address
- SpreadsheetSrc: FPGA register maps auto-generated from PSG spreadsheets (StrpFPGA, CtrlFPGA)

✅ verified 2026-04-17 — `collectorboxpi/Pre_EPICS_Collector/README.md` + `Add_Remove_Detectors.sh`

---

## 4. `Src/` — C Library Details

The Pre_EPICS_Collector programs are built from a shared C library (`Src/`).

### 4.1 `NonEPICS_SPI_lib.c`

(480 lines) ✅ verified 2026-04-23 — `wc -l NonEPICS_SPI_lib.c` = 480

Implements all SPI and DEVSEL GPIO operations for non-EPICS programs.

- `SPI1_setup(init_flag, RequestedSpeed)` — initializes BCM2835 SPI1 and DEVSEL GPIO outputs (GPIO 13/23/24/25/26 as binary 5-bit output bus). Drive strength set to 16 mA, slow slew for all connector GPIOs. Samples on rising edge of SCLK (changed 2023-02-17), data out on falling edge, SCLK starts low, MSbit first, 24-bit fixed transaction length. ✅ verified 2026-04-26 — `NonEPICS_SPI_lib.c:L67-71` (DEVSEL GPIOs), `L102` (`BCM2835_PAD_DRIVE_16mA`), `L131-137` (falling-edge clock-out, MSbit first, 24-bit via `_cntl0 |= 24`)
- `Set_DEVSEL(DEVSEL)` — asserts a 5-bit binary device select on the 5 GPIO lines. First clears all bits, then asserts the pattern for devices 0–31 via a switch/case with compile-time GPIO masks. 10 µs hold after assert (increased from 3 µs in 2022-12-16 debugging session). ✅ verified 2026-04-26 — `NonEPICS_SPI_lib.c:L247` (`usleep(10)` with comment "originally was usleep(3), increased 20221216")
- `Do_SPI1_transaction(RWflag, Bidx, UsrAddr, UsrData)` — single 24-bit SPI transaction. Sets DEVSEL, writes 3-byte SPI message (R/W | addr | data_hi | data_lo) to TX FIFO, polls BUSY, clears DEVSEL, returns 32-bit result (bits 23:16 = FPGA status, bits 15:0 = data). ✅ verified 2026-04-26 — `NonEPICS_SPI_lib.c:L287,L312` (bits 23:16 = status, bits 15:0 = data "by convention")
- `Do_Banked_SPI1_transaction(RWflag, Bidx, UsrAddr, UsrData)` — extends above for banked DPRAM: addr ≤127 = bank 0 (direct), addr 128–1023 = banks 1–7 (writes bank# to addr 127 first, then real addr, then resets bank to 0). Addr ≥1024 returns 0xFFFFFFFF (error). ✅ verified 2026-04-26 — `NonEPICS_SPI_lib.c:L325-356` (comments + `else if (UsrAddr < 1024)` branch; `0xFFFFFFFF` at L356)
- `Do_Banked_SPI1_BlockXfr(RWflag, Bidx, StartAddr, Nwords, *data_array)` — block transfer of Nwords from StartAddr; handles bank boundary crossings mid-transfer automatically; resets bank to 0 when done.
- `Do_Banked_SPI1_RMW(Bidx, UsrAddr, ANDmask, ORMask)` — read-modify-write: reads register, ANDs with ANDmask, ORs ORMask, writes back. Handles banked addresses. 16-entry helper macros: `SET_BIT_nn` / `CLEAR_BIT_nn` for each bit 0–15.
- `SPI1_exit()` — calls `bcm2835_aux_spi_end()` + `bcm2835_close()` (drops CE2 — only call at full shutdown). ✅ verified 2026-04-26 — `NonEPICS_SPI_lib.c:L473-477`

### 4.2 `NonEPICS_Collector_lib.c`

(477 lines) ✅ verified 2026-04-23 — `wc -l NonEPICS_Collector_lib.c` = 477

High-level collector box control functions, built on the SPI library above.

- **ADC scan loop** — The Control FPGA runs an ADS1158 ADC scanner ROM with 3 programs:
  - Program 0 (ROM addr 0): slow loop ~530 µs cycle time, all 16 channels + internal ADC values (REF/GAIN/TEMP/VCC/OFFSET)
  - Program 1 (ROM addr 64): fast loop ~50 µs, only 48V current monitor inputs (ADC ch 5–9)
  - Program 2 (ROM addr 128): fast loop ~50 µs, all channels
- `RESET_ADC_SCANNER(Program_Index)` — pauses scanner, selects program (writes ROM start address to CtrlFPGA), releases scanner.
- `PAUSE_ADC_SCANNER()` / `ENABLE_ADC_SCANNER()` — toggle GPIO `SCANNER_CONTROL_PIN` to halt/resume the ADC scan loop.
- `DO_ADC_CYCLE(delay1, delay2)` — enables scanner for `delay2` µs then stops it, giving one complete scan cycle.
- `COLLECT_AVERAGED_ADC_DATA(delay1, delay2, num_avg, enable_tracing)` — runs `num_avg` scan cycles, reads 128 words from DPRAM via block transfer, accumulates min/max/avg per address.
- **Relay control** — 4 relay types per cable (cables 1–30), each mapped to a bit in the corresponding stripe's `relay_control_sN` register (written to Stripe FPGA device 31):
  - `ENABLE/DISABLE_POWER(cable)` — bits 14:10 of relay_control_sx (prly = 48V power per cable)
  - `ENABLE/DISABLE_GNDFAULT_I(cable)` — bits 4:0 (irly = ground fault current injection)
  - `GROUND_DETECTOR(cable)` / `FLOAT_DETECTOR(cable)` — bits 9:5 (grly = ground fault check relay; GROUND = normal, FLOAT = lifted)
- `INITIALIZE_ALL_RELAYS()` — zeros relay_control for all 6 stripes (power off, no GFI, no float), then writes 0x0100 to stripe_control_sN for all 6 stripes (sets CRLY on one pole, clears clock/sync enable). Also sets GPIO J8_03 HIGH and J8_05 LOW as status markers.
- `ENABLE_ALL_COMMS()` — sets bits 4:0 of stripe_control_sN for all 6 stripes (enables clock and sync per cable).
- `convert_ADC_temp(ADCVAL, F_or_C)` — converts raw ADS1158 temperature reading to °C or °F: V = (ADC/30720)×4.096V; T = ((V_µV − 160000)/563) + 25.0°C. ✅ verified 2026-04-23 — `NonEPICS_Collector_lib.c:L459,L467,L470` (formula confirmed; code comment at L464 has typo `168000` but actual computation uses `160000`)

### 4.3 `DPRAM_access.c`

(172 lines) ✅ verified 2026-04-23 — `wc -l DPRAM_access.c` = 172

Lookup tables (arrays of DPRAM register addresses) used by commissioning utilities.

- `FPGA_VOLTAGE_ADDR[8][3]` — DPRAM addresses for stripe power supply voltages (12V/25V/33V) for stripes 1–6 plus BGO FPGA. Index 0 unused.
- `FPGA_IMON_ADDR[31]` — per-cable 48V current monitor DPRAM addresses, indexed by cable number 1–30 (index 0 unused).
- `FPGA_GFI_ADDR[31]` — per-cable ground fault injection status DPRAM addresses.
- `ADC_OFFSET_ADDR[7]`, `ADC_VCC_ADDR[7]`, `ADC_TEMP_ADDR[7]`, `ADC_GAIN_ADDR[7]`, `ADC_REF_ADDR[7]` — per-stripe DPRAM addresses for ADS1158 internal ADC self-calibration values (stripes 1–6, index 0 unused).

### 4.4 `Non_EPICS_Globvars.c`

(572 lines) ✅ verified 2026-04-26 — source-read 2026-04-26

Defines all shared global arrays used by Pre_EPICS_Collector programs.

- `DPRAM_IMAGE[1024]`, `DPRAM_IMAGE2[1024]`, `DPRAM_IMAGE3[1024]` — 16-bit word images of one complete DPRAM bank readback.
- `DATA_MIN[128]`, `DATA_MAX[128]`, `DATA_AVG[128]` — accumulator arrays for multi-cycle ADC averaging (used by `COLLECT_AVERAGED_ADC_DATA`).
- **`ADC_CONVERSION[128]`** — per-DPRAM-address floating-point conversion factor from raw ADS1158 counts to engineering units:
  - Normal voltage channels: `0.000133333` (V = ADC/30720 × 4.096)
  - Ground-fault current (GFI) channels: `0.16161616` µA/count (165 Ω sense resistor, ×5 instrumentation amp; I(µA) = ADC × 0.16161616; R_det(kΩ) = 15468.75/ADC)
  - 48V current channels: `0.33333325` mA/count (Hall-effect sensor, 400 mV/A, differential with hardware zero-current offset)
  - ADC VCC/reference: `0.000325520833` V/count; gain: `0.0000325520833` V/V/count; temperature: special flag `−2.0` → use `convert_ADC_temp()`; unused addresses: `−1.0`
  - DPRAM addresses 128–255 (6 stripes × 21 entries each + BGO FPGA): stripes 1,3,5 at addr 128–191, 2,4 at 192–233, 6 at 234–254
- **`ADC_NAMES[128][50]`** — string label per DPRAM address (e.g., `"S1_DVI1_GndFault_I"`, `"S3_ADC_TEMP"`, `"BGO_FPGA_33V"`).
- **`ADC_UNITS[128][50]`** — engineering unit string per address (`"uA"`, `"mA"`, `"V"`, `"deg C"`, `"V/V"`, `"cnt"`, `""`).
- **`ADC_MIN_MAX[128][3]`** — expected value ranges per address: `{negligible_floor, min_expected, max_expected}`. GFI: 0.1–10.0 µA; 48V current: 10–750 mA; rail voltages: 1.1–1.3 V (1.2V rail), 2.3–2.7 V (2.5V rail), 3.1–3.5 V (3.3V rail); ADC temp: 21–35 °C; gain: 0.98–1.02 V/V; ref: 4.08–4.10 V.

---

## 5. `Programs/` — Key Program Internals

### 5.1 `TurnOnAllConnected.c`

(613 lines) — turns on/off power per cable using the `SCAN_OUTPUT_2_POWER_<BOX>.txt` power scan file.

- Input file format (TSV, 30 data rows + 1 header): `cable_idx\tFPGAOK\tFPGA_populated\tCable_usable\tmeasured_current`
  - SCAN_OUTPUT_2 columns: `cable_idx` / `FPGAOK` / `FPGA_populated` / `Cable_usable` / `measured_current` ✅ verified 2026-04-26 — `TurnOnAllConnected.c:L493-519` (file read loop; 5-column TSV header confirmed)
- For each cable 1–30: if `Cable_usable==0` → `DoPowerOff(cable)`, else `DoPowerOn(cable)` ✅ verified 2026-04-26 — `TurnOnAllConnected.c:L541-544`
- `DoPowerOn(cable)` / `DoPowerOff(cable)`: writes to `StripeRegisters.relay_control_sN` (device 31, `Do_Banked_SPI1_RMW`). Bit assignments: cable 1/6/11/16/21/26 = bit 10, cable 2/7/12/17/22/27 = bit 11, …, cable 5/10/15/20/25/30 = bit 14 of the appropriate stripe register. ✅ verified 2026-04-26 — `TurnOnAllConnected.c:L52-85` (relay_control bit table), `L40-90` (DoPowerOn switch table)
- Also manipulates tristate control (bit per cable in `stripe_control_sN` bits [4:0]) and clock/sync enable (bits [13:12]) during power-on sequence.

### 5.2 `Dump_Preamp_EEPROM.c`

(305 lines) — reads and dumps preamp + dongle EEPROM data from SBX DPRAM for a specified cable.

- **Preamp EEPROM DPRAM bank:** base address 128 (bank 1); reads 128 words. Preamp data: 8 pages × 8 words starting at addr 2 (relative to bank start). Page 'F' (0x0F) = addr 66 onward (8 words). "Dig Pot EEPROM" at addr 82 (8 words).
- **Dongle EEPROM DPRAM bank:** base address 512 (bank 4); reads 128 words. Dongle data: same 8-page × 8-word layout starting at addr 2.
- **Dongle ID:** read from addr 641 (single word, separate `Do_Banked_SPI1_BlockXfr` call). ✅ verified 2026-04-26 — `Dump_Preamp_EEPROM.c:L244`
- Sanity check: DPRAM_IMAGE[0] (first word of bank) must be nonzero (= bank number); if zero, no SBX is present at that cable. Falls back to reading `addrbase+126,127` (code date/rev registers) for error reporting.
- Output: also writes `DPRAM_DATA.txt` with full 128-word bank dump in `AAAA: XXXX` format.
- ✅ verified 2026-04-26 — `Dump_Preamp_EEPROM.c:L143-250` (addrbase=128 bank1, addrbase=512 bank4, dongle ID at addr 641, pot page addr=82)

### 5.3 `Write_to_EEPROM.c`

(314 lines) — **partially implemented**. The file header describes the intended write API:

- Input file format (one operation per line):
  - `0,<addr>,<data>` — byte write to preamp EEPROM
  - `1,<addr>,<data×8>` — page write to preamp EEPROM (addr must be 8-byte aligned)
  - `2,<addr>,<data>` — byte write to dongle EEPROM
  - `3,<addr>,<data×8>` — page write to dongle EEPROM (addr must be 8-byte aligned)
- In page mode: validates page boundary (`addr % 8 == 0`); 8 data bytes follow addr.
- ⚠️ **Current implementation (as of 2026-04-26) does NOT implement the write path.** The main() function (ends at L260) only reads and dumps EEPROM data — identical in structure to `Dump_Preamp_EEPROM.c`. No `fscanf`/data file parsing, no `Do_Banked_SPI1_transaction()` write calls, no byte/page write logic is present. The write functionality described in the header is aspirational/unimplemented.
- ✅ verified 2026-04-26 — `Write_to_EEPROM.c:L60-260` (main() inspected; only SPI reads via Do_Banked_SPI1_BlockXfr; no write calls, no input file parsing)

### 5.4 `ALL_power_OFF.c`

(143 lines) — Emergency/diagnostic utility: turns off all power and clock/sync to all SBXs across all 6 stripes.

- Pauses ADC scanner (`PAUSE_ADC_SCANNER()` + `usleep(10)`) before any SPI activity.
- **Three register groups cleared for each of 6 stripes (device 31):**
  - `relay_control_sN` (addrs 64/72/80/88/96/104): bits 14:10 cleared via RMW mask 0x83FF/0x0000 → turns off all 5 PRLY relays per stripe.
  - `stripe_control_sN` (addrs 65/73/81/89/97/105): bits 13,12,4:0 cleared via mask 0xCFE0/0x0000 → disables clock_output_enable, clock_source, and clock/sync enable to all 5 SBX slots.
  - `tristate_control_sN` (addrs 66/74/82/90/98/106): bits 9:0 cleared via mask 0xFC00/0x0000 → tristates SPI and clock/sync control lines to all 5 SBX slots.
- Also defines `Set_Diag_GPIOS()`, `Init_Diag_GPIOS()`, `Init_ADC_Control()` utility helpers (same boilerplate as other programs).
- **Companion to `TurnOnAllConnected.c`** — restores all-off state without reading a scan file.
✅ verified 2026-04-26 — `ALL_power_OFF.c:L68-91` (PAUSE_ADC_SCANNER+usleep at L68-69; relay_control RMW 0x83FF/0x0000 at L72-77; stripe_control 0xCFE0/0x0000 at L79-84; tristate_control 0xFC00/0x0000 at L86-91)

### 5.5 `SPI_rw.c`

(160 lines) — Command-line SPI read/write diagnostic utility. Performs a single banked SPI transaction and prints the result. Most useful for manually probing FPGA registers before/after an EPICS PV write.

**Usage:** `sudo ./SPI_rw <r|w> <devsel> <addr> [data]`

| Argument | Range | Notes |
|----------|-------|-------|
| `r`\|`w` | — | Read or write flag |
| `devsel` | 0–31 | Device-select index (FPGA/DVI via GPIO DEVSEL bus) |
| `addr` | 0–1023 | Register address, decimal or 0x hex (banked) |
| `data` | 0–0xFFFF | 16-bit value; required for writes, optional for reads |

**Exit codes:** 0 = success, 1 = arg/init error.

**Output format:**
- Read: `READ   DEV=05 ADDR=0x0010  DATA=0xABCD  STATUS=0x02  RAW=0x02ABCD`
- Write: `WRITE  DEV=05 ADDR=0x0010  WROTE=0xABCD  STATUS=0x02  DATA_BACK=0xABCD  RAW=0x02ABCD`

**Return value anatomy:** `Do_Banked_SPI1_transaction()` returns 24-bit value: bits[23:16] = FPGA status byte, bits[15:0] = data word.

**Build:** `make SPI_rw` from `Pre_EPICS_Collector/` directory.
✅ verified 2026-04-26 — `SPI_rw.c:L9-31,L72-154` (usage/arg parsing confirmed; READ printf at L148 includes STATUS field; WRITE printf at L153 confirmed; 24-bit return anatomy at L141-143)

### 5.6 `Test_Port_Comms.c`

(497 lines) — Interactive diagnostic REPL for testing SPI communication to a specific DVI cable (SBX port). Initializes SPI, then enters an input loop accepting manual transactions.

**Interactive command format:** `X BB YY ZZZZ`
- `X` = operation code (see below)
- `BB` = DEVSEL (cable 1–30 for SBX, 0 or 31 for local collector FPGA)
- `YY` = address (decimal)
- `ZZZZ` = data value (hex)

**Operation codes:**

| Code | Meaning |
|------|---------|
| 0 | Write with full cable setup (power on, tristate, stripe_control) |
| 1 | Read with full cable setup |
| 2 | Write without setup (raw transaction) |
| 3 | Read without setup (raw transaction) |
| 5 | Set SPI speed (sets SPI clock speed register) |
| -1 | Assert GPIO 12 HIGH |
| -2 | Assert GPIO 12 LOW |
| -9 | Exit program |

**`DoTestTransaction()` setup sequence (for op 0 or 1):**
1. **Power relay:** writes `relay_control_sN` (device 31) — sets bit 10+N for cable N within stripe, enabling PRLY relay.
2. **Sleep 10s** after power-on.
3. **Tristate release:** writes `tristate_control_sN` — bit pair (SPI tristate + clock tristate) for cable's slot within stripe. Bit assignments: cable N in stripe → bits `slot` and `slot+5` (SPI[4:0] + clock[9:5]).
4. **Stripe control:** writes `stripe_control_sN` — sets clock_and_sync_enable bit for cable's slot (bits [4:0]) + crly_earth bit (bit 8). CRLY signal bit (bit 9) is cleared to open the signal-ground relay during comms test.
5. **User transaction:** calls `Do_Banked_SPI1_transaction()` with user's op, cable, addr, data.

**Helper:** `TestRlyCtlreg(cbl)` — reads and prints the relay_control register for a given cable (maps cable 1–30 → correct stripe register address).

**Also includes:** `Set_Diag_GPIOS()`, `Init_Diag_GPIOS()`, `Init_ADC_Control()` boilerplate.
✅ verified 2026-04-26 — `Test_Port_Comms.c:L126-300` (sleep(10) at L180 after relay write; tristate bits slot/slot+5 at L202-233 (pairs 0/5, 1/6, 2/7, 3/8, 4/9 confirmed); stripe_control ORmask bit8=crly_earth + bit(slot-1)=clock_and_sync_enable at L265-295; crly_signal bit9 cleared by ANDmask=0x0000)

### 5.7 `Dump_EEPROMs.c`

(306 lines) — Combined preamp + dongle EEPROM reader; slightly different from `Dump_Preamp_EEPROM.c` in that it:

- Accepts cable number as interactive prompt (`scanf`) or command-line argument.
- Optionally **turns on all power** (6 stripes, all relay bits 14:10 set + stripe_control + tristate) if user answers 'Y' — includes 5s sleep after power-on.
- Reads **bank 1 (preamp):** base address 128, 128 words. Displays 8 pages × 8 words (addr 2–65) + page F (addr 66–73) + dig-pot EEPROM (addr 82–89).
- Reads **bank 4 (dongle):** base address 512, 128 words. Displays same 8-page × page-F layout.
- Reads **dongle ID** from addr 641 (same as `Dump_Preamp_EEPROM.c`).
- Writes output to `DPRAM_DATA.txt` using format `0:NNNN:XXXX:MMMM` (bank 0 for preamp) and `2:NNNN:XXXX:MMMM` (bank 2 for dongle, despite being bank 4 in hardware).
- **Sanity check:** DPRAM_IMAGE[0] must be nonzero (= bank number); reads addrs 126+127 for code date/rev if check fails.
- Key difference from `Dump_Preamp_EEPROM.c`: this program has the interactive power-on option and combined preamp+dongle output in one pass.
✅ verified 2026-04-26 — `Dump_EEPROMs.c:L99-238` (scanf cable at L99, sscanf argv at L104, Y/N power-on prompt at L116, sleep(5) at L141, bank-1 prefix `0:` at L171, bank-4 output labeled "DUMP OF BANK 4" but prefix `2:` at L238, sanity check at L150-154 with addrs 126+127 fallback)

### 5.8 `Scan_Collector_FPGAs.c`

(330 lines) — Communication health check for all 6 Stripe FPGAs in a collector box. Runs a 3-pass SPI verification suite and outputs a per-cable validity file used by downstream commissioning tools.

**Purpose:** Verify that every Stripe FPGA responds correctly over SPI before running `TurnOnAllConnected.c`.

**Initialization sequence:**
1. `SPI1_setup()` — initialise SPI at `SPI_NOMINAL_SPEED` (speed=30, ~4.1 MHz).
2. `Init_ADC_Control()` — set GPIO 12 as output HIGH (ADC scanner held in reset).
3. `Init_Diag_GPIOS()` — GPIO 2 and 3 set as outputs LOW (diagnostic marker pins).
4. `INITIALIZE_ALL_RELAYS()` — all power off, all ground-fault relays reset.
5. `RESET_ADC_SCANNER(0)` + 10 µs delay — ensure ADC scanner stopped.
6. `Set_Diag_GPIOS(1)` — set GPIO2 HIGH to mark relay init done.

**Pass 1 — Control FPGA block-read consistency:** Reads 128 words from bank 0 three times (`Do_Banked_SPI1_BlockXfr`) and checks that all three reads agree word-for-word. 10 repetitions; any mismatch is a fatal error.

**Pass 2 — Per-stripe code_revision register:** For each of the 6 Stripe FPGAs (DEVSEL=31, stripe-specific `code_revision_sN` address), reads the register 20 times and checks for consistency and non-0/non-0xFFFFFFFF values. Any failure marks all 5 cables of that stripe as `cable_valid[]=0`.

**Pass 3 — LED control / scratchpad read-write pattern test:** For each of the 6 stripes, writes 6 test patterns (`0x000, 0x5555, 0xAAAA, 0x3C3C, 0xC3C3, 0x0000`) to the LED control or sandbox register and verifies the SPI return-data (previous write's readback) matches expected. Failure marks all 5 cables of that stripe invalid.

**Output file:** `SCAN_OUTPUT_1_COLL_<BOX>.txt` — 2-column TSV (`Cable`, `FPGAOK`) with rows 1–30. `FPGAOK=1` = cable valid; `FPGAOK=0` = stripe FPGA failed, cable unusable.

This file is consumed by `TurnOnAllConnected.c` (which reads `SCAN_OUTPUT_1_COLL_<BOX>.txt` to skip cables on failed stripes).
✅ verified 2026-04-26 — `Scan_Collector_FPGAs.c:L128-153` (Pass 1: 10-rep loop `i<10`, 3× BlockXfr of 128 words bank 0, word-for-word comparison, fatal on error); `L158-204` (Pass 2: 20-read loop `j<20`, `code_revision_sN` addr per-stripe, 0/0xFFFFFFFF guard, cable_valid[]=0 on fail); `L210-249` (Pass 3: TEST_PATTERN[6]={0x000,0x5555,0xAAAA,0x3C3C,0xC3C3,0x0000} pattern write+readback, 5-comparison loop); `L255-257` (output file `SCAN_OUTPUT_1_COLL_%s.txt`, TSV header `Cable\tFPGAOK`)

### 5.9 `Write_to_DPRAM.c`

(225 lines) — Selectively writes data from a file into a specific cable's DPRAM (SBX preamp configuration RAM), only updating addresses whose current value differs from the file.

**Usage:** `sudo ./Write_to_DPRAM [cable_number]`  
If no argument given, prompts for cable number interactively.

**Input file:** `DPRAM_DATA.txt` — line format `NNNN: XXXX` (decimal address : hex value).

**Sequence:**
1. `SPI1_setup()` + `PAUSE_ADC_SCANNER()` — init SPI and halt ADC scanner.
2. Prompt for cable number (1–30) or read from argv.
3. Optionally turn on all power (6 stripes, relay/stripe_control/tristate RMW writes, with `sleep(5)`) if user answers 'Y'.
4. Read entire 1024-word DPRAM of target cable into `DPRAM_IMAGE[]` (8 × 128-word block reads via `Do_Banked_SPI1_BlockXfr`).
5. Parse `DPRAM_DATA.txt` line-by-line; for each addr/value pair where `dataval != DPRAM_IMAGE[addr]`, issue a `Do_Banked_SPI1_transaction(SPI_WRITE, cable, addr, dataval)` to update only changed words.

**Note:** Write is update-on-diff only — unchanged addresses are not re-written, minimising SPI bus traffic.
✅ verified 2026-04-26 — `Write_to_DPRAM.c:L67` (DPRAM_IMAGE[1024]); `L146-149` (nwords=128, loop `addrbase=0;addrbase<1000;addrbase+=128` → 8 iterations × 128 words = 1024 total, BlockXfr each); `L156-164` (fopen DPRAM_DATA.txt, update-on-diff: `if (dataval != DPRAM_IMAGE[addr])` then `Do_Banked_SPI1_transaction(SPI_WRITE,...)`)

### 5.10 `Scan_DVI_Power.c` / `Scan_DVI.c`

Two related DVI cable scan utilities (616L and 714L respectively).

- **`Scan_DVI_Power.c`** (616 lines) — scans all accessible DVI cables (SBX ports), measures ADC power-rail voltages for each, and writes results. Also referenced from `collector_fpga.md` for its ADC calibration constants (the `GLBL_ADC_CONVERSION[]` array). Uses `Scan_DVI_Power.c`-local ADC readout loop to verify 48V rail, signal currents, and SBX board voltages for each cable.

- **`Scan_DVI.c`** (714 lines) — full DVI scan including communication tests (similar to `Scan_Collector_FPGAs.c` per-cable) plus power measurements. Reads EEPROM bank 1 (preamp) and bank 4 (dongle) for each accessible cable. Writes results to `SCAN_OUTPUT_2_POWER_<BOX>.txt` (the same 5-column TSV consumed by `TurnOnAllConnected.c`: `cable_idx / FPGAOK / FPGA_populated / Cable_usable / measured_current`).

**Note:** `Scan_DVI_Comms.c`, `Scan_DVI_Comms_No_Reg_Writes.c`, `Scan_DVI_Grounding.c`, and `Scan_DVI_Power_with_SBID.c` are additional scan utility variants with narrower scope (comms-only, no register writes, grounding check, and SBID readout respectively). All share the same SPI infrastructure.

### 5.11 `spi_with_b_mbo_debug.c`

(129 lines) — Minimal interactive SPI REPL used during early development/debugging by M. Oberling. Equivalent to `Test_Port_Comms.c` section 5.6 but simpler: uses `Do_SPI1_transaction()` (unbankable, without cable setup sequence) rather than `Do_Banked_SPI1_transaction()`.

**Command format:** `X BB YY ZZZZ`
- `X`: 0=write, 1=read, >2=set SPI speed, -1=GPIO12 HIGH, -2=GPIO12 LOW, -9=exit
- `BB`: DEVSEL (bank index)
- `YY`: address (decimal)
- `ZZZZ`: data (hex)

**Key difference from `Test_Port_Comms.c`:** No power-on/tristate/stripe-control cable setup — pure raw SPI transaction. Intended for low-level debug when you already know the hardware is powered and the SPI path is live.

**Build:** `gcc -o spi_with_b_mbo_debug spi_with_b_mbo_debug.c -l bcm2835`
✅ verified 2026-04-26 — `spi_with_b_mbo_debug.c:L71-72` (prompt text confirms `X BB YY ZZZZ` format; X: 0=write, 1=read, >2=speedset, -1=GPIO set, -2=GPIO clear, -9=exit; BB=DEVSEL, YY=address decimal, ZZZZ=hex); `L102` (`Do_SPI1_transaction()` — unbankable, no cable setup sequence)

---

### 5.12 `Scan_DVI_Comms.c`

(706 lines) — **Full DVI communications scan.** The 4th step in the normal commissioning sequence. Reads `SCAN_OUTPUT_2_POWER_<BOX>.txt` (produced by `Scan_DVI_Power`) to determine which cables are marked usable, then for each usable cable:

1. **Isolation setup:** Sets all 6 stripe `stripe_control_sx` registers to `0x011F` — enables clock and sync (bits 4:0), closes CRLY earth relay (bit 8) to tie the SBX comms return to GND_SIGNAL. Sets all `relay_control_sx` to 0x0000 (all PRLY power off, GRLY closed, IRLY off).
2. **Power-on loop:** For each cable 1–30 (skipping unusable), enables only that cable's PRLY bit in its stripe relay register (RMW via `Do_Banked_SPI1_RMW`), and enables the cable's SPI tristate + clock/sync tristate bits (bit pair `N%5` and `5+(N%5)` within each stripe's `tristate_control_sx` register). Sleeps 5 seconds to allow power to stabilize.
3. **Comms test:**
   - Reads addr 126 (`CODE_DATE`) and addr 127 (`CODE_REVISION`) from the SBX DPRAM. If either is 0 or 0xFFFF → `COMMS_OK=0`. Also reads nearby cables ±4 for diagnostic cross-talk information.
   - Reads addr 28 (`SBOX_ID`, bits 6:0). Valid range 1–125.
   - Reads addr 641 (`DONGLE_ID`). Valid range 1–120.
   - Block-reads DPRAM bank 128 (128 words) = preamp EEPROM data. Validates by checking: count of 0x0000 < 100 AND count of 0xFFFF < 100. If valid, parses page metadata: `PAGE_ID[i]` = bits[11:8] of word `2+8*i`, `PAGE_REV[i]` = bits[15:12]. Page 0 (rev 0 or 1) encodes: `GE_ID` (DPRAM_BANK_DATA[4] low byte), `GE_PREFIX_ID` (high byte), `GE_HV_NAMEPLATE` (DPRAM_BANK_DATA[5] low byte × 25V), `GE_HV_OPERATING` (DPRAM_BANK_DATA[6] high byte × 25V), `GE_TYPE` (DPRAM_BANK_DATA[5] high byte).
   - Block-reads DPRAM bank 512 (128 words) = dongle EEPROM. Same 0-count/FFFF-count validity check.
4. **Output file:** `SCAN_OUTPUT_3_COMM_<BOX>.txt` — 17-column TSV with all original POWER columns plus: `COMM_OK`, `SBOX_OK`, `SBOX_ID`, `DNG_OK`, `DNG_ID`, `PA_EE_OK`, `DNG_EE_OK`, `GE_HV_NAMEPLATE`, `GE_HV_OPERATING`, `GE_PREFIX`, `GE_ID`, `GE_TYPE`.

**This output is the primary input to `GenerateCmdFile.py`** (the EPICS IOC database generator — see [collectorboxpi.md](collectorboxpi.md)).

**SPI speed:** Hard-coded `SPI1_setup(1, 249)` — very slow (~500 kHz) for cable scan reliability over long DVI harnesses.

**GE_PREFIX_ID encoding:** 0=None, 1=GS, 2=E, 3=ANL, 4=T (matches the detector naming scheme on GS labels).

---

### 5.13 `Scan_DVI_Comms_No_Reg_Writes.c`

(373 lines) — **Same as `Scan_DVI_Comms.c` but without writing to stripe/relay FPGA registers.** Intended for diagnostic re-runs when the hardware is already in a known powered state. The per-cable power-on and tristate setup (steps 1–2 above) are omitted; the program jumps directly to the comms-test read loop (step 3). Produces the same `SCAN_OUTPUT_3_COMM_<BOX>.txt` output format.

**Use case:** Re-running the comms check without disturbing currently-powered SBXs, e.g., to re-read EEPROM contents after a warm reboot without going through the full power cycle.

---

### 5.14 `Scan_DVI_Grounding.c`

(601 lines) — **Ground fault / leakage current diagnostic scanner.** Does **not** follow the normal commissioning sequence; standalone diagnostic tool. Runs a multi-pass sweep cycling through all relay combinations (PRLY/GRLY/IRLY states) to systematically locate ground faults and leakage paths in the DVI cable harness.

**5-pass test strategy:**
| Pass | Configuration | Purpose |
|------|--------------|----------|
| 0 | All PRLY open, all GRLY open, IRLY sequentially on | Baseline injection, no supply rail |
| 1 | All PRLY open, all GRLY open, all IRLY off | Floating baseline check |
| 2 | All PRLY open, all GRLY closed, all IRLY off | Check for leakage with comms return tied |
| 3 | All PRLY open, IRLY sequentially on, GRLY open in matching stripe | Injection with matching ground reference open |
| 4+ | All PRLY open, IRLY sequentially on, GRLY closed in matching stripe | Injection with matching GRLY closed |

**Output file:** `GNDFAULT.TXT` — per-pass, per-cable current readings with relay state annotations. Does not produce a numbered `SCAN_OUTPUT_N` file; intended for manual review by a hardware engineer.

**Uses GPIO 2/3 as diagnostic pins** (via `Init_Diag_GPIOS()` — normally I2C bus repurposed as indicators during pre-EPICS operation).

---

### 5.15 `Scan_DVI_Power_with_SBID.c`

(594 lines) — **Extended power scan that also reads [Slope Box](sbx.md) IDs and Dongle IDs.** Functionally a superset of `Scan_DVI_Power.c`: performs the same per-cable power-on, current measurement, and `Cable_usable` determination, **plus** for each populated cable reads:
- `SBOX_ID` from SBX DPRAM address 28 (bits 7:0) via `Do_Banked_SPI1_transaction(SPI_READ, cable, 28, 0)`
- `DONGLE_ID` from DPRAM address 641

**Output file:** `POWER_SCAN.txt` (not `SCAN_OUTPUT_2_POWER_<BOX>.txt`). Uses a fixed filename (not box-environment-qualified), so this is not a drop-in replacement for `Scan_DVI_Power.c` in the standard commissioning sequence. The TSV format is the same 5-column base as `SCAN_OUTPUT_2` plus two SBID/DONGLE columns.

**Also calls `GROUND_DETECTOR(cable)`** (a grounding check helper) before current measurement — providing a combined power+grounding diagnostic in a single pass. This is the most feature-rich of the power-scan variants but not part of the standard automated sequence.

---

## 6. Cross-References

- [collectorboxpi.md](collectorboxpi.md) — Parent: EPICS soft IOC overview, database templates, HV control, PXE boot
- [collector_fpga.md](collector_fpga.md) — CtrlFPGA + StripeFPGA firmware (the hardware these programs talk to)
- [collector_ctrlFPGA_registers.md](collector_ctrlFPGA_registers.md) — CtrlFPGA register map (141 registers): pulsed_control mask, FPGA_CTL_REG bits, ADC monitoring layout — the register space these Pre_EPICS programs access via DPRAM bank 1
- [collector_box_fpga.md](collector_box_fpga.md) — ControlStripe + CtlFanout FPGAs (PSG SVN origin)
- [collectorbox_devicesupport.md](collectorbox_devicesupport.md) — EPICS device support: SPI driver, CAMAC_IO link (Pi IOC side)
- [sbx.md](sbx.md) — SBX hardware: GS_ID dongle, BGO HV, pickoff card; dongle ID format

*Created: 2026-04-26 | Split from collectorboxpi.md | Last reviewed: 2026-04-27*
