# link_sys.py — System Link & Time Sync Analysis

Source: `DGS_tools_pack/ANLDAQ/gui/link_sys.py`

---

## Glossary

| Term | Full Name | Meaning |
|------|-----------|---------|
| MTRG | Master Trigger | The top-level trigger board. Acts as the clock master and trigger decision authority for the entire system. One per experiment. |
| RTRG | Router Trigger | Mid-level trigger board. Aggregates multiplicity from up to 8 DIGs and passes it up to the MTRG. Also routes trigger decisions back down to DIGs. One per VME crate (up to 8). |
| DIG | Digitizer | The front-end digitizer board. Reads out germanium detector signals, applies discriminators, and sends hit data upstream to the RTRG. Up to 10 channels per board. |
| MDIG | Master Digitizer | The front-bus sender role of a DIG pair. Sends hit data upstream to the RTRG. |
| SDIG | Slave Digitizer | The front-bus receiver partner of a MDIG. Receives trigger/clock from MDIG. Locks passively via MDIG's transmit clock. |
| SERDES | Serializer/Deserializer | High-speed serial link used between boards (DIG↔RTRG↔MTRG). Clock-sensitive — switching clock source mid-operation drops lock. |
| OSC | Oscillator | Each board has its own local crystal oscillator. Used as the default clock source before synchronization. |
| IMP_SYNC | Imperative Sync | A special command broadcast by the MTRG (via Frame 1 ISYNC bit) that simultaneously resets all timestamp counters on all connected boards to zero. |
| ISYNC | Imperative Sync bit | The specific bit in MTRG Frame 1 that triggers IMP_SYNC. |
| DEN | Data Enable | Enables the SERDES transmitter to send data on a link. |
| REN | Receive Enable | Enables the SERDES receiver to accept data from a link. |
| SYNC | Sync Enable | Enables transmission of a known SYNC pattern on a link. Used during init so the receiver can lock its SERDES clock recovery before real data flows. |
| ILM | Input Link Mask | Masks a link from contributing to the multiplicity sum. |
| XLM | X Multiplicity Mask | Masks a link from the X-axis multiplicity sum specifically. |
| YLM | Y Multiplicity Mask | Masks a link from the Y-axis multiplicity sum specifically. |
| ILM/XLM/YLM | Link Masks | Three-bit masking system: ILM masks all multiplicity, XLM/YLM mask X/Y axes independently. |
| LOCK_RETRY | Lock Retry pulse | Tells the MTRG link-init state machine to re-attempt locking after a clock switch glitch. |
| LOCK_ACK | Lock Acknowledge pulse | Tells the MTRG that lock is confirmed stable. Advances the link-init state machine to ACKED. |
| sd_sync | Sync Data mode | When 1, the DIG transmits a constant SYNC pattern instead of real hit data. Allows the RTRG to lock SERDES without being confused by random hit data. |
| EPICS CA | EPICS Channel Access | The network protocol used to read and write PVs (Process Variables) on the IOC. |
| PV | Process Variable | A named control/monitor point in the EPICS system (e.g. `VME01:MTRG1:clk_select`). |
| IOC | Input/Output Controller | The VME-based computer (MVME5500, running VxWorks) that runs EPICS and controls the MTRG/RTRG/DIG boards via the VME backplane. |
| DAQ | Data Acquisition | The system that collects detector hit data for offline analysis. |
| DGS | Digital Gamma-ray Spectrometer | The full detector system at ANL. |

---

## Purpose

`LinkSys` is the **system-wide link initialization and time synchronization script** for the DGS trigger chain. It establishes SERDES (Serializer/Deserializer) lock across the full MTRG → RTRG → DIG hierarchy, propagates a common clock, and issues an Imperative Sync (IMP_SYNC) to reset all timestamp counters so all boards share a common time base.

It is run at the start of an experiment session, before DAQ can begin.

---

## Why Is This Necessary? — The 3 Core Problems

### Problem 1: Every board starts on its own independent clock

At power-on, every MTRG, RTRG, and DIG runs on its own local oscillator. They are nominally the same frequency but drift relative to each other — even a few ppm difference causes timestamps to diverge over time. A physics event is built from coincident hits across many detectors; if each detector's timestamp is in a different time reference, event reconstruction is impossible.

**Solution:** Elect the MTRG as clock master. Hand clock authority down the hierarchy (MTRG → RTRG → DIG) so every board ultimately derives its clock from one source.

### Problem 2: You cannot flip everyone to a common clock simultaneously

SERDES links are clock-sensitive. Switching a board's clock source mid-operation causes the link to drop lock. You must hand off clock authority in strict **top-down order**, verifying lock stability at each level before proceeding to the next:

```
MTRG (clock master)
  └─► RTRGs lock to MTRG clock  ← must be stable before touching DIGs
        └─► DIGs lock to RTRG clock
```

Skipping or reordering steps leaves boards on mixed clocks, making synchronization impossible.

**Solution:** The 5-stage sequence enforces this ordering, with explicit lock checks and stability windows between each clock hand-off.

### Problem 3: Shared clock ≠ aligned timestamps

Even after all boards share the same clock source, their timestamp counters started at different moments — they are still offset from each other. A common clock gives the same tick rate, but not the same time zero.

**Solution:** `IMP_SYNC` (Imperative Sync) — a special command in MTRG Frame 1 that propagates down to every RTRG and DIG in the same trigger cycle, forcing all timestamp counters to reset to zero simultaneously. It is issued **twice**: once after RTRGs switch to MTRG clock, and again after DIGs switch to link clock. ✅ verified 2026-04-08 — `link_sys.py:L459` (Stage 3F, no immediate clear) + `link_sys.py:L606-607` (Stage 4G, set then clear)

---

## 5-Stage Initialization Sequence

### Stage 1 — Initialize MTRG

**Why:** The MTRG (Master Trigger) is the clock master and trigger authority for the entire system. It must be in a clean, known state before any downstream boards (RTRGs, DIGs) are touched. Stale trigger enables or veto states from a previous run could cause spurious triggers or block the init sequence.

1. Set clock source (`ClkSrc`): 0 = local OSC (on-board crystal oscillator), 1 = external clock input
2. Set `LINK_L_PROPAGATE_F1` = same as clock source — controls whether the clock-sync frame is also forwarded on the inter-MTRG Link L (used when two MTRGs are chained)
3. Clear all trigger algorithm enables (`EN_SUM_X`, `EN_SUM_Y`, `EN_ALGO5`, etc.) — disables all trigger decisions so no spurious triggers fire during init
4. Clear all veto enables (NIM veto, RAM veto, throttle) — ensures nothing can block the data path during init
5. Set ALGO_5 = coincidence trigger mode; LINK_U = MYRIAD (another DAQ system) trigger type — configures trigger logic for this experiment
6. Enable MTRG L/R/U DEN/REN/SYNC (`LRUCtl00–10`):
   - **DEN** (Data Enable): allows the transmitter to send data on the link
   - **REN** (Receive Enable): allows the receiver to accept data from the link
   - **SYNC** (Sync Enable): transmits a known sync pattern so the downstream board can lock its SERDES receiver before real data flows
   - L/R/U = the three inter-trigger links on the MTRG (Left, Right, Up — connecting to other MTRGs or special systems)
7. Set link masks per link:
   - DGS links: ILM=0, XLM=0, YLM=0 — active, hits count in X and Y multiplicity sums
   - PIXIE/DFMA/DUB/DXA (other DAQ systems sharing the trigger): ILM=0, XLM=1, YLM=1 — active (receives triggers) but masked from DGS multiplicity sums
   - MASKED links: ILM=1, XLM=1, YLM=1 — fully ignored
8. Enable TX (transmit) / RX (receive) power on all 11 SERDES links (A–H = DIG links, L/R/U = inter-trigger links)
9. Enable pre-emphasis (boosts high-frequency signal components to compensate for cable loss), release `RESET_LINK_INIT` (allows the MTRG link-init state machine to start running)

**What is achieved:** The MTRG is now the active clock master running on its local oscillator, broadcasting the 20-frame command structure (including Frame 1 with Sync + timestamp) to all connected RTRGs. All trigger decisions and vetos are disabled — the system is safe to reconfigure downstream boards.

---

### Stage 2 — Initialize RTRGs

**Why:** RTRGs (Router Triggers) must be initialized on their own local oscillator first — before any clock hand-off from the MTRG. This ensures each RTRG is in a clean state (no stale clock selection, no loopback active, correct link masks) before the SERDES links between MTRG and RTRG are asked to lock. Configuring an RTRG while it is already running on the MTRG clock would cause lock glitches that pollute the subsequent lock verification steps.

For each RTRG (Router):
1. Set clock to local OSC (`ClkSrc=0`), clear `FORCE_SYNC` — ensures the RTRG is on its own independent clock and not forcing a sync condition
2. Toggle L/R/U DEN/REN/SYNC off then on:
   - **DEN** (Data Enable), **REN** (Receive Enable), **SYNC** (Sync pattern transmit) — see Stage 1 for definitions
   - The toggle-to-wrong-value-then-correct pattern is intentional: EPICS Channel Access (CA) only pushes a write when the value changes. If the register already holds the desired value from a previous run, CA would skip the write. Toggling through an intermediate value forces the write to actually reach the hardware. ✅ verified 2026-04-10 — `link_sys.py:L305-315, L318-344` (all Active/Masked/Disabled branches toggle to wrong values first, then set correct values)
3. Set pre-emphasis levels (`PrE_0/1/2`) — compensates for cable signal loss on SERDES links; disable loopback (loopback is only used for testing, must be off for normal operation)
4. Enable DC balance on Link L (`LinkL_DCbal=1`) — Link L is the SERDES link from MTRG to RTRG; DC balance encoding prevents long runs of 0s or 1s that would confuse the clock recovery circuit
5. Set link masks per DIG link:
   - Active DIG: ILM=0, TPwr=1 (TX power on), RPwr=1 (RX power on), loopbacks off — link carries real hit data
   - Masked DIG (powered but ignored): ILM=1 (Input Link Mask = ignore from multiplicity), TPwr=1, RPwr=1 — link is up but hits don't count
   - Disabled DIG: ILM=1, TPwr=0, RPwr=0 — link is completely off
6. Reset MTRG link-init state machine — clears any error flags that accumulated while reconfiguring the RTRGs

**What is achieved:** All RTRGs are running on their own local oscillators, with SERDES links to the MTRG powered and correctly configured. The MTRG–RTRG SERDES links can now begin locking. RTRGs are not yet on the MTRG clock — that critical hand-off happens in Stage 3.

---

### Stage 3 — Verify Lock + Hand Off Clock

**Why:** This is the most critical stage — it solves Problem 2 (clock hand-off ordering) and Problem 3 (timestamp alignment) for the MTRG–RTRG tier. Lock must be confirmed stable *before* flipping the clock, because switching to an unstable clock would cause SERDES to lose lock entirely. The 15-second stability check after the switch is essential: a link that briefly drops and re-locks during this window would cause all downstream timestamp counters to be off.

**3A:** Wait 3 seconds, then check that all active MTRG–RTRG SERDES links are locked (`LOCK_x_RBV == "Off"` means locked — note: "Off" = no lock-loss event, i.e. locked). The 3-second wait gives SERDES time to acquire lock after Stage 2 configuration. ✅ verified 2026-04-17 — `link_sys.py:L359` (`time.sleep(3.0)` before 3A lock check)

**3B:** Wait 2 more seconds, then check that each RTRG's Link L (the SERDES link from MTRG to that RTRG) is also locked.

**3C:** Verify the MTRG link-init state machine reports `ALL_LOCK` — meaning every active link has achieved and held lock.

**3D:** Reset each Router's `LOST_LOCK` flag, wait 5 seconds, then verify no lost-lock events occurred during those 5 seconds. ✅ verified 2026-04-17 — `link_sys.py:L401` (`time.sleep(5.0)` after `ResetRouterLostLock` before re-check)
- *Why reset and re-check:* the reset clears transient glitches left over from Stage 2 reconfiguration. Only a clean 5-second window with zero lost-lock events confirms the lock is genuinely stable, not just momentarily recovered.

**3E:** Flip each RTRG's clock source from its local oscillator → **Link L clock** (derived from the MTRG's oscillator).
- This is the key synchronization step: all RTRGs now run on one clock, sourced from the MTRG
- Switching the clock source causes a momentary SERDES glitch (the RTRG briefly loses lock as it transitions). After the switch: reset the MTRG link-init state machine and clear RTRG `LOST_LOCK` flags, then re-verify lock
- *Why must RTRGs switch to MTRG clock before IMP_SYNC:* IMP_SYNC resets timestamp counters simultaneously across all boards. If the RTRGs are still on their own independent oscillators, "simultaneously" means nothing — each board's counter resets at a slightly different real-world moment. Only when all boards share one clock does a single IMP_SYNC pulse produce a truly common time zero.

**3F:**
- Pulse `LOCK_RETRY` — tells the MTRG link-init state machine to re-attempt lock acquisition after the clock switch glitch
- Pulse `LOCK_ACK` — tells the MTRG that lock is confirmed stable; advances the state machine to the ACKED state
- Set `IMP_SYNC = 1` — **Imperative Sync**: the MTRG fires the ISYNC bit in Frame 1 of its 20-frame broadcast. This propagates to all connected RTRGs in the same trigger cycle (~2 µs), forcing every timestamp counter to reset to 0 at the same instant ✅ verified 2026-04-08 — `link_sys.py:L459` (no `IMP_SYNC=0` follows — stays asserted through the diagnostic counter clearing below)
- Clear Router diagnostic counters (resets the lock-event counters used in 3G)

**3G:** 15-second stability check:
- `reg_DiagnosticF` = RTRG lock state machine event counter — counts how many times the lock state machine cycled. Must be 0 (no lock events after IMP_SYNC)
- `reg_DiagnosticG` = RTRG Link L SERDES chip lock counter — counts SERDES lock acquisitions on Link L. Must be 0
- *Why 15 seconds:* intermittent lock glitches can take several seconds to appear. A link that looks locked for 2 seconds but drops at second 10 would corrupt all subsequent timestamps. 15 seconds is long enough to catch these. ✅ verified 2026-04-17 — `link_sys.py:L497` (`time.sleep(15.0)`)
- **Do not proceed if either counter is non-zero** — it means the link is not stable and timestamps will be unreliable

**What is achieved:** All RTRGs are now running on the MTRG's clock. The MTRG and all RTRGs share a common, stable 48-bit timestamp counter (reset to 0 by IMP_SYNC). The MTRG–RTRG tier is fully synchronized. DIGs are still on their own independent local oscillators — that is resolved in Stage 4.

---

### Stage 4 — Initialize DIGs + Verify Router Lock

**Why:** DIGs must be initialized on local OSC first (same reason as RTRGs in Stage 2 — clean state before clock hand-off). The RTRG–DIG SERDES links need to lock before the DIG clock can be switched. After the clock switch, a second IMP_SYNC is required because DIGs were not on the MTRG-derived clock during Stage 3's IMP_SYNC — their counters were not reset then.

**4A:** Initialize all DIGs (Digitizers) on their local oscillator:
- `clk_select=1` (OSC) — each DIG uses its own on-board crystal oscillator as clock source (see Glossary: `clk_select`)
- Disable loopbacks — loopback mode routes the TX signal back to RX internally, used only for testing; must be off for normal operation
- Set pre-emphasis to minimum — a safe starting point; can be tuned if link quality is poor
- `dc_balance_enable=0` — DC balance encoding is disabled initially
- `master_logic_enable=0` — keeps DIG trigger logic inactive during init
- `sd_sync=1` — each DIG transmits a constant SYNC pattern instead of real detector hit data
- *Why sd_sync=1:* the RTRG's SERDES receivers need a stable, predictable signal to lock onto. Real discriminator hits are random in timing and content — the SERDES clock recovery circuit cannot lock to a random data stream. The SYNC pattern is a known, repetitive sequence that allows clean lock acquisition.

**4B:** Reset each RTRG's link-init state machine.
- *Why:* clears any stale lock state accumulated during Stage 3. Gives a clean baseline so the subsequent DIG–RTRG lock check reflects only the current state.

**4C:** Check all active RTRG links are locked (only performed if `perform_error_checks=True` in the script configuration).

**4D:** Check MDIG (Master Digitizer) SERDES lock status (`serdes_lock_RBV == "Lock"`).
- Only the MDIG is checked, not the SDIG (Slave Digitizer / front-bus receiver partner)
- *Why MDIG only:* the SDIG does not transmit upstream — it only receives the trigger/clock signal from the MDIG on the front bus. The SDIG locks passively via the MDIG's transmit clock, so checking the MDIG is sufficient.

**4E:** Flip all DIGs to the desired clock source (`dig_clk_sel`):
- `dig_clk_sel=0` — each DIG stays on its own local oscillator (boards are independent; only use this for testing)
- `dig_clk_sel=1` — each DIG switches to the link clock (derived from the RTRG, which is itself derived from the MTRG) — **this is the normal production setting**
- After the clock switch: pulse RTRG `LOCK_RETRY` then `LOCK_ACK` to re-establish lock on the RTRG–DIG SERDES links (switching the DIG clock source causes the same momentary glitch as in Stage 3E)

**4F:** Verify RTRG link-init reports `ALL_LOCKED_RBV == 1` — all active DIG–RTRG SERDES links are locked after the clock switch.

**4G:**
- Issue `IMP_SYNC = 1` then `IMP_SYNC = 0` — **this is the second and final timestamp reset, now including all DIGs** ✅ verified 2026-04-08 — `link_sys.py:L606-607` (set then immediately cleared, unlike Stage 3F which left it asserted)
- *Why a second IMP_SYNC is required:* during Stage 3's IMP_SYNC, the DIGs were still on their own independent oscillators. That IMP_SYNC reset the MTRG and RTRG counters to zero, but the DIG counters were not on the same clock and could not be meaningfully aligned at that moment. Now that all DIGs share the MTRG-derived clock, a second IMP_SYNC resets every counter in the system simultaneously — completing the full three-tier timestamp alignment.
- Enable stringent lock mode on all DIGs (`sd_sm_stringent_lock=1`) — tightens the SERDES lock criteria for production data-taking; more sensitive to marginal links
- Reset DIG lost-lock flags — clears any transient events from the clock switch
- Set `sd_sync=0` — DIGs stop sending SYNC pattern and transition to sending real detector hit data

**What is achieved:** All DIGs are now running on the MTRG-derived clock (routed through the RTRG). The full three-tier hierarchy (MTRG → RTRG → DIG) shares one clock source and one common timestamp zero. The system is synchronized end-to-end. DIGs are ready to send real hit data to the RTRGs.

---

### Stage 5 — Switch to Real Data

**Why:** The SYNC pattern that kept SERDES links locked during init must be turned off in strict bottom-up order before real data can flow. If you turn off SYNC at the MTRG or RTRG level before the DIGs have switched, the Routers would see real hit data before their receivers are ready, causing data corruption.

**5A:** Set `sd_sync=0` on all DIGs — each DIG stops transmitting the SYNC pattern and begins sending real germanium detector discriminator hit data upstream to its RTRG.
- *DIGs go first:* they are the data source. The RTRGs are already locked and waiting. Switching the data source (DIG) before the receiver (RTRG) would cause the RTRG to see garbage data; but since RTRGs are already locked and merely waiting for the SYNC→data transition, this ordering is safe.

**5B:** Verify each RTRG's `ALL_LOCKED_RBV == 1` (all DIG–RTRG links still locked), then pulse `LOCK_RETRY` and `LOCK_ACK` on each RTRG.
- *Why verify again:* the transition from SYNC pattern to real hit data is a signal content change that can briefly stress the SERDES lock. Confirming lock held through this transition ensures no timestamp corruption occurred.

**5C:** Turn off SYNC on RTRGs (`LRUCtl02=0`) and on MTRG L/R/U inter-trigger links.
- This switches the RTRG–MTRG inter-trigger links from carrying a SYNC pattern to carrying real multiplicity (hit count) data and trigger decisions
- *RTRGs and MTRG go last:* once the DIGs are sending real data and the RTRG–DIG links are confirmed stable, the inter-trigger tier can safely switch to live data flow

**5D:** Verify MTRG `LINK_INIT_STATE_RBV == "ACKED"`.
- `ACKED` is the final state of the MTRG link-init state machine. It means the MTRG has confirmed that all downstream links are locked, IMP_SYNC has been issued and acknowledged, and the system is ready for DAQ.
- **If the state is not ACKED, do not start DAQ** — the system is not fully synchronized.

**What is achieved:** The full data and trigger chain is live:
- **Upstream (data):** real germanium detector hits flow DIG → RTRG → MTRG
- **Downstream (trigger):** trigger decisions flow back MTRG → RTRG → DIG, gating which events are readout
- All boards share one clock and one timestamp zero
- DAQ (Data Acquisition) can now begin recording events

---

## Assessment: Does link_sys.py Match the FPGA Code?

**Yes — the script is correct and well-structured.** Cross-referenced against MTRG, RTRG, and DIG VHDL source (`mstr_mach.vhd`, `link_init.vhd`, `register_block.vhd`, `TOP.VHD`).

| Stage | What it does | Match? | Notes |
|-------|-------------|--------|-------|
| 1 — MTRG init | Set clock source, clear triggers/vetos, configure ILM/XLM/YLM, enable links | ✅ | Safe init order; PIXIE/DXA correctly masked from X/Y multiplicity |
| 2 — RTRG init | Reset to local OSC, toggle DEN/REN/SYNC, set link masks | ✅ | Toggle-to-wrong-then-correct is intentional: forces EPICS CA write even when value unchanged |
| 3 — Lock + clock hand-off | Verify MTRG/RTRG lock, flip Routers to MTRG clock, issue IMP_SYNC | ✅ | IMP_SYNC = Frame 1 ISYNC bit; confirmed in MTRG mstr_mach.vhd |
| 4 — DIG init | Start on OSC, verify Router lock, flip to link clock, second IMP_SYNC | ✅ | MDIG-only lock check correct — SDIG is front-bus receiver, locks via MDIG |
| 5 — Go live | sd_sync=0 on DIGs, disable SYNC on RTRGs/MTRG, verify ACKED | ✅ | Correct ordering: DIGs first, then Routers, then MTRG |

**One minor observation:** Stage 3E flips Routers to link clock and calls `RESET_LINK_INIT`, but does not explicitly re-check `LINK_INIT_STATE = ALL_LOCK` before issuing `LOCK_RETRY`/`LOCK_ACK` in 3F. It relies on `time.sleep()` for settling. Works in practice, but an explicit lock re-check before ACK would be more robust.

---

## How the PV → VME Address → FPGA Variable Chain Works

Every control in `link_sys.py` goes through a 3-layer stack:

### Layer 1: EPICS PV → VME Address (EPICS db templates)

Each PV is defined in a `.template` or `.db` file (e.g. `MDigUserVME.template`, `MRtrUserVME.template`) with an `asynMask` output link:

```
record(mbbo, "VME$(CRATE):$(BOARD):clk_select") {
    field(OUT, "@asynMask($(BOARD), 0, 0x00000003, 1)vme_clk_ctrl")
}
```

The EPICS asyn driver translates this into a VME write at a specific address and bitmask.

### Layer 2: VME Address → FPGA Signal (`register_block.vhd`)

The VME FPGA has a `case` statement on the incoming VME address. Each address maps to one or more internal signals:

```vhdl
-- register_block.vhd (DIG VME FPGA)
when X"0910" =>
    sysclk_sel0 <= VME_data_in(0);   -- pin B9
    sysclk_sel1 <= VME_data_in(1);   -- pin B10
```

Same pattern for RTRG and MTRG — each board has its own `register_block.vhd`.

### Layer 3: FPGA Signal → Hardware (`.ucf` pin constraints or internal routing)

Signals either:
- Stay **internal**: routed to the main FPGA over the shared parallel bus (IDT FIFO or direct lines)
- Drive **output pins**: defined in the `.ucf` constraint file and routed off-chip

Example — `clk_select` drives a hardware clock mux on the PCB:
```
NET "sysclk_sel0_out" LOC = "B9";    # → hardware clock mux pin
NET "sysclk_sel1_out" LOC = "B10";
```

**Note:** `sysclk_sel` bits are **inverted** before the output pin (`sysclk_sel0_out <= NOT sysclk_sel0`) to match the original LBL board design. This is transparent to software — EPICS values are correct end-to-end.

### How to Trace Any PV

1. Find the `.template` or `.db` file for that board → get the `asynMask` address offset
2. Search `register_block.vhd` for that hex address → find the signal name(s)
3. Search the main FPGA source or `.ucf` for that signal → find what it controls (pin, internal bus, etc.)

---

## Utility Methods

### `ResetRouterLostLock(rtr_name)`
Pulses `SM_LOST_LOCK_RESET` (0→1→0) on the given Router. Used to clear a "lost lock" error in the Router's SERDES state machine — typically needed after a transient link glitch or power cycle. Call once per affected Router before retrying Stage 3 or Stage 5 error checks.

### `perform_error_checks` flag
Passed at construction (`LinkSys(..., perform_error_checks=True)`). When `False`, all lock-status verification steps (Stage 3, 4C, 4F, 5B, 5D) are skipped — useful for dry-run testing or scripted setups where the FPGA hardware isn't live.

---

## Link Map Format Reference

```python
# MTRG link map
mtrg_map = [
    ["A", "RTR1", 1],      # Link A → Router 1, propagate=True
    ["B", "MASKED", 0],    # Link B → unused
    ["L", "RTR2", 1],      # Inter-trigger Link L → second MTRG, propagate
]

# RTRG link map (per router, 11 entries: A-H, L, R, U)
rtrg_map = [
    [1, 1, 0, 0, 0, 0, 0, 0,  1, 0, 0],  # RTR1: A+B active, L active
    [1, 0, 0, 0, 0, 0, 0, 0,  1, 0, 0],  # RTR2: A active, L active
]
# 0=X(disabled), 1=Active, 2=M(masked+powered)
```

## Stage 5 — SYNC→Data Flip Sequence (Detailed)

Stage 5 is the critical final step that transitions from SYNC patterns to real physics data:

1. **5A:** Clear `sd_sync` on all DIGs → digitizers switch from SYNC to discriminator data ✅ verified 2026-04-17 — `link_sys.py:L625` (`SetPVManually(dig_name, "sd_sync", 0)` for all DIGs)
2. **5B:** Wait 2s, verify all RTRGs show `ALL_LOCKED_RBV = 1` — if any router fails, abort ✅ verified 2026-04-17 — `link_sys.py:L630` (`time.sleep(2.0)` then `ALL_LOCKED_RBV != 1` error check)
3. **Pulse** `LOCK_RETRY` then `LOCK_ACK` on all RTRGs (clears lock retry state) ✅ verified 2026-04-17 — `link_sys.py:L640-644`
4. **5C:** Clear `LRUCtl02` on all RTRGs (router Link L SYNC off) → routers send real data to MTRG
   - Also clear MTRG `LRUCtl02`, `LRUCtl06`, `LRUCtl10` (L/R/U SYNC off) ✅ verified 2026-04-17 — `link_sys.py:L651-658` (per-RTR `LRUCtl02=0`, then MTRG L/R/U all three SYNC bits cleared)
5. **5D:** Wait 2s, verify MTRG `LINK_INIT_STATE_RBV == "ACKED"` — if not, abort ✅ verified 2026-04-17 — `link_sys.py:L661-665` (`time.sleep(2.0)` then check `ACKED`)

The SYNC clear order matters: DIGs first, then RTRGs, then MTRG LRU SYNCs.

## Link Map Data Structures

### MTRG Link Map
```python
linkSys.Set_MTRG_LINK_MAP([
    ["A", "RTR1", propagate_flag],   # link A → RTR1, propagate trigger?
    ["B", "RTR2", propagate_flag],
    ...
    ["L", "DFMA", 1],               # link L → DFMA system, propagate=1
    ["U", "MyRIAD", 0],             # link U → MyRIAD, no propagation
])
```
Each entry: `[link_letter, board_type_string, propagate_flag]`. Board type strings: `"RTR1"–"RTR8"` (regular routers), `"DFMA"`, `"DUB"`, `"DXA"`, `"PIXIE"`, `"MASKED"`. Links L/R/U use special propagation flags (`LINK_x_PROPAGATE_F3–F7`).

### RTR Link Map
```python
linkSys.Set_RTR_LINK_MAP([
    [1, 0, 1, 1, 0, 0, 0, 0, 0, 0, 0],  # RTR1: links A-H,L,R,U states
    [1, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0],  # RTR2
    ...
])
```
One row per router, 11 values for links A–H, L, R, U in that order. State values:
- `0` = X (disabled: ILM=1, TPwr=0, RPwr=0, loopbacks off) ✅ verified 2026-04-10 — `link_sys.py:L333-344` (else branch)
- `1` = Active (ILM=0, TPwr=1, RPwr=1, loopbacks off) ✅ verified 2026-04-10 — `link_sys.py:L305-315` (state==1 branch)
- `2` = M (masked but powered: ILM=1, TPwr=1, RPwr=1, loopbacks off — link powered but doesn't contribute to trigger) ✅ verified 2026-04-10 — `link_sys.py:L318-330` (state==2 branch)

## ResetRouterLostLock Helper

```python
linkSys.ResetRouterLostLock(rtr_name)
```

Pulses `SM_LOST_LOCK_RESET` (0→1→0, 500ms between steps) on a specific Router. Used when a router's link-init state machine gets stuck after a lock-loss event. Does not restart the full link-up sequence.

## Error Check Mode

`LinkSys(MTRG, RTR_list, DIG_list, perform_error_checks=True)` — default True.

When `perform_error_checks=False`, Stages 3, 4, 5 skip all `ALL_LOCKED_RBV` and `LINK_INIT_STATE_RBV` checks and return True unconditionally. Useful for forced re-init when lock state is known good.

---

## Shell Script Counterpart (trig_setup_Stage*.sh)

The GUI's 5-stage Python sequence (`link_sys.py`) maps directly to 5 bash scripts (`trig_setup_Stage{1-5}.sh`) that perform the same EPICS PV writes in the same order. The scripts add sub-step granularity (1A–1I, 2A–2B, 4A–4C, 5A–5D) and contain key design comments (EPICS double-write pattern, fiber DC balance notes, author JTA).

Full shell script documentation: **`knowledgeBase/trig_setup_scripts.md`** (added 2026-04-17)

---
*Source: `DGS_tools_pack/ANLDAQ/gui/link_sys.py` (669 lines) — Python LinkSys class. Verified 2026-04-10 against source. Created: 2026-04-05.*

## Cross-References

- `knowledgeBase/trig_setup_scripts.md` — Full shell script counterpart: SYSTEM_DEFINES.sh GS topology, per-stage sub-step tables (1A–5D), key design notes (DC balance, fiber, EPICS double-write, JTA authorship)
- `knowledgeBase/ANLDAQ.md` — ANLDAQ GUI: trigger setup scripts (5-stage shell), SerdesLinkup button that calls this Python class
- `knowledgeBase/fpga.md` — FPGA firmware overview: SERDES link role in 3-tier hierarchy
- `knowledgeBase/deep_fpga_MTRG_MAIN.md` — MTRG Main FPGA: SERDES link registers driven by Stage 1
- `knowledgeBase/deep_fpga_RTRG.md` — RTRG firmware: SERDES link registers driven by Stages 2–4
- `knowledgeBase/troubleshooting.md` — Router lock loss, SYNC bit gotcha (Stage 5 clears SYNC)
