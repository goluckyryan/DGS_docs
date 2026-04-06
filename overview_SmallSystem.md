# DGS Small Systems Overview — DuoGe (DUO) and X-Array (DXA)

> These are smaller, standalone HPGe DAQ systems that share the onenet network
> (192.168.203.x) and the same ANLDAQ software as the full Gammasphere DGS,
> but run independently with their own EPICS CA ports and VME IOCs.
>
> See `overview_DGS.md` for the full Gammasphere system.

---

## DuoGe (DUO) — 2-HPGe Detector System

**Purpose:** Small 2-detector germanium DAQ, typically used for standalone experiments or detector testing.

### Computers

| Host | IP (192.168.203.x) | OS | Role |
|------|--------------------|----|------|
| tangerine | .78 | Rocky Linux 8 | DAQ host / FTP server for IOC |
| IOC (vme66) | .81 | VxWorks | VME IOC |
| sbxh3 | .164 | — | Raspberry Pi for SBX |
| sbxcc | .158 | — | Raspberry Pi for SBX |

### EPICS Ports

| System | CA Server | CA Repeater |
|--------|-----------|-------------|
| DuoGe (DUO) | 5080 | 5081 |
| DUB | 5078 | 5079 |

### Notes
- Uses the same DIG/RTRG/MTRG FPGA hardware chain as full DGS, scaled down
- SBX Raspberry Pis (sbxh3, sbxcc) run the `collectorboxpi/` soft IOC
- IOC boots from tangerine (FTP server), runs as vme66

---

## X-Array (DXA)

**Purpose:** Auxiliary germanium array, used alongside or independently of Gammasphere.

### EPICS Ports

| System | CA Server | CA Repeater |
|--------|-----------|-------------|
| DXA | 5072 | 5073 |

### Notes
- Shares onenet (192.168.203.x) network with DGS and DUO
- EPICS CA port separation provides logical isolation from full DGS (5064/5065)
- Specific computer IPs not yet documented — add here when known

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
