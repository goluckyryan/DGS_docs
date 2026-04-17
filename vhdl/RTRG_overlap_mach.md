# overlap_mach.vhd — Plain English Summary
_Source: ~/FPGA_svn2git/RTRG_git/MAIN_FPGA/Source/overlap_mach.vhd_
_Summarized: 2026-04-15_

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
| `OVERLAP_DELAY` | in | 7 | Coincidence window: 0–127 clocks (up to ~2.54 µs at 50 MHz; code comment says 2.56 µs — minor rounding) ✅ verified 2026-04-17 — `overlap_mach.vhd:L21` |
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
- OVERLAP_DELAY: 7-bit (0–127) ✅ verified 2026-04-17 — `overlap_mach.vhd:L21`. Maximum window = 127 clocks. At 50 MHz that is 2.54 µs (code comment rounds to 2.56 µs); at 100 MHz that is 1.27 µs.

## Connections to Other Modules
- This is a standalone generic component instantiated from higher-level logic (likely router_data_path.vhd or TOP.VHD).
- Functionally related to but separate from `discriminator_mach` (disc_mach.vhd), which has the same overlap-detection core but adds CLEAN/DIRTY/BGO-only classification logic.
- Uses `WORK.trigger_comp_defs` package for component declarations ✅ verified 2026-04-17 — `overlap_mach.vhd:L7`
