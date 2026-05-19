# DGS System Overview — Gammasphere (Full System)

Stability: C2 - Active / semi-stable

> See also: [`overview_SmallSystem.md`](overview_SmallSystem.md) for DuoGe (DUO) and X-Array (DXA) small systems.

## Table of Contents

1. [What is DGS?](#what-is-dgs)
2. [Full Data Flow](#full-data-flow)
3. [Hardware Inventory](#hardware-inventory)
4. [EPICS Port Assignment (DGS)](#epics-port-assignment-dgs)
5. [Subsystem Map](#subsystem-map)
6. [Trigger Cycle (2 µs)](#trigger-cycle-2-µs)
7. [Firmware Versions (Current / Active)](#firmware-versions-current--active)
8. [LN Cooling System](#ln-cooling-system)
9. [Raspberry Pi Camera (darekpi02)](#raspberry-pi-camera-darekpi02)
10. [Key Elog References](#key-elog-references)
11. [Cross-References](#cross-references)

---

## What is DGS?

The **Digital Gamma-ray Spectrometer (DGS)** is a large-scale nuclear physics detector system at **Argonne National Laboratory (ANL)**. It detects and digitizes gamma rays from an array of up to 110 high-purity germanium (HPGe) detectors arranged in the Gammasphere geometry, each cooled by liquid nitrogen.

DGS is a full software+firmware+hardware stack:
- Real-time FPGA trigger decisions at 2 µs cycle time
- EPICS-based control over VME hardware
- Multi-threaded TCP data acquisition
- Automated LN2 cooling system
- Post-experiment ROOT/Parquet analysis pipeline

---

## Full Data Flow

```
┌──────────────────────────────────────────────────────────────────┐
│              HPGe + BGO DETECTORS (up to 110 GS holes)           │
│  HPGe crystal (LN2 cooled) + up to 7 BGO Compton shield segments │
└──────────────────────────┬───────────────────────────────────────┘
                           │ preamp signals (single-ended)
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│               SLOPE BOX  (1 per detector)                        │
│  Generates Ge bias HV (~3500V) + BGO bias HV (~550–800V)         │
│  Multiplexed ADC: temp, actual HV, PSU monitoring                │
└──────────────────────────┬───────────────────────────────────────┘
                           │ analog signals + 48VDC
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│         SBX — Slope Box Extension  (1 per detector)              │
│  Single-ended → differential conversion                          │
│  BGO sum signal + BGO pattern discrimination                     │
│  GS_ID dongle (identifies GS hole number)                        │
│  Pickoff Card: routes signals to correct DIG channels            │
└──────────────────────────┬───────────────────────────────────────┘
                           │ DVI-I cable (signals + power + comms)
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│  COLLECTOR BOX — CollectorBox_RevA  (4 boxes, 25–30 dets each)    │
│  Aggregates SBX signals; routes to VME crate digitizers          │
│  Controlled by Raspberry Pi soft IOC (collectorboxpi/)           │
│  EPICS PVs: HV, temp, BGO, FET bias, fan speed (815 PVs/det) ✅ verified 2026-04-18 — st_201.cmd: 8 DBs per det (Pickoff.db:448 + Pickoff_reg.db:264 + DetSpec.db:2 + HV_STEP.db:9 + PickoffDiagCtl.db:40 + PowerBoardCalcChain.db:10 + PreampCalcChain.db:16 + SlopeBox.db:26 = 815)   │
└──────────────────────────┬───────────────────────────────────────┘
                           │ differential analog → VME
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                    DIG — Digitizer (fpga/)                       │
│  Spartan-3 XC3S5000, ISE 14.7                                    │
│  10 channels per board, 14-bit ADC at 100 MHz                    │
│  Per-channel: delay → filter → discriminator (LED/CFD) → hit     │
│  Buffers accepted events in FIFO                                 │
└───────────┬─────────────────────────────────┬────────────────────┘
            │ SERDES (18-bit, 50 MHz)         │ VME bus
            │ hit pattern + energy            │ (event data readout)
            ▼                                 ▼
┌───────────────────────────┐      ┌───────────────────────────────┐
│  RTRG — Router (fpga/)    │      │  MVME5500 VME IOC             │
│  Virtex-4 XC4VLX80        │      │  VxWorks 5.5 RTOS             │
│  Aggregates 8 DIGs        │      │  gretDet.munch (vxworks/)     │
│  Computes X/Y multiplicity│      │  DMA readout of DIG FIFOs     │
└───────────┬───────────┬───┘      │  EPICS IOC (ioc/)             │
            │ SERDES    │ trigger  │  Boot: vme66.cmd / vme99.cmd  │
            │ (Link L)  │ cmd ▲    └───────────────┬───────────────┘
            ▼           └────────┐                 │ EPICS CA + TCP data
┌───────────────────────────┐    │                 ▼
│  MTRG — Master Trigger    │    │ ┌───────────────────────────────┐
│  (fpga/)                  │    │ │  ANLDAQ (anldaq/)             │
│  Virtex-4 / KU060         │    │ │  commander.py — PyQt6 GUI     │
│  Runs trigger algorithms  │    │ │  DIG/RTR/MTRG board control   │
│  TDC ~30 ps resolution    │    │ │  Run control + live monitor   │
│  2 µs cycle (20 frames)   │    │ │  tcpReceiverMT — binary files │
└───────────┬───────────────┘    │ └───────────────┬───────────────┘
            │ trigger decision   │                 │
            │ (TTCL, Link L)     │                 │
            └──► RTRG ──► DIG ───┘                 │
                                                   │ raw binary files
                                                   ▼
                                  ┌───────────────────────────────┐
                                  │  dgs_analysis/                │
                                  │  fastEventConstructor (ROOT)  │
                                  │  parquet_pysort (Parquet)     │
                                  │  Event building, calibration  │
                                  └───────────────────────────────┘
```

---

## Hardware Inventory

### VME Boards

| Board | Chip | Count | Role |
|-------|------|-------|------|
| DIG (MDIG/SDIG) | Spartan-3 XC3S5000 + XC3S400 (VME) ✅ verified 2026-04-18 — `FPGA/README.md:L71,73` + ChipScope project files (`DIG_20140908.cpj:L4` `deviceName1=XC3S5000`) | up to 64 | 10-ch digitizer |
| RTRG | Virtex-4 XC4VLX80 + XC3S400 (VME) ✅ verified 2026-04-18 — `FPGA/README.md:L70,73` | up to 8 | Router trigger |
| MTRG | XC4VLX80 or KU060 + XC3S400 (VME) + XC95144XL (CPLD) ✅ verified 2026-04-18 — `FPGA/README.md:L69` (ISE: XC4VLX80; Vivado: KU060=XCK060) | 1 | Master trigger |
| MVME5500 | PowerPC MPC7455 (G4/AltiVec) ✅ verified 2026-04-06 — VxWorks sym table `sp7455_*` BSP symbols in SVN archive (DGS_SVN vxWorks.sym) | 12 | IOC computer (one per VME crate) |

### Per-Detector Hardware

| Hardware | Count | Role |
|----------|-------|------|
| HPGe detector | up to 110 | Gamma-ray sensing crystal, cooled by LN2 |
| BGO shield | up to 7 per detector | Compton suppression scintillator segments |
| Slope Box | 1 per detector | Generates Ge+BGO bias HV; multiplexed ADC for temp/HV monitoring |
| SBX (Slope Box Extension) | 1 per detector | Signal conditioning; BGO sum + pattern; GS_ID dongle; 48VDC distribution |
| Pickoff Card | 1 per SBX | Routes signals to correct DIG channels; BGO HV demand control |
| Collector Box (CollectorBox_RevA) | 4 total | Hub for 25–30 detectors each (CB201: 30, CB202: 25, CB203: 30, CB204: 25 = 110 total); interfaces to digitizers + Pi soft IOC ✅ verified 2026-04-15 — collectorboxpi/collectorBox.sh:L16,27,38,49 |
| DVI-I cable | 1 per detector | Carries analog signals + power + comms from SBX → Collector Box |

### Computers (192.168.203.x)

| Host | IP | OS | Role |
|------|----|----|------|
| DCS2 | .56 | Ubuntu 24.04 | Data analysis, InfluxDB, Grafana |
| dgs1 | .122 | SL 6.8 | ~~Main DAQ~~ **RETIRED** — confirmed by Ryan 2026-05-18 | ✅ verified 2026-04-19 — wiki.anl.gov/gsdaq/Computers_and_networks
| dgs2 | .123 | Rocky 8.7 | Main DAQ (4TB SSD) | ✅ verified 2026-04-19 — wiki.anl.gov/gsdaq/Computers_and_networks
| dgs4 | .125 | SL 7.9 | — | ✅ verified 2026-04-19 — wiki.anl.gov/gsdaq/Computers_and_networks
| dgs6 | .184 | Rocky 8.7 | — | ✅ verified 2026-04-19 — wiki.anl.gov/gsdaq/Computers_and_networks
| gs-ts-south | .186 | — | Terminal server south — console for ioc01–ioc06 | ✅ verified 2026-04-19 — `ANLDAQ/EPICS_para.sh:L47`; routing confirmed 2026-04-27 — wiki DAQ_system
| gs-ts-north | .91 | — | Terminal server north — console for ioc07–ioc12 | ✅ verified 2026-04-19 — `ANLDAQ/EPICS_para.sh:L47`; routing confirmed 2026-04-27 — wiki DAQ_system
| gs-csw | .26 | — | Collector box south-west | ✅ verified 2026-04-19 — `lnfill/LNFill_ping_cron.sh:L17` (`192.168.203.26  # gs-csw`)
| gs-cse | .42 | — | Collector box south-east | ✅ verified 2026-04-19 — `lnfill/LNFill_ping_cron.sh:L16` (`192.168.203.42  # gs-cse`)
| gs-cne | .88 | — | Collector box north-east | ✅ verified 2026-04-19 — `lnfill/LNFill_ping_cron.sh:L14` (`192.168.203.88  # gs-cne`)
| gs-cnw | .149 | — | Collector box north-west | ✅ verified 2026-04-19 — `lnfill/LNFill_ping_cron.sh:L15` (`192.168.203.149  # gs-cnw`)
| banyan | .167 | Windows 11 | Windows in data room | ✅ verified 2026-04-19 — wiki.anl.gov/gsdaq/Computers_and_networks
| lnfill IOC | .121 | — | VME IOC for LN valves/sensors |
| ln2con | .148 | Fedora 12 | lnfill IOC boot host |
| pi5-lnfill | .58 | Debian 13 | LN fill cron + HPGe temp push |
| spark-ca9f (DGX Spark) | .132 | Linux (arm64) | General DGS host — **this machine** (replaced pi5-dgs 2026-04-15) |
| pi5-dgs | .2 | Debian 13 | Ryan's former admin Pi (retired from General DGS role 2026-04-15) |
| piserver1 | .154 | Ubuntu 20.04 | ~~Pi PXE server~~ **RETIRED** — confirmed by Ryan 2026-05-18 | ✅ verified 2026-04-19 — wiki.anl.gov/gsdaq/Computers_and_networks
| gs-pdu-north | .224 | — | Power strip north |
| gs-pdu-south | .225 | — | Power strip south |
| fs2.onenet | .71 | — | NFS/tftp server for collector Pis | ✅ verified 2026-04-19 — `collectorboxpi/README.md:L287` (`nfsroot=192.168.203.71:/mnt/vol1/fs2/nfs/piserver/...`)
| Einstor | .1 | — | DHCP server (ANL-managed) | ✅ verified 2026-04-19 — `collectorboxpi/README.md:L255` (`Einstor, 192.168.203.1`)

### Test Stand / Dev

| Host | IP | OS | Role |
|------|----|----|------|
| slopebox | .28 | SL 6.8 | Test stand |
| ts99 | .139 | — | Terminal server |
| vme99 | .211 | VxWorks | IOC |
| con5 | .126 | Solaris 10 | — |
| con6 | .136 | Solaris 10 | — |
| wdgs | .232 | Windows 7 | FPGA dev laptop (GammaWare, JTAG, Chipscope, ISE 14.7, IMPACT; login: `topoadmin` via `rdesktop`) ✅ verified 2026-04-27 — wiki `/gsdaq/Engineer_access_to_the_system_from_LabWindows` |

### VME IOC IPs (12 crates)

| Host | IP | Description |
|------|-----|-------------|
| ioc01 | 192.168.203.141 | MVME5500 VME processor — control and readout |
| ioc02 | 192.168.203.142 | MVME5500 VME processor — control and readout |
| ioc03 | 192.168.203.143 | MVME5500 VME processor — control and readout |
| ioc04 | 192.168.203.144 | MVME5500 VME processor — control and readout |
| ioc05 | 192.168.203.145 | MVME5500 VME processor — control and readout |
| ioc06 | 192.168.203.177 | MVME5500 VME processor — control and readout |
| ioc07 | 192.168.203.178 | MVME5500 VME processor — control and readout |
| ioc08 | 192.168.203.179 | MVME5500 VME processor — control and readout |
| ioc09 | 192.168.203.180 | MVME5500 VME processor — control and readout |
| ioc10 | 192.168.203.183 | MVME5500 VME processor — control and readout |
| ioc11 | 192.168.203.181 | MVME5500 VME processor — control and readout |
| ioc12 | 192.168.203.182 | MVME5500 VME processor — control and readout |

_Source: wiki.anl.gov/gsdaq/DAQ_system ✅ visited 2026-04-27_

**Terminal server routing:**
- `gs-ts-south` (192.168.203.186) → console access for **ioc01–ioc06**
- `gs-ts-north` (192.168.203.91) → console access for **ioc07–ioc12**

_Source: wiki.anl.gov/gsdaq/DAQ_system ✅ visited 2026-04-27_

---

## EPICS Port Assignment (DGS)

| System | CA Server | CA Repeater |
|--------|-----------|-------------|
| DGS (Gammasphere) | 5064 | 5065 | ✅ verified 2026-04-05 — `ANLDAQ/EPICS_para.sh:L45-46` |
| DFMA | 5068 | 5069 | ✅ verified 2026-04-05 — `ANLDAQ/EPICS_para.sh:L5` (comment) |
| SlopeBox | 5074 | 5075 | ✅ verified 2026-04-06 — `ioc/boot/vme99.cmd:L21-22` + `EPICS_para.sh:L36-37` |

> DUO and DXA ports are in [`overview_SmallSystem.md`](overview_SmallSystem.md).

---

## Subsystem Map

| Repo / Folder | What It Does | Key Tech |
|---------------|-------------|----------|
| `FPGA/` | FPGA firmware source (DIG/RTRG/MTRG) | VHDL, ISE 14.7, Vivado 2018.3 |
| `ioc/` | EPICS IOC config + firmware binaries | EPICS db/dbd, VxWorks boot scripts, Git LFS |
| `vxworks/` | Cross-compiler + IOC build environment | VxWorks 5.5, EPICS 3.14, asyn, sncseq |
| `ANLDAQ/` | DAQ GUI + data receiver | PyQt6, pyEPICS, C++ TCP receiver |
| `collectorboxpi/` | Collector Box soft IOC on Raspberry Pi | EPICS 7.0.10, soft IOC, autosave, SPI |
| `lnfill/` | LN filling system — fills HPGe dewars, monitors temps | Python 3, EPICS CA, InfluxDB, Discord |
| `dgs_analysis/` | Post-experiment analysis | C++ ROOT, Python Parquet |
| `snapshot_pv/` | PV snapshot + watchdog utilities | Python, pyepics |
| `DGS_SVN/` | Legacy SVN archive | Historical reference |

---

## Trigger Cycle (2 µs)

- 20 frames × 100 ns/frame = 2 µs total cycle ✅ verified 2026-04-29 — `mstr_mach.vhd:L299` (CURRENT_FRAME wraps at 20, CURRENT_WORD wraps at 5); `top.vhd:L246-247` (mclk=50 MHz → 20 ns/tick; 5 words×20 ns=100 ns/frame; 20 frames×100 ns=2 µs)
- **Upstream:** DIG discriminates → RTRG aggregates multiplicity → MTRG runs algorithms → decision
- **Downstream:** MTRG broadcasts 20-frame command → RTRG forwards → DIG accepts/rejects events
- Trigger decision latency (DIG → RTRG → MTRG → decision → RTRG → DIG): **~2–4 µs** (1–2 trigger cycles) ✅ verified 2026-04-07 — `fpga.md` End-to-End Timeline section; must complete within ~20 µs TRIG_DELAY window

---

## Firmware Versions (Current / Active)

| Board | Date | Revision | Config | Source |
|-------|------|----------|--------|--------|
| MTRG | 0x1022 | 0x04A8 | TAC2 + Trigger Hold-Off | ✅ `ioc/README.md:L26` (2026-04-08) |
| RTRG | 0x0414 | 0x260E | Old but working | ✅ `ioc/README.md:L27` (2026-04-08) |
| MDIG | 20250704 | 0x4CD8 | — | ✅ `ioc/README.md:L28` (2026-04-08) |
| SDIG | 20250704 | 0x4CD8 | — | ✅ `ioc/README.md:L29` (2026-04-08) |

---

## LN Cooling System

- 4 tanks (A, B, C, D) — Tank B pressurizes A and D ✅ verified 2026-04-16 — `lnfill.md` Physical System (source: `lnfill/README.md:L14`)
- 4 manifolds × 28 valves = 112 solenoid valves ✅ verified 2026-04-16 — `lnfill.md` Physical System (`DetValve.py:L25` + `DetMan.py:L134`)
- Fill detection: LED resistance change when LN reaches sensor
- Controlled by `LNFill_App.py` on pi5 (192.168.203.58) ✅ verified 2026-04-16 — `lnfill.md` Computers table (`LNFill_ping_cron.sh:L19`)
- Scheduled: 6am + 6pm daily; 15-min emergency fills for warm detectors ✅ verified 2026-04-18 — live pi5-lnFill crontab (`00 06,18 * * *`); README.md was stale (said 07,19)
- Monitored via InfluxDB/Grafana on DCS2; alerts via Discord
- Full details: [`lnfill.md`](lnfill.md)

---

## Raspberry Pi Camera (darekpi02)

- **Host:** `darekpi02` / `192.168.203.2` (onenet) ⚠️ unverified - source needed: IP `.2` is also listed as `pi5-dgs` (Ryan's admin Pi, retired 2026-04-15); unclear if darekpi02 shares that IP, has a separate `.2`, or if the wiki is stale. Confirm actual IP on network.
- **Purpose:** Live video streaming inside the Gammasphere area
- **Boot:** Connect Cat6 to onenet switch + USB power; keyboard/monitor optional
- **Password:** Default DGS password (not the default Raspberry Pi password)
- **Streaming software:** `motion` (installed by Kalle)
- **Start stream:**
  ```bash
  sudo service motion stop
  ./motion -c motion-mmalcam-both.conf
  ```
- **View stream:** <http://192.168.203.2:8081> (must be on onenet)
- **Stop:** Ctrl-C in stream window, then `rm /dev/shm/*` to clean temp files
- **Source:** wiki `/gsdaq/Raspberry_Pi_camera_use`

---

## Key Elog References

- InfluxDB write token: <https://elog.phy.anl.gov/GS+maintenance/39>
- Discord webhook URLs: <https://elog.phy.anl.gov/GS+maintenance/45>

---
*Split from overview.md — 2026-04-06. See also: [`overview_SmallSystem.md`](overview_SmallSystem.md).*

## Cross-References

- [`hardware_architecture.md`](hardware_architecture.md) — Detailed hardware breakdown: DuoGe vs DGS, signal chain, collector box
- [`fpga.md`](fpga.md) — FPGA firmware overview: 3-tier hierarchy, signal flow, trigger cycle
- [`ioc.md`](ioc.md) — EPICS IOC configuration, firmware versions, boot scripts
- [`lnfill.md`](lnfill.md) — Liquid nitrogen cooling system
- [`ANLDAQ.md`](ANLDAQ.md) — DAQ GUI, TCP receiver, run control
- [`overview_SmallSystem.md`](overview_SmallSystem.md) — DuoGe (DUO) and X-Array (DXA) small system overview
