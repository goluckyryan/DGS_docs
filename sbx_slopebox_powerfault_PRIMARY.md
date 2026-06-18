# SBX → Slope Box Power-Fault — PRIMARY Investigation Report

**Status:** DRAFT — code-dive analysis. No production changes. Multiple hypotheses presented; none yet has a confirmed smoking gun.
**Date:** 2026-06-17
**Investigator:** General DGS, with Michael Oberling (hardware/systems lead)
**Companion:** `sbx_slopebox_powerfault_SECONDARY.md` (all other findings, preserved)
**Working notes:** `~/.openclaw/workspace/scratch/sbx_deepdive_notes.md`

---

## 1. Problem statement (from Michael)

- Writing the **`PreampI2C_OE_CTL`** PV — to **enable OR disable, even to the state it is already in** — **100% reliably** drives the SBX / slope box into a configuration that **draws excessive current and raises a power fault**. Real, repeatable hardware effect.
- Other PVs *sometimes* trigger a similar fault; `PreampI2C_OE_CTL` is the only 100% trigger.
- Recovery requires a **power-cycle of that SBX via the collector**.
- Michael's assessment: **bus contention with the preamp is NOT the cause** (precluded by part selection). Most likely the **SBX is communicating something incorrectly to the slope box, or corrupting it**. Corruption could also be **upstream** (Pi IOC → device support → SPI → stripe). Nothing fully ruled out without a clear smoking gun.

---

## 2. What `PreampI2C_OE_CTL` actually does (verified, exact)

EPICS record (`Pickoff.db`):
```
record(bo, "GS${DetNbr}_PreampI2C_OE_CTL")  -- "block" / "I2C_enbl"
   field(DTYP,"CollectorLocalSerial")
   field(OUT, "#B$(BusAddr) C0 N1 A0 F8 @")   -- RMW on FPGA_CTL_REG (addr 1), bit 8
```
Device support `devBoCollector` / `write_bo` (`CollectorSupport_BO.c`), RMW branch (bare `@`):
1. `Do_SPI1_transaction(READ, Bidx, addr=1, 0)` → read FPGA_CTL_REG
2. `SPI_data_out = read & 0x00FFFF`
3. `AndMask = ~(1<<8) = 0xFEFF`
4. `SPI_data_out &= 0xFEFF` (clear bit 8); if PV=1, `|= 0x0100` (set bit 8)
5. `Do_SPI1_transaction(WRITE, Bidx, addr=1, SPI_data_out)`

FPGA side (`SlopeboxInt_TopLevel_RevC.vhd`):
- `FPGA_CTL_REG(8)` → `int_Preamp_I2C_OE` → output **pin `PREAMP_I2C_OE` (UCF P64, LVCMOS33)** = enable for an **external I2C buffer/level-translator** ("active High, ref +3.3V").
- Write of addr 1 also shadows the value into DPRAM (`DPRAM_WEA<='1'`, `:1824`); read mux returns the live register (`:1971`). **So the RMW is logically self-consistent** — it reads the true value, changes only bit 8, writes back. The other 15 bits are preserved by value.

### 2.1 Critical constraint from "same-state still triggers it"
Because writing bit 8 to its existing value still triggers the fault, the cause **cannot depend on the logical value of bit 8 changing**. It must be one of:
- (a) a **hardware effect of the write/enable event** (the buffer-enable pin and/or the I2C electrical state), or
- (b) an **upstream transaction-level side effect** (the SPI/DEVSEL/stripe path), or
- (c) a side effect of the **RMW read** of addr 1, or
- (d) the **whole-register re-drive**: every RMW re-writes all 16 bits of FPGA_CTL_REG in one FPGA clock, momentarily re-driving bits 0/1 (gain reset), 9 (ResetAllScanMachines), 11/12/13 (I2C FIFO resets), 14 (I2C transactor reset), 7/15 (power converter EN/OE). Even though the *values* are unchanged, the act of the synchronous register load re-applies them.

---

## 3. What the firmware decouples (verified — narrows the search)

These were checked and **do NOT propagate** from a `PreampI2C_OE_CTL` (FPGA_CTL_REG) write to the slope box:

- **Slope-box scanner reset is independent.** `SLOPE_BOX_SCAN_RST` ← `ResetSlopeBoxScan` = `PULSED_CONTROL_REG(0)` (a different register), not FPGA_CTL_REG. `ResetAllScanMachines` = FPGA_CTL_REG(9) resets **POWER/PREAMP/DONGLE** scanners only (`:3316/3468/3601`), **not** the slope box scanner. → a bit-9 re-drive does not reset the slope box link.
- **Slope-box command FIFO is not fed by addr 1.** The slope box command FIFO write (`SLOPEBOX_CMD_BUF_FIFO_WE`) only fires for **device "1101" = MACH_ENABLEs(13)**. Addr 1 decodes to device **"1110" = MACH_ENABLEs(14)** (register write). → `PreampI2C_OE_CTL` does **not** push a command into the slope box FIFO.
- **Clock-source switch is value-gated.** `UseTrigClock` = FPGA_CTL_REG(10) drives `PLL_INPUT_SEL` via a latch that only changes on a value change of bit 10. Since the RMW preserves bit 10, no clock switch occurs.

**Implication:** From firmware logic alone, the only thing a correct `PreampI2C_OE_CTL` RMW changes is the **external buffer enable pin** plus a **one-cycle synchronous re-drive of the whole control register**. Everything else is value-preserved.

---

## 4. Ranked hypotheses

### H1 — Whole-register re-drive perturbs the preamp I2C engine / external buffer enable, and the resulting I2C-bus electrical state propagates to the slope box rail (MEDIUM, best fit to "same-state triggers")
Every RMW re-loads all 16 FPGA_CTL_REG bits in one clock. Bits 11/12 (`PowerBoardI2CFifoReset`, `PreampI2CFifoReset`), 9 (`ResetAllScanMachines` → preamp/power/dongle scanner reset), 14 (`I2CTransactorReset`) are re-driven. If the preamp/power I2C scanner is mid-transaction, this re-drive resets it asynchronously, potentially **leaving SDA/SCL or the external buffer in a partially-driven / wrong state**. Even if the preamp itself can't contend (per Michael), the **power board** shares the I2C transactor infrastructure (`POWER_BOARD_WRITE_PORT`, power I2C scanner). A disturbed power-board I2C transaction could mis-set a power-board control → excessive current / power fault.
- **Why it fits "same-state":** the disturbance is from the register *load event*, not bit 8's value.
- **Test:** with IOC trace on, toggle `PreampI2C_OE_CTL` and watch `POWER_ACK_ERR_COUNT` / `PwrStatus` / power-board I2C scanner state; check if a power-board I2C transaction is in flight at the time.
- **Caveat:** requires the I2C scanner to be active at the moment of the write; would not be 100% unless the scanner is essentially always busy (CYCLE_DELAY small / continuous).

### H2 — RMW **read** of FPGA_CTL_REG returns a corrupted value, so the write-back flips a dangerous bit (MEDIUM-HIGH, needs bus capture)
The RMW depends entirely on the read returning the true register. If the **read** is occasionally wrong (timing/prefetch/DEVSEL race — see §5 upstream), the write-back can flip bits 7/15 (power-converter EN/OE), 9, 11–14. Flipping **bit 7 (PowerConverterEN)** or **bit 15 (OE)** would directly cause a power state change → fault. Flipping I2C-reset bits could wedge the power-board I2C → mis-config.
- **Why it could be ~100%:** if the read corruption is systematic for addr 1 under the current DEVSEL/CE timing (not random), it would reproduce every time.
- **Tell:** the *value written back* would differ from the value read in a captured SPI trace. This is the **smoking gun to look for**: capture the read word and the write word on a `PreampI2C_OE_CTL` toggle; if `write != (read with only bit 8 changed)`, H2 is confirmed.
- **Strong because:** it's the cleanest explanation for "even a no-op write corrupts" — a no-op write still does the read→write, and a bad read poisons the write.

### H3 — Upstream DEVSEL/CE/stripe framing race injects a malformed transaction (MEDIUM, upstream)
See §5. `Do_SPI1_transaction` sets DEVSEL then **immediately** drops CE with no settle delay; `Set_DEVSEL(0)` is dead code so DEVSEL is sticky between transactions; DEVSEL momentarily passes through `00000` (stripe globals) during every setup. A framing race could cause the **stripe to forward the transaction to the wrong target or to globals**, or the FPGA to latch a wrong address/data. If a malformed write lands on a slope-box-forward register or a power control, → corruption/fault.
- **Why `PreampI2C_OE_CTL` specifically:** not obviously address-specific; would predict other PVs fail too (which they sometimes do). Could be the mechanism behind the "other PVs sometimes trigger it."
- **Test:** SPI/logic-analyzer capture of DEVSEL + CE + MOSI for a `PreampI2C_OE_CTL` write; check DEVSEL settled before CE and that the framing is clean.

### H4 — External buffer enable interacts with the slope-box/preamp shared analog domain in a way not visible in firmware (LOW-MEDIUM, hardware; Michael deprioritizes)
Asserting/re-pulsing `PREAMP_I2C_OE` (P64) enables an external buffer. Michael says part selection precludes bus contention. **Retained only as a hardware-doc-gap item** — the buffer part/topology and the board-level interaction between the preamp I2C domain and the slope-box supply are **not in the repo** (no schematic). Cannot confirm or fully exclude from source. **Will not guess the part.**

### H5 — The fault is on the **slope box** side from a previously-corrupted forwarded register, and `PreampI2C_OE_CTL` is merely the trigger that exposes it (LOW, but explains intermittency of "other PVs")
Independent, verified bug (see SECONDARY §S-HV): `GE_HV_CTRL`/`BGO_HV_CTRL` are RMW on a **write-only forward register** the SBX cannot read back, so they can forward stale/garbage HV-control words to the slope box. This is a real corruption-of-slope-box mechanism, but it is **not** driven by `PreampI2C_OE_CTL`. Listed here because it matches "SBX communicating something incorrectly to the slope box" and may be one of the "other PVs that sometimes trigger it."

---

## 5. Upstream (Pi → SPI → stripe) findings relevant to H2/H3

- **`spi.c` `Do_SPI1_transaction` never clears DEVSEL** — `Set_DEVSEL(0)` is **after** the `return` (dead code). DEVSEL stays parked on the last device between transactions.
- **No settle delay between `Set_DEVSEL(Bidx)` and dropping CE.** `Set_DEVSEL` does `clr_multi` (all DEVSEL→0) then `set_multi` (target), then the very next call `bcm2835_peri_write(io,...)` drops CE and clocks. The DEVSEL lines must propagate through the collector cable + stripe fanout before the first SPI edge; there is only GPIO write latency of margin. Potential leading-edge mis-select.
- **DEVSEL transiently = `00000` (stripe globals) during every setup** (between clr and set). CE is not yet asserted, so likely benign, but it means the "globals" path is briefly selected on every transaction.
- **Read-data alignment:** the read returns `bcm2835_peri_read(txhold)`; device support takes `& 0x00FFFF` (upper 8 bits are FPGA status). If the FPGA's read-data window (`DATA_PRESAMPLE_GUARD`/prefetch) is mis-timed relative to the AUX-SPI framing, the read could return stale/adjacent data → feeds H2.
- **`SpiMutexLock(0)` scope:** the read and the write in the RMW are **separately** locked/unlocked (two independent lock/unlock pairs). Between them the mutex is released, so **another EPICS thread can interleave an SPI transaction between the RMW read and write** — changing DEVSEL and potentially the targeted register's neighbors. This is a real **RMW atomicity hole**: the read-modify-write is not atomic across the SPI bus. If another transaction to the same Bidx/addr (or one that changes FPGA_CTL_REG) sneaks in between, the write-back is based on a stale read → corruption. (Applies to all RMW records, not just this one.)

> **The RMW atomicity hole (§5 last bullet) + H2 is, in my assessment, the most promising concrete, source-visible mechanism** and the first thing to test, because it explains corruption that is (a) write-event driven, (b) not value-dependent, and (c) intermittently affecting "other PVs" too.

---

## 6. Recommended diagnostics (non-destructive, to find the smoking gun)

0. **Read the stripe `DEVSEL_GLITCHED` / `PI_CE_GLITCHED` status bits** (`FPGA_STATUS(0)`/`(1)`) immediately after a fault — zero-cost test of **T6**. If set, the DEVSEL/CE timing race is confirmed as the comm-kill mechanism.
1. **SPI bus capture** (logic analyzer on `PI_DEVSEL[4:0]`, `PI_CE`, `FILT_PI_CE`, stripe `DEVSEL`, SCLK, MOSI, MISO, and the faulting `COLL_SDO/SCK/CE`) during a single `PreampI2C_OE_CTL` toggle:
   - Confirm DEVSEL settled before CE (tests T6/H3); watch for the SBX channel dropping to the parked state `COLL_SDO=1, SCK=0, CE=1` and staying there (tests T5/T6).
   - Capture the **read word** and the **write-back word**; verify `write == read with only bit 8 changed` (tests H2 — if not equal, H2 confirmed).
2. **IOC trace** (`GLBL_CollectorControlVals[Bidx][2/6]=1`) to print the read/write values from `write_bo` for that PV.
3. **Power-board telemetry** at the moment of fault: `PwrStatus`, `POWER_ACK_ERR_COUNT`, LV monitor rails, fan/temp — determine whether the fault is reported by the **power board** vs a **supply-rail collapse** (tests H1 vs H2/H4).
4. **Concurrency probe:** check whether multiple scan threads / CA puts can hit the SPI between the RMW read and write (tests §5 atomicity hole).
5. **Schematic:** obtain the SBX schematic for the `PREAMP_I2C_OE` (P64) external buffer part + the power-board control topology (resolves H4 and clarifies what "excessive current" path exists).

---

## 6b. DEEP PASS (Sweep 3) — FSM / async / unconstrained-path findings (Michael's "comm path goes dead, line stuck 1/0, latches" model)

Michael's refined fault model: in the fault state the **comm path to the SBX goes dead** and a line is **stuck asserted at 1 or 0**, producing a real **overcurrent + undervoltage that latches (loss of control)** until power cycle. He asked specifically for cross-logic / state-machine interactions, asynchronous signals, and unconstrained paths, and stressed that **the action of a write (even rewriting the same value) triggers it**.

### T6 (STRONGEST fit; testable NOW via existing status bits) — stripe DEVSEL freeze + Pi DEVSEL-sticky/no-settle → target SBX parked (comm dead)
Chain (source-visible, value-independent, timing/async):
- **Pi side (`spi.c`):** `Set_DEVSEL(Bidx)` does `clr_multi`(DEVSEL→00000) then `set_multi`(target); then `bcm2835_peri_write` **drops CE immediately, no settle delay**. And `Set_DEVSEL(0)` is **dead code**, so the Pi never returns DEVSEL to a defined idle — it holds the last target until the next set, and briefly passes through 00000 on every setup.
- **Stripe side (`Top.vhd` `DEVSEL_LATCH`):** the routing `DEVSEL` is updated **only when `PI_CE_PIPE="1111111111"`** (CE filtered high for 10 cycles); it is **frozen during a transaction**. `FILT_PI_CE` filters CE (2 stable clocks each way). Glitch flags `DEVSEL_GLITCHED`/`PI_CE_GLITCHED` are set but **do not abort** the transaction.
- **Race:** if the filtered CE is seen low before the new DEVSEL has propagated/stabilized through the collector cable + stripe, the stripe can **freeze DEVSEL on a stale or 00000 value** for that transaction. `DEVSEL=00000` routes to stripe **globals** (and, if `SBX_GLOBAL_WRITE_ARMED`, broadcasts to all SBX). A wrong DEVSEL routes the write to the wrong SBX/globals and leaves the **intended SBX channel in the `sbx_mux` default park state: `COLL_SDO='1', COLL_SCK='0', COLL_CE='1'`** = **comm dead, data line stuck high, no clock**. With no clock/CE, the SBX loses control updates → its outputs hold/lapse → overcurrent/undervoltage that persists until power cycle.
- **Why it fits all constraints:** value-independent (it's the transaction/DEVSEL timing, not the data); "the action of a write" triggers it; ends in a dead/stuck comm path that latches.
- **Built-in diagnostic (do this first):** the stripe already exposes `DEVSEL_GLITCHED` and `PI_CE_GLITCHED` on `FPGA_STATUS(0)`/`FPGA_STATUS(1)`. **Read these after a fault** — if set, this chain is confirmed.
- **Confidence:** medium; depends on real CE/DEVSEL propagation timing. Needs a logic-analyzer capture of `PI_DEVSEL[4:0]`, `PI_CE`, `FILT_PI_CE`, stripe `DEVSEL`, `COLL_SDO/SCK/CE`.

### T4 (REAL RTL defect) — `sbx_mux` incomplete sensitivity list
Stripe `sbx_mux : process(STRIPE_CODE, DEVSEL, PI_MOSI, PI_SCK, PI_CE)` **reads** `TRISTATE_CTL_REG(i-1)`, `SBX_GLOBAL_WRITE_ARMED`, and `FILT_PI_CE` — **none of which are in the sensitivity list**. Sim/synth mismatch and fragile combinational routing of the SBX serial lines. The default branch parks `COLL_SDO='1'/SCK='0'/CE='1'` (the dead/stuck state). This is exactly the class of unconstrained/async path Michael asked to hunt; it should be fixed regardless of whether it is the 100% trigger.

### T5 — SBX channel parks (comm dead) whenever DEVSEL never matches
`sbx_mux` drives a channel from the Pi only when `(STRIPE_CODE & DEVSEL)` matches the channel's slot; else parks SDO='1'/SCK='0'/CE='1'. A channel is **permanently parked** if (a) `STRIPE_CODE` strap ≠ DB `N`-value (S-STRIPE-STRAP) → no DEVSEL ever matches → dead until re-strap/power; (b) `TRISTATE_CTL_REG` leaves it hi-Z/parked; (c) DEVSEL-sticky from `spi.c`. State conditions, not value-driven.

### T1 — async SCK/CE edge detection without a true synchronizer
`PI_TRANSACTOR.vhd` SCK/CE edge detectors do `PIPE_1 <= async_input; PIPE_2 <= PIPE_1; edge=(PIPE_2 vs PIPE_1)`. `PIPE_1` is the first (metastability-prone) capture and is used directly for edge detection — **no dedicated metastability-resolving flop**. A metastable `PIPE_1` → spurious/missed SCK edge → shift FSM miscounts `SERIAL_CLOCK_CNT` → 24-bit frame misaligned → **wrong address/data latched and committed** (could form a bad slope-box command, or write a wrong register). Random in nature → better fits "other PVs sometimes" than 100%-on-one-PV, but a real robustness hole and a genuine corruption vector.

### T2 — SCK source mux and CE source mux selected by DIFFERENT logic
CE source = `CE_SWITCH_STATE` FSM (ROM/VIO/DVI via `COLLECTOR_IS_IDLE`/LOCAL via `LOCAL_PI_IS_IDLE`, priority + hold + `WAIT_NO_CE`). SCK source = a separate `if` chain keyed on `PIPRESENCESENSE`/`TRIG_CLK_PRESENT`. If the two **disagree** about the active source (e.g., CE in `CE_FROM_LOCAL` while SCK picks DVI, or SCK falls to the `else '0'` branch), the shift FSM is clocked with the wrong clock or **SCK stuck '0'** → no edges → FSM stalls in `WAIT_CLK_*` (only `WATCHDOG_RESET`/`CE_POS_EDGE` rescue) → transaction frozen/dead. (Note: `TRIG_CLK_PRESENT` is generated by a robust oscillator-domain frequency monitor, not by a register write, so a `PreampI2C_OE_CTL` write does not itself drop it.)

### Cross-logic note on "corrupted monitor misinterpreted"
The slope-box scanner continuously reads ADC monitors into the cross-domain FIFO/DPRAM. T1 (framing miscount) or a DPRAM access collision during the addr-1 write's `DPRAM_WEA<='1'` shadow could corrupt a monitor word; downstream IOC conversion (or a firmware interlock that consumes monitor data) could then misinterpret it. Not proven to drive HV, but consistent with Michael's "even corrupted normal monitor could be misinterpreted." Flagged for the monitor→interlock data path review.

### Updated overall ranking (combining all sweeps, for the comm-dead/latched model)
1. **T6** — DEVSEL freeze + Pi DEVSEL-sticky/no-settle → SBX comm parked. *Testable now via stripe `DEVSEL_GLITCHED`/`PI_CE_GLITCHED` status bits.*
2. **T4** — `sbx_mux` incomplete sensitivity list (real RTL defect; sim/synth mismatch).
3. **H2 / S-RMW** — RMW read corruption / non-atomic RMW flips power/I2C-reset bits.
4. **T1** — async SCK/CE edge detect (no true synchronizer) → framing miscount → wrong command/register.
5. **T2** — SCK vs CE source-select disagreement → SCK stuck '0' → comm stall.

## 6c. SCOPING CONSTRAINT (Michael, 2026-06-17 15:51) — affects ONE detector only, not a stripe or collector

This is a decisive filter. The fault is **localized to a single SBX / slope box**. It demotes or rules out every stripe-/collector-wide candidate and refocuses on per-detector mechanisms.

**Ruled out / demoted (stripe- or collector-wide scope):**
- **T6** (stripe DEVSEL freeze / Pi DEVSEL-sticky): a mis-latched/00000 DEVSEL routes to stripe **globals** or a *different* channel — stripe/collector-level or wrong-detector effect, not a clean single-detector latch. **Demoted.**
- **T4** (stripe `sbx_mux` sensitivity list), **T2** (SCK/CE source disagreement), **T1** (async edge detect): shared stripe logic or collector-wide clock — would tend to affect multiple detectors. **Demoted.**
- **S-STRIPE-GLOBAL / S-STRIPE-STRAP**, **DEVSEL stickiness**, **DPRAM collision**: stripe/collector scope or ruled out. (DPRAM collision explicitly checked: addr-1 write → DPRAM port A addr 1; slope-box monitors → port B addr 29–36; different addresses → no collision.) **Ruled out.**

**Promoted (per-detector scope — fit single-detector + write-action + latched):**
1. **H2** — RMW **read** of *that detector's* `FPGA_CTL_REG` returns a corrupted value → write-back flips *that detector's* `PowerConverterEN` (bit 7) / `OE` (bit 15) or I2C-reset bits → single-detector power/HV fault, latched until that SBX is power-cycled. **Best fit.** Test: capture read word vs write-back word for the PV (IOC trace or SPI capture).
2. **S-RMW** — non-atomic RMW (mutex released between read and write) → a concurrent SPI op to *that BusAddr* poisons the write-back. One BusAddr = one detector. Write-action-driven, value-independent.
3. **Slope-box-local latch** — once any bad word reaches the slope box's HV/interlock logic, it latches overcurrent/undervoltage until power cycle. This is the **latch stage**; it needs a trigger (H2/S-RMW/S-HV/hardware) to set it. Matches "latched until power cycle, one detector" exactly.
4. **S-HV** — RMW on the forwarded `SLOPE_BOX_HV_CTL` (per detector) → stale HV-ctrl word to one slope box. Fits scope, but driven by the HV_CTRL PVs — a likely "other PV that sometimes triggers it," not the `PreampI2C_OE_CTL` 100% case.
5. **Per-detector hardware interaction of `PREAMP_I2C_OE`** (buffer-enable event) — **not visible in firmware**; needs the SBX schematic. Michael deprioritized bus contention, but a per-detector hardware effect of the *enable event itself* cannot be excluded from source alone.

**Honest synthesis:** the firmware shows **no clean logical path** from the *value* of `PreampI2C_OE_CTL` (addr 1 bit 8) to that slope box. Combined with single-detector scope + write-action trigger + latched-until-power-cycle, the evidence points to either (a) a **corrupted RMW of that one detector's `FPGA_CTL_REG`** flipping its own power/I2C bits (testable via read-vs-write capture / IOC trace), or (b) a **per-detector hardware effect of the buffer-enable event** that requires the SBX schematic to resolve. The slope-box-local latch then holds the fault until power cycle.

## 7. Honest limits
- No schematic / netlist in the repo (history-less SVN snapshot) → cannot confirm hardware-level contention or the buffer part (H4) or the exact power-fault sense path.
- No live bus capture available in a code dive → H2/H3 need a logic-analyzer trace to confirm.
- The firmware logic **does not** show a direct, value-independent path from `PreampI2C_OE_CTL` to the slope box; the most defensible source-visible mechanisms are the **RMW read-corruption + non-atomic RMW** (H2 + §5) and the **whole-register re-drive disturbing the power-board I2C** (H1).
- Nothing is fully excluded; this is consistent with Michael's "can't rule out anything without a clear smoking gun."
