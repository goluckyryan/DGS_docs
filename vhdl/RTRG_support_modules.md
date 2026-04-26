# RTRG Support Modules — VHDL Deep-Dive

Stability: C3 - Structural / stable

**Source directory:** `FPGA/RTRG/Firmware/DGS_Version/MAIN_FPGA/Source/`
**Date documented:** 2026-04-25
**Author (original VHDL):** John Anderson (throttle units); MBO / unknown (overlap_machine, channel_resets, plane_bit_count)

This file documents the RTRG FPGA support modules that are not covered in `260E_trigger_scheme.md` (which covers `chan_in.vhd`, `disc_mach.vhd`, and `router_data_path.vhd`).

---

## Table of Contents

1. [overlap_machine](#1-overlap_machine--overlap_machvhd)
2. [throttle_monos](#2-throttle_monos--throttle_monosvhd)
3. [throttle_limiters](#3-throttle_limiters--throttle_limitersvhd)
4. [channel_resets (CHANNEL_RESETS)](#4-channel_resets--channel_resetsvhd)
5. [plane_bit_count](#5-plane_bit_count--plane_bit_countvhd)
6. [link_init](#6-link_init--link_initvhd)

---

## 1. overlap_machine — `overlap_mach.vhd`

**Purpose:** Generic 2-input coincidence detector. Asserts `OVERLAP_OCCURRED` for one clock tick if signals A and B both fire within a programmable window of each other.

**Entity ports:**

| Port | Dir | Type | Description |
|------|-----|------|-------------|
| CLK | in | std_logic | 50 MHz board clock |
| RST | in | std_logic | Global reset |
| SIG_A | in | std_logic | Input A (e.g., HPGe discriminator) |
| SIG_B | in | std_logic | Input B (e.g., BGO Compton shield) |
| OVERLAP_DELAY | in | std_logic_vector(6:0) | Coincidence window; loaded into 7-bit countdown — window = OVERLAP_DELAY+1 ticks → max 128 × 20ns = **2.56 µs** @ 50 MHz ✅ verified 2026-04-25 — overlap_mach.vhd:L21 comment "up to 2.56us"; L104 countdown; ⚠️ Correction: prior KB said 2.54 µs (127 ticks) — actual max is 2.56 µs (128 ticks, timer fires at = 0) |
| OVERLAP_OCCURRED | out | std_logic | One-tick pulse on confirmed overlap |

**State machine:** 4 states ✅ verified 2026-04-25 — overlap_mach.vhd:L33 (`type OVERLAP_MACH_STATES is (ST_IDLE, ST_OVERLAP_A_FIRST, ST_OVERLAP_B_FIRST, ST_OVERLAP_OCCURRED)`)

| State | Meaning |
|-------|---------|
| ST_IDLE | Waiting for either edge |
| ST_OVERLAP_A_FIRST | A fired; waiting for B within OVERLAP_DELAY clocks |
| ST_OVERLAP_B_FIRST | B fired; waiting for A within OVERLAP_DELAY clocks |
| ST_OVERLAP_OCCURRED | Both fired → assert output one tick, return to IDLE |

**Key behaviours:**
- Rising-edge detection via 1-clock pipe (`SIG_A_PIPE`/`SIG_B_PIPE`) ✅ verified 2026-04-25 — overlap_mach.vhd:L37,L47,L50
- Simultaneous A+B edges → jump directly to `ST_OVERLAP_OCCURRED` (skip wait states) — comment: *"Do not pass Go. Do not collect $200."* (MBO 20140610) ✅ verified 2026-04-25 — overlap_mach.vhd:L76,L90,L113
- Timer counts down from `OVERLAP_DELAY` in `ST_OVERLAP_A/B_FIRST`; retriggers if the same signal fires again (reloads timer from `OVERLAP_DELAY`) ✅ verified 2026-04-25 — overlap_mach.vhd:L102,L104,L125,L127
- `OVERLAP_OCCURRED` is de-asserted while waiting, asserted for exactly **1 clock tick** upon confirmed overlap ✅ verified 2026-04-25 — overlap_mach.vhd:L131-133 (ST_OVERLAP_OCCURRED → ST_IDLE on next clock)

**Context:** ⚠️ **NOT instantiated anywhere in RTRG.** ✅ verified 2026-04-25 — exhaustive grep across all `~/FPGA_svn2git/RTRG_git/MAIN_FPGA/Source/*.vhd` finds zero uses of `overlap_machine`. Despite the note in `RTRG_overlap_mach.md` implying disc_mach.vhd uses this, disc_mach.vhd implements its own inline coincidence detection and does NOT import or instantiate overlap_machine. This module exists in source as a reusable standalone but is never wired in.

---

## 2. throttle_monos — `throttle_monos.vhd`

**Purpose:** Simple (non-gated) throttle stretcher. Receives 8 SPARE_LVDS bits (one per DIG) and stretches each to a guaranteed 2 µs minimum. Aggregates to `ANY_THROTTLE_REQ_OUT` with a separately programmable monostable width.

**Entity ports:**

| Port | Dir | Type | Description |
|------|-----|------|-------------|
| mclk | in | std_logic | 50 MHz master clock |
| mrst | in | std_logic | Global reset |
| xSPARE_LVDS(8:1) | in | std_logic_vector | 8 throttle bits from DIGs |
| ANY_THROTTLE_WIDTH_REG(15:0) | in | std_logic_vector | Monostable width for ANY_THROTTLE_REQ_OUT |
| MISC_CTL2_REG(15:0) | in | std_logic_vector | bit 1: force all throttle on; bit 0: block all throttle ✅ verified 2026-04-25 — throttle_monos.vhd:L84-87 (MISC_CTL2_REG(1) = force-on, MISC_CTL2_REG(0) = block); ⚠️ L46 header comment erroneously says "bit 9: set all on, bit 8: block all" — stale/wrong; code uses bits 1 and 0 |
| INPUT_LINK_MASK_REG(15:0) | in | std_logic_vector | Per-channel mask (bit=1 → suppress) |
| ANY_THROTTLE_REQ_OUT | out | std_logic | OR of all stretched throttle bits, further stretched by ANY_THROTTLE_WIDTH_REG |
| THROTTLE_REQUEST_OUT(7:0) | out | std_logic_vector | 2 µs stretched per-channel throttle |
| THROTTLE_STATUS(15:0) | out | std_logic_vector | [15:8] = raw xSPARE_LVDS, [7:0] = masked THROTTLE_REQUESTS |

**Throttle input logic (per-bit priority table):**

| MISC_CTL2(1) force-on | MISC_CTL2(0) force-off | LINK_MASK(x) | Net effect |
|-----------------------|------------------------|--------------|------------|
| 0 | 0 | 0 | Pass through DIG bit |
| 0 | 0 | 1 | Always zero (masked) |
| 0 | 1 | x | Always zero (all blocked) |
| 1 | x | x | Always one (all forced on) |

**Per-channel monostable:**
- Counter `COUNTER_START` = 400 decimal = 2 µs @ 50 MHz ✅ verified 2026-04-25 — throttle_monos.vhd:L60 (`constant COUNTER_START : std_logic_vector(10 downto 0) := "00110010000"; --400 decimal (2us @ 50MHz)`)
- Retriggerable: if DIG bit still asserted, counter reloads immediately
- Output (`THROTTLE_REQUEST_OUT[i]`) de-asserts only when counter reaches zero and input is clear

**ANY_THROTTLE monostable:**
- Triggers on OR of all 8 `THROTTLE_REQUESTS`
- Width programmable via `ANY_THROTTLE_WIDTH_REG` (16-bit counter, units = 20 ns @ 50 MHz)

**Note:** This is the simpler "monos-only" variant (no limiter logic). The upgraded variant is `throttle_limiters.vhd`.

---

## 3. throttle_limiters — `throttle_limiters.vhd`

**Purpose:** Enhanced throttle logic adding a programmable hold-off timer before asserting throttle to the Master. This prevents nuisance throttle assertions from the LBNL digitizer's FIFO half-full flag (which doubles as the VME readout controller trigger) — the IOC should drain the FIFO before the throttle propagates to the MTRG.

**Why it exists:** The DIG board-wide FIFO half-full flag serves dual duty — it triggers VME readout and requests throttle. Under normal operation the IOC responds in time; the limiter only passes throttle if the IOC has failed to drain the FIFO within the configured hold-off.

**Entity ports (additions over throttle_monos):**

| Port | Dir | Type | Description |
|------|-----|------|-------------|
| THROTTLE_LIMIT_TIME_REG(15:0) | in | std_logic_vector | [10:0] count; [15:14] timer rank select |
| ALTERNATE_THROTTLE_REQ | out | std_logic | OR of stretched limited throttle bits (alternate ANY output) |
| SELECTED_THROTTLE | out | std_logic | Mux output for NIM2 diagnostic; index = DIAG_PIN_CTL_REG[5:2] into INT_THROTTLE_STATUS |
| LIMIT_MACH_ACTIVE(7:0) | out | std_logic_vector | Per-channel limiter-active flags (ILA / debug) |
| DIAG_LIMIT_FLAGS(3:0) | out | std_logic_vector | The four LIMIT_TIMER_FLAG outputs for ILA |

**Limit state machine (per channel, 8× instantiated via `generate`):**

| State | Description |
|-------|-------------|
| IDLE | Wait for THROTTLE_REQUESTS(i) |
| REQUEST_RECEIVED | Count down LIMIT_COUNTs(i); go back to IDLE if request removed, advance to ASSERT_THROTTLE when countdown hits 0 |
| ASSERT_THROTTLE | Assert LIMITED_THROTTLE_REQUESTS(i) until request removed |

**Limit timer hierarchy (cascaded ripple):**

| Timer | Period | Use |
|-------|--------|-----|
| LIMIT_TIMER (10-bit @ 50 MHz) | 20.48 µs | Base tick (LIMIT_TIMER_FLAG) |
| LIMIT_TIMER2 (10-bit @ LIMIT_TIMER_FLAG) | 20.97 ms | LIMIT_TIMER_FLAG2 |
| LIMIT_TIMER3 (10-bit @ LIMIT_TIMER_FLAG2) | 21.47 s | LIMIT_TIMER_FLAG3 |
| LIMIT_TIMER4 (10-bit @ LIMIT_TIMER_FLAG3) | ~6 hours | LIMIT_TIMER_FLAG4 |

`THROTTLE_LIMIT_TIME_REG[15:14]` selects which timer rank drives `LIMIT_COUNTs` decrement:
- `"00"` → LIMIT_TIMER_FLAG (20.48 µs ticks, max hold-off = 2048 × 20.48 µs ≈ 42 ms)
- `"01"` → LIMIT_TIMER_FLAG2 (20.97 ms ticks, max ≈ 43 s)
- `"10"` → LIMIT_TIMER_FLAG3 (21.47 s ticks, very long)
- `"11"` → LIMIT_TIMER_FLAG4 (hours — effectively disabled)

**Post-limiter monostable:** Same as `throttle_monos` — `COUNTER_START` = 400 = 2 µs. Applied to `LIMITED_THROTTLE_REQUESTS` to guarantee propagation.

**THROTTLE_STATUS layout:**
- `[15:8]` = raw `THROTTLE_REQUESTS` (post-mask, pre-limit)
- `[7:0]` = `LIMITED_THROTTLE_REQUESTS` (post-limit, pre-stretch)

**SELECTED_THROTTLE:** Mux driven by `DIAG_PIN_CTL_REG[5:2]` — selects any single bit of `INT_THROTTLE_STATUS[15:0]` for NIM2 output (diagnostic / oscilloscope monitoring).

---

## 4. channel_resets — `channel_resets.vhd`

**Purpose:** Generates synchronous reset signals for each of the 8 RTRG input channel pipelines (one per DIG link, A–H). Handles clock-domain crossing for the reset release.

**Entity ports:**

| Port | Dir | Type | Description |
|------|-----|------|-------------|
| GLOBAL_RESET | in | std_logic | Board-wide reset |
| switched_master_clock | in | std_logic | Board-wide 50 MHz clock |
| ROUTER_LOCKED | in | std_logic | RTRG locked to MTRG (high = locked) |
| PULSED_CTL1_REG(15:0) | in | std_logic_vector | bit 15: write 1 to pulse reset all pipelines |
| MISC_CTL2_REG(15:0) | in | std_logic_vector | bits [7:0]: per-channel hold-in-reset (bit=1 → channel permanently held in reset) |
| LINKA_RCLK … LINKH_RCLK | in | std_logic | Individual channel SERDES receive clocks (one per link) |
| CHANNEL_FIFO_RESETS(7:0) | out | std_logic_vector | Per-channel pipeline reset signals |

**Reset trigger conditions (any one → assert RESET_ALL_PIPELINES):** ✅ verified 2026-04-25 — channel_resets.vhd:L45-56
1. `GLOBAL_RESET = '1'` ✅ L45
2. `ROUTER_LOCKED = '0'` (RTRG not locked to MTRG → keep in reset) ✅ L45
3. `PULSED_CTL1_REG(15) = '1'` → loads `PIPELINE_RESET_COUNTER <= "1000"` (8 decimal), counts down to 0, holding reset for 8 master clocks then releases ✅ L49-56

**Clock-domain crossing:** `RESET_ALL_PIPELINES` is generated on `switched_master_clock`. Each channel resamples it on its own `LINK{x}_RCLK` to ensure release is synchronous with the SERDES receive clock feeding that pipeline FIFO. ✅ verified 2026-04-25 — channel_resets.vhd:L66-72 (CHAN1_RESET_PROC clocked on LINKA_RCLK; release: `CHANNEL_FIFO_RESETS(0) <= '0'` synchronous with LINKA_RCLK)

**Per-channel hold-in-reset:** `MISC_CTL2_REG[i] = 1` → `CHANNEL_FIFO_RESETS(i)` stays asserted regardless of RESET_ALL_PIPELINES. Used when a DIG input link is not populated to prevent spurious FIFO noise. ✅ verified 2026-04-25 — channel_resets.vhd:L69 (MISC_CTL2_REG(0) → CHANNEL_FIFO_RESETS(0)), L80 (bit 1 → ch1), L91 (bit 2 → ch2)

---

## 5. plane_bit_count — `Plane_bit_count.vhd`

**Purpose:** Combinatorial popcount (population count / Hamming weight) lookup table for a 10-bit input. Returns the number of set bits as a 4-bit value (max = 10 = `"1010"`).

**Entity ports:**

| Port | Dir | Type | Description |
|------|-----|------|-------------|
| CLK | in | std_logic | Clock (unused — output is purely combinatorial) |
| D_IN(9:0) | in | std_logic_vector | 10-bit input pattern |
| Count_out(3:0) | out | std_logic_vector | Number of set bits in D_IN |

**Implementation:** 1024-entry VHDL `when/else` truth table — fully unrolled combinatorial logic (no registers). ISE synthesizes this as a LUT cascade. ✅ verified 2026-04-25 — Plane_bit_count.vhd:L3 comment "no registers, totally combinatorial"; L25 (`xCount_out(3 downto 0) <=` pure concurrent assignment, no process)

**Context:** Used in `chan_in.vhd` (NOT in `router_data_path.vhd` as previously believed) ✅ verified 2026-04-25 — chan_in.vhd:L163 (component declaration), L502 (U2: X_SELECT → X_PLANE_COUNT), L509 (U3: Y_SELECT → Y_PLANE_COUNT). Two instances count X-plane and Y-plane hit multiplicity at the output of the channel's disc_mach/selector logic. These counts (`X_PLANE_COUNT`, `Y_PLANE_COUNT`) feed the trigger decision pipeline. See `260E_trigger_scheme.md`.

**Note:** Despite having a `CLK` port, the architecture comment and code confirm the output (`xCount_out`) is entirely combinatorial — `CLK` is vestigial / structural. ✅ verified 2026-04-25 — Plane_bit_count.vhd:L12 (CLK in port), L3 comment "no registers", no process block or CLK'event in the architecture

---

## Relationships and Context

```
RTRG MAIN FPGA
├── chan_in.vhd              — SERDES reception, DPRAM alignment [see 260E_trigger_scheme.md]
│   └── Plane_bit_count.vhd ← popcount LUT (X_PLANE_COUNT + Y_PLANE_COUNT) [this file] ✅ verified 2026-04-25 — chan_in.vhd:L502,L509
├── disc_mach.vhd            — clean/dirty/BGO classification [see 260E_trigger_scheme.md]
│   (inline overlap detection — does NOT use overlap_mach.vhd)
├── overlap_mach.vhd        — ← HPGe/BGO coincidence detector [this file] ⚠️ NOT instantiated anywhere in RTRG ✅ verified 2026-04-25
├── router_data_path.vhd    — multiplicity aggregation, TTCL output [see 260E_trigger_scheme.md]
├── channel_resets.vhd      — ← per-channel pipeline reset (CDC) [this file]
├── throttle_monos.vhd      — ← simple DIG throttle stretcher [this file]
├── throttle_limiters.vhd   — ← enhanced throttle with hold-off timer [this file]
├── link_init.vhd            — ← SERDES initialization FSM [this file, §6]
├── registers.vhd            — VME register map
├── timestamp.vhd            — timestamp logic
├── dc_balance_mach.vhd     — DC balance
└── TOP.VHD                  — top-level instantiation
```

---

## 6. link_init — `link_init.vhd`

**Purpose:** SERDES link initialization state machine for the RTRG. Drives DEN (transmitter enable), REN (receiver enable), and SYNC to all 8 downward SERDES links (to digitizers), waits for all links to assert LOCK*, then acknowledges and releases SYNC to allow normal data flow.

**Author:** John T. Anderson (JTA)  
**Original project:** GRETINA; adapted for DGS RTRG  
**FORCE_SYNC addition:** 2012-02-28 JTA (allows software override of SYNC per-link)

**Entity ports:**

| Port | Dir | Width | Description |
|------|-----|-------|-------------|
| CLK | in | 1 | 50 MHz board clock |
| RST | in | 1 | Power-on reset |
| RETRY | in | 1 | If asserted: holds EN_SERDES; re-INITs from ERROR |
| LOCK_ACK | in | 1 | Software acknowledgement of lock — releases SYNC |
| LINK_LOCKED | in | 8 | Active-low LOCK* from each SERDES chip |
| LINK_MASK | in | 8 | Masks dead links — masked link is treated as always locked |
| FORCE_SYNC | in | 16 | VME register; bits[7:0] ORed with SYNC_OUT per-link (allows forcing SYNC even when state machine would not) |
| SYNC_OUT | out | 8 | SYNC to each SERDES transmitter |
| DEN_OUT | out | 8 | Driver (transmitter) enable to each SERDES |
| REN_OUT | out | 8 | Receiver enable to each SERDES |
| LOCK_ERROR | out | 1 | High if lock was lost after once being achieved |
| ALL_LOCKED | out | 1 | High when all unmasked links are in lock |
| LOCKED_AND_ACKED | out | 1 | High after lock confirmed + LOCK_ACK received |
| STATE_MON | out | 4 | Current FSM state for VME diagnostic readout |

**LOCK_STATE masking:**
```vhdl
-- LOCK_STATE(i) = '0' when LINK_MASK(i)='1' (masked = always "locked")
-- active-low: LINK_LOCKED(i)='0' means the SERDES is locked
lockstateblock: for i in 0 to 7 generate
  LOCK_STATE(i) <= '0' when (LINK_MASK(i) = '1') else LINK_LOCKED(i);
end generate;
```
✅ verified 2026-04-25 — link_init.vhd:L114-116

**7-state FSM:**

| State | STATE_MON | DEN/REN | SYNC | ALL_LOCKED | LOCK_ERROR | LOCKED_AND_ACKED | Next state |
|-------|-----------|---------|------|------------|------------|------------------|------------|
| INIT | 0x0 | off | off | 0 | 0 | 0 | EN_SERDES (immediately) |
| EN_SERDES | 0x1 | on | off | 0 | 0 | 0 | stays if RETRY=1; else → SYNC |
| SYNC | 0x2 | on | on | 0 | 0 | 0 | → WAIT_FOR_LOCK |
| WAIT_FOR_LOCK | 0x3 | on | on | 0 | 0 | 0 | → ALL_LOCKED_UP when LOCK_STATE="00000000" |
| ALL_LOCKED_UP | 0x4 | on | on | 1 | 0 | 0 | → ACKED if LOCK_ACK='1' and all locked; → ERROR if lock lost |
| ACKED | 0x5 | on | **off** | 1 | 0 | 1 | stays; → ERROR if any link loses lock |
| ERROR | 0x6 | on | off | 0 | **1** | 1 | stays; → INIT if RETRY='1' |

✅ verified 2026-04-25 — link_init.vhd:L87 (type declaration) + L163-268 (case statements)

**Key behaviours:**
- SYNC is held asserted in states SYNC, WAIT_FOR_LOCK, ALL_LOCKED_UP; released only in ACKED and ERROR
- RETRY gate in EN_SERDES: holds the machine in EN_SERDES if set (prevents premature SYNC assertion during system bring-up)
- RETRY in ERROR: triggers a full re-initialization (→ INIT) to allow recovery after lock loss
- LOCKED_AND_ACKED releases the data path downstream (one clock later the SERDES RX machines see data)
- FORCE_SYNC (bits[7:0]) is ORed onto SYNC_OUT at the output flip-flops — allows forcing SYNC on any specific link regardless of state machine (use case: passing 50 MHz clock from DGS to DSSD Master via unused SERDES link)
- LOCK_ERROR is **sticky** (stays high in ERROR until RST or RETRY); must be actively cleared by software

**Architectural note — RTRG vs. MTRG:**
The RTRG `link_init.vhd` manages the **8 downward links** to digitizers. The MTRG `link_init.vhd` (documented in `vhdl/MTRG_link_init_and_input_pipeline.md`) manages the **8 links from routers**. Both share the same 7-state FSM design and FORCE_SYNC feature. The RTRG also has a separate Router-upward link (Link U/L/R) whose initialization is handled in `TOP.VHD` directly.

**See also:** `deep_fpga_RTRG.md` (RTRG overview), `vhdl/MTRG_link_init_and_input_pipeline.md` (MTRG equivalent)

---

## Cross-References

- `knowledgeBase/deep_fpga_RTRG.md` — RTRG FPGA overview: firmware type codes, register map summary, top-level architecture
- `knowledgeBase/260E_trigger_scheme.md` — End-to-end RTRG/MTRG trigger scheme (chan_in, disc_mach, router_data_path documented there; this file covers support modules)
- `knowledgeBase/vhdl/MTRG_link_init_and_input_pipeline.md` — MTRG equivalent `link_init.vhd` + `mstr_trigger_input_pipeline.vhd`
- `knowledgeBase/vhdl/RTRG_support_modules.md` — this file (self-reference for README)
- `knowledgeBase/VME_registers.md` — RTRG VME register map (DISC_DELAY, I/O control, status)
- `knowledgeBase/fpga.md` — FPGA firmware overview; RTRG's role in the 3-tier hierarchy

---

*Source: `FPGA/RTRG/Firmware/DGS_Version/MAIN_FPGA/Source/`*
