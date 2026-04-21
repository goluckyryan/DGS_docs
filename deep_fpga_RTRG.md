# RTRG — Router Firmware

## Table of Contents

- [Target Device](#target-device)
- [Role](#role)
- [Repository Layout](#repository-layout)
- [Source Files](#source-files)
  - [Top Level](#top-level)
  - [Master Trigger Interface](#master-trigger-interface)
  - [Digitizer Data Path](#digitizer-data-path)
  - [Throttle Control](#throttle-control)
  - [Link Management](#link-management)
  - [Register Interface & I/O](#register-interface--io)
- [Architecture](#architecture)
  - [Signal Flow](#signal-flow)
  - [Data Packet to Master Trigger](#data-packet-to-master-trigger-link-l-tx)
  - [Clock Domains](#clock-domains)
  - [SERDES Links](#serdes-links)
  - [Command Frame Handling](#command-frame-handling)
- [VME Register Map](#vme-register-map)
  - [Control Registers (Write)](#control-registers-write)
  - [Status Registers (Read)](#status-registers-read)
  - [MISC_STAT Register Bit Map](#misc_stat-register-bit-map-rtrg)
- [IP Cores](#ip-cores)
- [disc_mach.vhd — BGO/Ge Discriminator State Machine](#disc_machvhd--bgoge-discriminator-state-machine)
- [Router → MTRG SERDES Frame Format](#router--mtrg-serdes-frame-format-132-bit-word-per-channel-per-cycle)
- [Build Artifacts](#build-artifacts)

## Target Device

| Field | Value |
|-------|-------|
| Family | Virtex-4 |
| Part | xc4vlx80 | ✅ verified 2026-04-18 — `FPGA/Firmware_Tags/Router/Release_Dec_2014/.../Work13_4.xise` (`Device=xc4vlx80`) |
| Package | ff1148 | ✅ verified 2026-04-18 — same file (`Package=ff1148`) |
| Speed Grade | -11 | ✅ verified 2026-04-18 — same file (`Speed Grade=-11`) |
| Tool | Xilinx ISE 14.7 ✅ verified 2026-04-17 — `Work13_4.xise:L15` (`ise_version="14.7"`) + `router_top.par:L1` (`Release 14.7 par P.20131013`) — folder name `Work13_4` is a label, not the ISE version |
| Project File | `Firmware/DGS_Version/Rtr4704_mod_for_reset/MAIN_FPGA_4704_mod/Work13_4/Work13_4.xise` |
| Top Entity | `router_top` |
| Bitfile | `Firmware/DGS_Version/Rtr4704_mod_for_reset/MAIN_FPGA_4704_mod/Work13_4/router_top.bit` |

## Role

The Router sits between the Master Trigger and 8 Digitizers. It:
1. Receives the 20-frame command structure from the Master Trigger via Link L (SERDES)
2. Forwards commands to 8 Digitizers via Links A–H
3. Collects discriminator hit patterns from all 8 Digitizers
4. Aggregates X-plane and Y-plane multiplicity counts
5. Returns a compact status packet back to the Master Trigger
6. Manages throttle flow control — filters and stretches throttle requests from Digitizers to prevent FIFO overflows
7. Provides VME register access for configuration and monitoring

## Repository Layout

```
RTRG/
├── DGSRouterTriggerRegisterMap.xls     # Register map documentation
└── Firmware/DGS_Version/
    ├── MAIN_FPGA/                       # Initial version
    └── Rtr4704_mod_for_reset/           # Current production version
        └── MAIN_FPGA_4704_mod/
            ├── Source/                  # VHDL source files
            ├── Cores/                   # ISE IP cores
            └── Work13_4/               # Build outputs
```

## Source Files

**Location:** `Firmware/DGS_Version/Rtr4704_mod_for_reset/MAIN_FPGA_4704_mod/Source/`

### Top Level
| File | Description |
|------|-------------|
| `TOP.VHD` | Top-level entity (`router_top`) — integrates all submodules |
| `trigger_data_types.vhd` | Custom array type definitions |
| `trigger_comp_defs.vhd` | Component declarations |
| `trigger_top_comp_defs.vhd` | Top-level component declarations |

### Master Trigger Interface
| File | Description |
|------|-------------|
| `SERDES_RX_Mach_R2.vhd` | Receives and decodes 20-frame command structure from Master Trigger on Link L (Sync, Trigger, Frame 12/14/16/17) |
| `dc_balance_mach.vhd` | DC-balances data before retransmission to Digitizers |
| `timestamp.vhd` | 48-bit timestamp counter synchronized to Master Trigger |

### Digitizer Data Path
| File | Description |
|------|-------------|
| `router_data_path.vhd` | Aggregates 8 digitizer channels; extracts X/Y plane discriminator bits; counts multiplicity; generates throttle summary; outputs 16-bit packet to Master |
| `chan_in.vhd` | Per-channel input processing (8 instances) |
| `DCBAL_in.vhd` | DC-balance removal with FIFO |
| `DCBAL_in_nofifo.vhd` | DC-balance removal without FIFO |
| `disc_mach.vhd` | Discriminator state machine |
| `overlap_mach.vhd` | Overlap detection state machine |
| `disparity_lookup.vhd` | DC balance disparity lookup table |

### Throttle Control
| File | Description |
|------|-------------|
| `throttle_limiters.vhd` | Filters throttle requests from 8 Digitizers; requires continuous assertion for programmable time; stretches valid requests to >2 µs pulses ✅ verified 2026-04-14 — `throttle_limiters.vhd:L23-26,L80` ("minimum assertion time of 2us"; `COUNTER_START=400` @ 50 MHz = 2 µs) |
| `throttle_monos.vhd` | Per-channel retriggerable monostable stretcher: generates 2 µs pulse for each of the 8 Digitizer throttle requests on `SPARE_LVDS[8:1]`, ensuring any request (even a 20 ns glitch) propagates to the MTRG. Also produces `ANY_THROTTLE_REQ_OUT` with programmable width (`ANY_THROTTLE_WIDTH_REG`). `COUNTER_START=400` (400 × 20 ns = 2 µs). `MISC_CTL2_REG[9]` = force all on; `[8]` = block all; `INPUT_LINK_MASK_REG` prevents masked channels from requesting throttle. ✅ verified 2026-04-19 — `throttle_monos.vhd:L8-18,L50-60,L62` (entity ports + COUNTER_START=400 comment: "2us @ 50MHz") |

### Link Management
| File | Description |
|------|-------------|
| `link_init.vhd` | SERDES link initialization state machine; monitors LOCK on all 8 digitizer links and 3 special links (L, R, U) |
| `channel_resets.vhd` | Per-channel reset control |
| `DCM_CONTROLLER.vhd` | Clock distribution and multiplexing |

### Register Interface & I/O
| File | Description |
|------|-------------|
| `registers.vhd` | VME register map (157 `when X"..."` case entries, highest address 0x113C) — control, status, diagnostics ✅ verified 2026-04-21 — `registers.vhd`: `grep -c "when X\""` = 157; `sort -u` max = `X"113C"` |
| `AUX_IO.VHD` | Auxiliary front panel I/O multiplexing |
| `LED_CTL.VHD` | LED indicator control |

## Architecture

### Signal Flow

```
Master Trigger
    │
    │  Link L (SERDES, 18-bit DC-balanced)
    ▼
SERDES_RX_Mach  ──── timestamp sync
    │
    │  Decoded command frames (Sync, Trigger, Veto, etc.)
    ▼
dc_balance_mach  ──────────────────────────────────────────────┐
    │                                                           │
    │  DC-balanced commands                                     │
    ▼                                                           │
Links A–H TX ──► 8 Digitizers                                  │
                                                               │
8 Digitizers ──► Links A–H RX (18-bit, DC-balanced)           │
    │                                                           │
    ▼                                                           │
chan_in ×8  (DC balance removal, per-channel decode)           │
    │                                                           │
    ▼                                                           │
throttle_limiters  (filter & stretch throttle requests)        │
    │                                                           │
    ▼                                                           │
router_data_path                                               │
  - X-plane multiplicity count                                 │
  - Y-plane multiplicity count                                 │
  - Global throttle OR                                         │
    │                                                           │
    ▼                                                           │
Link L TX ◄────────────────────────────────────────────────────┘
(16-bit packet back to Master Trigger)
```

### Data Packet to Master Trigger (Link L TX)

| Bits | Field | Description |
|------|-------|-------------|
| 17 | CG | Clock guard (DC balance) |
| 16 | THR | Global throttle request (OR of all 8 channels) |
| 15:9 | Y-mult | Y-plane multiplicity (7 bits, 0–80) |
| 8 | VAL | Data valid: `ALL_DIGITIZERS_LOCKED AND ROUTER_LOCKED` |
| 7:1 | X-mult | X-plane multiplicity (7 bits, 0–80) |
| 0 | POL | Polarity (DC balance) |

✅ verified 2026-04-07 — `router_data_path.vhd` header comment (lines 7–11): `LINKL_RAW_DATA[14:8]`=Y-mult, `LINKL_RAW_DATA[7]`=VAL, `LINKL_RAW_DATA[6:0]`=X-mult; these map to frame bits [15:9], [8], [7:1] respectively (LINKL_RAW_DATA[15:0] → frame[16:1]). Adder tree: 4-bit per-link → 5-bit → 6-bit → 7-bit total (3 ranks for 8 links). Physical max=80 (8 links × 10 ch). VAL = `ALL_DIGITIZERS_LOCKED AND ROUTER_LOCKED` ✅ verified 2026-04-11 — `router_data_path.vhd:L222`.

### Clock Domains

| Clock | Frequency | Source | Used For |
|-------|-----------|--------|----------|
| switched_master_clock | 50 MHz | ICS mux (oscillator or Link L RX clock) | Main logic, data path, state machines | ✅ verified 2026-04-09 — `TOP.VHD:L420` ("actual, buffered, logic master 50MHz clock throughout the device")
| switched_master_clock_2x | 100 MHz | DCM from master clock | DC balance, high-speed logic | ✅ verified 2026-04-09 — `TOP.VHD:L421` ("actual, buffered, master 100MHz clock throughout the device")
| oscillator_clock | 50 MHz | Dedicated oscillator (always-on) | VME registers, link initialization | ✅ verified 2026-04-09 — `DCM_CONTROLLER.vhd:L24` (port comment: "50 Mhz Oscillator") + `L52` (lockup timeout: "1 second @ 50 Mhz")

### SERDES Links

| Link | Direction | Connected To |
|------|-----------|--------------|
| A–H (8 links) | Bidirectional | 8 Digitizers |
| L | RX from Master, TX to Master | Master Trigger |
| R | Spare / diagnostic | — |
| U | Spare / diagnostic | — |

### Command Frame Handling

The Router receives the 20-frame command stream from the MTRG on Link L and forwards it to all 8 Digitizers on Links A–H. The full frame protocol is defined in [deep_fpga_MTRG_MAIN.md — Command Frame Timing](deep_fpga_MTRG_MAIN.md#command-frame-timing).

| Frame | MTRG → Router | Router → Digitizers |
|-------|--------------|---------------------|
| 1 | Sync + 48-bit timestamp | Forwarded unchanged |
| 2 | Detector status | Forwarded unchanged |
| 3–10 | Trigger decisions (×8 slots) | Forwarded unchanged |
| 11 | Spare (null) | Forwarded unchanged |
| 12 | Router Command (counters, FIFOs) | **Stripped** — Digitizers see null (0xAAAA) ✅ verified 2026-04-13 — `SERDES_RX_Mach_R2.vhd:L191` |
| 13 | GRETINA Demand Slow Data | Forwarded unchanged |
| 14 | Router Command | **Stripped** — Digitizers see null (0xAAAA) ✅ verified 2026-04-13 — `SERDES_RX_Mach_R2.vhd:L191,L1183–1188` |
| 15 | Async Command | Forwarded unchanged |
| 16 | Sync Capture | Forwarded unchanged |
| 17 | Auxiliary Detector Command | Forwarded unchanged |
| 18–19 | Spare (null) | Forwarded unchanged |
| 20 | End-of-Cycle | Forwarded unchanged |

The Router **extracts** the per-channel veto mask (`VETO[9:0]`) from bits [9:0] of Word 5 of trigger-decision frames received from the MTRG (via `SERDES_RX_Mach_R2.vhd:L889,L1048`). The MTRG places the veto in those bits; the Router reads them and forwards the frame unchanged (`SANITIZED_CONTROL_DATA`) to all 8 Digitizers. The Router does **not** insert or modify VETO bits in outgoing frames. ✅ verified 2026-04-13 — `SERDES_RX_Mach_R2.vhd:L44,L889,L1048,L1119–1122` (SANITIZED_CONTROL_DATA passes LATCHED_CONTROL_DATA unchanged except frames 12/14)

## VME Register Map

### Control Registers (Write)

| Address | Register | Description |
|---------|----------|-------------|
| 0x000 | INPUT_LINK_MASK | Mask active digitizer links (power-up: 0x0000) ✅ verified 2026-04-18 — `TOP.VHD:L2230` |
| 0x004 | LED_REG | LED control (power-up: 0x0000) ✅ verified 2026-04-18 — `TOP.VHD:L2231` |
| 0x008 | SKEW_CTL_A | Clock skew control — U50 buffer chip, 8 outputs (power-up: 0x0000) ✅ verified 2026-04-18 — `TOP.VHD:L2232,L1074-1084` |
| 0x00C | SKEW_CTL_B | Clock skew control — U53 buffer chip (power-up: 0x0000) ✅ verified 2026-04-18 — `TOP.VHD:L2233` |
| 0x010 | SKEW_CTL_C | Clock skew control — third buffer chip (power-up: 0x0000) ✅ verified 2026-04-18 — `TOP.VHD:L2234` |
| 0x014 | MISC_CLK_CTL | Clock mux selection (power-up: 0xB800; bit 15=1 enables Link-L clock fallback when locked) ✅ verified 2026-04-18 — `TOP.VHD:L2235,L880` |
| 0x018 | AUX_IO_CTL | Auxiliary I/O mode (power-up: 0x0000) ✅ verified 2026-04-18 — `TOP.VHD:L2236` |
| 0x01C | AUX_IO_DATA | Software AUX I/O data (power-up: 0x0000) ✅ verified 2026-04-18 — `TOP.VHD:L2237` |
| 0x028 | SERDES_TPOWER | SERDES TX power-down control (bits 10:0 → Links A–H,L,R,U) ✅ verified 2026-04-18 — `TOP.VHD:L2240,L952-962` |
| 0x02C | SERDES_RPOWER | SERDES RX power-down control (bits 10:0 → Links A–H,L,R,U) ✅ verified 2026-04-18 — `TOP.VHD:L2241,L964-974` |
| 0x030 | SERDES_LOCAL_LE | SERDES LOCAL_LE pin control per link ✅ verified 2026-04-18 — `TOP.VHD:L2242` |
| 0x034 | SERDES_LINE_LE | SERDES LINE_LE pin control per link ✅ verified 2026-04-18 — `TOP.VHD:L2243` |
| 0x03C | LINK_LRU_CTL | DEN/REN/SYNC for Links L, R, U — bits: DEN_L[0], REN_L[1], SYNC_L[2], DEN_R[4], REN_R[5], SYNC_R[6], DEN_U[8], REN_U[9], SYNC_U[10] ✅ verified 2026-04-18 — `TOP.VHD:L2245,L895-932` |
| 0x040 | MISC_CTL1 | Global control (reset, veto enable, etc.) ✅ verified 2026-04-18 — `TOP.VHD:L2246` |
| 0x044 | MISC_CTL2 | Secondary control ✅ verified 2026-04-18 — `TOP.VHD:L2247` |
| 0x050 | FORCE_SYNC | Manual override of sync to SERDES links (added 2012-02-28) ✅ verified 2026-04-18 — `TOP.VHD:L2251` |
| 0x058–0x074 | X_PLANE_MAP[1–8] | X-plane discriminator type mapping per DIG channel (8 regs × 16 bits) ✅ verified 2026-04-18 — `TOP.VHD:L2253-2260` |
| 0x078–0x094 | Y_PLANE_MAP[1–8] | Y-plane discriminator type mapping per DIG channel (8 regs × 16 bits) ✅ verified 2026-04-18 — `TOP.VHD:L2261-2268` |
| 0x098 | ANY_THROTTLE_WIDTH | Throttle pulse width | ✅ verified 2026-04-09 — `TOP.VHD:L2270` (`REG_098 => ANY_THROTTLE_WIDTH_REG`) |
| 0x09C | THROTTLE_LIMIT_TIME | Min assertion time for throttle | ✅ verified 2026-04-09 — `TOP.VHD:L2271` (`REG_09C => THROTTLE_LIMIT_TIME_REG`; added 2016-03-02 for `throttle_limiters`) |
| 0x0C8 | TSCATTER_DELAY | Ge/BGO timing for dirty hits — bits[14:8]=ASSERTION_DELAY (7-bit, how long CLEAN/DIRTY pulses are stretched), bits[6:0]=OVERLAP_DELAY (7-bit Compton scatter coincidence window). **Default: 0x3020** (ASSERTION_DELAY=48 clocks=480 ns, OVERLAP_DELAY=32 clocks=320 ns at 100 MHz) ✅ verified 2026-04-21 — `RTRG/Firmware/DGS_Version/MAIN_FPGA/Source/registers.vhd:L895` (`xREG_0C8 <= X"3020"`) |
| 0x0CC | CLEAN_DIRTY | Clean/dirty/module detection mode — bit[15]=use delay-corrected DELAYED_DATA vs RECOVERED_DATA; bits[3:0]=X_SELECT source (0000=DFMA raw, 0001=HAVE_CLEAN, 0010=HAVE_DIRTY, 0100=HAVE_MODULE, 1000=HAVE_CLOVER_CLEAN); bits[7:4]=Y_SELECT source ✅ verified 2026-04-15 — `knowledgeBase/vhdl/RTRG_chan_in.md` §CLEAN_DIRTY control register modes (sourced from `chan_in.vhd`) |

### Status Registers (Read)

| Address | Register | Description |
|---------|----------|-------------|
| 0x100 | LOCK_BUS | Lock status of all 11 links | ✅ verified 2026-04-19 — `TOP.VHD:L2357` (`REG_100_IN => LOCK_BUS`; L980-989 builds LOCK_BUS from xLINKA_LOCK…xLINKL_LOCK) |
| 0x104–0x10C | DEN/REN/SYNC_BUS | Link enable/sync status | ✅ verified 2026-04-19 — `TOP.VHD:L2358-2360` (`REG_104_IN => DEN_BUS`, `REG_108_IN => REN_BUS`, `REG_10C_IN => SYNC_BUS`) |
| 0x114–0x11C | TIMESTAMP[47:0] | Current 48-bit timestamp (3 words) | ✅ verified 2026-04-19 — `TOP.VHD:L2362-2364` (`REG_114_IN => TIMESTAMP(47:32)`, `REG_118_IN => TIMESTAMP(31:16)`, `REG_11C_IN => TIMESTAMP(15:0)`) |
| 0x128 | MISC_STAT | See bit map below | ✅ verified 2026-04-19 — `TOP.VHD:L2367` (`REG_128_IN => MISC_STAT_REG`) |
| 0x12C–0x148 | DIAG_COUNTER[1–8] | Diagnostic counters per channel | ✅ verified 2026-04-19 — `TOP.VHD:L2368-2375` (`REG_12C_IN => DIAG_COUNTER(1)` … `REG_148_IN => DIAG_COUNTER(8)`) |
| 0x150 | THROTTLE_STATUS | Per-channel throttle status | ✅ verified 2026-04-09 — `TOP.VHD:L2315` (`REG_150_IN => THROTTLE_STATUS`) — both in `Rtr4704_mod_for_reset` (0x260E) and `20220705` tag |
| 0x158 | CODE_DATE | Firmware build date — `0x0414` (April 14) ✅ verified 2026-04-06 — TOP.VHD:L392 |
| 0x15C | CODE_REVISION | Code revision — `0x260E`: bits[15:12]=2 (PCB rev B), bits[11:8]=6 (DGS Router), bits[7:4]=0 (major), bits[3:0]=E (minor) ✅ verified 2026-04-06 — TOP.VHD:L369,L371–390 |
| 0x1B0–0x1CC | LOCK_COUNTER[1–8] | Lock event counters per link | ✅ verified 2026-04-19 — `TOP.VHD:L2384-2391` (`REG_1B0_IN => LOCK_COUNTER(1)` … `REG_1CC_IN => LOCK_COUNTER(8)`) |

### MISC_STAT Register Bit Map (RTRG)

EPICS PV: `VME$(CRATE):$(BOARD):reg_MISC_STAT_REG_RBV` (16-bit read-only)

| Bit | Mask | PV Suffix | Description |
|-----|------|-----------|-------------|
| 0 | 0x0001 | `NIM_IN1_RBV` | **=1 when NIM input 1 is high (signal present).** Reflects the instantaneous logic level of the NIM IN1 front-panel input on this RTRG. |
| 1 | 0x0002 | `NIM_IN2_RBV` | **=1 when NIM input 2 is high (signal present).** Same as bit 0 but for NIM IN2. |
| 2 | 0x0004 | `ROUTER_LOCKED_RBV` | **=1 when this RTRG’s upstream SERDES link (to the MTRG) is locked and communicating.** This is the key health indicator for the RTRG — if 0 during a run, the RTRG is not receiving trigger commands from the MTRG and no data will flow from its digitizers. |
| 7:3 | 0x00F8 | — | Reserved — always 0. |
| 11:8 | 0x0F00 | — | Reserved — always 0. (A STATE_MON hook existed in the old Spreadsheet_ref variant but is not connected in the current production source.) |
| 12 | 0x1000 | `LOST_LOCK_RBV` | **=1 if the SERDES state machine has lost lock at any point since last reset (latched).** Set by `SERDES_SM_LOST_LOCK_FLAG`. Unlike bit 2, this is sticky — it stays 1 even if lock is later re-acquired. A 1 here during a run means a transient link glitch occurred. |
| 13 | 0x2000 | — | Reserved — always 0. |
| 14 | 0x4000 | `ALL_LOCKED_RBV` | **=1 when all downstream SERDES links (to DIG boards on this RTRG’s channels) are locked.** Expected to be 1 during normal operation. If 0, one or more digitizer links on this RTRG are not communicating. |
| 15 | 0x8000 | `LOCK_ERROR_RBV` | **=1 if lock was lost after being successfully acquired (latched/sticky).** Normally 0. A 1 here indicates the link was healthy at some point but then degraded — worth investigating even if currently re-locked. |

✅ verified 2026-04-12 — `FPGA/RTRG/Firmware/DGS_Version/MAIN_FPGA/Source/TOP.VHD:L1214–1222,L2022,L2179–2180` + `ioc/db/RTrigUser.template:L3814–3864`

**Note:** RTRG has one MISC_STAT register (no MISC_STAT2). MTRG has two: MISC_STAT (link init + frame-pending flags) and MISC_STAT2 (SPARE_LVDS + throttle request).

## IP Cores

**Location:** `Firmware/DGS_Version/Rtr4704_mod_for_reset/MAIN_FPGA_4704_mod/Cores/`

| Core | Description |
|------|-------------|
| `chipscope_icon` | ChipScope controller |
| `chipscope_ila` | ChipScope logic analyzer |
| `fifo_16x1023_async` | 16-bit, 1K deep async FIFO |
| `fifo_16x64K_async` | 16-bit, 64K deep async FIFO |
| `BRAM_1024X16_REGSHADOW` | Block RAM register shadow |

## disc_mach.vhd — BGO/Ge Discriminator State Machine

_Source: `Router/20220705/Source/disc_mach.vhd` — Authors: J. Anderson + M. Oberling_

This is the per-channel module that classifies each hit as **CLEAN**, **DIRTY**, or **BGO_ONLY** — the core of Gammasphere's BGO Compton suppression trigger logic.

### Concept

Each Ge detector channel in the Router is paired with a BGO scintillator channel. The `disc_mach` watches the **leading edges** of both discriminator bits and applies a coincidence window (`OVERLAP_DELAY`) to classify events:

| Condition | Output |
|-----------|--------|
| Ge fires, BGO does **not** fire within OVERLAP window | `CLEAN_EVENT` — Ge without Compton scatter; good physics event |
| Ge fires, BGO fires within OVERLAP window | `DIRTY_EVENT` — Compton-scattered gamma; suppressed |
| BGO fires, Ge does **not** fire within OVERLAP window | `BGO_ONLY_EVENT` — BGO signal only (noise / cosmic) |
| Ge + BGO fire **simultaneously** | `DIRTY_EVENT` immediately (no timer needed) |

### Three Timing Parameters

1. **OFFSET** — compensates for relative discriminator timing between Ge and BGO (different rise times cause time walk); applied upstream before this state machine
2. **OVERLAP** (`OVERLAP_DELAY`, 7-bit, register-controlled) — the coincidence window; events within this window are dirty
3. **ASSERTION** — how long CLEAN/DIRTY/BGO_ONLY pulses are stretched before transmission to MTRG (separate register)

### State Machine (4 states)

```
ST_IDLE
  ├─ GE_EDGE only  → ST_OVERLAP_GE_FIRST  (start timer, wait for BGO)
  ├─ BGO_EDGE only → ST_OVERLAP_BGO_FIRST (start timer, wait for Ge)
  ├─ both edges    → ST_WAIT_DIRTY        (immediately dirty, skip wait)
  └─ no edge       → ST_IDLE

ST_OVERLAP_GE_FIRST:  (timer counts down each clock)
  ├─ timer = 0, no BGO   → assert CLEAN_EVENT (1 clock) → ST_IDLE
  ├─ BGO arrives, timer = 0 → assert DIRTY_EVENT immediately (1 clock) → ST_IDLE
  ├─ BGO arrives, timer > 0 → ST_WAIT_DIRTY  (count out remaining timer)
  └─ else               → ST_OVERLAP_GE_FIRST

ST_OVERLAP_BGO_FIRST:  (timer counts down; resets on repeated BGO edges)
  ├─ timer = 0, no Ge   → assert BGO_ONLY_EVENT (1 clock) → ST_IDLE
  ├─ GE arrives, timer = 0 → assert DIRTY_EVENT immediately (1 clock) → ST_IDLE
  ├─ GE arrives, timer > 0 → ST_WAIT_DIRTY  (count out remaining timer)
  └─ else               → ST_OVERLAP_BGO_FIRST

ST_WAIT_DIRTY:  (entered when the second discriminator fires with time still left)
  ├─ timer = 0 → assert DIRTY_EVENT (1 clock) → ST_IDLE
  └─ else      → ST_WAIT_DIRTY
```

> **Key subtlety** (MBO 20140610): When both discriminators fire and the overlap timer is still running, the machine enters `ST_WAIT_DIRTY` and counts out the *remaining* timer before asserting `DIRTY_EVENT`. If the second discriminator fires right as the timer hits zero (timer = 0 on that same clock), `DIRTY_EVENT` is asserted immediately (avoids the race where an edge arriving exactly at timer = 0 would produce a CLEAN or BGO_ONLY instead of DIRTY). ✅ verified 2026-04-17 — `disc_mach.vhd:L139,L155-161,L185-192,L207-220` (Rtr4704_mod_for_reset production source)

### Outputs

The one-clock-wide pulses `CLEAN_EVENT`, `DIRTY_EVENT`, and `BGO_ONLY_EVENT` are fed to `overlap_mach.vhd` which stretches them into the ASSERTION time window used by the multiplicity sum logic.

The stretched signals `HAVE_CLEAN` and `HAVE_DIRTY` (from `overlap_mach`) are transmitted to the MTRG as the X-sum (clean) and Y-sum (dirty) multiplicity contributions.

> This is why the MTRG has separate `EN_SUM_X` and `EN_SUM_Y` trigger enables — X = clean (Compton-suppressed Ge hits), Y = dirty (Compton-coincident). The Gammasphere physics trigger typically fires on `EN_SUM_X` with a threshold.

✅ verified 2026-04-13 — `disc_mach.vhd:L1-230` (full state machine), `SERDES_RX_Mach_R2.vhd` (upstream: feeds GE_DISC_FLAG + BGO_DISC_FLAG)

---

## Router → MTRG SERDES Frame Format (132-bit word per channel per cycle)

_Source: `DGS_SVN/dgs/Documentation/Formal/Unsorted Docs/router to master format notes.txt`_ ✅ verified 2026-04-16 — exact bit positions confirmed against SVN source

Each Router sends one 132-bit word to the MTRG per trigger cycle (2 µs), one word per connected DIG channel (up to 8). The MTRG receives these from all Routers to build its trigger decision.

| Bits | Field | Notes |
|------|-------|-------|
| 131:124 | **CRYSTAL ID** (8 bits) | Digitizer board ID |
| 123:120 | `1111` | Fixed marker |
| 119:112 | **BUFFER COUNT** (8 bits) | DIG event buffer fill level |
| 111:108 | `0000` | Fixed marker |
| 107:100 | **STATUS VALUE** (8 bits) | Aggregated status |
| 99 | **CHANMASK** | Channel masked by Router if set |
| 98 | **ANY_THROTTLE_REQUEST** | Set if any DIG is requesting throttle |
| 97 | (spare) | Reserved |
| 96 | **CHANNEL_THROTTLE_REQUEST** | Set if this DIG wants triggers throttled |
| 95:56 | **HIT PATTERN** (40 bits) | A(7:0), B(7:0), C(7:0), D(7:0), E(7:0) — discriminator hit bits from DIG channels |
| 55:48 | TIMESTAMP [7:0] | **Low byte** of 16-bit timestamp |
| 47:40 | TIMESTAMP [15:8] | **High byte** of 16-bit timestamp |
| 39:36 | CC ENERGY [3:0] | **Low nibble** of Central Contact Energy |
| 35:24 | CC ENERGY [15:4] | **High 12 bits** of Central Contact Energy |
| 23:16 | SPARE WORD 13 | 13th word from digitizer |
| 15 | **FS** — Frame Sync | Router pipeline synchronized to digitizer data |
| 14 | **GCE** — Gray Code Error | Router sees gray code error from digitizer |
| 13 | **DE** — Digitizer Error | Digitizer asserts ERROR bit |
| 12 | **PU** — Pileup | Digitizer asserts PILEUP bit |
| 11:4 | SPARE WORD 14 | 14th word from digitizer |
| 3:1 | **CHAN_CHECK** (3 bits) | Ordinal 0–7: which of 8 DIG channels this word represents |
| 0 | **PATTERN MATCH** | Set if Router's nth pattern register matched (n = CHAN_CHECK) |

> ⚠️ **Timestamp and CC Energy byte/nibble ordering:** TIMESTAMP[7:0] (low byte) is in bits 55:48; TIMESTAMP[15:8] (high byte) is in bits 47:40 — i.e. the low byte is transmitted first (higher bit position) and the high byte second (lower bit position). Same pattern for CC Energy: [3:0] at bits 39:36, [15:4] at bits 35:24. The source explicitly flags this: "yes, there's a swap". ✅ verified 2026-04-16 — `router to master format notes.txt` (SVN): exact summary table at end of file

---

## Build Artifacts

| File | Description |
|------|-------------|
| `Work13_4/router_top.bit` | Production bitfile (current) |
| `MAIN_FPGA/Work13_4/router_top.bit` | Earlier version bitfile |
| `MAIN_FPGA/Work13_4/experimental_router_top.bit` | Experimental variant |

---

## See Also

- `knowledgeBase/fpga.md` — System-level overview: trigger hierarchy, throttle mechanism, SERDES link summary
- `knowledgeBase/deep_fpga_DIG.md` — DIG firmware: upstream multiplicity bits the RTRG receives, downstream command frames DIG acts on
- `knowledgeBase/deep_fpga_MTRG_MAIN.md` — MTRG firmware: trigger algorithms consuming RTRG multiplicity data
- `knowledgeBase/ttcl.md` — TTCL: frame 12 (inter-trigger) and frame 14 (remote trigger) that RTRG replaces with null before forwarding to DIG
- `knowledgeBase/connectors.md` — RTRG connector pinouts: 125-pin SERDES links, NIM I/O, CPLD ribbons
- `knowledgeBase/260E_trigger_scheme.md` — Deep dive into RTRG 0x260E trigger scheme: `chan_in.vhd` serial reception + bit alignment, `router_data_path.vhd` multiplicity aggregation, X/Y plane maps, Link-L output format; verified against VHDL source
- `knowledgeBase/vhdl/RTRG_chan_in.md` — `chan_in.vhd` plain-English analysis: 18-bit SERDES word decoding, 640 ns DPRAM delay alignment, discriminator bit extraction, CLEAN_DIRTY register modes
- `knowledgeBase/vhdl/RTRG_disc_mach.md` — `disc_mach.vhd` analysis: discriminator classifier (clean/dirty/BGO-only), event tagging logic
- `knowledgeBase/vhdl/RTRG_overlap_mach.md` — `overlap_mach.vhd` analysis: trigger overlap and hold-off state machine (stretches CLEAN/DIRTY pulses into HAVE_CLEAN/HAVE_DIRTY assertion windows)
- `knowledgeBase/vhdl/RTRG_router_data_path.md` — `router_data_path.vhd` analysis: Link-L multiplicity aggregation, data forwarding to MTRG
- `knowledgeBase/vhdl/RTRG_top.md` — `TOP.VHD` analysis: top-level RTRG block wiring, port map, all sub-module instantiation

---
*Source: `DGS_tools_pack/raw_FPGA/Rtr4704*/` — VHDL source + bitfiles. Created: 2026-04-05.*
