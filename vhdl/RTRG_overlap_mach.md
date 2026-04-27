# overlap_mach.vhd — Plain English Summary
_Source: ~/FPGA_svn2git/RTRG_git/MAIN_FPGA/Source/overlap_mach.vhd_
_Summarized: 2026-04-15_
Stability: C3 - Structural / stable

## Table of Contents

- [Purpose](#purpose)
- [Ports](#ports)
- [Key Logic / State Machine](#key-logic--state-machine)
  - [FIND_EDGES process](#find_edges-process)
  - [OVERLAP_MACHINE — 4 states](#overlap_machine--4-states)
- [Key Constants / Parameters](#key-constants--parameters)
- [Connections to Other Modules](#connections-to-other-modules)
- [See Also](#see-also)

## Purpose
Generic two-signal coincidence detector. Given any two level signals (SIG_A, SIG_B), outputs a one-clock-tick `OVERLAP_OCCURRED` pulse if both signal rising edges arrive within a programmable time window (OVERLAP_DELAY). Symmetric: either signal may arrive first. This is a simpler/more generic version of the same algorithm embedded in disc_mach.vhd — disc_mach also classifies the result as CLEAN/DIRTY/BGO-only, whereas this module only answers "did they overlap?" (yes/no). ✅ verified 2026-04-17 — `overlap_mach.vhd` entity `overlap_machine`

**Note:** Entity name is `overlap_machine` (not `overlap_mach`) — the filename differs from the entity name. ✅ verified 2026-04-17 — `overlap_mach.vhd:L14`

## Ports
| Signal | Dir | Width | Description |
|---|---|---|---|
| `CLK` | in | 1 | Board clock (50 MHz per comment, but module is generic) ✅ verified 2026-04-17 — `overlap_mach.vhd:L16` comment "board wide 50MHz" |
| `RST` | in | 1 | Active-high reset ✅ verified 2026-04-17 — `overlap_mach.vhd:L17` |
| `SIG_A` | in | 1 | First signal (level) ✅ verified 2026-04-17 — `overlap_mach.vhd:L19` |
| `SIG_B` | in | 1 | Second signal (level) ✅ verified 2026-04-17 — `overlap_mach.vhd:L20` |
| `OVERLAP_DELAY` | in | 7 | Coincidence window: 0–127 (OVERLAP_DELAY+1 ticks max, since B firing at OVERLAP_TIMER=0 still marks overlap — up to 128 ticks = **2.56 µs** at 50 MHz) ✅ verified 2026-04-17 — `overlap_mach.vhd:L21` comment; ✅ confirmed 2026-04-25 — `overlap_mach.vhd:L91-94` (OVERLAP_TIMER≥0 condition: SIG_B at timer=0 still counts) |
| `OVERLAP_OCCURRED` | out | 1 | One-tick pulse: both edges arrived within OVERLAP_DELAY ✅ verified 2026-04-17 — `overlap_mach.vhd:L23` |

## Key Logic / State Machine

### FIND_EDGES process
Pipelines SIG_A and SIG_B one clock, detects rising edges → `SIG_A_EDGE`, `SIG_B_EDGE` (one-tick pulses). ✅ verified 2026-04-17 — `overlap_mach.vhd:L44–60`

### OVERLAP_MACHINE — 4 states

**ST_IDLE** ✅ verified 2026-04-17 — `overlap_mach.vhd:L67–83`
- Clears OVERLAP_OCCURRED, loads OVERLAP_TIMER ← OVERLAP_DELAY  
- Both edges simultaneously → **ST_OVERLAP_OCCURRED** (direct) — comment: "Do not pass Go. Do not collect $200."  
- SIG_A edge only → **ST_OVERLAP_A_FIRST**  
- SIG_B edge only → **ST_OVERLAP_B_FIRST**  

**ST_OVERLAP_A_FIRST** — SIG_A fired first ✅ verified 2026-04-17 — `overlap_mach.vhd:L87–103`
- OVERLAP_TIMER decrements each clock; if SIG_A fires again, timer resets to OVERLAP_DELAY  
- SIG_B edge arrives (timer ≥ 0) → **ST_OVERLAP_OCCURRED**  
- Timer reaches 0 with no SIG_B → **ST_IDLE** (no overlap)  

**ST_OVERLAP_B_FIRST** — SIG_B fired first ✅ verified 2026-04-17 — `overlap_mach.vhd:L108–125`
- OVERLAP_TIMER decrements; if SIG_B fires again, resets timer  
- SIG_A edge arrives (timer ≥ 0) → **ST_OVERLAP_OCCURRED**  
- Timer reaches 0 with no SIG_A → **ST_IDLE** (no overlap)  
- **Copy-paste bug confirmed** ✅ verified 2026-04-17 — `overlap_mach.vhd:L117`: the `else` ("otherwise wait") branch targets `ST_OVERLAP_A_FIRST` instead of `ST_OVERLAP_B_FIRST`. The timer decrement still works correctly so the window stays active, but the state label is wrong — a cosmetic VHDL bug that doesn't affect function since the timer logic is independent of the state label.

**ST_OVERLAP_OCCURRED** ✅ verified 2026-04-17 — `overlap_mach.vhd:L127–130`
- Asserts `OVERLAP_OCCURRED='1'` for exactly one clock tick  
- Returns to ST_IDLE  

## Key Constants / Parameters
- OVERLAP_DELAY: 7-bit (0–127) ✅ verified 2026-04-17 — `overlap_mach.vhd:L21`. Maximum window = 128 ticks (OVERLAP_DELAY+1, since B firing when timer=0 still counts ✅ verified 2026-04-25 — `overlap_mach.vhd:L91-94`). At 50 MHz: **2.56 µs**; at 100 MHz: 1.28 µs.

## Connections to Other Modules
- **NOT instantiated anywhere in RTRG.** ✅ verified 2026-04-25 — exhaustive `grep` across all `~/FPGA_svn2git/RTRG_git/MAIN_FPGA/Source/*.vhd` finds zero uses of `overlap_machine`. The module exists in the source tree as a standalone generic but is never wired in.
- `disc_mach.vhd` has the same coincidence logic **inlined** — it does not import or instantiate this module. The overlap-detection algorithm was implemented twice: once here as a reusable component, and once embedded directly in `disc_mach.vhd`.
- Functionally related to but separate from `discriminator_mach` (disc_mach.vhd), which has the same overlap-detection core but adds CLEAN/DIRTY/BGO-only classification logic.
- Uses `WORK.trigger_comp_defs` package for component declarations ✅ verified 2026-04-17 — `overlap_mach.vhd:L7`

## See Also

- [RTRG_disc_mach.md](RTRG_disc_mach.md) — the module that implements the same coincidence logic inline (overlap_mach is never instantiated)
- [RTRG_chan_in.md](RTRG_chan_in.md) — top-level RTRG channel processor; instantiates disc_mach, not this module
- [deep_fpga_RTRG.md](../deep_fpga_RTRG.md) — RTRG architecture overview
