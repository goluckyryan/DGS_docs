# MTRG: SERDES_RX_Mach_R2.vhd Analysis

Stability: C3 - Structural / stable

**Source:** `~/FPGA_svn2git/MTRG_git/MAIN_FPGA/trunk/Source/SERDES_RX_Mach_R2.vhd`
**Entity name:** `SERDES_RX_Mach`
**Authors:** JTA, MBO (Argonne National Laboratory, Jan–Feb 2014)
**Last analyzed:** 2026-04-24

---

## Table of Contents

- [Purpose](#purpose)
- [Generic Parameter](#generic-parameter)
- [Frame Structure (100 words per cycle)](#frame-structure-100-words-per-cycle)
- [Lock Acquisition (Prelock State Machine)](#lock-acquisition-prelock-state-machine)
- [Lock Maintenance (Stringent Lock)](#lock-maintenance-stringent-lock)
- [Data Pattern Checking (DCHECK Process)](#data-pattern-checking-dcheck-process)
- [Frame Decoders](#frame-decoders)
  - [Frame 1 — Sync / Imperative Sync](#frame-1--sync--imperative-sync)
  - [Frame 2 — Debug/Monitor](#frame-2--debugmonitor)
  - [Frames 3–10 — Trigger Decision Frames](#frames-310--trigger-decision-frames)
  - [Frame 11 — Spare (Null)](#frame-11--spare-null)
  - [Frame 12 — Router Internal Commands](#frame-12--router-internal-commands)
  - [Frame 13 — Gretina Demand Slow Data](#frame-13--gretina-demand-slow-data)
  - [Frame 14 — Router Internal Commands (MYRIAD / Digitizer Tester)](#frame-14--router-internal-commands-myriad--digitizer-tester)
  - [Frame 15 — GRETINA Asynchronous Commands](#frame-15--gretina-asynchronous-commands)
  - [Frame 16 — DGS Synchronous System Capture](#frame-16--dgs-synchronous-system-capture)
  - [Frame 17 — Auxiliary Detector Commands](#frame-17--auxiliary-detector-commands)
  - [Frames 18 & 19 — Spare Null Frames](#frames-18--19--spare-null-frames)
  - [Frame 20 — End-of-Cycle](#frame-20--end-of-cycle)
- [VETO_EVENT Output](#veto_event-output)
- [Propagation Control Register Bit Map](#propagation-control-register-bit-map)
- [Key Outputs Summary](#key-outputs-summary)
- [Implementation Notes](#implementation-notes)
- [See Also](#see-also)

---

## Purpose

`SERDES_RX_Mach` is the SERDES reception state machine instantiated inside each MTRG link receiver. It deserializes the 20-frame, 100-word (16-bit per word) control data stream arriving from the Master Trigger (or Gretina master), validates lock, decodes all defined command frames, and produces decoded flag outputs for use by the rest of the MTRG firmware.

This is one of the most central and complex modules in the MTRG. It replaces GRETINA-era raw SERDES handling with generalized support for DGS Master, DGS Router, and Gretina Master command formats.

---

## Generic Parameter

| Name | Type | Meaning |
|------|------|---------|
| `LINK_IS_L` | `std_logic` | `'1'` = this is the clock-providing (L) link; allows propagation of Sync, Frame 12, Frame 14, Frame 16 outputs. `'0'` = side link (R or U); trigger propagation only. |

---

## Frame Structure (100 words per cycle)

The MTRG control link carries 20 frames per cycle, each 5 words × 16 bits. Word index runs 0–99 in locked operation, with 100–104 used for prelock states.

| Frame | Word indices | Purpose |
|-------|-------------|---------|
| F1 (Sync) | 0–4 | Sync / Imperative-Sync; 48-bit timestamp |
| F2 (Debug) | 5–9 | Detector state, XTRA data, remote veto map |
| F3–F10 (Triggers) | 10–49 | 8 trigger decision frames (one per frame); cmd + 48-bit TS |
| F11 (Spare) | 50–54 | Null: 0xAAAA×4 + 0x0000 |
| F12 (Router Cmd) | 55–59 | Internal Router/Data-Gen commands; stripped to Null by Router at digitizer |
| F13 (Demand Slow Data) | 60–64 | Gretina demand-slow-data; fixed pattern 0x40FB/A5A5/5A5A/A5A5/A5A5 |
| F14 (Router Cmd) | 65–69 | Internal commands (Digitizer Tester / MYRIAD); stripped to Null by Router |
| F15 (GRETINA Async) | 70–74 | Asynchronous command (cal inject, latch status, FE reset, reset links) |
| F16 (DGS Sync Capture) | 75–79 | Synchronous system capture command: cmd byte + 32-bit TS + length + FIFO delay |
| F17 (Aux Det Cmd) | 80–84 | Auxiliary detector command frame |
| F18 (Spare) | 85–89 | Null frame |
| F19 (Spare) | 90–94 | Null frame |
| F20 (End-of-Cycle) | 95–99 | Fixed: 0xFFFF / 0x0000 / 0xFFFF / 0x0000 / 0x5555 |

---

## Lock Acquisition (Prelock State Machine)

On reset, the machine enters **IDLE** and waits for the 5-word End-of-Cycle (F20) signature before declaring lock.

Prelock sequence:
1. `IDLE`: wait for `0x0000` → `PRELOCK1`
2. `PRELOCK1`: wait for `0xFFFF` → `PRELOCK2`; mismatch → stays `PRELOCK1`
3. `PRELOCK2`: wait for `0x0000` → `PRELOCK3`; mismatch → `IDLE`
4. `PRELOCK3`: wait for `0xFFFF` → `PRELOCK4`; mismatch → `IDLE`
5. `PRELOCK4`: wait for `0x0000` → `PRELOCK5`; mismatch → `IDLE`
6. `PRELOCK5`: wait for `0x5555` → enter F1W1 (locked), `xWORD_INDEX ← 0`; mismatch → `IDLE`

✅ verified 2026-04-24 — SERDES_RX_Mach_R2.vhd:L549–L671 (full prelock state logic)
**Note:** PRELOCK1 stays in PRELOCK1 on any non-0xFFFF word; PRELOCK2–5 return to IDLE on mismatch. `xSERDES_SM_LOCKED` is held `'0'` throughout prelock.

---

## Lock Maintenance (Stringent Lock)

Once locked, each word is compared against the `FIXED_BITS` constant ROM (128 × 16-bit entries, indexed by `xWORD_INDEX`), enabled per-word per-format by `DATA_CHECK_EN`:

- `DATA_CHECK_FLAG` = `'1'` if data matches expected pattern (or check disabled for that word)
- **Stringent mode** (`STRINGENT_LOCK_FLAG = '1'`): any failed check returns to `PRELOCK1`
- **Non-stringent mode**: only F20 words enforce re-lock
- F20 always checks data regardless of stringent flag; a mismatch always causes re-lock

Three formats selectable via `COMMAND_FORMAT`:
- `DGS_MASTER`: only 5th word of most frames checked; Frames 11/13 fully checked
- `DGS_ROUTER`: no fixed-bit checking on frames 1–10; 5th word checked on 11–19; F20 always
- `GRETINA`: F2 and F11 fully checked; Frames 13/14/16/18/19 fully checked

**Lost-lock SR latch:** `xSERDES_SM_LOST_LOCK_FLAG` sets if lock was previously acquired then lost; cleared by `SERDES_SM_LOST_LOCK_RST`.

---

## Data Pattern Checking (DCHECK Process)

Implemented as a combinational ROM lookup against `FIXED_BITS(xWORD_INDEX)`:
- Synthesizes to three 128×16-bit ROMs in hardware
- `LATCHED_CONTROL_DATA` is registered one cycle behind `RECEIVED_CONTROL_DATA`
- `DATA_VALID_MON` (= `DATA_CHECK_FLAG`) and `LATCHED_CONTROL_DATA_MON` are monitoring outputs

---

## Frame Decoders

### Frame 1 — Sync / Imperative Sync
- Word 1 (`xWORD_INDEX=0`): command word
  - `0x01FF` / `0x0100` = non-imperative sync (with/without timestamp rollover)
  - `0x81FF` / `0x8100` = imperative sync
  - Anything else → `PRELOCK1`
- Words 2–4: 48-bit timestamp (latched [47:32], [31:16], [15:0])
- Word 5: asserts `SERDES_SYNC_FLAG` and `SERDES_ISYNC_FLAG` (Link L + propagation enabled only); `VETO_EVENT ← data[9:0]`
- All sync outputs gated by `PROPAGATION_CONTROL_REG(0)` and `LINK_IS_L`

### Frame 2 — Debug/Monitor
- Word 1 (index 5): `RECEIVED_TRIG_MON_DET_DATA` (target wheel state)  
  ✅ verified 2026-04-24 — SERDES_RX_Mach_R2.vhd:L376 (`if xWORD_INDEX = "0000101"` → `RECEIVED_TRIG_MON_DET_DATA <= RECEIVED_CONTROL_DATA`)
- Word 2 (index 6): `RECEIVED_TRIG_MON_XTRA_DATA` (X/Y plane multiplicities)  
  ✅ verified 2026-04-24 — SERDES_RX_Mach_R2.vhd:L379 (`if xWORD_INDEX = "0000110"` → `RECEIVED_TRIG_MON_XTRA_DATA`)
- Word 3 (index 7): `REMOTE_MASTER_VETO_STATE`  
  ✅ verified 2026-04-24 — SERDES_RX_Mach_R2.vhd:L382 (`if xWORD_INDEX = "0000111"` → `REMOTE_MASTER_VETO_STATE`)
- Words 4–5: ignored; `VETO_EVENT` updated on index 9

### Frames 3–10 — Trigger Decision Frames
- Handled by looping states `TRIGGER_DECISION_FRAME_WORD_1..5` across all 8 frames (40 words total, indices 10–49)
- `TRIG_PROPAGATION_VECTOR` = `PROPAGATION_CONTROL_REG(8:1)`, shifted right one bit per frame
- Word 1 of each trigger frame: if `TRIG_PROPAGATION_VECTOR(1)='1'` and first word ≠ null (not `0x5xxx` MSN):
  - Captures `xTRIG_TYPE ← data[10:8]`, sets `EARLY_TRIG_FLAG`
  - Null pattern detection: `data[15:11] = "10101"` (0xAAAA upper nibbles) → null  
    ✅ verified 2026-04-24 — SERDES_RX_Mach_R2.vhd:L916 (`LATCHED_CONTROL_DATA(15 downto 11) /= "10101"` is non-null check)
- Words 2–4: latch 48-bit trigger timestamp [47:32], [31:16], [15:0]
- Word 5: if `EARLY_TRIG_FLAG`, latches `TRIG_TIMESTAMP` and asserts `xTRIG_FLAG`; `VETO_EVENT ← data[9:0]`
- After word 5 (index 49): transition to FRAME11

### Frame 11 — Spare (Null)
- Loops until `xWORD_INDEX = 54`; expects all 0xAAAA + 0x0000
- `VETO_EVENT` updated on last word

### Frame 12 — Router Internal Commands
- Indices 55–59 (`FRAME_12_LATCH` process, separate from main FSM)
- **Sanitized output:** always replaced with 0xAAAA×4 + 0x0000 (strips commands from Routers before digitizer sees them)
- `FRAME_12_REQ_FLAG`: asserted when non-null frame received AND `PROPAGATION_CONTROL_REG(9)='1'` AND `LINK_IS_L='1'`
- `FRAME_12_DATA[1..5]`: captured 5-word array; held until `FRAME_12_ACK_FLAG` clears it
- Non-null detection: `F12W1 ≠ 0xAAAA`

### Frame 13 — Gretina Demand Slow Data
- Fixed pattern: `0x40FB / 0xA5A5 / 0x5A5A / 0xA5A5 / 0xA5A5`
- Retained for GRETINA compatibility; does nothing in DGS digitizer
- `MSTR_MACH_START_FLAG` asserted when `xWORD_INDEX = 64` (F13 last word), used to synchronize Remote Master's master state machine  
  ✅ verified 2026-04-24 — SERDES_RX_Mach_R2.vhd:L509–L521 (MSTR_MACH_START_PROC: `if xWORD_INDEX = 64 then MSTR_MACH_START_FLAG <= '1'`)

### Frame 14 — Router Internal Commands (MYRIAD / Digitizer Tester)
- Indices 65–69; handled identically to Frame 12 (`FRAME_14_LATCH`)
- `FRAME_14_REQ_FLAG`, `FRAME_14_DATA[1..5]`, `FRAME_14_ACK_FLAG`
- `VETO_FROM_REMOTE_MASTER` — declared as `out std_logic` with comment "from frame 14, word 4, bit 15" but **never assigned inside SERDES_RX_Mach_R2.vhd** — port is undriven (VHDL default `'0'` applies). ✅ verified 2026-04-24 — SERDES_RX_Mach_R2.vhd:L71 (port decl only, grep confirms 1 occurrence total). In `Generated_top.vhd`, signals `VETO_FROM_REMOTE_MASTER_L/R/U` (init `'0'`) receive this undriven output (lines 4657/4729/4819) and are OR'd into `ANY_VETO_FROM_REMOTE_MASTER` (line 2075), which feeds `SYSTEM_VETO_STATE[15:13]` and the per-algo veto check (line 2504). Because the port is never driven, **the Remote Master Veto feature is permanently inactive** — the intent (extract bit 15 of F14W4) is described in comments but was never implemented in SERDES_RX_Mach_R2. Added 2021-06-16 per revision history.
- Propagation gated by `PROPAGATION_CONTROL_REG(10)` (data latch) and `PROPAGATION_CONTROL_REG(9)` (req flag — copy-paste quirk) and `LINK_IS_L`

✅ verified 2026-04-24 — SERDES_RX_Mach_R2.vhd:L71 + Generated_top.vhd:L513-516,L2075,L2084-2086,L2504,L4657/4729/4819.

### Frame 15 — GRETINA Asynchronous Commands
- Indices 70–74
- Word 1 (`xWORD_INDEX=70`): decodes command byte `data[15:8]`:
  - `0x04` → `CAL_INJECT_FLAG`
  - `0x08` → `LATCH_STATUS_FLAG`
  - `0x10` → `FRONT_END_RESET_FLAG`
  - `0x18` → `RESET_LINKS_FLAG`
- All flags are one-clock pulses (cleared on next word at index 71)

✅ verified 2026-04-24 — SERDES_RX_Mach_R2.vhd:L1181–1197 (F15 decoder case statement: `X"04"`, `X"08"`, `X"10"`, `X"18"`)

### Frame 16 — DGS Synchronous System Capture
- Indices 75–79 (states F16W1..F16W5)
- Word 1: if `data[15:8] ≠ 0xAA` → sets `EARLY_CMD_FLAG` (non-null capture command)  
  ✅ verified 2026-04-24 — SERDES_RX_Mach_R2.vhd:L1234 (`if(LATCHED_CONTROL_DATA(15 downto 8) /= X"AA")`)
- Words 2–3: `xSYNC_CAPTURE_TS[31:16]`, `[15:0]` (32-bit start timestamp)  
  ✅ verified 2026-04-24 — SERDES_RX_Mach_R2.vhd:L1256 (`xSYNC_CAPTURE_TS(31 downto 16)`), L1281 (`xSYNC_CAPTURE_TS(15 downto 0)`)
- Word 4: `xSYNC_CAPTURE_LENGTH` (16-bit capture duration)  
  ✅ verified 2026-04-24 — SERDES_RX_Mach_R2.vhd:L1304 (`xSYNC_CAPTURE_LENGTH <= LATCHED_CONTROL_DATA`)
- Word 5: `xSYNC_CAPTURE_FIFO_DELAY` (16-bit FIFO delay); asserts `xSYNC_CAPTURE_FLAG` for one cycle  
  ✅ verified 2026-04-24 — SERDES_RX_Mach_R2.vhd:L1325 (`xSYNC_CAPTURE_FLAG <= EARLY_CMD_FLAG`), L1331 (`xSYNC_CAPTURE_FIFO_DELAY <= LATCHED_CONTROL_DATA`)
- All outputs gated by `PROPAGATION_CONTROL_REG(12)` and `LINK_IS_L`

### Frame 17 — Auxiliary Detector Commands
- Indices 80–84; spare currently; `VETO_EVENT` updated on last word

### Frames 18 & 19 — Spare Null Frames
- Indices 85–89, 90–94; expect 0xAAAA×4 + 0x0000

### Frame 20 — End-of-Cycle
- Indices 95–99 (states F20W1..F20W5)
- Data always checked (ignores STRINGENT_LOCK_FLAG)
- Expected: `0xFFFF / 0x0000 / 0xFFFF / 0x0000 / 0x5555`  
  ✅ verified 2026-04-24 — SERDES_RX_Mach_R2.vhd:L247 (FIXED_BITS ROM: `X"FFFF", X"0000", X"FFFF", X"0000", X"5555" -- FRAME 20`)
- Any mismatch → immediate return to `PRELOCK1`
- F20W5: resets `xWORD_INDEX ← 0`, transitions to `F1W1`

---

## VETO_EVENT Output

`VETO_EVENT[9:0]` carries per-channel veto commands from the Router. It is updated from the **5th word (W5)** of almost every frame (the Router embeds it there). The field is zeroed between frame W5 positions. In the DGS Router format, W5 of all frames carries veto data; in DGS Master format, Frame 20 W5 is reserved at 0x5555 so VETO_EVENT is zeroed there.

---

## Propagation Control Register Bit Map

| Bit | Frame | Notes |
|-----|-------|-------|
| 0 | F1 (Sync) | Enable sync/timestamp propagation |
| 1–8 | F3–F10 (Triggers) | Per-frame trigger propagation enable |
| 9 | F12 | Internal Router cmd propagation (Link L only) |
| 10 | F14 | Internal Router cmd propagation (Link L only) |
| 11 | F15 | (Gretina async — internal to trigger system) |
| 12 | F16 | Sync capture propagation (Link L only) |
| 13 | F17 | Aux detector command propagation |
| 14–15 | spare | Unused |

---

## Key Outputs Summary

| Signal | Asserted when |
|--------|---------------|
| `SERDES_SM_LOCKED` | In lock (not prelock), DATA_CHECK_FLAG ok if stringent |
| `SERDES_SM_LOST_LOCK_FLAG` | SR latch — set if lock lost after acquisition |
| `UNQUALIFIED_SM_LOCKED` | Same as DATA_CHECK_FLAG (fixed-bit check, ignores stringent) |
| `SERDES_SYNC_FLAG` | F1W5, Link L + propagation enabled |
| `SERDES_ISYNC_FLAG` | F1W5, Imperative Sync + Link L + propagation enabled |
| `SERDES_SYNC_TIMESTAMP` | 48-bit TS from F1W2-W4 |
| `TRIG_FLAG` | F3–F10 W5, non-null trigger detected |
| `TRIG_TIMESTAMP` | Latched 48-bit TS from trigger frame W2-W4; persists until next trigger |
| `TRIG_TYPE` | 3-bit type from trigger frame W1[10:8] |
| `FRAME_12_REQ_FLAG` | Non-null F12 received, propagation enabled, Link L |
| `FRAME_14_REQ_FLAG` | Non-null F14 received, propagation enabled, Link L |
| `VETO_FROM_REMOTE_MASTER` | F14 word 4 bit 15 — **intent only; never driven in SERDES_RX_Mach_R2.vhd; always '0'** ✅ verified 2026-04-24 |
| `CAL_INJECT_FLAG` | F15W1, cmd=0x04 (one clock) |
| `LATCH_STATUS_FLAG` | F15W1, cmd=0x08 (one clock) |
| `FRONT_END_RESET_FLAG` | F15W1, cmd=0x10 (one clock) |
| `RESET_LINKS_FLAG` | F15W1, cmd=0x18 (one clock) |
| `SYNC_CAPTURE_FLAG` | F16W5, non-null capture cmd, Link L + propagation |
| `SYNC_CAPTURE_TS` | 32-bit timestamp from F16W2-W3 |
| `SYNC_CAPTURE_LENGTH` | 16-bit from F16W4 |
| `SYNC_CAPTURE_FIFO_DELAY` | 16-bit from F16W5 |
| `MSTR_MACH_START_FLAG` | Pulses at xWORD_INDEX=64 (F13 last word); syncs Remote Master FSM |
| `VETO_EVENT[9:0]` | Per-channel veto from Router, embedded in W5 of each frame |
| `SANITIZED_CONTROL_DATA` | Received data passthrough; Frames 12/14 replaced with Null |

---

## Implementation Notes

- Clock: 50 MHz (`CLK50`), registered throughout
- `TRIG_TIMESTAMP` does NOT clear between triggers (changed 2015-09-18 by JTA) — once latched, persists until next trigger event  
  ✅ verified 2026-04-24 — SERDES_RX_Mach_R2.vhd:L303-304 (comment: "line commented out … so that TRIG_TIMESTAMP, once latched, stays asserted until"; L547 `TRIG_TIMESTAMP <= X"000000000000"` only in IDLE/reset)
- `DATA_CHECK_FLAG` synthesizes to three 128×16-bit ROMs (Xilinx)
- Multiple identical copies instantiated in MTRG — one per SERDES link (up to 11 for main + L/R/U)
- VETO_FROM_REMOTE_MASTER was added 2021-06-16 as an unimplemented stub — port is declared but never driven; always outputs '0'; the intent (extract F14W4[15]) is documented in comments only. ✅ verified 2026-04-24 — SERDES_RX_Mach_R2.vhd:L71 + Generated_top.vhd:L513-516,L4657/4729/4819.
- FRAME_14_LATCH uses `PROPAGATION_CONTROL_REG(9)` for the req flag but `PROPAGATION_CONTROL_REG(10)` for the data latch — likely a copy-paste quirk; only the data latch matters for downstream use  
  ✅ verified 2026-04-24 — SERDES_RX_Mach_R2.vhd:L471 (REQ: `PROPAGATION_CONTROL_REG(9)`) vs L481 (data latch: `PROPAGATION_CONTROL_REG(10)`)

---

## See Also

- [MTRG_top.md](MTRG_top.md) — top-level, shows how SERDES_RX_Mach is instantiated per link
- [MTRG_support_modules.md](MTRG_support_modules.md) — link_tx_block (transmit side), remote_trig_support
- [MTRG_mstr_mach.md](MTRG_mstr_mach.md) — uses MSTR_MACH_START_FLAG from this module
- [deep_fpga_MTRG_MAIN.md](../deep_fpga_MTRG_MAIN.md) — MTRG architecture overview
- [PROGRESS.md](PROGRESS.md) — coverage checklist
