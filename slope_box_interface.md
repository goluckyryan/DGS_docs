# Slope Box Interface — Knowledge Base

Stability: C3 - Structural / stable

**Source:** `DGS_SVN/dgs/SlopeBoxInterface/`, `DGS_SVN/dgs/SlopeBoxExtension/`, `DGS_SVN/dgs/slopebox_scripts/`
**Date documented:** 2026-04-21

---

## Overview

The Slope Box is a Gammasphere-era module that sets the gain (slope) and offset of each detector channel. In the original GS (pre-DGS), it was controlled via a serial bus from a PC running BASIC programs (see `GSBGOSBT.BAS`, 1997, by D. Landis). In the DGS system, control was moved to a Raspberry Pi with an EPICS IOC communicating over SPI/GPIO to a **Pickoff Card** FPGA.

The **Slope Box Extension** (`DGS_SVN/dgs/SlopeBoxExtension/`) is a hardware add-on that routes the Pi's SPI lines to the physical slope box modules via a pickoff card.

---

## Hardware Signal Chain

```
Raspberry Pi  ──SPI/GPIO──►  Pickoff Card FPGA  ──serial bus──►  Slope Box Module(s)
                                  (address/route)                  (Ge/BGO gain, offset)
```

- GPIO bits are asserted *before* each SPI transfer to select the target detector/slot
- The FPGA routes the SPI transaction to the correct slope box based on GPIO (collector box mode) or direct (local mode, GPIO=0)
- SPI protocol: simultaneous MOSI out + MISO in (true SPI, not asyn-compatible — see design note below)

---

## SPI Transaction Format (Pickoff Card)

Total transaction: **24 bits** ✅ verified 2026-04-21 — `PickoffSupport.c:L41-47`

| Bits | Meaning |
|------|---------|
| bit 23 | R/W flag: 0 = write, 1 = read |
| bits 22–16 | 7-bit register address |
| bits 15–0 | 16-bit data payload |

Timing rules:
- Pi drives MOSI so FPGA clocks data on **rising** edges of SCLK (including the first edge)
- Pi samples MISO on **falling** edges of SCLK

---

## EPICS IOC Design (PickoffApp)

**Location:** `DGS_SVN/dgs/SlopeBoxInterface/RaspberryPi/LocalEpics/PickoffApp/`

### Key design choice: CAMAC_IO link type

Because asyn only supports byte-granularity serial I/O and cannot do simultaneous MOSI/MISO, the custom EPICS device support uses the `CAMAC_IO` link structure instead. The `camacio` fields are repurposed:

| camacio field | DGS meaning |
|---------------|-------------|
| `b` | GPIO value → which detector (collector routing) |
| `c` | Transaction length in bits (typically 24) |
| `n` | 7-bit register address; upper byte non-zero → R/W flag set (read-modify-write) |
| `a` | AND mask (for read-modify-write) |
| `f` | OR mask (for read-modify-write) |
| `parm` | Unused (pass `@`) |

**OUT field example** (GeCenterGain PV, local mode):
```
OUT("#0 24 20 0 0 @")
  b=0 (local, no GPIO), c=24 bits, n=20 (register addr), a=0, f=0
```

### PV record type

All PVs are `ao` (analog output). The `write_ao` function performs the SPI transaction. For read-modify-write: read current MISO value, AND with mask, OR with new PV value, write back.

### Device support registration (`.dbd`)

```
device(ao,CAMAC_IO,devAoPickoff,"PickoffLocalSerial")
```

The `DTYP` field in each PV must be `"PickoffLocalSerial"` to activate this driver.

### Source files

| File | Purpose |
|------|---------|
| `PickoffSupport.c` (676 lines) | Full device support: init + write for ao records; extensive EPICS architecture notes by J. Anderson |
| `PickoffSupport.h` | Prototypes only (stub) |
| `PickoffSupport.dbd` | DBD device() declaration |
| `Db/user.substitutions` | Example substitutions (not production) |

**Note:** The `write_ao` function in the current SVN snapshot is a stub (`return 0;`) ✅ verified 2026-04-21 — `PickoffSupport.c:L617-619` — the actual SPI transaction logic was either in progress or kept elsewhere. `init_ao` (not `init_record_ao`) prints `"initializing ao\n"` on IOC init ✅ verified 2026-04-21 — `PickoffSupport.c:L622-624`.

---

## Design Note: Why Not Asyn?

The comment block at the top of `PickoffSupport.c` (by J. Anderson) is very explicit:

> "Asyn assumes that all serial communication is always a byte, and the hardware/firmware implementation of the interfaces within the pickoff card and the collector box do not restrict themselves in that way. The length of the transactions to/from the FPGAs will certainly be far longer than a byte, and there is ABSOLUTELY NO GUARANTEE THAT ANY FIELD OR TRANSACTION WILL BE AN EXACT MULTIPLE OF EIGHT BITS LONG."

> "Additionally, true SPI says that the initiator of the transaction shall simultaneously clock data out on MOSI and clock data in on MISO. Asyn does not support this."

This is why a fully custom EPICS device support layer was written from scratch.

---

## BGO Counter Monitoring (slopebox_scripts)

**Location:** `DGS_SVN/dgs/slopebox_scripts/`

Scripts for reading BGO scintillator counter rates via EPICS PVs:

- `Avg_all_BGO_count` — reads 8 PVs (`GS000_BGO1_counter` through `GS000_BGOSum_counter`), averaging each N times ✅ verified 2026-04-22 — `slopebox_scripts/Avg_all_BGO_count:L3-10` (8 caget_avg calls: BGO1–7 + BGOSum)
- `caget_avg` — bash script: calls `caget` N times, strips PV name (first 25 chars), extracts numeric value, computes average ✅ verified 2026-04-22 — `slopebox_scripts/caget_avg:L11-13` (`NEW_VAL2=${NEW_VAL:25}`, `tr -dc '0-9'`, accumulate)
- `BGO_counter_sweep.ods` — ODS spreadsheet (analysis)
- `BGO_Sweep_test` — BGO sweep test script

BGO PV naming pattern: `GS000_BGO<N>_counter` (N=1–7, plus BGOSum) ✅ verified 2026-04-22 — `slopebox_scripts/Avg_all_BGO_count:L3-10`

---

## Slope Box Extension Hardware

**Location:** `DGS_SVN/dgs/SlopeBoxExtension/`

Subdirectories:
- `PickoffCard/` — PCB layout, schematic, fabrication outputs (Rev A/B/C)
- `DVIBreakout/`, `I2C_Tools/`, `PowerConverter/`, `GS_ID/`
- `RaspberryPi/` — Pi interface code and docs
- `ProjectTracking/` — project management docs
- `Schematic/` — schematic files

Key design docs (in `SlopeBoxInterface/Documentation/`):
- `DesignReviewofSlopeBoxInterface.docx`
- `DesignReviewofPickoffInterface.docx`
- `GEOnly_Slope box disassembly and modification.docx`
- `pickoff_signal_flow.png` — signal flow diagram

---

## Original Gammasphere (Pre-DGS) Control

`GSBGOSBT.BAS` (1997, D. Landis) — IBM BASIC program for Gammasphere BGO + voltage regulator control via parallel port at base address `&H304`. Supported:
- Status readback (temperature alarm, BGO HV on/off, Ge HV on/off)
- ADC readout
- Gain/offset setting via serial bus
- Serial bus uses "SB" (strobe bit) synchronization at port `BA+2`

Historical reference only — not used in DGS.

`GSMODEXE.BAS`, `GSMODSER.BAS` — companion BASIC programs (module execution / serial protocol).

---

## Current Status (as of 2026-04-23)

- The `PickoffApp` EPICS IOC in SVN appears to be a **prototype/development version** — its local `PickoffSupport.c` has a stub `write_ao` (`return 0;`) ✅ verified 2026-04-23 — `DGS_SVN/dgs/SlopeBoxInterface/RaspberryPi/LocalEpics/PickoffApp/src/PickoffSupport.c:L617-L619`
- A working `write_ao` implementation does exist in the active `collectorboxpi` repo as `CollectorBox_RevA/CollectorApp/src/PickoffSupportBackup.c`; it parses `CAMAC_IO` fields, calls `Do_SPI1_transaction(RWFlag, UsrAddr, UsrData)`, and stores the low 16 bits of the reply into `rbv` ✅ verified 2026-04-23 — `collectorboxpi/CollectorBox_RevA/CollectorApp/src/PickoffSupportBackup.c:L923-L986`
- It is not clear if this IOC is currently deployed on any Pi or if it has been superseded
- The slope box hardware itself may still be in use for BGO gain/offset; check with Ryan
- The `collectorboxpi/` repo (active) handles the Collector Box Pi — separate from the Slope Box Pi

---

*Documented from SVN snapshot — verify deployment status with Ryan*

---

## See Also

- `knowledgeBase/deep_fpga_SBX_CtrlFPGA.md` — Deep VHDL analysis of SBX Motherboard Control FPGA (Spartan-6 XC6SLX9, entity SlopeBoxInt): SPI protocol, register map, I2C scanner machines, BGO disc/DDR outputs, slope box serial, preamp reset clamp, firmware version registers
- `knowledgeBase/sbx.md` — SBX hardware overview: slope box FPGA, HV generation, BGO pattern/sum, GS_ID dongle, Pi IOC
- `knowledgeBase/sbxPi_ioc.md` — SBX Pi standalone IOC (PickoffApp_RevC): active SPI1 device support, CAMAC_IO link, global mailboxes
- `knowledgeBase/collectorboxpi.md` — Collector box Raspberry Pi soft IOC (active); uses the same `PickoffSupportBackup.c` device support traced here
- `knowledgeBase/collectorbox_devicesupport.md` — EPICS device support internals: SPI driver, CAMAC_IO link type, conversion coefficients
- `knowledgeBase/EPICS_asyn.md` — asyn driver support (the alternative framework that was evaluated and rejected for SBX)
- `knowledgeBase/DGS_SVN.md` — SVN archive inventory; `SlopeBoxInterface/` and `SlopeBoxExtension/` entry
