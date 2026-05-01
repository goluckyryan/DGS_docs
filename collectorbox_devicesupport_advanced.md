# CollectorBox Device Support — Advanced DTYP Modules

Stability: C2 - Active / semi-stable

_Split from `collectorbox_devicesupport.md` on 2026-04-26. Source: `DGS_tools_pack/collectorboxpi/CollectorBox_RevA/CollectorApp/src/`_

**See also:** [`collectorbox_devicesupport.md`](collectorbox_devicesupport.md) — Core architecture, SPI flows, global data structure, conversion coefficients

---

## Table of Contents

- [I2C Device Support (CollectorI2C_AI/AO.c)](#i2c-device-support-collectori2c_aiaoc)
- [CollectorI2C — I2C Bridge Device Support](#collectori2c--i2c-bridge-device-support)
- [CollectorStep — Closed-Loop HV Stepping](#collectorstep--closed-loop-hv-stepping)
- [CollectorDPRSupport — Dual-Port RAM (DPRAM) Read](#collectordprsupport--dual-port-ram-dpram-read)
- [CollectorCalc AI — Multi-Mode Analog Input Calculation](#collectorcalc-ai--multi-mode-analog-input-calculation)
- [CollectorCalc BI — Mailbox Comparison / Interlock Logic](#collectorcalc-bi--mailbox-comparison--interlock-logic)
- [CollectorCtl_Waveform — Global Data Array Readback](#collectorcrl_waveform--global-data-array-readback)
- [CollectorADC — Direct SPI ADC Read](#collectoradc--direct-spi-adc-read)
- [CollectorSoftControl — RAM Mailbox DTYP (AI/BI/BO/MBBI)](#collectorsoftcontrol--ram-mailbox-dtyp-aibibombbbi)
- [DEVSEL Bus Driver (DEVSEL_bus.c)](#devsel-bus-driver-devsel_busc)
- [ScanADCs.c — ADC Scanning Placeholder](#scanadc-scanning-placeholder)
- [Cross-References](#cross-references)

---

## I2C Device Support (`CollectorI2C_AI/AO.c`)

_Source: `CollectorBox_RevA/CollectorApp/src/CollectorI2C_AI.c` and `CollectorI2C_AO.c`. ✅ verified 2026-04-19 against source._

The I2C handlers implement DTYP `"CollectorI2CSerial"` — a separate device support path (registered alongside `"Collector Local Serial"` in `CollectorSupport.dbd`) for controlling I2C peripherals **via the Collector FPGA's I2C command FIFO** rather than direct SPI register writes.

### How It Differs from Normal SPI Device Support

Normal `CollectorLocalSerial` PVs write/read FPGA registers directly with single 24-bit SPI transactions. I2C PVs are different: the EPICS device support writes **I2C command words into an FPGA-side command FIFO**, and the Collector FPGA's I2C state machine processes those commands autonomously.

### camacio Field Encoding (I2C AO)

The `camacio` link fields carry more meaning for I2C than for standard SPI:

| camacio field | I2C meaning |
|---|---|
| `B` (bits 4:0) | DEVSEL index — which device on the SPI bus (0–31) |
| `N` (bits 6:0) | Address of the I2C command FIFO in the Collector FPGA |
| `N` (bits 14:8) | Address of the I2C scan control register |
| `A` (bits 6:0) | 7-bit I2C device address (shifted left 1 to make room for R/W bit) |
| `A` (bits 11:8) | Bit index in scan control register to turn off scanning during write |
| `A` (bits 15:12) | Bit index in control reg that triggers FIFO processing ("send FIFO") |
| `F` (bits 7:0) | I2C register address within the target device |
| `F` (bits 15:8) | Address of the FIFO-process-control register in Collector FPGA |
| `C` (bits 7:0) | Transfer type (see below) |
| `C` (bits 15:8) | Second I2C register address (for type 0x50 dual-register writes) |

### I2C FIFO Command Word Format

Each 16-bit word pushed into the FPGA command FIFO:

```
Bit 15  : DONE — ACK4_CTL[1]: combined with RPTS to control end-of-byte action
Bit 14  : RPTS — ACK4_CTL[0]:
            00 = ACK, continue with next FIFO word
            01 = ACK, Repeated Start, continue
            10 = ACK, then STOP
            11 = STOP without ACK
Bit 13  : NACK — if 1, skip ACK check on 9th clock
Bit 12  : READ — if 1, sample SDA into readback latch
Bit 11  : SAVE — if 1, present sampled data with strobe at ACK time
Bit 10  : EXTD — if 1, fetch+execute next command without ACK (multi-byte seq)
Bit 9   : LOOP — if 1, data[7:0] is a loop count; repeat next FIFO word N times
Bit 8   : TOGS — toggle SAVE every other byte (avoids over-presenting data)
Bits 7:0: DATA — byte clocked out on SDA, or loop count if LOOP=1
```

### Transfer Type (`C` field, bits 7:0)

| Range | Meaning |
|---|---|
| `0x00–0x0F` | Write N data bytes: devaddr / regaddr / data×N. ACK on all bytes. |
| `0x10–0x1F` | Same, but NACK on last byte. |
| `0x20–0x2F` | Read N bytes: devaddr(W) / regaddr / Repeated-Start / devaddr(R) / data×N. ACK on all. |
| `0x30–0x3F` | Same read, but NACK on last byte (standard I2C read protocol). |
| `0x40` | No internal register address: devaddr / data. ACK both. |
| `0x41` | No internal register address: devaddr / data. NACK data byte. |
| `0x50` | Two-byte write to different register addresses (I2C_RegAddr + I2C_RegAddr2). |

### Scan Inhibit / FIFO Trigger Sequence (AO write path)

Before writing to the FIFO, the AO handler optionally disables the FPGA's autonomous I2C scanner:
1. Read-modify-write the scan control register (N bits 14:8) to clear the scanner-enable bit (A bits 11:8)
2. Write all I2C command words to the FIFO at address N[6:0]
3. Write the process-control register (F bits 15:8) to set the "send FIFO" bit (A bits 15:12), triggering the FPGA to process the FIFO
4. If scan inhibit was applied, re-enable scanning by setting the bit back

If `ScanCtlRegAddr == 0`, the scan-inhibit step is skipped.

### I2C AI (Read) Path

The AI handler is simpler: it writes I2C command words to the Collector FPGA FIFO for a **read-type** transaction (C = 0x20–0x3F), then reads the result from the FPGA's SPI reply word. The Collector FPGA autonomously clocks out the I2C sequence, samples the data byte(s), and makes them available for the Pi to read back via SPI.

### Data Width Limitation

Standard `ai`/`ao` records hold 16-bit values, limiting single-call transfers to 2 data bytes. For longer sequences (>2 bytes), the device support code automatically uses the `CollectorGlobDataStructure` global waveform buffer.

---

| File | Role |
|------|------|
| `spi.c` / `spi.h` | SPI1 hardware driver (bcm2835), DEVSEL GPIO bus |
| `CollectorSupport.c` / `.h` | Global data struct, conversion coefficients, mutex utils |
| `CollectorSupport.dbd` | EPICS device registration (ties DTYP strings to handlers) |
| `CollectorMain.cpp` | IOC main entry point |
| `CollectorCtl_AO.c` | ao write handler (SPI write) |
| `CollectorCtl_AI.c` | ai read handler (SPI read) |
| `CollectorCtl_BI.c` / `_BO.c` | bi/bo single-bit handlers |
| `CollectorCtl_MBBI.c` / `Mbbo` | mbbi/mbbo multi-bit handlers |
| `CollectorCtl_Waveform.c` | Waveform record handler |
| `CollectorSupport_AI/AO/BI/BO/MBBI/MBBO.c` | Higher-level support (mailbox/float PVs) |
| `CollectorCalc_AI/BI.c` | Calculated PVs (apply conversion coefficients) |
| `CollectorADCSupport_AI.c` | ADC scanner readout |
| `CollectorDPRSupport_AI.c` | Dual-port RAM support |
| `CollectorI2C_AI/AO.c` | I2C device support |
| `CollectorStep_AI.c` | HV step control logic |
| `DEVSEL_bus.c` | DEVSEL bus utilities |
| `ScanADCs.c` | ADC scanning loop |
| `initTrace.c` | Trace/debug init |
| `bcm2835.c` / `.h` | Raspberry Pi GPIO/SPI library (see [`collectorboxpi.md`](collectorboxpi.md) for Pi hardware setup) |

---

## CollectorI2C — I2C Bridge Device Support

**Files:** `CollectorI2C_AI.c`, `CollectorI2C_AO.c`  
**DTYP string:** `CollectorI2CSerial`  
✅ verified 2026-04-19 — `CollectorI2C_AI.c` lines 1–452

The collector FPGA has an on-board I2C controller with a **command FIFO**. The Pi writes 16-bit words into this FIFO via SPI to describe each I2C bus transaction; the FPGA executes the transaction autonomously after the FIFO is loaded. Loading the FIFO is an **AI PV** action (write to FIFO = "read" type record); a separate BO pulse PV kicks off the transaction.

### camacio Field Mapping for I2C

| camacio field | I2C parameter |
|---|---|
| `B` (Bidx) | Device index within collector (0–31) |
| `C` | Transfer type (mode byte — see below) |
| `N` (UsrAddr) | SPI register address of the I2C command FIFO |
| `A` | I2C device address (7-bit, shifted left by 1 to leave room for R/W) |
| `F` | I2C internal register address (8-bit) |

### FIFO Word Format (16-bit)

```
15   14   13   12   11   10   09   08   07  ...  00
+----+----+----+----+----+----+----+----+----------+
|DONE|RPTS|NACK|READ|SAVE|EXTD|LOOP|TOGS|  Data    |
+----+----+----+----+----+----+----+----+----------+
```

| Bit | Name | Meaning |
|-----|------|---------|
| 15 | DONE/ACK4_CTL[1] | With RPTS: 00=continue, 01=repeated start, 10=STOP, 11=STOP w/o ACK |
| 14 | RPTS/ACK4_CTL[0] | See above |
| 13 | NACK | If 1, skip ACK check on 9th clock |
| 12 | READ | If 1, sample SDA into readback latch |
| 11 | SAVE | If 1, present collected READ data at ACK strobe |
| 10 | EXTD | If 1, fetch next command immediately without ACK (multi-byte) |
| 9  | LOOP | If 1, data field = loop count; next command repeated that many times |
| 8  | TOGS | If 1, toggle SAVE flag every other byte |
| 7:0 | DATA | Byte to clock out on SDA, or loop count if LOOP=1 |

### C Parameter (Transfer Type) Modes

| C value | Transaction format |
|---------|-------------------|
| `0x00`–`0x0F` | Write: devaddr / regaddr / N data bytes (ACK all) |
| `0x10`–`0x1F` | Write: devaddr / regaddr / N data bytes (NACK last byte) |
| `0x20`–`0x2F` | Read: devaddr(W) / regaddr / repeated-start / devaddr(R) / N data bytes (ACK all) |
| `0x30`–`0x3F` | Read: devaddr(W) / regaddr / repeated-start / devaddr(R) / N data bytes (NACK last) |
| `0x40` | No internal register: devaddr / 1 data byte (ACK both) |
| `0x41` | No internal register: devaddr(ACK) / 1 data byte (NACK data) |

The lower nibble of `0x0x`/`0x1x`/`0x2x`/`0x3x` = number of data bytes. For >2 bytes the code pulls data from `GLBL_CollectorDataArray[Bidx][]` global buffer.

**Key design note:** Writing a `CollectorI2CSerial` AI PV only _loads the command FIFO_ — it does **not** trigger the I2C transaction. A separate `BO` PV must pulse a control register to start the transaction.

---

## CollectorStep — Closed-Loop HV Stepping

**File:** `CollectorStep_AI.c`  
**DTYP string:** `CollectorStep`  
✅ verified 2026-04-19 — `CollectorStep_AI.c` lines 1–362

`CollectorStep` is a **software closed-loop controller** implemented as an EPICS AI record. It does not directly write to hardware; instead it computes a new demand value that other PVs apply to the hardware (HV DAC — see [`collector_fpga.md`](collector_fpga.md) for the CtrlFPGA SPI DAC register map). The SCAN rate of the CollectorStep PV determines how fast the stepping occurs.

### Operating Modes

| `C` bit 15 | Mode | Mailbox array used |
|---|---|---|
| 0 | **Integer mode** | `GLBL_CollectorControlVals[Bidx][Cidx]` (unsigned 16-bit) |
| 1 | **Floating point mode** | `GLBL_CollectorFloatVals[Bidx][Cidx]` (epicsFloat64) |

### camacio Field Mapping for Step

| camacio field | Step parameter |
|---|---|
| `B` (Bidx) | Device index |
| `C` | Bit 15 = float/int mode; bits 5:0 = Cidx (mailbox index) |
| `A` | Aidx = index of interlock/enable mailbox |
| `F` | Enable bitmask — if `ControlVals[Bidx][Aidx] & F` is nonzero, stepping is blocked |

### Mailbox Layout (relative to Cidx)

| Mailbox offset | Name | Meaning |
|---|---|---|
| `[Cidx+0]` | DEMAND | Target value to move toward |
| `[Cidx+1]` | STEPSIZE | Max change per execution cycle |
| `[Cidx+2]` | HYSTERESIS | Dead band — no movement if abs(ACTUAL-DEMAND) < HYSTERESIS |
| `[Cidx+3]` | ABSMAX | Upper clamp (minimum is always 0) |
| `[Cidx+4]` | ACTUAL | Current monitored value (feedback) |

### Algorithm

1. Read `ACTUAL` and `DEMAND` from mailboxes.
2. Check interlock: if `ControlVals[Bidx][Aidx] & F` ≠ 0 → exit, no change.
3. If `abs(ACTUAL-DEMAND) < HYSTERESIS` → exit, already close enough.
4. If `ACTUAL < DEMAND` → ADD mode: new value = current PV + STEPSIZE, clamped to ABSMAX.
5. If `ACTUAL > DEMAND` → SUBTRACT mode: new value = current PV − STEPSIZE, clamped to 0.
6. Write new value to PV. The downstream PV picks this up and writes it to hardware.

This pattern allows **safe ramp-up/ramp-down** of HV by limiting the rate of change and blocking movement if hardware interlocks are asserted.

---

## CollectorDPRSupport — Dual-Port RAM (DPRAM) Read

**File:** `CollectorDPRSupport_AI.c`  
**DTYP string:** `CollectorDPRAM`  
✅ verified 2026-04-19 — `CollectorDPRSupport_AI.c` lines 1–405

The Collector FPGA has a banked register address space. The DPRAM device support handles both single-register reads and **loop (multi-read) transactions** into the global data buffer, with optional mailbox duplication and trace/debug controls.

### camacio Field Mapping for DPRAM

| camacio field | DPRAM parameter |
|---|---|
| `B` (Bidx) | Device index |
| `C` | bits 14:12 = MailboxMode (0–7); bits 7:0 = Cidx (mailbox index) |
| `N` | bits 9:7 = Bank (0 = default bank; nonzero → write bank# to addr 127 first); bits 6:0 = RegisterAddr |
| `A` | AndMask (bitmask applied to read data; 0 → treated as 0xFFFF) |
| `F` | bits 3:0 = ShiftFactor; bit 15 = ShiftDirection (1=left, 0=right) |

### Bank Switching

If `Bank` (bits 9:7 of N) ≠ 0, the device support writes the bank number to SPI address 127 before the data read. This selects the register bank in the FPGA before accessing the target register.

### MailboxModes

| Mode | Behavior |
|------|----------|
| 0 | Single read. N→addr. PV gets masked/shifted data. No mailbox dup. |
| 1 | Single read. N→addr. PV gets data + duplicated to `ControlVals[Bidx][Cidx]`. |
| 2 | Single read. N→addr. Data → mailbox only. PV returns FPGA status byte. |
| 3 | Single read. Mailbox is address. PV gets data, no dup. |
| 4 | Loop read (fixed addr). Mailbox = loop count. Data stored to global buffer. PV gets last data. |
| 5 | Loop read (incrementing addr). Mailbox = loop count. Data stored to global buffer. PV gets last data. |
| 6–7 | Extended modes (8-bit AndMask). |

### Debug / Trace

Print verbosity is controlled by `GLBL_CollectorControlVals[Bidx][5]` (overall flag) and `GLBL_CollectorControlVals[Bidx][17]` (address/mailbox filter). If bit 15 of `[17]` is set, prints only when N matches; if bit 14 is set, prints only when Cidx matches. This allows selective tracing without flooding the log.

---

## CollectorCalc AI — Multi-Mode Analog Input Calculation

**File:** `CollectorCalc_AI.c` (859 lines)  
**DTYP string:** `CollectorCalc`  
**Record type:** ai (analog input)  

`CollectorCalc` AI device support is a flexible computation engine for analog PVs. It performs no direct hardware I/O; instead it reads from the global mailbox (`GLBL_CollectorControlVals[][]`) or float mailbox (`GLBL_CollectorFloatVals[][]`), applies a transformation, and returns the result as the PV value. It is used for derived quantities (conversions, sensor readings, interlock math).

The DSET is `devAiCollectorCalc`. `init_record_ai` returns `2` (EPICS: no raw-value conversion). `init_ai` prints a one-line trace.

### camacio Field Mapping for CollectorCalc AI

✅ verified 2026-04-26 — all bit-field extractions confirmed against `CollectorCalc_AI.c:L396–L423`

| camacio field | Parameter |
|---|---|
| `B` bits 4:0 | Bidx — device index within collector (0–31) ✅ `b & 0x001F` (L396) |
| `C` bits 15:14 | CalculationMode (0–3) ✅ `C & 0xC000 >> 14` (L398) |
| `C` bits 13:12 | ConversionType (0=linear, 1=PT100, 2=PT500, 3=`m/x+b`) ✅ `C & 0x3000 >> 12` (L399) |
| `C` bits 11:8 | MultiMailboxMathMode (sub-mode when CalculationMode=3) ✅ `C & 0x0F00 >> 8` (L400) |
| `C` bits 7:0 | Cidx — mailbox or array index ✅ `C & 0x00FF` (L401) |
| `N` bit 15 | TransferIntFlag — copy result to int mailbox[Nidx] ✅ `N & 0x8000` (L406) |
| `N` bit 14 | TransferFloatFlag — copy result to float mailbox[Nidx] ✅ `N & 0x4000` (L407) |
| `N` bits 7:0 | Nidx ✅ `N & 0x00FF` (L405) |
| `A` bits 7:0 | Aidx — secondary mailbox index (modes 1–3) ✅ `A & 0x00FF` (L411) |
| `A` bit 15 | PT500_ModFlag — enable PT500 modifier from A mailbox ✅ `(A & 0x8000) >> 15` (L412) |
| `F` bits 13:8 | ConversionSet — index into `GLBL_ConversionCoefficients[][]` (mask 0x3F00; 6 bits) ✅ `(F & 0x3F00) >> 8` (L419) ⚠️ **Correction**: previously listed as bits 14:8 — actual mask is 0x3F00 = bits 13:8 |
| `F` bits 7:4 | ConversionSizeSign — data width/sign for modes 1–2 ✅ `(F & 0x00F0) >> 4` (L420) |
| `F` bits 3:0 | ShiftFactor (mode 0 only) ✅ `F & 0x000F` (L421) |
| `F` bit 15 | ShiftDirection (1=left, 0=right; mode 0 only) ✅ `F & 0x8000` (L422–424) |

### Calculation Modes

**Mode 0 — Mailbox read + bit-shift:**  
Reads `GLBL_CollectorControlVals[Bidx][Cidx]`, shifts left or right by `ShiftFactor` bits, stores result to PV. Optionally copies result to int mailbox `[Nidx]` and/or float mailbox `[Nidx]`.

**Mode 1 — Multi-byte mailbox read + unit conversion:**  
Reads 1–4 bytes (signed or unsigned) from `GLBL_CollectorControlVals[Bidx][Cidx]` (spanning adjacent mailboxes for multi-byte values). Calls `CollectorConversion()` to apply the selected unit conversion and returns the converted float as the PV value.

**Mode 2 — Global array read + unit conversion:**  
Same as mode 1 but reads from the linear global data array `GLBL_CollectorDataArray[Bidx][]` starting at `Cidx`. Conversion and optional mailbox copy same as mode 1.

**Mode 3 — Float mailbox math (`GlobalFloatMath`):**  
Performs one of 9 arithmetic operations on `GLBL_CollectorFloatVals[Bidx][]` entries, selected by `MultiMailboxMathMode` (bits 11:8 of C):

| Mode | Operation |
|---|---|
| 0 | `PV = Float[Cidx]` |
| 1 | `PV = Float[Cidx] + Float[FloatIdx2]` |
| 2 | `PV = Float[Cidx] - Float[FloatIdx2]` |
| 3 | `PV = Float[Cidx] * Float[FloatIdx2]` |
| 4 | `PV = Float[Cidx] / Float[FloatIdx2]` |
| 5 | `PV = Float[Cidx] + min(Float[FloatIdx2], Float[FloatIdx3])` |
| 6 | `PV = Float[Cidx] - min(Float[FloatIdx2], Float[FloatIdx3])` |
| 7 | `PV = min(Float[Cidx] + Float[FloatIdx2], Float[FloatIdx3])` (clamped add) |
| 8 | `PV = max(Float[Cidx] - Float[FloatIdx2], Float[FloatIdx3])` (clamped subtract) |

`FloatIdx2` comes from `A[12:8]` ✅ `(A & 0x1F00) >> 8` (L157); `FloatIdx3` from `A[4:0]` ✅ `A & 0x001F` (L156). GlobalFloatMath ops 0–8 verified at `CollectorCalc_AI.c:L140–195`.

### `CollectorConversion()` — Unit Conversion Function

Applied in modes 1 and 2. Selects conversion algorithm by `ConversionType`:

| Type | Algorithm |
|---|---|
| 0 | Linear: `y = m*x + b` using `GLBL_ConversionCoefficients[ConversionSet][0/1]` |
| 1 | PT100: 2-step — ADC→R (linear), then R→°C via CVD 2nd-order poly; saves *resistance* to mailbox |
| 2 | PT500: same 2-step as PT100 but uses `PT500Coefficients[]`; saves *resistance* to mailbox |
| 3 | Reciprocal linear: `y = m/x + b` |

For PT100/PT500: `PT100Coefficients[0..4]` are `[ADC→R slope, ADC→R intercept, a0, a1, a2]` where T = a0 + a1·R + a2·R². Resistance (not temperature) is the value optionally copied to the mailbox; temperature is the PV return value.

### Print / Trace Control

Verbosity controlled by three mailbox slots in `GLBL_CollectorControlVals[Bidx][]`:
- `[12]` — overall print enable flag (0 = no prints)
- `[17]` — filter mode when `[12]` is nonzero:
  - `== 0` → print everything
  - bit 15 → print only when `Nidx` matches `[17][7:0]`
  - bit 14 → print only when `Cidx` matches `[17][7:0]`
  - bit 13 → print only when `ConversionSet` matches `[17][5:0]`

✅ verified 2026-04-26 — `CollectorCalc_AI.c:L428–443` (print control logic); `GLBL_CollectorControlVals[Bidx][12]` overall flag, `[17]` filter register confirmed.

---

## CollectorCalc BI — Mailbox Comparison / Interlock Logic

**File:** `CollectorCalc_BI.c` (426 lines)  
**DTYP string:** `CollectorCalc`  
✅ verified 2026-04-19 — `CollectorCalc_BI.c` lines 1–426

`CollectorCalc` is an EPICS **BI (binary input) record** device support that evaluates a comparison expression on values from the global mailbox arrays and returns **0 or 1**. It is used for HV interlock logic: a `CollectorCalc` PV computes whether some condition is met (temperature OK, current within range, etc.) and that result gates other PVs.

The DSET is `devBiCollectorCalc`. The `init_record_bi` return value is `2` (EPICS: no raw-value conversion). Debug verbosity is controlled by `GLBL_CollectorControlVals[Bidx][13]` (per-device trace flag).

### camacio Field Mapping for CollectorCalc

| camacio field | Parameter |
|---|---|
| `B` (Bidx) | Device index within collector (0–31) |
| `C` | bits 15:12 = BitIndex; bits 11:8 = Function1 (comparison mode 0–15); bits 7:0 = Cidx |
| `N` (Nval) | Bitmask or loop count used by some modes |
| `A` bits 7:0 | Aidx1 — first mailbox index |
| `A` bits 15:8 | Aidx2 — second mailbox index |
| `A` bit 15 | ShiftDir (1=left, 0=right) — used by mode 12 |
| `A` bits 14:12 | Enbl_GT/EQ/LT — comparison enables for mode 12 |
| `A` bits 11:8 | ShiftAmount — used by mode 12 |
| `F` (Fval) | Threshold value for difference/mask comparisons |

### Function1 Modes (bits 11:8 of C)

| Mode | Comparison | Returns 1 if... |
|---|---|---|
| 0 | Always FALSE | Never |
| 1 | EQ | `mailbox[Aidx1] == mailbox[Aidx2]` |
| 2 | LT | `mailbox[Aidx1] < mailbox[Aidx2]` |
| 3 | GT | `mailbox[Aidx1] > mailbox[Aidx2]` |
| 4 | DIFF_EQ | `mailbox[Aidx1] - mailbox[Aidx2] == F` |
| 5 | DIFF_LT | `mailbox[Aidx1] - mailbox[Aidx2] < F` |
| 6 | DIFF_GT | `mailbox[Aidx1] - mailbox[Aidx2] > F` |
| 7 | BIT_AND_N | `(mailbox[Aidx1] & N) != 0` |
| 8 | BIT_XOR_N | `(mailbox[Aidx1] ^ N) != 0` |
| 9 | XOR_AND | `((mailbox[Aidx1] ^ N) & F) != 0` |
| 10 | OR | `(mailbox[Aidx1] | mailbox[Aidx2]) != 0` |
| 11 | AND | `(mailbox[Aidx1] & mailbox[Aidx2]) != 0` |
| 12 | SHIFT_CMP | `((mailbox[Aidx1] & N) << or >> ShiftAmount)` compared to F using Enbl_GT/EQ/LT enables |

Mode 12 is the most complex: it masks `mailbox[Aidx1]` with `N`, shifts the result left or right by `ShiftAmount`, then evaluates `>F`, `<F`, and `==F` comparisons as independently enabled by bits 14:12 of `A`. The function is TRUE if any enabled comparison succeeds.

### Usage Context

In the collector HV control chain (see [`collectorbox_PVs.md`](collectorbox_PVs.md) for HV-related PV names): `CollectorCalc` BI PVs compute interlock conditions (e.g., "is temperature below threshold?", "is HV current within range?") and write the result to a mailbox slot that `CollectorStep` checks in its enable-bitmask gate. This creates a pure-software interlock chain without any dedicated hardware interlock logic.

---

## CollectorCtl_Waveform — Global Data Array Readback

**File:** `CollectorCtl_Waveform.c` (183 lines)  
**DTYP string:** `CollectorSoftControl`  
**Record type:** waveform  
✅ verified 2026-04-26 — `CollectorCtl_Waveform.c` lines 1–183

`CollectorCtl_Waveform` implements a **waveform PV** that exposes the per-device global data array (`GLBL_CollectorDataArray[Bidx]`) as a 1024-element EPICS waveform record. It does **no SPI hardware I/O** — it is purely a memory readback mechanism.

### Purpose

The `GLBL_CollectorDataArray[Bidx]` is a shared uint16 buffer (1024 elements) used by other device support modules (e.g., `CollectorI2C_AI/AO`) as a bulk data staging area for multi-byte transfers. The waveform PV gives EPICS clients the ability to read this buffer back (e.g., for diagnostic display or to verify what was staged for an I2C transaction).

### camacio Field Mapping

| camacio field | Parameter |
|---|---|
| `B` (Bidx) | Device index (0–31) — selects which `GLBL_CollectorDataArray[Bidx]` is exposed |
| `C` (Cval) | Verbosity flag — printed at record load if `GLBL_CollectorControlVals[Bidx][10]` is set |
| `N`, `A`, `F`, `parm` | Not used |

### Read Flow

1. Extract `Bidx` from `camacio.b & 0x001F`.
2. If `GLBL_CollectorControlVals[Bidx][10]` is set, print a trace line.
3. Set `PtrToPVrecord->bptr = GLBL_CollectorDataArray[Bidx]` — points the EPICS waveform buffer pointer directly at the global array.
4. Set `PtrToPVrecord->nord = 1024` (all 1024 elements reported as valid).
5. Set `PtrToPVrecord->busy = 0` (not busy).
6. Optional: if `GLBL_CollectorControlVals[Bidx][31]` is set, dump the first 100 elements to stdout.
7. Returns `0`.

### DSET Registration

```c
struct { long number; DEVSUPFUN report; init; init_record; get_ioint_info; read_wf; }
devWaveCollectorCtl = { 5, NULL, init_wf, init_record, NULL, read_wf };
epicsExportAddress(dset, devWaveCollectorCtl);
```

**Note:** Uses the same DTYP string `"CollectorSoftControl"` as the non-waveform CTL records — the waveform record type causes EPICS to dispatch to this DSET rather than the AI/BI/AO/BO variants.

---

## CollectorADC — Direct SPI ADC Read

**File:** `CollectorADCSupport_AI.c` (224 lines)  
**DTYP string:** `CollectorADC`  
**Record type:** ai  
✅ verified 2026-04-25 — `CollectorADCSupport_AI.c` lines 1–224

`CollectorADC` is a lightweight **AI record device support** for doing a single SPI read from a Collector card register. It is simpler than `CollectorDPRAM`: no MailboxModes, no loop reads, no AndMask/shift — just a direct register read.

### camacio Field Mapping for CollectorADC

| camacio field | Parameter |
|---|---|
| `B` (Bidx) | Device index within collector (0–31) |
| `C` | Cidx (mailbox index, for debug/trace filtering only; not used for data routing) |
| `N` | bits 6:0 = RegisterAddr; bits 9:7 = Bank (0 = default; nonzero → write bank# to addr 127 first) |
| `A`, `F`, `parm` | Unused |

### Read Flow

1. Cast `inp.value` to `struct camacio *` and extract `B`, `C`, `N`.
2. If Bank (bits 9:7 of N) ≠ 0: issue `SPI_WITH_BLOCK(0, Bidx, 127, Bank)` to select the register bank.
3. Issue `SPI_WITH_BLOCK(1, Bidx, RegisterAddr, 0)` — a 24-bit SPI read.
4. Return value: lower 16 bits of `SPI_data_in` → `PtrToPVrecord->val` and `rval`.
5. Upper 8 bits of `SPI_data_in` (FPGA status byte) → `GLBL_CollectorControlVals[Bidx][32]` (always).
6. If Bank ≠ 0: reset bank with `SPI_WITH_BLOCK(0, Bidx, 127, 0)`.
7. Returns `0` (EPICS will apply record-level conversion from `rval`; unlike most other collector support files this does **not** return `2`). ✅ verified 2026-04-25 — `CollectorADCSupport_AI.c:L187` (comment explicitly notes EPICS overwrites VAL with RVAL integer unless return=2)

### Print/Trace Control

Verbosity follows the same scheme as `CollectorDPRSupport`:
- `GLBL_CollectorControlVals[Bidx][5]` = overall print enable flag
- `GLBL_CollectorControlVals[Bidx][17]` = filter:
  - Bit 15 set → print only when RegisterAddr matches low 7 bits of `[17]`
  - Bit 14 set → print only when Cidx matches low 8 bits of `[17]`
  - `[17]` == 0 → print unconditionally (if `[5]` is set)

### Key Difference from CollectorDPRAM

`CollectorADC` is a stripped-down read with no mailbox routing logic — the result always goes directly to the PV value. `CollectorDPRAM` has 8 MailboxModes and supports loop reads and global buffer writes. Use `CollectorADC` for simple ADC register reads where you only need the raw value in a PV; use `CollectorDPRAM` for reads requiring bank switching + mailbox duplication or multi-register buffer fills.

---

## CollectorSoftControl — RAM Mailbox DTYP (AI/BI/BO/MBBI)

_Source: `CollectorCtl_AI.c` (291 L), `CollectorCtl_BI.c` (198 L), `CollectorCtl_BO.c` (278 L), `CollectorCtl_MBBI.c` (186 L). DTYP: `CollectorSoftControl`._

The `CollectorSoftControl` DTYP family is the **RAM-mailbox counterpart** to `Collector Local Serial`. These handlers do **no SPI hardware transactions** — they read and write the in-memory global data structure (`GLBL_CollectorControlVals[][]`, `GLBL_CollectorFloatVals[][]`, `GLBL_CollectorDataArray[][]`) only. They are used for staging control values that other device support reads will later forward to hardware.

### camacio Field Decoding (all Ctl_ handlers)

| camacio field | Meaning |
|---|---|
| `B` (bits 4:0) | `Bidx` — device index within collector (0–31) |
| `C` (bits 7:0) | `Cidx` — mailbox column index (0–255) |
| `C` (bits 14:12) | `PV_mode` — selects read/write sub-mode (AI only) |
| `C` (bit 15) | `TransferFlag` — if set, copy result to a mailbox after read (AI only) |
| `A` | `AndMask` — applied to mailbox value before stuffing PV |
| `F` (bits 3:0) | `ShiftFactor` — number of bits to shift |
| `F` (bit 15) | `ShiftDirection` — 1=left, 0=right (AI only) |
| `F` (bits 13:8) | `CoefficientIdx` — index into conversion coefficient table (AI modes 4–7) |
| `N` | `Nidx` — output mailbox index for TransferFlag write (AI); multiply factor (BO mode 0); float set value numerator/256 (BO mode 6) |

### CollectorCtl_AI — 8 Read Modes

`read_ai()` switches on `PV_mode` (bits 14:12 of C):

| Mode | Action |
|------|--------|
| 0 | Read `GLBL_CollectorControlVals[Bidx][Cidx]`, apply AndMask + shift; if TransferFlag, copy result to `GLBL_CollectorControlVals[Bidx][Nidx]` via `TracedWriteToMailbox()` |
| 1 | Read `GLBL_CollectorFloatVals[Bidx][Cidx]` — floating-point readback |
| 2 | Read `GLBL_ConversionCoefficients[Cidx][0]` (coefficient A for slot Cidx) |
| 3 | Read `GLBL_ConversionCoefficients[Cidx][1]` (coefficient B for slot Cidx) |
| 4 | Read `GLBL_CollectorFloatVals[Bidx][Cidx]`, copy to `GLBL_ConversionCoefficients[CoefficientIdx][0]` |
| 5 | Read `GLBL_CollectorFloatVals[Bidx][Cidx]`, copy to `GLBL_ConversionCoefficients[CoefficientIdx][1]` |
| 6 | Same as mode 4 (float read + copy to coeff[0]) — variant copy path |
| 7 | Same as mode 5 (float read + copy to coeff[1]) — variant copy path |

All modes return EPICS magic value `2` (bypass RVAL→VAL overwrite).

### CollectorCtl_BI — Single-Bit Mailbox Read

`read_bi()` reads `GLBL_CollectorControlVals[Bidx][Cidx]` and applies `AndMask`:
- If `AndMask == 0` in the camacio field, the mask is synthesized as `1 << ShiftFactor` (bit-select convenience)
- Result: 1 if any bit of (mailbox & AndMask) is set, else 0
- Debug print gated on `GLBL_CollectorControlVals[Bidx][8] == 1`

### CollectorCtl_BO — 9 Write Modes

`write_bo()` switches on bits 15:12 of C (`ControlValue`):

| Mode | Action |
|------|--------|
| 0 | Write `PV.val × N` to `GLBL_CollectorControlVals[Bidx][Cidx]` (if N=0, writes `PV.val` directly) |
| 1 | Reset array pointer `GLBL_CollectorArrayPtr[Cidx]` to start of `GLBL_CollectorDataArray[Bidx]` |
| 2 | Set `GLBL_CollectorArrayPtr[Cidx]` to `&GLBL_CollectorDataArray[Bidx][GLBL_CollectorControlVals[Bidx][Cidx] & 0x3FF]` (offset by stored value) |
| 3 | Reset array pointer to start + zero all 1024 words of `GLBL_CollectorDataArray[Bidx]` |
| 4 | Load test pattern into `GLBL_CollectorDataArray[Bidx]`: ramp 0–249, decay 250–499, ramp×2 500–749, shifted index 750–1023 |
| 5 | Write `PV.val × N` to mailbox range `[Bidx][Aidx..Fidx]` via `TracedWriteToMailbox()` |
| 6 | Write float `N/256.0` to `GLBL_CollectorFloatVals[Bidx][Aidx..Fidx]` (range write) |
| 7 | Dump all mailbox and float values for `Bidx` to stdout (diagnostic); resets PV.val to 0 after dump |
| 8 | Read-modify-write: if PV=1, OR `N` into `GLBL_CollectorControlVals[Bidx][Cidx]`; if PV=0, AND with `~N` (bit-set/clear) |

### CollectorCtl_MBBI — Multi-Bit Mailbox Read

`read_mbbi()` reads `GLBL_CollectorControlVals[Bidx][Cidx]`, applies `AndMask`, then right-shifts by `ShiftFactor` (bits 3:0 of F). Result goes to both `rval` and `val`. No mode switch — single operation. Debug print gated on `GLBL_CollectorControlVals[Bidx][11] != 0`.

### Debug / Trace Controls

Each Ctl_ handler reads a dedicated `GLBL_CollectorControlVals[Bidx][N]` slot to gate verbose debug prints:

| Handler | Debug gate index |
|---------|------------------|
| CollectorCtl_AI | `[Bidx][7]` |
| CollectorCtl_BI | `[Bidx][8]` |
| CollectorCtl_BO | `[Bidx][9]` |
| CollectorCtl_MBBI | `[Bidx][11]` |

---

## DEVSEL Bus Driver (`DEVSEL_bus.c`)

_Source: `DEVSEL_bus.c` (93 L). Added 2022-08-16 (`usleep` for propagation timing)._

The **PI_DEVSEL bus** is a 5-bit GPIO bus on the Raspberry Pi that selects which physical device on the SPI bus receives the next transaction. The bus maps to 5 non-contiguous GPIO pins:

| DEVSEL bit | GPIO number | Pi header pin |
|-----------|------------|---------------|
| 0 | GPIO_13 | Pin 33 |
| 1 | GPIO_23 | Pin 16 |
| 2 | GPIO_24 | Pin 18 |
| 3 | GPIO_25 | Pin 22 |
| 4 | GPIO_26 | Pin 37 |

Because the BCM2835 library uses **separate set/clear registers** (no read-modify-write on GPIO), the bus cannot be updated by a simple write. The driver always:
1. Clears all DEVSEL bits atomically: `bcm2835_gpio_clr_multi(DEVSEL_BUS_BITMASK)`
2. Builds a `uint32_t` set mask: bits 4:1 of the device index → GPIO bits 26:23 (left-shift 22); bit 0 → GPIO bit 13 (OR 0x2000)
3. Asserts the new value: `bcm2835_gpio_set_multi(tempval)`
4. Waits `usleep(3)` to allow bus propagation before any SPI transaction begins

Allowed device values: 0–31 (`0x00–0x1F`).

**Key function:** `Set_Pi_Devsel(unsigned short int new_device)` — the only exported function in this file.

---

## ScanADCs.c — ADC Scanning Placeholder

_Source: `ScanADCs.c` (29 L)._

This file contains only the standard EPICS header includes, the `spi.h` / `CollectorSupport.h` includes, and a comment block explaining the intent:

> "Readout of the ADCs is an invasive process that can interfere with the 'normal' EPICs operation."

No functions are implemented. The actual ADC scanning logic is handled via `CollectorADCSupport_AI.c` (the `CollectorADC` DTYP). `ScanADCs.c` appears to be a stub placeholder from an earlier design iteration where a background scanning thread was considered but not pursued.

---

*Source: `DGS_tools_pack/collectorboxpi/CollectorBox_RevA/CollectorApp/src/` — C device support source. Created: 2026-04-05.*

## Cross-References

- [`collectorboxpi.md`](collectorboxpi.md) — Raspberry Pi soft IOC: PXE boot, HV control, collector assignments
- [`collector_fpga.md`](collector_fpga.md) — CtrlFPGA and StripeFPGA firmware; SPI register maps
- [`collectorbox_PVs.md`](collectorbox_PVs.md) — Full PV list (1,431 records/detector)
- [`sbx.md`](sbx.md) — Slope Box Extension; pickoff card; BGO HV; GS_ID dongle
- [`EPICS.md`](EPICS.md) — EPICS record types and device support concepts

*Created: 2026-04-06 | Last reviewed: 2026-04-25*
