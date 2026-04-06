# DGS FPGA Firmware

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
| [MTRG](MTRG/README.md) | `MTRG/` | Virtex-4 XC4VLX80 (ISE) / Kintex UltraScale XCK060 (Vivado) | ISE 13.4 / Vivado 2018.3 | Central trigger decision-maker | ✅ verified 2026-04-06 — MTRG/Firmware/MAIN_FPGA/trunk/Work13_4/Work13_4.xise |
| [RTRG](RTRG/README.md) | `RTRG/` | Virtex-4 XC4VLX80 | ISE 13.4 | Router — aggregates digitizer hits, forwards trigger commands | ✅ verified 2026-04-06 — RTRG/Firmware/DGS_Version/MAIN_FPGA/Work13_4/Work13_4.xise |
| [DIG](DIG/README.md) | `DIG/` | Spartan-3 XC3S5000 | ISE 14.7 | 10-channel waveform digitizer | ✅ verified 2026-04-06 — DIG/MAIN_FPGA/BuildBranches/DGS/Work/BUS_LEFT.xise |

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
    │  SERDES Link A–H (18-bit, 50 MHz, DC-balanced)
    │  TX word: [SYNC | COARSE_DISC[9:5] | ACCEPTED_HITS[9:0]]
    ▼
RTRG — Aggregates 8 digitizers
    │  X-plane hit count + Y-plane hit count + throttle OR
    │
    │  SERDES Link L (18-bit, 50 MHz)
    │  TX word: [THR | Y-mult[4:0] | X-mult[5:0]]
    ▼
MTRG — Runs trigger algorithms
    │  8 algorithms (GITMO, MYRIAD, multiplicity, coincidence, etc.)
    │  TDC timestamps events with ~1 ns resolution
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

Each **2 µs cycle** consists of 20 frames (100 ns per frame). Each frame carries **5 words × 16 bits = 80 bits (10 bytes)** of command data at 50 MHz, so one frame spans 5 SERDES clock cycles (100 ns). The upstream path (DIG → Router) sends an **18-bit word every 50 MHz clock cycle** (16 data bits + 2 DC-balance bits); there is no frame packetization in the upstream direction.

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
| 16 | Synchronous capture command |
| 17 | Auxiliary detector command |
| 18–20 | End-of-cycle / spare |

For the detailed word-by-word breakdown of each frame:
- **Downstream (MTRG → RTRG → DIG):** [MTRG/MAIN_FPGA.md — Command Frame Timing](MTRG/MAIN_FPGA.md#command-frame-timing) · [DIG/README.md — SERDES RX Frame Types](DIG/README.md#serdes-rx-frame-types-from-router)
- **Upstream (DIG → RTRG → MTRG):** [DIG/README.md — SERDES TX Format](DIG/README.md#serdes-tx-format-to-router)

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
    → RTRG stretches valid throttle to >2 µs
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
SERDES TX word: [THR | Y-mult[4:0] | X-mult[5:0]]
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

The **TDC** (Time-to-Digital Converter) uses a vernier chain to timestamp the trigger with **~1 ns resolution**, much finer than the 20 ns SERDES clock.

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

t = ~20 µs    EVENT_EXPIRED fires (discriminator flag delayed by TRIG_DELAY = 2 × 1024 samples at 100 MHz)
               → Any PEQ entry still PENDING (no decision received) is forcibly REJECTED
               → Accepted events: data already packed into channel readout FIFO ✓
               → Waveform data falls off the delay chain after this point

t >> 20 µs    Host computer reads accepted events from external FIFO via VME
```

**Key point:** The **system cycle is 2 µs** (20 frames × 5 words at 50 MHz), not 200 µs. Each cycle can carry up to 8 trigger decisions (Frames 3–10). A trigger decision typically arrives at the DIG within **~2–4 µs** (1–2 system cycles) — well within the **~20 µs TRIG_DELAY** window before `EVENT_EXPIRED` forces a reject. This is also why `TS_COMP_UPPER/LOWER_LIMIT` have a useful range of only ±10 µs (one BRAM depth at 100 MHz), even though the registers are 16 bits wide.

The critical design point: **DIGs digitize and buffer continuously without waiting for a trigger.** The trigger decision arrives retroactively within a few µs, selecting which buffered events to keep. This eliminates dead time in the front-end electronics under normal operating conditions.

#### Pending Event Queue (PEQ) details

Each channel has a **16-entry circular buffer** (the "trigger rondel") managed by three concurrent state machines:

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

  Both limits are 16-bit **signed** values (units: 100 MHz counts = 10 ns each), so the window can be asymmetric. Typical use: `LOWER_LIMIT` negative (events that fired slightly before the trigger), `UPPER_LIMIT` positive (events that fired slightly after). The useful range is ±1024 counts (±10.24 µs) — beyond that, the event has already expired from the PEQ.

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

This is likely why TRIG_DELAY uses **two** 1K BRAMs in series rather than one: doubling the delay from ~10 µs to ~20 µs doubles the tolerable cable length, at the cost of halving the maximum sustainable event rate per PEQ slot (from ~1.6 MHz to ~800 kHz). It is a deliberate design trade-off between physical reach and maximum count rate.

**What happens if the PEQ fills up?** If a new `ACCEPTED_HIT` arrives while the Filler is still busy (PEQ full), the firmware asserts `GENERAL_ERROR_FLAG` and resets the PEQ — that event is **lost**. This is an error condition, not normal operation. It is prevented by two mechanisms: (1) pileup rejection in the channel logic limits the rate of `ACCEPTED_HIT` pulses, and (2) the throttle mechanism signals the MTRG to raise the trigger threshold if the downstream readout FIFO approaches full, reducing the overall hit rate before the PEQ saturates.

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

| Module | README | Details |
|--------|--------|---------|
| Master Trigger | [MTRG/README.md](MTRG/README.md) | System overview, device table, layout |
| MTRG Main FPGA (ISE) | [MTRG/MAIN_FPGA.md](MTRG/MAIN_FPGA.md) | Virtex-4, full source listing, register map |
| MTRG Main FPGA (Vivado) | [MTRG/VIVADO_MAIN_FPGA.md](MTRG/VIVADO_MAIN_FPGA.md) | Kintex UltraScale port, IP core differences |
| MTRG VME FPGA | [MTRG/VME_FPGA.md](MTRG/VME_FPGA.md) | Spartan-3, VME slave, FPGA config manager |
| MTRG CPLD | [MTRG/DGS_CPLD.md](MTRG/DGS_CPLD.md) | XC9500XL, fast strobe multiplicity logic |
| Router | [RTRG/README.md](RTRG/README.md) | Virtex-4, full source listing, register map |
| Digitizer | [DIG/README.md](DIG/README.md) | Spartan-3, all build branches, ADC interface |

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
| Timestamp resolution | ~1 ns (vernier TDC in MTRG) |
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

All VME FPGAs share the same architecture (Spartan-3 XC3S400): they present an A32/D32 VME slave interface, store the main FPGA bitstream in external flash, and program the main FPGA via serial configuration on power-up or on demand.

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

## Building the Firmware

See **[BUILDING.md](BUILDING.md)** for the full build guide, including:

- Toolchain versions for each module (ISE 14.7 / Vivado 2018.3)
- ISE 14.7 device support (Virtex-4, Spartan-3 confirmed)
- Running ISE 14.7 on Ubuntu 24.04 via Docker/Podman container
- LD_PRELOAD native patch alternative

---
*Source: `DGS_tools_pack/fpga/` and `DGS_tools_pack/FPGA/` (gitlab.phy.anl.gov/dgs-tools-pack). Created: 2026-04-05.*
