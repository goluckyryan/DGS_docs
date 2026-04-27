# EPICS softIOC — dgsSoftIOC / JustGlobals.db

**Source:** `ANLDAQ/EPICS/softIOC/db/JustGlobals.db`, `dgsSoftIoc.cmd`  
**Location:** DFMA host (softIOC running on the DFMA Linux machine, distinct from the VxWorks VME IOCs)  
**Last edited:** 2022-07-11 (per file header, MBO)  
**Documented:** 2026-04-27  
Stability: C2 - Active / semi-stable

_Split from `EPICS_DB_templates.md` on 2026-04-27. Covers the Linux-hosted softIOC and the JustGlobals fanout PV architecture._

---

## Table of Contents

1. [Overview](#overview)
2. [Boot Script](#boot-script-dgssoftioccmd)
3. [JustGlobals.db — Purpose and Structure](#justglobalsdb--purpose-and-structure)
4. [GLBL PV Families (177 total)](#glbl-pv-families-177-total)
   - [Global (detector-type-agnostic) parameters](#global-detector-type-agnostic-parameters)
   - [Per-detector-type parameters](#per-detector-type-parameters)
5. [Key Design Notes](#key-design-notes)
6. [See Also](#see-also)

---

## Overview

The `dgsSoftIOC` is a standard EPICS base IOC (not VxWorks) that runs on the DFMA machine. It loads two databases:

1. `JustGlobals.db` — system-wide GLBL PV fanout trees (one per digitizer parameter)
2. `dgsSupport.db` — support PVs (scripts state machine, run control, etc.) — see `EPICS_implementation_tools.md` and `ANLDAQ.md`

This IOC is architecturally separate from the 12 VxWorks VME IOCs. It has no asyn drivers and no hardware links — purely software PVs that propagate values across all 12 VME crates via EPICS Channel Access.

---

## Boot Script (`dgsSoftIoc.cmd`)

```
dgsSoftIOC_registerRecordDeviceDriver pdbbase
dbLoadRecords "db/JustGlobals.db"
dbLoadRecords "db/dgsSupport.db"
iocInit
```

No asyn drivers, no hardware; purely software PVs propagating values across all 12 VME crates.

---

## JustGlobals.db — Purpose and Structure

`JustGlobals.db` implements the **EPICS fanout tree** mechanism: a single GUI-facing write PV (`GLBL:DIG:F00:<param>`) fans out a setting to all 12 VME crates simultaneously.

This is the standard DGS pattern for applying a single digitizer parameter value across all crates at once (e.g. setting `coarse_threshold` for all GeC channels system-wide).

**Pattern for each parameter:**

- Phase 0: `GLBL:DIG:F00:<param>` — entry point (GUI writes here)
- Phase 1: A chain of 12 `dfanout` records, `GLBL:DIG:01:<param>` through `GLBL:DIG:11:<param>`
  - Each record writes to the next relay and also to the per-crate actual PV: `VME<NN>:GLBL:<param>`
  - Chain terminates at `GLBL:DIG:11:<param>` which only writes to `VME12:GLBL:<param>`

**Result:** One caput to `GLBL:DIG:F00:<param>` propagates to `VME01:GLBL:<param>` through `VME12:GLBL:<param>` in a daisy-chain.

---

## GLBL PV Families (177 total)

✅ verified 2026-04-27 — `grep -c '"GLBL:DIG:F00:'` = 177; total dfanout records = 2124 = 177 × 12 (F00 + 11 relay nodes per param)

Parametrised by detector type prefix:
- **BGOs** — BGO shield (front), e.g. `BGOs_coarse_threshold`
- **BGOp** — BGO shield (rear/paired), e.g. `BGOp_coarse_threshold`
- **GeC** — Germanium crystal, e.g. `GeC_led_threshold`
- **GeS** — Germanium suppressed?, e.g. `GeS_pileup_mode`

### Global (detector-type-agnostic) parameters

| PV Suffix | Type | Notes |
|-----------|------|-------|
| `master_fifo_reset` | dfanout | Reset all FIFO across system |
| `win_comp_min` | dfanout | Coincidence window minimum (no PREC=3) | ✅ verified 2026-04-27 — JustGlobals.db: F00:win_comp_min has no PREC field
| `win_comp_max` | dfanout (PREC=3) | Coincidence window maximum | ✅ verified 2026-04-27 — JustGlobals.db: F00:win_comp_max has PREC=3
| `clk_select` | dfanout | Clock source selection |
| `EXT_DISC_REQ` | dfanout | External discriminator request |
| `trigger_mux_select` | dfanout | Trigger mux source selection |
| `ts_counter_mode` / `ts_counter_reset` | dfanout | Timestamp counter control |
| `master_counter_reset` | dfanout | Master counter reset |
| `master_logic_enable` | dfanout | Master logic enable/disable |
| `counter_mode` | dfanout | Discriminator counter mode |
| `veto_enable` | dfanout | Veto logic enable |
| `cfd_mode` | dfanout | CFD mode selection |
| `holdoff_time` | dfanout | Holdoff time |
| `peak_sensitivity` | dfanout | Peak detection sensitivity |
| `stop_ho_at_peak` | dfanout | Stop holdoff at peak |
| `FIFO_Prog_Thresh` | dfanout | FIFO programmable threshold |
| `load_delays` | dfanout | Load delay values |
| `diag_input` / `diag_input_en` | dfanout | Diagnostic input select/enable |
| `diag_mux_control` | dfanout | Diagnostic mux control |
| `DIAG_DISC_SEL` / `DIAG_WAVE_SEL` | dfanout | Diagnostic disc/wave channel select |
| `dac_channel_select` / `dac_attenuation` | dfanout | DAC channel and attenuation |
| `rj45_throttle_mode` | dfanout | RJ45 throttle mode |
| `lfsr_rate_sel` / `lfsr_seed` | dfanout | LFSR rate and seed |
| `sd_rx_pwr` / `sd_tx_pwr` / `sd_pem` | dfanout | SERDES power control |
| `sd_local_loopback_en` / `sd_line_loopback_en` | dfanout | SERDES loopback |
| `sd_sm_lost_lock_flag_rst` / `sd_sm_stringent_lock` / `sd_sync` | dfanout | SERDES sync/lock |
| `ext_disc_ts_sel` | dfanout | External discriminator timestamp select |

### Per-detector-type parameters

Each exists for BGOs/BGOp/GeC/GeS (4× total for each parameter):

| PV Suffix | Notes |
|-----------|-------|
| `*_coarse_threshold` (PREC=3) | Coarse discriminator threshold |
| `*_led_threshold` (PREC=3) | LED threshold |
| `*_CFD_fraction` (PREC=3) | CFD fraction |
| `*_coarse_width` | Coarse width |
| `*_disc_width` | Discriminator width |
| `*_trigger_polarity` | Trigger polarity |
| `*_channel_enable` | Per-channel enable |
| `*_pileup_mode` | Pileup mode |
| `*_pileup_waveform_only_mode` | Pileup waveform-only mode |
| `*_pileup_extension_enable` | Pileup extension enable |
| `*_event_extension_mode` | Event extension mode |
| `*_counter_reset` | Counter reset |
| `*_disc_count_mode` | Discriminator count mode |
| `*_ahit_count_mode` | Accepted-hit count mode |
| `*_event_count_mode` | Event count mode |
| `*_dropped_event_count_mode` | Dropped event count mode |
| `*_enable_dec_pause` | Enable decimate/pause |
| `*_write_flags` | Write flags |
| `*_downsample_factor` | Downsampling factor |
| `*_preamp_reset_delay_en` | Preamp reset delay enable |
| `*_preamp_reset_delay` (PREC=3) | Preamp reset delay |
| `*_raw_data_delay` | Raw data delay |
| `*_raw_data_length` | Raw data length |
| `*_d_window` | D-window |
| `*_k_window` | K-window |
| `*_k0_window` | K0-window |
| `*_m_window` | M-window |
| `*_d3_window` | D3-window |
| `*_p1_window` / `*_p2_window` | P1/P2 windows |
| `*_P2_mode` | P2 mode |
| `*_ext_disc_src` | External discriminator source |
| `*_ext_disc_sel` (PREC=3) | External discriminator select |
| `*_MultiplexWordSelect` | Multiplex word select |
| `*_Early_pre_m_sel` | Early pre-M select |

---

## Key Design Notes

- **`dfanout` record type**: EPICS built-in, drives up to 8 `OUTA..OUTH` links. Each step in the 12-crate chain uses `OUTA` (next relay) + `OUTB` (actual crate PV). All links use `PP NMS` (Process Passively, No Maximize Severity).
- **PREC=3**: Exactly 22 F00 records carry `PREC=3`: all 4 detector types × 5 per-type params (`coarse_threshold`, `led_threshold`, `CFD_fraction`, `ext_disc_sel`, `preamp_reset_delay`) = 20, plus `ext_disc_ts_sel` and `win_comp_max`. Note: `win_comp_min` does **not** have PREC=3. ✅ verified 2026-04-27 — python3 regex count on JustGlobals.db
- **VME12 terminus**: The last node (`GLBL:DIG:11:*`) only writes `OUTA` to `VME12:GLBL:*` (no next relay needed). ✅ verified 2026-04-27 — `grep -A5 'GLBL:DIG:11:clk_select'` shows only OUTA field; PREC=3 terminus records show OUTA + PREC only (no OUTB)
- **Crate vs board**: These GLBL PVs set the same value on all boards of the same detector type across all crates simultaneously. Per-channel adjustments use the individual `VME<NN>:<BOARD>:<param><ch>` PVs directly.
- **softIOC isolation**: The softIOC has no hardware I/O; all its PVs are pure EPICS channel-access software PVs that forward writes to the VME IOC PVs running on VxWorks.

---

## See Also

- `knowledgeBase/EPICS_DB_templates.md` — VME IOC template files (MDigUser, MTrigUser, dgsGlobals, etc.)
- `knowledgeBase/ioc.md` — per-crate VME IOC PVs that receive these forwarded values
- `knowledgeBase/ANLDAQ.md` — dgsSupport.db (setup scripts, run control) also loaded by dgsSoftIOC
- `knowledgeBase/EPICS_implementation_tools.md` — full DFMA machine description, EPICS env, GammaWare context
- `knowledgeBase/EPICS_asyn.md` — How asyn bridges EPICS CA to VME hardware registers (used by the VME IOCs receiving GLBL writes)

*Created: 2026-04-27. Split from EPICS_DB_templates.md.*
