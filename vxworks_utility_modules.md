# VxWorks Utility Modules — profile, asynDebugDriver, FlashMaintenance, equalSub, restoreSub

Stability: C2 - Active / semi-stable

_Source files: `vxworks/dgsDrivers/dgsDriverApp/src/`_
_Author: Michael Oberling (profile.c/h, asynDebugDriver.cpp/.h, FlashMaintenance.c)_
_Documented: 2026-04-25_

**See also:** [`vxworks_vme_devlayer.md`](vxworks_vme_devlayer.md) — VME hardware access layer (`devGVME.c`, `daqBoards[]`), [`vxworks_state_machines.md`](vxworks_state_machines.md) — inLoop/outLoop pipeline, [`EPICS_asyn.md`](EPICS_asyn.md) — asyn driver architecture

---

## Table of Contents

1. [profile.c / profile.h — CPU Profiling Framework](#profilec--profileh--cpu-profiling-framework)
2. [asynDebugDriver.cpp — Generic VME Peek/Poke Debug Driver](#asynDebugDrivercpp--generic-vme-peekpoke-debug-driver)
3. [FlashMaintenance.c — Flash Register Constants](#flashmaintenancec--flash-register-constants)
4. [equalSub.c — EPICS Equality Sub-routine](#equalSubc--epics-equality-sub-routine)
5. [restoreSub.c — EPICS PV Restore Sub-routine](#restoreSubc--epics-pv-restore-sub-routine)

---

## profile.c / profile.h — CPU Profiling Framework

**File:** `profile.c` (407 L) ✅ verified 2026-04-25 — `wc -l profile.c`, `profile.h` (105 L)
**Purpose:** A lightweight, multi-counter CPU timing profiler for VxWorks. Allows any block of code to be bracketed with `start_profile_counter()` / `stop_profile_counter()` calls to measure its CPU time and execution rate.

### Design

- **`NUM_PROFILE_COUNTERS`** named counters, each identified by a `unsigned char` index.
- **Clock source:** `vxTimeBaseGet()` — PowerPC timebase register, returns a 64-bit monotonic tick count. The frequency must be supplied at init via `init_profile_counters(double clock_frequency)`. ✅ verified 2026-04-25 — `profile.c:L70,L79`
- **Prescaling:** Each counter can be initialized with a prescale factor N (`init_profile_counter(index, name, N)`). The counter only runs 1 out of every N starts, reducing profiling overhead for tight loops at the cost of sample density.
- **Calibration overhead subtraction:** Each `start_profile_counter()` / `stop_profile_counter()` call also wraps a calibration counter (`start_profile_cal_counter` / `stop_profile_cal_counter`). The calibration delta is subtracted from the measured delta to compensate for the overhead of the profiling function calls themselves.
- **Thread safety:** All state updates use `taskLock()` / `taskUnlock()` to prevent race conditions between VxWorks tasks. ✅ verified 2026-04-25 — `profile.c:L82,L106,L113,L121,L129,L142,L151,L188`
- **NO_PROFILING:** If `NO_PROFILING` is `#define`d, `start_profile_counter` / `stop_profile_counter` compile to no-ops (zero overhead in production builds). ✅ verified 2026-04-25 — `profile.c:L31,L128,L150` (`#ifndef NO_PROFILING` guards)

### PAUSABLE_PROFILER mode

When `PAUSABLE_PROFILER` is `#define`d (enabled by default in `profile.h`):

- **`pause_profile_counter()`** / **`resume_profile_counter()`**: allows profiling a non-contiguous block that includes sleep periods. Time between pause and resume is excluded.
- **Task-switch hook** (`profile_counter_task_switch_hook(WIND_TCB* pOldTcb, WIND_TCB* pNewTcb)`): registered as a VxWorks task-switch hook. When the profiled task is preempted (context switch out), the running counter is automatically suspended; when it's resumed, the counter restarts. This gives true on-CPU time, excluding time spent preempted by other tasks. ✅ verified 2026-04-25 — `profile.c:L277-302`
- **Dual time tracking:** The PAUSABLE mode tracks both:
  - `total_time[]` — CPU-only time (excluding preemption)
  - `total_time_real_time[]` — wall-clock time (including preemption, for comparison) ✅ verified 2026-04-25 — `profile.c:L58`
- **`num_task_switches[]`** counts how often each profiled block was preempted. ✅ verified 2026-04-25 — `profile.c:L44`

### Key Functions

| Function | Description |
|----------|-------------|
| `init_profile_counters(freq)` | Initialize all counters; zero stats; set clock frequency |
| `run_profile_counters()` | Set all counters to STOPPED state; record start timestamp |
| `init_profile_counter(idx, name, prescale)` | Name a counter and set its prescale |
| `start_profile_counter(idx)` | Begin timing for counter `idx` |
| `stop_profile_counter(idx)` | End timing; accumulate delta; apply prescale scaling |
| `pause_profile_counter(idx)` | Pause (PAUSABLE mode only) — excludes sleep time |
| `resume_profile_counter(idx)` | Resume after pause |
| `print_profile_summary()` | Printf table: %CPU, %real-time, ticks/exec, execs/sec, task-switches/exec |
| `get_profile_counter_exec_second(idx)` | Returns double: executions per second |
| `get_profile_counter_percent_time(idx, scale)` | Returns integer: percentage × scale |

### Usage Pattern (from profile.h example)
```c
start_profile_counter(0);           // profile whole loop
  start_profile_counter(1);
  do_something();
  stop_profile_counter(1);

  start_profile_counter(2);
  do_something_different();
  pause_profile_counter(2);         // exclude sleep
  sleep_for_10ms();
  resume_profile_counter(2);
  do_something_interesting();
  stop_profile_counter(2);
stop_profile_counter(0);
```

### State Machine (per counter)
```
COUNTER_DISABLED  →  COUNTER_STOPPED  (run_profile_counters)
COUNTER_STOPPED   →  COUNTER_RUNNING  (start_profile_counter)
COUNTER_RUNNING   →  COUNTER_STOPPED  (stop_profile_counter)
COUNTER_RUNNING   →  COUNTER_PAUSED   (pause_profile_counter)    [PAUSABLE only]
COUNTER_RUNNING   →  COUNTER_SUSPENDED (task switch hook)        [PAUSABLE only]
COUNTER_PAUSED    →  COUNTER_RUNNING  (resume_profile_counter)   [PAUSABLE only]
COUNTER_SUSPENDED →  COUNTER_RUNNING  (task switch hook, on resume) [PAUSABLE only]
```
✅ verified 2026-04-25 — `profile.c:L14-20,L116,L139,L158,L207,L229,L285-302` (enum + state transitions)

---

## asynDebugDriver.cpp — Generic VME Peek/Poke Debug Driver

**File:** `asynDebugDriver.cpp` (551 L) ✅ verified 2026-04-25 — `wc -l asynDebugDriver.cpp`, `asynDebugDriver.h`
**Purpose:** A minimal EPICS asyn port driver that exposes raw VME register read/write access for any card in the crate, with no board-specific logic. Intended for diagnostic use — allows developers to peek/poke arbitrary registers from EPICS PVs or the IOC shell without a full driver.

### Configuration

In the IOC boot script:
```
dbLoadRecords("db/asynDebug.template","P=VME04:,R=DBG:,PORT=DBG,ADDR=0,TIMEOUT=1")
asynDebugConfig("DBG", 0)      # creates driver for card 0
asynDebugCard(cardno, slot)    # maps VME slot → daqBoards[] entry
```

### Constructor & asyn Registration

`asynDebugDriver` extends `asynPortDriver` with:
- `maxAddr = 1` ✅ verified 2026-04-25 — `asynDebugDriver.cpp:L118`, `numParams = 2048` ✅ verified 2026-04-25 — `asynDebugDriver.cpp:L119`
- Interfaces: `asynInt32Mask | asynFloat64Mask | asynOctetMask | asynGenericPointerMask | asynDrvUserMask`
- Autoconnect on, priority 100

**Parameters registered:**
| PV suffix | asyn param | Type | Description |
|-----------|------------|------|-------------|
| `dbg_address` | `dbg_address` | Int32 | 16-bit register offset |
| `dbg_long_address` | `dbg_long_address` | Int32 | Long (24-bit?) address |
| `dbg_value` | `dbg_value` | Int32 | Value to write |
| `dbg_value_read` | `dbg_value_read` | Int32 | Last value read back |
| `dbg_write_addr` | `dbg_write_addr` | Int32 | Trigger: write reg at `dbg_address` |
| `dbg_read_addr` | `dbg_read_addr` | Int32 | Trigger: read reg at `dbg_address` |
| `dbg_write_long_addr` | `dbg_write_long_addr` | Int32 | Trigger: write at long address |
| `dbg_read_long_addr` | `dbg_read_long_addr` | Int32 | Trigger: read at long address |
| `dbg_card_number` | `dbg_card_number` | Int32 | Which crate card to access |

### writeInt32() Flow

When the `dbg_write_addr` PV is written:
1. Reads `dbg_card_number`, `dbg_address`, `dbg_value` from param library.
2. Locks `vme_driver_mutex`.
3. Calls `viOut32(cardnum, 0, offset, data)` → direct memory write via `daqBoards[slot].base32`.
4. Unlocks mutex.

When `dbg_read_addr` is written:
1. Reads `dbg_card_number`, `dbg_address`.
2. Locks mutex; calls `viIn32()` → reads from `daqBoards[slot].base32 + reg_adr/4`.
3. Stores result in `dbg_value_read` PV.

### Global Debug Variables

These are globally visible integers, adjustable at the IOC shell, that control verbosity across multiple drivers: ✅ verified 2026-04-25 — `asynDebugDriver.cpp:L69-79`

| Variable | Controls |
|----------|----------|
| `asyn_debug_level` | asynDebugDriver VME peek/poke verbosity |
| `inloop_debug_level` | inLoop task debug output |
| `outloop_debug_level` | outLoop task debug output |
| `sender_debug_level` | MiniSender TCP send debug |
| `printevery` | Print-every-N throttle (default 1024) |
| `prog_flip_endian` | FPGA programming byte-swap flag (default 1 = swap) |
| `asyn_sleepusec` | Optional microsecond sleep in loops |

### FPGA Programming Buffer

The driver contains an internal buffer `fpga_prog_data[]` / `fpga_prog_data2[]` and `fpga_prog_size` for FPGA bitstream staging. The `read()` / `write()` methods transfer data to/from this buffer with optional endian-flip (`flipEndian()`). This is the mechanism used by the flash programming path to stage bitstream data before calling `FlashMaintenance` routines.

### asynGenReport() — IOC Shell Diagnostic

`asynGenReport(cmd)` accepts space-separated tokens:
- `"cards"` — dumps all `daqBoards[]` entries (base32, rev, subrev, mainOK, FIFO address, EnabledForReadout, board_type name)
- `"regs"` — dumps `daqDevPvt_list[]` entries (mask, signal, card, chan, reg addr)
- `"dbg0"` / `"dbg1"` / `"dbg2"` — sets `asyn_debug_level` 0/1/2

### CommandHandlerTask

A background epicsThread (`asynDebugDriver_Task`) at medium priority runs a `while(1)` loop sleeping 100 ms (`epicsThreadSleep(.1)`) between iterations. Currently the loop body is empty — placeholder for future PV-change-triggered actions. ✅ verified 2026-04-25 — `asynDebugDriver.cpp:L185,L185,L289-294` (epicsThreadPriorityMedium, sleep .1, empty body)

---

## FlashMaintenance.c — Flash Register Constants

**File:** `FlashMaintenance.c` (32 L) ✅ verified 2026-04-25 — `wc -l FlashMaintenance.c`
**Purpose:** Defines VME register addresses used for FPGA flash programming and board configuration control. These constants are used by `devGVME.c` flash routines and the `asynDebugDriver` firmware download path.

### Register Constants (VME A32 offsets)

| Constant | Address | Description |
|----------|---------|-------------|
| `FLASH_BLOCK_SIZE` | — | 128 × 1024 bytes per flash block |
| `FLASH_BLOCKS` | — | 128 blocks total in flash chip |
| `FLASH_BUFFER_BYTES` | — | 32-byte flash write buffer |
| `fpga_ctrl_reg` | `0x0900` | FPGA control register |
| `fpga_status_register` | `0x0904` | FPGA status register |
| `vme_aux_status` | `0x0908` | VME auxiliary status |
| `vme_config_control` | `0x090C` | VME configuration control |
| `fpga_gp_ctl` | `0x0910` | FPGA general-purpose control |
| `config_start` | `0x0918` | Start configuration command |
| `config_stop` | `0x091C` | Stop configuration command |
| `fpga_version` | `0x0920` | FPGA version register |
| `full_code_revision` | `0x0924` | Full code revision |
| `code_date_VME` | `0x0928` | Code date (VME FPGA) |
| `vme_sandbox_a–d` | `0x0930–0x093C` | Sandbox scratch registers (4 × 32-bit) |
| `code_date_2` | `0x0940` | Second code date field |
| `full_revision2` | `0x0944` | Second revision field |
| `vme_dtack_delay` | `0x0948` | VME DTACK delay setting |
| `flash_address` | `0x0980` | Flash memory address register |
| `flash_rd_wrt_autoinc` | `0x0984` | Flash read/write with auto-increment |
| `flash_rd_wrt_no_autoinc` | `0x098C` | Flash read/write without auto-increment |

**Notes:**
- Addresses 0x914, 0x92C, 0x94C–0x97C are explicitly marked unused in comments. ✅ verified 2026-04-25 — `FlashMaintenance.c:L12,18,26` (inline comments from spreadsheet)
- These are the VME FPGA configuration block registers (0x09xx range), distinct from the digitizer/trigger signal processing registers (0x00xx–0x08xx).
- The flash chip is 16 MB total (128 blocks × 128 KB), consistent with a Spansion/Cypress 128-Mbit NOR flash. ✅ verified 2026-04-25 — `FlashMaintenance.c:L3-4` (`FLASH_BLOCK_SIZE=128*1024`, `FLASH_BLOCKS=128`)

---

## equalSub.c — EPICS Equality Sub-routine

**File:** `equalSub.c` (72 L) ✅ verified 2026-04-25 — `wc -l equalSub.c`
**Purpose:** An EPICS `sub` record subroutine that checks whether up to 12 input values (INPA–INPL) are all equal, with configurable decimal precision.

### Logic

- Takes inputs A–L via `subRecord` link fields.
- `psub->prec` (PREC field, 0–3) controls comparison precision: values are scaled by `10^prec` before integer comparison (e.g., PREC=2 compares to hundredths). ✅ verified 2026-04-25 — `equalSub.c:L20-22`
- Skips `CONSTANT`-type links (unconnected inputs). ✅ verified 2026-04-25 — `equalSub.c:L30`
- First non-constant input becomes the comparisand.
- Returns 0 (success) and sets `psub->val` to the comparisand value if all non-constant inputs match.
- Returns -1 (alarm) if any mismatch is found, or if no non-constant inputs exist.

**EPICS registration:**
- `equalSubInit` — SUBL init routine (no-op)
- `equalSub` — SNAM process routine
- Registered via `iocshRegister` + `epicsExportRegistrar(equalSubRegistrar)`

**Typical use:** Comparing multiple crate/slot versions or status values to confirm they are all the same before allowing a run to proceed.

---

## restoreSub.c — EPICS PV Restore Sub-routine

**File:** `restoreSub.c` (99 L) ✅ verified 2026-04-25 — `wc -l restoreSub.c`
**Purpose:** An EPICS `sub` record subroutine that asynchronously restores PV values from a save/restore file using `fdbrestore()`. Implements PACT-style async completion to avoid blocking the IOC scan thread.

### Key Globals

- `restfilename[80]` — name of the save file to restore from (default: `"default.sav"`) ✅ verified 2026-04-25 — `restoreSub.c:L42`
- `devGDigSetRestFile(char *restfile)` — IOC shell function to change the restore filename at runtime ✅ verified 2026-04-25 — `restoreSub.c:L44-46`

### Async Restore Flow

1. `devGDigRestInit(psub)` — init routine: allocates `rcallback` struct, stores record pointer and filename pointer, sets callback function `restSubCallback` at the record's priority.
2. `devGDigRestore(psub)` — process routine:
   - If `!psub->pact`: sets `pact=TRUE`, calls `callbackRequest()` (schedules async execution), returns immediately — scan thread is not blocked. ✅ verified 2026-04-25 — `restoreSub.c:L70-72`
3. `restSubCallback(pcallback)` — runs in the EPICS callback task:
   - Calls `fdbrestore(filename)` to reload PV values from the save file.
   - Sets `psub->c` to the return code of `fdbrestore`.
   - Locks the record (`dbScanLock`), calls `process()`, unlocks — this clears `pact` and completes the async record processing. ✅ verified 2026-04-25 — `restoreSub.c:L25-38`

**EPICS registration:**
- `devGDigRestInit` — SUBL init
- `devGDigRestore` — SNAM process
- Registered via `epicsExportRegistrar(devGDigRestoreRegistrar)`

**Typical use:** Triggered from an EDM button or autosave/restore startup script to load previously saved digitizer or trigger configuration.

---

## Cross-References

- [`vxworks_vme_devlayer.md`](vxworks_vme_devlayer.md) — `daqBoards[]`, `devGVMECardInit()`, VME hardware abstraction used by `asynDebugDriver`
- [`vxworks_state_machines.md`](vxworks_state_machines.md) — `vme_driver_mutex` used by `asynDebugDriver::viOut32/viIn32`; `FlashMaintenance` acquires same mutex
- [`EPICS_asyn.md`](EPICS_asyn.md) — asyn driver architecture; `asynDigitizerDriver` as a fuller example of the same `asynPortDriver` pattern
- [`VME_registers.md`](VME_registers.md) — full digitizer/trigger register maps; FlashMaintenance 0x09xx range is in the VME FPGA config block
- [`ioc.md`](ioc.md) — boot script where `asynDebugConfig`, `asynDebugCard`, `devGDigSetRestFile` are called
