# chan_in.vhd — Plain English Summary
_Source: ~/FPGA_svn2git/RTRG_git/MAIN_FPGA/Source/chan_in.vhd_
_Summarized: 2026-04-15_
Stability: C3 - Structural / stable

## Table of Contents

- [Purpose](#purpose)
- [Ports](#ports)
- [Digitizer Data Word Format (18 bits)](#digitizer-data-word-format-18-bits)
- [Key Logic / State Machine](#key-logic--state-machine)
  - [Sub-components instantiated](#sub-components-instantiated)
  - [REMAP_BITS_PROC state machine (3 states)](#remap_bits_proc-state-machine-3-states)
  - [ONE_SHOTS process](#one_shots-process)
  - [CLEAN_DIRTY control register modes](#clean_dirty-control-register-modes)
  - [VETO_GEN_PROC](#veto_gen_proc)
  - [COARSE_GE_SUM](#coarse_ge_sum)
- [Key Constants / Parameters](#key-constants--parameters)
- [Connections to Other Modules](#connections-to-other-modules)
- [See Also](#see-also)

## Purpose
Encapsulates one complete Router input channel — everything required to receive 18-bit SERDES data from one digitizer, undo DC balance, apply per-bit timing delays, classify Ge/BGO hit pairs as CLEAN/DIRTY/BGO-only events, and report X-plane and Y-plane hit bitmaps plus multiplicity counts up to the Master Trigger. Supports both DFMA (simple X/Y bit mapping) and DGS (Ge+BGO coincidence classification) modes via a control register. Also handles Clover detector geometry (added 2015-01-19).

## Ports
Key signal ports (clocks `mclk`, `LVDS_RCLK`, and reset `mrst` omitted):

| Signal | Dir | Width | Description |
|---|---|---|---|
| `DATA_IN` | in | 18 | Raw SERDES word from digitizer |
| `BIT_DELAY` | in | CHAN_DELAY_ARRAY | Per-bit delay values (up to 32 taps = 640 ns) |
| `LOAD_BIT_DELAY` | in | 1 | Pulsed: load BIT_DELAY into delay buffers |
| `INPUT_MASK` | in | 1 | If '1', treat channel as dead (zero all outputs) |
| `SERDES_LOCK` | in | 1 | LOCK* from SERDES (active-low); '1' = lost lock, causes local reset |
| `X_PLANE_MAP` | in | 10 | Bitmap: which disc bits belong to X-plane |
| `Y_PLANE_MAP` | in | 10 | Bitmap: which disc bits belong to Y-plane |
| `CLEAN_DIRTY` | in | 16 | Mode control register (see Key Logic) |
| `TSCATTER_DELAY_REG` | in | 16 | [14:8]=assertion delay, [6:0]=overlap (Compton scatter) delay |
| `ENABLE_VETO` | in | 1 | '1': emit live channel vetoes; '0': hold vetoes at zero |
| `X_PLANE_BITS` | out | 10 | Bitmap of X-plane hits this clock |
| `Y_PLANE_BITS` | out | 10 | Bitmap of Y-plane hits this clock |
| `X_PLANE_COUNT` | out | 4 | Popcount of X-plane hits (0–10) |
| `Y_PLANE_COUNT` | out | 4 | Popcount of Y-plane hits (0–10) |
| `COARSE_GE_SUM` | out | 3 | Sum of fast/coarse Ge disc bits [4:0] (bypasses clock-crossing FIFO) |
| `LIVE_CHANNEL_VETO` | out | 10 | Per-Ge-channel veto requests driven by DIRTY/BGO-only events |
| `RAW_DATA_OUT` | out | 16 | Either DELAYED_DATA (DGS mode) or RECOVERED_DATA ✅ verified 2026-04-19 — `chan_in.vhd:L220` (`RECOVERED_DATA(15:10) & DELAYED_DATA` when `CLEAN_DIRTY(15)=1`, else `RECOVERED_DATA`) |

## Digitizer Data Word Format (18 bits)
```
[17]=CG (clock guard)  [16]=FLG (sync flag)
[15:10]=CD9..CD4 (fast/coarse Ge discriminators, channels 9:4)
[10:1]=D09..D00 (discriminator bits: 9:5=Ge center, 4:0=BGO sum)
[0]=POL (DC balance polarity)
```
After DCBAL_IN removes DC balance, RECOVERED_DATA[15] = FLG, [14:10] = coarse Ge, [9:5] = Ge, [4:0] = BGO.

## Key Logic / State Machine

### Sub-components instantiated
- **DCBAL_IN (U1)**: Undoes DC balance inversion from SERDES, crosses clock domains (RCLK→MCLK via FIFO), outputs `RECOVERED_DATA` (16-bit) and `FAST_D_OUT` (16-bit, bypasses FIFO for lowest-latency coarse Ge access).
- **DPRAM_RWA_RB_MxN × 10**: One delay-line FIFO per discriminator bit; depth=32 taps (20 ns/tap = up to 640 ns). Corrects timing skew between Ge and BGO discriminators. Output = `DELAYED_DATA[9:0]`.
- **discriminator_mach × 5**: One per Ge/BGO pair (Ge=bits[9:5], BGO=bits[4:0]). Outputs one-tick-wide pulses: `CLEAN_EVENT`, `DIRTY_EVENT`, `BGO_ONLY_EVENT`.
- **CLOVER_DISC_MACH (1 instance)**: Single disc_mach for Clover geometry; Ge input = OR of bits[3:0], BGO = bit[4].
- **plane_bit_count (U2, U3)**: LUT popcount over 10 bits → 4-bit count for X and Y planes respectively.

### REMAP_BITS_PROC state machine (3 states)
- **IDLE**: Zero all outputs. Exits to WAIT_DIG_FLAG when `INPUT_MASK='0'`. ✅ verified 2026-04-19 — `chan_in.vhd:L546-560` (REMAP_STATE <= IDLE reset, IDLE→WAIT_DIG_FLAG when not masked)
- **WAIT_DIG_FLAG**: Hold zeros. Advances to REMAP_BITS only when `RECOVERED_DATA[15]` (FLG) = '1'. This synchronizes all Router channels to the same data boundary. ✅ verified 2026-04-19 — `chan_in.vhd:L565-573` (WAIT_DIG_FLAG: `if RECOVERED_DATA(15)='1' then REMAP_BITS`)
- **REMAP_BITS**: Runs continuously. Masks discriminator bits against X_PLANE_MAP / Y_PLANE_MAP. If `CLEAN_DIRTY(15)='1'`, uses `DELAYED_DATA` (bit-delay compensated); else uses `RECOVERED_DATA`. Also drives `ANY_X` / `ANY_Y` flags. ✅ verified 2026-04-19 — `chan_in.vhd:L580-600` (X_BITS <= X_PLANE_MAP AND RECOVERED_DATA(9:0); stays in REMAP_BITS forever until reset)

### ONE_SHOTS process
Each of the 5 disc_mach pairs feeds three retriggerable one-shots (HAVE_CLEAN, HAVE_DIRTY, HAVE_MODULE). Timer width = 7 bits (ASSERTION_DELAY = TSCATTER_DELAY_REG[14:8], max 127 clocks). If a new event arrives before the timer expires, it restarts. Clover mode has only HAVE_CLOVER_CLEAN and HAVE_CLOVER_DIRTY.

### CLEAN_DIRTY control register modes
| CLEAN_DIRTY[3:0] | X_SELECT source | Y_SELECT (CLEAN_DIRTY[7:4]) |
|---|---|---|
| 0000 | `X_BITS` (DFMA raw mapping) | `Y_BITS` |
| 0001 | `HAVE_CLEAN[4:0]` (DGS clean) | HAVE_CLEAN |
| 0010 | `HAVE_DIRTY[4:0]` (DGS dirty) | HAVE_DIRTY |
| 0100 | `HAVE_MODULE[4:0]` (DGS module) | HAVE_MODULE |
| 0101 | `HAVE_BGO_ONLY[4:0]` (DGS BGO tuning mode) | HAVE_BGO_ONLY |
| 1000 | `HAVE_CLOVER_CLEAN` (Clover) | HAVE_CLOVER_DIRTY |

`CLEAN_DIRTY(15)='1'` additionally switches raw bit source to delay-corrected `DELAYED_DATA`.

✅ verified 2026-04-19 — `chan_in.vhd:L507-518` (X_SELECT/Y_SELECT mux: all 5 modes + DFMA default; added BGO-only mode 0101 confirmed at L510/517, added 2023-04-27 per header comment L501)

### VETO_GEN_PROC
When ENABLE_VETO='1': ✅ verified 2026-04-19 — `chan_in.vhd:L438-461`
- `LIVE_CHANNEL_VETO[10:6]` ← DIRTY_EVENT[4:0] (Ge center channels) ✅ verified 2026-04-19 — `chan_in.vhd:L446-450`
- `LIVE_CHANNEL_VETO[5:1]` ← DIRTY_EVENT[4:0] OR BGO_ONLY_EVENT[4:0] (BGO channels) ✅ verified 2026-04-19 — `chan_in.vhd:L454-458`

### COARSE_GE_SUM
5-bit sum of `FAST_D_OUT[14:10]` (coarse Ge bits), added one bit at a time into a 3-bit result. Zeroed if INPUT_MASK='1'. Low-latency path (bypasses FIFO). ✅ verified 2026-04-16 — `chan_in.vhd:L211-213` (`COARSE_GE_BITS <= FAST_D_OUT(14 downto 10)` when not masked; sum is bitwise add of 5 bits into 3-bit result)

## Key Constants / Parameters
- Delay depth: 32 taps × 20 ns = **640 ns max per-bit delay** ✅ verified 2026-04-16 — `chan_in.vhd:L266` (`DEPTH_pwr2 => 5` → 2^5=32 samples, comment confirms 640 ns)
- ASSERTION_DELAY: `TSCATTER_DELAY_REG[14:8]`, 7-bit (0–127 clocks) ✅ verified 2026-04-16 — `chan_in.vhd:L285` (`ASSERTION_DELAY <= TSCATTER_DELAY_REG(14 downto 8)`)
- OVERLAP_DELAY (Compton scatter window): `TSCATTER_DELAY_REG[6:0]`, 7-bit, passed to disc_mach ✅ verified 2026-04-16 — `chan_in.vhd:L294` (`OVERLAP_DELAY => TSCATTER_DELAY_REG(6 downto 0)`)

## Connections to Other Modules
- **Receives from**: Digitizer via SERDES (DATA_IN); VME control registers (X_PLANE_MAP, Y_PLANE_MAP, CLEAN_DIRTY, TSCATTER_DELAY_REG, BIT_DELAY, INPUT_MASK, ENABLE_VETO)
- **Sends to Master Trigger (via Router TOP)**: X_PLANE_BITS, Y_PLANE_BITS, X_PLANE_COUNT, Y_PLANE_COUNT, COARSE_GE_SUM, ANY_X, ANY_Y
- **Sends to Router TOP veto logic**: LIVE_CHANNEL_VETO[10:1] (consumed by ADD_VETOES process)
- **Instantiates**: disc_mach.vhd (×5+1 Clover), DCBAL_IN, DPRAM_RWA_RB_MxN (×10), plane_bit_count (×2)

## See Also

- [RTRG_disc_mach.md](RTRG_disc_mach.md) — discriminator coincidence machine instantiated 5+1× by this module
- [RTRG_router_data_path.md](RTRG_router_data_path.md) — parent module (instantiates chan_in × 8)
- [RTRG_top.md](RTRG_top.md) — RTRG top-level context; VME register wiring to chan_in
- [deep_fpga_RTRG.md](../deep_fpga_RTRG.md) — RTRG architecture overview; X/Y plane bit mapping
- [260E_trigger_scheme.md](../260E_trigger_scheme.md) — trigger scheme context; chan_in role in X/Y multiplicity counting
