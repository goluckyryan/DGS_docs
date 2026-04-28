# DGS Small Systems Overview — DuoGe (DUO) and X-Array (DXA)

Stability: C2 - Active / semi-stable

> These are smaller, standalone HPGe DAQ systems that share the onenet network
> (192.168.203.x) and the same ANLDAQ software as the full Gammasphere DGS,
> but run independently with their own EPICS CA ports and VME IOCs.
>
> See `overview_DGS.md` for the full Gammasphere system.

---

## Table of Contents

1. [Signal Chain (Small Systems)](#signal-chain-small-systems)
2. [DuoGe (DUO) — 2-HPGe Detector System](#duoge-duo--2-hpge-detector-system)
   - [Computers](#computers)
   - [EPICS Ports](#epics-ports)
   - [DUB System Computers](#dub-system-computers)
   - [Notes](#notes)
3. [X-Array (DXA)](#x-array-dxa)
   - [Network / Computers](#network--computers)
   - [Notes](#notes-1)
4. [Shared Infrastructure](#shared-infrastructure)

---

## Signal Chain (Small Systems)

> Small systems (DUO, DXA) do **not** use a Collector Box.
> The SBX Raspberry Pi IOC is an older version in `DGS_SVN/dgs/SlopeBoxInterface/RaspberryPi/LocalEpics/` — **not** `collectorboxpi/` (which is for the full GS Collector Box Pi). ✅ verified 2026-04-16 — `LocalEpics/db/PickoffPVs.db` uses custom `PickoffLocalSerial` SPI driver.

```
┌──────────────────────────────────────────────────────────────────┐
│           HPGe + BGO DETECTORS (small count)                     │
│  HPGe crystal (LN2 cooled) + BGO Compton shield segments         │
└──────────────────────────┬───────────────────────────────────────┘
                           │ preamp signals (single-ended)
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│               SLOPE BOX  (1 per detector)                        │
│  Generates Ge bias HV + BGO bias HV                              │
│  Multiplexed ADC: temp, actual HV, PSU monitoring                │
└──────────────────────────┬───────────────────────────────────────┘
                           │ analog signals + 48VDC
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│         SBX — Slope Box Extension  (1 per detector)              │
│  Single-ended → differential conversion                          │
│  BGO sum signal + BGO pattern discrimination                     │
│  GS_ID dongle; Pickoff Card for channel routing                  │
│  ⚠️ Controlled by Raspberry Pi IOC (older version, not in repo)  │
└──────────────────────────┬───────────────────────────────────────┘
                           │ differential signals → directly to VME
                           │ (NO Collector Box)
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                    DIG — Digitizer (fpga/)                       │
│  Same hardware as full DGS; 10-ch, 14-bit ADC at 100 MHz         │
│  Buffers accepted events in FIFO                                 │
└────────────┬───────────────────────────────┬─────────────────────┘
             │ SERDES                        │ VME bus (event data)
             ▼                               ▼
┌─────────────────────────────┐  ┌────────────────────────────────┐
│  RTRG — Router Trigger      │  │  MVME5500 VME IOC (VxWorks)    │
│  Virtex-4 XC4VLX80          │  │  DMA readout of DIG FIFOs      │
│  Aggregates DIGs            │  │  EPICS IOC (ioc/)              │
└─────────┬───────────────────┘  └───────────┬────────────────────┘
          │ SERDES (Link L)                  │ EPICS CA + TCP data
          ▼                                  ▼
┌─────────────────────────────┐  ┌────────────────────────────────┐
│  MTRG — Master Trigger      │  │  ANLDAQ (anldaq/)              │
│  Virtex-4 / KU060           │  │  commander.py — PyQt6 GUI      │
│  Trigger algorithms         │  │  Run control + live monitor    │
│  2 µs cycle                 │  │  tcpReceiverMT — binary files  │
└───────────┬─────────────────┘  └───────────┬────────────────────┘
            │ trigger → RTRG → DIG           │ raw binary files
                                             ▼
                                 ┌────────────────────────────────┐
                                 │  dgs_analysis/                 │
                                 │  fastEventConstructor (ROOT)   │
                                 │  parquet_pysort (Parquet)      │
                                 └────────────────────────────────┘
```

---

## DuoGe (DUO) — 2-HPGe Detector System

**Purpose:** Small 2-detector germanium DAQ, typically used for standalone experiments or detector testing.

### Computers

| Host | IP (192.168.203.x) | OS | Role |
|------|--------------------|----|------|
| tangerine | .78 | Rocky Linux 8 | DAQ host / FTP server for IOC | ✅ verified 2026-04-19 — `ANLDAQ/ioc/README.md:L33,L46` |
| IOC (vme66) | .81 | VxWorks | VME IOC | ✅ verified 2026-04-19 — `ANLDAQ/ioc/README.md:L45,L50` |
| sbxh3 | .164 | — | Raspberry Pi for SBX | ✅ verified 2026-04-19 — `snapshot_pv/EPICS_env.sh:L28` (IP .164, comment sbxh3); cross-ref `sbx.md` |
| sbxcc | .158 | — | Raspberry Pi for SBX | ✅ verified 2026-04-19 — `snapshot_pv/EPICS_env.sh:L28` (IP .158, comment sbxcc); cross-ref `sbx.md` |

### EPICS Ports

| System | CA Server | CA Repeater |
|--------|-----------|-------------|
| DuoGe (DUO) | 5080 | 5081 | ✅ verified 2026-04-17 — `ANLDAQ/EPICS_para.sh:L16-17` |
| DUB | 5078 | 5079 | ✅ verified 2026-04-17 — `ANLDAQ/EPICS_para.sh:L8` (comment) |

### DUB System Computers

| Name | IP (.203.x) | OS | Function |
|------|-------------|-----|----------|
| dub1 | .95 | Rocky 8.7 | DUB DAQ host |
| dub1ts | .96 | — | Terminal server |
| dub1ioc1 | .97 | VxWorks | VME IOC 1 |
| dub1ioc2 | .98 | VxWorks | VME IOC 2 |
| dub1ioc3 | .99 | VxWorks | VME IOC 3 |
| mpod1 | .221 | — | Iseg MPOD HV controller |
| xiatest | .25 | — | XIA test node |
| xiatemp | .31 | — | XIA temp node |
| xia14-250-27-lan | .219 | — | XIA 14-250 digitizer (LAN) |
| xia14-250-24-0 | .223 | — | XIA 14-250 digitizer |
| xia14-250-24-1 | .228 | — | XIA 14-250 digitizer |
| uballxia1 | .234 | — | XIA node (retired?) |

✅ verified 2026-04-25 — [wiki: Computers and networks](https://wiki.anl.gov/gsdaq/Computers_and_networks) (all IPs/hostnames confirmed; added XIA test nodes)

_Source: [wiki: Computers and networks](https://wiki.anl.gov/gsdaq/Computers_and_networks) ✅ visited 2026-04-06_

### Notes
- Uses the same DIG/RTRG/MTRG FPGA hardware chain as full DGS, scaled down
- SBX Raspberry Pis (sbxh3, sbxcc) run the older legacy IOC from `DGS_SVN/dgs/SlopeBoxInterface/RaspberryPi/` — **not** `collectorboxpi/` (which is for the GS Collector Box Pi). The SBX Pi IOC uses a custom `PickoffLocalSerial` EPICS driver for SPI/GPIO communication with the Pickoff card. ✅ verified 2026-04-16 — `DGS_SVN/dgs/SlopeBoxInterface/RaspberryPi/LocalEpics/db/PickoffPVs.db:L1` (custom driver, not collectorboxpi)
- IOC boots from tangerine (FTP server), runs as vme66 ✅ verified 2026-04-20 — `ioc/README.md:L43,L50-51` (host: tangerine, target name: vme66, startup: /global/ioc/boot/vme66.cmd)

---

## X-Array (DXA)

**Purpose:** Auxiliary germanium array, used alongside or independently of Gammasphere.

### EPICS Ports

| System | CA Server | CA Repeater |
|--------|-----------|-------------|
| DXA | 5072 | 5073 | ✅ verified 2026-04-19 — `ANLDAQ/EPICS_para.sh:L23-24` |

### Network / Computers

| Name | IP (.203.x) | OS | Function |
|------|-------------|-----|----------|
| dgs8 | .185 | Rocky 8.7 | X-Array DAQ host |
| xa_ts1 | .216 | — | Terminal server |
| xa_ioc1 | .212 | VxWorks | VME IOC 1 |
| xa_ioc2 | .213 | VxWorks | VME IOC 2 |
| xa_ioc3 | .214 | VxWorks | VME IOC 3 |

_Source: [wiki: Computers and networks](https://wiki.anl.gov/gsdaq/Computers_and_networks) ✅ visited 2026-04-06_

### Notes
- Shares onenet (192.168.203.x) network with DGS and DUO
- EPICS CA port separation provides logical isolation from full DGS (5064/5065)

---

## Shared Infrastructure

All small systems share:
- **onenet** (192.168.203.x) network
- **ANLDAQ** software (`commander.py` PyQt6 GUI, `tcpReceiverMT` data receiver)
- **Same FPGA firmware** (DIG/RTRG/MTRG) as full DGS
- **fs2.onenet** (.71) NFS/tftp for Pi PXE boot
- **Einstor** (.1) DHCP server

---

*Created 2026-04-06. Split from overview.md. See also: `overview_DGS.md`.*

## Cross-References

- `knowledgeBase/overview_DGS.md` — Full Gammasphere system overview (the large system)
- `knowledgeBase/hardware_architecture.md` — Hardware breakdown comparing DuoGe vs DGS configurations
- `knowledgeBase/ANLDAQ.md` — DAQ GUI and TCP receiver; CA ports per system (DUO: 5080/5081, DXA: 5072/5073)
- `knowledgeBase/wiki_gsdaq.md` — ANL wiki index; some pages cover small system specifics
