# EPICS asyn — Asynchronous Driver Support

_How asyn works under the hood, with DGS context._

---

## What Is asyn?

**asyn** = Asynchronous Driver Support for EPICS. It is a middleware layer between EPICS records and hardware drivers.

**The problem it solves:** Without asyn, every hardware driver must implement EPICS device support directly — handling record types, scan locking, callbacks, etc. asyn provides a **standardized read/write interface** so:
- Hardware drivers implement one clean API
- Standard device support handles all the EPICS record glue
- Drivers can be non-blocking without deadlocking the IOC scan threads

---

## Mental Model: What asyn Does

asyn is a **broker** between EPICS records and hardware:

- **Write (caput → hardware):** asyn queues the write request, a worker thread does the actual hardware access, then calls back with success/error. The calling thread does not wait.
- **Read (caget ← hardware):** same — asyn queues the read, worker thread fetches the value, callback delivers it to the record.
- **Errors do come back** — as record alarms (`SEVR`/`STAT` → `INVALID`/`COMM_ALARM`). It is not fire-and-forget.
- **The "async" part** = the IOC scan thread is never blocked. The actual hardware I/O is synchronous inside the worker thread.

---

## Full caput Flow (ASCII)

```
USER MACHINE                        IOC (VxWorks on MVME5500)
─────────────────                   ──────────────────────────────────────────

[caput VME01:MDIG1:threshold0 100]
   │
   │  Channel Access (UDP/TCP)
   │  over onenet (192.168.203.x)
   ▼
┌─────────────────────────────────────────────────────────────────┐
│  CA SERVER THREAD  (epics-base, runs inside IOC process)        │
│  Receives put request over network                              │
│  Looks up record "VME01:MDIG1:threshold0" in PV database        │
│  Calls dbPutField(record, value=100)                            │
└───────────────────────────┬─────────────────────────────────────┘
                            │ (same process, function call)
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  RECORD PROCESSING  (epics-base, ao/longout record)             │
│  Runs in CA server thread                                       │
│  Validates value, applies limits                                │
│  Calls device support: write_ao()                               │
│  → sees DTYP="asynInt32", calls asynInt32->write()              │
└───────────────────────────┬─────────────────────────────────────┘
                            │ (same process, queues work)
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  ASYN PORT MANAGER  (asyn library, inside IOC process)          │
│  Receives write request for port "VME01_MDIG1", addr=0          │
│  If port free: hands to worker thread immediately               │
│  If port busy: queues the request                               │
│  CA server thread returns immediately (non-blocking) ◄──────┐  │
└───────────────────────────┬─────────────────────────────────┼──┘
                            │ (wakes up worker thread)        │
                            ▼                                 │
┌─────────────────────────────────────────────────────────────┐  │
│  ASYN WORKER THREAD  (1 per port, inside IOC process)       │  │
│  Calls driver: myDriver->writeInt32(user, value=100)        │  │
│  → driver translates: register_offset = 0x0010             │  │
│  → calls vmeWrite32(baseAddr + 0x0010, 100)                 │  │
│    (blocks until VME bus completes — microseconds)          │  │
│  VME bus write completes                                    │  │
│  Driver calls callback: "done, status=OK"                   │──┘
└───────────────────────────┬─────────────────────────────────┘
                            │ (hardware write)
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  VME BUS → FPGA REGISTER (DIG / RTRG / MTRG board)         │
│  Passive hardware — just accepts the write                  │
│  No acknowledgment from hardware needed                     │
└─────────────────────────────────────────────────────────────┘
```

---

## Programs & Threads Involved in One caput

| What | Where | Thread |
|------|-------|--------|
| `caput` CLI | User's machine | User's shell process |
| CA server | Inside IOC process | CA server thread |
| Record processing | Inside IOC process | CA server thread |
| asyn port manager | Inside IOC process | CA server thread (queues, returns) |
| asyn worker | Inside IOC process | 1 dedicated thread per port |
| Hardware driver | Inside IOC process (compiled in) | Worker thread |
| VME bus + FPGA | Hardware | — |

**Key:** It's all **one process** on the IOC. The driver is not a separate program — it's a `.o` file compiled and linked into the IOC binary (`gretDet.munch` in DGS). The threads are VxWorks tasks.

---

## What Is an asyn Port?

A **port** is a named logical connection to one piece of hardware — registered by the driver at IOC startup:

```c
// In st.cmd or driver init:
dgsDigitizerConfig("VME01_MDIG1",   // port name (arbitrary string)
                    0xC0000000,      // VME base address of this board
                    ...)
```

**Port ≈ one addressable hardware unit.** In DGS, one port = one board:

| Port name | Maps to |
|-----------|---------|
| `"VME01_MDIG1"` | Master DIG board, slot X, crate 01 |
| `"VME01_MDIG2"` | Slave DIG board, slot Y, crate 01 |
| `"VME01_RTRG1"` | RTRG board, crate 01 |
| `"VME10_MTRG"`  | MTRG board, crate 10 |

Port names are abstract strings — asyn works the same way for VME, serial, USB, SPI, Ethernet. The driver knows internally what hardware the name maps to.

---

## Bulk caput: 100 Simultaneous Writes

When pyepics fires 100 `caput` calls at once with `wait=False`:

```
100 caput requests arrive at IOC
        │
        ▼
CA server distributes by port:
  50 → port "VME01_MDIG1"   (1 worker thread, 1 queue)
  50 → port "VME01_MDIG2"   (1 worker thread, 1 queue)
        │                         │
        ▼                         ▼
  QUEUE of 50 requests      QUEUE of 50 requests
  Worker drains one-by-one  Worker drains one-by-one
  (VME writes ~1µs each)    (in parallel with MDIG1)
```

- **1 worker thread per port** — writes to one board are always serialized (hardware can only handle one VME access at a time)
- **Multiple ports run in parallel** — MDIG1 and MDIG2 writes happen truly concurrently
- **No 100 threads needed** — the queue absorbs the burst
- **VME bus arbiter** handles contention at the hardware level when multiple boards are accessed simultaneously

---

## Typed Interfaces

asyn defines typed interfaces matching data types:

| Interface | Data type | Use case |
|-----------|-----------|----------|
| `asynInt32` | 32-bit integer | Register values, thresholds, counters |
| `asynUInt32Digital` | 32-bit with mask | Bit fields, enable/disable bits |
| `asynFloat64` | 64-bit float | Calibration coefficients, temperatures |
| `asynOctet` | String / byte array | Status strings, waveform data |

The driver implements whichever interfaces its hardware needs.

---

## Passive Hardware — How "Callbacks" Work

For a passive device (VME register, switch) that cannot initiate communication:

1. EPICS scan engine triggers the record at its configured scan rate (e.g., "1 second")
2. Record asks asyn: "read now"
3. asyn worker thread calls `vmeRead32(address)` — this **blocks** in the worker thread (microseconds for VME)
4. Worker gets value, **driver calls the callback itself**: `pasynUser->callback(value, status=OK)`
5. asyn delivers value to record; record updates VAL, notifies camonitor clients

The hardware is completely dumb — **the IOC polls**; the "callback" is the driver signaling completion internally, not hardware calling back.

For interrupt-driven hardware, the interrupt handler calls `setIntegerParam()` + `callParamCallbacks()` — pushing updates without polling.

---

## asyn in DGS

- Used in the VxWorks VME IOC (`gretDet.munch`) — asyn R4-37 cross-compiled for MVME5500
- Each DIG, RTRG, MTRG board = one asyn port
- Port name format: `VME<crate>_<board>` (e.g., `VME01_MDIG1`)
- The collector box Pi IOC uses an older `CAMAC_IO` device support style (pre-asyn) — direct device support, no asyn layer

---

*Created: 2026-04-06 from conversation with Ryan Tang.*
