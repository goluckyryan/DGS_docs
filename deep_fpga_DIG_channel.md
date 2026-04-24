# DIG Firmware — Per-Channel Signal Processing

Stability: C3 - Structural / stable

_Split from `deep_fpga_DIG.md` on 2026-04-10 (file exceeded 1200 lines)._
_Source: `DGS_tools_pack/raw_FPGA/Dig*/` — `jta_channel.vhd`, `thresh_disc.vhd`, `Digitizer.vhd`. PDF: `ANL Digitizer Firmware for Experts.pdf`._

## Table of Contents

- [Per-Channel Signal Processing: LED and CFD Modes](#per-channel-signal-processing-led-and-cfd-modes)
  - [Common Signal Path — Delay Chain and Filtering](#common-signal-path--delay-chain-and-filtering)
  - [LED Mode — Leading-Edge Threshold Discriminator](#led-mode--leading-edge-threshold-discriminator)
  - [CFD Mode — Constant Fraction Discriminator](#cfd-mode--constant-fraction-discriminator)
  - [Mode Selection](#mode-selection)
  - [After Discrimination — PEQ and Energy Integration](#after-discrimination--peq-and-energy-integration)
  - [Trigger Rondel — chan_trigger_control.vhd (PEQ State Machine)](#trigger-rondel--chan_trigger_controlvhd-peq-state-machine)
  - [Pileup Detection](#pileup-detection)
  - [VME Registers for Discriminator Configuration](#vme-registers-for-discriminator-configuration)
- [VME FPGA](#vme-fpga)
  - [Source Files](#source-files)
  - [Bitfiles](#bitfiles)
  - [Clock Select Register (`clk_select`)](#clock-select-register-clk_select)
- [Main FPGA Bitfiles](#main-fpga-bitfiles)
- [IP Cores](#ip-cores)
- [See Also](#see-also)

---

## Per-Channel Signal Processing: LED and CFD Modes

Each of the 10 channels runs an identical 100 MHz pipeline implemented in `jta_channel.vhd`. The pipeline has two discriminator modes selectable per channel: **LED** (Leading-Edge Discriminator, threshold-based) and **CFD** (Constant Fraction Discriminator, zero-crossing-based). Both modes share the same upstream delay chain and filters; they differ in how they derive the discriminator signal and fire the event timestamp.

### Common Signal Path — Delay Chain and Filtering

The raw 14-bit ADC sample passes through a series of programmable delay buffers before reaching the discriminators. All delays are in 100 MHz clock cycles (10 ns steps).

```
ADC_DATA[13:0]  (14-bit, 100 MHz)
    │
    ▼ P1 delay  (reg_p1_window, default 1 cycle) ✅ verified 2026-04-10 — Registers.vhd:L216 (to_std_logic_vector(1,32))
    │
    ▼ P2 delay  (reg_p2_window, default 2 cycles) ✅ verified 2026-04-10 — Registers.vhd:L227 (to_std_logic_vector(2,32))
    │
    ▼ M delay   (reg_m_window[9:0], default 200 cycles = 2 µs) ✅ verified 2026-04-10 — Registers.vhd:L176 (to_std_logic_vector(200,32))
    │   X_M  ← pre-event buffer; holds signal before the pulse arrives
    │
    ▼ K0 delay  (lower bits of reg_k_window)
    │   X_M_K0
    │
    ▼ K delay   (upper bits of reg_k_window, default 100 cycles) ✅ verified 2026-04-10 — Registers.vhd:L166 (to_std_logic_vector(100,32))
    │   X_M_K0_K
    │
    ▼ D delay   (reg_d_window[6:0], default 10 cycles) ✅ verified 2026-04-10 — Registers.vhd:L156 (to_std_logic_vector(10,32))
    │   X_M_K0_K_D
    │
    ▼ D3 delay  (reg_d3_window[6:0], default 23 cycles) ✅ verified 2026-04-06 — Registers.vhd:L186 (to_std_logic_vector(23,32))
    │   X_M_K0_K_D_D3  ← used for baseline tracking input
    │
    ▼ TRIPLE_FILTER  (triple_filter.vhd)
    │   Cascaded moving-average filter: 3× (1-2-1) stages
    │   Smooths the signal for cleaner threshold comparison
    │   Produces two taps: PROMPT (at K0) and DELAYED (at K0+K)
    │
    ▼ Baseline subtraction
        FILTERED_SIGNAL − BASELINE_VALUE  →  discriminator inputs
```

**Triple filter:** Each stage is a (1-2-1) moving average. Three cascaded stages produce an effective kernel of [1,8,28,56,70,56,28,8,1] / 256, reducing high-frequency noise without significantly broadening the pulse. ✅ verified 2026-04-18 — `FPGA/DIG/Sims/Filter/Source/triple_filter.vhd` header comments confirm the Pascal's triangle derivation (1-2-1 cycled 4 times → normalization factor 256); `single_filter.vhd` implements the kernel directly using a 9-sample pipeline + two MULT18X18 blocks (coefficients 28 and 70 visible at lines `B=>"000000000000011100"` and `B=>"000000000001000110"`). Note: `triple_filter` applies 3 parallel `single_filter` instances to three different input taps (X_M, X_M_K, X_M_K_D), not a serial cascade; each output is one filtered tap for the discriminator.

**Baseline tracker** (`baseline_tracker.vhd`): Estimates the DC baseline by accumulating a running difference `X(n) − X(n−T)` over a 1024-sample (10.24 µs) window. ✅ verified 2026-04-10 — jta_channel.vhd:L1393 ("Accumulate 1024 samples") + L1924 ("10.24 usec prior to the pre-rise sum") It holds off updates for a programmable time after every discriminator fire (`reg_baseline_delay`) to avoid pulling the baseline onto the pulse tail.

---

### LED Mode — Leading-Edge Threshold Discriminator

In LED mode (`CFD_MODE = '0'`), the discriminator fires as soon as the filtered, baseline-subtracted signal crosses a fixed threshold. This gives a coarse timestamp tied to the signal's leading edge.

```
THRESH_DISC_PROMPT  = triple_filter output at tap X_M_K0_K  (earlier)
THRESH_DISC_DELAYED = triple_filter output at tap X_M_K0_K_D (D cycles later)

Both taps − BASELINE_VALUE
    │
    ▼ thresh_disc.vhd
    Compare THRESH_DISC_DELAYED > DISCRIMINATOR_THRESHOLD
    ─── AND ───
    Compare THRESH_DISC_PROMPT  > DISCRIMINATOR_THRESHOLD
    │
    ▼
THRESH_DISC_FLAG  (one-shot pulse)
    │
    └─→ Opens PEQ entry, starts energy integration, latches 48-bit timestamp ✅ verified 2026-04-21 — `jta_channel.vhd:L49` (20211118): `TIMESTAMP : in std_logic_vector(47 downto 0)`
```

The two-tap comparison (PROMPT and DELAYED both above threshold) acts as a simple coincidence filter that suppresses single-sample noise spikes. The threshold value is set by `reg_led_threshold`.

**Timing:** The discriminator flag is asserted exactly **5 clock cycles (50 ns)** after the signal crosses threshold. ✅ verified 2026-04-14 — `thresh_disc.vhd:L259-260` (MBO comment 2014-09-12: "5 clocks from input to disc flag"; one pipeline delay added to PROMPT_INPUT to match CFD discriminator timing).

---

### CFD Mode — Constant Fraction Discriminator

In CFD mode (`CFD_MODE = '1'`), the discriminator fires at the zero-crossing of `(fraction × prompt_signal) − delayed_signal`. Because the zero-crossing position on the pulse shape is independent of amplitude, CFD gives significantly better timing resolution than LED for pulses of varying heights.

```
CFD_PROMPT  = triple_filter tap at X_M_K0_K    (same as LED prompt)
CFD_DELAYED = triple_filter tap at X_M_K0_K_D  (D cycles later)

Step 1 — Pre-trigger (thresh_disc.vhd fires first as a gate):
    THRESH_DISC_FLAG fires on leading edge (same as LED but used only as a gate)
    → triggers CFD_SAMPLE_ZERO: latches LOCAL_ZERO = current CFD_SUBTRACTION value
    → after K cycles, asserts CFD_PRE_TRIGGER

Step 2 — Fraction multiply (MULT17×17, 34-bit result):
    FRACTIONAL_PROMPT = CFD_PROMPT × CFD_FRACTION >> 13
    (CFD_FRACTION register encodes the fraction as N/8192;
     e.g. reg_cfd_fraction = 0x0C00 ≈ 75% of full scale)

Step 3 — CFD subtraction:
    CFD_SUBTRACTION = FRACTIONAL_PROMPT − CFD_DELAYED

Step 4 — Zero-crossing detection (cfd_disc.vhd):
    LOCAL_DIFFERENCE = CFD_SUBTRACTION − LOCAL_ZERO
    Track sign of LOCAL_DIFFERENCE each clock cycle
    When sign flips → CFD_DISC_FLAG asserted, 48-bit timestamp latched
    Three CFD_SAMPLES captured around the crossing for interpolation
```

The zero-crossing tracks the point on the pulse where `fraction × amplitude = delayed_amplitude`, which moves in time but not in amplitude — giving the amplitude-independent timestamp.

**Key difference from LED:** The timestamp latched in CFD mode is the zero-crossing time, not the threshold-crossing time. This typically improves coincidence timing resolution from ~10 ns (LED) to ~1.7–2.5 ns (CFD) for germanium detectors. ✅ verified 2026-04-14 — `DIG_firmware_expert.md:L100` ("~1.7 ns (1σ) for large signals, ~2.5 ns for small signals at 800–1000 ns rise time" — from ANL Digitizer Firmware for Experts PDF)

---

### Mode Selection

| Register | Address (Ch 0) | Bit | Function |
|----------|---------------|-----|----------|
| `reg_channel_control` | `0x040` | `CFD_MODE` | `0` = LED, `1` = CFD |

Channels 1–9 use addresses `0x044` through `0x064` (4-byte spacing). The `CFD_MODE` bit is distributed as four copies (`xCFD_MODE[3:0]`, with KEEP attribute) inside `jta_channel.vhd` to avoid long-path timing issues.

---

### After Discrimination — PEQ and Energy Integration

When a discriminator fires (LED or CFD), the channel opens a slot in the **Pending Event Queue (PEQ)** — a 16-deep FIFO. ✅ verified 2026-04-21 — `pehq.vhd` (20211118 tag): address counter `a` is `std_logic_vector(3 downto 0)` (4 bits → 16 entries); SRL_DELAY_256x16 + SRL_DELAY_68x16 blocks indexed by `a` confirm 16-entry SRL-based FIFO. The event remains pending until the trigger decision arrives from the Router (~2–4 µs later, within the ~20 µs TRIG_DELAY window). During that time, three energy sums are accumulated:

```
Discriminator fire
    │
    ├─ Latch 48-bit timestamp
    ├─ Open PEQ entry
    │
    ├─→ PRE_RISE integration
    │     Accumulates M cycles of samples before the pulse peak
    │     Duration: reg_m_window[9:0] clock cycles
    │
    ├─→ POST_RISE integration
    │     Accumulates samples from peak onwards
    │     Starts at PEAK_FLAG (peak-finding algorithm in thresh_disc.vhd)
    │
    └─→ P2 integration  (tail sum)
          Accumulates after POST_RISE for additional baseline/tail correction
          Duration: reg_p2_window[9:0] clock cycles

Trigger decision arrives (~2–4 µs later):
    Accepted → pack (timestamp + PRE_RISE + POST_RISE + P2 + pileup flags)
               into 36-bit external FIFO for VME readout
    Rejected → discard PEQ entry silently
```

In CFD mode with `CFD_ESUM_MODE = '1'`, the energy integration start is deferred to `THRESH_DISC_FLAG_DELAYED` (the LED crossing) rather than the CFD zero-crossing, so energy always integrates the same portion of the pulse regardless of discriminator mode.

---

### Trigger Rondel — chan_trigger_control.vhd (PEQ State Machine)

_Source: `FPGA/DIG/MAIN_FPGA/BuildBranches/DGS/Source/chan_trigger_control.vhd` (1,191 lines). Entity: `trigger_rondel`. Fully read 2026-04-24._ ✅ verified 2026-04-24 — `chan_trigger_control.vhd` (1,191 lines, DGS branch)

The **trigger rondel** is the per-channel arbiter between the discriminator and the readout machine. It implements a 16-entry **Pending Event Queue (PEQ)** — a circular buffer of timestamps and status bits — managed by a master state machine with five subsidiary machines (Filler, Remover, Searcher, Vetoer, Check). One instance runs per channel (10 total).

#### Purpose and Terminology

When the discriminator fires and is accepted by the pileup logic (`ACCEPTED_HIT`), the event is **not yet accepted for readout**. It is first entered into the PEQ as a **Pending** event. The PEQ holds it while the trigger system evaluates it. The trigger decision from the Router (via TTCL frames 3–10) later arrives as `TRIG_FLAG` + `TRIG_TIMESTAMP` and is used to search the PEQ for a matching event (within the programmed time window). If found, the event is marked **Accepted**; if the event expires before a matching trigger arrives, it is **Rejected**.

#### PEQ Memory Structure

The PEQ is a 16-entry (4-bit pointer) dual-port distributed RAM (Xilinx `RAM16X1D` primitives — not inferred; explicitly instantiated due to ISE inference difficulties with split record vectors). Each entry is 23 bits wide: ✅ verified 2026-04-24 — `chan_trigger_control.vhd:L449-482` (DIST_RAM_TIMESTAMP/FLAGS/VETO_FLAG/TRIGGER_TYPE generate loops)

| Bits | Field | Description |
|------|-------|-------------|
| [22:20] | `TRIGGER_TYPE[2:0]` | Trigger type from the TTCL frame that accepted the event |
| [19] | `VETO_FLAG` | Set if the event was vetoed (may still be accepted if `VETO_ENABLE='0'`) |
| [18] | `PENDING_FLAG` | `'1'` = decision not yet made; `'0'` = decision made |
| [17] | `ACCEPT_FLAG` | `'1'` = accepted; `'0'` = rejected (meaningful only when `PENDING_FLAG='0'`) |
| [16] | `ROMS_FLAG` | ReadOut Machine Signaled: set after accept/reject is passed to the readout machine |
| [15:0] | `TIMESTAMP[15:0]` | Lower 16 bits of the discriminator timestamp (10 ns per count at 100 MHz) |

Four separate write-enable lines (`PEQ_WE[3:0]`) allow selective field updates without re-writing the full entry. ✅ verified 2026-04-24 — `chan_trigger_control.vhd:L291-298` (`PEQ_WE[3]`=TRIGGER_TYPE, `[2]`=VETO_FLAG, `[1]`=FLAGS (PENDING/ACCEPT/ROMS), `[0]`=TIMESTAMP)

Pointers: `PEQ_BOTTOM` (oldest entry, Remover advances this) and `PEQ_TOP` (next write slot, Filler advances this). Queue is full when `PEQ_TOP + 1 = PEQ_BOTTOM` (15 max entries, not 16). Throttle (`RONDEL_THROTTLE`) is asserted when `PEQ_TOP - PEQ_BOTTOM = 15` (full) or when the machine is in ERROR. ✅ verified 2026-04-24 — `chan_trigger_control.vhd:L395-408`

A separate **Trigger Accept FIFO** (`fifo_20x17_sepclk_fwft`, 17 entries, 20 bits: 16-bit TS + 3-bit type + 1 spare) queues incoming TTCL trigger messages. The Searcher reads from this FIFO when the PEQ is non-empty. ✅ verified 2026-04-24 — `chan_trigger_control.vhd:L586-604`

#### Master State Machine — Priority and States

The master FSM runs at 100 MHz. It has 8 states with strict priority ordering in IDLE: ✅ verified 2026-04-24 — `chan_trigger_control.vhd:L753-780`

| Priority | State | Triggered by | Action |
|----------|-------|-------------|--------|
| 0 | `ERROR` | `ERROR_FLAG` | Permanently throttles channel; escape only via reset or `PEQ_BYPASS` |
| 1 | `FILL` | `FILL_FLAG` (on `ACCEPTED_HIT`) | Add new event to PEQ top |
| 2 | `REMOVE` | `REMOVE_FLAG` (on `EVENT_EXPIRED`) | Remove expired event from PEQ bottom |
| 3 | `VETO` | `VETO_FLAG` (on `VETO_LAST_EVENT`) | Attempt to veto the most recent PEQ entry |
| 4 | `SEARCH` | `SEARCH_FLAG` (Trigger Accept FIFO non-empty) | Search all PEQ entries for timestamp match |
| — | `BYPASS` | `PEQ_BYPASS='1'` | Bypass mode: `ACCEPTED_HIT` → `EVENT_ACCEPT` directly |
| — | `CHECK_FOR_NON_PENDING_EVENTS` | Called after REMOVE/SEARCH/VETO complete | Signal consecutive resolved entries to readout machine |

**Important:** FILL and REMOVE are higher priority than SEARCH. This guarantees the PEQ never overflows and events never expire silently while a slow search is in progress.

#### Subsidiary Machines

**FILLER (`FILL` state):** 2 states: `START` (check capacity, ack flag), `ADD_EVENT` (write timestamp + `PENDING_FLAG='1'` + zeroed other fields to `PEQ_TOP`; increment `PEQ_TOP`). If `EVENT_VALID='0'` at fill time, the event is written with `PENDING_FLAG='0'` (immediately rejected) and transitions to `CHECK_FOR_NON_PENDING_EVENTS` to signal the reject. ✅ verified 2026-04-24 — `chan_trigger_control.vhd:L865-931`

**REMOVER (`REMOVE` state):** 2 states: `START` (ack flag, advance `PEQ_BOTTOM`, read current bottom entry), `REMOVE_EVENT` (overwrite entry with zeros; if `PENDING_FLAG` was still `'1'` → assert `EVENT_REJECT` immediately). Transitions to `CHECK_FOR_NON_PENDING_EVENTS` unless the queue is empty or the removed event was already decided. ✅ verified 2026-04-24 — `chan_trigger_control.vhd:L932-985`

**SEARCHER (`SEARCH` state):** 8 states. Starts at `PEQ_BOTTOM` and walks entries. Because the timestamp comparison pipeline has 4 pipeline stages, the Searcher uses 4 `SEARCH_PIPE_WAIT_*` states before entering `SEARCH_EVENT`. In `SEARCH_EVENT`, the comparison result (`TS_COMPARISON_OUTPUT`) is checked against the 4-cycle-delayed PEQ entry (`xxxxSEARCH_PIPED_RD_DATA`). If `PENDING_FLAG='1'` and the comparison passes: mark `ACCEPT_FLAG='1'`, `PENDING_FLAG='0'`, and OR the trigger type into the entry's `TRIGGER_TYPE` field (the OR allows multiple trigger types to accumulate — added 2025-05-30). Consumes one Trigger Accept FIFO entry per search pass. Transitions to `CHECK_FOR_NON_PENDING_EVENTS` after completing the walk. ✅ verified 2026-04-24 — `chan_trigger_control.vhd:L988-1102`

**Timestamp comparison (4-stage pipeline):** ✅ verified 2026-04-24 — `chan_trigger_control.vhd:L606-634`
```
Stage 1: Latch EVENT_TS and TRIG_TS
Stage 2: DIFFERENCE = EVENT_TS − TRIG_TS  (16-bit unsigned subtraction)
Stage 3: OFFSET_DIFFERENCE = {NOT DIFFERENCE[15]} & DIFFERENCE[14:0]  (sign-flip MSB for unsigned window comparison)
Stage 4: MATCH if OFFSET_DIFFERENCE < TS_COMP_UPPER_LIMIT_OFFSET and > TS_COMP_LOWER_LIMIT_OFFSET
         (both limits also sign-flipped: TS_OFFSET_UPPER/LOWER_BOUND)
```
The sign-flip trick converts the signed window comparison into an unsigned magnitude comparison, centering the window at the zero-difference point.

**VETOER (`VETO` state):** 3 states: `START` (ack flag, address `PEQ_TOP − 1`), `VETO_EVENT` (if `ROMS_FLAG='0'`: mark `VETO_FLAG='1'`; if `VETO_ENABLE='1'` also clear `PENDING` and `ACCEPT` flags so the event is rejected; if `ROMS_FLAG='1'`: event already passed to readout — too late, abort), `VETO_WAIT` (one-cycle settle, then → `CHECK_FOR_NON_PENDING_EVENTS`). ✅ verified 2026-04-24 — `chan_trigger_control.vhd:L1104-1165`

**CHECK_FOR_NON_PENDING_EVENTS:** Walks from `PEQ_BOTTOM` (or current `PEQ_RD_ADDR`) upward. For each entry where `PENDING_FLAG='0'` and `ROMS_FLAG='0'`: assert `EVENT_ACCEPT` or `EVENT_REJECT` (also `EVENT_VETO` if `VETO_FLAG='1'`) and set `ROMS_FLAG='1'` to prevent re-signaling. Stops when it hits a still-pending entry or reaches `PEQ_TOP − 1`. This ensures decisions are delivered to the readout machine in chronological order. ✅ verified 2026-04-24 — `chan_trigger_control.vhd:L834-862`

#### Veto Gating

A 3-state `VETO_GATE` FSM (`ST_IDLE / ST_HIT / ST_VETOABLE`) with a 12-bit countdown counter (`VETO_COUNT`) controls when a cross-channel veto may be accepted. After each `ACCEPTED_HIT`, the counter loads `REG_VETO_GATE_WIDTH[11:0]` (default=255 → 2.55 µs; max=4095 → 40.95 µs) and starts counting down. A veto request (`VETO_LAST_EVENT`) is only accepted while `MAY_ACCEPT_VETO_REQUEST='1'` (i.e., within the countdown window). If `ENABLE_VETO_GATING='0'`, all vetoes are accepted unconditionally. This prevents stale cross-channel vetoes from incorrectly rejecting events. ✅ verified 2026-04-24 — `chan_trigger_control.vhd:L636-698`

#### Throttle Logic

- `RONDEL_THROTTLE`: asserted when PEQ is full (15 entries) or in ERROR state. Blocks `ACCEPTED_HIT` and `CHANNEL_THROTTLE` output. ✅ verified 2026-04-24 — `chan_trigger_control.vhd:L395-408`
- `LATCHED_THROTTLE`: controls whether `EXTENDED_EVENT_OUT` is passed through. A pileup train's extended events are allowed to continue until a new `ACCEPTED_HIT` is seen, at which point the latch releases. This prevents partial pileup train readout. ✅ verified 2026-04-24 — `chan_trigger_control.vhd:L373-394`

#### Outputs to Readout Machine

| Signal | Width | Description |
|--------|-------|-------------|
| `EVENT_ACCEPT` | 1 | One-clock pulse: event accepted for readout |
| `EVENT_REJECT` | 1 | One-clock pulse: event rejected |
| `EVENT_VETO` | 1 | One-clock pulse: event was vetoed (may accompany accept if `VETO_ENABLE='0'`) |
| `EVENT_TYPE[2:0]` | 3 | Trigger type from TTCL (OR of all matching trigger slots) |
| `ACCEPTED_EVENT_COUNT[31:0]` | 32 | Rate or accumulate counter of accepted events (`sync_capture_counter`) |
| `ACCEPTED_HIT_OUT` | 1 | Buffered pass-through of `ACCEPTED_HIT` (after throttle gating) |
| `EXTENDED_EVENT_OUT` | 1 | Buffered pass-through of pileup extension events |
| `CHANNEL_THROTTLE` | 1 | Backpressure to upstream discriminator |
| `DIAG_REG[31:0]` | 32 | Diagnostic: PEQ pointers, FSM state, flag bits |

#### BYPASS Mode

When `PEQ_BYPASS='1'` (register-controlled), the rondel enters `BYPASS` state and passes every `ACCEPTED_HIT` directly to `EVENT_ACCEPT` without PEQ involvement. All IRQ acks are driven continuously. This is the **internal trigger / accept-all** mode. ✅ verified 2026-04-24 — `chan_trigger_control.vhd:L796-833`

---

### Pileup Detection

The **pileup processor** (`pileup_processor.vhd`) tracks how many events are in-flight (discriminator fired but not yet readout-complete). It uses a 4-bit counter (`PILEUP_COUNT[3:0]`) and an 8-state machine: ✅ verified 2026-04-21 — `pileup_processor.vhd:L50,L54` (20230809 tag): 4-bit `PILEUP_COUNT`; 8 states: `CHECK_DISABLE, REJ_NO_HIT, REJ_ONE_HIT, REJ_MANY_HIT, OVERFLOW, ACC_NO_HIT, ACC_ONE_HIT, ACC_MANY_HIT`

```
States (actual VHDL names):
    CHECK_DISABLE                  ← entry point; checks PILEUP_DISABLE register
    REJ_NO_HIT / ACC_NO_HIT       ← idle (no in-flight events)
    REJ_ONE_HIT / ACC_ONE_HIT     ← one event in-flight
    REJ_MANY_HIT / ACC_MANY_HIT   ← multiple events in-flight
    OVERFLOW                       ← counter saturated (fatal error)

Counter increments: on each THRESH_DISC_FLAG
Counter decrements: on PILE_RELEASE_DLYD (end-of-event holdoff pulse)

PILEUP_DISABLE register:
    0 → reject second and subsequent pileup hits (standard spectroscopy) → REJ_* states
    1 → accept all hits (pileup recording mode) → ACC_* states

Outputs per event:
    ACCEPTED_HIT   — first hit (or any hit in accept-all mode); blocked if PU_TOO_SHORT
    EXTENDED_EVENT — subsequent pileup hits (accept-all mode only); blocked if PU_TOO_SHORT
    PILEUP_FLAG    — level: counter > 0 (rejection states only)
    OVERFLOW_FLAG  — counter saturated at 15 (4-bit overflow)
    PU_TOO_SHORT   — pileup interval shorter than retrigger holdoff; blocks ACCEPTED_HIT + EXTENDED_EVENT
    MARK_EXTENDED_AS_ACCEPTED — if set, all pileup hits become ACCEPTED_HIT (no EXTENDED_EVENT)
```

The holdoff time (`reg_holdoff_control[8:0]`) controls both the minimum inter-event spacing and the peak-finding window; it is shared between the threshold discriminator and the pileup counter.

---

### VME Registers for Discriminator Configuration

All addresses are per-channel. Channel 0 uses the base address shown; channels 1–9 add `4 × channel_number`.

**Discriminator mode and thresholds:**

| Register | Ch 0 Address | Bits | Description |
|----------|-------------|------|-------------|
| `reg_channel_control` | `0x040` | `CFD_MODE` bit | `0` = LED, `1` = CFD ✅ verified 2026-04-10 — `asynDigParams.c:L459` (`setAddress(reg_channel_control0,0x0040)`) |
| `reg_led_threshold` | `0x080` | `[13:0]` | Threshold in ADC counts (both LED and CFD pre-gate) ✅ verified 2026-04-10 — `asynDigParams.c:L469` (`setAddress(reg_led_threshold0,0x0080)`) |
| `reg_cfd_fraction` | `0x0C0` | `[12:0]` | CFD fraction encoded as N/8192 (e.g. `0x0C00` ≈ 75%) ✅ verified 2026-04-16 — `asynDigParams.c:L479` (`setAddress(reg_CFD_fraction0,0x00C0)`) |
| `reg_external_disc_mode` | `0x420` | 2 bits/ch | `00`=normal, `01`=OR with external, `10`=AND, `11`=external only ✅ verified 2026-04-19 — `Registers.vhd:L234` (`X"420"`, `reg_external_disc_mode`) |

**Delay chain:**

| Register | Ch 0 Address | Bits | Description |
|----------|-------------|------|-------------|
| `reg_p1_window` | `0x300` | `[3:0]` | P1 delay (cycles) ✅ verified 2026-04-19 — `Registers.vhd:L216` (`X"300"`, `reg_p1_window(0)`) |
| `reg_p2_window` | `0x404` | `[9:0]` | P2 delay and tail-sum window (cycles, global to board) ✅ verified 2026-04-19 — `Registers.vhd:L227` (`X"404"`, `reg_p2_window`) |
| `reg_m_window` | `0x200` | `[9:0]` | M delay = pre-event buffer depth (cycles) ✅ verified 2026-04-10 — `asynDigParams.c:L529` (`setAddress(reg_m_window0,0x0200)`) |
| `reg_k_window` | `0x1C0` | `[13:0]` | K0+K combined delay (cycles) ✅ verified 2026-04-11 — `asynDigParams.c:L519` (`setAddress(reg_k_window0,0x01C0)`) |
| `reg_d_window` | `0x180` | `[6:0]` | D delay — sets CFD fraction delay (cycles) ✅ verified 2026-04-16 — `asynDigParams.c:L509` (`setAddress(reg_d_window0,0x0180)`) |
| `reg_d3_window` | `0x240` | `[6:0]` | D3 delay — baseline tracker input offset (cycles) ✅ verified 2026-04-06 — `Registers.vhd:L186` (`to_std_logic_vector(23,32)`) |

**Baseline tracking:**

| Register | Ch 0 Address | Bits | Description |
|----------|-------------|------|-------------|
| `reg_baseline_start` | `0x2C0` | `[13:0]` | Initial baseline value (ADC counts; default = 1000) ✅ verified 2026-04-19 — `Registers.vhd:L206` (`X"2C0"`, `to_std_logic_vector(1000,32)`) |
| `reg_baseline_delay` | `0x418` | `[7:0]` | Holdoff after disc fire before resuming tracking (× 10.24 µs) ✅ verified 2026-04-19 — `Registers.vhd:L232` (`X"418"`, `reg_baseline_delay`) |
| `reg_baseline_delay` | `0x418` | `[10:8]` | Baseline update step size (tracking speed) ✅ verified 2026-04-19 — `Registers.vhd:L232` (comment: "bits 10:8 are speed, bits 7:0 are delay") |

**Holdoff and peak finding:**

| Register | Address | Bits | Description |
|----------|---------|------|-------------|
| `reg_holdoff_control` | `0x414` | `[8:0]` | Retrigger holdoff duration (cycles × 10 ns; default 150 cycles = 1.5 µs) ✅ verified 2026-04-19 — `Registers.vhd:L231` (`X"414"`, default `2198` = 0x896 → ho[8:0]=150, pksens[11:9]=4) |
| `reg_holdoff_control` | `0x414` | `[11:9]` | Peak sensitivity (controls peak-finding rate; default 4) ✅ verified 2026-04-19 — `Registers.vhd:L231` (comment: "pksens(11:9)=4, ho(8:0)=150") |
| `reg_disc_width` | `0x280` | `[7:0]` | Discriminator output pulse width (cycles) ✅ verified 2026-04-19 — `Registers.vhd:L196` (`X"280"`, `reg_disc_width(0)`) |

**Waveform capture:**

| Register | Ch 0 Address | Bits | Description |
|----------|-------------|------|-------------|
| `reg_raw_data_delay` | `0x100` | `[9:0]` | Pipeline delay before raw data capture (pre-trigger samples) ✅ verified 2026-04-21 — `vxworks/dgsDrivers/dgsDriverApp/src/asynDigParams.c:L489` (`setAddress(reg_raw_data_delay0,0x0100)`) |
| `reg_raw_data_length` | `0x140` | `[9:0]` | Number of waveform samples to capture ✅ verified 2026-04-21 — `vxworks/dgsDrivers/dgsDriverApp/src/asynDigParams.c:L499` (`setAddress(reg_raw_data_length0,0x0140)`) |

> ⚠️ Note: The VHDL source (`DGS_TAG_20180607_TWEAK/Registers.vhd:L136,L146`) labels `0x100` as `reg_raw_data_length` and `0x140` as `reg_raw_data_window`. The older SVN driver (`DGS_SVN/dgs/20180921/asynDigParams.c:L688,698`) also used those VHDL names. The **current live driver** (`vxworks/dgsDrivers/dgsDriverApp/src/asynDigParams.c`) renamed `0x100` → `reg_raw_data_delay` and `0x140` → `reg_raw_data_length`; `reg_raw_data_window` no longer exists in the current IOC. ✅ verified 2026-04-21 — cross-checked VHDL, SVN driver, and live driver.

**Diagnostic counters (read-only):**

| Register | Ch 0 Address | Description |
|----------|-------------|-------------|
| `reg_disc_count` | `0x7C0` | Total discriminator fires ✅ verified 2026-04-19 — `Registers.vhd:L287` |
| `reg_accepted_event_count` | `0x740` | Events accepted by trigger ✅ verified 2026-04-19 — `Registers.vhd:L267` |
| `reg_dropped_event_count` | `0x700` | Events dropped (FIFO full or vetoed) ✅ verified 2026-04-19 — `Registers.vhd:L257` |
| `reg_ahit_count` | `0x780` | Accepted-hit pulses from pileup processor ✅ verified 2026-04-19 — `Registers.vhd:L277` |

---

## VME FPGA

**Location:** `VME_FPGA_ANL/`

| Field | Value |
|-------|-------|
| Part | xc3s400 (Spartan-3) |
| Package | fg320 |
| Speed Grade | -5 |
| Tool | ISE 13.4 |
| Project | `VME_FPGA_ANL/Work11/vme_A32_D32.xise` |
| Top Entity | `vme_top` |

Same architecture as the MTRG VME FPGA: acts as A32/D32 VME slave, programs the main FPGA from external flash, and bridges host VME commands to the main FPGA.

### Source Files
| File | Description |
|------|-------------|
| `TOP.VHD` | Top-level entity (`vme_top`) |
| `vme_addr_decode.vhd` | VME address space decoder |
| `external_bus_controller.vhd` | Flash/FPGA bus multiplexer |
| `configuration_controller.vhd` | FPGA programming sequencer |
| `register_block.vhd` | Status and control registers |
| `register_block_FlashHi.vhd` | Upper flash address register block |

### Bitfiles
| File | Description |
|------|-------------|
| `Work11/vme_top.bit` | Standard VME FPGA bitfile |
| `Work11/vme_top_usehi.bit` | Variant using upper flash address |
| `Work11/20230928.mcs` | MCS flash image (Sept 2023) |
| `Work11/20230928_usehi.mcs` | MCS flash image, upper flash variant |

### Clock Select Register (`clk_select`)

**VME address:** `0x0910` bits[1:0] in `register_block.vhd`

Controls which clock the digitizer uses as its system clock. The two bits (`sysclk_sel[1:0]`) drive physical output pins `sysclk_sel0_out` (B9) and `sysclk_sel1_out` (B10) off-chip to a hardware clock mux on the digitizer PCB.

**Important:** the register bits are **inverted** before the output pins (`sysclk_sel0_out <= NOT sysclk_sel0`) to match the original LBL digitizer board design. EPICS values are correct end-to-end — the inversion is transparent to software.

Default at reset: `sysclk_sel0=1, sysclk_sel1=0` → OSC (local oscillator). ✅ verified 2026-04-17 — `VME_FPGA_ANL/Source/register_block.vhd:L165-166` (`sysclk_sel0 <= '1'; sysclk_sel1 <= '0'; -- initialize to use internal clock`)

| `clk_select` EPICS value | `sel[1:0]` | Meaning |
|---|---|---|
| 0 | `00` | S/D — SERDES derived (link clock from Router) |
| 1 | `01` | **OSC** — local on-board oscillator (default) |
| 2 | `10` | S/D — same as 0 |
| 3 | `11` | AUX — auxiliary clock input |

✅ verified 2026-04-17 — `MDigUserVME.template` (`clk_select` mbbo: ZRST=S/D, ONST=OSC, TWST=S/D, THST=AUX); `register_block.vhd:L351-352` (write to 0x0910 sets sysclk_sel[1:0] from VME_data_in[1:0])

**EPICS PV:** `VME$(CRATE):$(BOARD):clk_select` (mbbo, `MDigUserVME.template` / `SDigUserVME.template`)

**Usage in `link_sys.py`:**
- Stage 4A: `clk_select=1` (OSC) — initialize DIGs on independent local clock first ✅ verified 2026-04-18 — `ANLDAQ/gui/link_sys.py:L526` (`self.SetPVManually(dig_name, "clk_select", 1)`)
- Stage 4E: `clk_select=dig_clk_sel` (user-configured, normally 0=S/D) — switch DIGs to Router-derived link clock for full timestamp sync ✅ verified 2026-04-18 — `ANLDAQ/gui/link_sys.py:L581` (`self.SetPVManually(dig_name, "clk_select", dig_clk_sel)`)

## Main FPGA Bitfiles

| Branch | Bitfile | Description |
|--------|---------|-------------|
| DGS | `DGS/Work/BUS_LEFT.bit` | Production — front bus sender role |
| DGS | `DGS/Work/BUS_RIGHT.bit` | Production — front bus receiver role |
| Majorana | `Majorana/Work/digitizer.bit` | MAJORANA experiment variant |
| DGS_QUAD_M_SUMS | `DGS_QUAD_M_SUMS/Work/FB_SENDER.bit` / `FB_RCVR.bit` | Quad M-sum, sender/receiver |
| SumOverRise | `SumOverRise/Work/FB_SENDER.bit` / `FB_RCVR.bit` | Sum-over-rise, sender/receiver |
| DGSBramTest | `DGSBramTest/Work/MSTR_digitizer.bit` | BRAM test |
| — | `tag_4975_mod_fifo_digitizer.bit` (root) | Tagged release build |
| — | `Walter_Release_MDIG_6194/MSTR_digitizer-6194.bin` | Release candidate v6194 |

Note: DGS branch produces two bitfiles (`BUS_LEFT` / `BUS_RIGHT`) because the `FRONT_BUS_LEFT` generic changes the front bus direction — two digitizer modules are paired, one sender and one receiver.

## IP Cores

Located in each branch's `Cores/` directory:

| Core | Description |
|------|-------------|
| `chipscope_icon` | ChipScope controller |
| `chipscope_ila` | ChipScope logic analyzer |
| `fifo_16x1023_async` | 16-bit, 1K deep async FIFO |
| `fifo_16x64K_async` | 16-bit, 64K deep async FIFO |
| `BRAM_1024X16_REGSHADOW` | Block RAM register shadow |

---

## See Also

- `knowledgeBase/deep_fpga_DIG.md` — DIG firmware overview: Spartan-3 architecture, ADC pipeline, event packet format, master/slave config, FIFO readout (this file is a continuation of that)
- `knowledgeBase/deep_fpga_DIG_modules.md` — DIG selected module analysis: `SERDES_TX_Mach_DGS.vhd` (disc packer), `event_packer.vhd` (accordion FIFO), `pileup_processor.vhd` (8-state FSM), `SERDES_RX_Mach.vhd` (20-frame Router command receiver)
- `knowledgeBase/fpga.md` — System-level overview: trigger hierarchy, signal flow, PEQ explanation, end-to-end timeline
- `knowledgeBase/DIG_firmware_expert.md` — Operator-level guide: all 8 readout modes, register summary, discriminator config
- `knowledgeBase/deep_fpga_RTRG.md` — Router firmware: multiplicity aggregation, throttle, VME register map
- `knowledgeBase/deep_fpga_MTRG_MAIN.md` — Master trigger firmware: trigger algorithms, TAC-II TDC, 20-frame commands
- `knowledgeBase/ttcl.md` — TTCL spec: full frame-by-frame breakdown of the 20-frame command structure DIG receives
- `knowledgeBase/ANLDAQ.md` — DAQ software: `class_DIG.h` decodes DIG packet format documented here
- `knowledgeBase/connectors.md` — DIG connector pinouts: RJ45 SERDES, 36-pin Aux I/O, RTRG IEC cable
- `knowledgeBase/deep_fpga_building.md` — Build toolchain: ISE 14.7 on Ubuntu 24.04, Docker/Podman approach
- `knowledgeBase/preamp_reset_readme.md` — Detailed explanation of preamp reset (PRK) detection logic, blanking timing, BGO veto gate, and PARST timestamp fields

---
*Source: `DGS_tools_pack/raw_FPGA/Dig*/` — VHDL source. PDF: `ANL Digitizer Firmware for Experts.pdf`. Created: 2026-04-05.*
