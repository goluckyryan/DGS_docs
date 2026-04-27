# Analog Gammasphere (Legacy DAQ)

Stability: C3 - Structural / stable

**Source:** https://wiki.anl.gov/gsdaq/Analog_Gammasphere  
**Last fetched:** 2026-04-25  
*(Legacy/decommissioned system — facts unlikely to change)*

## Table of Contents

- [Overview](#overview)
- [Setting Up the Analog GS DAQ From Scratch](#setting-up-the-analog-gs-daq-from-scratch)
- [Stopping the DAQ and Removing Data](#stopping-the-daq-and-removing-data)
- [Online Data Monitoring (GSSort)](#online-data-monitoring-gssort)
- [VXI Processor Boot Configuration](#vxi-processor-boot-configuration)
- [Key Differences vs. Digital Gammasphere (DGS)](#key-differences-vs-digital-gammasphere-dgs)
- [Notes](#notes)
- [Cross-References](#cross-references)

---

## Overview

Analog Gammasphere was the version of data acquisition used **prior to the Digital Gammasphere (DGS) upgrade**. It used VXI-bus processors running VxWorks, a CES-based event builder, and USB-disk based data storage via NFS export. Data monitoring was done via the `GSSort` package under ROOT.

This system is no longer in active use but is documented here for historical reference and comparison with DGS.

---

## Setting Up the Analog GS DAQ From Scratch

From a Linux box:

```
telnet gsts1 2009
```

This gives a console to the analog DAQ system. Then:

```
ctrl-x           # reboot the VME processor
</home/sga2/cur/startup_sga2.cmd
sgaInit
```

### Data Storage (USB disk)

1. Insert USB disk into docking station — automounts as e.g. `/media/20140304`
2. Export it:
   ```
   gsexportfs 20140304   # do NOT specify /media/ prefix
   ```
3. Log into `dgs1` as `dgs`:
   ```
   cd /home/sga2/cur/config
   cp <cmdfile> c1.cmd
   # edit c1.cmd for your purpose
   ```

### Example `c1.cmd` configuration file

```
# cmd file created by "savepars" command
# SAT MAY 11 03:04:24 2013
datadir gslinux1 /media/20140304/user/tltmp
newexp myexp myexp
newrun myexp 5
getimewin 3950 4030
bgotimewin 1950 2030
feradelay 8
setrecversion 1
tgecomoff 4095
tbgocomoff 0
sendto gslinux1
sendfrac 100
setmode +GC_MODE
setmode -TIME_VETO
setmode +HC_SUPPRESS
setmode +WRITE_GE_TIME
setmode -WRITE_GE_FULL
setmode -WRITE_BGO
setmode -WRITE_ALL_GE
setmode -WRITE_ALL_BGO
setmode +RF_TIMING
fileheader on
checkebevlen 1
ratealarm off
badcesdump 0
```

### Starting Acquisition

```
sga2: start
acquisition started
# Status lines show event rate and CES load:
# 5968(20) ev/s; CES 54/46%; no ehi 0.5%; alarm off @ 03/05 12:30
```

Verify data is being written:
```
ls -lt /media/20140304/user/tltmp/myexp
```

---

## Stopping the DAQ and Removing Data

On the DAQ console:
```
stop
closeall
```

On the data storage machine:
```
gsunexportfs 20140304
```

Then safely eject the USB disk via the desktop icon or verify with `df`.

---

## Online Data Monitoring (GSSort)

Data monitoring used the `GSSort` package (ROOT-based).

### Getting GSSort

```bash
svn checkout https://svn.anl.gov/repos/gs_analysis/GSSort .
# or
wget http://www.phy.anl.gov/gammasphere/doc/GSSort/src/src.tgz
tar -zxvf src.tgz
```

### Building

```bash
make clean
make -B GSSort
# Also compile GSUtil_cc.so:
rootn.exe
.L GSUtil.cc++
.q
```

### Chat file for online monitoring

Key parameters (example):
```
input net 1101          # network input from DAQ
sharedmem c1.map 200000000
startmapaddress 0xab785000
nevents 1000000000
printevents 100
gerfoffset 1000
beta 0.0
hiresdatamult 0.66666666666
```

### Running GSSort

In one terminal (rootn.exe — must keep running to preserve shared memory address):
```
.L GSUtil_cc.so
sdummyload(200000000)
# Note the startmapaddress and enter it in the chat file
```

In another terminal:
```
./GSSort -chat c1.chat
```

In rootn.exe, attach to shared memory and view spectra:
```
sload("c1.map")
update()
d1("sumehi")     # view a spectrum
```

---

## VXI Processor Boot Configuration

The VXI processors booted from **dgs6** (Scientific Linux 6.4 blade in the DGS rack, with pull-out keyboard/monitor).

> **WARNING:** If rebooting dgs6, do NOT use the latest kernel — it has network interface problems. Use the second kernel in the boot list.

### Boot parameters for VXI processors:

| Parameter | lrc1 | lrc2 | lrc3 |
|-----------|------|------|------|
| Boot device | ln | ln | ln |
| Host name | dgs6 | dgs6 | dgs6 |
| VxWorks kernel | `/vxboot/kernels/boot/niCpu030-t/vxWorks` | same | same |
| IP (ethernet) | 192.168.203.170/24 ✅ verified 2026-04-27 — wiki Analog_Gammasphere boot params block (lrc1 `inet on ethernet (e) : 192.168.203.170:ffffff00`) | 192.168.203.171/24 ✅ verified 2026-04-27 — wiki Analog_Gammasphere boot params block (lrc2) | 192.168.203.172/24 ✅ verified 2026-04-27 — wiki Analog_Gammasphere boot params block (lrc3) |
| Host IP (h) | 192.168.203.184 | 192.168.203.184 | 192.168.203.184 |
| User | vxprod | vxprod | vxprod |
| Startup script | `/vxboot/daq/boot/resm/startup.lrc1` | `.../startup.lrc2` | `.../startup.lrc3` |

- Additional VXI processor at **192.168.203.173** (lrc4) ✅ verified 2026-04-27 — wiki Analog_Gammasphere shows a 4th boot params block with `inet on ethernet (e) : 192.168.203.173:ffffff00` and `startup script (s) : /vxboot/daq/boot/resm/startup.lrc4` (page text truncated before flags/target lines but IP and script confirmed)
- Boot files served via FTP from dgs6 at `/vxboot/` — `/vxboot/` path confirmed on con6 (ln2con/startup.cmd:L4,7); dgs6 as analog VXI FTP host ✅ verified 2026-04-27 — wiki Analog_Gammasphere boot params: `host name: dgs6`, `host inet (h): 192.168.203.184` for all 4 VXI processors; wiki prose: "The VXI processors have dgs6 as their boot host."
- Processor type: `niCpu030-t` (National Instruments CPU030, 68030-based) ✅ verified 2026-04-27 — wiki Analog_Gammasphere boot params: `file name : /vxboot/kernels/boot/niCpu030-t/vxWorks` confirmed for all 4 lrc processors; NI CPU030 hardware type also confirmed via [Con6_Inventory.md:L111](../ln2con/Con6_Inventory.md) (VxWorks 5.2, labeled "VXI CPU"); kernel suffix `-t` is the analog VXI boot variant (vs. `-8` seen in ln2con, which is a different VXI chassis)

---

## Key Differences vs. Digital Gammasphere (DGS)

| Aspect | Analog GS | Digital GS |
|--------|-----------|------------|
| Event builder | CES VME hardware | Software (GEBMerge/GEBsort) |
| Data storage | USB disk via NFS export | Network (collector to RAID/NFS) |
| Monitoring | GSSort + ROOT shared memory | Online sort, EPICS PVs |
| Boot host | dgs6 (SL 6.4 blade) | DGS1 (see ioc.md) |
| Processor | VXI / 68030 (niCpu030-t) | VME / PowerPC (VxWorks) |
| Console access | `telnet gsts1 2009` | EPICS IOC shell / serial |
| Config files | `c1.cmd` / `sga2` | IOC startup scripts |

---

## Notes

- `gsts1` was the analog DAQ console server (telnet port 2009)
- `gslinux1` was a Linux data collection/storage host
- `gsexportfs` / `gsunexportfs` are custom ANL scripts for NFS-exporting USB disks
- `dgs6` IP: 192.168.203.184 ✅ verified 2026-04-25 - overview_DGS.md (confirmed from wiki.anl.gov/gsdaq/Computers_and_networks 2026-04-19)
- CES = Compact Electronics Standard (VME-based event building hardware)
- The analog system used **FERA** (Fastbus Electronics for Readout Anywhere) ADCs; `feradelay 8` sets timing ✅ verified 2026-04-25 - FERA interface confirmed on MyRIAD hardware (myriad.md; FERA_FULL/ACK/OVF/WSI/VETO signals verified from MyRIAD.vhd:L142-148)
- `getimewin` / `bgotimewin`: Ge and BGO coincidence time windows (in ns or ADC channels)
- This system is superseded; wiki page is maintained for legacy reference only

---

## Cross-References

| File | Relationship |
|------|--------------|
| `overview_DGS.md` | DGS machine overview, IP table including dgs6 |
| `ioc.md` | DGS1 as digital GS boot host (successor) |
| `myriad.md` | MyRIAD provides FERA interface for the old analog chain |
| `hardware_architecture.md` | HPGe detector hardware common to both systems |
| `nfs_layout.md` | NFS storage includes some analog-era data directories |

---

*Created: 2026-04-25 | Last reviewed: 2026-04-25*
