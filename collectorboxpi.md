# collectorboxpi — EPICS Soft IOC for Collector Box (Raspberry Pi)

Stability: C2 - Active / semi-stable

> 🔗 **Related:** `sbx.md` — SBX (Slope Box Extension) hardware that this IOC controls | `collectorbox_PVs.md` — full PV list | `gammasphere_geometry.md` — GS hole numbering

## Table of Contents

1. [What It Is](#what-it-is)
2. [Key Hardware](#key-hardware)
3. [Repository Layout](#repository-layout)
4. [Key Files](#key-files)
   - [GenerateCmdFile.py Inputs](#generatecmdfilepy-inputs)
   - [special_detectors.txt — Per-Detector Overrides and DISPLACED Cables](#special_detectorstxt--per-detector-overrides-and-displaced-cables)
5. [PVs — High Voltage Control (per active detector)](#pvs--high-voltage-control-per-active-detector)
6. [Build Order](#build-order)
7. [Generating Startup Files](#generating-startup-files)
   - [Per-Detector Macro Assignments](#per-detector-macro-assignments)
   - [Geometry Helpers](#geometry-helpers)
   - [Autosave .req File Generation](#autosave-req-file-generation)
8. [Running the IOC](#running-the-ioc)
9. [softioc_postscript.sh — Post-Init Hardware Commissioning](#softioc_postscriptsh--post-init-hardware-commissioning)
   - [Phase 1 — Relay & SPI Enable](#phase-1--relay--spi-enable-loop-over-65-stripeport-grid)
   - [Phase 2 — I2C & Preamp Enable](#phase-2--i2c--preamp-enable-loop-over-detector-list)
   - [Phase 3 — BGO HV Interlock](#phase-3--bgo-hv-interlock)
   - [Phase 4 — HPGe HV Interlock & Ramp](#phase-4--hpge-hv-interlock--ramp)
   - [Phase 5 — PT100 Temperature Calibration](#phase-5--pt100-temperature-calibration)
   - [Phase 6 — Health Check & Error Report](#phase-6--health-check--error-report)
10. [Networking / PXE Boot](#networking--pxe-boot)
11. [Discord Integration](#discord-integration)
12. [EPICS Port Convention (inferred)](#epics-port-convention-inferred)
13. [SPI Hardware Communication Layer](#spi-hardware-communication-layer)
    - [Protocol](#protocol)
    - [DEVSEL Bus (Device Selection)](#devsel-bus-device-selection)
    - [ADC Scanner](#adc-scanner)
    - [Key Functions (spi.c)](#key-functions-spic)
    - [Global Data Structure (CollectorSupport.h)](#global-data-structure-collectorsupporth)
    - [I2C Control Flags (CollectorSupport.h)](#i2c-control-flags-collectorsupporth)
14. [Collector Box → GS Hole Assignments](#collector-box--gs-hole-assignments)
15. [db/ Template Files (18 templates)](#db-template-files-18-templates)
16. [Commissioning Workflow — Adding / Removing Detectors](#commissioning-workflow--adding--removing-detectors)
17. [Connections to Other Subsystems](#connections-to-other-subsystems)
18. [Cross-References](#cross-references)

## What It Is

An **EPICS soft IOC** running on Raspberry Pi (aarch64 / Debian 13) that controls and monitors **Collector Box** hardware (CollectorBox_RevA). All 4 Pi collector boxes now run this repo (as of commit 2309422, 2026-04-13 — "all pi changed to NFS server").

Compiled against **EPICS 7.0.10** (patch level 1). Self-contained repo: includes `epics-base` and `autosave` as git submodules. ✅ verified 2026-04-06 — collectorboxpi/epics-base/configure/CONFIG_BASE_VERSION; autosave tag R5-11 ✅ verified 2026-04-06 — collectorboxpi/autosave/documentation/RELEASE_NOTES

---

## Key Hardware

- **Hardware:** CollectorBox_RevA boards
- **Platform:** Raspberry Pi (aarch64), Debian 13
- **PXE Boot:** Pi boots over network from `fs2.onenet` (192.168.203.71) via DHCP/tftp on Einstor (192.168.203.1) ✅ verified 2026-04-06 — ping fs2.onenet resolves 192.168.203.71 live
- **Collector Pi IDs:** pi0, pi1, pi2, pi3 (identified by MAC address)
- **MAC → hostname mapping (production — `rc.local` active lines):**
  - `b8:27:eb:fc:97:08` → pi0 ✅ verified 2026-04-08 — `piserver/os/Debian13/etc/rc.local:L5` (NFS)
  - `b8:27:eb:57:19:db` → pi1 ✅ verified 2026-04-08 — `piserver/os/Debian13/etc/rc.local:L6` (NFS)
  - `b8:27:eb:5a:d0:8e` → pi2 ✅ verified 2026-04-08 — `piserver/os/Debian13/etc/rc.local:L7` (NFS)
  - `b8:27:eb:99:18:3f` → pi3 ✅ verified 2026-04-08 — `piserver/os/Debian13/etc/rc.local:L8` (NFS)
- **Testing/spare Pi MACs** (also in rc.local):
  - `b8:27:eb:39:f2:ce` → pi0 (spare, commented out)
  - `b8:27:eb:df:8c:d6` → pi1 (testing Pi — **active, not commented out**) ✅ verified 2026-04-08 — `rc.local:L11`
  - `b8:27:eb:91:bd:1b` → pi2 (commented out)
  - ~~**Note:** pi1/pi2/pi3 collector boxes are **not yet implemented** (hardware not deployed).~~ **CORRECTED 2026-04-23:** All 4 pi collector boxes (pi0–pi3) are deployed and running as of commit 2309422 (2026-04-13, "all pi changed to NFS server"). `st_202.cmd`, `st_203.cmd`, and `st_204.cmd` all have full `DetNbr` records (196, 229, 175 entries respectively). Commit was authored from `pi3-0`, confirming pi3 was live. ✅ verified 2026-04-23 — commit 2309422, `st_202.cmd` (196 DbLoadRecords), `st_203.cmd` (229), `st_204.cmd` (175)
  - These 3 also have tftpboot symlink dirs on piserver (→ debian13Boot)

---

## Repository Layout

```
/shared/EPICS/              ← mounted from fs2.onenet:/mnt/vol1/.../shared
├── epics-base/             ← EPICS base 7.0.10 (git submodule)
├── autosave/               ← autosave R5-11 (git submodule)
├── CollectorBox_RevA/      ← IOC application
│   ├── CollectorApp/       ← device support C source + .db templates
│   ├── configure/          ← RELEASE, CONFIG_SITE
│   ├── db/                 ← per-detector .db template files
│   ├── iocBoot/iocCollectorApp/
│   │   ├── st_201.cmd      ← generated (do NOT edit by hand)
│   │   ├── st_202.cmd
│   │   ├── st_203.cmd
│   │   ├── st_204.cmd
│   │   └── softIOC_<N>_settings.req  ← generated autosave request
│   └── GenerateCmdFile.py  ← generates st_<N>.cmd and .req files
├── Pre_EPICS_Collector/    ← hardware scan outputs for GenerateCmdFile.py
├── softIOC.service         ← systemd service unit
├── startSoftIOC.sh         ← service entry point
├── collectorBox.sh         ← sets THIS_PI_COLL_NUM and env vars
├── EPICS_env.sh            ← sets PATH / LD_LIBRARY_PATH for EPICS tools
└── softioc_postscript.sh   ← optional post-init PV setup
```

---

## Key Files

| File | Role |
|------|------|
| `GenerateCmdFile.py` | Generates `st_<N>.cmd` and `softIOC_<N>_settings.req` from hardware scan data |
| `collectorBox.sh` | Sets `THIS_PI_COLL_NUM`; must be sourced before generating configs |
| `softIOC.service` | systemd service — runs IOC via `procServ` (telnet-accessible) |
| `startSoftIOC.sh` | Service entry point |
| `softioc_postscript.sh` | Optional post-init PV setup; logs to `~/softioc_<N>_postScript_log.txt` |

### GenerateCmdFile.py Inputs

| File | Purpose |
|------|---------|
| `Pre_EPICS_Collector/SCAN_OUTPUT_3_COMM_<N>.txt` | Cable/detector hardware scan results |
| `mca_data_<N>.txt` | MCA resolution and reset period per detector |
| `special_detectors.txt` | Per-detector overrides + displaced cable re-assignments (see below) |

### special_detectors.txt — Per-Detector Overrides and DISPLACED Cables
_Source: `collectorboxpi/README.md:L120-185`, commits 6667ec3, 874cd1b, 092a3cc (2026-04-15/16)_

Read by `GenerateCmdFile.py`. Supports two kinds of entries:

**1. Per-detector macro overrides** — override any macro used in `dbLoadRecords` for a specific detector:
```
<detector_number> , <macro_name> , <value>
```
Example: GS048 uses a PT500 thermometer instead of the standard Pt1000:
```
048 , DV_TEMP_INPA , GS048_Calc_PT500_Temp
048 , DV_TEMP_PREC , 2
```

**2. DISPLACED** — cross-box cable re-assignments. When a GS detector hole is physically re-cabled from one collector box to another (hardware fault, geometry change), add a `DISPLACED` entry for the **original** box:
```
<detector_number> , DISPLACED , <box_number>
```
Example — GS070 re-cabled from box 202 to box 201:
```
070 , DISPLACED , 202
```

| PV | Behaviour on the listed (original) box |
|----|---------------------------------------|
| Active cable records (SlopeBox, Pickoff, etc.) | **Skipped** — no records loaded |
| `True_GS{X}_to_VME_GS` (via `unused_gs.db`) | **Omitted entirely** — receiving box emits the correct mapping via its active `SlopeBox.db` |
| `VME_GS{X}_to_True_GS` (via `unused_dvi.db`) | **`VAL=000`** if the VME slot is empty; normal active-cable value if another detector now occupies it |

The **receiving box** needs no entry — when its scan file reports an active cable with `DNG_ID=X`, `SlopeBox.db` is loaded and automatically emits both `VME_GS{VMEGS}_to_True_GS = X` and `True_GS{X}_to_VME_GS = VMEGS`. Chain displacements are handled naturally — each displaced GS needs only one `DISPLACED` line on its original box.

> ⚠️ **Old keyword `DISABLE`** is still accepted with a deprecation warning. It only skipped active cable records but did NOT suppress `True_GS{X}_to_VME_GS` on the original box, causing duplicate/conflicting PVs with the receiving box. `DISPLACED` fixes both issues. ✅ verified 2026-04-16 — commits 874cd1b + 092a3cc (`collectorboxpi/README.md:L120-185`, `GenerateCmdFile.py` DISPLACED logic)

---

## PVs — High Voltage Control (per active detector)

| PV Pattern | Description |
|------------|-------------|
| `GS<NNN>_GE_HV_DEMAND_VOLTS` | Operator HV demand (volts) |
| `GS<NNN>_GE_HV_STEP_SIZE` | Max HV step per cycle |
| `GS<NNN>_GE_HV_HYSTERESIS` | HV hysteresis |
| `GS<NNN>_GE_HV_ABSMAX` | Absolute HV ceiling |
| `MOD<NNN>_DS_GEHV` | Desired HV spec from detector database |

Hardware register PVs (`CollectorLocalSerial`) are **excluded** from autosave — stale values are never written to hardware on startup.

---

## Build Order

```sh
cd /shared/EPICS

# 1. EPICS base
cd epics-base && make -j$(nproc) && cd ..

# 2. autosave
cd autosave && make -j$(nproc) && cd ..

# 3. CollectorBox IOC
cd CollectorBox_RevA && make -j$(nproc) && cd ..
# Output binary: CollectorBox_RevA/bin/linux-aarch64/Collector
```

---

## Generating Startup Files

```sh
cd /shared/EPICS/CollectorBox_RevA
source /shared/EPICS/collectorBox.sh 201   # sets THIS_PI_COLL_NUM=201
python3 GenerateCmdFile.py
# Outputs: iocBoot/iocCollectorApp/st_201.cmd
#          iocBoot/iocCollectorApp/softIOC_201_settings.req
```

**Never edit `st_<N>.cmd` by hand** — it will be overwritten on next run.

### Per-Detector Macro Assignments
_Source: `GenerateCmdFile.py::build_macros()` ✅ verified 2026-04-19 — GenerateCmdFile.py:L299-378_

For each active cable, `build_macros()` computes the following macros passed to `dbLoadRecords`:

| Macro | Source | Description |
|-------|--------|-------------|
| `DetNbr` | `DNG_ID` (zero-padded 3 digits) | GS hole number assigned by dongle |
| `VMEGS` | computed `get_dvi_number(box, cable)` | DVI/GS VME slot number (zero-padded) |
| `DigChNum` | `(cable-1) % 5` | DIG channel within a digitizer (0–4) |
| `DigNum` | `((cable-1)//5) % 2 + 1` | Which DIG board (1 or 2) on this strip |
| `VMENum` | `get_vme_num(box, cable)` | VME crate number for this cable |
| `BusAddr` | `cable` (int) | Cable number — used as hardware bus address in DB templates |
| `DetNbr_MapVal` | `DNG_ID` unpadded | Raw GS hole number for map PVs |
| `VMEGS_MapVal` | `get_dvi_number(box, cable)` unpadded | Raw DVI number for map PVs |
| `DS_GEHV` | `GE_HV_OPERATING` | DOL init for `MOD${DetNbr}_DS_GEHV` |
| `DV_HIHI/HIGH/LOW/LOLO` | `GE_HV_OPERATING ± 2%/4%` | HV alarm limits |
| `GE_HV_ABSMAX` | `GE_HV_NAMEPLATE` | Absolute HV ceiling |
| `Ge_Prefix/ID/Type` | scan file columns 14–16 | Detector ID info |
| `Ge_MCA_Resolution/Reset_Period/GR/Depletion_Voltage` | `mca_data_<N>.txt` | MCA calibration data |
| `DV_TEMP_INPA` | `GS<NNN>_Conv_Temp.VAL` (default) | Temperature INPA link; overridable in `special_detectors.txt` |
| `DV_TEMP_PREC` | `1` (default, `2` for det 048 PT500) | Temperature display precision |

### Geometry Helpers
_Source: `GenerateCmdFile.py::get_dvi_number()`, `get_vme_num()` ✅ verified 2026-04-19 — GenerateCmdFile.py:L236-256_

**`get_dvi_number(box, cable)` — DVI/GS hole number mapping:**

| Box | Formula | Notes |
|-----|---------|-------|
| 201 | `cable × 2` | Strips 1–3, even GS numbers |
| 202 | `cable × 2 + 60` (cables ≤25) else 0 | Upper hemisphere |
| 203 | `cable × 2 − 1` | Strips 1–3, odd GS numbers |
| 204 | `cable × 2 − 1 + 60` (cables ≤5); `(cable−5) × 2 − 1 + 60` (cables 11–30); 0 otherwise | Cables 6–10 are strip-2 with no detectors; cable 30 out of range |

**`get_vme_num(box, cable)` — VME crate assignment:**

| Box | VME offset | Formula |
|-----|-----------|--------|
| 201 | 1 | `(cable−1) // 10 + 1` |
| 202 | 4 | `(cable−1) // 10 + 4` |
| 203 | 7 | `(cable−1) // 10 + 7` |
| 204 | 10 | `(cable−1) // 10 + 10` |

Each box spans up to 3 VME crates (10 cables per crate). Together the 4 boxes cover VME crates 1–12, matching the 12 IOC crates (`192.168.203.141–145, 177–183`). ✅ verified 2026-04-30 — `GenerateCmdFile.py:L248` (`_VME_OFFSETS = {201:1, 202:4, 203:7, 204:10}`) + `L251-252` (formula: `(cable-1)//10 + offset` → 3 crates/box); IOC IP list confirmed `ANLDAQ/EPICS_env.sh:L48-59`.

### Autosave .req File Generation
_Source: `GenerateCmdFile.py::generate_req()` ✅ verified 2026-04-19 — GenerateCmdFile.py:L388-430_

`generate_req()` writes `iocBoot/iocCollectorApp/softIOC_<N>_settings.req` listing only **`CollectorSoftControl`** output PVs for autosave — hardware register PVs (`CollectorLocalSerial`) are explicitly excluded so the IOC never writes stale data to hardware at startup. The PV list is parameterized by `AUTOSAVE_PVS_PER_ACTIVE_DET` (currently 5 PVs: HV demand, step size, hysteresis, absmax, DS_GEHV).

---

## Running the IOC

```sh
# Install service
sudo cp /shared/EPICS/softIOC.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable softIOC.service
sudo systemctl start softIOC.service

# Connect to IOC shell interactively
telnet localhost 20000
```

IOC startup sequence:
1. **Init** (~10s): loads databases, restores autosave PVs, starts scanning
2. **Post-init** (optional): `softioc_postscript.sh` runs additional PV setup
3. **Done**: sentinel file `/home/dgs/softIOC_<N>_is_done.log` created

Autosave saves listed PVs every 30 seconds to `/home/dgs/autosave/softIOC_<N>_settings.sav`.

---

## softioc_postscript.sh — Post-Init Hardware Commissioning
_Source: `collectorboxpi/softioc_postscript.sh` — 673-line Bash script_

Runs automatically after IOC init completes (launched by `startSoftIOC.sh`). Performs a 6-phase hardware commissioning sequence over EPICS CA using `caget`/`caput`. Logs to `~/softioc_<N>_postScript_log.txt`. All `caget`/`caput` calls use a 150 ms timeout (`-w 0.150 -t`). ✅ verified 2026-04-30 — `collectorboxpi/softioc_postscript.sh:L3` (`WAIT_TIME=0.150`).

**Input:** `$LIST_OF_VME_GS_NUMBERS` (exported by `collectorBox.sh` as space-separated string; converted internally to Bash array)

### Phase 1 — Relay & SPI Enable (loop over 6×5 stripe/port grid)
For each stripe (1–6) × port (1–5) position (= 30 slots total):
- **Placeholder (`NUM=000`):** immediately disables relay + tristates SPI/clock lines → skips caget
- **Live slot:** does `caget VME_GS<NUM>_to_True_GS` to resolve true GS number
  - If `TRUE_GS=000` or `SBX_Present=0`: disables relay + tristates → skips
  - Otherwise: enables relay (`GS<COLL>_prly_s<S>_<P> on`), activates SPI + clock lines, closes earth relay (`crly_earth_s<S>`), enables clock+sync
- Waits 10 s after all relays are on for power to settle

### Phase 2 — I2C & Preamp Enable (loop over detector list)
**Placeholder skip:** `NUM=000` → immediately `continue` (avoids channel-connect timeout on empty slots like box 204 stripe 2) ✅ verified 2026-04-20 — commit 8cf8952
For each connected detector with SBX present:
- `GS<NNN>_ResetAllI2CMach = run` — reset I2C state machines
- `GS<NNN>_ResetAllScanMachines = run` — reset all scan machines
- `GS<NNN>_PreampI2C_OE_CTL = I2C_enbl` — enable I2C output
- `GS<NNN>_PreampI2C_FIFO_RESET = run` — flush I2C FIFO
- `GS<NNN>_PA_QI_Mode = I2C` — set preamp to I2C mode
- `MOD<NNN>_DV_EN = 1` — enable DVI channel
- `GS<NNN>_Slopebox_Scan_control = read/write` — arm slopebox scanner
- Waits 2 s for PVs to settle

### Phase 3 — BGO HV Interlock
**Placeholder skip:** `NUM=000` → immediately `continue` ✅ verified 2026-04-20 — commit 8cf8952
For each connected detector with SBX present:
- Reads `GS<NNN>_SlopeBoxBGOInterlock`:
  - `"Interlock OPEN"` → `BGO_HV_CTRL = "BGO HV Off"`
  - `"Interlock Closed"` → `BGO_HV_CTRL = "BGO HV ON"`, sets all 14 BGO HV channels (0–13) to **200 V baseline**
  - Any other value → error; forces `"BGO HV Off"`

### Phase 4 — HPGe HV Interlock & Ramp
**Placeholder skip:** `NUM=000` → immediately `continue` ✅ verified 2026-04-20 — commit 8cf8952
For each connected detector with SBX present:
- Sets ramp parameters: `GE_HV_STEP_SIZE=10`, `MANUAL_GE_HV_DEMAND=0`, `GE_HV_HYSTERESIS=5`
- **Sets dynamic HV alarm limits on `Conv_GeHV`** (NEW — commit 8cf8952): reads `MOD<NNN>_DS_GEHV` (operating HV) and `GS<NNN>_GE_HV_ABSMAX` (nameplate HV), then sets:
  - `Conv_GeHV.HIGH = operating_HV + 100 V` (warning threshold)
  - `Conv_GeHV.HIHI = nameplate_HV + 100 V` (alarm threshold)
  - `Conv_GeHV.LOW = 0`, `Conv_GeHV.LOLO = 0` ✅ verified 2026-04-20 — `softioc_postscript.sh` commit 8cf8952
- Reads `GS<NNN>_SlopeBoxTempHigh`:
  - `"Temp HIGH"` → `GE_HV_CTRL = "Ge HV Off"` (interlock open)
  - `"Temp OK"` → `GE_HV_CTRL = "Ge HV ON"`, reads operating HV from `MOD<NNN>_DS_GEHV`, writes it to `GE_HV_DEMAND_VOLTS` (ramp starts)
  - Any other value → error; `GE_HV_DEMAND_VOLTS = 0` (safe ramp-down)

### Phase 5 — PT100 Temperature Calibration
For each detector with `GS<NNN>_Dig_Channel != -1`:

Reads both temperature sensors (PT100 via `MOD<NNN>_DV_TEMP`, PT500 via `GS<NNN>_Calc_PT500_Temp`) and applies a linear gain+offset correction to PT100 so it tracks PT500 (the ground truth).

**Skip conditions (CAL_SKIP=1):**
- PT500 > −175°C (detector too warm)
- PT100 PREC field = 2 (PT100 failed; PT500 is fallback sensor)
- PT500 between −243°C and −190°C (impossibly cold non-zero = bypassed sensor)

**Calibration formula** (all temps in °C; converted to K internally):
```
C_offset = 29.6945, C_error = 4.0
gain  = (T_PT500 + 273.15 - C_offset) / (T_PT100 + 273.15 - C_offset - C_error)
const = 273.15 x gain - (C_offset + C_error) x gain + C_offset
EPICS CALC field: "A * gain + const"
```
- **Validity check:** gain must be in [0.8, 1.25]; if out of range, waits 10 s and retries once
- On second failure → CAL_SKIP=1 (uncalibrated fallback)

**High-temperature alarm threshold:**
- Normal: `.HIGH = T_PT500 (K) + 6 K`; clamped to max 98 K
- Fallback: `.HIGH` based on whichever sensor is still valid; clamped to 100 K
- Fallback CALC: `"A + 273.15"` (raw PT100 degC to K, no gain)

### Phase 6 — Health Check & Error Report
**Placeholder skip + index tracking (NEW — commit 8cf8952):** Loop now uses `IDX` to track `STRIPE=IDX/5+1`, `PORT=IDX%5+1` for relay retry; `NUM=000` → immediately `continue`. ✅ verified 2026-04-20 — `softioc_postscript.sh` commit 8cf8952

For each detector, reads ~40 PVs and checks ranges:

| Check | Error condition |
|-------|-----------------|
| PT100 temp | >400 K → relay retry up to 3× then ERROR; >100 K (warm warning), <50 K (DVI fault) |
| PT500 temp | >105°C (fault), >−160°C (warm), bypassed warning |
| Slope Box ID | non-numeric, =255 (DVI fault), >128, <1 |
| Power board status | =0 (I2C comm error) or not 255 |
| 24V rail | outside [23, 24] V |
| 5V rail | outside [4, 5] V |
| +12V rail | outside [11, 12] V |
| −12V rail | outside [−12, −11] V |

**PT100 > 400 K relay retry (NEW — commit 8cf8952):** Instead of immediately logging an error, the script power-cycles the preamp relay (`GS<COLL>_prly_s<S>_<P>`) up to **3 times** (`off` → 1 s → `on` → 2 s), re-reads PT100 after each attempt. On success → WARNING only. If still > 400 K after 3 retries → ERROR + I2C_COMM_ERROR increment. ✅ verified 2026-04-20 — `softioc_postscript.sh` commit 8cf8952

**I2C comm error thresholds** (configured at top of script):
- `I2C_COMM_ERROR_TRIGGER_THRESHOLD = 5` — errors below this are suppressed (noise)
- `I2C_COMM_ERROR_MUTE_THRESHOLD = 7` — errors above this are fully muted (avoid log spam)
- **DVI comm error thresholds:** trigger=0, mute=3

**Summary output:**
```
GS XXX is OK           <- no errors or warnings
GS XXX -> Errors:N Warnings:M
Number of Detector: N -> Errors:E Warnings:W
N Connected GSs  / M Connected SBXs / K Interlocked BGO / J Interlocked HPGe
```
✅ verified 2026-04-18 — `collectorboxpi/softioc_postscript.sh` (full read, all 673 lines)
✅ updated 2026-04-20 — commit 8cf8952 ("set HV range"): placeholder slot skip in Phases 2–4 + health check; dynamic `Conv_GeHV` alarm limits in Phase 4; PT100 relay retry (3×) in Phase 6

---

## Networking / PXE Boot

- **DHCP server:** Einstor (192.168.203.1) — ANL-managed, no DGS control
- **tftp/file server:** fs2.onenet (192.168.203.71)
- **NFS mount:** `fs2:/mnt/vol1/fs2/nfs/piserver/shared` → `/shared` on each Pi
- **Per-Pi home:** `fs2:/mnt/vol1/fs2/nfs/piserver/home/<hostname>` → `/home/dgs` ✅ verified 2026-04-08 — `piserver/os/Debian13/etc/rc.local:L24-26` (NFS)
- `/var` is mounted in RAM (tmpfs) — OS is effectively read-only

**Pi identity** is determined at boot by MAC address via `rc.local`:
```sh
[ "$MAC" = "b8:27:eb:fc:97:08" ] && hostname pi0
[ "$MAC" = "b8:27:eb:57:19:db" ] && hostname pi1
[ "$MAC" = "b8:27:eb:5a:d0:8e" ] && hostname pi2
[ "$MAC" = "b8:27:eb:99:18:3f" ] && hostname pi3
```

**Hostname persistence fix (2026-04-03):** The shared NFS root has `/etc/hostname` = `raspberrypi-collectorBox`. After `rc.local` sets the hostname via `hostname pi0`, login/logout would reset it from `/etc/hostname`. Fix: bind-mount a tmpfs file over `/etc/hostname` in `rc.local`:
```sh
# bind-mount per-Pi hostname over shared /etc/hostname
MY_HOSTNAME=$(hostname)
echo "$MY_HOSTNAME" > /run/pi-hostname
mount --bind /run/pi-hostname /etc/hostname
```
`/run` is tmpfs (RAM), local to each Pi, always writable. This shadows the shared `/etc/hostname` without modifying the NFS root. Takes effect on next reboot.

---

## Discord Integration

When a collector Pi softIOC finishes starting, a Discord webhook message is sent. Webhook URL stored in `discord.WebHook` file (`export WEBHOOK=<url>`).

---

## EPICS Port Convention (inferred)

Collector boxes use their own CA port — check `collectorBox.sh` for the active port when troubleshooting CA connectivity.

---

## SPI Hardware Communication Layer

### Protocol
- Uses **SPI1** (auxiliary SPI, not SPI0) via the `bcm2835` library
- **24-bit transactions**: `[R/W(1) | Addr(7)] [DataHi(8)] [DataLo(8)]`
- Data clocked out MSB first; FPGA clocks in on rising edge, Pi captures on rising
- EPICS mutex (`spi_driver_mutex`) protects all SPI transactions — non-reentrant
- **SPI clock speed**: `SPI_DEFAULT_SPEED = 50` → `250 MHz / (2×(50+1)) ≈ 2.45 MHz` ✅ verified 2026-04-07 — `initTrace.c:L44` (changed from 48 on 2024-01-03). Formula: `f = 250MHz / (2×(speed+1))`.

### DEVSEL Bus (Device Selection)
- 5-bit GPIO bus selects up to 32 devices (DEVSEL 0–31) ✅ verified 2026-04-22 — `DEVSEL_bus.c:L31-35` (comment table: bit 0→GPIO13, bit1→GPIO23, bit2→GPIO24, bit3→GPIO25, bit4→GPIO26; L80-85 shift logic confirmed)
- GPIO pins: GPIO13(bit0), GPIO23(bit1), GPIO24(bit2), GPIO25(bit3), GPIO26(bit4) ✅ verified 2026-04-22 — `DEVSEL_bus.c:L31-35,L80-85`
- `Set_DEVSEL(Bidx)` asserts the correct GPIO pattern before each transaction
- DEVSEL=0 = no device selected (bus idle)

### ADC Scanner
- GPIO12 = `SCANNER_CONTROL_PIN` (`RPI_BPLUS_GPIO_J8_32`) — HIGH=block scanner, LOW=enable scanner ✅ verified 2026-04-08 — `spi.h:L7-9`, `spi.c:L195-206`
- `RESET_ADC_SCANNER()` = HIGH (blocks), `ENABLE_ADC_SCANNER()` = LOW (allows)
- During EPICS runtime: PV fires every 10s, grabs mutex, runs one ADC scan ✅ verified 2026-04-08 — `Pickoff_reg.db:L990` (SCAN="10 second")

### Key Functions (`spi.c`)
| Function | Purpose |
|----------|---------|
| `SPI1_setup(init_flag, speed)` | Init SPI1, configure GPIOs, set clock speed |
| `Set_DEVSEL(n)` | Assert 5-bit DEVSEL GPIO bus to select device n |
| `Do_SPI1_transaction(RW, Bidx, Addr, Data)` | Full 24-bit SPI read/write to device Bidx |
| `SPI1_exit()` | Cleanly shut down SPI1 |

### Global Data Structure (`CollectorSupport.h`)
```c
typedef struct {
    unsigned short GLBL_CollectorDataArray[32][1024];   // raw data per device  ✅ verified 2026-04-22 — CollectorSupport.h:L141
    unsigned short GLBL_CollectorControlVals[32][256];  // control values per device  ✅ verified 2026-04-22 — CollectorSupport.h:L142
    epicsFloat64   GLBL_CollectorFloatVals[32][256];    // float mailboxes  ✅ verified 2026-04-23 — CollectorSupport.h:L143
    unsigned short *GLBL_CollectorArrayPtr[32];         // walking pointers  ✅ verified 2026-04-23 — CollectorSupport.h:L144
    epicsFloat64   GLBL_ConversionCoefficients[64][2]; // m,b for 64 conversions  ✅ verified 2026-04-23 — CollectorSupport.h:L146
    epicsFloat64   PT100Coefficients[10][2];           // m,b for PT100 temp fitting (10 segments)  ✅ verified 2026-04-23 — CollectorSupport.h:L147
    epicsFloat64   PT500Coefficients[10][2];           // m,b for PT500 temp fitting (10 segments)  ✅ verified 2026-04-23 — CollectorSupport.h:L148
} CollectorGlobDataStructure;
```
First index (32) = DEVSEL device number. Second index = register/data slot.

### I2C Control Flags (`CollectorSupport.h`)
| Flag | Value | Meaning |
|------|-------|---------|
| `Collector_I2C_DONE` | 0x8000 | Transaction complete |
| `Collector_I2C_RPTS` | 0x4000 | Repeated start |
| `Collector_I2C_NACK` | 0x2000 | NACK received |
| `Collector_I2C_READ` | 0x1000 | Read transaction |
| `Collector_I2C_SAVE` | 0x0800 | Save result |
| `Collector_I2C_EXTD` | 0x0400 | Extended |
| `Collector_I2C_LOOP` | 0x0200 | Loop mode |
| `Collector_I2C_TOGS` | 0x0100 | Toggle |
| `Collector_I2C_CMDREAD` | 0x0001 | Command read |

---

## Collector Box → GS Hole Assignments

There are **4 collector boxes** total. This repo contains `st_201.cmd` through `st_204.cmd`, **all with active per-detector records** as of commit 2309422 (2026-04-13). All 4 Pi collector boxes (pi0–pi3) are deployed. ✅ verified 2026-04-26 — `st_201.cmd` (179 dbLoadRecords), `st_202.cmd` (196), `st_203.cmd` (229), `st_204.cmd` (175)

| Pi | IOC # | Location | GS Holes | Status |
|----|-------|----------|-----------|--------|
| pi0 | 201 | South-East | GS 018,020,022,024,026,028,030,036,038,040,042,044,052,054,056,070 — **16 detectors** | ✅ verified 2026-04-13 — commit 2309422, `st_201.cmd` |
| pi1 | 202 | South-West | GS 064,066,068,072,074,076,078,080,082,084,086,088,092,094,096,098,102,104,106,110 — **20 detectors** | ✅ verified 2026-04-13 — commit 2309422, `st_202.cmd` |
| pi2 | 203 | North-East | GS 011,013,017,019,021,023,025,027,029,031,033,035,037,041,043,045,049,051,055,057,059,065,105 — **23 detectors** | ✅ verified 2026-04-13 — commit 2309422, `st_203.cmd` |
| pi3 | 204 | North-West | GS 061,063,071,073,075,077,079,081,083,087,089,093,095 — **13 detectors** | ✅ verified 2026-04-13 — commit 2309422, `st_204.cmd` |

✅ verified 2026-04-08 — `CollectorBox_RevA/iocBoot/iocCollectorApp/st_20{1,2,3,4}.cmd` (grep DetNbr). Location labels verified against `collectorboxpi/README.md:L257-259` + `nfs_layout.md` piserver README table.

EPICS CA port is set via `EPICS_env.sh`: **5064/5065** for array use, **5074/5075** for test stand (G-wing lab). The st_20x.cmd scripts pass `${EPICS_CA_SERVER_PORT}` from the environment. ✅ verified 2026-04-07 — `EPICS_env.sh:L1-2` + `st_201.cmd:L5-8`

---

## db/ Template Files (18 templates)

| File | Purpose |
|------|---------|
| `CollectorDiagCtl.db` | Collector diagnostic control |
| `ControlGlobals.db` | Box-level global control PVs |
| `CtrlFPGA.db` | Control FPGA records |
| `CtrlFPGA_reg.db` | Control FPGA register-level records |
| `DetSpec.db` | Per-detector spec records |
| `HV_STEP.db` | HV step control |
| `Pickoff.db` | Pickoff board records |
| `PickoffDiagCtl.db` | Pickoff diagnostic control |
| `Pickoff_reg.db` | Pickoff register-level records |
| `PowerBoardCalcChain.db` | Power board calculated chain |
| `PreampCalcChain.db` | Preamp calculated chain |
| `SlopeBox.db` | Slope box records |
| `StripeGlobals.db` | Stripe-level global PVs |
| `StrpFPGA.db` | Stripe FPGA records |
| `StrpFPGA_reg.db` | Stripe FPGA register-level records |
| `unused_cable.db` | Stub for unused cable slots |
| `unused_dvi.db` | Stub for unused DVI slots |
| `unused_gs.db` | Stub for unused GS slots |

---

## Commissioning Workflow — Adding / Removing Detectors

> 📄 **Full detail in:** `collectorboxpi_commissioning.md` — Pre_EPICS_Collector programs, Src/ library (SPI/DEVSEL/ADC/relay), Programs/ internals (TurnOnAllConnected, Dump_Preamp_EEPROM, Write_to_EEPROM), Add_Remove_Detectors.sh sequence.

**Quick summary:** Run `Add_Remove_Detectors.sh` (as root) to stop the IOC, scan all cables, regenerate the startup command file, and restart the IOC. The 4-step commissioning sequence: `ALL_power_OFF` → `Scan_Collector_FPGAs` → `Scan_DVI_Power` → `Scan_DVI_Comms` → feed results to `GenerateCmdFile.py`.



---

## Connections to Other Subsystems

- **[ioc](ioc.md)** — shares the same IOC concept (VxWorks IOC for hardware boards; this is the Pi soft IOC for collector boxes)
- **[ANLDAQ](ANLDAQ.md)** — may monitor collector box PVs via EPICS CA
- **[lnfill](lnfill.md)** — parallel infrastructure (also Pi-based, also uses EPICS)

## Cross-References

- [collectorboxpi_commissioning.md](collectorboxpi_commissioning.md) — Pre_EPICS_Collector commissioning utilities detail: Src/ C library (SPI/DEVSEL/ADC/relay), Programs/ internals (TurnOnAllConnected, Dump_Preamp_EEPROM, Write_to_EEPROM)
- [collector_fpga.md](collector_fpga.md) — CtrlFPGA + StripeFPGA firmware detail (git repo); the hardware the Pi IOC talks to
- [collector_box_fpga.md](collector_box_fpga.md) — ControlStripe + CtlFanout FPGAs (PSG SVN origin): per-stripe 48V relay/clock/SYNC/LED control (Spartan-3) and RPi SPI gateway + ADS1158 ADC scanning (Spartan-6)
- [collectorbox_PVs.md](collectorbox_PVs.md) — Full PV list (1,431 records/detector); use exec grep for PV lookups
- [collectorbox_devicesupport.md](collectorbox_devicesupport.md) — EPICS device support internals: SPI driver, CAMAC_IO link
- [sbx.md](sbx.md) — Slope Box Extension hardware; BGO HV, GS_ID dongle, pickoff card
- [nfs_layout.md](nfs_layout.md) — PXE boot infrastructure on fs2.onenet; piserver NFS layout and MAC table
- [gammasphere_geometry.md](gammasphere_geometry.md) — GS hole numbering and collector box assignments
- [influxdb_grafana.md](influxdb_grafana.md) — Temperature data pushed to InfluxDB by SaveTemp.sh (runs on Pi)
- [collector_ctrlFPGA_registers.md](collector_ctrlFPGA_registers.md) — CtrlFPGA register interface detail (141 registers): pulsed_control bits, FPGA_CTL_REG bits, ILA mux, DPRAM bank select, per-stripe ADC monitoring, BGO FPGA voltages

*Created: 2026-04-06 | Last reviewed: 2026-04-27*
