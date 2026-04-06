# DGS System Overview — Gammasphere (Full System)

> See also: `overview_SmallSystem.md` for DuoGe (DUO) and X-Array (DXA) small systems.

## What is DGS?

The **Digital Gamma-ray Spectrometer (DGS)** is a large-scale nuclear physics detector system at **Argonne National Laboratory (ANL)**. It detects and digitizes gamma rays from an array of up to 110 high-purity germanium (HPGe) detectors arranged in the Gammasphere geometry, each cooled by liquid nitrogen.

DGS is a full software+firmware+hardware stack:
- Real-time FPGA trigger decisions at 2 µs cycle time
- EPICS-based control over VME hardware
- Multi-threaded TCP data acquisition
- Automated LN2 cooling system
- Post-experiment ROOT/Parquet analysis pipeline

---

## Signal Chain (Detector → Data)

```
HPGe + BGO DETECTORS (up to 110 GS holes)
  HPGe crystal (LN2 cooled) + up to 7 BGO Compton shield segments
  │ preamp signals (single-ended)
  ▼
SLOPE BOX (1 per detector)
  Generates Ge bias HV (~3500V) + BGO bias HV (~550–800V)
  Multiplexed ADC: temp, actual HV, PSU monitoring
  │ analog signals + 48VDC
  ▼
SBX — Slope Box Extension (1 per detector)
  Single-ended → differential conversion
  BGO sum signal + BGO pattern discrimination
  GS_ID dongle (identifies GS hole number)
  Pickoff Card: routes signals to correct DIG channels
  │ DVI-I cable (signals + power + comms)
  ▼
COLLECTOR BOX — CollectorBox_RevA (4 boxes × 28 detectors)
  Aggregates SBX signals; routes to VME crate digitizers
  Controlled by Raspberry Pi soft IOC (collectorboxpi/)
  EPICS PVs: HV, temp, BGO, FET bias, fan speed (1437 PVs/det)
  │ differential analog → VME
  ▼
DIG — Digitizer (10 ch, Spartan-3 XC3S5000)
  14-bit ADC at 100 MHz; per-channel LED/CFD discriminator
  Buffers accepted events in FIFO
  │ SERDES (hit pattern)        │ VME bus (event data)
  ▼                             ▼
RTRG — Router Trigger         MVME5500 VME IOC (VxWorks)
  Virtex-4, aggregates 8 DIGs    DMA readout of DIG FIFOs
  Computes X/Y multiplicity      EPICS IOC (ioc/)
  │ SERDES (Link L)
  ▼
MTRG — Master Trigger (Virtex-4 / KU060)
  Runs trigger algorithms; 2 µs cycle (20 frames)
  TDC ~1 ns resolution
  │ trigger decision → RTRG → DIG
  ▼
ANLDAQ — PyQt6 GUI + tcpReceiverMT
  Run control, board config, live monitor, binary file writing
  │ raw binary files
  ▼
dgs_analysis — fastEventConstructor (ROOT) + parquet_pysort
```

---

## Hardware Inventory

### VME Boards

| Board | Chip | Count | Role |
|-------|------|-------|------|
| DIG (MDIG/SDIG) | Spartan-3 XC3S5000 + XC3S400 (VME) | up to 64 | 10-ch digitizer |
| RTRG | Virtex-4 XC4VLX80 + XC3S400 (VME) | up to 8 | Router trigger |
| MTRG | XC4VLX80 or KU060 + XC3S400 (VME) + XC95144XL (CPLD) | 1 | Master trigger |
| MVME5500 | PowerPC 7455 | 12 | IOC computer (one per VME crate) |

### Per-Detector Hardware

| Hardware | Count | Role |
|----------|-------|------|
| HPGe detector | up to 110 | Gamma-ray sensing crystal, cooled by LN2 |
| BGO shield | up to 7 per detector | Compton suppression scintillator segments |
| Slope Box | 1 per detector | Generates Ge+BGO bias HV; multiplexed ADC for temp/HV monitoring |
| SBX (Slope Box Extension) | 1 per detector | Signal conditioning; BGO sum + pattern; GS_ID dongle; 48VDC distribution |
| Pickoff Card | 1 per SBX | Routes signals to correct DIG channels; BGO HV demand control |
| Collector Box (CollectorBox_RevA) | 4 total | Hub for 28 detectors each; interfaces to digitizers + Pi soft IOC |
| DVI-I cable | 1 per detector | Carries analog signals + power + comms from SBX → Collector Box |

### Computers (192.168.203.x)

| Host | IP | OS | Role |
|------|----|----|------|
| DCS2 | .56 | Ubuntu 24.04 | Data analysis, InfluxDB, Grafana |
| dgs1 | .122 | SL 6.8 | Main DAQ |
| dgs2 | .123 | Rocky 8.7 | Main DAQ (4TB SSD) |
| dgs4 | .125 | SL 7.9 | — |
| dgs6 | .184 | Rocky 8.7 | — |
| gs-ts-south | .186 | — | Terminal server south (even GS IDs) |
| gs-ts-north | .91 | — | Terminal server north (odd GS IDs) |
| gs-csw | .26 | — | Collector box south-west |
| gs-cse | .42 | — | Collector box south-east |
| gs-cne | .88 | — | Collector box north-east |
| gs-cnw | .149 | — | Collector box north-west |
| banyan | .167 | Windows 11 | Windows in data room |
| lnfill IOC | .121 | — | VME IOC for LN valves/sensors |
| ln2con | .148 | Fedora 12 | lnfill IOC boot host |
| pi5-lnfill | .58 | Debian 13 | LN fill cron + HPGe temp push |
| pi5-dgs | .2 | Debian 13 | Ryan's admin Pi — **this machine** |
| piserver1 | .154 | Ubuntu 20.04 | Pi PXE server |
| gs-pdu-north | .224 | — | Power strip north |
| gs-pdu-south | .225 | — | Power strip south |
| fs2.onenet | .71 | — | NFS/tftp server for collector Pis |
| Einstor | .1 | — | DHCP server (ANL-managed) |

### Test Stand / Dev

| Host | IP | OS | Role |
|------|----|----|------|
| slopebox | .28 | SL 6.8 | Test stand |
| ts99 | .139 | — | Terminal server |
| vme99 | .211 | VxWorks | IOC |
| con5 | .126 | Solaris 10 | — |
| con6 | .136 | Solaris 10 | — |
| wdgs | .232 | Windows | FPGA dev laptop |

### VME IOC IPs (12 crates)
192.168.203.141, .142, .143, .144, .145, .177, .178, .179, .180, .181, .182, .183

---

## EPICS Port Assignment (DGS)

| System | CA Server | CA Repeater |
|--------|-----------|-------------|
| DGS (Gammasphere) | 5064 | 5065 |
| DFMA | 5068 | 5069 |
| SlopeBox | 5074 | 5075 |

> DUO and DXA ports are in `overview_SmallSystem.md`.

---

## Subsystem Map

| Repo / Folder | What It Does | Key Tech |
|---------------|-------------|----------|
| `FPGA/` | FPGA firmware source (DIG/RTRG/MTRG) | VHDL, ISE 13.4/14.7, Vivado 2018.3 |
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

- 20 frames × 100 ns/frame = 2 µs total cycle
- **Upstream:** DIG discriminates → RTRG aggregates multiplicity → MTRG runs algorithms → decision
- **Downstream:** MTRG broadcasts 20-frame command → RTRG forwards → DIG accepts/rejects events
- End-to-end latency: ~1.3–1.5 µs

---

## Firmware Versions (Current / Active)

| Board | Date | Revision | Config |
|-------|------|----------|--------|
| MTRG | 0x1022 | 0x04A8 | TAC2 + Trigger Hold-Off |
| RTRG | 0x0414 | 0x260E | Old but working |
| MDIG | 20250704 | 0x4CD8 | — |
| SDIG | 20250704 | 0x4CD8 | — |

---

## LN Cooling System

- 4 tanks (A, B, C, D) — Tank B pressurizes A and D
- 4 manifolds × 28 valves = 112 solenoid valves
- Fill detection: LED resistance change when LN reaches sensor
- Controlled by `LNFill_App.py` on pi5 (192.168.203.58)
- Scheduled: 7am + 7pm daily; 15-min emergency fills for warm detectors
- Monitored via InfluxDB/Grafana on DCS2; alerts via Discord
- Full details: `lnfill.md`

---

## Key Elog References

- InfluxDB write token: <https://elog.phy.anl.gov/GS+maintenance/39>
- Discord webhook URLs: <https://elog.phy.anl.gov/GS+maintenance/45>

---
*Split from overview.md — 2026-04-06. See also: `overview_SmallSystem.md`.*
