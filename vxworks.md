# VxWorks Cross-Compiler for DGS IOC (MVME5500)

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
- [DMA Buffer Architecture — VME Readout Internals](#dma-buffer-architecture--vme-readout-internals)
- [Trigger FIFO Readout — readTrigFIFO.c](#trigger-fifo-readout--readtrigfifoc)
  - [FIFO Index Map](#fifo-index-map)
  - [FIFO Status Register](#fifo-status-register)
  - [CheckAndReadTrigger() Logic](#checkandreadtrigger-logic-in-inloopsupportc)
  - [transferTrigFifoData() Flow](#transfertrigfifodata-flow)
  - [Key Size Constants](#key-size-constants-from-dgs_defsh)
  - [Type-F Headers](#type-f-headers)
  - [Digitizer Type F Headers](#digitizer-type-f-headers-digitizertypefheader-in-readdigfifoc)
- [Connections to Other Subsystems](#connections-to-other-subsystems)
- [Quick Notes](#quick-notes)
- [Port 9010 On-Demand FIFO Grabber (Planned)](#port-9010-on-demand-fifo-grabber-planned--not-yet-implemented)
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

## DMA Buffer Architecture — VME Readout Internals
_Source: `DGS_SVN/dgs/Documentation/Formal/Software/howTheSenderWorks.docx` (T. Madden, APS-XSD Detector Group)_

### FIFO Poll → DMA → Buffer Queue pipeline

1. **`inLoop.st`** (per digitizer board, runs in VxWorks EPICS state machine): polls the digitizer FIFO status register at `*(board_base + 1)`. Returns one of: `Empty`, `HalfFull`, `Some`, `Wait`, `AlmostEmpty`.
2. On data available: calls `serviceOneBuffer()` in `readDigFIFO.c`, which:
   - Acquires **`DMASem`** (epicsEventFull semaphore) — DMA library in VxWorks 5.x is **not thread-safe**, so all 4 digitizers per crate share a single mutex.
   - Takes a free buffer from **`qFree`** via `msgQReceive()` (VxWorks message queue, FIFO order).
   - Initiates **DMA transfer** from digitizer VME FIFO directly into IOC memory (no CPU copy).
   - Posts the filled buffer onto **`qWritten`** via `msgQSend()` for the sender to pick up.
3. **`SendReceiveSupport.c`** drains `qWritten` → sends data to Linux cluster over TCP (port 9001) → returns buffers to `qFree` via `putFreeBuf()`.

**Three VxWorks msgQ queues** (all `MSG_Q_FIFO`, capacity `RAW_Q_SIZE=200`, message size = `sizeof(rawEvt*)` = 4 bytes — they pass *pointers*, not data):
| Queue | Role |
|-------|------|
| `qFree` | Available (unallocated) buffer slots |
| `qWritten` | Filled buffers waiting for TCP sender |
| `qSender` | (Legacy/reserved — exists but lightly used in current code) |

✅ verified 2026-04-11 — `QueueManagement.c:L83-97` (`msgQCreate` calls) + `readDigFIFO.c:L124-212` (receive from qFree, send to qWritten)

### Buffer Pool
- **200 buffers** total, shared across all 4 digitizers in a crate ✅ verified 2026-04-08 — `DGS_DEFS.h:L48` (`RAW_Q_SIZE = 200`, changed from 400 on 2023-04-12 JTA)
- Each buffer: **1 MB** (`RAW_BUF_SIZE`) ✅ verified 2026-04-08 — `DGS_DEFS.h:L34` (changed from 512 KB on 2023-04-12 JTA)
- Queue size: **`RAW_Q_SIZE = 200`** (defined in `DGS_DEFS.h`; previously 400 before April 2023)
- Each buffer has a **reference counter** — zero = free, non-zero = in use

> **Note:** The salvaged notes (`20180924_notes.txt`) document the older values (RAW_Q_SIZE=400, RAW_BUF_SIZE=512KB). These were doubled/halved in April 2023 — same total memory (~200MB), different trade-off.

### Trigger Throttle (software fallback)
- If buffers in Return Queue fall below **1/3** of `RAW_Q_SIZE` (i.e., <67 free with current Q=200), `TrigCon.st` asserts `TrigInhL` and `TrigInhD` via EPICS CA.
- This is a **software path** — latency can be 10+ ms at high rates. Hardware FIFO throttle (half-full flag → RTRG throttle line) is the primary fast mechanism.

### Garbage Collection (optional, compile-time)
- If enabled: when Return Queue falls below **100 buffers** (50%) or **25 buffers** (12.5%), a background process scans all 200 buffers, checks reference counters, and returns free ones to `gDigRawRetQ`.

---

## Trigger FIFO Readout — `readTrigFIFO.c`
_Source: `dgsDrivers/dgsDriverApp/src/readTrigFIFO.c`, `inLoopSupport.c` — updated 2026-04-16_

The trigger modules (MTRG and RTRG) have their own separate FIFO readout path, distinct from `readDigFIFO.c`. The key entry point from inLoop is `CheckAndReadTrigger()` (in `inLoopSupport.c`), which calls `transferTrigFifoData()` from `readTrigFIFO.c`.

### FIFO Index Map

Each trigger module exposes up to 16 FIFOs, accessed via VME address offsets:

| Index | VME Offset | Name |
|-------|-----------|------|
| 0 | 0x0160 | MON FIFO 1 |
| 1 | 0x0164 | MON FIFO 2 |
| 2 | 0x0168 | MON FIFO 3 |
| 3 | 0x016C | MON FIFO 4 |
| 4 | 0x0170 | MON FIFO 5 |
| 5 | 0x0174 | MON FIFO 6 |
| 6 | **0x5000** | MON FIFO 7 (primary DAQ FIFO — **moved 2025-05-28**) |
| 7 | 0x017C | MON FIFO 8 |
| 8–15 | 0x0180–0x019C | CHAN FIFOs 1–8 |

> **Note:** MON FIFO 7 (index 6) was previously at `0x0178`, moved to `0x5000` in firmware update 2025-05-28 (JTA). This is the main DAQ data FIFO for most applications.

✅ verified 2026-04-16 — `readTrigFIFO.c:L78-96` (FIFO_READ_ADDRESS array + comments) + `inLoopSupport.c:L678-694` (FIFO index cheat sheet)

### FIFO Status Register

- **MON_FIFO_STATE** at offset `0x01B4` (MTRG) / `0x01A0` — 16-bit register; 2 bits per FIFO:
  - Bit `2n+1` = Full flag for FIFO n
  - Bit `2n+0` = Empty flag for FIFO n
- **CHAN_FIFO_STATE** at offset `0x01A4`
- **MON FIFO 7 live depth** at `0x0154` (live counter, in longwords)
- **MON FIFO 7 latched depth** at `0x01AC` (latched at event boundaries — used for readout sizing)

### `CheckAndReadTrigger()` Logic (in `inLoopSupport.c`)

1. Read FIFO status register; extract `FullFlag` and `EmptyFlag` for the requested `FifoNum`.
2. **Overflow check** (MON_FIFO7 only): if full → send `TriggerTypeFHeader(mode=2)` error header, call `ClearTrigFIFO()`, return `-1`.
3. **Empty check**: if empty and `SendNextEmpty` flag set → send `TriggerTypeFHeader(mode=0)` informational header, return `0`.
4. **Depth determination**: MON_FIFO7 reads latched depth from `0x01AC`; all other FIFOs assume fixed 256 longwords.
5. **Transfer**: call `transferTrigFifoData(board, numLongwords, FifoNum, queueFlag, &nBytesXferred)` in a retry loop (`NoBufferAvail`).

### `transferTrigFifoData()` Flow

- Gets a free buffer from `qFree` queue (same shared pool as digitizer readout).
- Stores FIFO index as `rawBuf->data_type` — lets `outLoop` distinguish which trigger FIFO the data came from.
- DMA reads from `bd->base32 + FIFO_READ_ADDRESS[FifoNum]/4` in chunks of `DMA_CHUNK_SIZE_IN_BYTES = 0x10000` (64 KB) — chunked because empirical testing found DMA errors on transfers >0x10000 bytes (discovered 2025-06-07, JTA).
- Validates data: first word must be `0x0000AAAA`; mismatch prints error but does not abort.
- Posts filled buffer to `qWritten` for TCP sender.

### Key Size Constants (from `DGS_DEFS.h`)

| Constant | Value | Notes |
|----------|-------|-------|
| `TRIG_MON_FIFO_SIZE` | 4 KB | Max transfer for MON FIFOs 1–6, 8 |
| `MAX_TRIG_RAW_XFER_SIZE` | 256 KB (4 × 65536 bytes) | Max for MON FIFO 7 |
| `DMA_CHUNK_SIZE_IN_BYTES` | 0x10000 (64 KB) | DMA chunk limit (2025-06-07) |
| `MAX_DIG_RAW_XFER_SIZE` | 512 KB | Digitizer FIFO max |

✅ verified 2026-04-16 — `DGS_DEFS.h:L36-53`

### Type-F Headers

When a trigger FIFO is empty, overflowed, or at end-of-run, `TriggerTypeFHeader()` generates a synthetic 4-word GEB-format header and pushes it to `qWritten`. The 4-word format:

| Word | Content |
|------|---------|
| 0 | `0xAAAAAAAA` (GEB sync word) |
| 1 | `GeoAddr[31:27] / PacketLen[26:16] / UserPkgData[15:4] / ChannelID[3:0]` |
| 2 | LED Timestamp[31:0] (latched via pulsed control bit 4 at `0x8E0`) |
| 3 | `HeaderLen[31:26] / EventType[25:23] / HeaderType[19:16] / TS[47:32]` |

**Mode values:**
- `mode=0` (empty): ChannelID = `0xE` (Empty); HeaderType = `0xF` (informational)
- `mode=1` (end-of-run): ChannelID = `0xD` (Done); HeaderType = `0xF`
- `mode=2` (overflow/underflow): ChannelID = `0xF` (Error); EventType = 2 (underflow)

Controlled by compile-time `#ifdef` flags: `INLOOP_GENERATE_EMPTY_TYPEF`, `INLOOP_GENERATE_EOD_TYPEF`, `INLOOP_GENERATE_ERROR_TYPEF`.

✅ verified 2026-04-16 — `readTrigFIFO.c:L380-545` (TriggerTypeFHeader switch cases) + `inLoopSupport.c:L725-782` (CheckAndReadTrigger)

### Digitizer Type F Headers (`DigitizerTypeFHeader()` in `readDigFIFO.c`)

The **digitizer** side has a parallel mechanism: `DigitizerTypeFHeader()` generates synthetic 4-word GEB-format packets when a digitizer FIFO is empty, at end-of-run, or has overflowed/underflowed. Same 4-word format as trigger Type F:

| Word | Content |
|------|---------|
| 0 | `0xAAAAAAAA` (GEB sync word) |
| 1 | `GeoAddr[31:27] / PacketLen[26:16] / UserPkgData[15:4] / ChannelID[3:0]` |
| 2 | LED Timestamp[31:0] (latched via pulsed control at reg `0x40C`, bit 15; read from `0x484`) |
| 3 | `HeaderLen[31:26] / EventType[25:23] / HeaderType[19:16] / TS[47:32]` (TS from `0x488`) |

**Mode values (digitizer-specific):**
- `mode=0` (FIFO empty / update): ChannelID = `0xE` (Empty); EventType = 0 (informational); HeaderType = `0xF`
- `mode=1` (end-of-run / end-of-data): ChannelID = `0xD` (Done); EventType = 0; HeaderType = `0xF`
- `mode=2` (FIFO overflow): ChannelID = `0xF` (FIFO Error); EventType = 1 (overflow); HeaderType = `0xF`; word3 = `0x0C200000 + ...`
- `mode=3` (FIFO underflow): ChannelID = `0xF` (FIFO Error); EventType = 2 (underflow); HeaderType = `0xF`; word3 = `0x0D000000 + ...`

**Key difference vs. trigger:** Timestamp latch uses a **different register** (`0x40C` bit 15, self-clearing) and timestamp is read from `0x484` (LS) / `0x488` (MS), vs. trigger which uses `0x8E0` bit 4.

**UserPkgData** (bits [15:4] of word 1): taken from `daqBoards[BoardNumber].DigUsrPkgData` (low 12 bits). This is board-specific user package data read from the digitizer.

**`PacketLen`** is always 3 (the 3 words after word 0). **`HeaderLen`** is always 3 (same).

Controlled by compile-time flags: `INLOOP_GENERATE_EMPTY_TYPEF`, `INLOOP_GENERATE_EOD_TYPEF`, `INLOOP_GENERATE_ERROR_TYPEF`. If a flag is absent, the buffer is simply returned to the free queue (no data pushed).

A helper function `PushTypeFToQueue()` sets `rawBuf->board`, `rawBuf->len = 16`, and calls `putWrittenBuf()`. It increments `FBufferCount` (defined in `inLoopSupport.c`) on each push.

✅ verified 2026-04-17 — `readDigFIFO.c:L228-534` (DigitizerTypeFHeader full switch cases + word-by-word comments)

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

## Port 9010 On-Demand FIFO Grabber (Planned / Not Yet Implemented)

_Source: `vxworks/On-Demand-FIFO-Grabber-Plan.md` — design plan only; `fifoGrabber.c` does not exist yet as of 2026-04-18_

A planned standalone TCP diagnostic service that allows raw FIFO data to be grabbed from a specific digitizer board **without starting the full DAQ pipeline**. Currently, FIFO data is only accessible via the full chain: `Online_CS_StartStop` PV → inLoop polling → QSend queue → MiniSender (port 9001) → receiver.

### Purpose
- Diagnostics and board testing without requiring a full receiver
- Direct DMA reads of digitizer FIFO content on-demand
- Scripted spot-checks of raw data integrity

### Architecture

```
Client (Python)
    │
    │  TCP port 9010
    ▼
FifoGrabberTask (new VxWorks C task, fifoGrabber.c)
    ├── CMD_START_BOARD → ClearDigMstrLogicEnable() → ClearDigFIFO() → SetDigMstrLogicEnable()
    └── CMD_GRAB_FIFO  → read FIFOStatusReg depth → sysVmeDmaV2LCopy() → integrity check → send
```

**No interference with MiniSender or inLoop** — separate port, separate task, no queue access.

### Wire Protocol (binary, big-endian / network byte order)

**Request (16 bytes):**
```c
struct FG_Req {
    uint32_t cmd;    // 1=PING, 2=START_BOARD, 3=GRAB_FIFO, 4=STOP_BOARD
    uint32_t board;  // board number 0–6
    uint32_t param1; // GRAB_FIFO: max words; START_BOARD: 0
    uint32_t param2; // GRAB_FIFO: 0=digitizer FIFO, 1–15=trigger FIFO index
};
```

**Response (16 bytes + optional payload):**
```c
struct FG_Resp {
    uint32_t status; // 0=OK, 1=ERR_BOARD, 2=NO_DATA, 3=BAD_INTEGRITY
    uint32_t len;    // bytes of raw payload to follow (0 if no payload)
    uint32_t param1; // actual bytes DMA'd
    uint32_t param2; // FIFO depth at time of read
};
// followed by `len` bytes of raw FIFO data
```

### Key Implementation Details (planned)

| Item | Detail |
|------|--------|
| **File** | `dgsDrivers/dgsDriverApp/src/fifoGrabber.c` (+ `.h`) |
| **Task priority** | 150 — below inLoop (190) so inLoop wins; above most background tasks |
| **Socket options** | TCP_NODELAY, SO_RCVBUF/SNDBUF=65536 (matching `SendReceiveSupport.c`) |
| **Connection model** | One client at a time; stateless — connection closed after each request |
| **Init call** | `FifoGrabberInit(9010)` from VxWorks shell after `iocInit` |
| **Build change** | Add `devDGSDriverSupport_SRCS += fifoGrabber.c` to `dgsDrivers/dgsDriverApp/src/Makefile` |
| **Integrity check** | `buf[0] == 0xAAAAAAAA` (digitizer FIFO start word) |
| **DMA chunking** | Loop at `DMA_CHUNK_SIZE_IN_BYTES=0x10000` (same as `readDigFIFO.c`) |

### Existing Functions Reused (no changes)

| Function | File | Purpose |
|----------|------|---------|
| `ClearDigMstrLogicEnable()` | `inLoopSupport.c:272` | Disable digitizer |
| `ClearDigFIFO()` | `inLoopSupport.c:325` | Clear FIFO (bit 27 pulse) |
| `SetDigMstrLogicEnable()` | `inLoopSupport.c:294` | Arm digitizer + init pipeline |
| `InitializeDigPipeline()` | `inLoopSupport.c:314` | Load delays (called by above) |
| `sysVmeDmaV2LCopy()` | MV5500 BSP | DMA VME→local copy |
| `daqBoards[]` | `inLoopSupport.c` global | Board base addresses + FIFO pointers |
| `FIFOStatusReg[]` | `inLoopSupport.c` global | FIFO depth register pointers |

### Python Client (planned: `grab_fifo.py`)

```python
import socket, struct
FG_PORT = 9010
CMD_PING, CMD_START, CMD_GRAB, CMD_STOP = 1, 2, 3, 4

def grab_fifo(ioc_ip, board, max_words=65536):
    with socket.create_connection((ioc_ip, FG_PORT), timeout=5) as s:
        s.sendall(struct.pack('!4I', CMD_GRAB, board, max_words, 0))
        status, length, actual, depth = struct.unpack('!4I', s.recv(16))
        payload = b''
        while len(payload) < length:
            payload += s.recv(length - len(payload))
    return status, payload  # raw 32-bit words
```

### Status

> ⚠️ **Not implemented** — design plan only. `fifoGrabber.c` and `fifoGrabber.h` have not been created. No changes to the VxWorks build have been made. Plan documented in `vxworks/On-Demand-FIFO-Grabber-Plan.md`.

---

## Cross-References

- `knowledgeBase/ioc.md` — IOC config, boot scripts, firmware versions, MVME5500 setup
- `knowledgeBase/vxworks_migration.md` — Detailed migration notes from Solaris/con6 to Ubuntu 24
- `knowledgeBase/EPICS_asyn.md` — asyn driver internals: port model, worker threads, write flow
- `knowledgeBase/VME_registers.md` — VME register addresses used by the IOC driver
- `knowledgeBase/fpga.md` — FPGA firmware overview; the firmware binaries loaded by VxWorks
