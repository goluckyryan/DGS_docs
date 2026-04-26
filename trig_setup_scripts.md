# Trigger Setup Scripts — ANLDAQ GUI

Stability: C2 - Active / semi-stable

**Source:** `DGS_tools_pack/ANLDAQ/gui/scripts/trig_setup_Stage{1-5}.sh`  
**Config:** `DGS_tools_pack/ANLDAQ/gui/scripts/SYSTEM_DEFINES.sh`  
**Documented:** 2026-04-17

---

## Table of Contents

- [Overview](#overview)
- [SYSTEM_DEFINES.sh — Gammasphere Configuration](#system_definessh--gammasphere-configuration)
- [Stage 1 — MTRG Initialization](#stage-1--mtrg-initialization)
- [Stage 2 — RTRG Initialization](#stage-2--rtrg-initialization)
- [Stage 3 — Link Lock Verification + RTRG Clock Hand-Off](#stage-3--link-lock-verification--rtrg-clock-hand-off)
- [Stage 4 — DIG Initialization + Router Link Lock Verification + DIG Clock Hand-Off](#stage-4--dig-initialization--router-link-lock-verification--dig-clock-hand-off)
- [Stage 5 — Transition to Live Data](#stage-5--transition-to-live-data)
- [Key Design Notes](#key-design-notes)
- [Trigger Algorithm Reference](#trigger-algorithm-reference-from-stage-1)
- [Cross-References](#cross-references)

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
| `MT_USE_LINK_CLK` | 0 | Use local clock (not link clock) during MTRG init |
| `DIG_CLOCK_SEL` | 1 | DIG final clock: 0=S/D (SerDes), 1=OSC (internal oscillator), 2=S/D (SerDes), 3=AUX — sets `clk_select` PV. Default=1 → DIGs run on internal oscillator (OSC), not phase-locked to RTRG. ✅ verified 2026-04-26 — `MDigUserVME.template:L84` (`ZRST=S/D,ONST=OSC,TWST=S/D,THST=AUX`); `trig_setup_Stage4.sh:L163` echo "0=Serdes,1=internal" correctly identifies 1=internal; `SYSTEM_DEFINES.sh:L116-120` comment says 1=Serdes — WRONG, template is authoritative. |
| `SCRIPT_VERBOSITY` | 1 | 0=stage-level, 1=steps, 2=per-channel |
| `PERFORM_ERROR_CHECKS` | 0 | Enables EPICS PV lock checks at each stage (0=disabled in GS config) |

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
  - Clear all coincidence trigger mask bits (`COINC_TRIG_MASK_*`). ✅ verified 2026-04-20 — `link_sys.py:L119-131` (`reg_COINC_TRIG_MASK`, `COINC_TRIG_MASK_A1`…`B7` all set to 0)
  - Set `ALGO_5_SELECT=1` (Algorithm 5 = coincidence trigger). ✅ verified 2026-04-20 — `link_sys.py:L137`
  - Set `LINK_U_IS_TRIGGER_TYPE=1` (Link U = Remote Trigger, not MYRIAD). ✅ verified 2026-04-20 — `link_sys.py:L143`
- **1D:** Clear all trigger veto enables (NIM, RAM, throttle) for all algorithms A–H. ✅ verified 2026-04-20 — `link_sys.py:L159-163` (`EN_RAM_VETO_A/B/C/D/E` → 0 pattern)
- **1E:** Clear global veto enables (`SOFTWARE_VETO`, `EN_RAM_VETO`, `ENBL_MON7_VETO`, `ENBL_NIM_VETO`, `ENBL_THROTTLE_VETO`). ✅ verified 2026-04-20 — `link_sys.py:L182-186`
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

## Stage 3 — Link Lock Verification + RTRG Clock Hand-Off

**Goal:** Verify all active MTRG–RTRG links are locked, flip RTRGs to MTRG-derived clock, issue first IMP_SYNC.

### Step-by-step:
- Wait 3 seconds for EPICS PV refresh.
- **3A:** Check `LOCK_{link}_RBV` for each unmasked MTRG link (should be "Off" = 0 = LOCKED on DS92LV18). Fail if any unmasked link is not locked.
- **3B:** Check `ALL_LOCKED_RBV` on each RTRG's link L (must be 1). Fail if not.
- **3C+:** Additional link-level checks on each RTRG active link using firmware's more stringent lock-state machine.
- On failure: script exits with error code 1.
- **Clock switch:** Flip each RTRG clock source from local OSC to link clock (`ClkSrc=1`). This synchronizes all RTRG timing to the MTRG. ✅ verified 2026-04-17 — `trig_setup_Stage3.sh:L232`
- **IMP_SYNC:** Assert `IMP_SYNC=1` on MTRG (left asserted, not cleared) — broadcasts ISYNC bit in Frame 1, resetting all MTRG and RTRG timestamp counters simultaneously to zero. DIGs are not yet on a common clock, so their counters are not meaningfully aligned here. ✅ verified 2026-04-17 — `trig_setup_Stage3.sh:L316`

**Note:** EPICS scans lock state ~1 Hz. Stage 3 is a "good enough" pre-check; more stringent firmware checks follow in later stages.

---

## Stage 4 — DIG Initialization + Router Link Lock Verification + DIG Clock Hand-Off

**Goal:** Initialize all DIGs to drive SYNC toward their RTRG, verify RTRG locks, flip DIGs to RTRG-derived clock, issue second IMP_SYNC.

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
- **Clock switch:** Flip each DIG to the desired clock source (`clk_select=DIG_CLOCK_SEL`). In the Gammasphere config `DIG_CLOCK_SEL=1` (OSC = internal oscillator). All DIGs run on their own local oscillator — they are NOT phase-locked to the RTRG clock; timestamp alignment is achieved only through IMP_SYNC. ✅ verified 2026-04-26 — `MDigUserVME.template:L84` (ZRST=S/D, ONST=OSC, TWST=S/D, THST=AUX; value 1 = OSC = internal oscillator); `SYSTEM_DEFINES.sh:L116-120` comments say 1=Serdes — WRONG, template is authoritative; `trig_setup_Stage4.sh:L163` echo "0=Serdes, 1=internal" correctly identifies value 1 as internal (OSC). ⚠️ Prior KB had 1=Serdes — corrected.
- **Second IMP_SYNC:** Assert then immediately clear `IMP_SYNC` (set=1 then set=0). This resets all DIG timestamp counters simultaneously with all RTRG and MTRG counters — completing full three-tier timestamp alignment. ✅ verified 2026-04-17 — `trig_setup_Stage4.sh:L240-241`

**End state:** All DIGs driving SYNC → RTRGs; all links DIG→RTRG→MTRG locked; all boards on MTRG-derived clock with aligned timestamps.

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

- **Double-write pattern:** EPICS can lose track of hardware state. Many PVs are written to the *wrong* value first, then the *correct* value, to force the driver to re-assert the desired state (seen extensively in Stage 2). ✅ verified 2026-04-17 — `trig_setup_Stage2.sh:L62-71` (comment: "EPICS often forgets the state of things... set things twice — once to what you don't want, then again to what you do want")
- **Fiber expander (2022+):** DC balance must be enabled on MTRG (`EN_RTR_DCBAL`) and each RTRG (`LinkL_DCbal`). Cable pre-emphasis (`PEHLRU/PEEFG/PEABCD`) must be 0 when using fiber.
- **Clock synchronization:** All boards start on local clock. Stage 3 flips all RTRGs to the MTRG-derived link clock (`ClkSrc=1`) after verifying MTRG–RTRG lock, then issues `IMP_SYNC` to align timestamps. Stage 4 sets DIGs to `clk_select=DIG_CLOCK_SEL=1` (OSC = internal oscillator — NOT the RTRG link clock) and issues a second `IMP_SYNC` (set+clear) to zero all DIG timestamp counters simultaneously. DIGs run on their own oscillators; timestamp alignment is maintained by the shared IMP_SYNC pulse, not clock distribution. ✅ verified 2026-04-26 — `MDigUserVME.template:L84` (1=OSC); `trig_setup_Stage4.sh:L174-176,L240-241`
- **Author:** JTA — initials appear in comments for 2022–2024 additions (e.g., "addition 20220711 JTA turn ON the DC balance logic"). ✅ verified 2026-04-17 — `trig_setup_Stage2.sh:L86`
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

---

## Cross-References

- [`link_sys_analysis.md`](link_sys_analysis.md) — Python counterpart: `link_sys.py` (same 5-stage sequence, full timing/clock analysis, IMP_SYNC two-shot sync, EPICS CA call chain, error-check mode)
- [`ANLDAQ.md`](ANLDAQ.md) — ANLDAQ GUI: SerdesLinkup button that invokes `link_sys.py`; `Serdes_Linkup.sh` wrapper
- [`ANLDAQ_GUI_windows.md`](ANLDAQ_GUI_windows.md) — GUI Windows: trigger setup scripts section, gui_LinkSys.py integration
- [`deep_fpga_MTRG_MAIN.md`](deep_fpga_MTRG_MAIN.md) — MTRG Main FPGA: registers driven by Stage 1
- [`deep_fpga_RTRG.md`](deep_fpga_RTRG.md) — RTRG firmware: registers driven by Stages 2–4
- [`troubleshooting.md`](troubleshooting.md) — Router lock loss, SYNC bit gotcha (Stage 5 clears SYNC)
