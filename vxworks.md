# VxWorks Cross-Compiler for DGS IOC (MVME5500)

Stability: C2 - Active / semi-stable

## Table of Contents

- [What this project does](#what-this-project-does)
- [Directory Structure](#directory-structure)
- [Component Roles](#component-roles)
  - [`x86-linux/` — Cross-compiler toolchain](#x86-linux----cross-compiler-toolchain)
  - [`vxWorks/Tornado2.2/` — VxWorks header files](#vxworkstornado22----vxworks-header-files)
  - [`epics/base-3.14.12.1/` — EPICS base framework](#epicsbase-314121----epics-base-framework)
  - [`synApps/asyn4-17/` — Hardware driver abstraction (asyn)](#synappsasyn4-17----hardware-driver-abstraction-asyn)
  - [`sncseq/sncseq-2.0.12/` — State machine compiler and runtime](#sncseqsncseq-2012----state-machine-compiler-and-runtime)
  - [`dgsDrivers/` — DGS hardware driver library](#dgsdrivers----dgs-hardware-driver-library)
  - [`dgsIoc/` — Final IOC application](#dgsioc----final-ioc-application)
  - [`munch.tcl` — C++ startup table generator](#munchtcl----c-startup-table-generator)
- [Target Hardware](#target-hardware)
- [Host Requirements](#host-requirements)
- [Build](#build)
- [Build Pipeline](#build-pipeline)
- [Build Outputs](#build-outputs)
  - [Stage 1 — Intermediate libraries](#stage-1----intermediate-libraries-inputs-to-dgsioc)
  - [Stage 2 — Final loadable](#stage-2----final-loadable-produced-by-dgsioc)
- [VxWorks Munch Process](#vxworks-munch-process)
- [Loading on VxWorks (MVME5500)](#loading-on-vxworks-mvme5500)
- [Glossary](#glossary)
- [Source](#source)
- [VxWorks API Reference Docs](#vxworks-api-reference-docs)
- [FIFO Readout Internals](#fifo-readout-internals) → see [vxworks_fifo_readout.md](vxworks_fifo_readout.md)
- [Connections to Other Subsystems](#connections-to-other-subsystems)
- [Quick Notes](#quick-notes)
- [devGData.c — Legacy VME_IO Device Support (Removed 2025-04-21)](#devgdatac--legacy-vme_io-device-support-removed-2025-04-21)
- [Utility / Support Modules](#utility--support-modules-minor-files)
  - [equalSub.c — EPICS Equality Subroutine](#equalsub.c--epics-equality-subroutine)
  - [restoreSub.c — EPICS Restore Subroutine](#restoresub.c--epics-restore-subroutine)
  - [profile.c — VxWorks Performance Profiler](#profilec--vxworks-performance-profiler)
  - [MergedAsynDigParams.c — DIG Asyn Parameter Registration](#mergedasyndigparamsc--dig-asyn-parameter-registration)
  - [FlashMaintenance.c — VME Flash Register Constants](#flashmaintenancec--vme-flash-register-constants)
  - [devGVME.c — VME Board Management Layer](#devgvmec--vme-board-management-layer)
- [State Machines & Runtime Drivers](#state-machines--runtime-drivers) → see [vxworks_state_machines.md](vxworks_state_machines.md)
- [Cross-References](#cross-references)

---

## What this project does

The DGS (Digital Gamma-ray Spectrometer) detector system uses a small dedicated computer — a Motorola MVME5500 board running inside a VME crate — to read out data from gamma-ray detector electronics in real time. That computer runs **VxWorks 5.5** ✅ verified 2026-04-07 — `vxworks/Makefile:L2`, a real-time operating system designed for embedded hardware where timing and reliability matter more than user-friendliness.

To write software for the MVME5500, you cannot compile it on the board itself — the board has no development tools and is too slow for that. Instead, you compile on a modern Linux PC and produce a binary that runs on the PowerPC CPU inside the MVME5500. This is called **cross-compilation**: the machine you compile on (Ubuntu x86-64) is different from the machine that runs the result (PowerPC VxWorks).

This repository contains everything needed to reproduce that cross-compilation on Ubuntu 24, starting from source code and ending with `gretDet.munch` — the single binary file that gets loaded onto each MVME5500 crate to bring the detector IOC online. ✅ verified 2026-04-07 — `dgsDrivers/dgsDriverApp/src/README.md:L744` ("IOC boot: gretDet.munch loaded by VxWorks shell")

---

## Directory Structure

```
VxWorks/
├── Makefile                        # Top-level build orchestrator — run 'make' here
├── README.md                       # This file
├── migration.md                    # Notes on migration from con6 (Solaris)
├── munch.tcl                       # Script that adds C++ startup tables to the binary
├── x86-linux/                      # Cross-compiler tools (not in git — see below)
│   └── bin/                        # ccppc, g++ppc, nmppc, ldppc, ...
├── vxWorks/Tornado2.2/             # VxWorks system header files (not compiled, just #included)
│   └── target/
│       ├── h/                      # VxWorks OS headers (vxWorks.h, semLib.h, ...)
│       └── config/mv5500/          # MVME5500 board-specific headers (universe.h, etc.)
├── epics/base-3.14.12.1/           # EPICS base framework — provides the build system and core libraries  ✅ verified 2026-04-07 — ls vxworks/epics/
├── synApps/asyn4-17/               # asyn — hardware driver abstraction layer  ✅ verified 2026-04-07 — ls vxworks/synApps/
├── sncseq/sncseq-2.0.12/           # State machine compiler and runtime  ✅ verified 2026-04-07 — ls vxworks/sncseq/
├── dgsDrivers/                     # DGS hardware driver library (built here, linked into dgsIoc)
│   └── lib/vxWorks-ppc604_long/
│       └── libdevDGSDriverSupport.a
└── dgsIoc/                         # Final IOC application — produces the file loaded on the board
    └── bin/vxWorks-ppc604_long/
        └── gretDet.munch           # The single binary loaded on each MVME5500
```

---

## Component Roles

### `x86-linux/` — Cross-compiler toolchain

This folder contains the compiler tools that run on your Ubuntu PC but produce code for the PowerPC CPU in the MVME5500. These are not the same as the system `gcc` on your PC — they speak a completely different instruction set.

Key tools:
- `ccppc` / `g++ppc` — C and C++ compilers targeting PowerPC VxWorks
- `ldppc` — linker that combines compiled pieces into a single binary
- `nmppc` — reads a binary and lists all its exported symbols (function/variable names)

These are pre-built binaries from Jefferson Lab; they are not compiled from source and are excluded from git (too large). Re-download from https://coda.jlab.org/drupal/content/ppc-cross-compilers. ✅ verified 2026-04-07 — `vxworks/README.md:L78,L194`

---

### `vxWorks/Tornado2.2/` — VxWorks header files

These are not compiled — they are **header files** (`.h`) that the compiler reads while compiling the driver code, so it knows the interface to VxWorks OS functions. Think of them as the VxWorks API reference that the compiler uses to check your code.

- `target/h/` — general VxWorks OS headers (`semLib.h`, `taskLib.h`, `vxWorks.h`, ...)
- `target/config/mv5500/` — MVME5500-specific headers, most importantly `universe.h` which describes the Universe II VME bridge chip registers

---

### `epics/base-3.14.12.1/` — EPICS base framework ✅ verified 2026-04-18 — `vxworks/epics/base-3.14.12.1/` directory exists

EPICS (Experimental Physics and Industrial Control System) is the control system framework used across most physics labs worldwide. It provides:

- A **build system** (based on GNU make with EPICS-specific rules) that all downstream components inherit. It knows how to cross-compile for VxWorks, how to run the munch step, etc.
- **Core runtime libraries** for the VxWorks target: channel access networking (`libca.a`), database processing (`libdbCore.a`), common utilities (`libCom.a`), and more.
- **Host tools** used during the build: `dbExpand` (merges database definition files), `makeBaseApp`, etc.

Building EPICS base is the first step — everything else depends on it.

---

### `synApps/asyn4-17/` — Hardware driver abstraction (asyn) ✅ verified 2026-04-18 — `vxworks/synApps/asyn4-17/` directory exists

asyn is a standard EPICS add-on that provides a clean interface between EPICS database records and hardware drivers. Instead of each driver implementing its own threading, locking, and EPICS integration from scratch, drivers implement a standard "port driver" interface and asyn handles the rest.

The DGS digitizer and trigger drivers (`asynDigitizerDriver.cpp`, `asynTriggerDriver.cpp`) are all asyn port drivers.

Produces `libasyn.a` for the VxWorks target.

---

### `sncseq/sncseq-2.0.12/` — State machine compiler and runtime ✅ verified 2026-04-18 — `vxworks/sncseq/sncseq-2.0.12/` directory exists

The State Notation Compiler (snc/seq) lets you write control logic as **state machines** in a high-level language (`.st` files), then compiles them to C code that runs as tasks on VxWorks.

Two parts:
- `snc` — a **host tool** (runs on Ubuntu) that compiles `.st` → `.c`
- `libseq.a` / `libpv.a` — **VxWorks libraries** that execute those compiled state machines at runtime

The DGS driver uses three state machine files for its data flow control logic: `inLoop.st` (VME readout), `outLoop.st` (data validation), and `MiniSender.st` (network transmission).

---

### `dgsDrivers/` — DGS hardware driver library

This is the physics-lab-specific code: reading VME digitizer and trigger FIFOs via DMA, managing data buffers, controlling acquisition, sending data over the network. It uses all the frameworks above (EPICS, asyn, sncseq) and adds the hardware-specific logic on top.

Produces:
- `libdevDGSDriverSupport.a` — the compiled driver library for VxWorks
- `dgsDriver.dbd` — EPICS database definition file listing all the PV (Process Variable) record types this driver provides

This library is an **input** to `dgsIoc` — it is not directly loaded on the hardware.

---

### `dgsIoc/` — Final IOC application

This is the top-level application that ties everything together and produces the actual binary loaded on the MVME5500. "IOC" stands for Input/Output Controller — the software instance running on the hardware board.

`dgsIoc/tcDetApp/src/Makefile` declares the final product and lists every library it needs:

```makefile
gretDet_LIBS += devDGSDriverSupport  # from dgsDrivers/
gretDet_LIBS += asyn                 # from synApps/
gretDet_LIBS += seq pv               # from sncseq/
gretDet_LIBS += $(EPICS_BASE_IOC_LIBS)  # from epics/
```

The EPICS build system bundles all of those libraries into a single binary (`gretDet.munch`) via the munch process described below.

Also contains:
- `iocBoot/iocArray/vme*.cmd` — startup scripts run on each MVME5500 at boot
- `db/` — EPICS database template files (`.template`, `.db`) defining PVs
- `dbd/` — merged database definition

---

### `munch.tcl` — C++ startup table generator

VxWorks 5.5 does not automatically call C++ static constructors (code that runs when a program loads) or destructors (code that runs when it unloads). The "munch" step works around this: it scans the compiled binary for constructor/destructor symbols and generates a small C file that registers them, which is then compiled and linked back into the final binary. Without this step, any C++ objects with static initialization would silently fail to initialize. ✅ verified 2026-04-12 — `vxworks/munch.tcl:L124` (matches `__STI__`/`__STD__` ctor/dtor symbols from `nm` output); `L64-65` (generates `ctors`/`dtors` arrays "to be called by runtime system")

---

## Target Hardware

| Item | Detail |
|---|---|
| **Board** | Motorola MVME5500 ✅ verified 2026-04-07 — `dgsDrivers/src/README.md:L4` |
| **CPU** | PowerPC 604 (`ppc604_long` ABI) ✅ verified 2026-04-07 — `dgsDrivers/src/README.md:L4` ("PowerPC board") + ABI dir name `vxWorks-ppc604_long` |
| **OS** | Wind River VxWorks 5.5 ✅ verified 2026-04-07 — `vxworks/migration.md:L26` ("VxWorks 5.5 compatible") |
| **Crate bus** | VME — boards talk to each other over the VME backplane |
| **VME bridge** | Universe II chip — connects the PowerPC CPU to the VME bus ✅ verified 2026-04-07 — `dgsDrivers/src/README.md:L229` ("MVME5500's Universe II chip bridges the PowerPC") |
| **Toolchain** | Tornado 2.2 / GCC 2.96 (`ccppc`, `g++ppc`, `nmppc`, `ldppc`) ✅ verified 2026-04-07 — `vxworks/migration.md:L21,L26` (Tornado2.2 dirs + "GCC 2.96, powerpc-wrs-vxworks") |

---

## Host Requirements

**OS**: Ubuntu 24.04 LTS (linux-x86_64)

**Packages**:
```bash
sudo apt-get install gcc-12 g++-12 flex libfl-dev make perl tcl
```

| Package | Why it is needed |
|---|---|
| `gcc-12` / `g++-12` | Host C/C++ compiler. GCC 13 is too strict for the older C++98 code in EPICS 3.14. |
| `flex` + `libfl-dev` | Lexer generator used to build `snc` (the state machine compiler). |
| `perl` | Used by EPICS build system scripts. |
| `tcl` | Used by `munch.tcl`. |

**Cross-compiler**: pre-installed at `x86-linux/` (from https://coda.jlab.org/drupal/content/ppc-cross-compilers)

---

## Build

```bash
# Build everything in order (epics → asyn → sncseq → dgsDrivers → dgsIoc)
make

# Build individual components
make epics
make asyn
make sncseq
make dgsDrivers
make dgsIoc

# Clean everything
make clean

# Clean individual components
make clean-epics
make clean-asyn
make clean-sncseq
make clean-dgsDrivers
make clean-dgsIoc
```

The Makefile automatically sets `EPICS_HOST_ARCH=linux-x86_64` and prepends `x86-linux/bin` to `PATH` so the cross-compiler tools are found first.

All paths inside the build system are self-relative — no absolute paths need to be updated when the project directory is moved or renamed.

---

## Build Pipeline

The diagram below shows how each component depends on the others. Arrows represent "provides input to". Build order must follow this dependency chain — EPICS base must be built before anything else.

```mermaid
flowchart TD
    subgraph PREREQS["Prerequisites — present on disk, no build step needed"]
        TOOLS["x86-linux/bin/\nCross-compiler tools for PowerPC VxWorks\nccppc / g++ppc — compile C/C++ → PowerPC code\nnmppc — list symbols in a binary\nldppc — link objects into a binary\nAdded to PATH by top-level Makefile"]
        HDRS["vxWorks/Tornado2.2/\nVxWorks OS header files — read by compiler, not compiled\ntarget/h/ — OS API headers\ntarget/config/mv5500/ — MVME5500 board headers\nTold to EPICS via WIND_BASE setting"]
    end

    EPICS["epics/base-3.14.12.1/\nBuilds the EPICS framework\nConfigures cross-compiler: CC=ccppc, NM=nmppc\n─────────────────────────────\nHost tools: dbExpand, makeBaseApp, build rules\nVxWorks libraries: libCom.a, libca.a + 12 more"]

    ASYN["synApps/asyn4-17/\nDriver abstraction layer\nVxWorks library: libasyn.a"]

    SNCSEQ["sncseq/sncseq-2.0.12/\nState machine compiler and runtime\nHost tool: snc  (compiles .st files → C)\nVxWorks libraries: libseq.a, libpv.a"]

    DGS["dgsDrivers/\nDGS hardware driver\n.c/.cpp source → compiled by ccppc\n.st state machines → snc → C → ccppc\n─────────────────────────────\nVxWorks library: libdevDGSDriverSupport.a\nEPICS DB definition: dgsDriver.dbd"]

    DGSIOC["dgsIoc/tcDetApp/src/\nFinal IOC application\nLinks all libraries into one binary:\n  devDGSDriverSupport + asyn + seq + pv + EPICS base libs\nRuns munch.tcl to add C++ startup tables\n─────────────────────────────\nOutput: bin/vxWorks-ppc604_long/gretDet.munch"]

    TARGET(["MVME5500 — VxWorks 5.5\nld < gretDet.munch\nLoads and starts the IOC"])

    PREREQS -->|"Cross-compiler in PATH\nVxWorks headers available"| EPICS
    EPICS -->|"Build rules + core libraries"| ASYN
    EPICS -->|"Build rules + core libraries"| SNCSEQ
    EPICS -->|"Build rules + core libraries"| DGS
    ASYN -->|libasyn.a| DGS
    SNCSEQ -->|"snc tool + libseq.a + libpv.a"| DGS
    EPICS -->|"Core IOC libraries"| DGSIOC
    ASYN -->|libasyn.a| DGSIOC
    SNCSEQ -->|"libseq.a + libpv.a"| DGSIOC
    DGS -->|"libdevDGSDriverSupport.a + dgsDriver.dbd"| DGSIOC
    DGSIOC -->|gretDet.munch| TARGET
```

---

## Build Outputs

The build is a two-stage pipeline. The first stage builds libraries; the second stage links them all into the single file that runs on the hardware.

### Stage 1 — Intermediate libraries (inputs to dgsIoc)

These are compiled code libraries (`.a` files — think of them as packages of pre-compiled functions). They are not loaded on the hardware directly; they are ingredients for the final binary.

| File | What it contains |
|---|---|
| `epics/base-3.14.12.1/lib/vxWorks-ppc604_long/*.a` | 14 EPICS base libraries — channel access, database engine, utilities |
| `synApps/asyn4-17/lib/vxWorks-ppc604_long/libasyn.a` | Driver abstraction layer |
| `sncseq/sncseq-2.0.12/lib/vxWorks-ppc604_long/libseq.a` | State machine runtime |
| `dgsDrivers/lib/vxWorks-ppc604_long/libdevDGSDriverSupport.a` | DGS hardware driver code |
| `dgsDrivers/dbd/dgsDriver.dbd` | EPICS process variable definitions |

### Stage 2 — Final loadable (produced by dgsIoc)

| File | What it is |
|---|---|
| `dgsIoc/bin/vxWorks-ppc604_long/gretDet.munch` | The complete IOC binary — all libraries bundled in, ready to load on VxWorks |
| `gretDet.munch` | Copy in the project root — convenient for deployment to the MVME5500 |

---

## VxWorks Munch Process

VxWorks 5.5 is an older embedded OS that does not automatically run C++ initialization code when a binary is loaded. The "munch" process works around this limitation. It is run automatically by the EPICS build system as part of producing any VxWorks binary.

**Step by step:**

1. **Compile** — each `.c`, `.cpp`, and `.st` source file is compiled to a `.o` object file (a compiled but not yet combined fragment).

2. **Link** — all `.o` object files and all `.a` libraries are combined by `ldppc` into a single binary file called `gretDet`. This binary is in **ELF format** (Executable and Linkable Format — the standard binary format on Linux and VxWorks). At this point the binary is complete but is missing C++ initialization.

3. **Extract constructor symbols** — `nmppc gretDet` reads the binary's symbol table (the list of all function and variable names inside) and pipes it into `munch.tcl`. The script looks for special symbols named `_GLOBAL__I_*` (C++ static constructors — functions that must run at load time) and `_GLOBAL__D_*` (destructors — run at unload time).

4. **Generate registration code** — `munch.tcl` writes a small C file (`gretDet_ctdt.c`) containing arrays of those constructor/destructor function pointers.

5. **Re-link** — `gretDet_ctdt.c` is compiled and linked back into the binary, producing `gretDet.munch`. When VxWorks loads this file, the startup code walks those arrays and calls every constructor, properly initializing all C++ objects.

```
Source files (.c, .cpp, .st)
        │ compile with ccppc
        ▼
  Object files (.o)  +  Libraries (.a)
        │ link with ldppc
        ▼
  gretDet  (ELF binary, missing C++ init)
        │ nmppc | munch.tcl
        ▼
  gretDet_ctdt.c  (C++ constructor/destructor table)
        │ compile + re-link
        ▼
  gretDet.munch  (complete ELF binary, ready for VxWorks)
```

---

## Loading on VxWorks (MVME5500)

Only one file needs to be transferred to the board and loaded. `gretDet.munch` contains everything — EPICS base, asyn, the state machine runtime, and the DGS driver — all statically bundled in. On the VxWorks shell:

```
ld < gretDet.munch
```

`ld` is the VxWorks dynamic loader command. The `<` redirects the file into it. VxWorks loads the binary into memory, resolves addresses, calls all the C++ constructors registered by the munch step, and the IOC starts up.

This matches the existing startup scripts in `dgsIoc/iocBoot/iocArray/vme*.cmd`.

---

## Glossary

| Term | Meaning |
|---|---|
| **Cross-compilation** | Compiling code on one machine (Ubuntu x86-64) to run on a different machine (PowerPC VxWorks) |
| **VxWorks** | A real-time operating system by Wind River, used in embedded hardware where deterministic timing is critical |
| **EPICS** | Experimental Physics and Industrial Control System — a widely used control system framework in physics labs |
| **IOC** | Input/Output Controller — the software instance running on a hardware board, managing detector readout and control |
| **VME** | A hardware bus standard used in physics and industrial instrumentation for connecting boards in a crate |
| **Universe II** | The chip on the MVME5500 that connects the PowerPC CPU to the VME bus; all VME register and DMA access goes through it |
| **ELF** | Executable and Linkable Format — the standard binary file format on Linux and VxWorks. Stores compiled machine code, data, and a symbol table |
| **`.a` file** | A static library — a collection of compiled `.o` object files bundled together. Linked into the final binary at build time |
| **`.o` file** | Object file — a single compiled source file, not yet linked with others |
| **Munch** | A VxWorks-specific post-processing step that adds C++ constructor/destructor registration tables to a binary |
| **DMA** | Direct Memory Access — hardware reads data directly into memory without CPU involvement, used for high-speed VME readout |
| **PV** | Process Variable — an EPICS named data channel that can be read or written over the network |
| **DBD** | Database Definition — an EPICS file describing what record types and PVs a driver provides |
| **asyn** | EPICS driver framework that standardizes the interface between hardware drivers and EPICS records |
| **snc / seq** | State Notation Compiler / Sequencer — compiles state machine `.st` files to C, runs them as tasks on VxWorks |
| **GCC 2.96** | The old C/C++ compiler version in the cross-toolchain. Modern GCC cannot replace it for this target |
| **ppc604_long** | The specific PowerPC ABI (binary interface) used by the MVME5500 — "long" means 32-bit long integers |

---

## Source

Original code lives on `con6` (Solaris, `192.168.203.136`, user `dgs`).
See `migration.md` for a full list of what was copied and what was changed.

## VxWorks API Reference Docs

HTML and PDF reference manuals are archived in `DGS_tools_pack/DGS_SVN/dgs/VxWorksDocs/`:

| File | Contents |
|------|----------|
| `sockLib.html` | Socket library API (`socket()`, `bind()`, `listen()`, `accept()`, `connect()`) — used in `SendReceiveSupport.c` |
| `msgQLib.html` | Message queue API (`msgQCreate()`, `msgQSend()`, `msgQReceive()`) — used in `inLoop.st` queuing |
| `msgQShow.html` | Message queue diagnostic/show functions |
| `msgQSmLib.html` | Shared-memory message queue API |
| `hostLib.html` | Host name/IP resolution (`hostGetByName()`) — used for IOC network setup |
| `Tornado-Guide.pdf` | Tornado 2 IDE user guide |
| `Vx-Progr-Guide1.pdf` | VxWorks Programmer's Guide (task model, I/O, networking) |
| `Tornado-getStart.pdf` | Getting Started guide |

Useful when reading `SendReceiveSupport.c`, `inLoop.st`, or any networking/IPC code in the IOC.

---
*Source: `DGS_tools_pack/vxworks/README.md`. Created: 2026-04-05.*

---

## FIFO Readout Internals

> 📄 **See [`vxworks_fifo_readout.md`](vxworks_fifo_readout.md)** for full detail on:
> - DMA Buffer Architecture (buffer pool, msgQ queues, trigger throttle)
> - Trigger FIFO Readout (`readTrigFIFO.c`, `CheckAndReadTrigger()`, FIFO index map)
> - Type-F synthetic headers (both trigger and digitizer paths)

_Split to separate file 2026-04-20 to keep `vxworks.md` under 500 lines._

---

## Connections to Other Subsystems

- **ioc/** — `gretDet.munch` ends up in `ioc/bin/`; `dgsDriver.dbd` ends up in `ioc/dbd/`
- **fpga/** — drivers in `dgsDrivers/` communicate with DIG/RTRG/MTRG firmware via VME registers
- **ANLDAQ** — EPICS PVs exposed by the IOC are the interface ANLDAQ uses
- **EPICS.md** — EPICS primer: record types, CA protocol, device support, .dbd/.db/.cmd roles
- **EPICS_asyn.md** — asyn driver internals used by the VME IOC records
- **vxworks_migration.md** — Solaris/con6 → Ubuntu 24 migration notes
- **troubleshooting.md** — IOC connectivity issues, SYNC bit gotcha, FIFO problems

---

## Quick Notes

- EPICS 3.14 (not 7.0) — VxWorks IOC uses the older EPICS. Do not mix with EPICS 7.0 paths.
- Cross-compiler is NOT in git — must download from JLab before building
- `migration.md` documents how this was ported from Solaris/con6 to Ubuntu
- Each VME crate loads the same `gretDet.munch` but uses a different boot script with crate-specific parameters

---

## devGData.c — Legacy VME_IO Device Support (Removed 2025-04-21)

`devGData.c` was an EPICS device support module for `VME_IO`-type records. It provided three device sets:
- **`devAiGData`** — `ai` records; signal codes 0x01–0x0b mapped to per-board channel rates, 0xc to card latency, 0xd/0xe to KB sent/read diff. All handlers were **commented out** — effectively a no-op.
- **`devBoGData`** — `bo` records; writing to signal=1 set `daqBoards[card].EnabledForReadout`. This was the original PV-controlled mechanism for enabling boards in inLoop.
- **`devAoGData`** — `ao` records; simply stored `pao->rval` to `pao->val` — a pass-through for state machine use.

**Removed from the build:** `dgsDrivers/dgsDriverApp/src/Makefile:L63` — `JTA 20250421: removed`. The file still exists in the source tree but is no longer compiled into `gretDet.munch`.

**Current mechanism for `EnabledForReadout`:** `devGVME.c` initializes it to 0 in `devGVMECardInit()`. `inLoop.st` maps `VME{CRATE}:{BOARD}:CS_Ena` PVs (monitored via `DECLMON`) to per-board enable flags via `SetupBoardAddresses()` — replacing the old VME_IO PV approach. The `CS_Ena` PV pattern is documented in `ANLDAQ.md`.

✅ verified 2026-04-19 — `devGData.c` full read + `dgsDriverApp/src/Makefile:L63` + `inLoop.st:L167,535`

---

## Utility / Support Modules (Minor Files)

Several small helper files in `dgsDriverApp/src/` are not part of the main data path but provide supporting functionality:

### equalSub.c — EPICS Equality Subroutine

_Author: unknown; included in VxWorks build_ ✅ verified 2026-04-20 — `equalSub.c:L1-72`

An EPICS `sub` record device support function. It reads up to 12 input fields (A–L) from a `subRecord` and tests whether all non-CONSTANT links are equal (within a precision of `prec` decimal places).

- `equalSubInit()` — init routine; no-op, returns 0.
- `equalSub()` — processing routine:
  - Reads `prec` (0–3) to determine precision (multiply by 10^prec before integer compare)
  - Iterates through inputs A–L, skipping CONSTANT links
  - Compares all linked values against the first linked value
  - Sets `val` to the comparisand value; returns 0 if all equal, -1 if any differ or no real link found
- Registered to IOC shell as `equalSubInit` / `equalSub` iocsh commands
- Used with EPICS `sub` records in `.db` files to detect when a set of related PVs are all equal

### restoreSub.c — EPICS Restore Subroutine

_Restores PV values from a `.sav` file via `fdbrestore()`_ ✅ verified 2026-04-20 — `restoreSub.c:L1-99`

Another EPICS `sub` record device support. Provides a triggered, asynchronous save-file restore:

- `devGDigSetRestFile(filename)` — sets the save file name (default: `"default.sav"`; max 79 chars)
- `devGDigRestInit(psub)` — init: allocates a `rcallback` struct, wires up EPICS callback chain to `restSubCallback()`
- `devGDigRestore(psub)` — processing: sets `pact = TRUE` (marks record busy), schedules callback; callback calls `fdbrestore(filename)` and re-processes the record with the returned status code in `psub->c`
- Pattern: async EPICS subroutine — correct way to call a blocking function from a `sub` record without stalling the scan thread
- Registered to IOC shell as `devGDigRestInit` / `devGDigRestore`
- Used when the IOC needs to restore a snapshot of PV values after reboot or reconfiguration

### profile.c / profile.h — VxWorks Performance Profiler

_Author: Michael Oberling_ ✅ verified 2026-04-20 — `profile.h:L1-6`, `profile.c:L1-407`

A VxWorks-specific CPU time profiling library for measuring how long different code blocks take:

- **`NUM_PROFILE_COUNTERS`** named timers, each with: start/stop times, accumulated total, last-cycle delta, execution count, prescale factor, state machine (DISABLED / RUNNING / STOPPED)
- `PAUSABLE_PROFILER` mode: supports `pause_profile_counter()` / `resume_profile_counter()` to exclude sleep time; hooks VxWorks task-switch callback (`profile_counter_task_switch_hook`) to track context switches mid-profile
- **API:**
  - `init_profile_counters(clock_frequency)` — must be called once at startup
  - `init_profile_counter(idx, name, prescale)` — name and configure a counter; prescale=N means timer only actually runs 1-in-N starts (reduces overhead)
  - `start_profile_counter(idx)` / `stop_profile_counter(idx)` — bracket the code to profile
  - `print_profile_summary()` — prints all counters: name, total time, % of run time, executions/sec
  - `get_profile_counter_percent_time()`, `get_profile_counter_exec_second()`, etc. — read individual stats
- VxWorks timebase: **33.3 MHz** (`PROFILE_TICK_FREQUENCY = 33333333.333 Hz`) ✅ verified 2026-04-21 — `DGS_DEFS.h:L64`; passed as `clock_frequency` arg to `init_profile_counters()`
- Used in the IOC to measure timing of inLoop/outLoop data path sections during development/debugging
- Not directly exposed as EPICS PVs; output via VxWorks console (`printf`)

### MergedAsynDigParams.c — DIG Asyn Parameter Registration

_Source: `dgsDrivers/dgsDriverApp/src/MergedAsynDigParams.c` (672 lines, all `createParam()` calls)_ ✅ verified 2026-04-22 — `MergedAsynDigParams.c:L1-222` (222 `createParam()` calls)

This file is **not a standalone translation unit** — it is `#include`-d directly into the asyn digitizer driver (`drvAsynDigitizer.c`) as a body of `createParam()` calls executed in the driver constructor. It registers every DIG-related PV name with the asyn framework, mapping string names → integer param IDs.

**What it registers:** 222 asyn parameters in total, covering all DIG EPICS PV names. Each `createParam()` call has the form:
```c
createParam("reg_led_threshold0", asynParamUInt32Digital, &reg_led_threshold0);
```
All parameters use type `asynParamUInt32Digital`.

**Naming conventions:**
- `reg_<name>N` — writable register, channel N (0–9). E.g. `reg_led_threshold0`–`reg_led_threshold9`
- `regin_<name>N` — read-back-only (input) register, channel N
- Board-level (no digit suffix): e.g. `reg_programming_done`, `regin_board_id`, `SERIAL_NUMBER`, `vme_clk_ctrl`

**Parameter groups (unique base names, 35 total):**

| Group | Per-channel? | Purpose |
|-------|-------------|----------|
| `reg_channel_control` | ✅ (×10) | Per-channel mode/control bits |
| `reg_channel_pulsed_control` | ✅ | Pulsed-write channel control |
| `reg_led_threshold` | ✅ | LED discriminator threshold |
| `reg_CFD_fraction` | ✅ | CFD fraction (stored as integer ×1000) |
| `reg_raw_data_delay` | ✅ | Waveform delay (samples) |
| `reg_raw_data_length` | ✅ | Waveform length (samples) |
| `reg_d_window` / `reg_k_window` / `reg_m_window` / `reg_d3_window` | ✅ | Trapezoidal filter D/K/M/D3 window |
| `reg_disc_width` | ✅ | Discriminator pulse width |
| `reg_baseline_delay` | ✅ | Baseline averaging delay |
| `reg_downsample_holdoff` | ✅ | Downsample hold-off |
| `reg_led_control` | ✅ | LED control bits |
| `reg_holdoff_control` | ✅ | Hold-off control bits |
| `reg_veto_gate_width` | ✅ | Veto gate width |
| `reg_p1_window` / `reg_p2_window` | ✅ | Pileup detection windows |
| `reg_dac` | ✅ | Per-channel DAC |
| `reg_diag_channel_input` | ✅ | Diagnostic channel input select |
| `reg_diag_mux_control` | ✅ | Diagnostic mux control |
| `reg_sd_config` | ✅ | Signal-detect config |
| `reg_trigger_config` | ✅ | Per-channel trigger config |
| `reg_external_disc_mode` | ✅ | External discriminator mode |
| `reg_ila_config` | ✅ | ILA (in-logic analyzer) config |
| `regin_disc_count` | ✅ | Discriminator event count (readback) |
| `regin_ahit_count` | ✅ | Accepted-hit count (readback) |
| `regin_led_state` | ✅ | LED state readback |
| `regin_hilo_` / `regin_hihilolo_` | ✅ | HiLo / HiHiLoLo alarm readbacks |
| `regin_phase_offset_a/b/c` | ✅ | SERDES phase offsets |
| `regin_serdes_phase_value` | ✅ | SERDES phase value |
| `regin_phase_value` | ✅ | Phase value readback |
| `regin_phase_errors` | ✅ | Phase error count |
| `regin_board_id` | board | Board identity register |
| `regin_code_revision` / `regin_code_date` | board | Firmware version readbacks |
| `regin_hardware_status` | board | Hardware status bits |
| `regin_accepted_event_count` | board | Global accepted event counter |
| `regin_dropped_event_count` | board | Global dropped event counter |
| `regin_lat_timestamp_lsb/msb` | board | Latched timestamp (split 16-bit) |
| `regin_live_timestamp_lsb/msb` | board | Live timestamp (split 16-bit) |
| `regin_ts_err_count` | board | Timestamp error count |
| `reg_ts_err_count_ctrl` | board | Timestamp error counter control |
| `reg_master_logic_status` | board | Master logic status register |
| `reg_programming_done` | board | FPGA programming complete flag |
| `reg_external_discriminator_src` | board | External discriminator source |
| `reg_vme_ext_delay` | board | VME extension delay |
| `reg_user_package_data` | board | User package data |
| `reg_win_comp_min` / `reg_win_comp_max` | board | Window comparator range |
| `reg_rj45_spare_dout_control` | board | RJ45 spare digital output |
| `SERIAL_NUMBER` | board | Board serial number |
| `vme_clk_ctrl` | board | VME clock control |
| `vme_gp_ctrl` | board | VME general-purpose control |
| `VME_MON_STATUS` | board | VME monitor status |

**Role in the driver:** `MergedAsynDigParams.c` is `#include`-d in the `drvAsynDigitizer.c` constructor body, so all 222 params are registered when the driver initializes. The integer handles (e.g. `&reg_led_threshold0`) are member variables of the driver class defined in `asynDigParams.h` / `MergedAsynDigParams.h`. After `createParam()`, the driver's `readInt32`/`writeInt32` dispatch table routes `caput`/`caget` traffic to the matching VME register read/write.

**Cross-reference:** `drvAsynDigitizer.c` → VME register map in `knowledgeBase/VME_registers.md`; PV naming convention in `knowledgeBase/ANLDAQ.md`.

---

### FlashMaintenance.c — VME Flash Register Constants

_Defines VME register addresses for flash memory access on the MVME5500_ ✅ verified 2026-04-20 — `FlashMaintenance.c:L1-32`

This file is a constants-only C file — no functions, just `const int` definitions for the VME register offsets used for FPGA flash programming:

| Constant | Offset | Description |
|----------|--------|-------------|
| `fpga_ctrl_reg` | 0x0900 | FPGA control register |
| `fpga_status_register` | 0x0904 | FPGA status register |
| `vme_aux_status` | 0x0908 | VME auxiliary status |
| `vme_config_control` | 0x090C | VME configuration control |
| `fpga_gp_ctl` | 0x0910 | FPGA general-purpose control |
| `config_start` | 0x0918 | FPGA configuration start |
| `config_stop` | 0x091C | FPGA configuration stop |
| `fpga_version` | 0x0920 | FPGA version register |
| `full_code_revision` | 0x0924 | Full code revision |
| `code_date_VME` | 0x0928 | Code date (primary) |
| `vme_sandbox_a–d` | 0x0930–0x093C | Sandbox test registers |
| `code_date_2` | 0x0940 | Code date (secondary) |
| `full_revision2` | 0x0944 | Full revision (secondary) |
| `vme_dtack_delay` | 0x0948 | VME DTACK delay |
| `flash_address` | 0x0980 | Flash address register |
| `flash_rd_wrt_autoinc` | 0x0984 | Flash read/write with auto-increment |
| `flash_rd_wrt_no_autoinc` | 0x098C | Flash read/write without auto-increment |

Additional constants: `FLASH_BLOCK_SIZE = 128 kB`, `FLASH_BLOCKS = 128` (total flash = 16 MB), `FLASH_BUFFER_BYTES = 32`.

These registers match the VME FPGA firmware register map for flash access — used during FPGA firmware update. Cross-reference with `knowledgeBase/VME_registers.md` for the full register map context.

**Note:** `FlashMaintenance.c` is constants-only. The actual flash functions (`ProgramFlash`, `VerifyFlash`, `EraseFlash`, `DownloadFlash`, `ConfigureFlash`) all live in **`devGVME.c`**. ✅ verified 2026-04-23 — `devGVME.c:L363–1083`

---

### devGVME.c — VME Board Management Layer

_1,083-line C file: core VME board abstraction, VMERead32/VMEWrite32 primitives, board type detection, FPGA flash programming, and IOCShell command registration._ ✅ verified 2026-04-23 — `devGVME.c`

#### Global State

| Variable | Type | Description |
|----------|------|-------------|
| `daqBoards[GVME_MAX_CARDS]` | `struct daqBoard[7]` | Per-slot board state array (7 slots per VME crate) |
| `BoardTypeNames[16][30]` | `char[][]` | String names for board type codes 0–15 |
| `OL_Hdr_Chk_En` | `unsigned short` | outLoop header-check enable (default 1) |
| `OL_TS_Chk_En` | `unsigned short` | outLoop timestamp-check enable (default 1) |
| `OL_Deep_Chk_En` | `unsigned short` | outLoop deep-check enable (default 1) |
| `OL_Hdr_Summ_En` | `unsigned short` | outLoop header summary enable (default 0) |
| `OL_Hdr_Summ_PS` | `unsigned int` | Header summary prescale (default 0x1000) |
| `OL_Hdr_Summ_Evt_PS` | `unsigned int` | Event summary prescale (default 0x100) |

#### `struct daqBoard` (defined in `DGS_DEFS.h`)

The central per-board data structure, one per VME slot:

| Field | Type | Description |
|-------|------|-------------|
| `vmeRegisters[0x24]` | `struct daqRegister[]` | 0x24 per-register structs (addr + mutex + tick + copy + dibs) |
| `base32` | `volatile uint*` | Local CPU address mapped to VME base of this board |
| `FIFO` | `volatile uint*` | Pointer to the board's hardware FIFO |
| `vmever` | `unsigned short` | VME FPGA code_revision value (bits 31:16 of reg 0x920) |
| `rev` / `subrev` | `unsigned int` | Main FPGA major/minor revision |
| `mainOK` | `unsigned short` | Flag: main FPGA responded at init |
| `board` | `unsigned short` | VME slot number |
| `EnabledForReadout` | `unsigned short` | inLoop enable flag (set by `CS_Ena` PV) |
| `DigUsrPkgData` | `int` | Type-F header payload for digitizer |
| `TrigUsrPkgData` | `int` | Type-F header payload for trigger (added 2022-07-13) |
| `router` | `unsigned short` | Router flag |
| `board_type` | `unsigned short` | Board type index (0–15, see `BrdType_*` defines) |

**Board type codes** (bits 11:8 of `code_revision` for trigger boards; arbitrary for digitizers):

| Code | Constant | Board |
|------|----------|-------|
| 0 | `BrdType_NO_BOARD` | No board present |
| 1 | `BrdType_GRETINA_RTRIG` | GRETINA Router Trigger |
| 2 | `BrdType_GRETINA_MTRIG` | GRETINA Master Trigger |
| 3 | `BrdType_LBNL_DIG` | LBNL Digitizer |
| 4 | `BrdType_DGS_MTRIG` | DGS Master Trigger |
| 6 | `BrdType_DGS_RTRIG` | DGS Router Trigger |
| 8 | `BrdType_MYRIAD` | MγRIAD |
| 12 | `BrdType_ANL_MDIG` | ANL Master Digitizer |
| 13 | `BrdType_ANL_SDIG` | ANL Slave Digitizer |
| 14 | `BrdType_MAJORANA_MDIG` | Majorana Master Digitizer |
| 15 | `BrdType_MAJORANA_SDIG` | Majorana Slave Digitizer |

#### Key Functions

**`InitializeDaqBoardStructure()`** — Zeroes all 7 `daqBoards[]` slots. Must be called before `devGVMECardInit()`. Note: a bug existed (fixed 2023-09-21) where the loop used `<=` instead of `<`, causing an off-by-one write past the array.

**`devGVMECardInit(int cardno, int slot)`** — Initializes one VME board slot:
1. Converts VME slot number → base address: `base = slot << 20` (VME64x A32 addressing)
2. Calls `sysBusToLocalAdrs()` to map VME bus address to local CPU address space (space=0x0a normal, 0x0b on RIO3 processor boards)
3. Performs a `devReadProbe()` to read the VME FPGA version register at offset 0x248 (= 0x920 >> 2, longword-indexed); extracts bits 31:16 as `vmever`
4. Advances `newbase` by `0x900/4` to the start of the VME FPGA register map
5. Allocates one `epicsMutexCreate()` semaphore per register slot (0x24 total, covering 0x900–0x98C)
6. Exposed to IOCShell as `devGVMECardInit(cardno, slot)`

**`VMEWrite32(int bdnum, int regaddr, unsigned int data)`** — Locks `vme_driver_mutex`, converts register address to longword pointer via `daqBoards[bdnum].base32 + regaddr/4`, writes the value, unlocks. This is the universal VME write primitive used by all flash and driver functions.

**`VMERead32(int bdnum, int regaddr)`** — Same pattern as write: mutex-protected read; result stored in global `VMERead32TempVal` (needed because IOCShell call functions can't return values directly). Exposed to IOCShell as `VMERead32(bdnum, regaddr)` — prints value to console.

#### Flash Programming Functions

All flash functions hold `vme_driver_mutex` for their entire duration — **ctrl-X and ctrl-C cannot interrupt them**.

The flash chip uses byte-swapped word ordering relative to Linux file byte ordering. All read and write paths include a 4-byte endian swap:
```c
file_word  = ((raw & 0xFF000000) >> 24)  // bits 31:24 → 07:00
           + ((raw & 0x00FF0000) >>  8)  // bits 23:16 → 15:08
           + ((raw & 0x0000FF00) <<  8)  // bits 15:08 → 23:16
           + ((raw & 0x000000FF) << 24); // bits 07:00 → 31:24
```

Flash geometry: 128 blocks × 128 kB = 16 MB total. Each programming operation targets one **bank** (32 blocks = 4 MB), selected by `address_control` (0 = lower bank, 1 = upper bank).

| Function | IOCShell Args | Description |
|----------|---------------|-------------|
| `VerifyFlash(bdnum, addr_ctrl, stop_on_err, fname)` | 4 | Compare flash contents to .bin file. Reads 32-byte chunks, byte-swaps, compares. Reports mismatch count. |
| `EraseFlash(bdnum, addr_ctrl)` | 2 | Block-erases one 4 MB flash bank. Issues cmd 0x20 (block erase) + 0xD0 (confirm) per block. Polls status reg 0x904 bit 0x80 (busy). Timeout at 10,000 polls per block. |
| `ProgramFlash(bdnum, addr_ctrl, fname)` | 3 | Calls EraseFlash first, then programs from .bin file in 32-byte write-buffer commands (cmd 0xE8). Two-stage poll: buffer-write status then NV array commit. |
| `DownloadFlash(bdnum, addr_ctrl, fname)` | 3 | Reads one 4 MB flash bank back to a file (byte-swapped to match original .bin layout). Mirrors ProgramFlash in reverse. |
| `ConfigureFlash(bdnum, addr_ctrl)` | 2 | Commands the VME FPGA to reconfigure the main FPGA from flash. Writes to `vme_config_control` (0x090C). Polls status 0x904 bit 0x0002 (config complete). Timeout at 40 × 10-tick cycles. Reports `DoneError`, `ConfigComplete`, `InitLowErr`, `InitHighErr` status bits on timeout. |

All five functions are registered with the IOCShell via EPICS `iocshRegister()` / `epicsExportRegistrar()` so they can be called from the VxWorks IOC shell prompt.

#### `DGS_DEFS.h` — Central Type and Constants Header

All modules share types/constants defined here (moved from `devGVME.h` in 2020-06-11 refactor):

- **`rawEvt` struct** — Buffer descriptor: `id`, `datapcrosscheck`, `board`, `len`, `data*`, `owner_enum`, `board_type`, `data_type`
- **`owner_enum`** — Buffer ownership state: `OWNER_Q_FREE(1)` → `OWNER_INLOOP(2)` → `OWNER_Q_WRITTEN(3)` → `OWNER_OUTLOOP(4)` → `OWNER_Q_SENDER(5)` → `OWNER_SENDER(6)`
- **`evtServerRetStruct` / `ResponseMsg`** — TCP server response header: `type`, `recLen` (deprecated for DGS), `status`, `recs`
- **`ReqMsg`** — TCP receiver request (4-byte int union)
- **Buffer size constants** (MV5500 only): `RAW_BUF_SIZE=1MB`, `MAX_DIG_RAW_XFER_SIZE=512kB`, `DMA_CHUNK_SIZE_IN_BYTES=64kB` (actual max DMA size; the 512kB limit is of the FIFO, not DMA)
- **Profiling counters** (9 counters, normally disabled via `NO_PROFILING`)
- **outLoop tuning**: `SENDER_BUF_BYPASS_THRESHOLD = RAW_Q_SIZE × 0.5`, `MAX_EVENTS_TO_CHECK_PER_BUFFER = 128`
- **Type-F generation flags**: compile-time `#define`/`#undef` to enable/disable empty, error, EOD Type-F headers

---

## State Machines & Runtime Drivers

> 📄 **See [`vxworks_state_machines.md`](vxworks_state_machines.md)** for full detail on:
> - **inLoop.st** — VME FIFO readout state machine (data acquisition, FIFO polling, board enable)
> - **outLoop.st** — data validation and buffer routing state machine
> - **MiniSender.st** — TCP data send state machine (port 9001)
> - **Port 9010 On-Demand FIFO Grabber** (planned, not implemented)
> - **Trigger board drivers** (asynTrigCommonDriver, asynTrigRouterDriver, asynTrigMasterDriver, RTRG/MTRG)
> - **vmeDriverMutex** — shared VME bus mutex for flash programming synchronization
> - **QueueManagement.c** — three-queue buffer pool (qFree/qWritten/qSender)

_Split to separate file 2026-04-23 to keep `vxworks.md` under 650 lines._

---

## Cross-References

- `knowledgeBase/ioc.md` — IOC config, boot scripts, firmware versions, MVME5500 setup
- `knowledgeBase/vxworks_migration.md` — Detailed migration notes from Solaris/con6 to Ubuntu 24
- `knowledgeBase/vxworks_fifo_readout.md` — DMA buffer architecture, trigger FIFO readout, Type-F headers
- `knowledgeBase/vxworks_state_machines.md` — inLoop/outLoop/MiniSender state machines, trigger drivers, queue management
- `knowledgeBase/EPICS_asyn.md` — asyn driver internals: port model, worker threads, write flow
- `knowledgeBase/VME_registers.md` — VME register addresses used by the IOC driver
- `knowledgeBase/fpga.md` — FPGA firmware overview; the firmware binaries loaded by VxWorks
- `knowledgeBase/ANLDAQ.md` — High-level pipeline overview (inLoop/outLoop/MiniSender data flow diagram + key PVs)
- `knowledgeBase/ANLDAQ_tcpReceiver.md` — tcpReceiverMT protocol; the TCP receiver MiniSender connects to
- `knowledgeBase/deep_fpga_RTRG.md` / `knowledgeBase/deep_fpga_MTRG_MAIN.md` — RTRG/MTRG FPGA firmware

*Created: 2026-04-05 | Last reviewed: 2026-04-23*
