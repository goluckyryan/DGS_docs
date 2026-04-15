# calc_total_sum.vhd — Plain English Summary
_Source: ~/FPGA_svn2git/MTRG_git/MAIN_FPGA/trunk/Source/calc_total_sum.vhd_
_Summarized: 2026-04-15_

## Purpose
Sums X-plane and Y-plane multiplicity counts from up to 8 Routers into single detector-wide totals. Uses a 3-stage pipelined adder tree (3-clock latency) to produce X_TOTAL and Y_TOTAL, which feed the `sum_hits_x` trigger algorithm (and presumably a Y-plane equivalent). This is the aggregation stage between per-Router link decoding and the threshold comparison in the trigger algorithm.

## Ports
| Signal | Dir | Width | Description |
|---|---|---|---|
| `CLK` | in | 1 | 50 MHz board clock |
| `RST` | in | 1 | Active-high reset |
| `RTR_SUM_OF_X` | in | JTA_8X8_Array | X-plane sum from each of 8 Routers (8 × 8-bit) |
| `RTR_SUM_OF_Y` | in | JTA_8X8_Array | Y-plane sum from each of 8 Routers (8 × 8-bit) |
| `X_TOTAL` | out | 11 | Total X-plane multiplicity across all Routers |
| `Y_TOTAL` | out | 11 | Total Y-plane multiplicity across all Routers |

## Key Logic / State Machine

### 3-stage pipelined adder tree (separate processes per stage)

**SUMPROC1** (Stage 1, 1 clock):  
Pairs of Router sums → 4× 11-bit subtotals:
- `XSUBTOTAL1 = RTR_SUM_OF_X(1) + RTR_SUM_OF_X(2)` (8+8 → 11-bit)
- `XSUBTOTAL2 = RTR_SUM_OF_X(3) + RTR_SUM_OF_X(4)`
- `XSUBTOTAL3 = RTR_SUM_OF_X(5) + RTR_SUM_OF_X(6)`
- `XSUBTOTAL4 = RTR_SUM_OF_X(7) + RTR_SUM_OF_X(8)`
- Same for Y → YSUBTOTAL1..4 (11-bit)

**SUMPROC2** (Stage 2, 1 clock):  
- `XSUBTOTAL5 = XSUBTOTAL1 + XSUBTOTAL2` (11-bit result)
- `XSUBTOTAL6 = XSUBTOTAL3 + XSUBTOTAL4` (11-bit result)
- `YSUBTOTAL5 = YSUBTOTAL1 + YSUBTOTAL2`
- `YSUBTOTAL6 = YSUBTOTAL3 + YSUBTOTAL4`
- **Code note**: YSUBTOTAL5/6 are declared as `std_logic_vector(1 downto 0)` (2-bit) while XSUBTOTAL5/6 are correctly 11-bit. This appears to be a declaration bug — the Y path intermediate results are truncated to 2 bits at stage 2, making Y_TOTAL unreliable. XSUBTOTAL5/6 are correctly 11-bit.

**SUMPROC3** (Stage 3, 1 clock):  
- `X_TOTAL = XSUBTOTAL5 + XSUBTOTAL6` (11-bit)
- `Y_TOTAL = YSUBTOTAL5 + YSUBTOTAL6` (11-bit, but input truncated by bug above)

**Total pipeline latency**: 3 clocks from Router input to X_TOTAL/Y_TOTAL output.

## Key Constants / Parameters
- Input range per Router: 0–255 (8-bit); physical maximum is 80 strips/Router (8 channels × 10 disc bits)
- Output range: 0–640 across 8 Routers (10-bit needed, 11-bit allocated for X)
- Adder tree is fully pipelined (separate process per stage) to meet timing at 50 MHz

## Connections to Other Modules
- **Receives from**: eight_mt_channel.vhd (which extracts per-Router X/Y sums from incoming SerDes data) via RTR_SUM_OF_X, RTR_SUM_OF_Y
- **Sends to**: sum_hits_x.vhd (SUM_OF_X ← X_TOTAL) and presumably a Y-plane equivalent trigger algorithm (Y_TOTAL)
- **Instantiates**: nothing (pure combinational + registered adder tree)
