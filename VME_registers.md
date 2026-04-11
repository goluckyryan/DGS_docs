# DGS VME Register Map

Complete VME register addresses for all DGS FPGA boards, extracted from asyn driver source code.

**Source files:**
- `VxWorks/dgsDrivers/dgsDriverApp/src/asynDigParams.c` — DIG main FPGA
- `VxWorks/dgsDrivers/dgsDriverApp/src/asynMTrigParams.c` — MTRG main FPGA
- `VxWorks/dgsDrivers/dgsDriverApp/src/asynRTrigParams.c` — RTRG main FPGA
- `VxWorks/dgsDrivers/dgsDriverApp/src/devGVME.c` — VME FPGA (all boards), flash

> All registers are **32-bit** (A32/D32 = Address 32-bit. Data 32-bit). Addresses are **byte offsets** from the board's VME base address.
> Base address formula: `base = slot << 20` (from `devGVME.c`). ✅ verified 2026-04-09 — `devGVME.c:L127` (`base = slot << 20; //In VME64x bits 20 and higher are used to form the base VME address of the card`)
> `VMERead32(bdnum, regaddr)` — `regaddr` is a **byte offset** (divided by 4 internally to get longword address); `bdnum` is the **cardno** (2nd argument of `asynDigitizerConfig`/`asynTrigMasterConfig1`/`asynTrigRouterConfig1`), not the slot number. ✅ verified 2026-04-09 — `devGVME.c:L245` (`ptr = (int *)(daqBoards[bdnum].base32 + (regaddr)/4)`)

---

## Table of Contents

- [Address Space Overview](#address-space-overview)
- [DIG — Digitizer Main FPGA](#dig--digitizer-main-fpga-spartan-3)
  - [0x0000–0x002C: Global Registers](#0x0000-0x002c-global-registers)
  - [0x0040–0x0064: Per-Channel Control](#0x0040-0x0064-per-channel-control)
  - [0x0080–0x00A4: LED Thresholds](#0x0080-0x00a4-led-thresholds)
  - [0x00C0–0x00E4: CFD Fraction](#0x00c0-0x00e4-cfd-fraction)
  - [0x0100–0x0124: Raw Data Delay](#0x0100-0x0124-raw-data-delay)
  - [0x0140–0x0164: Raw Data Length](#0x0140-0x0164-raw-data-length)
  - [0x0180–0x01A4: D Window](#0x0180-0x01a4-d-window-pulse-filter)
  - [0x01C0–0x01E4: K Window](#0x01c0-0x01e4-k-window)
  - [0x0200–0x0224: M Window](#0x0200-0x0224-m-window)
  - [0x0240–0x0264: D3 Window](#0x0240-0x0264-d3-window)
  - [0x0280–0x02A4: Discriminator Width](#0x0280-0x02a4-discriminator-width)
  - [0x0300–0x0324: P1 Window](#0x0300-0x0324-p1-window)
  - [0x0400–0x0540: Miscellaneous Control](#0x0400-0x0540-miscellaneous-control)
  - [0x0600–0x060C: Code Revision / Timestamp Error](#0x0600-0x060c-code-revision--timestamp-error)
  - [0x0700–0x07E4: Per-Channel Event Counters](#0x0700-0x07e4-per-channel-event-counters)
  - [0x07E8–0x0834: ADC Saturation Counters](#0x07e8-0x0834-adc-saturation-counters)
  - [0x0848: Misc / SD Config](#0x0848-misc--sd-config)
  - [0x1000: Data FIFO Port](#0x1000-data-fifo-port)
  - [DIG VME FPGA](#dig-vme-fpga-dig-board-only)
- [MTRG — Master Trigger Main FPGA](#mtrg--master-trigger-main-fpga-virtex-4)
  - [0x0100–0x011C: Bus Control / Timestamp](#0x0100-0x011c-bus-control--timestamp)
  - [0x0120–0x01FC: Status and Monitoring](#0x0120-0x01fc-status-and-monitoring)
  - [0x0200–0x02CC: Control / Configuration](#0x0200-0x02cc-control--configuration)
  - [0x0300–0x05FC: RAM Blocks](#0x0300-0x05fc-ram-blocks)
  - [0x0600–0x067C: Rate Counters](#0x0600-0x067c-rate-counters)
  - [0x0800–0x08F0: I/O and Control](#0x0800-0x08f0-io-and-control)
  - [MTRG VME FPGA](#mtrg-vme-fpga)
- [RTRG — Router Trigger Main FPGA](#rtrg--router-trigger-main-fpga-virtex-4)
  - [0x0100–0x01CC: Status / Control / Counters](#0x0100-0x01cc-status--control--counters)
  - [0x0600–0x073C: Discriminator Delay](#0x0600-0x073c-discriminator-delay-disc_delay)
  - [0x0800–0x08F0: I/O and Control](#0x0800-0x08f0-io-and-control-1)
- [VME FPGA — Common Registers](#vme-fpga--common-registers-dig-and-mtrg)
- [Flash Access Registers](#flash-access-registers-dig-and-mtrg-vme-fpga)
- [CONN / CPLD Registers](#conn--cpld-registers-mtrg-and-rtrg)
- [IOC Shell Usage Examples](#ioc-shell-usage-examples)

---

## Address Space Overview

| Range | Board | Content |
|-------|-------|---------|
| `0x0000–0x088C` | DIG | DIG main FPGA registers |
| `0x0100–0x08F0` | MTRG | MTRG main FPGA registers |
| `0x0100–0x08F0` | RTRG | RTRG main FPGA registers |
| `0x0300–0x05FC` | MTRG | VETO/TRIG/SWEEP RAM blocks (3 × 256 B) |
| `0x0600–0x067C` | MTRG | Rate counters (8 × 2 registers) |
| `0x0600–0x073C` | RTRG | DISC_DELAY per-channel delays (8 links × 10 ch) |
| `0x0900–0x093C` | DIG/MTRG | VME FPGA control/status |
| `0x0980–0x098C` | DIG/MTRG | Flash access registers |
| `0x1000–0x4004` | MTRG/RTRG | CONN (4-port) and CPLD registers |

---

## DIG — Digitizer Main FPGA (Spartan-3)

EPICS PV prefix: `VME<CRATE>:<BOARD>:` (e.g. `VME66:MDIG1:`)

### 0x0000–0x002C: Global Registers

| Offset | Register | R/W | Description |
|--------|----------|-----|-------------|
| `0x0000` | `regin_board_id` | R | Board identification |
| `0x0004` | `reg_programming_done` | R | FPGA programming done + FIFO status bits |
| `0x0008` | `reg_external_discriminator_src` | R/W | External discriminator source select |
| `0x000C` | `reg_vme_ext_delay` | R/W | VME external delay |
| `0x0020` | `regin_hardware_status` | R | Hardware status bits |
| `0x0024` | `reg_user_package_data` | R/W | User package data tag (set at boot per crate/board) |
| `0x0028` | `reg_win_comp_min` | R/W | Window comparator minimum |
| `0x002C` | `reg_win_comp_max` | R/W | Window comparator maximum |

**`reg_programming_done` FIFO status bits** (read after all board firmware loads):

| Bits | Field | Meaning |
|------|-------|---------|
| `[18:0]` | depth | FIFO occupancy (words) |
| `[19]` | PROG_FULL | FIFO prog-full threshold reached |
| `[20:21]` | EMPTY | FIFO empty flags |
| `[22]` | ALMOST_EMPTY | Below almost-empty threshold |
| `[23]` | HALF_FULL | FIFO half full |
| `[24]` | ALMOST_FULL | Above almost-full threshold |
| `[25:26]` | ALL_FULL | FIFO completely full |

### 0x0040–0x0064: Per-Channel Control

**Pattern:** 10 channels at offsets `0x0040 + ch*4` (ch = 0–9)

| Offset | Register | Description |
|--------|----------|-------------|
| `0x0040 + ch×4` | `reg_channel_control<ch>` | Channel enable, mode, trigger type | ✅ verified 2026-04-10 — `asynDigParams.c:L459` (`setAddress(reg_channel_control0,0x0040)`)

Range: `0x0040` (ch0) … `0x0064` (ch9)

### 0x0080–0x00A4: LED Thresholds

**Pattern:** 10 channels at `0x0080 + ch*4`

| Offset | Register | Description |
|--------|----------|-------------|
| `0x0080 + ch×4` | `reg_led_threshold<ch>` | Leading-edge discriminator threshold | ✅ verified 2026-04-10 — `asynDigParams.c:L469` (`setAddress(reg_led_threshold0,0x0080)`)

Range: `0x0080` (ch0) … `0x00A4` (ch9)

### 0x00C0–0x00E4: CFD Fraction

**Pattern:** 10 channels at `0x00C0 + ch*4`

| Offset | Register | Description |
|--------|----------|-------------|
| `0x00C0 + ch×4` | `reg_CFD_fraction<ch>` | CFD fraction parameter |

Range: `0x00C0` (ch0) … `0x00E4` (ch9)

### 0x0100–0x0124: Raw Data Delay

**Pattern:** 10 channels at `0x0100 + ch*4`

| Offset | Register | Description |
|--------|----------|-------------|
| `0x0100 + ch×4` | `reg_raw_data_delay<ch>` | Pipeline delay before raw data capture |

Range: `0x0100` (ch0) … `0x0124` (ch9)

### 0x0140–0x0164: Raw Data Length

**Pattern:** 10 channels at `0x0140 + ch*4`

| Offset | Register | Description |
|--------|----------|-------------|
| `0x0140 + ch×4` | `reg_raw_data_length<ch>` | Number of samples captured in raw mode |

Range: `0x0140` (ch0) … `0x0164` (ch9)

### 0x0180–0x01A4: D Window (Pulse Filter)

**Pattern:** 10 channels at `0x0180 + ch*4`

| Offset | Register | Description |
|--------|----------|-------------|
| `0x0180 + ch×4` | `reg_d_window<ch>` | D (differentiation) window length |

Range: `0x0180` … `0x01A4`

### 0x01C0–0x01E4: K Window

| Offset | Register | Description |
|--------|----------|-------------|
| `0x01C0 + ch×4` | `reg_k_window<ch>` | K (integration/peaking) window length |

Range: `0x01C0` … `0x01E4`

### 0x0200–0x0224: M Window

| Offset | Register | Description |
|--------|----------|-------------|
| `0x0200 + ch×4` | `reg_m_window<ch>` | M window (baseline/gap) length |

Range: `0x0200` … `0x0224`

### 0x0240–0x0264: D3 Window

| Offset | Register | Description |
|--------|----------|-------------|
| `0x0240 + ch×4` | `reg_d3_window<ch>` | D3 window parameter |

Range: `0x0240` … `0x0264`

### 0x0280–0x02A4: Discriminator Width

| Offset | Register | Description |
|--------|----------|-------------|
| `0x0280 + ch×4` | `reg_disc_width<ch>` | Discriminator output width |

Range: `0x0280` … `0x02A4`

### 0x0300–0x0324: P1 Window

| Offset | Register | Description |
|--------|----------|-------------|
| `0x0300 + ch×4` | `reg_p1_window<ch>` | P1 window (pole-zero correction) |

Range: `0x0300` … `0x0324`

### 0x0400–0x0540: Miscellaneous Control

| Offset | Register | R/W | Description |
|--------|----------|-----|-------------|
| `0x0400` | `reg_dac` | R/W | DAC control |
| `0x0404` | `reg_p2_window0` | R/W | P2 window ch0 |
| `0x0408` | `reg_ila_config` | R/W | ILA (in-circuit logic analyzer) config |
| `0x040C` | `reg_channel_pulsed_control` | R/W | Channel pulsed control |
| `0x0410` | `reg_diag_mux_control` | R/W | Diagnostic mux select |
| `0x0414` | `reg_holdoff_control` | R/W | Trigger holdoff control |
| `0x0418` | `reg_baseline_delay` | R/W | Baseline delay |
| `0x041C` | `reg_diag_channel_input` | R/W | Diagnostic channel input |
| `0x0420` | `reg_external_disc_mode` | R/W | External discriminator mode (IntAcptAll/ExtTTL/ExtTTCL/Diag) |
| `0x0424` | `reg_rj45_spare_dout_control` | R/W | RJ45 spare digital output control |
| `0x0428` | `regin_led_state` | R | LED discriminator state readback |
| `0x0430` | `reg_led_control` | R/W | LED front-panel control |
| `0x0434` | `reg_downsample_holdoff` | R/W | Downsampling holdoff |
| `0x0484` | `regin_lat_timestamp_lsb` | R | Latched timestamp [31:0] |
| `0x0488` | `regin_lat_timestamp_msb` | R | Latched timestamp [47:32] |
| `0x048C` | `regin_live_timestamp_lsb` | R | Live timestamp [31:0] |
| `0x0490` | `regin_live_timestamp_msb` | R | Live timestamp [47:32] |
| `0x0494` | `reg_veto_gate_width` | R/W | Veto gate width |
| `0x0500` | `reg_master_logic_status` | R | Master logic status |
| `0x0504` | `reg_trigger_config` | R/W | Trigger configuration word |
| `0x0508` | `regin_phase_errors` | R | SERDES phase error counter |
| `0x050C` | `regin_phase_value` | R | Current SERDES phase value |
| `0x0510` | `regin_phase_offset_a` | R | Phase offset A |
| `0x0514` | `regin_phase_offset_b` | R | Phase offset B |
| `0x0518` | `regin_phase_offset_c` | R | Phase offset C |
| `0x051C` | `regin_serdes_phase_value` | R | SERDES phase value |
| `0x0520–0x0540` | `reg_p2_window1–9` | R/W | P2 window ch1–9 (ch1 at 0x0520) |

### 0x0600–0x060C: Code Revision / Timestamp Error
✅ verified 2026-04-09 — `asynDigParams.c:L604-605` (`setAddress(regin_code_revision,0x0600)`, `setAddress(regin_code_date,0x0604)`)

| Offset | Register | R/W | Description |
|--------|----------|-----|-------------|
| `0x0600` | `regin_code_revision` | R | FPGA code revision word (bits[11:8] = board type) |
| `0x0604` | `regin_code_date` | R | FPGA code build date |
| `0x0608` | `reg_ts_err_count_ctrl` | R/W | Timestamp error counter control |
| `0x060C` | `regin_ts_err_count` | R | Timestamp error count |

**`regin_code_revision` board type decode** (bits[11:8]):

| Code | Board |
|------|-------|
| `0xC` | ANL Master Digitizer (MDIG) |
| `0xD` | ANL Slave Digitizer (SDIG) |
| `0x4` | DGS Master Trigger (MTRG) |
| `0x6` | DGS Router Trigger (RTRG) |

### 0x0700–0x07E4: Per-Channel Event Counters

| Offset | Register | Description |
|--------|----------|-------------|
| `0x0700 + ch×4` | `regin_dropped_event_count<ch>` | Dropped event counter |
| `0x0740 + ch×4` | `regin_accepted_event_count<ch>` | Accepted event counter |
| `0x0780 + ch×4` | `regin_ahit_count<ch>` | Accepted hit counter |
| `0x07C0 + ch×4` | `regin_disc_count<ch>` | Discriminator hit counter |

Each group: ch0–ch9, stride 4 bytes per channel. Range per group: 0x28 bytes (10 × 4).

### 0x07E8–0x0834: ADC Saturation Counters

| Offset | Register | Description |
|--------|----------|-------------|
| `0x07E8 + ch×4` | `regin_hihilolo<ch>` | Hi-hi / lo-lo (double saturation) count, ch0–9 |
| `0x0810 + ch×4` | `regin_hilo<ch>` | Hi / lo (single saturation) count, ch0–9 |

### 0x0848: Misc / SD Config

| Offset | Register | Description |
|--------|----------|-------------|
| `0x0848` | `reg_sd_config` | SD (signal detection) configuration |

### 0x1000: Data FIFO Port

The DIG data FIFO is **not a memory range** — it is a single read port. Every read to `0x1000` pops the next 32-bit word off the FIFO and advances its internal read pointer (like reading from a pipe).

| Property | Value | Source |
|----------|-------|--------|
| **Port address** | `0x1000` (byte offset) | `DIG_FIFO = 0x1000/4` in `inLoopSupport.c` |
| **Physical depth** | 262,144 × 32-bit words (256 K longwords = 1 MB) | `DGS_DEFS.h` |
| **Max DMA transfer** | 512 KB (`MAX_DIG_RAW_XFER_SIZE`) | `DGS_DEFS.h` |
| **Live depth / status** | `VMERead32(bdnum, 0x0004)` bits[18:0] | `reg_programming_done` |

**Board detection:** Reading from `0x1000` without a bus error confirms the board is an ANL digitizer (used in `devAsynDigCardInit`).

**FIFO depth from `reg_programming_done` (0x0004):**

| Bits | Field | Meaning |
|------|-------|---------|
| `[18:0]` | depth | Words currently in FIFO |
| `[19]` | PROG_FULL | Programmable full threshold crossed |
| `[21:20]` | EMPTY | FIFO empty flags (both bits set = empty) |
| `[22]` | ALMOST_EMPTY | Below almost-empty threshold |
| `[23]` | HALF_FULL | FIFO half full |
| `[24]` | ALMOST_FULL | Above almost-full threshold |
| `[25:26]` | ALL_FULL | FIFO completely full |

**Normal readout path:** The `inLoop` task continuously polls `reg_programming_done` for depth, then bulk-reads the FIFO via VME DMA (`sysVmeDmaV2LCopy`) into a 1 MB IOC buffer, then forwards the data to the TCP sender (port 9001) for the data host.

> **Warning:** Each read to `0x1000` destructively removes a word. Reading via `VMERead32` while `inLoop` is running steals words from the normal data stream and will corrupt packet boundaries seen by the collector. Only do this during idle (no beam, no triggers) or after stopping the DAQ.

---

## DIG VME FPGA (DIG board only)

These registers sit above the main FPGA space and control the Spartan-3 VME interface FPGA.

| Offset | Register | R/W | Description |
|--------|----------|-----|-------------|
| `0x0900` | `reg_vme_fpga_ctl` | R/W | VME FPGA control (`fpga_ctrl_reg`) |
| `0x0908` | `reg_vme_aux_status` | R | VME auxiliary status |
| `0x0910` | `vme_clk_ctrl` | R/W | VME clock control / select |
| `0x0920` | `SERIAL_NUMBER` | R | Board serial number |
| `0x0924` | `reg_vme_code_rev` | R | VME FPGA code revision |
| `0x0928` | `reg_vme_code_date` | R | VME FPGA code build date |

---

## MTRG — Master Trigger Main FPGA (Virtex-4)

EPICS PV prefix: `VME<CRATE>:<BOARD>:` (e.g. `VME66:MTRG:`)

### 0x0100–0x011C: Bus Control / Timestamp
✅ verified 2026-04-10 — `asynMTrigParams.c:L741-746` (vxworks/dgsDrivers/dgsDriverApp/src/)

| Offset | Register | R/W | Description |
|--------|----------|-----|-------------|
| `0x0100` | `reg_LOCK_BUS` | R/W | SERDES bus lock control |
| `0x0104` | `reg_DEN_BUS` | R/W | Data enable (transmit) |
| `0x0108` | `reg_REN_BUS` | R/W | Receive enable |
| `0x010C` | `reg_SYNC_BUS` | R/W | Sync bus control |
| `0x0114` | `reg_TIMESTAMP_A` | R | Timestamp word A [31:0] |
| `0x0118` | `reg_TIMESTAMP_B` | R | Timestamp word B [31:0] |
| `0x011C` | `reg_TIMESTAMP_C` | R | Timestamp word C [31:0] |

### 0x0120–0x01FC: Status and Monitoring

| Offset | Register | R/W | Description |
|--------|----------|-----|-------------|
| `0x0120` | `reg_MSTR_MACH_STATE` | R | Master state machine state |
| `0x0124` | `reg_AUX_INPUT_STATE` | R | Auxiliary input state |
| `0x0128` | `reg_MISC_STAT` | R | Miscellaneous status |
| `0x012C–0x0148` | `reg_DiagnosticA–H` | R | Diagnostic registers A–H (8 × 4 bytes) |
| `0x014C` | `reg_LINK_LRU_MACH_STAT` | R | Link L/R/U state machine status |
| `0x0150` | `reg_MISC_STAT2` | R | Miscellaneous status 2 |
| `0x0154` | `reg_MON7_FIFO_DEPTH` | R | Monitor-7 FIFO depth |
| `0x0158` | `reg_CODE_DATE` | R | Firmware build date |
| `0x015C` | `reg_CODE_REVISION` | R | Firmware code revision |
| `0x01A0` | `reg_MON_FIFO_STATE` | R | Monitor FIFO state |
| `0x01A4` | `reg_CHAN_FIFO_STATE` | R | Channel FIFO state |
| `0x01A8` | `reg_OTHER_FIFO_STATE` | R | Other FIFO state |
| `0x01AC` | `reg_latched_fifo_depth` | R | Latched FIFO depth |
| `0x01B0` | `reg_SYSTEM_THROTTLE_MAP` | R | System throttle map |
| `0x01B4` | `reg_MON7_FIFO_STATE` | R | Monitor-7 FIFO state |
| `0x01BC` | `reg_SUM_CONN_BUF_MON` | R | Sum/connector buffer monitor |
| `0x01C4` | `reg_FRAME_12_CMD_CNT` | R | Frame-12 command counter |
| `0x01C8` | `reg_FRAME_14_CMD_CNT` | R | Frame-14 command counter |
| `0x01CC` | `reg_FRAME_16_CMD_CNT` | R | Frame-16 command counter |
| `0x01D0` | `reg_FRAME_17_CMD_CNT` | R | Frame-17 command counter |
| `0x01E0` | `reg_STARTING_TIMESTAMP_HI` | R | Starting timestamp high |
| `0x01E4` | `reg_STARTING_TIMESTAMP_MID` | R | Starting timestamp mid |
| `0x01E8` | `reg_STARTING_TIMESTAMP_LOW` | R | Starting timestamp low |
| `0x01EC–0x01FC` | `reg_FRAME_17_DATA_1–5` | R/W | Frame-17 data words 1–5 |

### 0x0200–0x02CC: Control / Configuration

| Offset | Register | R/W | Description |
|--------|----------|-----|-------------|
| `0x0200` | `reg_ENCODER_SOURCE_SELECT` | R/W | Encoder source select |
| `0x0204` | `reg_MYRIAD_TRIG_DELAY` | R/W | MyRIAD trigger delay |
| `0x0208` | `reg_MYRIAD_OVERLAP_CTL` | R/W | MyRIAD overlap control |
| `0x020C` | `reg_MON7_FILL_CTL` | R/W | Monitor-7 fill control |
| `0x0210` | `reg_ENCODER_TEST` | R/W | Encoder test mode |
| `0x0214` | `reg_USER_PACKAGE_DATA` | R/W | User package data word |
| `0x0218` | `reg_TDC_TRIG_SEL` | R/W | TDC trigger select (monitor source for TAC-II TDC) |
| `0x021C` | `reg_TRIG_ALGO_MUX_SEL` | R/W | Trigger algorithm mux select |
| `0x0220–0x023C` | `reg_TRIGGER_PRESCALE_A–H` | R/W | Per-link trigger prescale (8 links) |
| `0x0240` | `reg_REMOTE_TRIGGER_TS_OFFSET_L` | R/W | Remote trigger timestamp offset (Link L) |
| `0x0244` | `reg_REMOTE_TRIG_DELAY_L` | R/W | Remote trigger delay (Link L) |
| `0x0248` | `reg_REMOTE_TRIG_OVERLAP_CTL_L` | R/W | Remote trigger overlap control (Link L) |
| `0x024C` | `reg_REMOTE_TRIG_DIG_OFFSET_L` | R/W | Remote trigger digitizer offset (Link L) |
| `0x0250–0x026C` | `reg_TRIG_VETO_SELECT_A–H` | R/W | Per-link trigger veto select (8 links) |
| `0x0270` | `reg_LOCAL_TRIG_DELAY_L` | R/W | Local trigger delay (Link L) |
| `0x0274` | `reg_CPLD_EXTRA` | R/W | CPLD extra register |
| `0x0278` | `reg_SSI_CTL` | R/W | SSI (Strip-Serial Interface) control |
| `0x027C` | `reg_SSI_ENCODE_TIME` | R/W | SSI encode time |
| `0x0280` | `reg_LATCHED_TS_HIGH` | R | Latched timestamp [47:32] |
| `0x0284` | `reg_LATCHED_TS_MID` | R | Latched timestamp [31:16] |
| `0x0288` | `reg_LATCHED_TS_LOW` | R | Latched timestamp [15:0] |
| `0x0290` | `reg_REMOTE_TRIG_OVERLAP_CTL_R` | R/W | Remote trigger overlap control (Link R) |
| `0x0294` | `reg_REMOTE_TRIG_OVERLAP_CTL_U` | R/W | Remote trigger overlap control (Link U) |
| `0x0298` | `reg_REMOTE_TRIGGER_TS_OFFSET_R` | R/W | Remote trigger TS offset (Link R) |
| `0x029C` | `reg_REMOTE_TRIGGER_TS_OFFSET_U` | R/W | Remote trigger TS offset (Link U) |
| `0x02A0` | `reg_REMOTE_TRIG_DIG_OFFSET_R` | R/W | Remote trigger DIG offset (Link R) |
| `0x02A4` | `reg_REMOTE_TRIG_DIG_OFFSET_U` | R/W | Remote trigger DIG offset (Link U) |
| `0x02A8` | `reg_REMOTE_TRIG_DELAY_R` | R/W | Remote trigger delay (Link R) |
| `0x02AC` | `reg_REMOTE_TRIG_DELAY_U` | R/W | Remote trigger delay (Link U) |
| `0x02B0` | `reg_COINC_TRIG_MASK` | R/W | Coincidence trigger mask (base; 110 detector × 14 groups in PVs) |
| `0x02B4` | `reg_COINC_OVERLAP_CONTROL` | R/W | Coincidence overlap control |
| `0x02B8` | `reg_LOCAL_TRIG_DELAY_R` | R/W | Local trigger delay (Link R) |
| `0x02BC` | `reg_LOCAL_TRIG_DELAY_U` | R/W | Local trigger delay (Link U) |
| `0x02C0` | `reg_X_PLANE_LINK_MASK` | R/W | X-plane link mask |
| `0x02C4` | `reg_Y_PLANE_LINK_MASK` | R/W | Y-plane link mask |
| `0x02C8` | `reg_TRIGGER_HOLDOFF` | R/W | Trigger hold-off window |
| `0x02CC` | `reg_UNUSED_2CC` | — | Unused / reserved |

### 0x0300–0x05FC: RAM Blocks

Three 256-byte RAM blocks, each organized as **16 rows (A–P) × 4 columns (A–D)** = 64 entries × 4 bytes.

**Address pattern:** `base + (row_index × 0x10) + (col_index × 0x4)` where row A=0, B=1, … P=15.

| Range | Block | Purpose |
|-------|-------|---------|
| `0x0300–0x03FC` | `VETO_RAM` | Veto decision lookup table |
| `0x0400–0x04FC` | `TRIG_RAM` | Trigger decision lookup table |
| `0x0500–0x05FC` | `SWEEP_RAM` | Sweep/rate lookup table |

Example entries: `reg_VETO_RAM_AA` @ `0x0300`, `reg_VETO_RAM_AB` @ `0x0304`, `reg_VETO_RAM_BA` @ `0x0310`, … `reg_VETO_RAM_PD` @ `0x03FC`.

### 0x0600–0x067C: Rate Counters

Eight trigger algorithm pairs (prescaled + raw), each 32+32 bits:

| Offset | Register | Description |
|--------|----------|-------------|
| `0x0600 + N×8` | `reg_TRIG_RATE_COUNTER_<N>_LOW` | Trigger rate counter N, low 32 bits (N=1–8) |
| `0x0604 + N×8` | `reg_TRIG_RATE_COUNTER_<N>_HIGH` | Trigger rate counter N, high 32 bits |
| `0x0640 + N×8` | `reg_RAW_TRIG_RATE_COUNTER_<N>_LOW` | Raw (pre-prescale) rate counter N low |
| `0x0644 + N×8` | `reg_RAW_TRIG_RATE_COUNTER_<N>_HIGH` | Raw rate counter N high |

Range: `0x0600–0x063C` (prescaled 1–8) and `0x0640–0x067C` (raw 1–8).

### 0x0800–0x08F0: I/O and Control

| Offset | Register | R/W | Description |
|--------|----------|-----|-------------|
| `0x0800` | `reg_INPUT_LINK_MASK` | R/W | Input link mask (disable inputs from links) |
| `0x0804` | `reg_LED` | R/W | Front-panel LED source control |
| `0x0814` | `reg_MISC_CLK_CTL` | R/W | Miscellaneous clock control |
| `0x0818` | `reg_AUX_IO_CTL` | R/W | Auxiliary I/O port direction/control |
| `0x0820` | `reg_TARGET_WHEEL_AUX_CTL` | R/W | Target wheel auxiliary control |
| `0x0824` | `reg_TRIGGER_WIDTH` | R/W | Trigger output width |
| `0x0828` | `reg_SERDES_TPOWER` | R/W | SERDES transmit power control |
| `0x082C` | `reg_SERDES_RPOWER` | R/W | SERDES receive power control |
| `0x0830` | `reg_SERDES_LOCAL_LE` | R/W | SERDES local loopback enable |
| `0x0834` | `reg_SERDES_LINE_LE` | R/W | SERDES line loopback enable |
| `0x0838` | `reg_LVDS_PREEMPHASIS` | R/W | LVDS pre-emphasis |
| `0x083C` | `reg_LINK_LRU_CTL` | R/W | Link L/R/U control (DEN, REN, Sync per link) |
| `0x0840` | `reg_MISC_CTL1` | R/W | Miscellaneous control 1 |
| `0x0844` | `reg_MISC_CTL2` | R/W | Miscellaneous control 2 |
| `0x084C` | `reg_DIAG_PIN_CTL` | R/W | Diagnostic pin control |
| `0x0850` | `reg_TRIG_MASK` | R/W | Trigger mask |
| `0x0894` | `reg_MASTER_FIFO_RESETS` | R/W | Master FIFO reset controls |
| `0x0898` | `reg_MASTER_COUNTER_RESETS` | R/W | Master counter reset controls |
| `0x089C` | `reg_REMOTE_TRIGGER_PLANE_THRESHOLD` | R/W | Remote trigger plane threshold |
| `0x08A0–0x08BC` | `reg_MON1–8_FIFO_SEL` | R/W | Monitor FIFO 1–8 selection |
| `0x08C0` | `reg_CHAN_FIFO_CTL` | R/W | Channel FIFO control |
| `0x08C4` | `reg_REMOTE_MULTIPLCITY_CONTROL` | R/W | Remote multiplicity control |
| `0x08C8` | `reg_SUM_OF_X_THRESH` | R/W | Sum-of-X multiplicity threshold |
| `0x08CC` | `reg_SUM_OF_Y_THRESH` | R/W | Sum-of-Y multiplicity threshold |
| `0x08D0` | `reg_LINK_L_PROPAGATION_CONTROL` | R/W | Link L frame propagation control (F3–F7) |
| `0x08D4` | `reg_LINK_R_PROPAGATION_CONTROL` | R/W | Link R frame propagation control |
| `0x08D8` | `reg_LINK_U_PROPAGATION_CONTROL` | R/W | Link U frame propagation control |
| `0x08DC` | `reg_RATE_COUNTER_CTL` | R/W | Rate counter control (clear/latch) |
| `0x08E0` | `reg_PULSED_CTL1` | R/W | Pulsed control 1 (one-shot actions) |
| `0x08E4` | `reg_PULSED_CTL2` | R/W | Pulsed control 2 |
| `0x08E8` | `reg_NIM1_DELAY` | R/W | NIM IN 1 programmable delay |
| `0x08EC` | `reg_NIM2_DELAY` | R/W | NIM IN 2 programmable delay (RF/TDC input) |
| `0x08F0` | `reg_FIFO_RESETS` | R/W | FIFO reset control |

---

## MTRG VME FPGA

| Offset | Register | R/W | Description |
|--------|----------|-----|-------------|
| `0x0904` | `reg_fpga_status` | R | VME FPGA status (`fpga_status_register`) |
| `0x0920` | `reg_fpga_version` | R | VME FPGA version |
| `0x0924` | `reg_full_code_revision` | R | Full code revision |
| `0x0928` | `reg_code_date_VME` | R | VME FPGA code date |

---

## RTRG — Router Trigger Main FPGA (Virtex-4)

EPICS PV prefix: `VME<CRATE>:<BOARD>:` (e.g. `VME01:RTR1:`)

### 0x0100–0x01CC: Status / Control / Counters

| Offset | Register | R/W | Description |
|--------|----------|-----|-------------|
| `0x0100` | `reg_LOCK_BUS` | R/W | SERDES bus lock |
| `0x0104` | `reg_DEN_BUS` | R/W | Data enable |
| `0x0108` | `reg_REN_BUS` | R/W | Receive enable |
| `0x010C` | `reg_SYNC_BUS` | R/W | Sync bus |
| `0x0114` | `reg_TIMESTAMP_A` | R | Timestamp word A |
| `0x0118` | `reg_TIMESTAMP_B` | R | Timestamp word B |
| `0x011C` | `reg_TIMESTAMP_C` | R | Timestamp word C |
| `0x0128` | `reg_MISC_STAT_REG` | R | Miscellaneous status |
| `0x012C–0x0148` | `reg_DiagnosticA–H` | R | Diagnostic registers A–H |
| `0x0150` | `reg_THROTTLE_STATUS` | R | Throttle status |
| `0x0158` | `reg_CODE_DATE` | R | Firmware build date |
| `0x015C` | `reg_CODE_REVISION` | R | Firmware code revision |
| `0x0160–0x017C` | `reg_MON1–8_FIFO` | R | Monitor FIFO depths 1–8 |
| `0x0180–0x019C` | `reg_CHAN1–8_FIFO` | R | Channel FIFO depths 1–8 |
| `0x01A0` | `reg_MON_FIFO_STATE` | R | Monitor FIFO state |
| `0x01A4` | `reg_CHAN_FIFO_STATE` | R | Channel FIFO state |
| `0x01B0–0x01CC` | `reg_LOCK_COUNTER_A–H` | R | SERDES lock counters per link (8 links) |

### 0x0600–0x073C: Discriminator Delay (DISC_DELAY)

Per-DIG-link (A–H), per-channel (0–9) input delay registers.

**Pattern:** Link A starts at `0x0600`, link B at `0x0628`, with stride 0x04 per channel within each link. Each link covers 10 × 4 = 40 bytes.

> **Note:** Several registers in links B, C, D have anomalous addresses that overlap with the status block (e.g. `B_5` @ `0x013C`, `B_9` @ `0x014C`, `C_9` @ `0x0174`, `D_5` @ `0x018C`, `D_9` @ `0x019C`). This appears to be a copy-paste error in `asynRTrigParams.c`; the pattern is otherwise sequential.

| Link | First Offset | Last Offset | Register names |
|------|-------------|-------------|----------------|
| A | `0x0600` | `0x0624` | `reg_DISCRIMINATOR_DELAY_A_0–9` |
| B | `0x0628` | `0x0648` | `reg_DISCRIMINATOR_DELAY_B_0–9` |
| C | `0x0650` | `0x0670` | `reg_DISCRIMINATOR_DELAY_C_0–9` |
| D | `0x0678` | `0x0698` | `reg_DISCRIMINATOR_DELAY_D_0–9` |
| E | `0x06A0` | `0x06C4` | `reg_DISCRIMINATOR_DELAY_E_0–9` |
| F | `0x06C8` | `0x06EC` | `reg_DISCRIMINATOR_DELAY_F_0–9` |
| G | `0x06F0` | `0x0714` | `reg_DISCRIMINATOR_DELAY_G_0–9` |
| H | `0x0718` | `0x073C` | `reg_DISCRIMINATOR_DELAY_H_0–9` |

### 0x0800–0x08F0: I/O and Control

| Offset | Register | R/W | Description |
|--------|----------|-----|-------------|
| `0x0800` | `reg_INPUT_LINK_MASK` | R/W | Input link mask |
| `0x0804` | `reg_LED_REG` | R/W | Front-panel LED |
| `0x0808` | `reg_SKEW_CTL_A` | R/W | SERDES skew control A |
| `0x080C` | `reg_SKEW_CTL_B` | R/W | SERDES skew control B |
| `0x0810` | `reg_SKEW_CTL_C` | R/W | SERDES skew control C |
| `0x0814` | `reg_MISC_CLK_CTL` | R/W | Miscellaneous clock control |
| `0x0818` | `reg_AUX_IO_CTL` | R/W | Auxiliary I/O control |
| `0x081C` | `reg_AUX_IO_DATA` | R/W | Auxiliary I/O data |
| `0x0820` | `reg_AUX_INPUT_SELECT` | R/W | Auxiliary input select |
| `0x0824` | `reg_AUX_COUNTDOWN` | R/W | Auxiliary countdown timer |
| `0x0828` | `reg_SERDES_TPOWER` | R/W | SERDES transmit power |
| `0x082C` | `reg_SERDES_RPOWER` | R/W | SERDES receive power |
| `0x0830` | `reg_SERDES_LOCAL_LE` | R/W | SERDES local loopback |
| `0x0834` | `reg_SERDES_LINE_LE` | R/W | SERDES line loopback |
| `0x0838` | `reg_LVDS_PREEMPHASIS` | R/W | LVDS pre-emphasis |
| `0x083C` | `reg_LINK_LRU_CTL` | R/W | Link L/R/U control |
| `0x0840` | `reg_MISC_CTL1` | R/W | Miscellaneous control 1 |
| `0x0844` | `reg_MISC_CTL2` | R/W | Miscellaneous control 2 |
| `0x0848` | `reg_GENERIC_TEST_FIFO` | R/W | Generic test FIFO |
| `0x084C` | `reg_DIAG_PIN_CTL` | R/W | Diagnostic pin control |
| `0x0850` | `reg_FORCE_SYNC_REG` | R/W | Force sync register |
| `0x0854` | `reg_SPARE_854` | — | Spare / reserved |
| `0x0858–0x0874` | `reg_X_PLANE_MAP_A–H` | R/W | X-plane multiplicity map per link (8 registers) |
| `0x0878–0x0894` | `reg_Y_PLANE_MAP_A–H` | R/W | Y-plane multiplicity map per link (8 registers) |
| `0x0898` | `reg_ANY_THROTTLE_WIDTH` | R/W | Throttle gate width |
| `0x089C` | `reg_THROTTLE_LIMIT_TIME` | R/W | Throttle rate limit time |
| `0x08A0–0x08BC` | `reg_MON1–8_FIFO_SEL` | R/W | Monitor FIFO 1–8 selection |
| `0x08C0` | `reg_CHAN_MON_FIFO_CTL` | R/W | Channel monitor FIFO control |
| `0x08C4` | `reg_CHAN_MON_FIFO_WE_CTL` | R/W | Channel monitor FIFO write-enable control |
| `0x08C8` | `reg_TSCATTER_DELAY` | R/W | Time scatter delay |
| `0x08CC` | `reg_CLEAN_DIRTY` | R/W | Clean/dirty flag |
| `0x08E0` | `reg_PULSED_CTL1` | R/W | Pulsed control 1 |
| `0x08E4` | `reg_PULSED_CTL2` | R/W | Pulsed control 2 |
| `0x08E8` | `reg_SPARE_8E8` | — | Spare |
| `0x08EC` | `reg_SPARE_8EC` | — | Spare |
| `0x08F0` | `reg_FIFO_RESETS` | R/W | FIFO reset |

---

## VME FPGA — Common Registers (DIG and MTRG)

These registers are in the VME FPGA (Spartan-3), which interfaces the main FPGA to VME. The `devGVME.c` driver accesses them via a separate base pointer offset by `0x900` from the main FPGA base.

| Offset | Name (devGVME.c) | R/W | Description |
|--------|-----------------|-----|-------------|
| `0x0900` | `fpga_ctrl_reg` | R/W | FPGA control register |
| `0x0904` | `fpga_status_register` | R | FPGA status register |
| `0x0908` | `vme_aux_status` | R | VME auxiliary status |
| `0x090C` | `vme_config_control` | R/W | VME configuration control |
| `0x0910` | `fpga_gp_ctl` | R/W | General purpose control |
| `0x0918` | `config_start` | W | Start FPGA reconfiguration from flash |
| `0x091C` | `config_stop` | W | Stop FPGA reconfiguration |
| `0x0920` | `fpga_version` | R | FPGA version / serial number |
| `0x0924` | `full_code_revision` | R | Full code revision (board type in bits[11:8]) |
| `0x0928` | `code_date_VME` | R | VME FPGA code build date |
| `0x0930–0x093C` | sandbox registers | R/W | Sandbox (test/debug registers) |

**Board detection probe:** `devGVME.c` probes offset `0x0920` (word offset `0x248` from the word-based base) to detect board presence.

---

## Flash Access Registers (DIG and MTRG VME FPGA)

Used by `ProgramFlash` / `EraseFlash` / `VerifyFlash` / `ConfigureFlash` IOC shell commands.

> **DANGEROUS — Never access without authorization.** Writing to flash is irreversible without physical intervention.

| Offset | Name | R/W | Description |
|--------|------|-----|-------------|
| `0x0980` | `flash_address` | R/W | Flash byte address |
| `0x0984` | `flash_rd_wrt_autoinc` | R/W | Flash read/write with auto-increment of address |
| `0x098C` | `flash_rd_wrt_no_autoinc` | R/W | Flash read/write without auto-increment |

Flash layout: 2 banks (bank 0 = lower, bank 1 = upper). Banks are selected via the `bank` parameter to `ProgramFlash(bdnum, bank, "file.bin")`.

---

## CONN / CPLD Registers (MTRG and RTRG)

These access sub-modules via separate VME address pages (4 KB each), reaching SERDES connector boards (CONN A–D) and the CPLD fast-strobe logic.

| Offset | Register | Description |
|--------|----------|-------------|
| `0x1000` | `reg_CONN_A_DATA` | Connector A data (SERDES link A) |
| `0x1004` | `reg_CONN_B_DATA` | Connector B data (SERDES link B) |
| `0x2000` | `reg_CONN_C_DATA` | Connector C data (SERDES link C) |
| `0x2004` | `reg_CONN_D_DATA` | Connector D data (SERDES link D) |
| `0x3000` | `reg_CPLD_MASK` | CPLD fast-strobe multiplicity mask |
| `0x3004` | `reg_FS_SOURCE` | Fast-strobe source select (RTRG only) |
| `0x4000` | `reg_FS_MULT_THRESH` | Fast-strobe multiplicity threshold (MTRG only) |
| `0x4004` | `reg_CPLD_MULT` | CPLD multiplicity output (MTRG only) |

---

## IOC Shell Usage Examples

```
# Read DIG board ID (byte offset 0x0000, bdnum=cardno)
VMERead32(0, 0x0000)

# Read DIG firmware code revision and date (MDIG1 = cardno 0 on vme99/tangerine)
VMERead32(0, 0x0600)   # regin_code_revision  → e.g. 0x4CD8 (bits[11:8]=0xC = ANL MDIG)
VMERead32(0, 0x0604)   # regin_code_date

# Read DIG accepted event count ch0
VMERead32(0, 0x0740)

# Read MTRG code revision (cardno as configured in startup .cmd)
VMERead32(0, 0x015C)

# Read MTRG rate counter 1
VMERead32(0, 0x0600)
```

> `VMERead32(bdnum, regaddr)` — `bdnum` = cardno (2nd arg of `asynDigitizerConfig`/`asynTrigMasterConfig1`/`asynTrigRouterConfig1` in the startup `.cmd` file), `regaddr` = byte offset from the board's VME base.

> **cardno ≠ slot**: on `vme99.cmd` (tangerine), MDIG1 is `asynDigitizerConfig("MDIG1", 0, 2)` → `VMERead32(0, ...)` not `VMERead32(2, ...)`.

---

*Verified against source: `VxWorks/dgsDrivers/dgsDriverApp/src/{asynDigParams.c,asynMTrigParams.c,asynRTrigParams.c,devGVME.c}`. Last updated: 2026-04-08.*

---

## See Also

- `dgs/reference_index.md` — CSV-based register maps (from Excel spreadsheets with VBA); higher-level register groups and field descriptions. Use for register names → field meanings.
- `dgs/EPICS_asyn.md` — How asyn driver translates EPICS caput/caget into VME reads/writes
- `dgs/ioc.md` — IOC startup configuration; board cardno assignments (which cardno maps to which physical slot)
- `dgs/deep_fpga_DIG.md`, `dgs/deep_fpga_MTRG_MAIN.md`, `dgs/deep_fpga_RTRG.md` — FPGA firmware context for what each register does
- `dgs/IOC_cmd.md` — `VMERead32`/`VMEWrite32` VxWorks shell commands for direct register access
