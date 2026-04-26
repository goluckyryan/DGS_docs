# VxWorks VME Device Support Layer — devGVME / devGData / DGS_DEFS

Stability: C3 - Structural / stable

_Source: `vxworks/dgsDrivers/dgsDriverApp/src/devGVME.c` (1,083 L), `devGData.c` (265 L), `devGVME.h`, `DGS_DEFS.h` (392 L)_
_Documented: 2026-04-24_

**See also:** [`EPICS_asyn.md`](EPICS_asyn.md) — asyn driver layer above this, [`ioc.md`](ioc.md), [`VME_registers.md`](VME_registers.md), [`vxworks_fifo_readout.md`](vxworks_fifo_readout.md) — DMA buffer pipeline using `daqBoards[]`/`DGS_DEFS.h` constants, [`vxworks_state_machines.md`](vxworks_state_machines.md) — inLoop/outLoop state machines that call VMERead32/VMEWrite32

---

## Overview

`devGVME.c` is the **core VxWorks VME hardware abstraction layer** for the DGS IOC. It provides:

1. **Board initialization** — maps VME slot addresses into local memory, detects board type
2. **Raw VME read/write** — mutex-protected 32-bit register access
3. **Flash programming** — verify, erase, program, download, and reconfigure firmware flash
4. **IOCShell commands** — exposes key functions as interactive shell commands
5. **Global data structures** — `daqBoards[]` array used by all VxWorks drivers

`devGData.c` provides **EPICS device support records** (`ai`, `ao`, `bo`) for a subset of board-level state that does not go through the asyn driver path.

---

## Key Data Structures (`DGS_DEFS.h`)

### `daqBoard` (one entry per VME slot, indexed by `cardno`)

| Field | Type | Description |
|-------|------|-------------|
| `vmeRegisters[0x24]` | `daqRegister[]` | Per-register metadata: mapped address, mutex, shadow copy (`copy`), `dibs` flag |
| `base32` | `volatile uint32_t *` | Base pointer to start of main FPGA address space (byte 0x000) |
| `FIFO` | `volatile uint32_t *` | Pointer to digitizer external FIFO |
| `vmever` | `uint16_t` | Firmware version from register 0x920 (bits 31:16) |
| `rev` / `subrev` | `uint32_t` | Firmware revision sub-fields |
| `mainOK` | `uint16_t` | Flag: 1 = main FPGA OK |
| `board` | `uint16_t` | VME slot number |
| `EnabledForReadout` | `uint16_t` | Runtime enable flag for inLoop (set via `devBoGData` record) |
| `DigUsrPkgData` / `TrigUsrPkgData` | `int` | User package data for Type F GEB headers |
| `router` | `uint16_t` | Non-zero if this board is a Router Trigger |
| `board_type` | `uint16_t` | Firmware-derived board type index (see table below) |

**Global array:** `struct daqBoard daqBoards[7]` — one entry per VME slot (`GVME_MAX_CARDS = 7`)

### `daqRegister` (per-VME-register metadata)

```c
struct daqRegister {
   volatile unsigned int *addr;   // mapped VME address
   epicsMutexId sem;              // per-register write mutex
   int tick;                      // last-update tick count
   unsigned int copy;             // last-written value (shadow)
   unsigned int dibs;             // "dibs" flag (ownership indicator)
};
```

`GVME_NUM_REGISTERS = 0x24 = 36` — covers VME FPGA registers 0x900–0x98C (9 bytes per register × 36 = 0x8C, VME FPGA register block).

### Board Type Codes (`board_type`)

| Index | `BrdType_*` | Name |
|-------|------------|------|
| 0 | `BrdType_NO_BOARD` | No Board Present |
| 1 | `BrdType_GRETINA_RTRIG` | GRETINA Router Trigger |
| 2 | `BrdType_GRETINA_MTRIG` | GRETINA Master Trigger |
| 3 | `BrdType_LBNL_DIG` | LBNL Digitizer |
| 4 | `BrdType_DGS_MTRIG` | DGS Master Trigger |
| 5 | (undefined) | — |
| 6 | `BrdType_DGS_RTRIG` | DGS Router Trigger |
| 7 | (undefined) | — |
| 8 | `BrdType_MYRIAD` | MyRIAD (assigned by JTA) |
| 9–11 | (undefined) | — |
| 12 | `BrdType_ANL_MDIG` | ANL Master Digitizer (code_revision = 4XYZ) |
| 13 | `BrdType_ANL_SDIG` | ANL Slave Digitizer |
| 14 | `BrdType_MAJORANA_MDIG` | Majorana Master Digitizer (code_revision = FXYZ) |
| 15 | `BrdType_MAJORANA_SDIG` | Majorana Slave Digitizer |

Board type assignment — **corrected 2026-04-24** (prior description was wrong): ✅ verified 2026-04-24
- **Triggers (MTRIG/RTRIG):** `board_type` is set directly and unconditionally by the asyn driver at init time — `BrdType_DGS_MTRIG` by `asynTrigMasterDriver.cpp:L154`, `BrdType_DGS_RTRIG` by `asynTrigRouterDriver.cpp:L142`. No register lookup; determined by which driver was invoked.
- **Digitizers:** `board_type` is derived from bits 15:8 of the main FPGA register at offset **0x600** (`daqBoards[].rev`), using `(rev & 0x0000FF00) >> 8` — `0x4C`→MDIG, `0x4D`→SDIG, `0xFC`→Majorana MDIG, `0xFD`→Majorana SDIG. Source: `asynDigitizerDriver.cpp:L188-201`.
- The `full_code_revision` register (0x924) comment that mentions "bits 11:8" refers to the VME FPGA firmware type field, not the runtime code path for setting `board_type`.

### `rawEvt` — Buffer Descriptor

Used by inLoop/outLoop state machines to pass DMA buffer ownership:

```c
typedef struct {
   unsigned int id;            // unique buffer ID (immutable)
   unsigned int *datapcrosscheck; // sanity-check copy of data pointer
   unsigned int board;         // source board number
   unsigned int len;           // data length in BYTES
   unsigned int *data;         // pointer to raw DMA data buffer
   owner_enum owner;           // OWNER_FREE / INLOOP / Q_WRITTEN / OUTLOOP / Q_SENDER / SENDER
   unsigned short board_type;  // board type code (added 20220801)
   unsigned short data_type;   // 0 = normal; non-zero = board-specific special type
} rawEvt;
```

`owner_enum` values: `OWNER_UNDEF=0, OWNER_Q_FREE=1, OWNER_INLOOP=2, OWNER_Q_WRITTEN=3, OWNER_OUTLOOP=4, OWNER_Q_SENDER=5, OWNER_SENDER=6` ✅ verified 2026-04-25 — DGS_DEFS.h:L202-209

---

## `devGVMECardInit(cardno, slot)` — Board Initialization

Called from the IOC boot script for each DIG/trigger board. Steps:

1. Compute VME base address: `base = slot << 20` (VME64x slot addressing) ✅ verified 2026-04-24 — devGVME.c:L131
2. Call `sysBusToLocalAdrs(0x0a, base, &newbase)` to map into CPU address space (`0x0b` for RIO3 boards) ✅ verified 2026-04-24 — devGVME.c:L140-142 (`#ifdef RIO3 space=0x0b #else space=0x0a`)
3. Store mapped address in `daqBoards[cardno].base32`; probe register 0x920 (`newbase + 0x248`) via `devReadProbe` to confirm board responds ✅ verified 2026-04-24 — devGVME.c:L163,L185 (comment: "0x248 is 0x920 >> 2")
4. Extract firmware version: `vmever = (vmever >> 16) & 0xFFFF` (modified 2017 for DGS — was `>> 24`) ✅ verified 2026-04-25 — devGVME.c:L199-201 (comment: `cut 20171016 JTA`)
5. Advance local `newbase` to `newbase + 0x900/4` (start of VME FPGA register block); `base32` retains the raw slot base ✅ verified 2026-04-24 — devGVME.c:L204-207
6. Allocate one `epicsMutex` per register for the `GVME_NUM_REGISTERS` (36) VME FPGA registers ✅ verified 2026-04-24 — devGVME.c:L220-222

**Key VME register constants (byte addresses):**

| Constant | Address | Description |
|----------|---------|-------------|
| `fpga_ctrl_reg` | 0x0900 | FPGA control |
| `fpga_status_register` | 0x0904 | FPGA status |
| `vme_aux_status` | 0x0908 | VME aux status |
| `vme_config_control` | 0x090C | Reconfiguration control |
| `fpga_gp_ctl` | 0x0910 | General-purpose control |
| `config_start` | 0x0918 | Flash config start address |
| `config_stop` | 0x091C | Flash config stop address |
| `fpga_version` | 0x0920 | Firmware version (bits 31:16 = code_revision[15:0], bits 11:0 = serial_number) |
| `full_code_revision` | 0x0924 | Full revision (board_type in bits 11:8) |
| `code_date_VME` | 0x0928 | Code date |
| `vme_sandbox_a–d` | 0x0930–0x093C | Sandbox registers |
| `code_date_2` | 0x0940 | Second code date register |
| `vme_dtack_delay` | 0x0948 | DTACK delay setting |
| `flash_address` | 0x0980 | Flash access address pointer |
| `flash_rd_wrt_autoinc` | 0x0984 | Flash read/write with auto-increment |
| `flash_rd_wrt_no_autoinc` | 0x098C | Flash read/write without auto-increment |

---

## Raw VME Access Functions

### `VMEWrite32(bdnum, regaddr, data)`
- Locks `vme_driver_mutex`
- Computes pointer: `ptr = (int*)(daqBoards[bdnum].base32 + regaddr/4)`
- Writes `data` to VME
- Unlocks mutex

### `VMERead32(bdnum, regaddr)` → `unsigned int`
- Same locking pattern
- Stores result in global `VMERead32TempVal` (shell-accessible)
- Returns value

Both are registered as IOCShell commands: `VMEWrite32 bdnum regaddr data`, `VMERead32 bdnum regaddr`

---

## Flash Programming Functions

All flash functions hold `vme_driver_mutex` for their entire duration (no ctrl-X/ctrl-C abort).

Flash geometry: **128 blocks × 128 KB = 16 MB total chip** ✅ verified 2026-04-24 — devGVME.c:L367-368. Each operation uses one bank = `FLASH_BLOCKS/4` = 32 blocks × 128 KB = **4 MB** ✅ verified 2026-04-24 — devGVME.c:L414,L427. `address_control=0` → bank 0 (primary, offset=0), `=1` → bank 1 (backup, offset=4 MB). Only 8 MB (2 banks) of the 16 MB chip is used.

Byte-swap note: VxWorks reads flash bytes in reversed order vs. `.bin` files (big-endian vs. little-endian). All read/write functions apply a 4-byte reversal per 32-bit word.

| Function | IOCShell command | Description |
|----------|-----------------|-------------|
| `VerifyFlash(bdnum, address_control, StopOnErrorCount_flag, fname)` | `VerifyFlash` | Compare flash bank to file; reports mismatch count; stops early after >100 errors if flag=1 |
| `EraseFlash(bdnum, address_control)` | `EraseFlash` | Erase one 4 MB bank (32 × 128 KB blocks) |
| `ProgramFlash(bdnum, address_control, fname)` | `ProgramFlash` | Write `.bin` file into one 4 MB bank |
| `DownloadFlash(bdnum, address_control, fname)` | `DownloadFlash` | Read one 4 MB bank back to file (byte-swapped back to .bin layout) |
| `ConfigureFlash(bdnum, address_control)` | `ConfigureFlash` | Trigger FPGA reconfiguration from flash; polls `fpga_status_register` bit 1 (`CONFIG_COMPLETE`, mask `0x0002`) for up to 40 × `taskDelay(10)` cycles; reports timeout or OK ✅ verified 2026-04-24 — devGVME.c:L1019-1040 |

`ConfigureFlash` sequence: write `vme_config_control` with: 0x2 (ack) → 0x0 → `address_offset` → `address_offset + 4MB` → 0x2 → 0x0 → 0x1 (request reconfigure), then poll status bit 1.

---

## `devGData.c` — EPICS Record Device Support

Provides EPICS device support for records using `VME_IO` link type (`#C$(DC) S<signal> @`):

- `C` field = card number (maps to `daqBoards[]` index)
- `S` field = signal code (function selector)
- `@` field = unused parameter string

### `devAiGData` — `ai` record (read-only)

Signal codes 0x01–0x0B were formerly channel rate monitors (functions now commented out — stubs). Signal 0x0C was `getCardLatest`. Signals 0xD/0xE were `getKBSentDiff`/`getKBReadDiff`. **All ai signal handlers are currently stub/commented — they return success but produce no real data.** These represent legacy GammaWare-era rate monitoring that was never fully ported to DGS.

### `devBoGData` — `bo` record (write-only)

Signal 1: writes `pbo->rval` into `daqBoards[card].EnabledForReadout`. This is the mechanism by which an EPICS PV (e.g., DIG "enabled for readout" checkbox) sets the per-board enable flag consumed by inLoop.

### `devAoGData` — `ao` record (write, added 2022-07-18)

All signals: simply stores `pao->rval` into `pao->val`. Used to hold arbitrary PV values in IOC memory for state machine consumption — no direct hardware access.

### `daqDevPvt` structure (per-record private data)

```c
struct daqDevPvt {
   struct daqRegister *reg;   // VME register pointer (unused in devGData)
   unsigned int mask;         // mask (unused in devGData)
   unsigned short shft;       // shift (unused in devGData)
   unsigned short signal;     // signal code from S field
   unsigned short card;       // card number from C field
   unsigned short chan;        // channel (unused in devGData)
};
```

---

## Outloop Global Variables (from `devGVME.c`)

Global vars set by `outloop.st` (EPICS sequencer) and read by `outloopsupport.c`:

| Variable | Default | Purpose |
|----------|---------|---------|
| `OL_Hdr_Chk_En` | 1 (on) | Enable GEB header validation in outLoop |
| `OL_TS_Chk_En` | 1 (on) | Enable timestamp check |
| `OL_Deep_Chk_En` | 1 (on) | Enable deep packet content check |
| `OL_Hdr_Summ_En` | 0 (off) | Enable header summary logging |
| `OL_Hdr_Summ_PS` | 0x1000 | Header summary prescale |
| `OL_Hdr_Summ_Evt_PS` | 0x100 | Event-level summary prescale |

✅ verified 2026-04-25 — devGVME.c:L64-69 (all defaults confirmed)

---

## Key `DGS_DEFS.h` Constants

| Constant | Value | Meaning |
|----------|-------|---------|
| `RAW_BUF_SIZE` | 1 MB | DMA buffer allocation size | ✅ verified 2026-04-25 — DGS_DEFS.h:L34 (`1024*1024`, changed from 512KB on 20230412) |
| `MAX_DIG_RAW_XFER_SIZE` | 512 KB | DIG FIFO transfer size limit | ✅ verified 2026-04-25 — DGS_DEFS.h:L41 |
| `MAX_TRIG_RAW_XFER_SIZE` | 256 KB | Trigger FIFO transfer size limit (4×65536 bytes) | ✅ verified 2026-04-25 — DGS_DEFS.h:L50 |
| `DMA_CHUNK_SIZE_IN_BYTES` | 0x10000 (64 KB) | Actual max DMA chunk size (discovered 2025-06-07) | ✅ verified 2026-04-25 — DGS_DEFS.h:L53 |
| `RAW_Q_SIZE` | 200 | Buffer pool depth (changed from 400 on 20230412) | ✅ verified 2026-04-25 — DGS_DEFS.h:L48 |
| `GVME_MAX_CARDS` | 7 | VME slots per crate (annotated 20230921) | ✅ verified 2026-04-25 — DGS_DEFS.h:L192 |
| `GVME_NUM_REGISTERS` | 0x24 (36) | VME FPGA registers tracked per board | ✅ verified 2026-04-25 — DGS_DEFS.h:L191 |
| `MAX_EVENTS_TO_CHECK_PER_BUFFER` | 128 | outLoop event check limit per buffer | ✅ verified 2026-04-25 — DGS_DEFS.h:L126 |
| `PROG_FULL_MASK` | 0x00800000 | External FIFO programmed-full flag (from `regin_programming_done[19]`) | ✅ verified 2026-04-25 — DGS_DEFS.h:L160 |
| `EMPTY_MASK` | 0x00300000 | External FIFO empty flags (bits 20–21) | ✅ verified 2026-04-25 — DGS_DEFS.h:L164 |
| `SCAN_LOOP_MINIMUM_DELAY` | 0.001 s | inLoop minimum poll delay | ✅ verified 2026-04-25 — DGS_DEFS.h:L92 |
| `SCAN_LOOP_MAXIMUM_DELAY` | 0.300 s | inLoop maximum poll delay (at 4 DIGs) | ✅ verified 2026-04-25 — DGS_DEFS.h:L95 (comment: "Good for 4 digitizer") |

---

## Aug 2022 `rawEvt` Redesign — Historical Context

**Source:** `DGS_tools_pack/vxworks/dgsDrivers/dgsDriverApp/src/ChangeNotes_20220801.txt` (email thread, Oberling ↔ Anderson, 2022-07-20 to 2022-08-01)

**Decision:** Eliminated the old requirement that "trigger data must look like digitizer data." Before Aug 2022, trigger buffers had to be formatted to resemble digitizer data for the receiver to process them. The redesign decoupled trigger data from digitizer data at the sender/receiver boundary.

**Key changes made (2022-08-01):**
- Added `board_type` (uint16) and `data_type` (uint16) fields to `rawEvt` struct (replacing the old `char *board_type` pointer and removing `trig_buffer_converter.c`)
- `inLoop` assigns `board_type` (from `daqBoards[]`) and `data_type` (0 = normal digitizer; FIFO index for triggers) when pulling buffers from the queue
- `outLoopSupport.c` → `CheckAndMoveBuffers()` now uses a `switch(rawBuf->board_type)` to route by board type: digitizers decomposed normally, triggers and MYRIAD passed forward as-is, unknown type logs an error and returns
- The receiver (`dgsReceiver` / gtReceiver) was updated to dump non-digitizer data to a `<run_name>_DIAG` file; the current production receiver (`tcpReceiverMT`) does not implement this DIAG file feature
- `data_type = 0` = nominal for digitizers (previously implied `0xFF` was considered as the "unspecified" type but `0x00` was adopted for backwards compatibility)

**Slot map for firmware upload (`uploadFW.cmd`)** — also confirmed 2022-08:

| Slot | Board | File | Function |
|------|-------|------|----------|
| 3 (bdnum 0) | MDIG1 | `BUS_LEFT.bin` | DIG front bus sender |
| 4 (bdnum 1) | MDIG2 | `BUS_RIGHT.bin` | DIG front bus receiver |
| 6 (bdnum 4) | RTRG | `router_top.bin` (or `V4747_mod_router_top.bin`) | Router trigger FPGA |
| 7 (bdnum 5) | MTRG | `trigger_top.bin` | Master trigger FPGA |

Sequence: `ProgramFlash(bdnum, bank=0, file)` → `taskDelay(100)` → `ConfigureFlash(bdnum, 0)`. All four boards re-configured a second time at end of script for safety. ✅ verified 2026-04-26 — `ANLDAQ/ioc/firmware/uploadFW.cmd` (full file read)

---

## Cross-References

| File | Relationship |
|------|--------------|
| `vxworks.md` | VxWorks IOC overview including devGVME.c context |
| `vxworks_fifo_readout.md` | FIFO readout layer built on top of this devlayer |
| `vxworks_state_machines.md` | inLoop/outLoop state machines using the structures defined here |
| `vxworks_trigger_drivers.md` | Trigger asyn drivers (asynTrigMaster/Router) use same VME mutex |
| `ioc.md` | IOC boot scripts calling `devGVMECardInit` |
| `IOC_cmd.md` | VMERead32/VMEWrite32 shell commands wrapping this layer |

---

*Created: 2026-04-24 | Last reviewed: 2026-04-25*
