# GS DAQ Wiki — Index & Key Notes

**URL:** https://wiki.anl.gov/gsdaq  
**Status:** Official ANL wiki for Gammasphere/DGS DAQ. May be outdated — check page edit dates when relying on specifics.

---

## Wiki Structure

### Understanding Gammasphere Interactively
- `/gsdaq/Interactive_Image_Map` — clickable block diagram of the system
- `/gsdaq/Gammasphere` — overview of GS hardware and DAQ
- `/gsdaq/DGS_Commander_EDM_Screens` — EDM GUI screens for run control
- `/gsdaq/DAQ_system` — full DAQ system description with network table

### Digital Gammasphere Upgrade Project
- `/gsdaq/Digital_Gammasphere_Upgrade_Project` — top-level page
- `/gsdaq/User_Guides_for_Experiments` — experiment user guides
- `/gsdaq/Advanced_User_Guides` — DGS Commander EPICS, GEBSort, firmware advanced guide, triggers & digitizers
- `/gsdaq/Expert_Documentation` — ANL Digitizer Firmware for Experts, piserver instructions

### Hardware Pages
- `/gsdaq/Gammasphere_Detectors` — HPGe + BGO, segmented vs non-segmented
- `/gsdaq/Detector_Signals` — GeCenter, GeSides, BGOSum, BGOPattern, Clean/Dirty events, Electric Honeycomb
- `/gsdaq/Collector_Box` — collector box hardware, layout, stripes, electric honeycomb, replaces VXI
- `/gsdaq/VME_Crates` — VME crate info
- `/gsdaq/Digitizers` — digitizer hardware
- `/gsdaq/Triggers` — trigger modules
- `/gsdaq/IOC` → `/gsdaq/Digitizer_IOCs` — IOC info
- `/gsdaq/The_Slope_Box` — slope box hardware
- `/gsdaq/The_Slope_Box_Extension` — SBX hardware
- `/gsdaq/Liquid_Nitrogen` / `/gsdaq/LN_system` — LN2 system
- `/gsdaq/Preamplifier` — preamp
- `/gsdaq/The_Pickoff_Card` — pickoff card

### Support Devices
- `/gsdaq/CrateAndBoardMapping` — VME crate naming (VME01–VME11 + VME32=trigger crate)
- `/gsdaq/Network_Accessible_Power_Control_Units_of_DGS` — PDU control
- MyRIAD user manual PDF: `https://wiki.anl.gov/wiki_gsdaq/images/4/40/MyRIAD_User_Manaual.pdf`

### On-Site Experts
- `/gsdaq/Building_the_Entire_System`
- `/gsdaq/Linking_Systems_Together`
- `/gsdaq/Updating_Firmware_in_Digitizers_and_Triggers`
- `/gsdaq/IOC_Code_Design`

---

## Key Facts from Wiki (2026-04-05 crawl)

### System Overview
- **110 HPGe detectors**, each Compton-shielded by 6 side BGOs + 1 back plug BGO
- **4 collector boxes** (NE, NW, SE, SW), each serving ~30 detectors
- Even GS numbers = south hemisphere; odd GS numbers = north hemisphere
- **12 VME crates** (VME01–VME12; wiki says VME11 + VME32, but Ryan's system has 12 named ioc01–ioc12)
- DAQ fully digital since 2023 (Gammasphere Upgrade Project 2019–2023)

### Collector Box
- Replaces the old VXI crate system (which needed a dedicated shack + 100+ heavy 60-ft cables)
- 6 FPGAs inside, each handling 5 detectors in units called **"stripes"**
- Routes 4 differential signals per detector (Ge center, BGO sum + 2 configurable)
- Configurable channels: Ge side, BGO hit pattern, or copy of Ge center at fixed energy range (8 or 20 MeV)
- Has **Electric Honeycomb** FPGA — designed to compute cross-detector BGO suppression, sends to dedicated Router via fiber
- Electric Honeycomb improves Compton suppression by up to 10% @ 1 MeV **⚠️ Still in development — not yet realized/operational (as of 2026-04-05, per Ryan)**
- 48V power to SBX units via collector box

### Detector Signals
| Signal | What It Is |
|--------|-----------|
| **GeCenter** | Center contact of the cylindrical Ge crystal — primary gamma-ray energy signal |
| **GeSide A/B** | Outer contact(s) of Ge — segmented detectors have two sides; non-segmented have one |
| **BGOSum** | Sum of all 7 BGO signals (6 side + 1 back) — used for Compton suppression |
| **BGOPattern** | Discriminator pattern from individual BGO segments — designed for Electric Honeycomb (not yet operational) |

**Clean event:** Ge fires, BGO does NOT fire → full-energy deposit  
**Dirty event:** Ge + BGO both fire within time window → Compton scatter → Router marks as dirty/veto  
**Electric Honeycomb:** Cross-detector BGO suppression using BGOPattern — ⚠️ **still in development, not yet realized** (wiki describes design intent; confirmed by Ryan 2026-04-05)

### Network Nodes (from wiki, confirmed matches our MEMORY.md)
| Name | IP | Role |
|------|-----|------|
| ioc01–ioc05 | 192.168.203.141–145 | MVME5500 IOC processors |
| ioc06–ioc12 | 192.168.203.177–183 | MVME5500 IOC processors |
| piserver1 | 192.168.203.154 | Collector box boot host |
| gs-cne | 192.168.203.88 | NE collector box Pi |
| gs-cnw | 192.168.203.149 | NW collector box Pi |
| gs-cse | 192.168.203.42 | SE collector box Pi |
| gs-csw | 192.168.203.26 | SW collector box Pi |
| gs-pdu-north | 192.168.203.224 | North hemisphere PDU |
| gs-pdu-south | 192.168.203.225 | South hemisphere PDU |
| gs-ts-north | 192.168.203.91 | Terminal server ioc07–ioc12 |
| gs-ts-south | 192.168.203.186 | Terminal server ioc01–ioc06 |
| lnfill | 192.168.203.121 | LN2 VME IOC |
| ln2con | 192.168.203.148 | LN2 Linux boot host |

### VME Crate Naming (from CrateAndBoardMapping)
- 12 VME crates = VME01–VME11 + "VME32" (trigger crate)
- Each apparent rack crate = 3 separate 7-slot backplanes with common power supply
- VME01, VME02, VME03 = three backplanes in top-right rack
- Crate mapping spreadsheet: `DGS_docs/DGS_System_Documentation/System/DGS crate mapping.xlsx`

### DAQ Data Flow
1. Preamp → Slope box → SBX (converts to differential) → Collector box → Digitizers
2. Digitizers: FIFO → IOC (inLoop → outLoop → MiniSender/VxWorks sender)
3. `tcpReceiverMT` (on DCS2) connects to each IOC via **TCP** (`SOCK_STREAM`, port 9001) and pulls data

> ⚠️ **Wiki says "UDP packets" — this is WRONG.** Verified exhaustively from source code (2026-04-05, confirmed by Ryan):
> - **VxWorks IOC** (`SendReceiveSupport.c`): `socket(AF_INET, SOCK_STREAM, 0)` → TCP server on port 9001; uses `bind()` + `listen()` + `accept()` — these calls don't exist in UDP
> - **ANLDAQ receiver** (`receiver.h`): `socket(AF_INET, SOCK_STREAM, 0)` + `connect()` → TCP client connecting to IOC:9001
> - **Old SVN receiver** (`gtReceiver6.c`, `dgsReceiver.cpp`): same `SOCK_STREAM` pattern — TCP all the way back
> - `SOCK_STREAM` = TCP by definition; UDP = `SOCK_DGRAM`. IOC uses `listen()`/`accept()` which are TCP-only calls.
> - `tcpReceiverUDP.cpp` is a separate experimental variant, not the production receiver
> - Full code evidence: see **`ANLDAQ.md` § "TCP Protocol — Verified from Source Code"**
> - Wiki UDP claim written by J. Anderson, March 2023 — never accurate

### Piserver (older system, boxes 202–204)
- Runs on Ubuntu machine `piserver1` (192.168.203.154) as user `dgs`
- Start: `sudo piserver &`
- Network-booted Pis share `/home/rpdgs` (NFS home directory)
- Pis get IP from onenet DHCP (not piserver); register via `onereg.phy.anl.gov`
- SoftIOC starts automatically on SSH login (checks for existing instance first)

---

## Trigger & Digitizer Setup Procedure (from wiki)

**Key principle:** Initialize top-down (Master → Router → Digitizer to lock timestamps), then configure bottom-up (Digitizer → Router → Master) to get triggers working.

### Step-by-step (DGS)

1. **Digitizers → transmit real discriminator data to Routers**
   - Should happen by default; watch for accidentally disabled SERDES or SYNC-pattern-instead-of-data bit
   - Set discriminator bit assertion time per channel (e.g. 100 ns) — use a script

2. **Verify Routers lock onto all digitizer data**
   - Use Router counters to confirm no channel dropping in/out of lock
   - Ignore X-plane/Y-plane map registers (DFMA only)

3. **Load `CLEAN_DIRTY_CONTROL` register on each Router (addr `0x8CC`)**
   - Bits 7:4 = Y sum source; bits 3:0 = X sum source
   - Values: `0001` = clean sum, `0010` = dirty sum, `0100` = module sum

4. **Load `TSCATTER_DELAY` register on each Router (addr `0x08C8`)**
   - Time window after Ge hit during which BGO hit marks event as dirty (Scattering Time)
   - Typical value: **30 counts = 600 ns** (1 count = 20 ns)

5. **Verify Router data using FIFOs**
   - Channel FIFOs: check digitizer bits 9:0
   - Monitor FIFO 4: check X sum data being sent to Master

6. **Repeat for every Router**

7. **Set up Master Trigger**
   - Verify Master receives data from all Routers via channel FIFOs
   - Common error: forgetting to clear the SYNC bit in `LINK_LRU_CTRL` of the Router
   - Set multiplicity thresholds for X and Y sums
   - Enable trigger algorithms via Trigger Mask register (`SumX`, `SumY`, etc.)
   - SumX + SumY enabled simultaneously = OR trigger
   - If both fire at same timestamp, two triggers issued but likely select same event

### DFMA differences
- Step 3: must load X-plane and Y-plane mapping registers (not used in DGS)
- Step 3: load `CLEAN_DIRTY_CONTROL` = 0
- Step 4 (TSCATTER_DELAY): skip for DFMA

---

## Pages Not Yet Browsed (worth checking)
- `/gsdaq/Triggers_and_digitizers` — trigger/digitizer setup guide ✅ visited 2026-04-05
- `/gsdaq/Data_formats` — data format details
- `/gsdaq/Firmware_documentation` — firmware docs list
- `/gsdaq/Typical_DGS_run_procedures` — run procedure reference
- `/gsdaq/Some_problems_and_their_solutions` — troubleshooting
- `/gsdaq/ANL_Digitizer_Firmware_for_Advanced_Users` — advanced firmware guide
- `/gsdaq/High_Purity_Germanium_(HPGe)_and_BGO` — HPGe + BGO details

---

*Created: 2026-04-05 (initial crawl)*
