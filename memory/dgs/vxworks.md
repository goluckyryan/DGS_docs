# vxworks — VxWorks Cross-Compiler for DGS IOC (MVME5500)

## What It Is

A complete **cross-compilation environment** for building the DGS IOC binary (`gretDet.munch`) that runs on the **Motorola MVME5500** VME single-board computer (PowerPC CPU, VxWorks 5.5 RTOS).

The MVME5500 has no development tools — you compile on Ubuntu x86-64 and produce a PowerPC binary. This is cross-compilation.

**Output:** `gretDet.munch` — the single binary loaded on each VME crate to bring the detector IOC online.

---

## Target Hardware

- **Board:** Motorola MVME5500
- **CPU:** PowerPC 7455 (G4)
- **OS:** VxWorks 5.5
- **Bus:** VME64x
- **Role:** Reads out DIG/RTRG/MTRG via VME DMA; controls acquisition; sends data over TCP

---

## Directory Structure

```
vxworks/
├── Makefile                         ← Top-level build — run 'make' here
├── munch.tcl                        ← Adds C++ startup tables to binary
├── migration.md                     ← Notes on migration from con6 (Solaris)
├── x86-linux/                       ← Cross-compiler tools (NOT in git — download separately)
│   └── bin/                         ← ccppc, g++ppc, nmppc, ldppc, ...
├── vxWorks/Tornado2.2/              ← VxWorks header files (not compiled)
│   └── target/
│       ├── h/                       ← VxWorks OS headers (vxWorks.h, semLib.h, ...)
│       └── config/mv5500/           ← MVME5500 board headers (universe.h, etc.)
├── epics/base-3.14.12.1/            ← EPICS base framework + build system
├── synApps/asyn4-17/                ← Hardware driver abstraction layer
├── sncseq/sncseq-2.0.12/            ← State machine compiler + runtime
├── dgsDrivers/                      ← DGS hardware driver library
│   └── lib/vxWorks-ppc604_long/
│       └── libdevDGSDriverSupport.a
└── dgsIoc/                          ← Final IOC application
    └── bin/vxWorks-ppc604_long/
        └── gretDet.munch            ← THE OUTPUT — loaded on each MVME5500
```

---

## Component Roles

### `x86-linux/` — Cross-Compiler Toolchain

Pre-built binaries from Jefferson Lab. NOT in git (too large).

| Tool | Role |
|------|------|
| `ccppc` / `g++ppc` | C/C++ compilers → PowerPC VxWorks |
| `ldppc` | Linker |
| `nmppc` | Symbol table reader (used in munch step) |

**Download:** https://coda.jlab.org/drupal/content/ppc-cross-compilers

### `vxWorks/Tornado2.2/` — Header Files

Not compiled — just `.h` files the compiler includes to understand VxWorks APIs.
- `target/h/` — general VxWorks headers
- `target/config/mv5500/` — MVME5500-specific headers, especially `universe.h` (VME Universe II bridge chip registers)

### `epics/base-3.14.12.1/` — EPICS Framework

> **Note:** This is EPICS 3.14 (older), unlike the EPICS 7.0 used in ANLDAQ/collectorboxpi. VxWorks IOC requires 3.14.

Provides:
- Build system (EPICS-aware GNU make rules for cross-compilation)
- Core VxWorks runtime libraries: `libca.a`, `libdbCore.a`, `libCom.a`
- Host tools: `dbExpand`, `makeBaseApp`

### `synApps/asyn4-17/` — Hardware Driver Abstraction

Provides standard interface between EPICS records and hardware drivers — threading, locking, EPICS integration. DGS drivers are asyn port drivers.

Produces: `libasyn.a` for VxWorks target

### `sncseq/sncseq-2.0.12/` — State Machine Compiler

Two parts:
- `snc` — host tool: compiles `.st` (State Notation) → `.c`
- `libseq.a` / `libpv.a` — VxWorks libraries that run compiled state machines

**DGS state machine files:**
| File | Role |
|------|------|
| `inLoop.st` | VME readout loop |
| `outLoop.st` | Data validation |
| `MiniSender.st` | Network transmission (TCP) |

### `dgsDrivers/` — DGS Hardware Driver Library

The physics-specific code: VME FIFO readout via DMA, buffer management, acquisition control, TCP data transmission.

Produces:
- `libdevDGSDriverSupport.a` — compiled driver for VxWorks
- `dgsDriver.dbd` — EPICS DB definition (PV record types this driver provides)

Key driver files:
- `asynDigitizerDriver.cpp` — DIG digitizer board driver
- `asynTriggerDriver.cpp` — RTRG/MTRG trigger board driver

### `dgsIoc/` — Final IOC Application

Ties everything together. Links all libraries into `gretDet.munch`.

```makefile
# dgsIoc/tcDetApp/src/Makefile (simplified)
gretDet_LIBS += devDGSDriverSupport   # DGS drivers
gretDet_LIBS += asyn                  # asyn layer
gretDet_LIBS += seq pv                # state machines
gretDet_LIBS += $(EPICS_BASE_IOC_LIBS)  # EPICS core
```

---

## Build Pipeline

```
Stage 1: Libraries
  epics/base-3.14.12.1  →  libCom.a, libca.a, libdbCore.a, ...
  synApps/asyn4-17      →  libasyn.a
  sncseq/sncseq-2.0.12  →  snc (host tool), libseq.a, libpv.a
  dgsDrivers/           →  libdevDGSDriverSupport.a, dgsDriver.dbd

Stage 2: Final Binary
  dgsIoc/               →  gretDet.munch
                             (via: compile → link → munch step)
```

### The Munch Step

VxWorks requires C++ global constructors to be called explicitly at startup. `munch.tcl` runs `nmppc` to extract all symbols from the linked binary, generates a `symTbl.c` with startup/shutdown tables, compiles it, and re-links. The final output is `gretDet.munch`.

---

## Building

```sh
cd vxworks/
make
# Full build: ~30-60 minutes first time
# Output: dgsIoc/bin/vxWorks-ppc604_long/gretDet.munch
```

**Requirements:**
- Ubuntu (x86-64) host
- `x86-linux/` cross-compiler (download separately from JLab)
- Standard build tools: `make`, `tcl`

---

## Loading on MVME5500

After build, `gretDet.munch` is placed in `ioc/bin/` and served via FTP from the host. The MVME5500 boot script (`vme66.cmd`, etc.) loads it at startup:

```
# VxWorks boot prompt
ld < /global/ioc/bin/gretDet.munch
```

---

## Connections to Other Subsystems

- **ioc/** — `gretDet.munch` ends up in `ioc/bin/`; `dgsDriver.dbd` ends up in `ioc/dbd/`
- **fpga/** — drivers in `dgsDrivers/` communicate with DIG/RTRG/MTRG firmware via VME registers
- **ANLDAQ** — EPICS PVs exposed by the IOC are the interface ANLDAQ uses

---

## Notes

- EPICS 3.14 (not 7.0) — VxWorks IOC uses the older EPICS. Do not mix with EPICS 7.0 paths.
- Cross-compiler is NOT in git — must download from JLab before building
- `migration.md` documents how this was ported from Solaris/con6 to Ubuntu
- Each VME crate loads the same `gretDet.munch` but uses a different boot script with crate-specific parameters

---
*Source: `DGS_tools_pack/vxworks/` (git repo). Created: 2026-04-05.*
