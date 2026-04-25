# ANLDAQ tcpReceiver/Aux — Offline ROOT Analysis Framework

**Source:** `ANLDAQ/tcpReceiver/Aux/`
Stability: C2 - Active / semi-stable
**Last updated:** 2026-04-25

## Overview

`tcpReceiver/Aux/` is a standalone offline ROOT-based analysis framework for reading and processing DGS binary data files. It is separate from the production `tcpReceiverMT` receiver — it is used for **post-hoc inspection, timing analysis, and calibration** of DIG and TAC-II data files written by `tcpReceiverMT`.

**Not a production runtime component.** Requires ROOT (CERN data analysis framework).

## Files

| File | Description |
|------|-------------|
| `class_DIG.h` | `DIG_Hit` class: decodes raw DIG event packet header (type 7 or 8) |
| `class_TDC.h` | `TDC_Hit` + `TDC200MHz` classes: decodes TAC-II trigger packets, computes vernier timing |
| `reader.h` | `Reader` class: opens DGS binary files, scans blocks, reads DIG or TAC data |
| `script.cpp` | ROOT analysis script (main): correlates DIG+TAC hits, computes CFD zero-crossings |
| `script_LED.cpp` | Variant of `script.cpp` tuned for LED-mode DIG data |
| `downloadData.sh` | Shell helper (presumably for fetching data files) |
| `readHexFile.sh` | Shell helper for hex dumping |

Depends on `../constant.h` (`TRIG_DATA_SIZE=16`, `TRIG_PACKET_LENGTH=10`). ✅ verified 2026-04-25 - ANLDAQ/tcpReceiver/constant.h

---

## DIG_Hit (class_DIG.h)

Decodes DGS digitizer event packets (header type 7 or 8). **Does not call `ntohl` itself** — byte-order conversion (network→host) is performed by `reader.h` before the raw array is passed to `DecodeHeader_7_8()`. ✅ verified 2026-04-25 - class_DIG.h (no ntohl calls), reader.h:L191 (ntohl applied to data[] before decode)

### Key fields decoded

| Field | Source words | Description |
|-------|-------------|-------------|
| `CH_ID` | Word 1 [3:0] | Channel ID | ✅ verified 2026-04-25 - class_DIG.h:L24,L236 |
| `GEO_ADDR` | Word 1 [31:27] | Board geographic address | ✅ verified 2026-04-25 - class_DIG.h:L27,L239 |
| `PACKET_LENGTH` | Word 1 [26:16] | Number of 32-bit words in packet | ✅ verified 2026-04-25 - class_DIG.h:L26,L238 |
| `EVENT_TIMESTAMP` | Words 2–3 [15:0] | 48-bit event timestamp (×10 ns = real time in ns) | ✅ verified 2026-04-25 - class_DIG.h:L29 |
| `HEADER_TYPE` | Word 3 [19:16] | Header type (7 or 8) | ✅ verified 2026-04-25 - class_DIG.h:L31 |
| `CFD_ESUM_MODE` | Word 3 bit 22 | CFD/ESUM mode flag (added 2021-08-17) | ✅ verified 2026-04-25 - class_DIG.h:L34 |
| `EVENT_TYPE` | Word 3 [25:23] | Trigger type code from trigger accept message bits [10:8] | ✅ verified 2026-04-25 - class_DIG.h:L35 |
| `PEAK_VALID_FLAG` | Word 4 bit 9 | Valid peak detected | ✅ verified 2026-04-25 - class_DIG.h:L42 |
| `PILEUP_FLAG` | Word 4 bit 15 | Pileup detected | ✅ verified 2026-04-25 - class_DIG.h:L45 |
| `SAMPLED_BASELINE` | Word 6 [23:0] | Sampled baseline value | ✅ verified 2026-04-25 - class_DIG.h:L48 |
| `PRE_RISE_ENERGY` | Word 8 [23:0] | Pre-rise energy sum | ✅ verified 2026-04-25 - class_DIG.h:L52 |
| `POST_RISE_ENERGY` | Word 9 [15:0] + Word 8 [31:24] | Post-rise energy sum | ✅ verified 2026-04-25 - class_DIG.h:L53 |
| `CFD_SAMPLE_0/1/2` | Words 5, 7 | 3 CFD samples (CFD mode only), used for zero-crossing fit | ✅ verified 2026-04-25 - class_DIG.h:L79-81,L306,L311-312 (CFD_SAMPLE_0 from Word 5[29:16]; CFD_SAMPLE_1/2 from Word 7[13:0]/[29:16]) |
| `trace` | Words 14+ | Optional waveform trace: two 14-bit samples per 32-bit word | ✅ verified 2026-04-25 - class_DIG.h:L89-90,L359-363 |

**Decode entry point:** `DIG_Hit::DecodeHeader_7_8(uint32_t* data, int packageLen)` ✅ verified 2026-04-25 - class_DIG.h:L233

---

## TDC_Hit + TDC200MHz (class_TDC.h)

Decodes TAC-II trigger packets (GEB type 15, 10-word format). Computes sub-4-ns vernier timing.

### TDC200MHz struct

Holds the 200 MHz TDC timing data from a single TAC-II trigger event:

| Field | Description |
|-------|-------------|
| `fourNanoSecCounter[4]` | 4 ns counters for each of 4 TDC channels |
| `vernierAB`, `vernierCD` | Raw vernier words (packed 6-bit fields per channel) |
| `vernier[4]` | Decoded vernier values (0–63), 50 ps/LSB |
| `valid[4]` | Per-channel validity flags (from validBit nibble) |
| `phaseTime[4]` | Per-channel phase time in ns: `baseTime + 4ns_count×4 + phaseOffset[i] − 0.05×vernier[i]` |
| `avgPhaseTimestamp` | Average of valid phase times (ns) |
| `standardDeviation` | σ of valid phase times |
| `maxDiff` | Max deviation from average among valid channels |

**Phase offset constants** (in `TDC_Hit`): `{0, 1, 2, 3}` ns per channel. ✅ verified 2026-04-25 - class_TDC.h:L111

**Vernier resolution:** 50 ps/LSB (`vernier × 0.05 ns`). ✅ verified 2026-04-25 - class_TDC.h:L235 (comment: `0.05 ns = 50 ps`)

### TDC_Hit class

| Field | Description |
|-------|-------------|
| `timestampTrig` | MTRG trigger timestamp (ns), from TAC-II packet words 1–2 |
| `timestampTDC` | TDC timestamp (ns), derived from coarseTS + 48-bit MTRG base |
| `trigType` | Trigger type word |
| `wheel` | Wheel position word |
| `multiplicity` | Hit multiplicity |
| `userRegister` | User register value |
| `coarseTS` | Coarse TDC timestamp (16-bit, 10 ns LSB) |
| `triggerBitMask` | Trigger bit mask |
| `trashData` | True if 4ns counters == known trash pattern (`0x1006,0x1005,0x1004,0x1003`) | ✅ verified 2026-04-25 - class_TDC.h:L200-202 |

**Key decode method:** `FillTDC(uint32_t* data, bool debug)` ✅ verified 2026-04-25 - class_TDC.h:L183-212
- Extracts 48-bit trigger timestamp from words [2:1] ✅ verified - class_TDC.h:L190
- Derives `timestampTDC` from coarseTS with rollover handling (`+0x10000` raw units if behind trigger, then ×10) ✅ verified - class_TDC.h:L204-210
- Converts both to ns (×10) ✅ verified - class_TDC.h:L209-210
- Calls `TDC200MHz::CalVernier()` for sub-ns timing ✅ verified - class_TDC.h:L212

**Timing computation:** `CalTAC_simple(bool debug)` ✅ verified 2026-04-25 - class_TDC.h:L221-262
- Base time = `timestampTrig` rounded down to 2^18 = 262144 ns (≈262 µs) boundary ✅ verified - class_TDC.h:L224
- Phase time per channel = `baseTime + counter×4 + offset[i] − vernier[i]×0.05` (ns) ✅ verified - class_TDC.h:L235
- Averages valid channels → `avgPhaseTimestamp` ✅ verified - class_TDC.h:L237,L241

---

## Reader (reader.h)

Reads DGS binary files block by block (each block = one event).

### Constructor / open

```cpp
Reader(std::string fileName, bool isTAC = true, bool isGEB = false);
```
- `isTAC`: true → TAC-II data (fixed 10-word packets); false → DIG data (variable-length) ✅ verified 2026-04-25 - reader.h:L21-22,L38
- `isGEB`: true → file has GEB headers prepended (4 words each); false → raw FPGA packets ✅ verified 2026-04-25 - reader.h:L127-134

### Key methods

| Method | Description |
|--------|-------------|
| `ScanNumBlock()` | Fast-scan entire file; builds `blockPos[]` index; returns total block count | ✅ verified 2026-04-25 - reader.h:L230-251 |
| `ReadNextBlock(fastRead, debug)` | Read next event block; populates `TAC_hit` or `digHit`; returns 0 on success, -10 on EOF, -1 on header error | ✅ verified 2026-04-25 - reader.h:L112-114,L124,L143 |
| `ReadBlock(index, verbose)` | Seek to block `index` and read it (requires prior `ScanNumBlock()`) |
| `PrintPayLoad()` | Print raw payload words for debugging |

### Block format (non-GEB)

- First word must be `0xAAAAAAAA` (sync header) ✅ verified 2026-04-25 - reader.h:L141
- TAC: fixed `TRIG_PACKET_LENGTH=10` words total
- DIG: variable; length from word 1 bits [26:16] + 1

### GEB mode

When `isGEB=true`, each block is preceded by a 4-word GEB header (skipped for payload parsing; first payload word is replaced with `0xAAAAAAAA`).

---

## script.cpp — Offline Analysis Script

ROOT macro combining DIG and TAC data from parallel file streams.

### Key logic

- Opens separate `Reader` instances for DIG data file and TAC trigger file
- **`HitCollection` class**: unified hit container holding either `DIG_Hit` or `TDC_Hit`
  - For DIG: timestamp = `EVENT_TIMESTAMP × 10` ns; computes CFD zero-crossing from 3 samples
  - For TAC: timestamp = `timestampTrig`; zero-crossing from `tdcData.avgPhaseTimestamp`
- **CFD zero-crossing**: fits 3 CFD samples (at offsets 0, −10, −20 ns) using quadratic or linear interpolation via `ZeroCrossing()` function → sub-sample timing

### ZeroCrossing function

- 2 points: linear interpolation between sign-change pair
- 3 points: quadratic (Lagrange interpolation), returns x at y=0
- Returns −1 if insufficient data or no zero crossing

### packData function

Repacks a 38-word raw TAC VME packet → 21-word offline format (NOTE: the production `receiver.h` repacks to 10 words; `packData` here is a different/extended format for offline use, appears incomplete/experimental based on out-of-bounds array access in comments).

---

## script_LED.cpp

Variant of `script.cpp` for LED-mode DIG data (LED = Leading-Edge Discriminator, as opposed to CFD = Constant Fraction Discriminator). Opens paired DIG data files and applies `ZeroCrossing` for timing analysis.

---

## Notes

- These files are **analysis tools** — not compiled into the production `tcpReceiverMT` binary
- Requires a working ROOT installation to compile/run `script.cpp` and `script_LED.cpp`
- `class_DIG.h` and `class_TDC.h` are self-contained header-only libraries reusable in other analysis contexts
- The `packData` function in `script.cpp` has an out-of-bounds bug (allocates 10 words, writes indices up to 20) — it appears to be an experimental/in-progress function

---

## Cross-References

| File | Relationship |
|------|--------------|
| `ANLDAQ_tcpReceiver.md` | Production `tcpReceiverMT` binary these Aux files accompany |
| `data_structures.md` | DIG event packet format decoded by `class_DIG.h` |
| `dgs_analysis.md` | Main analysis tools; these aux scripts are lighter-weight alternatives |
| `guceiver.md` | Another online DIG data viewer (uses same `class_DIG.h` patterns) |

---

*Created: 2026-04-25 | Last reviewed: 2026-04-25*
