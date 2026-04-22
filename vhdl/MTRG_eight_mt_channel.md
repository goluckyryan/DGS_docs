# eight_mt_channel.vhd — Plain English Summary
_Source: ~/FPGA_svn2git/MTRG_git/MAIN_FPGA/trunk/Source/eight_mt_channel.vhd_
_Summarized: 2026-04-15 | Last verified: 2026-04-22_

## Purpose
Aggregates eight `mt_input_channel` instances (one per Router, links A–H) and one `calc_total_sum` instance into a single block, keeping top.vhd clean. Produces the detector-wide X-plane and Y-plane totals (`GLOBAL_X_TOTAL`, `GLOBAL_Y_TOTAL`) and the global throttle request. This is the direct feeder into the trigger algorithm layer.

## Ports
| Signal | Dir | Width | Description |
|---|---|---|---|
| `xLINKA_RX..xLINKH_RX` | in | 8×18 | DC-balanced SERDES data from Routers A–H |
| `xLINKA_RCLK..xLINKH_RCLK` | in | 8 | Per-link SERDES receive clocks |
| `xLINKA_LOCK..xLINKH_LOCK` | in | 8 | Per-link SERDES LOCK* status |
| `INPUT_LINK_MASK_REG` | in | 16 | [7:0] = per-channel mask bits |
| `GLOBAL_X_TOTAL` | out | 16 | Total X-plane multiplicity across all 8 Routers ✅ verified 2026-04-22 — eight_mt_channel.vhd:L57 (15 downto 0) |
| `GLOBAL_Y_TOTAL` | out | 16 | Total Y-plane multiplicity across all 8 Routers ✅ verified 2026-04-22 — eight_mt_channel.vhd:L58 (15 downto 0) |
| `GLOBAL_THROTTLE_REQUEST` | out | 1 | '1' if any Router is requesting throttle |
| `ROUTER_THROTTLE_REQUESTS` | out | 8 | Per-Router throttle request bits |
| `CHANNEL_STATUS` | out | 16 | [15:8]=throttle requests, [7:0]=LOAD_ERR flags |
| `RAW_DATA_MONs` | out | 8×16 | Raw recovered data per channel (for monitor FIFOs) |

## Key Logic / State Machine

### CHANNEL_BLOCK (for i in 1 to 8)
Instantiates one `mt_input_channel` per Router link. The component interface used here is a **DSSD-specific variant** that includes `RTR_SUM_OF_X` and `RTR_SUM_OF_Y` (8-bit each) outputs, which are not present in the base `mt_input_channel.vhd` at `MAIN_FPGA/` — the trunk version appears to have these additional X/Y sum extraction ports added for the DSSD/DGS trigger chain.

Port connections per channel:
- `ROUTER_DATA_IN ← LINK_RXs(i)` (18-bit SERDES)
- `LVDS_RCLK ← LINK_RCLKs(i)`, `SERDES_LOCK ← LINK_LOCKs(i)`
- `CHANNEL_MASK ← INPUT_LINK_MASK_REG(i-1)`
- `THROTTLE_REQUEST → xROUTER_THROTTLE_REQUESTS(i)`
- `LOAD_ERR → CHANNEL_STATUS(i-1)` (bits [7:0])
- `RTR_SUM_OF_X → RTR_SUM_OF_X(i)` (8-bit X-plane sum from this Router)
- `RTR_SUM_OF_Y → RTR_SUM_OF_Y(i)` (8-bit Y-plane sum from this Router)
- `RAW_DATA → RAW_DATA_MONs(i)`

### THROTTLE_PROC
Registered OR of all 8 throttle request bits → `GLOBAL_THROTTLE_REQUEST`:  
If `xROUTER_THROTTLE_REQUESTS /= "00000000"` → '1', else '0'.

### CALC_TOTALS — calc_total_sum instance
- Receives: `RTR_SUM_OF_X(1..8)` and `RTR_SUM_OF_Y(1..8)`
- Produces: `GLOBAL_X_TOTAL` (16-bit), `GLOBAL_Y_TOTAL` (16-bit) via 3-stage pipelined adder tree ✅ verified 2026-04-22 — calc_total_sum.vhd:L22-23,L68-123 (SUMPROC1/2/3, X_TOTAL 15 downto 0)

### CHANNEL_STATUS assembly
- `[15:8]` ← `xROUTER_THROTTLE_REQUESTS(8:1)` (throttle requests per Router)
- `[7:0]` ← LOAD_ERR bits from each mt_input_channel, assigned via `CHANNEL_STATUS(i-1)`

### Link bus aggregation (async)
The 8 individual xLINKx_RX/RCLK/LOCK ports are aggregated into JTA_8X18 and slv(8:1) arrays (LINK_RXs, LINK_RCLKs, LINK_LOCKs) to enable the for-generate loop. This is a structural convenience only.

## Key Constants / Parameters
- 8 Router channels (links A–H, for i in 1 to 8)
- RTR_SUM_OF_X/Y: 8-bit per Router (std_logic_vector(7 downto 0)) ✅ verified 2026-04-22 — eight_mt_channel.vhd:L81-82
- GLOBAL totals: 16-bit (std_logic_vector(15 downto 0)) ✅ verified 2026-04-22 — calc_total_sum.vhd:L22-23; eight_mt_channel.vhd:L57-58 (⚠️ note: previous doc stated 11-bit — corrected)

## Connections to Other Modules
- **Instantiates**: mt_input_channel × 8 (DSSD/trunk variant), calc_total_sum × 1
- **Receives**: SERDES data from 8 Routers (links A–H)
- **Sends to trigger algorithms** (via top.vhd): GLOBAL_X_TOTAL, GLOBAL_Y_TOTAL → `sum_hits_x` and Y-plane equivalent; GLOBAL_THROTTLE_REQUEST
- **Sends to top.vhd**: CHANNEL_STATUS, RAW_DATA_MONs (for monitor FIFOs), ROUTER_THROTTLE_REQUESTS
- **Data flow**: SERDES → mt_input_channel → RTR_SUM_OF_X/Y → calc_total_sum → GLOBAL_X/Y_TOTAL → sum_hits_x
