# MTRG trigger_comp_defs.vhd + trigger_top_comp_defs.vhd — Component Declaration Packages
Stability: C2 - Active / semi-stable

Sources:
- `~/FPGA_svn2git/MTRG_git/MAIN_FPGA/trunk/Source/trigger_comp_defs.vhd` (332 lines)
- `~/FPGA_svn2git/MTRG_git/MAIN_FPGA/trunk/Source/trigger_top_comp_defs.vhd` (1295 lines)

---

## Purpose

These are VHDL **package** files containing only component declarations — no logic. They allow other VHDL files to instantiate sub-modules without needing to know their full port lists inline. Both are referenced via `use WORK.trigger_comp_defs.all` and `use WORK.trigger_top_comp_defs.all`.

---

## trigger_comp_defs Package

Components declared (sub-design level):

| Component | Description |
|-----------|-------------|
| `DCBAL_IN` | DC-balanced SERDES input demux: RCLK → MCLK domain, 18-bit in → 16-bit out, async FIFO inside |
| `DCBAL_IN_NOFIFO` | Simpler DCBAL_IN variant without internal FIFO |
| `fifo_16x1023_async` | 16-bit × 1023-deep async FIFO (Xilinx core) |
| `fifo_16x64K_async` | 16-bit × 64K async FIFO with almost-full/empty, prog thresholds |
| `fifo_16x16K_async` | 16-bit × 16K async FIFO with rd_data_count |
| `FIFO_FWFT_80X16IN_20X64OUT_SEPCLK` | Width-converting FIFO: 80-bit write × 16 deep → 20-bit read × 64 deep, separate clocks, FWFT |
| `FIFO_IND_16Wx1024D_STD` | 16-bit × 1024 FIFO with programmable full threshold |
| `trig_algo_support` | Generic trigger algorithm support layer (all MTRG trigger algorithms use this) |
| `disparity_lookup` | 8-bit → 4-bit disparity lookup table (for DC-balance encoding) |
| `dc_balance_mach` | DC balance state machine: takes 18-bit input, produces 18-bit balanced output |
| `DELAY_LINE` | Programmable shift-register delay line (16-bit delay register) |
| `pos_finder` | TDC thermometer-code edge position finder (11- or 12-bit slice → 4-bit position + valid) |
| `vernier_pos_finder` | Full TDC vernier position decoder: 64-bit TDC data → 6-bit position + 16-bit offset |
| `pipeline_unit` | One stage of pipelined vernier position resolver |
| `dly_buf_16kx1` | 1-bit × 16K dual-port BRAM delay buffer |
| `EVENT_FIFO` | Event-aware FIFO with event counter (tracks full events vs. individual words) |

---

## trigger_top_comp_defs Package

Components declared (top-level / integration):

| Component | Description |
|-----------|-------------|
| `timestamp` | 48-bit timestamp counter with test rates and slave sync |
| `sync_capture_controller` | Sync capture controller |
| `sync_capture_counter` | Sync capture counter |
| `mstr_mach` | Master State Machine — full 20-frame TTCL output |
| `LINK_TX_BLOCK` | DC-balanced SERDES fan-out to 11 output links |
| `LINK_LRU_RX` | Link receiver (LRU variant) |
| `dc_balance_mach` | DC balance machine (also in comp_defs) |
| `link_init` | Link initialization helper |
| `registers` | Full VME register map (~120 registers, FIFOs, RAMs) |
| `led_ctl` | LED controller |
| `aux_io` | AUX I/O port mux (AUX A/B, NIM outputs, target wheel encoder) |
| `fifo_16x1023_async` | (re-declared) |
| `eight_mt_channel` | 8-channel MT data handler |
| `cpld_trig` | CPLD fast-sum trigger (thin wrapper over trig_algo_support) |
| `trig_collect` | Trigger collector state machine |
| `CHAN_FIFO_WRITE_CTL` | Channel FIFO write controller |
| `ila_64`, `ila_80`, `ila_128` | Xilinx ILA debug cores (64/80/128-bit data widths) |
| `icon` | Xilinx ICON debug core |
| `tdc_chain_cont` | TDC chain controller: 250 MHz carry-chain TDC, 4-phase clock, auto-sample, output FIFO |
| `trig_mon_collect` | Trigger monitor FIFO collector → Mon FIFO 7 |
| `DCBAL_IN` | (re-declared) |
| `sum_hits_xy` | XY coincidence trigger (fires when both X and Y sums exceed threshold) |
| `sum_hits_x` | X-plane sum trigger |
| `GITMO_RCV_MACH` | GITMO receive state machine |
| `MYRIAD_RCV_MACH` | MYRIAD receive state machine |
| `MYRIAD_TRIGGER` | MYRIAD trigger algorithm |
| `SERDES_RX_Mach` | 20-frame SERDES receiver FSM (lock, frame decoders, VETO_EVENT) |
| `GITMO_TRIGGER` | GITMO trigger algorithm |
| `REMOTE_MASTER_TRIG_SUPPORT` | Remote/cross-system trigger algorithm (Link R) |
| `NIM_DELAY_LINE` | NIM output delay line |
| `dly_buf_16kx1` | (re-declared) |
| `SLOW_CLOCKS` | Slow clock generator |
| `local_trig_coinc` | Local-vs-local trigger coincidence algorithm |

---

## tdc_chain_cont Key Ports (from trigger_top_comp_defs L722–L751)

Notable because it was listed as not-yet-analyzed:

| Port | Direction | Description |
|------|-----------|-------------|
| `TDC_CLOCK_0/90/180/270` | in | 4-phase 250 MHz TDC sampling clocks |
| `CLOCK_50MHz` | in | Added 2025-01-31 for alternate reset logic |
| `CLOCK_100MHz` | in | Autosample machine runs in mclk_2x domain |
| `TIMESTAMP[15:0]` | in | Lower 16 bits of running timestamp |
| `TS_SAMP_PHASE` | in | Timestamp sampling phase control |
| `BIT_IN` | in | Signal to time-stamp (the TDC input bit) |
| `TDC_RESET` | in | Master reset (resampled into 100 MHz domain internally) |
| `ENABLED_NONVETOED_TRIG_ACK[8:1]` | in | Collection of trigger-accepted signals from all algorithms |
| `TDC_TRIG_SEL_MASK[8:1]` | in | Selects which trig_acks capture TDC events |
| `ABORT_TDC_HIT` | in | Asserted if no TDC hit within expected window after trigger |
| `DIAG_ALLOWED_LATENCY[7:0]` | in | Diagnostic: allowed latency window |
| `NUM_TDC_WORDS[9:4]` | in | Number of TDC words to pull from Mon FIFO |
| `BUF_WANT_NEXT_TDC` | out | Handshake: ready for next TDC word |
| `TDC_FIFO_WE` | out | Write enable to FIFO (implemented in trig_mon_collect) |
| `TDC_FIFO_DATA_READY` | out | Pulse when FIFO is fully written |
| `TDC_FIFO_DATA[15:0]` | out | Data to FIFO |

---

## Notes

- These files contain **no synthesizable logic** — purely component declarations for use in instantiating sub-modules.
- Both packages use `XilinxCoreLib` (ISE era) and `UNISIM` (translate_off guards).
- The `tdc_chain_cont` component has 4-phase clock inputs suggesting it drives Xilinx carry-chain primitives for sub-clock TDC resolution.

---

## See Also

- Individual module analysis files for each component listed above
- [`MTRG_pos_finder.md`](./MTRG_pos_finder.md) — pos_finder analysis
- [`MTRG_sum_hits_XY.md`](./MTRG_sum_hits_XY.md) — sum_hits_xy analysis
- [`deep_fpga_MTRG_MAIN.md`](../deep_fpga_MTRG_MAIN.md) — MTRG top-level architecture
- [`fpga.md`](../fpga.md) — VHDL index

---

*Analysis date: 2026-04-24*
