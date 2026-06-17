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

### Problem 2: Mailboxes hold stale/uninitialized values at startup

ADC and I2C mailboxes reset to 0x0000 or 0xFFFF. EPICS PV chains fire immediately after IOC start, before the first scan completes. First value read is garbage.

**Symptom:**
- `Conv_GeHV` reads **4999V** — ADC mailbox = 0xFFF (max 12-bit) → 4095 × (5000/4096) = 4998.8V
- `MOD###_DV_TEMP` reads **>400K** — PT100 mailbox stale/zero → CVD poly returns ~127°C → +273 = 400K

**Current workaround:** postscript checks for these sentinel values and retries. `single_detector.sh` retries up to 3× on 4999V detection.

### Problem 3: No "scan complete" or "data valid" signal

No status bit indicates whether the ADC/I2C scanner has completed its first pass. EPICS has no way to know if a mailbox value is fresh or stale.

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

Add ROM opcodes in `I2C_STARTUP_ROM` to assert and deassert `PREAMP_I2C_OE` as part of the scan sequence itself. The ROM already supports delay and write opcodes — a write to the FPGA_CTL_REG bit 8 can be encoded as a ROM entry.

**Result:** I2C_OE is managed entirely by the FPGA scan sequence. It is never left on accidentally. The Pi/postscript does not need to touch it.

**Files to modify:**
- `SlopeBoxInt_TopLevel_RevC.vhd` — add internal path for ROM to write FPGA_CTL_REG(8)
- `I2C_STARTUP_ROM.vhd` (in `PickoffCard_SBX_Extension/Revision_B/Source/`) — add OE-assert and OE-deassert entries to the ROM sequence

### 2. "Scan valid" status register bit

Add a single status bit (e.g. in FPGA_STATUS or a new register) that goes HIGH after the first complete pass of both ADC and I2C scanners, and resets LOW when a new scan cycle starts.

**Result:** EPICS can poll this bit before enabling PV processing. Eliminates the 4999V and 400K transients entirely at the IOC level. No postscript sleep/check needed.

**Files to modify:**
- `SlopeBoxInt_TopLevel_RevC.vhd` — add `FIRST_SCAN_COMPLETE` signal, set on scanner done, clear on scanner reset
- `CtrlFPGA.db` / `Pickoff.db` — add a BI PV for this bit; use it to gate dependent PV processing

### 3. Mailbox initialization to safe sentinel values

On FPGA reset, initialize all ADC and I2C mailboxes to 0x0000 instead of 0xFFFF.

**Result:** A missed or incomplete scan reads 0V / 0K — obviously wrong but does not trigger false alarms or false interlocks.

**Magnitude:** Small — just change DPRAM initialization in the VHDL.

### 4. Autonomous I2C scanner retry on ACK error

When the I2C scanner detects an ACK error (`ABORT_FLAG`), automatically re-assert scanner reset and restart the ROM from address 0, up to N retries before giving up and setting an error flag.

**Result:** Transient I2C glitches (e.g. power-on noise, slow preamp startup) self-recover without postscript intervention.

**Files to modify:**
- `SlopeBoxInt_TopLevel_RevC.vhd` — add retry counter logic around `PREAMP_SCANNER_RESET`

### 5. Consider: persistent scan mode (continuous background I2C polling)

Rather than a one-shot ROM scan triggered at startup, run the I2C scanner continuously in the background (with appropriate inter-scan delay). I2C_OE would be pulsed on/off per scan cycle by the FPGA itself.

**Trade-off:** Requires careful timing to avoid DAQ interference. May not be appropriate if the preamp SDA/SCL lines are shared with DAQ signals during normal operation. Needs hardware-level verification before implementing.

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
