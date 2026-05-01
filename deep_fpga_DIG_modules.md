# DIG Firmware — Selected Module Analysis (Part 1: Signal Chain & SERDES)

Stability: C3 - Structural / stable

_Analysis of DIG production VHDL modules: signal chain, discriminator packing, SERDES, timestamp, trigger mux, and per-channel readout FSMs._  
_For DC balance, waveform FIFOs, decimator, event header, channel collection FIFO, VME interface, and register map → see [deep_fpga_DIG_modules2.md](deep_fpga_DIG_modules2.md)._  
_Source tag: `MAIN_FPGA_TAGS/20180507/MAIN_FPGA/BuildBranches/DGS/Source/`_  
_Authors: John Anderson (JTA), Michael Oberling (MBO)_

## Table of Contents

- [SERDES_TX_Mach_DGS.vhd — Discriminator Hit Packer](#serdes_tx_mach_dgsvhd--discriminator-hit-packer)
- [event_packer.vhd — Accepted Event FIFO Writer](#event_packervhd--accepted-event-fifo-writer)
- [pileup_processor.vhd — Pileup Detection State Machine](#pileup_processorvhd--pileup-detection-state-machine)
- [SERDES_RX_Mach.vhd — Router Command Frame Receiver](#serdes_rx_machvhd--router-command-frame-receiver)
- [Timestamp_Generator.vhd — 48-bit Timestamp Synchronization](#timestamp_generatorvhd--48-bit-timestamp-synchronization)
- [Trigger_Mux.vhd — Trigger Source Selector](#trigger_muxvhd--trigger-source-selector)
- [Channel_Readout_Controller.vhd — Readout Timing & Decision Logic](#channel_readout_controllervhd--readout-timing--decision-logic)
- [Channel_Readout_Mach.vhd — Per-Channel Readout Wrapper](#channel_readout_machvhd--per-channel-readout-wrapper)

---

## SERDES_TX_Mach_DGS.vhd — Discriminator Hit Packer

**Entity:** `SERDES_TX_Mach_DGS`  
**Author:** Michael Oberling  
**Lines:** 94  
**Clock domains:** CLK100 (stretch logic) + CLK50 (SERDES output latch)

### Role

Packs the 10-channel discriminator hit bits and coarse discriminator flags into a single 16-bit SERDES word for transmission to the Router FPGA. Also stretches discriminator pulses to ensure the Router can sample them reliably at 50 MHz.

### Ports

| Port | Dir | Width | Description |
|------|-----|-------|-------------|
| `CLK50` | in | 1 | SERDES logic clock (transmit clock domain) |
| `CLK100` | in | 1 | 100 MHz pipeline clock (discriminator domain) |
| `RESET` | in | 1 | Global active-high reset |
| `reg_disc_width` | in | 10 × 32-bit | Per-channel discriminator stretch width (`Width = reg + 1` in 100 MHz ticks) |
| `COARSE_DISC_FLAGS` | in | 10-bit | Coarse discriminator flags from all channels |
| `DISC_BITS` | in | 10-bit | Discriminator hit bits (1 per channel, 100 MHz) |
| `PILEUP_BITS` | in | 10-bit | Pileup flags (1 per channel) — used internally for stretch, not transmitted |
| `SERDES_SYNC_FLAG` | in | 1 | Sync flag from timestamp synchronization logic |
| `STRETCHED_DISCBITS` | out | 10-bit | Stretched discriminator bits (feedback to jta_channel for other uses) |
| `TX_DATA_OUT` | out | 16-bit | Packed SERDES word to send to Router |

### TX Word Format

```
TX_DATA_OUT[15:0]:
  [15]    = SERDES_SYNC_FLAG
  [14:10] = COARSE_DISC_FLAGS[9:5]   (upper 5 channels only)
  [9:0]   = DISC_ONE_SHOT[9:0]       (stretched discriminator bits, all 10 channels)
```
✅ verified 2026-04-24 — SERDES_TX_Mach_DGS.vhd:L90-92 (LATCH_DATA process: TX_DATA_OUT(15)<=SERDES_SYNC_FLAG; TX_DATA_OUT(14:10)<=COARSE_DISC_FLAGS(9:5); TX_DATA_OUT(9:0)<=DISC_ONE_SHOT)

**Note:** Only `COARSE_DISC_FLAGS[9:5]` (channels 5–9) appear in the TX word. Channels 0–4 coarse disc flags are not transmitted in this format — they are OR'd or handled via the front bus mechanism.

### Discriminator Stretching (CLK100 domain)

Each channel runs an independent 7-bit countdown stretch:

1. When `DISC_BITS(i) = '1'`: reload `DISC_COUNT(i)` with `reg_disc_width(i)[5:0] & '1'` — minimum stretch = 2 × 100 MHz ticks (20 ns).
2. While `DISC_COUNT(i) ≠ 0`: decrement counter, hold `DISC_ONE_SHOT(i) = '1'`.
3. When counter reaches 0: deassert `DISC_ONE_SHOT(i)`.

**Maximum stretch width:** 7-bit counter → max 127 × 10 ns = 1.27 µs (limited by `reg_disc_width` 6-bit field + appended '1').

### Pileup ANY flag (CLK100 domain)

`PILEUP_BITS` are OR-reduced into `ANY_PILEUP` with a 2-stage pipeline:
- `xANY_PILEUP <= '1'` immediately when any pileup bit fires; `ANY_PILEUP <= xANY_PILEUP` (1-clock latency).
- This creates a 1-clock-extended pulse. **`ANY_PILEUP` is not currently driven to any output** — it is a dead internal signal in the DGS branch (the port `PILEUP_BITS` is used in stretch logic only conceptually — the pileup stretch doesn't actually gate the TX word).

### Output Latch (CLK50 domain)

`LATCH_DATA` process clocks on CLK50:
- Samples `DISC_ONE_SHOT` (stretched at 100 MHz) and re-registers at 50 MHz — this is the CDC crossing point. Because stretch pulses are held for at least 2 × CLK100 cycles, any 50 MHz sample will capture them.
- Simultaneously latches `COARSE_DISC_FLAGS[9:5]` and `SERDES_SYNC_FLAG`.

---

## event_packer.vhd — Accepted Event FIFO Writer

**Entity:** `event_packer`  
**Author:** Michael Oberling (created 2014-06-20)  
**Lines:** 395  
**Clock domain:** Single clock (CLK — 100 MHz in context)

### Role

Controls the "accordion" FIFO mechanism: injects event headers from the `Channel_Readout_Controller` and then transfers waveform sample pairs from the ADC data FIFO into the `acptd_event_fifo`. Also manages the decimator enable/pause signals to implement on-the-fly waveform downsampling with timing-mark-driven pause logic.

### Ports

| Port | Dir | Width | Description |
|------|-----|-------|-------------|
| `clk` | in | 1 | Clock (100 MHz) |
| `reset_fifo` | in | 1 | Synchronous FIFO reset |
| `acptd_event_fifo_wr_en` | out | 1 | Write enable to accepted event FIFO |
| `acptd_event_fifo_din` | out | 36-bit | Data word to accepted event FIFO |
| `timing_mark_in` | in | 2-bit | Timing mark flags from jta_channel pipeline (added 2016-03-03) |
| `event_data` | in | 2 × 16-bit | Two consecutive ADC waveform samples |
| `event_data_almost_empty` | in | 1 | ADC data FIFO almost-empty flag |
| `event_data_empty` | in | 1 | ADC data FIFO empty flag |
| `event_data_read_en` | out | 1 | Read enable to ADC data FIFO |
| `dec_enable` | out | 1 | Enable decimator |
| `dec_pause` | out | 1 | Pause decimation (added 2016-03-04) |
| `reg_decimate_holdoff` | in | 8-bit | Decimation holdoff counter reload value (addr 0x434, added 2016-03-06 by JTA) |
| `next_event_header_word` | in | 32-bit | Next header word from Channel_Readout_Controller |
| `next_event_header_stop` | in | 1 | Signals end of header word stream |
| `next_event_waveform_length` | in | 11-bit | Waveform word count from readout timing logic |
| `read_event` | in | 1 | High when next event should be read out |
| `pending_read_event` | in | 1 | High when an event is pending readout |
| `readout_mach_ready` | out | 1 | '1' when state machine is ready for next event |

### State Machine (6 states)

```
WAIT_FOR_ACCEPTED_EVENT
        │  read_event='1'
        ▼
   LOAD_HEADER  ──────────────────────────────────────────────────┐
        │  next_event_header_stop='1'                             │ (loop: header words)
        │                                                         │
        ├── waveform_length=0 → write end-of-packet, return to WAIT
        │
        ▼ (waveform to transfer)
  ST_XFER_PAUSED  ◄─────────────────────────────────────────────────┐
        │  event_data_empty='0' AND waveform_count ≠ 0              │
        ▼                                                            │
  ST_RESUME_XFER                                                     │ (back-pressure)
        │                                                            │
        ▼                                                            │
  ST_XFER_WAVEFORM ─────────────────────────────────────────────────┘
        │  waveform_count=0 → write end-of-packet, return to WAIT
        │  event_data_almost_empty → ST_XFER_PAUSED
        │
        ▼ (event_data_empty='0' when count=0)
  ST_XFER_LAST_WORD → WAIT_FOR_ACCEPTED_EVENT
```

### FIFO Word Format (36-bit)

```
acptd_event_fifo_din[35:0]:
  [35:32] = control nibble:
              "0000" = normal header/waveform word
              "0001" = end-of-packet marker
✅ verified 2026-04-24 — event_packer.vhd:L165 ("0001" -- Mark the end of packet), L239, L380 (all end-of-packet writes use "0001"; normal words use "0000" at L169/174/241)
  [31:0]  = data:
              During LOAD_HEADER: next_event_header_word[31:0]
              During ST_XFER_WAVEFORM / ST_RESUME_XFER:
                event_data(0)[15:0] & event_data(1)[15:0]
                (two consecutive 16-bit ADC samples packed into 32 bits)
```

### On-the-Fly Decimation Pause Logic

During waveform transfer (`ST_XFER_WAVEFORM`, `ST_XFER_PAUSED`, `ST_RESUME_XFER`), a 4-bit shift register `timing_mark_pipe` detects timing mark flags embedded in the waveform data pipeline:

| Flag constant | Value | Meaning |
|---------------|-------|---------|
| `PAUSE_START_FLAG` | `"0100"` | Start decimation pause (coarse discriminator fired) |
| `PAUSE_STOP_FLAG` | `"0111"` | Stop pause early (peak flag mark — DGS builds only) |
| `DISABLE_CHECK_FLAG` | `"0101"` | Disable flag recognition temporarily (prevents false "0100" detection when "0101" shifts) |

**DGS-specific behavior (added 2017-01-19):**
- `PAUSE_START_FLAG = "0100"` → assert `int_dec_pause`, reload `dec_pause_count` from `reg_decimate_holdoff`.
- Pause persists until `dec_pause_count` reaches zero OR `PAUSE_STOP_FLAG = "0111"` is seen.
- The `DISABLE_CHECK_FLAG` guard: when `"0101"` is seen, disable flag recognition for `FLAG_DISABLE_TIME = 4` clocks, preventing the two-clock-shifted value `"0100"` from falsely triggering PAUSE_START.
- Pause ensures the rise time of the detector pulse is always transmitted at full sample rate (decimation is off during the region of interest around the pulse peak).

### Reset Behavior

On `reset_fifo`:
- `readout_mach_ready` is held **high** — critical design decision: this allows the `Event_Header_FIFO` and event decision FIFO in `Channel_Readout_Controller` to continue draining during reset, preventing FIFO overflow and the general error flag. (MBO comment.)
- `event_data_read_en <= '1'` — keep draining ADC data FIFO.
- All write enables deasserted.

---

## pileup_processor.vhd — Pileup Detection State Machine

**Entity:** `pileup_processor`  
**Authors:** John Anderson (original), Michael Oberling (2014-09-02 refactor)  
**Lines:** 359 ✅ verified 2026-04-25 — `pileup_processor.vhd` (line count confirmed) (including ~100 lines of commented-out PU_ONLY mode code)  
**Clock domain:** CLK (100 MHz)

### Role

Implements the pileup detection and rejection/acceptance logic for a single digitizer channel. Receives the discriminator firing flag (`THRESH_DISC_FLAG`) and the delayed pileup-release signal (`PILE_RELEASE_DLYD`), maintains a 4-bit pileup counter, and outputs `ACCEPTED_HIT` / `EXTENDED_EVENT` / `PILEUP_FLAG` for the readout machine.

### Ports

| Port | Dir | Width | Description |
|------|-----|-------|-------------|
| `CLK` | in | 1 | 100 MHz clock |
| `SUBSECTION_RESET` | in | 1 | Active-high channel reset |
| `PU_TOO_SHORT` | in | 1 | Block ACCEPTED_HIT and EXTENDED_EVENT if set |
| `PILEUP_DISABLE` | in | 1 | When '1': enter pileup-reject path; when '0': enter pileup-accept path |
| `THRESH_DISC_FLAG` | in | 1 | Discriminator firing pulse |
| `PILE_RELEASE_DLYD` | in | 1 | Delayed pileup release (3-clock delayed from `DISCBIT_PILEUP_RELEASE`) |
| `PILEUP_FLAG` | out | 1 | '1' when in a pileup state |
| `ACCEPTED_HIT` | out | 1 | Clean discriminator hit (no pileup) |
| `EXTENDED_EVENT` | out | 1 | Pileup extension hit (second+ hit in a pileup train) |
| `OVERFLOW_FLAG` | out | 1 | Pileup counter overflow/underflow — fatal error |
| `PEHQ_RD_ADDR_ADV` | out | 1 | Advance PEHQ read pointer (driven directly from `PILE_RELEASE_DLYD`) |
| `DIAG_PILEUP_COUNT` | out | 4-bit | Diagnostic: current pileup counter value |

### Pileup Counter (`pile_counter` process)

4-bit saturating counter (increments on new `THRESH_DISC_FLAG` with no simultaneous release, decrements on `PILE_RELEASE_DLYD` with no simultaneous new hit):
- Overflow: if count reaches `"1111"` and another hit arrives → `INT_OVERFLOW_FLAG = '1'` (fatal, machine locks in OVERFLOW state until channel reset).
- Underflow: if count is `"0000"` and a release arrives → also sets `INT_OVERFLOW_FLAG = '1'`.
✅ verified 2026-04-25 — `pileup_processor.vhd:L76-91` (4-bit PILEUP_COUNT; "1111" overflow L84-85; "0000" underflow L90-91)

**Note (from code comment):** The 3-clock delay on `PILE_RELEASE_DLYD` vs `DISCBIT_PILEUP_RELEASE` was introduced in 2012-11-28 to accommodate PEHQ instantiation timing. ✅ verified 2026-04-25 — `pileup_processor.vhd:L70-72` ("20121128: instantiation of PEHQ requires that pileup process be delayed by, we think, three clock ticks")

### State Machine (`pile_proc` process)

8-state FSM with two parallel halves — **pileup reject** and **pileup accept** — selected by `PILEUP_DISABLE`:
✅ verified 2026-04-25 — `pileup_processor.vhd:L49` (`type PILEUP_STATES is (CHECK_DISABLE, REJ_NO_HIT, REJ_ONE_HIT, REJ_MANY_HIT, OVERFLOW, ACC_NO_HIT, ACC_ONE_HIT, ACC_MANY_HIT)`)

```
CHECK_DISABLE ──► REJ_NO_HIT ──► REJ_ONE_HIT ──► REJ_MANY_HIT ──► OVERFLOW
     │                                                                   ▲
     └──────────► ACC_NO_HIT ──► ACC_ONE_HIT ──► ACC_MANY_HIT ──────────┘
```

**Cross-machine transitions:** Both halves monitor `PILEUP_DISABLE` and can switch to the other half on any clock cycle (machine is not locked to one half). ✅ verified 2026-04-25 — `pileup_processor.vhd:L134-247` (REJ_NO_HIT checks `PILEUP_DISABLE='0'` to cross to ACC; ACC_NO_HIT checks `PILEUP_DISABLE='1'` to cross to REJ)

#### Pileup Reject Half (PILEUP_DISABLE = '1')

| State | Condition | Action |
|-------|-----------|--------|
| `REJ_NO_HIT` | THRESH_DISC_FLAG fires | → `REJ_ONE_HIT` |
| `REJ_ONE_HIT` | Another THRESH_DISC_FLAG | → `REJ_MANY_HIT`; suppress hit |
| `REJ_ONE_HIT` | PILE_RELEASE_DLYD | → `REJ_NO_HIT`; assert `ACCEPTED_HIT` |
| `REJ_MANY_HIT` | Counter → 1 and release | → `REJ_NO_HIT` (exit "jail") |
| `REJ_MANY_HIT` | Any time | `PILEUP_FLAG='1'`, suppress all hits |

**Code comment (known bug):** "Pileup flag is not set for first event in pileup train." This is intentional/accepted; the flag is only set once the second discriminator fires. ✅ verified 2026-04-25 — `pileup_processor.vhd:L97` (`-- BUGUBG: Pileup flag is not set for first event in pileup train.`)

#### Pileup Accept Half (PILEUP_DISABLE = '0')

| State | Condition | Action |
|-------|-----------|--------|
| `ACC_NO_HIT` | THRESH_DISC_FLAG | → `ACC_ONE_HIT` |
| `ACC_ONE_HIT` | PILE_RELEASE_DLYD | → `ACC_NO_HIT`; assert `ACCEPTED_HIT` |
| `ACC_ONE_HIT` | Another THRESH_DISC_FLAG | → `ACC_MANY_HIT`; `PILEUP_FLAG='1'` |
| `ACC_MANY_HIT` | PILE_RELEASE_DLYD (first release ever) | Assert `ACCEPTED_HIT` |
| `ACC_MANY_HIT` | PILE_RELEASE_DLYD (subsequent releases) | Assert `EXTENDED_EVENT` |
| `ACC_MANY_HIT` | Counter → 1 and final release | → `ACC_NO_HIT` |

`FIRST_RELEASE_FLAG` tracks whether the first release of a pileup train has been seen; subsequent releases produce `EXTENDED_EVENT` rather than `ACCEPTED_HIT`. ✅ verified 2026-04-25 — `pileup_processor.vhd:L54,L210,L235,L237` (signal declared L54; set on first release L210,L235; checked L237 to branch ACCEPTED_HIT vs EXTENDED_EVENT)

#### `PU_TOO_SHORT` Gate

Output block (combinatorial, outside FSM): `ACCEPTED_HIT` and `EXTENDED_EVENT` are gated to '0' when `PU_TOO_SHORT = '1'`. This flag is set upstream when the pileup window is too short to be valid. ✅ verified 2026-04-25 — `pileup_processor.vhd:L65-67` (`ACCEPTED_HIT <= INT_ACCEPTED_HIT when (PU_TOO_SHORT = '0') else '0'`; same for `EXTENDED_EVENT`)

#### `PEHQ_RD_ADDR_ADV`

Driven directly (not gated) from `PILE_RELEASE_DLYD` regardless of pileup state — the PEHQ read pointer must always advance to prevent FIFO stall, whether the event is accepted or rejected.
✅ verified 2026-04-24 — pileup_processor.vhd:L108 ("PEHQ_RD_ADDR_ADV <= PILE_RELEASE_DLYD; -- MBO: must always advance read pointer.")

#### Commented-out PU_ONLY mode

Three states (`PU_ONLY_NO_HIT`, `PU_ONLY_ONE_HIT`, `PU_ONLY_MANY_HIT`) were prototyped in 2013-06-04 by JTA to accept only pileup events and reject clean singles. This code is fully commented out and never compiled. ✅ verified 2026-04-25 — `pileup_processor.vhd:L51` (comment `--20130604 allow for pileup-only events: PU_ONLY_NO_HIT, PU_ONLY_ONE_HIT, PU_ONLY_MANY_HIT`); L264-306 (states fully commented out)

---

## SERDES_RX_Mach.vhd — Router Command Frame Receiver

**Entity:** `SERDES_RX_Mach`  
**Source:** `MAIN_FPGA/BuildBranches/DGS_TAG_20180607_TWEAK/DGS/Source/SERDES_RX_Mach.vhd` (1,252 lines)  
**Clock domain:** CLK50 (50 MHz)  
**Author:** ANL (JTA/MBO); last significant change ≈2016-12-12 (F2 decoder addition)

The SERDES_RX_Mach is the DIG counterpart to the MTRG's `SERDES_RX_Mach_R2.vhd`. It parses the 20-frame, 100-word TTCL command cycle from the Router into actionable flags for the rest of the digitizer. One instance per digitizer.

### Clock and Lock Architecture

The machine runs on CLK50 (50 MHz = 20 ns/word). It processes one 16-bit word per clock. With 20 frames × 5 words = 100 words per cycle, the full cycle period is 2 µs. A 5-stage prelock sequence (matching the F20 End-of-Cycle pattern) must be observed before the machine transitions to the locked operating state.

**Lock logic:**
- `DATA_CHECK_FLAG` — combinatorial; compares current word against `FIXED_BITS[WORD_INDEX]` table if `DATA_CHECK_EN[WORD_INDEX]` = 1
- `STRINGENT_LOCK_FLAG` — if set, any `DATA_CHECK_FLAG` = 0 returns machine to PRELOCK1 (except F20, which always checks)
- `SERDES_SM_LOCKED` — registered; falls immediately if data mismatch occurs in stringent mode or on F20 mismatch
- `SERDES_SM_LOST_LOCK_FLAG` — SR flip-flop; set when locked→unlocked transition occurs; cleared by `SERDES_SM_LOST_LOCK_RST`
- `UNQUALIFIED_SM_LOCKED` — diagnostic; equals `DATA_CHECK_FLAG` regardless of STRINGENT_LOCK_FLAG

### Prelock Sequence

The machine initializes into `PRELOCK1` (WORD_INDEX = 0xDF). The 5-step prelock looks for the F20 End-of-Cycle pattern:

| State | WORD_INDEX | Condition to advance |
|-------|-----------|---------------------|
| PRELOCK1 | 0xDF | DATA_CHECK_FLAG = 1 (= F20W1 pattern 0xFFFF) |
| PRELOCK2 | 0xE0 | DATA_CHECK_FLAG = 1 (= F20W2 pattern 0x0000) |
| PRELOCK3 | 0xE1 | DATA_CHECK_FLAG = 1 (= F20W3 pattern 0xFFFF) |
| PRELOCK4 | 0xE2 | DATA_CHECK_FLAG = 1 (= F20W4 pattern 0x0000) |
| PRELOCK5 | 0xE3 | DATA_CHECK_FLAG = 1 (= F20W5 pattern 0x5555) |

Any mismatch at any prelock stage resets to PRELOCK1. On success, machine enters F1W1 (WORD_INDEX = 0x80 = 128, then wraps to word-count index 0x01 on next cycle).

_Note: TRIG_TIMESTAMP is zeroed during prelock states (WORD_INDEX 223–227)._

### Frame Structure (20 frames × 5 words)

| Frame | States | Function | DIG action |
|-------|--------|----------|------------|
| F1 (Sync) | F1W1..F1W5 | 48-bit timestamp + sync/imperative sync flag | Assert `SERDES_SYNC_FLAG`; optionally `SERDES_ISYNC_FLAG`; latch `SERDES_SYNC_TIMESTAMP[47:0]` in W2/W3/W4; W1 detects ISY vs SY |
| F2 (Debug) | DEBUG_FRAME (×5) | Detector state data | Latch `TRIG_MON_DET_DATA` (word index 5), `TRIG_MON_XTRA_DATA` (word index 6) in separate F2_DECODER process |
| F3–F10 (Trigger Decision) | TDF_W1..TDF_W5 (×8) | 8 trigger decision frames (looped) | W1: detect trigger vs null (0xAAAA = null); latch TRIG_TYPE[2:0] = bits[10:8]; set EARLY_TRIG_FLAG; W2–W4: assemble TRIG_TIMESTAMP[47:0]; W5: assert TRIG_FLAG (EARLY_TRIG_FLAG persists until W5) |
| F11 | FRAME11 (×5) | Spare — null (0xAAAA×4 + 0x0000) | Pass-through; data checked in stringent mode |
| F12 | FRAME12 (×5) | Router Command Frame (stripped by Router → null 0xAAAA at DIG) | Pass-through |
| F13 | FRAME13 (×5) | Demand Slow Data (GRETINA compat.; unused in DGS) | Pass-through; pattern 0x40FB, 0xA5A5, 0x5A5A, 0xA5A5 checked |
| F14 | FRAME14 (×5) | Internal Trigger commands (stripped by Router → null 0xAAAA) | Pass-through |
| F15 | FRAME15 (×5) + F15_DECODER | GRETINA Asynchronous Command Frame | Decoded by separate F15_DECODER process (see below) |
| F16 (Sync Capture) | F16W1..F16W5 | DGS Synchronous System Capture Command | Detect command (W1 MSB ≠ 0xAA); latch SYNC_CAPTURE_TS[31:16] (W2), [15:0] (W3), LENGTH (W4), FIFO_DELAY (W5); assert SYNC_CAPTURE_FLAG |
| F17 | FRAME17 (×5) | Auxiliary Detector Command Frame (indeterminate data) | Pass-through |
| F18 | FRAME18 (×5) | Spare — null (0xAAAA×4 + 0x0000) | Pass-through; data checked |
| F19 | FRAME19 (×5) | Spare — null (0xAAAA×4 + 0x0000) | Pass-through; data checked |
| F20 (EOC) | F20W1..F20W5 | End-of-Cycle | Always checks data (0xFFFF/0x0000/0xFFFF/0x0000/0x5555); any mismatch → PRELOCK1; success → wrap to F1W1 |

**Null pattern for all stripped/spare frames:** `0xAAAA, 0xAAAA, 0xAAAA, 0xAAAA, 0x0000` (words 1–4 = 0xAAAA, word 5 = 0x0000).

### Trigger Decision Frame Detail

Frames 3–10 are all handled by a single 5-state loop (`TRIGGER_DECISION_FRAME_WORD_1..5`). The machine iterates through all 8 frames sharing the same 5 states, distinguished by `xWORD_INDEX`.

**Trigger detection (W1, word index 10/15/20/.../45):**
- If `RECEIVED_CONTROL_DATA[15:11]` ≠ `10101` → not a null (0xAAAA), latch `TRIG_TYPE = DATA[10:8]`, assert `EARLY_TRIG_FLAG`
- `10101` = upper 5 bits of `0xAAAA` → null means no trigger this frame
- Local triggers: upper byte 0x5n (n=0..7); Remote triggers: upper byte 0x6n

**Timestamp assembly (W2–W4):**
- W2: `xTRIG_TIMESTAMP[47:32]` ← received data
- W3: `xTRIG_TIMESTAMP[31:16]` ← received data
- W4: `xTRIG_TIMESTAMP[15:0]` ← received data

**Output (W5):** `TRIG_FLAG` ← `EARLY_TRIG_FLAG` (one-clock pulse). The `OUTPUT_BUFFER` process registers `TRIG_TIMESTAMP` into the final output only when `TRIG_FLAG` is asserted.

### F15 Decoder (Asynchronous Command Frame)

Separate process `F15_DECODER` added 2016-04-18 to decode Frame 15 without bloating the main machine. Operates on `xINT_WORD_INDEX`.

| Word index | Field decoded |
|-----------|---------------|
| 70 (W1) | Command byte = `DATA[15:8]`: 0x04 → `CAL_INJECT_FLAG`; 0x08 → `LATCH_STATUS_FLAG`; 0x10 → `FRONT_END_RESET_FLAG`; 0x18 → `RESET_LINKS_FLAG`; 0x22 → `EXTERNAL_DISC_FLAG_PENDING` |
| 71 (W2) | If EXT_DISC pending: `EXT_DISC_CHAN_SELECT` ← `DATA[9:0]`; bits[15:14] control latch enable (`10`=enable, `01`=disable) |
| 72 (W3) | `EXT_DISC_MODULE_SELECT` ← `reg_user_package_data[11:0]` AND `DATA[11:0]`; latch set/clear bits |
| 73 (W4) | `EXT_DISC_MODULE_SELECT` ← previous result OR `DATA[11:0]` |
| 74 (W5) | If latch mode: `EXTERNAL_DISC_FLAG` ← `F15_LATCH_STATE`; else if pending AND module_select ≠ 0: assert `EXTERNAL_DISC_FLAG` for one clock |

**Latch mode:** When `ENABLE_F15_LATCH = 1`, the external discriminator persists between frames (state held in `F15_LATCH_STATE`). Latch mode allows long-duration gating from a single TTCL command.
✅ verified 2026-04-24 — SERDES_RX_Mach.vhd:L110-111 (ENABLE_F15_LATCH, F15_LATCH_STATE signal declarations); L364-365 (latch drives EXTERNAL_DISC_FLAG)
- **W2 (index 71):** bits[15:14]='10' → `ENABLE_F15_LATCH <= '1'` (enable); bits[15:14]='01' → disable latch and clear F15_LATCH_STATE.
- **W3 (index 72):** bits[15:14]='10' → `F15_LATCH_STATE <= '1'` (SL = set latch); bits[15:14]='01' → `F15_LATCH_STATE <= '0'` (CL = clear latch). Logic is correct — no bug.
✅ verified 2026-04-24 — SERDES_RX_Mach.vhd:L347-350 (DGS_TAG_20180607_TWEAK branch: SL assigns '1', CL assigns '0' with comment "clear the latch if CL bit set"); prior KB entry incorrectly transcribed '0' as '1' for CL case.

### Veto Decoder

Separate process `VETO_DECODER` added 2016-04-18. Fires on the 5th word of every frame (word indices 4, 9, 14, 19, 24, 29, 34, 39, 44, 49, 54, 59, 64, 69, 74, 79, 84, 89, 94). Outputs `VETO_EVENT[9:0]` directly from `RECEIVED_CONTROL_DATA[9:0]`.
✅ verified 2026-04-24 — SERDES_RX_Mach.vhd:L395-408 (case xINT_WORD_INDEX when 4|9|14|19|24|29|34|39|44|49|54|59|64|69|74|79|84|89|94; VETO_EVENT <= RECEIVED_CONTROL_DATA(9:0))

**SLAVE_MODE special case:** If `SLAVE_MODE=TRUE`, channels 9:5 are forced to `"00000"` (no veto); channels 4:0 always receive `RECEIVED_CONTROL_DATA(4:0)` regardless of mode.
✅ verified 2026-04-24 — SERDES_RX_Mach.vhd:L399-403 (VETO_EVENT(4:0) always gets RECEIVED_CONTROL_DATA(4:0); SLAVE_MODE zeroes out VETO_EVENT(9:5))
✅ verified 2026-04-24 — SERDES_RX_Mach.vhd:L397-398 (DGS_TAG_20180607_TWEAK): the VHDL comment "only channels 9:5 (Ge Side) respond to vetoes" is poorly worded but the code logic is clear: SLAVE_MODE forces VETO_EVENT(9:5) to '0' (Ge channels suppressed) while VETO_EVENT(4:0) always gets RECEIVED_CONTROL_DATA(4:0) (BGO pattern signals always pass through). Comment's intended meaning: "in SLAVE_MODE, Ge channels do NOT get vetoes; BGO always reads out" — the word "respond" refers to the veto suppression path, not the data path. No code bug; comment resolved.

### Outputs Summary

| Output | Width | Description |
|--------|-------|-------------|
| `SERDES_SM_LOCKED` | 1 | Machine in lock |
| `SERDES_SM_LOST_LOCK_FLAG` | 1 | Lock was lost (SR FF, cleared by RST) |
| `SERDES_SYNC_FLAG` | 1 | Sync frame received (one clock pulse, F1W5) |
| `SERDES_ISYNC_FLAG` | 1 | Imperative Sync received (one clock pulse, F1W5) |
| `SERDES_SYNC_TIMESTAMP` | 48 | Timestamp of last sync |
| `TRIG_FLAG` | 1 | Trigger accepted (one clock pulse, TDF_W5) |
| `TRIG_TIMESTAMP` | 48 | Timestamp from trigger message (held until next trigger) |
| `TRIG_TYPE` | 3 | Trigger type from DATA[10:8] of TDF_W1 |
| `VETO_EVENT` | 10 | Per-channel veto commands (5th word of every frame) |
| `CAL_INJECT_FLAG` | 1 | F15 calibration inject command (one clock) |
| `LATCH_STATUS_FLAG` | 1 | F15 latch status command (one clock) |
| `FRONT_END_RESET_FLAG` | 1 | F15 front-end reset command (one clock) |
| `RESET_LINKS_FLAG` | 1 | F15 link reset command (one clock) |
| `EXTERNAL_DISC_FLAG` | 1 | F15 external discriminator assertion (pulse or latched) |
| `EXT_DISC_CHAN_SELECT` | 10 | F15 external discriminator channel mask |
| `SYNC_CAPTURE_FLAG` | 1 | F16 synchronous capture command received (one clock) |
| `SYNC_CAPTURE_TS` | 32 | F16 capture start timestamp |
| `SYNC_CAPTURE_LENGTH` | 16 | F16 capture length |
| `SYNC_CAPTURE_FIFO_DELAY` | 16 | F16 FIFO capture delay |
| `TRIG_MON_DET_DATA` | 16 | F2 W1: detector state at time of trigger (e.g. target wheel) |
| `TRIG_MON_XTRA_DATA` | 16 | F2 W2: multiplicity information (live or latched-at-trigger); only present in TTCL mode (zeroed when PEQ_BYPASS='1') ✅ verified 2026-04-30 — Channel_Readout_Mach.vhd:L331-337 (20180507 + 20230809 tags: "the TRIG_MON_XTRA_DATA is, as of 20180425, multiplicity information… data should only be present in the header if in TTCL mode") |
| `WORD_INDEX` | 8 | Diagnostic: current word index (bit7=1 during prelock) |
| `UNQUALIFIED_SM_LOCKED` | 1 | Diagnostic: equals DATA_CHECK_FLAG (ignores STRINGENT_LOCK_FLAG) |

### Differences from MTRG SERDES_RX_Mach_R2.vhd

| Feature | DIG (`SERDES_RX_Mach.vhd`) | MTRG (`SERDES_RX_Mach_R2.vhd`) |
|---------|--------------------------|--------------------------------|
| Clock | CLK50 (50 MHz) | CLK50 (50 MHz) |
| Frames | 20 frames × 5 words | Same |
| Trigger frames | F3–F10 (8 frames, looped) | F3–F10 (same) |
| Veto output | `VETO_EVENT[9:0]` (per-channel, 5th word of every frame) | Similar veto structure |
| F2 content | DET_DATA + XTRA_DATA (2016-12-12) | Similar |
| F15 latch mode | Supported (`ENABLE_F15_LATCH`, `F15_LATCH_STATE`) | Not present |
| SLAVE_MODE | Suppresses BGO veto (ch 4:0) | Not applicable |
| SYNC_CAPTURE (F16) | Full decode: TS, length, FIFO_DELAY | Not present |
| Prelock matching | F20 5-word pattern (0xFFFF/0x0000/0xFFFF/0x0000/0x5555) | Same F20 EOC pattern |

---

## Timestamp_Generator.vhd — 48-bit Timestamp Synchronization

**Entity:** `TIMESTAMP_GENERATOR`  
**Authors:** Michael Oberling (original), John Anderson (modifications)  
**Lines:** 268  
**Clock domain:** CLK (100 MHz, passed from top level)

### Role

Maintains the DIG's local 48-bit timestamp counter and keeps it synchronized to the master trigger (MTRG) via SERDES SYNC and ISYNC frames. Also generates TS_EDGE_FLAGS — one-tick-wide pulse outputs tied to specific timestamp bit transitions, used for diagnostic and test-rate generation.

### State Machine — Synchronization FSM

Three states control timestamp behavior:
✅ verified 2026-04-24 — Timestamp_Generator.vhd:L68 (type declaration), L183-222 (FSM: RESET_MACH exits on LOCAL_TS_RESET_FLAG='0'; RUN_UNSYNCHRONIZED transitions on SERDES_SM_LOCKED='1' AND (ISYNC_EDGE OR SYNC_EDGE); RUN_SYNCHRONIZED falls back on SERDES_SM_LOCKED='0')

| State | Behavior |
|-------|----------|
| `RESET_MACH` | Counter held at zero; stays here while `LOCAL_TS_RESET_FLAG=1` |
| `RUN_UNSYNCHRONIZED` | Counter free-running; transitions to `RUN_SYNCHRONIZED` on first SYNC or ISYNC edge from a locked SERDES machine |
| `RUN_SYNCHRONIZED` | Counter free-running; falls back to `RUN_UNSYNCHRONIZED` if SERDES SM loses lock; stays in sync on each SYNC frame |

### Timestamp Loading

The `TS_COUNTER` process runs on the 100 MHz clock:

- On `TS_RESET`: clears INTERNAL_TIMESTAMP to zero
- On `SERDES_ISYNC_FLAG_EDGE`: **always** loads `SERDES_SYNC_TIMESTAMP` (unconditional hard-sync)
- On `SERDES_SYNC_FLAG_EDGE` while not yet synchronized: loads timestamp (initial sync)
- Otherwise: increments by 1 each clock (free-run at 100 MHz)
✅ verified 2026-04-24 — Timestamp_Generator.vhd:L110-128 (TS_COUNTER: ISYNC_FLAG_EDGE unconditional load L120, SYNC_FLAG_EDGE+TS_SYNCHRONIZED='0' load L122, else increment L126)

**Edge detection:** Both SYNC and ISYNC flags from the SERDES RX machine run at 50 MHz. A 2-stage pipeline detects the 0→1 transition to produce a single 100 MHz clock wide edge pulse, preventing double-increments.
✅ verified 2026-04-24 — Timestamp_Generator.vhd:L88-103 (SYNC_EDGE_DETECT: 2-element pipe per flag; rising edge = PIPE(1)='0' AND PIPE(0)='1')

### Sync Error Detection

`SYNC_ERROR_DETECT` compares `INTERNAL_TIMESTAMP` against `SERDES_SYNC_TIMESTAMP` every time a sync frame arrives (2-clock pipeline delay for the comparison to settle). If synchronized and the values disagree, `SYNC_ERROR` and `SYNC_ERROR_FLAG` are asserted. If the SERDES SM loses lock while in synchronized state, the error is also raised. `SYNC_ERROR` is one-tick wide (internal); `SYNC_ERROR_FLAG` remains set until reset.

A `basic_capture_counter` instance counts cumulative sync errors; readable via `REG_TS_ERR_COUNT`.

### TS_EDGE_FLAGS

Seven one-tick-wide edge pulses tied to specific timestamp bit transitions (implicit ÷2 from the bit transition detection gives the listed rates):

| Flag bit | TS bit | Period | Rate | Use |
|----------|--------|--------|------|-----|
| 0 | 8 | ~5.12 µs | ~195 kHz | Pileup test rate |
| 1 | 10 | ~20.48 µs | ~48.8 kHz | DGS target rate emulation |
| 2 | 15 | ~655 µs | ~1.5 kHz | — |
| 3 | 19 | ~10.5 ms | ~95 Hz | — |
| 4 | 21 | ~41.9 ms | ~23.8 Hz | — |
| 5 | 23 | ~167.8 ms | ~6.0 Hz | — |
| 6 | 26 | ~1.34 s | ~0.745 Hz | Slowest test rate |
| 7 | — | — | — | Hardwired to '0' |
✅ verified 2026-04-24 — Timestamp_Generator.vhd:L244-268 (constant TS_IDX(6 downto 0):=(26,23,21,19,15,10,8) so TS_IDX(0)=8…TS_IDX(6)=26; TS_EDGE_FLAGS(7)<='0' hardwired L249; 0→1 transition per-bit in generate loop L251-265)

These flags feed `reg_external_disc_src` mode 3 (timestamp-edge-triggered discriminator) and general test/diagnostic use in `Digitizer.vhd`.

---

## Trigger_Mux.vhd — Trigger Source Selector

**Entity:** `Trigger_Mux`  
**Lines:** 121  
**Clock domain:** CLK (100 MHz)

### Role

Selects which trigger system drives the channel readout machines. The 2-bit `TRIGGER_SELECT` register chooses among four modes, outputting a unified `TRIG_FLAG_OUT`, `TRIG_TIMESTAMP_OUT`, and `TRIG_TYPE_OUT`.

### Trigger Select Modes
✅ verified 2026-04-24 — Trigger_Mux.vhd (121 lines): mode 00→FLAG='0'/TS=0 (L84-87); mode 01→EXT falling edge PIPE(2)=1 AND PIPE(1)=0 (L91-94, 3-element pipe L71-73); mode 10→TTCL rising edge PIPE='0' AND FLAG='1' (L100-103, 1-stage pipe L74); mode 11→TS(15:0)=0x0000 (L108-111); OUTPUT_LATCH registers all 3 outputs (L59-65)

| `TRIGGER_SELECT` | Mode | Source | Flag Logic |
|-----------------|------|--------|------------|
| `00` | Disabled | None | FLAG always 0; TS = 0 |
| `01` | External trigger | `EXT_TRIG_*` | FLAG on falling edge of `EXT_TRIG_FLAG` (3-stage pipeline, detects 1→0) |
| `10` | TTCL (Router) | `TTCL_TRIG_*` | FLAG on rising edge of `TTCL_TRIG_FLAG` (1-stage pipeline, detects 0→1) |
| `11` | Diagnostic | `DIAG_TRIG_TIMESTAMP` | FLAG when lower 16 bits of current TS = 0x0000 (periodic rate trigger) |

**Note on mode 01:** EXT_TRIG_FLAG is pipelined 3 stages; the flag is asserted when `PIPE(2)=1` AND `PIPE(1)=0` — this detects the **falling** edge (end of accept window), not the rising edge. This is consistent with the external trigger system asserting a level that stays high during the accept window.

**Output latch:** All three outputs are registered in `OUTPUT_LATCH` before leaving the module (additional 1-cycle pipeline).

---

## Channel_Readout_Controller.vhd — Readout Timing & Decision Logic

**Entity:** `channel_readout_controller`  
**Author:** Michael Oberling  
**Lines:** 697  
**Clock domain:** CLK (100 MHz)

### Role

Computational core of the per-channel readout pipeline. Calculates *when* a waveform readout should begin based on the M-buffer size and PRETRIGGER offset, monitors the Event Header FIFO for pending events, coordinates with the Trigger Rondel (PEQ) accept/reject decisions, and issues `READ_EVENT` or `DROP_EVENT` commands. It also manages the accepted-event FIFO's programmable full threshold to guarantee that no partial events enter the buffer.

This module is instantiated inside `channel_readout_mach` (the wrapper), not directly from `Digitizer.vhd`.

### Key Constants
✅ verified 2026-04-24 — Channel_Readout_Controller.vhd:L135-141 (cHEADER_SIZE=28, cREPORTED=cHEADER-2=26, cT_BUFFER=2048, cACPTD_FIFO_DEPTH=1025, cMINIMUM_PROG_FULL=13, cPROG_EMPTY=DEPTH-HEADER/2=1011); LED/CFD calibration as generics (L59-60), set in Channel_Readout_Mach.vhd:L92-93 (led=1, cfd=2)

| Constant | Value | Meaning |
|----------|-------|---------|
| `cHEADER_SIZE` | 28 | Words injected ahead of waveform |
| `cREPORTED_HEADER_SIZE` | 26 | Per GRETINA spec (cHEADER_SIZE − 2) |
| `cT_BUFFER_SIZE` | 2048 | T-buffer size in 100 MHz cycles (20.48 µs) |
| `cACPTD_EVENT_FIFO_DEPTH` | 1025 | Accepted event FIFO depth |
| `cMINIMUM_PROG_FULL_VAL` | 13 | Min allowed prog_full threshold (CoreGen constraint) |
| `cPROG_EMPTY_THRESH` | 1011 | = FIFO_DEPTH − HEADER_SIZE/2; used for header-only mode |
| `LED_TRIGGER_OFFSET_CALIBRATION` | 1 | Pipeline delay constant for LED mode |
| `CFD_TRIGGER_OFFSET_CALIBRATION` | 2 | Pipeline delay constant for CFD mode |

### 3-Stage Readout Timing Pipeline

The readout start timestamp is calculated in a 3-stage pipeline (one clock per stage):

| Stage | Calculation |
|-------|-------------|
| S1 | Latch `READOUT_WINDOW` (force even), `READOUT_PRETRIGGER`, compute `WAVEFORM_DELAY = M + T = INTERNAL_READOUT_OFFSET + 2048` |
| S2 | `WAVEFORM_WINDOW_WIDTH = READOUT_WINDOW − HEADER_SIZE` (floor 0); `PACKET_LENGTH = READOUT_WINDOW`; `WAVEFORM_WINDOW_OFFSET = WAVEFORM_DELAY − PRETRIGGER` |
| S3 | Apply CFD/LED calibration offset; compute `ACPTD_EVENT_FIFO_FULL_THRESH`; latch final parameters (only updated while in `WAIT_FOR_EVENT_FIFO` state) |

### Rollover-Safe Timestamp Comparison

Because the discriminator timestamp can roll over during readout (24-bit comparison window), the logic applies an MSB-flip trick: if `NEXT_EVENT_TIMESTAMP[23]=1` (i.e. close to rollover), bit 23 of both `READOUT_START_TIMESTAMP` and `CURRENT_TIMESTAMP` is cleared/complemented. This maps the comparison into a safe 23-bit window.

The 24-bit `EVENT_OFFSET_TIME = CURRENT_TS − READOUT_START_TS` has bit 23 inverted before comparison:
- `(NOT xEVENT_OFFSET_TIME(23)) & xEVENT_OFFSET_TIME(22 downto 0) = 0x800000` → exactly on time → `EVENT_READOUT_READY_FLAG=1, OFFSET_COMPARISON=0`
- `> 0x800000` → event is late → `EVENT_READOUT_READY_FLAG=1, OFFSET_COMPARISON=1`
- Otherwise → event not yet ready
✅ verified 2026-04-24 — Channel_Readout_Controller.vhd:L393-396 ((NOT xEVENT_OFFSET_TIME(23)) & xEVENT_OFFSET_TIME(22:0) compared against X"800000")

### State Machine (7 states)

| State | Description |
|-------|-------------|
| `WAIT_FOR_EVENT_FIFO` | Idle; updates readout timing parameters each cycle; branches to NORMAL or EXTENDED wait on `NEXT_EVENT_DATA_READY` |
| `WAIT_FOR_EVENT_FIFO_DELAY` | 1-clock delay to let Event Header FIFO respond after a drop/read |
| `NORMAL_EVENT_WAIT` | Waits for accept/reject decision from PEQ FIFO; if rejected → DROP_THIS_EVENT; if accepted + ready + FIFO has room → EVENT_READ_TRIG; if offset with mode 00 → DROP_AND_COUNT |
| `EXTENDED_EVENT_WAIT` | Like NORMAL but uses `LAST_EVENT_DECISION` (carry-over from initial ACCEPTED_HIT); never reads Event Decision FIFO (PEQ only judges ACCEPTED_HITs, not extension events) |
| `EVENT_READ_TRIG` | Applies `EVENT_EXTEND_MODE` policy; computes `LAST_EVENT_OFFSET_TIME`; asserts `READ_EVENT` or `DROP_EVENT` based on FIFO readiness; transitions back to `WAIT_FOR_EVENT_FIFO_DELAY` |
| `DROP_THIS_EVENT` | Asserts `DROP_EVENT` for one clock; no counter increment |
| `DROP_AND_COUNT_THIS_EVENT` | Asserts `DROP_EVENT` + `DROP_TRAIN` + `DROPPED_EVENT_COUNT_EN` for one clock |

### EVENT_EXTEND_MODE Behavior (in EVENT_READ_TRIG)

| Mode | Behaviour |
|------|----------|
| `00` | Drop offset events (`xOFFSET_FLAG=1` → DROP_AND_COUNT); for non-offset, drop if FIFO not ready; `LAST_EVENT_OFFSET_TIME=0` (full waveform read) |
| `01` | Full extension — never truncate; drop only if FIFO not ready; `LAST_EVENT_OFFSET_TIME=0` |
| `10` | Truncation — reads `SHIFTED_EVENT_OFFSET_TIME` (offset divided by DEC_FACTOR); waveform window shrinks by offset amount |
| `11` | Headers only for offset events — if `xOFFSET_FLAG=1`, set `LAST_EVENT_OFFSET_TIME=WAVEFORM_WINDOW_WIDTH` (waveform omitted) |

**DROP_TRAIN:** Once any event in a pileup train is dropped, `DROP_TRAIN` is latched high and all subsequent events in that train are also dropped (avoids partial train readout).
✅ verified 2026-04-24 — Channel_Readout_Controller.vhd:L556 (DROP_TRAIN='1' → DROP_AND_COUNT_THIS_EVENT); L583-627 (DROP_TRAIN latched in EVENT_READ_TRIG on ACPTD_EVENT_FIFO_READY='0'); L683 (DROP_AND_COUNT state holds DROP_TRAIN<='1')

### Event Decision FIFO

A 5×34-entry common-clock FIFO (`fifo_5X34_comclk_fwft`) buffers PEQ accept/reject decisions:
- `DIN[0]` = ACCEPT, `DIN[1]` = VETO, `DIN[4:2]` = EVENT_TYPE (3-bit trigger type)
- Write enable gated by NOT FULL; if FULL when decision arrives, `EVENT_DECISION_DROPPED` is set (non-recoverable, requires reset)
- Read enable issued in `NORMAL_EVENT_WAIT` when decision is consumed
✅ verified 2026-04-24 — Channel_Readout_Controller.vhd:L315 (fifo_5X34_comclk_fwft instance); L332-334 (DIN: bit0=ACCEPT, bit1=VETO, bits4:2=EVENT_TYPE)

### Dropped Event Counter

A `sync_capture_counter` instance counts events dropped due to FIFO full or offset violations (not PEQ rejects or pileup drops). Controlled by `COUNTER_RESET` and `DROPPED_EVENT_COUNTER_MODE` (rate vs accumulate).

---

## Channel_Readout_Mach.vhd — Per-Channel Readout Wrapper

**Entity:** `channel_readout_mach`  
**Author:** Michael Oberling  
**Lines:** 491  
**Clock domain:** CLK (100 MHz) + ACPTD_EVENT_FIFO_READ_CLK (50 MHz for FIFO read side)

### Role

Structural wrapper that integrates the five sub-components of a single channel's readout pipeline:

1. **`channel_readout_controller`** — timing calculation + accept/drop FSM (see above)
2. **`event_header_fifo`** (`Event_Header_FIFO.vhd`) — buffers pre-formatted event headers from `jta_channel.vhd`; outputs header words one-at-a-time on demand
3. **`decimator`** — programmable power-of-2 waveform decimation (factor 1–128); averaging + decimation of ADC samples
4. **`event_data_fifo`** — intermediate FIFO between decimator output and `event_packer`; tracks `dec_enable` to correctly gate waveform data
5. **`event_packer`** — "accordion" FIFO controller; interleaves header words and waveform samples into the `acptd_event_fifo` (see `deep_fpga_DIG_modules.md § event_packer`)
6. **`fifo_36x1025_sepclk_pfiport_fwft`** — the accepted-event FIFO (36-bit × 1025 entries, dual-clock, prog-full/prog-empty)

### Data Flow

```
jta_channel.vhd
    │ accepted_hit / event_data / timing_mark / adc_out
    ▼
 event_header_fifo         decimator
    │ next_event_header_*       │ dec_data / dec_data_valid
    │                           ▼
    │                    event_data_fifo
    │                           │ event_data_out[1:0]
    └──────────────────────────►│
                         event_packer
                                │ acptd_event_fifo_wr_en / din
                                ▼
                     acptd_event_fifo (36-bit × 1025)
                                │ dout[35:0]
                                ▼
                     Fifo.vhd (IDT 7007) → VME readout
```

### Trigger Data Modification (added 2018-04-25)

Before passing trigger metadata to `event_header_fifo`, the wrapper conditionally zeroes fields based on `PEQ_BYPASS`:
✅ verified 2026-04-24 — Channel_Readout_Mach.vhd:L324-334 (MOD_TRIG_ARRIVAL/MUX_TS zeroed when PEQ_BYPASS='1' L324-325; DET_DATA always passed L329; XTRA_DATA zeroed when PEQ_BYPASS='1' L334; comment confirms 20180425 multiplicity context)

| Signal | PEQ_BYPASS=0 (TTCL mode) | PEQ_BYPASS=1 (bypass) |
|--------|--------------------------|------------------------|
| `TRIG_ARRIVAL_TIMESTAMP` | passed through | forced to 0x0000 |
| `TRIGGER_MUX_TIMESTAMP` | passed through | forced to 0x0000 |
| `TRIG_MON_DET_DATA` | passed through | passed through (always — target wheel data) |
| `TRIG_MON_XTRA_DATA` | passed through | forced to 0x0000 (multiplicity data only relevant in TTCL mode) |

### Decimation Pause

`dec_pause = dec_pause_request AND enable_dec_pause` — `enable_dec_pause` allows the on/off decimation feature to be globally disabled without changing timing. `dec_pause_request` comes from `event_packer` (timing-mark driven). Changed 2016-04-19.
✅ verified 2026-04-24 — Channel_Readout_Mach.vhd:L399 (dec_pause <= dec_pause_request AND enable_dec_pause; -- 20160419)

### Waveform Output Format (event_data_in → event_data_fifo)

When `write_flags=1`:
```
event_data_in[15:14] = dec_timing_mark[1:0]  (bit 15 = dec_pause resample; bit 14 = disc/peak timing mark)
event_data_in[13:0]  = dec_data[15:2]          (upper 14 bits of 16-bit decimated ADC value)
```
When `write_flags=0`: full 16-bit `dec_data` without flag insertion.
✅ verified 2026-04-24 — Channel_Readout_Mach.vhd:L421 (event_data_in <= dec_timing_mark & dec_data(15 downto 2) when write_flags='1' else dec_data; 20160308 change noted)

**Calibration offsets** (hardcoded in this file):
- LED: `led_trigger_offset_calibration = 1`
- CFD: `cfd_trigger_offset_calibration = 2`
✅ verified 2026-04-24 — Channel_Readout_Mach.vhd:L92-93 (led=1, cfd=2)

---

## See Also

- [deep_fpga_DIG_modules2.md](deep_fpga_DIG_modules2.md) — Part 2: DC balance, waveform FIFOs, decimator, event header FIFO, channel collection FIFO, VME interface, register map
- [deep_fpga_DIG.md](deep_fpga_DIG.md) — top-level DIG architecture, source file table, SERDES RX format
- [deep_fpga_DIG_channel.md](deep_fpga_DIG_channel.md) — per-channel signal processing, LED/CFD discriminators, Trigger Rondel
- [deep_fpga_DIG_eventpacket.md](deep_fpga_DIG_eventpacket.md) — full event packet format
- [fpga.md](fpga.md) — FPGA overview and cross-reference index
