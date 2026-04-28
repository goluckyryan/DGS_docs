# Preamp Reset Handling in the DGS Digitizer FPGA

Stability: C3 - Structural / stable

**Source:** `FPGA/DIG/PREAMP_RESET.md` (Ryan Tang, 2026-04-17)  
**Audience:** Undergraduates / anyone new to the DGS system  
**Scope:** `thresh_disc.vhd`, `jta_channel.vhd`, `Digitizer.vhd`, `SERDES_RX_Mach.vhd`

---

## Table of Contents

1. [What is a Preamp Reset?](#1-what-is-a-preamp-reset)
2. [How the FPGA Detects a Preamp Reset](#2-how-the-fpga-detects-a-preamp-reset)
3. [What Happens When a Reset Is Detected](#3-what-happens-when-a-reset-is-detected)
4. [Side Effect: BGO Veto Gate](#4-side-effect-bgo-veto-gate)
5. [Timestamp of the Last Reset (PARST)](#5-timestamp-of-the-last-reset-parst)
6. [Remote Reset via Frame 15 (FRONT_END_RESET)](#6-remote-reset-via-frame-15-front_end_reset)
7. [Summary Flow](#7-summary-flow)
8. [Key Registers](#8-key-registers)
9. [Key Source Files](#9-key-source-files)

---

## 1. What is a Preamp Reset?

A germanium detector preamplifier works like a leaky bucket: charge from each gamma-ray hit accumulates on a capacitor, building up a step on the output voltage. Eventually the capacitor fills up and the preamp **resets** — it rapidly discharges, swinging the output voltage hard (up or down, depending on polarity) before settling back to baseline.

This reset is **not a gamma-ray event.** It's a maintenance glitch. If the FPGA doesn't recognize and ignore it, it will:
- Fire the discriminator falsely (fake events)
- Mess up the baseline tracker
- Corrupt energy measurements for the next few microseconds

The DGS digitizer has dedicated logic to detect a reset and **kill the channel** for a programmable blanking time until things settle.

---

## 2. How the FPGA Detects a Preamp Reset

The ADC digitizes the preamp output continuously at 100 MHz. The firmware watches for extreme ADC values using four threshold levels baked into `jta_channel.vhd`:

```
ADC value < 255     → LOLO (way too low)   → ADC_VAL_WARN(3)
ADC value < 2047    → LO   (too low)       → ADC_VAL_WARN(2)
ADC value > 14336   → HI   (too high)      → ADC_VAL_WARN(1)
ADC value > 16128   → HIHI (way too high)  → ADC_VAL_WARN(0)
```
✅ verified 2026-04-17 — thresh_disc.vhd:L922-925 (20230809 tag); comments on L323-326 give exact binary values

A normal gamma-ray pulse never swings that far. A preamp reset does.

**Detection logic in `thresh_disc.vhd` (`PA_RST_DET_BLK`):**

- For a **positive-polarity** detector: watch for a rising edge on `ADC_VAL_WARN(3)` (LOLO — the reset swings the output hard negative first)
- For a **negative-polarity** detector: watch for a rising edge on `ADC_VAL_WARN(0)` (HIHI — the reset swings hard positive first)

When the edge is detected → `RESET_FLAG` pulses high for one clock cycle.

> This block is only compiled in when the generic `ENABLE_HILO_DET = TRUE`. The coarse BGO channels (channels 0–4, instantiated without that generic) skip it entirely — they don't need it.
> ✅ verified 2026-04-17 — jta_channel.vhd:L683 (`ENABLE_HILO_DET => FALSE` for BGO, L1102 `ENABLE_HILO_DET => TRUE` for Ge, 20230809 tag)
> ⚠️ Note: `ENABLE_HILO_DET` / `ADC_VAL_WARN` / `PA_RST_DET_BLK` are only present in 20230809 firmware — **not** in the currently deployed 20211118 (DIG rev 0x4CD8). The deployed firmware uses an older preamp reset scheme without these signals.

---

## 3. What Happens When a Reset Is Detected

The discriminator state machine in `thresh_disc.vhd` has these states:

```
IDLE → INITIAL_FALSE_EDGE → WAIT_EDGE → PULSE → WAIT_DELAY → PREAMP_DELAY
```

When `RESET_FLAG` pulses high from **any** of the normal states (`WAIT_EDGE`, `PULSE`, `WAIT_DELAY`), the state machine immediately jumps to:

```
→ PREAMP_DELAY
```

In `PREAMP_DELAY`:
- `CHANNEL_KILLED` is asserted high
- The discriminator is suppressed — no new events are accepted
- The channel waits until a holdoff counter expires

The holdoff duration is:

```
PREAMP_RESET_DELAY[7:0] × 512 clock cycles
```

At 100 MHz (10 ns/cycle):  
`PREAMP_RESET_DELAY = 1` → 5.12 µs blanking  
`PREAMP_RESET_DELAY = 10` → 51.2 µs blanking  
✅ verified 2026-04-17 — thresh_disc.vhd:L634 comment "PREAMP_RESET_DELAY * 512 counts"; L661 `DISCBIT_HOLDOFF_COUNT <= PREAMP_RESET_DELAY & "000000000"` (shift left 9 = ×512) in 20230809 tag; same counter structure confirmed in 20211118 tag.

This register lives at `reg_led_threshold[i](23:16)` in `Digitizer.vhd`. The blanking can be disabled entirely with `PREAMP_RESET_DELAY_EN = 0` (register bit `reg_channel_control(i)(3)`).

---

## 4. Side Effect: BGO Veto Gate

When a Ge channel is blanked (`CHANNEL_KILLED = 1`), the FPGA can use this as a **gate for the paired BGO channels** — but only when the BGO channel's `external_disc_mode` bits are set to `"101"`.

In `Digitizer.vhd` (mode `"101"`, Front Bus Left configuration):
```vhdl
external_disc_flag(i) <= not(CHANNEL_KILLED(i+5));
```
✅ verified 2026-04-18 — `Digitizer.vhd:L1121-1126` (DGS BuildBranch, mode "101" case: "use the CHANNEL_KILLED of the Ge channel (5:9) as a gate for the BGO channel (0:4)")

Channels 0–4 are BGO scintillators. Channels 5–9 are the paired Ge detectors. When Ge channel `i+5` is in reset-kill mode, BGO channel `i` is also suppressed. This prevents the BGO from triggering on the electromagnetic noise that often accompanies a preamp reset.

> **Note:** This BGO gating is not hardwired — it requires the BGO channel's `reg_external_disc_mode` bits [2:0] to be set to `"101"` (mode 101). The `external_disc_flag` is a mux with multiple modes (000–111); mode 101 on FRONT_BUS_LEFT boards is specifically the CHANNEL_KILLED gate. Other modes route different sources (REMOTE_DISCBIT, AUX_DIN, VME_EXT_DISC_REQUEST, etc.).
✅ verified 2026-04-18 — `Digitizer.vhd:L968-1130` (full external_disc_flag mux case statement for channels 0–8)

---

## 5. Timestamp of the Last Reset (PARST)

The firmware records **when** the preamp reset happened, so analysis code can:
- Know how recently the preamp reset
- Flag events that occurred right after a reset (still recovering)
- Correct for baseline disturbances

**`PARST_TSM`** — "Preamp Reset Timestamp Match" — is a 1-bit flag in the event header (Word 13, bit 12). It means: the upper 20 bits of the timestamp of the last preamp reset match the current event's timestamp. In other words: the reset was recent (within the current ~10 ms timestamp rollover window). ✅ verified 2026-04-18 — `Event_Header_FIFO.vhd:L381,L477` (`header(13)(12) <= event_data.PARST_TSM` in both LED and CFD header modes)

**`TS_OF_LAST_PREAMP_RESET[23:0]`** — the lower 24 bits of the 48-bit timestamp when the reset was detected. Stored in the multiplexed header field `MPX_FIELD` (Word 11, bits 23:0) when `CAPTURE_PARST_TS = 1` (controlled by bit 7 of `reg_d3_window`). ✅ verified 2026-04-18 — `Event_Header_FIFO.vhd:L372,L468` (`header(11)(23 downto 0) <= event_data.MPX_FIELD` in both LED and CFD modes); `jta_channel.vhd:L51` confirms CAPTURE_PARST_TS controls whether MPX_FIELD holds preamp reset timestamp or 2nd early energy.

In `jta_channel.vhd`:
```vhdl
if (CAPTURE_PARST_TS = '1') then
    -- MPX_FIELD = TS_OF_LAST_PREAMP_RESET[23:0]
else
    -- MPX_FIELD = EARLY_PRE_RISE_SUM (a second baseline window)
end if;
```

So bit 7 of `reg_d3_window` is a mode switch: timestamp of last reset vs. early pre-rise energy sum.

---

## 6. Remote Reset via Frame 15 (FRONT_END_RESET)

The preamp reset can also be **commanded remotely** by the MTRG (Master Trigger) over the SERDES link. Frame 15 of the 20-frame command cycle is the "Async Command" frame. One of its commands is `FRONT_END_RESET`.

In `SERDES_RX_Mach.vhd`:
```vhdl
FRONT_END_RESET_FLAG <= '1';   -- one clock pulse
```
✅ verified 2026-04-19 — `SERDES_RX_Mach.vhd:L312` (20211118 DGS_TAG_20180607_TWEAK branch: `FRONT_END_RESET_FLAG <= '1'` in Frame 15 async command decode)

In `Digitizer.vhd`, `front_end_reset_flag` is declared as a signal (L250) and connected as `FRONT_END_RESET_FLAG` output of `SERDES_RX_Mach` (L1883). However, in the deployed 20211118 DGS firmware, `LOAD_vals` is wired only to `reg_channel_pulsed_control(0)` (L990) — **`front_end_reset_flag` does not drive `LOAD_vals` in the deployed build**.

> ⚠️ The `LOAD_DELAYS_COMBO <= front_end_reset_flag OR reg_channel_pulsed_control(0)` expression does not appear in the 20211118 DGS firmware (DGS_TAG_20180607_TWEAK branch) or the active fpga/ tree DGS build. It may describe a future/unreleased firmware variant. `front_end_reset_flag` appears to be received but not used in the current deployed DGS Digitizer.vhd. ✅ verified 2026-04-19 — `Digitizer.vhd:L250,L990,L1883` (20211118 tag + active fpga/ tree: no LOAD_DELAYS_COMBO signal exists; LOAD_vals => reg_channel_pulsed_control(0) only)

Think of `FRONT_END_RESET` as a soft reset that re-arms the delay chain without touching the ADC or the SERDES link (when implemented).

This is separate from the hardware-detected preamp reset above. It does **not** trigger the `PREAMP_DELAY` kill state.

---

## 7. Summary Flow

```
ADC output swings hard (LOLO or HIHI)
        │
        ▼
PA_RST_DET_BLK (thresh_disc.vhd)
        │  RESET_FLAG = '1' (one clock)
        ▼
Discriminator state machine → PREAMP_DELAY
        │  CHANNEL_KILLED = '1'
        │  Wait PREAMP_RESET_DELAY × 512 cycles
        ▼
WAIT_EDGE (normal operation resumes)
        │
        ├─→ BGO channel gate released (CHANNEL_KILLED deasserted)
        └─→ TS_OF_LAST_PREAMP_RESET latched in event packets
```

---

## 8. Key Registers

| Register | Bit(s) | Function |
|----------|--------|----------|
| `reg_led_threshold[ch](23:16)` | 8 bits | Blanking duration after reset (× 512 cycles at 100 MHz) ✅ verified 2026-04-27 — `Digitizer.vhd:L1004` (`PREAMP_RESET_DELAY => reg_led_threshold(i)(23 downto 16)`, DGS_TAG_20180607_TWEAK) |
| `reg_channel_control[ch](3)` | 1 bit | Enable/disable blanking (`PREAMP_RESET_DELAY_EN`) ✅ verified 2026-04-27 — `Digitizer.vhd:L1005` (`PREAMP_RESET_DELAY_EN => reg_channel_control(i)(3)`, DGS_TAG_20180607_TWEAK) |
| `reg_d3_window[ch](7)` | 1 bit | `CAPTURE_PARST_TS`: 1 = store reset timestamp in MPX_FIELD; 0 = store early pre-rise sum ✅ verified 2026-04-27 — `SumOverRise/Source/Digitizer.vhd:L1166` (`CAPTURE_PARST_TS => reg_d3_window(i)(7)`); note: internal `reg_d3_window` in Digitizer is 32-bit; port to `jta_channel` uses only bits [6:0] |

---

## 9. Key Source Files

| File | Role in preamp reset |
|------|-------------------------------|
| `thresh_disc.vhd` | Detects reset via ADC_VAL_WARN; runs PREAMP_DELAY kill state |
| `jta_channel.vhd` | Passes PREAMP_RESET_DELAY registers; latches PARST timestamp into PEHQ |
| `Digitizer.vhd` | Routes CHANNEL_KILLED to BGO gates; connects LOAD_DELAYS_COMBO |
| `SERDES_RX_Mach.vhd` | Decodes Frame 15 FRONT_END_RESET command from MTRG |

---

## See Also

- [`DGS_PVs.md`](DGS_PVs.md) — `preamp_reset_delayN` / `preamp_reset_delay_enN` PV descriptions
- [`DIG_firmware_expert.md`](DIG_firmware_expert.md) — DIG firmware expert guide; modes, trigger_mux_select, aux I/O
- [`deep_fpga_DIG.md`](deep_fpga_DIG.md) — DIG firmware architecture overview
- [`VME_registers.md`](VME_registers.md) — VME register map; `reg_led_threshold[ch](23:16)` and `reg_channel_control[ch](3)` physical addresses
- [`ioc.md`](ioc.md) — EPICS IOC where these PVs are instantiated

---

*Added to knowledge base: 2026-04-17 from FPGA/DIG/PREAMP_RESET.md*  
*Verified 2026-04-17: ADC thresholds, PA_RST_DET_BLK, ENABLE_HILO_DET confirmed in 20230809 firmware tag (thresh_disc.vhd, jta_channel.vhd). PREAMP_DELAY state machine and CHANNEL_KILLED confirmed in both 20211118 and 20230809 tags. Key note: ADC_VAL_WARN-based detection is a 20230809 addition — currently deployed firmware (20211118, rev 0x4CD8) uses older preamp reset logic without these signals.*
