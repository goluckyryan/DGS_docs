# RTRG 0x260E Trigger Scheme (RTRG side)
Stability: C3 - Structural / stable
_Firmware: RTRG 0x260E — RTRG-side deep-dive (sections 1–4). For MTRG side see [260E_MTRG_scheme.md](260E_MTRG_scheme.md)._
_Example: 2-detector DUO (BGO ch-0 → Ge ch-5, BGO ch-1 → Ge ch-6; Y_MAP=A5/A6)_
_Source: `DGS_tools_pack/FPGA/` — chan_in.vhd, disc_mach.vhd, router_data_path.vhd, router_top (TOP.VHD)_
_Last updated: 2026-04-26_

---

## Table of Contents

1. [Channel Input (RTRG: chan_in.vhd)](#1-channel-input-rtrg-chan_invhd)
2. [Discriminator & Clean/Dirty Logic (disc_mach.vhd)](#2-discriminator--cleandirty-logic-rtrg-disc_machvhd)
3. [RTRG → MTRG Link Format (router_data_path.vhd)](#3-rtrg--mtrg-link-format-router_data_pathvhd)
4. [RTRG Top-Level (TOP.VHD)](#4-rtrg-top-level-topvhd)
5. [See Also](#see-also)

> **MTRG sections moved** → [260E_MTRG_scheme.md](260E_MTRG_scheme.md) (split 2026-04-26)

---

## 1. Channel Input (RTRG: chan_in.vhd)

Each RTRG board instantiates one `chan_in` block per digitizer (DIG) connection. `chan_in` is the front-end of the trigger data path: it recovers the discriminator bits from the serial SERDES stream, aligns timing between Ge and BGO channels, classifies hits, and outputs per-plane hit maps and multiplicity counts upstream toward the Master Trigger.

### 1.1 Serial Data Reception

The DIG sends a **18-bit SERDES word** every 20 ns clock: ✅ verified 2026-04-15 — `chan_in.vhd:L16` (comment header: `|CG|FLG|CD9..CD5|D09..D00|POL|` = 18 bits)

```
[17] = CG  (clock guard)
[16] = FLG (sync/frame flag)
[15:10] = CD9..CD4  (coarse/fast Ge discriminators, channels 9:4)
[9:5]  = D09..D05  (Ge center discriminator bits, 5 channels)
[4:0]  = D04..D00  (BGO sum discriminator bits, 5 channels)
[0]    = POL  (DC balance polarity bit)
```

The **DCBAL_IN** sub-block strips the DC balance inversion applied by the SERDES transmitter, then crosses the data from the SERDES clock domain (LVDS_RCLK) into the system master clock (MCLK) via a FIFO. The result is `RECOVERED_DATA[15:0]`: bit 15 = FLG, bits [14:10] = coarse Ge, bits [9:5] = Ge, bits [4:0] = BGO.

A parallel **fast path** (`FAST_D_OUT`) bypasses the FIFO to provide the lowest-latency coarse Ge bits — these feed `COARSE_GE_SUM`, a 3-bit popcount of bits [14:10] used for rapid pre-screening.

### 1.2 Per-Bit Timing Alignment

Ten individual **DPRAM delay-line FIFOs** (one per discriminator bit) correct timing skew between Ge and BGO channels. Each FIFO is up to 32 taps deep (20 ns/tap = **640 ns maximum delay**). ✅ verified 2026-04-15 — `chan_in.vhd:L260,266` (`DEPTH_pwr2 => 5` → 2^5=32 samples × 20 ns = 640 ns; inline comment: "or 640nsec") The per-bit delay values are loaded from `BIT_DELAY` via a VME register write with `LOAD_BIT_DELAY`. When `CLEAN_DIRTY(15)='1'`, the delay-corrected `DELAYED_DATA[9:0]` is used for all downstream logic; otherwise the raw `RECOVERED_DATA` bits are used. ✅ verified 2026-04-16 — `chan_in.vhd:L219,228-229` (`RAW_DATA_OUT <= ... DELAYED_DATA when CLEAN_DIRTY(15)='1' else RECOVERED_DATA`)

### 1.3 Synchronization State Machine (REMAP_BITS_PROC)

Before producing any output, `chan_in` runs a 3-state machine:

| State | Condition | Action |
|---|---|---|
| **IDLE** | `INPUT_MASK='0'` → advance | Zero all outputs |
| **WAIT_DIG_FLAG** | `RECOVERED_DATA[15]` (FLG) = '1' → advance | Hold zeros; wait for frame boundary |
| **REMAP_BITS** | Continuous | Apply X_PLANE_MAP / Y_PLANE_MAP bitmasks; drive outputs |

This ensures every RTRG channel is locked to the same data-frame boundary before forwarding hits.

### 1.4 Discriminator Machines and Hit Classification

Five `discriminator_mach` sub-blocks handle the five Ge/BGO channel pairs (Ge = bits [9:5], BGO = bits [4:0]). Each `disc_mach` examines the timing overlap between a Ge discriminator firing and the corresponding BGO discriminator firing to classify each event:

- **CLEAN_EVENT** — Ge fired, BGO did **not** fire within the overlap window → Ge-only (full-energy deposit) ✅ verified 2026-04-20 — `disc_mach.vhd:L165` (timer expires without BGO → `CLEAN_EVENT <= '1'`)
- **DIRTY_EVENT** — Both Ge and BGO fired within the overlap window → Compton scatter coincidence ✅ verified 2026-04-20 — `disc_mach.vhd:L153-157` (BGO fires while timer active → `DIRTY_EVENT <= '1'`)
- **BGO_ONLY_EVENT** — BGO fired, Ge did not → BGO-shield-only event ✅ verified 2026-04-20 — `disc_mach.vhd:L93` (comment: "BGO fires, but Ge does not before timer expires : this is a BGO_ONLY event")

One-shot timers (7-bit wide, up to 127 clocks) stretch these single-tick pulses into an **assertion window** (`ASSERTION_DELAY = TSCATTER_DELAY_REG[14:8]`). A retriggerable one-shot restarts its timer on each new event, so a burst of events within the window produces a single extended assertion. The resulting extended flags are:

- `HAVE_CLEAN[4:0]` — asserted for the assertion window after each CLEAN_EVENT per pair ✅ verified 2026-04-20 — `chan_in.vhd:L107,L327-334` (5-bit signal; retriggerable one-shot driven by CLEAN_EVENT)
- `HAVE_DIRTY[4:0]` — asserted for the assertion window after each DIRTY_EVENT per pair ✅ verified 2026-04-20 — `chan_in.vhd:L108,L337-344` (retriggerable one-shot driven by DIRTY_EVENT)
- `HAVE_MODULE[4:0]` — asserted for the assertion window after either a CLEAN or DIRTY event ✅ verified 2026-04-20 — `chan_in.vhd:L109,L347-354` (retriggerable one-shot; comment L311: "HAVE_MODULE is a little special")

### 1.5 X-Plane and Y-Plane Hit Maps

The `CLEAN_DIRTY` register selects which flag set drives the X and Y output bitmaps:

| CLEAN_DIRTY[3:0] | X_PLANE source | Typical use |
|---|---|---|
| `0000` | `X_BITS` (raw DFMA mapping) | DFMA mode |
| `0001` | `HAVE_CLEAN[4:0]` | DGS clean-hit counting |
| `0010` | `HAVE_DIRTY[4:0]` | DGS dirty-hit counting |
| `0100` | `HAVE_MODULE[4:0]` | DGS any-hit counting |
| `1000` | `HAVE_CLOVER_CLEAN` | Clover geometry |

✅ verified 2026-04-16 — `chan_in.vhd:L490-499` (X_SELECT driven by `CLEAN_DIRTY(3 downto 0)`; Y_SELECT by `CLEAN_DIRTY(7 downto 4)` with same encoding)

`Y_SELECT` (bits [7:4]) picks the Y-plane source independently, allowing X and Y to count different event classes simultaneously.

These bitmaps are further masked by `X_PLANE_MAP` (10-bit) and `Y_PLANE_MAP` (10-bit) registers that specify which discriminator bits belong to each plane. **In our DUO example**, Y_MAP = `A5/A6` means only bits 5 and 6 (Ge ch-5 and Ge ch-6) contribute to the Y-plane count.

Two `plane_bit_count` LUT popcounters produce `X_PLANE_COUNT` and `Y_PLANE_COUNT` (4-bit, 0–10), which are the multiplicity counts forwarded toward the Master Trigger.

### 1.6 Live-Channel Veto Generation

When `ENABLE_VETO='1'`, the `VETO_GEN_PROC` drives `LIVE_CHANNEL_VETO[10:1]` based on DIRTY and BGO-only events. These veto requests propagate to the RTRG top-level ADD_VETOES process to suppress re-triggering during a dirty event window.

### 1.7 DUO Example — Channel Input

In the 2-detector DUO:
- **BGO ch-0 / Ge ch-5**: `disc_mach` instance 0 monitors pair (Ge bit 5, BGO bit 0). A gamma depositing full energy in Ge ch-5 fires Ge bit 5 only → `CLEAN_EVENT[0]` → `HAVE_CLEAN[0]` asserted for ASSERTION_DELAY clocks.
- **BGO ch-1 / Ge ch-6**: `disc_mach` instance 1 monitors pair (Ge bit 6, BGO bit 1). Same logic.
- Y_PLANE_MAP has bits 5 and 6 set → only `HAVE_CLEAN[0]` and `HAVE_CLEAN[1]` feed `Y_PLANE_BITS[5]` and `Y_PLANE_BITS[6]`. `Y_PLANE_COUNT` reflects 0, 1, or 2 clean Ge hits.

---

## 2. Discriminator & Clean/Dirty Logic (RTRG: disc_mach.vhd)

`disc_mach` is the hit-classification engine at the heart of each Ge/BGO detector pair. Five instances run in parallel inside `chan_in`, one per pair. It uses leading-edge detection and a programmable coincidence (overlap) window to produce single-clock-tick output pulses.

### 2.1 Edge Detection

Each clock, `disc_mach` pipelines `GE_DISC_FLAG` and `BGO_DISC_FLAG` one tick and detects rising edges:

- `GE_EDGE` — fires for one tick when Ge transitions from low to high ✅ verified 2026-04-20 — `disc_mach.vhd:L54,L105,L107` (signal declared; `GE_EDGE <= '1'` on rising edge)
- `BGO_EDGE` — fires for one tick when BGO transitions from low to high ✅ verified 2026-04-20 — `disc_mach.vhd:L54,L111,L113` (signal declared; `BGO_EDGE <= '1'` on rising edge)

### 2.2 Four-State Overlap Machine

The state machine uses a 7-bit `OVERLAP_TIMER`, loaded from `OVERLAP_DELAY` (= `TSCATTER_DELAY_REG[6:0]`, 0–127 clocks = 0–1270 ns at 100 MHz): ✅ verified 2026-04-25 — `disc_mach.vhd:L44` (type enum: ST_IDLE, ST_OVERLAP_GE_FIRST, ST_OVERLAP_BGO_FIRST, ST_WAIT_DIRTY); `L45` (`OVERLAP_TIMER: std_logic_vector(6 downto 0)`)

| State | Description |
|---|---|
| **ST_IDLE** | Waiting. Timer pre-loaded to OVERLAP_DELAY. Exits on any edge. ✅ verified 2026-04-25 — `disc_mach.vhd:L131` (`OVERLAP_TIMER <= OVERLAP_DELAY` in ST_IDLE) |
| **ST_OVERLAP_GE_FIRST** | Ge fired first; counting down. BGO within window → DIRTY; timer expires → CLEAN. ✅ verified 2026-04-25 — `disc_mach.vhd:L144-173` (BGO_EDGE → DIRTY or ST_WAIT_DIRTY; OVERLAP_TIMER=0 no BGO → CLEAN) |
| **ST_OVERLAP_BGO_FIRST** | BGO fired first; counting down. BGO retriggering resets timer. Ge within window → DIRTY; timer expires → BGO_ONLY. ✅ verified 2026-04-25 — `disc_mach.vhd:L177-203` (BGO_EDGE reloads OVERLAP_DELAY:L180; GE_EDGE → DIRTY; timer=0 → BGO_ONLY_EVENT:L192) |
| **ST_WAIT_DIRTY** | Both fired (or one side entered early); counts out remainder of OVERLAP_TIMER, then pulses DIRTY. ✅ verified 2026-04-25 — `disc_mach.vhd:L207-220` (MBO 20140610: count down remaining time, assert DIRTY_EVENT when OVERLAP_TIMER=0) |

### 2.3 Event Classification Summary

| Scenario | One-tick output |
|---|---|
| Ge rises, no BGO within overlap window | **CLEAN_EVENT** |
| Ge rises first, BGO within overlap window | **DIRTY_EVENT** |
| BGO rises first, Ge within overlap window | **DIRTY_EVENT** |
| BGO rises, no Ge within overlap window | **BGO_ONLY_EVENT** |
| Both rise simultaneously | **DIRTY_EVENT** (after timer expires from OVERLAP_DELAY) ✅ verified 2026-04-25 — `disc_mach.vhd:L139` (MBO 20140610: simultaneous GE_EDGE+BGO_EDGE in ST_IDLE → jump directly to ST_WAIT_DIRTY; timer counts from OVERLAP_DELAY) |

The single-tick pulses from `disc_mach` are handed back to `chan_in`'s ONE_SHOTS process, which stretches each one into an **assertion window** of up to 127 clocks (`ASSERTION_DELAY = TSCATTER_DELAY_REG[14:8]`). If a new event arrives before the timer expires, the one-shot restarts — producing a continuous assertion across a burst of hits.

### 2.4 DUO Example — Discriminator Classification

**Scenario A — clean Ge hit in ch-5:**

1. A gamma ray deposits full energy in Ge ch-5. Only Ge bit 5 fires.
2. `disc_mach` instance 0: `GE_EDGE='1'`, BGO stays low. Machine enters `ST_OVERLAP_GE_FIRST`.
3. Overlap timer counts down from `OVERLAP_DELAY` (e.g., 10 clocks = 100 ns). No BGO edge arrives.
4. At timer = 0: `CLEAN_EVENT='1'` for one clock tick.
5. `chan_in` ONE_SHOTS: `HAVE_CLEAN[0]` asserted for ASSERTION_DELAY clocks.
6. With `CLEAN_DIRTY[3:0]=0001` and Y_PLANE_MAP bit 5 set: `Y_PLANE_BITS[5]='1'`, `Y_PLANE_COUNT` increments.

**Scenario B — Compton scatter in BGO ch-0 / Ge ch-5:**

1. Gamma Compton-scatters into BGO ch-0 first. BGO bit 0 fires → `BGO_EDGE='1'`.
2. Machine enters `ST_OVERLAP_BGO_FIRST`.
3. Within the overlap window, the scattered gamma deposits energy in Ge ch-5 → `GE_EDGE='1'`.
4. Machine transitions to `ST_WAIT_DIRTY`, counts out timer, then asserts `DIRTY_EVENT='1'`.
5. `HAVE_DIRTY[0]` asserted. With `CLEAN_DIRTY[3:0]=0010` for Y: `Y_PLANE_BITS[5]='1'` in dirty mode; the event is counted differently or vetoed depending on firmware configuration.

---

## 3. RTRG → MTRG Link Format (router_data_path.vhd)

`router_data_path` aggregates all eight `chan_in` instances on one RTRG board and assembles their hit counts into a single 16-bit word — **Link-L** — transmitted to the Master Trigger over a SerDes serial link.

### 3.1 Adder Tree — Pipelined Summation

The eight `chan_in` blocks each produce a 4-bit `X_PLANE_COUNT` and `Y_PLANE_COUNT`. A three-rank pipelined adder tree (one pipeline register per rank) sums all eight:

- **Rank 1**: Four 5-bit partial sums — channels 1+2, 3+4, 5+6, 7+8 (separately for X and Y)
- **Rank 2**: Two 6-bit sums — (ch1-4) and (ch5-8) for each plane
- **Rank 3**: One 7-bit total — final X sum and final Y sum (max 80 each)

The 3-rank pipeline introduces 3 clock cycles (60 ns) of latency between a hit arriving at `chan_in` and the corresponding count appearing in Link-L. ✅ verified 2026-04-15 — `router_data_path.vhd:L210-233` (SUM_HITS process runs on `mclk` = 50 MHz = 20 ns/clock; rank1→rank2→rank3+LINKL are all registered signals so each rank costs one clock edge = 3 clocks × 20 ns = 60 ns total)

### 3.2 Link-L Word Bit-Field Layout

```
Bit [15]    = ANY_THROTTLE_REQUEST  (OR of all 8 digitizer throttle requests)
Bits [14:8] = Y-plane multiplicity sum (7-bit, range 0–80)
Bit [7]     = DATA_VALID  (= ALL_DIGITIZERS_LOCKED AND ROUTER_LOCKED)
Bits [6:0]  = X-plane multiplicity sum (7-bit, range 0–80)
```

✅ verified 2026-04-15 — `router_data_path.vhd:L230-233` (LINKL_RAW_DATA(15) = ANY_THROTTLE_REQUEST; (14:8) = Y sum; (7) = ALL_DIGITIZERS_LOCKED AND ROUTER_LOCKED; (6:0) = X sum)

This 16-bit word occupies bits [16:1] of the 18-bit SerDes frame transmitted to the MTRG.

### 3.3 DATA_VALID Qualification

`DATA_VALID` (bit 7) is asserted only when all eight digitizer SERDES links are locked **and** the Router itself is locked to the Master. If any link loses lock, DATA_VALID drops and the MTRG will discard the word.

### 3.4 Coarse Ge Fast Path

In parallel with the main adder tree, `FAST_COARSE_GE_SUM_PROC` sums the 3-bit `COARSE_GE_SUM` outputs from all eight `chan_in` instances into a 6-bit `TOTAL_COARSE_GE_SUM` (max 40). Each `COARSE_GE_SUM` is a popcount of 5 coarse Ge discriminator bits (bits [14:10] from the SERDES word), so the per-channel max is 5 and the 8-channel max is 8×5=40. ✅ verified 2026-04-25 — `chan_in.vhd:L213` (COARSE_GE_SUM is sum of 5 COARSE_GE_BITS; max=5 per channel); `router_data_path.vhd:L55` (TOTAL_COARSE_GE_SUM: 6-bit out; max=40 confirmed)

The process uses 2 pipeline ranks inside a single clocked block: Rank 1 sums channels 1–4 and 5–8 into two 5-bit `INTER_COARSE_GE_SUM` values (registered); Rank 2 sums the two inter-values into `TOTAL_COARSE_GE_SUM`. Since both ranks use registered outputs, the full result has **2 clock cycles (40 ns) latency** relative to the `COARSE_GE_SUM` inputs. MBO noted this: "lets try multiple ranks of adders in a single clock" (meaning within one process block, not one cycle). ✅ verified 2026-04-25 — `router_data_path.vhd:L139-148` (INTER registered at cycle N, TOTAL uses previous-cycle INTER values → 2-cycle pipeline)

### 3.5 DUO Example — Link-L Contents

In the 2-detector DUO with a clean hit on Ge ch-5 and Ge ch-6 (both firing within the assertion window):

- `chan_in` instances for channels 5 and 6 each contribute 1 to `Y_PLANE_COUNT`
- After the 3-rank adder tree: **Y-plane sum = 2, X-plane sum = 0** (assuming X-plane not mapped)
- Link-L word = `0b 0 0000010 1 0000000` → bits [14:8] = 0b0000010 = 2, bits [6:0] = 0

The MTRG receives `RTR_SUM_OF_Y = 2` for this RTRG, indicating multiplicity-2 in the Y plane.

### 3.6 Diagnostic FIFOs

Each of the eight `chan_in` channels feeds a diagnostic FIFO. A 2-bit mode register selects what is stored: raw discriminator data, any-hit flags, plane counts + disc_mach states, or combined count/raw data. Write-enable modes allow capture on any hit or on a specific channel hit. These are used for commissioning and debugging only; they do not affect the trigger path.

---

## 4. RTRG Top-Level (TOP.VHD)

`TOP.VHD` (`router_top` entity) is the integration layer that wires all RTRG sub-blocks together and manages the bidirectional links to both the digitizers and the Master Trigger.

### 4.1 Uplink to Master Trigger

The uplink chain (Digitizers → MTRG):

1. `ROUTER_DATA_PATH` produces `LINKL_RAW_DATA[15:0]` (X/Y sums, throttle, data-valid).
2. `dc_balance_mach` (U1) applies DC balance encoding → `LINKL_BAL_DATA[17:0]`.
3. `TX_PROC` drives `LINKL_TX` every clock at 50 MHz (100 Mbps SerDes).

### 4.2 Downlink from Master Trigger

The downlink chain (MTRG → Digitizers):

1. `SERDES_RX_Mach_R2` (instantiated as part of the link-receive path) decodes the incoming `LINKL_RX` stream. Frame 12 carries router-specific commands (counter/FIFO resets); these are extracted by the `FRAME_12_LATCH` process and the frame is **replaced with null words (0xAAAA/0xAAAA/0xAAAA/0xAAAA/0x0000)** in `SANITIZED_CONTROL_DATA` before forwarding to digitizers — so digitizers never see raw router commands. ✅ verified 2026-04-25 — `SERDES_RX_Mach_R2.vhd:L44` (port comment: "RECEIVED_CONTROL_DATA, with frames 12 & 14 replaced by Null frames"); `L1108-1116` (frame 12 state: words except last → 0xAAAA, word #59 → 0x0000; `VETO_EVENT` extracted from LATCHED_CONTROL_DATA[9:0] on word #59)
   - **Correction (2026-04-25):** Earlier text said "router_main_mach" and "F12_DECODE" handle the replacement. The actual block is `SERDES_RX_Mach_R2` (the SERDES_RX_MACH instantiation). The 5th word of frame 12 (word index 59) is replaced with 0x0000 (not 0xAAAA); all other frame 12 words use 0xAAAA.
2. `ADD_VETOES_BLK` (one process per digitizer, `ADD_VETOES_PROC`): monitors the 5-word frame structure. On the 5th word of every frame 1–19, it replaces the outgoing word with the **accumulated live-channel veto bitmap** for that digitizer (10 bits, from `CHANNEL_VETOES`). Between insertions the latch accumulates new veto requests via OR. The 5th word of frame 20 is reserved for machine synchronization and is not overridden. ✅ verified 2026-04-25 — `TOP.VHD:L2015-2055` (`ADD_VETOES_PROC` process; comment: "5th word of a frame is being processed"; word indices 5/10/15…95 (frames 1–19) → insert `"000000" & LATCHED_CHANNEL_VETOES(i)` at bits [16:1]; otherwise pass through `SANITIZED_CONTROL_DATA` and OR-accumulate `CHANNEL_VETOES(i)` into latch)
3. Eight `dc_balance_mach` instances re-encode the modified words and drive `LINKA..LINKH_TX` to the digitizers.

### 4.3 Key VME Registers Relevant to Trigger

| Register | Role |
|---|---|
| `Y_PLANE_MAP_REG` | 10-bit bitmap per channel: which disc bits count toward Y-plane (DUO: bits 5,6) |
| `X_PLANE_MAP_REG` | Same for X-plane |
| `CLEAN_DIRTY_REG` | Selects CLEAN/DIRTY/MODULE/Clover source for X and Y hit maps |
| `TSCATTER_DELAY_REG` | [14:8] = assertion window; [6:0] = overlap (coincidence) window |
| `THROTTLE_LIMIT_TIME_REG` | Minimum continuous assertion length before a throttle request is accepted. Bits[10:0] = count; bits[15:14] select prescaler (00=20.48 µs/tick, 01=20.97 ms/tick, 10=~21.47 s/tick, 11=~6 hr/tick). ✅ verified 2026-04-19 — `throttle_limiters.vhd:L63,L272,L288-296` (entity port comment + LIMIT_MACH decrement logic + prescaler select) |
| `INPUT_LINK_MASK_REG` | Per-channel dead-channel mask |
| `ENABLE_VETO` | Enables live-channel veto generation |

### 4.4 Coarse Ge Fast Path to CPLD

`TOTAL_COARSE_GE_SUM[5:0]` (summed by `router_data_path` in a single cycle) is transmitted to an on-board CPLD via 3 DDR lines on `SUMCOPY[3:1]`: lower 3 bits on the rising edge, upper 3 bits on the falling edge of the 50 MHz clock. The CPLD uses this 6-bit count for a fast pre-trigger multiplicity decision. The resulting `FAST_STROBE` signal returns from the CPLD to the RTRG for NIM output/diagnostic use. ✅ verified 2026-04-17 — `TOP.VHD:L877-895` (ODDR_inst generate loop ×3: D1=TOTAL_COARSE_GE_SUM(i), D2=TOTAL_COARSE_GE_SUM(i+3) → lower 3 bits on positive edge, upper 3 bits on negative edge of MCLK)

### 4.5 Clock Architecture

- `switched_master_clock` (50 MHz) — used by SerDes TX, router_main_mach, veto insertion, DDR outputs, `chan_in`, and `router_data_path` (all processing). Source switchable between local oscillator and `LINKL_RCLK` (recovered from MTRG) via a DCM. ✅ verified 2026-04-21 — `TOP.VHD:L2165` (`mclk => switched_master_clock` for ROUTER_DATA_PATH instantiation); `router_data_path.vhd:L169` (`mclk => mclk` for all chan_in instantiations; `SUM_HITS` process also clocked by `mclk`)
- `switched_master_clock_2x` (100 MHz) — derived from the same DCM 2× output. Used for: DDR outputs (SUMCOPY bus to CPLD), test point (TEST_POINT[9]), and the three `dc_balance_mach` instances (DC balance encoding for downstream links and Link-L). **Not** used by `chan_in` or `router_data_path`. ✅ verified 2026-04-23 — `TOP.VHD:L1087` (test point), `TOP.VHD:L877-895` (ODDR SUMCOPY), `TOP.VHD:L2134,2153,2266` (dc_balance_mach clk_2x); `dc_balance_mach.vhd:L54,73` (DISP_LOOKUP_MUX + PARTIAL_DISP_SHIFTER run on clk_2x)

> **Correction (2026-04-21):** An earlier version of this section incorrectly stated that `chan_in` and `router_data_path` used `switched_master_clock_2x` (100 MHz). Both are clocked by `switched_master_clock` (50 MHz). The SUM_HITS adder pipeline latency of 60 ns (3 × 20 ns) is consistent with this — see section 3.1.

### 4.6 DUO Example — TOP.VHD Role

In the DUO, TOP.VHD:
- Distributes `Y_PLANE_MAP_REG` = `0b0000001100000000` (bits 5 and 6 set) to all `chan_in` instances.
- Forwards the MTRG trigger decision received on `LINKL_RX` downstream to both digitizers via `LINKA_TX` and `LINKB_TX`, embedding the veto bitmap on every 5th frame word.
- If the MTRG fires a trigger (multiplicity ≥ 2), the digitizers receive the trigger strobe on the next veto-insertion boundary.

---

---

## See Also

- [260E_MTRG_scheme.md](260E_MTRG_scheme.md) — **MTRG side** of this trigger chain: `mt_input_channel.vhd`, `eight_mt_channel.vhd`, `sum_hits_X.vhd`, `calc_total_sum.vhd`, `top.vhd`; full end-to-end DUO example and Mermaid diagram
- [deep_fpga_RTRG.md](deep_fpga_RTRG.md) — RTRG firmware overview: Virtex-4 device, source file index, VME register map, disc_mach.vhd, Router→MTRG SERDES frame format
- [fpga.md](fpga.md) — System-level trigger hierarchy: DIG→RTRG→MTRG signal flow, end-to-end timing, throttle mechanism
- [deep_fpga_MTRG_MAIN.md](deep_fpga_MTRG_MAIN.md) — MTRG: consumes RTRG multiplicity/hit data to make global trigger decisions
- [deep_fpga_DIG.md](deep_fpga_DIG.md) — DIG: upstream source of the 18-bit SERDES words that chan_in.vhd receives
- [ttcl.md](ttcl.md) — TTCL frame format: frames 12/14 (inter-trigger/remote trigger) that the RTRG forwards with nulls
- [vhdl/](vhdl/) — Per-module plain-English VHDL summaries: `RTRG_chan_in.md`, `RTRG_disc_mach.md`, `RTRG_router_data_path.md`
- [EPICS_RTrig_templates.md](EPICS_RTrig_templates.md) — EPICS PVs that configure the RTRG firmware described here (ILM, X/Y maps, throttle, delays, SERDES power)
- [vxworks_trigger_drivers.md](vxworks_trigger_drivers.md) — VxWorks asyn drivers (asynTrigRouterDriver, asynTrigMasterDriver) that write the VME registers these modules read
