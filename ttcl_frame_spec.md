# TTCL — Frame Specification (Section 7)

Stability: C3 - Structural / stable

**Part of the TTCL documentation series.** This file covers the per-frame wire-format details for all 20 frames in the Trigger Timing and Control Link (TTCL) 2 µs system cycle.

_For overview, physical/electrical specs, protocol layers, and initialization → see [ttcl.md](ttcl.md)._

**Source:** `20160418 trig command link.pdf` (v2.1, last revised Dec 3, 2013)
**Authors:** J. Anderson / S. Zimmermann

---

## Table of Contents

- [Frame 1 — Sync Command](#frame-1--sync-command)
- [Frame 2 — Debugging Control Command](#frame-2--debugging-control-command)
- [Frames 3–10 — Trigger Decision Commands](#frames-310--trigger-decision-commands)
- [Frame 11 — Spare](#frame-11--spare)
- [Frame 12 — Internal Trigger Command](#frame-12--internal-trigger-command)
- [Frame 13 — Demand Front End Slow Data (GRETINA only)](#frame-13--demand-front-end-slow-data-gretina-only)
- [Frame 14 — Internal Trigger Command (DGS/DFMA only)](#frame-14--internal-trigger-command-dgsdfma-only)
- [Frame 15 — GRETINA Front End Asynchronous Command](#frame-15--gretina-front-end-asynchronous-command)
- [Frame 16 — DGS Synchronous System Capture Command](#frame-16--dgs-synchronous-system-capture-command)
- [Frame 17 — Auxiliary Detector Commands](#frame-17--auxiliary-detector-commands)
- [Frames 18 & 19 — Null Command](#frames-18--19--null-command)
- [Frame 20 — End-of-Cycle](#frame-20--end-of-cycle)
- [Cross-References](#cross-references)

---

## Frame 1 — Sync Command

**Purpose:** Marks the start of the 2 µs system master timing interval; commands all front ends to compare their internal timestamp counter to the transmitted value.

| Word | Bits 16..9 | Bits 8..1 |
|------|-----------|----------|
| 1 | 0x01 (Sync) or 0x81 (Imperative Sync) | Rollover Notification |
| 2 | System Timestamp[47..40] | System Timestamp[39..32] |
| 3 | System Timestamp[31..24] | System Timestamp[23..16] |
| 4 | System Timestamp[15..8] | System Timestamp[7..0] |
| 5 | 0x0000 | |

- On timestamp mismatch: front end latches internal "out of sync" status
- **Rollover Notification byte** (bits 8..1 of word 1):
  - 0xFF = timestamp has rolled over (stays 0xFF until timestamp reaches 0x000000010000)
  - 0x00 = no rollover (or past 0x0000000FFFF since last rollover)
  - Normal rollover time: >30 days; systemic resets expected more frequently
- Routers **must** synchronize to timestamp and pass it on to all links (A-H, L, R, U)
- **Imperative Sync (0x81):** All front ends immediately reset timestamps and buffer counts, synchronous with receipt of the next Command Byte
- When not running off SERDES recovered clock: MγRIADs, Digitizers, Digitizer Testers must generate their own local timestamp

**DGS/DFMA Sync additions:**
- Link L Propagation Control Register controls whether Master distributes its own timestamp or the remote Master's timestamp (via Link L) — only takes effect when Link L SERDES is locked, state machine is locked to a command stream, and clock-source control bit is set
- Master Trigger provides user-settable registers for the timestamp value sent during Imperative Sync (default: 0)
- Routers and Digitizers reset to the **transmitted** timestamp (not assumed to be zero)

---

## Frame 2 — Debugging Control Command

- **GRETINA:** TTCS can suspend normal TTCL and send arbitrary data from a VME-loaded FIFO
  - Sync frame (#1) and End-of-Cycle frame (#20) always sent normally; frames 3–19 derived from FIFO (85 words)
  - Continues until FIFO empty; optional continuous recycle mode
  - Trigger pipelines and Fast Strobe multiplicity counts cleared on entry; System Timestamp unaffected
- **DGS/DFMA:** Debugging Control Command **removed** — redefined as a **Null frame**

---

## Frames 3–10 — Trigger Decision Commands

All frames 3–10 are identically formatted:

| Word | Bits 16..9 | Bits 8..1 |
|------|-----------|----------|
| 1 | Trigger Type Code | Front End Selection |
| 2 | Event Timestamp[47..40] | Event Timestamp[39..32] |
| 3 | Event Timestamp[31..24] | Event Timestamp[23..16] |
| 4 | Event Timestamp[15..8] | Event Timestamp[7..0] |
| 5 | 0x0000 | |

- Receipt of Trigger Decision command = **positive** decision (no command issued if event doesn't meet criteria)
- Non-triggerable events simply flush out of buffer memories in front-end devices, given sufficient time
- Trigger Decision carries **Event Timestamp** = timestamp of when the trigger system determined the trigger occurred (NOT the current system timestamp)

**GRETINA Trigger Type Codes:**
| Code | Trigger Type | Event Timestamp |
|------|-------------|----------------|
| 0x55 | Multiplicity trigger from "fast" data (Fast Strobe fired) | System Timestamp when Fast Strobe changed state |
| 0x5A | Auxiliary detector input only | System Timestamp when auxiliary detector edge sensed |
| 0xA5 | Combination (energy sum, pattern, etc.) | Lowest (earliest) timestamp of all included data |

**DGS/DFMA Trigger Type Codes (actual firmware values):**
| Code | Trigger Type | Event Timestamp |
|------|-------------|----------------|
| 0x50 | Local trigger algorithm 1 (CPLD fast strobe) | System Timestamp at decision time | ✅ verified 2026-04-17 — `VIVADO_MAIN_FPGA/trunk/Source/top.vhd:L2240`; `MAIN_FPGA/trunk/Source/last_manual_top.vhd:L2341`
| 0x51 | Local trigger algorithm 2 (sum_hits_X) | Same | ✅ verified 2026-04-17 — `top.vhd:L2288`; `last_manual_top.vhd:L2389`
| 0x52 | Local trigger algorithm 3 (sum_hits_X) | Same | ✅ verified 2026-04-17 — `top.vhd:L2339`; `last_manual_top.vhd:L2440` (comment: "change trigger type code from 02 to 03" — codes were changed from 0x01–0x05 to 0x50–0x54)
| 0x53 | Local trigger algorithm 4 (sum_hits_XY) | Same | ✅ verified 2026-04-17 — `top.vhd:L2390`; `last_manual_top.vhd:L2491`
| 0x54 | Local trigger algorithm 5 (CPLD fast strobe) | Same | ✅ verified 2026-04-17 — `top.vhd:L2435`; `last_manual_top.vhd:L2536`
| 0x56 | GITMO trigger (external GS) | Same | ✅ verified 2026-04-17 — `GITMO_TRIGGER.vhd:L279`
| 0x6X | Re-propagated trigger from Link L (X = remote algo type[2:0]) | Same | ✅ verified 2026-04-17 — `remote_trig_support.vhd:L388` (`TRIGGER_SUBTYPE <= X"6" & '0' & REMOTE_TRIG_TYPE`)
| 0x7X | Re-propagated trigger from Link R/U (delay mode; X = type[2:0]) | Same | ✅ verified 2026-04-17 — `remote_trig_support.vhd:L411` (`TRIGGER_SUBTYPE <= X"7" & '0' & REMOTE_TRIG_TYPE`)
| 0x00 | External input (NIM via GITMO, no timestamp match) | System Timestamp at edge detection | ✅ verified 2026-04-17 — `GITMO_TRIGGER.vhd:L204` (`TRIGGER_SUBTYPE <= X"00"`)

> **⚠️ Note:** The TTCL spec PDF (v2.1, 2013) lists codes 0x01–0x05 for local algorithms 1–5. The actual firmware uses **0x50–0x54** — the codes were deliberately changed. The DIG `SERDES_RX_Mach.vhd` confirms: "Local triggers are, by fiat, 0x5nxx (n=0-7). Remote trigs are 0x6nXX." ✅ verified 2026-04-17 — `DIG SERDES_RX_Mach.vhd:L735` (comment at TRIGGER_DECISION_FRAME_WORD_1 state)

**Front End Selection byte:**
- 0x00 = broadcast trigger (all front ends respond)
- Non-zero = only the front end whose Crystal ID matches responds
- **Note:** Both GRETINA and DGS/DFMA digitizers currently ignore this byte

**Multiple triggers per cycle:**
- Up to 8 trigger algorithms, each with its own FIFO buffer
- Collection state machine collects up to 1 trigger per enabled algorithm per 2 µs period
- Issued in order: first trigger in Frame 3, second in Frame 4, etc.
- No more than 1 trigger per algorithm per 2 µs cycle; extras queue in algorithm FIFO for next cycle
- Unused frames replaced by null frames
- Per-algorithm decision time: ~120 ns (time to store in FIFO)

**DGS/DFMA Selective Propagation (Frames 8, 9, 10):**
- Frame 8: reserved for triggers from Link L
- Frame 9: reserved for triggers from Link R
- Frame 10: reserved for triggers from Link U
- Each Master implements a "trigger algorithm" collecting triggers from remote Masters via those frames
- Control bits per-algorithm enable/disable collection from specific trigger frames (3–10) from each remote Master
- Triggers are **never echoed back** to the sender (no deadly embrace)

---

## Frame 11 — Spare

Always null command frame. Reserved for future use.

---

## Frame 12 — Internal Trigger Command

- Normally Null frame; Master may send non-null to cause **synchronous resets** of diagnostic counters and FIFOs in other trigger modules
- **Routers:** Internally process Frame 12, but **always send Null** to digitizers (links A-H). Pass unprocessed Frame 12 to links R and U (for Data Generators/test modules)
- **Digitizers:** May always assume Frame 12 = Null

| Word | Content |
|------|---------|
| 1 | Command Byte (0x00/0x01) |
| 2 | Router Counter Resets Bitmask |
| 3 | Router FIFO Resets Bitmask |
| 4 | Data Generator Resets Bitmask |
| 5 | 0x0000 |

**Router Counter Resets (bits 0:7):** Reset counters at addresses 0x12C, 0x130, …, 0x148 ✅ verified 2026-04-23 — RTRG `registers.vhd:L151-158,L567-577` (REG_12C_IN through REG_148_IN confirmed as real VME addresses)
**Router FIFO Resets:**
- Bits 0:7: Clear channel-specific FIFOs at 0x180, 0x184, …, 0x19C ✅ verified 2026-04-23 — RTRG `registers.vhd:L591-598,L799-806` (CHAN_MON_FIFO_OUTs(1)–(8) at 0x0180–0x019C)
- Bits 8:15: Clear board-wide "monitor" FIFOs at 0x160, 0x164, …, 0x17C ✅ verified 2026-04-23 — RTRG `registers.vhd:L583-590,L788-795` (MON_FIFO_OUTs(1)–(8) at 0x0160–0x017C)

**Data Generator Resets (bits 3:0):**
- Bit 0: Reset event generation state machine (restart pattern from beginning)
- Bit 1: Clear all diagnostic counters
- Bit 2: Clear all channel diagnostic FIFOs
- Bit 3: Clear all board-wide "monitor" diagnostic FIFOs

Trigger issued synchronously with a known timestamp → system-wide "snapshot" of trigger activity.

---

## Frame 13 — Demand Front End Slow Data (GRETINA only)

| Word | Bits 16..9 | Bits 8..1 |
|------|-----------|----------|
| 1 | 0x40 | 0xFB |
| 2 | 0xA5A5 | |
| 3 | 0x5A5A | |
| 4 | 0xA5A5 | |
| 5 | 0xA5A5 | |

- In GRETINA: Trigger Data link continuously sends "fast" data (discriminator bits); this command triggers front ends to send "slow" data (energy sums, segment hit patterns)
- Issued at a regular rate every 2 µs without interruption
- A Trigger Data frame always issued by front ends in response (even if no segments hit)
- **DGS/DFMA:** Digitizers ignore Frame 13 (DGS Trigger Data link is all "fast" data; no "slow" data). DGS Master retains GRETINA format for backwards compatibility.

---

## Frame 14 — Internal Trigger Command (DGS/DFMA only)

- Similar to Frame 12 but targets **Digitizer Tester** boards and **MγRIAD** boards
- GRETINA trigger boards do not implement Frame 14 (only send Null)
- Routers replace Frame 14 with Null when retransmitting to digitizers
- Frame 14 as sent to digitizers = Null (0xAAAA / 0x0000 format)
- When non-null: Command Byte + Timestamp Bitmask (matches TS bits 25:10) + Pulse Count + Pulse Delay

**Frame 14 Command Byte values for Digitizer Tester:**
| Code | Action |
|------|--------|
| 0x00 | Stop sending pulses |
| 0x01 | Issue single pulse immediately |
| 0x02 | Enter Loop mode, begin issuing signals now |
| 0x03 | Cease Loop mode at end of next buffer set |
| 0x04 | Begin issuing pulses tied to timestamp (uses Timestamp Bitmask, Pulse Count, Pulse Delay) |
| 0x05–0x0F | Reserved |
| 0xAA | Null (no command) |

Pulse Delay units: 100s of ns (time from end of pulse n to beginning of pulse n+1).

---

## Frame 15 — GRETINA Front End Asynchronous Command

- Normally null; sent only in response to slow control system requests
- Implemented via a **256-word deep, 16-bit wide FIFO** (written from VME)
- Slow control writes arbitrary commands; interpretation left to the Digitizer
- **DGS digitizer firmware ignores Frame 15**

**Suggested command formats (GRETINA-specific):**

**Calibration Inject (0x04 or 0x08):**
- Addressing Control = 0x00: all front ends inject calibration
- Non-zero: 8 bytes of selection bitmask (up to 64 front ends)

**Latch Status (0x08):**
- All front ends save a status record for slow control readout
- Any front end error on Trigger Input Data link should force this command

**Front End Reset (0x10):**
- Full reset of selected digitizers
- TTCS masks Trigger Data input from selected front ends until slow control releases mask (synchronous with next Sync command)

**Front End Links-only Reset (0x18):**
- Partial reset: only serial Trigger Data output links reset
- Handled identically to 0x10 from TTCS perspective

**External Discriminator Request (0x22) — Added 20160418, DGS:**
- Fires a discriminator synchronously across multiple digitizer channels
- Selection: User Package Data AND mask + OR mask (10-bit channel select bitmask per digitizer)
- OR mask 0xFFF = broadcast to all digitizers
- Latched mode: Set Latch (SL), Clear Latch (CL), Enable Latch (EL), Disable Latch (DL) bits
  - Default: non-latched (External Discriminator bit asserted for one clock cycle per frame)
  - EL bit: enter latched mode; SL/CL bits control set/clear; DL bit: return to non-latched
  - Intended for use with DGS digitizer "AND" mode to gate internal discriminators on/off

---

## Frame 16 — DGS Synchronous System Capture Command

- Normally null; issued by writing to a Master Trigger register
- All devices below Master in hierarchy respond (Routers, Digitizers, Data Generators, Digitizer Testers)
- MγRIADs and peer Masters only respond if programmed via their propagation control registers

**Format:**
| Word | Content |
|------|---------|
| 1 | Any value ≠ 0xAA (command active) |
| 2 | Timestamp[15:0] |
| 3 | Capture Length |
| 4 | FIFO Capture Delay |
| 5 | 0x0000 |

**Operation:**
- All rate counters zeroed when device timestamp matches Timestamp[31:0] in command
- Counters enabled for (Capture Length × 65.536 µs); max ~4.3 seconds ✅ verified 2026-04-23 — arithmetic: 65535 × 65536 ns = 4.295 s; 16-bit register max → ~4.3 s cap
- A Capture Length of 15,258 = ~1 second collection ✅ verified 2026-04-23 — 15258 × 65536 ns = 999,948,288 ns ≈ 1.000 s
- FIFOs reset immediately at timestamp match; start filling after FIFO Capture Delay (in same units as Capture Length)
  - Example: Capture Length = 15,258 (≈1 sec), FIFO Capture Delay = 12,250 → FIFOs start at 0.802 sec after start
- "Capture complete" status bit set when collection window ends

**Master Trigger counters (Frame 16):**
- 16 counters total (8 × `TRIG_RATE_COUNTERs` + 8 × `RAW_TRIG_RATE_COUNTERs`); reset and captured synchronously during same interval as front ends ✅ verified 2026-04-24 — `MTRG/top.vhd:L4494-4516` (RATE_COUNTER_BLOCK generate loop)
- First 8 (`TRIG_RATE_COUNTERs`, VME 0x2000–0x203C): count `ENABLED_NONVETOED_TRIG_ACK` — triggers actually sent to digitizers (veto-filtered)
- Second 8 (`RAW_TRIG_RATE_COUNTERs`, VME 0x2040–0x207C): count `ENABLED_TRIG_ACK` — algorithm satisfied, whether or not vetoed
- ⚠️ **Correction:** KB previously stated "second 8 = blocked by throttle/veto" — this is wrong. Neither counter directly measures blocked triggers. Dead-time estimate = (RAW − TRIG) / RAW per type. ✅ verified 2026-04-28 — `MTRG/top.vhd:L4494-4516` (RATE_COUNTER_BLOCK generate; TRIG_RATE_COUNTERs counts ENABLED_NONVETOED_TRIG_ACK, RAW_TRIG_RATE_COUNTERs counts ENABLED_TRIG_ACK)
- These counters are **separate** from the running counters used by Frame 12

---

## Frame 17 — Auxiliary Detector Commands

- Generic commands; format indeterminate
- Implemented identically to Frame 15 (Front End Async Command) with its own 256-word Auxiliary Command FIFO
- Reserved for **non-digitizer front-end devices** (not MγRIAD; for MγRIAD use Frame 14)

---

## Frames 18 & 19 — Null Command

- Always null (0xAAAA / 0x0000 pattern)
- 0xAA command byte preferred for all null commands (DC balance + easily oscilloscope-recognizable)

---

## Frame 20 — End-of-Cycle ✅ verified 2026-04-07 — SERDES_RX_Mach_R2.vhd:L125,L145 (F20W1..F20W5 states, "Frame 20 is the End-of-cycle frame")

| Word | Bits 16..9 | Bits 8..1 |
|------|-----------|----------|
| 1 | 0xFF | 0xFF |
| 2 | 0x0000 | |
| 3 | 0xFFFF | |
| 4 | 0x0000 | |
| 5 | 0x5555 | |

✅ verified 2026-04-23 — word pattern confirmed: RTRG `SERDES_RX_Mach_R2.vhd:L246` (`X"FFFF", X"0000", X"FFFF", X"0000", X"5555"` — Frame 20 Fixed End-of-Cycle frame)

- Special form of null command; marks frame boundary
- Pattern chosen to be easily distinguishable in data dumps

---

*Documented from: 20160418 trig command link.pdf, v2.1*
*Split from ttcl.md: 2026-04-27*

## Cross-References

- `knowledgeBase/ttcl.md` — TTCL overview, physical/electrical specs, protocol layers (sections 1–6, 8)
- `knowledgeBase/fpga.md` — FPGA firmware overview; TTCL role in the 3-tier trigger hierarchy
- `knowledgeBase/deep_fpga_MTRG_MAIN.md` — MTRG Main FPGA: 20-frame command generation
- `knowledgeBase/deep_fpga_RTRG.md` — RTRG firmware: TTCL reception, throttle logic
- `knowledgeBase/deep_fpga_DIG.md` — DIG firmware: TTCL reception, trigger decision window
- `knowledgeBase/data_structures.md` — GEB binary data format; TTCL-driven event timestamps
- `knowledgeBase/multi_system_linking.md` — Multi-system TTCL ImpSync and Selective Propagation
