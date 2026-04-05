# DGS System Overview

## What is DGS?

The **Digital Gamma-ray Spectrometer (DGS)** is a large-scale nuclear physics detector system developed at **Argonne National Laboratory (ANL)**. It detects and digitizes gamma rays emitted during nuclear reactions using an array of high-purity germanium (HPGe) detectors, each cooled by liquid nitrogen.

DGS is a full software+firmware+hardware stack:
- Real-time FPGA trigger decisions at 2 µs cycle time
- EPICS-based control over VME hardware
- Multi-threaded TCP data acquisition
- Automated LN2 cooling system
- Post-experiment ROOT/Parquet analysis pipeline

---

## System Architecture

### Full Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                     PHYSICAL DETECTORS                          │
│          HPGe germanium γ-ray detectors (up to 640 ch)         │
│          Each cooled by liquid nitrogen (LN) dewars             │
└──────────────────────────┬──────────────────────────────────────┘
                           │ analog signals
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    DIG — Digitizer (fpga/)                      │
│  Spartan-3 XC3S5000, ISE 14.7                                   │
│  10 channels per board, 14-bit ADC at 100 MHz                   │
│  Per-channel: delay → filter → discriminator (LED/CFD) → hit   │
│  Buffers accepted events in FIFO                                │
└───────────┬────────────────────────────────┬────────────────────┘
            │ SERDES (18-bit, 50 MHz)         │ VME bus
            │ hit pattern + energy            │ (event data readout)
            ▼                                 ▼
┌───────────────────────────┐    ┌───────────────────────────────┐
│  RTRG — Router (fpga/)    │    │  MVME5500 VME IOC             │
│  Virtex-4 XC4VLX80        │    │  VxWorks 5.5 RTOS             │
│  Aggregates 8 DIGs        │    │  gretDet.munch (vxworks/)     │
│  Computes X/Y multiplicity│    │  DMA readout of DIG FIFOs     │
└───────────┬───────────────┘    │  EPICS IOC (ioc/)             │
            │ SERDES (Link L)    │  Boot: vme66.cmd / vme99.cmd  │
            ▼                    └───────────────┬───────────────┘
┌───────────────────────────┐                    │ EPICS CA + TCP data
│  MTRG — Master Trigger    │                    ▼
│  (fpga/)                  │    ┌───────────────────────────────┐
│  Virtex-4 / KU060         │    │  ANLDAQ (anldaq/)             │
│  Runs trigger algorithms  │    │  commander.py — PyQt6 GUI     │
│  TDC ~1 ns resolution     │    │  DIG/RTR/MTRG board control   │
│  2 µs cycle (20 frames)   │    │  Run control + live monitor   │
└───────────┬───────────────┘    │  tcpReceiverMT — binary files │
            │ trigger decision   └───────────────┬───────────────┘
            └──────────────────────────────────┘ │
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
| DIG | Spartan-3 XC3S5000 + XC3S400 (VME) | up to 64 | 10-ch digitizer |
| RTRG | Virtex-4 XC4VLX80 + XC3S400 (VME) | up to 8 | Router trigger |
| MTRG | XC4VLX80 or KU060 + XC3S400 (VME) + XC95144XL (CPLD) | 1 | Master trigger |
| MVME5500 | PowerPC 7455 | multiple | IOC computer (one per VME crate) |

### Computers (all 192.168.203.x unless noted)

**Main GS Digital DAQ:**
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

**New 2 HPGe DAQ (DuoGe):**
| Host | IP | OS | Role |
|------|----|----|------|
| tangerine | .78 | Rocky Linux 8 | DAQ host / FTP server for IOC |
| IOC (vme66) | .81 | VxWorks | VME IOC |
| sbxh3 | .164 | — | Raspberry Pi for SBX |
| sbxcc | .158 | — | Raspberry Pi for SBX |

**Test Stand / Dev:**
| Host | IP | OS | Role |
|------|----|----|------|
| slopebox | .28 | SL 6.8 | Test stand |
| ts99 | .139 | — | Terminal server |
| vme99 | .211 | VxWorks | IOC |
| con5 | .126 | Solaris 10 | — |
| con6 | .136 | Solaris 10 | — |
| wdgs | .232 | Windows | FPGA dev laptop |

**Other:**
| Host | IP | OS | Role |
|------|----|----|------|
| DCS2.onenet | .56 | Ubuntu 24.04 | InfluxDB, Grafana, health checks |
| fs2.onenet | .71 | — | NFS/tftp server for collector Pis |
| Einstor | .1 | — | DHCP server (ANL-managed) |

### DGS VME IOC IPs (12 crates)
192.168.203.141, .142, .143, .144, .145, .177, .178, .179, .180, .181, .182, .183

### Terminal Servers (DGS)
192.168.203.186, 192.168.203.91

---

## EPICS Port Assignments

| System | CA Server Port | CA Repeater Port |
|--------|---------------|-----------------|
| DGS | 5064 | 5065 |
| DFMA | 5068 | 5069 |
| DXA | 5072 | 5073 |
| SlopeBox | 5074 | 5075 |
| DUB | 5078 | 5079 |
| DUO | 5080 | 5081 |

---

## Subsystem Map

| Repo / Folder | What It Does | Key Tech |
|---------------|-------------|---------|
| `fpga/` | FPGA firmware source (production) | VHDL, ISE 13.4/14.7, Vivado 2018.3 |
| `raw_FPGA/` | FPGA firmware source (raw/upstream) | Same as fpga/ |
| `ioc/` | EPICS IOC config + firmware binaries | EPICS db/dbd, VxWorks boot scripts, Git LFS |
| `vxworks/` | Cross-compiler + IOC build environment | VxWorks 5.5, EPICS 3.14, asyn, sncseq |
| `ANLDAQ/` | DAQ GUI + data receiver | PyQt6, pyEPICS, C++ TCP receiver |
| `collectorboxpi/` | Collector box HV control (Pi) | EPICS 7.0.10, soft IOC, autosave |
| `lnfill/` | LN filling system control | Python 3, EPICS CA, InfluxDB, Discord |
| `dgs_analysis/` | Post-experiment analysis | C++ ROOT, Python Parquet |
| `DGS_SVN/` | Legacy SVN archive | Historical reference |

---

## Trigger Cycle (2 µs)

```
Frame 0-19 at 50 MHz (20 ns/frame = 400 ns total... no wait: 20 frames × 100 ns ≈ 2 µs)

Upstream path (hits → decision):
  DIG discriminates → RTRG aggregates multiplicity → MTRG runs algorithms → Decision

Downstream path (decision → action):
  MTRG broadcasts 20-frame command → RTRG forwards → DIG accepts or rejects events

Latency: ~1.3–1.5 µs end-to-end
```

---

## Firmware Versions (Current / Active)

| Board | Date | Revision | Config |
|-------|------|----------|--------|
| MTRG | 0x1022 | 0x04A8 | TAC2 + Trigger Hold-Off |
| RTRG | 0x0414 | 0x260E | Old but working |
| MDIG | 20250704 | 0x4CD8 | — |
| SDIG | 20250704 | 0x4CD8 | — |

---

## LN Cooling System Summary

- 4 tanks (A, B, C, D) — Tank B pressurizes A and D
- 4 manifolds × 28 valves = 112 solenoid valves total
- Fill detection: LED resistance change when LN reaches sensor
- Controlled by `LNFill_App.py` on pi5 (192.168.203.58)
- Scheduled: 7am + 7pm daily full fills; 15-min emergency fills for warm detectors
- Monitored via InfluxDB/Grafana on DCS2; alerts via Discord

---

## Key Elog References

- InfluxDB write token: https://elog.phy.anl.gov/GS+maintenance/39
- Discord webhook URLs: https://elog.phy.anl.gov/GS+maintenance/45
