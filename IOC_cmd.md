# IOC Shell Commands

Commands available in the DGS VxWorks IOC shell (EPICS 3.14.12.1 + asyn 4.17).

Access via terminal server: `telnet <ts-ip> <port>` — see `ANLDAQ/EPICS_para.sh` for per-system IPs/ports.

> **Safety:** Wrong commands can erase boot settings or crash the IOC. Never run destructive commands without authorization. See the [safe vs. dangerous](#safety-classification) section.

---

## DGS Custom Commands

Registered via `iocshRegister` in `VxWorks/dgsDrivers/dgsDriverApp/src/`.

### Driver Initialization (boot-time only)

These are called automatically by the boot script. Do not re-run on a live IOC without authorization.

| Command | Parameters | Description | Source |
|---------|-----------|-------------|--------|
| `InitializeDaqBoardStructure()` | none | Init all VME board data structures | `devGVME.c` |
| `asynDigitizerConfig` | `portName, card#, slot` | Init DIG board asyn driver | `drvAsynDigitizer.c` |
| `asynTrigRouterConfig1` | `portName, card#, slot` | Init RTRG asyn driver | `drvAsynTrigRouter.c` |
| `asynTrigMasterConfig1` | `portName, card#, slot` | Init MTRG asyn driver | `drvAsynTrigMaster.c` |
| `asynDebugConfig` | `portName, card#` | Init debug asyn driver | `drvAsynDebug.c` |
| `setupFIFOReader()` | none | Start FIFO reader thread for DMA readout | `QueueManagement.c` |

### Flash Firmware — DANGEROUS

**Never run without explicit user authorization.** Erasing or mis-programming flash requires physical board recovery.

| Command | Parameters | Description | Source |
|---------|-----------|-------------|--------|
| `ProgramFlash` | `bdnum, bank, "file.bin"` | Erase + write firmware to flash bank (0=lower, 1=upper) | `devGVME.c` |
| `EraseFlash` | `bdnum, bank` | Erase one flash bank | `devGVME.c` |
| `VerifyFlash` | `bdnum, bank, stopOnErr, "file.bin"` | Verify flash content against binary file | `devGVME.c` |
| `ConfigureFlash` | `bdnum, bank` | Instruct VME FPGA to reconfigure main FPGA from flash | `devGVME.c` |

Typical firmware upload sequence (from `ioc/boot/uploadFW.cmd`):
```
ProgramFlash(0, 0, "/global/ioc/firmware/BUS_LEFT.bin")
ConfigureFlash(0, 0)
```

### VME Register I/O

| Command | Parameters | Description | Safe? | Source |
|---------|-----------|-------------|-------|--------|
| `VMERead32` | `bdnum, regaddr` | Read 32-bit VME register (byte offset); prints value. `bdnum` = cardno (2nd arg of `asynDigitizerConfig`) | Yes | `devGVME.c` |
| `VMEWrite32` | `bdnum, regaddr, data` | Write 32-bit VME register (byte offset) directly. `bdnum` = cardno | **No** | `devGVME.c` |
| `devGVMECardInit` | `cardno, slot` | Map VME address space for a card | Boot-only | `devGVME.c` |

`regaddr` is a 32-bit word offset into the board's VME address space.

### Diagnostics

| Command | Parameters | Description | Source |
|---------|-----------|-------------|--------|
| `debugGenReport` | `"cmd"` | Asyn diagnostic report for a board | `drvAsynDebug.c` |
| `devGDigSetRestFile` | `"path"` | Set register restore file path | `restoreSub.c` |

### State Machines (DAQ control)

Started via `seq &name` in boot script. Not normally called interactively.

| Symbol | Description |
|--------|-------------|
| `inLoop` | Readout control state machine (board enable/disable, CS logic) |
| `outLoop` | Data output state machine |
| `MiniSender` | TCP data sender (port 9001) |

---

## EPICS Base 3.14.12.1 Commands

### Database — Most Useful

| Command | Parameters | Description |
|---------|-----------|-------------|
| `dbl` | `[rectype] [fields]` | List all PVs, optionally filtered by record type and field names |
| `dbgrep` | `"pattern"` | Search PV names by wildcard pattern |
| `dbla` | `"pattern"` | Search PV aliases by pattern |
| `dbpr` | `"PV", level` | Print all fields of a record (level 0–3) |
| `dbgf` | `"PV.FIELD"` | Get a field value |
| `dbpf` | `"PV.FIELD", "val"` | Put a field value |
| `dbtr` | `"PV"` | Test-process a record manually |
| `dbior` | `"driver", level` | Driver I/O report |
| `dbstat` | — | Database statistics summary |
| `dbap` | `"PV"` | Show alarm state for a record |
| `dbcar` | `"PV", level` | Show CA links (causal analysis report) |
| `dblsr` | `"PV", level` | Scan list report |
| `dbLockShowLocked` | `level` | Show currently locked records |
| `dbnr` | `verbose` | List records without resolution |
| `dbhcr` | — | Show database hit count registers |
| `dbNotifyDump` | — | Dump pending notify requests |

### Database Schema / Static

| Command | Parameters | Description |
|---------|-----------|-------------|
| `dbLoadDatabase` | `"file", "path", "subs"` | Load database definition (.dbd) |
| `dbLoadRecords` | `"file", "subs"` | Load records from .db/.template with macro substitution |
| `dbLoadTemplate` | `"file"` | Load template file |
| `dbDumpRecord` | `pdbbase, "rectype", level` | Dump record type definition |
| `dbDumpMenu` | `pdbbase, "menu"` | Dump menu definition |
| `dbDumpField` | `pdbbase, "rectype", "field"` | Dump field definition |
| `dbDumpDevice` | `pdbbase, "rectype"` | Dump device support definitions |
| `dbDumpDriver` | `pdbbase` | Dump driver definitions |
| `dbDumpBreaktable` | `pdbbase, "table"` | Dump breakpoint table |
| `dbPvdDump` | `pdbbase, verbose` | Dump PV data table |
| `dbReportDeviceConfig` | `pdbbase` | Report device configuration |
| `dbPvdTableSize` | `size` | Set PV data table size |

### Breakpoints / Test

| Command | Parameters | Description |
|---------|-----------|-------------|
| `dbb` | `"PV"` | Set breakpoint on record |
| `dbd` | `"PV"` | Display/clear breakpoints |
| `dbc` | `"PV"` | Continue past breakpoint |
| `dbs` | `"PV"` | Show scan info at breakpoint |
| `dbp` | `"PV", level` | Print record at breakpoint |
| `dbtgf` | `"PV"` | Test get field |
| `dbtpf` | `"PV", "val"` | Test put field |
| `dbtpn` | `"PV", "val"` | Test put notify |
| `gft` | `"PV"` | Get field test |
| `pft` | `"PV", "val"` | Put field test |
| `tpn` | `"PV", "val"` | Test put notify shorthand |

### Scan

| Command | Parameters | Description |
|---------|-----------|-------------|
| `scanppl` | `rate` | Print periodic scan list for given rate (Hz) |
| `scanpel` | `event#` | Print event scan list for event number |
| `scanpiol` | — | Print I/O interrupt scan list |
| `scanOnceSetQueueSize` | `size` | Set scan-once queue size |
| `callbackSetQueueSize` | `size` | Set callback queue size |

### IOC Control

| Command | Parameters | Description |
|---------|-----------|-------------|
| `iocInit` | — | Initialize and start IOC (parse DB + begin record processing) |
| `iocBuild` | — | Parse database without starting record processing |
| `iocRun` | — | Resume record processing after `iocPause` |
| `iocPause` | — | Pause record processing |
| `coreRelease` | — | Print EPICS base version string |

### CA Server

| Command | Parameters | Description |
|---------|-----------|-------------|
| `casr` | `level` | CA server report: clients, channels, beacons (level 0–2) |

### Access Security

| Command | Parameters | Description |
|---------|-----------|-------------|
| `asSetFilename` | `"file"` | Set access security rules file |
| `asSetSubstitutions` | `"subs"` | Set AS substitution macros |
| `asInit` | — | Initialize access security |
| `asdbdump` | — | Dump access security database |
| `aspuag` | `"uag"` | Print user access group definition |
| `asphag` | `"hag"` | Print host access group definition |
| `asprules` | `"asg"` | Print ASG rules |
| `aspmem` | `"asg", clients` | Print ASG memory usage |
| `astac` | `"PV", "user", "host"` | Test access control for user@host on PV |
| `ascar` | `level` | AS callback analysis report |
| `asDumpHash` | — | Dump AS hash tables |

### Registry

| Command | Parameters | Description |
|---------|-----------|-------------|
| `registryDump` | — | Dump all registry entries |
| `registryRecordTypeFind` | `"name"` | Look up record type in registry |
| `registryDeviceSupportFind` | `"name"` | Look up device support in registry |
| `registryDriverSupportFind` | `"name"` | Look up driver support in registry |
| `registryFunctionFind` | `"name"` | Look up function in registry |

### Environment & System

| Command | Parameters | Description |
|---------|-----------|-------------|
| `epicsEnvShow` | `[name]` | Show one or all environment variables |
| `epicsEnvSet` | `"name", "val"` | Set environment variable |
| `epicsParamShow` | — | Show EPICS parameters |
| `epicsPrtEnvParams` | — | Print EPICS env parameters |
| `date` | `[format]` | Print current date/time |
| `cd` | `"dir"` | Change working directory |
| `pwd` | — | Print working directory |
| `errlog` | `"msg"` | Log an error message |
| `errlogInit` | `bufsize` | Initialize error log buffer |
| `errlogInit2` | `bufsize, maxMsgSize` | Initialize error log with max message size |
| `eltc` | `0/1` | Enable (1) or disable (0) error log to console |
| `setIocLogDisable` | `0/1` | Enable/disable IOC logging |
| `iocLogInit` | — | Initialize IOC logging |
| `iocLogShow` | `level` | Show IOC log |
| `generalTimeReport` | `level` | Time provider status report |
| `installLastResortEventProvider` | — | Install fallback event time provider |

### Threads & Concurrency

| Command | Parameters | Description |
|---------|-----------|-------------|
| `epicsThreadShowAll` | `level` | Show all threads with detail |
| `epicsThreadShow` | `thread` | Show one thread by name or ID |
| `epicsThreadSleep` | `seconds` | Sleep for N seconds |
| `epicsThreadResume` | `thread` | Resume a suspended thread |
| `epicsMutexShowAll` | `onlyLocked, level` | Show mutexes (filter to locked-only if onlyLocked=1) |
| `taskwdShow` | `level` | Task watchdog status |

---

## Asyn 4.17 Commands

| Command | Parameters | Description |
|---------|-----------|-------------|
| `asynReport` | `level, "port"` | Report asyn port status and queues |
| `asynSetTraceMask` | `"port", addr, mask` | Set trace logging mask |
| `asynSetTraceIOMask` | `"port", addr, mask` | Set I/O trace detail mask |
| `asynSetTraceFile` | `"port", addr, "file"` | Redirect trace output to file |
| `asynSetTraceIOTruncateSize` | `"port", addr, size` | Set trace I/O truncation size |
| `asynSetOption` | `"port", addr, "key", "val"` | Set port option |
| `asynShowOption` | `"port", addr, "key"` | Show port option value |
| `asynEnable` | `"port", addr, 0/1` | Enable (1) or disable (0) asyn port |
| `asynAutoConnect` | `"port", addr, 0/1` | Enable/disable auto-reconnect |
| `asynWaitConnect` | `"port", timeout` | Wait for port to connect (seconds) |
| `asynOctetConnect` | `"dev", "port", addr, timeout, buflen, "drvInfo"` | Connect octet device to port |
| `asynOctetDisconnect` | `"dev"` | Disconnect octet device |
| `asynOctetRead` | `"dev", maxbytes` | Read from octet device |
| `asynOctetWrite` | `"dev", "str"` | Write to octet device |
| `asynOctetWriteRead` | `"dev", "str", maxbytes` | Write then read |
| `asynOctetFlush` | `"dev"` | Flush octet device |
| `asynOctetSetInputEos` | `"port", addr, "eos"` | Set input end-of-string |
| `asynOctetGetInputEos` | `"port", addr` | Get input end-of-string |
| `asynOctetSetOutputEos` | `"port", addr, "eos"` | Set output end-of-string |
| `asynOctetGetOutputEos` | `"port", addr` | Get output end-of-string |

DGS asyn port names (from boot scripts): `MDIG1`, `MDIG2`, `RTR1`, `MTRG` (DUO/vme66 example).

---

## Safety Classification

### Always Safe (read-only, no side effects)
`dbl`, `dbgrep`, `dbla`, `dbpr`, `dbgf`, `dbcar`, `dbap`, `dbstat`, `dbior`, `dblsr`,
`dbLockShowLocked`, `casr`, `asynReport`, `epicsEnvShow`, `epicsParamShow`,
`epicsThreadShowAll`, `epicsThreadShow`, `epicsMutexShowAll`, `taskwdShow`,
`VMERead32`, `registryDump`, `pwd`, `date`, `coreRelease`, `generalTimeReport`,
`i`, `version`, `checkStack` (VxWorks built-ins)

### Use with Care (modifies state)
`dbpf` — writes to a live PV (triggers record processing)
`iocPause` / `iocRun` — stops/starts all record processing
`asynSetTraceMask` — changes logging verbosity
`epicsEnvSet` — modifies runtime environment
`epicsThreadSleep` — blocks the shell task

### Never Without Authorization (destructive / irreversible)
`ProgramFlash`, `EraseFlash`, `ConfigureFlash` — modifies FPGA firmware in flash
`VMEWrite32` — direct hardware register write
`iocInit` — re-initializes IOC (only valid once at boot)
`reboot`, `rebootLine` (VxWorks built-ins) — **erases boot settings if NVRAM is modified first**

---

## Terminal Server Map

From `ANLDAQ/EPICS_para.sh`:

| System | Terminal Server IP | Example Port | IOC Data IP(s) |
|--------|--------------------|--------------|----------------|
| DUO (tangerine/vme66) | 192.168.203.54 | 2001 | 192.168.203.81 |
| DXA | 192.168.203.47 | ? | 192.168.203.212, .213 |
| SlopeBox (vme99) | 192.168.203.139 | ? | — |
| DGS south | 192.168.203.186 | per crate | .141–.145, .177–.183 |
| DGS north | 192.168.203.91 | per crate | .141–.145, .177–.183 |

IOC IPs are for **data stream only** — do not telnet to them for shell access.

---

*Verified against source: `VxWorks/dgsDrivers/dgsDriverApp/src/`, `VxWorks/epics/base-3.14.12.1/src/`, `VxWorks/asyn*/`. Last updated: 2026-04-08.*
