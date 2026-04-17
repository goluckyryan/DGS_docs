# Trigger Setup Scripts — ANLDAQ GUI

**Source:** `DGS_tools_pack/ANLDAQ/gui/scripts/trig_setup_Stage{1-5}.sh`  
**Config:** `DGS_tools_pack/ANLDAQ/gui/scripts/SYSTEM_DEFINES.sh`  
**Documented:** 2026-04-17

---

## Overview

The trigger setup is a 5-stage scripted initialization procedure that brings the MTRG→RTRG→DIG link chain from a cold/unknown state into a fully synchronized, data-passing configuration.  
Each stage is a bash script invoked with two arguments:
1. `SYSTEM_DEFINE_FILE` — name of the system defines file (e.g. `SYSTEM_DEFINES.sh`)
2. Path to the scripts directory

---

## SYSTEM_DEFINES.sh — Gammasphere Configuration

Defines the physical topology used by all five stage scripts.

| Variable | Value | Meaning |
|---|---|---|
| `MT_VME_LEADER` | `VME10` | VME crate containing the MTRG |
| `MT_USE_LINK_CLK` | 0 | Use local clock (not link clock) during init |
| `SCRIPT_VERBOSITY` | 1 | 0=stage-level, 1=steps, 2=per-channel |
| `PERFORM_ERROR_CHECKS` | (set in script) | Enables EPICS PV lock checks at each stage |

**Routers (LIST_OF_ROUTERS):**
| RTRG | Crate | Active Links | Link L connects to |
|---|---|---|---|
| RTR1 | VME03 | A B C D E F | MTRG link A |
| RTR2 | VME06 | A B C D E F | MTRG link B |
| RTR3 | VME09 | A B C D E F | MTRG link C |
| RTR4 | VME12 | A B C D E F | MTRG link D |

Each RTRG uses links A–F (6 DIGs each × 10 ch = 60 ch per RTRG × 4 = 240 ch — half the GS install).  
Links G, H, R, U are unused (`X`) in this config. Link L of each RTRG connects back to the MTRG.

**Digitizers (LIST_OF_DIGITIZERS):** 12 VME crates × 2–4 MDIGs each:
- VME01–05, 07–09, 11–12: 4 MDIGs each (4 × 10 = 40 ch/crate)
- VME06, VME10: 2 MDIGs each (2 × 10 = 20 ch/crate)
- Total: (10 × 4 + 2 × 2) = 44 DIG boards = 440 channels ✅ matches hardware_architecture.md

---

## Stage 1 — MTRG Initialization

**Goal:** Put the master trigger into a known clean state and drive SYNC pattern out all links.

### Step-by-step:
- **1A:** Set MTRG clock to local (`ClkSrc`). Clear any link propagation from external masters (DFMA, GITMO, MYRIAD).
- Clear starting timestamp registers to zero.
- **1B:** Assert/release `RESET_LINK_INIT`. Clear `LOCK_RETRY`, `LOCK_ACK`, stringent lock, `IMP_SYNC`.
- **1C:** Disable all trigger types: MAN/AUX, SUM_X, SUM_Y, SUM_XY, ALGO5 (coincidence), LINK_L/R/U.
  - Clear all coincidence trigger mask bits (`COINC_TRIG_MASK_*`).
  - Set `ALGO_5_SELECT=1` (Algorithm 5 = coincidence trigger).
  - Set `LINK_U_IS_TRIGGER_TYPE=1` (Link U = Remote Trigger, not MYRIAD).
- **1D:** Clear all trigger veto enables (NIM, RAM, throttle) for all algorithms A–H.
- **1E:** Clear global veto enables (`SOFTWARE_VETO`, `EN_RAM_VETO`, `ENBL_MON7_VETO`, `ENBL_NIM_VETO`, `ENBL_THROTTLE_VETO`).
- **1F:** Enable DEN/REN/SYNC on MTRG links L, R, U (fiber inter-master connections).
- **Enable DC balance** on MTRG (`EN_RTR_DCBAL=1`) — required for fiber expander (2022+).
- **1G:** Set input link mask (`ILM_*`) for links A–H based on `MT_LINK_MAP`:
  - Unmasked ("RTR4"-type): `ILM=0` (active)
  - Masked ("MASKED"): `ILM=1` (inactive)
  - Special types: PIXIE, DFMA, DXA, DUB — set propagation enables accordingly
  - Also sets X/Y sum masks (`XLM_*`, `YLM_*`) for each link
  - Sets `LINK_*_PROPAGATE_F3/F4/F5/F6/F7` for L/R/U links based on connection type
- **1H:** Enable Transmit/Receive power on all MTRG links; disable line/local loopback. Clear whole-register loopback PVs.
- **1I:** Enable LVDS pre-emphasis drivers for MTRG (bits 0,1,2). Disable cable-level pre-emphasis (`PEHLRU=PEEFG=PEABCD=0`) — fiber expander 2022+.
- Release `RESET_LINK_INIT`.

**End state:** MTRG driving SYNC patterns to all RTRGs, waiting for lock-in from routers.

---

## Stage 2 — RTRG Initialization

**Goal:** Initialize all RTRGs to receive SYNC from MTRG and drive SYNC back on link L.

### Step-by-step:
- **2A:** For each RTRG (`LIST_OF_ROUTERS`):
  - Set clock to local (`ClkSrc=local`), clear `FORCE_SYNC_REG`.
  - Set link L (toward MTRG): DEN/REN/SYNC all ON (write OFF then ON — workaround for EPICS state desync).
  - Enable LVDS pre-emphasis (0→1 for bits 0,1,2), disable cable pre-emphasis.
  - Enable DC balance on link L (`LinkL_DCbal=1`) — fiber expander requirement.
  - Clear whole-register loopback PVs.
  - For each RTRG channel link (A–H based on `LIST_OF_ROUTERS` per-link entries):
    - `X` (unused): mask it, power off TX/RX, disable loopbacks
    - `M` (masked but powered): mask it, power TX/RX on, loopbacks off
    - Active link: unmask, power TX/RX on, loopbacks off
- Wait 2 seconds for EPICS.
- **2B:** Re-assert/release `RESET_LINK_INIT` on MTRG (router setup may have disturbed master link-init state).

**End state:** All RTRGs driving SYNC on link L → MTRG; MTRG driving SYNC to RTRGs on links A–H.

---

## Stage 3 — Link Lock Verification (MTRG + RTRGs)

**Goal:** Verify all active links are locked before proceeding.

### Step-by-step:
- Wait 3 seconds for EPICS PV refresh.
- **3A:** Check `LOCK_{link}_RBV` for each unmasked MTRG link (should be "Off" = 0 = LOCKED on DS92LV18). Fail if any unmasked link is not locked.
- **3B:** Check `ALL_LOCKED_RBV` on each RTRG's link L (must be 1). Fail if not.
- **3C+:** Additional link-level checks on each RTRG active link using firmware's more stringent lock-state machine.
- On failure: script exits with error code 1.

**Note:** EPICS scans lock state ~1 Hz. Stage 3 is a "good enough" pre-check; more stringent firmware checks follow in later stages.

---

## Stage 4 — DIG Initialization + Router Link Lock Verification

**Goal:** Initialize all DIGs to drive SYNC toward their RTRG, verify RTRG locks.

### Step-by-step:
- **4A:** For each active DIG (`LIST_OF_DIGITIZERS`):
  - `clk_select=1` (local oscillator)
  - Disable line/local loopback
  - Set pre-emphasis minimum
  - Power on RX/TX
  - Assert SYNC bit (`sd_sync=1`)
  - Disable stringent lock and DC balance
  - Disable `master_logic_enable`
- Wait 2 seconds.
- **4B:** Pulse `RESET_LINK_INIT` on all RTRGs (so RTRGs can re-establish lock with DIGs now driving SYNC).
- Wait 2 seconds.
- **4C+:** Verify all RTRG active links (A–H) are locked (`caget ${RTR}:LOCK_{link}_RBV`). Fail if any unlocked active link found.

**End state:** All DIGs driving SYNC → RTRGs; all links DIG→RTRG→MTRG locked.

---

## Stage 5 — Transition to Live Data

**Goal:** Flip all boards from SYNC to real discriminator data; verify each step.

### Step-by-step:
- **5A:** For each active DIG: turn off SYNC bit (`sd_sync=0`) → DIG now sends discriminator data.
- **5B:** Re-check `ALL_LOCKED_RBV` on all RTRGs (must still be 1 after DIGs switched to data).
  - If error: exit 1.
- Pulse `LOCK_RETRY` then `LOCK_ACK` on each RTRG (clears any transient lock-lost events).
- **5C:** Turn off SYNC on all RTRG link L's (`LRUCtl02=OFF`) → RTRGs now send real data to MTRG.
  - Also turn off SYNC on MTRG links L/R/U (`LRUCtl02/06/10=0`).
- **5D:** Check MTRG link-init state machine: `LINK_INIT_STATE_RBV` must be "ACKED".
  - If not: exit 1.

**End state:** Full system running — DIGs sending discriminator bits to RTRGs, RTRGs forwarding trigger sums to MTRG, MTRG issuing accept/reject decisions.

---

## Key Design Notes

- **EPICS whole-register vs breakout PV gotcha:** Writing to a whole-register PV (e.g., `VME10:MTRG:reg_INPUT_LINK_MASK`) updates `reg_INPUT_LINK_MASK_RBV` but does **not** update the associated breakout PVs (`ILM_A`–`ILM_H`). Conversely, writing to a breakout PV updates the hardware register and `reg_INPUT_LINK_MASK_RBV`, but does **not** update the `reg_INPUT_LINK_MASK` process variable. Because the GUI control windows use breakout PVs exclusively, all scripts must also use **breakout PVs only** (not whole-register writes) to keep the GUI synchronized with hardware state. ✅ verified 2026-04-17 — `Serdes_Linkup.sh:L7–29` (full explanation in header comment)

- **Double-write pattern:** EPICS can lose track of hardware state. Many PVs are written to the *wrong* value first, then the *correct* value, to force the driver to re-assert the desired state (seen extensively in Stage 2).
- **Fiber expander (2022+):** DC balance must be enabled on MTRG (`EN_RTR_DCBAL`) and each RTRG (`LinkL_DCbal`). Cable pre-emphasis (`PEHLRU/PEEFG/PEABCD`) must be 0 when using fiber.
- **Clock synchronization:** All boards start on local clock. After link lock is confirmed, the system can optionally switch to MTRG-distributed clock (not shown in basic 5-stage init).
- **Author:** JTA (J. T. Anderson, likely) — initials appear in comments for 2022–2024 additions.
- **`Serdes_Linkup.sh`:** Sequential wrapper that invokes Stages 1–5 in order. Sets `SCRIPT_DIR` to `${ANLDAQ_DIR}/gui/scripts/new_scripts` (legacy path — `new_scripts/` subdir does not exist in the current repo; stage scripts live directly in `scripts/`). ✅ verified 2026-04-17 — `Serdes_Linkup.sh:L3` (SCRIPT_DIR assignment), `L48/57/67/78/87` (stage invocations)
  - ⚠️ After completion, **Link L F1 propagation is removed** — cross-system clock/triggering must be reconstituted manually after this script finishes (noted in end-of-script echo block: `Serdes_Linkup.sh:L106–111`)
  - Also documents the EPICS **whole-register vs breakout PV gotcha** (see Key Design Notes below)

---

## Trigger Algorithm Reference (from Stage 1)

| MTRG PV | Algorithm | Notes |
|---|---|---|
| `EN_MAN_AUX` | Manual/Auxiliary trigger | Software-controlled |
| `EN_SUM_X` | X-sum trigger | Sum of discriminator bits in X direction |
| `EN_SUM_Y` | Y-sum trigger | Sum of discriminator bits in Y direction |
| `EN_SUM_XY` | X+Y coincidence sum | |
| `EN_ALGO5` | Algorithm 5 | Configured as coincidence trigger (`ALGO_5_SELECT=1`) |
| `EN_LINK_L` | GITMO / Link L | External system trigger |
| `EN_LINK_R` | MSTR / Link R | Master-Master remote trigger |
| `EN_MYRIAD_LINK_U` | MYRIAD / Link U | Remote trigger (configured as Remote, not MYRIAD: `LINK_U_IS_TRIGGER_TYPE=1`) |
| `COINC_TRIG_MASK_A1`–`B7` | Coincidence mask bits | Which link groups participate in coincidence |
