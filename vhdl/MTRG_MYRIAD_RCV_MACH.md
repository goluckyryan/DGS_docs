# MTRG: `MYRIAD_RCV_MACH.vhd` — MγRIAD Receive State Machine

**Module:** `MYRIAD_RCV_MACH`  
**File:** `FPGA/Firmware_Tags/MasterTrigger/20220705/Source/MYRIAD_RCV_MACH.vhd`  
**Author:** John T. Anderson (ANL), created 2011-09-03  
**Part of:** MTRG Main FPGA (Virtex-4 / Kintex UltraScale in Vivado port)  
**Clock:** 50 MHz board-wide clock  
**Source tag used:** `20220705` (most recent tag available)

---

## Purpose

Locks onto the 5-word serial data stream sent by the MγRIAD module over SERDES Link U and extracts:
- NIM input states (8 bits)
- ECL input states (8 bits)
- FERA control states (8 bits)
- Raw trigger bit (leading-edge trigger from NIM/ECL input)
- Gated trigger bit (output of MγRIAD local coincidence logic)
- 10 MHz clock flag bit

This module is purely a **receiver / decoder** — it does not make trigger decisions. The outputs feed `MYRIAD_TRIGGER.vhd` which handles trigger formation.

---

## MγRIAD → MTRG Serial Data Frame

The MγRIAD sends a **repeating 5-word frame** of 16-bit words over SERDES at 50 MHz:

| Bit | Meaning (all words) |
|-----|---------------------|
| 15 | 10 MHz flag (set once per 5-word frame, roughly every 5 clocks) |
| 14:13 | Always `00` — used as sync check |
| 12 | **Raw trigger** — fires on NIM or ECL input designated as aux trigger |
| 11 | **Gated trigger** — fires when MγRIAD's internal coincidence logic is satisfied |
| 10:8 | 3-bit ordinal counter (word index 0–4, for sync verification) |
| 7:0 | Word-specific payload (see table below) |

| Word | Ordinal (bits 10:8) | Bits 7:0 |
|------|---------------------|----------|
| 00 | — | `0xAD` (reset sync marker — part of `0x0BAD`) |
| 01 | `000` | NIM input states (8 inputs, sampled at 100 MHz) |
| 02 | `001` | ECL input states |
| 03 | `010` | FERA control states |
| 04 | `011` | 8-bit secondary frame counter (sync check) |
| 05 | `100` | Fixed value `0xA5` (end-of-frame marker / lock anchor) |

> **Note:** Words are labeled 00–05 in the comment header, but the state machine uses states W01–W05 (the "word 00 = 0xAD" is only sent during reset/unlock and is handled implicitly by the prelock states).

---

## State Machine

States: `PRELOCK1` → `PRELOCK2` → `W01` → `W02` → `W03` → `W04` → `W05` → (back to W01)

### PRELOCK1
- Waits for SERDES to achieve lock (`LINK_LOCK = '0'`, active LOW)
- All outputs held at defaults (MACHINE_LOCKED='0')

### PRELOCK2
- SERDES is locked; searching for word W05 anchor (`bits[14:13]="00"`, `bits[7:0]=0xA5`)
- If found → advance to W01
- If SERDES lock drops → return to PRELOCK1

### W01 — NIM status word
- Captures `bits[7:0]` → `NIM_STAT`
- Captures `bit[12]` → `RAW_TRIGGER`
- Captures `bit[11]` → `GATED_TRIGGER`
- Captures `bit[15]` → `CLK_10_FLAG`
- Verifies `bits[14:13]="00"` AND `bits[10:8]="000"` → if OK sets `MACHINE_LOCKED='1'`, advances to W02
- Mismatch → `MACHINE_LOCKED='0'`, return to PRELOCK1

### W02 — ECL status word
- Captures `bits[7:0]` → `ECL_STAT`
- Same trigger bits captured each word
- Verifies ordinal `bits[10:8]="001"` → advance to W03 or unlock

### W03 — FERA control states
- Captures `bits[7:0]` → `FERA_STAT`
- Verifies ordinal `bits[10:8]="010"` → advance to W04 or unlock

### W04 — Frame counter
- Does not capture bits[7:0] (frame counter; comment says "may be replaced with something of interest later")
- Verifies ordinal `bits[10:8]="011"` → advance to W05 or unlock

### W05 — End-of-frame anchor
- Verifies `bits[14:13]="00"`, `bits[7:0]=0xA5"`, `bits[10:8]="100"`
- If OK: `MACHINE_LOCKED='1'`, `EOF_ERROR='0'`, loop back to W01
- Mismatch: `MACHINE_LOCKED='0'`, `EOF_ERROR='1'`, return to PRELOCK1

---

## Ports

| Port | Dir | Width | Description |
|------|-----|-------|-------------|
| `CLK` | in | 1 | 50 MHz board clock |
| `RST` | in | 1 | Active-HIGH power-on reset |
| `RECEIVED_MYRIAD_DATA` | in | 16 | 16-bit data word from SERDES (re-latched into CLK domain) |
| `LINK_LOCK` | in | 1 | SERDES lock signal (active LOW = locked) |
| `CLK_10_FLAG` | out | 1 | 10 MHz flag bit extracted from bit 15 |
| `MACHINE_LOCKED` | out | 1 | '1' when state machine is synchronized to MγRIAD frame |
| `NIM_STAT` | out | 8 | NIM input states (bits 8:1 from W01) |
| `ECL_STAT` | out | 8 | ECL input states (from W02) |
| `FERA_STAT` | out | 8 | FERA control input states (from W03) |
| `RAW_TRIGGER` | out | 1 | MγRIAD raw trigger bit (bit 12, present every word) |
| `GATED_TRIGGER` | out | 1 | MγRIAD gated trigger bit (bit 11, present every word) |
| `EOF_ERROR` | out | 1 | Set if W05 sync check fails (MBO-added error flag) |

---

## Integration in MTRG

- Instantiated in `top.vhd` (MTRG Main FPGA top level)
- Input `RECEIVED_MYRIAD_DATA` ← `LAT_LINKU_RX` (SERDES Link U receive data, re-latched) ✅ verified 2026-04-07 — `top.vhd:L3562`
- Output `NIM_STAT`, `ECL_STAT`, `FERA_STAT`, `RAW_TRIGGER`, `GATED_TRIGGER` → `MYRIAD_TRIGGER` entity
- Output `MACHINE_LOCKED` → `MYRIAD_TRIGGER.MYRIAD_DATA_LOCK` (used to gate trigger processing)
- A parallel receiver `GITMO_RCV_MACH` exists for Link L (remote master / GRETINA); a VME control bit selects which receiver is held in reset

---

## Notes

- `NIM_STAT` port is declared `(8 downto 1)` — 1-indexed, matching MγRIAD's NIM input numbering (NIM In 1–8 in hardware documentation, though MYRIAD.vhd itself uses 0-indexed)
- The `CLK_10_FLAG` bit (bit 15) provides a 10 MHz reference for timing verification between modules
- Lock robustness: any single-word ordinal mismatch immediately resets to PRELOCK1 — tight sync discipline

---

*Created: 2026-04-21 from source `MYRIAD_RCV_MACH.vhd` (tag 20220705)*
