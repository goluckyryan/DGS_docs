# VxWorks FIFO Readout Internals

Deep-dive documentation for the VME FIFO readout pipeline in the DGS IOC. Covers DMA buffer architecture, trigger FIFO readout (`readTrigFIFO.c`), and Type-F synthetic headers for both digitizer and trigger paths.

_Split from `vxworks.md` on 2026-04-20. Sources: `dgsDrivers/dgsDriverApp/src/`, `DGS_SVN/dgs/Documentation/`_

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
- Queue size: **`RAW_Q_SIZE = 200`** (defined in `DGS_DEFS.h`; previously 400 before April 2023) ✅ verified 2026-04-20 — `vxworks/dgsDrivers/dgsDriverApp/src/DGS_DEFS.h:L48` (`#define RAW_Q_SIZE 200 //changed from 400 to 200 20230412 JTA`)
- Each buffer has a **reference counter** — zero = free, non-zero = in use

> **Note:** The salvaged notes (`20180924_notes.txt`) document the older values (RAW_Q_SIZE=400, RAW_BUF_SIZE=512KB). These were doubled/halved in April 2023 — same total memory (~200MB), different trade-off.

### Trigger Throttle (software fallback)
- If buffers in Return Queue fall below **1/3** of `RAW_Q_SIZE` (i.e., <67 free with current Q=200), a sequencer program asserts trigger inhibit PVs via EPICS CA. ⚠️ Note: The PV names `TrigInhD`/`TrigInhL` (and sequencer `TrigCon.st`) are from the **legacy GRETINA/PSG DAQ** (`DGS_SVN/psg/CodeGeneratingSpreadsheetGeneric/dgs_databases_20220714/daqCrate.template`) — they do **not** exist in the current DGS IOC (`ANLDAQ/`, `ioc/`, `collectorboxpi/`). The current DGS throttle mechanism uses hardware FIFO half-full flags rather than CA sequencer inhibit. ✅ verified 2026-04-20 — confirmed absence of `TrigInhD`/`TrigInhL` in all current DGS repos; origin traced to `psg/daqCrate.template` (GRETINA era)
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

✅ verified 2026-04-19 — `readTrigFIFO.c:L78-96` (FIFO_READ_ADDRESS array; MON FIFO 7 = `0x5000` with comment "JTA 20250528 moved MON FIFO 7 to 0x5000"). Note: the cheat sheet comment in `inLoopSupport.c:L678-694` still shows `0x0178` for MON FIFO 7 — that comment is **outdated** and disagrees with `FIFO_READ_ADDRESS[6] = 0x5000` in `readTrigFIFO.c`, which is the ground truth.

### FIFO Status Register

- **`MTRG_MON_FIFO_STATE_REG`** at offset `0x01B4` — 16-bit register covering all 8 MON FIFOs; 2 bits per FIFO:
  - Bit `2n+1` = Full flag for MON FIFO n
  - Bit `2n+0` = Empty flag for MON FIFO n
  - ✅ verified 2026-04-19 — `inLoopSupport.c:L73` (`MTRG_MON_FIFO_STATE_REG = (0x1B4/4)`) + L709 (used for all FifoNum < 8). Note: in-code comment at L666 says `0x01A0` but that contradicts the variable definition — `0x01B4` is the ground truth.
- **`MTRG_CHAN_FIFO_STATE_REG`** (CHAN_FIFO_STATE) at offset `0x01A4` ✅ verified 2026-04-19 — `inLoopSupport.c:L74` (`MTRG_CHAN_FIFO_STATE_REG = (0x01A4/4)`)
- **MON FIFO 7 live depth** at `0x0154` (live counter, in longwords) ✅ verified 2026-04-19 — `inLoopSupport.c:L79` (`MTRG_MON7_LIVE_DEPTH = (0x0154/4)`)
- **MON FIFO 7 latched depth** at `0x01AC` (latched at event boundaries — used for readout sizing) ✅ verified 2026-04-19 — `inLoopSupport.c:L78` (`MTRG_MON7_LATCHED_DEPTH = (0x01AC/4)`)

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

---

## Type-F Headers

Synthetic 4-word GEB-format packets generated when a FIFO is empty, overflowed, or at end-of-run. Both trigger and digitizer paths produce these; the format is the same but register addresses differ.

### Trigger Type-F Headers (`TriggerTypeFHeader()` in `readTrigFIFO.c`)

The 4-word format:

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

### Digitizer Type-F Headers (`DigitizerTypeFHeader()` in `readDigFIFO.c`)

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

## Cross-References

- `knowledgeBase/vxworks.md` — VxWorks build, IOC overview, munch process, glossary
- `knowledgeBase/VME_registers.md` — VME register addresses used by the driver
- `knowledgeBase/ioc.md` — IOC config, boot scripts, firmware versions
- `knowledgeBase/fpga.md` — FPGA firmware overview; counterpart to the IOC driver
- `knowledgeBase/deep_fpga_DIG_eventpacket.md` — Digitizer event packet format (counterpart to Type-F headers)
