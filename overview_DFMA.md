# DFMA — Digital Fragment Mass Analyzer

_Created: 2026-05-18 — confirmed by Ryan_

## Overview

The DFMA (Digital Fragment Mass Analyzer) is a separate DAQ system from DGS/Gammasphere. It shares the same OneNet network (192.168.203.x) and uses the same DGS digitizer hardware and EPICS IOC architecture, but operates with its own EPICS port space, IOC crates, and DAQ computers.

## EPICS

| Parameter | Value |
|-----------|-------|
| CA Server Port | 5068 |
| CA Repeater Port | 5069 |
| GEB Type ID | 16 |
| EPICS Base | 3.14.12.8 (shared `/global/base/` install) |

(DGS/Gammasphere uses 5064/5065)

## Computers

### dgsx — 192.168.203.32

| Item | Value |
|------|-------|
| Hostname | dgsx.onenet |
| OS | Scientific Linux 7.9 (Nitrogen) |
| Kernel | 3.10.0-1160.80.1.el7 (x86_64) |
| CPU | Intel Core i5-2400 @ 3.10GHz (4 cores) |
| RAM | 3.6 GB |
| Storage | 466 GB HDD — / (98G), /export/home/data1 (239G), /home (50G), swap (68G) |
| NFS mount | fs2.onenet:/mnt/vol5/atlasdata → /dk/fs3a (264 TB) |
| Network | em1: 192.168.203.32/24 |
| EPICS | `/global/base/base-3.14.12.8/` (caget/caput available) |
| EDM | `/global/devel/extensions/dgs1/` (display manager for IOC GUIs) |
| Uptime | 27 days (as of 2026-05-18) |

**Listening ports (external):**

| Port | Service |
|------|---------|
| 21 | FTP |
| 22 | SSH |
| 111 | rpcbind |
| 5068 | EPICS CA repeater (DFMA) |

Localhost only: 25 (postfix), 53 (libvirt DNS), 631 (CUPS), 6010–6016 (X11 forwarding)

### dgs3 — 192.168.203.124

| Item | Value |
|------|-------|
| Hostname | dgs3 |
| OS | Rocky Linux 8.10 (Green Obsidian) |
| Kernel | 4.18.0-553.8.1.el8_10 (x86_64) |
| CPU | Intel Core i5-7500 @ 3.40GHz (4 cores) |
| RAM | 30 GB |
| Storage | 466 GB HDD — / (100G), /export/home/data1 (229G), /home (100G), swap (32G) |
| NFS | NFS **server** running (exports data, port 2049) |
| Network | eno1: 192.168.203.124/24 |
| EPICS | `/global/base/base-3.14.12.8/` (shared, same as dgsx) |
| EDM | Same setup as dgsx |

**Listening ports (external):**

| Port | Service |
|------|---------|
| 22 | SSH |
| 111 | rpcbind |
| 512–514 | rsh/rlogin/rexec (legacy remote shell) |
| 2049 | NFS server |
| 20048 | NFS mountd |

Localhost only: 25 (postfix), 53 (libvirt DNS), 631 (CUPS), 6010 (X11 forwarding)

### Key Differences

| | dgsx | dgs3 |
|---|------|------|
| OS | SL 7.9 (older) | Rocky 8.10 (newer) |
| RAM | 3.6 GB | 30 GB |
| CPU | i5-2400 (2011) | i5-7500 (2017) |
| NFS role | Client (mounts fs2) | Server (exports data) |
| FTP | Yes (port 21) | No |
| DFMA CA repeater | Running (port 5068) | Not running |

## Shared EPICS / Software Environment

Both machines use a shared `/global/` tree (likely NFS-mounted from fs2.onenet):

```
/global/base/base-3.14.12.8/          — EPICS base
/global/devel/extensions/dgs1/        — EDM display manager
/global/devel/gretTop/9-22/           — GRETINA/DGS IOC apps (gretDig, gretTrig, gretVME)
/global/devel/supTop/31410/           — support top (procServ, caPython, caMonitor, etc.)
/global/devel/gtreceiver/             — data receiver binaries
```

Setup via `~/setupepics` (sourced manually or from `.bashrc`).

## VME IOC Crates (14 crates: .190–.203)

| IP | Role |
|----|------|
| 192.168.203.190 | Trigger IOC — no digitizer connected |
| 192.168.203.191 | Digitizer IOC |
| 192.168.203.192 | Digitizer IOC |
| 192.168.203.193 | Digitizer IOC |
| 192.168.203.194 | Digitizer IOC |
| 192.168.203.195 | Digitizer IOC |
| 192.168.203.196 | Digitizer IOC |
| 192.168.203.197 | Digitizer IOC |
| 192.168.203.198 | Digitizer IOC |
| 192.168.203.199 | Digitizer IOC |
| 192.168.203.200 | Digitizer IOC |
| 192.168.203.201 | Digitizer IOC |
| 192.168.203.202 | Digitizer IOC |
| 192.168.203.203 | Digitizer IOC |

13 digitizer IOCs + 1 trigger IOC = 14 crates total.

## DAQ Scripts

Located in `ANLDAQ/tcpReceiver/`:

- `start_run_dfma.sh` — start DFMA DAQ run (sets EPICS ports 5068/5069, opens tcpReceiver per digitizer IOC)
- `stop_run_dfma.sh` — stop DFMA DAQ run, kill receivers

The start script has the IOC IP list configurable at the top (`IPList` variable). Currently only .192 and .193 are active; the full list (.192–.203) is commented out.

## Relationship to DGS

- Same network (192.168.203.x)
- Same hardware (DGS digitizer boards, MVME5500 IOC processors)
- Same EPICS IOC software (templates, drivers)
- Same shared `/global/` software tree
- **Different** EPICS port space (5068 vs 5064)
- **Different** DAQ computers (dgsx/dgs3 vs dgs2)
- Can run simultaneously with Gammasphere DAQ

## SSH Access

From spark-ca9f:
```
ssh dgs@dgsx    # 192.168.203.32
ssh dgs@dgs3    # 192.168.203.124
```
Same credentials as dgs2 (stored in `workspace/secrets/dgs.env`).

## See Also

- [overview_DGS.md](overview_DGS.md) — Gammasphere system overview
