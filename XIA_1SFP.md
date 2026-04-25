# XIA 1-SFP Interface Module — FPGA Firmware

**Source:** `~/FPGA_svn2git/XIA_1SFP_git/Source/` (23 files, 12,380 lines total)
**Top entity:** `TriggerInterface` (`XIA_TOP.vhd`, 1,145 lines)
**Author:** John T. Anderson (ANL), created 2021-03-01; active development 2024
**Target device:** Xilinx Spartan-6 XC6SLX9-2TQG144 ✅ verified 2026-04-25 — `XIA_TOP.vhd:L8`
**Tool:** Xilinx ISE 14.7
**See also:** [`myriad.md`](myriad.md), [`fpga.md`](fpga.md), [`FPGA_svn2git/INDEX.md`](../../FPGA_svn2git/INDEX.md)

Stability: C3 - Structural / stable

---

## Table of Contents

1. [Purpose](#purpose)
2. [Hardware Interface](#hardware-interface)
3. [Source File Inventory](#source-file-inventory)
4. [Clock Architecture](#clock-architecture)
5. [SERDES / TTCL Link Interface](#serdes--ttcl-link-interface)
6. [Delayed Trigger FSM](#delayed-trigger-fsm)
7. [VME Register Map](#vme-register-map)
8. [Output Signals to Pixie](#output-signals-to-pixie)
9. [ILA / ChipScope Debug](#ila--chipscope-debug)
10. [Key Design Decisions & Notes](#key-design-decisions--notes)

---

## Purpose

The XIA 1-SFP module bridges **XIA Pixie-based commercial digitizers** into the DGS/GRETINA/GRETA trigger and data acquisition system. It sits between the DGS trigger chain (via an SFP fiber to the DS92LV18 LVDS SerDes chip) and the XIA Pixie digitizer (via a 40-pin connector).

Key functions:
- Receives the TTCL (Trigger Timing and Control Link) data stream from the DGS Master Trigger over SFP fiber
- Extracts the 48-bit timestamp and synchronizes a local counter to it (using SYNC / Imperative Sync frames)
- When a trigger accept message arrives (frames 3–10), computes a fixed-delay output: DECODED_TRIG_FLAG fires at a precise time = trigger timestamp + programmable offset, making the NIM output predictable regardless of cable or SERDES latency
- Passes the recovered trigger clock (50 MHz from Si5324 VCXO locked to SERDES RCLK) to the Pixie for use as ADC/processing clock — ensuring Pixie runs phase-synchronous to DGS
- Provides a SPI control interface back to the Pixie system for status readback and configuration

This module is **MyRIAD-compatible** in its link receiver format (uses the same SERDES_RX_Mach and COMMAND_FORMAT = cCMD_FORMAT_DGS_MASTER) but is simpler: it does not generate trigger messages back to the MTRG — it is receive-only.

---

## Hardware Interface

### To Pixie (40-pin connector)

| Pin | Signal | Direction | Description |
|-----|--------|-----------|-------------|
| 9 | `DECODED_TRIG_FLAG` | → Pixie | Delayed trigger output (TRIG_FLAG delayed by configurable offset) |
| 16 | `FPGA_SCLK` | ← Pixie | SPI clock for register control |
| 18 | `DECODED_SYNC_FLAG` | → Pixie | SYNC flag from SERDES (frame 1 received) |
| 20 | `FPGA_CS` | ← Pixie | SPI chip select (active low) |
| 22 | `FPGA_MOSI` | ← Pixie | SPI MOSI (data to FPGA) |
| 24 | `FPGA_MISO` | → Pixie | SPI MISO (data from FPGA) |
| 27 | `DECODED_SM_LOCKED` | → Pixie | SERDES state machine locked (link acquired) |
| 40 | `PIXIE_25MHZ` | ← Pixie | 25 MHz clock from Pixie (spare/unused in current design) |

### To DS92LV18 LVDS SerDes

| Signal | Direction | Description |
|--------|-----------|-------------|
| `DS92LV18_RXDATA[17:0]` | ← SerDes | 18-bit received data (bit 17 = polarity/DC balance) |
| `DS92LV18_TXDATA[17:0]` | → SerDes | 18-bit transmit data (DC-balanced, runs at 50 MHz) |
| `DS92LV18_LOCK` | ← SerDes | Low = SerDes locked to incoming clock |
| `DS92LV18_TXCLK` | → SerDes | 50 MHz transmit clock (CLK_100MHz ÷ 2) |
| `DS92LV18_REFCLK` | → SerDes | Reference clock |
| `DS92LV18_SYNC` | → SerDes | Sync pattern control |
| `DS92LV18_DEN/REN` | → SerDes | Driver/receiver enable |
| `DS92LV18_TPWRDN/RPWRDN` | → SerDes | TX/RX power down (active low) |

> **Note:** The DS92LV18 RCLK is NOT connected directly to the FPGA — it goes to the Si5324 PLL instead. The Si5324 outputs a jitter-cleaned `LOCAL_FPGA_CLKP/N` differential pair which is used as the SERDES-derived clock to the FPGA. ✅ verified 2026-04-25 — `XIA_TOP.vhd:L137-141` (comment: "no direct connection from DS92LV18 RCLK to FPGA")

### To Si5394A Clock Generator

| Signal | Direction | Description |
|--------|-----------|-------------|
| `SI5394A_RST` | → Si | Chip reset |
| `SI5394A_LOS_XAXB` | ← Si | Loss of sync status |
| `SI5394A_LOLB` | ← Si | Loss of lock status |
| `SI5394A_INTR` | ← Si | Interrupt request |

---

## Source File Inventory

| File | Lines | Description |
|------|-------|-------------|
| `XIA_TOP.vhd` | 1,145 | Top-level entity: clock arch, SERDES I/O, delayed-trigger FSM, SPI host interface, ILA/VIO |
| `SERDES_RX_Mach_R2.vhd` | — | SERDES receiver state machine (R2 = with DET_DATA/XTRA_DATA; same as in MTRG) |
| `SERDES_RX_Mach.vhd` | — | Earlier version of SERDES RX (for reference) |
| `SERDES_TX_MACH.vhd` | — | SERDES transmitter state machine |
| `mstr_mach.vhd` | — | Master trigger/readout state machine (MTRG shared) |
| `MyRIAD_pkg.vhd` | — | MyRIAD trigger packet type definitions; package `XIA_data_types` + `XIA_comp_defs` |
| `Timestamp_Generator.vhd` | 268 | 48-bit timestamp counter, sync/isync logic (same module as DGS digitizer) |
| `GITMO_RCV_MACH.vhd` | — | GITMO receiver machine (shared; used if Link L connects GITMO) |
| `GITMO_TOP.vhd` | — | GITMO top-level (shared) |
| `Phase_Hunter_SerDes.vhd` | — | SERDES phase alignment (same as MTRG/DIG) |
| `NIM_Delay.vhd` | — | Programmable NIM input delay |
| `tdc_unit2.vhd` | 261 | TDC unit (carry-chain) |
| `tdc_short_chain.vhd` | — | Short TDC carry chain |
| `DCM_CONTROLLER.vhd` | — | DCM controller |
| `DCBAL_in.vhd` / `DCBAL_in_nofifo.vhd` | — | DC balance input decoders |
| `Fifo.vhd` | — | External FIFO wrapper (IDT 7007) |
| `basic_capture_counter.vhd` | — | Capture counter |
| `trigger_data_types.vhd` | 113 | DGS trigger data type definitions |
| `registers.vhd` | — | Register map |
| `PI_TRANSACTOR.vhd` | — | SPI transactor (same pattern as MγRIAD) |

Most source files are **shared verbatim** with the MγRIAD, MTRG, and DGS Digitizer designs.

---

## Clock Architecture

```
SFP fiber → DS92LV18 → RCLK → Si5324 VCXO → LOCAL_FPGA_CLKP/N (50 MHz, jitter-cleaned)
                                                       |
                                              IBUFGDS → BUFG → BUFGMUX
                                                                  |
Board oscillator (50 MHz) → IBUFGDS → BUFG → BUFGMUX ←-----------+
                                                  |
                                                  ▼
                                            XIA_1 PLL (Spartan-6 PLL_BASE)
                                            ├── CLK_100MHz  (main logic clock)
                                            ├── CLK_125MHz  (spare)
                                            └── CLK_50MHz   (SERDES RX / timestamp)
```

**Clock switching:** `BUFGMUX_1 (SYNC mode)` glitch-free mux between oscillator and trigger clock. Default = oscillator. Switch to trigger clock via `PULSED_CONTROL_REG[15] = 1` (only honored if trigger clock is detected as present). Falls back to oscillator automatically if trigger clock disappears. ✅ verified 2026-04-25 — `XIA_TOP.vhd:L414-505` (`CLOCK_FALLBACK_PROC`)

**Trigger clock monitor:** A 4-bit counter on `BUFG_CLK_FROM_TRIG` is sampled by the oscillator clock. A 3-state FSM (`WAIT_FOR_LOCK` → `SAW_A_CLOCK` → `CLOCK_IS_LOST`) detects if the counter is changing. If 4 consecutive oscillator samples see the same counter value, clock is declared lost and system falls back to oscillator. ✅ verified 2026-04-25 — `XIA_TOP.vhd:L443-497`

**Slow clock:** CLK_100MHz ÷ 25 = 4 MHz used for optional slow-mode ILA capture. ✅ verified 2026-04-25 — `XIA_TOP.vhd:L557-568`

---

## SERDES / TTCL Link Interface

Uses **`SERDES_RX_Mach`** instantiated with:
- `COMMAND_FORMAT => cCMD_FORMAT_DGS_MASTER` — expects full 20-frame MTRG TTCL stream ✅ verified 2026-04-25 — `XIA_TOP.vhd:L857`
- `LINK_IS_L => '1'` — allows any bit in `PROPAGATION_CONTROL` to pass through ✅ verified 2026-04-25 — `XIA_TOP.vhd:L858`

**PROPAGATION_CONTROL default = 0x01FF:** ✅ verified 2026-04-25 — `XIA_TOP.vhd:L336` (`signal PROPAGATION_CONTROL : std_logic_vector(15 downto 0) := X"01FF"`)
- Bits [8:0] = 1 → only Sync frame (bit 0) and trigger decision frames (bits 1–8, frames 3–10) pass
- Bits [15:9] = 0 → Aux Det commands, sync capture, async commands, internal-to-trigger frames are ignored
- This matches "receive-only TTCL client" — XIA sees triggers and sync but generates no response

**DC balance decoding:** `SERDES_RX_LATCHED = SERDES_RX[16:1]` if `SERDES_RX[17]=0`, else `NOT SERDES_RX[16:1]`. Bit 17 is the DC-balance parity bit. ✅ verified 2026-04-25 — `XIA_TOP.vhd:L822-823`

**Outputs extracted from SERDES_RX_Mach:**
- `TTCL_TRIG_FLAG` — high for one clock when trigger accept message received (frames 3–10)
- `TTCL_TRIG_TIMESTAMP[47:0]` — timestamp embedded in trigger accept message
- `TTCL_TRIG_TYPE[2:0]` — trigger type code
- `serdes_sync_flag`, `serdes_isync_flag`, `serdes_sync_timestamp` — for timestamp sync
- `TRIG_MON_DET_DATA[15:0]`, `TRIG_MON_XTRA_DATA[15:0]` — target wheel / multiplicity from frame 2 (DGS-specific; added 2018-04-02)

Frame 17 (AUX detector frame) is routed to `FRAME_17_REQ_FLAG` / `FRAME_17_DATA` — available for future use. All other frames (12, 14, 15, 16) are ignored (`open`).

---

## Delayed Trigger FSM

**Purpose:** Issue `DECODED_TRIG_FLAG` at a deterministic time relative to the trigger timestamp, regardless of cable/SERDES latency.

**States:** `IDLE` → `CALC_OFFSET_TIME` → `LET_FLAGS_SETTLE1` → `LET_FLAGS_SETTLE2` → `WAIT_MATCH` ✅ verified 2026-04-25 — `XIA_TOP.vhd:L1021-1111`

| State | Action |
|-------|--------|
| `IDLE` | Wait for `TTCL_TRIG_FLAG`. Only active if `GATING_REG[1] = 1`. |
| `CALC_OFFSET_TIME` | Latch `TS_MATCH_VALUE = TTCL_TRIG_TIMESTAMP + TTCL_TIME_OFFSET` (48-bit adder, offset is 16-bit) ✅ verified 2026-04-25 — `XIA_TOP.vhd:L925` |
| `LET_FLAGS_SETTLE1/2` | Two-cycle pipeline delay for flag combinational logic to settle |
| `WAIT_MATCH` | Compare `SYSTEM_TIMESTAMP` against `TS_MATCH_VALUE`. Fire `DLYD_TTCL_TRIG_OUT` on match. Error if `SYSTEM_TIMESTAMP > TS_MATCH_VALUE` (match missed). |

> **⚠️ Implementation note:** `GATING_REG` and `TTCL_TIME_OFFSET` are declared in `XIA_TOP.vhd:L358,L298` but **never driven by the SPI register interface**. The writable register case statement (L681-684) only covers PULSED_CONTROL_REG, FPGA_CTL_REG, and MISC_CTL_REG. `GATING_REG` defaults to `X"0000"`, so `GATING_REG[1]=0` always — the DELAYED_TRIG FSM stays in IDLE and the delayed trigger output is never asserted in current firmware. This appears to be a partial port from MyRIAD.vhd where `GATING_REG` is driven via `registers.vhd` at VME address 0x0702. ✅ verified 2026-04-25 — `XIA_TOP.vhd:L681-684` (no GATING_REG case); `MyRIAD.vhd:L927` (GATING_REG driven from registers.vhd REG_702)

**Stretch:** On `DLYD_TTCL_TRIG_OUT`, a one-shot stretches the output to 11 × 10 ns = 110 ns via a 4-bit countdown (`DLYD_TTCL_STRETCH_COUNT = "1010"`). ✅ verified 2026-04-25 — `XIA_TOP.vhd:L1001-1019`

**Error counters:**
- `MISSED_TRIG_COUNT` — another trigger arrived while waiting for first match
- `DLYD_TRIG_ERR_COUNT` — match time already passed when we entered WAIT_MATCH

Both cleared by `PULSED_CONTROL_REG[3] = 1`.

---

## VME Register Map

> Note: The XIA 1-SFP uses a SPI serial interface (not VME), but the register file follows the same pattern as other JTA FPGA designs.

| Address (7-bit) | Register | R/W | Default | Description |
|-----------------|----------|-----|---------|-------------|
| 0x00 | `PULSED_CONTROL_REG` | W (clears next clock) | 0x0000 | One-shot control bits | ✅ verified 2026-04-25 — `XIA_TOP.vhd:L266,L257` (addr="0000000", default X"0000") |
| 0x01 | `FPGA_CTL_REG` | R/W | 0x5A00 | General control (persistent) | ✅ verified 2026-04-25 — `XIA_TOP.vhd:L267,L258` (addr="0000001", default X"5A00") |
| 0x02 | `MISC_CTL_REG` | R/W | 0x0000 | Miscellaneous control; ILA source select [7:4]; slow-clock ILA mode [13] | ✅ verified 2026-04-25 — `XIA_TOP.vhd:L268,L259` (addr="0000010", default X"0000") |
| 0x7E | `CODE_DATE_REG` | R | 0x0305 | Firmware date code (March = 03, 2021→05?) | ✅ verified 2026-04-25 — `XIA_TOP.vhd:L269,L260` (addr="1111110", constant X"0305") |
| 0x7F | `CODE_REVISION_REG` | R | 0x0040 | Firmware revision (Rev B started at 0x0020) | ✅ verified 2026-04-25 — `XIA_TOP.vhd:L270,L261` (addr="1111111", constant X"0040", comment: "revision B started with 0x0020") |

**PULSED_CONTROL_REG bit assignments:**
| Bit | Function |
|-----|----------|
| 15 | Request switch to trigger clock (only honored if `TRIG_CLK_PRESENT`) ✅ verified 2026-04-25 — `XIA_TOP.vhd:L508` |
| 4 | Local TS reset (simulates Imperative Sync from oscillator) ✅ verified 2026-04-25 — `XIA_TOP.vhd:L938` (→ `LOCAL_TS_RESET_FLAG`) |
| 3 | Reset serial machine + clear error counters ✅ verified 2026-04-25 — `XIA_TOP.vhd:L613,L1128` |
| 2 | Reset 100 MHz logic domain + reset SERDES lost-lock flag ✅ verified 2026-04-25 — `XIA_TOP.vhd:L585` (RESET_TO_LOGIC_100MHZ), `L872` (SERDES_SM_LOST_LOCK_RST) |
| 1 | Global reset (→ Timestamp Generator reset) ✅ verified 2026-04-25 — `XIA_TOP.vhd:L933` |
| 0 | (unused) |

**FPGA_STATUS** (8-bit, returned during SPI address phase):
- Fixed pattern `0b10110011` = 0xB3 ✅ verified 2026-04-25 — `XIA_TOP.vhd:L392-400`

---

## Output Signals to Pixie

| Signal | Description |
|--------|-------------|
| `DECODED_TRIG_FLAG` | Delayed trigger output — fires at `TTCL_TRIG_TIMESTAMP + TTCL_TIME_OFFSET`. Stretched to ~110 ns. Gated by `GATING_REG[1]`. |
| `DECODED_SYNC_FLAG` | SERDES sync flag — high when frame 1 (timestamp sync) received |
| `DECODED_SM_LOCKED` | SERDES state machine locked — link is acquired and stable |
| `FPGA_MISO` | SPI readback (PREFETCH_DATA from register mux) |

The recovered 50 MHz clock (from Si5324, driven by DS92LV18 RCLK extracted from the fiber) is routed from the Si5324 output pins directly to Pixie via board traces — **not through the FPGA** (the Si5324 output goes to `PIXIE_FPGA_CLKP/N` on pins 10/12 of the Pixie connector, which is not connected to the FPGA). The FPGA merely controls whether the PLL locks to oscillator or trigger clock via `PLL_INPUT_SEL`.

---

## ILA / ChipScope Debug

Two ChipScope cores (`ICON2`): one VIO, one ILA (16,384 × 17-bit).

**VIO interface:** `VIO_7ADDR_1RW_1GO_1TAKE_16DIN_16DOUT` — allows back-door SPI register access without a physical host.

**ILA source selection via `MISC_CTL_REG[7:4]` (`ILA_SEL_CODE[4:1]`):**
| Code | Monitored signals |
|------|-------------------|
| 0000 | Main SPI serial control port: CE, MOSI, SCK, data valid, transaction in progress, MISO, address, data direction |
| others | All zeros (not implemented yet) |

Slow-clock mode: `MISC_CTL_REG[13] = 1` → ILA only captures on `SLOW_CLK` pulses (4 MHz), useful for very-slow debug.

---

## Key Design Decisions & Notes

1. **Receive-only TTCL client:** Unlike MγRIAD, the XIA 1-SFP does not generate trigger messages back to the MTRG. The SERDES TX path exists (DC-balanced data is sent) but carries a passive frame stream.

2. **Clock source hierarchy:** Oscillator (default) → trigger clock (if SERDES lock acquired and register bit set). Automatic fallback to oscillator prevents lockup if SFP disconnected.

3. **Timestamp runs at CLK_50MHz, increments by 2:** TIMESTAMP + 2 per rising edge = 100 MHz effective count rate matching DGS timestamp convention. ✅ verified 2026-04-25 — `XIA_TOP.vhd:L762-769`

4. **`PROPAGATION_CONTROL` default 0x01FF:** Frame bits 0 (sync) and 1–8 (trigger decision frames 3–10) pass; all higher-level commands ignored. This is appropriate for a "pure trigger receiver" role.

5. **LINK_IS_L = '1':** Allows all propagation control bits, giving flexibility to enable frame 17 (AUX det) data extraction in future without hardware change.

6. **`GATING_REG[1]` must be set** to enable trigger output. Defaults to 0 at power up (safe default: no spurious NIM outputs during initialization).

7. **Firmware revision tracking:** `CODE_REVISION_REG = 0x0040`. Comment says "Rev B started with 0x0020". Rev A presumably = 0x0000–0x001F range. Active development was 2024.

8. **Shared IP with MγRIAD:** SERDES_RX_Mach, SERDES_TX_MACH, Timestamp_Generator, GITMO_RCV_MACH, Phase_Hunter_SerDes all pulled verbatim from the MγRIAD/MyRIAD codebase. See [`myriad.md`](myriad.md) for protocol details.
