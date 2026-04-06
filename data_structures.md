# DGS Data Structures — Binary Output Format

_Source: `dgs_analysis/armory/fastEventContructor/class_Hit.h`, `class_DIG.h`, `class_TDC.h`_

The IOC sender (`tcpReceiverMT`) writes raw binary files in **GEB (GRETINA Event Builder) format**. Each event = one `GEBHeader` + payload.

---

## 1. GEBHeader (16 bytes)

Every event starts with a fixed 16-byte header:

```
Offset  Size  Field                  Description
──────────────────────────────────────────────────────
 0      4     type                   GEB type code (see table below)
 4      4     payload_length_byte    Payload size in bytes
 8      8     timestamp              64-bit timestamp (10 ns ticks from MTRG)
```

**GEB Type codes used by DGS:**

| Code | Constant | Description |
|------|----------|-------------|
| 14 | `GEB_TYPE_DGS` | DGS digitizer hit data |
| 15 | `GEB_TYPE_DGSTRIG` | DGS trigger data (TAC-II TDC) |
| 99 | *(internal)* | Decoded TAC2 trigger (used internally by fastEventConstructor) |

Other type codes (1–13, 16–23) are GRETINA/NSCL/S800/aux detector formats — present in merged GEB streams but not produced by DGS IOC directly.

---

## 2. DIG Event Payload (GEB type 14)

The DIG payload is produced by the digitizer FPGA and forwarded by the IOC sender. It is decoded into the `DIG` class. Payload is big-endian 32-bit words (`ntohl()` applied on read).

### Header Words (always present)

```
Word  Bits    Field                Notes
────────────────────────────────────────────────────────────────────────
 0    31:0    0xAAAAAAAA           Fixed sync word (always 0xAAAAAAAA)
 1    3:0     CH_ID                Channel index 0–9
 1    15:4    USER_DEF             User-defined field
 1    26:16   PACKET_LENGTH        Total packet length in 32-bit words
 1    31:27   GEO_ADDR             VME geographic address (slot number)
 2    31:0    EVENT_TIMESTAMP[31:0] Lower 32 bits of 48-bit event timestamp
 3    15:0    EVENT_TIMESTAMP[47:32] Upper 16 bits of 48-bit event timestamp
 3    19:16   HEADER_TYPE          Data format type (see below)
 3    20      PEQ_BYPASS           1 = PEQ energy filter bypassed
 3    21      TRIG_TS_MODE         Trigger timestamp mode
 3    22      CFD_ESUM_MODE        1 = CFD energy sum mode active
 3    25:23   EVENT_TYPE           Trigger type code (bits 10:8 of trigger accept)
 3    31:26   HEADER_LENGTH        Header length in 32-bit words
```

### HEADER_TYPE Values

| Value | Mode | Description |
|-------|------|-------------|
| 1 | LED minimal | LED discriminator, minimal header |
| 2 | LED minimal+trace | LED + waveform trace |
| 3 | LED full | LED + full timing data |
| 4 | LED full+trace | LED + full timing + trace |
| 5 | LED+E | LED + energy sums |
| 6 | LED+E+trace | LED + energy sums + trace |
| 7 | LED standard | Standard LED mode (most common) |
| 8 | CFD standard | Standard CFD mode |

### Word 4 — Flag Word

```
Bit   Field                    Notes
────────────────────────────────────────────────────────────────
 4    EARLY_PRE_RISE_SELECT    Early pre-rise window select
 5    WRITE_FLAGS              Write flags from firmware config
 6    VETO_FLAG                Event was vetoed by BGO shield
 7    TIMESTAMP_MATCH_FLAG     CFD only: timestamp match
 8    EXTERNAL_DISC_FLAG       External discriminator fired
 9    PEAK_VALID_FLAG          Peak detection valid
10    OFFSET_FLAG              Offset correction applied
11    CFD_VALID_FLAG           CFD only: CFD interpolation valid
14    PILEUP_ONLY_FLAG         Event is pileup-only
15    PILEUP_FLAG              Pileup detected
```

### Words 5–13 — Energy, Timing, Multiplex

```
Word  Bits    Field                     Notes
────────────────────────────────────────────────────────────────────────────
  5   29:16   CFD_SAMPLE_0              CFD only: interpolation sample 0
  6   23:0    SAMPLED_BASELINE          Baseline at trigger time
  6   27:24   PILEUP_COUNT              Pileup count (LED) or bits[1:0] (CFD)
  7   15:0    TRIG_MON_XTRA_DATA        LED only: trigger monitor extra
  7   13:0    CFD_SAMPLE_1              CFD only: interpolation sample 1
  7   29:16   CFD_SAMPLE_2              CFD only: interpolation sample 2
  8   23:0    PRE_RISE_ENERGY           Energy before rise (trapezoidal filter)
  8   31:24   POST_RISE_ENERGY[7:0]     Lower 8 bits of post-rise energy
  9   15:0    POST_RISE_ENERGY[23:8]    Upper 16 bits of post-rise energy
  9   31:16   PEAK_TIMESTAMP            16-bit peak timestamp (ns within event)
 10   13:0    P2_SUM[13:0]              P2 sum lower bits
 10   14      P2_MODE                   P2 mode flag
 10   15      CAPTURE_PARST_TS          Capture PARST timestamp
 10   31:16   TS_OF_TRIGGER             16-bit timestamp of trigger relative to event
 11   23:0    MULTIPLEX_DATA            Multiplex data field
 11   31:24   LAST_POST_RISE_M_SUM[7:0] Last post-rise M sum lower 8 bits
 12   23:0    EARLY_PRE_RISE_ENERGY     Early pre-rise energy sum
 12   31:24   LAST_POST_RISE_M_SUM[15:8] Last post-rise M sum upper 8 bits
 13    9:0    P2_SUM[23:14]             P2 sum upper bits
 13   10      SECOND_THRESH_DISC_FLAG   Second threshold discriminator flag
 13   11      PARST_TSM                 PARST timestamp match
 13   12      PREVIOUS_CFD_VALID        Previous CFD valid (CFD mode only)
 13   13      COARSE_FIRED              Coarse discriminator fired
 13   23:14   TS_OF_COARSE              10-bit coarse discriminator timestamp
 13   31:24   LAST_POST_RISE_M_SUM[23:16] Last post-rise M sum upper 8 bits
```

### Words 14+ — Waveform Trace (optional, types 2/4/6)

Raw 16-bit ADC samples packed into 32-bit words (2 samples per word):
- ADC data = 14-bit unsigned offset binary (0 = most negative, 0x3FFF = most positive)
- Bit 14 = timing mark
- Bit 15 = downsampling marker
- Number of trace words = `(PACKET_LENGTH - HEADER_LENGTH)` words

---

## 3. TAC-II / TDC Payload (GEB type 15)

The MTRG TAC-II TDC produces trigger timing data decoded into the `TDC` class:

```
Field                Type     Description
──────────────────────────────────────────────────────────────
timestampTrig        uint64   64-bit trigger timestamp
timestampTDC         uint64   64-bit TDC timestamp
coarseTime           uint32   Coarse time value
tdcOffset            int      TDC offset in ns
trigType             uint32   Trigger type code
wheel                uint32   TAC wheel selector
multiplicity         uint32   Hit multiplicity
userRegister         uint32   User register value
triggerBitMask       uint32   Trigger bit pattern
fourNanoSecCounter   uint32[] 4 ns counter values
vernierAB            uint32   Vernier A/B
vernierCD            uint32   Vernier C/D
vernier[4]           int[]    Vernier interpolation values (ps)
phaseTime[4]         double[] Phase times per vernier
avgPhaseTime         double   Average phase time (final TDC result, ps)
```

**TACTrash validity codes:**
| Value | Meaning |
|-------|---------|
| 0 | Valid |
| 1 | NoTrigger |
| 2 | TDCoffsetInvalid |
| 3 | VernierInvalid |

A `TRASH_DATA` marker (value 666) in the GEB stream is skipped by the binary reader.

---

## 4. UniqueID Convention

The binary reader assigns a `UniqueID` to each hit:
```
UniqueID = DigID * 100 + channel
```
Where `DigID` is a sequential index per digitizer board in the system (0-based), and `channel` = 0–9.

---

## 5. Full Event Flow

```
FPGA (DIG board)
  │ Discriminates ADC samples → builds event packet (HEADER_TYPE 1–8)
  │ Stores in FIFO
  ▼
VME IOC (VxWorks, inLoop)
  │ DMA reads DIG FIFO → prepends GEBHeader (type=14, timestamp from MTRG)
  │ Writes to shared memory buffer (qFree → qWritten)
  ▼
MiniSender (VxWorks, outLoop)
  │ Dequeues from qWritten
  │ Sends via TCP port 9001 to tcpReceiverMT
  ▼
tcpReceiverMT (C++, on dgs1/dgs2)
  │ Receives GEB stream → writes raw binary files
  ▼
fastEventConstructor / parquet_pysort (post-run)
  │ Reads GEBHeader + DIG/TDC payload
  │ Sorts by timestamp → event-builds → ROOT / Parquet output
```

---

## References

- `dgs_analysis/armory/fastEventContructor/class_DIG.h` — DIG payload decoder
- `dgs_analysis/armory/fastEventContructor/class_Hit.h` — GEBHeader + HIT class
- `dgs_analysis/armory/fastEventContructor/class_TDC.h` — TAC-II TDC decoder
- `dgs/DIG_firmware_expert.md` — DIG firmware readout modes and packet structure
- `dgs/dgs_analysis.md` — fastEventConstructor and parquet_pysort documentation
- `dgs/ANLDAQ.md` — tcpReceiverMT and IOC sender documentation

---
*Created: 2026-04-06. Source: class_DIG.h + class_Hit.h + class_TDC.h*
