# RTrig EPICS DB Templates — Receiver Trigger Board

Stability: C3 - Structural / stable

Source: `ANLDAQ/ioc/db/RTrigRegisters.template` (1390 lines) and `RTrigUser.template` (8452 lines)
Date documented: 2026-04-25

---

## Table of Contents

- [Overview](#overview)
- [RTrigRegisters.template — Raw Register Map](#rtrigregisterstemplate--raw-register-map)
  - [Write registers (longout)](#write-registers-longout)
  - [Read registers (longin)](#read-registers-longin-scan1-second)
- [RTrigUser.template — User-Facing PV Groups](#rtrigusertemplate--user-facing-pv-groups)
  - [1. NIM Output / LED Control](#1-nim-output--led-control-mbbo/mbbi)
  - [2. LVDS Pre-emphasis](#2-lvds-pre-emphasis-mbbombbi)
  - [3. Channel Monitor FIFO Mode](#3-channel-monitor-fifo-mode-mbbo-cf1cf8)
  - [4. X/Y Plane Data Sources](#4-xy-plane-data-sources-mbbo)
  - [5. Fast-Strobe Select](#5-fast-strobe-select-mbbo)
  - [6. Input Link Mask](#6-input-link-mask--ilm_a-through-ilm_lru-bo)
  - [7. LED Manual Control](#7-led-manual-control--led4-through-led12-bo)
  - [8. Clock Source](#8-clock-source--clksrc-bo)
  - [9. AUX I/O Direction](#9-aux-io-direction--a_3_0_dir-a_7_4_dir-b_3_0_dir-b_7_4_dir-bo)
  - [10. SERDES Power Control](#10-serdes-power-control)
  - [11. SERDES Lock / Line Control](#11-serdes-lock--line-control)
  - [12. Link Reset / Lock Management](#12-link-reset--lock-management-bo)
  - [13. Trigger / Throttle Control](#13-trigger--throttle-control-bo)
  - [14. DC Balance](#14-dc-balance-bo)
  - [15. Force SYNC](#15-force-sync--link_a-through-link_h-bo)
  - [16. X-Plane Map](#16-x-plane-map--xmap_a_0-through-xmap_h_9-bo)
  - [17. Y-Plane Map](#17-y-plane-map--ymap_a_0-through-ymap_h_9-bo)
  - [18. Connection Connector Masks](#18-connection-connector-masks--conn_a_mask-through-conn_d_mask-bo)
  - [19. FIFO Resets](#19-fifo-resets--fiforeset00-through-fiforeset15-bo)
  - [20. Timing / Delay Parameters](#20-timing--delay-parameters-longout)
  - [21. Discriminator Delays](#21-discriminator-delays--discriminator_delay_a_0-through-discriminator_delay_h_9-longout)
  - [22. Status Readbacks](#22-status-readbacks-longin-scan1-second)
- [Key Architecture Notes](#key-architecture-notes)
- [Related Files](#related-files)

---

## Overview

The **Receiver Trigger (RTrig)** board is the intermediate trigger hub in the DGS VME chain.
It sits between the digitizer links and the Master Trigger (MTrig), collecting discriminator
bits from up to 8 input links (A–H), building X/Y multiplicity planes, applying throttle logic,
and forwarding sync/trigger information.

Two EPICS DB templates define its IOC interface:

| File | Records | Purpose |
|------|---------|---------|
| `RTrigRegisters.template` | ~140 raw longout/longin pairs | Direct register read/write (bit-access via asynUInt32Digital mask) |
| `RTrigUser.template` | ~500+ structured PVs | User-facing mbbo/mbbi, bo/bi, longout/longin records |

**PV prefix:** `VME$(CRATE):$(BOARD):` — e.g. `VME01:RTRIG01:`

All records use `asynUInt32Digital` driver type with `asynMask($(BOARD), addr, mask, shift)`.

---

## RTrigRegisters.template — Raw Register Map

Each raw register has a matching `_RBV` readback (SCAN=1 second). Below is the complete
register inventory extracted from the template.

### Write registers (longout)

| Register Name | Description |
|---------------|-------------|
| `reg_MON1_FIFO` – `reg_MON8_FIFO` | Board-wide monitor FIFO (8 monitors) |
| `reg_CHAN1_FIFO` – `reg_CHAN8_FIFO` | Channel-specific monitor FIFO (8 channels) |
| `reg_INPUT_LINK_MASK` | Mask out unused links |
| `reg_LED_REG` | Select front panel LED function |
| `reg_SKEW_CTL_A/B/C` | Link clock skew control (3 registers) |
| `reg_MISC_CLK_CTL` | Overall clock control |
| `reg_AUX_IO_CTL` | AUX I/O direction control |
| `reg_AUX_IO_DATA` | Manual Aux I/O data |
| `reg_AUX_INPUT_SELECT` | RS-485 routing |
| `reg_AUX_COUNTDOWN` | Test pattern control |
| `reg_SERDES_TPOWER` | SERDES transmitter power control |
| `reg_SERDES_RPOWER` | SERDES receiver power control |
| `reg_SERDES_LOCAL_LE` | SERDES local loop-enable |
| `reg_SERDES_LINE_LE` | SERDES line reconfiguration enable |
| `reg_LVDS_PREEMPHASIS` | LVDS pre-emphasis configuration |
| `reg_LINK_LRU_CTL` | Manual LRU (Link Reuse?) controls |
| `reg_MISC_CTL1` / `reg_MISC_CTL2` | Miscellaneous controls |
| `reg_GENERIC_TEST_FIFO` | FIFO function test |
| `reg_DIAG_PIN_CTL` | Diagnostic pin group selection |
| `reg_FORCE_SYNC_REG` | Override data to bad links (force SYNC) |
| `reg_SPARE_854` | Spare, unused |
| `reg_X_PLANE_MAP_A` – `reg_X_PLANE_MAP_H` | X-plane map assignments (8 links) |
| `reg_Y_PLANE_MAP_A` – `reg_Y_PLANE_MAP_H` | Y-plane map assignments (8 links) |
| `reg_ANY_THROTTLE_WIDTH` | Throttle pulse width |
| `reg_THROTTLE_LIMIT_TIME` | Delay before digitizer is allowed to re-trigger |
| `reg_MON1_FIFO_SEL` – `reg_MON8_FIFO_SEL` | Board-wide Monitor FIFO selection (8) |
| `reg_CHAN_MON_FIFO_CTL` | Channel-specific FIFO input mode |
| `reg_CHAN_MON_FIFO_WE_CTL` | Channel FIFO write-enable control |
| `reg_TSCATTER_DELAY` | Overlap time to distinguish scatter events |
| `reg_CLEAN_DIRTY` | Router-to-master control |
| `reg_PULSED_CTL1` / `reg_PULSED_CTL2` | Pulsed control registers |
| `reg_SPARE_8E8` / `reg_SPARE_8EC` | Spare unused |
| `reg_FIFO_RESETS` | Reset all monitor FIFOs |
| `reg_CPLD_MASK` | CPLD mask |
| `reg_FS_SOURCE` | Fast-strobe source select |
| `reg_FS_MULT_THRESH` | Fast-strobe multiplicity threshold |

### Read registers (longin, SCAN=1 second)

| Register Name | Description |
|---------------|-------------|
| `reg_LOCK_BUS_RBV` | Current link lock bus status |
| `reg_DEN_BUS_RBV` | Link driver enabled bus |
| `reg_REN_BUS_RBV` | Link receiver enabled bus |
| `reg_SYNC_BUS_RBV` | Link sync bus |
| `reg_TIMESTAMP_A/B/C_RBV` | 48-bit timestamp: bits [47:32], [31:16], [15:0] |
| `reg_MISC_STAT_REG_RBV` | Miscellaneous status bits |
| `reg_DiagnosticA/B/C/D/E/F/G/H_RBV` | 8 diagnostic counter registers |
| `reg_THROTTLE_STATUS_RBV` | Throttle status register |
| `reg_CODE_DATE_RBV` | Firmware code modification date |
| `reg_CODE_REVISION_RBV` | PCB/Firmware revision number |
| `reg_MON1_FIFO_RBV` – `reg_MON8_FIFO_RBV` | Monitor FIFO readbacks |
| `reg_CHAN1_FIFO_RBV` – `reg_CHAN8_FIFO_RBV` | Channel FIFO readbacks |
| `reg_MON_FIFO_STATE_RBV` | Collected empty & full flags for monitor FIFOs |
| `reg_CHAN_FIFO_STATE_RBV` | Collected empty & full flags for channel FIFOs |
| `reg_LOCK_COUNTER_A/B/C/D/E/F/G/H_RBV` | Per-link lock-count counters (8 links) |
| `reg_INPUT_LINK_MASK_RBV` | Readback of input link mask |
| `reg_CONN_A/B/C/D_DATA_RBV` | Connector data readbacks (4 connectors) |
| `reg_CPLD_MULT_RBV` | Current CPLD multiplicity sum |

---

## RTrigUser.template — User-Facing PV Groups

### 1. NIM Output / LED Control (mbbo/mbbi)

| PV | Options | Description |
|----|---------|-------------|
| `LEDControl` | Link Status / Trig Status / Reserved / Manual | Front panel LED source |
| `NIMSrc1` | SubSrc / AnyTrig / Sync / FastStrobe | NIM output 1 source | ✅ verified 2026-04-26 — `RTrigUser.template:NIMSrc1` (ZRST=SubSrc/0, ONST=AnyTrig/1, TWST=Sync/2, THST=FastStrobe/3; OUT mask `0x00003000` shift 12 → reg_AUX_IO_CTL)
| `NIMSrc2` | SubSrc / AnyTrig / ImpSync / RemoteSync | NIM output 2 source | ✅ verified 2026-04-26 — `RTrigUser.template:NIMSrc2` (ZRST=SubSrc/0, ONST=AnyTrig/1, TWST=ImpSync/2, THST=RemoteSync/3; OUT mask `0x0000C000` shift 14 → reg_AUX_IO_CTL)

### 2. LVDS Pre-emphasis (mbbo/mbbi)

Three group controls, each with options: none / some / more / most

| PV | Bits | Mask | Shift | Link Group |
|----|------|------|-------|------------|
| `PEABCD` | bits[4:3] of reg_LVDS_PREEMPHASIS | 0x18 | 3 | Links A–D | ✅ verified 2026-04-26 — `RTrigUser.template:L52` (`@asynMask($(BOARD),0,0x00000018,3)reg_LVDS_PREEMPHASIS`) |
| `PEEFG` | bits[6:5] | 0x60 | 5 | Links E–G | ✅ verified 2026-04-26 — `RTrigUser.template:L43` (`@asynMask($(BOARD),0,0x00000060,5)reg_LVDS_PREEMPHASIS`) |
| `PEHLRU` | bits[8:7] | 0x180 | 7 | Links H/L/R/U | ✅ verified 2026-04-26 — `RTrigUser.template:L34` (`@asynMask($(BOARD),0,0x00000180,7)reg_LVDS_PREEMPHASIS`) |

### 3. Channel Monitor FIFO Mode (mbbo CF1–CF8)

8 channel FIFOs, each configurable:
- **Mode** (`CF1_MODE` – `CF8_MODE`): RAW DATA / X/Y/THROTTLE / Diag / COUNTS ✅ verified 2026-04-26 — `RTrigUser.template:CF1_MODE` (ZRST=RAW DATA/0, ONST=X/Y/THROTTLE/1, TWST=Diag/2, THST=COUNTS/3; `0x00000003` bits[1:0] → reg_CHAN_MON_FIFO_CTL)
- **Write-enable** (`CF1_MODE_WE` – `CF8_MODE_WE`): OFF / ON / Non-zero / Any non-zero

### 4. X/Y Plane Data Sources (mbbo)

| PV | Options | Description |
|----|---------|-------------|
| `X_SELECT` | CLEAN / DIRTY / MODULE / DISCBITS | Source for X-plane discriminator data | ✅ verified 2026-04-25 — `RTrigUser.template:L217-224` (mbbo; ZRST=CLEAN/ZRVL=1, ONST=DIRTY/ONVL=2, TWST=MODULE/TWVL=4, THST=DISCBITS/THVL=0; OUT mask `0x0000000F` → reg_CLEAN_DIRTY bits [3:0])
| `Y_SELECT` | CLEAN / DIRTY / MODULE / DISCBITS | Source for Y-plane discriminator data | ✅ verified 2026-04-25 — `RTrigUser.template:L226-232` (same encoding; OUT mask `0x000000F0` → reg_CLEAN_DIRTY bits [7:4])

### 5. Fast-Strobe Select (mbbo)

| PV | Options |
|----|---------|
| `FS_SEL` | fast or / fast mult / discbit or / (AorB)AND(CorD) / A or B / C or D / 0 / 1 | ✅ verified 2026-04-26 — `RTrigUser.template:FS_SEL` (8 options 0–7; OUT mask `0x000000E0` shift 5 → reg_FS_SOURCE bits [7:5])

Controls what logic drives the FastStrobe output.

### 6. Input Link Mask — `ILM_A` through `ILM_L/R/U` (bo)

One binary PV per physical link connector. Setting to 1 masks that link out of the
discriminator OR/multiplicity logic.

Links covered: A, B, C, D, E, F, G, H, L, R, U
(11 links total — note L/R/U are additional connectors beyond the 8 main links A–H) ✅ verified 2026-04-26 — `RTrigUser.template`: `grep -c 'record(bo.*ILM_'` = 11; links are ILM_A/B/C/D/E/F/G/H/L/R/U

### 7. LED Manual Control — `LED4` through `LED12` (bo)

Individual manual LED control bits for front panel LEDs when `LEDControl=Manual`.

### 8. Clock Source — `ClkSrc` (bo)

Selects the board clock source: **local** (internal oscillator) / **link** (recovered from SERDES link). ✅ verified 2026-04-26 — `RTrigUser.template:L706` (ZNAM=local, ONAM=link; `OUT @asynMask($(BOARD),0,0x00008000,1)reg_MISC_CLK_CTL`)

### 9. AUX I/O Direction — `A_3_0_DIR`, `A_7_4_DIR`, `B_3_0_DIR`, `B_7_4_DIR` (bo)

Sets direction of auxiliary I/O nibbles on connectors A and B.

### 10. SERDES Power Control

Per-link transmit power (`TPwr_A` – `TPwr_U`, 11 links) and receive power
(`RPwr_A` – `RPwr_U`, 11 links). Binary on/off.

### 11. SERDES Lock / Line Control

| PV Group | Description |
|----------|-------------|
| `SLoL_A` – `SLoL_U` (11) | SERDES local loop-enable per link |
| `SLiL_A` – `SLiL_U` (11) | SERDES line loop-enable per link |
| `PrE_0/1/2` | Pre-emphasis individual bit controls |
| `LRUCtl00–LRUCtl10` (9) | LRU control bits |

### 12. Link Reset / Lock Management (bo)

| PV | Description |
|----|-------------|
| `LostLockRstL/R/U` | Reset lost-lock counter for L/R/U links |
| `LOCK_RETRY` | Retry link lock sequence |
| `LOCK_ACK` | Acknowledge lock status |
| `RESET_LINK_INIT` | Reset link initialization state machine |

### 13. Trigger / Throttle Control (bo)

| PV | Description |
|----|-------------|
| `ENABLE_VETO` | Enable veto input |
| `STRINGENT_LOCK` | Require more stringent lock criteria |
| `FORCE_THRTL_ON` | Force throttle on |
| `FORCE_THRTL_OFF` | Force throttle off |
| `DIAG_THROTTLE_TYPE` | Select diagnostic throttle type |
| `NIM_THROTTLE_SELECT` | Which throttle channel (A–H) drives NIM output |
| `THROTTLE_TIME_RANGE` | Throttle time multiplier: ×20.48µs / ×20.97ms / ×21.47s / forever | ✅ verified 2026-04-25 — `RTrigUser.template:L64-71` (mbbo; ZRST="* 20.48 us"/0, ONST="*20.97 ms"/1, TWST="*21.47 sec"/2, THST="forever"/3; OUT mask `0x0000C000` shift 14 → reg_THROTTLE_LIMIT_TIME bits [15:14])

### 14. DC Balance (bo)

| PV | Description |
|----|-------------|
| `LinkL_DCbal` | DC balance for L-link |
| `Link_A-H_R_U_TX_DCBAL` | DC balance for A–H, R, U transmit links |

### 15. Force SYNC — `LINK_A` through `LINK_H` (bo)

Forces SYNC data onto a link even if the link is in a bad state.
One bit per main link (A–H). Maps to `reg_FORCE_SYNC_REG`.

### 16. X-Plane Map — `XMAP_A_0` through `XMAP_H_9` (bo)

Per-link, per-channel mapping into the X-plane multiplicity bus.
8 links × 10 channels = 80 individual bit-mapped PVs. ✅ verified 2026-04-25 — `RTrigUser.template`: `grep -c 'record(bo.*XMAP_'` = 80 (links A–H, channels 0–9; each maps one bit of reg_X_PLANE_MAP_A–H via asynMask 0x0001–0x0200)
Each bit (when set to "Map") includes that digitizer channel in the X-plane count.

### 17. Y-Plane Map — `YMAP_A_0` through `YMAP_H_9` (bo)

Same structure as X-plane but for Y-plane multiplicity. 80 PVs total. ✅ verified 2026-04-25 — `RTrigUser.template`: `grep -c 'record(bo.*YMAP_'` = 80

### 18. Connection Connector Masks — `conn_a_mask` through `conn_d_mask` (bo)

Masks for the 4 external multiplicity connector outputs (A–D).

### 19. FIFO Resets — `FIFOReset00` through `FIFOReset15` (bo)

16 individual FIFO reset bits. Maps to `reg_FIFO_RESETS`. ✅ verified 2026-04-26 — `RTrigUser.template`: `grep -c 'record(bo.*FIFOReset'` = 16 (FIFOReset00–FIFOReset15)

### 20. Timing / Delay Parameters (longout)

| PV | Description |
|----|-------------|
| `PULSE_COUNTDOWN` | AUX countdown pulse length |
| `BIT_5_OFFSET` | Bit 5 offset adjustment |
| `THROTTLE_WIDTH` | Duration to assert throttle (in clock ticks) |
| `THROTTLE_FILTER_TIME` | Number of slow clocks required before re-trigger |
| `OVERLAP_DELAY` | Overlap time between Ge events (trigger scatter window) |
| `ASSERTION_DELAY` | Duration to assert status signal |
| `Threshold` | Multiplicity threshold for fast-strobe logic |

### 21. Discriminator Delays — `DISCRIMINATOR_DELAY_A_0` through `DISCRIMINATOR_DELAY_H_9` (longout)

Per-link, per-channel programmable delay applied to discriminator inputs before
the multiplicity logic. 8 links × 10 channels = 80 PVs. ✅ verified 2026-04-26 — `RTrigUser.template`: `grep -c 'record(longout.*DISCRIMINATOR_DELAY'` = 80

### 22. Status Readbacks (longin, SCAN=1 second)

| PV | Description |
|----|-------------|
| `DEN_BUS_RBV` | Link driver enabled bus |
| `REN_BUS_RBV` | Link receiver enabled bus |
| `SYNC_BUS_RBV` | Link sync bus status |
| `Timestamp_Big_RBV` | Upper 16 bits of 48-bit timestamp (reg_TIMESTAMP_A) |
| `Timestamp_Middle_RBV` | Middle 16 bits (reg_TIMESTAMP_B) |
| `Timestamp_Little_RBV` | Lower 16 bits (reg_TIMESTAMP_C) |
| `Diag_A/B/C/D/E/F/G/H_RBV` | 8 diagnostic counters |
| `CODE_DATE_RBV` | Firmware date |
| `Code_Revision_RBV` | Firmware/PCB revision |
| `MON_FIFO_FLAGS_RBV` | Monitor FIFO empty/full flags |
| `CHAN_FIFO_FLAGS_RBV` | Channel FIFO empty/full flags |
| `LOCK_COUNT_A/B/C/D/E/F/G/H_RBV` | Per-link lock event counters |
| `PULSE_COUNTDOWN_RBV` | Readback of pulse countdown |
| `BIT_5_OFFSET_RBV` | Readback of bit 5 offset |
| `THROTTLE_WIDTH_RBV` | Readback of throttle width |
| `THROTTLE_FILTER_TIME_RBV` | Readback of throttle filter time |
| `OVERLAP_DELAY_RBV` | Readback of overlap delay |
| `ASSERTION_DELAY_RBV` | Readback of assertion delay |
| `masked_sum_a/b/c/d_RBV` | Per-connector masked multiplicity sums (4 connectors) |
| `Threshold_RBV` | Multiplicity threshold readback |
| `Mult_sum_RBV` | Current total multiplicity sum (from CPLD) |

---

## Key Architecture Notes

1. **8 input links (A–H):** Each carries discriminator bits from digitizers. The RTrig assembles
   these into X/Y multiplicity planes and sends summary data upstream to the MTrig.

2. **Additional connectors L/R/U:** Beyond the 8 main data links, there are L/R/U connectors
   (left/right/up?) likely used for inter-crate or daisy-chain connections.

3. **Multiplicity planes:** X_SELECT and Y_SELECT allow routing CLEAN (gated) or DIRTY (ungated)
   discriminator data into the respective planes. The XMAP/YMAP 80-bit arrays define which
   digitizer channels feed into each plane.

4. **Throttle system:** A multi-stage throttle:
   - `THROTTLE_WIDTH` = how long to assert throttle
   - `THROTTLE_FILTER_TIME` = refractory period
   - `THROTTLE_TIME_RANGE` = scale factor (µs to forever)
   - `NIM_THROTTLE_SELECT` = which of 8 throttle channels drives the NIM output

5. **Fast Strobe (FS):** Separate fast logic path: `FS_SEL` picks the OR/mult/bit logic,
   `Threshold` sets the multiplicity cutoff, `FS_SOURCE` selects among logic variants.

6. **SERDES per-link control:** Full independent enable/disable, power, pre-emphasis, and
   loopback for all 11 links — useful for commissioning and fault isolation.

7. **Monitor FIFOs:** 8 board-wide + 8 channel-specific FIFOs with configurable input modes
   (raw data / X/Y/throttle / diagnostic / counts). Used for offline monitoring.

8. **Discriminator delays:** Per-link, per-channel programmable delays (DISCRIMINATOR_DELAY_*)
   allow fine-tuning of hit coincidence windows across the array.

---

## Related Files

- `knowledgeBase/EPICS_DB_templates.md` — MTrig/SDig/MDig template overview (parent file; links here)
- `knowledgeBase/vxworks_trigger_drivers.md` — VxWorks RTRG/MTRG asyn drivers that write these PVs
- `knowledgeBase/260E_trigger_scheme.md` — End-to-end RTRG/MTRG trigger firmware logic these PVs control
- `knowledgeBase/hardware_architecture.md` — RTrig/MTrig hardware description (board-level: DIG/RTRG/MTRG roles, signal flow)
- `knowledgeBase/connectors.md` — NIM/LVDS connector assignments
- `knowledgeBase/multi_system_linking.md` — Cross-system clock/trigger sharing (uses L/R/U link PVs documented here)
- `knowledgeBase/VME_registers.md` — Full RTrig/MTrig VME register map
