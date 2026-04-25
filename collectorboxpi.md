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
    - [Add_Remove_Detectors.sh (must run as root)](#add_remove_detectorssh-must-run-as-root)
    - [Pre_EPICS_Collector/ — Commissioning Utilities](#pre_epics_collector--commissioning-utilities)
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

Each box spans up to 3 VME crates (10 cables per crate). Together the 4 boxes cover VME crates 1–12, matching the 12 IOC crates (`192.168.203.141–145, 177–183`).

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

Runs automatically after IOC init completes (launched by `startSoftIOC.sh`). Performs a 6-phase hardware commissioning sequence over EPICS CA using `caget`/`caput`. Logs to `~/softioc_<N>_postScript_log.txt`. All `caget`/`caput` calls use a 150 ms timeout (`-w 0.150 -t`).

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

There are **4 collector boxes** total. This repo contains `st_201.cmd` through `st_204.cmd`, but **only 201 and 202 have per-detector records**; 203 and 204 have no detector entries in the current repo (odd-numbered GS holes use an older piserver).

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

The `collectorboxpi/` root contains two top-level scripts that manage the detector add/remove lifecycle:

### `Add_Remove_Detectors.sh` (must run as root)

> ⚠️ **Known bug (2026-04-18):** Line 11 sets `EPIC_dir=/share/EPICS` (missing trailing `d`) — the `cd $EPIC_dir` at L12 silently fails. The script continues because all subsequent paths are hardcoded absolute (`/shared/EPICS/Pre_EPICS_Collector`), so the actual scan and IOC restart steps still work correctly. The failing `cd` is a no-op bug. ✅ verified 2026-04-18 — `Add_Remove_Detectors.sh:L10-12` vs working paths at L24

Full sequence executed:
1. `systemctl stop softIOC.service` — stop the EPICS IOC
2. `cd /shared/EPICS/Pre_EPICS_Collector` (via hardcoded absolute path — the earlier `cd $EPIC_dir` fails silently)
3. `./ALL_power_OFF` — cut power to all SBXs on all stripes
4. `sleep 5` — allow power to fully discharge
5. `./Scan_DVI_Power` — scan 48V power state per DVI cable; exit code 0 = all OK, exit 148 (= C exit 404 mod 256) = cables not usable
6. `./Scan_DVI_Comms` — scan DVI communications; reads SBID from pickoff address; generates `SCAN_OUTPUT_3_COMM_<BOX>.txt`
7. `./GenerateCmdFile.py` — re-generate `st_20x.cmd` and `softIOC_<N>_settings.req` from scan output
8. `systemctl start softIOC.service` — restart IOC with new configuration

Note: `Scan_Collector_FPGAs` is optional (commented out) — use when Stripe FPGA reachability is in question.

### `Pre_EPICS_Collector/` — Commissioning Utilities

Standalone C programs that run on the Raspberry Pi **before EPICS is active**. They communicate directly with Stripe and Control FPGAs via BCM2835 SPI. Must run as root.

**Normal 4-step commissioning sequence:**
1. `ALL_power_OFF` — cut all SBX power
2. `Scan_Collector_FPGAs` → `SCAN_OUTPUT_1_COLL_<BOX>.txt`
3. `Scan_DVI_Power` → `SCAN_OUTPUT_2_POWER_<BOX>.txt`
4. `Scan_DVI_Comms` → `SCAN_OUTPUT_3_COMM_<BOX>.txt` ← this feeds `GenerateCmdFile.py`

**Full program inventory (15 executables):**

| Executable | Purpose |
|---|---|
| `ALL_power_OFF` | Turns off all power to all SBXs on all stripes |
| `Dump_EEPROMs` | Reads and dumps preamp EEPROM contents to screen |
| `Dump_Preamp_EEPROM` | Reads preamp and dongle EEPROM data for a specific cable |
| `Scan_Collector_FPGAs` | Tests communication with all Stripe FPGAs; generates SCAN_OUTPUT_1 |
| `Scan_DVI` | Scans DVI power with per-stripe enable; reads Slope Box ID and Dongle ID |
| `Scan_DVI_Comms` | Scans DVI comms; reads SBID from pickoff address; generates SCAN_OUTPUT_3 |
| `Scan_DVI_Comms_No_Reg_Writes` | Same as above but without writing to FPGA registers |
| `Scan_DVI_Grounding` | Scans DVI grounding / ground fault status |
| `Scan_DVI_Power` | Scans 48V power state per DVI cable; generates SCAN_OUTPUT_2 |
| `Scan_DVI_Power_with_SBID` | Same as above + reads Slope Box IDs |
| `Test_Port_Comms` | Interactive: select channel, enable power, do SPI I/O — for debugging |
| `TurnOnAllConnected` | Turns on power to all connected cables from a turn-on data file |
| `Write_to_DPRAM` | Reads a data file and writes it to the DPRAM of a specified SBX |
| `Write_to_EEPROM` | Reads address/data file and writes to 24AA002 EEPROM (byte or page mode) |
| `spi_with_b_mbo_debug` | Low-level SPI debug utility for verifying raw SPI transfers |
| `SPI_rw` | Interactive SPI read/write for testing arbitrary register access. Usage: `sudo ./SPI_rw <r|w> <devsel> <addr> <data>` (devsel 0–31, addr 0–1023, data 16-bit). See `collectorboxpi/Pre_EPICS_Collector/SPI_Address.md` for full register map and examples. ✅ verified 2026-04-18 — `SPI_Address.md:L1-16` |

**SPI architecture (brief):**
- Uses **SPI1** (Aux SPI), not SPI0; enabled via `dtoverlay=spi1-1cs` in `/boot/config.txt`
- 24-bit transactions: bit 23 = R/W, bits 22:16 = 7-bit register address, bits 15:0 = 16-bit data
- **5-bit DEVSEL bus** on GPIO 13/23/24/25/26 (binary device-select multiplexer, 0–31 devices)
- Banked addressing for FPGAs with >127 registers: bank# written to addr 127, then real address
- SpreadsheetSrc: FPGA register maps auto-generated from PSG spreadsheets (StrpFPGA, CtrlFPGA)

✅ verified 2026-04-17 — `collectorboxpi/Pre_EPICS_Collector/README.md` + `Add_Remove_Detectors.sh`

#### `Src/` — C Library Details

The Pre_EPICS_Collector programs are built from a shared C library (`Src/`). These files are not yet surfaced in the executable-level docs above.

**`NonEPICS_SPI_lib.c` / `NonEPICS_SPI_lib.h`** (480 lines) ✅ verified 2026-04-23 — `wc -l NonEPICS_SPI_lib.c` = 480
- Implements all SPI and DEVSEL GPIO operations for non-EPICS programs.
- `SPI1_setup(init_flag, RequestedSpeed)` — initializes BCM2835 SPI1 and DEVSEL GPIO outputs (GPIO 13/23/24/25/26 as binary 5-bit output bus). Drive strength set to 16 mA, slow slew for all connector GPIOs. Samples on rising edge of SCLK (changed 2023-02-17), data out on falling edge, SCLK starts low, MSbit first, 24-bit fixed transaction length.
- `Set_DEVSEL(DEVSEL)` — asserts a 5-bit binary device select on the 5 GPIO lines. First clears all bits, then asserts the pattern for devices 0–31 via a switch/case with compile-time GPIO masks. 10 µs hold after assert (increased from 3 µs in 2022-12-16 debugging session).
- `Do_SPI1_transaction(RWflag, Bidx, UsrAddr, UsrData)` — single 24-bit SPI transaction. Sets DEVSEL, writes 3-byte SPI message (R/W | addr | data_hi | data_lo) to TX FIFO, polls BUSY, clears DEVSEL, returns 32-bit result (bits 23:16 = FPGA status, bits 15:0 = data).
- `Do_Banked_SPI1_transaction(RWflag, Bidx, UsrAddr, UsrData)` — extends above for banked DPRAM: addr ≤127 = bank 0 (direct), addr 128–1023 = banks 1–7 (writes bank# to addr 127 first, then real addr, then resets bank to 0). Addr ≥1024 returns 0xFFFFFFFF (error).
- `Do_Banked_SPI1_BlockXfr(RWflag, Bidx, StartAddr, Nwords, *data_array)` — block transfer of Nwords from StartAddr; handles bank boundary crossings mid-transfer automatically; resets bank to 0 when done.
- `Do_Banked_SPI1_RMW(Bidx, UsrAddr, ANDmask, ORMask)` — read-modify-write: reads register, ANDs with ANDmask, ORs ORMask, writes back. Handles banked addresses. 16-entry helper macros: `SET_BIT_nn` / `CLEAR_BIT_nn` for each bit 0–15.
- `SPI1_exit()` — calls `bcm2835_aux_spi_end()` + `bcm2835_close()` (drops CE2 — only call at full shutdown).

**`NonEPICS_Collector_lib.c`** (477 lines) ✅ verified 2026-04-23 — `wc -l NonEPICS_Collector_lib.c` = 477
- High-level collector box control functions, built on the SPI library above.
- **ADC scan loop** — The Control FPGA runs an ADS1158 ADC scanner ROM with 3 programs:
  - Program 0 (ROM addr 0): slow loop ~530 µs cycle time, all 16 channels + internal ADC values (REF/GAIN/TEMP/VCC/OFFSET)
  - Program 1 (ROM addr 64): fast loop ~50 µs, only 48V current monitor inputs (ADC ch 5–9)
  - Program 2 (ROM addr 128): fast loop ~50 µs, all channels
- `RESET_ADC_SCANNER(Program_Index)` — pauses scanner, selects program (writes ROM start address to CtrlFPGA), releases scanner.
- `PAUSE_ADC_SCANNER()` / `ENABLE_ADC_SCANNER()` — toggle GPIO `SCANNER_CONTROL_PIN` to halt/resume the ADC scan loop.
- `DO_ADC_CYCLE(delay1, delay2)` — enables scanner for `delay2` µs then stops it, giving one complete scan cycle.
- `COLLECT_AVERAGED_ADC_DATA(delay1, delay2, num_avg, enable_tracing)` — runs `num_avg` scan cycles, reads 128 words from DPRAM via block transfer, accumulates min/max/avg per address.
- **Relay control** — 4 relay types per cable (cables 1–30), each mapped to a bit in the corresponding stripe's `relay_control_sN` register (written to Stripe FPGA device 31):
  - `ENABLE/DISABLE_POWER(cable)` — bits 14:10 of relay_control_sx (prly = 48V power per cable)
  - `ENABLE/DISABLE_GNDFAULT_I(cable)` — bits 4:0 (irly = ground fault current injection)
  - `GROUND_DETECTOR(cable)` / `FLOAT_DETECTOR(cable)` — bits 9:5 (grly = ground fault check relay; GROUND = normal, FLOAT = lifted)
- `INITIALIZE_ALL_RELAYS()` — zeros relay_control for all 6 stripes (power off, no GFI, no float), then writes 0x0100 to stripe_control_sN for all 6 stripes (sets CRLY on one pole, clears clock/sync enable). Also sets GPIO J8_03 HIGH and J8_05 LOW as status markers.
- `ENABLE_ALL_COMMS()` — sets bits 4:0 of stripe_control_sN for all 6 stripes (enables clock and sync per cable).
- `convert_ADC_temp(ADCVAL, F_or_C)` — converts raw ADS1158 temperature reading to °C or °F: V = (ADC/30720)×4.096V; T = ((V_µV − 160000)/563) + 25.0°C. ✅ verified 2026-04-23 — `NonEPICS_Collector_lib.c:L459,L467,L470` (formula confirmed; code comment at L464 has typo `168000` but actual computation uses `160000`)

**`DPRAM_access.c`** (172 lines) ✅ verified 2026-04-23 — `wc -l DPRAM_access.c` = 172
- Lookup tables (arrays of DPRAM register addresses) used by commissioning utilities.
- `FPGA_VOLTAGE_ADDR[8][3]` — DPRAM addresses for stripe power supply voltages (12V/25V/33V) for stripes 1–6 plus BGO FPGA. Index 0 unused.
- `FPGA_IMON_ADDR[31]` — per-cable 48V current monitor DPRAM addresses, indexed by cable number 1–30 (index 0 unused).
- `FPGA_GFI_ADDR[31]` — per-cable ground fault injection status DPRAM addresses.
- `ADC_OFFSET_ADDR[7]`, `ADC_VCC_ADDR[7]`, `ADC_TEMP_ADDR[7]`, `ADC_GAIN_ADDR[7]`, `ADC_REF_ADDR[7]` — per-stripe DPRAM addresses for ADS1158 internal ADC self-calibration values (stripes 1–6, index 0 unused).

**`Non_EPICS_Globvars.c`** (572 lines)
- Definitions of global arrays used across the Non-EPICS programs (DPRAM image, min/max/avg accumulator arrays, etc.).

---

## Connections to Other Subsystems

- **ioc/** — shares the same IOC concept (VxWorks IOC for hardware boards; this is the Pi soft IOC for collector boxes)
- **ANLDAQ** — may monitor collector box PVs via EPICS CA
- **lnfill/** — parallel infrastructure (also Pi-based, also uses EPICS)

## Cross-References

- `knowledgeBase/collector_fpga.md` — CtrlFPGA + StripeFPGA firmware detail (git repo); the hardware the Pi IOC talks to
- `knowledgeBase/collector_box_fpga.md` — ControlStripe + CtlFanout FPGAs (PSG SVN origin): per-stripe 48V relay/clock/SYNC/LED control (Spartan-3) and RPi SPI gateway + ADS1158 ADC scanning (Spartan-6)
- `knowledgeBase/collectorbox_PVs.md` — Full PV list (1,431 records/detector); use exec grep for PV lookups
- `knowledgeBase/collectorbox_devicesupport.md` — EPICS device support internals: SPI driver, CAMAC_IO link
- `knowledgeBase/sbx.md` — Slope Box Extension hardware; BGO HV, GS_ID dongle, pickoff card
- `knowledgeBase/nfs_layout.md` — PXE boot infrastructure on fs2.onenet; piserver NFS layout and MAC table
- `knowledgeBase/gammasphere_geometry.md` — GS hole numbering and collector box assignments
- `knowledgeBase/influxdb_grafana.md` — Temperature data pushed to InfluxDB by SaveTemp.sh (runs on Pi)

*Created: 2026-04-06 | Last reviewed: 2026-04-20*
