# MTRG Master State Machine — `mstr_mach.vhd`
Stability: C3 - Structural / stable

**Source:** `~/FPGA_svn2git/MTRG_git/MAIN_FPGA/trunk/Source/mstr_mach.vhd`  
**Author:** John T. Anderson (ANL)  
**Lines:** 1158  
**Generated:** 2026-04-22 by General DGS

---

## Overview

`mstr_mach` is the **TTCL Master State Machine** inside the MTRG Main FPGA. It runs continuously at 50 MHz and emits the 20-frame, 5-word-per-frame command sequence defined by the Trigger Timing and Control Link (TTCL) specification. Everything downstream in the DGS system (RTRGs, DIGs) receives its timing and control from the output of this machine.

The machine operates in two modes:
- **Local master:** advances its own frame/word counter freely at 50 MHz, completing a full 20-frame cycle every 2 µs (100 clocks × 20 ns).
- **Remote master (satellite):** when `PROPAGATE_SYNC='1'`, it waits for `MSTR_MACH_START_FLAG` from the Link L receive machine before beginning Frame 1. This synchronizes it with a higher-level "Monarch" master trigger.

---

## Port Summary

| Port | Direction | Description |
|------|-----------|-------------|
| `CLK` | in | 50 MHz system clock |
| `RST` | in | Active-high power-on reset |
| `SYS_TIME[47:0]` | in | Continuously running 48-bit timestamp |
| `ROLLOVER` | in | Timestamp rollover flag; sets argument byte to 0xFF in Frame 1 |
| `IMPERATIVE_FLAG_REQ` | in | From VME: triggers an Imperative SYNC (restart timestamp) |
| `STARTING_TIMESTAMP[47:0]` | in | Timestamp value to broadcast in Imperative SYNC (Frame 1) |
| `TRIG_DES_FIFO_RE` | out | Read enable to the Trigger Decision FIFO |
| `TRIG_DES_FIFO_DATA[15:0]` | in | Data from Trigger Decision FIFO |
| `TRIG_DES_FIFO_EMPTY` | in | Empty flag of Trigger Decision FIFO |
| `TRIG_COLLECT_FLAG` | out | Enables trigger collection FIFO fill machine (asserted at F12/W1) |
| `TRIG_COLLECT_RST` | out | Resets trigger Decision FIFO (asserted at F11/W1) |
| `MSM_MON_FIFO_SELECT_REG[15:0]` | in | Bit mask: selects what frames/conditions to capture in monitor FIFO |
| `MSM_MON_FIFO_WE` | out | Write enable to board-wide Monitor FIFO |
| `MSM_MON_DATA[15:0]` | out | Data presented to monitor FIFO (CMDout or TRIG_DES_FIFO_DATA) |
| `LOCAL_TRIG_MON_DET_DATA[15:0]` | in | Detector state at time of trigger (e.g., target wheel) → Frame 2 W1 |
| `LOCAL_TRIG_MON_XTRA_DATA[15:0]` | in | Extra state data at trigger (GLOBAL_X_TOTAL & GLOBAL_Y_TOTAL) → Frame 2 W2 |
| `SYSTEM_VETO_STATE[15:0]` | in | Full veto state vector → Frame 2 W3 (broadcast to links L/R/U) |
| `FRAME_12_REQ_FLAG` | in | VME: request Frame 12 (Router/DIG reset) command |
| `FRAME_12_SENT_FLAG` | out | Frame 12 was transmitted |
| `FRAME_12_DATA[1..5]` | in | 5-word payload for Frame 12 |
| `FRAME_14_REQ_FLAG` | in | VME: request Frame 14 (Digitizer Tester / MγRIAD) command |
| `FRAME_14_SENT_FLAG` | out | Frame 14 was transmitted |
| `FRAME_14_DATA[1..5]` | in | 5-word payload for Frame 14 |
| `ASYNC_CMD_FLAG` | in | Pulse: async (GRETINA-style) command data is available in FIFO |
| `ASYNC_CMD_FIFO_RE` | out | Read enable to async command FIFO |
| `ASYNC_CMD_FIFO_DATA[15:0]` | in | Data from async command FIFO |
| `ASYNC_CMD_FIFO_EMPTY` | in | Async command FIFO empty flag |
| `FRAME_16_REQ_FLAG` | in | VME: request Frame 16 (synchronous system capture) command |
| `FRAME_16_SENT_FLAG` | out | Frame 16 was transmitted |
| `FRAME_16_DATA[1..5]` | in | 5-word payload for Frame 16 |
| `PROPAGATE_SYNC` | in | 1 = running as remote master; wait for Link L sync |
| `MSTR_MACH_START_FLAG` | in | From Link L RX machine: OK to begin a new cycle (remote master mode) |
| `ANY_TRIGGER_VETO` | in | Any trigger currently vetoed → sent on Frame 14 W4 to L/R links |
| `REMOTE_MASTER_VETO` | in | Remote master is in veto → sent on Frame 14 W4 to L/R links |
| `LINK_L_COMMAND_OUT[15:0]` | out | Command word to Link L (F8 trigger data replaced by NULL) |
| `LINK_R_COMMAND_OUT[15:0]` | out | Command word to Link R (F9 trigger data replaced by NULL) |
| `LINK_U_COMMAND_OUT[15:0]` | out | Command word to Link U (F10 trigger data replaced by NULL) |
| `COMMAND_OUT[15:0]` | out | Command word to all Routers (links A–H) |
| `SYNC_FLAG` | out | Pulsed during Frame 1 (any SYNC) |
| `IMP_SYNC_FLAG` | out | Pulsed at Frame 1 W5 during Imperative SYNC |
| `LATCHED_IMPERATIVE_FLAG` | out | Set from IMPERATIVE_FLAG_REQ; released at F1/W5 |
| `STATE_MON[7:0]` | out | Diagnostic: CURRENT_FRAME & CURRENT_WORD for ILA |

---

## Frame-by-Frame Command Map

The machine emits four simultaneous outputs: `CMDout` (links A–H), `LCMDout` (link L), `RCMDout` (link R), `UCMDout` (link U). Frame 8, 9, and 10 null out the corresponding satellite link to prevent re-propagation of externally-originated triggers back to themselves.

| Frame | Name | Description |
|-------|------|-------------|
| **1** | SYNC | Always emitted. Cmd byte 0x01 (normal) or 0x81 (Imperative). Arg byte 0x00 (normal) or 0xFF (timestamp rollover). Words 2–4: 48-bit timestamp. Word 5: DC balance (0x0000). During Imperative SYNC, timestamp words come from `STARTING_TIMESTAMP` instead of the live counter. ✅ verified 2026-04-24 — mstr_mach.vhd:L362-376 (Imperative→0x81/normal→0x01; ROLLOVER→0xFF/normal→0x00; W2-W4 STARTING_TIMESTAMP vs TIMESTAMP_OF_SYNC) |
| **2** | Debug / Detector Status | Words 1–2: `LOCAL_TRIG_MON_DET_DATA` and `LOCAL_TRIG_MON_XTRA_DATA` (detector state at last trigger). Word 3: `SYSTEM_VETO_STATE` (broadcast on L/R/U with own remote-master bit masked out). Word 4: 0xAAAA (null); also pre-reads TRIG_DES_FIFO if non-empty. Word 5: 0x0000 (filler). Monitor FIFO can optionally capture this frame on bit 6 (nonzero data) or bit 7 (changed data). |
| **3–7** | Trigger Decision (internal) | If TRIG_DES_FIFO has data, words 1–4 stream trigger decision words; word 5 is filler. FIFO RE is carefully pipelined so data lands one clock after RE (one-cycle FIFO latency). All four outputs (A–H, L, R, U) receive the trigger data. Continues across multiple frames as long as FIFO is non-empty. |
| **8** | Trigger Decision (Link L external) | Same as frames 3–7 but `LCMDout` is forced to NULL (0xAAAA) — prevents re-propagation of an external trigger from Link L back to itself. |
| **9** | Trigger Decision (Link R external) | Same but `RCMDout` is forced to NULL. |
| **10** | Trigger Decision (Link U external) | Same but `UCMDout` is forced to NULL. Final frame for FIFO draining — RE is explicitly dropped mid-frame. |
| **11** | Spare | NULL frame (0xAAAA for words 1–4, filler for W5). Also asserts `TRIG_COLLECT_RST` at W1 to reset decision FIFO. |
| **12** | Router / Data Generator Reset | Synchronous command frame for resetting Routers and DIGs. Words 1–5 carry `FRAME_12_DATA[1..5]` (nominally: 0x0001 reset command, Router counter resets, Router FIFO resets, DIG resets, spare). Nulled if `FRAME_12_REQ_FLAG` not set. `FRAME_12_SENT_FLAG` pulses at W4. Frame 12 also asserts `TRIG_COLLECT_FLAG` at W1, triggering the trigger collection machine. |
| **13** | Demand Front-End Slow Data | Fixed pattern: W1=0x40FB (cmd byte 0x40), W2=0xA5A5, W3=0x5A5A, W4=0xA5A5, W5=0xA5A5. This causes all DIGs to upload their slow-data registers. ✅ verified 2026-04-24 — mstr_mach.vhd:L803-816 (W1=X"40FB"; W2=NULL_VALUE_A5; W3=NULL_VALUE_5A; W4=NULL_VALUE_A5; W5=NULL_VALUE_A5) |
| **14** | Digitizer Tester / MγRIAD | Synchronous command frame for Digitizer Tester control (words 1–3: cmd, timestamp comparison, pulse-count/delay). W4: veto status bits on L/R (`ANY_TRIGGER_VETO[15] & REMOTE_MASTER_VETO[14]` | lower 14 bits of FRAME_14_DATA[4]). Added 2021-06-16. W5: spare / pipeline for Frame 15 async setup. |
| **15** | Async Front-End Command (GRETINA) | If `ASYNC_CMD_REQUEST` is set (a VME pulse latched and synchronized to F19/W5 boundary), words 1–5 are streamed from the async command FIFO. All four outputs receive same data. `ASYNC_CMD_ACK` pulses at W5 to clear the request. Otherwise, NULL frame. |
| **16** | Synchronous System Capture | Synchronous command frame; words 1–5 from `FRAME_16_DATA[1..5]`. `FRAME_16_SENT_FLAG` pulses at W4. |
| **17** | Auxiliary Detector (removed) | Defined in older spec; removed 2022-04-19 and replaced with NULL. |
| **18** | Spare | NULL. |
| **19** | Spare | NULL. Last frame before EOC; used to pipeline async command request to Frame 15. |
| **20** | End-of-Cycle (EOC) | Fixed DC-balance alternating pattern: W1=0xFFFF, W2=0x0000, W3=0xFFFF, W4=0x0000, W5=0x5555. Timestamp is latched at F20/W4 for use in the next SYNC frame. ✅ verified 2026-04-24 — mstr_mach.vhd:L1138-1160 (W1=X"FFFF"; W2=X"0000"; W3=X"FFFF"; W4=X"0000"; W5=X"5555") |

---

## Key Sub-Processes

### Word/Frame Counter (`WORD_COUNTER`)
- Counts `CURRENT_WORD` 1→5, `CURRENT_FRAME` 1→20, then wraps.
- **Local master:** wraps at Frame 20 / Word 5.
- **Remote master** (`PROPAGATE_SYNC='1'`): resets to F1/W1 on `MSTR_MACH_START_FLAG` — synchronizes to external Monarch.

### Imperative Flag (`IMPERATIVE_FLAG_PROC`)
- `IMPERATIVE_FLAG_REQ` sets `xLATCHED_IMPERATIVE_FLAG` immediately.
- Released synchronously only when `IMPERATIVE_FLAG_REQ='0'` AND `CURRENT_FRAME=1, CURRENT_WORD=5`.
- While latched: holds timestamp counter in reset (ensuring Imperative SYNC restarts count from zero at F2/W1), and sets command byte to 0x81.

### Timestamp Capture (`SYNC_CAPTURE_PROC`)
- Latches `SYS_TIME` into `TIMESTAMP_OF_SYNC` at F20/W4.
- This captured value is what gets broadcast in Frame 1 words 2–4 — a one-frame-old snapshot, not the live counter.

### Async Command Sync (`ASYNC_REQ_PROC`)
- VME writes a one-tick pulse via `ASYNC_CMD_FLAG`.
- Machine latches this as `ASYNC_CMD_REQUEST_INT` immediately, then synchronizes it to `ASYNC_CMD_REQUEST` at F19/W5 boundary.
- This ensures Frame 15 async data always starts clean at the beginning of a frame.
- `ASYNC_CMD_ACK` at F15/W5 clears the request.

### Monitor FIFO Write Enable (`MSM_MON_FIFO_WE`)
Combinatorial. Bit-selectable capture modes via `MSM_MON_FIFO_SELECT_REG`:

| Bit | Capture condition |
|-----|-------------------|
| 0 | Frame 1 (any SYNC) |
| 1 | Frame 20 (EOC) |
| 2 | Any frame while triggers available (FIFO non-empty) |
| 3 | Frame 13 (Slow Data demand) |
| 5 | Frame 15 only when async commands transmitted |
| 6 | Frame 2 with non-zero DET/XTRA data |
| 7 | Frame 2 when DET/XTRA data changed |
| 8 | Everything (unconditional) |
| 9 | Any frame while TRIG_DES_FIFO_RE asserted |
| 10 | Any frame while reading trigger collection FIFO |
| 11 | F2–F11 block when trigger collector has data (`MON_CAPTURE_FLAG`) |
| 12 | Non-null Frame 12 commands |
| 13 | Non-null Frame 14 commands |
| 14 | Non-null Frame 16 commands |
| 15 | Selects what data is captured: 0=CMDout, 1=TRIG_DES_FIFO_DATA |

### Monitor Capture Flag (`MON_PROCX`)
- Set at F1 if `LATCHED_TRIG_DES_FIFO_EMPTY='0'` (triggers in FIFO).
- Cleared at F12.
- Allows bit 11 of `MSM_MON_FIFO_SELECT_REG` to capture the full trigger processing block (F2–F11).

---

## FIFO Pipelining Detail (Trigger Decision FIFO)

The TRIG_DES_FIFO has one clock cycle of latency between RE assertion and valid data. The machine accounts for this explicitly:

1. **F2/W4**: If FIFO non-empty → assert `TRIG_DES_FIFO_RE`. Data appears in F2/W5.
2. **F2/W5**: Continue asserting RE (data for F3/W2 is in transit). First data word clocks in but is not yet output.
3. **F3/W1**: Output first FIFO word (captured in response to RE at F2/W4).
4. Continue streaming through frames 3–7 (internal), 8, 9, 10 for external re-propagation triggers.
5. At F10/W3: drop RE so FIFO gets no RE at W4; last word lands at W4.

This pipelined scheme allows back-to-back trigger frame processing without gaps when multiple trigger decisions are queued.

---

## SYSTEM_VETO_STATE Bit Layout (Frame 2 W3)

```
Bit:  15    14    13    12    11    10     9     8     7     6     5     4     3     2     1     0
     [RM U][RM R][RM L][SW V][VRAM][MON7][GLOB][ NIM][Alg7][Alg6][Alg5][Alg4][Alg3][Alg2][Alg1][Alg0]
```
- RM U/R/L: Remote Master U/R/L in veto
- SW V: Software veto
- VRAM: Veto RAM active
- MON7: Monitor algorithm 7
- GLOB: Global veto
- NIM: NIM input veto
- Alg0–7: Individual trigger algorithm vetoes

When broadcasting to Link L, bit 12 (RM L) is masked out (0) to prevent the satellite from seeing its own veto reflected back.  
Similarly for R (bit 13) and U (bit 14). ✅ verified 2026-04-24 — mstr_mach.vhd:L438-447 (SYSTEM_VETO_STATE bit layout comment + L445 AND X"DFFF" for L; X"BFFF" for R; X"EFFF" for U)

---

## Relationship to Other Modules

- **`top.vhd`**: Instantiates `mstr_mach`; provides `SYS_TIME` from the 48-bit timestamp counter, connects TRIG_DES_FIFO output from `calc_total_sum.vhd` pipeline.
- **`calc_total_sum.vhd`**: Produces the trigger decisions that fill the TRIG_DES_FIFO consumed here.
- **TTCL spec (v2.1)**: Defines the 20-frame cycle, command byte values, and DC balance requirements implemented here.
- **`remote_trig_support.vhd`**: Handles the Link L/R/U receive side; provides `MSTR_MACH_START_FLAG` for satellite mode.
- **`local_trig_coinc.vhd`**: One of the trigger algorithm modules that feeds the TRIG_DES_FIFO.

---

## Notable Design History

| Date | Change |
|------|--------|
| 2012-04-26 | Added synchronous command guarding for Frames 12, 14, 16 (retiming flags) |
| 2012-08-01 | Simplified word/frame counter for practical synchronization with router state machines |
| 2015-10-xx | Added trigger monitor FIFO interface (Frame 2 detector status capture) |
| 2018-02-14 | Frame 2 W1–W2: repurposed from debug to actual detector status data |
| 2018-03-28 | Frame 2 diagnostic capture: F2_DATA_CHANGE and F2_DATA_NONZERO flags |
| 2021-06-16 | Frame 14 W4: added ANY_TRIGGER_VETO + REMOTE_MASTER_VETO broadcast |
| 2022-04-19 | Frame 17 (Auxiliary Detector): removed, replaced with NULL |
| 2025-10-22 | Added TRIGGER_HOLDOFF support (passed to `trig_algo_support` via `local_trig_coinc`) |

---

## See Also

- `knowledgeBase/deep_fpga_MTRG_MAIN.md` — MTRG Main FPGA overview, register map
- `knowledgeBase/ttcl.md` — TTCL specification (20-frame cycle, all command bytes)
- `knowledgeBase/vhdl/MTRG_top.md` — Top-level MTRG wiring
- `knowledgeBase/vhdl/MTRG_calc_total_sum.md` — Trigger decision output that feeds TRIG_DES_FIFO
- `knowledgeBase/vhdl/MTRG_MYRIAD_RCV_MACH.md` — MγRIAD receiver (remote trigger source)
