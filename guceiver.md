# Guceiver — GUI Receiver for Live DIG/TAC-II Monitoring

Stability: C2 - Active / semi-stable

_Source: `DGS_tools_pack/ANLDAQ/gui/Guceiver/` (local repo)_
_Explored: 2026-04-07_

---

## Overview

**Guceiver** is a PyQt6 application that connects directly to an IOC's TCP data stream (port 9001) to provide live monitoring of digitizer and TAC-II data. It is distinct from the main ANLDAQ commander GUI — it's a lightweight standalone viewer for online diagnostics. ✅ verified 2026-04-10 — `Guceiver.py:L1` (`from PyQt6...`); confirmed standalone (not part of commander) by absence of commander imports

Key capabilities:
- Live waveform display (oscilloscope-style, per channel/board)
- Energy spectrum accumulation (online histogram, no EPICS needed)
- Raw event data inspection (header fields decoded)
- TAC-II TDC data display (timing, vernier, multiplicity)

The `Guceiver.py` main class can also be embedded as a child widget inside the commander GUI (it checks for `parent().btn_startRun`).

---

## Architecture

```
Guceiver.py (QMainWindow)
├── class_Receiver.py  — TCP socket thread, packet parser
│   ├── class_DIG.py   — DIG event decoder + field storage
│   └── class_TAC.py   — TAC-II event decoder + field storage
├── class_waveTab.py   — Waveform display tab (matplotlib)
├── class_spectrumTab.py — Energy spectrum tab (matplotlib histogram)
├── class_dataTab.py   — Raw data/header display tab
└── class_tacTab.py    — TAC-II data display tab
✅ verified 2026-04-10 — `Guceiver.py:L115-127` (4 `addTab` calls: "Waveform", "Spectrum", "Data", "TAC-II")
```

### Threading model
- `Receiver` lives in a dedicated `QThread`
- GUI updates via `QTimer` at 500 ms interval ✅ verified 2026-04-23 — `Guceiver.py:L138` (`self.timer.setInterval(500)`)
- `QMutex` protects shared arrays between receiver thread and GUI thread ✅ verified 2026-04-23 — `class_Receiver.py:L3,L26` (`QMutex`; `data_mutex = QMutex()`)
- Four separate arrays: `waveformArray` (cap 100), `dataArray` (cap 100), `TACArray` (cap 100) — all capped at 100 entries and pop from front when full. `energyArray` is unbounded (accumulates full spectrum; each entry = `(POST_RISE_ENERGY - PRE_RISE_ENERGY) / M_windows` where `M_windows=1000`). ✅ verified 2026-04-23 — `class_Receiver.py:L37-50,L177-210`

---

## Connection

- Connects to IOC TCP port **9001** (same port as tcpReceiver) ✅ verified 2026-04-08 — `Guceiver.py:L63` (`{ip}:9001`)
- IOC IP list loaded from `$IOC_IP` environment variable (space-separated list) ✅ verified 2026-04-23 — `Guceiver.py:L58-60` (`os.environ.get("IOC_IP", "")`)
- User selects which IOC from a dropdown; connection established on "Start Receiver"

### EPICS side effects
On Start: `caput("Online_CS_SaveData", "Save")` + `caput("Online_CS_StartStop", "Start")` ✅ verified 2026-04-21 — `Guceiver.py:L172-173`
On Stop: `caput("Online_CS_StartStop", "Stop")` + `caput("Online_CS_SaveData", "No Save")` ✅ verified 2026-04-21 — `Guceiver.py:L210-211`

These PVs control the online data-saving state in the IOC.

---

## Packet Framing

The receiver identifies packet type by the first word (magic):

| Magic word | Type |
|---|---|
| `0xAAAAAAAA` | Digitizer (DIG) event ✅ verified 2026-04-08 — `class_Receiver.py:L145` |
| `0x0000AAAA` | TAC-II event ✅ verified 2026-04-08 — `class_Receiver.py:L147` |

Packets that don't match header types 7 or 8 are discarded (except `channel_id == 0xD` = "Type D" end-of-run sentinel → stops acquisition). ✅ verified 2026-04-21 — `class_Receiver.py:L160-162` (`if channel_id == 0xD: print("Type D data received, stopping data taking.")`)

Packet length is extracted from word[1] bits `[26:16]` (`payloadMaxIndex` in 32-bit words, includes header + waveform).

---

## DIG Event Decoder (`class_DIG.py`)

Decodes header type 7 (LED) and type 8 (CFD) packets. Parsed fields:

### Identity / Routing
| Field | Description |
|---|---|
| `CH_ID` | Channel ID (0–9) |
| `USER_DEF` | Board ID (user-defined, read from EPICS `user_package_data` PV) |
| `GEO_ADDR` | VME geographic address |
| `HEADER_TYPE` | 7 = LED, 8 = CFD |
| `EVENT_TYPE` | Readout mode (0–10, see DIG_firmware_expert.md) |
| `PACKET_LENGTH` | Total packet size in 32-bit words |
| `HEADER_LENGTH` | Header size in 32-bit words (14) ✅ verified 2026-04-21 — `class_DIG.py:L212` (comment: "word 14 and beyound are waveform data", i.e. words 0–13 = 14-word header) |

### Timestamps (all in 10 ns units unless noted)
| Field | Description |
|---|---|
| `EVENT_TIMESTAMP` | 48-bit: time when discriminator fired |
| `PEAK_TIMESTAMP` | Time of signal peak (0 if peak not found) |
| `TS_OF_TRIGGER` | Trigger timestamp from MTRG |
| `TS_OF_COARSE` | Coarse discriminator timestamp |
| `LAST_DISC_TIMESTAMP` | Timestamp of previous discriminator firing on this channel |

### Energy
| Field | Description |
|---|---|
| `PRE_RISE_ENERGY` | Accumulator sum before discriminator (baseline region) |
| `POST_RISE_ENERGY` | Accumulator sum after discriminator (signal region) |
| `SAMPLED_BASELINE` | Instantaneous baseline sample |
| `P2_SUM` | Secondary accumulator sum |
| `LAST_POST_RISE_M_SUM` | Post-rise sum from previous event |
| `EARLY_PRE_RISE_ENERGY` | Early pre-rise sum (if `EARLY_PRE_RISE_SELECT` set) |

Energy = `POST_RISE_ENERGY - PRE_RISE_ENERGY` (baseline-subtracted). Pole-zero correction applied separately in analysis (GEBSort / `pz_from_parquet.py`).

### Flags
| Flag | Meaning |
|---|---|
| `PILEUP_FLAG` | Pileup detected |
| `PEAK_VALID_FLAG` | Peak was found within holdoff |
| `VETO_FLAG` | Event vetoed |
| `CFD_VALID_FLAG` | CFD zero-crossing found (CFD mode only) |
| `COARSE_FIRED` | Coarse discriminator fired |
| `EXTERNAL_DISC_FLAG` | External discriminator input triggered |

### Multiplicity extras (LED mode only)
- `TRIG_MON_XTRA_DATA` — lower 16 bits of word 7; described as "multiplicity or other data" in class_DIG ✅ verified 2026-04-23 — `class_DIG.py:L19,L138` (`payload[7] & 0x0000FFFF`)
- `TRIG_MON_DET_DATA` — upper 16 bits of word 7 (LED mode) or reconstructed from word 4 nibble + words 5–6 (CFD mode); described as "target wheel or other data" ✅ verified 2026-04-23 — `class_DIG.py:L20,L139,L146`

> **Note:** Earlier KB entries incorrectly described TRIG_MON_XTRA_DATA as "X-plane multiplicity (clean)" and TRIG_MON_DET_DATA as "Y-plane multiplicity (dirty)" — these labels are not present in the source code. Actual usage depends on what the IOC configures in the TTCL monitor frame.

---

## TAC-II Decoder (`class_TAC.py`)

Decodes TAC-II TDC packets (magic `0x0000AAAA`). The TAC-II provides fine-timing (vernier interpolation) for coincidence analysis.

### Packet layout (16-word payload) ✅ verified 2026-04-21 — `class_Receiver.py:L199` (`payloadMaxIndex = 15` → fixed 16 words for TAC-II)

| Word index | Field |
|---|---|
| [1] | `trigType` — trigger type code |
| [2:4] | `timestampTrig` — 48-bit MTRG timestamp (×10 ns) |
| [5] | `wheel` — target wheel position |
| [6] | `multiplicity` — event multiplicity |
| [7] | `userRegister` — user-defined register |
| [8] | `coarseTime` — TDC coarse time |
| [9] | `triggerBitMask` — trigger bit pattern |
| [10:13] | `fourNanoSecCounter[4]` — 4 ns resolution counters for 4 TDC channels |
| [14] | `vernierAB` — vernier for TDC channels 0 & 1 (6 bits each + 4-bit valid) |
| [15] | `vernierCD` — vernier for TDC channels 2 & 3 |

✅ verified 2026-04-17 — `class_TAC.py:L36-50` (`decode()` method: payload indices match exactly; `timestampTrig = (payload[2]<<32)+(payload[3]<<16)+payload[4]`; `vernierAB=payload[14]`, `vernierCD=payload[15]`)

### Timing resolution
- Coarse: 4 ns (`fourNanoSecCounter` × 4)
- Vernier: 0.05 ns steps (6-bit, 0–63 → 0–3.15 ns interpolation)
- Final `phaseTime[i]` = baseTime + coarse×4 + channel_offset − vernier×0.05 (channel_offset = `[0, 1, 2, 3]` ns for channels 0–3 ✅ verified 2026-04-17 — `class_TAC.py:L28` `self.offset = [0, 1, 2, 3]`)
- `avgPhaseTime` = average over valid channels

`timestampTrig` and `timestampTDC` both converted to ns (×10).

---

## GUI Tabs

| Tab | Content |
|---|---|
| **Waveform** | Live oscilloscope: last N waveforms for selected board/channel |
| **Spectrum** | Online energy histogram: `(POST_RISE_ENERGY - PRE_RISE_ENERGY) / M_windows` per event; configurable bins (100–2000, default 500), x-range adjustable ✅ verified 2026-04-11 — `class_Receiver.py:L182`, `class_spectrumTab.py:L267` |
| **Data** | Raw header field display (all DIG fields decoded) |
| **TAC-II** | TAC timing data: trigger/TDC timestamps, vernier values, multiplicity |

Plot update interval: **50 ms default** (selectable via UI spinbox); paused when tab not visible. Status bar updates every **500 ms** via a separate timer. ✅ verified 2026-04-19 — `Guceiver.py:L137-138` (status timer=500ms), `class_waveTab.py:L112-113` (plot timer=50ms default; comment erroneously says 100ms), `class_spectrumTab.py:L108-109` (50ms), `class_dataTab.py:L268-269` (50ms)

---

## Board ID Lookup

At startup, Guceiver reads `{board_name}:user_package_data` via EPICS CA for each board in the `dig_board_list` argument. This maps board name → integer board ID used to filter events in the receiver. ✅ verified 2026-04-11 — `Guceiver.py:L35-38` (`pv_name = f"{bd_name}:user_package_data"`, `board_id = epics.caget(pv_name, timeout=1)`)

---

## Key Files

| File | Role |
|---|---|
| `Guceiver.py` | Main window, layout, start/stop logic |
| `class_Receiver.py` | TCP receiver thread, packet framing, routing to DIG/TAC decoders |
| `class_DIG.py` | DIG event class + `decode_data()` method |
| `class_TAC.py` | TAC-II event class + `decode()` method |
| `class_waveTab.py` | Waveform tab (matplotlib canvas) |
| `class_spectrumTab.py` | Spectrum tab (matplotlib histogram) |
| `class_dataTab.py` | Data inspection tab |
| `class_tacTab.py` | TAC-II display tab |

---

## Relationship to tcpReceiver

Guceiver connects to the **same TCP port 9001** as `tcpReceiver` (the production data-to-disk receiver). It is a diagnostic tool, not a replacement — it can run instead of `tcpReceiver` for quick checks, or it may be run simultaneously if the IOC supports multiple connections.

See `knowledgeBase/ANLDAQ.md` for the tcpReceiver / production DAQ pipeline.
See `knowledgeBase/data_structures.md` for the full binary event format.
See `knowledgeBase/DIG_firmware_expert.md` for DIG header type 7/8 field definitions.

## Cross-References

- [`ANLDAQ.md`](ANLDAQ.md) — Production DAQ pipeline: tcpReceiverMT also connects to IOC TCP:9001
- [`ANLDAQ_GUI_windows.md`](ANLDAQ_GUI_windows.md) — ANLDAQ GUI Windows: Guceiver launch context, Data Taking window
- [`data_structures.md`](data_structures.md) — Full binary event format: DIG (0xAAAAAAAA) and TAC-II (0x0000AAAA) packets
- [`DIG_firmware_expert.md`](DIG_firmware_expert.md) — DIG header type 7/8 field definitions decoded by Guceiver
- [`tac2.md`](tac2.md) — TAC-II TDC: vernier interpolation, data format decoded by the TAC-II tab
