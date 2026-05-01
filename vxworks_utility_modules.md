# VxWorks Utility Modules — profile, asynDebugDriver, FlashMaintenance, equalSub, restoreSub, MergedAsynDigParams, QueueManagement

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
6. [MergedAsynDigParams.c — DIG Asyn Parameter Registration](#mergedasyndigparamsc--dig-asyn-parameter-registration)
7. [QueueManagement.c — Three-Queue Event Buffer Pool](#queuemanagementc--three-queue-event-buffer-pool)

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

## MergedAsynDigParams.c — DIG Asyn Parameter Registration

**File:** `MergedAsynDigParams.c` (672 L) ✅ verified 2026-04-22 — `wc -l MergedAsynDigParams.c` (222 `createParam()` calls)
**Purpose:** Registers all DIG EPICS PV names with the asyn framework. Not a standalone translation unit — `#include`d directly into the `drvAsynDigitizer.c` constructor body (see [`IOC_cmd.md`](IOC_cmd.md) for `asynDigitizerConfig` boot script call).

### Design

Each `createParam()` call maps a string PV name to an integer parameter ID:
```c
createParam("reg_led_threshold0", asynParamUInt32Digital, &reg_led_threshold0);
```
All 222 parameters use type `asynParamUInt32Digital`. Integer handles are member variables declared in `asynDigParams.h` / `MergedAsynDigParams.h`.

### Naming Conventions

| Prefix | Scope | Description |
|--------|-------|-------------|
| `reg_<name>N` | Per-channel (N = 0–9) | Writable register. E.g. `reg_led_threshold0`–`reg_led_threshold9` |
| `regin_<name>N` | Per-channel | Read-back-only register |
| (no suffix) | Board-level | E.g. `reg_programming_done`, `regin_board_id`, `SERIAL_NUMBER`, `vme_clk_ctrl` |

### Parameter Groups (35 unique base names)

**Per-channel (×10 each):** `reg_channel_control`, `reg_channel_pulsed_control`, `reg_led_threshold`, `reg_CFD_fraction`, `reg_raw_data_delay`, `reg_raw_data_length`, `reg_d_window`, `reg_k_window`, `reg_m_window`, `reg_d3_window`, `reg_disc_width`, `reg_baseline_delay`, `reg_downsample_holdoff`, `reg_led_control`, `reg_holdoff_control`, `reg_veto_gate_width`, `reg_p1_window`, `reg_p2_window`, `reg_dac`, `reg_diag_channel_input`, `reg_diag_mux_control`, `reg_sd_config`, `reg_trigger_config`, `reg_external_disc_mode`, `reg_ila_config`, `regin_disc_count`, `regin_ahit_count`, `regin_led_state`, `regin_hilo_*`, `regin_hihilolo_*`, `regin_phase_offset_a/b/c`, `regin_serdes_phase_value`, `regin_phase_value`, `regin_phase_errors`

**Board-level (no digit suffix):** `regin_board_id`, `regin_code_revision`, `regin_code_date`, `regin_hardware_status`, `regin_accepted_event_count`, `regin_dropped_event_count`, `regin_lat_timestamp_lsb/msb`, `regin_live_timestamp_lsb/msb`, `regin_ts_err_count`, `reg_ts_err_count_ctrl`, `reg_master_logic_status`, `reg_programming_done`, `reg_external_discriminator_src`, `reg_vme_ext_delay`, `reg_user_package_data`, `reg_win_comp_min`, `reg_win_comp_max`, `reg_rj45_spare_dout_control`, `SERIAL_NUMBER`, `vme_clk_ctrl`, `vme_gp_ctrl`, `VME_MON_STATUS`

**Role:** After `createParam()`, the driver's `readInt32`/`writeInt32` dispatch table routes `caput`/`caget` traffic to the matching VME register read/write in `drvAsynDigitizer.c`.

**Cross-reference:** `drvAsynDigitizer.c` → VME register map in [`VME_registers.md`](VME_registers.md); PV naming convention in [`ANLDAQ.md`](ANLDAQ.md).

---

## Cross-References

- [`vxworks_vme_devlayer.md`](vxworks_vme_devlayer.md) — `daqBoards[]`, `devGVMECardInit()`, VME hardware abstraction used by `asynDebugDriver`
- [`vxworks_state_machines.md`](vxworks_state_machines.md) — `vme_driver_mutex` used by `asynDebugDriver::viOut32/viIn32`; `FlashMaintenance` acquires same mutex
- [`EPICS_asyn.md`](EPICS_asyn.md) — asyn driver architecture; `asynDigitizerDriver` as a fuller example of the same `asynPortDriver` pattern
- [`VME_registers.md`](VME_registers.md) — full digitizer/trigger register maps; FlashMaintenance 0x09xx range is in the VME FPGA config block
- [`ioc.md`](ioc.md) — boot script where `asynDebugConfig`, `asynDebugCard`, `devGDigSetRestFile` are called
- [`vxworks_fifo_readout.md`](vxworks_fifo_readout.md) — FIFO readout pipeline (DMA buffer arch, readTrigFIFO.c, Type-F headers); QueueManagement section there is older (2026-04-22) — this file has the authoritative updated version
- [`ANLDAQ_tcpReceiver.md`](ANLDAQ_tcpReceiver.md) — PC-side TCP receiver that MiniSender connects to; handles multi-crate aggregation and event building
- [`IOC_cmd.md`](IOC_cmd.md) — IOC boot script commands including `asynDigitizerConfig` (calls `drvAsynDigitizer.c`), `setupFIFOReader`, sequencer launches

---

## QueueManagement.c — Three-Queue Event Buffer Pool

**File:** `QueueManagement.c` (495 L), `QueueManagement.h` (61 L) ✅ verified 2026-04-26 — `wc -l QueueManagement.c QueueManagement.h`  
**Author:** Michael Oberling  
**Purpose:** Manages the VxWorks message-queue-based event buffer pool that mediates data flow between the inLoop FIFO reader, the outLoop data packer, and the MiniSender TCP transmitter (all three state machines documented in [`vxworks_state_machines.md`](vxworks_state_machines.md); MiniSender connects to [`ANLDAQ_tcpReceiver.md`](ANLDAQ_tcpReceiver.md) on the PC side).

### Architecture: Three-Queue Buffer Pool

The DGS IOC VxWorks data pipeline moves raw digitizer/trigger events between tasks using a **fixed-size pool** of `rawEvt` buffer descriptors shared across three VxWorks `MSG_Q_ID` message queues:

```
  qFree  ──► inLoop reads FIFO ──► qWritten ──► outLoop packs ──► qSender ──► MiniSender sends ──► qFree
              (getFreeBuf)            (putWrittenBuf)              (putSenderBuf)                   (putFreeBuf)
```

| Queue | VxWorks Handle | Direction | Description |
|-------|---------------|-----------|-------------|
| `qFree` | `MSG_Q_ID` | recycled in | Pool of available (empty) buffers — inLoop dequeues from here |
| `qWritten` | `MSG_Q_ID` | written-to | Buffers filled by inLoop and ready for outLoop |
| `qSender` | `MSG_Q_ID` | ready-to-send | Buffers packed by outLoop and ready for MiniSender |

**Buffer count:** `RAW_Q_SIZE` (defined in `DGS_DEFS.h`) buffers exist total. The pool is pre-allocated at startup and never grows — all buffers are `rawEvt` structs with a `data[]` field of size `RAW_BUF_SIZE` bytes. ✅ verified 2026-04-26 — `DGS_DEFS.h:L48` (`RAW_Q_SIZE = 200`, changed from 400 on 2023-04-12 JTA); `DGS_DEFS.h:L34` (`RAW_BUF_SIZE = 1024*1024` bytes = 1 MB, changed from 512 KB on 2023-04-12 JTA)

### Buffer Lifecycle

Each buffer has an `owner` field (enum) that tracks who currently holds it:

| State | `owner` value | Set by |
|-------|--------------|--------|
| In free queue | `OWNER_Q_FREE` | `putFreeBuf()` |
| Dequeued by inLoop | `OWNER_INLOOP` | `getFreeBuf()` |
| In written queue | `OWNER_Q_WRITTEN` | `putWrittenBuf()` |
| Dequeued by outLoop | `OWNER_OUTLOOP` | `getWrittenBuf()` |
| In sender queue | `OWNER_Q_SENDER` | `putSenderBuf()` |
| Dequeued by MiniSender | `OWNER_SENDER` | `getSenderBuf()` |
| Undefined | `OWNER_UNDEF` | `newEventBuffer()` at alloc time |

### Key Functions

| Function | Description |
|----------|-------------|
| `setupFIFOReader()` | Called once from VxWorks startup script (`vme01.cmd`) before sequencers. Creates all three message queues, allocates all `rawEvt` buffers, puts them all on `qFree`. Re-callable (deletes and re-creates queues; buffers are not reallocated). |
| `getFreeBuf(rawEvt **)` | Removes a buffer from `qFree` (NO_WAIT). Sets `data[0]=0x87654321` sentinel, marks `OWNER_INLOOP`. Returns `NoBufferAvail` if queue empty. ✅ verified 2026-04-26 — `QueueManagement.c:L244-245` |
| `putWrittenBuf(rawEvt *)` | Puts filled buffer onto `qWritten`. Marks `OWNER_Q_WRITTEN`. |
| `getWrittenBuf(rawEvt **)` | Removes buffer from `qWritten`. Marks `OWNER_OUTLOOP`. |
| `putSenderBuf(rawEvt *)` | Puts packed buffer onto `qSender`. Marks `OWNER_Q_SENDER`. |
| `getSenderBuf(rawEvt **)` | Removes buffer from `qSender`. Marks `OWNER_SENDER`. |
| `putFreeBuf(rawEvt *)` | Returns used buffer to `qFree`. Resets `len=0`, `board=-1`, `data[0]=0x12345678`. Marks `OWNER_Q_FREE`. ✅ verified 2026-04-26 — `QueueManagement.c:L298-301` |
| `getFreeBufCount()` | Returns `msgQNumMsgs(qFree)` — available buffer count. |
| `getWrittenBufCount()` | Returns `msgQNumMsgs(qWritten)`. |
| `getSenderBufCount()` | Returns `msgQNumMsgs(qSender)`. |
| `DumpRawEvt(rawEvt*, char*, int, int)` | Debug: prints `id`, `board`, `len`, `owner`, and optionally dumps `data[]` words. |
| `bufDiag(rawEvt*, char*, ...)` | Internal diagnostic: checks for NULL pointer, data pointer corruption, short-length buffers, invalid `0xAAAA...` sentinel — compiled in only when `PRINT_BUFFER_ERRORS` is defined. |
| `newEventBuffer(rawEvt **)` | Internal: `calloc(sizeof(rawEvt), 1)` + `calloc(RAW_BUF_SIZE, 1)` for `data`. DMA variant: `cacheDmaMalloc` + 256-byte alignment (for CES rio3 DMA engine). |

### DMA Support (Conditional)

When `READOUT_USE_DMA` + `MV5500` are both `#define`d at build time:
- `setupFIFOReader()` initializes the **Universe VME DMA engine** (`sysVmeDmaInit()`)
- DMA is configured as A32, 32-bit wide, block transfer (`DCTL_VDW_32 | DCTL_VCT_BLK`)
- A DMA semaphore `DMASem` (type `epicsEventId`) is created for DMA-completion signaling
- Buffers are allocated with `cacheDmaMalloc` instead of `calloc`, with 256-byte alignment padding

In the current production build (non-DMA PIO mode), all three of these are skipped and standard `calloc` is used.

✅ verified 2026-04-26 — `QueueManagement.c:L22-24` (`READOUT_USE_DMA`+`MV5500` guard); `L63-65` (`sysVmeDmaInit()`, `DCTL_VDW_32|DCTL_VCT_BLK`, `DCTL_VAS_A32`); `L67` (`DMASem = epicsEventCreate(epicsEventFull)`); `L150-158` (`cacheDmaMalloc(RAW_BUF_SIZE + 256)`, 256-byte alignment for CES rio3 DMA engine)

### Message Queue Internals (Developer Note)

The file contains an extended comment block (L411–L495) explaining how VxWorks `MSG_Q_ID` works at the implementation level — included verbatim in the source for developer reference:
- `MSG_Q_ID` is `struct msg_q *` (defined in `private/msgQLibP.h`)
- `msgQNumMsgs()` is the correct way to query current queue depth
- `msgQInfoGet()` provides `MSG_Q_INFO` struct with `numMsgs`, `numTasks`, `sendTimeouts`, `recvTimeouts`, `maxMsgs`, `maxMsgLength`

### Startup Sequence

From `vxworks/dgsIoc/iocBoot/iocArray/vme01.cmd`:
```
iocInit()           # EPICS IOC init
dumpFIFO = 0        # FIFO dump flag (0 = normal, 1 = dump to stdout)
setupFIFOReader()   # creates queues + allocates buffer pool
seq &inLoop,...     # starts FIFO reader (uses getFreeBuf/putWrittenBuf)
seq &outLoop,...    # starts packer (uses getWrittenBuf/putSenderBuf)
seq &MiniSender,... # starts TCP sender (uses getSenderBuf/putFreeBuf)
```
✅ verified 2026-04-26 — `vxworks/dgsIoc/iocBoot/iocArray/vme01.cmd:L121-139` (setupFIFOReader + seq calls)

### Board Type Validation in `bufDiag`

When buffer validation is enabled (`PRINT_BUFFER_ERRORS`), `bufDiag()` checks the `0xAAAAAAAA` sentinel that the FIFO readout code writes at `data[0]` per board type:

| Board type | Expected `data[0]` |
|------------|-------------------|
| `BrdType_ANL_MDIG`, `BrdType_ANL_SDIG`, `BrdType_MAJORANA_MDIG`, `BrdType_MAJORANA_SDIG`, `BrdType_LBNL_DIG` | `0xAAAAAAAA` |
| `BrdType_DGS_MTRIG` | `0x0000AAAA` (lower 16-bit only, upper 16-bit must be 0) |

This is consistent with the FIFO readout formats in [`vxworks_fifo_readout.md`](vxworks_fifo_readout.md).
