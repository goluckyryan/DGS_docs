# ANLDAQ — tcpReceiver/Aux Classes and Analysis Scripts

Stability: C2 - Active / semi-stable

**Source:** `DGS_tools_pack/ANLDAQ/tcpReceiver/Aux/`

These are C++ header-only classes and ROOT macro scripts used for offline decoding and analysis of DGS binary data files produced by `tcpReceiver`. They are not part of the online DAQ pipeline — they run interactively in ROOT (CINT/PyROOT) for calibration and timing studies.

## Table of Contents
1. [class_DIG.h — DIG_Hit Decoder](#class_digh)
2. [class_TDC.h — TDC_Hit / TAC-II Decoder](#class_tdch)
3. [script.cpp — TAC–DIG Coincidence Analysis](#scriptcpp)
4. [script_LED.cpp — LED/CFD Mode Comparison](#script_ledcpp)
5. [Supporting Files (reader.h, shells)](#supporting-files)
6. [See Also](#see-also)

---

## 1. class_DIG.h — DIG_Hit Decoder {#class_digh}

**File:** `Aux/class_DIG.h` (370 lines)

Provides the `DIG_Hit` C++ class: decodes the 14-word ANL digitizer event packet header (type 7 = LED, type 8 = CFD) from raw 32-bit words sent by the IOC without byte-swapping. ✅ verified 2026-04-27 — `class_DIG.h:L21` (class comment: "DIG data header type 7 or 8"), `L19` (comment: "receiver does not change byte order").

### Helper

```cpp
bool ExtractBit(unsigned int value, unsigned int bitPosition)
```
Extracts a single bit from a 32-bit word. Returns false for positions > 31. ✅ verified 2026-04-27 — `class_DIG.h:L12-16`.

### DIG_Hit Fields

**Common (Words 0–13):**

| Field | Word | Bits | Notes |
|-------|------|------|-------|
| `CH_ID` | 1 | 3:0 | Channel ID |
| `USER_DEF` | 1 | 15:4 | User-defined |
| `PACKET_LENGTH` | 1 | 26:16 | Total packet length |
| `GEO_ADDR` | 1 | 31:27 | Geographic address |
| `EVENT_TIMESTAMP` | 2+3 | 48-bit | Words 2 (31:0) + Word 3 (15:0) → 48-bit |
| `HEADER_TYPE` | 3 | 19:16 | 7=LED, 8=CFD |
| `PEQ_BYPASS` | 3 | 20 | Pole-zero bypass flag |
| `TRIG_TS_MODE` | 3 | 21 | Trigger timestamp mode |
| `CFD_ESUM_MODE` | 3 | 22 | Added 2021-08-17; always 0 in LED mode |
| `EVENT_TYPE` | 3 | 25:23 | Trigger type from trigger accept message bits 10:8 |
| `HEADER_LENGTH` | 3 | 31:26 | Header length |
| `EARLY_PRE_RISE_SELECT` | 4 | 4 | |
| `WRITE_FLAGS` | 4 | 5 | |
| `VETO_FLAG` | 4 | 6 | |
| `EXTERNAL_DISC_FLAG` | 4 | 8 | |
| `PEAK_VALID_FLAG` | 4 | 9 | |
| `OFFSET_FLAG` | 4 | 10 | |
| `PILEUP_ONLY_FLAG` | 4 | 14 | |
| `PILEUP_FLAG` | 4 | 15 | |
| `SAMPLED_BASELINE` | 6 | 23:0 | 24-bit baseline |
| `PRE_RISE_ENERGY` | 8 | 23:0 | |
| `POST_RISE_ENERGY` | 8+9 | Word 8 (31:24) + Word 9 (15:0) | Cross-word |
| `PEAK_TIMESTAMP` | 9 | 31:16 | 16-bit |
| `P2_SUM` | 10+13 | Word 10 (13:0) + Word 13 (9:0) << 14 | |
| `P2_MODE` | 10 | 14 | |
| `CAPTURE_PARST_TS` | 10 | 15 | |
| `TS_OF_TRIGGER` | 10 | 31:16 | 16-bit |
| `MULTIPLEX_DATA` | 11 | 23:0 | Can be energy or time |
| `LAST_POST_RISE_M_SUM` | 11+12+13 | 31:24 of each | Spans 3 words |
| `EARLY_PRE_RISE_ENERGY` | 12 | 23:0 | |
| `SECOND_THRESH_DISC_FLAG` | 13 | 10 | |
| `PARST_TSM` | 13 | 11 | |
| `COARSE_FIRED` | 13 | 13 | |
| `TS_OF_COARSE` | 4+13 | Word 4 (13:12) + Word 13 (23:14) | 10-bit across words |

**LED only (HEADER_TYPE == 7):**

| Field | Word | Bits |
|-------|------|------|
| `TRIG_MON_XTRA_DATA` | 7 | 15:0 |
| `TRIG_MON_DET_DATA` | 7 | 31:16 |
| `LAST_DISC_TIMESTAMP` | 4+5 | Word 4 (31:16) + Word 5 (31:0) → 48-bit |
| `PILEUP_COUNT` | 6 | 27:24 |

**CFD only (HEADER_TYPE == 8):**

| Field | Word | Bits | Notes |
|-------|------|------|-------|
| `TIMESTAMP_MATCH_FLAG` | 4 | 7 | |
| `CFD_VALID_FLAG` | 4 | 11 | |
| `CFD_SAMPLE_0` | 5 | 29:16 | Signed 14-bit (sign-extended from bit 13) |
| `CFD_SAMPLE_1` | 7 | 13:0 | Signed 14-bit |
| `CFD_SAMPLE_2` | 7 | 29:16 | Signed 14-bit |
| `PREVIOUS_CFD_VALID` | 13 | 12 | |
| `LAST_DISC_TIMESTAMP` | 4+5 | Word 4 (31:16) + Word 5 (13:0) → 30-bit |
| `TRIG_MON_DET_DATA` | 4+5+6 | Scattered bits (complex extraction) | |
| `PILEUP_COUNT` | 7 | (31:30)→(3:2) + (15:14)→(1:0) | 4-bit, split |

**Trace/waveform (Words 14+):**
- `std::vector<unsigned short> trace` — two 14-bit unsigned samples per 32-bit word
  - bits 13:0 = first sample, bits 29:16 = second sample ✅ verified 2026-04-27 — `class_DIG.h:L359-363` (decode loop; `& 0x3FFF` for bits 13:0, `>> 16 & 0x3FFF` for bits 29:16)

### Key Methods

- `Clear()` — zeroes all fields, clears trace vector
- `DecodeHeader_7_8(uint32_t Raw_Header[14], int dataLen)` — main decode; branches on HEADER_TYPE; auto-populates trace from words 14+
- `Print()` — pretty-print all fields; shows CFD-only fields only if HEADER_TYPE==8
- `PrintTrace()` — prints trace samples as decimal and hex

### Notes
- The receiver does **not** byte-swap; `DIG_Hit` handles byte order itself ✅ verified 2026-04-27 — `class_DIG.h:L19`
- `EVENT_TIMESTAMP` is 48-bit: combined from Words 2 and 3 (bits 15:0) ✅ verified 2026-04-27 — `class_DIG.h:L29` (field declaration comment)
- All Word 1 fields (CH_ID, USER_DEF, PACKET_LENGTH, GEO_ADDR), Word 3 header fields, Word 4 flag bits, and `TS_OF_COARSE` cross-word extraction all verified ✅ 2026-04-27 — `class_DIG.h:L24-69,L255-357`
- `PEAK_TIMESTAMP` is 16-bit; to get full peak time: `(EVENT_TIMESTAMP & 0xFFFFFFFF0000) + PEAK_TIMESTAMP`, handle rollover by adding 0x10000 if result < EVENT_TIMESTAMP
- CFD_SAMPLE_* are sign-extended from 14-bit two's complement

---

## 2. class_TDC.h — TDC_Hit / TAC-II Decoder {#class_tdch}

**File:** `Aux/class_TDC.h` (268 lines)

Provides two classes: `TDC200MHz` (sub-hit vernier data) and `TDC_Hit` (full TAC-II packet decoder).

### TDC200MHz Struct

Models one TAC-II 200 MHz TDC measurement. Four independent phase channels (0–3), each with a 4 ns counter and a 6-bit vernier (50 ps resolution).

| Field | Type | Description |
|-------|------|-------------|
| `baseTime` | double | Base time (2^18 grid = 262144 ns) |
| `fourNanoSecCounter[4]` | uint16_t | 4 ns tick counters for channels 0–3 |
| `vernierAB` | uint16_t | Raw vernier word for channels A (0) and B (1) |
| `vernierCD` | uint16_t | Raw vernier word for channels C (2) and D (3) |
| `validBit` | uint8_t | 4-bit validity mask (bit i = channel i valid) |
| `valid[4]` | bool | Per-channel validity |
| `vernier[4]` | int | 6-bit vernier values; -1 if invalid |
| `phaseTime[4]` | double | Computed phase timestamps in ns |
| `vernierOrder` | int | Sort order of verniers (packed 4-digit decimal: units digit = rank-0 channel index) |
| `avgPhaseTimestamp` | double | Average of valid phase times |
| `standardDeviation` | double | Std dev of phase times |
| `maxDiff` | double | Max deviation from average among valid channels |

**CalVernier():** Extracts per-channel vernier values from `vernierAB`/`vernierCD`:
- `vernier[0]` = `vernierAB[11:6]`, `vernier[1]` = `vernierAB[5:0]`
- `vernier[2]` = `vernierCD[11:6]`, `vernier[3]` = `vernierCD[5:0]`
- `validBit` = `vernierAB[15:12]`

**FindVernierOrder():** Sorts channels by vernier value (ascending) and encodes rank order as a decimal integer.

### TDC_Hit Class

Decodes a complete TAC-II 16-word data packet.

**TAC-II packet layout (16-bit words, from `PrintAsIfRaw()`):**

| Word | Content |
|------|---------|
| 0 | 0xAAAA sync |
| 1 | trigType |
| 2 | timestampTrig[47:32] |
| 3 | timestampTrig[31:16] |
| 4 | timestampTrig[15:0] |
| 5 | wheel |
| 6 | multiplicity |
| 7 | userRegister |
| 8 | coarseTS |
| 9 | triggerBitMask |
| 10–13 | fourNanoSecCounter[0–3] |
| 14 | vernierAB |
| 15 | vernierCD |

**Key fields:**

| Field | Type | Description |
|-------|------|-------------|
| `timestampTrig` | uint64_t | Trigger timestamp in ns (×10 from raw) |
| `timestampTDC` | uint64_t | TDC coarse timestamp in ns |
| `trigType` | uint16_t | Trigger type code |
| `wheel` | uint16_t | Target wheel position |
| `multiplicity` | uint16_t | Hit multiplicity |
| `userRegister` | uint16_t | User register |
| `coarseTS` | uint16_t | Coarse TDC timestamp |
| `triggerBitMask` | uint16_t | Trigger bit mask |
| `tdcData` | TDC200MHz | Vernier TDC measurement |
| `trashData` | bool | True if counters match trash pattern (0x1006,0x1005,0x1004,0x1003) |
| `phaseOffset[4]` | const float | {0,1,2,3} ns per-channel phase offset |

**FillTDC(uint32_t * data, bool debug):**
- `data[]` is the 32-bit packed payload (each 32-bit word = two 16-bit TAC words)
- Extracts all fields; timestamps converted ×10 to ns
- `timestampTDC` computed as: `(timestampTrig & 0xFFFFFFFF0000) + coarseTS`, roll-forward if < timestampTrig
- Calls `CalVernier()` at end

**CalTAC_simple(bool debug):**
- Computes absolute phase times: `baseTime + fourNanoSec + phaseOffset[i] - 0.05 * vernier[i]`
  - baseTime = `timestampTrig` rounded to 2^18 (262144 ns) grid
  - vernier step = 50 ps (0.05 ns)
- Returns `avgPhaseTimestamp`; returns -1 if trashData

---

## 3. script.cpp — TAC–DIG Coincidence Analysis {#scriptcpp}

**File:** `Aux/script.cpp` (775 lines)  
**Run as:** ROOT macro (CINT): `.x script.cpp`

This is a ROOT analysis script correlating TAC-II hits (from trigger data files) with DIG hits (from digitizer files) to study timing resolution. Contains three historical test sections (A, B, C) — only section C is active (others commented out).

### Helper Functions

**ZeroCrossing(vector\<pair\<double,double\>\> points):**
- 2 points: linear interpolation to find Y=0 crossing
- 3 points: quadratic (Lagrange-style) zero finding
- Used to extract sub-sample CFD timing from CFD_SAMPLE_0/1/2

**packData(uint32_t * data):**
- Reformats a 16-word TAC-II raw array into 32-bit GEB-style packet (mostly for debugging; allocates but may over-index — `payload[10..20]` written but only 10 allocated)

### HitCollection Class

Bridges DIG and TDC hits into a common timestamp-sortable container:

| Field | Meaning |
|-------|---------|
| `timestamp` | Event time in ns (DIG: `EVENT_TIMESTAMP×10`; TAC: `timestampTrig`) |
| `isTAC` | True = TAC-II hit, False = DIG hit |
| `zeroCrossing` | CFD zero-crossing time (DIG hits only, via ZeroCrossing on CFD_SAMPLE_0/1/2) |
| `tdcData` | `TDC200MHz` data (TAC hits only) |

**AddDig():** Computes CFD zero crossing from three sample points at offsets 0, −10, −20 ns.  
**AddTAC():** Sets phase TDC data from TAC hit.

### script() — Active Section C

1. Opens two binary files: one DIG (type 7, non-trigger) and one TAC-II (trigger)
2. Reads all blocks → builds sorted `hitList` (by timestamp)
3. Pairs adjacent hits within 16 µs window → extracts DIG+TAC pairs
4. Fills ROOT TTree (`output.root`) with timing branches:
   - `digTime`, `trigTime`, `avgPhaseTime`, `phaseTime[4]`, `valid[4]`, `vernier[4]`, `vernierOrder`, `baseTime`, `fourNanoSecCounter[4]`, `TDC`, `zeroCrossing`
5. Fills histograms:
   - `hTrigDig0`: TRIG − DIG time difference
   - `hTrigTDC0`: TDC − TRIG difference
   - `hPhaseDigVernier`: phase time − DIG vs vernier index (2D)
   - `hprePhaseVernier`: same but vernier-corrected (adds `+0.05*vernier[i]`)
   - Split by even/odd digitizer timestamp (G = even, G2 = odd)
   - `hTOF`, `hTOF2`: avg phase − DIG; avg phase − CFD zero-crossing
   - `hDiff`: DIG time − CFD zero-crossing

**Timing offset correction:** `offset = -2 ns` when `fmod(digTime, 20) != 0` (odd 10 ns ticks), 0 otherwise.

### Commented Test Sections (A and B)

Historical TAC-only studies:
- **Test A/B:** Single-reader TAC studies: vernier distribution, vernier ordering, 2D phaseTime-vs-TRIG histograms, multi-canvas display. Shows older `tdcData[]` array API (two TDC measurements per hit — superseded by current single `tdcData` member).

---

## 4. script_LED.cpp — LED/CFD Mode Comparison {#script_ledcpp}

**File:** `Aux/script_LED.cpp` (147 lines)  
**Run as:** ROOT macro

Compares LED vs CFD digitizer files: peak timestamp distribution and energy spectrum.

**void script_LED():**
1. Opens 3 binary files: `data/LED_01_000_0160_7`, `data/CFD_01_000_0160_7`, `data/CFD_03_000_0160_7`
2. LED loop: selects hits where `CFD_VALID_FLAG` is set and `CFD_SAMPLE_0` sign is consistent with energy (`POST_RISE_ENERGY - PRE_RISE_ENERGY`); fills `hPeakTime` and `hEnergy`
3. CFD loops: fills peak time and energy for all hits
4. Draws 2-pad canvas: peak time overlay (LED=black, CFD1=red, CFD2=green) + energy overlay

**Note:** LED histogram label is misleading — it actually reads a CFD file (`LED_01_...` is type 7 but the DIG_Hit is decoded in CFD mode based on `CFD_VALID_FLAG` check). The `ZeroCrossing` call present in the loop uses a bug: samples at offsets (0, −10, −20) but passes `CFD_SAMPLE_1` for both the second and third point (copy-paste error). Result is discarded (not filled into any histogram).

---

## 5. Supporting Files {#supporting-files}

| File | Description |
|------|-------------|
| `reader.h` | `Reader` class — block navigator for DGS binary files; owns `DIG_Hit digHit` and `TDC_Hit TAC_hit` members; documented in `ANLDAQ_tcpReceiver.md` |
| `downloadData.sh` | rsync from `slopebox:/global/ioc/dgsReceiver/data/` |
| `readHexFile.sh` | hexdump + awk: print N 32-bit words as 0-indexed hex |
| `constant.h` | Shared constants (included by class_TDC.h) |

---

## 6. See Also {#see-also}

- `ANLDAQ_tcpReceiver.md` — `reader.h`, `tcpReceiver.cpp`, `tcpReceiverMT.cpp`, shell scripts
- `data_structures.md` — GEB header, DIG packet format, TAC-II packet layout
- `DIG_firmware_expert.md` — digitizer firmware: LED/CFD modes, packet header fields
- `ttcl.md` / `ttcl_frame_spec.md` — trigger packet structures
- `tac2.md` — TAC-II hardware overview
