# EPICS — Experimental Physics and Industrial Control System

_A primer for DGS: how EPICS works under the hood, record types, tools, and Python integration._

---

## 1. What Is EPICS?

EPICS is a distributed control system framework used in particle accelerators, nuclear physics labs, and large instruments worldwide (including DGS at ANL). It is:

- **Distributed:** control data lives on many IOCs (Input/Output Controllers), not one central server
- **Real-time:** IOCs run on bare-metal or RTOS (e.g., VxWorks) for deterministic timing
- **Network-based:** any machine on the network can read/write any PV using Channel Access (CA) protocol
- **Record-based:** each controlled quantity is a "Process Variable" (PV), backed by a strongly-typed record

**Key concepts:**
- **PV (Process Variable):** a named data channel — e.g., `VME01:MDIG1:coarse_threshold0`
- **IOC (Input/Output Controller):** a process (or embedded system) that hosts PVs and talks to hardware
- **CA (Channel Access):** the UDP/TCP protocol that connects clients to IOCs
- **Record:** the in-memory data structure inside an IOC that represents one PV

---

## 2. Record Types

Every PV has a **record type** that determines what it stores and how it behaves.

### Analog (floating-point)
| Type | Direction | Use |
|------|-----------|-----|
| `ai` | Input (from hardware) | Read analog value — e.g., temperature, voltage readback |
| `ao` | Output (to hardware) | Write analog setpoint — e.g., HV demand, threshold |

### Binary (1-bit on/off)
| Type | Direction | Use |
|------|-----------|-----|
| `bi` | Input | Read binary status — e.g., HV on/off, interlock state |
| `bo` | Output | Write binary command — e.g., reset, enable |

### Long integer
| Type | Direction | Use |
|------|-----------|-----|
| `longin` | Input | Read 32-bit integer — e.g., event count, firmware version |
| `longout` | Output | Write 32-bit integer |

### Multi-bit binary (enum)
| Type | Direction | Use |
|------|-----------|-----|
| `mbbi` | Input | Read enum (up to 16 states) — e.g., link state: IDLE/INIT/LOCKED | ✅ verified 2026-04-11 — `mbboRecord.h:L97,L112` (ZRST=Zero String through FFST=Fifteen String, 16 enum fields) |
| `mbbo` | Output | Write enum — e.g., trigger mode select |

### String / waveform
| Type | Direction | Use |
|------|-----------|-----|
| `stringin` / `stringout` | In/Out | 40-char string — e.g., firmware date ✅ verified 2026-04-09 — `ANLDAQ/EPICS/base-7.0/include/epicsTypes.h:L59` (`#define MAX_STRING_SIZE 40`) |
| `waveform` | In/Out | Array — e.g., ADC trace, lookup table |

### Calculated
| Type | Use |
|------|-----|
| `calc` | Compute a value from up to 12 input PVs (e.g., convert raw ADC to voltage) |
| `calcout` | Like `calc` but also writes the result to another PV |

### Special
| Type | Use |
|------|-----|
| `seq` | Sequence: execute a chain of PV writes in order with delays |
| `fanout` | Trigger processing of multiple records |
| `compress` | Statistical reduction of arrays |

---

## 3. The IOC — Input/Output Controller

The IOC is the heart of EPICS. It:
1. **Loads record definitions** from `.db` files (the database)
2. **Initializes device support** — links records to hardware drivers
3. **Starts the CA server** — listens on port 5064 (UDP) and 5065 (TCP) by default ✅ verified 2026-04-09 — `ANLDAQ/EPICS_para.sh:L45-46` (DGS system CA ports)
4. **Scans records** periodically or on events — updates PV values from hardware

### Types of IOC

**Hardware IOC:** runs on embedded hardware (e.g., MVME5500 VxWorks) — directly accesses VME registers, SPI buses, etc. DGS VME crates run this type.

**Soft IOC:** runs as a normal Linux process — no direct hardware access. Uses drivers that talk to hardware over network, serial, or shared memory. DGS collector box Pis run softIOC (`softIoc` binary from EPICS base).

### softIOC
```bash
softIoc -d myrecords.db
```
Starts a full CA server hosting whatever records are in `myrecords.db`. Useful for:
- Collector box control (Python logic + EPICS interface)
- Testing
- Software-only PVs (calculations, alarms, etc.)

### Record scanning
Records are processed (hardware read → value updated → monitors notified) by:
- **Periodic scan:** every N seconds (0.1, 0.2, 0.5, 1, 2, 5, 10 sec)
- **I/O Intr:** hardware interrupts the IOC when new data is ready (fastest)
- **Event:** triggered by another record or external event
- **Passive:** only processed when written to or explicitly triggered

---

## 4. Channel Access (CA) — Under the Hood

CA is a client-server protocol. Clients (caget, caput, PyEPICS, CSS) talk to IOC servers.

### Discovery (name resolution)
1. Client wants PV `VME01:MDIG1:coarse_threshold0`
2. Sends **UDP broadcast** on port 5064: "Who has this PV?"
3. IOC that owns it replies with its IP + TCP port
4. Client opens a **TCP connection** to that IOC for subsequent operations
5. Connection is kept alive for subscriptions; re-established on IOC restart

**CA_ADDR_LIST:** If broadcast doesn't work (different subnets), set this env var to list specific IOC IPs:
```bash
export EPICS_CA_ADDR_LIST="192.168.203.141 192.168.203.142"
export EPICS_CA_AUTO_ADDR_LIST=NO
```
DGS uses this — all 12 VME IOCs are on `192.168.203.141-145, 177-183`. ✅ verified 2026-04-09 — `ANLDAQ/tcpReceiver/start_run.sh:L12`

### caget — read a PV
```bash
caget VME01:MDIG1:coarse_threshold0
# Under the hood:
# 1. UDP search broadcast
# 2. TCP connect to IOC
# 3. CA_PROTO_READ request
# 4. IOC reads hardware (or cached value), returns it
# 5. TCP connection closed (for one-shot caget)
```

### caput — write a PV
```bash
caput VME01:MDIG1:coarse_threshold0 500
# Under the hood:
# 1. UDP search + TCP connect
# 2. CA_PROTO_WRITE request with value 500
# 3. IOC receives it → record processes → device support writes to hardware
# 4. IOC sends write-complete acknowledgment
# 5. caput returns
```

`caput -w 5` waits up to 5 seconds for processing to complete (useful for records that trigger slow hardware ops).

### camonitor — subscribe to a PV
```bash
camonitor VME01:MDIG1:coarse_threshold0
# Under the hood:
# 1. UDP search + TCP connect
# 2. CA_PROTO_EVENT_ADD (subscribe) request
# 3. IOC adds client to monitor list for this PV
# 4. Whenever PV value changes (or on periodic scan), IOC sends CA_PROTO_EVENT (monitor update)
# 5. Client prints each update
# Connection stays open indefinitely
```

### Subscriptions (monitors)
The most efficient CA pattern. Instead of polling (caget in a loop), you **subscribe once** and the IOC pushes updates:
- IOC sends update only when value changes (or at defined rate)
- Minimal network traffic
- Used by CSS, EDM, PyEPICS `ca.monitor()`, ANLDAQ GUI

**Dead-band (MDEL/ADEL):** IOC only sends monitor updates if change exceeds a threshold — avoids flooding clients with noise.

---

## 5. PyEPICS — Python CA Client

`pyepics` is the Python binding for Channel Access. Used in DGS collector box IOC logic, monitoring scripts, and ANLDAQ.

### Install
```bash
pip install pyepics
```
Requires EPICS base libraries and `EPICS_CA_ADDR_LIST` set correctly.

### Basic usage
```python
import epics

# Read
val = epics.caget('VME01:MDIG1:coarse_threshold0')
print(val)  # e.g., 500.0

# Write
epics.caput('VME01:MDIG1:coarse_threshold0', 600)

# Read with metadata
pv = epics.PV('VME01:MDIG1:coarse_threshold0')
print(pv.value)      # current value
print(pv.units)      # engineering units
print(pv.count)      # number of elements (1 for scalar)
print(pv.connected)  # True if IOC is up
```

### Subscriptions in PyEPICS
```python
import epics

def my_callback(pvname=None, value=None, **kw):
    print(f"{pvname} changed to {value}")

# Subscribe — callback fires every time PV changes
pv = epics.PV('VME01:MDIG1:coarse_threshold0', callback=my_callback)

# Or via ca module:
import epics.ca as ca
ca.monitor('VME01:MDIG1:coarse_threshold0', callback=my_callback)
```
The callback runs in a background thread managed by PyEPICS.

### Waiting for connections
```python
epics.caput('GS1_GE_HV_DEMAND_VOLTS', 3000, wait=True, timeout=10)
# wait=True blocks until IOC confirms processing complete
```

### PV object (persistent connection)
```python
pv = epics.PV('VME01:MDIG1:coarse_threshold0')
pv.get()           # explicit read
pv.put(700)        # write
pv.add_callback(my_callback)   # subscribe
pv.clear_callbacks()           # unsubscribe
pv.disconnect()    # close CA connection
```
PV objects keep the TCP connection open — much faster for repeated access than `caget`/`caput` calls.

---

## 6. DGS PV Hierarchy — User PVs vs reg_* vs DAQC

The DGS IOC exposes PVs at three distinct layers:

```
Layer 1 — User PVs (config, writable)
  VME01:MDIG1:CFD_fraction0         (ao, scaled %, user-facing)
  VME01:MDIG1:channel_enable0       (bo, enable/disable)
  VME01:MDIG1:coarse_threshold0     (ao, ADC counts)
       │
       │ asyn write → read-modify-write on hardware register
       ▼
Layer 2 — Raw Register PVs (reg_*)
  VME01:MDIG1:reg_CFD_fraction0     (asynUInt32Digital, raw bit field)
  VME01:MDIG1:reg_channel_control0  (asynUInt32Digital, full 32-bit register)
       │
       │ VME bus write
       ▼
Layer 3 — FPGA Hardware Register
  Physical 32-bit register on DIG/RTRG/MTRG board
```

### reg_* PVs

- **What:** Raw FPGA register records — one per hardware register, 32-bit
- **How:** `DTYP=asynUInt32Digital` with a bit mask (`asynMask`) for bit-field access ✅ verified 2026-04-09 — `ioc/db/MDigUser.template:L462` (`reg_CFD_fraction0` uses `asynMask` with mask `0xaaaa0D00`; 1053 asynUInt32Digital records in MDigUser.template alone)
- **Why they exist:** User PVs write individual bit fields within a shared register; the `reg_*` PV represents the full register. The asyn driver does read-modify-write using the mask.
- **Rule:** Never set `reg_*` PVs directly — set the user-facing PVs instead. The `reg_*` records update automatically.
- **In snapshots:** Always excluded from `dumpPVs.py` and `putPVs.py` via `pv_filter.py` (`EXCLUDE_CONTAINS: 'reg_'`)
- **RBV variants:** `reg_*_RBV` are `bi` records scanning at 1 second — read the hardware register back for monitoring ✅ verified 2026-04-09 — `ioc/db/MDigUser.template` (all `_RBV` ai/longin records use `SCAN="1 second"`)

### DAQC PVs (DAQ Crate monitoring)

- **What:** Per-crate data acquisition performance monitoring PVs
- **Naming:** `DAQC{NN}_*` where `NN` = crate number (01–12 for DGS; 66 for DuoGe)
- **All read-only** — `ai`, `mbbi` record types, reflect live runtime state
- **Key PVs:**

| PV Pattern | Description |
|------------|-------------|
| `DAQC{NN}_CV_CRATENUM` | Crate number |
| `DAQC{NN}_CV_InLoop1` | MB/s read rate from DIG FIFOs |
| `DAQC{NN}_CV_InLoop2` | Type-F buffer count (raw) |
| `DAQC{NN}_CV_InLoop3` | VME transfer count |
| `DAQC{NN}_CV_InLoop4` | Result of last transfer |
| `DAQC{NN}_CV_BuffersAvail` | Available DMA buffers |
| `DAQC{NN}_CV_SendRate` | TCP send rate |
| `DAQC{NN}_OL_BufLostPerecnt` | Buffer loss percentage (**note:** typo in template — "Perecnt" not "Percent") |
| `DAQC{NN}_BoardType{N}` | Board type in VME slot N |

✅ verified 2026-04-09 — `ioc/db/daqCrate.template:L2,L9-12,L284,L323,L325` (all PV names confirmed; typo `OL_BufLostPerecnt` is in the template itself)

- **In snapshots:** Always excluded (`EXCLUDE_STARTSWITH: 'DAQC'`) — runtime stats, not config

---

## 7. EPICS DB File Syntax

A `.db` file defines records:

```
# Actual DGS example (from MDigUser.template):
record(ao, "VME$(CRATE):$(BOARD):coarse_threshold0") {
    field(DTYP, "Raw Soft Channel")
    field(EGU,  "adu")
    field(DESC, "coarse discriminator thr")
    field(OUT,  "VME$(CRATE):$(BOARD):coarse_threshold0LONGOUT PP NMS")
}
record(longout, "VME$(CRATE):$(BOARD):coarse_threshold0LONGOUT") {
    field(OUT,  "@asynMask($(BOARD),0,0x00003fff,1)reg_coarse_threshold0")
    field(DTYP, "asynUInt32Digital")
}
```
✅ verified 2026-04-09 — `ioc/db/MDigUser.template:L33-50` (ao front-end with Raw Soft Channel → longout intermediary with asynUInt32Digital; EGU=adu)

Key fields:
| Field | Meaning |
|-------|---------|
| `DESC` | Description string |
| `DTYP` | Device type (which driver handles this) |
| `INP` / `OUT` | Hardware link (for input/output records) |
| `EGU` | Engineering units |
| `PREC` | Display precision |
| `SCAN` | Scan rate or method |
| `HIHI/LOLO` | Alarm limits |
| `MDEL/ADEL` | Monitor/archive dead-band |
| `FLNK` | Forward link — trigger another record after processing |

**Macros:** `$(CRATE)`, `$(BOARD)` are substituted when the DB is loaded:
```
dbLoadRecords("MDigUser.template", "CRATE=01,BOARD=MDIG1,PORT=VME01_MDIG1")
```

---

## 8. DGS-Specific EPICS Setup

### CA Port assignments
| System | CA port |
|--------|---------|
| DGS | 5064/5065 |
| X-Array (DXA) | 5072/5073 |
| DuoGe (DUO) | 5080/5081 |

Set before using CA tools:
```bash
export EPICS_CA_SERVER_PORT=5064
export EPICS_CA_REPEATER_PORT=5065
export EPICS_CA_ADDR_LIST=""   # empty = use subnet broadcast (onenet)
export EPICS_CA_AUTO_ADDR_LIST=YES

# Note: ANLDAQ uses empty CA_ADDR_LIST for DGS/DXA (broadcast on 192.168.203.x subnet).
# Only SBX/DUO use specific addresses: EPICS_CA_ADDR_LIST="192.168.203.28 vme99.onenet"
# ✅ verified 2026-04-13 — ANLDAQ/EPICS_para.sh:L14,L43 (DGS=empty), L34 (SBX=192.168.203.28+vme99.onenet)
```

### dgsSoftIOC — DGS Central Soft IOC (DFMA host)

The **dgsSoftIOC** is a Linux-process IOC that runs on the DFMA host machine alongside `commander.py`. It is not a VME hardware IOC — it hosts software-only PVs that the GUI and setup scripts depend on.

**Location:** `ANLDAQ/EPICS/softIOC/` ✅ verified 2026-04-21 — directory listing

**Boot command:** `ANLDAQ/EPICS/softIOC/iocBoot/iocdgsSoftIOC/dgsSoftIoc.cmd` ✅ verified 2026-04-21 — `dgsSoftIoc.cmd` contents
```
dbLoadRecords "db/JustGlobals.db"
dbLoadRecords "db/dgsSupport.db"
iocInit
```

**Auto-spawn:** `commander.py` tries `caget Online_CS_StartStop` on startup; if unreachable (3 retries), spawns the softIOC in a `gnome-terminal`. Checks `ps ax` first to avoid double-spawn. ✅ verified 2026-04-17 — `commander.py:L751-798`

**CA port:** 5064/5065 (shared with DGS system — same subnet broadcast)

#### dgsSupport.db — Run Control PVs

235-line file. Key PVs: ✅ verified 2026-04-21 — `softIOC/db/dgsSupport.db` contents

| PV | Type | Purpose |
|----|------|---------|
| `RunNum` | `longout` | Current run ID number (added by Ryan 2025-03-23) |
| `Online_CS_StartStop` | `bo` | Main run/stop control (big red button). `0=Stop`, `1=Start` |
| `Online_CS_SaveData` | `bo` | Data save toggle. `0=No Save`, `1=Save` |
| `Setup_Script_State` | `mbbo` | Setup script state machine: `UNKNOWN/TRIG OK/DIG OK/OTHER/TRIG ERROR/DIG ERROR/OTHER ERROR/SCRIPT RUNNING` (8 states) |
| `ScriptStage` | `ao` | Stage counter shown during long setup scripts (progress indicator for users) |
| `VME10:MTRG:TIMESTAMP_RBV` | `calcout` | 32-bit MTRG timestamp: `(B<<16)+C` from `_A/_B/_C` registers, scanned 0.1 s |
| `VME10:MTRG:TRIG_RATE_COUNTER_1–8_RBV` | `calcout` | 32-bit trigger rate counters (post-filter): `(B<<16)+A` from high/low halves, scanned 1 s |
| `VME10:MTRG:RAW_TRIG_RATE_COUNTER_1–8_RBV` | `calcout` | Same but raw (pre-filter) trigger rates |

#### JustGlobals.db — Global Fanout PVs

14,248-line file containing **2,124 `dfanout` records** that broadcast global settings to all VME crates. ✅ verified 2026-04-21 — `wc -l JustGlobals.db`, `grep -c "^record(dfanout"` count

**Fanout pattern:** Chained `dfanout` records cascade writes from a single root PV to all 12 VME crates:
```
caput GLBL:DIG:F00:master_fifo_reset 1
  → GLBL:DIG:01:master_fifo_reset → VME01:GLBL:master_fifo_reset
  → GLBL:DIG:02:master_fifo_reset → VME02:GLBL:master_fifo_reset
  → ...chain continues...
  → GLBL:DIG:11:master_fifo_reset → VME12:GLBL:master_fifo_reset
```
Each link fans to one next-in-chain record AND one VME crate (`PP NMS` — Process Passively, No Maximize Severity).

**Fanout target groups (177 unique PV suffixes broadcast):**
- `master_fifo_reset`, `BGOs_ext_disc_src` — basic DIG control
- `BGOp_*` / `BGOs_*` — BGO plastic/shield digitizer settings (~35 params each): CFD fraction, thresholds, windows, channel enable, preamp reset, write flags, etc.
- `GeC_*` — Ge crystal digitizer settings (same parameter set)
- `holdoff_time`, `veto_enable`, `peak_sensitivity`, `stop_ho_at_peak` — trigger/pileup control
- `sd_*` — SerDes link parameters: `sd_sync`, `sd_pem`, `sd_rx_pwr`, `sd_tx_pwr`, `sd_sm_lost_lock_flag_rst`, `sd_sm_stringent_lock`, `sd_line_loopback_en`, `sd_local_loopback_en`
- `trigger_mux_select`, `EXT_DISC_REQ`, `ext_disc_ts_sel` — trigger routing
- `ts_counter_mode`, `ts_counter_reset` — timestamp counter control
- `clk_select`, `cfd_mode`, `counter_mode` — digitizer mode settings
- `dac_attenuation`, `dac_channel_select` — DAC control
- `DIAG_DISC_SEL`, `DIAG_WAVE_SEL`, `diag_input`, `diag_input_en`, `diag_mux_control` — diagnostics
- `FIFO_Prog_Thresh`, `lfsr_rate_sel`, `lfsr_seed`, `load_delays`, `rj45_throttle_mode` — misc
- `win_comp_max`, `win_comp_min` — window comparator

**Why dfanout?** The VME IOC `asynUInt32Digital` records don't chain natively. The soft IOC acts as a broadcast hub: one `caput` to a `GLBL:` PV cascades to all crates. Used by setup scripts and the GUI's "apply global setting" operations.

### Collector box (softIOC on Pi)
Each Raspberry Pi runs a softIOC for one detector. The IOC:
1. Loads DB templates from `CollectorBox_RevA/db/`
2. Uses custom device support (`CollectorApp`) to talk to the pickoff FPGA via SPI
3. Exposes 1,431 PVs per detector (GS hole numbering) ✅ verified 2026-04-16 — `grep -c "^record(" CollectorBox_RevA/db/*.db` sum = 1,431 (Pickoff.db:448, StrpFPGA.db:294, Pickoff_reg.db:264, CtrlFPGA_reg.db:121, etc.) — see `collectorbox_PVs.md` header for full breakdown
4. Runs `caput`/`caget` against the VME IOCs to coordinate HV with the digitizer

### Useful one-liners for DGS
```bash
# Check all coarse thresholds on VME01 MDIG1
for ch in $(seq 0 9); do caget VME01:MDIG1:coarse_threshold${ch}; done

# Monitor Ge HV for GS hole 5
camonitor GS5_GE_HV_DEMAND_VOLTS GS5_Conv_GeHV

# Check trigger mode on all boards
for c in 01 02 03; do for b in MDIG1 MDIG2; do caget VME${c}:${b}:trigger_mux_select_RBV; done; done

# Check MTRG threshold
caget VME10:MTRG:Threshold
```

---

## 9. Database Definition Files (.dbd)

`.dbd` files are the **schema layer** of EPICS — they define what record types, device supports, and drivers exist. Without them, the IOC cannot interpret `.db` files at load time.

### What a .dbd declares

**Record type definitions** — the fields a record has, their types, defaults:
```
recordtype(ai) {
    field(VAL,  DBF_DOUBLE)  { prompt("Current EGU Value") }
    field(EGU,  DBF_STRING)  { prompt("Engineering Units") }
    field(DTYP, DBF_DEVICE)  { prompt("Device Type") }
    ...
}
```

**Device support registrations** — maps `DTYP` strings to C device support structs:
```
device(ai, INST_IO,   devAiSoft,   "Soft Channel")
device(ai, CAMAC_IO,  devAiCamac,  "CAMAC")
device(ao, INST_IO,   devAoSoft,   "Soft Channel")
```
This is how EPICS knows that `field(DTYP, "CAMAC")` maps to `devAiCamac.c`.

**Driver table entries**:
```
driver(drvCamac)
```

**Menu enumerations** (shared across all records):
```
menu(menuScan) {
    choice(menuScanPassive,   "Passive")
    choice(menuScanEvent,     "Event")
    choice(menuScan1_second,  "1 second")
    ...
}
```

**Registrar functions** (called at IOC init):
```
registrar(myModuleRegister)
```

### How .dbd files are assembled

At build time, EPICS uses `dbExpand` to merge multiple `.dbd` fragments into one final `.dbd`:

```
epics-base records (ai.dbd, ao.dbd, ...) \
    + device support (devSoft.dbd, devCamac.dbd, ...) \
    + driver (drvCamac.dbd) \
    → final App.dbd  (loaded by IOC at startup)
```

In the DGS collector box IOC:
- `CollectorApp.dbd` is assembled from EPICS base + `CollectorSupport.dbd` (custom device support)
- Declares the SPI driver and `CAMAC_IO` link type used by all Collector Box PVs

### Mental model: .dbd vs .db vs .cmd

| File | Role | Analogy |
|------|------|---------|
| `.dbd` | Schema — what record types and device supports exist | SQL `CREATE TABLE` |
| `.db` / `.template` | Data — actual PV instances | SQL `INSERT INTO` |
| `.cmd` | Startup script — load DB, configure hardware, start IOC | Shell script |

### In practice

- You never edit `.dbd` files unless you're writing new device support in C
- If you add a new DTYP in a `.db` file, it must be registered in a `.dbd` or the IOC will refuse to load
- `dbd/` directories in EPICS repos contain the fragments; `O.*/` build dirs contain the merged result

---

## See Also

- `knowledgeBase/EPICS_asyn.md` — asyn driver internals: caput/caget flow, port concept, worker threads, bulk writes
- `knowledgeBase/ioc.md` — DGS EPICS IOC config, boot scripts, firmware versions, MVME5500 setup
- `knowledgeBase/collectorbox_devicesupport.md` — Collector box device support: SPI driver, CAMAC_IO link, conversion coefficients
- `knowledgeBase/collectorboxpi.md` — Raspberry Pi soft IOC, PXE boot, HV control
- `knowledgeBase/vxworks.md` — VxWorks build pipeline (the RTOS running the VME IOCs)
- `knowledgeBase/DGS_PVs.md` / `knowledgeBase/collectorbox_PVs.md` — Full PV lists for DGS and collector box

---

_Source: EPICS documentation, DGS source code, operational experience._
_Created: 2026-04-05. .dbd section added 2026-04-06. Cross-references added 2026-04-07._
