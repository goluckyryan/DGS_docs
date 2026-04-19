# DIG Firmware Expert Reference
_Source: ANL Digitizer Firmware for Experts PDF (SVN rev #6185, Sept 13 2021, J. Anderson)_
_72 pages. Intended for users working directly with firmware parameters._

---

## Table of Contents
- [Overview](#overview)
- [Memory Structures in Xilinx FPGAs](#memory-structures-in-xilinx-fpgas)
- [Channel Initialization](#channel-initialization)
- [Discriminator Operation](#discriminator-operation)
- [Constant-Fraction Discriminator (CFD) Mode](#constant-fraction-discriminator-cfd-mode)
- [Pileup Logic](#pileup-logic)
- [Pileup Handling and Event Formation](#pileup-handling-and-event-formation)
- [External Discriminator Modes](#external-discriminator-modes)
- [Timing Marks](#timing-marks)
- [Energy Summation](#energy-summation)
- [Data Readout](#data-readout)
- [Trigger Window Calculation](#trigger-window-calculation)
- [Data Format](#data-format)
- [Energy Summation Logic — Operational Details](#energy-summation-logic--operational-details)
- [Discriminator Filter Details](#discriminator-filter-details)
- [CFD Mode — Detailed Settings](#cfd-mode--detailed-settings)
- [Trigger Delay Buffer](#trigger-delay-buffer)
- [Waveform Readout — Details](#waveform-readout--details)
- [Readout Modes (8 Combinations)](#readout-modes-8-combinations)
- [Trigger Interface](#trigger-interface)
- [Master and Slave Digitizers](#master-and-slave-digitizers)
- [Slave Digitizer Operational Modes](#slave-digitizer-operational-modes)
- [On-Board DAC Diagnostics](#on-board-dac-diagnostics)
- [Auxiliary Input Interface (ExtTTL Injection Point)](#auxiliary-input-interface-extttl-injection-point)
- [Glossary](#glossary)
- [See Also](#see-also)

---

## Overview

The 10-channel DIG firmware runs as a continuously operating data pipeline. Each channel is based on multiple delay buffers of varying width. Many timing functions derive from these delay settings.

### ADC Input Range & Linearity
_Source: [ANL Digitizer Firmware for Advanced Users](https://wiki.anl.gov/gsdaq/ANL_Digitizer_Firmware_for_Advanced_Users) (wiki)_

Inputs are differential. ADC chip nominally supports ±2V differential, but input buffer circuitry limits usable range:
- **Best linearity:** ±1.2V differential = ±4,916 ADC counts from 0V (nominal 0V = 8,192 counts → range 3,276–13,108) ✅ verified 2026-04-13 — jta_channel.vhd:L572 (midscale 0+ recoded to offset binary = 0x2000 = 8192)
- **Relaxed use:** ±1.5V differential = ±6,144 ADC counts (range 2,048–14,336)
- **Beyond ±1.5V:** marked non-linearity / signal compression
- Conversion: **4,096 counts/volt** (ADC designed for 16,384 codes over 4V) ✅ verified 2026-04-13 — AD6645 is 14-bit (Digitizer.vhd:L59: ADC_DATA_PINS slv_13_0); 16384/4V = 4096 counts/V; ±1.2V×4096=4915.2≈4916 ✓, ±1.5V×4096=6144 ✓

Summary: For Ge signals, keep within ±1.2V for best energy resolution; ±1.5V acceptable for most uses.

---

## Memory Structures in Xilinx FPGAs

Two forms of memory:

- **Block RAM**: 18kbit units, fixed sizes (1Kx18, 512x36, etc.). Width always a multiple of 9 (parity bits). Scattered at fixed locations — routing difficulty increases as more are used.
- **Distributed RAM**: Uses logic slice memory (16x1 LUTs). Arbitrary width, always near the logic using it. Practical limit ~128x16 before ganging overhead dominates.

**SRL shift registers**: Distributed RAM can be configured as 16-element shift registers (SRL macros) using internal connections. Serially ganging SRLs creates arbitrarily long delay buffers without address counters — significant resource savings. Used throughout the channel pipelines.

ADC data is 14 bits wide. Mix of Distributed RAM and 1Kx18 Block RAM leaves 4 bits available for timing across the delay chain. These extra bits are used for state machine delays: a pulse enters the delay buffer input, the state machine waits for it to exit the other end — eliminates delay counters and comparison logic.

**Terminology (important):**
- **Latch**: D flip-flop internal to the FPGA (no connection outside). May have multiple inputs via built-in MUX.
- **Register**: D flip-flop that connects outside the FPGA (used to load control values or read status).

---

## Channel Initialization

All delay parameters can be loaded into registers in any order without affecting pipeline operation. After loading, a single write to **LOAD_VALS** (pulsed control register) triggers a self-timed initialization sequence.

**Sequence:**
1. LOAD_VALS → resets all delay buffer lengths to new values
2. **CHAIN_INVALID_PULSE** enters pipeline at P1 input and propagates through
3. Tap delays of CHAIN_INVALID_PULSE initialize energy sum logic at appropriate times:
   - INVALID_P1_TO_P2 → P2 sum starts adding
   - INVALID_P2_TO_M1 → P2 tracks, M1 starts adding
   - INVALID_M1_TO_K0 → M1 starts tracking
   - INVALID_D3_TO_M2 → M2 starts adding
   - INVALID_M2_TO_T1 → M2 starts tracking

**CHAIN_INVALID** stays asserted for the entire duration; **CHAIN_INVALID_PULSE** is a short pulse at start.

**SUBSECTION_RESETS(9:0)**: Set to "1111111111" on LOAD_VALS, held until CHAIN_VALID_PULSE exits pipeline, then zeroes shift in from bit 0 to bit 9. Different bits control different channel subsystems (discriminator, pileup, etc.) allowing ordered release.

**Best init practice**: Set initial ADC value register to estimated baseline for each channel before LOAD_VALS. This prevents false step at start. ENERGY_INVALID (derived from CHAIN_INVALID delayed by post-rise buffer length) further suppresses the threshold discriminator after chain release. First discriminator edge after reset is ignored as secondary safeguard. Full tracking can take ~25 µs.

---

## Discriminator Operation

Two discriminators per channel:
- **Coarse discriminator**: Always leading-edge mode. Fixed delay buffer depth. Same polarity as main. Used for fast multiplicity trigger for auxiliary detectors. Also captures early copies of pre-rise sum before main discriminator fires.
- **Main discriminator**: Configurable — leading-edge or CFD mode, per-channel polarity.

**Digital filter** (main discriminator): A 9-tap FIR filter with Pascal's triangle coefficients 1-8-28-56-70-56-28-8-1 (= (1+2z⁻¹+z⁻²)⁴). Equivalent to convolving Y(n) = X(n) + 2·X(n-1) + X(n-2) with itself 4 times. Coefficients sum to 256 → lower 8 bits discarded in normalization; output is 15-bit (14-bit magnitude + 1 sign bit). Implemented in `single_filter.vhd` using DSP48 multipliers (×28, ×70) + shifts for ×1, ×8, ×56 terms. ✅ verified 2026-04-12 — `single_filter.vhd:L1-85` (coefficients, 15-bit output `dout(14 downto 0)`), `triple_filter.vhd:L18` ("sign bit added" comment)

Three copies of this filter are applied to ADC samples: entering K buffer, between K and D, and exiting D. A MUX routes two copies to the leading-edge discriminator based on operating mode.

### Leading-Edge (Threshold) Discriminator

Subtracts "delayed" input from "prompt" input every 10 ns. For positive pulses, difference grows as signal rises. Fires when difference exceeds threshold — **discriminates slope, not level**.

- Amplitude depends on 'd' buffer length and signal slope
- Larger 'd' → more sensitive to small/slow signals; smaller 'd' → less time walk
- 'd' < 10 not recommended (numerical truncation errors)
- Recommended 'd' = 16 for HPGe detectors (<1 µs rise time) → 160 ns coarse discriminator delay

**Discriminator Hold-off**: On first threshold crossing, loads hold-off counter; discriminator disabled for that many 10-ns ticks. Ensures one fire per edge. Output pulse is 10 ns wide (one clock).

**Preamp Reset Kill**: Handles transistor-reset preamplifiers (Gammasphere uses these). Preamplifier integrates leakage current → slow sawtooth ramp → periodic transistor resets. SBX differentiator converts signals to exponentials, turns resets into large opposite-polarity spikes. Preamp Reset Kill uses an opposite-polarity discriminator with large preset threshold to detect reset, then starts a delay count during which main discriminator is disabled. Reset rate: every few ms to few hundred ms depending on neutron damage. SBX clamp time: ~200–250 µs → dead time from PRK not significant unless detector is near needing anneal.

**Peak Detector** (in LED mode): When discriminator fires, saves filtered ADC value at "prompt" input. Every clock, compares current prompt to saved value; if still climbing, updates. Peak declared when saved value not updated for N consecutive clocks (N = peak sensitivity value, typically 4). If peak found before holdoff expires → timestamp saved in header, PEAK_VALID set. If holdoff expires first → peak timestamp = 0, PEAK_VALID not set. Optional: early holdoff termination when peak found (useful for variable rise time signals).

---

## Constant-Fraction Discriminator (CFD) Mode

Channel design changes slightly:
- Leading-edge discriminator moves forward one delay block to span 'k' buffer (instead of 'd')
- CFD monitors 'd' buffer
- LED used to gate CFD (prevents false firings)
- All three filter elements used: middle filter → LED (delayed input) AND CFD (prompt input)

**CFD operation**: CFD state machine idle until LED fires. Saves LOCAL_ZERO (CFD equation value at LED fire time). Then calculates CFD equation – LOCAL_ZERO each clock. Fires when sign changes. LOCAL_ZERO subtraction ensures consistent operation even when DC level wanders (e.g. during pileup). ✅ verified 2026-04-15 — `cfd_disc.vhd:L209-211` (`LOCAL_DIFFERENCE <= CFD_SUBTRACTION - LOCAL_ZERO`; `LOCAL_ZERO` latched on `CFD_SAMPLE_ZERO`); sign change detected at `L314` (`LOCAL_DIFFERENCE(13) = SIGN_TO_TRACK`)

**Timestamp interpolation**: Three CFD sample values in header (at latch time, one tick earlier, two ticks earlier). Linear regression → intercept at zero → subtract fractional offset from timestamp. Accuracy ~1.7 ns (1σ) for large signals, ~2.5 ns for small signals at 800–1000 ns rise time. ✅ verified 2026-04-15 — `cfd_disc.vhd:L190-191, L332-334` (`INT_CFD_SAMPLES` 3-entry shift register of `LOCAL_DIFFERENCE`; captured on `CAPTURE_TIMESTAMP` pulse as `CFD_SAMPLES(2..0)`) — 1.7/2.5 ns figures from expert PDF

**Improper CFD detection**: If holdoff falls during second pulse rise, LOCAL_ZERO latched late → CFD fires at wrong fraction. Sign check: correct firings have first two samples same sign, third opposite (or zero). All-same-sign → erroneous. Minimized by enabling early holdoff termination at peak.

---

## Pileup Logic

**Pileup counter**: 4-bit counter (max 15). Increments when discriminator fires; decrements when a delayed copy of discriminator bit exits delay chain formed by M2 (pre-rise) + K + K0. In CFD mode, LED (across 'k') used for pileup timing (LED must work for CFD to work; safer to base pileup on LED). ✅ verified 2026-04-06 — `jta_channel.vhd:L219` (`PILEUP_COUNT : std_logic_vector(3 downto 0)`), overflow at `"1111"` confirmed L1224

Pileup inspection time = m + k0 + k. Example: m=6 µs → max hit rate = 1/400 ns without overflow (2× typical HPGe rise time). Proper holdoff ending at peak prevents overflow. Counter overflow → pileup overflow state → requires software reset.

**First hit in train** = Accepted Hit (if pileup allowed or no pileup). **Second and subsequent hits** in train = Extended Events.

---

## Pileup Handling and Event Formation

**Pileup rejection ON**: All pileup-train hits discarded; only non-pileup hits become Accepted Hits; no Extended Events.
**Pileup rejection OFF**: Accepted Hits and Extended Events accepted.

**Header formation stages**:
1. Discriminator fires → wide latch captures timestamp, energy sums, most header data
2. During holdoff: peak timestamp/value added if found
3. End of holdoff → latch copied to **PEHQ** (Putative Event Header Queue, depth 16)
4. End of pileup inspection time → PEHQ entry transferred to **Event Header FIFO** (Accepted Hit) or discarded

**Holdoff < pileup time** is a hard requirement. If violated: PU_TIME_ERROR_FLAG set → readout blocked. Fix by adjusting holdoff/pileup and re-initializing channels (no power cycle needed).

**PEHQ latch timing**: Collection starts at holdoff end of event(n), ends at holdoff end of event(n+1). Items loaded at different times:
- Most items: when discriminator fires
- Early pre-rise energy: when coarse discriminator fires
- Third pre-rise copy: at m + k0 after coarse discriminator fires
- Coarse discriminator timestamp delta (for pre-rise timing calculation)
- Peak timestamp: when peak found
- CFD_VALID: initially '0' on LED fire, overwritten to '1' if CFD fires

**PEQ (Pending Event Queue)**: Used in triggered modes. Accepted Hit timestamps enter PEQ. Each entry has: 3-bit trigger type, pending bit, accepted bit, ROMS (ReadOutMachineSignaled) bit.

**Triggered vs. Internal Accept All**: In "internal accept all" mode, PEQ bypassed → all Accepted Hits immediately become Accepted Events.

**Event Expiry**: Accepted Hit inserts 10ns pulse into T1+T2 (Trigger Delay Buffer, 2048 cells, 20.48 µs total). When pulse exits as EVENT_EXPIRED → Event Header FIFO popped, PEQ entry marked invalid. Events whose pending bit was set but never accepted die silently at expiry.

---

## External Discriminator Modes

Per-channel selection of external discriminator source, then second selection:
- Ignored
- ANDed with main discriminator (external gates main)
- ORed with main discriminator
- Replaces main discriminator

**External discriminator sources** (8 selections, vary by master/slave and channel):
- '0' (disabled)
- Channel 9 signal (master) or ch9 of master digitizer (slave)
- Front panel RS-485 input (AUX_DIN[10], MSbit of 11-bit Aux I/O) — only works when bit 10 is configured as input
- Timestamp bit (periodic signals at 0.745 Hz / 6.0 Hz / 23.8 Hz / 95.4 Hz / 1.5 kHz / 48.8 kHz / 195.3 kHz) ✅ verified 2026-04-18 — `Timestamp_Generator.vhd:L238-244` (TEST RATE 0–6 comments match exactly)
- VME command (pulsed control register; no system-wide sync, diagnostic use)
- Trigger command (Frame 15 of 20-frame cycle, with channel selection mask; system-wide sync)
- Preamp Reset Kill signal of adjacent channel (ch. n+5, master only)
- Another channel's signal

**Timestamp-based external discriminator uses:**
- "External only" mode → acts like auto-trig oscilloscope (baseline measurement)
- "OR" mode → background heartbeat to force periodic hits + capture edges (useful for low-rate detectors)
- Higher speeds: test systemic bandwidth, pileup response, or cable delays

---

## Timing Marks

ADC data = 14 bits; 32-bit VME words pack two 16-bit halves. Bit 14 of each 16-bit half encodes timing marks. Bit 14 = 0 normally; rising to 1 starts a 3-bit serial timing mark.

Four valid 3-bit codes (all start with 1): 100, 101, 110, 111

| Code | LED mode | CFD mode (energy@LED) | CFD mode (energy@CFD) |
|------|----------|-----------------------|------------------------|
| 100 | Discriminator fired | CFD is armed | CFD is armed |
| 101 | Delayed coarse discriminator | Delayed coarse discrim | Delayed coarse discrim |
| 110 | Holdoff time end | When energy sampled | When energy sampled |
| 111 | Peak found | Peak found | Peak found |

Note: "101" mark is a **delayed copy** of coarse discriminator (delayed by M+K0+K in LED mode, M in CFD mode) so it appears near the rising edge rather than far to the left.

**Timing marks with down-sampling**: At down-sample factors > 2–3, timing marks become unreliable. "101" and "110" become indistinguishable. In on-off down-sampling mode, marks in the full-speed region remain valid.

**GammaWare** processes and displays timing marks overlaid on waveforms. Use overlaid buffer-width rectangles to understand the interplay between marks and delay buffer sizes.

---

## Energy Summation

Post-Rise – Pre-Rise subtraction subject to pole-zero effects, baseline drift, decay tails. Corrections done in software analysis; sum values in header sufficient for HPGe resolution after pole-zero/baseline correction.

**P2 buffer** (added Aug 2021 firmware): Accumulator identical to Pre/Post-Rise, sum reported in header. Extends integration time when no pileup.

**Two P2 modes:**
- **SEPARATE mode**: Post-Rise length = Pre-Rise length = m. P2 length set by P2 parameter. Use when P2 needed as delay, or for simplest energy calc.
- **SPAN mode**: Pre-Rise = m, Post-Rise + P2 = m (so Post-Rise = m – P2). Use when P2 not needed as delay, or at high rates. Can sum P2 + Post-Rise for full energy, or just Post-Rise for partial pileup recovery. Pileup timing always defined by m.

**Multiple Pre-Rise sums** (early pre-rise): Coarse discriminator captures earlier copies of pre-rise sum before main discriminator fires. Positions shown in Figure 21. CAPTURE_PARST_TS bit in channel control register: when 0 = normal; when 1 = second early pre-rise sample replaced by timestamp of last Preamp Reset.

---

## Data Readout

### Pileup Rejected (simple case)

Channel Readout Machine monitors PEQ and timestamp. T1 + T2 buffers each 1024 samples (20.48 µs at 100 MHz). Readout aligned to discriminator firing. ✅ verified 2026-04-11 — `Channel_Readout_Mach.vhd:L137` (11-bit `next_event_waveform_length`, max 1024 set by physical T1+T2 buffer size)

User controls:
- **Waveform offset**: samples before discriminator to start readout
- **Waveform length**: number of samples to read (LSB ignored → always even)

**Internal accept all + pileup rejected**: PEQ bypassed; every accepted hit → accepted event immediately. Data transferred from channel pipeline to Accepted Event FIFO as it exits T buffers.

**Readout Interference**: If waveform offset + length > pileup inspection time (M+K0+K), event n may still be reading when event n+1 is ready. Result: event n+1's waveform included in event n's data, but event n+1's header is lost. Safe condition: ReadoutLength + ReadoutOffset < M+K0+K+30.

### Down-Sampled Readout

Down-sampling factors 0–7 → 2^N samples averaged per data point (1 to 128 ADC samples/point). Running accumulator, shifted right N bits. Data packed as two 16-bit halves per 32-bit word; bit 14 marks transition between down-sampled and normal-speed.

**On-off mode**: Starts down-sampled, pauses at coarse discriminator timing mark for N words (full speed around the event), then resumes down-sampling.

**Readout Interference risk is much higher with down-sampling** because internal processing still runs at 100 MHz regardless.

### Pileup Readout (Pileup Extension)

**Pileup Extension Disabled**: Only Accepted Hits that became Accepted Events are read out. Extended Events (2nd+ hits in pileup train) discarded.

**Pileup Extension Enabled**: All Accepted Hits (Accepted Events) + all their associated Extended Events are read out.

**Four readout modes** (combine with Pileup Extension → 8 total behaviors): No Offset, Offset, Offset with Truncation, Header Only.

### Channel Interactions

When event selected for readout: Event Header FIFO header + ADC samples from Channel FIFO → board-wide **Accepted Event FIFO**. This FIFO is drained to external FIFO memory. 10 channels share one Accepted Event FIFO; dropped events if channels contend and FIFO fills.

---

## Trigger Window Calculation

On trigger acceptance: for each PEQ entry, compute: `PEQ_timestamp – trigger_timestamp`. Normally negative (discriminator fires before trigger). Compare against upper and lower window registers; both must pass.

Window registers should be **negative**, set as:
- min_window = –(m+k+1.4 µs) (earliest → farthest from trigger)
- max_window = –(m+k) (latest → closest to trigger)

For Gammasphere locally-generated multiplicity triggers: trigger formation delay ~0.7 µs (digitizer→router→MTRG round trip). Transmission latency variable (~0.8–2.8 µs) but ignored since timestamp from message is used, not arrival time.

Timing test results: window width of 50–100 ns achieves 100% acceptance with synchronous test pulses; 120 ns sufficient with real array signals.

---

## Data Format

Two modes: Leading-Edge Discriminator (type 7) and CFD (type 8). Data = header + waveform, all 32-bit VME words. Header length field + packet length field in each header.

Prior header types: 1&2 (pre-May 2015), 3&4, 5&6, 7&8 (Aug 2021).

### Header Flag Bits

| Flag | Meaning |
|------|---------|
| TTS | TRIG_TS_MODE: 0=trigger msg arrival time, 1=trigger msg timestamp |
| PBYP | 0=TTCL mode, 1=PEQ bypassed (Internal Accept All) |
| PF | Pileup Flag: event was piled up |
| PO | Pileup Waveform Only: only pileup waveforms allowed |
| GE | General Error: firmware should be reloaded |
| SE | Sync Error: timestamp synchronization error |
| OF | Offset Flag: Extended Event with offset readout due to readout interference |
| PV | Peak Valid: peak found before holdoff expired |
| ED | External Discriminator Flag: event caused by external discriminator |
| VF | Veto Flag: would have been vetoed if router vetoes enabled |
| WF | Write Flags: 0=14-bit ADC with flag bits, 1=14.2 format no flags |
| PTE | Pileup Time Error: illegal holdoff/pileup combination; must reset |
| P2M | P2 Buffer Mode: 0=P2 set by reg_p_window, 1=P2+Post set by M |
| CF | Coarse Fired: coarse discriminator fired for this event |
| PTSM | Preamp Reset TS Match: bits 47:28 of preamp reset TS match current |
| 2DF | 2nd threshold discriminator crossed before holdoff elapsed |
| CPTS | Multiplex field mode: 0=2nd early pre-rise sum, 1=preamp reset timestamp |
| PCV | Previous CFD Valid (carry-forward from previous firing) |
| CEM | CFD only: 0=capture energy at CFD fire, 1=capture at delayed LED |
| CV | CFD Valid Flag: 0=invalid (timestamp is LED timestamp), 1=CFD OK |
| TSM | Time Stamp Match: bits 37:30 of prev CFD match bits 47:30 of this event |

### Sum Fields (Header Data)

- **Sampled Baseline**: Sum across T1 buffer (10.24 µs) at discriminator fire time
- **Post-Rise Sum**: Sum across post-rise M buffer at discriminator fire
- **Pre-Rise Sum**: Sum across pre-rise M buffer at discriminator fire
- **P2 Sum**: Sum across P2 buffer at discriminator fire
- **Early Pre-Rise Sum**: Sum across pre-rise M buffer at coarse discriminator fire
- **Multiplex Data Field**: If CPTS=0 → 2nd early pre-rise sum; if CPTS=1 → bits 27:4 of preamp reset timestamp

### Timestamp Fields

- **Timestamp of Discriminator**: 48-bit timestamp of main discriminator fire (the event timestamp)
- **Timestamp of Previous Discriminator**: Last main discriminator timestamp before this event (for pole-zero/baseline correction). May not have been selected for readout.
- **Timestamp of Peak**: Lower 16 bits of peak detector timestamp. Invalid if PV=0.
- **Timestamp of Coarse**: Lower 10 bits of coarse discriminator timestamp (for early pre-rise timing)
- **Multiplex Data Field** (if CPTS=1): Bits 27:4 of last preamp reset timestamp. 24-bit value → 2.68 s span at 160 ns resolution. PTSM bit ensures bits 47:28 also match.
- **Trigger Timestamp Data**: Lower 16 bits of trigger-associated timestamp (TTS=0 → arrival time, TTS=1 → timestamp from message; TTS=1 is normal)
- **Detector Data From Trigger**: Sweep_RAM/Trig_RAM/Veto_RAM lookup table addresses + current Sweep_RAM address (usually target wheel rotational position)
- **Extra Data From Trigger** (LED mode only, not CFD): 8-bit multiplicity totals from trigger virtual planes X and Y (typically "clean" and "dirty" sums for Gammasphere); used to sort events by multiplicity

### CFD Header Differences (Type 8 vs Type 7)

1. Three CFD Sample values added (signed, from CFD subtraction equation):
   - CFD Sample 0: sample at sign crossing
   - CFD Sample 1: 10 ns before Sample 0
   - CFD Sample 2: 20 ns before Sample 0
2. Extra Data From Trigger sacrificed to make room
3. Timestamp of Previous Discriminator cut from 48-bit to 30-bit
4. PCV, CEM, CV, TSM bits have meaning (always 0 in LED mode)

### Waveform Data Format

32-bit word = two 16-bit halves. ADC data = 14-bit unsigned offset binary (0 = most negative, 0x3FFF = most positive). Bit 14 = timing mark, bit 15 = down-sampling marker.
- Bits 13:0 = earlier sample
- Bits 29:16 = sample captured one 10ns tick later

---

## Energy Summation Logic — Operational Details

Accumulators initialized at LOAD_VALS: filled with Assumed Baseline × buffer_length. Every 10ns: subtract eldest, add newest. After buffer_length ticks, accumulator tracks valid data. When discriminator fires, running sum is latched.

All ADC values are 14-bit unsigned. Promoted to 24-bit in VHDL for accumulation (max buffer = 1024 samples).

**Naming note**: M1/M2 in firmware = K1/K2 in Jordanov & Knoll (1994). Delay between integration buffers = sum of k0+k+d+d2+d3 (firmware) = "m" in J&K. Integration time = "m" (firmware) = "k" in J&K.

Firmware samples both Post-Rise and Pre-Rise accumulators simultaneously at discriminator fire. The user's analysis software computes energy from these two partial sums plus pole-zero correction. Carrying forward previous hit's Post-Rise sum enables baseline + pole-zero correction for quickly-following pulses.

---

## Discriminator Filter Details

Four-pass implementation: Y(n) = X(n) + 8·X(n-1) + 28·X(n-2) + 56·X(n-3) + 70·X(n-4) + 56·X(n-5) + 28·X(n-6) + 8·X(n-7) + X(n-8). Implemented with adders + hardware multipliers. Coefficients sum to 256 → lower 8 bits discarded. Attenuates >20 MHz (at 100 MHz clock).

**Recommended holdoff**: 10–20% longer than detector rise time, with early peak termination enabled. Too-short holdoff causes multiple firings per rise.

**Preamp Reset Kill (detailed)**: Any signal with difference >512 ADC counts across LED delay buffer, in opposite polarity to discriminator polarity setting, triggers PRK. Disables discriminator for 1–255 × max holdoff time (register controlled). PRK only works in positive-only or negative-only polarity mode.

---

## CFD Mode — Detailed Settings

**CFD fraction**: 50% is typical; 10–90% can work; practically 40–70% most reliable.

**Setup procedure:**
1. Set CFD fraction (50% common)
2. Set 'd' to ~10 samples as starting point (never < 10)
3. Set 'k' ≥ 16 samples for reliable LED operation
4. Take sample events; adjust 'd3' to align pre-rise sum (use GammaWare buffer overlay)
5. Adjust 'k0' for post-rise sum alignment
6. Iterate 'k' and 'd' to optimize
7. Smaller 'd' → better time-walk but smaller usable CFD fraction range

Note: In CFD mode, discriminator fires at **left edge of 'd'** (vs. right edge in LED mode).

---

## Trigger Delay Buffer

Fixed-length 20.48 µs delay buffer. Discriminator bit reported to trigger is delayed by pileup inspection time (m+k) after discriminator fires. Purpose: allow external trigger system time to collect multiplicity across many channels and make accept/reject decision.

**Total delay from edge entering digitizer to trigger report**: approximately 2×(m+k). For typical Gammasphere settings (~6 µs m) this is >12 µs total.

**Coarse discriminator**: fast path for non-buffered auxiliary detectors; avoids the full pileup delay.

**Dead time**: Only the discriminator holdoff time. Demonstrated >100 kHz in field, 400 kHz in lab, approaching 1 MHz achievable.

---

## Waveform Readout — Details

**Channel-level FIFO**: Each channel fills its own FIFO. Board-wide state machine round-robins across all 10 channels, transferring complete events to external FIFO. Only whole events transferred. One event per channel per round-robin pass. Round-robin prevents one busy channel from blocking others, but a fast channel can fill its FIFO between visits.

**VME bandwidth (typical embedded processor)**: ~10 MB/s after overhead. With 4 digitizers in crate: 2.5 MB/s per digitizer.
- Full waveform (1024 samples): ~2100 bytes. At 2.5 MB/s → 840 µs/channel/event. Max round-robin cycle = 8.4 ms. With double-buffering (post-May 2014 firmware) → ~2.4 kHz at 10% channel occupancy with full waveform.
- 100-sample waveform → ~24 kHz practical rate.

**Down-sampling (`downsample_factor` PV):** Factor 0–7 → 1×/2×/4×/8×/16×/32×/64×/128×. At factor 7: 1024-sample readout spans 1.31 ms. Down-sample factor also applies to waveform offset (offset time covered = full-speed_offset / 2^N).

**Implementation (`decimator.vhd`, M. Oberling):**
- True **block-averaging decimation** — accumulates N consecutive 14-bit ADC samples and outputs a 16-bit average (wider to preserve precision)
- **Not a moving average** — non-overlapping, non-sliding blocks
- Alignment: `dec_enable` driven by `pending_read_event` in `event_packer.vhd` — averaging starts synchronously with the readout window, aligned to trigger via `readout_pretrigger`
- Block boundaries are fixed relative to window start; choose `readout_pretrigger` so the rising edge falls at a block boundary to avoid blending rise + decay in one block

**`dec_pause` / On-Off down-sampling (added 2016, `enable_dec_pause` PV):** Starts down-sampled, switches to full-rate at the coarse discriminator timing mark, stays full-rate for a configurable holdoff, then returns to down-sampled. Transitions synchronized to down-sampled block boundaries. This enables:
- Rising edge at full 100 MHz (precise timing + shape)
- Exponential tail at decimated rate (e.g. 8× = 80 ns/sample, 8 points covers 640 ns) for offline PZ correction

**Use for offline pole-zero calibration:** A decimated trace at 8× provides enough tail samples to fit $k$ per crystal and validate against `GeCenterTimeConstant`. See `pole_zero.md` Level 4 for the proposed algorithm.

---

## Readout Modes (8 Combinations)

Two controls: **Pileup Extension Enable** (0/1) × **Readout Mode** (4 options) = 8 behaviors.
_Register: `reg_channel_control[25:24]` = event_extend_mode (2-bit); Pileup Extension Enable = separate bit. ✅ verified 2026-04-07 — `Digitizer.vhd:L1323`, `Channel_Readout_Mach.vhd:L37`_

### Pileup Extension DISABLED (only Accepted Events read out, Extended Events always discarded)

| Mode | Behavior on Readout Interference |
|------|-----------------------------------|
| 00 No Offset | Later event is dropped |
| 01 Offset | Later event's readout delayed until previous finishes; full-length packet; may drop if offset exceeds Accepted Event FIFO size |
| 10 Offset w/ Truncation | Later event starts immediately after first finishes; packet length shortened (variable) |
| 11 Header Only | Overlapping event reduced to header only; no waveform data |

**Down-sampling + No Offset**: Strongly discouraged — many events end up headerless. Use Offset with Truncation when down-sampling is active.

### Pileup Extension ENABLED (Accepted + Extended Events read out)

| Mode | Behavior |
|------|----------|
| 00 No Offset | Discards any event (accepted or extended) if full readout doesn't fit before next; **discouraged** (may skip accepted while passing extended) |
| 01 Offset | Delays each event until previous finishes; full-length packets; drops if offset exceeds Accepted Event FIFO |
| 10 Offset w/ Truncation | Delays + shortens waveform to non-overlapping region only; variable length; **best mode for pileup waveforms** |
| 11 Header Only | Collapses interfering events to header-only; non-interfering events read in full |

---

## Trigger Interface

**SERDES link**: Similar to GRETINA between digitizer and trigger, but different data format. Digitizer receives control stream from trigger, synchronizes to trigger clock and timestamp.

**Timestamp Synchronization**: Uses 'stringent lock' like RTRG. Non-imperative Sync commands compared against local timestamp; single mismatch ignored; 3 consecutive mismatches → status bit set. Imperative Sync commands reset the timestamp.

**Data sent to trigger** (every 20 ns): Snapshot of all 10 channels' discriminator bits + coarse discriminator bits from channels 5–9. In DGS: ch 5–9 = Ge center contacts, ch 0–4 = BGO sums. Coarse bits enable fast multiplicity pre-triggers system-wide. Discriminator bit assertion time is programmable.

**No energy data sent to trigger**: DGS/DFMA only use multiplicity-based triggers.

**Event Veto**: Enabled per-channel by register. When trigger asserts Veto bit, most recent PEQ entry erased. In DGS: Router boards independently calculate Veto bits (Ge/BGO pairs, clean/dirty/module modes) and insert into frame 5 without MTRG intervention. Vetoing an accepted event automatically suppresses all associated extended events.

**Throttle**: LVDS bit from digitizer to Router; asserted when board-wide FIFO reaches half-full. MTRG can suspend trigger accepts while any digitizer throttles.

**Trigger Decision Latency**: Variable due to trigger algorithm time (Ttf) + 0–2 µs TTCL frame alignment + potential FIFO backup at high rates.

**Acceptance window settings** (Gammasphere multiplicity triggers):
- Discriminator-to-trigger propagation: ~0.75 µs (DIG→RTRG→MTRG)
- Multiplicity trigger formation: ~0.2 µs
- Total: trigger timestamp ≈ (m+k) + 0.95 µs after discriminator firing
- Window: min = –(m+k+1.8 µs), max = –(m+k) [both negative]

### trigger_mux_select — Trigger Mode Selection

PV: `VME<CRATE>:<BOARD>:trigger_mux_select` (mbbo, global fanout via `GLBL:DIG:trigger_mux_select`)

Selects the digitizer's trigger acceptance source. Four modes:

| Value | Name | Description |
|-------|------|-------------|
| 0 | `IntAcptAll` | **Internal Accept All** — PEQ bypassed; every discriminator hit is immediately accepted as an event. Used for LED/pulser calibration runs. No TTCL trigger decision needed. |
| 1 | `ExtTTL` | **External TTL** — accept trigger from a TTL signal injected via the Auxiliary Input Interface on the digitizer front panel. Timestamp = local timestamp at signal application. |
| 2 | `ExtTTCL` | **External TTCL** — normal physics-run mode. Digitizer operates in triggered mode: each hit enters the PEQ and waits for an accept/reject decision from the Master Trigger delivered over the TTCL link (MTRG → RTRG → DIG). Hits outside the acceptance time window are discarded. |
| 3 | `Diag` | **Diagnostic** — auto-trigger on lower 16-bit timestamp rollover (0xFFFF → 0x0000). Used for synchronization diagnostics. |

**Typical usage:**
- Physics runs: `caput GLBL:DIG:trigger_mux_select ExtTTCL`
- LED/calibration runs: `caput GLBL:DIG:trigger_mux_select IntAcptAll`

The `PBYP` flag in the data header reflects this: `PBYP=1` means PEQ bypassed (IntAcptAll), `PBYP=0` means TTCL mode (ExtTTCL).

**Special trigger modes**:
- External Trigger (`ExtTTL`): trigger from test point signal, timestamp = local TS at signal application
- Diagnostic Trigger (`Diag`): auto-trigger on lower 16-bit timestamp rollover (0xFFFF→0x0000)

**Other trigger commands**:
- Frame 13 (Demand Slow Data): no effect in ANL firmware
- Frame 16 (Synchronous System Capture): resets then collects diagnostic counters for programmable duration

---

## Master and Slave Digitizers

Compile-time build option: SLAVE_MODE. Front bus ribbon cable connects master to slave(s).

**Front bus signals:**
- Differential clock (direction set by hardware jumpers, not firmware): Slave uses master's switched 50 MHz clock (from trigger SERDES link)
- 18 bits master→slave (`FBUS_MDATA[17:0]`):
  - Bits 15:0 = direct copy of trigger SERDES data (DC-balanced, clock guard removed) → fed to Slave's SERDES receive state machine ✅ verified 2026-04-19 — `Front_Bus.vhd:L129,L135` (FBUS_MDATA[15:0] ↔ fbus_serdes_data)
  - Bit 16 = copy of **channel 9** discriminator bit from master (external discriminator source for slave channels) — note: was channel 0 before 2014-12-02, changed to channel 9 ✅ verified 2026-04-19 — `Digitizer.vhd:L1387` (`fbus_disc_flag_in => diag_disc_flag(9)` with comment "changed from zero to nine 20141202"); `Front_Bus.vhd:L136`
  - Bit 17 = master reset signal replicated to slave ✅ verified 2026-04-19 — `Front_Bus.vhd:L77,L81` (`FBUS_MDATA(17) <= fbus_reset_in` on master; `fbus_reset_out <= FBUS_MDATA(17)` on slave)
  - Note: Slave P1/P2 delay buffers must be set a few clocks larger than master to compensate for cable delay
- 3 bits slave→master:
  - Slave external FIFO status (master includes this in its Throttle Request to trigger)
  - Slave SERDES lock status
  - One spare
- Wire-Or bit: for multi-slave Throttle Request aggregation (requires correct jumpers/terminations)

**Slave external discriminator sources** differ from master (see Table 1 earlier): slave uses ch9 of master digitizer rather than local ch9.

**Critical**: Slave has no way to send discriminator data back to the trigger. Trigger is blind to Slave operation. Only communication from Slave to trigger is the Throttle Request bit (via Master).

---

## Slave Digitizer Operational Modes

### 1. Independent Mode
All Slave channels use internal discriminators; external discriminator ignored. Slave runs alongside Master with synchronized clocks and receives trigger commands. Each Slave channel only fires on its own signals. Useful for Ge Side + BGO Pattern readout independent of Master. BGO Pattern always fires when Ge Center fires (BGO Pattern logic has its own parallel discriminator).

### 2. Clean/Dirty/Module Mode (with Router Vetoes)
Router trigger monitors Ge Center + BGO Sum discriminator bit pairs from Master. Generates 5 Veto bits (one per channel pair). Veto asserted if both channels fired within programmable time window → removes top PEQ entry from both channels. Router encodes Veto bits into frame 5 word of trigger command stream.

**Critical requirement**: 'm' and 'k' must be identical in Ge Center and BGO Sum channels for Veto timing to work.

**Slave Veto handling**: Slave receives same Veto signals. Ge Side channels process Vetoes; BGO Pattern channels do NOT (preserves full BGO Pattern data for offline scatter pattern reconstruction).

**Protecting PEQ integrity at high rate**: If dirty event fires Ge Center but not Ge Side (drift direction), previous Ge Side event risks spurious Veto. Fix: Set Ge Side channels to OR mode with Ge Center discriminator bit → Ge Center forces Ge Side event. Can carry all 5 Ge Center bits via unused bits of command frame word 5.

**Router multiplicity reporting options**: Raw, clean-only, or lone-BGO multiplicity to MTRG X or Y sums → enables filtered triggers (e.g. clean Ge multiplicity > threshold).

### 3. Pseudo-GRETINA Mode
Slave completely slaved to one channel of Master (ch 9 in current firmware; historically ch 0 before 2014-12-02). Master ch 9 discriminator bit transmitted on **front bus bit 16** (`FBUS_MDATA[16]`). All Slave channels set to External Only mode (external disc source = `fbus_disc_flag`, i.e. `reg_bit_slices = "001"`). All channels in both Master (ch 0–8 also set to External Only → ch 9 via diag_disc_flag) and Slave capture an event every time Master ch 9 fires — identical to GRETINA operation. Same trigger accept messages, same timing windows. Delay chain parameters (m, k, d, d3) must match exactly in all channels of both boards. ✅ verified 2026-04-19 — `Front_Bus.vhd:L77,L136` (FBUS_MDATA[17]=reset, [16]=disc_flag); `Digitizer.vhd:L1387` (disc_flag_in = diag_disc_flag(9)); `Digitizer.vhd:L896` comment "GRETINA master mode"

---

## On-Board DAC Diagnostics

Global DAC MUX selects any of 10 channels to drive the front panel DAC output. Per-channel diagnostic waveform selection:

| Selection | What is output |
|-----------|----------------|
| 00 | Raw ADC data at P2 buffer output (direct input monitoring) |
| 01 | Digital signal showing when baseline tracking logic is active |
| 10 | ADC data entering LED discriminator buffer |
| 11 | Baseline tracker value (BASE_SAMPLE) — view slew rate and filter characteristics |

**Startup check**: Monitor channel baseline with oscilloscope, no beam. Reset baseline logic. Signal should home to estimated baseline register value, then track. Should hold during events, re-track between them.

---

## Auxiliary Input Interface (ExtTTL Injection Point)

The digitizer board has an **Auxiliary Input Interface** — a 16-pin header (0.1" × 0.1" / 2.54 mm pitch) on the front panel, providing **16 general-purpose TTL/RS485-compatible I/O signals** wired directly into the Main FPGA.

- **Standard:** RS485 differential transceivers (high-speed), configurable in pairs as inputs or outputs
- **Configuration:** I/O direction settable in increments of 2 signal pairs (0–11 inputs, 0–11 outputs, various combinations)
- **One dedicated external clock input pair** also present (separate from the 16 I/O signals)

**ExtTTL trigger injection:**
When `trigger_mux_select = ExtTTL`, one of these auxiliary inputs serves as the external trigger. A TTL pulse on this input:
1. Starts the event timers across all channels simultaneously
2. Latches DSP algorithm results (energy, CFD timestamps, etc.) into header memory for all enabled channels
3. Propagates via `FB_LED` line to slave digitizers in the same crate

This is a hardware strobe mode — no TTCL/MTRG involvement. Useful for bench testing, pulser injection, or external-detector-gated readout.

**Full connector pinout (36-pin, 3-column):** *(see also `connectors.md`)*

| Pin | Function | Pin | Function | Pin | Function |
|-----|----------|-----|----------|-----|----------|
| 1 | GND | 2 | Aux0 I/O + | 3 | Aux0 I/O − |
| 4 | GND | 5 | Aux1 I/O + | 6 | Aux1 I/O − |
| 7 | GND | 8 | Aux2 I/O + | 9 | Aux2 I/O − |
| 10 | GND | 11 | Aux3 I/O + | 12 | Aux3 I/O − |
| 13 | GND | 14 | Aux4 I/O + | 15 | Aux4 I/O − |
| 16 | GND | 17 | Aux5 I/O + | 18 | Aux5 I/O − |
| 19 | GND | 20 | Aux6 I/O + | 21 | Aux6 I/O − |
| 22 | GND | 23 | Aux7 I/O + | 24 | Aux7 I/O − |
| 25 | GND | 26 | Aux8 I/O + | 27 | Aux8 I/O − |
| 28 | GND | 29 | Aux9 I/O + | 30 | Aux9 I/O − |
| 31 | GND | 32 | Aux10 I/O + | 33 | Aux10 I/O − |
| 34 | GND | 35 | Ext Clock In + | 36 | Ext Clock In − |

**Key signal: AUX_DIN[10] = Aux10 (pins 32/33)**
- This is the MSbit of the 11-bit Auxiliary I/O (`AUX_DIN[10:0]`)
- Used as the **external discriminator input** from the front panel
- Referenced in firmware as "Auxiliary I/O bit 10 on the front panel"
- When a channel's external discriminator source is set to "front panel" (option 3 in the source matrix), it reads `AUX_DIN[10]`
- ⚠️ If the Aux I/O is configured such that bit 10 is an **output**, it cannot be used as an external discriminator input

---

## Glossary

- **Digitizer**: Measures analog waveforms at fixed sampling rate; identifies events from waveform features
- **Trigger**: Collects data from multiple digitizers + detectors + other systems; real-time selects which events are read out
- **Discriminator**: Digital signal when amplitude change exceeds threshold
  - **Leading-Edge (LED)**: Fires when slope (dV/dt) exceeds threshold (differential between two filtered samples)
  - **Constant-Fraction (CFD)**: Fires at fixed fraction of total amplitude regardless of amplitude; better timing than LED when properly tuned
- **Hit**: Digital output of a discriminator
- **Event**: Data packet initiated by a hit, after pileup/trigger processing to determine fitness for readout
- **Rise Time**: Time/samples from start of discriminator-triggering amplitude change to end of monotonic rise. Usually expressed as 0–100% or 10–90% of final amplitude (not exponential time constant).
- **Decay Time**: Exponential decay time constant (e.g. 50 µs). Waveforms = dual exponential: X(t) = A·[1-exp((t0-t)/tr)]·exp((t0-t)/tf). Rise time tr ~1 µs; decay time tf much longer.
- **Pipeline**: Logic where data is processed every clock tick continuously; never stops. Accumulators: zero init → add new data for buffer_length ticks → switch to tracking (add newest, subtract eldest every tick).
- **Down-sampling**: Replace n samples with their average (n must be power of 2). Output one sample every n clocks. Allows fixed data volume to span longer time, at cost of longer readout time.

---

_Document complete. PDF: 72 pages, SVN rev #6185, Sept 2021. Written to file: 2026-04-05._

## See Also

- `knowledgeBase/deep_fpga_DIG.md` — DIG firmware architecture overview: target devices, memory, source file list, build branches
- `knowledgeBase/deep_fpga_DIG_channel.md` — Per-channel signal processing VHDL deep-dive: LED/CFD discriminator, delay chain, pileup logic
- `knowledgeBase/deep_fpga_DIG_eventpacket.md` — DIG event packet format: LED/CFD header layout, field reconstruction, waveform samples
- `knowledgeBase/preamp_reset_readme.md` — Detailed preamp reset (PRK) explanation: detection logic, blanking, BGO gate, PARST timestamp
- `knowledgeBase/data_structures.md` — Complete DIG event packet format (all header types, GEB header, TAC-II)
- `knowledgeBase/connectors.md` — DIG connector pinouts (36-pin Aux I/O, RJ45, front bus)

---

*Created: 2026-04-05 | Last reviewed: 2026-04-19*

