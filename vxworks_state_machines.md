# VxWorks DAQ State Machines & Runtime Drivers

Stability: C2 - Active / semi-stable

Runtime components of the DGS VxWorks IOC: three-machine data pipeline (inLoop → outLoop → MiniSender), trigger board drivers (RTRG/MTRG), shared VME bus mutex, and queue management.

_Split from [`vxworks.md`](vxworks.md) on 2026-04-23. Sources: `dgsDrivers/dgsDriverApp/src/`_

---

## Table of Contents

- [inLoop.st — VME FIFO Readout State Machine](#inloopst--vme-fifo-readout-state-machine)
- [outLoop.st — Data Validation and Buffer Routing State Machine](#outloopst--data-validation-and-buffer-routing-state-machine)
- [MiniSender.st — Network Send State Machine](#minisenderst--network-send-state-machine)
- [Port 9010 On-Demand FIFO Grabber (Planned)](#port-9010-on-demand-fifo-grabber-planned--not-implemented)
- [Trigger Board Drivers](#trigger-board-drivers-asynTrigCommonDriver-asynTrigRouterDriver-asynTrigMasterDriver)
- [vmeDriverMutex — Cross-Driver VME Bus Lock](#vmedrivermutex--cross-driver-vme-bus-lock)
- [QueueManagement.c — Three-Queue Buffer Pool](#queuemanagementc--three-queue-buffer-pool)
- [asynDigitizerDriver — DIG VME Asyn Driver](#asyndigitizerdriver--dig-vme-asyn-driver)
- [Cross-References](#cross-references)

---

### inLoop.st — VME FIFO Readout State Machine

_Source: `dgsDrivers/dgsDriverApp/src/inLoop.st` (969 lines) + `inLoopSupport.c` (933 lines), EPICS State Notation Language + C_ ✅ verified 2026-04-23 — full read

**Purpose:** The first stage of the three-machine pipeline (inLoop → outLoop → MiniSender). One instance runs per VME crate. It scans across all enabled boards in the crate, pulls data from each board's FIFO, and deposits filled buffers into `QWritten` for `outLoop.st` to validate.

**Two concurrent state sets:**
1. `ss InLoopStats` — PV statistics refresh (runs every 0.5 s in parallel)
2. `ss inLoop` — main data readout control flow

#### Stats State Set (`ss InLoopStats`)

Runs independently of the main machine. Updates every `UpdateDelay = 0.5 s`:

| PV | Variable | Content |
|----|----------|---------|
| `DAQC{CRATE}_CV_InLoop1` | `MBytesPerSec` | Total MB transferred (cumulative, not rate — despite name) |
| `DAQC{CRATE}_CV_InLoop2` | `Local_FBufferCount` | Free buffer count mirror |
| `DAQC{CRATE}_CV_InLoop3` | `NumXfers` | Cumulative DIG readout count |

Also manages the `SendNextEmpty[]` prescale array — a per-board flag that throttles how often type-F (empty) headers are injected when a board has no data.

#### Main State Machine (`ss inLoop`)

| State | Trigger | Action |
|-------|---------|--------|
| `INIT` | Entry | Task priority 190; reset counters; call `SetupBoardAddresses()`; set `InloopIsRunning = 0`; publish `BoardType0–6` PVs |
| `INIT` | `AcqRun == 1` | Re-call `SetupBoardAddresses()`; set `InloopIsRunning = 1`; go to `INITIAL_FIFO_CLEAR` |
| `INIT` | `delay(10)` | Refresh `FIFO_index[]` array from `B0–B6FifoNum` PVs; stay in `INIT` |
| `INITIAL_FIFO_CLEAR` | Entry (always) | Loop all boards: DIG → `ClearDigMstrLogicEnable()` + `ClearDigFIFO()` + `CalcDigMaxEventsPerRead()`; MTRG → `ClearTrigFIFO()`; RTRG/GRETINA → no-op |
| `INITIAL_FIFO_CLEAR` | `NumBoardsEnabled == 0` | No boards enabled → go to `IDLE_ERROR` |
| `INITIAL_FIFO_CLEAR` | else | Go to `ENABLE_DIGITIZERS` |
| `IDLE_ERROR` | `!AcqRun` | Return to `INIT` (waits for operator to stop run) |
| `ENABLE_DIGITIZERS` | Entry | Loop all boards: call `EnableModule()` (type-aware); go to `SCAN_FOR_DATA` |
| `SCAN_FOR_DATA` | `getFreeBufCount() < 10` | Go to `SCAN_DELAY` (back-pressure — let outLoop catch up) |
| `SCAN_FOR_DATA` | `!AcqRun` | Go to `DISABLE_COLLECTION` |
| `SCAN_FOR_DATA` | else | Loop all boards: DIG → `CheckAndReadDigitizer()`; MTRG → `CheckAndReadTrigger()`; go to `SCAN_DELAY` |
| `SCAN_DELAY` | Entry | Call `UpdateScanDelay()` to set adaptive `ScanDelay` based on buffer utilization |
| `SCAN_DELAY` | `!AcqRun` | Go to `DISABLE_COLLECTION` |
| `SCAN_DELAY` | `delay(ScanDelay)` | Go back to `SCAN_FOR_DATA` |
| `DISABLE_COLLECTION` | Entry (always) | Loop all boards: DIG → `ClearDigMstrLogicEnable()`; MTRG → `SetTrigSoftwareVeto()` is **commented out** (no-op in practice — `//SetTrigSoftwareVeto(BoardNumber);`); GRETINA_RTRIG/DGS_RTRIG/MYRIAD → no-op with TODO comment about TAC-II master trigger disable ✅ verified 2026-04-23 — `inLoop.st:L827–851` |
| `DISABLE_COLLECTION` | always | Go to `DRAIN_REMAINING_DATA` |
| `DRAIN_REMAINING_DATA` | Entry (always) | Loop all boards: drain FIFO until empty (do-while loop on `CheckAndRead*()`); call `SendEndOfRun()` per board; set `InloopIsRunning = 0` |
| `DRAIN_REMAINING_DATA` | `!InloopIsRunning` | Go to `INIT` (run complete) |

**Board type handling (switch on `daqBoards[BoardNumber].board_type`):**
- `BrdType_ANL_MDIG` / `BrdType_ANL_SDIG` / `BrdType_MAJORANA_MDIG` / `BrdType_MAJORANA_SDIG` — full DIG readout path
- `BrdType_DGS_MTRIG` — MTRG readout via `CheckAndReadTrigger()` with `FIFO_index[]` selection
- `BrdType_DGS_RTRIG` / `BrdType_GRETINA_RTRIG` / `BrdType_GRETINA_MTRIG` / `BrdType_MYRIAD` — no readout (break/no-op)
- `BrdType_LBNL_DIG` — defined but not implemented (break)

**Key PV declarations (macros):**

| Macro | PV | Variable | Purpose |
|-------|----|----------|---------|
| `DECLMON` | `Online_CS_StartStop` | `AcqRun` | Master run/stop (from softIOC) |
| `DECL` | `DAQC{CRATE}:inLoop_Running` | `InloopIsRunning` | Handshake flag to outLoop |
| `DECLMON` | `VME{CRATE}:{B0–B6}:CS_Ena` | `B0–B6En` | Per-board software enable |
| `DECLMON` | `VME{CRATE}:{B0–B6}:FifoNum` | `B0–B6FifoNum` | Per-board FIFO index (triggers only) |
| `DECLMON` | `DAQC{CRATE}_CV_CRATENUM` | `CRATENUM` | Crate number |
| `DECL` | `DAQC{CRATE}_BoardType0–6` | `BoardType0–6` | Board type codes pushed out to GUI |

**FIFO index cheat sheet (for trigger modules):**

| Address | Name | Index |
|---------|------|-------|
| 0x0160 | MON FIFO 1 | 0 |
| 0x0164 | MON FIFO 2 | 1 |
| … | … | … |
| 0x0178 | MON FIFO 7 | 6 (usual) |
| 0x0180–0x019C | CHAN FIFO 1–8 | 8–15 |

#### Key C Support Functions (`inLoopSupport.c`)

| Function | Purpose |
|----------|---------|
| `SetupBoardAddresses(CRATENUM, MaxBoardNum, B0–B6_SW_en)` | Maps board enables to `daqBoards[]`; sets VME register pointers (`MstrLogicReg`, `FIFOStatusReg`, `RawDataLengthReg`, `PulsedControlReg`, FIFO base addresses); returns `NumBoardsEnabled` |
| `ClearDigMstrLogicEnable(BoardNumber)` | Writes 0 to `MstrLogicReg[BoardNumber]` — stops digitizer event acceptance |
| `SetDigMstrLogicEnable(BoardNumber)` | Writes enable bit to `MstrLogicReg[BoardNumber]` — starts digitizer event acceptance |
| `ClearDigFIFO(BoardNumber)` | Pulsed write to `PulsedControlReg` to reset DIG FIFO |
| `InitializeDigPipeline(BoardNumber)` | Calls `ClearDigMstrLogicEnable()` + `ClearDigFIFO()` |
| `CalcDigMaxEventsPerRead(BoardNumber)` | Reads `RawDataLengthReg[][]` for all channels; calculates `MinimumCalcEventSize`, `MaximumCalcEventSize`, `MaxEventsToRead[]`, sets buffer sizing |
| `EnableModule(BoardNumber)` | Dispatcher: DIG → `SetDigMstrLogicEnable()`; MTRG/RTRG → no-op (already enabled) |
| `CheckAndReadDigitizer(BoardNumber, SendNextEmpty, globQueueUsageFlag)` | Main DIG readout: checks FIFO depth, reads up to `MaxEventsToRead` events via DMA, prepends/appends type-F headers if empty, stuffs buffer to `QWritten`; returns bytes read (≥0) or error code (<0) |
| `ClearTrigFIFO(BoardNumber, FIFO_index)` | Writes FIFO reset register for MTRG |
| `CheckAndReadTrigger(BoardNumber, FifoNum, SendNextEmpty, globQueueUsageFlag)` | Main MTRG readout: checks MON FIFO depth, reads available data, stuffs to `QWritten`; returns bytes read or error |
| `SendEndOfRun(BoardNumber, globQueueUsageFlag)` | Called during drain: sends end-of-run marker buffer to `QWritten` |
| `UpdateScanDelay()` | Returns adaptive scan delay based on `getUsedBufCount()` — increases delay when buffers are congested, decreases when free |
| `DumpInLoopArrays()` | Debug dump of all per-board arrays (FIFO depths, event sizes, etc.) |

**VME register constants (from `inLoopSupport.c`):**

| Constant | Offset | Board | Description |
|----------|--------|-------|-------------|
| `DIG_MSTR_LOGIC_REG` | 0x500 | DIG | Master Logic Status |
| `DIG_PROGRAMMING_DONE_REG` | 0x004 | DIG | FIFO depth/status (Programming Done) |
| `DIG_RAW_DATA_WINDOW_REG[0–9]` | 0x140–0x164 | DIG | Per-channel raw data window size |
| `DIG_PULSED_CTRL_REG` | 0x40C | DIG | Pulsed control (FIFO reset) |
| `DIG_FIFO` | 0x1000 | DIG | DIG FIFO base |
| `MTRG_MON_FIFO_STATE_REG` | 0x1B4 | MTRG | MON7 FIFO state |
| `MTRG_CHAN_FIFO_STATE_REG` | 0x1A4 | MTRG | CHAN FIFO state |
| `MTRG_FIFO_RESET_REG` | 0x8F0 | MTRG | FIFO reset pulsed register |
| `MTRG_FIFO` | 0x5000 | MTRG | MTRG MON FIFO base |
| `MTRG_MON7_LATCHED_DEPTH` | 0x1AC | MTRG | MON7 latched depth (event-boundary counter) |
| `MTRG_MON7_LIVE_DEPTH` | 0x154 | MTRG | MON7 live depth counter |

**Error return codes from `CheckAndReadDigitizer()`:**
- `0` — board empty (no data), type-F header emitted if `SendNextEmpty` flag set
- `> 0` — bytes successfully read
- `-5` — board not empty but less than one full event available (ignorable during drain)
- `< 0` (other) — FIFO error; board is reset and re-enabled

**Cross-references:** [`vxworks_fifo_readout.md`](vxworks_fifo_readout.md) (DMA buffer architecture, FIFO index map, type-F header details); `QueueManagement.c` section below; [`vxworks.md`](vxworks.md) § outLoop.st (next stage); [`VME_registers.md`](VME_registers.md) (full register map).

---

### outLoop.st — Data Validation and Buffer Routing State Machine

_Source: `dgsDrivers/dgsDriverApp/src/outLoop.st` (490 lines, EPICS State Notation Language)_ ✅ verified 2026-04-22 — full read

**Purpose:** The middle layer of the three-machine pipeline (inLoop → outLoop → MiniSender). Takes data buffers filled by `inLoop.st` from the **written queue** (`QWritten`), validates them (event headers, timestamps, consistency), then moves them to the **send queue** (`QSend`) for `MiniSender.st` to transmit to `tcpReceiverMT`.

**Two concurrent state sets:**
1. `ss outLoop` — main control flow (run/stop, buffer movement)
2. `ss outLoopTraceMon` — PV refresh timer (runs every 0.5 s in parallel) ✅ verified 2026-04-22 — `outLoop.st:L374` (`when (delay(0.5))`)

#### Main State Machine (`ss outLoop`)

| State | Trigger | Action |
|-------|---------|--------|
| `INIT` | Entry | Set task priority 190; print idle message; `msgFilter=0`, `pvRefreshReady=0` |
| `INIT` | `AcqRun == 1` | Call `ResetStats()` → go to `CHECK_FOR_DATA` |
| `INIT` | `delay(0.5)` | Service trace/PV refresh if flagged; stay in `INIT` |
| `CHECK_FOR_DATA` | `!AcqRun && !Running` | Go to `CHECK_FOR_EMPTY_WRITTEN_Q` (drain remaining buffers) |
| `CHECK_FOR_DATA` | `written_bufs > 0` | Go to `PROCESS_DATA` |
| `CHECK_FOR_DATA` | `delay(0.05)` | Poll `getWrittenBufCount()` / `getSenderBufCount()`; log AcqRun/Running mismatch (rate-limited); stay in `CHECK_FOR_DATA` | ✅ verified 2026-04-22 — `outLoop.st:L330` |
| `PROCESS_DATA` | Entry | Call `CheckAndMoveBuffers(written_bufs, send_bufs, sendEnable)`; accumulate `send_bufs += written_bufs`; reset `written_bufs=0`; go to `CHECK_FOR_DATA` |
| `CHECK_FOR_EMPTY_WRITTEN_Q` | Entry | Update buffer counts; log flush message |
| `CHECK_FOR_EMPTY_WRITTEN_Q` | `written_bufs > 0` | Go to `PROCESS_DATA` (drain last buffers) |
| `CHECK_FOR_EMPTY_WRITTEN_Q` | else | Go to `INIT` (run done) |

**Key monitored PVs:**

| PV | Variable | Function |
|----|----------|----------|
| `Online_CS_SaveData` | `sendEnable` | 1 = save data; 0 = discard (no-save run); passed to `CheckAndMoveBuffers()` |
| `Online_CS_StartStop` | `AcqRun` | Master run/stop from GUI — triggers INIT→run transition |
| `DAQC{CRATE}:inLoop_Running` | `Running` | inLoop handshake — 1 when inLoop is actively running |
| `DAQC{CRATE}_CS_TraceBd/TraceChan/TraceHorns` | `traceBoard/Chan/Horns` | Selects which board/channel to capture waveform trace |
| `DAQC{CRATE}_OL_HeaderCheckEnable` | `outLoopHeaderCheckEnable` | Enable event header validation |
| `DAQC{CRATE}_OL_TimestampCheckEnable` | `outLoopTimestampCheckEnable` | Enable timestamp consistency check |
| `DAQC{CRATE}_OL_DeepCheckEnable` | `outLoopDeepCheckEnable` | Enable deeper data integrity checks |
| `DAQC{CRATE}_OL_HeaderSummaryEnable` | `outLoopHeaderSummaryEnable` | Enable periodic header dumps to console |
| `DAQC{CRATE}_OL_HeaderSummaryPrescale` | `outLoopHeaderSummaryPrescale` | Prescale for header dumps (default 0x1000) | ✅ verified 2026-04-23 — `outLoop.st:L119` comment |

**Check control flow (via `outLoopTraceMon`):** Every 0.5 s, the monitor state set copies check-control PVs to global C variables (`OL_Hdr_Chk_En`, `OL_TS_Chk_En`, `OL_Deep_Chk_En`, `OL_Hdr_Summ_En`, etc.) readable from `outLoopSupport.c`.

**Reported PVs (updated every 0.5 s by `outLoopTraceMon`):**

| PV | Description |
|----|-------------|
| `DAQC{CRATE}_CV_OutLoop0–6` | Per-board error counts (repurposed; was DataLost in KB) | ✅ verified 2026-04-23 — `outLoop.st:L196-202` comment "Repurposed for error count reporting"; populated by `GetErrorCount()` |
| `DAQC{CRATE}_OL_DataRate0–6` | Per-board read rate in Bytes/s ⚠️ currently always 0 — `UpdateDataRates()` is commented out in outLoopTraceMon; rates not actively updated | ✅ verified 2026-04-23 — `outLoop.st:L215` comment "Board read rates in Bytes/s"; `GetDataRate()` comment "return Bytes/s"; `DataRate_Board*_Raw = 0` always set (L216-222) |
| `DAQC{CRATE}_OL_Data0–6` | Per-board cumulative data in KB | ✅ verified 2026-04-23 — `outLoop.st:L224` comment "Board total data in KB"; `GetDataTotal()` comment "return KB" (`outLoopSupport.c:L792`) |
| `DAQC{CRATE}_OL_NumFreeBuffers` | Current count of free buffers in pool |
| `DAQC{CRATE}_OL_NumWrittenBuffers` | Current count of written buffers (ready for validation) |
| `DAQC{CRATE}_OL_NumSendBuffers` | Current count of send buffers (ready for MiniSender) |
| `DAQC{CRATE}_OL_TotalBufsWritten` | Total buffers written since run start |
| `DAQC{CRATE}_OL_TotalFBufsWritten` | Total "flush" buffers written |
| `DAQC{CRATE}_OL_TotalBufsLost` | Total buffers lost/dropped |
| `DAQC{CRATE}_OL_BufLostPerecnt` | Lost-buffer percentage (note: original typo in PV name) | ✅ verified 2026-04-23 — `outLoop.st:L103` (`LostBuffer_Percent, DAQC{CRATE}_OL_BufLostPerecnt`) |
| `DAQC{CRATE}_CV_SendRate` | Send data rate in KB/s |
| `DAQC{CRATE}_CV_TraceLen` / `DAQC{CRATE}_CV_Trace` | Waveform trace length + data (1024-sample array) |
| `DAQC{CRATE}_CV_BuffersAvail` / `DAQC{CRATE}_CV_NumSendBuffers` | Legacy compatibility copies of buffer counts |

**Key C support functions (from `outLoopSupport.c`):** ✅ verified 2026-04-22 — all function signatures confirmed in `outLoopSupport.c` (L82, L171, L209, L729, L786, L791, L796, L801, L817)

| Function | Description |
|----------|-------------|
| `ResetStats()` | Zero all stats at run start |
| `CheckAndMoveBuffers(written, send, enable)` | Core function: validate buffers from `QWritten`, move valid ones to `QSend` (if `enable=1`) or discard |
| `UpdateDataRates()` | Recalculate per-board throughput rates |
| `GetTrace(buf, board, ch)` | Copy waveform trace for board/channel into caller buffer; returns length |
| `GetDataRate(board)` | Per-board throughput in Bytes/s |
| `GetDataTotal(board)` | Cumulative per-board data in KB |
| `GetErrorCount(board)` | Per-board error event count |
| `GetErrorData(board, idx)` | Per-board raw error diagnostic data (idx 0–6) |
| `GetTotalBuffers_Written()` | Total buffer write count |
| `GetTotalBuffers_Lost()` | Total buffer loss count |
| `GetTotalFBuffers_Written()` | Total flush buffer count |
| `GetSendDataRate()` | Aggregate send rate in Bytes/s |

#### `CheckAndMoveBuffers()` — Inner Validation Logic
_Source: `outLoopSupport.c` — code-read 2026-04-23_

The core data-validation loop. Called from `outLoop.st` state `PROCESS_DATA` with `written_bufs` (from `getWrittenBufCount()`), `send_bufs` (from `getSenderBufCount()`), and `sendEnable` (0 = no-save run, 1 = save data).

**Send queue overflow guard:** If `(send_bufs + written_bufs) > SENDER_BUF_BYPASS_THRESHOLD`, sets `emergency_data_dump=1` and overrides `sendEnable=0`. All buffers in this call are counted as lost and returned to `qFree`. ✅ verified 2026-04-23 — `outLoopSupport.c:L245-260`

**Per-buffer processing loop** (iterates `written_bufs` times):
1. Pulls next buffer from `qWritten` via `getWrittenBuf()`.
2. Validates `board_num` range (0 ≤ board < `GVME_MAX_CARDS`); fatal on out-of-range → `goto MOVE_BUFFER_AFTER_CHECK`. (No `ErrorData` code set — `board_num` is corrupt, unsafe to index.) ✅ verified 2026-04-23 — `outLoopSupport.c:L303-311`
3. Validates `rawBuf->len != 0`; fatal on zero-length → sets `ErrorData[brd][0] = 1` → `goto MOVE_BUFFER_AFTER_CHECK`. ✅ verified 2026-04-23 — `outLoopSupport.c:L315-332`
4. Dispatches on `rawBuf->board_type`:
   - **Trigger boards** (MTRG, RTRIG) and MYRIAD: pass through with no validation.
   - **Digitizer boards** (ANL MDIG/SDIG, Majorana, LBNL): full per-event inner loop (see below).

**Per-event inner loop** (for digitizer buffers, `while offset < BufLengthInLongwords`):

| Check | Condition | Error code | Action |
|-------|-----------|-----------|--------|
| 0xAAAAAAAA alignment | `*dptr != 0xAAAAAAAA` | `ErrorData[brd][0] = 2` | If `OUTLOOP_TRY_REALIGNMENT` defined: scan forward to next `0xAAAAAAAA`; else fatal |
| Header cutoff | `(offset + MIN_HEADER_LENGTH) > BufLengthInLongwords` | `= 3` | Fatal |
| Event cutoff | `(offset + packet_length + 1) > BufLengthInLongwords` | `= 4` | Fatal |
| Packet length | `packet_length > MAX_PACKET_LENGTH` | `= 5` | Fatal |
| Type F channel ID | `header_type==0xF` and `ch_id ∉ {0xD, 0xE, 0xF}` | `= 6` | Fatal |
| Normal channel ID | `header_type!=0xF` and `ch_id > 9` | `= 7` | Fatal (break) |
| Timestamp monotonicity | `last_timestamp[brd][ch_id] > timestamp_check` | `= 8` | Non-fatal: log + break inner loop (buffer still forwarded) |
| Queue put failure | `putSenderBuf()` returns error | `= 9` | Log only |

Timestamp check uses bits 47:16 of the 48-bit hardware timestamp: `timestamp_check = (timestamp_upper << 16) | (timestamp_lower >> 16)`. A non-monotonic timestamp increments `TotalErrors[board_num]` but does **not** set `fatal_buffer_error` — the buffer is still forwarded. ✅ verified 2026-04-23 — `outLoopSupport.c:L569-592`

**Header field decode** (from `raw_header[]` at 0xAAAAAAAA offset):
- Word 0: `0xAAAAAAAA`
- Word 1: `ch_id` [3:0], `user_def` [15:4], `packet_length` [26:16], `geo_addr` [31:27]
- Word 2: `timestamp_lower` [31:0]
- Word 3: `timestamp_upper` [15:0], `header_type` [19:16], `event_type` [25:23], `header_length` [31:26]

✅ verified 2026-04-23 — `outLoopSupport.c:L456-465` (field decode comments + bit-extract code)

**Buffer routing at end of loop:**
- `fatal_buffer_error == 0` AND `sendEnable > 0` → `putSenderBuf()` (forward to MiniSender)
- Otherwise → `putFreeBuf()` (discard / lost)

Optional compile-time features: `TRACE_ENABLE` (captures one waveform per channel per buffer into `ChannelTrace[][]`), `HISTO_ENABLE` (builds noise histograms from raw ADC differences), `OUTLOOP_TRY_REALIGNMENT` (attempts resync on misaligned headers). ✅ verified 2026-04-23 — `outLoopSupport.c:L595-640`

---

### MiniSender.st — Network Send State Machine

_Source: `dgsDrivers/dgsDriverApp/src/MiniSender.st` (231 lines, EPICS State Notation Language)_ ✅ verified 2026-04-22 — full read

**Purpose:** The final stage of the pipeline — takes validated send buffers from `QSend` and transmits them over TCP to `tcpReceiverMT` running on the DAQ host. One instance per VME crate. Uses `SendReceiveSupport.c` for all socket I/O.

**Single state set:** `ss ReceiveRequest` (runs as task priority 190)

| State | Trigger | Action |
|-------|---------|--------|
| `init` | Entry | Set priority 190; `SenderRunning=0` |
| `init` | `RunStopButton==1 && Save_NoSave_Button==1` | Call `InitRequestSocket()` (opens TCP server socket); `RequestMsgStatus=1`, `ConnectionAccepted=0`; go to `DelayAfterStart` |
| `init` | `delay(1)` | Call `FlushAllBuffers()`; stay in `init` |
| `DelayAfterStart` | `delay(2)` | Wait 2 s after run start (let inLoop/outLoop initialize); go to `WaitForConnection` |
| `WaitForConnection` | Entry (every time, `-e`) | Call `AcceptConnection()` — blocks until `tcpReceiverMT` connects |
| `WaitForConnection` | `!RunStopButton or !Save_NoSave_Button` | Go to `cleanup` |
| `WaitForConnection` | `ConnectionAccepted > 0` | Call `FlushAllBuffers()` (start clean); `SenderRunning=1`; go to `HandleRequests` |
| `WaitForConnection` | `delay(0.05)` | Retry `AcceptConnection()` |
| `HandleRequests` | Entry (every time, `-e`) | Profile counter #4 start; call `getReceiverRequest()` (polls socket for request from receiver); profile stop |
| `HandleRequests` | `RequestMsgStatus==0` (message received) | Go to `ProcessRequest` |
| `HandleRequests` | `RequestMsgStatus==1` (no message yet) | Stay in `HandleRequests` |
| `HandleRequests` | `!RunStopButton` | Go to `cleanup` |
| `ProcessRequest` | Entry | Profile counter #5 start; call `sendServerResponse()` — responds to receiver with data-available count; profile stop; `NumBufsAvailable` = result |
| `ProcessRequest` | `NumBufsAvailable==0` | No data ready; go to `HandleRequests` |
| `ProcessRequest` | `NumBufsAvailable>0` | Profile counter #6 start; call `sendDataBuffer()` — send one buffer; profile stop; go to `HandleRequests` |
| `cleanup` | Entry | Log; drain `QSend` via `FlushAllBuffers()`; call `CloseAllSockets()`; go to `init` |

**Key C functions (from `SendReceiveSupport.c`):**

| Function | Description |
|----------|-------------|
| `InitRequestSocket()` | Open TCP server socket, bind, listen — waits for `tcpReceiverMT` to connect |
| `AcceptConnection()` | Accept incoming TCP connection from `tcpReceiverMT`; returns >0 on success |
| `getReceiverRequest()` | Poll socket for a request message from receiver; returns 0=message, 1=no message, negative=error |
| `sendServerResponse()` | Send response header to receiver indicating how many buffers are available; fetches first buffer from `QSend` if available; returns buffer count |
| `sendDataBuffer()` | Send one data buffer (already fetched by `sendServerResponse`) to receiver over TCP |
| `FlushAllBuffers()` | Discard all pending send buffers (called at cleanup or no-save mode) |
| `CloseAllSockets()` | Close TCP socket(s) after run ends |

**Profile counters used (from `profile.h`):**
- Counter 4: `PROF_MS_GET_RECEIVER_REQUEST` — time spent in `getReceiverRequest()`
- Counter 5: `PROF_MS_SEND_SERVER_RESPONCE` — time spent in `sendServerResponse()`
- Counter 6: `PROF_MS_SEND_DATA_BUFFER` — time spent in `sendDataBuffer()`

**Run/no-save behavior:**
- `Save_NoSave_Button == 0` (no-save): MiniSender stays in `init`; `FlushAllBuffers()` drains without TCP send. Data is counted by outLoop but never transmitted.
- `Save_NoSave_Button == 1` (save): normal TCP send path activates.

#### `SendReceiveSupport.c` — TCP Protocol Details

_Source: `dgsDrivers/dgsDriverApp/src/SendReceiveSupport.c` (507 lines). ✅ verified 2026-04-27 — full read_

**Socket configuration (`setsocketoption()`, added MBO 20200617):**
- `SO_RCVBUF` = 65536 bytes ✅ verified 2026-04-27 — `SendReceiveSupport.c:L76`
- `SO_SNDBUF` = 65536 bytes ✅ verified 2026-04-27 — `SendReceiveSupport.c:L77`
- `TCP_NODELAY` = 1 (disable Nagle algorithm — minimize latency) ✅ verified 2026-04-27 — `SendReceiveSupport.c:L86`
- `TCP_MAXSEG` (MSS) — intentionally commented out; VxWorks docs say you can only reduce MSS pre-connect (default 512), and the negotiated MTU will be used at connect time anyway ✅ verified 2026-04-27 — `SendReceiveSupport.c:L95-103` (`#if 0` block)
- Server socket (`SocketForRequests`) is set **non-blocking** via `ioctl(FIONBIO)` so `MiniSender.st` can still react to run stop without blocking on `AcceptConnection()` ✅ verified 2026-04-27 — `SendReceiveSupport.c:L143`

**TCP port:** `SERVER_PORT = 9001` (hard-coded `#define`; matches `tcpReceiverMT`) ✅ verified 2026-04-27 — `SendReceiveSupport.c:L120`

**`InitRequestSocket()` flow:**
1. If socket already open, close it first (handles re-init after error) ✅ verified 2026-04-27 — `SendReceiveSupport.c:L129`
2. `socket(AF_INET, SOCK_STREAM, 0)` — stream TCP socket ✅ verified 2026-04-27 — `SendReceiveSupport.c:L134`
3. Set non-blocking via `ioctl(FIONBIO)` ✅ verified 2026-04-27 — `SendReceiveSupport.c:L143`
4. `gethostname()` + `hostGetByName()` — resolves the IOC's **own IP** (not INADDR_ANY) ✅ verified 2026-04-27 — `SendReceiveSupport.c:L149–164`
5. `setsocketoption()` — set buffer sizes + TCP_NODELAY ✅ verified 2026-04-27 — `SendReceiveSupport.c:L181`
6. `bind()` to this IOC's own resolved IP on port 9001 (not `0.0.0.0` — corrected 2026-04-27) ✅ verified 2026-04-27 — `SendReceiveSupport.c:L184`
7. `listen()` with backlog = 10 ✅ verified 2026-04-27 — `SendReceiveSupport.c:L189`
8. Return codes: 0=success, -1=socket fail, -2=hostname fail, -3=IP lookup fail, -4=bind fail, -5=listen fail ✅ verified 2026-04-27 — `SendReceiveSupport.c:L136,L155,L170,L187,L193`

**`AcceptConnection()` behavior:**
- Calls `accept()` on `SocketForRequests`
- Non-blocking: `EWOULDBLOCK` is expected during polling → returns 0 (retry)
- On `ERROR` (any other error): calls `InitRequestSocket()` to reset, returns -1
- On success: `ReadWriteSocket` global is set to accepted socket fd, returns 1
- Note: JTA 20230919 comment suggests it would be useful to close the socket before re-init, which `InitRequestSocket()` now does

**Wire protocol — Request (Receiver → IOC):**
```c
typedef union {
    int type;           // single int: CLIENT_REQUEST_EVENTS=1 expected
    char RawMsg[4];     // 4 bytes over the wire
} ReqMsg;
```
- Request message is just **4 bytes** (a single int in network byte order)
- `getReceiverRequest()` uses `recv()` in a loop to ensure all 4 bytes are received
- Returns: 0=message received, 1=EWOULDBLOCK (no message yet — normal), -2=socket not valid, -3=recv error, -4=partial read failed, -5=buffer overrun

**Wire protocol — Response (IOC → Receiver):**
```c
typedef struct {
    int type;    // SERVER_SUMMARY=4 (data follows) or INSUFF_DATA=5 (no data)
    int recLen;  // byte length of the data buffer to follow (0 if INSUFF_DATA)
    int status;  // always 0 (unused by gtReceiver4 per Torben 20200609)
    int recs;    // 1 if data follows, 0 if no data
} evtServerRetStruct;  // 16 bytes total
typedef union {
    evtServerRetStruct Fields;
    char RawMsg[16];    // 16 bytes over the wire
} ResponseMsg;
```
- All fields sent via `htonl()` (network byte order) ✅ verified 2026-04-27 — `SendReceiveSupport.c:L375,L380,L382,L383`
- `sendServerResponse()` uses a partial-send loop for the 16-byte header; also handles `EWOULDBLOCK`/`EAGAIN`/`ENOBUFS` with `taskDelay(1)` retry ✅ verified 2026-04-27 — `SendReceiveSupport.c:L418-432`
- If `BufsAvailable > 0`: pops one buffer from `QSend` (via `getSenderBuf()`), sets `type=SERVER_SUMMARY`, `recLen=buf->len`, `recs=1` ✅ verified 2026-04-27 — `SendReceiveSupport.c:L389-410`
- If `BufsAvailable == 0`: sets `type=INSUFF_DATA`, `recLen=0`, `recs=0` (no data buffer sent) ✅ verified 2026-04-27 — `SendReceiveSupport.c:L371-385`
- Returns `BufsAvailable` to the state machine (determines whether `sendDataBuffer()` is called) ✅ verified 2026-04-27 — `SendReceiveSupport.c:L445`
- On send error: puts buffer back to `QFree` (avoids leak), returns 0 ✅ verified 2026-04-27 — `SendReceiveSupport.c:L424-428`

**Wire protocol — Data payload (IOC → Receiver):**
- `sendDataBuffer()` sends `WorkingDescriptor->len` bytes from `WorkingDescriptor->data` ✅ verified 2026-04-27 — `SendReceiveSupport.c:L453-482`
- Uses partial-send loop; `EWOULDBLOCK`/`EAGAIN`/`ENOBUFS` → `taskDelay(1)` and retry; any other error → `putFreeBuf()` + return ✅ verified 2026-04-27 — `SendReceiveSupport.c:L466-476`
- After successful send: `putFreeBuf(WorkingDescriptor)` returns buffer to `QFree` ✅ verified 2026-04-27 — `SendReceiveSupport.c:L482`
- `WorkingDescriptor` is a file-static global `rawEvt *` (set during `sendServerResponse()`) ✅ verified 2026-04-27 — `SendReceiveSupport.c:L62,L389`

**Protocol message type codes (`DGS_DEFS.h`):**
| Code | Value | Direction | Meaning |
|------|-------|-----------|----------|
| `CLIENT_REQUEST_EVENTS` | 1 | Receiver→IOC | Receiver asking for data |
| `SERVER_NORMAL_RETURN` | 2 | IOC→Receiver | (legacy, not used by current code) |
| `SERVER_SENDER_OFF` | 3 | IOC→Receiver | (legacy, not used by current code) |
| `SERVER_SUMMARY` | 4 | IOC→Receiver | Data buffer follows |
| `INSUFF_DATA` | 5 | IOC→Receiver | No data available right now |

✅ verified 2026-04-27 — `DGS_DEFS.h:L184-188`

**Global sockets:** `SocketForRequests` (listen socket), `ReadWriteSocket` (accepted connection socket) — both file-static with `sender_debug_level`-gated diagnostic prints. `sender_debug_level` is a VxWorks shell-accessible `extern int` for live debug.

**Cross-reference:** [`ANLDAQ_tcpReceiver.md`](ANLDAQ_tcpReceiver.md) — `tcpReceiverMT` protocol details; how it connects, sends requests, and receives data from MiniSender.

---

## Port 9010 On-Demand FIFO Grabber (Planned — Not Implemented)

> ⚠️ Design plan only as of 2026-04-18. `fifoGrabber.c` does not exist. No VxWorks build changes made.

- **Purpose:** Standalone TCP diagnostic service on port 9010 — grab raw FIFO data from a specific digitizer board without starting the full DAQ pipeline (no MiniSender, no receiver required).
- **Plan document:** `vxworks/On-Demand-FIFO-Grabber-Plan.md` — full wire protocol, implementation details, Python client.
- **Requires:** New `dgsDrivers/dgsDriverApp/src/fifoGrabber.c`, new task (`FifoGrabberTask`, priority 150), `FifoGrabberInit(9010)` call from VxWorks shell post-`iocInit`.

---

## Trigger Board Drivers (`asynTrigCommonDriver`, `asynTrigRouterDriver`, `asynTrigMasterDriver`)

_Source: `dgsDrivers/dgsDriverApp/src/asynTrigCommonDriver.{h,cpp}`, `asynTrigRouterDriver.{h,cpp}`, `asynTrigMasterDriver.{h,cpp}`, `asynRTrigParams.h`, `asynMTrigParams.h`. Code-read 2026-04-23._

The DGS VxWorks IOC uses a three-layer class hierarchy for RTRG and MTRG trigger board control, mirroring the asyn digitizer driver structure.

### Class Hierarchy

```
asynPortDriver          (asyn base class)
  └── asynTrigCommonDriver   (shared register poll loop + VME R/W helpers)
        ├── asynTrigRouterDriver  (RTRG: 188 params, devAsynTrigRouterCardInit)
        └── asynTrigMasterDriver  (MTRG: 369 params, devAsynTrigMasterCardInit)
```

### `asynTrigCommonDriver` — Base Class (`asynTrigCommonDriver.cpp`, 412 lines)

**Purpose:** Shared foundation for both RTRG and MTRG drivers. Provides:

- **Background register poll loop** (`simTask`, spawned at construction): every 1 s, acquires `vme_driver_mutex`, iterates over `address_list[]` (a mapping of asyn param → VME register offset), reads each via `viIn32()`, and calls `setUIntDigitalParam()` / `callParamCallbacks()` to push fresh values to EPICS records. ✅ verified 2026-04-23 — `asynTrigCommonDriver.cpp:L78-110`
- **VME R/W helpers:** `viIn32(slot, adr_space, reg_adr, &data)` and `viOut32(slot, adr_space, reg_adr, data)` — thin wrappers around `devGVME` bus access with slot/address-space abstraction.
- **Mask-based read decoding (`readUInt32Digital`):** Supports a special `0xAAAA_nn_ss` mask encoding for extracting bit-fields from 32-bit registers — `nn` = number of bits, `ss` = shift amount; used by EPICS `longin` records to read sub-fields. ✅ verified 2026-04-23 — `asynTrigCommonDriver.cpp:L161-183`
- **Address map:** `address_list[1024]` (param_num → VME register address pairs); subclasses populate this via `setAddress()`. `findAddress()` searches for a param's VME offset at read/write time.
- **Constructor:** Creates `run_counter` param (incremented by `simTask`), spawns `asynTrigCommonDriver_Task` thread at medium priority.
- **Param capacity:** 1024 params (increased from 256 on 2015-10-29 by MPC/TM).

### `asynTrigRouterDriver` — RTRG Driver (`asynTrigRouterDriver.cpp`, 274 lines)

**Init function:** `devAsynTrigRouterCardInit(cardno, slot)` — called from VxWorks IOC startup script.
- Calls `initVmeDrvMutex()`, then `devGVMECardInit(cardno, slot)` to register the VME base address.
- Reads CODE_REVISION register (offset 0x15C/4) to detect firmware type (`ftype = boardid & 0xf00 >> 8`). ✅ verified 2026-04-23 — `asynTrigRouterDriver.cpp:L97-100`
- **Firmware type table** (same as used in MTRG; extracted from `CODE_REVISION[11:8]`):
  - 0=proto, 1=GRETINA Router, 2=GRETINA Master, 3=GRETINA Data Gen
  - 4=DGS Master, **5=DSSD Master**, **6=DGS Router** ✓ (accepted), 7=DSSD Router
  - 8=DGS Data Gen, 9=DSSD Data Gen, A=Digitizer Tester, B=MγRIAD, C=DGS Digitizer, D=DSSD Digitizer, F=VME FPGA
- Only type 6 (DGS Router) is accepted; all others log an error and return `ERROR`. Sets `daqBoards[cardno].router=1` and `mainOK=1`.
- Constructs an `asynTrigRouterDriver` object (an asyn port driver named by slot) and registers it.

**Params:** `asynRTrigParams.h` defines **188 int parameters** — all RTRG VME register fields: `reg_LOCK_BUS`, `reg_SYNC_BUS`, `reg_TIMESTAMP_{A,B,C}`, `reg_DiagnosticA–H`, `reg_THROTTLE_STATUS`, `reg_CODE_DATE/REVISION`, `reg_MON1-8_FIFO`, `reg_CHAN1-8_FIFO`, `reg_MON/CHAN_FIFO_STATE`, `reg_LOCK_COUNTER_{A–H}`, `reg_INPUT_LINK_MASK` (+ per-bit `ILM_{A–H}`), `reg_LED_REG`, `reg_SKEW_CTL_{A,B,C}`, throttle, trigger algorithm controls, etc. ✅ verified 2026-04-23 — `asynRTrigParams.h:L1-193`

### `asynTrigMasterDriver` — MTRG Driver (`asynTrigMasterDriver.cpp`, 264 lines)

**Init function:** `devAsynTrigMasterCardInit(cardno, slot)` — same VME init pattern as RTRG.
- Reads CODE_REVISION; only type 4 (DGS Master Trigger) accepted. Sets `router=0`, `mainOK=1`.
- Constructs `asynTrigMasterDriver` object and registers it. Global `maddog_t` pointer set for diagnostic access.

**Params:** `asynMTrigParams.h` defines **369 int parameters** — comprehensive MTRG register coverage: `reg_MSTR_MACH_STATE`, `reg_AUX_INPUT_STATE`, `reg_LINK_LRU_MACH_STAT`, `reg_MISC_STAT/MISC_STAT2`, `reg_SYSTEM_THROTTLE_MAP`, `reg_FRAME_12/14/16/17_CMD_CNT`, `reg_STARTING_TIMESTAMP_{HI,MID,LOW}`, `reg_FRAME_17_DATA_1–5`, `reg_ENCODER_SOURCE_SELECT`, `reg_MYRIAD_TRIG_DELAY/OVERLAP_CTL`, `reg_TDC_TRIG_SEL`, `reg_TRIG_ALGO_MUX_SEL`, all trigger algorithm controls (MYRIAD, local coincidence, prescale, holdoff, thresholds), TAC-II controls, NIM I/O controls, etc. ✅ verified 2026-04-23 — `asynMTrigParams.h:L1-375`

---

## `vmeDriverMutex` — Cross-Driver VME Bus Lock

_Source: `dgsDrivers/dgsDriverApp/src/vmeDriverMutex.{h,c}`. Code-read 2026-04-23._

A **shared `epicsMutex`** that serializes VME bus access across **all** asyn driver instances (digitizer, RTRG, MTRG). Its primary purpose is to allow safe flash programming: `FlashMaintenance.c` acquires this mutex before programming flash, blocking all concurrent VME reads by background poll threads.

**Key details:**
- `vme_driver_mutex` — a single global `epicsMutexId`, initialized lazily via `initVmeDrvMutex()` (checks `is_vme_mutex_exist` flag to prevent double-init). ✅ verified 2026-04-23 — `vmeDriverMutex.c:L52-57`
- `vmeMutexLock(caller_id)` / `vmeMutexUnLock(caller_id)` — acquire/release with optional **per-caller bypass**: if `disable_mutex_id == caller_id`, the lock is skipped. Allows developers to disable the mutex for specific subsystems during debugging without recompiling. ✅ verified 2026-04-23 — `vmeDriverMutex.c:L60-68`
- Called at the top of each driver's `simTask()` poll loop and around every `viOut32()` (write) operation.
- `initVmeDrvMutex()` called by both `devAsynTrigRouterCardInit` and `devAsynTrigMasterCardInit` at driver init time.

---

## `QueueManagement.c` — Three-Queue Buffer Pool

_Source: `dgsDrivers/dgsDriverApp/src/QueueManagement.{h,c}` (495 lines). Code-read 2026-04-23._

Manages the three VxWorks message queues (`qFree`, `qWritten`, `qSender`) that connect the `inLoop` → `outLoop` → `MiniSender` pipeline. Also owns all DMA buffer allocations.

**Three queues (VxWorks `MSG_Q_FIFO`):**
| Queue | Handle | Direction | Description |
|-------|--------|-----------|-------------|
| `qFree` | `qFree` | Pool | Pre-allocated empty `rawEvt` buffers available for `inLoop` |
| `qWritten` | `qWritten` | inLoop→outLoop | Filled buffers ready for validation |
| `qSender` | `qSender` | outLoop→MiniSender | Validated buffers ready for TCP send |

All three hold pointers to `rawEvt` structs (not inline data) — messages are `sizeof(rawEvt *)` = 4 bytes. Queue depth: `RAW_Q_SIZE` (defined in `DGS_DEFS.h`). ✅ verified 2026-04-23 — `QueueManagement.c:L84-109`

**`setupFIFOReader()`** — called once from the IOC startup script before sequencer tasks start:
1. Optionally initializes the Universe VME DMA engine (`READOUT_USE_DMA` + `MV5500` — legacy CES RIO3 path; not active on MVME5500). ✅ verified 2026-04-23 — `QueueManagement.c:L55-68`
2. Deletes and recreates the three `MSG_Q` handles if already initialized (safe re-init for IOC restarts).
3. Allocates `RAW_Q_SIZE` `rawEvt` buffers via `newEventBuffer()` and places them all on `qFree`.

**`newEventBuffer()`** — allocates one `rawEvt` via `calloc`. With `READOUT_USE_DMA`, uses `cacheDmaMalloc` + 256-byte alignment (CES RIO3 DMA constraint). Assigns a unique monotonic `uid` per buffer. ✅ verified 2026-04-23 — `QueueManagement.c:L140-175`

**Queue access functions (used by inLoop/outLoop/MiniSender state machines):**
- `getFreeBuf(rawEvt **)` / `putFreeBuf(rawEvt *)` — take/return from free pool
- `getWrittenBuf(rawEvt **)` / `putWrittenBuf(rawEvt *)` — take/put written queue
- `getSenderBuf(rawEvt **)` / `putSenderBuf(rawEvt *)` — take/put sender queue
- `getFreeBufCount()`, `getWrittenBufCount()`, `getSenderBufCount()` — poll queue depths (used by `outLoop.st` for diagnostics)
- `DumpRawEvt(rawEvt *, CallingRoutine, dumplength, dumpstart)` — debug dump of buffer contents

**`rawevent_bufferlist[RAW_Q_SIZE]`** — secondary flat array of all allocated buffers, for debugging/maintenance inspection outside of the queues.

---

## `asynDigitizerDriver` — DIG VME Asyn Driver

_Source: `dgsDrivers/dgsDriverApp/src/asynDigitizerDriver.{cpp,h}` (600 + 107 lines). Code-read 2026-04-26._

The **`asynDigitizerDriver`** is the VxWorks asyn port driver for ANL digitizer boards. It handles all EPICS CA ↔ VME register I/O for the DIG FPGA. Unlike the trigger drivers (which inherit from `asynTrigCommonDriver`), the digitizer driver inherits directly from `asynPortDriver`.

### Initialization — `asynDigitizerConfig()` / `devAsynDigCardInit()`

Called from the VME crate boot script (`.cmd` file):

```
asynDigitizerConfig(portName, card_number, slot)
```

This calls `devAsynDigCardInit(cardno, slot)` first, then constructs a new `asynDigitizerDriver` instance.

**`devAsynDigCardInit()` steps:**
1. Calls `initVmeDrvMutex()` — ensures the VME bus mutex is created (shared with trigger drivers).
2. Calls `devGVMECardInit(cardno, slot)` — fills `daqBoards[cardno].base32`, `.vmever`, `.board`, `.registers`, and `.vmeRegisters[]` (flash addresses).
3. Sets `daqBoards[cardno].FIFO = base32 + 0x1000/4` — FIFO buffer pointer.
4. Probes `base32 + 0x1000/4` via `devReadProbe()`: if readable → `mainOK=1`; bus error → `mainOK=0`.
5. Reads `base32[0x600/4]` → `daqBoards[cardno].rev` (Main FPGA code revision).
6. Reads `base32[0x604/4]` → `daqBoards[cardno].subrev` (Main FPGA code date).
7. Determines **board type** from `rev` bits [15:8]:

| `rev[15:8]` | Board Type Constant | Description |
|---|---|---|
| `0x4C` | `BrdType_ANL_MDIG` | ANL (DGS) Master Digitizer |
| `0x4D` | `BrdType_ANL_SDIG` | ANL (DGS) Slave Digitizer |
| `0xFC` | `BrdType_MAJORANA_MDIG` | ANL Majorana Master Digitizer |
| `0xFD` | `BrdType_MAJORANA_SDIG` | ANL Majorana Slave Digitizer |
| other | `0` | Unknown type |

Board type `0x4C`/`0x4D` encode `'C'`/`'D'` (Master/Slave) after nibble `0x4` (ANL). Majorana uses nibble `0xF`.

### Constructor — `asynDigitizerDriver(portName, card_number)`

- Inherits from `asynPortDriver` with 1024 max parameters (enlarged from 256 by MPC, Oct 2015). ✅ verified 2026-04-26 — `asynDigitizerDriver.cpp:L298` (`1024, // changed from 256 by mpc 10/29/15 per tm`)
- Allocates `address_list[]` array (up to 256 entries) — the param-to-VME-offset map. As of 2025-04-24 the driver has 222 whole-register parameters, safely under the limit. ✅ verified 2026-04-26 — `asynDigitizerDriver.cpp:L313` (`new int_int[256]`); L315 comment (`Digitizer as of this date has 222.`)
- Creates `run_counter` param (incremented by poll task, used as a heartbeat indicator). ✅ verified 2026-04-26 — `asynDigitizerDriver.cpp:L321-322`
- **Textual include:** `#include "asynDigParams.c"` — registers all DIG parameters by calling `createParam()` + `setAddress()` for each. As of 2025-08-15, reverted to a single file: `asynDigParamsVME.c` include was wrapped in `#if 0` (disabled) since the VME FPGA is considered stable; all VME FPGA registers now handled as EPICS-only objects in the digitizer spreadsheet. ✅ verified 2026-04-26 — `asynDigitizerDriver.cpp:L333,L337-340` (only `#include "asynDigParams.c"` is active; `#include "asynDigParamsVME.c"` is inside `#if 0` block with comment `20250815`)
- Spawns `asynDigitizerDriver_Task` thread at medium priority. ✅ verified 2026-04-26 — `asynDigitizerDriver.cpp:L347-350` (`epicsThreadPriorityMedium`)

### Poll Task — `simTask()` (background thread)

Despite the name, not a simulation — this is the **live PV update task**:
- Sleeps 2 seconds per iteration → **2-second update rate** for all digitizer PVs. ✅ verified 2026-04-26 — `asynDigitizerDriver.cpp:L253` (`epicsThreadSleep(2.0)`)
- Increments `run_counter` each cycle (visible from EPICS as a heartbeat). ✅ verified 2026-04-26 — `asynDigitizerDriver.cpp:L255`
- Iterates over `address_list[]` (all params registered with `setAddress()`), reads each VME register via `viIn32()`, updates the asyn parameter via `setUIntDigitalParam()`. ✅ verified 2026-04-26 — `asynDigitizerDriver.cpp:L259-263`
- Calls `callParamCallbacks()` at end of each cycle to propagate changes to EPICS CA. ✅ verified 2026-04-26 — `asynDigitizerDriver.cpp:L266`
- Holds `vme_driver_mutex` during the full read loop to prevent bus contention with other drivers. ✅ verified 2026-04-26 — `asynDigitizerDriver.cpp:L258,L265` (`epicsMutexLock/Unlock`)

### VME I/O — `viIn32()` / `viOut32()`

Thin wrappers around raw pointer dereference:
- `viIn32(slot, adr_space, reg_adr, *data)` → `*data = *(daqBoards[slot].base32 + reg_adr/4)` ✅ verified 2026-04-26 — `asynDigitizerDriver.cpp:L528-537`
- `viOut32(slot, adr_space, reg_adr, data)` → `*(daqBoards[slot].base32 + reg_adr/4) = data` ✅ verified 2026-04-26 — `asynDigitizerDriver.cpp:L490-503`
- `adr_space` parameter is unused (always 0); legacy from GRETINA heritage. ✅ verified 2026-04-26 — `asynDigitizerDriver.cpp:L261` (always passes `0` for `adr_space`)
- Debug print if `asyn_debug_level_d > 1`. ✅ verified 2026-04-26 — `asynDigitizerDriver.cpp:L503,L537`

### Sub-Field Mask Encoding — `0xaaaa0000` sentinel

The same sub-field mask scheme as the trigger drivers (see [`vxworks_trigger_drivers.md`](vxworks_trigger_drivers.md)):
- Masks with `(mask & 0xffff0000) == 0xaaaa0000` signal a sub-field extraction.
- `(mask & 0x0000ff00) >> 8` = number of bits in field.
- `(mask & 0x000000ff)` = bit shift (LSB position within register).
- **Read:** value is masked then right-shifted by `shift` before returning.
- **Write:** value is left-shifted by `shift` before writing to VME.

### Parameter ↔ VME Address Map — `address_list[]` / `setAddress()` / `findAddress()`

- `setAddress(param, address)` — appends `{param_num, address}` pair to `address_list[]`. Called by `#include "asynDigParams.c"` during construction. ✅ verified 2026-04-26 — `asynDigitizerDriver.cpp:L577-579`
- `findAddress(param)` — linear scan through `address_list[]`; returns VME offset or `-1` if not found (write skipped). ✅ verified 2026-04-26 — `asynDigitizerDriver.cpp:L588-593`
- `param_address_cnt` tracks current number of registered (address-mapped) parameters. ✅ verified 2026-04-26 — `asynDigitizerDriver.cpp:L317,L579`
- Note: not all parameters need addresses (counters, status fields); only those needing VME writes call `setAddress()`.

### Global Debug Variables

| Variable | Default | Purpose |
|---|---|---|
| `asyn_debug_level_d` | 0 | Print VME r/w details when > 1 |
| `printevery_d` | 1024 | (legacy, unused in current code) |
| `prog_flip_endian_d` | 1 | (legacy endian-flip flag, `flipEndian()` utility present but not used in main path) |
| `asyn_sleepusec_d` | 0 | Extra sleep in µs (debugging only) |
| `maddog_d` | pointer | Global pointer to last-created driver instance (Tim Madden debug handle) |
| `recLenGDig` | 25 | Record length for GRETINA DIG (legacy reference) |

_All defaults ✅ verified 2026-04-26 — `asynDigitizerDriver.cpp:L65-73`_

---

## Cross-References

- `knowledgeBase/ioc.md` — IOC config, boot scripts, firmware versions, MVME5500 setup
- `knowledgeBase/vxworks_migration.md` — Detailed migration notes from Solaris/con6 to Ubuntu 24
- `knowledgeBase/vxworks_fifo_readout.md` — DMA buffer architecture, trigger FIFO readout, Type-F headers
- `knowledgeBase/EPICS_asyn.md` — asyn driver internals: port model, worker threads, write flow
- `knowledgeBase/VME_registers.md` — VME register addresses used by the IOC driver; full register map extracted from `asynDigParams.c` / `MergedAsynDigParams.c`
- `knowledgeBase/fpga.md` — FPGA firmware overview; the firmware binaries loaded by VxWorks
- `knowledgeBase/ANLDAQ.md` — High-level pipeline overview (inLoop/outLoop/MiniSender data flow diagram + key PVs); complements the detailed state machine docs in this file
- `knowledgeBase/ANLDAQ_tcpReceiver.md` — tcpReceiverMT protocol; the TCP receiver MiniSender connects to
- `knowledgeBase/deep_fpga_RTRG.md` / `knowledgeBase/deep_fpga_MTRG_MAIN.md` — RTRG/MTRG FPGA firmware; maps to trigger driver params in this file
- `knowledgeBase/vxworks_trigger_drivers.md` — **Deep-dive into the trigger asyn drivers** (`asynTrigCommonDriver`, `asynTrigMasterDriver`, `asynTrigRouterDriver`): poll loop internals, `address_list[]` map, `0xaaaa0000` sub-field mask, firmware type code table, boot sequence; split from the summary in this file
- `knowledgeBase/vxworks_vme_devlayer.md` — VME hardware abstraction layer (`devGVME.c`/`devGData.c`/`DGS_DEFS.h`): board init, VMERead32/VMEWrite32, flash programming, `daqBoard`/`daqRegister` structs, board type codes (`BrdType_*`); foundational layer below the state machines and asyn drivers
- `knowledgeBase/vxworks_utility_modules.md` — `MergedAsynDigParams.c` / `asynDigParams.c` parameter registration; `MergedAsynDigParams` asyn class details

*Created (split): 2026-04-23 | Last reviewed: 2026-04-24*
