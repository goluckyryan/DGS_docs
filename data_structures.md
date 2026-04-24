# DGS Data Structures — Binary Output Format

Stability: C3 - Structural / stable

_Source: `dgs_analysis/armory/fastEventContructor/class_Hit.h`, `class_DIG.h`, `class_TDC.h`_

The IOC sender (`tcpReceiverMT`) writes raw binary files in **GEB (GRETINA Event Builder) format**. Each event = one `GEBHeader` + payload.

---

## Table of Contents

- [1. GEBHeader (16 bytes)](#1-gebheader-16-bytes)
- [Note: IOC Output vs Saved File Format](#note-ioc-output-vs-saved-file-format)
- [2. DIG Event Payload (GEB type 14)](#2-dig-event-payload-geb-type-14)
  - [DIG Event Layout (ASCII)](#dig-event-layout-ascii)
  - [Header Words (always present)](#header-words-always-present)
  - [HEADER_TYPE Values](#header_type-values)
  - [Word 4 — Flag Word](#word-4--flag-word)
  - [Words 5–13 — Energy, Timing, Multiplex](#words-513--energy-timing-multiplex)
  - [Words 14+ — Waveform Trace (optional)](#words-14--waveform-trace-optional-types-246)
- [3. TAC-II / TDC Payload (GEB type 15)](#3-tac-ii--tdc-payload-geb-type-15)
  - [TAC2 Repacking (IOC → Receiver)](#tac2-repacking-ioc--receiver)
- [4. UniqueID Convention](#4-uniqueid-convention)
- [5. Full Event Flow](#5-full-event-flow)
- [References & Cross-References](#references--cross-references)

---

## 1. GEBHeader (16 bytes)

Every event starts with a fixed 16-byte header: ✅ verified 2026-04-07 — `receiver.h:L51-55` (`struct gebData`: int32_t type + int32_t length + uint64_t timestamp = 4+4+8 = 16 bytes)

```
Offset  Size  Field                  Description
──────────────────────────────────────────────────────
 0      4     type                   GEB type code (see table below)
 4      4     payload_length_byte    Payload size in bytes
 8      8     timestamp              64-bit timestamp (10 ns ticks from MTRG) ✅ verified 2026-04-07 — BinaryReader.h:L58 ("in 10 ns")
```

**GEB Type codes used by DGS:**

| Code | Constant | Description |
|------|----------|-------------|
| 14 | `GEB_TYPE_DGS` | DGS digitizer hit data ✅ verified 2026-04-07 — `dgsReceiver_Ryan.cpp:L141` |
| 15 | `GEB_TYPE_DGSTRIG` | DGS trigger data (TAC-II TDC) ✅ verified 2026-04-07 — `dgsReceiver_Ryan.cpp:L142` |
| 99 | *(internal)* | Decoded TAC2 trigger (used internally by fastEventConstructor) |

**Complete GEB type table** (from `dgsReceiver_Ryan.cpp:L129-150` + `geb_format.py`) ✅ verified 2026-04-14:

| Code | Constant | Description |
|------|----------|-------------|
| 1 | `GEB_TYPE_DECOMP` | Decomposed GRETINA |
| 2 | `GEB_TYPE_RAW` | Raw GRETINA |
| 3 | `GEB_TYPE_TRACK` | GRETINA tracking |
| 4 | `GEB_TYPE_BGS` | BGS recoil separator |
| 5 | `GEB_TYPE_S800_RAW` | S800 spectrometer raw |
| 6 | `GEB_TYPE_NSCLnonevent` | NSCL non-event |
| 7 | `GEB_TYPE_GT_SCALER` | GRETINA scaler |
| 8 | `GEB_TYPE_GT_MOD29` | GRETINA Mod29 _(see note)_ |
| 9 | `GEB_TYPE_S800PHYSDATA` | S800 physics |
| 10 | `GEB_TYPE_NSCLNONEVTS` | NSCL non-events |
| 11 | `GEB_TYPE_G4SIM` | Geant4 sim |
| 12 | `GEB_TYPE_CHICO` | CHICO detector |
| 14 | `GEB_TYPE_DGS` | **DGS digitizer** |
| 15 | `GEB_TYPE_DGSTRIG` | **DGS TAC-II trigger** |
| 16 | `GEB_TYPE_DFMA` | DFMA digitizer |
| 17 | `GEB_TYPE_PHOSWICH` | Phoswich |
| 18 | `GEB_TYPE_PHOSWICHAUX` | Phoswich aux |
| 19 | `GEB_TYPE_GODDESS` | GODDESS |
| 20 | `GEB_TYPE_LABR` | LaBr₃ |
| 21 | `GEB_TYPE_LENDA` | LENDA neutron |
| 22 | `GEB_TYPE_GODDESSAUX` | GODDESS aux |
| 23 | `GEB_TYPE_XA` | X-Array (DXA) |
| 24 | `GEB_TYPE_DUB` | DuoGe (DUO) |
| 25 | `GEB_TYPE_FT` | FT detector |

> ⚠️ **Type 8 ambiguity:** Some old DGS runs may have data under type 8 (GT_MOD29) rather than type 14. `dgsReceiver_Ryan.cpp:L1448` allows overriding `GEB_TYPE_DGS` from the command line. `geb_format.py` notes: _"Some experiments use 8 for DGS, it should be 14 in principle."_

---

## Note: IOC Output vs Saved File Format

The **IOC TCP sender** and the **tcpReceiverMT saved file** differ:

| | IOC TCP stream | Saved binary file |
|--|----------------|-------------------|
| **DIG data** | Raw DIG payload only (no GEB header) | GEBHeader (type=14) + DIG payload |
| **TAC2/trigger data** | Raw 16-word VME trigger packet | Receiver repacks → 10-word DIG-compatible packet, then GEBHeader (type=15) prepended |

The receiver (`tcpReceiverMT`) adds GEB headers to DIG data and **repacks** TAC2 data from the 16-word VME format into a compact 10-word format before saving.

---

## 2. DIG Event Payload (GEB type 14)

The DIG payload is produced by the digitizer FPGA and forwarded by the IOC sender. It is decoded into the `DIG` class. Payload is big-endian 32-bit words (`ntohl()` applied on read).

### DIG Event Layout (ASCII)

```
Bytes   Content
┌───────────────────────────────────────────────────────────┐
│ GEBHeader (16 bytes, added by tcpReceiverMT)              │
│  [0- 3] type         = 14 (GEB_TYPE_DGS)                  │
│  [4- 7] payload_len  = packet_length × 4 (bytes)          │
│  [8-15] timestamp    = 64-bit (10 ns ticks)               │
├───────────────────────────────────────────────────────────┤
│ DIG Payload (from FPGA, big-endian 32-bit words)          │
│  Word 0:  0xAAAAAAAA  (sync word)                         │
│  Word 1:  [31:27]=GEO_ADDR [26:16]=PKT_LEN                │
│           [15:4]=USER_DEF  [ 3: 0]=CH_ID                  │
│  Word 2:  EVENT_TIMESTAMP[31:0]                           │
│  Word 3:  [31:26]=HDR_LEN  [25:23]=EVT_TYPE               │
│           [22]=CFD_ESUM    [21]=TRIG_TS [20]=PEQ_BYPASS   │
│           [19:16]=HDR_TYPE [15:0]=TIMESTAMP[47:32]        │
│  Word 4:  FLAGS (VETO, PILEUP, CFD_VALID, PEAK_VALID...)  │
│  Word 5:  CFD_SAMPLE_0 (CFD mode) / padding (LED mode)    │
│  Word 6:  [27:24]=PILEUP_CNT  [23:0]=SAMPLED_BASELINE     │
│  Word 7:  LED: TRIG_MON_XTRA / CFD: CFD_SAMPLE_1+2        │
│  Word 8:  [31:24]=POST_RISE_E[7:0]  [23:0]=PRE_RISE_E     │
│  Word 9:  [31:16]=PEAK_TIMESTAMP    [15:0]=POST_RISE_E    │
│  Word 10: [31:16]=TS_OF_TRIGGER     [14]=P2_MODE          │
│           [13:0]=P2_SUM[13:0]                             │
│  Word 11: [31:24]=M_SUM[7:0]  [23:0]=MULTIPLEX_DATA       │
│  Word 12: [31:24]=M_SUM[15:8] [23:0]=EARLY_PRE_RISE_E     │
│  Word 13: [31:24]=M_SUM[23:16][23:14]=TS_OF_COARSE        │
│           [13]=COARSE_FIRED  [10]=2ND_THRESH  [9:0]=P2    │
│  Word 14+: Trace (optional, 2×16-bit ADC samples/word)    │
└───────────────────────────────────────────────────────────┘
```

### Header Words (always present)

```
Word  Bits    Field                Notes
────────────────────────────────────────────────────────────────────────
 0    31:0    0xAAAAAAAA           Fixed sync word (always 0xAAAAAAAA) ✅ verified 2026-04-07 — receiver.h:L359 (`if(data[index] == 0xAAAAAAAA)`)
 1    3:0     CH_ID                Channel index 0–9 ✅ verified 2026-04-07 — class_DIG.h:L24
 1    15:4    USER_DEF             User-defined field ✅ verified 2026-04-07 — class_DIG.h:L25
 1    26:16   PACKET_LENGTH        Total packet length in 32-bit words ✅ verified 2026-04-07 — class_DIG.h:L26
 1    31:27   GEO_ADDR             VME geographic address (slot number) ✅ verified 2026-04-07 — class_DIG.h:L27
 2    31:0    EVENT_TIMESTAMP[31:0] Lower 32 bits of 48-bit event timestamp ✅ verified 2026-04-20 — `receiver.h:L402` (`header[1] & 0xFFFFFFFF`)
 3    15:0    EVENT_TIMESTAMP[47:32] Upper 16 bits of 48-bit event timestamp ✅ verified 2026-04-20 — `receiver.h:L402` (`(header[2] & 0x0000FFFF) << 32`)
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
| 1 | Old LED | Old LED header (legacy) |
| 2 | Old CFD | Old CFD header (legacy) |
| 3 | New LED | New LED header |
| 4 | New CFD | New CFD header |
| 5 | LED+Pileup | LED header with pileup info |
| 6 | CFD+Pileup | CFD header with pileup info |
| 7 | DIG LED | Current DIG LED header (most common) |
| 8 | DIG CFD | Current DIG CFD header |

> **Correction note (2026-04-22):** Prior table incorrectly described types 1–6 as "LED minimal/trace/full/E" variants. `class_DIG.h:L213-228` confirms actual labels: 1=Old LED, 2=Old CFD, 3=New LED, 4=New CFD, 5=LED+Pileup, 6=CFD+Pileup, 7=DIG LED, 8=DIG CFD. ✅ verified 2026-04-22 — class_DIG.h:L213-228

### Word 4 — Flag Word

```
Bit   Field                    Notes
────────────────────────────────────────────────────────────────
 4    PU_TIME_ERROR_FLAG       Pileup timing error flag ✅ verified 2026-04-18 — Event_Header_FIFO.vhd:L306,L395 (DGS_TAG_20180607_TWEAK)
 5    WRITE_FLAGS              ADC trace format: 0 = 14-bit+flag bits, 1 = 16-bit no flags (stored as NOT write_flags) ✅ verified 2026-04-18 — Event_Header_FIFO.vhd:L305,L394
 6    VETO_FLAG                Event was vetoed by BGO shield (replaced at readout)
 7    TIMESTAMP_MATCH_FLAG     CFD only: timestamp match (tsm_flag); LED = 0 ✅ verified 2026-04-18 — Event_Header_FIFO.vhd:L392 (CFD) L303 (LED=0)
 8    EXTERNAL_DISC_FLAG       External discriminator fired ✅ verified 2026-04-18 — Event_Header_FIFO.vhd:L302,L391
 9    PEAK_VALID_FLAG          Peak detection valid ✅ verified 2026-04-18 — Event_Header_FIFO.vhd:L301,L390
10    OFFSET_FLAG              Offset correction applied (replaced at readout)
11    CFD_VALID_FLAG           CFD only: CFD interpolation valid ✅ verified 2026-04-18 — Event_Header_FIFO.vhd:L388 (CFD); LED = 0 per L299
12    SYNC_ERROR_FLAG          Sync error flag ✅ verified 2026-04-18 — Event_Header_FIFO.vhd:L298 (LED) L387 (CFD)
13    GENERAL_ERROR_FLAG       Replaced by GENERAL_ERROR_FLAG at readout (stored as 0 in header FIFO) ✅ verified 2026-04-18 — Event_Header_FIFO.vhd:L297 (comment "Replaced by GENERAL_ERROR_FLAG at time of readout")
14    PILEUP_WAVEFORM_ONLY     Pileup: waveform-only mode (pileup_waveform_only) ✅ verified 2026-04-18 — Event_Header_FIFO.vhd:L296,L385
15    PILEUP_FLAG              Pileup detected ✅ verified 2026-04-18 — Event_Header_FIFO.vhd:L295,L384
```

> **Correction note (2026-04-18):** Bits 4 and 5 were previously mislabeled as `EARLY_PRE_RISE_SELECT` and `WRITE_FLAGS (firmware config)`. VHDL source confirms bit 4 = `pu_time_error_flag`, bit 5 = `NOT write_flags` (ADC trace mode). Bits 12 and 13 were previously missing.

### Words 5–13 — Energy, Timing, Multiplex

```
Word  Bits    Field                       Notes
────────────────────────────────────────────────────────────────────────────
  5   29:16   CFD_SAMPLE_0                CFD only: interpolation sample 0 ✅ verified 2026-04-18 — class_DIG.h:L80,L690
  6   23:0    SAMPLED_BASELINE            Baseline at trigger time ✅ verified 2026-04-09 — `jta_channel.vhd:L1796` (PEHQ bits 347:324 = SAMPLED_BASELINE = RUNNING_T1_SUM latched at trigger); also class_DIG.h:L699
  6   27:24   PILEUP_COUNT                Pileup count in LED mode; bits[3:2] from Word 7(31:30) and [1:0] from Word 7(15:14) in CFD mode ✅ verified 2026-04-18 — class_DIG.h:L658 (LED), L694 (CFD)
  7   15:0    TRIG_MON_XTRA_DATA          LED only: trigger monitor extra ✅ verified 2026-04-18 — class_DIG.h:L661 (`Raw_Header[7] & 0x0000FFFF`)
  7   13:0    CFD_SAMPLE_1                CFD only: interpolation sample 1 ✅ verified 2026-04-18 — class_DIG.h:L81,L686
  7   29:16   CFD_SAMPLE_2                CFD only: interpolation sample 2 ✅ verified 2026-04-18 — class_DIG.h:L82,L688
  8   23:0    PRE_RISE_ENERGY             Energy before rise (trapezoidal filter) ✅ verified 2026-04-18 — class_DIG.h:L702 (`Raw_Header[8] & 0x00FFFFFF`)
  8   31:24   POST_RISE_ENERGY[7:0]       Lower 8 bits of post-rise energy ✅ verified 2026-04-18 — class_DIG.h:L700 (`(Raw_Header[8] & 0xFF000000) >> 24`)
  9   15:0    POST_RISE_ENERGY[23:8]      Upper 16 bits of post-rise energy ✅ verified 2026-04-18 — class_DIG.h:L701 (`(Raw_Header[9] & 0x0000FFFF) << 8`)
  9   31:16   PEAK_TIMESTAMP              16-bit peak timestamp (ns within event) ✅ verified 2026-04-18 — class_DIG.h:L705 (`(Raw_Header[9] & 0xFFFF0000) >> 16`)
 10   13:0    P2_SUM[13:0]                P2 sum lower bits ✅ verified 2026-04-18 — class_DIG.h:L717 (`Raw_Header[10] & 0x00003FFF`)
 10   14      P2_MODE                     P2 mode flag ✅ verified 2026-04-18 — class_DIG.h:L714 (`ExtractBit(Raw_Header[10], 14)`)
 10   15      CAPTURE_PARST_TS            Capture PARST timestamp ✅ verified 2026-04-18 — class_DIG.h:L715 (`ExtractBit(Raw_Header[10], 15)`)
 10   31:16   TS_OF_TRIGGER               16-bit timestamp of trigger relative to event ✅ verified 2026-04-18 — class_DIG.h:L707 (`(Raw_Header[10] & 0xFFFF0000) >> 16`)
 11   23:0    MULTIPLEX_DATA              Multiplex data field ✅ verified 2026-04-18 — class_DIG.h:L724 (`Raw_Header[11] & 0x00FFFFFF`)
 11   31:24   LAST_POST_RISE_M_SUM[23:16] M sum bits 23:16 ✅ verified 2026-04-18 — class_DIG.h:L719 (`(Raw_Header[11] & 0xFF000000) >> 8` → bits 23:16)
 12   23:0    EARLY_PRE_RISE_ENERGY       Early pre-rise energy sum ✅ verified 2026-04-18 — class_DIG.h:L727 (`Raw_Header[12] & 0x00FFFFFF`)
 12   31:24   LAST_POST_RISE_M_SUM[15:8]  M sum bits 15:8 ✅ verified 2026-04-18 — class_DIG.h:L720 (`(Raw_Header[12] & 0xFF000000) >> 16` → bits 15:8)
 13    9:0    P2_SUM[23:14]               P2 sum upper bits ✅ verified 2026-04-18 — class_DIG.h:L716 (`(Raw_Header[13] & 0x000003FF) << 14`)
 13   10      SECOND_THRESH_DISC_FLAG     Second threshold discriminator flag ✅ verified 2026-04-18 — class_DIG.h:L729 (`ExtractBit(Raw_Header[13], 10)`)
 13   11      PARST_TSM                   PARST timestamp match ✅ verified 2026-04-18 — class_DIG.h:L730 (`ExtractBit(Raw_Header[13], 11)`)
 13   12      PREVIOUS_CFD_VALID          Previous CFD valid (CFD mode only) ✅ verified 2026-04-18 — class_DIG.h:L734 (`ExtractBit(Raw_Header[13], 12)`)
 13   13      COARSE_FIRED                Coarse discriminator fired ✅ verified 2026-04-18 — class_DIG.h:L732 (`ExtractBit(Raw_Header[13], 13)`)
 13   23:14   TS_OF_COARSE                10-bit coarse timestamp (low 8 bits); combines with Word 4 bits 13:12 (high 2 bits) ✅ verified 2026-04-18 — class_DIG.h:L733 (`((Raw_Header[4] & 0x00003000) >> 2) + ((Raw_Header[13] & 0x00FFC000) >> 14)`)
 13   31:24   LAST_POST_RISE_M_SUM[7:0]   M sum bits 7:0 ✅ verified 2026-04-18 — class_DIG.h:L721 (`(Raw_Header[13] & 0xFF000000) >> 24` → bits 7:0)
```

> **Note (LAST_POST_RISE_M_SUM bit ordering):** Bits 23:16 come from Word 11, bits 15:8 from Word 12, bits 7:0 from Word 13. The doc table was previously ordered incorrectly; corrected 2026-04-18. ✅ verified — class_DIG.h:L719-L721
> **Note (TS_OF_COARSE):** The full 10-bit value uses Word 13 bits 23:14 (8 bits, low) **plus** Word 4 bits 13:12 (2 bits, high). Not purely in Word 13. ✅ verified — class_DIG.h:L733
> **Note (EARLY_PRE_RISE_SELECT vs PU_TIME_ERROR_FLAG):** `class_DIG.h:L38,L632` labels bit 4 of Word 4 as `EARLY_PRE_RISE_SELECT`, but `Event_Header_FIFO.vhd:L306` (FPGA source, verified 2026-04-18) names it `pu_time_error_flag`. The FPGA is authoritative; `class_DIG.h` uses an older name. Both refer to the same bit.

### Words 14+ — Waveform Trace (optional, types 2/4/6)

Raw 16-bit ADC samples packed into 32-bit words (2 samples per word):
- ADC data = 14-bit unsigned offset binary (0 = most negative, 0x3FFF = most positive) ✅ verified 2026-04-21 — `class_DIG.h:L244` (`(word >> 16) & 0x3FFF`) + `DIG_firmware_expert.md:L360`
- Bit 14 = timing mark ✅ verified 2026-04-21 — `DIG_firmware_expert.md:L360` ("Bit 14 = timing mark") + `DIG_firmware_expert.md:L213` ("Bit 14 of each 16-bit half encodes timing marks")
- Bit 15 = downsampling marker ✅ verified 2026-04-21 — `DIG_firmware_expert.md:L360` ("bit 15 = down-sampling marker")
- Number of trace words = `(PACKET_LENGTH - HEADER_LENGTH)` words ✅ verified 2026-04-21 — `class_DIG.h:L239` (`num_trace_words = PACKET_LENGTH - trace_start_index` where `trace_start_index = 14 = HEADER_LENGTH`)

---

## 3. TAC-II / TDC Payload (GEB type 15)

### TAC2 Repacking (IOC → Receiver)

The IOC sends the raw VME trigger packet (16 words). `tcpReceiverMT` **repacks** it into a 10-word DIG-compatible format before saving.

```
IOC TCP stream: Raw TAC2 VME packet (16 words, 32-bit each, big-endian)
┌───────────────────────────────────────────────────┐
│ Word  0:  [15:0] 0xAAAA                           │
│ Word  1:  [15:0] trigType data                    │
│ Word  2:  [15:0] timestamp high                   │
│ Word  3:  [15:0] timestamp middle                 │
│ Word  4:  [15:0] timestamp low                    │
│ Word  5:  [15:0] wheel                            │
│ Word  6:  [15:0] multiplicity                     │
│ Word  7:  [15:0] userRegister                     │
│ Word  8:  [15:0] coarseTime                       │
│ Word  9:  [15:0] triggerBitMask                   │
│ Word 10:  [15:0] 4nsCounter[0]                    │
│ Word 11:  [15:0] 4nsCounter[1]                    │
│ Word 12:  [15:0] 4nsCounter[2]                    │
│ Word 13:  [15:0] 4nsCounter[3]                    │
│ Word 14:  [15:12] vernierByte [11 :0] vernierAB   │
│ Word 15:  [11:0] vernierCD                        │
└───────────────────────────────────────────────────┘
             │ tcpReceiverMT repacks 16→10 words:
             │ pairs of adjacent 32-bit words are
             │ merged into one word as [hi<<16 | lo]
             ▼
Saved file: GEBHeader + repacked TAC2 packet (10 words)
┌─────────────────────────────────────────────────────────────────┐
│ GEBHeader (16 bytes):                                           │
│   type = 15 (GEB_TYPE_DGSTRIG)                                  │
│   payload_len = 10 × 4 = 40 bytes                               │
│   timestamp = extracted from header[2] (VME word 2)             │
├─────────────────────────────────────────────────────────────────┤
│ Repacked payload (10 words, DIG-compatible):                    │
│  Word 0: 0xAAAAAAAA  (sync word)                                │
│  Word 1: [31:16] data len  [15:4] Board_id [3:0] Ch_id          |
│  Word 2: [31:16] ts middle     [15:0] ts_low                    │
│  Word 3: [31:26] 3 [25:16] header_type [15:0] ts_high           │
│  Word 4: [31:26]  trigType     [15:0] wheel                     │
│  Word 5: [31:16] multiplicity  [15:0] userRegister              │
│  Word 6: [31:16] coarseTime    [15:0] triggerBitMask            │
│  Word 7: [31:16] 4nsCounter[0] [15:0] 4nsCounter[1]             │
│  Word 8: [31:16] 4nsCounter[2] [15:0] 4nsCounter[3]             │
│  Word 9: [31:28] vernierByte [27:16] vernierAB [11:0] vernierCD │
└─────────────────────────────────────────────────────────────────┘
  Source: `tcpReceiverMT.cpp` / `receiver.h:L447–524`
  Note: board_id=99, CH_ID=0xA, header_Type=0xE are sentinel values identifying this as trigger data. ✅ verified 2026-04-22 — receiver.h:L462-464
```

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

**NoTrigger detection:** If `packedData[9] == 0x10021001`, event is flagged `NoTrigger` and discarded. ✅ verified 2026-04-12 — `class_TDC.h:L113-114`

**TDC offset validity:** `tdcOffset = timestampTDC - timestampTrig`; must satisfy `0 ≤ tdcOffset ≤ 200 ns`, else `TDCoffsetInvalid`. ✅ verified 2026-04-12 — `class_TDC.h:L137-139`

**Vernier timing formula** (`class_TDC.h:CalTime()`):
```
baseTime = timestampTrig - (timestampTrig % 262144)
         = timestampTrig aligned to 2^18 × 10 ns = 2,621,440 ns period

phaseTime[i] = baseTime + fourNanoSecCounter[i] × 4 ns
                         + offset[i]            (default: 0,1,2,3 ns) ✅ verified 2026-04-12 — `class_TDC.h:L46`
                         - vernier[i] × 0.05 ns

avgPhaseTime = mean of valid phaseTime[i] values  (in ns)
```
- **Vernier resolution:** 50 ps per count (6-bit, range 0–63 → 0–3.15 ns) ✅ verified 2026-04-12 — `class_TDC.h:L184` (`vernier[i] * 0.05`)
- **validBit:** `vernierAB[15:12]`; bit `i` set → vernier channel `i` is valid ✅ verified 2026-04-12 — `class_TDC.h:L161` (`validBit = vernierAB >> 12`)
- **Final result:** `avgPhaseTime` (ns) → converted to 10 ns timestamp units for `EVENT_TIMESTAMP`
- **Sub-sample remainder:** stored as `POST_RISE_ENERGY = (avgPhaseTime mod 10 ns) × 1000` (in ps)

_Source: `class_TDC.h:CalTime(),DecodePackedData()` — verified 2026-04-07_

---

## 4. UniqueID Convention

The binary reader assigns a `UniqueID` to each hit:
```
UniqueID = DigID * 100 + channel
```
Where `DigID` is parsed from the **filename** (4-digit field), and `channel` = 0–9. File naming convention: `dgs_runXXX.gtd01_<fileIndex>_<DigID>_<Channel>`. `DigID` is NOT a 0-based sequential index — it comes directly from the filename. ✅ verified 2026-04-07 — `BinaryReader.h:L33` (`GetUniqueID()`), `L105` (DigID = last digits of filename), `L372` (filename format comment)

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

## References & Cross-References

**Primary sources:**
- `dgs_analysis/armory/fastEventContructor/class_DIG.h` — DIG payload decoder
- `dgs_analysis/armory/fastEventContructor/class_Hit.h` — GEBHeader + HIT class
- `dgs_analysis/armory/fastEventContructor/class_TDC.h` — TAC-II TDC decoder

**Related knowledge base files:**
- `knowledgeBase/ANLDAQ.md` — tcpReceiverMT and IOC sender; how GEB data is produced and transmitted
- `knowledgeBase/guceiver.md` — Guceiver live GUI; decodes DIG and TAC-II packets from TCP stream
- `knowledgeBase/dgs_analysis.md` — fastEventConstructor and parquet_pysort; consume GEB binary data
- `knowledgeBase/DIG_firmware_expert.md` — Full DIG event header field definitions + readout modes (expert reference)
- `knowledgeBase/ttcl.md` — TTCL spec; timestamp generation that populates GEB timestamp field
- `knowledgeBase/gebsort.md` — GEBSort event builder; reads GEB format, builds coincidence events

---
*Created: 2026-04-06. Source: class_DIG.h + class_Hit.h + class_TDC.h. Last updated: 2026-04-17 (merged duplicate References/Cross-References sections).*
