# SBX Pi IOC — PickoffApp_RevC

Stability: C2 - Active / semi-stable

**Source:** `DGS_tools_pack/PickoffPi/PickoffApp_RevC/`  
**Last updated:** 2026-04-26  
**See also:** [`sbx.md`](sbx.md) — SBX hardware + Collector Box Pi IOC overview

---

## Table of Contents

- [Purpose](#purpose)
- [Hardware Interface](#hardware-interface)
  - [SPI1 (Pi → Pickoff FPGA)](#spi1-pi--pickoff-fpga)
- [EPICS Device Support Design](#epics-device-support-design)
  - [Key Design Note: Why Not asyn?](#key-design-note-why-not-asyn)
  - [Link Type: CAMAC_IO](#link-type-camac_io-camacio)
  - [Device Support Identifiers](#device-support-identifiers-from-dbd)
  - [PickoffLocalSerial AO — Write Modes](#pickofflocalserial-ao-devaoPickoff--write-modes)
  - [PickoffCalc BI — Comparison Logic](#pickoffcalc-bi--comparison-logic)
  - [PickoffStep AI — Step-wise Control (Full Detail)](#pickoffstep-ai--step-wise-control-full-detail)
  - [PickoffSoftControl AO — Mode Reference](#pickoffsoftcontrol-ao--mode-reference)
- [Global Mailboxes](#global-mailboxes)
  - [PickoffCalc Conversion Types](#pickoffcalc-conversion-types-from-pickoffcalc_aic)
- [I2C via Pickoff FPGA](#i2c-via-pickoff-fpga)
- [Database Files](#database-files)
- [Startup / IOC Configuration](#startup--ioc-configuration-stcmd)
- [HV Ramp Logic](#hv-ramp-logic-hv_stepdb)
- [Detector Specification](#detector-specification-detspecdb)
- [Relationship to Collector Box Pi IOC](#relationship-to-collector-box-pi-ioc)
- [PickoffSupportBackup.c — Historical / Educational Version](#pickoffsupportbackupc--historical--educational-version)
- [Cross-References](#cross-references)

---

## Purpose

The `PickoffApp_RevC` is a standalone EPICS soft IOC designed to run on a **Raspberry Pi mounted directly on the SBX (Slope Box Extension)** for single-detector operation. It controls and monitors one detector (GeCenter, GeSide, BGO sum/pattern, Ge HV, BGO HV, slope box ADC readbacks, power board, preamp) via direct **SPI1** communication from the Pi to the Pickoff Card FPGA.

This differs from the full system where the **Collector Box Pi** IOC handles up to 28 detectors via a shared SPI bus. The PickoffPi IOC is used for:
- **Standalone detector operation** (e.g. G-wing lab test stand)
- **Small system configurations** where one Pi per SBX is practical
- **Commissioning and characterization** of individual detectors

---

## Hardware Interface

### SPI1 (Pi → Pickoff FPGA)

The IOC uses **SPI1** (not SPI0) of the Raspberry Pi BCM2835 via the `bcm2835` library. SPI1 must be enabled in `/boot/config.txt`:
```
dtoverlay=spi1-1cs
```

**Transaction format (24-bit, fixed):**
```
Bit 23 (MSbit):  R/W flag  (1 = read, 0 = write)
Bits 22:16:      7-bit address
Bits 15:0:       16-bit data
```

- Data clocked out **MSbit first** on MOSI
- FPGA pre-loads first bit before rising edge; Pi samples on **rising edge** of SCLK
- Data output on falling edge of SCLK
- SCLK starts in **low** state
- CS: **CS2 only** (`BCM2835_AUX_SPI_CNTL0_CS2_N`)

**Core SPI functions (`spi.c`):**
| Function | Description |
|----------|-------------|
| `SPI1_setup(init_flag, RequestedSpeed)` | Initialize SPI1 via bcm2835; set clock divider, CS, bit order, transaction length (24-bit fixed) |
| `Do_SPI1_transaction(RWflag, UsrAddr, UsrData)` | Perform one 24-bit SPI transaction; polls BUSY status before returning received data |
| `SPI1_exit()` | Shut down SPI; drops CE2 — only call at IOC shutdown |

**Clock speed:** Set via `RequestedSpeed` parameter passed into `SPI1_setup()`. The `bcm2835_aux_spi_setClockDivider()` formula: `SPICLKfreq = 250MHz / (2 × (speed+1))`. The IOC uses **divider=30** → nominal 250MHz/(2×31) ≈ 4.03 MHz, but measured at ~5 MHz via Chipscope (400 ns/period at 100 MHz FPGA clock — so the divider may represent edge period, not full period). ✅ verified 2026-04-24 — `PickoffCtl_AO.c:L52` (`SPI1_setup(1,30)`); inline comment: "20 SPI clock periods take 400 ticks of the 100MHz clock → 200ns/clock = 5MHz".

---

## EPICS Device Support Design

### Key Design Note: Why Not asyn?

The embedded comments in `PickoffSupport.c` (JTA) explicitly document why `asyn` is **not used** in any DGS Raspberry Pi IOC:
- `asyn` assumes byte-granularity for all serial transactions
- DGS SPI transactions can be **arbitrary bit lengths** — not restricted to multiples of 8 bits
- True SPI requires **simultaneous** MOSI/MISO; `asyn` does not support this
- *"asyn is COMPLETELY VALUELESS to DGS projects"* — PickoffSupport.c comment

### Link Type: CAMAC_IO (`camacio`)

All PVs use `CAMAC_IO` link type, repurposing the `camacio` struct fields for SPI/GPIO control:

| `camacio` field | DGS use |
|----------------|---------|
| `B` | GPIO / detector address (0 = local Pi-to-pickoff, no Collector Box routing) |
| `C` | Transaction length in bits (typically 24) — or control flags for soft PVs |
| `N` | 7-bit SPI register address (bits 6:0, masked with `& 0x007F` in device support); **R/W direction is NOT stored here** — it is hardcoded per record type (AI/BI/MBBI always read; AO/BO/MBBO always write) ✅ verified 2026-04-22 — `PickoffSupport_AI.c:L92` (`RegisterAddr = PtrToLinkStruct->n & 0x007F`), `PickoffSupport_MBBO.c:L144` (same mask); `Do_SPI1_transaction(1,...)` for reads, `Do_SPI1_transaction(0,...)` for writes |
| `A` | AND mask (for bo/bi/mbbi/mbbo bit-field extraction) ✅ verified 2026-04-22 — `PickoffSupport_AI.c:L93` (`AndMask = PtrToLinkStruct->a`) |
| `F` | Shift factor: bits[3:0] = shift amount; bit 15 = shift direction (1=right, 0=left). **Not an OR mask.** ✅ verified 2026-04-22 — `PickoffSupport_AI.c:L183-187` (`ShiftFactor = PtrToLinkStruct->f & 0x000F; if (f & 0x8000) ShiftDirection=1 else 0`) |
| `parm` | Additional parameters after `@` in OUT/INP string |

**Example PV OUT field (ao write to register 0x14, 24-bit, local Pi):**
```
OUT(0, 24, 20, 0, 0, "")
      ^   ^   ^
      B=0 C=24 N=0x14 (address 20 decimal = 0x14; R/W determined by record type, not N)
```

### Device Support Identifiers (from `.dbd`)

| Identifier | Record Types | Description |
|-----------|-------------|-------------|
| `PickoffLocalSerial` | ao, ai, bo, bi, mbbi, mbbo | Direct SPI1 transaction to/from Pickoff FPGA |
| `PickoffI2CSerial` | ai | I2C transaction via Pickoff FPGA I2C command FIFO |
| `PickoffCalc` | ai | Conversion chain — applies y=mx+b or PT100/PT500 polynomial to raw ADC values |
| `PickoffSoftControl` | ao, ai | Software mailbox r/w — no hardware transaction, reads/writes `GLBL_PickoffControlVals[]` or `GLBL_PickoffFloatVals[]`. AO has 8 distinct modes (see table below). |
| `PickoffStep` | ai | Controlled step-wise adjustment (used for HV ramp logic) |

### `PickoffLocalSerial` AO (`devAoPickoff`) — Write Modes

The `C` field of a `PickoffLocalSerial` ao record (i.e., `devAoPickoff`, `write_ao` in `PickoffSupport_AO.c`) encodes a 2-bit MailboxMode and mailbox index:
- `MailboxMode = (C & 0x3000) >> 12` — 2-bit mode selector (bits 13:12) ✅ verified 2026-04-25 — `PickoffSupport_AO.c:L183`
- `Cidx = C & 0x00FF` — mailbox index (bits 7:0) ✅ verified 2026-04-25 — `PickoffSupport_AO.c:L184`
- `AndMask` from `A`; `ShiftFactor = F & 0xF` (left shift of PV value before write)

| Mode | Action |
|------|--------|
| 0 | **Normal:** `UsrAddr = N & 0x7F`; `UsrData = PV_val << ShiftFactor`. If `strlen(parm)==0` → read-modify-write (AND+OR); else write-only (OR with `A` mask). ✅ verified 2026-04-25 — `PickoffSupport_AO.c:L192-200,227-247` |
| 1 | **Normal with copy:** Same as mode 0, but after the write, copies the written value to `GLBL_PickoffControlVals[Cidx]`. ✅ verified 2026-04-25 — `PickoffSupport_AO.c:L201-210,254-258` |
| 2 | **Indirect data:** `UsrAddr = N & 0x7F`; `UsrData = GLBL_PickoffControlVals[Cidx] << ShiftFactor` (data comes from mailbox, not PV). ✅ verified 2026-04-25 — `PickoffSupport_AO.c:L211-215` |
| 3 | **Indirect address:** `UsrAddr = GLBL_PickoffControlVals[Cidx] & 0x7F` (address from mailbox); `UsrData = PV_val << ShiftFactor`. ✅ verified 2026-04-25 — `PickoffSupport_AO.c:L216-222` |

> Note: This is **distinct** from `PickoffSoftControl` AO (8 modes, `0x7000`, no hardware SPI). `devAoPickoff` always performs a real SPI transaction.

---

### `PickoffCalc` BI — Comparison Logic

**Source:** `PickoffCalc_BI.c`  
**Record type:** `bi`  
**Device identifier:** `PickoffCalc`

The `PickoffCalc` bi device support evaluates a boolean comparison against global mailbox values and:
1. Writes the result (0 or 1) to `PtrToPVrecord->val` / `rval`
2. Also **writes the result bit** into a specific bit position of `GLBL_PickoffControlVals[Cidx]` — this is the key difference from a pure read

✅ verified 2026-04-26 — `PickoffCalc_BI.c:L365-366` (`PtrToPVrecord->val = Function1Result; PtrToPVrecord->rval = Function1Result;`); bit-packing at L353-363

**`camacio` field mapping for `PickoffCalc` bi:**

| Field | Bits | Use |
|-------|------|-----|
| `C[11:8]` | 4 bits | `Function1` — comparison mode (0–14, see table below) |
| `C[15:12]` | 4 bits | `BitIndex` — bit position (0–15) in `GLBL_PickoffControlVals[Cidx]` to write result into |
| `C[7:0]` | 8 bits | `Cidx` — index into `GLBL_PickoffControlVals[]` for the output bit-pack target |
| `N` | 16 bits | `Nval` — mask or comparison value used by modes 7, 8, 9, 12, 13, 14 |
| `A[7:0]` | 8 bits | `Aidx1` — first mailbox index (lower byte) |
| `A[15:8]` | 8 bits | `Aidx2` — second mailbox index (upper byte) |
| `A[15]` | 1 bit | `ShiftDir` (mode 12): 1=left shift, 0=right shift |
| `A[14]` | 1 bit | `Enbl_GT` (mode 12): enable > comparison |
| `A[13]` | 1 bit | `Enbl_EQ` (mode 12): enable == comparison |
| `A[12]` | 1 bit | `Enbl_LT` (mode 12): enable < comparison |
| `A[11:8]` | 4 bits | `ShiftAmount` (mode 12) |
| `F` | 16 bits | `Fval` — threshold or second mask (modes 4–6, 9, 12, 13, 14) |

✅ verified 2026-04-26 — `PickoffCalc_BI.c:L127-130` (`Function1=(c & 0x0F00)>>8; BitIndex=(c & 0xF000)>>12; Cidx=c & 0x00FF`); `L139-145` (`Aidx1=a & 0x00FF; Aidx2=(a & 0xFF00)>>8; ShiftDir=(a & 0x8000)>>15; Enbl_GT=(a & 0x4000)>>14; Enbl_EQ=(a & 0x2000)>>13; Enbl_LT=(a & 0x1000)>>12; ShiftAmount=(a & 0x0F00)>>8`)

**Debug print gate:** `GLBL_PickoffControlVals[13]` — if nonzero, verbose printf output for all `PickoffCalc` bi evaluations. ✅ verified 2026-04-26 — `PickoffCalc_BI.c:L133` (`if (GLBL_PickoffControlVals[13]) printf(...)` pattern repeated throughout)

**Function1 comparison modes:**

| Mode | Condition (TRUE → result=1) |
|------|-----------------------------|
| 0 | Always FALSE |
| 1 | `mailbox[Aidx1] == mailbox[Aidx2]` |
| 2 | `mailbox[Aidx1] < mailbox[Aidx2]` |
| 3 | `mailbox[Aidx1] > mailbox[Aidx2]` |
| 4 | `mailbox[Aidx1] - mailbox[Aidx2] == F` |
| 5 | `mailbox[Aidx1] - mailbox[Aidx2] < F` |
| 6 | `mailbox[Aidx1] - mailbox[Aidx2] > F` |
| 7 | `(mailbox[Aidx1] & N) != 0` |
| 8 | `(mailbox[Aidx1] ^ N) != 0` |
| 9 | `((mailbox[Aidx1] ^ N) & F) != 0` |
| 10 | `(mailbox[Aidx1] \| mailbox[Aidx2]) != 0` |
| 11 | `(mailbox[Aidx1] & mailbox[Aidx2]) != 0` |
| 12 | `((mailbox[Aidx1] & N) shift ShiftAmount)` compared to F with GT/EQ/LT enable bits (any enabled TRUE → result=1) |
| 13 | `(mailbox[Aidx1] & N) != 0` **AND** `(mailbox[Aidx2] & F) != 0` |
| 14 | `(mailbox[Aidx1] & N) != 0` **OR** `(mailbox[Aidx2] & F) != 0` |
| default | Always TRUE |

✅ verified 2026-04-26 — `PickoffCalc_BI.c:L150-348` (all 15 cases confirmed; modes 0–14 + default match table exactly)

**Side effect (output bit packing):**  
After computing `Function1Result`, the code:
```c
tempval = GLBL_PickoffControlVals[Cidx];       // read current packed word
tempval &= ~(1 << BitIndex);                    // clear target bit
tempval |= (Function1Result << BitIndex);       // set or leave cleared
GLBL_PickoffControlVals[Cidx] = tempval;        // write back
```
This allows multiple `PickoffCalc` bi records to pack 16 independent boolean conditions into a single mailbox word, which can then be inspected by `PickoffStep` (via F mask) or `PickoffSoftControl` (via mode 6/7 readback).
✅ verified 2026-04-26 — `PickoffCalc_BI.c:L353-363` (exact code match)

---

### `PickoffStep` AI — Step-wise Control (Full Detail)

**Source:** `PickoffStep_AI.c`  
**Record type:** `ai`  
**Device identifier:** `PickoffStep`

The `PickoffStep` device support performs **one incremental step** per scan cycle toward a demand value. It does not write to hardware directly — it updates a mailbox value that a downstream PV (e.g. `GS${DetNbr}_GE_HV_DEMAND_DAC`) reads and writes to the DAC.

**`camacio` field mapping for `PickoffStep` ai:**

| Field | Use |
|-------|-----|
| `C[15]` | Mode: 1=floating-point mode (uses `GLBL_PickoffFloatVals`), 0=integer mode (uses `GLBL_PickoffControlVals`) |
| `C[7:0]` | `Cidx` — base index into mailbox array (see layout below) |
| `A[7:0]` | `Aidx` — index of interlock mailbox in `GLBL_PickoffControlVals[Aidx]` |
| `F` | `EnableMask` — bitmask ANDed with `GLBL_PickoffControlVals[Aidx]`; if result ≠ 0, stepping is blocked |

✅ verified 2026-04-26 — `PickoffStep_AI.c:L178-198` (`Aidx=a & 0x00FF`; `EnableMask=GLBL_PickoffControlVals[Aidx] & f`; `if (c & 0x8000) FloatOrInt=1`; `Cidx=c & 0x00FF`)

**Mailbox layout (starting at `Cidx`):**

| Offset | Float mode (`GLBL_PickoffFloatVals`) | Integer mode (`GLBL_PickoffControlVals`) |
|--------|--------------------------------------|------------------------------------------|
| [Cidx+0] | DEMAND (target value) | DEMAND |
| [Cidx+1] | STEPSIZE (max V per cycle) | STEPSIZE |
| [Cidx+2] | HYSTERESIS (dead-band) | HYSTERESIS |
| [Cidx+3] | ABSMAX (hard ceiling) | ABSMAX |
| [Cidx+4] | ACTUAL (current measured value, written by upstream conversion PV) | ACTUAL |
| [Cidx+5] | LAST_ACTUAL (saved by PickoffStep for comparison) | LAST_ACTUAL |
| [Cidx+6] | NEW_PV_VAL (output written by PickoffStep for downstream use) | NEW_PV_VAL |

**Step algorithm:**
1. Read `GLBL_PickoffControlVals[Aidx] & F` → if nonzero, **abort** (interlock active), return immediately with `val=0.0`
2. Read DEMAND, STEPSIZE, HYSTERESIS, ABSMAX, ACTUAL from [Cidx+0..+4]
3. If `abs(ACTUAL − DEMAND) < HYSTERESIS` → **no-op** (already close enough), save ACTUAL to [Cidx+5], return
4. If `ACTUAL − DEMAND < 0` → **ADD mode:** `NEW = val + STEPSIZE`, capped at min(DEMAND, ABSMAX)
5. If `ACTUAL − DEMAND > 0` → **SUBTRACT mode:** `NEW = val − STEPSIZE`, floored at max(DEMAND, 0.0)
6. Save ACTUAL to [Cidx+5]; set PV.val = NEW; write NEW to [Cidx+6]

✅ verified 2026-04-26 — `PickoffStep_AI.c:L179-301` (interlock abort at L180-185; DEMAND/STEPSIZE/HYSTERESIS/ABSMAX/ACTUAL reads at L201-205/L213-217; hysteresis exit at L228-233; ADD/SUBTRACT modes at L248-295; LAST_ACTUAL save at L272,L286)

**Debug print gate:** `GLBL_PickoffControlVals[16]` — if nonzero, verbose step calculations are printed. ✅ verified 2026-04-26 — `PickoffStep_AI.c:L181,L187,L209,L230,L263...` (`if (GLBL_PickoffControlVals[16]) printf(...)` pattern throughout)

> **Note:** There is a commented-out check comparing ACTUAL to LAST_ACTUAL (Cidx+5). If the ACTUAL change per cycle is less than HYSTERESIS (when ACTUAL > 50V), it would abort the step as a stuck-DAC guard. This check is **disabled** in the current code. ✅ verified 2026-04-26 — `PickoffStep_AI.c:L247-254` (lines commented out with `//`)

---

### `PickoffSoftControl` AO — Mode Reference

The `C` field of a `PickoffSoftControl` ao record encodes both the mode and mailbox index:
- `PV_mode = (C & 0x7000) >> 12` — 3-bit mode selector (bits 14:12)
- `Cidx = C & 0x00FF` — mailbox index (bits 7:0; for modes 4/5 further masked to `& 0x003F`)
- Bit 15 of C: if set in mode 1, auto-advance array pointer after write

| Mode | Action |
|------|--------|
| 0 | Write PV value to `GLBL_PickoffControlVals[Cidx]`. If `strlen(parm)==0` — read-modify-write using ANDMask and ShiftFactor; if nonzero — write-only (shift + OR with mask). |
| 1 | Write PV value to `GLBL_PickoffDataArray[ptr]` (the roving array pointer). If `C & 0x8000` is set, auto-advance pointer after write (capped at index 1023). |
| 2 | Write PV value to `GLBL_PickoffFloatVals[Cidx]` (floating-point mailbox). |
| 3 | Set array pointer `GLBL_PickoffArrayPtr` to `&GLBL_PickoffDataArray[PV_val]` (capped at 1023). |
| 4 | Write PV value to `GLBL_ConversionCoefficients[Cidx][0]` — the `m` (slope) for conversion set `Cidx`. |
| 5 | Write PV value to `GLBL_ConversionCoefficients[Cidx][1]` — the `b` (offset) for conversion set `Cidx`. |
| 6 | Treat PV value as a mailbox index, read `GLBL_PickoffControlVals[PV_val]`, set PV.val and PV.rval to that integer. |
| 7 | Treat PV value as a mailbox index, read `GLBL_PickoffFloatVals[PV_val]`, set PV.val and PV.rval to that float. |

✅ verified 2026-04-24 — `PickoffCtl_AO.c:L143-244` (`write_ao` switch(PV_mode) cases 0–7).

**Note on `PickoffSoftControl` AI:** Reads back from `GLBL_PickoffControlVals[Cidx]` or `GLBL_PickoffFloatVals[Cidx]` (bit 15 of C selects float mode), with same ANDMask and ShiftFactor unpacking logic as the hardware AI.

**Waveform record** (`PickoffSoftControl` waveform): sets `bptr = &GLBL_PickoffDataArray[0]` and `nord = 1024` — exposes the entire 1024-element data buffer as a single waveform PV. Debug print if `GLBL_PickoffControlVals[31]` is set. ✅ verified 2026-04-24 — `PickoffCtl_Waveform.c:L120-132`.

---

## Global Mailboxes

The IOC maintains two global arrays shared across all device support modules:

| Array | Size | Type | Use |
|-------|------|------|-----|
| `GLBL_PickoffControlVals[256]` | 256 shorts | `unsigned short` | Integer mailboxes — alarm enable flags, mode selects, inter-PV state |
| `GLBL_PickoffDataArray[1024]` | 1024 shorts | `unsigned short` | Raw ADC scan data from SPI burst reads |
| `GLBL_PickoffFloatVals[32]` | declared 32, used up to ~157 | `epicsFloat64` | Floating-point mailboxes — converted engineering values (volts, Kelvin, etc.) ✅ verified 2026-04-24 — `PickoffSupport.h:L37` + `PickoffSupport.c:L589` declare `[32]`; `HV_STEP.db:L79-158` uses Cidx=150 (C=0x2096..0x409C) accessing indices 150–156; `PickoffStep_AI.c:L200-205,234,302` accesses Cidx+0 through Cidx+6 with `Cidx = c & 0x00FF` (no bounds check); `PickoffCtl_AO.c:L212` mode-2 write is also unbounded. **Confirmed OOB** — works in practice due to C static global layout (adjacent arrays absorb the overflow). |
| `GLBL_ConversionCoefficients[64][2]` | 64 × {m,b} | `epicsFloat64` | y=mx+b coefficients for 64 different conversion types |
| `PT100Coefficients[10][2]` | 10 segments | `epicsFloat64` | Piecewise linear PT100 resistance-to-temperature conversion |
| `PT500Coefficients[10][2]` | 10 segments | `epicsFloat64` | Piecewise linear PT500 resistance-to-temperature conversion |

### `PickoffCalc` Conversion Types (from `PickoffCalc_AI.c`)

| `ConversionType` | Description |
|-----------------|-------------|
| 0 | Simple `y = mx + b` using `GLBL_ConversionCoefficients[ConversionSet]` ✅ verified 2026-04-25 — `PickoffCalc_AI.c:L43-56` (case 0: `GLBL_ConversionCoefficients[ConversionSet][0/1]` mx+b) |
| 1 | PT100 RTD: ADC counts → resistance → temperature (**6-segment** piecewise linear, indices 0–5) ✅ verified 2026-04-25 — `PickoffCalc_AI.c:L457-481` (PT100Coefficients[0..5][0/1] initialized; 6 coefficient pairs, **not 5** as previously stated) |
| 2 | PT500 RTD: ADC counts → resistance → temperature (**6-segment** piecewise linear, indices 0–5) ✅ verified 2026-04-25 — `PickoffCalc_AI.c:L493-526` (PT500Coefficients[0..5][0/1] initialized; 6 coefficient pairs, **not 5** as previously stated) |
| 3 | Reciprocal: `y = (m/x) + b` using `GLBL_ConversionCoefficients[ConversionSet]` ✅ verified 2026-04-25 — `PickoffCalc_AI.c:L140-154` (case 3: `m/HardwareValue + b`) — **previously undocumented** |

---

## I2C via Pickoff FPGA

The `PickoffI2CSerial` device support sends I2C commands **indirectly** — the Pi writes a command sequence to the Pickoff FPGA's I2C command FIFO via SPI, and the FPGA executes the I2C bus transaction.

**I2C Command FIFO word format (16-bit):** ✅ verified 2026-04-23 — `PickoffSupport.h:L9-17` (defines: DONE=0x8000, RPTS=0x4000, NACK=0x2000, READ=0x1000, SAVE=0x0800, EXTD=0x0400, LOOP=0x0200, TOGS=0x0100); `PickoffI2C_AI.c:L143-168` (format diagram + bit descriptions). Note: DONE+RPTS form a 2-bit pair; DONE alone (RPTS=0) = stop after ACK; DONE+RPTS = stop without ACK; RPTS alone = repeated start; EXTD overrides LOOP.
```
Bit 15: DONE  — if 1, stop after this byte
Bit 14: RPTS  — if 1, do Repeated Start after byte
Bit 13: NACK  — if 1, skip ACK check
Bit 12: READ  — if 1, sample SDA into readback latch
Bit 11: SAVE  — if 1, present READ data with strobe at ACK
Bit 10: EXTD  — if 1, fetch next command immediately (multi-byte, no ACK gap)
Bit  9: LOOP  — if 1, next command is repeated N times (LOOP overridden by EXTD)
Bit  8: TOGS  — if 1, toggle SAVE on each ACK (present data every other byte)
Bits 7:0: DATA — byte to clock on SDA (or loop count if LOOP=1)
```

**`camacio` field mapping for `PickoffI2CSerial`:**
| Field | Use |
|-------|-----|
| `B` | `DetAddr` — routing selector |
| `C` | `I2C_TransferType` — controls how many bytes and what format (0x00-0x0F, 0x10-0x1F, 0x20-0x2F, 0x30-0x3F) |
| `N` | `UsrAddr` — SPI register address for the I2C command FIFO |
| `A` | `I2C_DevAddr` — 7-bit I2C device address (bit 0 = R/W direction) |
| `F` | `I2C_RegAddr` — 8-bit I2C register address |

**I2C transfer type encoding (`C` field):**
| Range | Description |
|-------|-------------|
| `0x00`–`0x0F` | N-byte write: devaddr/regaddr/data×N; ACK on all bytes |
| `0x10`–`0x1F` | N-byte write: devaddr/regaddr/data×N; no ACK on last byte |
| `0x20`–`0x2F` | N-byte read: devaddr(write)/regaddr/Repeated-Start/devaddr(read)/data×N; ACK all |
| `0x30`–`0x3F` | N-byte read: same as 0x20 but no ACK on last byte |

---

## Database Files

| File | Description |
|------|-------------|
| `Pickoff.db` | Core pickoff PVs: GeCenter gain/offset, GeSide mux, BGO attenuation, BGO discriminator thresholds, preamp reset parameters |
| `Pickoff_reg.db` | Register-level access PVs for raw readback/control of Pickoff FPGA registers |
| `SlopeBox.db` | Slope box ADC readbacks: Ge HV, BGO HV, temperature sensors, power supply rails — via Pickoff FPGA ADC scanner |
| `PreampCalcChain.db` | Conversion chain PVs for preamp readback values (ADC → engineering units) |
| `PowerBoardCalcChain.db` | Conversion chain PVs for power board readbacks (voltage rails, currents) |
| `DetSpec.db` | Detector-specific static parameters: `MOD${DetNbr}_DS_GEHV` sets the Ge HV maximum (e.g. 3700V); used as input to HV step control |
| `PickoffDiagCtl.db` | Diagnostic/control PVs for FPGA diagnostic features |
| `HV_STEP.db` | Ge HV ramp logic: controlled step-wise HV adjustment with interlocks; reads GeHV actual, compares to demand, steps toward target |

---

## Startup / IOC Configuration (`st.cmd`)

```shell
epicsEnvSet(EPICS_CA_SERVER_PORT, "5080")    # standalone: 5080/5081
epicsEnvSet(EPICS_CA_REPEATER_PORT, "5081")  # (test stand uses 5074/5075 per comments)

dbLoadDatabase "dbd/Pickoff.dbd"
dbLoadDatabase "dbd/PickoffSupport.dbd"
Pickoff_registerRecordDeviceDriver pdbbase

# All DBs loaded with same DetNbr (e.g. 033)
dbLoadRecords "db/Pickoff_reg.db",         "DetNbr=033", "CollPort=033", "DetType=NoSeg"
dbLoadRecords "db/Pickoff.db",             "DetNbr=033", ...
dbLoadRecords "db/SlopeBox.db",            "DetNbr=033", ...
# ... etc.
```

**Notes:**
- `DetNbr` is the 3-digit GS detector number (e.g. `033`)
- `CollPort` appears to be the Collector Box port (same as DetNbr in standalone mode)
- `DetType=NoSeg` = unsegmented detector (no GeSide A/B distinction)
- CA port 5080/5081 used (not 5064/5065 standard) to avoid conflict when running alongside other IOCs
- IOC must run as **root (`sudo`)** — bcm2835 SPI access requires elevated privileges; EPICS init crashes with segfault otherwise (comment in `PickoffSupport_AO.c`)

---

## HV Ramp Logic (`HV_STEP.db`)

The Ge HV is not simply written to a DAC — it is **ramped step-by-step** via a chain of PVs triggered on each slope box ADC readback:

1. `GS${DetNbr}_SlopeBox_GeHV_Readback` (ai, 1s scan) → reads raw ADC → copies raw to mailbox 41 → triggers `GS${DetNbr}_Conv_GeHV`
2. `GS${DetNbr}_Conv_GeHV` (PickoffCalc) → converts ADC counts to volts → stores in `GLBL_PickoffFloatVals[154]` → triggers `GS${DetNbr}_StepInterlock1`
3. `GS${DetNbr}_StepInterlock1` → evaluates interlock conditions → writes to mailbox 125 → triggers `GS${DetNbr}_StepInterlock2`
4. `GS${DetNbr}_StepInterlock2` → evaluates more interlocks → chains to step PV
5. `GS${DetNbr}_Adjust_HV_DAC` (PickoffStep) → reads demand from `GLBL_PickoffFloatVals[150]` (set by `GS${DetNbr}_GE_HV_DEMAND_VOLTS`), stepsize from [151], hysteresis from [152], absmax from [153], actual HV from [154]; interlock disable from `GLBL_PickoffControlVals[125]` (F mask 0xFFFF = any bit stops stepping) → computes one step → stores new voltage value → triggers `GS${DetNbr}_GE_HV_DEMAND_DAC` (converts volts→DAC counts via conversion set 27) → triggers `GS${DetNbr}_MANUAL_GE_HV_DEMAND` → writes DAC at SPI register 84 ✅ verified 2026-04-23 — `HV_STEP.db:L78` (`INP #B0 C0x8096 N0 A0x007D F0xFFFF @X`; Cidx=0x96=150, Aidx=0x7D=125); `PickoffStep_AI.c:L200-205` (FloatVals[Cidx..Cidx+4] layout)

**Interlock behavior:** If `GLBL_PickoffControlVals[125]` ANDed with F=0xFFFF is nonzero → HV adjustment suppressed. ✅ verified 2026-04-23 — `PickoffStep_AI.c:L178` (`GLBL_PickoffControlVals[Aidx] & PtrToLinkStruct->f`); Aidx from A=0x007D=125.

**User-settable HV parameters (all in Volts, defaults):**
| PV | Default | FloatVals index | Description |
|----|---------|-----------------|-------------|
| `GS${DetNbr}_GE_HV_DEMAND_VOLTS` | (user sets) | [150] | Desired HV target |
| `GS${DetNbr}_GE_HV_STEP_SIZE` | 5 V | [151] | Max V change per cycle |
| `GS${DetNbr}_GE_HV_HYSTERESIS` | 5 V | [152] | Dead-band; no step if \|actual−demand\| < HYSTERESIS |
| `GS${DetNbr}_GE_HV_ABSMAX` | 3700 V | [153] | Hard ceiling — will not step above this |
| (actual, written by Conv_GeHV) | — | [154] | Current HV in volts |

---

## Detector Specification (`DetSpec.db`)

`MOD${DetNbr}_DS_GEHV` — ao record (type `PickoffSoftControl`, OUT `C=0x4203`):
- Default **3700V** (comment notes GS081/sbxY/Ge60 uses 4500V max)
- C=0x4203 → `PV_mode=(0x4203 & 0x7000)>>12 = 4`, `Cidx=0x4203 & 0x3F = 3` → writes PV value to `GLBL_ConversionCoefficients[3][0]` (the `m` slope for conversion set 3) ✅ verified 2026-04-23 — `PickoffCtl_AO.c:L224-226` (mode 4: `Cidx = c & 0x003F; ConversionCoefficients[Cidx][0] = PV.val`); `HV_STEP.db:L6` (C=0x4203). Note: the DB comment "save to GLBL_PickoffControlVals[2][3]" is incorrect — mode 4 writes to ConversionCoefficients, not ControlVals.
- This PV is NOT the demand source for `Adjust_HV_DAC`. The actual demand comes from `GS${DetNbr}_GE_HV_DEMAND_VOLTS` at `GLBL_PickoffFloatVals[150]`.
- Alarm monitoring PV `MOD${DetNbr}_DV_GEHV` reads `GLBL_PickoffFloatVals[14]` (actual HV in volts) and applies alarms: HIHI/LOLO at ±10%, HIGH/LOW at ±5% ✅ verified 2026-04-23 — `DetSpec.db:L22-38` (INP C=0x800E → bit15 set = float mode, Cidx=14; alarm thresholds hardcoded)

---

## Relationship to Collector Box Pi IOC

The `PickoffApp_RevC` (PickoffPi) and the Collector Box Pi IOC (`collectorboxpi`) share the same fundamental approach:
- Both use `bcm2835` library for SPI
- Both use `CAMAC_IO` link type for PVs
- Both use the same global mailbox pattern for inter-PV state sharing
- Key difference: `PickoffApp_RevC` has **one Pi per detector** (direct SPI); Collector Box Pi handles **up to 28 detectors** (multiplexed via Collector Box FPGA routing using GPIO `DetAddr`)

The `DetAddr` (`B` field) = 0 in PickoffPi PVs because there is no routing needed — the Pi is wired directly to a single Pickoff FPGA.

---

## PickoffSupportBackup.c — Historical / Educational Version

`PickoffSupportBackup.c` (1025 lines) is an **older, heavily-commented educational version** of the device support code preserved alongside the active `PickoffSupport*.c` files. It is **not compiled into the IOC** — the `Makefile` only includes `PickoffSupport.c` and its siblings. Its primary value is as documentation of the design reasoning.

### What Makes It Different

The backup uses a **different `camacio` field mapping** from the current code:

| Field | Backup (`PickoffSupportBackup.c`) | Current (`PickoffSupport_AI.c` etc.) |
|-------|----------------------------------|---------------------------------------|
| `b` | `DetAddr` — GPIO/routing byte (which detector in a collector-box scenario) | Unused in PickoffPi (always 0) |
| `c` | `TransactionLength` — # of bits in SPI transaction (nominally 24) | Mode + mailbox index (e.g. `0x8096`) |
| `n` | Combined RW flag + 7-bit address: upper byte nonzero → RW=1, lower byte = addr | 7-bit SPI register address only; RW direction hardcoded by record type |
| `a` | AND mask for read-modify-write | AND mask (bit-field extraction) |
| `f` | OR mask | Shift factor (bits[3:0]=amount, bit15=direction) |

In the backup design, a GPIO `DetAddr` field (`b`) was reserved for the case where the same device support could multiplex to multiple detectors through a routing FPGA — exactly what the **Collector Box Pi** does. In `PickoffApp_RevC` this was simplified to `b=0` always (direct single-detector connection), and the field repurposing evolved into the current design.

### Exported Structs

The backup exports two device structs (but they are **not active** in the build):
- **`devJTAx`** — `ai` read support (`"PickoffLocalSerial"`)
- **`devAoPickoff`** — `ao` write support (`"PickoffLocalSerial"`)

The real active build exports a richer set via `PickoffSupport.c` (`devPickoffLocalSerial`, `devPickoffSoftControl`, etc.).

### Educational Content

The file contains extensive inline tutorials (pp. 1–580 are mostly comments) explaining:
- Why `asyn` is not used (identical rationale to current code + `collectorbox_devicesupport.md`)
- How EPICS `CAMAC_IO` / `camacio` struct works from first principles
- How `device()` + `epicsExportAddress()` wiring works
- How `init_record` → `read_ai` / `write_ao` call sequence works
- A worked example of a `VME_IO` device support (from Gretina) for comparison

This file is the best single reference for **understanding the DGS Pi EPICS device support design philosophy**.

---

## Cross-References

- [`hardware_architecture.md`](hardware_architecture.md) — Gammasphere hardware overview; slope box and pickoff card context
- [`sbx.md`](sbx.md) — Slope Box Extension (SBX) hardware: signal chain, BGO pattern/sum, GS_ID dongle, HV map
- [`pickoff_card_fpga.md`](pickoff_card_fpga.md) — Pickoff Card FPGA (SBX Extension RevC, Spartan-6): the hardware this IOC communicates with via SPI; full 128-register map, PULSED_CONTROL/FPGA_CTL bit maps
- [`deep_fpga_SBX_CtrlFPGA.md`](deep_fpga_SBX_CtrlFPGA.md) — SBX Motherboard Control FPGA (Spartan-6): 24-bit SPI interface, 128-addr register file, I2C buses, BGO disc/DDR outputs, preamp reset clamp
- [`collectorboxpi.md`](collectorboxpi.md) — Collector Box Pi IOC: shares same CAMAC_IO/global-mailbox/SPI pattern as PickoffPi
- [`slope_box_interface.md`](slope_box_interface.md) — SBX slope box interface; PickoffLocalSerial framework shared with PickoffPi
- [`EPICS_DB_templates.md`](EPICS_DB_templates.md) — EPICS DB template system; same record types (ao, ai) used in DetSpec.db and HV_STEP.db
- [`collectorbox_devicesupport.md`](collectorbox_devicesupport.md) — Device support layer; `CAMAC_IO` link type shared with PickoffPi

*Source: `DGS_tools_pack/DGS_SVN/psg/` (PickoffApp_RevC). Created: 2026-04-23.*
