# DGS Register Maps

_Created 2026-04-04. For lookups, use `grep` on the CSV files._

---

## Overview

The DGS register maps are stored as large tab-delimited CSV files generated from Excel spreadsheets with VBA macros. The same spreadsheets generate the EPICS `.db` files for the IOC.

**CSV locations:**
```
/home/ryan/DGS_tools_pack/DGS_docs/RegisterMaps/
  MasterDigitizerRegisterMap.csv    — DIG (988 entries)
  DGSMasterTriggerRegisterMap.csv   — MTRG (4656 entries)
  DGSRouterTriggerRegisterMap.csv   — RTRG (1112 entries)
  MDRM.csv                          — DIG (duplicate/alternate version)
```

**Spreadsheet sources (with VBA db generator):**
```
/home/ryan/DGS_tools_pack/DGS_docs/DGS_System_Documentation/Firmware/Digitizer/
  Dig reg map with vba.xls          — DIG register map + EPICS db generator
  Master Digitizer Register Map.xls
  Slave Digitizer Register Map.xls
  DGS_Digitizer.xlsx                — likely current version

/home/ryan/DGS_tools_pack/DGS_docs/DGS_System_Documentation/Firmware/Master_Trigger/DGS Master Trigger/
  DGSMasterTriggerRegisterMap.xls

/home/ryan/DGS_tools_pack/DGS_docs/DGS_System_Documentation/Firmware/Router/
  DGSRouterTriggerRegisterMap.xls
```

---

## CSV Format

Row 25 (data header): `Comment | Address | Register Mode | Register Name | Function | Access Type | Address | Bit | Field Mode | EPICS type | EPICS units | Database Group | Initial Value | Register Name | Software field name | Bitfield Sub-Descriptor | Human field name | Field Description for database | EPICS PV Fields`

Data starts at row 33. Lines starting with `#` or `%` are comments/metadata.

**EPICS Units convention:** `25;us` = 25 ns per LSB, display in µs. Plain `us` = scale from register time unit.

---

## DIG Register Groups (key registers)

| Register Name | Description |
|--------------|-------------|
| `cfd_mode` | LED or CFD mode select |
| `CFD_fraction0`–`9` | CFD threshold fraction per channel |
| `p1_window0`–`9` | Pre-rise integration window |
| `p2_window0`–`9` | Post-rise integration window |
| `m_window0`–`9` | M-sum integration window |
| `k_window0`–`9` | K delay window |
| `d_window0`–`9` | D delay window |
| `led_threshold0`–`9` | LED discriminator threshold |
| `raw_data_delay0`–`9` | Waveform capture delay |
| `raw_data_length0`–`9` | Waveform capture length |
| `trigger_polarity0`–`9` | Rise/fall edge trigger select |
| `baseline_delay` | Baseline holdoff after discriminator |
| `CS_Ena` | Channel select enable |
| `master_logic_enable` | Master logic enable/reset |
| `master_fifo_reset` | FIFO reset |
| `veto_enable` | Veto enable mask |
| `accepted_event_count0`–`9` | Per-channel accepted event counter |
| `ahit_count0`–`9` | Discriminator hit counter |
| `programming_done` | Firmware loaded flag |
| `channel_control0`–`9` | Per-channel control word |

---

## MTRG Register Groups (key registers)

| Register Name | Description |
|--------------|-------------|
| `CODE_DATE` / `CODE_REVISION` | Firmware build date and revision |
| `LOCK_BUS` | SERDES link lock status |
| `MSTR_MACH_STATE` | Master state machine state |
| `SYSTEM_THROTTLE_MAP` | Per-link throttle mask |
| `CHAN1_FIFO`–`CHAN8_FIFO` | Per-channel monitor FIFOs |
| `CHAN_FIFO_CTL` / `CHAN_FIFO_STATE` | FIFO control/status |
| `ASYNC_CMD_FIFO` | Frame 15 async command FIFO |
| `AUX_CMD_FIFO` | Frame 17 auxiliary command FIFO |
| `AUX_IO_CTL` / `AUX_IO_DATA` | Front panel aux I/O |
| `CPLD_MULT` / `CPLD_MASK` / `CPLD_EXTRA_REG` | CPLD multiplicity registers |
| `DiagnosticA` / `DiagnosticB` | Diagnostic register readback |
| `config_start` / `config_stop` | Configuration control |
| `DEN_BUS` | Data enable bus |
| `CONN_A_DATA`–`CONN_D_DATA` | Connector data readbacks |

---

## RTRG Register Groups

1,112 entries — covers per-link SERDES control, throttle limiters, multiplicity sums, VME register map. See `DGSRouterTriggerRegisterMap.csv` for full list.

Key registers:
| Register Name | Description |
|--------------|-------------|
| `CODE_DATE` / `CODE_REVISION` | Firmware build date and revision |
| `LOCK_BUS` | SERDES lock status |
| `THROTTLE` | Throttle control |
| `MULT_SUM` | Multiplicity sum |

---

## How to Look Up a Register

```bash
# Find a DIG register by name
grep -i "cfd_fraction" /home/ryan/DGS_tools_pack/DGS_docs/RegisterMaps/MasterDigitizerRegisterMap.csv

# Find a MTRG register address
grep -i "CODE_REVISION" /home/ryan/DGS_tools_pack/DGS_docs/RegisterMaps/DGSMasterTriggerRegisterMap.csv

# Find all registers at a given VME address (e.g. 0x015C)
grep "0x015C" /home/ryan/DGS_tools_pack/DGS_docs/RegisterMaps/DGSMasterTriggerRegisterMap.csv
```
