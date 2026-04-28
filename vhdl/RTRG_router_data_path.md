# router_data_path.vhd — Plain English Summary
_Source: ~/FPGA_svn2git/RTRG_git/MAIN_FPGA/Source/router_data_path.vhd_
_Summarized: 2026-04-15_
Stability: C3 - Structural / stable

## Table of Contents

- [Purpose](#purpose)
- [Ports](#ports)
- [Link-L Output Word Format](#link-l-output-word-format-linkl_raw_data-16-bit-becomes-bits-161-of-18-bit-serdes-word)
- [Key Logic / State Machine](#key-logic--state-machine)
  - [CHANNEL_BLOCK (for i in 1 to 8)](#channel_block-for-i-in-1-to-8)
  - [SUM_HITS process](#sum_hits-process--3-rank-pipelined-adder-tree)
  - [FAST_COARSE_GE_SUM_PROC](#fast_coarse_ge_sum_proc--single-cycle-ge-aggregation)
  - [DIGITIZER_LOCK + DIGITIZER_ALLLOCK](#digitizer_lock--digitizer_alllock)
  - [CHAN_FIFO_CTL_BLOCK](#chan_fifo_ctl_block-for-i-in-1-to-8--diagnostic-channel-fifos)
- [Key Constants / Parameters](#key-constants--parameters)
- [Connections to Other Modules](#connections-to-other-modules)
- [See Also](#see-also)

## Purpose
Aggregates all eight Router input channels into a single 16-bit word sent to the Master Trigger via SerDes link L. Instantiates eight `chan_in` instances (one per digitizer), sums their X-plane and Y-plane hit counts via a 3-stage pipelined adder tree, builds the link-L output word, and routes per-channel data into diagnostic FIFOs. Also aggregates the 8-channel coarse Ge sum.

## Ports
Key ports:

| Signal | Dir | Width | Description |
|---|---|---|---|
| `DIG_LINK_RXs` | in | 8×18 | Raw SERDES data from digitizers 1–8 |
| `DIG_LINK_RCLK` | in | 8 | Per-channel SERDES receive clocks |
| `THROTTLE_REQUESTS` | in | 8 | Per-digitizer throttle requests (active high) |
| `ANY_THROTTLE_REQUEST` | in | 1 | OR of all throttle requests |
| `LOCK_BUS` | in | 8 | SERDES LOCK* status per channel |
| `ROUTER_LOCKED` | in | 1 | Router locked to Master |
| `INPUT_LINK_MASK_REG` | in | 16 | [7:0] = per-channel mask bits |
| `CLEAN_DIRTY_REG` | in | 16 | Mode control forwarded to all chan_in instances |
| `TSCATTER_DELAY_REG` | in | 16 | Timing delays forwarded to all chan_in instances |
| `X_PLANE_MAP_REG` | in | 8×16 | Per-channel X-plane bit mapping |
| `Y_PLANE_MAP_REG` | in | 8×16 | Per-channel Y-plane bit mapping |
| `DISCRIMINATOR_DELAYS` | in | BOARD_DELAY_ARRAY | Per-channel per-bit delay values |
| `LINKL_RAW_DATA` | out | 16 | Link-L word to Master Trigger (see format below) |
| `TOTAL_COARSE_GE_SUM` | out | 6 | Total fast Ge hit count across all 8 digitizers |
| `LIVE_CHANNEL_VETOES` | out | 8×10 | Per-channel veto bits (to TOP.VHD ADD_VETOES) |
| `CHAN_MON_FIFO_INs` | out | 8×16 | Data to 8 diagnostic FIFOs |
| `CHAN_MON_FIFO_WEs` | out | 8 | Write enables to 8 diagnostic FIFOs |
| `DIAG_INTER_COARSE_GE_SUM` | out | 16 | Two 5-bit intermediate Ge sums packed for diagnostic |
| `CHAN_MON_FIFO_CTL_REG` | in | 16 | 2-bit mode per channel: selects FIFO data source |
| `CHAN_MON_FIFO_WE_CTL_REG` | in | 16 | 2-bit WE mode per channel |
| `DIAG_PIN_CTL_REG` | in | 16 | Sub-mode for FIFO input mode "10" |
| `ENABLE_VETO` | in | 1 | Forwarded to all chan_in instances |

✅ verified 2026-04-27 — `router_data_path.vhd:L28-59` (entity port list; all port names, directions, and widths confirmed; note: DIAG_INTER_COARSE_GE_SUM VHDL comment erroneously says "six-bit" but signal type is `MBO_2x5_Array` = 5-bit — KB "5-bit" is correct)

## Link-L Output Word Format (LINKL_RAW_DATA, 16-bit becomes bits [16:1] of 18-bit SerDes word)
```
[15]    = ANY_THROTTLE_REQUEST
[14:8]  = Y-plane multiplicity sum (0–80, 7-bit)
[7]     = DATA_VALID = ALL_DIGITIZERS_LOCKED AND ROUTER_LOCKED
[6:0]   = X-plane multiplicity sum (0–80, 7-bit)
```
✅ verified 2026-04-24 — `router_data_path.vhd:L235-239` (SUM_HITS process rank-3 output; comment header L1-20 confirms full 18-bit SerDes word with CG/POL framing bits added by DC balance)

## Key Logic / State Machine

### CHANNEL_BLOCK (for i in 1 to 8)
Instantiates one `chan_in` per digitizer. Distributes shared control registers (CLEAN_DIRTY_REG, TSCATTER_DELAY_REG, ENABLE_VETO) identically to all channels; per-channel inputs are X/Y_PLANE_MAP_REG(i), DISCRIMINATOR_DELAYS(i), INPUT_LINK_MASK_REG(i-1), LOCK_BUS(i), DIG_LINK_RXs(i), DIG_LINK_RCLK(i).

### SUM_HITS process — 3-rank pipelined adder tree
Each clock builds LINKL_RAW_DATA via three sequential pipeline ranks (all in one clocked process, creating 1-cycle latency per rank):
- **Rank 1**: Pairs → 4× 5-bit sums (X: ch1+2, ch3+4, ch5+6, ch7+8; same for Y)
- **Rank 2**: Pairs of rank-1 → 2× 6-bit sums
- **Rank 3**: Final pair → 7-bit X and Y totals → LINKL_RAW_DATA[6:0] and [14:8]  
  Also sets [15]=throttle, [7]=data valid
✅ verified 2026-04-24 — `router_data_path.vhd:L216-239` (SUM_HITS process; rank-1 L218-225, rank-2 L227-231, rank-3 L232-239; note all 3 ranks execute in a single clocked process so latency is 1 clock, not 3)

### FAST_COARSE_GE_SUM_PROC — single-cycle Ge aggregation
- `INTER_COARSE_GE_SUM(0)` = sum of COARSE_GE_SUM(1..4) (each 3-bit → result 5-bit)
- `INTER_COARSE_GE_SUM(1)` = sum of COARSE_GE_SUM(5..8)
- `TOTAL_COARSE_GE_SUM` = INTER_COARSE_GE_SUM(0) + INTER_COARSE_GE_SUM(1) (6-bit, max 40)
✅ verified 2026-04-24 — `router_data_path.vhd:L150-158` (FAST_COARSE_GE_SUM_PROC; INTER_COARSE_GE_SUM declared as MBO_2x5_Array i.e. 2×5-bit; TOTAL is 6-bit slv(5 downto 0)). Note: DIAG_INTER_COARSE_GE_SUM packs as "000"&INTER(0)&"000"&INTER(1) into 16-bit word (L144-148).

### DIGITIZER_LOCK + DIGITIZER_ALLLOCK
Each channel: `DIGITIZER_LOCKED(i)='1'` if masked OR if LOCK_BUS(i)='0' (active-low lock).  
`ALL_DIGITIZERS_LOCKED='1'` only when all 8 bits locked. Used to qualify DATA_VALID in link-L word.
✅ verified 2026-04-24 — `router_data_path.vhd:L312-335` (DIGITIZER_LOCK and DIGITIZER_ALLLOCK processes; masked→locked='1', else NOT LOCK_BUS(i); ALL_DIGITIZERS_LOCKED='1' only if DIGITIZER_LOCKED="11111111")

### CHAN_FIFO_CTL_BLOCK (for i in 1 to 8) — diagnostic channel FIFOs
Two-bit mode per channel (CHAN_MON_FIFO_CTL_REG[2i-1:2i-2]):
| Mode | Data in FIFO |
|---|---|
| "00" | `RAW_DATA(i)` — 16-bit post-DC-balance data ✅ verified 2026-04-27 — `router_data_path.vhd:L265` |
| "01" | `ANY_Xs(i) & ANY_Ys(i) & ANY_THROTTLE_REQUEST & "00000" & THROTTLE_REQUESTS` — any-X/Y hit flags + throttle data ✅ verified 2026-04-27 — `router_data_path.vhd:L268` |
| "10" | Sub-mux via DIAG_PIN_CTL_REG[9:7] (8 cases): cases 0–4 = `Y_PLANE_COUNT[15:12] & '0' & STATE_MON(n)[2:0] & X_PLANE_COUNT[7:4] & "00" & GE_EDGE & BGO_EDGE`; case 5 = `ANY_Xs & ANY_Ys & "0000" & GE_EDGE_DIAGS(5) & BGO_EDGE_DIAGS(5)`; cases 6/7 = 0x0110/0x0111 (debug sentinels); others = 0x0BAD ✅ verified 2026-04-27 — `router_data_path.vhd:L273-284` |
| "11" | `Y_PLANE_COUNT[2:0] & X_PLANE_COUNT[2:0] & RAW_DATA[9:0]` ✅ verified 2026-04-27 — `router_data_path.vhd:L287` |

Two-bit write enable per channel (CHAN_MON_FIFO_WE_CTL_REG):
| Mode | Write when… |
|---|---|
| "00" | Never ✅ verified 2026-04-27 — `router_data_path.vhd:L298-300` |
| "01" | Always ✅ verified 2026-04-27 — `router_data_path.vhd:L300-301` |
| "10" | This channel's `CHAN_FIFO_DATA_NONZERO(i)` nonzero (checked from `xxxCHAN_MON_FIFO_INs[9:0]`) ✅ verified 2026-04-27 — `router_data_path.vhd:L302-307` |
| "11" | Any channel's `CHAN_FIFO_DATA_NONZERO` byte nonzero (`/= X"00"`) ✅ verified 2026-04-27 — `router_data_path.vhd:L308-311` |

Input is **2-stage clocked pipeline**: mux assigns `xxxCHAN_MON_FIFO_INs` (combinatorial) → clocked to `xxCHAN_MON_FIFO_INs` → clocked to `xCHAN_MON_FIFO_INs` → combinatorial output to `CHAN_MON_FIFO_INs` (L153). Only `xx` and `x` stages are registered; `xxx` is purely combinatorial. ✅ verified 2026-04-27 — `router_data_path.vhd:L84-86,L153,L252-253` (signals declared; x←xx←xxx clocked at L252-253; output L153 combinatorial). **Correction from prior summary**: KB previously said "triple-registered" — accurate count is 2 clock-stage pipeline.

## Key Constants / Parameters
- 8 input channels (for i in 1 to 8)
- X/Y sum max: 80 (8 channels × 10 bits each)
- COARSE_GE_SUM max: 40 (8 channels × 5 coarse Ge bits each)

## Connections to Other Modules
- **Instantiates**: chan_in × 8
- **Receives control from**: TOP.VHD (all VME register signals)
- **Sends to Master Trigger** (via TOP.VHD → SerDes link L): LINKL_RAW_DATA[15:0]
- **Sends to TOP.VHD**: LIVE_CHANNEL_VETOES (8×10), TOTAL_COARSE_GE_SUM, CHAN_MON_FIFO_INs, CHAN_MON_FIFO_WEs

## See Also

- [RTRG_chan_in.md](RTRG_chan_in.md) — per-digitizer channel processor (instantiated × 8 by this module)
- [RTRG_top.md](RTRG_top.md) — parent; instantiates this module as U8
- [deep_fpga_RTRG.md](../deep_fpga_RTRG.md) — RTRG architecture; X/Y plane aggregation and SerDes uplink format
- [260E_trigger_scheme.md](../260E_trigger_scheme.md) — trigger scheme context; RTRG→MTRG data path
