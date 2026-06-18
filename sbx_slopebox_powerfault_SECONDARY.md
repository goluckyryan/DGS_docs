# SBX / Slope Box — SECONDARY Findings (Full Preservation)

**Status:** DRAFT — code-dive findings. No production changes.
**Date:** 2026-06-17
**Investigator:** General DGS, with Michael Oberling
**Companion:** `sbx_slopebox_powerfault_PRIMARY.md` (ranked hypotheses for the `PreampI2C_OE_CTL` power-fault)
**Source of truth:** SBX FPGA = `FPGA/collectorBox/PickoffCard_SBX_Extension/Revision_C/Source/SlopeboxInt_TopLevel_RevC.vhd` (+ `LOOK_UP_TABLE1.VHD`, `PI_TRANSACTOR.vhd`, `SlopeBoxScan.vhd`, `I2C_*`); Stripe FPGA = `CollectorBox_StripeFPGA/Source/Top.vhd`; IOC = `collectorboxpi/CollectorBox_RevA/` (`db/*.db`, `CollectorApp/src/*.c`).

> This document preserves **all** findings from the deep dive so nothing is lost, including ones not on the critical path for the 100% trigger.

---

## Architecture reference (verified)

**Layer stack (Michael's authoritative framing):** EPICS Pi softIOC → HW↔EPICS device support (mailboxes/scanning/SPI, the custom hacks) → control/fanout FPGA → stripe FPGA (3 stripes, ~10 ports each; power relays, clock/sync fanout, SPI tristate) → BGOp FPGA (honeycomb suppression; possibly DS92LV18 trigger link, not deployed) → SBX pickoff FPGA (pickoff, slopebox, power board, BGO dongle).

**FPGA parts:** CtrlFPGA = Spartan-6 `XC6SLX9-2TQG144` (older boards LX4). Stripe = originally Spartan-6 `XC6SLX4`, switched to Spartan-3 `XC3S400-4TQG144C` (Oct 2021, supply chain).

**SBX transaction format:** 24-bit: bit23 R/W, bits22-16 = 7-bit address, bits15-0 = data. `PI_TRANSACTOR.vhd` (entity `SERIAL_CTL_MACH`) latches it; `LOOK_UP_TABLE1` decodes address → 4-bit DEVICE_ADDRESS → one-hot `MACH_ENABLEs`.

**Address → machine map (LOOK_UP_TABLE1.VHD), key device codes:**
- "1110" (MACH_ENABLEs 14) = register read/write inside the FPGA (e.g. addr 1 = FPGA_CTL_REG)
- "1101" (MACH_ENABLEs 13) = **forward to slope box** via `SLOPEBOX_CMD_BUF_FIFO` (BGO_DAC_DEMAND0-13, SLOPE_BOX_HV_CTL 0x53, SLOPE_BOX_GEHV_DEMAND 0x54, and the LAST_SLOPEBOX_ADC_VAL readback addrs 0x1D-0x24)
- "0111"/"1000"/"1010" = preamp / power-board / dongle I2C command FIFOs
- "1001" = DPRAM bank read

**FPGA_CTL_REG (addr 1) bit map (verified `:2047-2061`), reset `X"7A03"`:**
0 GeC_GainReset · 1 BGOsum_GainReset · 2 BGOCounterMode · 3 PARSTCounterMode · 4 PARSTContinuousMode · 5-6 PARST_EdgeSel · **7 PowerConverterEN** · **8 Preamp_I2C_OE** · 9 ResetAllScanMachines · 10 UseTrigClock · 11 PowerBoardI2CFifoReset · 12 PreampI2CFifoReset · 13 DongleI2CFifoReset · 14 I2CTransactorReset · **15 PowerConverterOE**.
Reset `X"7A03"` → bit7=0, bit8=0, bit15=0 (power converter + I2C OE OFF at reconfig); bit9=1, 11/12/13/14=1 (held in reset at boot).

---

## S-HV — RMW on write-only forward registers corrupts slope-box HV (REAL, separate bug)

`GE_HV_CTRL` and `BGO_HV_CTRL` (`Pickoff.db`) BOTH target **N83 = SLOPE_BOX_HV_CTL = 0x53**, both **bare `@` (RMW)**, mode C0 (no DPRAM copy):
- `GE_HV_CTRL`  → `#B.. C0 N83 A0xFFF3 F2 @` (bits 3:2)
- `BGO_HV_CTRL` → `#B.. C0 N83 A0xFFFC F0 @` (bits 1:0)

0x53 is device "1101" (write-forward to slope box). In the SBX read mux, 0x53/0x54/BGO_DAC_DEMAND are **NOT decoded for readback** — a read falls through to bank-0 default `PREFETCH_DATA <= DPRAM_DOA` (`:2021-2022`). The SBX has **no readback of the slope box's true HV-control state**.

So the RMW reads DPRAM[0x53] (stale / init 0, never refreshed since mode C0), ORs the new HV bits, and writes the whole word **forward to the slope box**. The other HV channel's bits (and any other bits) are written from DPRAM, not the slope box's true state → can drive an **invalid HV-control combination** into the slope box → trips interlock/power fault → latches (SBX can't read back to repair) → power-cycle to recover.

- Smoking gun that this is a real slip: the HV **demand** records (`CHAINED_GE_HV_DEMAND`, `MANUAL_GE_HV_DEMAND`, `DIRECT_MANUAL_GE_HV_DEMAND`) all correctly use **write-only `@X`**. The on/off controls didn't.
- **Fix (software-only):** make HV on/off `@X` + IOC shadow of SLOPE_BOX_HV_CTL (compute both fields together from shadow). No FPGA rebuild.
- **Relevance to PRIMARY:** matches "SBX communicating something incorrectly to the slope box." Plausibly one of the "other PVs that sometimes trigger it." NOT the `PreampI2C_OE_CTL` trigger.

### S-HV-2 — General RMW-on-forwarded-address hazard
Any `CollectorLocalSerial` writable record using bare `@` whose address decodes to "1101" has the same stale-readback corruption risk. **Audit every such record.** The HV demand path is safe (`@X`); audit DAC demands and any other 0x45–0x54 writers.

---

## S-RMW — Non-atomic RMW across the SPI bus (REAL, affects all RMW records)

In `CollectorSupport_BO.c` / `CollectorSupport_AO.c`, the RMW does:
```
SpiMutexLock(0); read = Do_SPI1_transaction(READ,...); SpiMutexUnLock(0);
... modify ...
SpiMutexLock(0); Do_SPI1_transaction(WRITE,...); SpiMutexUnLock(0);
```
The mutex is released **between** read and write. Another EPICS thread can interleave an SPI transaction in that gap (changing DEVSEL, or writing the same register's neighbors). The read-modify-write is therefore **not atomic** → a concurrent write can be lost or the write-back can be based on a stale read. Affects every RMW record (incl. FPGA_CTL_REG / `PreampI2C_OE_CTL`). See PRIMARY H2/§5.

---

## S-DEVSEL — `Do_SPI1_transaction` never clears DEVSEL; no settle delay (REAL, medium)

`spi.c`: `Set_DEVSEL(0)` is placed **after** `return` → dead code. DEVSEL stays parked on the last device between transactions (never returns to idle/0). Also, `Set_DEVSEL(Bidx)` (clr_multi then set_multi) is immediately followed by `bcm2835_peri_write(io,...)` which drops CE and clocks — **no explicit settle delay** for DEVSEL propagation through the collector cable + stripe fanout. DEVSEL also momentarily passes through `00000` (stripe globals) during each setup (between clr and set; CE not yet asserted). See PRIMARY H3.

---

## S-AO2 — `CollectorSupport_AO.c` MailboxMode 2 double-shift (REAL)

Case 2 (indirect data) shifts `UsrData` by `ShiftFactor` **twice**:
```
case 2: UsrData = GLBL...[Bidx][Cidx] << ShiftFactor;
        UsrData = UsrData << ShiftFactor;   // shifted again
```
Cases 0/1/3 shift once. Any mode-2 PV with F!=0 writes an over-shifted value. Audit for mode-2 records with nonzero F.

---

## S-SBXCMD — Slope-box command FIFO data/pop timing (POSSIBLE, for "1101" writes)

`SLOPEBOX_CMD_BUF_FIFO` stores `{LATCHED_PI_ADDRESS, LATCHED_PI_DATA}`; `SlopeBoxScan` builds the slope-box transaction from the FIFO DOUT (`LATCHED_PI_DATA`/`LATCHED_PI_Address`). `IMPLIED_SLOPEBOX_COMMAND` (LUT `SlopeBoxScan.vhd:673-801`) maps the FIFO address → 4-bit command (most → X"C" ADC read; BGO_DAC → X"4"/X"6"; HV_CTL → X"2"; GEHV_DEMAND → X"8"). `COMMAND_BUFFER_RE` pops the FWFT FIFO in state `PI_TRANSACTION_START`; the transactor latches `SLOPEBOX_COMMAND` from `LATCHED_PI_DATA` at IDLE→START. **Concern:** if `SLOPEBOX_COMMAND` samples FIFO DOUT after the RE pop changes it (or when the FIFO is FWFT-empty → stale DOUT), the slope box gets a command with wrong/stale data. **Unconfirmed** — needs exact cycle-alignment trace of the transactor START vs RE pop. Only reachable via "1101" writes (not `PreampI2C_OE_CTL`).

---

## S-STRIPE-GLOBAL — Stripe `SBX_GLOBAL_WRITE` broadcast (LATENT, not used today)

Stripe `Top.vhd`: writing stripe **global addr 48** (DEVSEL=00000) arms `SBX_GLOBAL_WRITE`; the **next** Pi transaction is broadcast to **all 5 SBX channels at once** (`sbx_mux`: `((SBX_GLOBAL_WRITE_ARMED='1') and (DEVSEL="00000"))` forwards PI_MOSI/SCK/CE to every channel with `TRISTATE_CTL_REG(i)='1'`). No current EPICS record issues this (only an unrelated N48 *read* exists on the SBX side). **Latent hazard:** accidental arming, or future use, or — combined with S-DEVSEL passing through `00000` — worth confirming the arm cannot leak into an unintended transaction. If ever exercised, one write hits all SBX/slope boxes.

---

## S-STRIPE-STRAP — Stripe power-relay RMW depends on STRIPE_CODE straps (medium)

Stripe power/relay controls (48V enable, ground-fault relays, SPI/clock tristate) are single-bit **RMW** into shared registers whose address depends on the board's `STRIPE_CODE` (from `STRIPE_ID` strap pins, `Top.vhd:456`). Write/read of `RELAY_CTL_REG` is self-consistent **only if** the DB's `N` value matches the board's strap (stripe2→N72, stripe3→N80, …). On a **strap/DB mismatch**, the write decodes to `when others => null` (dropped) while the **read returns `0xFFFF`** (read-mux `when others`); the RMW then writes `0xFFFF & andmask | data` into whatever register *does* decode → could turn on/off all channels' 48V or trip ground relays. Verify each stripe's straps vs DB N-values. `PWR_48V_EN = NOT RELAY_CTL_REG(bit)` (inverted), reset `RELAY_CTL_REG=0`.

---

## S-SCAN — `SlopeBoxScan.vhd` has no no-response detection (REAL; explains 4999V/400K, not power fault)

The slope-box ADC scanner shifts in whatever appears on the serial line with no "valid/no-response" detection. An unpowered/disconnected/booting slope box → floating line read as all-ones (0xFFF) → GeHV 4999V; stale PT100 → ~400K. Separate from the power-fault path; documented for completeness and for `collectorBox_redo.md`.

---

## CLEARED (investigated, not the cause of the `PreampI2C_OE_CTL` trigger)

- **PI_TRANSACTOR watchdog** — only re-arbitrates the serial bus on SCK/CE stall (~655 µs); does not cut power/HV/config.
- **DPRAM / mailbox init** — already `0x0000` (BRAM_TDP_MACRO INIT_A/B=0, SRVAL=0, all INIT_xx=0; `LAST_SLOPEBOX_ADC_VAL` inits 0). Refutes earlier `collectorBox_redo.md` claim of 0xFFFF.
- **Slope-box scanner reset via FPGA_CTL_REG** — slope box scanner reset is `PULSED_CONTROL_REG(0)`, not FPGA_CTL_REG; bit 9 resets only preamp/power/dongle scanners.
- **Slope-box command FIFO via addr 1** — addr 1 is device "1110" (register write), not "1101"; does not feed the slope box FIFO.
- **Clock switch via bit 10** — value-gated; RMW preserves bit 10, so no switch.
- **SBX `FPGA_CTL_REG` RMW logical correctness** — addr 1 IS readable + DPRAM-shadowed; RMW self-consistent in steady state (so corruption, if any, is from a bad *read* (S-RMW/S-DEVSEL) or the whole-register re-drive, not from logic).

---

## Corrections to earlier `collectorBox_redo.md` review (verified this dive)
- Mailboxes already init 0x0000 (not 0xFFFF) → redo-doc Problem 2 / Proposal 3 are wrong.
- Scan-valid status ALREADY exists: FPGA reg `SCANNER_GENERAL_STATUS` (addr 38) + EPICS PVs `GS###_*_SCAN_RUNNING/ABORT/RESET`. Gap is only that nothing gates conversion PVs on it → IOC-DB-only fix (SDIS/DISV), no FPGA rebuild.
- 4999V is a floating-line read (no no-response detection in SlopeBoxScan), not a stale-register init bug.
- "Encode I2C_OE in the ROM" is not a table edit — `I2C_STARTUP_ROM` opcodes (00/01/10/11) only touch the I2C cmd FIFO + DPRAM; no register-write opcode.

---

## S-TIMING — Bidirectional Stripe↔SBX serial timing (read-source + read-back reliability) [verified 2026-06-17]

**Where does the RMW read come from? LIVE from the SBX register — no cache.**
- IOC: `CollectorSupport_BO.c` / `_AO.c` RMW issues `Do_SPI1_transaction(1, Bidx, UsrAddr, 0)` = a real SPI1 read; returns `bcm2835_peri_read(txhold)`. `GLBL_CollectorControlVals[]` is debug flags / indirect-data mailboxes only — NOT a register cache for this read.
- SBX: read of addr 1 → `PREFETCH_DATA <= FPGA_CTL_REG` (live register; `SlopeboxInt_TopLevel_RevC.vhd:1971`). The value is transported back through the stripe. Its correctness depends entirely on the **return-path serial timing**.

**Forward path (Pi→SBX) = combinational (robust).** Stripe `sbx_mux` drives `COLL_SDO<=PI_MOSI; COLL_SCK<=PI_SCK; COLL_CE<=FILT_PI_CE` straight through. The SBX `SERIAL_CTL_MACH` runs on its own PLL `CLK_100MHz_buf` and oversamples the forwarded SCK (`BUFSERCOMMCLK`) via a 2-FF edge detector.

**Return path (SBX→Pi) = re-registered on a DIFFERENT async clock (weak).** The SBX asserts read data synchronized to its `CLK_100MHz_buf`; at the stripe, `READ_DATA_MUX` does `MISO_TO_PI <= COLL_SDI(i)` **registered on `CLK_OSC_100MHZ`** (the stripe's own free-running oscillator), then `PI_MISO <= MISO_TO_PI` (`Top.vhd:951`). `COLL_SDI` is asynchronous to `CLK_OSC_100MHZ` and is **registered without a 2-FF synchronizer** → metastability risk + variable 0–1 cycle (~10 ns) latency on every read bit.

**Three asynchronous ~100 MHz domains in series on a read, no shared clock:** (1) Pi SPI SCK, (2) SBX PLL `CLK_100MHz_buf`, (3) stripe oscillator `CLK_OSC_100MHZ`. Each boundary is an async crossing. The Pi latches MISO on the SPI falling edge, but the stripe inserts an extra async re-register with variable latency between SBX assertion and Pi sample → at higher SPI rates the MISO transition can cross the Pi sample point → **intermittent read-back bit errors**.

**Relevance:** This is a plausible structural cause of the **frequently observed preamp read-back errors** (a read-reliability issue), and it is the **same path the RMW read depends on** — so a corrupted RMW read of `FPGA_CTL_REG` (PRIMARY H2) could share this root cause. Writes (combinational forward path) are more reliable than reads, consistent with "reads error, writes land." If the RMW read of `FPGA_CTL_REG` is corrupted here, the robust write-back faithfully writes the corrupted value → flips that one detector's power-converter EN(bit7)/OE(bit15) or I2C-reset bits → single-detector latched power fault (fits scope/profile). So the "frequent preamp read errors" and the power-fault may **not be fully independent**.

**Limits:** exact failure depends on SPI rate, cable length, and place&route delays — not determinable from RTL alone. Confirm with a scope: PI_SCK vs PI_MISO setup/hold at the Pi, and COLL_SDI vs CLK_OSC_100MHZ at the stripe. The stripe re-register may be intentional retiming, but the missing 2-FF synchronizer on async COLL_SDI is a real CDC gap.
