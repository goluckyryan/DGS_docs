# top.vhd — Plain English Summary (MTRG)
_Source: ~/FPGA_svn2git/MTRG_git/MAIN_FPGA/trunk/Source/top.vhd_
_Summarized: 2026-04-15_
_Note: 5068-line file; summary covers trigger-chain sections only._

## Purpose
Top-level VHDL for the MTRG (Master Trigger) FPGA board. Entity: `trigger_top`. Wires together: 8-channel Router data aggregation (`eight_mt_channel`), 8 trigger algorithm slots, a master state machine (`mstr_mach`) that distributes trigger decisions to Routers, per-algorithm veto logic, throttle management, NIM I/O, remote-master interconnects, and the CPLD multiplicity bus.

## Key Trigger-Chain Ports
| Signal | Dir | Description |
|---|---|---|
| `LINKA_RX..LINKH_RX` | in | 18-bit SERDES data from 8 Routers (uplink) |
| `LINKA_TX..LINKH_TX` | out | 18-bit trigger decisions + commands → 8 Routers |
| `LINKL_RX/TX` | in/out | Link to/from remote Master Trigger (sub/sub-master config) |
| `LINKR_RX/TX` | in/out | Link R to/from remote Master Trigger |
| `LINKU_RX/TX` | in/out | Link U to/from remote Master Trigger |
| `FAST_STROBE` | in | Fast strobe from external CPLD |
| `SUMCOPY[3:0]` | out | DDR bus carrying TEST_SUM[5:0] to CPLD |
| `NIM_IN1`, `NIM_IN2` | in | NIM trigger/veto inputs |
| `NIM_OUT1`, `NIM_OUT2` | out | NIM output signals |

## Key Signal Declarations
- `GLOBAL_X_TOTAL`, `GLOBAL_Y_TOTAL` (11-bit): total X/Y-plane multiplicity from all Routers
- `SUM_OF_X_THRESH`, `SUM_OF_Y_THRESH` (16-bit): VME thresholds; bit [15]=COMPARE_MODE_CTL ('0'=`>`, '1'=`==`)
- `TRIGGER_VETOES[8:1]` (8-bit): per-algorithm veto bits
- `ANY_TRIGGER_VETO` (1): OR of all veto bits (for monitoring)
- `TRIG_MASK_REG` (16-bit): per-algorithm enable bits + global veto enables
- `TRIG_PRESCALE_REG[8:1]` (8×16-bit): per-algorithm prescale values
- `ALGO_THROTTLE_REQUEST[8:1]`: per-algorithm FIFO-full flags
- `GLOBAL_THROTTLE_REQUEST`: any Router wants throttle (from eight_mt_channel)

## Key Logic: Trigger Algorithm Slots

### U10 — eight_mt_channel
Receives SERDES data from 8 Routers (links A–H). Produces `GLOBAL_X_TOTAL` and `GLOBAL_Y_TOTAL` (each 11-bit) for threshold comparison, `GLOBAL_THROTTLE_REQUEST`, and per-Router throttle map.

### Trigger Logic Instances (TRIG_LOGIC1–8)

| Instance | Module | Input | Type Code | Enable |
|---|---|---|---|---|
| TRIG_LOGIC1 | cpld_trig | SIMPLE_TRIGGER (NIM/manual) | 0x50 | TRIG_MASK_REG[0] |
| TRIG_LOGIC2 | sum_hits_X | GLOBAL_X_TOTAL | 0x51 | TRIG_MASK_REG[1] |
| TRIG_LOGIC3 | sum_hits_X (Y reuse) | GLOBAL_Y_TOTAL | — | TRIG_MASK_REG[2] |
| TRIG_LOGIC4 | sum_hits_XY | GLOBAL_X_TOTAL + Y | — | TRIG_MASK_REG[3] |
| TRIG_LOGIC5 | cpld_trig | FAST_STROBE (CPLD) | — | TRIG_MASK_REG[4] |
| TRIG_LOGIC6A/B | mult_overlap / REMOTE_MASTER_TRIG_SUPPORT | Link L remote master | — | TRIG_MASK_REG[5] |
| TRIG_LOGIC7 | REMOTE_MASTER_TRIG_SUPPORT | Link R remote master | — | TRIG_MASK_REG[6] |
| TRIG_LOGIC8A/B | MYRIAD_TRIGGER / REMOTE_MASTER_TRIG_SUPPORT | Link U / MYRIAD | — | TRIG_MASK_REG[7] |

- TRIG_LOGIC3 reuses `sum_hits_X` but passes `GLOBAL_Y_TOTAL` as `SUM_OF_X` input and `SUM_OF_Y_THRESH` as threshold — the VHDL entity is unchanged, the Y trigger is purely a different port mapping.
- TRIG_LOGIC6A/B are dual-mode: `mult_overlap` (local+remote coincidence) vs remote-only; `TRIG_LOGIC_6_ALGO_SEL` register selects which algorithm's output drives TRIGGER_FIFO_OUT(6).

### U2 — mstr_mach
Master state machine. Collects trigger FIFOs (TRIGGER_FIFO_OUT, TRIGGER_FIFO_FLAG), builds trigger packets, and distributes them to all 8 Routers via LINKA..LINKH_TX. Also manages system sync (Sync and Imperative Sync frames) and communicates with remote Masters via LINKL/R/U.

## Veto Logic (per-algorithm TRIGGER_VETOES process, priority order)
Each of TRIGGER_VETOES[8:1] is set if any of the following applies:
1. `ALGO_THROTTLE` = '1' (any algorithm FIFO >50% full) — **unconditional**, cannot be disabled
2. NIM_IN2 active AND `TRIG_MASK_REG(14)` AND `TRIG_VETO_SELECT(i)(0)`
3. `GLOBAL_THROTTLE_REQUEST` AND `TRIG_MASK_REG(15)` AND `TRIG_VETO_SELECT(i)(2)`
4. `MON7_VETO_REQUEST` AND `TRIG_MASK_REG(13)` (monitor FIFO #7 overflow)
5. `VETO_FROM_VETO_RAM` AND `TRIG_MASK_REG(12)` AND `TRIG_VETO_SELECT(i)(1)` (target wheel veto)
6. `TRIG_MASK_REG(11)` AND `TRIG_VETO_SELECT(i)(1)` (software veto bit)
7. `ANY_VETO_FROM_REMOTE_MASTER` AND `TRIG_MASK_REG(10)` AND `TRIG_VETO_SELECT(i)(2)`

`ANY_TRIGGER_VETO_TO_REMOTE` = same but excludes algorithm 6 veto (prevents remote-master veto loop-back).

`SYSTEM_VETO_STATE[15:0]`: packed veto status register sent to remote Masters:
- [15] = VETO_FROM_REMOTE_MASTER_U, [14] = _R, [13] = _L
- [12] = TRIG_MASK_REG(11) (software veto enable)
- [11] = VETO_FROM_VETO_RAM, [7:0] = TRIGGER_VETOES

## CPLD SUMCOPY Interface
Three ODDR DDR instances drive SUMCOPY[3:1] with `TEST_SUM[5:0]` (a 6-bit value from VME register REG_210[5:0]). SUMCOPY[0] = xLOC_ADDR(2). In the MTRG, TEST_SUM is VME-writeable (unlike RTRG where it carries TOTAL_COARSE_GE_SUM). FAST_STROBE from CPLD triggers TRIG_LOGIC5.

## Key Constants / Parameters
- 8 trigger algorithm slots (TRIG_LOGIC1–8)
- COMPARE_MODE_CTL: SUM_OF_X_THRESH[15] selects `>` vs `==` comparison
- Remote master links: L, R, U (three possible remote Master connections)
- NIM_IN1 (bit TRIG_MASK_REG(8)): auxiliary trigger input; NIM_IN2: veto/TDC input

## Connections to Other Modules
- **Instantiates**: eight_mt_channel (U10), mstr_mach (U2), sum_hits_X (TRIG_LOGIC2+3), sum_hits_XY (TRIG_LOGIC4), cpld_trig (TRIG_LOGIC1+5), mult_overlap (TRIG_LOGIC6A), REMOTE_MASTER_TRIG_SUPPORT (TRIG_LOGIC6B+7+8B), MYRIAD_TRIGGER (TRIG_LOGIC8A), VME bus interface
- **Receives from Routers** (LINKA..LINKH RX): X/Y-plane multiplicity sums, throttle requests
- **Distributes to Routers** (LINKA..LINKH TX): trigger decisions, channel vetoes, system sync
- **Remote Master exchanges**: trigger type, veto state, sync via LINKL/R/U
- **CPLD**: SUMCOPY DDR (TEST_SUM or address), FAST_STROBE input
