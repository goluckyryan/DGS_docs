# TOP.VHD — Plain English Summary
_Source: ~/FPGA_svn2git/RTRG_git/MAIN_FPGA/Source/TOP.VHD_
_Summarized: 2026-04-15_
_Note: 3094-line file; summary covers trigger-chain sections only (SerDes init, clock management, VME bus, and diagnostic counter sections omitted)._

## Purpose
Top-level VHDL for the RTRG (Router Trigger) FPGA board. Wires together: the 8-channel data path (ROUTER_DATA_PATH), the master-trigger communications state machine (router_main_mach), throttle logic, downlink veto insertion, DC balance encoding for all 8 digitizer output links, and the CPLD coarse-Ge-sum interface. Entity name: `router_top`.

## Key Trigger-Chain Ports
| Signal | Dir | Description |
|---|---|---|
| `LINKL_TX` | out | 18-bit uplink to Master Trigger (carries X/Y plane sums) |
| `LINKL_RX` | in | 18-bit downlink from Master Trigger (trigger decisions, commands) |
| `LINKL_RCLK` | in | Recovered 50 MHz clock from Master |
| `LINKL_LOCK` | in | LOCK* status of link L SERDES |
| `LINKA..LINKH_TX` | out | 8× downlinks to digitizers (trigger decisions + vetoes) |
| `LINKA..LINKH_RX` | in | 8× uplinks from digitizers (discriminator data) |
| `FAST_STROBE` | in | Fast trigger strobe from external CPLD |
| `SUMCOPY[3:0]` | out | 4-bit DDR bus to CPLD (carries TOTAL_COARSE_GE_SUM + address) |
| `NIM_OUT1`, `NIM_OUT2` | out | NIM outputs (throttle and selected diagnostic signals) |

## Key Logic / Data Flow

### Uplink Path (Digitizers → Master Trigger)
1. **ROUTER_DATA_PATH (U8)**: Receives SERDES data from 8 digitizers; runs 8 `chan_in` instances; builds `LINKL_RAW_DATA[15:0]` ✅ verified 2026-04-16 — `router_data_path.vhd:L230–233`:
   - `[15]` = ANY_THROTTLE_REQUEST ✅ verified 2026-04-16 — `router_data_path.vhd:L230`
   - `[14:8]` = Y-plane sum (7-bit, max 80) ✅ verified 2026-04-16 — `router_data_path.vhd:L231`
   - `[7]` = DATA_VALID (ALL_DIGITIZERS_LOCKED AND ROUTER_LOCKED) ✅ verified 2026-04-16 — `router_data_path.vhd:L232`
   - `[6:0]` = X-plane sum (7-bit, max 80) ✅ verified 2026-04-16 — `router_data_path.vhd:L233`
2. **DC balance (U1)**: `LINKL_RAW_DATA` → `LINKL_BAL_DATA` (18-bit DC-balanced)
3. **TX_PROC**: Drives `xLINKL_TX ← LINKL_BAL_DATA` every clock

### Downlink Path (Master Trigger → Digitizers)
1. **router_main_mach**: Decodes `LINKL_RX`. Extracts Frame 12 data (router-specific commands) → FRAME_12_DATA and FRAME_12_REQ_FLAG. Produces `SANITIZED_CONTROL_DATA` (Master commands with Frame 12 replaced by 0xAAAA — safe to distribute).
2. **F12_DECODE process**: When FRAME_12_REQ_FLAG asserted ✅ verified 2026-04-16 — `TOP.VHD:L1420–1421`:
   - `FRAME_12_DATA(2)` → `MASTER_COUNTER_RESETS` ✅ verified 2026-04-16 — `TOP.VHD:L1420`
   - `FRAME_12_DATA(3)` → `MASTER_FIFO_CLEARS` ✅ verified 2026-04-16 — `TOP.VHD:L1421`
3. **ADD_VETOES_BLK (for i in 1 to 8)**: Per-digitizer veto insertion state machine.
   - Normal words: pass `SANITIZED_CONTROL_DATA` unchanged; accumulate `LATCHED_CHANNEL_VETOES(i) |= CHANNEL_VETOES(i)` ✅ verified 2026-04-16 — `TOP.VHD:L2116`
   - 5th word of frames 1–19 (WORD_INDEX = 4, 9, 14, …, 94, 0-indexed): Replace outgoing word with `"000000" & LATCHED_CHANNEL_VETOES(i)[9:0]`; then reset latch to current live CHANNEL_VETOES(i) ✅ verified 2026-04-16 — `TOP.VHD:L2089–2113` (comment: "words 4,9,14...94 are the 5th words of frames 1–19")
   - (5th word of frame 20 is reserved for machine sync — not overridden) ✅ verified 2026-04-16 — `TOP.VHD:L2090`
4. **dc_balance_mach (U3) × 8**: DC-balances `SANITIZED_CONTROL_DATA_WITH_VETOES(i)` → `SANITIZED_DCBAL_CMD_OUT(i)` → drives `xLINKA..LINKH_TX`

### Coarse Ge Sum → CPLD Fast Multiplicity Trigger
- `TOTAL_COARSE_GE_SUM[5:0]` (6-bit count of coarse Ge hits across all 8 digitizers) from ROUTER_DATA_PATH ✅ verified 2026-04-16 — `TOP.VHD:L842`
- Sent to CPLD via **DDR ODDR primitives** on SUMCOPY[3:1] ✅ verified 2026-04-16 — `TOP.VHD:L870–891`:
  - ODDR instance i (0..2): D1 = `TOTAL_COARSE_GE_SUM(i)`, D2 = `TOTAL_COARSE_GE_SUM(i+3)` ✅ verified 2026-04-16 — `TOP.VHD:L889–890`
  - Clock = switched_master_clock (50 MHz): lower 3 bits on rising edge, upper 3 bits on falling edge
  - Effective serial transfer rate: all 6 bits presented to CPLD in each clock cycle
- `SUMCOPY[0]` = `xLOC_ADDR(2)` (VME address bit for CPLD register decode) ✅ verified 2026-04-16 — `TOP.VHD:L896`
- `FAST_STROBE` in from CPLD → goes to NIM/trigger output logic

### Throttle Handling
- **throttle_limiters (U5)**: Receives `THROTTLE_REQUESTS[8:1]` from digitizers; applies `THROTTLE_LIMIT_TIME_REG` (digitizer must assert throttle continuously for N clocks before it is accepted); outputs `ANY_THROTTLE_REQUEST`, `STRETCHED_ANY_THROTTLE_REQUEST`, `SELECTED_THROTTLE`
- `SELECTED_THROTTLE` → NIM_OUT2 (configurable)
- `STRETCHED_ANY_THROTTLE_REQUEST` → counted for diagnostics (COUNT_THROTTLES process)

### Clock Architecture
- `switched_master_clock` = 50 MHz clock (switchable between local oscillator and LINKL_RCLK via DCM)
- `switched_master_clock_2x` = 100 MHz (from DCM 2× output) — used by chan_in and router_data_path
- Clock source selected by `MISC_CLK_CTL_REG(15)` and `LINKL_LOCK` state

## Key Constants / Parameters
- 8 input channels (LINKA–LINKH from digitizers)
- Frame format: 5-word frames, 20 frames per super-frame (100 words total, WORD_INDEX 0–99)
- Frames 1–19: veto data replaces 5th word; Frame 20 5th word reserved for sync
- SUMCOPY DDR: 6-bit coarse Ge sum in 3 DDR bits (lower 3 on rising edge, upper 3 on falling edge)

## Connections to Other Modules
- **Instantiates**: ROUTER_DATA_PATH (U8), router_main_mach, throttle_limiters (U5), dc_balance_mach (U3 ×8, U3B), dc_balance_mach for uplink (U1), VME bus interface, channel and monitor FIFOs, clock DCMs
- **Receives from Master (via LINKL_RX)**: Trigger decisions, counter/FIFO reset commands (Frame 12), machine sync
- **Sends to Master (via LINKL_TX)**: X/Y plane multiplicity sums, throttle flag, data valid
- **Sends to digitizers (via LINKA..LINKH_TX)**: Master trigger decisions + per-channel veto bitmaps
- **Sends to CPLD (via SUMCOPY)**: 6-bit coarse Ge sum (DDR), VME address bit
- **Receives from CPLD (FAST_STROBE)**: Fast trigger strobe for NIM output selection
