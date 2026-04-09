# Preamp Reset Handling in the DGS Digitizer FPGA

**Audience:** Undergraduates joining the DGS group  
**Scope:** `thresh_disc.vhd`, `jta_channel.vhd`, `Digitizer.vhd`, `SERDES_RX_Mach.vhd`

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
✅ verified 2026-04-08 — `thresh_disc.vhd:L323-326` (MAIN_FPGA_TAGS/20230809, comments confirm all four threshold values)

A normal gamma-ray pulse never swings that far. A preamp reset does.

**Detection logic in `thresh_disc.vhd` (`PA_RST_DET_BLK`):**

- For a **positive-polarity** detector: watch for a rising edge on `ADC_VAL_WARN(3)` (LOLO — the reset swings the output hard negative first)
- For a **negative-polarity** detector: watch for a rising edge on `ADC_VAL_WARN(0)` (HIHI — the reset swings hard positive first)

When the edge is detected → `RESET_FLAG` pulses high for one clock cycle.

> This block is only compiled in when the generic `ENABLE_HILO_DET = TRUE`. The coarse BGO channels (channels 0–4, instantiated without that generic) skip it entirely — they don't need it. ✅ verified 2026-04-08 — `thresh_disc.vhd:L328` (`PA_RST_DET_BLK: if (ENABLE_HILO_DET = TRUE) generate`)

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

This register lives at `reg_led_threshold[i](23:16)` in `Digitizer.vhd`. The blanking can be disabled entirely with `PREAMP_RESET_DELAY_EN = 0` (register bit `reg_channel_control(i)(3)`).

---

## 4. Side Effect: BGO Veto Gate

When a Ge channel is blanked (`CHANNEL_KILLED = 1`), the FPGA uses this as a **gate for the paired BGO channels**.

In `Digitizer.vhd` (Front Bus Left configuration):
```vhdl
external_disc_flag(i) <= not(CHANNEL_KILLED(i+5));
```
✅ verified 2026-04-08 — `Digitizer.vhd:L1117` (20230809 tag; comment: "use CHANNEL_KILLED of Ge channel (5:9) as gate for BGO channel (0:4)")

Channels 0–4 are BGO scintillators. Channels 5–9 are the paired Ge detectors. When Ge channel `i+5` is in reset-kill mode, BGO channel `i` is also suppressed. This prevents the BGO from triggering on the electromagnetic noise that often accompanies a preamp reset.

---

## 5. Timestamp of the Last Reset (PARST)

The firmware records **when** the preamp reset happened, so analysis code can:
- Know how recently the preamp reset
- Flag events that occurred right after a reset (still recovering)
- Correct for baseline disturbances

**`PARST_TSM`** — "Preamp Reset Timestamp Match" — is a 1-bit flag in the event header (Word 13, bit 12). It means: the upper 20 bits of the timestamp of the last preamp reset match the current event's timestamp. In other words: the reset was recent (within the current ~10 ms timestamp rollover window).

**`TS_OF_LAST_PREAMP_RESET[23:0]`** — the lower 24 bits of the 48-bit timestamp when the reset was detected. Stored in the multiplexed header field `MPX_FIELD` (Word 11, bits 23:0) when `CAPTURE_PARST_TS = 1` (controlled by bit 7 of `reg_d3_window`).

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

In `Digitizer.vhd`:
```vhdl
LOAD_DELAYS_COMBO <= front_end_reset_flag OR reg_channel_pulsed_control(0);
```
✅ verified 2026-04-08 — `Digitizer.vhd:L1158,L1189` (DIG/MAIN_FPGA/BuildBranches/DGS/Source)

`LOAD_DELAYS_COMBO` is passed to each channel as `LOAD_vals` — it reloads all programmable delay values from registers. Think of it as a soft reset that re-arms the delay chain without touching the ADC or the SERDES link.

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
| `reg_led_threshold[ch](23:16)` | 8 bits | Blanking duration after reset (× 512 cycles at 100 MHz) | ✅ verified 2026-04-08 — `Digitizer.vhd:L1193`
| `reg_channel_control[ch](3)` | 1 bit | Enable/disable blanking (`PREAMP_RESET_DELAY_EN`) | ✅ verified 2026-04-08 — `Digitizer.vhd:L1194`
| `reg_d3_window[ch](7)` | 1 bit | `CAPTURE_PARST_TS`: 1 = store reset timestamp in MPX_FIELD; 0 = store early pre-rise sum | ✅ verified 2026-04-08 — `Digitizer.vhd:L1168` (20230809 tag)

---

## 9. Key Files

| File | What it does for preamp reset |
|------|-------------------------------|
| `thresh_disc.vhd` | Detects reset via ADC_VAL_WARN; runs PREAMP_DELAY kill state |
| `jta_channel.vhd` | Passes PREAMP_RESET_DELAY registers; latches PARST timestamp into PEHQ |
| `Digitizer.vhd` | Routes CHANNEL_KILLED to BGO gates; connects LOAD_DELAYS_COMBO |
| `SERDES_RX_Mach.vhd` | Decodes Frame 15 FRONT_END_RESET command from MTRG |

---

## 10. Cross-References

- `dgs/DIG_firmware_expert.md` — Full DIG firmware reference; covers all modes, discriminator config, pileup, readout
- `dgs/deep_fpga_DIG.md` — Lower-level FPGA architecture: ADC pipeline, FIFO, SERDES, pole-zero
- `dgs/data_structures.md` — DIG event packet format: `PARST_TSM` bit (Word 13, bit 12), `MPX_FIELD` (Word 11)
- `dgs/ttcl.md` — Frame 15 (Async Command frame) definition — source of `FRONT_END_RESET` command
- `dgs/VME_registers.md` — VME register addresses: `reg_led_threshold` (0x0080+ch×4), `reg_channel_control` (0x0040+ch×4), `reg_d3_window` (0x0240+ch×4)
- `dgs/pole_zero.md` — Pole-zero correction context: why baseline stability after a reset matters
- `dgs/QUEUE.md` (workspace) — `Python script to set preamp reset PVs` task (pending)

---
*Source: `DGS_tools_pack/FPGA/` preamp reset documentation + `ANL Digitizer Firmware for Experts.pdf`. Created: 2026-04-05. Cross-refs added: 2026-04-09.*
