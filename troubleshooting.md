# DGS Troubleshooting Guide

**Sources:** [wiki: Some Problems and Their Solutions](https://wiki.anl.gov/gsdaq/Some_problems_and_their_solutions) + operational experience

---

## General Approach: Things Are Broken and You Don't Know Why

### Step 1 — Check IOC status

Open **DGS Commander** main screen → click **VME Status** → brings up IOC Summary page.

The IOC Summary shows whether all VME crate controllers are responding. A non-talking IOC will be flagged (e.g. "IOC 9 not talking").

### Step 2 — Check the broken IOC's terminal

Open the terminal window for the miscreant IOC from DGS Commander.

---

## Problem: IOC Not Talking — Ethernet Issues

**Symptom:** IOC never loads; boot message shows:
```
0x3ff73940 (tBoot): miiPhyInit check cable connection
```

**Cause:** Broken or disconnected Ethernet cable between IOC and network switch.

**Solution:**
1. Go to the shack, trace the Ethernet cable from the IOC back to the router
2. Verify the green "link" light is on for that IOC's port
3. Unplug and re-seat both ends of the cable
4. If no link after reseating → try one power cycle of the IOC
5. If still no link → test the cable by plugging both ends into unused router ports; both green link lights must illuminate — if not, **replace the cable**

---

## Problem: Router Not Locking to Digitizer Data

**Symptom:** Router counter shows a channel dropping in/out of lock.

**Common causes:**
- SERDES accidentally disabled
- SYNC bit still set in `LINK_LRU_CTRL` register of the Router — data not actually being sent to Master
- Cable length mismatch (jitter budget exceeded)

**Check:** Use Router channel FIFOs to verify real discriminator data (bits 9:0) is arriving.

*(Source: [wiki: Triggers and Digitizers](https://wiki.anl.gov/gsdaq/Triggers_and_digitizers) — see `wiki_gsdaq.md`)*

---

## Problem: Master Trigger Not Receiving Router Data

**Symptom:** Master trigger shows no multiplicity even though digitizers and routers appear healthy.

**Common cause:** Forgot to clear the SYNC bit in `LINK_LRU_CTRL` of the Router. Data streams SYNC patterns to Master instead of real event data.

**Fix:** Clear SYNC bit in Router's `LINK_LRU_CTRL` register.

*(Source: [wiki: Triggers and Digitizers](https://wiki.anl.gov/gsdaq/Triggers_and_digitizers))*

---

## Problem: IOC Boot Fails — VxWorks

**See:** `vxworks_migration.md` for known build/boot issues from Solaris→Ubuntu migration.

Key issues documented:
- Missing `makeSymTbl` tool → replaced with `munch` process
- Missing `wtxtcl` → workaround via Docker/ISE environment
- `ld_LINKER_VERSION` mismatch → fixed by cross-compiler flags

---

## Problem: Data Rate / FIFO Issues

**Symptom:** `TotalBuffers_Lost` climbing, data gaps in output files.

**Check:**
- `DigitizerFull[board]` / `DigitizerAlmostFull[board]` flags in inLoop
- Throttle line from digitizer to Router (FIFO half-full flag)
- `SendDataRate` in MiniSender stats

**Cause:** Trigger rate too high, FIFO filling faster than TCP readout can drain it.

**Fix:** Raise multiplicity threshold or enable software veto to reduce trigger rate.

---

## Problem: Timestamp Sync Errors

**Symptom:** `SE` (Sync Error) flag set in digitizer event headers; `DLYD_TDC_IN_NIM_IN2_RBV` shows unexpected values.

**Check:**
- SERDES lock status on all links
- Clock jitter budget — total accumulated jitter must be < 250 ps
- Link L clock recovery on Routers

*(Source: `ttcl.md`, `DIG_firmware_expert.md`)*

---

## Problem: Missing BGO Events on Specific Detectors

**Symptom:** BGO channels for one or more Ge detectors show zero or very low rate even when beam is present.

**Possible cause:** Ge preamp reset blanking (`CHANNEL_KILLED`) is gating the paired BGO channels. When a Ge channel is in preamp-reset kill state, `external_disc_flag(i) <= not(CHANNEL_KILLED(i+5))` suppresses BGO channels 0–4 paired to that Ge.

**Check:**
- High preamp reset rate for the Ge channel → check `MOD###_DIG_PREAMP_RESET_DELAY` is reasonable (too large = blanks BGO for too long)
- `MOD###_DIG_CHANNEL_CONTROL` bit 3 (`PREAMP_RESET_DELAY_EN`) — if enabled and delay is large, BGO can be effectively silenced
- Monitor `MOD###_DIG_HIHILOLO_CNT` to see how often the preamp is resetting

**Fix:** Reduce `PREAMP_RESET_DELAY` or disable it (`PREAMP_RESET_DELAY_EN=0`) for the affected channel.

*(Source: `preamp_reset_readme.md`, `Digitizer.vhd:L1117`)*

---

## Useful PVs for Diagnostics

| PV | What It Shows |
|----|--------------|
| `VMExx:MDIG1:trigger_mux_select_RBV` | Current trigger mode (ExtTTCL/IntAcptAll/etc) |
| `VME99:MTRG:EN_NIM1_DELAY` | NIM In 1 delay enabled? |
| `VMExx:MTRG:MISC_STAT` | NIM IN 1/2 state + misc status bits |
| `GS${N}_SBX_Present` | Is SBX connected for this detector? |
| `GS${N}_SlopeBoxGe_HV_On` | Is Ge HV on for this detector? |

---

*Source: [wiki: Some Problems and Their Solutions](https://wiki.anl.gov/gsdaq/Some_problems_and_their_solutions) + `wiki_gsdaq.md` trigger setup section + operational notes. Created: 2026-04-05.*
