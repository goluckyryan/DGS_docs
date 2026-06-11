# DGS FPGA Firmware

Stability: C3 - Structural / stable

This repository contains FPGA firmware for the DGS (Digital Gamma-ray Spectrometer) trigger and readout system, developed at Argonne National Laboratory. The system digitizes signals from germanium gamma-ray detectors and makes real-time trigger decisions to select physics events of interest.

> **Two firmware trees exist:** `fpga/` (active development, more recent) and `FPGA/` (raw upstream/unmodified reference). Architecture is identical; `FPGA/` was last modified a few days before `fpga/`. Use `fpga/` for building production firmware; use `FPGA/` for comparison against unmodified upstream.

## Table of Contents

- [System Overview](#system-overview)
- [Module Summary](#module-summary)
- [System Architecture](#system-architecture)
  - [Signal Flow](#signal-flow)
  - [Trigger Cycle](#trigger-cycle)
  - [SERDES Link Summary](#serdes-link-summary)
  - [Data Path vs. Trigger Path](#data-path-vs-trigger-path)
  - [Throttle Mechanism](#throttle-mechanism)
- [How Triggering Works](#how-triggering-works)
  - [Step 1 — Detectors Fire: DIG Discriminates](#step-1--detectors-fire-dig-discriminates)
  - [Step 2 — Router Aggregates: RTRG Counts Multiplicity](#step-2--router-aggregates-rtrg-counts-multiplicity)
  - [Step 3 — Master Trigger Decides: MTRG Runs Algorithms](#step-3--master-trigger-decides-mtrg-runs-algorithms)
  - [Step 4 — Decision Propagates Back Down](#step-4--decision-propagates-back-down)
  - [End-to-End Timeline](#end-to-end-timeline)
- [Module Documentation](#module-documentation)
- [Timing](#timing)
- [VME Control Hierarchy](#vme-control-hierarchy)
- [Building the Firmware](#building-the-firmware)
- [Related Repositories](#related-repositories)

---

## System Overview

The DGS trigger system is organized as a three-tier hierarchy:

```
                        ┌─────────────────────┐
                        │        MTRG          │
                        │   Master Trigger     │
                        │  (Virtex-4 / KU060)  │
                        └────────┬────────────┘
                                 │  up to 8 Links (L)
              ┌──────────────────┼─────────────────────┐
              │                  │                      │
       ┌──────┴──────┐    ┌──────┴──────┐      ┌──────┴──────┐
       │    RTRG      │    │    RTRG      │  ... │    RTRG      │
       │    Router    │    │    Router    │      │    Router    │
       │  (Virtex-4)  │    │  (Virtex-4)  │      │  (Virtex-4)  │
       └──────┬───────┘    └──────┬───────┘      └──────┬───────┘
              │ up to 8 Links (A–H)                      │
     ┌────────┼──────────┐                    ┌──────────┼────────┐
     │        │          │                    │          │        │
  ┌──┴──┐  ┌──┴──┐  ┌──┴──┐              ┌──┴──┐  ┌──┴──┐  ┌──┴──┐
  │ DIG │  │ DIG │  │ DIG │  ...         │ DIG │  │ DIG │  │ DIG │
  │(S3) │  │(S3) │  │(S3) │              │(S3) │  │(S3) │  │(S3) │
  └──┬──┘  └──┬──┘  └──┬──┘              └──┬──┘  └──┬──┘  └──┬──┘
     │        │         │                    │         │        │
  10 ch    10 ch     10 ch                10 ch     10 ch    10 ch
  detectors  detectors  detectors         detectors  detectors  detectors
```

**Maximum system scale:** 1 MTRG × 8 RTRG × 8 DIG × 10 channels = **640 detector channels**

*Source: J. Anderson, "Thoughts on Trigger and Timing Distribution Across Multiple Detectors", FRIB Community Meeting, August 2013*

---

## Module Summary

| Module | Folder | Chip | Tool | Role |
|--------|--------|------|------|------|
| [MTRG](deep_fpga_MTRG.md) | `MTRG/` | Virtex-4 XC4VLX80 (ISE) / Kintex UltraScale XCK060 (Vivado) | ISE 14.7 / Vivado 2018.3 | Central trigger decision-maker ✅ verified 2026-04-17 — `MTRG/Firmware/MAIN_FPGA/trunk/Work13_4/Work13_4.xise:L2` (`ise_version="14.7"`) — folder named Work13_4 is a label, not the version |
| [RTRG](deep_fpga_RTRG.md) | `RTRG/` | Virtex-4 XC4VLX80 | ISE 14.7 | Router — aggregates digitizer hits, forwards trigger commands ✅ verified 2026-04-17 — `RTRG/Firmware/DGS_Version/.../Work13_4.xise:L15` (`ise_version="14.7"`) — folder named Work13_4 is a label, not the version |
| [DIG](deep_fpga_DIG.md) | `DIG/` | Spartan-3 XC3S5000 | ISE 14.7 | 10-channel waveform digitizer ✅ verified 2026-04-06 — `DIG/MAIN_FPGA/BuildBranches/DGS/Work/BUS_LEFT.xise` |

Each module also has a VME FPGA (Spartan-3 XC3S400) that handles VME bus access and programs the main FPGA from flash memory. ✅ verified 2026-04-06 — DIG/VME_FPGA_ANL/Work11/vme_A32_D32.xise (Device=xc3s400, Spartan3)

---

## System Architecture

### Signal Flow

The system operates on a **2 µs trigger cycle** divided into 20 frames. Data flows in two directions simultaneously:

#### Upstream (detector hits → trigger decision):

```
Detector signals
    │  (analog)
    ▼
DIG — ADC (14-bit, 100 MHz) ✅ verified 2026-04-06 — decimator.vhd:adc_data(13 downto 0); LEFT_RIGHT.ucf: CLK50_OSC×2 = 100 MHz logic clock
    │  Per-channel: delay chain → filters → discriminator → hit flag
    │
    │  SERDES Link A–H (18-bit, 50 MHz, DC-balanced) ✅ verified 2026-04-20 — `trigger_data_types.vhd:L42` ("array of eight 18-bit registers ... raw SERDES"); `router_data_path.vhd:L87` ("DIG_LINK_RXs, however these are 18 bits wide"); MTRG BOM Y1=50.000 MHz oscillator
    │  TX word: [SYNC | COARSE_DISC[9:5] | ACCEPTED_HITS[9:0]]
    ▼
RTRG — Aggregates 8 digitizers
    │  X-plane hit count + Y-plane hit count + throttle OR
    │
    │  SERDES Link L (18-bit, 50 MHz) ✅ verified 2026-04-20 — `trigger_data_types.vhd:L42`; `FPGA/README.md:L164` ("All links use 18-bit DC-balanced LVDS SERDES running at 50 MHz")
    │  TX word: [THR | Y-mult[6:0] | VAL | X-mult[6:0] | POL] (16-bit) ✅ verified 2026-04-14 — `router_data_path.vhd:L10-15,L230-233`
    ▼
MTRG — Runs trigger algorithms
    │  8 algorithms (GITMO, MYRIAD, multiplicity, coincidence, etc.)
    │  TDC timestamps events with ~30 ps resolution (vernier, 4-phase 250 MHz, 64-tap chain) ✅ verified 2026-04-07 — tdc_chain_cont.vhd:L28-31 (4 phases assumed 250MHz), L77-80 (64-tap chains); MAIN_FPGA.md:L344
    ▼
  Trigger Decision
```

#### Downstream (trigger decision → digitizers):

```
MTRG — Master state machine
    │  Generates 20-frame command structure every 2 µs
    │  Frames carry: Sync, Trigger decisions, Veto, Aux commands
    │
    │  SERDES Links A–H (one per RTRG)
    ▼
RTRG — Sanitizes and forwards commands
    │  Replaces router-specific frames (12, 14) with null
    │  Injects per-channel veto bits into word 5
    │
    │  SERDES Links A–H (one per DIG)
    ▼
DIG — Receives trigger decisions
    │  Accepts or rejects pending events based on trigger flags
    │  Packs accepted event data (timestamp, energy, waveform)
    ▼
External FIFO → VME readout → Host computer
```

### Trigger Cycle

Each **2 µs cycle** consists of 20 frames (100 ns per frame). Each frame carries **5 words × 16 bits = 80 bits (10 bytes)** of command data at 50 MHz, so one frame spans 5 SERDES clock cycles (100 ns). The upstream path (DIG → Router) sends an **18-bit word every 50 MHz clock cycle** (16 data bits + 2 DC-balance bits); there is no frame packetization in the upstream direction. ✅ verified 2026-04-07 — `FPGA/MTRG/Firmware/MAIN_FPGA/trunk/ss_variant/remote_sources/_/Source/mstr_mach.vhd:L107-108,L237,L303` (CURRENT_FRAME 1–20, CURRENT_WORD 1–5 → 20×5×20ns=2µs)

| Frame | Content |
|-------|---------|
| 1 | Sync — system-wide timestamp synchronization |
| 2 | Detector status |
| 3–10 | Trigger decisions (8 independent trigger algorithm results) |
| 11 | Reserved |
| 12 | Inter-trigger / Router-specific command |
| 13 | Reserved |
| 14 | Remote trigger / Router-specific command |
| 15 | Async command (VME-injected, e.g. calibration, front-end reset) |
| 16 | Synchronous capture command ✅ verified 2026-04-19 — `top.vhd:L582` ("Frame 16 (synchronous front end) commands") |
| 17 | Auxiliary detector command ✅ verified 2026-04-19 — `top.vhd:L597` ("Frame 17 (Auxiliary Detector) commands") |
| 18–20 | End-of-cycle / spare |

For the detailed word-by-word breakdown of each frame:
- **Downstream (MTRG → RTRG → DIG):** [deep_fpga_MTRG_MAIN.md — Command Frame Structure](deep_fpga_MTRG_MAIN.md) · [deep_fpga_DIG.md — SERDES RX Frame Types](deep_fpga_DIG.md)
- **Upstream (DIG → RTRG → MTRG):** [deep_fpga_DIG.md — SERDES TX Format](deep_fpga_DIG.md)

*Source: J. Anderson, FRIB Community Meeting, August 2013*

### SERDES Link Summary

| Link | Between | Direction | Purpose |
|------|---------|-----------|---------|
| A–H (MTRG↔RTRG) | MTRG ↔ RTRG | Bidirectional | Commands downstream, status/lock upstream |
| L (RTRG↔MTRG) | RTRG → MTRG | Upstream | Multiplicity data + throttle to Master Trigger |
| L (MTRG→RTRG) | MTRG → RTRG | Downstream | 20-frame command structure to Router |
| A–H (RTRG↔DIG) | RTRG ↔ DIG | Bidirectional | Commands downstream, discriminator hits upstream |
| L, R, U | MTRG ↔ MTRG | Bidirectional | Inter-trigger chaining (multiple Master Triggers) |

All links use 18-bit DC-balanced LVDS SERDES running at 50 MHz. ✅ verified 2026-04-06 — `FPGA/RTRG/Firmware/DGS_Version/MAIN_FPGA/Source/TOP.VHD:L354,L424,L440` (50 MHz logic/SERDES clocks)

DC-balance encoding (upstream, DIG→RTRG): 18-bit word = bit[0] (inversion flag) + bits[17:1] (16-bit payload, sent true if bit[0]=0, inverted if bit[0]=1). Receiver strips bit[0] and optionally inverts bits[17:1] to recover the 16-bit data word. ✅ verified 2026-04-07 — `FPGA/RTRG/Firmware/DGS_Version/MAIN_FPGA/Source/DCBAL_in.vhd:L12-14,L38-39`

### Data Path vs. Trigger Path

| | Trigger Path | Data Path |
|-|-------------|-----------|
| **What** | Hit flags, multiplicity counts, trigger decisions | Full waveform/energy event data |
| **Speed** | Real-time, every 2 µs cycle | Asynchronous, event-driven |
| **Transport** | SERDES links (MTRG ↔ RTRG ↔ DIG) | External FIFO → VME bus |
| **Latency** | ~2 µs (one trigger cycle) | ~ms (VME DMA readout) |
| **Direction** | Bidirectional | DIG → Host computer only |

### Throttle Mechanism

When a DIG's internal FIFO is near-full, it asserts a throttle request upstream:

```
DIG FIFO almost full
    → DIG asserts throttle on SERDES TX
    → RTRG throttle_limiters filter (must be held for programmable time)
    → RTRG stretches valid throttle to ≥2 µs (monostable: COUNTER_START=400 @ 50 MHz) ✅ verified 2026-04-15 — `throttle_limiters.vhd:L80` (COUNTER_START=400), L341 (reset to COUNTER_START); L24-26 (comment: minimum 2us ensures propagation to Master)
    → RTRG sends throttle bit to MTRG in Link L TX packet
    → MTRG suspends trigger formation while throttle is asserted
```

This prevents event data loss due to FIFO overflow without requiring the host computer to intervene.

---

## How Triggering Works

The DGS uses a **pipelined trigger** architecture. Digitizers never stop digitizing — they buffer every event and wait up to ~20 µs (the TRIG_DELAY window, ~10 trigger cycles) to learn whether it was accepted or rejected.

### Step 1 — Detectors Fire: DIG Discriminates

When a gamma ray hits a detector, the ADC in the **DIG** sees the pulse. The per-channel pipeline runs continuously at 100 MHz:

```
ADC data (14-bit, 100 MHz)
  → Delay chain (P1, P2, M, K, D, D3 taps)
  → Moving-average filter (noise reduction)
  → Threshold or CFD discriminator
  → COARSE_DISC_FLAG asserted for that channel
```

When the discriminator fires, the channel opens a **Pending Event Queue (PEQ)** entry and begins integrating energy sums (PRE_RISE, POST_RISE) and recording a timestamp. The event sits pending — waiting to be accepted or rejected by the trigger decision that will arrive one cycle later.

Every 50 MHz clock the DIG sends its current hit flags upstream to the Router:
```
SERDES TX word: [SYNC | COARSE_DISC[9:5] | ACCEPTED_HITS[9:0]]
```

### Step 2 — Router Aggregates: RTRG Counts Multiplicity

The **RTRG** receives hit flags from up to 8 DIGs simultaneously on Links A–H. For each clock cycle it:
- Separates hits into **X-plane** (even strips) and **Y-plane** (odd strips)
- Counts total X multiplicity and Y multiplicity across all 8 DIGs
- ORs all per-channel throttle requests into a single throttle bit

It then sends a compact packet upstream to the MTRG on Link L:
```
SERDES TX word (16-bit, DC-balance wrapped to 18): `[bit15=THR | bits14:8=Y-sum(0-80, 7-bit) | bit7=VAL(all DIGs locked) | bits6:0=X-sum(0-80, 7-bit)]` ✅ verified 2026-04-14 — `router_data_path.vhd:L10-15` (header comment) + `L230-233` (signal assignments)
- THR: OR of all digitizer throttle requests (`ANY_THROTTLE_REQUEST`)
- Y-sum bits[14:8]: sum of Y-plane multiplicities from all 8 digitizers (0–80 max: 8 DIGs × 10 ch)
- VAL bit[7]: `ALL_DIGITIZERS_LOCKED AND ROUTER_LOCKED` — data valid flag
- X-sum bits[6:0]: sum of X-plane multiplicities from all 8 digitizers (0–80 max)
```

The MTRG therefore sees a continuously updated multiplicity stream from each Router, refreshed every 20 ns.

### Step 3 — Master Trigger Decides: MTRG Runs Algorithms

*Source: DGS Master Trigger Firmware User Guide, J. Anderson, August 2015*

The **MTRG** receives multiplicity streams from up to 8 RTRGs simultaneously. It runs **8 independent trigger algorithms** in parallel, each evaluating a different condition:

- **GITMO** — germanium channel coincidences and energy thresholds
- **MYRIAD** — auxiliary detector coincidences
- **Multiplicity** — total hit count above threshold
- **Coincidence** — X/Y plane overlap within a time window
- **Energy sum** — summed energy across channels
- And others configured via VME registers

The **TDC** (Time-to-Digital Converter) uses a 4-phase vernier chain (0°/90°/180°/270° at 250 MHz) to timestamp the trigger with **~30 ps resolution** (~50 ps per vernier tap, 4 ns coarse period subdivided by 4 phases × 64-bit vernier chain), much finer than the 20 ns SERDES clock. ✅ verified 2026-04-18 — `tdc_chain_cont.vhd:L29-32` (4-phase 250 MHz ports); `tdc_unit_cont.vhd:L30,L35` (250 MHz clock, 64-bit `TDC_VERNIER_OUT`); `jta_vernier_pos_finder.vhd:L43` (`TDC_POS` 6-bit output → 64 positions over 4 ns = ~62.5 ps/tap; ~30 ps resolution from 4-phase combination)

At the end of each 2 µs cycle, the **master state machine** (`mstr_mach`) collects available algorithm results and writes them into the outgoing command frames:
```
Frame 3  → Algorithm 1 result: accepted/rejected + trigger timestamp
Frame 4  → Algorithm 2 result
...
Frame 10 → Algorithm 8 result
```

### Step 4 — Decision Propagates Back Down

The MTRG broadcasts the 20-frame command structure downstream to all RTRGs. Each **RTRG** sanitizes the frames (replacing Router-specific fields with nulls, injecting per-channel veto bits) and forwards them to all 8 of its DIGs.

Each **DIG** receives the trigger decision frames and for each pending event in the PEQ checks:
- Does the trigger timestamp match my event timestamp?
- Is this event accepted or vetoed by any of the 8 trigger algorithms?

**If accepted:** event data (timestamp, energy sums, optional waveform samples) is packed into the 36-bit external FIFO for VME readout by the host computer.

**If rejected:** the pending event is silently discarded.

### End-to-End Timeline

```
t = 0          Ge detector fires
               → DIG discriminator asserts hit flag
               → DIG opens Pending Event Queue (PEQ) entry, begins energy integration
               → DIG sends hit flag to RTRG via SERDES upstream link (every 20 ns)

t ~ 100–200 ns RTRG receives hit flag, aggregates multiplicity, forwards to MTRG
               MTRG trigger algorithms evaluate continuously

t ~ 2–4 µs    MTRG algorithm decision ready
               → trig_collect gathers result during frames 14–20 of current cycle
               → mstr_mach broadcasts ACCEPT message in frames 3–10 of next cycle
               → Decision propagates: MTRG → RTRG → DIG (one SERDES hop each)
               → DIG Searcher matches trigger timestamp to PEQ entry:
                 TS_COMP_LOWER_LIMIT < (TS_event − TS_trigger) < TS_COMP_UPPER_LIMIT
               → PEQ entry marked ACCEPTED

t = ~20 µs    EVENT_EXPIRED fires (discriminator flag delayed by TRIG_DELAY = 2 × 1024 samples at 100 MHz) ✅ verified 2026-04-07 — jta_channel.vhd:L1019,L1026,L1051 (two DP_BRAM_RWA_RB_1Kx18 in series)
               → Any PEQ entry still PENDING (no decision received) is forcibly REJECTED
               → Accepted events: data already packed into channel readout FIFO ✓
               → Waveform data falls off the delay chain after this point

t >> 20 µs    Host computer reads accepted events from external FIFO via VME
```

**Key point:** The **system cycle is 2 µs** (20 frames × 5 words at 50 MHz), not 200 µs. Each cycle can carry up to 8 trigger decisions (Frames 3–10). A trigger decision typically arrives at the DIG within **~2–4 µs** (1–2 system cycles) — well within the **~20 µs TRIG_DELAY** window before `EVENT_EXPIRED` forces a reject. This is also why `TS_COMP_UPPER/LOWER_LIMIT` have a useful range of only ±10 µs (one BRAM depth at 100 MHz), even though the registers are 16 bits wide.

The critical design point: **DIGs digitize and buffer continuously without waiting for a trigger.** The trigger decision arrives retroactively within a few µs, selecting which buffered events to keep. This eliminates dead time in the front-end electronics under normal operating conditions.

#### Pending Event Queue (PEQ) details

Each channel has a **16-entry circular buffer** (the "trigger rondel") managed by three concurrent state machines: ✅ verified 2026-04-12 — `jta_channel.vhd:L94` (`DIAG_PEHQ_ADDR : out std_logic_vector(3 downto 0)` — 4-bit address space = 16 entries)

- **Filler** — writes a new entry (timestamp + status bits) on every `ACCEPTED_HIT`.
- **Remover** — advances the read pointer on every `EVENT_EXPIRED`. `EVENT_EXPIRED` is the discriminator flag delayed by `M + TRIG_DELAY` (the waveform capture window, a few µs). This means entries are continuously retired as their waveform data window closes — the PEQ does **not** accumulate events over a long window — entries expire after ~20 µs.
- **Searcher** — when a trigger decision arrives, scans PEQ entries and marks each one accept or reject based on a timestamp window comparison. The MTRG ACCEPT message carries `TS_trigger` (the timestamp of the triggering hit). Each PEQ entry holds `TS_event` (when this channel's discriminator fired). An event is accepted if `TS_event` falls inside the window:

  ```
  time
   ↓
   ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄  TS_trigger + TS_COMP_LOWER_LIMIT  (e.g. −50 counts = −500 ns)
              ↑
         accept zone      ← any PEQ entry with TS_event here is ACCEPTED
              |
   ●  ─ ─ ─ ─ ─ ─ ─ ─ ─  TS_trigger   (timestamp in MTRG ACCEPT message)
              |
         accept zone      ← any PEQ entry with TS_event here is ACCEPTED
              ↓
   ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄  TS_trigger + TS_COMP_UPPER_LIMIT  (e.g. +50 counts = +500 ns)

   (outside either bound → REJECTED)
  ```

  Both limits are 16-bit **signed** values (units: 100 MHz counts = 10 ns each), so the window can be asymmetric. ✅ verified 2026-04-08 — `Digitizer.vhd:L356-357` (`reg_win_comp_max/min: std_logic_vector(15 downto 0)`) + `MDigRegisters.template:L52-62` (EPICS PVs). Typical use: `LOWER_LIMIT` negative (events that fired slightly before the trigger), `UPPER_LIMIT` positive (events that fired slightly after). The useful range is ±1024 counts (±10.24 µs) — beyond that, the event has already expired from the PEQ. ✅ verified 2026-04-08 — T1+T2 buffer = 2048 cells × 10 ns = 20.48 µs (`DIG_firmware_expert.md:L141`).

  **Important:** `TS_COMP_*_LIMIT` is a **physics coincidence window**, not a trigger latency window. `TS_trigger` is the timestamp of the hit that *caused* the MTRG to fire (recorded at the moment of the original hit, e.g. t = 0), not the time the ACCEPT message arrived at the DIG (~2–4 µs later). The Searcher compares channel hit times against this original physics timestamp, so the ±10.24 µs hardware ceiling has nothing to do with the 20 µs TRIG_DELAY:

  ```
  t = 0 µs     ● Central contact fires  →  TS_trigger = 0  (embedded in ACCEPT message)
               ● Segment also fires     →  TS_event   = +30 counts (+300 ns)

  t ~ 2–4 µs   ACCEPT message arrives at DIG (carrying TS_trigger = 0)
               Searcher: TS_event − TS_trigger = +30 counts
               LOWER_LIMIT < 30 < UPPER_LIMIT  →  ACCEPTED ✓

  t = ~20 µs   EVENT_EXPIRED — any still-pending entries forcibly rejected
  ```

  In practice the coincidence window is set to match detector physics — typically ±500 ns to ±2 µs for germanium arrays — well within the ±10.24 µs hardware ceiling.

#### Cable length and communication latency constraint

The TRIG_DELAY (~20 µs) also sets a hard constraint on the **round-trip communication time** from DIG to MTRG and back. If the ACCEPT message arrives *after* `EVENT_EXPIRED` fires, the PEQ entry has already been removed by the Remover — the Searcher finds no matching entry and the event is **silently lost** (not a PEQ overflow, just a missed match). The round-trip budget is therefore:

```
Round-trip budget:  DIG → RTRG → MTRG → algorithm → MTRG → RTRG → DIG  <  ~20 µs
One-way budget:     ~10 µs
Max cable length:   ~10 µs × 2×10⁸ m/s (speed of light in fiber)  ≈  2 km
                    (less in practice — SERDES pipeline latency per hop consumes part of the budget)
```

TRIG_DELAY uses **two** 1K BRAMs in series (`TRIG_DELAY_T1_EXP` + `TRIG_DELAY_T2`) rather than one: doubling the delay from ~10 µs to ~20 µs doubles the tolerable cable length, at the cost of halving the maximum sustainable event rate per PEQ slot (from ~1.6 MHz to ~800 kHz). It is a deliberate design trade-off between physical reach and maximum count rate. ✅ verified 2026-04-19 — `jta_channel.vhd:L1040-1097` (20211118 tag: comment "Two BRAMs stitched together to give 20us delay"; `TRIG_DELAY_T1_EXP` each 1K×18 BRAM, chained T1→T2)

**What happens if the PEQ fills up?** If a new `ACCEPTED_HIT` arrives while the Filler detects `PEQ_TOP + 1 = PEQ_BOTTOM` (PEQ full), the Filler sets `ERROR_FLAGS(0) <= '1'` and transitions to the `ERROR` state — **the channel freezes until an external reset (`RST`) is applied**. That event is **lost**, and `GENERAL_ERROR_FLAG` is asserted in subsequent event headers (bit 13 of header word 4, injected at readout time). ✅ verified 2026-04-25 — `chan_trigger_control.vhd:L873` (PEQ full check: `PEQ_TOP + 1 = PEQ_BOTTOM → ERROR_FLAGS(0) <= '1'`; `MASTER_STATE <= ERROR`), `Digitizer.vhd:L907` (`general_error_flag(i) <= peq_diag_reg(i)(26)`), `Event_Header_FIFO.vhd:L726` (header bit 13 injected at readout). The ERROR state has no self-recovery: `MASTER_MACH` only exits ERROR via async `RST` signal (`chan_trigger_control.vhd:L710`). This is an error condition, not normal operation. It is prevented by two mechanisms: (1) pileup rejection in the channel logic limits the rate of `ACCEPTED_HIT` pulses, and (2) the throttle mechanism signals the MTRG to raise the trigger threshold if the downstream readout FIFO approaches full, reducing the overall hit rate before the PEQ saturates.

#### Maximum per-channel event rate

The TRIG_DELAY (~20 µs) and PEQ depth (16 entries) together determine the maximum sustainable hit rate per channel:

```
Theoretical PEQ limit:  16 entries / 20 µs  =  800 kHz
```

However the dominant practical limit is the **pileup holdoff** (`reg_holdoff_control[8:0]`), which prevents the channel from re-triggering until the holdoff window expires. For example, a holdoff of 200 cycles at 100 MHz (2 µs) limits the channel to 500 kHz. The three rate limits in priority order are:

| Limit | Mechanism | Typical value |
|-------|-----------|--------------|
| Pileup holdoff | `reg_holdoff_control` — minimum inter-event spacing | 100–500 kHz (user-configurable) |
| PEQ capacity | 16 entries ÷ ~20 µs TRIG_DELAY | ~800 kHz ceiling |
| Readout bandwidth | External FIFO + VME DMA — managed by throttle | Much lower; experiment-dependent |

Note: the TRIG_DELAY alone does **not** limit the rate to 1/20 µs = 50 kHz, because up to 16 events can be buffered simultaneously in the PEQ.

---

## Module Documentation

| Module | Doc | Details |
|--------|-----|---------|
| Master Trigger (hub) | [deep_fpga_MTRG.md](deep_fpga_MTRG.md) | Device table, repo layout, links to sub-docs |
| MTRG Main FPGA (ISE) | [deep_fpga_MTRG_MAIN.md](deep_fpga_MTRG_MAIN.md) | Virtex-4, trigger algorithms, TAC-II TDC, VME register map |
| MTRG Main FPGA (Vivado) | [deep_fpga_MTRG_VIVADO.md](deep_fpga_MTRG_VIVADO.md) | Kintex UltraScale port, Vivado 2018.3 differences |
| MTRG VME FPGA | [deep_fpga_MTRG_VME.md](deep_fpga_MTRG_VME.md) | Spartan-3, A32/D32 VME slave, FPGA config manager |
| MTRG CPLD | [deep_fpga_MTRG_CPLD.md](deep_fpga_MTRG_CPLD.md) | XC9500XL, fast strobe multiplicity logic (~1 µs latency) |
| Router (RTRG) | [deep_fpga_RTRG.md](deep_fpga_RTRG.md) | Virtex-4, multiplicity aggregation, throttle, VME register map |
| Digitizer (DIG) | [deep_fpga_DIG.md](deep_fpga_DIG.md) | Spartan-3, ADC pipeline, discriminators, event packet format |
| DIG Firmware Expert | [DIG_firmware_expert.md](DIG_firmware_expert.md) | All 8 readout modes, timing, pileup, master/slave, ADC linearity |

---

## Firmware Type Codes (BUILD_TYPE)

All ANL FPGA firmware variants share a common `trigger_top` component parameterized by `BUILD_TYPE`. The `CODE_REVISION` PV encodes this in bits\[11:8\] (e.g. RTRG `0x260E` → `6` = DGS Router). ✅ verified 2026-04-08 — `FPGA/others/Trig_sys_sim/MstrTrig_pkg.vhd` generic declaration comments.

| BUILD_TYPE | Hex | Firmware Variant |
|------------|-----|------------------|
| 0 | 0x0 | Prototype |
| 1 | 0x1 | GRETINA Router |
| 2 | 0x2 | GRETINA Master Trigger |
| 3 | 0x3 | GRETINA Data Generator |
| 4 | 0x4 | **DGS Master Trigger** |
| 5 | 0x5 | DSSD Master Trigger |
| 6 | 0x6 | **DGS Router** |
| 7 | 0x7 | DSSD Router |
| 8 | 0x8 | **DGS Data Generator** |
| 9 | 0x9 | DSSD Data Generator |
| A | 0xA | **Digitizer Tester** |
| B | 0xB | **MyRIAD Trigger Expansion** |
| C | 0xC | **DGS Digitizer** |
| D | 0xD | DSSD Digitizer |
| E | 0xE | (unused) |
| F | 0xF | **VME FPGA** |

**DGS-relevant types in bold.** Current production firmware: MTRG=0x4, RTRG=0x6, DIG=0xC/0xD.

> ⚠️ **The `BUILD_TYPE` enum above is a `trigger_top` simulation-only generic from `MstrTrig_pkg.vhd` — it is NOT the field layout actually written to the DIG `regin_code_revision` register.** The DIG firmware writes a different format. See [`VME_registers.md`](VME_registers.md#0x0600-0x060c-code-revision--timestamp-error) and [`ioc.md`](ioc.md) for the real DIG `regin_code_revision` decode.
>
> Briefly: DIG `regin_code_revision` is hard-coded as `X"00004" & X"D" & cCODE_VERSION_MAJOR & cCODE_VERSION_MINOR` when `SLAVE_MODE=TRUE`, else `X"00004" & X"C" & ...`. So `0x4CD8` = master DIG (major=D, minor=8); `0x4DD8` = slave DIG (same source build). The `0xC`/`0xD` nibble here is a **master/slave selector for the DIG board type**, not the MTRG BUILD_TYPE enum value (which happens to also list `C=DGS Digitizer`/`D=DSSD Digitizer` — coincidental nibble reuse, unrelated meaning). ✅ verified 2026-06-10 — `DGSDIG_git/BuildBranches/DGS_TAG_20180607_TWEAK/DGS/Source/Digitizer.vhd:L471`
>
> MTRG `0x04A8` and RTRG `0x260E` use yet a different layout — those values are MTRG `trigger_top` instantiations and the bits[11:8] interpretation differs again. The MTRG/RTRG `BUILD_TYPE` codes (4=DGS MTRG, 6=DGS Router) are passed as a Verilog/VHDL generic at synthesis time but the exact register-format mapping for those boards has not been re-verified here.

---

## Timing

| Parameter | Value |
|-----------|-------|
| Trigger cycle period | 2 µs (100 words × 20 ns at 50 MHz) |
| Frames per cycle | 20 |
| Frame period | 100 ns (5 words × 20 ns) |
| Frame size (downstream) | 80 bits (5 × 16-bit words) |
| Words per frame | 5 |
| SERDES clock | 50 MHz |
| ADC sampling rate | 100 MHz |
| Timestamp resolution | ~30 ps (vernier TDC in MTRG — 4-phase 250 MHz, ~50 ps/tap) ✅ verified 2026-04-06 — `tdc_chain_cont.vhd` (4 phases × 250 MHz) + `deep_fpga_MTRG_MAIN.md:L315` (TAC.pdf: 50 ps/tap) |
| Timestamp width | 48 bits |
| Throttle min pulse width | >2 µs (RTRG stretched) |
| Fast strobe latency | ~1 µs (CPLD, analog multiplicity) |

---

## VME Control Hierarchy

Each module is controlled via VME bus. The VME FPGA in each module acts as the VME slave and programs the main FPGA:

```
Host Computer (VME Controller)
    │
    │  VME Bus (A32/D32)
    ├──► MTRG VME FPGA (XC3S400) ──► MTRG Main FPGA (XC4VLX80)
    ├──► RTRG VME FPGA (XC3S400) ──► RTRG Main FPGA (XC4VLX80)
    └──► DIG  VME FPGA (XC3S400) ──► DIG  Main FPGA (XC3S5000)
```

All VME FPGAs share the same architecture (Spartan-3 XC3S400): they present an A32/D32 VME slave interface, store the main FPGA bitstream in external flash, and program the main FPGA via serial configuration on power-up or on demand. ✅ verified 2026-04-25 — all three VME FPGA `.xise` project files confirm `xc3s400`: DIG (`VME_FPGA_ANL/Work11/vme_A32_D32.xise:Device=xc3s400`), RTRG (`Router/VME_FPGA/Work13.4/vme_A32_D32.xise:Device=xc3s400`), MTRG (`MasterTrigger/Release_Dec_2014/Firmware/VME_FPGA/A32D32_VME_FPGA/Work13.4/vme_A32_D32.xise:Device=xc3s400`)

---

## Related Repositories

### SVN — DGS Software, EPICS, and Documentation

The DGS software stack, EPICS IOC, VME/PV register maps, and system documentation live in a separate SVN repository:

```
https://svn.inside.anl.gov/repos/dgs
Local checkout: ~/DGS_SVN/dgs/
```

#### Key paths for VME registers and EPICS PVs

| Path | Contents |
|------|----------|
| `jta_SS/DGSMasterTriggerRegisterMap.xls` | MTRG VME register map with PV names |
| `jta_SS/DGSRouterTriggerRegisterMap.xls` | RTRG VME register map with PV names |
| `jta_SS/MasterDigitizerRegisterMap.xls` | DIG VME register map with PV names |
| `Router/DGSRouterTriggerRegisterMap.xls` | Router register map (alternate copy) |
| `jta_SS/*.db` | EPICS PV database files per VME crate |
| `dgsSoftIOC/db/dgsCombinedDatabase.db` | Combined EPICS database for all DGS PVs |
| `dgsSoftIOC/db/dgs_Fanouts.db` | EPICS fanout records |

CSV versions of the register maps are also available alongside the `.xls` files in `jta_SS/`.

#### Basic SVN commands

```bash
# Check out the full repository (large — use sparse if needed)
svn checkout https://svn.inside.anl.gov/repos/dgs ~/DGS_SVN/dgs

# Update to latest revision
svn update

# Check current revision and URL
svn info

# Browse without checking out
svn list https://svn.inside.anl.gov/repos/dgs
svn list https://svn.inside.anl.gov/repos/dgs/jta_SS

# Check out only a subdirectory (sparse)
svn checkout https://svn.inside.anl.gov/repos/dgs/jta_SS ~/DGS_SVN/jta_SS

# See recent changes
svn log -l 20

# See what changed in a specific revision
svn diff -c 7231
```

---

## FPGA/others/ — Auxiliary Firmware

The `DGS_tools_pack/FPGA/others/` directory contains related but non-production firmware:

| Directory | Contents |
|-----------|----------|
| `Trig_sys_sim/` | **Full trigger system simulation testbench** — ISE VHDL testbench (`top_tb1.VHD`) for the complete MTRG `trigger_top` entity. Includes: `MstrTrig_pkg.vhd` (BUILD_TYPE generics and type definitions), `MyRIAD_pkg.vhd` (MγRIAD interface types), `bus_pkg.vhd` / `bus_trans.vhd` / `bus_io.vhd` (VME bus simulation models), `crate_def_tb.vhd` (crate instantiation), `regio_tb.vhd` (register I/O testbench). Used for verifying trigger logic in simulation without hardware. Author: J. Anderson (ANL). |
| `MyRIAD/` | **MγRIAD FPGA firmware** — ISE project (Spartan-3) for the MγRIAD auxiliary detector interface module. Source: `MAIN_FPGA/Source/MyRIAD.vhd` (2,027 lines). See [myriad.md](myriad.md) for full documentation. |
| `Majorana_Digitizer/` | **Majorana Demonstrator digitizer firmware** — ISE project (Spartan-3 **XC3S5000-5FG900C** ✅ verified 2026-04-21 — `Digitizer.ucf:L9`) for the Majorana Demonstrator experiment. Branched from DGS DIG firmware on **2015-08-31** ✅ verified 2026-04-21 — `Digitizer.vhd:L12-18` (branch comment). **Key architectural differences from DGS DIG:** (1) CFD discriminator removed; (2) `CFD_MODE` control flag removed; (3) triple-filter removed; (4) threshold discriminator runs on full 24-bit M1+M2 sums; (5) threshold register widened to 24 bits. **Firmware revision:** `cCODE_VERSION_MAJOR=0xB`, `cCODE_VERSION_MINOR=0xB`; code type `0xC` (master) / `0xD` (slave) — `SLAVE_MODE` generic determines direction of front bus and disables internal cross-channel veto ✅ verified 2026-04-21 — `Digitizer.vhd:L351-352,L391`. **Code date:** `0x20150601` ✅ verified 2026-04-21 — `Digitizer.vhd:L416`. Shares many VHDL modules with ANL DIG: `Channel_Readout_Mach.vhd`, `Channel_FIFO_Readout_Mach.vhd`, `cfd_disc.vhd` (present but not wired to trigger path), `baseline_tracker.vhd` (new — tracks baseline level with programmable delay+speed; 24-bit sampled baseline output ✅ verified 2026-04-21 — `baseline_tracker.vhd` port declarations), `chan_trigger_control.vhd`, `CLOCK_MANAGER.vhd`. AUX I/O remapped for Majorana cabling: `aux_in_en=101100`, `aux_out_en=010011` ✅ verified 2026-04-21 — `Digitizer.vhd:L962-967`. Includes ChipScope debugging configs and `dg_pulse_estimator.xls`. Source: `MAIN_FPGA/Source/` (50 VHDL files). See [majorana_digitizer.md](majorana_digitizer.md) for full documentation. |
| `LBL_Digitizer/` | **Lawrence Berkeley National Laboratory (LBNL) GRETINA digitizer firmware** — ISE project (Virtex-2 Pro), top-level `original_greta14bitse/chip.vhd` (1,695 lines). LBNL proprietary; last updated **2006-08-30** by Dionisio Doering. Firmware version **1.06** (`chip.vhd:L797-798`). Stored as historical reference — the GRETINA digitizer design predates the ANL DIG and shows the design lineage. Not built or deployed in DGS. Includes PDFs: `GRETA Algorithm Architecture-v3.1b.pdf` and `Gretina VHDL Modules description_14bits_v2_1.pdf`. Also contains JTA branches (`jta_temp_branch/`, `LBL_20100804/`, `20110930/`) and a `VME FGPA/` [sic] subdirectory with ISE project for the VME interface FPGA.<br>**Architecture (original_greta14bitse):** 10-channel 14-bit ADC digitizer. Key components instantiated in `chip.vhd`: `Ten_Channel` (all 10 per-channel pipelines), `VMEControl` (VME register I/O), `DACControl` (8-ch DAC for thresholds), `MasterLogicModule` (Front Bus master arbiter), `FrontBusSlaveModule` (Front Bus slave for register broadcast), `SelfTrigger` (local LED+validate logic), `FIFOOccupancyCounter` (external FIFO depth tracking). Each `Channel.vhd` (680 lines, designer Vincent Riot) contains: `TapDelay` (programmable delay line), `LED` (leading-edge discriminator), `trapezoid_disc` (trapezoidal filter discriminator on 25-bit trapezoid output), `CFD` (constant-fraction discriminator, 17-bit resolution), `Energy` (trapezoid energy integrator), `ProcCore` (per-channel processing controller with BRAM prebuffer). 47-bit timestamp counter clocked at 100 MHz. Front Bus carries discriminator/validate signals between Master and up to 9 Slave channels on the same board. External IDT FIFO (36-bit wide, occupancy-tracked) stores event data packets. SERDES link (18-bit, 50 MHz) connects to Router/trigger system — uses same DS92LV18 physical layer as ANL DIG. |

---

## Building the Firmware

See **[deep_fpga_building.md](deep_fpga_building.md)** for the full build guide, including:

- Toolchain versions for each module (ISE 14.7 / Vivado 2018.3)
- ISE 14.7 device support (Virtex-4, Spartan-3 confirmed)
- Running ISE 14.7 on Ubuntu 24.04 via Docker/Podman container
- LD_PRELOAD native patch alternative

---

## Related Documentation

| Doc | Relevance |
|-----|-----------|
| [tac2.md](tac2.md) | TAC-II TDC in MTRG: vernier interpolation, 64-tap delay lines, 250 MHz 4-phase clock, state machines |
| [ttcl.md](ttcl.md) | TTCL spec: all 20 frames, frame/word/cycle format, MTRG command encoding |
| [connectors.md](connectors.md) | All connector pinouts: DIG (RJ45, 36-pin Aux I/O), MTRG/RTRG (125-pin SERDES, NIM I/O, ECL) |
| [reference_index.md](reference_index.md) | VME register map index + hardware drawings index |
| [preamp_reset_readme.md](preamp_reset_readme.md) | PRK holdoff timing, PREAMP_RESET_DELAY register in DIG firmware |
| [digitizer_tester.md](digitizer_tester.md) | Digitizer Tester: dual 200 MHz DAC, analog switch matrix, TTCL link, waveform generation |
| [260E_trigger_scheme.md](260E_trigger_scheme.md) | RTRG 0x260E trigger scheme: chan_in.vhd (serial reception, DPRAM delay alignment, X/Y plane maps), router_data_path.vhd (multiplicity aggregation, Link-L output); VHDL-verified |
| [deep_fpga_DIG.md](deep_fpga_DIG.md) | DIG firmware deep dive: Spartan-3, ADC pipeline, event packet format, pole-zero |
| [deep_fpga_DIG_channel.md](deep_fpga_DIG_channel.md) | DIG per-channel signal processing: LED/CFD discriminator modes, delay chain, pileup, VME FPGA, IP cores |
| [deep_fpga_DIG_modules.md](deep_fpga_DIG_modules.md) | DIG selected module analysis Part 1 (signal chain & SERDES): `SERDES_TX_Mach_DGS.vhd` (disc packer), `event_packer.vhd` (accordion FIFO), `pileup_processor.vhd` (8-state FSM), `SERDES_RX_Mach.vhd` (20-frame Router command receiver), `Timestamp_Generator.vhd`, `Trigger_Mux.vhd`, `Channel_Readout_Controller.vhd`, `Channel_Readout_Mach.vhd` |
| [deep_fpga_DIG_modules2.md](deep_fpga_DIG_modules2.md) | DIG selected module analysis Part 2 (DC balance, FIFOs, VME): `dc_balance_mach.vhd`, `disparity_lookup.vhd`, `event_data_fifo.vhd`, `decimator.vhd`, `Event_Header_FIFO.vhd`, `Channel_FIFO_Readout_Mach.vhd`, `Lvme.vhd`, `Registers.vhd` (199-entry register map), `dp_srl_template.vhd` |
| [trig_sys_sim.md](trig_sys_sim.md) | Full MTRG trigger system simulation testbench (`FPGA/others/Trig_sys_sim/`): two `trigger_top` (BUILD_TYPE=4) instances wired via 79 ns fake SERDES, VME bus mux model, 8-step stimulus sequence (link init → manual triggers → propagation → DC balance → rapid-fire), full VME address constants table, BUILD_TYPE code map 0–F |
| [DIG_firmware_expert.md](DIG_firmware_expert.md) | All 8 readout modes, discriminator modes, pileup, timing, ADC linearity specs |
| [ioc.md](ioc.md) | EPICS IOC: boot scripts, DB loading, firmware version PVs, MVME5500 setup |
| [vxworks.md](vxworks.md) | VxWorks cross-compilation: build pipeline, munch process, IOC connections |
| [ANLDAQ.md](ANLDAQ.md) | DAQ GUI + TCP data receiver; trigger setup scripts; EPICS CA config per system |
| [guceiver.md](guceiver.md) | Guceiver live diagnostic GUI: reads IOC TCP stream; DIG + TAC-II packet decoders |

### VHDL Module Analysis (`vhdl/` subdirectory)

Detailed plain-English summaries of FPGA VHDL source files. See [vhdl/PROGRESS.md](vhdl/PROGRESS.md) for coverage checklist.

| Doc | Module | Description |
|-----|--------|-------------|
| [vhdl/RTRG_chan_in.md](vhdl/RTRG_chan_in.md) | `chan_in.vhd` | RTRG: serial SERDES input, 18-bit word decoding, 640 ns DPRAM delay alignment, discriminator extraction |
| [vhdl/RTRG_disc_mach.md](vhdl/RTRG_disc_mach.md) | `disc_mach.vhd` | RTRG: BGO/Ge discriminator state machine (clean/dirty/BGO-only classification) |
| [vhdl/RTRG_overlap_mach.md](vhdl/RTRG_overlap_mach.md) | `overlap_mach.vhd` | RTRG: trigger overlap and hold-off state machine |
| [vhdl/RTRG_router_data_path.md](vhdl/RTRG_router_data_path.md) | `router_data_path.vhd` | RTRG: Link-L multiplicity aggregation, data forwarding to MTRG |
| [vhdl/RTRG_top.md](vhdl/RTRG_top.md) | `TOP.VHD` | RTRG top-level: all sub-block wiring, port map, SERDES link management |
| [vhdl/MTRG_top.md](vhdl/MTRG_top.md) | `top.vhd` | MTRG top-level: 8 Router aggregation, trigger decision distribution, NIM I/O, CPLD bus |
| [vhdl/MTRG_eight_mt_channel.md](vhdl/MTRG_eight_mt_channel.md) | `eight_mt_channel.vhd` | MTRG: instantiates 8 `mt_input_channel` blocks, one per Router link |
| [vhdl/MTRG_mt_input_channel.md](vhdl/MTRG_mt_input_channel.md) | `mt_input_channel.vhd` | MTRG: per-Router SERDES receiver, hit extraction, multiplicity contribution |
| [vhdl/MTRG_sum_hits_X.md](vhdl/MTRG_sum_hits_X.md) | `sum_hits_X.vhd` | MTRG: X-plane hit count summation (north/south hemisphere aggregation) |
| [vhdl/MTRG_calc_total_sum.md](vhdl/MTRG_calc_total_sum.md) | `calc_total_sum.vhd` | MTRG: final multiplicity sum and trigger decision comparator |
| [vhdl/MTRG_MYRIAD_RCV_MACH.md](vhdl/MTRG_MYRIAD_RCV_MACH.md) | `MYRIAD_RCV_MACH.vhd` | MTRG: MγRIAD receiver — locks onto 5-word SERDES frame, extracts NIM/ECL/FERA/trigger bits |
| [vhdl/MTRG_MYRIAD_TRIGGER.md](vhdl/MTRG_MYRIAD_TRIGGER.md) | `MYRIAD_TRIGGER.vhd` | MTRG: MγRIAD trigger algorithm — delay line, optional coincidence, subtypes 0x78/0x79 |
| [vhdl/MTRG_mstr_mach.md](vhdl/MTRG_mstr_mach.md) | `mstr_mach.vhd` | MTRG: Master State Machine — 20-frame TTCL command cycle, trigger FIFO pipelining |
| [vhdl/MTRG_local_trig_coinc.md](vhdl/MTRG_local_trig_coinc.md) | `local_trig_coinc.vhd` | MTRG: local-vs-local coincidence trigger algorithm |
| [vhdl/MTRG_trig_algo_support.md](vhdl/MTRG_trig_algo_support.md) | `trig_algo_support.vhd` | MTRG: shared base component for all trigger algorithms (dual FIFO, prescaler, holdoff, throttle) |
| [vhdl/MTRG_support_modules.md](vhdl/MTRG_support_modules.md) | `timestamp.vhd`, `data_compressor.vhd`, `link_tx_block.vhd`, `remote_trig_support.vhd`, `trig_mon_collect.vhd`, `trigger_data_types.vhd` | MTRG support/infrastructure: 48-bit timestamp, TDC vernier compressor, DC-balanced SERDES fan-out, cross-system trigger (Link R), monitor FIFO collector, VHDL type definitions |
| [vhdl/MTRG_registers.md](vhdl/MTRG_registers.md) | `registers.vhd` | MTRG VME register map: ~120 R/O + R/W registers, 3 lookup RAMs (VETO/TRIG/SWEEP), 8+8 monitor FIFOs (MON1-8 + CHAN1-8), VME FSM state machine, rate counters |
| [vhdl/MTRG_AUX_IO.md](vhdl/MTRG_AUX_IO.md) | `AUX_IO.VHD` | MTRG: AUX port mux, NIM output mux (4 modes), target wheel encoder (parallel FILTER FSM + SSI SLIDE FSM + SSI serial receiver), polarity inversion, BEAM_SWEEP_OUT |
| [vhdl/MTRG_SERDES_RX_Mach.md](vhdl/MTRG_SERDES_RX_Mach.md) | `SERDES_RX_Mach_R2.vhd` | MTRG: 20-frame SERDES receiver FSM — lock/prelock, all frame decoders (F1 sync/ISY, F3-F10 triggers, F11-F19 spare/cmd, F20 EOC), VETO_EVENT, sanitized output |
| [vhdl/MTRG_pos_finder.md](vhdl/MTRG_pos_finder.md) | `pos_finder.vhd` | MTRG TDC: thermometer-code edge position lookup — 11/12-bit slice → 4-bit position + valid flag; 2048/4096-entry ROM; 1-cycle pipeline; used by vernier_pos_finder |
| [vhdl/MTRG_sum_hits_XY.md](vhdl/MTRG_sum_hits_XY.md) | `sum_hits_XY.vhd` | MTRG: XY coincidence trigger — fires when both X and Y global sums simultaneously exceed VME-configurable thresholds; 2-state FSM + trig_algo_support |
| [vhdl/MTRG_comp_defs.md](vhdl/MTRG_comp_defs.md) | `trigger_comp_defs.vhd`, `trigger_top_comp_defs.vhd` | MTRG component declaration packages — all sub-design and top-level component port lists; includes tdc_chain_cont (4-phase 250 MHz TDC chain) port documentation |
| [vhdl/MTRG_tdc_chain_cont.md](vhdl/MTRG_tdc_chain_cont.md) | `tdc_chain_cont.vhd` | MTRG TDC chain controller — 4-phase carry-chain TDC units, fine counters, trigger ACK resampling + accumulation, 5-state autosample FSM, 80→20-bit FIFO repacking, 8-word TDC event packet format, TDC_FIFO_DATA_READY |
| [vhdl/MTRG_Generated_top.md](vhdl/MTRG_Generated_top.md) | `Generated_top.vhd` (entity `trigger_top`) | MTRG top-level structural glue — all 24 component instantiations + wiring, trigger algorithm slot map (algos 1–8), veto system (SYSTEM_VETO_STATE format), 8 monitor FIFO assignments, inline logic inventory (DCM, pad buffers, Frame 12/14/16/17 FSMs, rate counters, AUX direction), clock infrastructure, firmware type codes |

### SBX (Slope Box Extension) — Motherboard Control FPGA

| Doc | Module | Description |
|-----|--------|-------------|
| [deep_fpga_SBX_CtrlFPGA.md](deep_fpga_SBX_CtrlFPGA.md) | `SlopeBoxInt_TopLevel_RevC.vhd`, `PI_TRANSACTOR.vhd`, `I2C_template.vhd`, `LOOK_UP_TABLE1.VHD` | SBX Control FPGA (Spartan-6 XC6SLX9): 24-bit SPI interface to Raspberry Pi/Collector, 7-bit address decode, 128-register file, 3× I2C buses (power/preamp/dongle) with scanner machines, BGO discriminator DDR outputs, slope box 3-wire serial interface, analog switch control (TAU/GAIN/SIDE/BGO/DAC), preamp reset clamp, 48-bit timestamp, fake-Pi detection, fan control readout |

---
*Source: `DGS_tools_pack/fpga/` and `DGS_tools_pack/FPGA/` (gitlab.phy.anl.gov/dgs-tools-pack). Created: 2026-04-05.*

## Cross-References

- [deep_fpga_DIG.md](deep_fpga_DIG.md) — DIG firmware deep dive: ADC pipeline, discriminators, event packet format
- [deep_fpga_RTRG.md](deep_fpga_RTRG.md) — RTRG firmware deep dive: multiplicity aggregation, throttle, VME map
- [deep_fpga_MTRG.md](deep_fpga_MTRG.md) — MTRG overview: 3 devices (Main FPGA, VME FPGA, CPLD)
- [deep_fpga_MTRG_MAIN.md](deep_fpga_MTRG_MAIN.md) — MTRG Main FPGA: trigger algorithms, 20-frame command, TAC-II
- [ttcl.md](ttcl.md) — TTCL spec: trigger command link word/frame/cycle format
- [tac2.md](tac2.md) — TAC-II TDC in MTRG: vernier interpolation, delay chains, pipeline
- [DIG_firmware_expert.md](DIG_firmware_expert.md) — DIG firmware expert guide: all readout modes, timing registers
- [ioc.md](ioc.md) — IOC config: firmware binary versions that must match hardware
- [deep_fpga_building.md](deep_fpga_building.md) — Build toolchain: ISE 14.7 / Vivado 2018.3
- [260E_trigger_scheme.md](260E_trigger_scheme.md) — RTRG 0x260E trigger scheme deep-dive: chan_in.vhd, router_data_path.vhd, X/Y plane maps, hit classification, full signal flow; verified against VHDL source
- [vhdl/MTRG_support_modules.md](vhdl/MTRG_support_modules.md) — MTRG infrastructure modules: timestamp.vhd, data_compressor.vhd (TDC vernier), link_tx_block.vhd (DC-balanced SERDES fan-out), remote_trig_support.vhd (Link R cross-system trigger), trig_mon_collect.vhd, trigger_data_types.vhd
- [vhdl/MTRG_trig_algo_support.md](vhdl/MTRG_trig_algo_support.md) — shared base VHDL component for all MTRG trigger algorithms
- [vhdl/MTRG_registers.md](vhdl/MTRG_registers.md) — MTRG VME register map: all ~120 R/O + R/W registers, lookup RAMs, monitor FIFOs, VME FSM
- [vhdl/MTRG_pos_finder.md](vhdl/MTRG_pos_finder.md) — TDC thermometer edge position lookup: 11/12-bit slice → 4-bit position + valid; ROM table, 1-cycle pipeline
- [vhdl/MTRG_sum_hits_XY.md](vhdl/MTRG_sum_hits_XY.md) — XY coincidence trigger algorithm: both X and Y global sums must exceed threshold simultaneously
- [vhdl/MTRG_comp_defs.md](vhdl/MTRG_comp_defs.md) — trigger_comp_defs + trigger_top_comp_defs packages: all MTRG component declarations, tdc_chain_cont ports documented
- [vhdl/MTRG_AUX_IO.md](vhdl/MTRG_AUX_IO.md) — AUX_IO.VHD: AUX port mux, NIM outputs, target wheel encoder FSMs, SSI serial receiver
- [vhdl/MTRG_SERDES_RX_Mach.md](vhdl/MTRG_SERDES_RX_Mach.md) — SERDES_RX_Mach_R2.vhd: 20-frame lock FSM, all frame decoders, VETO_EVENT
- [vhdl/MTRG_tdc_chain_cont.md](vhdl/MTRG_tdc_chain_cont.md) — tdc_chain_cont.vhd: full TDC chain controller — 4-phase carry-chain units, fine counters, trigger ACK resampling, 5-state autosample FSM, 8-word output packet format
- [vhdl/MTRG_Generated_top.md](vhdl/MTRG_Generated_top.md) — Generated_top.vhd: top-level structural glue (entity trigger_top) — all 24 component instances, trigger algo slot map, veto system (SYSTEM_VETO_STATE), 8 monitor FIFOs, inline logic, clock infrastructure, firmware type codes
- [vhdl/PROGRESS.md](vhdl/PROGRESS.md) — coverage checklist for all VHDL module analysis pages (RTRG + MTRG) — **all MTRG files now complete**
- [trig_sys_sim.md](trig_sys_sim.md) — full MTRG trigger system simulation testbench: two `trigger_top` (BUILD_TYPE=4) wired via fake SERDES delay, VME bus simulation model, 8-step stimulus sequence (link init, manual triggers, propagation, DC balance), full VME register address constants, BUILD_TYPE code table
- [XIA_1SFP.md](XIA_1SFP.md) — XIA 1-SFP Interface FPGA (Spartan-6): receive-only TTCL client bridging XIA Pixie digitizers into DGS; recovers 48-bit timestamp over SFP fiber, drives delayed NIM trigger to Pixie, shares IP (SERDES_RX_Mach, Timestamp_Generator, GITMO_RCV_MACH) with MγRIAD codebase
