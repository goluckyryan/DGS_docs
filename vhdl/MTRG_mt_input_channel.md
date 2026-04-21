# mt_input_channel.vhd — Plain English Summary
_Source: ~/FPGA_svn2git/MTRG_git/MAIN_FPGA/mt_input_channel.vhd_
_Summarized: 2026-04-15_

## Purpose
Encapsulates one complete Master Trigger input block for one Router. Bundles DC-balance recovery (DCBAL_IN) and Router data stream parsing (mt_pipeline) into a single entity, applies channel masking, and exposes per-digitizer data plus aggregate status maps. Eight of these are instantiated (one per Router connection) in `eight_mt_channel.vhd`.

## Ports
Key ports:

| Signal | Dir | Width | Description |
|---|---|---|---|
| `ROUTER_DATA_IN` | in | 18 | DC-balanced SERDES data from Router |
| `LVDS_RCLK` | in | 1 | SERDES receive clock from Router |
| `SERDES_LOCK` | in | 1 | Active-low LOCK* from SERDES chip |
| `CHANNEL_MASK` | in | 1 | If '1', zero all outputs (channel disabled at MTRG level) |
| `DCBAL_BYPASS` | in | 1 | If '1', skip DC-balance decoding (pass raw data) |
| `MULTIPLICITY` | out | 4 | 4-bit hit multiplicity from Router data word [15:12] |
| `RAW_DATA` | out | 16 | Recovered 16-bit Router data (for channel monitor FIFO) |
| `LOAD_ERR` | out | 1 | '1' if not locked to Router |
| `TRAILER_FLAG` | out | 1 | One-tick pulse at end of each Router data cycle (when locked) |
| `THROTTLE_REQUEST` | out | 1 | OR of all digitizer throttle requests (zeroed if masked) |
| `CHANNEL_ID` | out | JTA_8X8_Array | Digitizer IDs from Router (8 channels × 8-bit) |
| `SPARE_A`, `SPARE_B` | out | JTA_8X8_Array | Words 13 and 14 from each digitizer (undefined content) |
| `CHAN_MASK_MAP` | out | 8 | Per-digitizer mask flags from Router |
| `ROUTER_LOCK_MAP` | out | 8 | Per-digitizer sync lock bits from Router |
| `DIG_CODE_ERR_MAP` | out | 8 | Per-digitizer grey code error flags |
| `DIG_ERROR_MAP` | out | 8 | Per-digitizer error bits |
| `DIG_PILEUP_MAP` | out | 8 | Per-digitizer pileup flags |
| `THROTTLE_REQ_MAP` | out | 8 | Per-digitizer throttle request bits |
| `ROUTER_SPARE_MAP` | out | 8 | Spare status bits per Router channel |

Ports related to time-windowed energy sum (`LATCHED_TIMESTAMP`, `SHIFT_FACTOR_REG_BITS`, `WINDOW_SIZE`) are present in the port list but the corresponding logic is commented out in this version.

## Key Logic / State Machine

### U1 — DCBAL_IN
- Input: 18-bit `ROUTER_DATA_IN` (DC-balanced SERDES word)
- Recovers the original 16-bit data → `UNBAL_ROUTER_DATA`
- If `DCBAL_BYPASS='1'`, simply passes data through
- Handles RCLK→MCLK domain crossing via internal FIFO

### U2 — mt_pipeline
- Input: `UNBAL_ROUTER_DATA` (16-bit recovered data stream)
- Parses the Router serial frame (one frame per 2 µs system cycle, 8 digitizers × ~15 words each)
- Outputs update every ~2 µs, staggered ~225 ns apart per digitizer
- Extracts: CHANNEL_ID (8×8-bit), SPARE_A (8×8-bit), SPARE_B (8×8-bit), DIGVALID (8-bit)
- Extracts status maps: CHAN_MASK_MAP, ROUTER_LOCK_MAP, DIG_CODE_ERR_MAP, DIG_ERROR_MAP, DIG_PILEUP_MAP, THROTTLE_REQ_MAP, ROUTER_SPARE_MAP
- Produces: THROTTLE_REQUEST, LOAD_ERR, TRAILER_FLAG
- **Note**: HIT_PATTERN, CC_ENERGY, HIT_TIMESTAMP, GROUP_ENERGY_SUM, and PATTERN_MATCH_MAP ports exist in the component but are commented out — not used in this implementation

### Masking logic (maskblock, for i in 1 to 8)
- If `CHANNEL_MASK='1'`: CHANNEL_ID(i) ← 0x00, SPARE_A/B(i) ← 0x00
- If `DIGVALID(i)='0'` (invalid): CHANNEL_ID(i) ← 0xFF, SPARE_A/B(i) ← 0x00
- Else: pass through unmasked values

### MULTIPLICITY extraction
`MULTIPLICITY <= UNBAL_ROUTER_DATA(15 downto 12) when CHANNEL_MASK='0' else "0000"`  
This extracts the top 4 bits of the recovered 16-bit Router word. Based on the RTRG link-L word format, bits [15:12] = throttle bit + Y-plane sum bits [14:12], suggesting the MULTIPLICITY extraction here reflects the older 4-bit Router data format that may pre-date the current 7+7 X/Y sum format.

`THROTTLE_REQUEST <= '0' when CHANNEL_MASK='1' else INTERNAL_THROTTLE_REQUEST`

## Key Constants / Parameters
- System data cycle: ~2 µs (driven by Router framing)
- Per-digitizer data extracted once per 2 µs cycle, staggered ~225 ns per digitizer
- 8 digitizers per Router ✅ verified 2026-04-21 — `eight_mt_channel.vhd:L147` (`FOR i in 1 to 8 generate`) instantiates 8 × `mt_input_channel`; `ROUTER_THROTTLE_REQUESTS` is `std_logic_vector(8 downto 1)`

## Connections to Other Modules
- **Instantiated by**: `eight_mt_channel.vhd` (8× for 8 Router connections)
- **Receives**: SERDES data from one Router (via DC balance recovery)
- **Sends upward**: MULTIPLICITY (→ eight_mt_channel → calc_total_sum), THROTTLE_REQUEST, status maps, CHANNEL_ID
- **Instantiates**: DCBAL_IN (U1), mt_pipeline (U2)
