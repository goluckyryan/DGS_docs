# Guceiver — GUI Receiver for Live DIG/TAC-II Monitoring

Stability: C2 - Active / semi-stable

_Source: `DGS_tools_pack/ANLDAQ/gui/Guceiver/` (local repo)_
_Explored: 2026-04-07_

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
  - [Threading model](#threading-model)
- [Connection](#connection)
  - [EPICS side effects](#epics-side-effects)
- [TCP Wire Protocol](#tcp-wire-protocol-get_data)
- [Packet Framing](#packet-framing)
- [DIG Event Decoder](#dig-event-decoder-class_digpy)
  - [Identity / Routing](#identity--routing)
  - [Timestamps](#timestamps-all-in-10-ns-units-unless-noted)
  - [Energy](#energy)
  - [Flags](#flags)
  - [Multiplicity extras](#multiplicity-extras-led-mode-only)
- [TAC-II Decoder](#tac-ii-decoder-class_tacpy)
  - [Packet layout](#packet-layout-16-word-payload)
  - [Timing resolution](#timing-resolution)
- [GUI Tabs](#gui-tabs)
- [Tab Internals](#tab-internals)
  - [class_dataTab.py — DIG Header Inspection Tab](#class_datatabpy--dig-header-inspection-tab)
  - [class_waveTab.py — Waveform Display Tab](#class_wavetabpy--waveform-display-tab)
  - [class_tacTab.py — TAC-II Inspection Tab](#class_tactabpy--tac-ii-inspection-tab)
  - [class_spectrumTab.py — Energy Spectrum Tab](#class_spectrumtabpy--energy-spectrum-tab)
- [Board ID Lookup](#board-id-lookup)
- [Key Files](#key-files)
- [Relationship to tcpReceiver](#relationship-to-tcpreceiver)
- [Cross-References](#cross-references)

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

## TCP Wire Protocol (`get_data()`)

Each polling cycle the Guceiver receiver sends a 4-byte big-endian request and reads a 16-byte reply header, then reads the payload:

**Request** (4 bytes, big-endian `uint32`):
```
0x00000001  — poll request
```

**Reply header** (16 bytes, 4 × big-endian `uint32`):
| Field | Size | Description |
|---|---|---|
| `reply_type` | uint32 | Response type code |
| `record_size` | uint32 | Size of each record in bytes |
| `status` | uint32 | Status code |
| `num_record` | uint32 | Number of records to follow |

The payload is `record_size × num_record` bytes, read in chunks until complete. The entire payload is then unpacked as big-endian `uint32` words and handed to the packet parser.

Socket timeouts: 2 s for connect retries, 5 s for receive during operation.

✅ verified 2026-04-26 — `class_Receiver.py:L226-264` (`get_data()`: `struct.pack(">I", 1)` request; `struct.unpack(">IIII", reply)` 4-field header; loop recv until `record_size * num_record`; `struct.unpack(f">{n}I", data)` big-endian uint32 array)

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

## Tab Internals

### `class_dataTab.py` — DIG Header Inspection Tab

_Source: `ANLDAQ/gui/Guceiver/class_dataTab.py` (400L, code-read 2026-04-26)_ ✅ verified 2026-04-26 — class_dataTab.py:L185-223 (16 GFlagDisplay flags), L307-313 (fillDataArray pause), L339-341 (printPayload+print)

Displays all decoded fields from a single `DIG` event (type 7 or 8), selected by board/channel/data-index spinboxes. Data is pulled from `receiver.dataArray[]` under `data_mutex`.

**Top controls:**
- `Data Index` spinbox: selects which event in `dataArray` to show (auto-advances to latest when not paused)
- `Dig` combobox: selects board by `board_id` (from `board_id_list`); sets `receiver.dig_index`
- `Channel` spinbox (0–9): sets `receiver.channel_index`
- `Update time [msec]` (min 20 ms): changes `plot_timer` interval
- `Pause Update` button: freezes `receiver.fillDataArray = False`, enables index spinbox for manual stepping; also enables `Print Payload` checkbox (prints payload + header to stdout via `data.printPayload()` / `data.print()`)

**Displayed fields (from `DIG` object):**

| Group | Fields |
|---|---|
| Top row | `PACKET_LENGTH`, `GEO_ADDR`, `EVENT_TYPE`, `HEADER_TYPE` |
| Row 2 | `PILEUP_COUNT`, `TRIG_MON_XTRA_DATA` (Multiplicity), `TRIG_MON_DET_DATA` (Target wheel), `MULTIPLEX_DATA` |
| Timestamps | `EVENT_TIMESTAMP` (48-bit, x10 ns), `LAST_DISC_TIMESTAMP` (Previous), `PEAK_TIMESTAMP` (high 32 bits of event TS + 16-bit peak TS), `TS_OF_TRIGGER` (Accept, reconstructed similarly), `TS_OF_COARSE` (12-bit coarse) |
| Energy | `PRE_RISE_ENERGY`, `POST_RISE_ENERGY`, `SAMPLED_BASELINE`, `P2_SUM`, `EARLY_PRE_RISE_ENERGY`, `LAST_POST_RISE_M_SUM` |
| CFD (type 8 only) | `CFD_SAMPLE_0/1/2`, `TRIG_MON_DET_DATA`, `TIMESTAMP_MATCH_FLAG`, `CFD_VALID_FLAG`, `PREVIOUS_CFD_VALID`; group disabled for type 7 |

**Flags (16 `GFlagDisplay` widgets, 4 rows x 4):**

| Flag field | 0 meaning | 1 meaning |
|---|---|---|
| `P2_MODE` | Post-rise buffer len = m (separate); P2 buffer len = p2 | Post-rise buffer len = m-P2 (span); P2 buffer len = p2 | ✅ verified 2026-04-26 — class_dataTab.py:L186
| `CAPTURE_PARST_TS` | Multiplex = extra pre-rise sum at coarse disc. time | Multiplex = bits 27:4 of PARST timestamp |
| `SECOND_THRESH_DISC_FLAG` | Not fired | Coarse threshold also fired |
| `PARST_TSM` | Multiplex is energy, not PARST timestamp | bits 48:28 of PARST TS match current TS |
| `COARSE_FIRED` | No trigger between last/current disc. | Coarse disc. fired between events |
| `PEQ_BYPASS` | Selected by trigger (TTCL mode) | Internal accept-all mode |
| `TRIG_TS_MODE` | Trigger TS = when message arrived at digitizer | Trigger TS = from trigger accept message |
| `CFD_ESUM_MODE` | Pre/post-rise sampled at threshold then at CFD | Pre/post-rise only at threshold + delay d |
| `EARLY_PRE_RISE_SELECT` | Sampled at TS_OF_COARSE + M(pre) | Sampled at TS_OF_COARSE + M(post) + K0 |
| `WRITE_FLAGS` | Waveform bits 15:2 = integer, 1:0 = fractional | Waveform bits 15/14 = timing marks, 13:0 = ADC data |
| `VETO_FLAG` | Not a veto event | Router vetoed but digitizer not honoring veto |
| `EXTERNAL_DISC_FLAG` | Internal discriminator logic | External discriminator source |
| `PEAK_VALID_FLAG` | No peak found; peak TS = 0 | Peak found before holdoff elapsed |
| `OFFSET_FLAG` | Normal readout | Readout delayed due to readout interference |
| `PILEUP_ONLY_FLAG` | Non-pileup events: waveforms suppressed | Waveforms stored only for pileup events |
| `PILEUP_FLAG` | Not pileup | Pileup detected; digitizer set to accept |

---

### `class_waveTab.py` — Waveform Display Tab

_Source: `ANLDAQ/gui/Guceiver/class_waveTab.py` (308L, code-read 2026-04-26)_ ✅ verified 2026-04-26 — class_waveTab.py:L96-98 (Reset Plot Scale button), L117 (RectangleSelector), L128 (button_press_event), L217-222 (onselect syncs Y spinboxes), L226-227 (right-click resets scale)

Displays live ADC waveform traces from `receiver.waveformArray[]` for the selected board/channel using matplotlib.

**Controls:**
- `Wave Index` spinbox: selects waveform from array (auto-advances to latest when not paused)
- `Dig` combobox + `Channel` spinbox: changing either clears `receiver.waveformArray` and resets spinbox
- `Update time [msec]` (min 20 ms): adjustable timer interval
- `Y-min` / `Y-max` spinboxes (range -100000 to +100000): manual Y-axis range; synced to auto-scale result on each update
- `Trace Length` / `Trace Delay`: live EPICS PV widgets reading `{board_name}:raw_data_length{ch}` / `{board_name}:raw_data_delay{ch}`; updated on every redraw
- `Pause Waveform Update` button: freezes `receiver.fillWaveformArray = False`, enables index spinbox
- `Reset Plot Scale` button: rescales Y to waveform data range (5% margin)

**Plot mechanics:**
- Matplotlib `FigureCanvasQTAgg` + `NavigationToolbar2QT` for pan/zoom
- Left-click drag: rectangle-select zoom via `RectangleSelector` (syncs Y spinboxes to zoomed range)
- Right-click: resets plot scale
- X-axis: sample index (clock ticks, 10 ns each); Y-axis: ADC counts
- Auto Y-range computed from waveform min/max with 5% margin on every plot_waveform() call
- Data source: `receiver.waveformArray` under `data_mutex`; array cleared on board/channel change

---

### `class_tacTab.py` — TAC-II Inspection Tab

_Source: `ANLDAQ/gui/Guceiver/class_tacTab.py` (233L, code-read 2026-04-26)_ ✅ verified 2026-04-26 — class_tacTab.py:L109-132 (trigType/wheel/triggerBitMask/multiplicity fields), L178-184 (fillTACArray pause), L210-211 (printPayload), L224-230 (phaseTime/avgPhaseTime/trigType/wheel/userRegister display)

Displays decoded fields from a `TAC` event selected from `receiver.TACArray[]`. No board/channel selector — TAC-II data is global, not per-channel.

**Displayed fields:**

| Group | Fields |
|---|---|
| Top | `timestampTrig` (trigger TS, hex / 10 + raw ns), `coarseTime` (hex + decimal) |
| Row 2 | `timestampTDC` (TDC-derived TS, hex / 10 + raw), `baseTime` (ns) |
| Phase Time group (4 phases) | `fourNanoSecCounter[i]`, `vernier[i]`, `valid[i]`, `phaseTime[i]` (ns, 2dp or N/A if 0) |
| Phase Time row 5 | `avgPhaseTime` (ns, 2dp, or N/A) |
| Misc | `trigType`, `wheel` (target wheel), `userRegister`, `triggerBitMask`, `multiplicity` |

**Pause/index behavior:** identical to Data tab — pause freezes `receiver.fillTACArray = False`, enables spinbox for manual event stepping. `Print Payload` checkbox calls `tac.printPayload()` + `tac.print()` when stopped.

---

### `class_spectrumTab.py` — Energy Spectrum Tab

_Source: `ANLDAQ/gui/Guceiver/class_spectrumTab.py` (295 L)_ ✅ verified 2026-04-26 — full code read

Accumulating online energy histogram for a single configurable DIG channel. Energy values are sourced from `receiver.energyArray` (accumulated across calls, cleared on each plot cycle; unbounded list in Receiver). Energy per event = `(POST_RISE_ENERGY − PRE_RISE_ENERGY) / M_windows` (computed in `class_Receiver.py`).

**Controls:**

| Control | Default | Description |
|---------|---------|-------------|
| Dig selector (ComboBox) | first board | Selects `receiver.dig_index`; clears histogram and plot on change |
| Channel selector (SpinBox) | 0 | Selects `receiver.channel_index` (0–9); clears histogram and plot on change |
| Update time (ms) | 50 | `plot_timer` interval; minimum enforced at 20 ms |
| M-windows (SpinBox) | 1000 | Sets `receiver.M_windows` divisor; clears histogram on change |
| No. Bin | 500 | Histogram bins (100–2000); triggers `clear_histogram()` on change |
| X-min / X-max | 0 / 5000 | x-axis range for histogram; spinboxes interlock (xmin.max = xmax−1, xmax.min = xmin+1) |
| Reset Plot Scale | — | Restores xlim to xRange, ylim to `max(histCount)×1.1` |
| Pause/Resume Spectrum Update | — | Toggles `receiver.fillEnergyArray`; button turns red when paused |

**Histogram accumulation:** On each timer tick, `energyArray` is copied under mutex lock and cleared from the receiver. Events outside [xmin, xmax] are counted as underflow/overflow (not plotted). `np.histogram(energy_data, bins=500, range=xRange)` is used — note: bin count is hardcoded at 500 in `np.histogram()` call regardless of the UI `nBin` spinbox (UI spinbox updates `self.nBin` but `plot_spectrum` passes `bins=500`; appears to be a latent bug). Cumulative counts (`histCount`) accumulate across ticks until cleared. Legend shows total count, underflow, and overflow.

**Interaction:** Left-click drag → rectangular zoom (via `RectangleSelector`). Right-click context menu → Reset Scale or Clear Plot.

**Pause logic:** When paused, `receiver.fillEnergyArray = False` stops energy values from being appended in the receiver thread; the plot tick still fires but finds `energyArray` empty and returns immediately.

**Bug note (latent):** `np.histogram` call uses `bins=500` literal rather than `self.nBin`, so the No. Bin spinbox does not currently affect the actual histogram bin count — it only clears the accumulated data.

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

See [`ANLDAQ.md`](ANLDAQ.md) for the tcpReceiver / production DAQ pipeline.
See [`data_structures.md`](data_structures.md) for the full binary event format.
See [`DIG_firmware_expert.md`](DIG_firmware_expert.md) for DIG header type 7/8 field definitions.

## Cross-References

- [`ANLDAQ.md`](ANLDAQ.md) — Production DAQ pipeline: tcpReceiverMT also connects to IOC TCP:9001
- [`ANLDAQ_GUI_windows.md`](ANLDAQ_GUI_windows.md) — ANLDAQ GUI Windows: Guceiver launch context, Data Taking window
- [`data_structures.md`](data_structures.md) — Full binary event format: DIG (0xAAAAAAAA) and TAC-II (0x0000AAAA) packets
- [`DIG_firmware_expert.md`](DIG_firmware_expert.md) — DIG header type 7/8 field definitions decoded by Guceiver
- [`tac2.md`](tac2.md) — TAC-II TDC: vernier interpolation, data format decoded by the TAC-II tab
- [`gebsort.md`](gebsort.md) — GEBMerge/GEBSort offline analysis: processes the same TCP data stream written to disk by tcpReceiverMT
