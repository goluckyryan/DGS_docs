# Majorana Digitizer Firmware Branch

Stability: C3 - Structural / stable

**Source:** `DGS_tools_pack/FPGA/others/Majorana_Digitizer/MAIN_FPGA/Source/`
**Generated:** 2026-04-25 from primary source (VHDL files)

## Table of Contents

- [Overview](#overview)
- [Key Differences from Standard DGS Digitizer](#key-differences-from-standard-dgs-digitizer)
- [Top-Level Structure (Digitizer.vhd)](#top-level-structure-digitizervhd)
- [Discriminator](#discriminator-jta_channelvhd--thresh_discvhd)
- [Baseline Tracker](#baseline-tracker-baseline_trackervhd)
- [SERDES Interface](#serdes-interface)
- [Front Bus (FBUS)](#front-bus-fbus)
- [External FIFO](#external-fifo)
- [Firmware Identification](#firmware-identification)
- [File Inventory](#file-inventory)
- [Relationship to DGS Digitizer](#relationship-to-dgs-digitizer)
- [Cross-References](#cross-references)

---

## Overview

The Majorana Digitizer is a branch of the standard ANL DGS digitizer firmware (Spartan-3 XC3S5000), split on **2015-08-31** to support the **Majorana Demonstrator** experiment at SURF (South Dakota Underground Research Facility). It is based on the same hardware PCB as the standard DGS digitizer but with significant modifications to the discriminator logic. This code lives in `FPGA/others/Majorana_Digitizer/`, separate from the main DGS trunk in `FPGA/DIG/`.

**FPGA:** Spartan-3 XC3S5000-5F900C (same as DGS digitizer)
**Code date:** 0x20150601 (June 1, 2015) ✅ verified 2026-04-25 — `Digitizer.vhd:L416` (`regin_code_date <= X"20150601"`)
**Code version:** Major=0xB, Minor=0xB ✅ verified 2026-04-25 — `Digitizer.vhd:L351-352` (`cCODE_VERSION_MAJOR := X"B"`, `cCODE_VERSION_MINOR := X"B"`)
**Firmware type:** 0xC (LED mode, `SLAVE_MODE=FALSE`) or 0xD (CFD mode, `SLAVE_MODE=TRUE`) ✅ verified 2026-04-25 — `Digitizer.vhd:L391` (`X"D" & ... when SLAVE_MODE=TRUE else X"C" & ...`)

> Note: Despite the CFD/LED naming convention for type codes, in the Majorana branch CFD logic is removed. The 0xD code applies when `SLAVE_MODE=TRUE` (slave digitizer).

---

## Key Differences from Standard DGS Digitizer

Per the 2015-08-31 branch separation comments, the Majorana version differs in:

1. **CFD removed** — the Constant Fraction Discriminator (CFD) path is entirely removed. ✅ verified 2026-04-25 — `jta_channel.vhd:L968` (`DISC_IS_EXTERNAL <= THRESH_DISC_IS_EXTERNAL; --20150831: Majorana doesn't have a cfd`)
2. **`CFD_MODE` control flag removed** — no software selection between LED/CFD. ✅ verified 2026-04-25 — `jta_channel.vhd`: no `CFD_MODE` port; `triple_filter.vhd:L99` uses `CFD_MODE='1'` only to swap filter channels but CFD_PROMPT/DELAYED outputs are zeroed when not in use.
3. **Triple-filter present but adapted** — ⚠️ Correction: The triple filter (`triple_filter.vhd`) is **present** in the Majorana branch. However `THRESH_DISC_PROMPT` and `THRESH_DISC_DELAYED` are driven by filtered 15-bit signals from `triple_filter.vhd:L99` (14-bit ADC + sign bit), not raw 24-bit sums. ✅ verified 2026-04-25 — `triple_filter.vhd:L56,L57,L99`
4. **Threshold discriminator input:** The THRESH_DISC_PROMPT/DELAYED signals are 15-bit filtered ADC values (sign-extended 14-bit). The `DISCRIMINATOR_THRESHOLD` port at the `thresh_disc_mach` interface remains **14-bit** (`13 downto 0`). ✅ verified 2026-04-25 — `thresh_disc.vhd:L54`, `jta_channel.vhd:L190-191`
5. ⚠️ **Correction to original file comments:** The header comments in `jta_channel.vhd` (L16-17) and `thresh_disc.vhd` (L21-22) say "threshold uses full 24-bit sums" and "threshold extended to 24-bit" — but the actual VHDL port declarations and signal widths are still 14-bit (threshold) and 15-bit (prompt/delayed). These comments appear to describe **intent or a planned change** that was not fully implemented in the VHDL as of 2015-06-01. ✅ verified 2026-04-25 — cross-checked `thresh_disc.vhd:L54` (`13 downto 0`) vs comments at L21-22.

These changes reflect the Majorana experiment's use of p-type point-contact (PPC) germanium detectors, which have very different pulse shapes from the HPGe coaxial detectors in Gammasphere.

---

## Top-Level Structure (Digitizer.vhd)

### Entity
- **Entity:** `DIGITIZER`
- **Generics:**
  - `SLAVE_MODE : boolean` — if TRUE, this digitizer is a slave (reverses front bus direction, disables cross-channel event veto)
  - `RUN_EXT_FIFO_AT_100MHZ : boolean` — controls the clock speed for the external FIFO path

### Clock Architecture

| Signal | Source | Frequency |
|--------|--------|-----------|
| `clk50` | DCM from SERDES or FBUS | 50 MHz |
| `clk100` | DCM (2x) | 100 MHz (main acquisition clock) |
| `clk200` | DCM (4x) | 200 MHz |
| `clk_osc50` | Local oscillator | 50 MHz |

The external FIFO clock selection:
- `RUN_EXT_FIFO_AT_100MHZ=TRUE`: ext_fifo_proc_clk = clk200, col_fifo_proc_clk = clk100
- `RUN_EXT_FIFO_AT_100MHZ=FALSE`: ext_fifo_proc_clk = clk100, col_fifo_proc_clk = clk50

### Data Flow

```
10x channels:
  [DISC] → [READOUT_MACH] → [CH_ACCEPTED_EVENT_FIFO] ──┐
  ...                                                    ├──> [COLLECTION_MACH] → [COLLECTION_FIFO] → [FIFO_MACH] → External FIFO → IOC
  [DISC] → [READOUT_MACH] → [CH_ACCEPTED_EVENT_FIFO] ──┘
```

Data rates (100 MHz mode): ~3.6 Gb/s into collection, ~3.2 Gb/s to external FIFO.

### Master vs Slave Operation

- **Master:** SERDES is the external clock source; FBUS_LVDS_DIR drives output; FBUS_MDATA is output; cross-channel event veto enabled.
- **Slave:** FBUS_CLK is the external clock source; FBUS_MDATA is input; event veto from front bus gate (VETO_EVENT).

Channel resets:
- `master_channel_reset` released when `reg_master_logic_status[0]=1` (Master Logic Enable) AND pipeline not in reset AND (FBUS not asserting reset in FBUS-clock mode).
- Per-channel reset released when `reg_channel_control[i][0]=1` (channel START) AND `master_channel_reset=0`.

---

## Discriminator (`jta_channel.vhd` / `thresh_disc.vhd`)

### `discriminator` entity (jta_channel.vhd)
The per-channel discriminator in the Majorana branch. Key ports:

| Port | Width | Description |
|------|-------|-------------|
| `RAW_ADC_DATA` | 14 | 2's complement ADC input |
| `reg_p1_window` | 32 | Delay factor p1 |
| `reg_p2_window` | 32 | Delay factor p2 |
| `reg_m_window` | 32 | Delay factor m (moving average window) |
| `reg_k_window` | 32 | Delay factor k |
| `reg_d_window` | 32 | Delay factor d |
| `reg_d2_window` | 32 | Delay factor d2 |
| `DISCRIMINATOR_THRESHOLD` | 14 | Threshold (14-bit at this interface; comments claim 24-bit internally but port confirmed 14-bit — see Key Differences note above) ✅ verified 2026-04-25 — `jta_channel.vhd:L62`, `thresh_disc.vhd:L54` |
| `TRIGGER_POLARITY` | 2 | 01=pos only, 10=neg only, 11=both, 00=none |
| `PREAMP_RESET_DELAY` | 8 | Blanking time after preamp reset |
| `COARSE_DISC_FLAG` | 1 | Unfiltered discriminator output |
| `ACCEPTED_HIT` | 1 | Filtered discriminator (no pileups) |
| `PILEUP_FLAG` | 1 | Indicates pileup state |
| `EVENT_VALID` | 1 | If low, PEQ automatically rejects event |

### `thresh_disc_mach` entity (thresh_disc.vhd)
Internal threshold discriminator and peak finder:
- Runs subtraction: `THRESH_DISC_PROMPT - THRESH_DISC_DELAYED` (15-bit signed, sign bit appended)
- Compares to `DISCRIMINATOR_THRESHOLD` (14-bit from VME)
- In Majorana branch: internally operates on 24-bit M1+M2 sums
- Includes auto-holdoff with configurable `HOLDOFF_CONTROL` (9-bit)
- `STOP_HOLDOFF_AT_PEAK` terminates holdoff early when peak is found
- External discriminator flag injection via `EXTERNAL_DISC_FLAG` + `EXTERNAL_DISC_MODE[1:0]` (added 2013-05-16)

---

## Baseline Tracker (`baseline_tracker.vhd`)

A dedicated baseline tracking state machine, shared with standard DGS. States:

| State | Description |
|-------|-------------|
| `IDLE` | Initial/startup state |
| `ENTER_FLAG` | Entry into tracking after reset |
| `TRACKING` | Active baseline estimation |
| `DISABLE_TRACK` | Tracking suspended (discriminator fired) |
| `DISC_DELAY` | Counting off delay before re-tracking |

Key behavior:
- On power-up, delay chain fills with zeros, baseline is assumed from first ADC data
- After a discriminator firing: idle for `BASELINE_DELAY × 10.24 µs` (length of T buffer) ✅ verified 2026-04-26 — `baseline_tracker.vhd:L34` (comment: "after a discriminator firing, the baseline tracker is idle for (BASELINE_DELAY) * 10.24us (length of T)")
- Running sum is 24-bit (`RUNNING_BASELINE_SUM`); output `SAMPLED_BASELINE` is 24-bit ✅ verified 2026-04-26 — `baseline_tracker.vhd:L40,L55` (signal declarations)
- `BASELINE_SPEED[2:0]` controls tracking rate (filter speed) ✅ verified 2026-04-26 — `baseline_tracker.vhd:L27`
- `BASELINE_START[13:0]` sets the initial baseline seed (VME programmable) ✅ verified 2026-04-26 — `baseline_tracker.vhd:L25`; `L200`: `RUNNING_BASELINE_SUM <= BASELINE_START & "0000000000"` (14-bit seed × 1024 = 24-bit accumulator)
- Chain validity flag (`CHAIN_INVALID`) released after `2M + K + D + D2` samples ✅ verified 2026-04-26 — `baseline_tracker.vhd:L81-82` (comment: "2M + K + D + D2 samples until ADC samples start filling the T buffer")

---

## SERDES Interface

Same DS92LV18 SERDES as standard DGS digitizer:
- `SERDES_RX_DIN[17:0]` / `SERDES_TX_DOUT[17:0]` — 18-bit LVDS
- `SERDES_LOCK_N` — active-low lock indicator (inverted to `serdes_lock` internally)
- `SERDES_SYNC` / `SERDES_RX_CLK50` / `SERDES_TX_CLK50` — sync and clocking
- `SERDES_LINE_LE` / `SERDES_LOCAL_LE` — latch enables

---

## Front Bus (FBUS)

10-channel shared parallel bus between master and slave digitizers:
- `FBUS_MDATA[17:0]` — bidirectional, 18-bit (Master output, Slave input)
- `FBUS_SDATA[2:0]` — bidirectional (Slave output, Master input)
- `FBUS_WOR_IN/OUT[1:0]` — Wired-OR lines (external 10 kΩ pullup)
- `FBUS_LVDS_DIR` — controls LVDS transceiver direction
- Direction of FBUS_MDATA controlled by `SLAVE_MODE` generic

---

## External FIFO

Dual external FIFOs (same hardware as standard DGS):
- `FIFO_DATA[35:0]` — 36-bit wide data output to FIFO
- `FIFO_EF_N[1:0]` / `FIFO_FF_N[1:0]` — empty/full flags (dual)
- `FIFO_HF_N` — half-full flag
- `FIFO_PAE_N` / `FIFO_PAF_N` — programmable almost-empty/full
- `FIFO_RCLK[1:0]` / `FIFO_WCLK[1:0]` — independent read/write clocks
- Master reset via `FIFO_MRS_N` / partial reset via `FIFO_PRS_N` / retransmit via `FIFO_RT_N`

---

## Firmware Identification

Board ID register layout (same convention as standard DGS):
- `bits[31:16]` — unused
- `bits[15:12]` — PCB revision (0x2 = RevB; RevC boards are firmware-identical to B)
- `bits[11:8]` — firmware type: **0xC** = DGS/DSSD LED Digitizer, **0xD** = DGS/DSSD CFD Digitizer
- `bits[7:4]` — major code revision (0xB)
- `bits[3:0]` — minor code revision (0xB)

Full code revision value: `0x00004CBB` (LED/master) or `0x00004DBB` (CFD/slave) ✅ verified 2026-04-26 — `Digitizer.vhd:L391` (`regin_code_revision <= X"00004" & X"D" & cCODE_VERSION_MAJOR & cCODE_VERSION_MINOR when(SLAVE_MODE = TRUE) else X"00004" & X"C" & cCODE_VERSION_MAJOR & cCODE_VERSION_MINOR`)

---

## File Inventory

| File | Description |
|------|-------------|
| `Digitizer.vhd` | Top-level entity (1547 L) |
| `Digitizer_pkg.vhd` | Package: type definitions, array types |
| `Digitizer.ucf` | UCF pin constraints |
| `jta_channel.vhd` | Per-channel discriminator (1624 L) |
| `thresh_disc.vhd` | Threshold discriminator + peak finder (636 L) |
| `baseline_tracker.vhd` | Baseline tracking state machine (417 L) |
| `Channel_FIFO_Readout_Mach.vhd` | Channel readout state machine |
| `Channel_Readout_Controller.vhd` | Readout controller |
| `Channel_Readout_Mach.vhd` | Channel readout FSM |
| `chan_trigger_control.vhd` | Channel trigger control |
| `CLOCK_MANAGER.vhd` | DCM clock management |
| `dc_balance_mach.vhd` | DC balance for SERDES |
| `DCM_CONTROLLER.vhd` | DCM lock controller |
| `decimator.vhd` | Waveform decimator |
| `Event_Header_FIFO.vhd` | Event header FIFO |
| `event_packer.vhd` | Event packer |
| `event_data_fifo.vhd` | Event data FIFO |
| `Front_Bus.vhd` | Front bus logic |
| `pileup_processor.vhd` | Pileup detection/rejection |
| `Register_Logic.vhd` | VME register logic |
| `Registers.vhd` | VME register map |
| `SERDES_RX_Mach.vhd` | SERDES RX state machine |
| `SERDES_TX_Mach_DGS.vhd` | SERDES TX state machine |
| `Timestamp_Generator.vhd` | 48-bit timestamp counter |
| `Trigger_Mux.vhd` | Trigger mux (IntAcptAll / ExtTTL / ExtTTCL) |
| `pehq.vhd` | Peak event hold queue |
| `filtered_subtraction.vhd` | Filtered subtraction for discriminator |
| `Phase_Hunter.vhd` | ADC phase alignment |
| `sync_capture_controller.vhd` | Sync capture state machine |
| `jta_bram_dlybuf.vhd` | BRAM delay buffer |
| `single_filter.vhd` | Single moving-average filter stage |
| `triple_filter.vhd` | Triple filter (NOT used in Majorana branch) |
| `cfd_disc.vhd` | CFD discriminator (NOT used in Majorana branch) |

---

## Relationship to DGS Digitizer

The Majorana branch shares ~90% of its structure with `FPGA/DIG/` (main DGS digitizer). Files that are identical or near-identical: `CLOCK_MANAGER.vhd`, `SERDES_RX_Mach.vhd`, `SERDES_TX_Mach_DGS.vhd`, `Timestamp_Generator.vhd`, `dc_balance_mach.vhd`, `event_packer.vhd`, `pileup_processor.vhd`, `Register_Logic.vhd`, `Registers.vhd`, and most FIFO modules. The divergence is entirely in the per-channel discriminator path (`jta_channel.vhd`, `thresh_disc.vhd`).

Also present: **LBL Digitizer** (`FPGA/others/LBL_Digitizer/`) — the GRETINA digitizer firmware (PDF documentation present); not documented in the KB as of 2026-04-25.

---

## Cross-References

| File | Relationship |
|------|--------------|
| `deep_fpga_DIG.md` | Main DGS digitizer firmware (trunk from which Majorana branched) |
| `deep_fpga_DIG_channel.md` | Per-channel signal processing detail (shared with Majorana) |
| `fpga.md` | FPGA firmware overview including this branch |
| `hardware_architecture.md` | HPGe + BGO detector hardware context |
| `preamp_reset_readme.md` | Preamp reset detection (FPGA feature in shared codebase) |

---

*Created: 2026-04-25 | Last reviewed: 2026-04-25*
