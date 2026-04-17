# DIG — Event Packet Format

→ Part of the DIG Digitizer Firmware documentation. See also **[deep_fpga_DIG.md](deep_fpga_DIG.md)** (architecture, source files, signal flow) and **[deep_fpga_DIG_channel.md](deep_fpga_DIG_channel.md)** (per-channel signal processing).

## Table of Contents

- [Overview](#overview)
- [36-bit FIFO word framing (internal)](#36-bit-fifo-word-framing-internal)
- [LED Header (14 words, words 0–13)](#led-header-14-words-words-013)
- [Split field reconstruction](#split-field-reconstruction)
- [Signal and integration timeline](#signal-and-integration-timeline)
- [Two-event timeline — inter-event decay correction](#two-event-timeline--inter-event-decay-correction)
- [Measured quantities](#measured-quantities)
- [Pole-zero correction](#pole-zero-correction)
- [CFD Header differences (words 4–7 only)](#cfd-header-differences-words-47-only)
- [CFD zero-crossing interpolation](#cfd-zero-crossing-interpolation)
- [Waveform samples (after header)](#waveform-samples-after-header)
- [Cross-References](#cross-references)

## Overview

When a trigger is accepted, the channel readout machine assembles a packet and writes it into the per-channel `acptd_event_fifo` (36-bit, `Channel_Readout_Mach.vhd`). The `channel_FIFO_readout_mach` arbitrates across all 10 channels and feeds the external IDT 7007 FIFO. LVME reads the FIFO over VME (A32/D32, 32-bit words).

Each packet consists of a **14-word header** (words 0–13) followed by zero or more **waveform sample words**. Source: `Event_Header_FIFO.vhd`, `event_packer.vhd`.

## 36-bit FIFO word framing (internal)

| Bits | Field | Description |
|------|-------|-------------|
| 35:32 | EOP | `0001` = last word of packet; `0000` = normal ✅ verified 2026-04-16 — `event_packer.vhd:L165,169` (`acptd_event_fifo_din(35 downto 32) <= "0001"` = EOP; `"0000"` = normal) |
| 31:0 | Data | 32-bit header word or waveform sample pair |

Bits [35:32] are used internally for packet framing and are not forwarded to VME — the host sees 32-bit words only. ✅ verified 2026-04-16 — `Channel_FIFO_Readout_Mach.vhd:L78` (`coll_fifo_data_extra: std_logic_vector(35 downto 32)` — "upper four bits of the din die here, as they can't go to the external FIFO")

## LED Header (14 words, words 0–13)

```
Bit layout:  31 30 29 28 27 26 25 24 23 22 21 20 19 18 17 16 15 14 13 12 11 10 09 08 07 06 05 04 03 02 01 00
Word  0:  |                              FIXED 0xAAAAAAAA (event delimiter)  ✅ verified 2026-04-16 — `Event_Header_FIFO.vhd:L90` (`cEVENT_DELIMITER := X"AAAAAAAA"`)                             |
Word  1:  |  GeoAddr[4:0]  | PacketLen[10:0] (filled at readout) |     UserDef[11:0]     |  CH_ID[3:0]   |
Word  2:  |                              TIMESTAMP_OF_EVENT[31:0]                                         |
Word  3:  | HDR_LEN[5:0] | EVT_TYPE[2:0] | 0 |TM|PM| HEADER_TYPE[3:0] |  TIMESTAMP_OF_EVENT[47:32]       |
Word  4:  |     TS_OF_LAST_EVENT[15:0]    |PU|PO|GE|SE| 0 |OF|PV|ED| 0 |VF|WF|PE|       "00000"          |
Word  5:  |                              TS_OF_LAST_EVENT[47:16]                                          |
Word  6:  |  "0000"  | PU_CNT[3:0] |                    SAMPLED_BASELINE[23:0]                           |
Word  7:  |              TRIG_MON_DET_DATA[15:0]              |       TRIG_MON_XTRA_DATA[15:0]            |
Word  8:  | POST_RISE_SUM[7:0]  |                       PRE_RISE_SUM[23:0]                               |
Word  9:  |          TS_OF_PEAK[15:0]             |            POST_RISE_SUM[23:8]                        |
Word 10:  |       TS_OF_TRIGGER[15:0] (filled at readout)     |CP|P2|          P2_SUM[13:0]              |
Word 11:  |  PREV_POST[23:16]  |                       MPX_FIELD[23:0]                                   |
Word 12:  |  PREV_POST[15:8]   |                     EARLY_PRE_RISE_SUM[23:0]                            |
Word 13:  |  PREV_POST[7:0]    |  TS_OF_COARSE[9:0]  |CF|PC|PT|ST|          P2_SUM[23:14]               |
```

Note: `MPX_FIELD[23:0]` in Word 11 is the multiplexed field — when `CP` (Word 10 bit 15) = 0 it holds a 2nd early pre-rise energy; when CP = 1 it holds `TS_OF_LAST_PREAMP_RESET[23:0]`.

**Single-word or already-contiguous fields:**

| Field | Location | Description |
|-------|----------|-------------|
| GeoAddr[4:0] | W1[31:27] | Board geographic address (VME slot ID) |
| PacketLen[10:0] | W1[26:16] | Total packet length in 32-bit words (filled at readout) |
| UserDef[11:0] | W1[15:4] | User tag from `reg_user_package_data` |
| CH_ID[3:0] | W1[3:0] | Channel number (0–9) |
| HDR_LEN[5:0] | W3[31:26] | Header length constant = 28 ✅ verified 2026-04-16 — `Event_Header_FIFO.vhd` DGS build: `constant cHEADER_LENGTH := 28` (36-bit FIFO words; 14 VME words × 2 per FIFO word) |
| EVT_TYPE[2:0] | W3[25:23] | Event type (filled at readout) |
| TM | W3[21] | `TRIG_TS_MODE`: 0 = use arrival TS; 1 = use trigger-mux TS |
| PM | W3[20] | `PEQ_BYPASS`: 1 = pending event queue bypassed |
| HEADER_TYPE[3:0] | W3[19:16] | Format: `0100` = LED; `0101` = CFD |
| SAMPLED_BASELINE[23:0] | W6[23:0] | Baseline estimate latched at event time (ADC counts × M) |
| TRIG_MON_DET_DATA[15:0] | W7[31:16] | Detector trigger monitor data from Frame 2 |
| TRIG_MON_XTRA_DATA[15:0] | W7[15:0] | Extra trigger monitor data from Frame 2 |
| PRE_RISE_SUM[23:0] | W8[23:0] | Pre-peak energy integral (M samples) — see Energies |
| TS_OF_PEAK[15:0] | W9[31:16] | Lower 16 bits of 48-bit timestamp at pulse peak |
| TS_OF_TRIGGER[15:0] | W10[31:16] | Lower 16 bits of 48-bit timestamp when trigger arrived |
| TS_OF_COARSE[9:0] | W13[23:14] | Coarse discriminator timestamp (10-bit) |
| PU_CNT[3:0] | W6[27:24] | Number of simultaneous pileup events |

**Status flag bits (Word 4):**

| Bit | Abbrev | Meaning |
|-----|--------|---------|
| 15 | PU | Pileup flag: another event was in-flight when this one fired |
| 14 | PO | Pileup-waveform-only mode active |
| 13 | GE | `TS_OF_COARSE` extension bit 1 (replaced GENERAL_ERROR in 2023) |
| 12 | SE | `TS_OF_COARSE` extension bit 0 (replaced SYNC_ERROR in 2023) |
| 10 | OF | Offset flag (filled at readout) |
| 9 | PV | Peak valid: peak-finding algorithm found a clean peak |
| 8 | ED | External discriminator flag: event was triggered externally |
| 6 | VF | Veto flag (filled at readout) |
| 5 | WF | Write-flags mode: 1 = header-only, no waveform data written |
| 4 | PE | `EARLY_PRE_M_SEL`: which early pre-rise window was used |

**Status flag bits (Word 13):**

| Bit | Abbrev | Meaning |
|-----|--------|---------|
| 13 | CF | `COARSE_FIRED`: coarse discriminator fired on this event |
| 12 | PC | `PARST_TSM`: preamp reset timestamp matched |
| 10 | ST | `SECOND_THRESH`: second (higher) threshold of thresh_disc satisfied |

**Word 10 control bits:**

| Bit | Abbrev | Meaning |
|-----|--------|---------|
| 15 | CP | `CAPTURE_PARST_TS`: 1 = MPX_FIELD (Word 11[23:0]) holds `TS_OF_LAST_PREAMP_RESET` |
| 14 | P2 | `P2_MODE`: P2 sum integration mode |

## Split field reconstruction

Several values are spread across non-contiguous bit fields. Reconstruction (all values are unsigned unless noted):

```
TIMESTAMP_OF_EVENT[47:0]   = (Word3[15:0]  << 32) | Word2[31:0]
TS_OF_LAST_EVENT[47:0]     = (Word5[31:0]  << 16) | Word4[31:16]      (LED only)
POST_RISE_SUM[23:0]        = (Word9[15:0]  <<  8) | Word8[31:24]
P2_SUM[23:0]               = (Word13[9:0]  << 14) | Word10[13:0]
PREV_POST[23:0]            = (Word11[31:24]<< 16) | (Word12[31:24]<< 8) | Word13[31:24]
```

For CFD mode (Words 4–7 differ):

```
TS_OF_LAST_EVENT[29:0]     = (Word5[13:0]  << 16) | Word4[31:16]      (CFD, 30-bit only)
TRIG_MON_DET_DATA[15:0]    = (Word4[3:0]   << 12) | (Word5[31:30]<<10) | (Word5[15:14]<<8) | Word6[31:24]
PILEUP_COUNT[3:0]          = (Word7[31:30] <<  2) | Word7[15:14]      (CFD, split around CFD samples)
```

## Signal and integration timeline

The vertical axis is time (one tick = 10 ns at 100 MHz). The horizontal axis shows the ADC signal amplitude from a charge-sensitive germanium preamplifier: a flat baseline, a fast step rise (~100–200 ns charge collection), a flat plateau, and a slow exponential decay tail (preamp RC feedback). Energy sums and timestamps are annotated on the right.

```
  Time  Amplitude (ADC)         Integration windows         Timestamps
  ▼     0────────────────────   ─────────────────────────   ──────────────────────────
        │
        │  baseline ─────────   ┐
        │  baseline ─────────   │ EARLY_PRE_RISE_SUM         ← TS_OF_LAST_PREAMP_RESET
        │  baseline ─────────   ┘   (optional early baseline;   [23:0] (MPX_FIELD, CP=1)
        │  baseline ─────────       captured at coarse disc      anywhere in the past)
        │
        │  baseline ─────────   ┐
        │  baseline ─────────   │
        │  baseline ─────────   │ PRE_RISE_SUM
        │  baseline ─────────   │ (M samples = reg_m_window × 10 ns)
        │  baseline ─────────   │ baseline reference; subtract from POST_RISE for energy
        │  baseline ─────────   ┘
        │ /
        │/ rising edge ───────────────────────────────────   ← TIMESTAMP_OF_EVENT [47:0]
        /  (~100–200 ns                                          LED: threshold crossing
        │   charge collection)                                   CFD: sign-flip clock
        │\                                                        (±10 ns, see CFD interp.)
        │ ─── plateau ───────   ┐
        │ ─── plateau ───────   │                            ← TS_OF_PEAK [15:0]
        │ ─── plateau ───────   │ POST_RISE_SUM                  (lower 16 bits; at peak
        │ ─── slow decay ────   │ (M samples = reg_m_window × 10 ns)  of filtered signal)
        │ ─── slow decay ────   │ signal + baseline
        │ ─── slow decay ────   │ net energy ≈ POST_RISE − PRE_RISE
        │ ─── slow decay ────   ┘
        │ ─── tail ──────────   ┐
        │ ─── tail ──────────   │ P2_SUM
        │ ─── tail ──────────   ┘ (reg_p2_window × 10 ns; tail/pileup characterization)
        │  baseline ─────────
        ┊
        ┊  (2–20 µs gap)        ─────────────────────────   ← TS_OF_TRIGGER [15:0]
        ┊                                                        (lower 16 bits; trigger
        ┊                                                        accept/reject from Router)
```

**Notes on timestamps not on the main pulse timeline:**

- `TIMESTAMP_OF_EVENT [47:0]` is the full 48-bit counter value (10 ns/tick). All other `TS_OF_*` fields are the **lower N bits** only — you reconstruct absolute time by combining with the upper bits of `TIMESTAMP_OF_EVENT`.
- `TS_OF_LAST_EVENT [47:0]` (LED) — the previous discriminator fire on this channel. Precedes the current event; see two-event timeline below. `ΔT = TIMESTAMP_OF_EVENT − TS_OF_LAST_EVENT` gives inter-event interval and drives decay-tail correction of baseline windows.
- `TS_OF_COARSE [11:0]` — coarse (BGO/NaI scintillator) discriminator timestamp. Asynchronous; can fall before or after `TIMESTAMP_OF_EVENT`. Coincidence gates check `|TS_OF_COARSE − TIMESTAMP_OF_EVENT| < window`.
- `TS_OF_LAST_PREAMP_RESET [23:0]` (MPX_FIELD when CP=1) — time of last Ge preamp reset (inhibit pulse). Typically µs to ms in the past; used to assess whether the preamp was still recovering at event time.
- `EARLY_PRE_RISE_SUM` — captured at the coarse discriminator time, not at the main discriminator time. It provides a second, earlier baseline window and is useful when the preamp baseline is drifting or when standard PRE_RISE might catch the tail of a preceding pulse.

## Two-event timeline — inter-event decay correction

`TS_OF_LAST_EVENT` and `PREV_POST` belong to the **previous event**. The preamp RC
tail from that event persists into the current event's baseline windows, biasing
`PRE_RISE_SUM` and `POST_RISE_SUM`. With known τ the packet supplies all inputs
needed to remove this bias.

```
  Time  Amplitude (ADC)         Integration windows         Timestamps / notes
  ▼     0────────────────────   ─────────────────────────   ──────────────────────────

  ── PREVIOUS EVENT ──────────────────────────────────────────────────────────────────
        │  baseline ─────────
        │ /
        │/ prev rising edge ──────────────────────────────   ← TS_OF_LAST_EVENT [47:0]
        /  (~100–200 ns)                                         (stored in current packet)
        │ ─── plateau ───────   ┐
        │ ─── slow decay ────   │ PREV_POST                      step height A_prev
        │ ─── slow decay ────   │ (POST_RISE_SUM of prev event;  ≈ PREV_POST/M − V_base
        │ ─── slow decay ────   ┘  M samples from prev peak)
        │ ─── tail ──────────   ┐
        │ ─── tail ──────────   │ (prev P2, not stored)
        │ ─── tail ──────────   ┘
        │
        ┊ ← ΔT = TIMESTAMP_OF_EVENT − TS_OF_LAST_EVENT ──────────────────────────────
        ┊   exponential tail A_prev · exp(−t/τ) contaminates baseline windows below
        │
  ── CURRENT EVENT ───────────────────────────────────────────────────────────────────
        │  "baseline"────────   ┐                           ← TS_OF_LAST_PREAMP_RESET
        │  "baseline"────────   │ EARLY_PRE_RISE_SUM            [23:0] (MPX_FIELD, CP=1)
        │  "baseline"────────   ┘ = V_base·M + tail_e           anywhere in the past
        │  "baseline"────────
        │
        │  "baseline"────────   ┐
        │  "baseline"────────   │ PRE_RISE_SUM
        │  "baseline"────────   │ = V_base·M + tail_p
        │  "baseline"────────   │ (tail_p < tail_e: more time elapsed)
        │  "baseline"────────   ┘
        │ /
        │/ rising edge ───────────────────────────────────   ← TIMESTAMP_OF_EVENT [47:0]
        /  (~100–200 ns)
        │ ─── plateau ───────   ┐
        │ ─── plateau ───────   │                            ← TS_OF_PEAK [15:0]
        │ ─── plateau ───────   │ POST_RISE_SUM
        │ ─── slow decay ────   │ = (V_base + A_curr)·M + tail_q
        │ ─── slow decay ────   │ (tail_q < tail_p: more time elapsed)
        │ ─── slow decay ────   ┘
        │ ─── tail ──────────   ┐
        │ ─── tail ──────────   │ P2_SUM
        │ ─── tail ──────────   ┘ (reg_p2_window × 10 ns)
        │  baseline ─────────
        ┊
        ┊  (2–20 µs gap)                                     ← TS_OF_TRIGGER [15:0]
```

`tail_e`, `tail_p`, `tail_q` = `A_prev · exp(−t/τ) · M` evaluated at the midpoint
of each window, where `t` is measured from `TS_OF_LAST_EVENT`.

**Packet inputs for decay-tail correction:**

| Input | Derived from | Notes |
|-------|-------------|-------|
| `ΔT` | `TIMESTAMP_OF_EVENT − TS_OF_LAST_EVENT` | Full 48-bit, 10 ns/tick |
| `A_prev` (prev step height) | `PREV_POST/M − SAMPLED_BASELINE` | Needs `reg_m_window` from run config |
| POST_RISE start offset | `TS_OF_PEAK[15:0] − TIMESTAMP_OF_EVENT[15:0]` | Variable; accounts for pulse rise time |
| `τ` | Detector calibration | Not in packet |

**Alternative — two-window baseline solve.** With τ known, EARLY_PRE and PRE provide
two measurements of the same decaying tail at different times, giving two equations
in two unknowns (V_base and A_prev), without needing PREV_POST:

```
EARLY_PRE / M = V_base + A_prev · exp(−t_e / τ)
PRE       / M = V_base + A_prev · exp(−t_p / τ)
```

The time offsets `t_e` and `t_p` (from `TS_OF_LAST_EVENT` to each window's midpoint)
are fixed for a run and come from register settings (coarse disc timing, `reg_k_window`).

---

## Measured quantities

**Energies** — all are 24-bit unsigned sums of ADC counts:

| Quantity | Words | Physical meaning |
|----------|-------|-----------------|
| `PRE_RISE_SUM` | W8[23:0] | Integral of M samples ending at the pulse onset. Represents baseline × M. Used as the baseline reference for energy subtraction. |
| `POST_RISE_SUM` | W8[31:24] + W9[15:0] | Integral of the signal region from just before the peak onwards, for a fixed window. This is the primary energy measurement. Net energy ≈ POST_RISE_SUM − PRE_RISE_SUM. |
| `P2_SUM` | W10[13:0] + W13[9:0] | Integral of the pulse tail after POST_RISE, for `reg_p2_window` cycles. Used for pileup characterization and decay-tail correction. |
| `EARLY_PRE_RISE_SUM` | W12[23:0] | A second, earlier baseline integral window (before M). Available when `EARLY_PRE_M_SEL` = 1. Useful for double-pulse baseline isolation. |
| `SAMPLED_BASELINE` | W6[23:0] | Running baseline estimate from `baseline_tracker.vhd` latched at event time. Tracks DC level on a ~10 µs timescale. |
| `PREV_POST` | W11[31:24]+W12[31:24]+W13[31:24] | The POST_RISE_SUM of the immediately preceding event on this channel. Available in pileup-recording mode (`PILEUP_DISABLE = 1`) for pileup correction. |

Typical net energy computation:
```
Energy = POST_RISE_SUM - PRE_RISE_SUM
```
For precise spectroscopy (baseline drift correction):
```
Energy = POST_RISE_SUM - SAMPLED_BASELINE × (POST_RISE window length in samples)
```

**Timestamps** — the 48-bit counter runs at 100 MHz (10 ns/tick), synchronized to MTRG SYNC frames:

| Quantity | Words | Physical meaning |
|----------|-------|-----------------|
| `TIMESTAMP_OF_EVENT[47:0]` | W2 + W3[15:0] | Full 48-bit event time. In LED mode: leading-edge threshold crossing. In CFD mode: the clock tick when the zero-crossing sign flip was detected (see CFD interpolation below). |
| `TS_OF_LAST_EVENT[47:0]` | W4[31:16] + W5 | Full 48-bit timestamp of the previous event on this channel (LED). Difference `TIMESTAMP_OF_EVENT − TS_OF_LAST_EVENT` gives the inter-event interval for dead-time and pileup analysis. |
| `TS_OF_PEAK[15:0]` | W9[31:16] | Lower 16 bits of the timestamp at the pulse peak (the sample where the peak-finding algorithm detected the maximum). `TS_OF_PEAK − TIMESTAMP_OF_EVENT[15:0]` ≈ signal rise time. |
| `TS_OF_TRIGGER[15:0]` | W10[31:16] | Lower 16 bits of the timestamp when the trigger message arrived from the Router (~2–20 µs after the event, within the TRIG_DELAY window). |
| `TS_OF_COARSE[9:0]` + GE/SE | W13[23:14] + W4[13:12] | 12-bit coarse discriminator timestamp (TS_OF_COARSE is 10 bits; GE and SE extend it by 2 more). Marks when the coarse (BGO/NaI) discriminator fired, for Ge–BGO coincidence gating. |
| `TS_OF_LAST_PREAMP_RESET[23:0]` | W11[23:0] (MPX_FIELD, when CP=1) | Lower 24 bits of the timestamp of the most recent preamp reset pulse. Used to measure and correct for decay-tail artifacts in Ge detectors after a reset. |

## Pole-zero correction

The Ge charge-sensitive preamplifier output decays exponentially after each pulse
with a characteristic time τ (the feedback RC constant). This tail biases the energy
sums of subsequent events. Correcting for it requires two steps: baseline
reconstruction and sum correction.

**Signal model**

After a pulse of step height A at time t₀ (full charge collection), the preamp output is:

```
V(t) = V_base + A · exp(−(t − t₀) / τ)     for t ≥ t₀
```

With multiple events, tails from all prior pulses accumulate on V_base. In practice
only the immediately preceding event contributes significantly, and the packet
provides the fields needed for that correction.

**Step 1 — Reconstruct the tail amplitude**

Two equivalent approaches, both computable from the packet:

*Approach A — PREV_POST + ΔT:*
```
A_prev = PREV_POST / M − SAMPLED_BASELINE          (previous pulse step height)
tail_0 = A_prev · exp(−ΔT / τ)                    (tail level at TIMESTAMP_OF_EVENT)

where ΔT = TIMESTAMP_OF_EVENT − TS_OF_LAST_EVENT  (full 48-bit difference, 10 ns/tick)
```

*Approach B — two-window baseline solve (PREV_POST not required):*

EARLY_PRE and PRE measure the same decaying tail at two different, known times,
giving two equations in two unknowns (V_base, A_prev):
```
EARLY_PRE / M = V_base + A_prev · exp(−t_e / τ)
PRE       / M = V_base + A_prev · exp(−t_p / τ)
```
Subtracting eliminates V_base; solving gives A_prev directly. The offsets t_e and
t_p (from TS_OF_LAST_EVENT to each window midpoint) come from register settings
(coarse disc timing, `reg_k_window`) — fixed for a run, not per-event.

**Step 2 — Correct PRE_RISE_SUM and POST_RISE_SUM**

The previous pulse tail contributes a bias to each integration window. The exact
integral of the tail over an M-sample window starting at time t_start (measured
from TS_OF_LAST_EVENT) is:

```
tail_in_window(t_start) = A_prev · exp(−t_start / τ) · τ · (1 − exp(−M / τ))
```

For M << τ (typical: M ~ 2 µs, τ ~ 50–500 µs for Ge) this simplifies to:
```
tail_in_window(t_start) ≈ A_prev · exp(−t_start / τ) · M
```

The two window start times (measured from TS_OF_LAST_EVENT):
```
t_p0 = ΔT − M                                           (start of PRE_RISE window)
t_q0 = ΔT + (TS_OF_PEAK[15:0] − TIMESTAMP_OF_EVENT[15:0])  (start of POST_RISE window)
```

`TS_OF_PEAK` supplies the variable POST_RISE start offset, which depends on the
pulse rise time and varies event-by-event.

Corrected sums:
```
PRE_corrected  = PRE_RISE_SUM  − tail_in_window(t_p0)
POST_corrected = POST_RISE_SUM − tail_in_window(t_q0)
```

**Corrected net energy**

```
E_corrected = POST_corrected − PRE_corrected
            = (POST_RISE_SUM − PRE_RISE_SUM)
              − A_prev · τ · (1 − exp(−M/τ)) · [exp(−t_q0/τ) − exp(−t_p0/τ)]
```

For M << τ:
```
E_corrected ≈ (POST_RISE_SUM − PRE_RISE_SUM)
              − A_prev · M · [exp(−t_q0/τ) − exp(−t_p0/τ)]
```

The second term is the pole-zero correction. Since t_q0 > t_p0, exp(−t_q0/τ) <
exp(−t_p0/τ), so the correction is negative — it reduces the apparent energy,
removing the tail that was artificially elevating PRE_RISE relative to POST_RISE.

**Within-pulse decay correction (systematic)**

Independently of inter-pulse tails, the current pulse itself decays during the
M-sample POST_RISE window, so POST_RISE_SUM underestimates A_curr · M:

```
POST_RISE_SUM(from current pulse) = A_curr · τ · (1 − exp(−M/τ)) ≈ A_curr · M · (1 − M/(2τ))
```

Dividing by the scale factor recovers the true energy. This is a run-constant
multiplicative correction that depends only on M and τ:
```
scale = (τ / M) · (1 − exp(−M / τ))
E_true = E_corrected / scale
```

**Inputs summary**

| Quantity | Source | Comment |
|----------|--------|---------|
| `ΔT` | `TIMESTAMP_OF_EVENT − TS_OF_LAST_EVENT` | Full 48-bit, 10 ns/tick |
| `A_prev` | `PREV_POST/M − SAMPLED_BASELINE` (A) or two-window solve (B) | Step height of previous pulse |
| POST_RISE start | `TS_OF_PEAK[15:0] − TIMESTAMP_OF_EVENT[15:0]` | Variable rise-time offset, per-event |
| `M` | `reg_m_window` from run config | Samples per integration window (cycles × 10 ns) |
| `τ` | Detector calibration | Not stored in packet |

**Limitations**

- Only the immediately preceding event is correctable via PREV_POST and
  TS_OF_LAST_EVENT. At high rates or with long τ, tails from earlier events
  accumulate; SAMPLED_BASELINE partially absorbs the DC drift but lags by ~10 µs.
- After a preamp reset the tail structure is disrupted; `TS_OF_LAST_PREAMP_RESET`
  (MPX_FIELD, CP=1) identifies affected events for flagging or exclusion.

---

## CFD Header differences (words 4–7 only)

In CFD mode (`HEADER_TYPE = 0101`), words 0–3 and 8–13 are identical to LED. Words 4–7 carry different fields:

```
Word  4:  |     TS_OF_LAST_EVENT[15:0]    |PU|PO|GE|SE|CV|OF|PV|ED|TF|VF|WF|PE| TDD[15:12]             |
Word  5:  | TDD[11:10] |  CFD_SAMPLE(0)[13:0]   | TDD[9:8] |   TS_OF_LAST_EVENT[29:16]                  |
Word  6:  |    TDD[7:0]    |                        SAMPLED_BASELINE[23:0]                               |
Word  7:  | PUC[3:2] |  CFD_SAMPLE(2)[13:0]  | PUC[1:0] |         CFD_SAMPLE(1)[13:0]                   |
```

Additional CFD status bits in Word 4:

| Bit | Abbrev | Meaning |
|-----|--------|---------|
| 11 | CV | `CFD_VALID_FLAG`: the zero-crossing was clean and valid |
| 7 | TF | `TSM_FLAG`: upper bits of timestamp match previous event (pileup proximity check) |

Note: In CFD mode `TS_OF_LAST_EVENT` is only 30 bits (not 48). `TRIG_MON_DET_DATA` replaces the `PU_CNT + SAMPLED_BASELINE` slot in Words 4/6; SAMPLED_BASELINE is still in W6[23:0]. `PILEUP_COUNT` is split across Word 7 flanking the two CFD samples. See reconstruction formulas above.

## CFD zero-crossing interpolation

The three CFD samples are 14-bit signed values of `LOCAL_DIFFERENCE = CFD_SUBTRACTION − LOCAL_ZERO`, the shifted CFD waveform. They are captured at the sign-flip clock:

| Sample | Clock relative to sign flip | Sign | Description |
|--------|-----------------------------|------|-------------|
| `CFD_SAMPLE(1)` | T − 1 (10 ns before) | **positive** | Last sample above zero |
| `CFD_SAMPLE(0)` | T = `TIMESTAMP_OF_EVENT` | **negative** | First sample below zero |
| `CFD_SAMPLE(2)` | T − 2 (20 ns before) | positive | Earlier sample (quadratic correction) |

`TIMESTAMP_OF_EVENT` is latched at the same clock as `CFD_SAMPLE(0)`, so the actual zero-crossing lies somewhere in the 10 ns interval `[T−1, T]`.

**Linear interpolation** (sufficient for most purposes):

```
S1 = CFD_SAMPLE(1)   (positive, 14-bit signed)
S0 = CFD_SAMPLE(0)   (negative, 14-bit signed)

sub_sample_fraction  = S1 / (S1 - S0)        -- value in [0, 1]

t_zero = (TIMESTAMP_OF_EVENT - 1 + sub_sample_fraction) × 10 ns
```

Equivalently, as a correction referenced directly to `TIMESTAMP_OF_EVENT`:
```
correction = -S0 / (S1 - S0) × 10 ns         -- value in (-10 ns, 0]
t_zero = TIMESTAMP_OF_EVENT × 10 ns + correction
```

**Quadratic interpolation** using all three samples (for best timing resolution):

The three points at t = T−2, T−1, T with values S2, S1, S0 define a parabola. The zero crossing of the parabola:
```
a = (S0 - 2×S1 + S2) / 2
b = (4×S1 - S0 - 3×S2) / 2
c = S2
root = (-b - sqrt(b²- 4×a×c)) / (2×a)     -- root in [0, 2], relative to T-2
t_zero = (TIMESTAMP_OF_EVENT - 2 + root) × 10 ns
```

In practice the linear formula is used; quadratic only matters when the signal slope changes rapidly at the crossing (very asymmetric pulse shapes).

The `CFD_VALID_FLAG` (CV bit) confirms that the zero-crossing logic completed normally within the holdoff window. If CV = 0, the samples may be unreliable and the event should be discarded from timing analysis.

## Waveform samples (after header)

After word 13, zero or more waveform words follow. The number of samples is controlled by `reg_raw_data_length`.

| Bits | Field | Description |
|------|-------|-------------|
| 31:16 | Sample N | ADC sample (16-bit, sign-extended from 14-bit ADC) |
| 15:0 | Sample N+1 | Next ADC sample |

Two consecutive ADC samples (100 MHz, 10 ns/sample) are packed into each 32-bit word. The waveform window is positioned relative to the discriminator fire by `reg_raw_data_window`. Decimation (1×, 2×, 4×, …, 128×) is applied by `decimator.vhd` and automatically pauses around the pulse rise time to preserve full-rate timing accuracy near the discriminator crossing.

If `reg_raw_data_length = 0`, no waveform words are written and the packet ends after word 13.

---

## Cross-References

- `knowledgeBase/deep_fpga_DIG.md` — DIG architecture, source files, signal flow, SERDES, build options
- `knowledgeBase/deep_fpga_DIG_channel.md` — Per-channel signal processing: LED/CFD discriminator, delay chain, pileup, VME FPGA
- `knowledgeBase/DIG_firmware_expert.md` — Expert guide: all readout modes, data format, timing, trigger_mux_select
- `knowledgeBase/data_structures.md` — GEB binary format: DIG event packet layout
- `knowledgeBase/pole_zero.md` — Pole-zero correction detail and register settings

---
*Source: `DGS_tools_pack/raw_FPGA/Dig*/` — VHDL source. PDF: `ANL Digitizer Firmware for Experts.pdf`. Split from `deep_fpga_DIG.md`: 2026-04-16.*
