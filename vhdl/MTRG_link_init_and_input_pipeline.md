# MTRG Link Init & Input Pipeline — VHDL Reference

Stability: C3 - Structural / stable

_Source: `FPGA/Firmware_Tags/MasterTrigger/20180507/` — code-read 2026-04-24_

These two modules handle the low-level SERDES link bring-up (link_init) and per-Router data unpacking (mstr_trigger_input_pipeline / mt_pipeline) on the Master Trigger board.

---

## Table of Contents

1. [link_init.vhd — SERDES Link Initializer](#link_initvhd--serdes-link-initializer)
2. [mstr_trigger_input_pipeline.vhd (mt_pipeline) — Per-Router Data Unpacker](#mstr_trigger_input_pipelinevhd-mt_pipeline--per-router-data-unpacker)

---

## link_init.vhd — SERDES Link Initializer

**File:** `MasterTrigger/20180507/Firmware/MAIN_FPGA/trunk/Source/link_init.vhd`  
**Author:** John T. Anderson (ANL)  
**Lines:** 238  
**Purpose:** Manages SERDES DEN/REN/SYNC signals and monitors LOCK* for the 8 main data-input / command-output links on the Master Trigger. This is MTRG-only; a Router version would additionally handle the upward link.

### Overview

On reset release, this FSM steps through link initialization in sequence:

1. Disable all SERDES transmitters and receivers (INIT)
2. Enable all 8 SERDES chips (EN_SERDES)
3. Assert SYNC to all 8 links, forcing sync-pattern transmission (SYNC)
4. Wait until all unmasked links assert LOCK* (WAIT_FOR_LOCK)
5. Assert ALL_LOCKED flag; wait for software acknowledgment via LOCK_ACK (ALL_LOCKED_UP)
6. Release SYNC (allowing real data flow) and monitor for lock loss (ACKED)
7. On lock loss: assert LOCK_ERROR, hold state (ERROR) until RETRY forces reset

### FSM States

| State | STATE_MON | SYNC_OUT | DEN/REN | ALL_LOCKED | LOCK_ERROR | Next |
|-------|-----------|----------|---------|-----------|-----------|------|
| INIT | 0x0 | 0x00 | 0x00 | 0 | 0 | EN_SERDES |
| EN_SERDES | 0x1 | 0x00 | 0xFF | 0 | 0 | SYNC (or hold if RETRY='1') |
| SYNC | 0x2 | 0xFF | 0xFF | 0 | 0 | WAIT_FOR_LOCK |
| WAIT_FOR_LOCK | 0x3 | 0xFF | 0xFF | 0 | 0 | ALL_LOCKED_UP when LOCK_STATE=0x00 |
| ALL_LOCKED_UP | 0x4 | 0xFF | 0xFF | 1 | 0 | ACKED (if LOCK_ACK='1' and all locked), ERROR (if lock lost) |
| ACKED | 0x5 | 0x00 | 0xFF | 1 | 0 | ERROR if any lock lost; else stay |
| ERROR | 0x6 | 0x00 | 0xFF | 0 | 1 | INIT if RETRY='1'; else stay |

**Key design points:**

- `LINK_MASK[7:0]`: VME-controlled mask register. If bit N = '1', link N is treated as always LOCK*ed — enabling operation with dead links. ✅ verified 2026-04-25 - link_init.vhd:L110 (LOCK_STATE(i) <= '0' when LINK_MASK='1' else LINK_LOCKED(i)). ⚠️ Note: file header comment (L43–44) claims masked transmitters are also "held in SYNC all the time," but the code does NOT implement per-bit SYNC masking — SYNC_OUT is set uniformly for all 8 links in each FSM state. The per-bit SYNC claim may be aspirational/stale comment.
- `LOCK_STATE[i]` = `'0'` when masked OR when `LINK_LOCKED[i]='0'` (i.e., LOCK* is active-low from SERDES). ALL_LOCKED is true when `LOCK_STATE = 0x00` (all bits zero). ✅ verified 2026-04-25 - link_init.vhd:L109–110,L190 (LOCK_STATE generate block; WAIT_FOR_LOCK → ALL_LOCKED_UP transition on "00000000").
- `RETRY` input: in EN_SERDES, holds the machine from progressing to SYNC (software-controlled retry gate). In ERROR, resets the whole sequence back to INIT. ✅ verified 2026-04-25 - link_init.vhd:L164 (EN_SERDES: RETRY='1'→EN_SERDES, else→SYNC); L233 (ERROR: RETRY='1'→INIT).
- `LOCK_ACK`: software-asserted to acknowledge lock achieved; transitions ALL_LOCKED_UP → ACKED and removes SYNC_OUT, allowing normal data to flow. ✅ verified 2026-04-25 - link_init.vhd:L205 (LOCK_ACK='1' AND LOCK_STATE=0→ACKED); L217 (ACKED: xSYNC_OUT<=0x00).
- `LOCK_ERROR`: latches high and stays high in ERROR until a RETRY/reset. Designed for intermittent link monitoring — software must check and reset to catch transient drops. ✅ verified 2026-04-25 - link_init.vhd:L232 (xLOCK_ERROR<='1' in ERROR state only).
- The 11 SERDES chips on the MTRG board: 8 data-input links (managed here) + 2 unused sideward links + 1 upward link (not applicable at MTRG, which is top of hierarchy).

### Port Summary

| Port | Dir | Width | Description |
|------|-----|-------|-------------|
| CLK | in | 1 | Board master clock |
| RST | in | 1 | Power-on reset (synchronous clear) |
| RETRY | in | 1 | SW-controlled: hold EN_SERDES or reset from ERROR |
| LOCK_ACK | in | 1 | SW acknowledgment of all-locked; releases SYNC |
| LINK_LOCKED | in | 8 | LOCK* from each SERDES (0=locked, 1=unlocked) |
| LINK_MASK | in | 8 | VME mask: treat masked links as always locked |
| SYNC_OUT | out | 8 | SYNC control to SERDES transmitters |
| DEN_OUT | out | 8 | Driver (TX) enable |
| REN_OUT | out | 8 | Receiver enable |
| LOCK_ERROR | out | 1 | Lock was once acquired but later lost |
| ALL_LOCKED | out | 1 | All unmasked links are currently locked |
| STATE_MON | out | 4 | State debug hook | ✅ verified 2026-04-25 - link_init.vhd:L78 (STD_LOGIC_VECTOR(3 downto 0))

---

## mstr_trigger_input_pipeline.vhd (mt_pipeline) — Per-Router Data Unpacker

**File:** `MasterTrigger/20180507/Gretina Trigger/VHDL/MasterTrigger/MAIN_FPGA/20120424/mstr_trigger_input_pipeline.vhd`  
**Author:** John T. Anderson (ANL); revised 2009-04-24 (throttle bits), 2009-09-01 (port arrays)  
**Lines:** 828  
**Purpose:** Receives a packed SERDES stream from one Router board (containing data from 8 digitizers) and unpacks it into wide parallel registers refreshed once per 2 µs system cycle. One instance per Router input (up to 8 Routers → 8 instances on MTRG).

### System Cycle Format (per Router → MTRG)

Each 2 µs cycle carries: **3-word preamble + 8 × 12-word digitizer blocks + 1 end-of-event trailer**

| Segment | Lower 12 bits | Purpose |
|---------|---------------|---------|
| Preamble word 1 | 0xFFF | Sync pattern |
| Preamble word 2 | 0x000 | Sync pattern |
| Preamble word 3 | 0x0F0 | Sync pattern |
| Digitizer block (×8) | see bit layout below | Per-digitizer data |
| End of event | 0xE0E | Trailer |

Upper 4 bits of every word carry **fast timing** (bits [15:12]) — not extracted in this module.

### Per-Digitizer Block Structure (12 words × 12 bits = 132-bit TEMP_REG)

Words are loaded into `TEMP_REG[131:0]` serially; WORD_COUNT counts down from 11 (0x0B) to 0. ✅ verified 2026-04-24 - mstr_trigger_input_pipeline.vhd:L253 (WORD_COUNT <= "01011" = 11 decimal)

| TEMP_REG bits | Word (WORD_COUNT) | Contents |
|--------------|-------------------|---------|
| [131:124] | 11 (0x01B) | Crystal/Channel ID [7:0] |
| [123:120] | 11 | Fixed pattern "1111" (validity check) |
| [119:112] | 10 | Buffer count [7:0] |
| [111:108] | 10 | Fixed pattern "0000" (validity check) |
| [107:100] | 9 | Status value [7:0] |
| [99] | 9 | CHAN_MASK (Router channel masked if '1') |
| [98] | 9 | ANY_THROTTLE_REQUEST (any digitizer asking for throttle) |
| [97] | 9 | Spare Router status bit |
| [96] | 9 | CHANNEL_THROTTLE_REQUEST (this digitizer asking for throttle) |
| [95:56] | 8–5 | Hit Pattern [39:0] (5 words × 8 bits, plus 4-bit boundaries) |
| [55:48] | 5 (upper half) | Timestamp [15:8] |
| [47:40] | 5 (lower half) | Timestamp [7:0] |
| [39:36] | 4 | CC Energy [3:0] |
| [35:24] | 4–3 | CC Energy [15:4] |
| [23:16] | 2 | Spare Word A (13th word from digitizer) |
| [15] | 2 | ROUTER_LOCK (frame sync) |
| [14] | 2 | GREY_CODE_ERR |
| [13] | 2 | DIGITIZER_ERROR |
| [12] | 2 | DIGITIZER_PILEUP |
| [11:4] | 1 | Spare Word B (14th word from digitizer) |
| [3:1] | 1 | CHAN_CHECK (ordinal 0-7 from Router packer) |
| [0] | 1 | PATTERN_MATCH (Router pattern register match) |

**Note on byte swaps:**
- Timestamp: `CC_ENERGY(n) <= TEMP_REG(35 downto 24) & TEMP_REG(39 downto 36)` — upper 12 bits then lower 4 bits
- Energy: `HIT_TIMESTAMP(n) <= TEMP_REG(47 downto 40) & TEMP_REG(55 downto 48)` — byte swap in TEMP_REG order

**Preamble/trailer patterns:** ✅ verified 2026-04-24 - mstr_trigger_input_pipeline.vhd:L227/242/255/373 — 0xFFF/0x000/0x0F0/0xE0E confirmed
**TRAILER WORD_COUNT:** ✅ verified 2026-04-24 - mstr_trigger_input_pipeline.vhd:L368 (WORD_COUNT <= "11000" = 24 = 0x18)
**TEMP_REG width:** ✅ verified 2026-04-24 - mstr_trigger_input_pipeline.vhd:L179 (signal TEMP_REG : std_logic_vector(131 downto 0))

### Data Validity Check (per digitizer)

A digitizer output is only valid (`DIGVALID(n) = '1'`) when ALL of:
1. `TEMP_REG[123:120] = "1111"` — crystal ID word fixed pattern OK
2. `TEMP_REG[111:108] = "0000"` — buffer count word fixed pattern OK
3. `TEMP_REG[99] = '0'` — Router channel not masked
4. `TEMP_REG[15] = '1'` — Router machine locked to digitizer
5. `TEMP_REG[14] = '0'` — no grey code error
6. `TEMP_REG[13] = '0'` — digitizer not asserting error bit
7. `TEMP_REG[3:1] = "00N"` — ordinal channel count matches expected position (0-7)

If invalid: `CHANNEL_ID(n) <= 0xFF`, `CC_ENERGY/HIT_PATTERN/TIMESTAMP <= 0`, DIGVALID(n)='0', not added to energy sum.

### FSM (PARSER process)

States: IDLE → PREAMBLE1 → PREAMBLE2 → PREAMBLE3 → DIG1 → DIG2 → ... → DIG8 → TRAILER → PREAMBLE1

| State | Pattern expected | On mismatch |
|-------|-----------------|-------------|
| PREAMBLE1 | data[11:0] = 0xFFF | stay in PREAMBLE1, LOAD_ERR='1' |
| PREAMBLE2 | data[11:0] = 0x000 | back to PREAMBLE1, LOAD_ERR='1' |
| PREAMBLE3 | data[11:0] = 0x0F0 | back to PREAMBLE1, LOAD_ERR='1' |
| DIG1-8 | counts 11→0 (12 words each) | — |
| TRAILER | data[11:0] = 0xE0E | LOAD_ERR='1' but still proceeds |

WORD_COUNT is initially set to 0x10 (16) in IDLE/PREAMBLE states (decoded as a do-nothing in OUTPROC). Set to 0x0B (11) at start of each digitizer block, counting down to 0. Set to 0x18 (24) in TRAILER state (marks end-of-event, outputs GROUP_ENERGY_SUM).

### Throttle Aggregation

- `SAMP_ANY_THROTTLE_REQUEST`: Set/cleared from TEMP_REG[98] at DIG1 capture; OR'd with TEMP_REG[98] at DIG2–DIG8. Only output on TRAILER clock.
- `SAMP_THROTTLE_REQ_MAP[8:1]`: Individual per-digitizer throttle requests (TEMP_REG[96]) sampled during each DIG state. Output on TRAILER.
- `THROTTLE_REQUEST` (aggregate) and `THROTTLE_REQ_MAP` are updated once per 2 µs cycle on the TRAILER clock.

### Group Energy Sum

`GROUP_ENERGY_SUM[23:0]`: Running 24-bit accumulation of all valid, unmasked digitizer CC energies in this Router group:
- Cleared at DIG1 capture if DIG1 invalid; set to DIG1 CC_ENERGY if valid
- Accumulated (`+= CC_ENERGY`) for DIG2-DIG8 when valid
- Transferred to `GROUP_ENERGY_SUM` output on `WORD_COUNT = 0x18` (End-of-Event capture)

### Port Summary

| Port | Dir | Width | Description |
|------|-----|-------|-------------|
| MASTER_CLK | in | 1 | Board-wide master clock |
| RST | in | 1 | Power-on reset |
| SERDES_LOCK | in | 1 | SERDES lock status (0=locked, 1=unlocked) |
| SERDES_DATA | in | 16 | Input from SERDES after DC restoration |
| CHANNEL_ID | out | 8×8 | Crystal ID for each digitizer |
| HIT_PATTERN | out | 8×40 | 40-bit hit patterns |
| CC_ENERGY | out | 8×16 | Central contact energies |
| HIT_TIMESTAMP | out | 8×16 | Timestamps of energy values |
| SPARE_A | out | 8×8 | 13th word from each digitizer |
| SPARE_B | out | 8×8 | 14th word from each digitizer |
| DIGVALID | out | 8 | Valid flag per digitizer |
| CHAN_MASK_MAP | out | 8 | Router channel mask bits |
| ROUTER_LOCK_MAP | out | 8 | Frame sync bits |
| DIG_CODE_ERR_MAP | out | 8 | Grey code error bits |
| DIG_ERROR_MAP | out | 8 | Digitizer error bits |
| DIG_PILEUP_MAP | out | 8 | Digitizer pileup bits |
| THROTTLE_REQ_MAP | out | 8 | Per-digitizer throttle requests |
| PATTERN_MATCH_MAP | out | 8 | Router pattern match flags |
| ROUTER_SPARE_MAP | out | 8 | Spare Router status bits |
| THROTTLE_REQUEST | out | 1 | Any channel requesting throttle |
| LOAD_ERR | out | 1 | Channel not locked to Router |
| TRAILER_FLAG | out | 1 | One tick at end of every Router cycle |
| STATE_MON | out | 9 | Debug: [8:5] = FSM state, [4:0] = WORD_COUNT |
| GROUP_ENERGY_SUM | out | 24 | Sum of CC energies for this Router group |

### Context

The MTRG has 8 Router inputs. Each has one `mt_pipeline` instance, giving 8 × 8 = 64 digitizer data lanes available to the trigger algorithms. The `GROUP_ENERGY_SUM` from each Router is an intermediate step toward the global energy sum used by algorithms like `calc_total_sum`.

## See Also

- [MTRG_calc_total_sum.md](MTRG_calc_total_sum.md) — downstream aggregator of per-Router GROUP_ENERGY_SUM / RTR_SUM_OF_X/Y
- [MTRG_trig_algo_support.md](MTRG_trig_algo_support.md) — trigger algorithm support logic used in pipeline processing
- [MTRG_mstr_mach.md](MTRG_mstr_mach.md) — Master Trigger state machine that governs SERDES framing
- [MTRG_mt_input_channel.md](MTRG_mt_input_channel.md) — individual digitizer input channel (instantiated by this pipeline)
- [deep_fpga_MTRG_MAIN.md](../deep_fpga_MTRG_MAIN.md) — MTRG top-level architecture and SERDES link format
