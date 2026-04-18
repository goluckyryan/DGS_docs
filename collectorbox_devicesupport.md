# CollectorBox Device Support — EPICS Driver Internals

Source: `DGS_tools_pack/collectorboxpi/CollectorBox_RevA/CollectorApp/src/`
✅ Key claims verified 2026-04-06 against `CollectorSupport.c` (lines 30–326): 24-bit SPI transactions, CAMAC_IO/camacio link structure, asyn rejection rationale.

---

## Why Not asyn?

The device support is custom C — **asyn is explicitly rejected** (see comments in `CollectorSupport.c`):
- asyn assumes byte-granularity; collector transactions are 24-bit (not byte-aligned)
- True SPI requires simultaneous MOSI/MISO — asyn doesn't support this
- Transaction length is FPGA-defined and can be arbitrary

---

## EPICS Device Support Architecture

### How PVs Connect to Hardware

EPICS uses a **device support function table** pattern:

```c
struct {
    long         number;
    DEVSUPFUN    report;
    DEVSUPFUN    init;
    DEVSUPFUN    init_record;   // ← called once at IOC start
    DEVSUPFUN    get_ioint_info;
    DEVSUPFUN    read/write;    // ← called on each PV process
    DEVSUPFUN    special_linconv;
} devXxxCollector = { 6, NULL, NULL, init_record_xxx, NULL, action_xxx, NULL };
epicsExportAddress(dset, devXxxCollector);
```

This table is registered in `CollectorSupport.dbd`:
```
device(ao, CAMAC_IO, devAoCollector, "Collector Local Serial")
device(ai, CAMAC_IO, devAiCollector, "Collector Local Serial")
...
```

**DTYP field in .db files** must match the string (e.g. `"Collector Local Serial"`) to route PV processing to the correct handler.

### Link Structure Used: CAMAC_IO

Uses the **camacio** EPICS link structure (not VME_IO) to embed SPI transaction parameters:

```c
struct camacio {
    short b;    // DEVSEL (GPIO bus value) — which device to select
    short c;    // transaction length in bits (always 24)
    short n;    // bits 7:0 = register address; bit 7 = R/W flag
    short a;    // AND mask (for bi/bo/mbbi/mbbo bit extraction)
    short f;    // OR mask / shift
    char  *parm;
};
```

In `.db` files this appears as:
```
field(OUT, "#<b> <c> <n> <a> <f> @")
```
Example: `OUT("#5 24 20 0 0 @")` → DEVSEL=5, 24-bit, addr=20, write

### Record Init Flow
1. `init_record_xxx()` called once at IOC load
2. Casts `pai->inp.value` (or `pao->out.value`) to `struct camacio *`
3. Extracts b/c/n/a/f into local vars
4. Stores in `pai->dpvt` (device private pointer) if needed for bit games (mbbi/mbbo/bi/bo)

### Record Process Flow
1. `read_xxx()` / `write_xxx()` called on scan or CA put
2. Sets DEVSEL via `Set_DEVSEL(b)`
3. Calls `Do_SPI1_transaction(RWflag, addr, data)`
4. For reads: stores result back into PV's `rval`/`val` field
5. For bi/bo/mbbi/mbbo: applies AND/OR masks to extract/set individual bits

---

## Global Data Structure

Initialized once at IOC start via `GetDataArrayPtr(1)`:

```c
typedef struct {
    unsigned short GLBL_CollectorDataArray[32][1024];   // raw scan data per device ✅ verified 2026-04-08 — CollectorSupport.h:L30
    unsigned short GLBL_CollectorControlVals[32][256];  // integer mailboxes per device
    epicsFloat64   GLBL_CollectorFloatVals[32][256];    // float mailboxes per device
    unsigned short *GLBL_CollectorArrayPtr[32];         // walking pointers into DataArray
    epicsFloat64   GLBL_ConversionCoefficients[64][2]; // [idx][0]=slope, [idx][1]=offset
    epicsFloat64   PT100Coefficients[5];               // RTD temperature fit
    epicsFloat64   PT500Coefficients[5];               // RTD temperature fit (500Ω)
} CollectorGlobDataStructure;
```

First index (32) = DEVSEL device number. All PVs for the same device share the same mailbox arrays.

---

## Conversion Coefficients Table

Loaded by `InitializeCoefficients()` at IOC init. Used by `CollectorCalc` PVs.

| Index | Signal | Conversion |
|-------|--------|------------|
| 0 | Sandbox | y = x (passthrough) |
| 1 | ADC450 voltage | V = (500/4096) × count ✅ verified 2026-04-07 — CollectorSupport.c:L680 |
| 2 | Ge HV | V = (5000/4096) × count ✅ verified 2026-04-07 — CollectorSupport.c:L685 |
| 3 | ADC400 voltage | V = (500/4096) × count ✅ verified 2026-04-07 — CollectorSupport.c:L690 |
| 4 | +24V supply | V = [5/(4096×0.149)] × count ✅ verified 2026-04-07 — CollectorSupport.c:L695 (909Ω/6019Ω divider) |
| 5 | +12V supply | V = [5/(4096×0.338)] × count ✅ verified 2026-04-07 — CollectorSupport.c:L700 (1K/2.96K divider) |
| 6 | -12V supply | V = -[5/(4096×0.348)] × count ✅ verified 2026-04-07 — CollectorSupport.c:L704 (4.22K/12.1K divider) |
| 7 | +5V supply | V = [5/(4096×0.808)] × count ✅ verified 2026-04-07 — CollectorSupport.c:L708 (422/522 divider) |
| 8 | CenterFET bias | V = [11/(125×32768)] × count ✅ verified 2026-04-16 — CollectorSupport.c:L717 (sign flipped 20220223 per M. Oberling; stored as positive slope) |
| 9 | CenterFET current | I = [11/(5×32768)] × count (mA) ✅ verified 2026-04-16 — CollectorSupport.c:L721 (stored as positive; sign flip noted in comment only) |
| 10,11 | CenterFET/SideFET Vds | V = [1249/(31125×32768)] × count ✅ verified 2026-04-16 — CollectorSupport.c:L724,L728 |
| 12,13 | U10 gain 1X/4X offset | passthrough (slope=1.0, offset=0) ✅ verified 2026-04-16 — CollectorSupport.c:L731-735 |
| 14 | SideFET bias | V = [11/(125×32768)] × count ✅ verified 2026-04-16 — CollectorSupport.c:L739 |
| 15 | SideFET current | I = -[11/(5×32768)] × count (mA) ✅ verified 2026-04-16 — CollectorSupport.c:L742 (slope stored as negative) |
| 16,17 | Side B/A offset voltage | V = [2249/(31125×131072)] × count ✅ verified 2026-04-16 — CollectorSupport.c:L745,L749 |
| 18,19 | U7 gain 1X/4X offset | passthrough (slope=1.0, offset=0) ✅ verified 2026-04-16 — CollectorSupport.c:L752-756 |
| 20 | Enclosure temp | T = (175/65535)×count − 45 (°C) ✅ verified 2026-04-17 — CollectorSupport.c:L763-765 (slope=[0]=175/65535, offset=[1]=−45.0) |
| 21 | Enclosure humidity | H = (100/65535)×count (%) ✅ verified 2026-04-17 — CollectorSupport.c:L767-768 |
| 22 | PCB temp | T = count/256 (°C) ✅ verified 2026-04-17 — CollectorSupport.c:L770-772 (coeff[22][0]=1.0/256.0) |
| 23–26 | Fan RPM | RPM = 675000/(count×N), N=1,2,4,8 ✅ verified 2026-04-07 — CollectorSupport.c:L778-787 (MAX6653 datasheet p.12) |
| 27 | Slope box DAC | DAC = volts × (4096/5000) ✅ verified 2026-04-16 — CollectorSupport.c:L800 |
| 28 | °C → Kelvin | K = C + 273 (slope=1, offset=273) ✅ verified 2026-04-16 — CollectorSupport.c:L807-809 (added 20230309, previously unused) |
| 29 | Power board temp | T = count × 0.125 (°C) ✅ verified 2026-04-16 — CollectorSupport.c:L813-815 |
| 30 | Collector remote temp | T = count × 0.125 + 17 (°C) ✅ verified 2026-04-16 — CollectorSupport.c:L817-819 (empirical offset) |
| 31 | Timestamp (bits 26:11) | ms = count × 0.02048 ✅ verified 2026-04-16 — CollectorSupport.c:L821-824 (bit 11 = 20.48 µs at 100 MHz) |

### PT100 / PT500 Temperature Fit (2-step)
1. **ADC → Resistance:** R = slope×count + intercept
2. **R → Temperature:** T = a0 + a1×R + a2×R² (CVD polynomial)

| Sensor | ADC→R slope | ADC→R intercept | a0 | a1 | a2 |
|--------|------------|-----------------|----|----|-----|
| PT100 | 0.054267 | 1.0284 | −242.362 | 2.25179 | 1.84598×10⁻³ | ✅ verified 2026-04-07 — CollectorSupport.c:L843-848 (empirical, slope box #92, 2023-03-09) |
| PT500 | 0.041199 | 0.0 | −242.362 | 0.45036 | 7.38393×10⁻⁵ | ✅ verified 2026-04-07 — CollectorSupport.c:L872-877 (derived: PT100 coeffs scaled by 1/5, 1/25) |

---

## Source Files Overview

## Two-Layer PV Architecture

There are **two separate device support layers**, both using CAMAC_IO but different DTYP strings:

| DTYP String | Files | What it does |
|-------------|-------|--------------|
| `"Collector Local Serial"` | `CollectorSupport_AO/AI/BI/BO/MBBI/MBBO.c` | **Directly drives SPI** — writes/reads hardware registers |
| `"CollectorSoftControl"` | `CollectorCtl_AO/AI/BI/BO/MBBI.c` | **Writes to RAM mailboxes only** — no SPI, used for staging control values |

PVs that need to send data to hardware use `"Collector Local Serial"`. PVs that just store operator settings use `"CollectorSoftControl"`.

---

## CollectorSupport_AO — Full SPI Write Flow

When a CA client writes an ao PV tagged `DTYP = "Collector Local Serial"`:

1. `write_ao()` is called with the aoRecord pointer
2. Extract camacio fields: `Bidx`=B&0x1F (DEVSEL), `UsrAddr`=N&0x7F (register), `AndMask`=A, `ShiftFactor`=F&0xF
3. `MailboxMode` = bits 13:12 of C field:

| Mode | Address source | Data source |
|------|---------------|-------------|
| 0 | N field (direct) | PV val (shifted) |
| 1 | N field (direct) | PV val (shifted) + copy to mailbox |
| 2 | N field (direct) | Mailbox`[Bidx][Cidx]` (indirect data) |
| 3 | Mailbox`[Bidx][Cidx]` (indirect addr) | PV val (shifted) |

4. **If `parm` string after `@` is empty → Read-Modify-Write:**
   - SPI read: `Do_SPI1_transaction(1, Bidx, UsrAddr, 0)` → `SPI_data_in`
   - Strip upper bits: `SPI_data_out = SPI_data_in & 0xFFFF`
   - AND with mask: `SPI_data_out = SPI_data_out & AndMask`
   - OR user data: `SPI_data_out = SPI_data_out | UsrData`
5. **If `parm` is non-empty → Write-Only:**
   - `SPI_data_out = UsrData | AndMask` (A field acts as OR mask here)
6. SPI write: `Do_SPI1_transaction(0, Bidx, UsrAddr, SPI_data_out)`
7. If mode 1: copy result to mailbox

**Init behavior:** On IOC start, every RMW-style ao PV does a hardware READ to pre-populate its value. Write-only PVs initialize to 0.

---

## CollectorCtl_AO — Mailbox Write Flow

For `DTYP = "CollectorSoftControl"` ao PVs — **no SPI involved**. C bits 14:12 select mode:

| Mode | Action |
|------|--------|
| 0 | Write PV → integer mailbox `[Bidx][Cidx]` (optional RMW with AND/shift) |
| 1 | Write PV → current buffer pointer location in DataArray |
| 2 | Write PV → float mailbox `[Bidx][Cidx]` |
| 3 | Set buffer pointer to PV value |
| 4 | Write PV → conversion coefficient `[Cidx][0]` (slope m) |
| 5 | Write PV → conversion coefficient `[Cidx][1]` (offset b) |
| 6 | Use PV value as index → read integer mailbox back into PV |
| 7 | Use PV value as index → read float mailbox back into PV |

Debug prints enabled when `GLBL_CollectorControlVals[Bidx][0] != 0` (mailbox[device][0] is a debug flag).

---

## CollectorDPRSupport_AI — DPRAM Read Flow

`DTYP = "CollectorDPRAM"` ai PVs — reads from the CtrlFPGA dual-port RAM (bank 1, addrs 128-255) via SPI.

**CAMAC_IO field mapping:**
- `B` bits[4:0] -> `Bidx` (DEVSEL, 0-31)
- `N` bits[6:0] -> `RegisterAddr` (register within bank)
- `N` bits[9:7] -> `Bank` (bank select; 0 = default, else write bank# to addr 127 first)
- `C` bits[14:12] -> `MailboxMode` (0-7, see table)
- `C` bits[7:0] -> `Cidx` (mailbox index for modes 1-5)
- `A` -> `AndMask` (applied after read; 0 treated as 0xFFFF)
- `F` bits[3:0] -> `ShiftFactor`; bit[15]=1 -> shift left, bit[15]=0 -> shift right

**Mailbox modes** (C bits 14:12): verified 2026-04-08 against CollectorDPRSupport_AI.c:L221-231

| Mode | Description |
|------|-------------|
| 0 | Single read. N = address. PV gets masked+shifted data. No mailbox copy. |
| 1 | Single read. N = address. PV gets data AND data copied to mailbox[Bidx][Cidx]. |
| 2 | Single read. N = address. Data goes to mailbox only. PV gets FPGA status byte (bits[23:16] of SPI transaction). |
| 3 | Single read. Mailbox[Bidx][Cidx] provides the address (indirect). PV gets data. No mailbox copy. |
| 4 | Loop read, fixed address. N = address (constant). Mailbox = loop count. Results stored sequentially in DataArray from current buffer pointer. PV gets last read value. |
| 5 | Loop read, incrementing address. N = start address, increments each iteration. Mailbox = loop count. Results stored in DataArray. PV gets last read value. |
| 6-7 | AndMask limited to 8 bits (`A & 0x00FF`). **No SPI read executed** — no `case 6:` or `case 7:` exists in the main read switch; these modes appear **unimplemented**. ✅ verified 2026-04-08 — `CollectorDPRSupport_AI.c:L179-184` (AndMask set) + switch body (no case 6/7 handlers). |

**Bank select:** If `Bank != 0`, a write to addr 127 with the bank number is performed before the data read. Used to access CtrlFPGA DPRAM bank 1 (ADC scan results, addrs 128-255). Verified: CollectorDPRSupport_AI.c:L145

**SPI transaction:** `Do_SPI1_transaction(RWflag, Bidx, addr, data)` returns 32 bits: bits[23:16] = FPGA status byte, bits[15:0] = 16-bit read data.

---

## What Pi Source Can vs Cannot Answer

**Fully answered by Pi source code:**
- SPI transaction format: 24-bit `[R/W|Addr(7)][Data(15:8)][Data(7:0)]` ✅ verified 2026-04-17 — `NonEPICS_SPI_lib.c:L283-285` (`spi_message[0]=(RWflag<<7)+(Addr&0x7F)`, `[1]=Data>>8`, `[2]=Data&0xFF`)
- DEVSEL bus: 5-bit GPIO selects up to 32 devices on the SPI bus ✅ verified 2026-04-17 — `NonEPICS_SPI_lib.c:L66-71,L159-175` (DEVSEL(4:0) = GPIO26/25/24/23/13; DEVSEL31 is the max value)
- Two-layer architecture (mailbox vs direct SPI)
- Read-modify-write vs write-only distinction (empty vs non-empty `parm`)
- Indirect address/data modes via mailboxes
- Init: ao PVs pre-read hardware on IOC start
- Upper 8 bits of SPI return value are FPGA status bits (stripped before use)

**Requires FPGA code to answer:**
- Register map: what each address (N field) actually controls in the FPGA
- DEVSEL routing: how the FPGA decodes DEVSEL to select sub-circuits
- FPGA status bits (upper 8 of 24-bit return): what they mean
- Exact timing/sequencing requirements for multi-step operations

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
| `bcm2835.c` / `.h` | Raspberry Pi GPIO/SPI library |

---
*Source: `DGS_tools_pack/collectorboxpi/CollectorBox_RevA/CollectorApp/src/` — C device support source. Created: 2026-04-05.*

## Cross-References

- `knowledgeBase/collectorboxpi.md` — Raspberry Pi soft IOC: PXE boot, HV control, collector assignments
- `knowledgeBase/collector_fpga.md` — CtrlFPGA and StripeFPGA firmware; SPI register maps
- `knowledgeBase/collectorbox_PVs.md` — Full PV list (1,431 records/detector)
- `knowledgeBase/sbx.md` — Slope Box Extension; pickoff card; BGO HV; GS_ID dongle
- `knowledgeBase/EPICS.md` — EPICS record types and device support concepts
