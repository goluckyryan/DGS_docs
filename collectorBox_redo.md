# Collector Box System — Redesign Considerations

**Status:** DRAFT — analysis and proposals only. Not yet implemented.
**Date:** 2026-06-17
**Author:** General DGS (analysis session with Ryan Tang)

---

## Stability: TOC

- [Current Architecture Summary](#current-architecture-summary)
- [What the System is Trying to Achieve](#what-the-system-is-trying-to-achieve)
- [Sources of Fragility](#sources-of-fragility)
- [Proposed FPGA Improvements](#proposed-fpga-improvements)
- [Proposed IOC / Software Improvements](#proposed-ioc--software-improvements)
- [Goal: What the Postscript Should Shrink To](#goal-what-the-postscript-should-shrink-to)

---

## Current Architecture Summary

The Collector Box system consists of:

- **CtrlFPGA** (`SlopeBoxInt_TopLevel_RevC.vhd`) — per-slope-box FPGA on the motherboard. Handles SPI to Pi, I2C to preamp/power board/dongle, ADC scanning, BGO pattern, HV interlock logic.
- **Stripe FPGA** (`ControlStripe/Source/Top.vhd`) — one per stripe (6 total). Handles clock/sync fanout, relay control, DVI power enable, ground fault detection, LED I2C (PCAL6416A).
- **Collector Pi** (one per box) — runs EPICS softIOC via `CollectorBox_RevA`. Communicates with FPGAs over SPI (bcm2835). Hosts all PV conversion logic.
- **`softioc_postscript.sh`** — bash script that runs after IOC startup to enable I2C, trigger scans, calibrate PT100, set HV ramp parameters, check interlocks, and turn HV on.
- **`single_detector.sh`** (on spark `~/GS_model/`) — same flow as postscript but for a single detector, called from the web interface.

Key source locations:
- CtrlFPGA full source: `FPGA/collectorBox/CollectorBox_CtrlFPGA/Source/SlopeBoxInt_TopLevel_RevC.vhd`
- ControlStripe full source: `FPGA/collectorBox/CollectorBox_StripeFPGA/Source/Top.vhd`
- Pi IOC: `collectorboxpi/CollectorBox_RevA/`
- Conversion logic: `collectorboxpi/CollectorBox_RevA/CollectorApp/src/CollectorSupport.c`

---

## What the System is Trying to Achieve

1. **Power management** — relay on/off per slot, 48V DVI power, earth relay, ground fault detection
2. **Clock/sync distribution** — deliver trigger clock and sync to each slope box
3. **SPI communication** — bidirectional control to each slope box FPGA
4. **Temperature monitoring** — PT100 (via I2C through preamp) and PT500 for cold detector temp
5. **Voltage monitoring** — Ge HV readback, BGO HV, ±12V, 24V, 5V supply rails
6. **HV interlocking** — cut Ge HV if temp too high, cut BGO HV if interlock open
7. **PT100 calibration** — gain/offset correction using PT500 as ground truth
8. **Startup sequencing** — bring everything up in safe order

---

## Sources of Fragility

### Problem 1: I2C is a shared stateful bus with externally-managed enable

`PREAMP_I2C_OE` (FPGA_CTL_REG bit 8) must be driven high by the Pi before any preamp I2C reads, and explicitly driven low afterward. The FPGA scanner does not manage this itself.

**Symptom:** If the postscript fails, is skipped, or runs before the FPGA is ready, I2C_OE stays in an unknown state. I2C left on in normal DAQ operation can conflict with the preamp signal path.

**Current workaround:** postscript explicitly enables, reads, disables. `single_detector.sh` also explicitly disables at end. Postscript now also disables at end (added 2026-06-17).

### Problem 2: Stale/garbage values read before the first scan completes

> ⚠️ CORRECTION 2026-06-17 (verified): The original claim here — "mailboxes reset to 0x0000 or 0xFFFF" — is **wrong**. The SBX DPRAM and the readback registers initialize to **0x0000**, not 0xFFFF. Verified: `BRAM_TDP_MACRO` in `SlopeboxInt_TopLevel_RevC.vhd` has `INIT_A/B=0`, `SRVAL_A/B=0`, and every `INIT_xx` row zero; `LAST_SLOPEBOX_ADC_VAL` initializes to `X"0000"` (`:557`). So the 4999V is **not** an uninitialized-mailbox/0xFFFF artifact.

The real mechanism: EPICS PV chains can fire before the slope-box scanner has read fresh data, and the slope-box ADC read path (`SlopeBoxScan.vhd`) has **no no-response / data-valid detection** — it shifts in whatever is on the serial line. If the slope box is unpowered / disconnected / still booting, the line floats and reads as all-ones (0xFFF).

**Symptom:**
- `Conv_GeHV` reads **4999V** — a real read of a **floating serial line** returns 0xFFF → 4095 × (5000/4096) = 4998.8V (NOT a stale 0xFFFF register).
- `MOD###_DV_TEMP` reads **>400K** — PT100 path returns garbage/zero before valid I2C data → CVD poly → ~127°C → +273 = ~400K.

**Current workaround:** postscript checks for these sentinel values and retries. `single_detector.sh` retries up to 3× on 4999V detection.

### Problem 3: Nothing GATES the conversion PVs on the existing scan-status bits

> ⚠️ CORRECTION 2026-06-17 (verified): The original claim — "No status bit indicates whether the scanner completed" — is **wrong**. A scan-status register **already exists**: `SCANNER_GENERAL_STATUS` at SBX address 38 exposes per-scanner RUNNING / ABORT / RESET bits (preamp, power, dongle), and the IOC already surfaces them as PVs `GS###_PREAMP_SCAN_RUNNING/ABORT/RESET` (+ POWER/DONGLE) in `Pickoff.db`. Verified: `SlopeboxInt_TopLevel_RevC.vhd` `SCANNER_GENERAL_STATUS_ADDR` + read mux; `Pickoff.db` bit records.

The actual gap is that **nothing uses these existing PVs to gate the conversion records** (`Conv_GeHV`, `MOD###_DV_TEMP`, etc.). The fix is therefore an **IOC-DB-only change** (add `SDIS`/`DISV` links to the scan-status PV, or a derived "scan complete & not aborted" calc) — **no FPGA rebuild required**. (Note: the slope-box scanner itself is separate from the preamp/power/dongle scanners; a true "slope box data valid" flag for the floating-line case in Problem 2 would still need an RTL change in `SlopeBoxScan.vhd`.)

### Problem 4: PT100 calibration is a one-shot bash computation at startup

The PT100 gain correction (using PT500 as reference) is computed once in bash, written to `MOD###_DV_TEMP.CALC`, and never updated unless the postscript re-runs. If the PT500 reading is unsettled at the moment calibration runs, the baked-in gain is wrong for the entire session.

**Current workaround:** `single_detector.sh` has a settle-wait loop (up to 40s) before calibrating. Postscript does not.

### Problem 5: HV interlock enable lives only in the postscript

The FPGA has hardware interlock bits (`SlopeBoxTempHigh`, `SlopeBoxBGOInterlock`) but the actual HV ON command is issued only by the postscript. If the postscript is skipped or fails, HV never turns on.

### Problem 6: I2C ACK errors do not self-recover

The scanner has an `ABORT_FLAG` output and error detection, but on failure it stops. Recovery requires the postscript to issue a reset and re-run. There is no autonomous retry.

---

## Proposed FPGA Improvements

### 1. Scanner-controlled I2C_OE (highest priority)

> ⚠️ CORRECTION 2026-06-17 (verified): The claim "the ROM already supports... a write to FPGA_CTL_REG bit 8 can be encoded as a ROM entry" is **wrong**. The `I2C_STARTUP_ROM` opcodes are only: 00 = write I2C command FIFO, 01 = delay, 10 = GO (poll I2C machine), 11 = GO + copy read data to DPRAM. **There is no opcode that writes FPGA_CTL_REG or any control register.** Encoding an OE toggle in the ROM therefore requires NEW opcode + datapath logic in both the ROM player and the top level — it is real RTL work, not a ROM-table edit. Verified: `I2C_STARTUP_ROM.vhd` opcode decode (CHECK_OPCODE state).

Intent: have the FPGA manage `PREAMP_I2C_OE` itself so the Pi/postscript never leaves it in a bad state. Feasible, but scope it as RTL (new control-write opcode/path), not a table entry.

**Files to modify:**
- `SlopeboxInt_TopLevel_RevC.vhd` — add internal path for the scanner/ROM to drive FPGA_CTL_REG(8)
- `I2C_STARTUP_ROM.vhd` — requires a new opcode + datapath to write a control register (not currently supported)

### 2. Gate conversion PVs on the EXISTING scan-status bits (IOC-only; no FPGA change)

> ⚠️ CORRECTION 2026-06-17 (verified): The original proposal to "add a scan-valid status bit to the FPGA" is largely **unnecessary** — the bits already exist. `SCANNER_GENERAL_STATUS` (SBX address 38) exposes per-scanner RUNNING/ABORT/RESET (preamp/power/dongle), already surfaced as PVs `GS###_*_SCAN_RUNNING/ABORT/RESET` in `Pickoff.db`.

**Do this instead:** add `SDIS`/`DISV` (or a derived "scan complete & not aborted" calc) to `Conv_GeHV` / `MOD###_DV_TEMP` / other conversion records, pointing at the existing scan-status PVs. **IOC-DB-only change, no FPGA rebuild.**

**Caveat:** the slope-box scanner is separate from the preamp/power/dongle scanners. The floating-line 4999V case (Problem 2) is a *slope-box* read; fully gating that would still need a slope-box data-valid flag (see Improvement 6).

### 3. ~~Mailbox initialization to 0x0000~~ — NOT NEEDED (already 0x0000)

> ⚠️ CORRECTION 2026-06-17 (verified): This proposal is moot — the DPRAM and readback registers **already initialize to 0x0000** (`BRAM_TDP_MACRO INIT_A/B=0`, `SRVAL=0`, all `INIT_xx=0`; `LAST_SLOPEBOX_ADC_VAL` inits `X"0000"`). The 4999V is a floating-line read (Problem 2), not a 0xFFFF init artifact. No change required. The real RTL fix for the floating-line case is no-response detection in `SlopeBoxScan.vhd` (see Improvement 6).

### 4. Autonomous I2C scanner retry on ACK error

When the I2C scanner detects an ACK error (`ABORT_FLAG`), automatically re-assert scanner reset and restart the ROM from address 0, up to N retries before giving up and setting an error flag.

**Result:** Transient I2C glitches (e.g. power-on noise, slow preamp startup) self-recover without postscript intervention.

**Files to modify:**
- `SlopeBoxInt_TopLevel_RevC.vhd` — add retry counter logic around `PREAMP_SCANNER_RESET`

### 5. Consider: persistent scan mode (continuous background I2C polling)

Rather than a one-shot ROM scan triggered at startup, run the I2C scanner continuously in the background (with appropriate inter-scan delay). I2C_OE would be pulsed on/off per scan cycle by the FPGA itself.

**Trade-off:** Requires careful timing to avoid DAQ interference. May not be appropriate if the preamp SDA/SCL lines are shared with DAQ signals during normal operation. Needs hardware-level verification before implementing.

### 6. Slope-box read no-response / data-valid detection (fixes the real 4999V cause)

`SlopeBoxScan.vhd` shifts in whatever appears on the slope-box serial line with **no detection of a non-responding slope box** — an unpowered/disconnected/booting slope box floats the line, read as all-ones (0xFFF) → the 4999V GeHV and ~400K temp transients. Add a per-read validity check (e.g. detect all-ones / no edges / no expected framing) that sets an "invalid" flag, surfaced as a PV the conversion records can gate on. This is the RTL fix referenced by Problem 2 / Improvement 2's caveat.

**Files to modify:**
- `SlopeBoxScan.vhd` — add no-response/all-ones detection → per-channel valid flag
- `Pickoff.db` — expose the valid flag; gate `Conv_GeHV` / `MOD###_DV_TEMP` on it

---

## Proposed IOC / Software Improvements

### 1. Gate PV processing on "scan valid" bit

Add a `DISV` or `SDIS` link on the `Conv_GeHV`, `MOD###_DV_TEMP`, and other conversion PVs pointing to the scan-valid status PV. EPICS will not process them until the FPGA signals data is ready.

### 2. PT100 calibration as continuous EPICS calc, not one-shot bash

Replace the one-shot bash calibration with a `calcout` that continuously computes the PT100 gain from live PT500 and PT100 readings, with a deadband to avoid constant CALC field updates. This way calibration self-corrects over time rather than being frozen at startup.

### 3. HV interlock logic in IOC DB

Move the HV ON/OFF interlock decision from the postscript into the EPICS DB:
- A `calcout` monitoring `SlopeBoxTempHigh` and `SlopeBoxBGOInterlock` drives `GE_HV_CTRL` and `BGO_HV_CTRL` autonomously.
- Postscript only sets ramp parameters (step size, hysteresis, demand voltage) — it does not need to command HV ON.

---

## Goal: What the Postscript Should Shrink To

After these improvements, `softioc_postscript.sh` should only need to:

1. Enable relay + SPI/clock/sync per slot (still needs physical sequencing)
2. Set HV ramp parameters (step size, hysteresis, demand voltage) from the detector database
3. Optionally trigger one explicit calibration update
4. Report summary

Everything else — I2C enable/disable, scan sequencing, mailbox validity, interlock logic, HV on/off — should be self-managed by the FPGA and IOC DB.
