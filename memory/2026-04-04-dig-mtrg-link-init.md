# Session: 2026-04-04 11:24:30 UTC

- **Session Key**: agent:main:discord:channel:1489370812875145408
- **Session ID**: 61001509-8877-4a45-9282-d1494224e377
- **Source**: discord

## Conversation Summary

assistant: We worked in `raw_FPGA/DIG/` — that's where the preamp reset README lives now.

The source code we read through to write it:

- `DGS_tools_pack/DGS_SVN/dgs/Digitizer/MAIN_FPGA/BuildBranches/DGS/Source/thresh_disc.vhd` — the detection and kill state machine
- `jta_channel.vhd` — PARST timestamp latching, PEHQ integration
- `Digitizer.vhd` — BGO gate logic, register wiring, LOAD_DELAYS
- `SERDES_RX_Mach.vhd` — Frame 15 FRONT_END_RESET decode

The README ended up at `raw_FPGA/DIG/PREAMP_RESET.md` and got pushed to GitLab.
user: Sender (untrusted metadata):
```json
{
  "label": "openclaw-control-ui",
  "id": "openclaw-control-ui"
}
```

[Fri 2026-04-03 20:00 CDT] How about other folders in DGS_toos_pack ?
assistant: Here's what we have documented vs. not:

**Well documented (in `memory/dgs/`):**
- `raw_FPGA/` — full deep dive (DIG, RTRG, MTRG, build toolchain)
- `ANLDAQ/` — DAQ GUI + TCP receiver
- `collectorboxpi/` — soft IOC, HV control, PXE boot
- `ioc/` — EPICS IOC config, boot scripts, firmware
- `lnfill/` — LN2 filling system
- `vxworks/` — cross-compilation, build pipeline
- `dgs_analysis/` — fastEventConstructor, parquet_pysort

**Less explored:**
- `DGS_SVN/` — legacy SVN archive; we only dipped in to read source files. No dedicated README written from it.

The SVN tree is the historical source for everything — it has the original Digitizer, GRETINA trigger firmware, older MTRG variants, etc. Worth exploring if you want undergrad docs for any of those. Want to dig into any specific folder?
user: Sender (untrusted metadata):
```json
{
  "label": "openclaw-control-ui",
  "id": "openclaw-control-ui"
}
```

[Fri 2026-04-03 20:41 CDT] back to the FPGA and MTRG, what is L init stat 1, 2, 4, 8 ?
assistant: Good question. Let me look at the actual link init code.
assistant: Got it — the full state machine is right there. Here's what `STATE_MON` values mean:

| Value | State | What's happening |
|-------|-------|-----------------|
| `0x0` | `INIT` | Reset just released. All SERDES disabled. Transitions immediately to EN_SERDES. |
| `0x1` | `EN_SERDES` | Transmitters and receivers enabled, but no SYNC yet. One clock, then moves to SYNC (unless RETRY=1, which holds here). |
| `0x2` | `SYNC` | SYNC pattern asserted on all 8 links. One clock, then moves to WAIT_FOR_LOCK. |
| `0x3` | `WAIT_FOR_LOCK` | Sitting here sending SYNC, waiting for all 8 downstream SERDES to assert LOCK. This can take a long time — SYNC has to propagate all the way down to the DIGs and come back up through the RTRGs. Stuck here if hardware is dead. |
| `0x4` | `ALL_LOCKED_UP` | All links locked. `ALL_LOCKED=1` asserted. Still sending SYNC. Waiting for software to send `LOCK_ACK` to acknowledge. |
| `0x5` | `ACKED` | Software acknowledged. **SYNC released — real data now flows.** Steady-state normal operation. Watching for any link to drop out. |
| `0x6` | `ERROR` | A link fell out of lock after once being established. `LOCK_ERROR=1`. System keeps running but flag is asserted. Requires a reset (`RETRY=1`) to recover. |

**In practice:** you expect to see `0x1 → 0x2 → 0x3` (waiting) → `0x4` (locked) → `0x5` (acked, running). If it's stuck on `0x3`, a link never came up. If it's on `0x6`, a link dropped mid-run.

The `STATE_MON` bits land in `MISC_STAT` register bits `[11:8]`, readable over VME.
user: Sender (untrusted metadata):
```json
{
  "label": "openclaw-control-ui",
  "id": "openclaw-control-ui"
}
```

[Sat 2026-04-04 01:53 CDT] do you knwo what is DISCRIMINATOR_DELAY in RTR ?
assistant: Now I have enough. Let me read a bit more of the context around `BIT_DELAY_BLK`.
assistant: Got the full picture. Here's the answer:

---

**`DISCRIMINATOR_DELAY`** in the RTRG is a **per-channel, per-bit programmable digital delay line** on the discriminator bits arriving from the DIG.

### What it is

Each DIG sends 10 discriminator bits (channels 0–9) to the Router over the SERDES link. The Router receives these as `RECOVERED_DATA[9:0]`. Before those bits are used for trigger logic, each one passes through an independent delay buffer:

```
RECOVERED_DATA[i]  →  DPRAM delay buffer  →  DELAYED_DATA[i]
```

The delay buffer is a **32-entry distributed RAM shift register** (5-bit address, depth 2⁵ = 32 samples). At 50 MHz that's:

```
1 tap = 20 ns
Max delay = 32 × 20 ns = 640 ns
```

Each channel has its own 5-bit delay value: `BIT_DELAY(i)` — that's `DISC_DELAY[board][channel]` from the VME registers. 80 registers total (8 boards × 10 channels each).

### Why it exists

The comment in the code says it directly:

> *"to allow the user to time-offset the Ge discriminators relative to the BGO discriminators — this 'offset' parameter would compensate for time-walk in the discriminators due to variance in rise time."*

In other words: different detectors have different cable lengths, different detector capacitances, different rise times. A Ge pulse and the BGO pulse from the same gamma-ray scatter event may arrive at the Router at slightly different times. The `DISCRIMINATOR_DELAY` lets you nudge each channel's bit independently so that **Ge and BGO hits that are physically coincident arrive at the coincidence logic at the same time**.

### How it's loaded

Write the 5-bit delay value to the VME register (`0x1000` + offset for each channel), then pulse `LOAD_BIT_DELAY` — that reloads the address into both ports of the shift register RAM simultaneously.

### Where it feeds

`DELAYED_DATA[9:0]` is used when `CLEAN_DIRTY[15] = '1'` (the delay-path mode bit). In that mode the coincidence logic (`discriminator_mach`, Ge/BGO clean/dirty event formation) uses the time-aligned delayed bits instead of the raw recovered bits.
user: Sender (untrusted metadata):
```json
{
  "label": "openclaw-control-ui",
  "id": "openclaw-control-ui"
}
```

[Sat 2026-04-04 01:58 CDT] are you sure, may be my fault, the full PV is DISCRIMINATOR_DELAY_X_Y, where X is A to H, Y is 0 -9
assistant: Good catch — that's a different beast then. Let me look at the actual PV names.
assistant: OK — so `X` is **A–H** (which DIG board, i.e. which of the 8 SERDES input links on the RTRG) and `Y` is **0–9** (which of the 10 discriminator channels on that DIG). That matches exactly what I described — just the naming is from the RTRG's perspective of its 8 input links (A–H) rather than board numbers.

So my earlier answer was correct in substance, just needed this clarification:

- **`DISCRIMINATOR_DELAY_A_0`** = delay for channel 0 of the DIG connected to RTRG link A
- **`DISCRIMINATOR_DELAY_B_3`** = delay for channel 3 of the DIG on link B
- ...and so on up to `H_9`

80 PVs total (8 links × 10 channels). Each is a **5-bit value (0–31)**, giving 0–640 ns of delay in 20 ns steps at 50 MHz, applied to that individual discriminator bit before the Ge/BGO coincidence logic sees it.

The DESC field confirms it: *"Delay to apply to discriminator"* — used to time-align Ge and BGO hits from the same physical detector module that might arrive at slightly different times due to cable length or signal rise-time differences.
user: Sender (untrusted metadata):
```json
{
  "label": "openclaw-control-ui",
  "id": "openclaw-control-ui"
}
```

[Sat 2026-04-04 02:00 CDT] interesting, do you knwo the unit of the delay ?
assistant: From the VHDL we already read in `chan_in.vhd`:

```vhdl
DEPTH_pwr2 => 5   -- depth is 2^5 = 32 samples
```

The shift register runs on `mclk` — let me confirm what that clock is.
assistant: Confirmed. `mclk` = `switched_master_clock` = **50 MHz**.

So the unit is:

```
1 tap = 1 clock cycle at 50 MHz = 20 ns
Range: 0–31 taps = 0–620 ns  (not 640 — it's 0 to 31 inclusive)
```

So when you write `DISCRIMINATOR_DELAY_A_3 = 5`, you're adding **100 ns** of delay to channel 3 on link A before it hits the coincidence logic.
