# VHDL Summary Progress
Stability: C2 - Active / semi-stable

Quick checklist/index of detailed VHDL module analysis pages under `knowledgeBase/vhdl/`. This file tracks coverage status rather than firmware behavior details.

## RTRG (~/FPGA_svn2git/RTRG_git/MAIN_FPGA/Source/)
- [x] chan_in.vhd → RTRG_chan_in.md
- [x] disc_mach.vhd → RTRG_disc_mach.md
- [x] overlap_mach.vhd → RTRG_overlap_mach.md
- [x] router_data_path.vhd → RTRG_router_data_path.md
- [x] TOP.VHD → RTRG_top.md

## MTRG (~/FPGA_svn2git/MTRG_git/MAIN_FPGA/trunk/Source/)
- [x] top.vhd → MTRG_top.md
- [x] eight_mt_channel.vhd → MTRG_eight_mt_channel.md
- [x] mt_input_channel.vhd → MTRG_mt_input_channel.md  (NOTE: at ~/FPGA_svn2git/MTRG_git/MAIN_FPGA/mt_input_channel.vhd)
- [x] sum_hits_X.vhd → MTRG_sum_hits_X.md
- [x] calc_total_sum.vhd → MTRG_calc_total_sum.md
- [x] MYRIAD_RCV_MACH.vhd → MTRG_MYRIAD_RCV_MACH.md  (tag 20220705)
- [x] MYRIAD_TRIGGER.vhd → MTRG_MYRIAD_TRIGGER.md     (tag 20220705)
- [x] mstr_mach.vhd → MTRG_mstr_mach.md               (Master State Machine, full 20-frame TTCL output)
- [x] local_trig_coinc.vhd → MTRG_local_trig_coinc.md (Local-vs-local trigger coincidence algorithm)

---

**Last reviewed:** 2026-04-22  
**Purpose:** Coverage checklist for per-module VHDL notes under `knowledgeBase/vhdl/`  
**See also:** [fpga.md](../fpga.md), [deep_fpga_RTRG.md](../deep_fpga_RTRG.md), [deep_fpga_MTRG_MAIN.md](../deep_fpga_MTRG_MAIN.md)
