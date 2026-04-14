# collectorboxpi — EPICS Soft IOC for Collector Box (Raspberry Pi)

> 🔗 **Related:** `sbx.md` — SBX (Slope Box Extension) hardware that this IOC controls | `collectorbox_PVs.md` — full PV list | `gammasphere_geometry.md` — GS hole numbering

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
  - **Note:** pi1/pi2/pi3 collector boxes are **not yet implemented** (hardware not deployed). The testing pi1 MAC (`df:8c:d6`) is active in rc.local — if plugged in it claims hostname `pi1`, but this is harmless since there is no production pi1 yet.
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
| `special_detectors.txt` | Per-detector overrides (HV, type, etc.) |

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
- 5-bit GPIO bus selects up to 32 devices (DEVSEL 0–31)
- GPIO pins: GPIO13(bit0), GPIO23(bit1), GPIO24(bit2), GPIO25(bit3), GPIO26(bit4)
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
    unsigned short GLBL_CollectorDataArray[32][1024];   // raw data per device
    unsigned short GLBL_CollectorControlVals[32][256];  // control values per device
    epicsFloat64   GLBL_CollectorFloatVals[32][256];    // float mailboxes
    unsigned short *GLBL_CollectorArrayPtr[32];         // walking pointers
    epicsFloat64   GLBL_ConversionCoefficients[64][2]; // m,b for 64 conversions
    epicsFloat64   PT100Coefficients[5];               // PT100 temp sensor fit
    epicsFloat64   PT500Coefficients[5];               // PT500 temp sensor fit
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

## Connections to Other Subsystems

- **ioc/** — shares the same IOC concept (VxWorks IOC for hardware boards; this is the Pi soft IOC for collector boxes)
- **ANLDAQ** — may monitor collector box PVs via EPICS CA
- **lnfill/** — parallel infrastructure (also Pi-based, also uses EPICS)

## Cross-References

- `knowledgeBase/collector_fpga.md` — CtrlFPGA + StripeFPGA firmware detail; the hardware the Pi IOC talks to
- `knowledgeBase/collectorbox_PVs.md` — Full PV list (1,431 records/detector); use exec grep for PV lookups
- `knowledgeBase/collectorbox_devicesupport.md` — EPICS device support internals: SPI driver, CAMAC_IO link
- `knowledgeBase/sbx.md` — Slope Box Extension hardware; BGO HV, GS_ID dongle, pickoff card
- `knowledgeBase/nfs_layout.md` — PXE boot infrastructure on fs2.onenet; piserver NFS layout and MAC table
- `knowledgeBase/gammasphere_geometry.md` — GS hole numbering and collector box assignments
- `knowledgeBase/influxdb_grafana.md` — Temperature data pushed to InfluxDB by SaveTemp.sh (runs on Pi)
