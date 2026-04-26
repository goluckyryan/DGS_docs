# DGS Hardware Architecture

Stability: C3 - Structural / stable

## Table of Contents

- [Minimal System — DuoGe (DUO)](#minimal-system--duoge-duo)
  - [Required Hardware](#required-hardware)
  - [MDIG vs SDIG — Master and Slave Digitizers](#mdig-vs-sdig--master-and-slave-digitizers)
  - [Minimal Crate Layout (vme66 example)](#minimal-crate-layout-vme66-example)
  - [Networking](#networking)
  - [VME Crate Structure](#vme-crate-structure)
  - [VME Backplane](#vme-backplane)
  - [Signal Chain](#signal-chain)
  - [EPICS Control](#epics-control)
- [Full System — DGS](#full-system--dgs)
  - [Physical Scale](#physical-scale)
  - [Distributed Trigger Topology](#distributed-trigger-topology)
  - [Network Topology](#network-topology)
  - [Timing and Trigger Distribution](#timing-and-trigger-distribution)
  - [Data Paths](#data-paths)
  - [Auxiliary Systems](#auxiliary-systems)
  - [Terminal Servers / Console Access](#terminal-servers--console-access)
  - [VME Crate Power PDUs](#vme-crate-power-pdus)
- [See Also](#see-also)

## Minimal System — DuoGe (DUO)

A single VME crate with one MTRG, one RTRG, and two DIGs (BUS_LEFT + BUS_RIGHT pair).

### Required Hardware

| Hardware | Role |
|----------|------|
| MVME5500 (IOC board) | VxWorks IOC — runs EPICS, controls all boards via VME ✅ verified 2026-04-18 — `ioc/README.md:L16,L44` (mvme5500 vxWorks boot file) |
| MTRG board | Master trigger — runs trigger algorithms, distributes decisions |
| RTRG board | Router — aggregates DIG hits, forwards trigger commands |
| DIG board × 2 | Digitizer pair (BUS_LEFT + BUS_RIGHT) — 10 ch each, 20 ch total |
| Ge detector | Germanium crystal — primary gamma-ray detector (two types: segmented and non-segmented, see below) |
| BGO detector | Anti-Compton shield around Ge |
| Slope box | Signal conditioning — shapes and conditions Ge/BGO signals before DIG |
| Raspberry Pi | Runs soft IOC (EPICS) for slope box / collector box HV and monitoring |
| Terminal server | Serial console access to MVME5500 and boards (gs-ts-south: 192.168.203.186, gs-ts-north: 192.168.203.91) ✅ verified 2026-04-14 — `ANLDAQ/EPICS_para.sh:L47` |
| Network switch | Connects IOC, Pi, host computer, EPICS CA traffic |

### MDIG vs SDIG — Master and Slave Digitizers

**Same physical board, different firmware** (compile-time generic: `FRONT_BUS_LEFT` — replaces old `SLAVE_MODE` flag, removed 2022-08-15). ✅ verified 2026-04-13 — `Digitizer.vhd:L43-48` (SLAVE_MODE removed; FRONT_BUS_LEFT=1 in BUS_LEFT.xise, FRONT_BUS_LEFT=0 in BUS_RIGHT.xise)

| | MDIG (BUS_LEFT Digitizer) | SDIG (BUS_RIGHT Digitizer) |
|---|---|---|
| SERDES link to RTRG | ✅ Yes — full upstream/downstream | ❌ None — invisible to trigger |
| Receives trigger/clock | From RTRG via SERDES | From RTRG directly via digitizer fanout board (not from MDIG) ✅ verified 2026-04-13 — `Digitizer.vhd:L34-42` (front-bus clock scheme abandoned 2022-08-15; "external digitizer fanout board copies TTCL link to both boards") |
| Sends multiplicity upstream | ✅ Yes | ❌ No (only throttle request bit via master) |
| IOC talks to it | ✅ Yes | ✅ Yes (VME registers) |
| SERDES lock checked by link_sys | ✅ Yes | ❌ No (locks passively via MDIG's clock) |
| External discriminator source | Local ch9 (ch0–8 can select `Ch 9` as ext disc src; ch9 itself has `N/A` as option 0) ✅ verified 2026-04-08 — `MDigUser.template:L10038-10055` | ch9 of this SDIG board (same template structure as MDIG — `Ch 9` available for ch0–8) ✅ verified 2026-04-08 — `SDigUser.template:L10038-10055` |
| Data channels | 10 (ch 0–9) | 10 (ch 0–9) — second set |
| Front-bus ribbon cable use | Sends discriminator bits (FRONT_BUS_LEFT=TRUE) ✅ verified 2026-04-13 — `Digitizer.vhd:L987-1125` | Receives discriminator bits (FRONT_BUS_LEFT=FALSE) |

**In a crate:** each MDIG+SDIG pair covers up to 20 detector channels. Connected by front-bus ribbon cable — used **only for discriminator bit exchange** (not clock/trigger since 2022-08-15). Both boards now receive TTCL independently via digitizer fanout board. ✅ verified 2026-04-21 — `FPGA/DIG/MAIN_FPGA/BuildBranches/DGS/Source/Digitizer.vhd:L34-41` (header comment: "front bus cable scheme has been found to no longer work... abandoned in favor of external digitizer fanout board... Center/Sum digitizer will use front bus cable to send discriminator bits to Side/Pattern digitizer"; `FRONT_BUS_LEFT` generic controls send/receive direction)

**Naming:** NFS scripts use `MDIG1/SDIG1/MDIG2/SDIG2` (accurate). ANLDAQ `SYSTEM_DEFINES.sh` uses `MDIG1/MDIG2/MDIG3/MDIG4` (simplified — all called MDIG regardless of role). The IOC boot cmd only configures MDIGs (`asynDigitizerConfig`); SDIGs are accessed via VME but don't need separate SERDES init.

*Sources: `DIG_firmware_expert.md`, `link_sys_analysis.md`, NFS `DGS_SYSTEM_DEFINES.sh` vs ANLDAQ `SYSTEM_DEFINES.sh` diff — 2026-04-05*

---

### Minimal Crate Layout (vme66 example)

```
Slot 1: MVME5500 (IOC)
Slot 3: MDIG1 (BUS_LEFT)
Slot 4: MDIG2 (BUS_RIGHT)
Slot 6: RTRG
Slot 7: MTRG
```

### Networking

- All devices run on **onenet: 192.168.203.xxx**
- Network switch = just a port extender (no special role)
- The IOC (MVME5500) connects to onenet via Ethernet
- EPICS CA traffic (PV reads/writes) flows over onenet between IOC, host PC, and Pis

### VME Crate Structure

- Each VME crate contains **3 individual VME backplanes**, each acting as an independent VME system ✅ verified 2026-04-18 — [wiki.anl.gov/gsdaq/CrateAndBoardMapping](https://wiki.anl.gov/gsdaq/CrateAndBoardMapping): "Each of these crates is actually three separate 7-slot backplanes with a common power supply"; VME01+VME02+VME03 = one physical crate (top of right-side relay rack)
- Each VME system hosts **up to 4 digitizers** → up to 12 DIGs per crate, 40 channels per VME system, 120 ch per crate ✅ verified 2026-04-18 — `DGS_SVN/dgs/daq_system_tags/SL6_DGS_20220923/ioc/boot/vme01-10.cmd`: most VME systems configure 4 DIGs (MDIG1/SDIG1/MDIG2/SDIG2 at slots 4–7); VME06 configures only 2 (short backplane — MDIG2+SDIG2 commented out), VME04 configures 3 (SDIG1 slot empty). Actual Gammasphere install: 44 DIG boards per MEMORY.md (10×4 + 2×2 crates).
- **4 RTRGs total** service all 11 DIG VME systems; each RTRG shares a VME system with DIGs (not a dedicated VME system): RTR1 (IOC1/slot 7) → IOC1/2/3; RTR2 (IOC4/slot 3) → IOC4/5/6; RTR3 (IOC32/slot 6, same VME as MTRG) → IOC7/8; RTR4 (IOC10/slot 3) → IOC9/10/11 ✅ verified 2026-04-20 — `DGS_SVN/dgs/daq_system_tags/SL6_DGS_20220923/ioc/boot/vme32.cmd:L48-51`
- Each VME backplane also has **one IOC board (MVME5500)** ✅ verified 2026-04-19 — `ioc/boot/vme66.cmd:L133-140` (Slot 1 = IOC MVME5500); all other crate cmd files follow same slot-1 IOC pattern
- **VME Fiber Expander** board (PCB #3174, ANL part `21pc032`, Rev A, Sept 2021) provides fully optical interface between MTRG (System Trigger) and RTRGs — replaced original copper/Cat5 Trigger Paddle Cards; installed July 2022. Requires DC balance enabled (`EN_RTR_DCBAL`, `LinkL_DCbal`) and cable pre-emphasis **disabled** (`PEHLRU=PEEFG=PEABCD=0`). ✅ verified 2026-04-17 — `knowledgeBase/DGS_SVN.md` (PCB #3174), `knowledgeBase/link_sys_analysis.md:1I`, `knowledgeBase/trig_setup_scripts.md` (fiber expander notes)
- Prior to digital upgrade: VXI crates used (larger, housed in a separate electronics "shack" room); VXI system dismantled post-upgrade. VXI5 specifically decommissioned by September 2021 (wiki: Digital_Gammasphere_Upgrade_Project), allowing grey cables for GS 81-109 to be removed from array. [Note: "before 2023" may reflect full teardown date vs. partial VXI5 decommission in Sept 2021 - unverified]

### VME Backplane

- The IOC accesses MTRG, RTRG, and DIG **directly via the VME backplane** (not over the network)
- VME = the physical bus inside the crate connecting all boards
- Universe II chip on MVME5500 bridges PowerPC CPU ↔ VME bus ✅ verified 2026-04-14 — `dgsDrivers/src/README.md:L229` ("MVME5500's Universe II chip bridges the PowerPC")
- All register reads/writes, DMA, and FIFO readout happen over this backplane

### Signal Chain

```
Ge detector → (analog) → Slope box → DIG (ADC, discriminator, digitizer)
BGO detector → (analog) → Slope box → DIG

DIG ─SERDES──► RTRG ─SERDES──► MTRG   (trigger decision, hardware links)
MTRG ─SERDES──► RTRG ─SERDES──► DIG   (accept/reject)

Accepted events → VME FIFO → MVME5500 (VME backplane) → tcpReceiver → host PC (onenet)
```

### EPICS Control

- MVME5500 runs `gretDet.munch` — accesses DIG/RTRG/MTRG registers via **VME backplane** ✅ verified 2026-04-19 — `ioc/boot/vme99.cmd:L37`, `ioc/boot/vme66.cmd:L24` (`ld < gretDet.munch`)
- Publishes PVs over onenet (EPICS CA)
- Raspberry Pi runs soft IOC — controls slope box / collector box HV via **SPI** (local hardware)
- CA ports: DuoGe = 5080/5081

---

## Full System — DGS

Scales up to 1 MTRG × 8 RTRG × 8 DIG × 10 ch = **640 channels**. ✅ verified 2026-04-06 — `FPGA/MTRG/Firmware/MAIN_FPGA/trunk/.../trigger_top_comp_defs.vhd`: `JTA_8X...` arrays confirm 8 data links (A–H, one per RTRG); 11 total SERDES (8 data + L/R/U)
Each RTRG manages a "sector" of 8 DIGs = 80 channels = one VME crate.

### Digitizer Hardware Origin

- DGS digitizer boards are **GRETINA-origin hardware** (designed by LBNL; firmware completely new for ANL experiments) ✅ wiki `/gsdaq/Digitizers` 2026-04-17
- ADC samples: **signed 14-bit @ 100 MHz (10 ns period)** ✅ verified 2026-04-17 — `jta_channel.vhd:L39,L41` (20230809 tag: `CLK: in std_logic — 100MHz system clock`; `RAW_ADC_DATA: in std_logic_vector(13 downto 0) — 14 bits 2's comp data from ADC`)
- Experiments using this hardware/firmware family: DGS, DFMA (Digital FMA), X-Array, DuoGe (DUO) — HELIOS used DGS digitizers only for preamp characterization testing (2016 SVN data, not a full firmware user) ✅ verified 2026-04-17 — `DGS_SVN.md`: `DFMA_20220711`/`DXA_20220720`/`DUB_20211101` tags confirm DFMA/DXA/DUO as firmware users; HELIOS limited to `HELIOS_Preamp_data` (one alpha-source test run, 2016); "since 2018" X-Array claim dropped — earliest verifiable X-Array experiment tag is July 2022 (`DXA_20220720`); DFMA naming appears in CSS OPI screens as early as 2013 (`how_to_python.txt`)
- **Center/Sum DIGs** (MDIG/BUS_LEFT): connected to RTRG — participate in trigger and veto logic
- **Side/Pattern DIGs** (SDIG/BUS_RIGHT): receive clock+trigger via front-bus cable from neighbor; one-way link — cannot participate in trigger or veto
- As of 2023: **Slope Box Extension (SBX) replaced old pickoff cards** — provides programmable time constants and replaces original power/control/monitoring system

### Additional Hardware vs. DuoGe

| Hardware | Role |
|----------|------|
| Collector box × 4 | Aggregates slope boxes from multiple detectors; provides HV control for all channels in a sector |
| Raspberry Pi (1 per collector box) | Runs EPICS soft IOC for that collector box's PVs (HV, BGO, monitoring) |
| fs2.onenet (file server) | NFS/PXE boot server for collector Pis |
| 12 VME crates | One per RTRG sector (192.168.203.141–145, 177–183) ✅ verified 2026-04-05 — ping confirmed live |

### Collector Box Architecture

```
Many slope boxes (1 per Ge detector)
    │
    ▼
Collector box (1 per sector, ~25–31 detectors) ✅ verified 2026-04-07 — collectorBox.sh:L8-49
    │  SPI bus (bcm2835, 5-bit DEVSEL, 24-bit transactions) ✅ verified 2026-04-25 — DEVSEL_bus.c:L28-35 (5 GPIO lines: 13,23,24,25,26); sbxPi_ioc.md:L26 (bcm2835 SPI1); slope_box_interface.md:L33 (24-bit)
    ▼
Raspberry Pi (PXE boot from fs2.onenet)
    │  EPICS soft IOC (CollectorBox_RevA)
    ▼
EPICS CA (5064/5065) → host PC / ANLDAQ
```

### 4 Collector Boxes

| Pi | IOC # | Name | GS Holes | Status |
|----|-------|------|-----------|--------|
| pi0 | 201 | South-East | GS 2–60 (even) + GS 70 = 31 holes ✅ verified 2026-04-15 — `st_201.cmd` (31 unique DetNbr values: 002–060 even + 070) | New repo (CollectorBox_RevA) |
| pi1 | 202 | South-West | GS 62–110 (even, 25 det) ✅ verified 2026-04-06 — `collectorBox.sh:L19-27` | Old piserver (undocumented) |
| pi2 | 203 | North-East | GS 1–59 (odd, 30 det) ✅ verified 2026-04-06 — `collectorBox.sh:L30-38` | Old piserver (undocumented) |
| pi3 | 204 | North-West | GS 61–109 (odd, 25 det) ✅ verified 2026-04-06 — `collectorBox.sh:L41-49` | Old piserver (undocumented) |

### EPICS CA Port Isolation

All systems share the **same physical network (onenet, 192.168.203.x)** but are logically separated by their CA server ports. An EPICS client must set `EPICS_CA_SERVER_PORT` to talk to the right system.

| System | Alias | CA Server Port | CA Repeater Port |
|--------|-------|---------------|------------------|
| DGS | — | 5064 ✅ | 5065 |
| DFMA | — | 5068 | 5069 |
| X-Array | DXA | 5072 ✅ | 5073 |
| SlopeBox teststand | — | 5074 | 5075 |
| DUB | — | 5078 | 5079 |
| DuoGe | DUO | 5080 ✅ | 5081 |

✅ All CA ports verified 2026-04-05/07 — `ANLDAQ/EPICS_env.sh` (DGS:L45-46, DXA:L23-24, DUO:L16-17, DFMA:L5 comment-only, SlopeBox:L36-37, DUB:L8 comment-only) ✅ re-verified 2026-04-23 — DGS/XDA/DUO/SlopeBox ports confirmed as exported vars; DFMA 5068/5069 and DUB 5078/5079 appear only in comment block L5/L8 (no exported vars found in this file)

### VME Crate Power PDUs

Three network-accessible power distribution units (PDUs) were installed June/July 2020 to allow remote power-cycling of VME crates. Each PDU has individually controllable outlets. Access requires being on the onenet network (including sonata); credentials: `localadmin` / `localadmin`.

| PDU | IP | Hostname | Controls |
|-----|----|----------|----------|
| vmepdu3 | 192.168.203.9 | vmepdu3 | Load A: VME32, VME07, VME08 — Load B: VME09, VME10, VME11 |
| vmepdu1 | 192.168.203.64 | vmepdu1 | Load A: VME04, VME05, VME06 — Load B: VME01, VME02, VME03 |
| vmepdu2 | 192.168.203.8 | vmepdu2 | Load 1: unused — Load 2: Microball/neutron detector VME crate |

**Web UI:** `http://<ip>/mainWK.html#/app/loads`  
**Operation:** Click the slider to change load state (off → on or on → off); confirm the popup.  
**VME32** is the master trigger crate (MTRG + RTR3); the other VMEs are digitizer/RTRG crates.

*Source: [wiki.anl.gov/gsdaq/Network_Accessible_Power_Control_Units_of_DGS](https://wiki.anl.gov/gsdaq/Network_Accessible_Power_Control_Units_of_DGS) — JTA, 2020-07-02. Visited 2026-04-23.*

### DAQ Power Supply (48V Redundant)

*Source: [wiki.anl.gov/gsdaq/DAQ_Power_Supply](https://wiki.anl.gov/gsdaq/DAQ_Power_Supply) — visited 2026-04-23*

Each of the four relay racks has a **redundant 48V DC supply** at the top (above the collector box and VME crate). Key characteristics:

- **Manufacturer:** Acopian (not ANL-designed). Datasheet: `wiki_gsdaq/images/f/f0/AcopianCatRedundantPower.pdf`
- **Redundancy modes:** Cold spare (one off) or hot spare (both on simultaneously)
- **Fault monitoring:** Internal fault relays → collector box → EPICS alarm status
- **Floating outputs:** Outputs are not referenced to ground; grounding point is defined at the digitizer (not at the supply)
- **Double isolation:** Each detector is doubly isolated — 48V supply is isolated from rack chassis; SBX power board additionally isolates the point-of-load supply from the detector's ground
- **Coverage:** Each supply powers the collector box and up to **30 detectors**
- **Connection:** Overhead cord reel (one per rack)
- **Chassis isolation:** Supply chassis can be isolated from the relay rack frame to eliminate ground loops between the two cord reels per rack

---

## Hardware Summary Comparison

| Component | DuoGe (minimal) | DGS (full) |
|-----------|----------------|------------|
| VME crates | 1 | 12 |
| MTRG | 1 | 1 |
| RTRG | 1 | 8 |
| DIG pairs | 1 | up to 64 |
| Detector channels | 20 | up to 640 |
| Slope boxes | 2 | 110 |
| Collector boxes | — | 4 |
| Raspberry Pis | 1 | 4 (collector box Pis: pi0–pi3) |
| IOC boards (MVME5500) | 1 | 12 |

---

## Gammasphere Ge Detector Types

_Source: `DGS_SVN/dgs/Detector_Repair/DetectorRepairProcedure.docx` (JTA, 2019-06-02)_

Gammasphere has **two kinds of HPGe detectors**: segmented and non-segmented.

| Type | Ge crystal | Signals | Electronics |
|------|-----------|---------|-------------|
| **Non-segmented** (older) | Single chunk, one contact | 2 Ge contacts (Center only used); 2 signals to digitizer | 1 circuit board + HV filter |
| **Segmented** (newer) | Ge crystal split in half — 3 contacts: Center, Side A, Side B ✅ verified 2026-04-23 — wiki `/gsdaq/Gammasphere_Detectors` ("split in half resulting in three Ge contacts (Center, Side A, Side B)") | 2 signals to digitizer: Center (ch N) + mux'd Side A/B (ch N+1) via SBX pickoff ✅ verified 2026-04-23 — wiki: "Only one of the side connections is brought out to the digitizers"; `sbx.md:L83` (Ge Side A/B → DIG ch N+1) | 2 circuit boards + HV filter |

**Operating conditions (both types):**
- Vacuum: 10⁻⁵ to 10⁻⁶ Torr inside detector cold volume
- Temperature: liquid nitrogen (~77 K)
- Bias voltage: 2,900–4,800 V across the array (most common values: 4000V and 4800V) ✅ verified 2026-04-13 — `collectorboxpi/CollectorBox_RevA/iocBoot/iocCollectorApp/st_202.cmd` DS_GEHV values (non-zero range: 2900–4800V); DS_GEHV=0 = empty/disabled slot
- Preamp type: **transistor-reset preamplifier** (no resistor feedback; NPN transistor bleeds accumulated charge when output reaches ~+10V; second comparator turns NPN off at ~0V) ✅ verified 2026-04-20 — `DGS_SVN/dgs/Detector_Repair/DetectorRepairProcedure.docx:JTA,2019-06-02` ("When the output gets to about +10V the comparator turns on" / "A 2nd comparator ensures that the NPN is turned off when the output gets near 0V")
- Normal reset rate: every few ms to a few hundred ms depending on neutron damage to crystal ✅ verified 2026-04-23 — DIG_firmware_expert.pdf p.12 ("Preamp Reset occurs at rates of once every few milliseconds to once every few hundred milliseconds depending upon how much neutron-induced damage the detector has")
- Leakage current: typically ~1 nA ✅ verified 2026-04-20 — `DGS_SVN/dgs/Detector_Repair/DetectorRepairProcedure.docx:JTA,2019-06-02` ("Inevitably there is a certain amount of DC leakage (typically in the ~1nA range)")

**Common repair reasons:** bad resolution, excessive reset rate, no signal.

---

## See Also

- `overview_DGS.md` — Full Gammasphere system overview (signal chain, network map, software stack)
- `overview_SmallSystem.md` — DuoGe (DUO) and X-Array (DXA) small system details
- `fpga.md` — FPGA firmware architecture (trigger cycle, signal flow, PEQ details)
- `collectorboxpi.md` — Collector box Raspberry Pi soft IOC (SPI, PXE boot, HV control)
- `collector_fpga.md` — Collector box FPGA firmware (git repo): CtrlFPGA, StripeFPGA, pickoff card FPGAs
- `collector_box_fpga.md` — Collector box FPGAs (PSG SVN origin): ControlStripe (Spartan-3, 48V relay/clock/SYNC/LED per stripe) + CtlFanout (Spartan-6, RPi SPI gateway + ADS1158 ADC scanning)
- `ioc.md` — EPICS IOC configuration (MVME5500 boot, firmware versions, VME setup)
- `sbx.md` — Slope Box Extension (SBX): signal conversion, BGO pattern/sum, GS_ID dongle, HV map
- `sbxPi_ioc.md` — SBX Pi standalone IOC (PickoffApp_RevC): SPI1 24-bit transactions, CAMAC_IO device support, global mailboxes, I2C command FIFO protocol, HV ramp logic
- `myriad.md` — MγRIAD auxiliary detector interface: NIM I/O, ECL, TTCL, DGS link U
- `gammasphere_geometry.md` — 110 GS detector holes, 17 rings, θ/φ angles, collector box assignments
- `connectors.md` — All hardware connector pinouts (DIG RJ45, MTRG/RTRG 125-pin SERDES, NIM I/O)

---

*Source: DGS_tools_pack code exploration, link_sys_analysis.md, nfs_layout.md. Created: 2026-04-05. Updated: 2026-04-23 (added VME Crate Power PDUs section from wiki; previous update: 2026-04-22 added Table of Contents, sbxPi_ioc.md to See Also).*
