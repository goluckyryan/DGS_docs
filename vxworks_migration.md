# Migration Notes: con6 (Solaris SPARC) → Ubuntu 24 (linux-x86_64)

## Source System

- **Host**: con6, hostname `192.168.203.136`
- **OS**: Sun Solaris 10 (SPARC) ✅ verified 2026-04-11 — `DecodingTheMakefile.txt:L4` (`EPICS_HOST_ARCH=solaris-sparc-gnu`)
- **User**: dgs
- **Original build root**: `/global/devel/dgsDrivers/` ✅ verified 2026-04-11 — `DecodingTheMakefile.txt:L12` (`TOP = . ##for us this is /global/devel/dgsDrivers`)
- **Build environment on con6:** `EPICS_BASE=/global/devel/epics/base`, `WIND_BASE=/global/devel/vxWorks/Tornado2.2`, `EPICS_HOST_ARCH=solaris-sparc-gnu`, `WIND_HOST_TYPE=sun4-solaris2` ✅ verified 2026-04-11 — `DecodingTheMakefile.txt:L4-8`

## What Was Copied from con6

All data copied read-only via `scp`/`rsync` over SSH.
SSH required legacy options: `-o KexAlgorithms=+diffie-hellman-group1-sha1 -o HostKeyAlgorithms=+ssh-rsa -o Ciphers=+aes128-cbc,3des-cbc`

| Source on con6 | Destination on Ubuntu | Notes |
|---|---|---|
| `/global/devel/dgsDrivers/` | `dgsDrivers/` | Driver source + original Solaris build artifacts |
| `/global/devel/epics/base/` | `epics/base-3.14.12.1/` | EPICS 3.14.12.1 |
| `/global/devel/asyn/` | `synApps/asyn4-17/` | asyn 4-17 |
| `/global/devel/sncseq/` | `sncseq/sncseq-2.0.12/` | State Notation Compiler/Sequencer 2.0.12 |
| `/global/devel/vxWorks/Tornado2.2/` | `vxWorks/Tornado2.2/` | VxWorks headers only (target/h) |
| `/global/devel/vxWorks/Tornado2.2/target/config/mv5500/universe.h` | `vxWorks/Tornado2.2/target/config/mv5500/universe.h` | MVME5500 BSP Universe VME bridge header (fetched separately) |
| `/global/devel/vxWorks/Tornado2.2/host/src/hutils/munch.tcl` | `munch.tcl` | VxWorks C++ ctor/dtor munch script ✅ verified 2026-04-12 — `vxWorks/Tornado2.2/target/h/tool/gnu/defs.gnu:L126` (`MUNCH = wtxtcl $(WIND_BASE)/host/src/hutils/munch.tcl`) |

**Cross-compiler**: Downloaded separately from https://coda.jlab.org/drupal/content/ppc-cross-compilers
- File: `x86-linux.tar` (GCC 2.96, powerpc-wrs-vxworks, VxWorks 5.5 compatible)
- Extracted to: `x86-linux/`

**Note**: Solaris `tar` followed NFS symlinks, so all files are real copies (no broken symlinks).
Stale Solaris build artifacts (`O.solaris-sparc-gnu/`, `O.linux-x86/`) were deleted before rebuilding.

---

## Path Changes

All hardcoded `/global/devel/...` paths from con6 were updated to `/home/ryan/DGS_Tools_Pack/VxWorks_Compiler/...`.

### configure/RELEASE files

| File | Variable | Old Value | New Value |
|---|---|---|---|
| `dgsDrivers/configure/RELEASE` | `SNCSEQ` | `/global/devel/sncseq/sncseq-2.0.12` | `/home/ryan/DGS_Tools_Pack/VxWorks_Compiler/sncseq/sncseq-2.0.12` |
| `dgsDrivers/configure/RELEASE` | `ASYN` | `/global/devel/asyn` | `/home/ryan/DGS_Tools_Pack/VxWorks_Compiler/synApps/asyn4-17` |
| `dgsDrivers/configure/RELEASE` | `EPICS_BASE` | `/global/devel/epics/base` | `/home/ryan/DGS_Tools_Pack/VxWorks_Compiler/epics/base-3.14.12.1` |
| `synApps/asyn4-17/configure/RELEASE` | `SNCSEQ` | `/global/devel/sncseq/sncseq-2.0.12` | `/home/ryan/DGS_Tools_Pack/VxWorks_Compiler/sncseq/sncseq-2.0.12` |
| `synApps/asyn4-17/configure/RELEASE` | `EPICS_BASE` | `/global/devel/epics/base` | `/home/ryan/DGS_Tools_Pack/VxWorks_Compiler/epics/base-3.14.12.1` |
| `sncseq/sncseq-2.0.12/configure/RELEASE` | `EPICS_BASE` | `/global/devel/epics/base` | `/home/ryan/DGS_Tools_Pack/VxWorks_Compiler/epics/base-3.14.12.1` |

### EPICS base configure files

| File | Variable | Old Value | New Value |
|---|---|---|---|
| `epics/base-3.14.12.1/Makefile` | `TOP` | `/global/devel/epics/base` | `/home/ryan/DGS_Tools_Pack/VxWorks_Compiler/epics/base-3.14.12.1` |
| `epics/base-3.14.12.1/configure/CONFIG_SITE` | `CROSS_COMPILER_HOST_ARCHS` | `solaris-sparc-gnu` | `linux-x86_64` | ✅ verified 2026-04-11 — `vxworks/epics/base-3.14.12.1/configure/CONFIG_SITE:L115`
| `epics/base-3.14.12.1/configure/os/CONFIG_SITE.Common.vxWorksCommon` | `WIND_BASE` | `/global/devel/vxWorks/Tornado2.2` | `/home/ryan/DGS_Tools_Pack/VxWorks_Compiler/vxWorks/Tornado2.2` |

### Source file path fixes

| File | Change |
|---|---|
| `dgsDrivers/dgsDriverApp/src/QueueManagement.h:15` | `#include "/global/devel/..."` → `#include "/home/ryan/DGS_Tools_Pack/VxWorks_Compiler/..."` |

---

## Source Code Fixes (Compatibility with GCC 12/13 and Ubuntu 24)

### EPICS base-3.14.12.1

| File | Fix | Reason |
|---|---|---|
| `configure/os/CONFIG_SITE.linux-x86_64.UnixCommon` | Removed stray backtick on line 2 | Caused "missing separator" Makefile parse error |
| `configure/os/CONFIG_SITE.linux-x86_64.Common` | Added `CC=gcc-12`, `CCC=g++-12`, `CXX=g++-12`, `USR_CXXFLAGS += -fpermissive` | GCC 13 too strict; GCC 12 with `-fpermissive` handles EPICS 3.14 C++98 code | ✅ verified 2026-04-11 — `vxworks/epics/base-3.14.12.1/configure/os/CONFIG_SITE.linux-x86_64.Common:L15-20`
| `src/libCom/cxxTemplates/epicsSingleton.h` | Added `#include <cstddef>` | GCC 13: `size_t` not implicitly declared |
| `src/libCom/cxxTemplates/tsDLList.h` | Added `#if __cplusplus >= 201103L` guard: use `= delete` in C++11 mode, private declaration in C++98 mode | GCC 12 C++11 mode treats private copy constructor differently from `= delete` |
| `src/cas/generic/ioBlocked.h` | Changed `ioBlockedList` from `private tsDLList<ioBlocked>` inheritance to composition (`tsDLList<ioBlocked> m_list` member) | GCC 12 C++11: injected class name from private base leaked into derived class name lookup, causing access errors in `caServerI` and `casPVI` |
| `src/cas/generic/st/ioBlocked.cc` | Updated list operations to use `m_list.get()`, `m_list.add()`, `m_list.remove()` | Required by `ioBlocked.h` composition change |
| `src/cas/build/Makefile` | `LIBRARY = cas` → `LIBRARY_IOC = cas` | Prevent building linux host CAS library (not needed) |
| `src/gdd/Makefile` | `LIBRARY = gdd` → `LIBRARY_IOC = gdd` | Prevent building linux host GDD library (not needed) |
| `src/Makefile` | Commented out `libCom/test` and `db/test` DIRS | GCC test code had array-too-large errors incompatible with host compiler |

### asyn4-17

| File | Fix | Reason |
|---|---|---|
| `asyn/asynDriver/asynDriver.h:215` | `__VAR_ARGS__` → `__VA_ARGS__` | Typo in variadic macro; fatal error with GCC 13 in C99/C17 mode |
| `asyn/asynDriver/asynDriver.h:229` | `__VAR_ARGS__` → `__VA_ARGS__` | Same typo in `asynPrintIO` macro |
| `asyn/Makefile` | Disabled vxi11 sources and test/example app DIRS | vxi11 requires `rpc/rpc.h` (ONC RPC) not present in Ubuntu 24 glibc; test apps not needed for MVME5500 target |

### sncseq-2.0.12

| File | Fix | Reason |
|---|---|---|
| `Makefile` | Commented out `test` from DIRS | `snc` example programs not needed for MVME5500 target |

### dgsDrivers

| File | Fix | Reason |
|---|---|---|
| `dgsDriverApp/Makefile` | Commented out `Db` from DIRS | `Db/` directory had no Makefile; `.dbd` already handled by `src/` |
| `dgsDriverApp/src/Makefile` | Commented out `DBEXPAND = msi` | `msi` not part of EPICS 3.14.12.1 base; default `dbExpand` is equivalent |

### Symlink created

```
vxWorks/Tornado2.2/host/x86-linux → /home/ryan/DGS_Tools_Pack/VxWorks_Compiler/x86-linux
```
Required so EPICS `GNU_DIR` path (`$(WIND_BASE)/host/$(WIND_HOST_TYPE)`) resolves to the jlab cross-compiler.

---

## Ubuntu 24 Packages Required

```bash
sudo apt-get install flex libfl-dev
```

- `flex` + `libfl-dev`: required to build `snc` (State Notation Compiler) from sncseq, which compiles `.st` sequencer files used in dgsDrivers
- `gcc-12` / `g++-12`: required for host build (GCC 13 is too strict for EPICS 3.14 C++98 code)

---

## Build Targets Produced

| File | Description |
|---|---|
| `epics/base-3.14.12.1/lib/vxWorks-ppc604_long/*.a` | 14 EPICS base static libraries for VxWorks PPC604 |
| `synApps/asyn4-17/lib/vxWorks-ppc604_long/libasyn.a` | asyn driver support library |
| `dgsDrivers/lib/vxWorks-ppc604_long/libdevDGSDriverSupport.a` | DGS driver support library |
| `dgsDrivers/dgsDriverApp/src/O.vxWorks-ppc604_long/dgsDriver.munch` | Final VxWorks loadable object (ELF 32-bit MSB PowerPC relocatable) |

---
*Source: Migration session notes + `DGS_tools_pack/vxworks/` build environment. Created: 2026-04-05.*
