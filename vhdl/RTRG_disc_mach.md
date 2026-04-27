# disc_mach.vhd — Plain English Summary
_Source: ~/FPGA_svn2git/RTRG_git/MAIN_FPGA/Source/disc_mach.vhd_
_Summarized: 2026-04-15_
Stability: C3 - Structural / stable

## Table of Contents

- [Purpose](#purpose)
- [Ports](#ports)
- [Key Logic / State Machine](#key-logic--state-machine)
  - [FIND_EDGES process](#find_edges-process)
  - [DISCRIMINATOR_MACHINE — 4 states](#discriminator_machine--4-states)
  - [Timing summary](#timing-summary)
- [Key Constants / Parameters](#key-constants--parameters)
- [Connections to Other Modules](#connections-to-other-modules)
- [See Also](#see-also)

## Purpose
Classifies one Ge+BGO detector pair event as **CLEAN**, **DIRTY**, or **BGO-only** using leading-edge detection and a programmable overlap coincidence window. One instance per Ge/BGO pair; chan_in.vhd instantiates five of these per digitizer channel (plus one for Clover mode). Outputs are one-clock-tick wide pulses; chan_in.vhd stretches them with retriggerable one-shots.

## Ports
| Signal | Dir | Width | Description |
|---|---|---|---|
| `CLK` | in | 1 | 100 MHz system clock |
| `RST` | in | 1 | Active-high reset |
| `OVERLAP_DELAY` | in | 7 | VME-programmable coincidence window (0–127 clocks = 0–1270 ns) |
| `GE_DISC_FLAG` | in | 1 | Level signal from Ge discriminator |
| `BGO_DISC_FLAG` | in | 1 | Level signal from BGO discriminator |
| `CLEAN_EVENT` | out | 1 | One-tick pulse: Ge fired, BGO did not within overlap window |
| `DIRTY_EVENT` | out | 1 | One-tick pulse: both Ge and BGO fired within overlap window |
| `BGO_ONLY_EVENT` | out | 1 | One-tick pulse: BGO fired, Ge did not within overlap window |
| `GE_EDGE_DIAG` | out | 1 | Rising-edge detect of GE_DISC_FLAG (diagnostic) |
| `BGO_EDGE_DIAG` | out | 1 | Rising-edge detect of BGO_DISC_FLAG (diagnostic) |
| `STATE_MON` | out | 3 | State encoding for ILA debug |

## Key Logic / State Machine

### FIND_EDGES process
Each clock, pipelines GE_DISC_FLAG and BGO_DISC_FLAG one tick. Detects rising edge: `GE_EDGE <= '1'` when `GE_PIPE='0'` and `GE_DISC_FLAG='1'`. Same for BGO. These one-tick pulses feed the state machine. ✅ verified 2026-04-25 — `disc_mach.vhd:L101–115` (GE_PIPE/BGO_PIPE latched at L101-102; edge detect L103-114; GE_EDGE_DIAG/BGO_EDGE_DIAG assigned L119-120)

### DISCRIMINATOR_MACHINE — 4 states ✅ verified 2026-04-25 — `disc_mach.vhd:L122–216` (async RST L124; state type ST_IDLE/ST_OVERLAP_GE_FIRST/ST_OVERLAP_BGO_FIRST/ST_WAIT_DIRTY L49)

**ST_IDLE** — Waiting ✅ verified 2026-04-25 — `disc_mach.vhd:L132–146`  
- Loads `OVERLAP_TIMER ← OVERLAP_DELAY` (L133)  
- Clears all outputs (CLEAN, DIRTY, BGO_ONLY all '0') (L134-136)  
- Transitions:  
  - Both edges simultaneously → **ST_WAIT_DIRTY** (MBO 20140610; L137-138)  
  - GE_EDGE only → **ST_OVERLAP_GE_FIRST** (L139-140)  
  - BGO_EDGE only → **ST_OVERLAP_BGO_FIRST** (L141-142)  

**ST_OVERLAP_GE_FIRST** — Ge fired first, counting down ✅ verified 2026-04-25 — `disc_mach.vhd:L148–168`  
- Decrements OVERLAP_TIMER each clock (L149); BGO_ONLY forced '0' (L148)  
- If BGO_EDGE arrives:  
  - Timer = 0 → `DIRTY_EVENT='1'`, return to IDLE immediately (L152-155; MBO note: resolves edge case where BGO fires exactly as counter expires)  
  - Timer > 0 → `DIRTY_EVENT='0'`, go to **ST_WAIT_DIRTY** to finish the window (L156-159)  
- If timer reaches 0 with no BGO: `CLEAN_EVENT='1'`, return to IDLE (L161-165)  

**ST_OVERLAP_BGO_FIRST** — BGO fired first, counting down ✅ verified 2026-04-25 — `disc_mach.vhd:L170–197`  
- Decrements OVERLAP_TIMER each clock; if BGO fires again, resets timer to OVERLAP_DELAY (L174; comment: this shouldn't occur under normal hold-off conditions but handled)  
- CLEAN_EVENT forced '0' (L171)  
- If GE_EDGE arrives:  
  - Timer = 0 → `DIRTY_EVENT='1'`, return to IDLE immediately (L181-184; same edge-case fix as GE_FIRST)  
  - Timer > 0 → `DIRTY_EVENT='0'`, go to **ST_WAIT_DIRTY** (L185-188)  
- If timer reaches 0 with no Ge: `BGO_ONLY_EVENT='1'`, return to IDLE (L190-193)  

**ST_WAIT_DIRTY** — Both fired, waiting for window to expire ✅ verified 2026-04-25 — `disc_mach.vhd:L199–214` (MBO 20140610: 'New state. Jump here when you know you're dirty.')  
- Decrements OVERLAP_TIMER each clock (L200)  
- When timer = 0: `DIRTY_EVENT='1'`, return to IDLE (L202-208); CLEAN and BGO_ONLY forced '0' throughout  

### Timing summary
| Scenario | Output |
|---|---|
| Ge edge, no BGO within OVERLAP window | CLEAN_EVENT |
| Ge edge first, BGO within OVERLAP window | DIRTY_EVENT |
| BGO edge first, Ge within OVERLAP window | DIRTY_EVENT |
| BGO edge, no Ge within OVERLAP window | BGO_ONLY_EVENT |
| Both edges simultaneously | DIRTY_EVENT (after OVERLAP_TIMER counts down) |

## Key Constants / Parameters
- `OVERLAP_DELAY`: 7-bit input, range 0–127 clocks (0–1270 ns at 100 MHz). Controls the coincidence window width. Sourced from `TSCATTER_DELAY_REG[6:0]` in chan_in.vhd. ✅ verified 2026-04-16 — `chan_in.vhd:L294` (`OVERLAP_DELAY => TSCATTER_DELAY_REG(6 downto 0)`)
- `TSCATTER_DELAY_REG[14:8]` → `ASSERTION_DELAY` (7-bit one-shot stretch duration for CLEAN/DIRTY/MODULE outputs in chan_in.vhd). ✅ verified 2026-04-16 — `chan_in.vhd:L285`

## Connections to Other Modules
- **Instantiated by**: chan_in.vhd (5× normal instances for Ge/BGO pairs 4:0 via `DISC_MACH_BLK: for i in 0 to 4 generate`, plus 1× `CLOVER_DISC_MACH`) ✅ verified 2026-04-16 — `chan_in.vhd:L288–368`
- **Receives**: GE_DISC_FLAG and BGO_DISC_FLAG from DELAYED_DATA or RECOVERED_DATA (selected in chan_in.vhd), OVERLAP_DELAY from TSCATTER_DELAY_REG[6:0] ✅ verified 2026-04-16 — `chan_in.vhd:L294`
- **Sends**: CLEAN_EVENT, DIRTY_EVENT, BGO_ONLY_EVENT → ONE_SHOTS process in chan_in.vhd (stretched into HAVE_CLEAN, HAVE_DIRTY, HAVE_MODULE using ASSERTION_DELAY = TSCATTER_DELAY_REG[14:8]) ✅ verified 2026-04-16 — `chan_in.vhd:L329,339,349`

## See Also

- [RTRG_chan_in.md](RTRG_chan_in.md) — parent module; instantiates disc_mach 5+1× and extends output with one-shot stretching
- [RTRG_overlap_mach.md](RTRG_overlap_mach.md) — standalone overlap machine (same coincidence logic, but not used in RTRG — inlined here instead)
- [deep_fpga_RTRG.md](../deep_fpga_RTRG.md) — RTRG architecture; disc_mach role in BGO/Ge coincidence classification
