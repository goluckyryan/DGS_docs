# Multi-System Linking — Clock and Trigger Sharing

Stability: C2 - Active / semi-stable

**Source:** [wiki: Linking_Systems_Together](https://wiki.anl.gov/gsdaq/Linking_Systems_Together)  
**See also:** [`link_sys_analysis.md`](link_sys_analysis.md) (single-system linkup), [`ttcl.md`](ttcl.md) (link protocol), [`deep_fpga_MTRG_MAIN.md`](deep_fpga_MTRG_MAIN.md) (MTRG firmware)

---

## Table of Contents

1. [Overview](#overview)
2. [Terminology](#terminology)
3. [Clock Sharing Architecture](#clock-sharing-architecture)
4. [SERDES Jitter Budget and Hop Limit](#serdes-jitter-budget-and-hop-limit)
5. [Firmware Controls for Cross-System Links](#firmware-controls-for-cross-system-links)
6. [Step-by-Step Recipe for Multi-System Linking](#step-by-step-recipe-for-multi-system-linking)
7. [Cross-References](#cross-references)

---

## Overview

The DGS trigger hardware supports linking multiple independent DAQ systems together for **clock synchronization** and **trigger sharing**. Common use cases: DGS ↔ DFMA, DGS ↔ MyRIAD-hosted ancillary detectors (CHICO, Microball, ORRUBA), DGS ↔ Analog Gammasphere (via GITMO).

**Key principle:** Each system runs its own internal MTRG → RTRG → DIG hierarchy. Cross-system linking only connects the Master Trigger boards — it does not change the internal router/digitizer hierarchy.

---

## Terminology

| Term | Meaning |
|------|---------|
| **Timing Monarch** | The master trigger board designated as the source of the common clock. Defined purely by cabling. |
| **Subservient system** | A master trigger that uses the timing monarch's clock (received on its Link L). |
| **Link L** | The only link capable of receiving an external clock for use as the system master clock. |
| **Link R** | Intended to receive trigger/data from another master trigger. |
| **Link U** | Historically used with MyRIAD; can also be programmed to receive from another master trigger. |
| **DEN** | Driver Enable — enables the SERDES cable driver chip (output direction). |
| **REN** | Receiver Enable — enables the SERDES cable receiver chip (input direction). |
| **SYNC** | Controls whether the SERDES chip sends a sync pattern. Must be **OFF** for cross-system links. |
| **ILM** | Input Link Mask — if set, the trigger module ignores data on that link. Both sending and receiving are affected by masking. |
| **F1** | Frame 1 of the TTCL — carries Sync/ImpSync bits. The subservient system listens for clock on Link L once F1 propagation is enabled. |
| **F3–F7** | Higher frames — carry trigger decisions. Propagation control bits select which systems share triggers. |
| **Imperative Sync** | A reset command broadcast to all boards, resetting all timestamp counters to zero simultaneously. Required after clock switchover. |

---

## Clock Sharing Architecture

### How the Timing Monarch is Determined

- Any master trigger can be the timing monarch — it is defined **solely by cabling**, not by software.
- If a master trigger has a cable connected to its **Link L**, it can be told to use the clock from the other end of that cable.
- The hierarchy is defined by what plugs into **Link L** of each board.
- This is why all master-to-router internal cables go from any master link → **Link L** of the router.

### Known Clock Distribution Hierarchy

```
Analog Gammasphere (via GITMO)
    └── DGS MTRG  [timing monarch in DGS+DFMA setup]
            ├── DGS RTRGs (links A–H)
            │       └── DGS DIGs
            └── DFMA MTRG  (subservient, via Link L)
                    ├── DFMA RTRGs
                    │       └── DFMA DIGs
                    └── MyRIAD (ancillary detectors via Link U)
```

### Clock Switchover Process

1. Any master trigger starts on its own **internal oscillator clock**.
2. Writing a register bit causes the subservient master to switch to the clock arriving on Link L.
   - **MTRG register 0x814 (MISC_CLK_CTL_REG), bit 15:** set to '1' to request Link L clock. ✅ verified 2026-04-25 — `top.vhd:L1095` (`xCLK_SRC_SEL <= '0' when ((MISC_CLK_CTL_REG(15) = '1') and (xLINK_LOCK(9) = '0')) else '1'`; comment: `'0' => INB (LINKL_RCLK)  '1' => INA (oscillator)`)
3. The firmware has **automatic fallback**: if the SERDES chip on Link L does not show LOCK, the master automatically reverts to internal clock regardless of the software setting. ✅ verified 2026-04-25 — `top.vhd:L1095`: `xLINK_LOCK(9)` is LOCK* (active-low) from Link L SERDES (`top.vhd:L3288`); the condition `xLINK_LOCK(9) = '0'` means Link L IS locked — if not locked, firmware always selects oscillator, ignoring the software bit.
4. Therefore: **SERDES lock on Link L must be established and verified BEFORE writing the clock-source bit.**

---

## SERDES Jitter Budget and Hop Limit

Each SERDES "hop" (one board's transmitter → another board's receiver) accumulates **~35 ps** of clock jitter (per wiki measurement). ✅ verified 2026-04-25 — wiki: "Measurements show that the accumulated jitter per 'hop' is about 35ps"

**DGS → DFMA round-trip hops:**
1. DGS MTRG → DFMA MTRG (Link L)
2. DFMA MTRG → DFMA Router (Link A–H)
3. DFMA Router → DFMA Digitizer
4. DFMA Digitizer → DFMA Router (return path)
5. DFMA Router → DFMA MTRG
6. DFMA MTRG → DGS MTRG (return on Link R or trigger path)

**Total: 6 hops × 35 ps = 210 ps (arithmetic); wiki states "360ps" total — ⚠️ arithmetic discrepancy: 6×35=210≠360; possible wiki error or actual per-hop jitter is higher than 35 ps.** ✅ verified 2026-04-25 — wiki confirms 6-hop count and 360ps stated total; "35ps/hop" label is wiki-sourced

The TTCL serial stream runs at **1 Gbps = 1 ns per bit**. If accumulated jitter exceeds 50% of the bit period (500 ps), the link will have bit errors. The current DGS-DFMA 6-hop configuration is at or near the practical maximum. ✅ verified 2026-04-25 — wiki: 1Gbps / 1ns per bit / 50% jitter threshold confirmed

**Consequence: A 3rd system hanging behind DFMA will NOT work** — jitter would exceed the bit error threshold.

---

## Firmware Controls for Cross-System Links

### Per Link: DEN / REN / SYNC

| Control | Default (within-system) | For cross-system |
|---------|------------------------|-----------------|
| DEN | ON (internal) | ON |
| REN | ON (internal) | ON |
| SYNC | ON (within-system linkup) | **OFF** — SYNC must be OFF for cross-trigger links L/R/U |

System linkup scripts (`link_sys.py`, `Serdes_Linkup.sh`) leave the SYNC bit ON for internal links. For cross-system links, SYNC must be manually turned OFF before linking.

**MTRG register 0x83C (LINK_LRU_CTL_REG) bit assignments:** ✅ verified 2026-04-26 — `top.vhd:L3040` (`REG_83C => LINK_LRU_CTL_REG`), `L1130-1156` (pin assignments)

| Bit | Signal | Link |
|-----|--------|------|
| 0 | DEN | Link L |
| 1 | REN | Link L |
| 2 | SYNC | Link L |
| 4 | DEN | Link R |
| 5 | REN | Link R |
| 6 | SYNC | Link R |
| 8 | DEN | Link U |
| 9 | REN | Link U |
| 10 | SYNC | Link U |

Links A–H (internal to the system) are controlled by `DEN_BUS[7:0]`/`REN_BUS[7:0]`/`SYNC_BUS[7:0]` driven by the master machine. Readable via VME registers 0x104 (DEN_BUS), 0x108 (REN_BUS), 0x10C (SYNC_BUS). ✅ verified 2026-04-26 — `top.vhd:L3093-3095`

### Input Link Mask (ILM / Gray color in GUI)

- **GRAY** in the GUI = link is masked (disabled)
- **YELLOW** = link is in use/valid
- Both inbound data processing and outbound data sending are blocked when a link is masked (see firmware note below).
- Adjust ILM on links L, R, U of both systems before cross-linking.

**ILM PVs:** ✅ verified 2026-04-26 — `MTrigUser.template:L32405-32516` — ILM PVs exist for A–H (bits 0–7) and L/R/U (bits 8–10) in `reg_INPUT_LINK_MASK` (`REG_800`).  
**Firmware scope:** ✅ verified 2026-04-26 — `top.vhd:L2130` (`link_init` LINK_MASK port = `INPUT_LINK_MASK_REG(7 downto 0)` bits 0–7 only); `eight_mt_channel.vhd:L159` (`CHANNEL_MASK => INPUT_LINK_MASK_REG(i-1)` for i=1..8 = bits 0–7 only). **Bits 8–10 (ILM_L/R/U) are NOT connected to any FPGA masking logic in the 20180507 firmware** — `LINK_LRU_RX` does not accept an INPUT_LINK_MASK port. These bits may be software-only conventions (GUI state only) or may control logic not present in this firmware version.

### F1 Propagation (Clock)

- Enable the **F1 propagation bit** (bit 0 of PROPAGATION_CONTROL_REG) on the subservient system to allow it to listen for the monarch's clock on Link L and propagate ImpSync.
- **MTRG registers:** 0x8D0 = `LINK_L_PROPAGATION_CONTROL_REG`, 0x8D4 = `LINK_R_PROPAGATION_CONTROL_REG`, 0x8D8 = `LINK_U_PROPAGATION_CONTROL_REG`. ✅ verified 2026-04-26 — `top.vhd:L3077-3079` (register address assignments), `L1973` (`REMOTE_TIMESTAMP_ENABLE <= LINK_L_PROPAGATION_CONTROL_REG(0) AND LINK_L_IS_TRIGGER_TYPE`)

**PROPAGATION_CONTROL_REG bit layout** (same structure for L/R/U, applies to 0x8D0/0x8D4/0x8D8): ✅ verified 2026-04-26 — `top.vhd:L4031-4040` (comment block with full bit table)

| Bit | Frame | Content |
|-----|-------|---------|
| 0 | F1 | Sync Frame — clock & ImpSync |
| 1 | F3 | Trigger decision |
| 2 | F4 | Trigger decision |
| 3 | F5 | Trigger decision |
| 4 | F6 | Trigger decision |
| 5 | F7 | Trigger decision |
| 6 | F8 | Trigger decision (internal to trigger system) |
| 7 | F9 | Trigger decision (internal) |
| 8 | F10 | Trigger decision |
| 9 | F12 | Gretina Async Command |
| 10 | F14 | Gretina Async Command |
| 11 | F15 | DGS Sync system capture |
| 12 | F16 | Aux Det Cmd |
| 13 | F17 | Aux Det Cmd |

If a bit is **set**, the machine can propagate (assert) data for that frame. If **clear**, data received in that frame is blocked.

### Clock Source Selection

- Write "link" (vs "internal") to the Clock Source register of the subservient system.
- Some digitizers and routers may drop their link or show "lost lock" after clock switchover — reset and re-link as needed.

### Imperative Sync

- After clock switchover: turn on **Imperative Sync** on the timing monarch.
  - **MTRG register 0x840 (MISC_CTL1_REG), bit 6:** set to '1' to request ImpSync. ✅ verified 2026-04-25 — `top.vhd:L1105` (`IMPERATIVE_SYNC_REQUEST <= ... else MISC_CTL1_REG(6)`); `mstr_mach.vhd:L340` (ImpSync sends command byte 0x81 vs normal 0x01 in Frame 1).
- Verify all boards in all subservient systems respond (master, routers, digitizers show sync acknowledgment).

### F3–F7 Propagation (Triggers)

- Enable F3–F7 propagation control bits (bits 1–5 of PROPAGATION_CONTROL_REG at 0x8D0/0x8D4/0x8D8) in **both systems** to enable bidirectional or unidirectional trigger sharing.
- The monarch enables these bits to send trigger decisions to subservient systems.
- The subservient systems enable them to receive trigger decisions from the monarch.

---

## Step-by-Step Recipe for Multi-System Linking

1. **Run local linkup scripts for each system individually** (each system fully linked internally, all internal SERDES locks verified).

2. **Adjust ILM on cross-system links (L, R, U)** so the triggers are ready to send/receive cross-system data.

3. **Adjust SYNC bits on L/R/U of cross-connected systems** — set SYNC OFF on the cross-system links.

4. **Turn on F1 propagation bit** of the subservient system(s).

5. **Set Clock Source of subservient system(s) to "link"** (will use clock arriving on Link L).
   - Expect some digitizers/routers to drop lock — reset errors and re-link as needed.

6. **Turn on Imperative Sync on the timing monarch** — verify all levels of all subservient systems respond.

7. **Enable F3–F7 Propagation Control bits** in all systems as desired to enable trigger sharing.

---

## Cross-References

- [`link_sys_analysis.md`](link_sys_analysis.md) — single-system linkup procedure (MTRG → RTRG → DIG)
- [`trig_setup_scripts.md`](trig_setup_scripts.md) — individual setup scripts (Serdes_Linkup.sh, etc.)
- [`ttcl.md`](ttcl.md) — full TTCL protocol: frame formats, sync, ImpSync, trigger propagation
- [`deep_fpga_MTRG_MAIN.md`](deep_fpga_MTRG_MAIN.md) — MTRG FPGA: clock mux (ICS581), Link L/R/U receivers, F1/F3 propagation
- [`deep_fpga_RTRG.md`](deep_fpga_RTRG.md) — RTRG register map: LINK_LRU_CTL (DEN/REN/SYNC for L/R/U)
- [`myriad.md`](myriad.md) — MyRIAD module (ancillary detector trigger expansion, Link U)
- [`connectors.md`](connectors.md) — physical connections: Links A–H (RTRG), L/R/U (cross-system)
- [`260E_trigger_scheme.md`](260E_trigger_scheme.md) — RTRG 0x260E trigger data path and MTRG trigger decision end-to-end (timing diagram includes DUO example)

*Created: 2026-04-25. Source: wiki.anl.gov/gsdaq/Linking_Systems_Together.*
