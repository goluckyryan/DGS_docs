# VxWorks Trigger Asyn Drivers

Stability: C2 - Active / semi-stable

**Source:** `DGS_tools_pack/vxworks/dgsDrivers/dgsDriverApp/src/`  
**Files:** `asynTrigCommonDriver.cpp/.h`, `asynTrigMasterDriver.cpp/.h`, `asynTrigRouterDriver.cpp/.h`, `asynMTrigParams.c`, `asynRTrigParams.c`  
**Date documented:** 2026-04-24  
**See also:** [`EPICS_asyn.md`](EPICS_asyn.md) — asyn architecture + digitizer driver; [`VME_registers.md`](VME_registers.md) — full register maps; [`vxworks.md`](vxworks.md) — VxWorks build system; [`deep_fpga_MTRG_MAIN.md`](deep_fpga_MTRG_MAIN.md) — MTRG FPGA internals; [`deep_fpga_RTRG.md`](deep_fpga_RTRG.md) — RTRG FPGA internals; [`vxworks_state_machines.md`](vxworks_state_machines.md) — inLoop/outLoop/MiniSender pipeline + summary-level trigger driver overview that this file extends; [`ioc.md`](ioc.md) — IOC boot scripts, VME01–12 slot map, firmware versions, FTP/NFS setup

---

## Table of Contents

1. [Overview](#overview)
2. [Class Hierarchy](#class-hierarchy)
3. [asynTrigCommonDriver — Base Class](#asyntrigcommondriver--base-class)
4. [asynTrigMasterDriver — MTRG](#asyntrigmasterdriver--mtrg)
5. [asynTrigRouterDriver — RTRG](#asyntrigrouterdriver--rtrg)
6. [Parameter Registration — asynMTrigParams.c / asynRTrigParams.c](#parameter-registration)
7. [Firmware Type Codes](#firmware-type-codes)
8. [Boot Sequence](#boot-sequence)

---

## Overview

The DGS VxWorks IOC controls the Master Trigger (MTRG) and Router Trigger (RTRG) boards through a two-level asyn driver hierarchy. This mirrors the `asynDigitizerDriver` pattern used for digitizer boards, but is specialized for trigger hardware.

All trigger drivers use:
- **`asynUInt32Digital`** as the primary interface (same as the digitizer driver)
- The **`0xaaaa0000` mask sentinel** for multi-bit sub-field access (same encoding as `asynDigitizerDriver` — documented in [`EPICS_asyn.md`](EPICS_asyn.md))
- A **VME mutex** (`vme_driver_mutex`) to serialize concurrent VME bus transactions
- A **1-second polling background thread** that reads all registered VME parameters and calls `callParamCallbacks()` to push updates to EPICS records

---

## Class Hierarchy

```
asynPortDriver  (EPICS asyn base)
    └── asynTrigCommonDriver   (common VME R/W + polling thread)
            ├── asynTrigMasterDriver   (MTRG: 369 params from asynMTrigParams.c)
            └── asynTrigRouterDriver   (RTRG: 188 params from asynRTrigParams.c)
```

---

## asynTrigCommonDriver — Base Class

**File:** `asynTrigCommonDriver.cpp` (412 lines), `asynTrigCommonDriver.h`

### Constructor

```cpp
asynTrigCommonDriver(const char *portName, int card_number)
    : asynPortDriver(portName, 1, 1024, asynUInt32DigitalMask | ..., ...)
```

- Max parameters: **1024** (expanded from original 256 by MPC 2015-10-29) ✅ verified 2026-04-24 — asynTrigCommonDriver.cpp:L132 (`1024, // changed from 256 by mpc 10/29/15`), L147 (`address_list = new int_int[1024]; // changed from 256 by mpc 10/30/15`)
- Allocates `address_list` array of 1024 `{param_num, address}` pairs
- Registers one built-in param: `run_counter` (incremented every second by the polling thread)
- Spawns `asynTrigCommonDriver_Task` at `epicsThreadPriorityMedium`

### Background Polling Thread (`simTask()`)

Runs forever, every **1.0 second** (reduced from 2.0s by JTA 2022-09-16): ✅ verified 2026-04-24 — asynTrigCommonDriver.cpp:L96 (`epicsThreadSleep(1.0); //changed from 2.0 to 1.0 JTA 20220916`)

1. Increments `run_counter` (gives EPICS clients a heartbeat)
2. Locks `vme_driver_mutex`
3. Iterates through all `param_address_cnt` entries in `address_list[]`
4. Reads each VME address with `viIn32()`, stores result with `setUIntDigitalParam()`
5. Unlocks mutex
6. Calls `callParamCallbacks()` → pushes updated values to all EPICS records

**Key implication:** trigger register values seen by EPICS clients are at most ~1 second stale. On `caput`, the write goes through immediately (`writeUInt32Digital`), not via the polling thread.

### VME Access Methods

| Method | Direction | Implementation |
|--------|-----------|----------------|
| `viOut32(slot, adr_space, reg_adr, data)` | Write | `*addr = data` where `addr = daqBoards[slot].base32 + reg_adr/4` |
| `viIn32(slot, adr_space, reg_adr, *data)` | Read | `*data = *addr` (same address formula) |

- `adr_space` is always 0 (`VI_A32_SPACE`) — A32 VME space
- `reg_adr` is a **byte offset** into the board's VME address space
- Both calls are guarded by `vme_driver_mutex` in the polling thread; writes also lock the mutex in `writeUInt32Digital`

### `0xaaaa0000` Sub-Field Mask Encoding

**Shared** with `asynDigitizerDriver` (see [`EPICS_asyn.md`](EPICS_asyn.md) §Custom Sub-Field Mask Encoding):

```
mask = 0xaaaa_NNSS
       ^^^^ sentinel
            ^^ NN = number of bits (0x01–0xFF)
              ^^ SS = shift amount (bit position of LSB)
```

- If `(mask & 0xffff0000) == 0xaaaa0000`: sub-field mode → extract/insert `NN`-bit field at position `SS`
- Otherwise: raw mask → direct bitwise AND

Applied in both `readUInt32Digital()` and `writeUInt32Digital()`. The same encoding is used in `MTrigUser.template` / `RTrigUser.template` DB records.

### Parameter-to-Address Mapping

`setAddress(param, address)` — called once per parameter during subclass construction:

```cpp
address_list[param_address_cnt].param_num = param;
address_list[param_address_cnt].address   = address;
param_address_cnt++;
```

`findAddress(param)` — linear scan through `address_list[]`, returns VME byte offset or `-1` if not mapped.

Parameters with `address = -1` (not found) are **not written to VME** on `writeUInt32Digital` — they are stored only in the asyn parameter library (used as soft controls or staging registers).

### Debug Trace

`int asyntrig_trace` (global, default -1): set to a param index to enable per-call `printf` for read/write of that parameter.

---

## asynTrigMasterDriver — MTRG

**File:** `asynTrigMasterDriver.cpp` (274 lines), `asynTrigMasterDriver.h`

### Card Initialization — `devAsynTrigMasterCardInit(cardno, slot)`

1. Calls `initVmeDrvMutex()` (idempotent; initializes global VME mutex on first call)
2. Calls `devGVMECardInit(cardno, slot)` — maps VME A32 address space into `daqBoards[cardno].base32`
3. Reads firmware type register at **byte offset 0x15C** (`Code_Revision` register) via `devReadProbe()` ✅ verified 2026-04-24 — asynTrigMasterDriver.cpp:L102-104
4. Extracts `ftype = (boardid & 0xf00) >> 8` ✅ verified 2026-04-24 — asynTrigMasterDriver.cpp:L116-117
5. Sets `daqBoards[cardno].board_type = BrdType_DGS_MTRIG` ✅ verified 2026-04-24 — asynTrigMasterDriver.cpp:L154
6. Accepts types **4** (DGS Master Trigger) and **6** (DGS Router — accepted here too, sets `mainOK=1` for both); rejects all others with `mainOK=0` + `return ERROR` (except type 2 GRETINA which has a misplaced `return ERROR` — dead code bug) ✅ verified 2026-04-24 — asynTrigMasterDriver.cpp:L157-204
7. Sets `daqBoards[cardno].FIFO = base32 + 0x0178/4` — **Monitor FIFO 7** pointer (legacy address; JTA 2025-05-28 moved FIFO 7 to 0x5000 in the FPGA — actual DAQ reads now use `MTRG_FIFO = 0x5000/4` in `inLoopSupport.c:L77`; the `daqBoards[].FIFO` pointer at 0x0178 is stale for MTRG) ✅ verified 2026-04-24 — asynTrigMasterDriver.cpp:L213-216; readTrigFIFO.c:L86-88; inLoopSupport.c:L77
8. Sets `daqBoards[cardno].mainOK = 1` (gate for record init_record functions)

### Driver Instantiation — `asynTrigMasterConfig1(portName, card_number, slot)`

Called from IOC startup script (e.g., `vme10.cmd`):
```
asynTrigMasterConfig1("VME10_MTRG", 12, 7)
```

1. Calls `devAsynTrigMasterCardInit(card_number, slot)`
2. `new asynTrigMasterDriver(portName, card_number)` → calls base constructor + `#include "asynMTrigParams.c"`

### Constructor — `asynTrigMasterDriver(portName, card_number)`

Calls `asynTrigCommonDriver(portName, card_number)` then:

```cpp
#include "asynMTrigParams.c"
```

This is a **textual include** — the file is `#include`-d directly inside the constructor body, so every `createParam()` / `setAddress()` call runs as constructor code. After this include, the driver has **369 MTRG parameters** registered.

---

## asynTrigRouterDriver — RTRG

**File:** `asynTrigRouterDriver.cpp` (274 lines), `asynTrigRouterDriver.h`

### Card Initialization — `devAsynTrigRouterCardInit(cardno, slot)`

Identical pattern to the master driver:
1. `initVmeDrvMutex()` → `devGVMECardInit()` → read reg 0x15C
2. Sets `daqBoards[cardno].board_type = BrdType_DGS_RTRIG`
3. Accepts types **6** (DGS Router) **and 4** (DGS Master — sets `mainOK=1` + `router=0`); rejects all others with `mainOK=0` + `return ERROR` (same type-2 dead-code bug as master driver) ✅ verified 2026-04-24 — asynTrigRouterDriver.cpp:L145-192

   **Correction vs. prior documentation:** The Router driver does NOT reject type 4 — it accepts it with mainOK=1. Both Master and Router drivers have identical switch logic for ftype 4 and 6.
4. Sets `daqBoards[cardno].FIFO = base32 + 0x0178/4` (same FIFO 7 pointer, same caveat)
5. Sets `mainOK = 1`

### Driver Instantiation — `asynTrigRouterConfig1(portName, card_number, slot)`

```
asynTrigRouterConfig1("VME01_RTR1", 1, 5)
```

1. `devAsynTrigRouterCardInit(card_number, slot)`
2. `new asynTrigRouterDriver(portName, card_number)`

### Constructor — `asynTrigRouterDriver(portName, card_number)`

Same pattern: calls base, then `#include "asynRTrigParams.c"` → **188 RTRG parameters** registered.

---

## Parameter Registration

Both `asynMTrigParams.c` and `asynRTrigParams.c` use the same 2-line pattern per parameter:

```c
createParam("reg_LOCK_BUS", asynParamUInt32Digital, &reg_LOCK_BUS);
setAddress(reg_LOCK_BUS, 0x0000);
```

- `createParam()` registers the name + type in the asyn parameter library; stores the integer handle into the member variable
- `setAddress()` stores the VME byte-offset for this parameter in `address_list[]`

Parameters that have no VME address (soft controls or staging regs) appear in other include mechanisms or are handled differently.

### MTRG Parameter Summary (`asynMTrigParams.c`)

- **Lines:** 1,111 (1,111 = 369 pairs + comments + spacing) ✅ verified 2026-04-26 — `wc -l asynMTrigParams.c = 1111`
- **Parameters:** 369 (`createParam()` calls = 369) ✅ verified 2026-04-26 — `grep -c createParam asynMTrigParams.c = 369`
- **`setAddress()` calls:** 369 (every param has a VME address)
- Covers all MTRG VME registers (mirrors `MTrigRegisters.template` and the MTRG VHDL register map in [`VME_registers.md`](VME_registers.md))

Key parameter groups (sampled):

| Parameter Name | Notes |
|---------------|-------|
| `reg_LOCK_BUS`, `reg_DEN_BUS`, `reg_REN_BUS`, `reg_SYNC_BUS` | Bus control/status |
| `reg_TIMESTAMP_A/B/C` | 48-bit timestamp read (3 × 16-bit) |
| `reg_MSTR_MACH_STATE` | Master state machine state |
| `reg_CODE_DATE`, `reg_CODE_REVISION` | Firmware date and version |
| `reg_MON7_FIFO_DEPTH`, `reg_MON7_FIFO_STATE` | Monitor FIFO 7 status |
| `reg_MON_FIFO_STATE`, `reg_CHAN_FIFO_STATE` | All FIFO status |
| `reg_SYSTEM_THROTTLE_MAP` | Trigger throttle bitmap |
| `reg_FRAME_12/14/16/17_CMD_CNT` | TTCL command frame counters |
| `reg_STARTING_TIMESTAMP_HI/MID/LOW` | Run-start timestamp (3-word) |
| `reg_FRAME_17_DATA_1..5` | Frame 17 (AUX detector) data words |
| `reg_ENCODER_SOURCE_SELECT`, `reg_ENCODER_TEST` | Target wheel encoder |
| `reg_MYRIAD_TRIG_DELAY`, `reg_MYRIAD_OVERLAP_CTL` | MYRIAD control |
| `reg_TDC_TRIG_SEL`, `reg_TRIG_ALGO_MUX_SEL` | TDC / trigger algo mux |
| `reg_MON7_FILL_CTL` | Monitor FIFO 7 fill control |

### RTRG Parameter Summary (`asynRTrigParams.c`)

- **Lines:** 568 ✅ verified 2026-04-26 — `wc -l asynRTrigParams.c = 568`
- **Parameters:** 188 (`createParam()` calls) ✅ verified 2026-04-26 — `grep -c createParam asynRTrigParams.c = 188`
- **`setAddress()` calls:** 188
- Covers all RTRG VME registers (mirrors `RTrigRegisters.template`)

---

## Firmware Type Codes

Read from register offset **0x15C** (`Code_Revision`) bits `[11:8]`:

✅ verified 2026-04-26 — `asynTrigMasterDriver.cpp:L125-140` (comment block: type list last updated September 2012)

| ftype | Board Type | Accepted by |
|-------|-----------|-------------|
| 0 | Proto | — |
| 1 | GRETINA Router | — |
| 2 | GRETINA Master Trigger | — |
| 3 | GRETINA Data Generator | — |
| 4 | **DGS Master Trigger** | `asynTrigMasterCardInit` ✅ |
| 5 | DSSD Master Trigger | — |
| **6** | **DGS Router Trigger** | Both Master and Router init ✅ |
| 7 | DSSD Router | — |
| 8 | DGS Data Generator | — |
| 9 | DSSD Data Generator | — |
| A | Digitizer Tester | — |
| B | MYRIAD Trigger expansion | — |
| C | DGS Digitizer | — |
| D | DSSD Digitizer | — |
| E | (unused) | — |
| F | VME FPGA | — |

Bits `[7:4]` = major revision ordinal; bits `[3:0]` = minor revision ordinal. ✅ verified 2026-04-26 — `asynTrigMasterDriver.cpp:L141-142`

**Note:** The Master driver `devAsynTrigMasterCardInit` accepts both type 4 (DGS Master) and type 6 (DGS Router), but sets `board_type = BrdType_DGS_MTRIG` regardless. The Router driver `devAsynTrigRouterCardInit` also accepts **both type 4 and type 6** (type 4 sets `router=0`, `mainOK=1`; type 6 sets `router=1`, `mainOK=1`). ✅ corrected 2026-04-26 — `asynTrigRouterDriver.cpp:L171-177` (`case 4: router=0, mainOK=1`; `case 6: router=1, mainOK=1`)

---

## Boot Sequence

IOC startup script (`iocBoot/vmeXX.cmd`) sequence for a trigger board:

```
# For MTRG (one per system, e.g., VME10 slot 7):
asynTrigMasterConfig1("VME10_MTRG", 12, 7)
dbLoadRecords("MTrigRegisters.template", "CRATE=10,BOARD=MTRG,PORT=VME10_MTRG")
dbLoadRecords("MTrigUser.template",      "CRATE=10,BOARD=MTRG,PORT=VME10_MTRG")

# For RTRG (multiple boards per crate, e.g., VME01 slot 5):
asynTrigRouterConfig1("VME01_RTR1", 1, 5)
dbLoadRecords("RTrigRegisters.template", "CRATE=01,BOARD=RTR1,PORT=VME01_RTR1")
dbLoadRecords("RTrigUser.template",      "CRATE=01,BOARD=RTR1,PORT=VME01_RTR1")
```

**PV naming:** `GS:{CRATE}:{BOARD}:reg_LOCK_BUS` etc. (macro expansions from template `$(CRATE)` / `$(BOARD)`).

---

## Cross-References

| File | Relationship |
|------|--------------|
| [`vxworks.md`](vxworks.md) | VxWorks IOC overview, asyn framework context |
| [`vxworks_state_machines.md`](vxworks_state_machines.md) | inLoop/outLoop state machines (work above the trigger drivers) |
| [`vxworks_vme_devlayer.md`](vxworks_vme_devlayer.md) | VME device layer (devGVME.c) these drivers rely on |
| [`EPICS_asyn.md`](EPICS_asyn.md) | asyn framework explanation |
| [`EPICS_DB_templates.md`](EPICS_DB_templates.md) | MTrigUser/MDig/SDig template overview (split — see EPICS_RTrig_templates.md for RTrig) |
| [`EPICS_RTrig_templates.md`](EPICS_RTrig_templates.md) | Complete RTrigRegisters + RTrigUser PV inventory (deep-dive, split from EPICS_DB_templates.md) |
| [`260E_trigger_scheme.md`](260E_trigger_scheme.md) | End-to-end RTRG/MTRG firmware trigger algorithm that the driver registers map to |
| [`deep_fpga_MTRG_MAIN.md`](deep_fpga_MTRG_MAIN.md) | MTRG FPGA internals (registers the MTRG driver reads/writes) |
| [`deep_fpga_RTRG.md`](deep_fpga_RTRG.md) | RTRG FPGA internals (registers the RTRG driver reads/writes) |

---

*Created: 2026-04-24 | Last reviewed: 2026-04-25*
