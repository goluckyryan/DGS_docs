# top.vhd / last_manual_top.vhd — Plain English Summary (MTRG)
_Primary source: `FPGA/MTRG/Firmware/MAIN_FPGA/trunk/Source/last_manual_top.vhd` (trunk; ~5000+ lines)_  
_20180507 tag: `FPGA/Firmware_Tags/MasterTrigger/20180507/Firmware/MAIN_FPGA/trunk/Source/top.vhd` (5024 lines)_  
_Summarized: 2026-04-15; verified 2026-04-24 against both trunk and 20180507 tag_  
Stability: C3 - Structural / stable

> **Note:** This file documents `top.vhd` — a structural wrapper top. There is a separate, larger file `Generated_top.vhd` (6,286 lines, entity `trigger_top`) that is the **actual synthesis top-level** used by ISE. See [`MTRG_Generated_top.md`](MTRG_Generated_top.md) for a complete analysis of that file.

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
- `GLOBAL_X_TOTAL`, `GLOBAL_Y_TOTAL` (**16-bit** `std_logic_vector(15 downto 0)`): total X/Y-plane multiplicity from all Routers ✅ verified 2026-04-24 — `last_manual_top.vhd:L827-828`; `top.vhd (20180507):L782-783`. **Previous KB entry stated 11-bit — that was incorrect.**
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
| TRIG_LOGIC1 | cpld_trig | SIMPLE_TRIGGER (NIM/manual) | 0x50 ✅ verified 2026-04-24 — `last_manual_top.vhd:L2341` | TRIG_MASK_REG[0] |
| TRIG_LOGIC2 | sum_hits_X | GLOBAL_X_TOTAL | 0x51 ✅ `L2389` | TRIG_MASK_REG[1] |
| TRIG_LOGIC3 | sum_hits_X (Y reuse; passes GLOBAL_Y_TOTAL as SUM_OF_X) | GLOBAL_Y_TOTAL | 0x52 ✅ `L2440` | TRIG_MASK_REG[2] |
| TRIG_LOGIC4 | sum_hits_XY | GLOBAL_X_TOTAL + Y | 0x53 ✅ `L2491` | TRIG_MASK_REG[3] |
| TRIG_LOGIC5 | cpld_trig | FAST_STROBE (CPLD) | 0x54 ✅ `L2536` | TRIG_MASK_REG[4] |
| TRIG_LOGIC6A (20180507) | **GITMO_TRIGGER** ✅ verified 2026-04-24 — `top.vhd (20180507):L2540`; **not `mult_overlap`** (prior KB error) | GITMO / Link L | — | TRIG_MASK_REG[5] |
| TRIG_LOGIC6B (20180507) / TRIG_LOGIC6 (trunk) | REMOTE_MASTER_TRIG_SUPPORT; **GITMO removed 20220412** ✅ `last_manual_top.vhd:L2566` | Link L remote master | — | TRIG_MASK_REG[5] |
| TRIG_LOGIC7 | REMOTE_MASTER_TRIG_SUPPORT | Link R remote master | — | TRIG_MASK_REG[6] ✅ `last_manual_top.vhd:L2632` |
| TRIG_LOGIC8A/B | MYRIAD_TRIGGER / REMOTE_MASTER_TRIG_SUPPORT | Link U / MYRIAD | — | TRIG_MASK_REG[7] ✅ `last_manual_top.vhd:L2775,2839` |

- TRIG_LOGIC3 reuses `sum_hits_X` but passes `GLOBAL_Y_TOTAL` as `SUM_OF_X` input and `SUM_OF_Y_THRESH` as threshold ✅ verified 2026-04-24 — `last_manual_top.vhd:L2431-2432` — the VHDL entity is unchanged, the Y trigger is purely a different port mapping.
- **20180507:** TRIG_LOGIC6A/B dual-mode: TRIG_LOGIC6A = `GITMO_TRIGGER` (not `mult_overlap`), TRIG_LOGIC6B = `REMOTE_MASTER_TRIG_SUPPORT`; `TRIG_LOGIC_6_ALGO_SEL` register selects which output drives TRIGGER_FIFO_OUT(6). ✅ verified 2026-04-24 — `top.vhd (20180507):L2484-2595`
- **Trunk (post-20220412):** GITMO_TRIGGER removed; TRIG_LOGIC6 = single `REMOTE_MASTER_TRIG_SUPPORT` only. ✅ `last_manual_top.vhd:L2566` + L3801 comment

### U2 — mstr_mach
Master state machine. Collects trigger FIFOs (TRIGGER_FIFO_OUT, TRIGGER_FIFO_FLAG), builds trigger packets, and distributes them to all 8 Routers via LINKA..LINKH_TX. Also manages system sync (Sync and Imperative Sync frames) and communicates with remote Masters via LINKL/R/U.

## Veto Logic (per-algorithm TRIGGER_VETOES process, priority order)

**Firmware version note:**
- **20180507** `top.vhd`: `ALGO_THROTTLE_REQUEST(i)` is **LAST** in the if-elsif chain. Software veto (bit 11) and remote master veto (bit 10) are **absent** in this version. ✅ verified 2026-04-24 — `top.vhd (20180507):L1492`
- **Trunk** `last_manual_top.vhd` (post-20220412): `ALGO_THROTTLE_REQUEST(i)` moved to **FIRST** (highest priority). All 7 conditions present. ✅ verified 2026-04-24 — `last_manual_top.vhd:L1557-1585`

Trunk priority order (if-elsif chain):
1. `ALGO_THROTTLE_REQUEST(i)` = '1' (per-algo FIFO >50% full) — **unconditional**, cannot be disabled; fatal if ignored
2. NIM_IN2 active AND `TRIG_MASK_REG(14)` AND `TRIG_VETO_SELECT(i)(0)`
3. `GLOBAL_THROTTLE_REQUEST` AND `TRIG_MASK_REG(15)` AND `TRIG_VETO_SELECT(i)(2)`
4. `MON7_VETO_REQUEST` AND `TRIG_MASK_REG(13)` (monitor FIFO #7 overflow; no per-algo enable)
5. `VETO_FROM_VETO_RAM` AND `TRIG_MASK_REG(12)` AND `TRIG_VETO_SELECT(i)(1)` (target wheel veto)
6. `TRIG_MASK_REG(11)` AND `TRIG_VETO_SELECT(i)(1)` (software veto; no separate global enable)
7. `ANY_VETO_FROM_REMOTE_MASTER` AND `TRIG_MASK_REG(10)` AND `TRIG_VETO_SELECT(i)(2)` (added 20210616)

`ANY_TRIGGER_VETO_TO_REMOTE` = OR of TRIGGER_VETOES, excluding algo 6 veto (prevents remote-master veto loop-back).

`SYSTEM_VETO_STATE[15:0]`: packed veto status sent to remote Masters (**trunk only, added 20220412**) ✅ verified 2026-04-24 — `last_manual_top.vhd:L1124-1139`:
- [15] = VETO_FROM_REMOTE_MASTER_U, [14] = _R, [13] = _L
- [12] = TRIG_MASK_REG(11) (software veto enable)
- [11] = VETO_FROM_VETO_RAM, [10] = MON7_VETO_REQUEST, [9] = GLOBAL_THROTTLE_REQUEST, [8] = NON_TDC_NIM_IN2
- [7:0] = TRIGGER_VETOES (per-algo bits)

## CPLD SUMCOPY Interface
Three ODDR DDR instances drive SUMCOPY[3:1] with `TEST_SUM[5:0]` (a 6-bit value from VME register REG_210[5:0]). SUMCOPY[0] = xLOC_ADDR(2). In the MTRG, TEST_SUM is VME-writeable (unlike RTRG where it carries TOTAL_COARSE_GE_SUM). FAST_STROBE from CPLD triggers TRIG_LOGIC5.

## Key Constants / Parameters
- 8 trigger algorithm slots (TRIG_LOGIC1–8)
- COMPARE_MODE_CTL: SUM_OF_X_THRESH[15] selects `>` vs `==` comparison
- Remote master links: L, R, U (three possible remote Master connections)
- NIM_IN1 (bit TRIG_MASK_REG(8)): auxiliary trigger input; NIM_IN2: veto/TDC input

## Connections to Other Modules
- **Instantiates (trunk)**: eight_mt_channel (U10), mstr_mach (U2), sum_hits_X (TRIG_LOGIC2+3), sum_hits_XY (TRIG_LOGIC4), cpld_trig (TRIG_LOGIC1+5), REMOTE_MASTER_TRIG_SUPPORT (TRIG_LOGIC6+7+8B), MYRIAD_TRIGGER (TRIG_LOGIC8A), VME bus interface ✅ verified 2026-04-24 — `last_manual_top.vhd:L2325-2839`
- **Instantiates (20180507)**: same but TRIG_LOGIC6 is split into TRIG_LOGIC6A=GITMO_TRIGGER + TRIG_LOGIC6B=REMOTE_MASTER_TRIG_SUPPORT ✅ verified 2026-04-24 — `top.vhd (20180507):L2540,L2595`
- **Receives from Routers** (LINKA..LINKH RX): X/Y-plane multiplicity sums, throttle requests
- **Distributes to Routers** (LINKA..LINKH TX): trigger decisions, channel vetoes, system sync
- **Remote Master exchanges**: trigger type, veto state, sync via LINKL/R/U
- **CPLD**: SUMCOPY DDR (TEST_SUM or address), FAST_STROBE input
