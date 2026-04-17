# MAIN_FPGA — Master Trigger Main FPGA (ISE)

## Table of Contents

- [Target Device](#target-device)
- [Role](#role)
- [Source Files](#source-files)
  - [Top Level](#top-level)
  - [Master State Machine](#master-state-machine)
  - [SERDES Reception & Links](#serdes-reception--links)
  - [Input Channel Processing](#input-channel-processing)
  - [Trigger Algorithms](#trigger-algorithms)
  - [Hit Summation & Multiplicity](#hit-summation--multiplicity)
  - [TDC & Timestamp](#tdc--timestamp)
  - [Register Interface & VME](#register-interface--vme)
  - [I/O & Support](#io--support)
  - [Monitoring](#monitoring)
- [Architecture](#architecture)
  - [Signal Flow](#signal-flow)
  - [Clock Domains](#clock-domains)
  - [Command Frame Timing](#command-frame-timing)
  - [TAC-II / TDC](#tac-ii--tdc)
  - [VME Register Map](#vme-register-map-partial)
- [IP Cores](#ip-cores)
- [Build Artifacts](#build-artifacts)

## Target Device

| Field | Value |
|-------|-------|
| Family | Virtex-4 |
| Part | xc4vlx80 |
| Package | ff1148 |
| Speed Grade | -11 |
| Tool | Xilinx ISE 13.4 |
| Project File | `Firmware/MAIN_FPGA/trunk/Work13_4/Work13_4.xise` |
| Top Entity | `trigger_top` |
| Bitfile | `Firmware/MAIN_FPGA/trunk/Work13_4/trigger_top.bit` |

## Role

The main FPGA is the core of the Master Trigger. It:
1. Receives digitizer summary data from up to 8 Routers via SERDES links (A-H)
2. Runs up to 8 independent trigger algorithms ✅ verified 2026-04-10 — `top.vhd:L2456–3103` (20220705 tag)

**The 8 algorithms (EPICS PV enable bits A–H):**

| # | VHDL label | Component | Source | EPICS enable |
|---|-----------|-----------|--------|--------------|
| 1 | `TRIG_LOGIC1` | `cpld_trig` | CPLD fast strobe (coarse multiplicity) | `EN_ALGO5` |
| 2 | `TRIG_LOGIC2` | `sum_hits_X` | Sum X — X-plane multiplicity | `EN_SUM_X` |
| 3 | `TRIG_LOGIC3` | `sum_hits_X` | Sum Y — Y-plane multiplicity (note: reuses `sum_hits_X` entity with Y sum/threshold inputs; trigger type = 03) | `EN_SUM_Y` |
| 4 | `TRIG_LOGIC4` | `sum_hits_XY` | Sum XY — X+Y coincidence | `EN_SUM_XY` |

✅ verified 2026-04-12 — `MasterTrigger/20220705/Source/top.vhd:L2456,2503,2554,2603` (TRIG_LOGIC1=cpld_trig, 2=sum_hits_X, 3=sum_hits_X, 4=sum_hits_XY); EN PVs confirmed `ioc/db/MTrigUser.template:L33535-33575`
| 5 | `TRIG_LOGIC5` | `cpld_trig` | CPLD fast strobe **edge-detected** (`xxxxFAST_STROBE`) — coincidence trigger (selected via `ALGO_5_SELECT`). PV name `EN_MAN_AUX` is misleading; this is the coincidence/fast-strobe algo. ✅ verified 2026-04-10 — `top.vhd:L2651,L2666,L4153` (20220705 tag) | `EN_MAN_AUX` |
| 6 | `TRIG_LOGIC6A/B` | `GITMO_TRIGGER` (6A) / remote (6B) | Link L: GITMO or remote trigger (muxed via `TRIG_LOGIC_6_ALGO_SEL` + `LINK_L_IS_TRIGGER_TYPE`) | `EN_LINK_L` | ✅ verified 2026-04-11 — `top.vhd:L2778,L2833,L939,L1104` (20220705): `TRIG_LOGIC6A : GITMO_TRIGGER`, `TRIG_LOGIC6B : REMOTE_MASTER_TRIG_SUPPORT`; `LINK_L_IS_TRIGGER_TYPE <= TRIG_ALGO_MUX_SEL_REG(0)` |
| 7 | `TRIG_LOGIC7` | `REMOTE_MASTER_TRIG_SUPPORT` | Link R: remote master trigger | `EN_LINK_R` | ✅ verified 2026-04-11 — `top.vhd:L2896,L2899` (20220705 tag): comment "TRIG_LOGIC7 will be remote triggers coming in from link R", instantiates `REMOTE_MASTER_TRIG_SUPPORT` |
| 8 | `TRIG_LOGIC8A/B` | `MYRIAD_TRIGGER` (8A) / remote (8B) | Link U: MyRIAD or remote trigger (muxed via `LINK_U_IS_TRIGGER_TYPE`) | `EN_MYRIAD_LINK_U` | ✅ verified 2026-04-11 — `top.vhd:L3038-3106` (20220705 tag): `TRIG_LOGIC8A : MYRIAD_TRIGGER`, `TRIG_LOGIC8B : REMOTE_MASTER_TRIG_SUPPORT`; `LINK_U_IS_TRIGGER_TYPE <= TRIG_ALGO_MUX_SEL_REG(2)` at L1106 |

✅ verified 2026-04-10 — `top.vhd:L2456,L2503,L2554,L2603,L2651,L2778,L2897,L3042` (20220705 tag); comments confirm per-link rules for algos 6/7/8.

### Coincidence Trigger Mask (`ALGO_5_SELECT=1` mode)

When `ALGO_5_SELECT=1` (coincidence trigger), the `reg_COINC_TRIG_MASK` register selects which pairs of algorithms must fire together. Each bit = one algorithm-pair select:

| Bits | PV names | Meaning |
|------|----------|---------|
| 0–3 (0x01–0x08) | `COINC_TRIG_MASK_A1–A4` | Group A, algos 1–4 |
| 4–6 (0x10–0x40) | `COINC_TRIG_MASK_A6–A8` | Group A, algos 6–8 |
| 8–11 (0x100–0x800) | `COINC_TRIG_MASK_B1–B4` | Group B, algos 1–4 |
| 12–14 (0x1000–0x4000) | `COINC_TRIG_MASK_B6–B8` | Group B, algos 6–8 |

✅ verified 2026-04-13 — `backup_Generated_top.vhd:L1688-1701` (bit assignments confirmed: A1=bit0, A2=bit1, ..., B1=bit8, B2=bit9, ... B8=bit14; note bit7 unused — no A5/B5). Register comment in `registers.vhd` says "secondary ts offset" but signal names and bit assignments confirm coincidence mask function.

Algo 5 itself (the CPLD/coincidence trigger) is **absent** from both groups — it cannot coincide with itself. To disable all coincidence selection: `caput VME32:MTRG:reg_COINC_TRIG_MASK 0`. ✅ verified 2026-04-12 — `MTrigUser.template:L1385-1495` (A1=0x01, A2=0x02, A3=0x04, A4=0x08, A6=0x10, A7=0x20, A8=0x40; B1=0x100 … B8=0x4000; A5/B5 absent).

3. Collects trigger decisions and issues synchronized command frames back to all Routers
4. Supports chaining to other Master Triggers via inter-trigger links (L, R, U)
5. Generates precise event timestamps via a vernier TDC (TAC-II, ~30 ps resolution)
6. Provides VME register access for configuration and status

## Source Files

**Location:** `Firmware/MAIN_FPGA/trunk/Source/`

### Top Level
| File | Description |
|------|-------------|
| `Generated_top.vhd` | Top-level entity (`trigger_top`) — auto-generated wrapper |
| `last_manual_top.vhd` | Backup of last hand-edited top |
| `trigger_top_comp_defs.vhd` | Component declarations for all submodules |
| `trigger_data_types.vhd` | Custom array type definitions (JTA_8X16_Array, etc.) |

### Master State Machine
| File | Description |
|------|-------------|
| `mstr_mach.vhd` | Master state machine — generates 20 command frames per 2 µs cycle (at 50 MHz); coordinates trigger decision collection and dispatch to Routers |

### SERDES Reception & Links
| File | Description |
|------|-------------|
| `SERDES_RX_Mach_R2.vhd` | Receives command data on inter-trigger links L, R, U; decodes sync, DetStatus, TrigDecision, and Frame12/14/15/16/17 frames |
| `link_init.vhd` | SERDES link initialization and lock acquisition |
| `link_tx_block.vhd` | SERDES transmitter for inter-trigger links L, R, U |
| `link_lru_rx.vhd` | Link L/R/U receiver clock domain synchronization |
| `DCBAL_in.vhd` | DC-balance removal (converts 18-bit encoded to 16-bit data) |
| `DCBAL_in_nofifo.vhd` | DC-balance removal variant without FIFO |
| `dc_balance_mach.vhd` | DC-balance encoder state machine |

### Input Channel Processing
| File | Description |
|------|-------------|
| `eight_mt_channel.vhd` | Wrapper instantiating 8 `mt_input_channel` units; handles link masking and X/Y multiplicity sums |
| `mt_input_channel.vhd` | Per-channel processing: DC-balance removal, FIFO handoff between SERDES clock and master clock, link status monitoring |

#### Per-Link Mask Registers (ILM / XLM / YLM)

Each MTRG input link (A–H, plus L/R/U for remote systems) has **three independent mask bits** that control how it participates in trigger processing:

| Mask | Register | VME Addr | EPICS PV | Effect when set (=1) |
|------|----------|----------|----------|----------------------|
| **ILM** — Input Link Mask | `reg_INPUT_LINK_MASK` | `0x0800` | `VMExx:MTRG:ILM_<link>` | Completely block this link — no data reception, no participation in sums |
| **XLM** — X-Plane Link Mask | `reg_X_PLANE_LINK_MASK` | `0x02C0` | `VMExx:MTRG:XLM_<link>` | Exclude this link from X-plane multiplicity sum (Sum-X trigger, algo 2) |
| **YLM** — Y-Plane Link Mask | `reg_Y_PLANE_LINK_MASK` | `0x02C4` | `VMExx:MTRG:YLM_<link>` | Exclude this link from Y-plane multiplicity sum (Sum-Y trigger, algo 3) |

From `mt_input_channel.vhd`:
- X-sum contribution gated by: `CHANNEL_MASK OR X_PLANE_MASK OR (NOT SERDES_LOCK)`
- Y-sum contribution gated by: `CHANNEL_MASK OR Y_PLANE_MASK OR (NOT SERDES_LOCK)`

So a link contributes to the X or Y sum only if it is **not ILM-masked**, **not XLM/YLM-masked**, and **SERDES is locked**. ✅ verified 2026-04-15 — `eight_mt_channel.vhd:L165-166`, `mt_input_channel.vhd:L121,L131`

#### Link Type Mask Policy (`trig_setup_Stage1.sh`)

During trigger setup, the Stage 1 script (`ANLDAQ/gui/scripts/trig_setup_Stage1.sh`) applies ILM/XLM/YLM based on the **type** of device connected to each link (from `MT_LINK_MAP` in `SYSTEM_DEFINES.sh`):

| Link Type | ILM | XLM | YLM | Notes |
|-----------|-----|-----|-----|-------|
| `RTR` (Router) | 0 (unmasked) | 0 (unmasked) | 0 (unmasked) | Full participation in all sums |
| `PIXIE` | 0 | 1 | 1 | Receives ILM data but excluded from X/Y sums |
| `DFMA` | 0 | 1 | 1 | Same; PROPAGATE_TRIG_FROM_DFMA sets F3-F7 propagation |
| `DUB` | 0 | 1 | 1 | Same; PROPAGATE_TRIG_FROM_DUB sets F3-F7 propagation |
| `DXA` | 0 | 1 | 1 | Same; PROPAGATE_TRIG_FROM_DXA sets F3-F7 propagation |
| `MASKED` | 1 | 1 | 1 | Fully masked — no data, no sums |

Links L, R, and U are **excluded from XLM/YLM processing** — the script explicitly skips XLM/YLM `caput` calls for these links (they connect to remote master triggers, not routers). ✅ verified 2026-04-15 — `trig_setup_Stage1.sh:L217-228`

### Trigger Algorithms
| File | Description |
|------|-------------|
| `trig_collect.vhd` | Multiplexes 8 algorithm outputs into trigger decision FIFO for the master state machine |
| `GITMO_TRIGGER.vhd` | Trigger algorithm for GRETINA digitizer channels |
| `MYRIAD_TRIGGER.vhd` | Trigger algorithm for MyRIAD auxiliary detector |
| `GITMO_RCV_MACH.vhd` | Receiver state machine for GRETINA data frames |
| `MYRIAD_RCV_MACH.vhd` | Receiver state machine for MyRIAD data frames |
| `trig_algo_support.vhd` | Common infrastructure shared by trigger algorithms |
| `remote_trig_support.vhd` | Remote trigger propagation via inter-trigger links |
| `local_trig_coinc.vhd` | Local coincidence logic |

### Hit Summation & Multiplicity
| File | Description |
|------|-------------|
| `sum_hits_X.vhd` | X-plane hit summation |
| `sum_hits_XY.vhd` | X+Y combined plane summation |
| `calc_total_sum.vhd` | Total multiplicity calculation |
| `overlap_mach.vhd` | Overlap detection state machine |
| `data_compressor.vhd` | Event data compression |
| `chan_fifo_write_ctl.vhd` | Channel FIFO write control |

### TDC & Timestamp
| File | Description |
|------|-------------|
| `tdc_chain_cont.vhd` | TDC chain controller — chains vernier stages for ~1 ns resolution |
| `tdc_unit_cont.vhd` | Individual TDC unit control |
| `pos_finder.vhd` | Hit position finder |
| `jta_vernier_pos_finder.vhd` | Vernier position detection |
| `vernier_pos_finder.vhd` | Vernier refinement logic |
| `timestamp.vhd` | Timestamp generation and synchronization |
| `sync_capture_controller.vhd` | System-wide time capture controller |
| `sync_capture_counter.vhd` | Counter for sync events |

### Register Interface & VME
| File | Description |
|------|-------------|
| `registers.vhd` | VME register map (addresses 0x0000–0x08FC) — ~150 read/write registers for status, trigger control, algorithm configuration |

### I/O & Support
| File | Description |
|------|-------------|
| `AUX_IO.VHD` | Front panel auxiliary I/O mux, encoder, SSI |
| `LED_CTL.VHD` | LED indicator control |
| `NIM_Delay.vhd` | Programmable delay for NIM inputs |
| `Delay_Line.vhd` | Generic delay element |
| `slow_clocks.vhd` | Decade clock generation (10 MHz down to 1 Hz) |
| `cpld_trig.vhd` | CPLD trigger pulse generation |
| `pipeline_unit.vhd` | Pipeline delay stage |
| `jta_odelay.vhd` | I/O delay primitives |

### Monitoring
| File | Description |
|------|-------------|
| `trig_mon_collect.vhd` | Trigger monitor data collection |
| `trig_monitor_controller.vhd` | TDC monitor FIFO synchronization |

## Architecture

### Signal Flow

```
Routers × 8 (Links A–H, 18-bit SERDES, DC-balanced)
    ↓
eight_mt_channel → mt_input_channel ×8
    ↓  (16-bit decoded data)
Trigger Algorithms ×8
  ├── GITMO_TRIGGER   (Ge detector channels)
  ├── MYRIAD_TRIGGER  (auxiliary detector)
  └── Algorithms 3–8  (energy sums, patterns, coincidences)
    ↓
trig_collect FIFO
    ↓
mstr_mach  →  Command frames → All Routers (Links A–H)

Remote Masters ↔ Links L, R, U ↔ SERDES_RX_Mach_R2 / link_tx_block
```

### Clock Domains

| Clock | Frequency | Source | Used For |
|-------|-----------|--------|----------|
| mclk | 50 MHz | ICS581 mux (local osc or Link L RX) ✅ verified 2026-04-10 — `top.vhd:L179` (`CLK_SRC_SEL: out std_logic -- pin AK19, drives SELA of ICS581 clock mux`); `top.vhd:L239` (`xLOGIC_CLOCK: output of buffer driven by switched clock from ICS581`) | Main logic, state machines |
| mclk_2x | 100 MHz | DCM from mclk | TDC, high-speed counters, FIFOs |
| RCLK (per link) | 50 MHz | SERDES receiver | DC-balance removal, link sync |
| VME_CLOCK | 50 MHz | Dedicated oscillator | VME register interface |

### Command Frame Timing

- Cycle period: **2 µs** (20 frames × 5 words × 20 ns at 50 MHz)
- Frame period: **100 ns** (5 words × 20 ns)
- Generated by `mstr_mach.vhd`; transmitted to all Routers on Links A–H (18-bit DC-balanced SERDES)
- Routers forward the stream to their 8 Digitizers on Links A–H, stripping Frames 12 and 14

#### Frame Summary

| Frame | Name | Transmitted to |
|-------|------|----------------|
| 1 | Sync | Routers + Digitizers |
| 2 | Detector Status | Routers + Digitizers |
| 3–10 | Trigger Decision ×8 | Routers + Digitizers |
| 11 | Spare (null) | Routers + Digitizers |
| 12 | Router Command | Routers only (stripped before Digitizers) |
| 13 | GRETINA Demand Slow Data | Routers + Digitizers |
| 14 | Router Command | Routers only (stripped before Digitizers) |
| 15 | Async Command | Routers + Digitizers |
| 16 | Sync Capture | Routers + Digitizers |
| 17 | Auxiliary Detector | Routers + Digitizers |
| 18–19 | Spare (null) | Routers + Digitizers |
| 20 | End-of-Cycle | Routers + Digitizers |

The **5th word of every frame** carries `VETO[9:0]` in bits [9:0] — a per-channel veto mask applied in real time at each Digitizer.

#### Frame 1 — Sync

| Word | Bits | Field | Description |
|------|------|-------|-------------|
| 1 | 15 | ISYNC | 1 = Imperative Sync (all timestamp counters reset); 0 = normal sync |
| 1 | 14:9 | — | Reserved (0) |
| 1 | 8 | Sync marker | Always 1 |
| 1 | 7:0 | Rollover | 0xFF = timestamp counter rolled over this cycle; 0x00 = no rollover |
| 2 | 15:0 | SYNC_TS[47:32] | Bits 47–32 of the 48-bit master timestamp |
| 3 | 15:0 | SYNC_TS[31:16] | Bits 31–16 |
| 4 | 15:0 | SYNC_TS[15:0] | Bits 15–0 |
| 5 | 9:0 | VETO[9:0] | Per-channel veto mask |

Valid Word 1 values: `0x0100` (normal), `0x01FF` (rollover), `0x8100` (ISYNC), `0x81FF` (ISYNC + rollover).

#### Frame 2 — Detector Status

| Word | Bits | Field | Description |
|------|------|-------|-------------|
| 1 | 15:0 | DET_DATA | Detector state snapshot (e.g. target wheel position) |
| 2 | 15:0 | XTRA_DATA | Extra state data |
| 3–4 | — | — | Reserved |
| 5 | 9:0 | VETO[9:0] | Per-channel veto mask |

#### Frames 3–10 — Trigger Decision (×8 slots)

Eight consecutive frames, each carrying one trigger decision slot. Unused slots contain null (`0xAAAA`). Multiple triggers within one 2 µs cycle occupy separate frames.

| Word | Bits | Field | Description |
|------|------|-------|-------------|
| 1 | 15:11 | Frame type | `10101` = NULL (no trigger); `01010` (0x5x) = local trigger; `01100` (0x6x) = remote trigger | ✅ verified 2026-04-11 — `SERDES_RX_Mach.vhd:L735` (20230809): "nulls are 0xAAAA. Local triggers are, by fiat, 0x5nxx (n=0-7). Remote trigs are 0x6nXX" |
| 1 | 10:8 | Trigger type | 0 = manual/aux/RAM; 1 = sumX/local coinc; 2 = sumY; 3 = sumXY; 4 = CPLD/fast strobe |
| 1 | 7:0 | Trigger data | Algorithm-specific payload |
| 2 | 15:0 | TRIG_TS[47:32] | Physics timestamp of the triggering event, bits 47–32 |
| 3 | 15:0 | TRIG_TS[31:16] | Bits 31–16 |
| 4 | 15:0 | TRIG_TS[15:0] | Bits 15–0 |
| 5 | 9:0 | VETO[9:0] | Per-channel veto mask; Digitizer asserts TRIG_FLAG on this word |

#### Frame 11 — Spare

All 5 words: `0xAAAA`, `0xAAAA`, `0xAAAA`, `0xAAAA`, `0x0000`. Checked by Digitizer state machine for lock integrity.

#### Frame 12 — Router Command

Used by MTRG to command Router/Data Generator counters and FIFOs. Content is stripped by the Router before forwarding; Digitizers always see `0xAAAA` null pattern.

#### Frame 13 — GRETINA Demand Slow Data

Fixed pattern retained for GRETINA digitizer compatibility. No DGS action.
Expected: `0x40FB`, `0xA5A5`, `0x5A5A`, `0xA5A5`, `0xA5A5`.

#### Frame 14 — Router Command

Same as Frame 12. Router strips content; Digitizers see null.

#### Frame 15 — Async Command

Sourced from the `ASYNC_CMD_FIFO` VME register (address `0x08F4`). ✅ verified 2026-04-12 — `registers.vhd:L20` (Vivado trunk): "Address 0x08F4 is reserved for the ASYNC COMMAND FIFO that is writable by VME and read by the master machine."

| Word | Bits | Field | Description |
|------|------|-------|-------------|
| 1 | 15:8 | Command | Command byte (see below) |
| 1 | 7:0 | — | Unused |
| 2 | 15 | Latch set | 1 = enable latch mode for EXTERNAL_DISC |
| 2 | 14 | Latch clear | 1 = disable latch mode |
| 2 | 9:0 | Chan select | Channel mask for EXTERNAL_DISC (one bit per channel) |
| 3 | 15 | Latch set | 1 = set latch (if latch enabled) |
| 3 | 14 | Latch clear | 1 = clear latch |
| 3 | 11:0 | AND-mask | ANDed with `reg_user_package_data` to select target module |
| 4 | 11:0 | OR-mask | ORed with AND result; non-zero = this module is addressed |
| 5 | — | — | EXTERNAL_DISC_FLAG resolved at Digitizer on this word |

Command byte values (bits [15:8] of Word 1):

| Value | Command |
|-------|---------|
| `0x04` | CAL_INJECT — inject calibration pulse |
| `0x08` | LATCH_STATUS — capture status snapshot |
| `0x10` | FRONT_END_RESET — reset front-end electronics |
| `0x18` | RESET_LINKS — reset SERDES links |
| `0x22` | EXTERNAL_DISC — assert/de-assert external discriminator |

#### Frame 16 — Synchronous Capture

| Word | Bits | Field | Description |
|------|------|-------|-------------|
| 1 | 15:8 | Command | Any value ≠ `0xAA` activates capture; `0xAA` = idle |
| 2 | 15:0 | Capture TS[31:16] | Capture start timestamp bits 31–16 |
| 3 | 15:0 | Capture TS[15:0] | Capture start timestamp bits 15–0 |
| 4 | 15:0 | Capture length | Duration of capture window |
| 5 | 15:0 | FIFO delay | Delay from capture TS to FIFO write start; SYNC_CAPTURE_FLAG asserted at Digitizer |

#### Frame 17 — Auxiliary Detector Command

Sourced from `AUX_CMD_FIFO` VME register (`0x08F8`). Content depends on auxiliary detector. Not decoded by DIG main FPGA. Word 5 carries VETO[9:0]. ✅ verified 2026-04-13 — `registers.vhd:L21` (comment: "Address 0x08F8 is reserved for the AUX COMMAND FIFO"); `top.vhd:L586-599` (Frame 17 signals)

#### Frames 18–19 — Spare

Same as Frame 11: `0xAAAA`, `0xAAAA`, `0xAAAA`, `0xAAAA`, `0x0000`.

#### Frame 20 — End-of-Cycle

Fixed pattern, always checked by Digitizer for lock integrity. Any mismatch forces relock.

| Word | Value |
|------|-------|
| 1 | `0xFFFF` |
| 2 | `0x0000` |
| 3 | `0xFFFF` |
| 4 | `0x0000` |
| 5 | `0x5555` |

This sequence is also used for initial lock acquisition (prelock state in Digitizer `SERDES_RX_Mach.vhd`).

### TAC-II / TDC

TAC-II is the trigger system's Time-to-Digital Converter, implemented entirely inside the FPGA fabric using Xilinx Virtex-4 carry-chain primitives. It measures the arrival time of a reference signal (e.g. an RF clock edge on NIM_IN2) relative to the trigger. No external TAC chip is used.

**Resolution:** The trigger user manual states **"better than 300 ps" (single shot, no averaging)** — this is the coarse NIM-to-clock-edge measurement via the Rev D NIM receiver. The vernier carry chain provides sub-ns refinement at **50 ps per tap** (fixed assumption for most applications; actual tap delays vary slightly). ✅ verified 2026-04-05 — TAC.pdf: "simply assuming a fixed delay of 50ps per tap will suffice" + "multiplying that by 50ps (0.050ns)"
- `Trigger user manual 20140901.pdf` p.6: "better than 300 ps accuracy (single shot, no averaging)" ✅ verified 2026-04-05
- `raw_FPGA/Firmware_Tags/MasterTrigger/20220705/Source/tdc_chain_cont.vhd`: 4-phase vernier chains (0°/90°/180°/270°), 64-bit per chain ✅ verified 2026-04-05

#### Physical Input: RF Clock → NIM IN 2

**The RF clock (or any timing reference) must be connected to the NIM IN 2 input** on the MTRG front panel:

```
RF clock → LEMO cable → NIM IN 2 (bottom row, MTRG front panel)
                              ↓
                         MTRG FPGA (top.vhd)
                              ↓
              NIM_IN2 split into:
                ├─ TDC_IN_NIM_IN2   → TAC-II carry chain (BIT_IN)
                └─ NON_TDC_NIM_IN2  → other logic (non-TDC use)
                              ↓
                    TAC-II → ~30 ps resolution timestamp
```

**EPICS PV:** `VME99:MTRG:EN_NIM2_DELAY` — enable delay on NIM IN 2 if signal timing adjustment needed.

> ⚠️ Do not confuse with NIM IN 1 (auxiliary trigger input). NIM IN 2 is exclusively the TDC/veto input.
> See `connectors.md` §3 for the full physical connector layout.

#### Signal Input (Firmware)

The signal to be timed (`BIT_IN`) enters the carry chain at `DELAY_CHAIN_EVEN(0)`. In `top.vhd` this is routed from `NIM_IN2`, split as `TDC_IN_NIM_IN2` vs `NON_TDC_NIM_IN2`.

#### Vernier Delay Chain (`tdc_unit2.vhd`)

The signal propagates through a 64-element carry chain built from `MUXCY_L` primitives. Each CLB slice provides two tap points (odd and even), giving 128 sample positions at a nominal spacing of ~70 ps per `MUXCY` stage (within-slice spacing ~10 ps):

```
BIT_IN → MUXCY(Even0) → MUXCY(Odd0) → MUXCY(Even1) → ... → bit 63
                  ↓               ↓               ↓
               XORCY            XORCY           XORCY
                  ↓               ↓               ↓
               FF1d(0)          FF2d(0)         FF2d(1)   ← captured at 250 MHz rising edge
```

The result is a thermometer code: the position of the `1→0` transition gives fine time within the current clock cycle. The `MAXDELAY` constraint is set to `100 ps` to keep tap spacing uniform.

#### Four-Phase Sampling (`tdc_short_chain.vhd`)

`tdc_short_chain` instantiates four copies of `tdc_unit2`, each driven by the same 250 MHz clock at a different phase:

| Chain | Clock Phase | Counter |
|-------|------------|---------|
| TDC_A | 0° | `TDC_COARSE_COUNT` (16-bit) — increments every 4 ns |
| TDC_B | 90° (+1 ns) | `TDC_FINE_COUNT_B` (4-bit) |
| TDC_C | 180° (+2 ns) | `TDC_FINE_COUNT_C` (4-bit) |
| TDC_D | 270° (+3 ns) | `TDC_FINE_COUNT_D` (4-bit) |

Sampling with clocks at 0°/90°/180°/270° subdivides the 4 ns clock period into 1 ns slices. Combined with the sub-ns tap resolution of the vernier chain, this yields ~30 ps effective resolution.

#### Stop-When-Hit Operation

All flip-flops in `tdc_unit2` have `CE = TDC_RUN`. At reset, all flops are preset to `'1'` (PRE input), so `TDC_RUN` is high and the chain is live. When the signal edge propagates past bit 3, `TDC_RUNx(3)` goes low through a loopback XOR gate, freezing the entire chain. The thermometer pattern is then stable for readout.

A continuously-running variant (`tdc_unit_cont.vhd`) holds CE at `'1'`; external logic must sample the vernier every clock. Used for a separate operating mode.

#### Arming Sequence

Arming is driven by `AUTO_READ_TDC` in `top.vhd`, a delayed copy of `TRIGGER_OCCURRED`:

1. Trigger fires → `TRIGGER_OCCURRED` pulses → `AUTO_READ_TDC` asserted after programmable delay
2. `AUTO_READ_TDC` → `TDC_RESET` — a one-cycle pulse issued to each chain independently (edge-detected in each phase's 250 MHz domain via `SAMP_RESET(0..3)`)
3. All FDCPE flops preset → `TDC_RUN` goes high → chain is armed and waiting
4. Reference signal arrives → propagates down carry chain → chain locks at the bit position where the 250 MHz clock edge next occurs

#### Timing Summary

```
Coarse resolution : 4 ns   (250 MHz period, counted by TDC_COARSE_COUNT)
Phase subdivision : ÷ 4    (0°/90°/180°/270°)  → 1 ns per phase step
Vernier tap delay : ~70 ps per MUXCY stage; ~10 ps within same CLB slice
Effective result  : ~30 ps (limited by tap-to-tap uniformity across silicon)
MAXDELAY constraint: 100 ps (forces router to keep chain compact)
```

#### Data Path — Mon FIFO Triple-Buffer

After the chain locks, the TDC data is packetized into a "trigger event" for VME/IOC readout via three FIFOs:

| FIFO | Width | Content |
|------|-------|---------|
| MON7A | 16-bit (async) | Raw TDC vernier data written by `tdc_chain_cont` / data compressor |
| MON7 | 16-bit EVENT_FIFO | Trigger monitor words from selected `TRIG_MON_FIFO` (one per trigger algorithm) |
| MON7B | 16-bit (async) | Merged output packet built by `trig_mon_collect` state machine |

Source: `trig_mon_collect.vhd` (verified 2026-04-04)

#### MON7B Packet Format (16-bit words, IOC reads these over VME → TCP)

```
Word  0:        0xAAAA                     ← packet header delimiter (written in IDLE state) ✅ verified 2026-04-08 — trig_mon_collect.vhd:L227 (`TRIG_MON_COLLECT_FIFO_OUT <= X"AAAA"` in IDLE)

Words 1..N:     TRIG_MON_FIFO words        ← N = NUM_TRIG_WORDS (nominally constant)
                (trigger monitor data from selected algorithm's shadow FIFO)

Word  N+1:      [PACKET_LENGTH[15:10] | USER_PACKAGE_DATA[9:0]]
                  PACKET_LENGTH = NUM_TRIG_WORDS + NUM_TDC_WORDS (6-bit field in [15:10])
                  USER_PACKAGE_DATA = reg_user_package_data VME register (10-bit, [9:0])
                  → decoded by class_TDC.h as userRegister = data[4] & 0xFFFF

Words N+2..end: TDC vernier words          ← NUM_TDC_WORDS words from TDC_DATA_FIFO
                (four-phase counters + vernier AB/CD)
```

**If TDC data is skipped** (`SKIP_TDC_DATA = MON7_FILL_CTL_REG(15)` VME register bit set): ⚠️ Note: there is no countdown timeout — `CHECK_TDC` state polls `LATCHED_TDC_READY` indefinitely until ready or `SKIP_TDC_DATA` is set. The `TDC_IGNORED` state fills TDC word slots with fake data:
```
X"000" & PULL_COUNT
```
✅ verified 2026-04-08 — `trig_mon_collect.vhd:L281-295` (20220705 tag): no ALLOWED_TDC_LATENCY countdown; SKIP_TDC_DATA sourced from `MON7_FILL_CTL_REG(15)` per `top.vhd:L4698`. Original ~960 ns estimate was incorrect.
This is detectable in software — `class_TDC.h` checks for the `0x1006/1005/1004/1003` counter pattern as one specific trash-data sentinel.

**Timeout window:** 0x60 = 96 clock cycles @ 100 MHz = **960 ns** (updated 2025-07-22 from 0x40 = 640 ns). ✅ verified 2026-04-14 — `MTRG/MAIN_FPGA/trunk/Source/trig_mon_collect.vhd:L271` (`ALLOWED_TDC_LATENCY <= X"60"; -- changed from X"40" 20250722`). Note: Vivado trunk does NOT have this countdown — it polls indefinitely; only ISE (Virtex-4) trunk has `ALLOWED_TDC_LATENCY`.

**FIFO selection:** `TDC_TRIG_SEL_MASK` (one-hot, max 1 bit set) selects which algorithm's `TRIG_MON_FIFO` feeds the packet. All other algorithm FIFOs are continuously drained (RE held high) to prevent overflow.

**Packet boundary:** `WROTE_MON7_EVENT` pulses for one 100 MHz clock on the last WE of each packet (either `PULL_TDC_REST` or `TDC_IGNORED` when `PULL_COUNT = 1`).

`TDC_TRIG_SEL_REG` (`top.vhd:691`) selects which trigger algorithm(s) save TDC results. A bitmask (`TDC_TRIG_SEL_MASK`) enforces that only one algorithm is linked to the TDC at a time.

### VME Register Map (partial)

| Address | Register |
|---------|----------|
| 0x0100 | LOCK_BUS (SERDES lock status) |
| 0x0120 | MSTR_MACH_STATE |
| 0x0158 | CODE_DATE (firmware build date) |
| 0x015C | CODE_REVISION |
| 0x01B0 | SYSTEM_THROTTLE_MAP |
| 0x0200–0x027C | Algorithm config (delay, overlap, prescale, veto) |
| 0x0240–0x024C | Remote trigger settings |
| 0x0200 | MISC_STAT (see bit map below) |
| 0x0258 | MISC_STAT2 (see bit map below) |
| 0x08F4 | ASYNC_CMD_FIFO (Frame 15 commands from VME) |
| 0x08F8 | AUX_CMD_FIFO |

### MISC_STAT Register Bit Map (MTRG)

EPICS PV: `VME$(CRATE):$(BOARD):reg_MISC_STAT_RBV` (16-bit read-only)

| Bit | Mask | PV Suffix | Description |
|-----|------|-----------|-------------|
| 0 | 0x0001 | `xNIM_IN1_RBV` | **=1 when NIM input 1 is high (signal present).** Reflects the raw instantaneous logic level of the NIM IN1 front-panel input. |
| 1 | 0x0002 | `DLYD_TDC_IN_NIM_IN2_RBV` | **=1 when NIM input 2 is high (signal present).** Same as bit 0 but for NIM IN2. Formerly named `xNIM_IN2`; repurposed as the delayed TDC trigger input in later firmware. |
| 2 | 0x0004 | `TIMESTAMP_ROLLOVER_RBV` | **=1 when the 48-bit timestamp counter has rolled over** (wrapped from max back to 0). Normally 0 during a run; a 1 here indicates the run has been going long enough to overflow the counter (≈3.3×10¹³ clock ticks at 100 MHz ≈ 388 days). |
| 3 | 0x0008 | `FRAME_12_PENDING_RBV` | **=1 when a Frame 12 command is queued but not yet sent.** Frame 12 is the internal trigger command sent to RTRGs (routers replace it with Null for digitizers). A stuck high value here may indicate the TTCL command pipeline is stalled. ✅ verified 2026-04-13 — `mstr_mach.vhd:L62-63` (FRAME_12_REQ_FLAG/FRAME_12_SENT_FLAG ports) |
| 4 | 0x0010 | `FRAME_14_PENDING_RBV` | **=1 when a Frame 14 command is queued but not yet sent.** Frame 14 targets Digitizer Tester boards and MγRIAD. Normally pulses briefly; stuck high = pipeline stall. ✅ verified 2026-04-13 — `mstr_mach.vhd:L71-72` (FRAME_14_REQ_FLAG/FRAME_14_SENT_FLAG ports) |
| 5 | 0x0020 | `FRAME_16_PENDING_RBV` | **=1 when a Frame 16 synchronous capture command is queued but not yet sent.** Frame 16 is the DGS system-wide synchronous capture command (e.g. snapshot of scalers/counters). |
| 6 | 0x0040 | `FRAME_17_PENDING_RBV` | **=1 when a Frame 17 auxiliary detector command is queued but not yet sent.** Frame 17 targets non-digitizer front-end devices. |
| 7 | 0x0080 | — | Reserved — always 0. |
| 11:8 | 0x0F00 | `LINK_INIT_STATE_RBV` (mbbi) | **SERDES link initialization state machine status.** Value indicates current state: 0=INIT (reset), 1=EN_SERDES (enabling transceivers), 2=SYNC (synchronizing), 3=WAIT_LOCK (waiting for all links to lock), 4=ALL_LOCK (all links locked, normal running), 5=ACKED (lock acknowledged), 6=ERROR (lock failed). Expected value during normal operation: **4 (ALL_LOCK)**. |
| 12 | 0x1000 | `ANY_TRIGGER_VETO_RBV` | **=1 when at least one trigger veto source is currently active** (e.g. buffer full, throttle, manual veto). While 1, the MTRG suppresses trigger acceptance. Added 2016-04-01. |
| 13 | 0x2000 | — | Reserved — always 0. |
| 14 | 0x4000 | `ALL_LOCKED_RBV` | **=1 when all downstream SERDES links (to RTRGs) are in lock.** This is the normal healthy state for a running system. If 0 during a run, data from one or more RTRGs is not being received. |
| 15 | 0x8000 | `LOCK_ERROR_RBV` | **=1 if any SERDES link lost lock at any point since last reset (latched/sticky).** Unlike bit 14, this bit does not clear when lock is re-acquired — it stays high until explicitly reset. Normally 0; any 1 here means a link glitch occurred and should be investigated. |

✅ verified 2026-04-12 — `FPGA/MTRG/Firmware/VIVADO_MAIN_FPGA/trunk/Source/top.vhd:L1306–1319,L2140–2142` + `ioc/db/MTrigUser.template:L35283–35383`

### MISC_STAT2 Register Bit Map (MTRG)

EPICS PV: `VME$(CRATE):$(BOARD):reg_MISC_STAT2_RBV` (16-bit read-only). Provides access to SPARE_LVDS and auxiliary status bits.

| Bit | Mask | PV Suffix | Description |
|-----|------|-----------|-------------|
| 7:0 | 0x00FF | — | **`xSPARE_LVDS[7:0]`** — 8 spare LVDS input bits from the front panel. FCO 200/201 pin-strapped as inputs (added 2009-08-26). Useful for monitoring external LVDS signals or debugging. Normally all 0 if nothing is connected. |
| 9:8 | 0x0300 | — | **`xXXLVDS[1:0]`** — two additional LVDS input bits (added 2009-08-26). Same purpose as xSPARE_LVDS. |
| 10 | 0x0400 | `GLOBAL_THROTTLE_REQUEST_RBV` | **=1 when at least one RTRG is asserting a global throttle request** (i.e. its local buffers are near full and it wants the MTRG to pause triggering). The MTRG uses this to engage system-wide throttle. Normally 0 during healthy running; a stuck 1 indicates sustained buffer pressure. |
| 14:11 | 0x7800 | — | Reserved — always 0. |
| 15 | 0x8000 | `L_SM_LOCKED_RBV` | **=1 when the SERDES Link L state machine is locked** (link to the downstream RTRG on the L port). In GITMO mode (added 2011-10-05), reflects GITMO machine lock instead. Expected to be 1 during normal operation; 0 means the L-link is not communicating. |

✅ verified 2026-04-12 — `FPGA/MTRG/Firmware/VIVADO_MAIN_FPGA/trunk/Source/top.vhd:L1324–1341` + `ioc/db/MTrigUser.template:L35473–35483`

## IP Cores

Located in `Firmware/MAIN_FPGA/trunk/Cores/`. XCO-based cores generated by ISE CoreGen:
- Async FIFOs (various widths: 16, 22, 48, 80 bits)
- FWFT FIFO variants
- Dual-port RAMs
- ChipScope ILA cores (80-bit, 64-bit, 1×64/2×64 configurations)

## Build Artifacts

| File | Description |
|------|-------------|
| `Work13_4/trigger_top.bit` | Main production bitfile |
| `Work13_4/GRET_L_trigger_top.bit` | Variant for GRETINA Link L mode |
| `Source/Chipscope/20250205.cpj` | ChipScope project (most recent) |
| `Source/Chipscope/20230927.cpj` | ChipScope project (2023) |

---

## See Also

- `knowledgeBase/fpga.md` — System-level overview: trigger hierarchy, 20-frame cycle, end-to-end timeline
- `knowledgeBase/deep_fpga_MTRG.md` — MTRG overview: device layout (Main FPGA + VME FPGA + CPLD)
- `knowledgeBase/deep_fpga_MTRG_VME.md` — VME FPGA: A32/D32 VME slave, FPGA config manager
- `knowledgeBase/deep_fpga_MTRG_CPLD.md` — CPLD: fast strobe multiplicity logic (~1 µs latency)
- `knowledgeBase/deep_fpga_MTRG_VIVADO.md` — Vivado port: Kintex UltraScale XCK060 differences
- `knowledgeBase/deep_fpga_RTRG.md` — RTRG firmware: multiplicity data the MTRG receives on Link L
- `knowledgeBase/ttcl.md` — TTCL spec: 20-frame downstream command structure generated by the MTRG
- `knowledgeBase/myriad.md` — MγRIAD auxiliary detector interface (connects to MTRG trigger algorithms)
- `knowledgeBase/connectors.md` — MTRG connector pinouts: 125-pin SERDES, NIM I/O, CPLD ribbons, ECL
