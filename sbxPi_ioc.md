# SBX Pi IOC — PickoffApp_RevC

**Source:** `DGS_tools_pack/sbxPi/PickoffApp_RevC/`  
**Last updated:** 2026-04-22  
**See also:** [`sbx.md`](sbx.md) — SBX hardware + Collector Box Pi IOC overview

---

## Purpose

The `PickoffApp_RevC` is a standalone EPICS soft IOC designed to run on a **Raspberry Pi mounted directly on the SBX (Slope Box Extension)** for single-detector operation. It controls and monitors one detector (GeCenter, GeSide, BGO sum/pattern, Ge HV, BGO HV, slope box ADC readbacks, power board, preamp) via direct **SPI1** communication from the Pi to the Pickoff Card FPGA.

This differs from the full system where the **Collector Box Pi** IOC handles up to 28 detectors via a shared SPI bus. The sbxPi IOC is used for:
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

**Clock speed:** Set via `RequestedSpeed` parameter passed into `SPI1_setup()`. The `bcm2835_aux_spi_setClockDivider()` formula: `SPICLKfreq = 250MHz / (2 × (speed+1))`.

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
| `N` | SPI address + R/W bit (bit 7 = R/W; bits 6:0 = 7-bit register address) |
| `A` | AND mask (for bo/bi/mbbi/mbbo bit-field extraction) |
| `F` | OR mask / shift factor |
| `parm` | Additional parameters after `@` in OUT/INP string |

**Example PV OUT field (ao write to register 0x14, 24-bit, local Pi):**
```
OUT(0, 24, 20, 0, 0, "")
      ^   ^   ^
      B=0 C=24 N=0x14 (address 20, R/W=0 → write)
```

### Device Support Identifiers (from `.dbd`)

| Identifier | Record Types | Description |
|-----------|-------------|-------------|
| `PickoffLocalSerial` | ao, ai, bo, bi, mbbi, mbbo | Direct SPI1 transaction to/from Pickoff FPGA |
| `PickoffI2CSerial` | ai | I2C transaction via Pickoff FPGA I2C command FIFO |
| `PickoffCalc` | ai | Conversion chain — applies y=mx+b or PT100/PT500 polynomial to raw ADC values |
| `PickoffSoftControl` | ao, ai | Software mailbox r/w — no hardware transaction, reads/writes `GLBL_PickoffControlVals[]` or `GLBL_PickoffFloatVals[]` |
| `PickoffStep` | ai | Controlled step-wise adjustment (used for HV ramp logic) |

---

## Global Mailboxes

The IOC maintains two global arrays shared across all device support modules:

| Array | Size | Type | Use |
|-------|------|------|-----|
| `GLBL_PickoffControlVals[256]` | 256 shorts | `unsigned short` | Integer mailboxes — alarm enable flags, mode selects, inter-PV state |
| `GLBL_PickoffDataArray[1024]` | 1024 shorts | `unsigned short` | Raw ADC scan data from SPI burst reads |
| `GLBL_PickoffFloatVals[32]` | 32 doubles | `epicsFloat64` | Floating-point mailboxes — converted engineering values (volts, Kelvin, etc.) |
| `GLBL_ConversionCoefficients[64][2]` | 64 × {m,b} | `epicsFloat64` | y=mx+b coefficients for 64 different conversion types |
| `PT100Coefficients[10][2]` | 10 segments | `epicsFloat64` | Piecewise linear PT100 resistance-to-temperature conversion |
| `PT500Coefficients[10][2]` | 10 segments | `epicsFloat64` | Piecewise linear PT500 resistance-to-temperature conversion |

### `PickoffCalc` Conversion Types (from `PickoffCalc_AI.c`)

| `ConversionType` | Description |
|-----------------|-------------|
| 0 | Simple `y = mx + b` using `GLBL_ConversionCoefficients[ConversionSet]` |
| 1 | PT100 RTD: ADC counts → resistance → temperature (5-segment piecewise linear) |
| 2 | PT500 RTD: ADC counts → resistance → temperature (5-segment piecewise linear) |

---

## I2C via Pickoff FPGA

The `PickoffI2CSerial` device support sends I2C commands **indirectly** — the Pi writes a command sequence to the Pickoff FPGA's I2C command FIFO via SPI, and the FPGA executes the I2C bus transaction.

**I2C Command FIFO word format (16-bit):**
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
5. `GS${DetNbr}_Adjust_HV_DAC` (PickoffStep) → reads demand (from `MOD${DetNbr}_DS_GEHV`), current (mailbox 154), and interlock state (mailbox 125) → computes one step → writes new DAC value

**Interlock behavior:** If mailbox 125 is nonzero → HV adjustment suppressed.

---

## Detector Specification (`DetSpec.db`)

`MOD${DetNbr}_DS_GEHV` — ao record (type `PickoffSoftControl`):
- Static demand: default **3700V** (comment notes GS081/sbxY/Ge60 uses 4500V max)
- Stored to `GLBL_PickoffControlVals[2][3]` AND `GLBL_PickoffFloatVals[13]`
- Alarm monitoring PV `MOD${DetNbr}_DV_GEHV` reads `GLBL_PickoffFloatVals[14]` (actual HV in volts) and applies alarms: HIHI/LOLO at ±10%, HIGH/LOW at ±5%

---

## Relationship to Collector Box Pi IOC

The `PickoffApp_RevC` (sbxPi) and the Collector Box Pi IOC (`collectorboxpi`) share the same fundamental approach:
- Both use `bcm2835` library for SPI
- Both use `CAMAC_IO` link type for PVs
- Both use the same global mailbox pattern for inter-PV state sharing
- Key difference: `PickoffApp_RevC` has **one Pi per detector** (direct SPI); Collector Box Pi handles **up to 28 detectors** (multiplexed via Collector Box FPGA routing using GPIO `DetAddr`)

The `DetAddr` (`B` field) = 0 in sbxPi PVs because there is no routing needed — the Pi is wired directly to a single Pickoff FPGA.
