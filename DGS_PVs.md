# DGS PV Namespace — Pattern Reference

Stability: C2 - Active / semi-stable

Source: `DGS_tools_pack/ioc/db/*.template` + `boot/vme*.cmd`
EPICS CA: DGS 5064/5065 | DXA 5072/5073 | DUO 5080/5081
Total raw PVs: ~88,700 VME IOC PVs (DIG+RTR+MTRG) + ~158,000 Collector Box PVs = ~247,000 system-wide (see Section 6 summary)

## PV Naming Convention

```
VME{CC}:{BOARD}:{register}
```
- `CC` = crate number, 01–12 (zero-padded)
- `BOARD` = module type: MTRG, RTR1–RTR4, MDIG1/2, SDIG1/2
- `register` = register name (see sections below)
- `_RBV` suffix = read-back value (actual hardware state)
- Per-channel registers append `N` = channel index 0–9

**Crate→Board map (from DGS_SYSTEM_DEFINES.sh):**
- VME01–02, 04–05, 07–08, 11: MDIG1, MDIG2, SDIG1, SDIG2
- VME03: MDIG1, MDIG2, SDIG1, SDIG2, RTR1
- VME06: MDIG1, SDIG1, RTR2
- VME09: MDIG1, MDIG2, SDIG1, SDIG2, RTR3
- VME10: MDIG1, SDIG1, MTRG
- VME12: MDIG1, MDIG2, SDIG1, SDIG2, RTR4

---

## PV Nature Reference

Each PV table includes a **Nature** column describing the role of the PV:

| Nature | Meaning | Snapshot? |
|--------|---------|----------|
| `Config` | Writable setting — persists across restarts, user-facing physics config | ✅ Included |
| `Monitor` | Read-only hardware readback — temp, actual HV, counters, status, lock flags | ❌ Excluded |
| `Diag` | Diagnostic / debug / ILA / printf control — not physics config | ❌ Excluded |
| `Reset` | One-shot reset bit — dangerous to blindly restore | ❌ Excluded |
| `Global` | Global fanout PV — managed by IOC startup sequence, not snapshot | ❌ Excluded |
| `Register` | Raw hardware register (`reg_*`) — auto-updated by Config PV writes | ❌ Excluded |

The `snapshot_pv/pv_filter.py` script implements these exclusions automatically for both `dumpPVs.py` and `putPVs.py`.

---

## 1. Digitizer PVs — MDIG / SDIG (MDigUser.template)

Pattern: `VME{CC}:{MDIG1|SDIG1|...}:{register}`
~1,743 PVs per board instance (MDigUser: 1,368 + MDigRegisters: 359 + MDigRegistersVME: 6 + MDigUserVME: 10). MDIG and SDIG use parallel templates with identical record counts. ✅ verified 2026-04-19 — `record` count in each template + boot/vme99.cmd + boot/vme66.cmd

### 1a. Board-wide (single instance per board)

| Register | RBV? | Type | Nature | Description |
|----------|------|------|--------|-------------|
| `trigger_mux_select` | ✓ | mbbo | Config | Trigger source: 0=IntAcptAll, 1=ExtTTL, 2=ExtTTCL (normal), 3=Diag | ✅ verified 2026-04-19 — `MDigUser.template:L10572-10573`
| `master_logic_enable` | ✓ | bo | Global | Global fanout — handled by IOC startup |
| `master_fifo_reset` | — | bo | Reset | Reset output FIFO |
| `master_counter_reset` | ✓ | bo | Reset | Reset all counters |
| `vme_clk_ctrl` | ✓ | bo | Global | VME clock control — managed by IOC startup |
| `FifoNum` | ✓ | mbbo | Config | Readout FIFO select (soft channel) |
| `diag_isync` | ✓ | bo | Diag | Diagnostic use only (per DESC) |
| `sd_sync` | ✓ | bo | Config | SERDES sync mode enable |
| `sd_sm_lost_lock_flag_rst` | ✓ | bo | Reset | SERDES lost-lock flag reset |
| `sd_line_loopback_en` | ✓ | bo | Diag | SERDES line loopback (test mode) |
| `sd_local_loopback_en` | ✓ | bo | Diag | SERDES local loopback (test mode) |
| `ts_counter_reset` | ✓ | bo | Reset timestamp counter |
| `ts_counter_mode` | ✓ | mbbo | Timestamp counter mode |
| `latch_timestamp` | — | bo | Latch current timestamp |
| `lat_timestamp_high_RBV` | — | longin | Latched TS high word |
| `lat_timestamp_low_RBV` | — | longin | Latched TS low word |
| `live_timestamp_lsb_RBV` | — | longin | Live timestamp LSB |
| `live_timestamp_msb_RBV` | — | longin | Live timestamp MSB |
| `CV_LiveTS` | — | longin | Live timestamp (combined) |
| `cfd_mode` | ✓ | bo | CFD mode (board-wide) | ✅ verified 2026-04-21 — `MDigUser.template:L6930`
| `veto_enable` | ✓ | bo | Enable veto input |
| `veto_gate_width` | ✓ | longout | Veto gate width | ✅ verified 2026-04-21 — `MDigUser.template:L11254`
| `counter_inhibit` | ✓ | bo | Inhibit all counters |
| `counter_mode` | ✓ | bo | Counter mode | ✅ verified 2026-04-21 — `MDigUser.template:L6890`
| `FIFO_Prog_Thresh` | ✓ | ao | FIFO programmable threshold |
| `win_comp_min` | ✓ | ao | Window comparator minimum |
| `win_comp_max` | ✓ | ao | Window comparator maximum |
| `dac_attenuation` | ✓ | ao | DAC attenuation value |
| `dac_channel_select` | ✓ | mbbo | DAC channel select |
| `holdoff_time` | ✓ | ao | Global holdoff time |
| `delay` | ✓ | ao | Global delay |
| `peak_sensitivity` | ✓ | ao | Peak sensitivity |
| `tracking_speed` | ✓ | longout | Phase tracking speed | ✅ verified 2026-04-21 — `MDigUser.template:L11219`
| `stop_ho_at_peak` | ✓ | bo | Stop holdoff at peak |
| `downsample_holdoff` | ✓ | longout | Downsample holdoff | ✅ verified 2026-04-21 — `MDigUser.template:L11247`
| `dc_balance_enable` | ✓ | bo | DC balance enable |
| `phase_hunt` | — | bo | Start phase hunt |
| `load_baseline` | — | bo | Load baseline |
| `load_delays` | — | bo | Load delay values |
| `BGO_discbit_select` | ✓ | bo | BGO discriminator bit select | ✅ verified 2026-04-21 — `MDigUser.template:L6910`
| `rj45_discbit_mode` | ✓ | mbbo | RJ45 discbit mode |
| `rj45_throttle_mode` | ✓ | mbbo | RJ45 throttle mode |
| `aux_output_mode` | ✓ | mbbo | Aux output mode |
| `ext_disc_ts_sel` | ✓ | mbbo | Ext discriminator timestamp select |
| `EXT_DISC_REQ` | — | bo | Request ext disc |
| `RJ45_TEST` | — | bo | RJ45 test mode |
| `TEST_RESET_ENABLE` | ✓ | bo | Test reset enable |
| `LFSR_LOAD` | — | bo | Load LFSR |
| `lfsr_rate_sel` | ✓ | mbbo | LFSR rate select |
| `lfsr_seed` | ✓ | ao | LFSR seed value |
| `manual_data` | ✓ | ao | Manual data value |
| `user_package_data` | ✓ | ao | User package data |
| `diag_mux_control` | ✓ | mbbo | Diagnostic mux control |
| `diag_input` | ✓ | ao | Diagnostic input |
| `diag_input_en` | ✓ | bo | Diagnostic input enable |
| `diag_isync` | ✓ | bo | Diagnostic isync |
| `DIAG_DISC_SEL` | ✓ | bo | Diagnostic discriminator bit select | ✅ verified 2026-04-21 — `MDigUser.template:L6770`
| `DIAG_WAVE_SEL` | ✓ | mbbo | Diagnostic waveform select |
| `sd_sync` | ✓ | bo | SERDES sync |
| `sd_pem` | ✓ | mbbo | SERDES PEM mode |
| `sd_tx_pwr` | ✓ | bo | SERDES TX power | ✅ verified 2026-04-21 — `MDigUser.template:L6990`
| `sd_rx_pwr` | ✓ | bo | SERDES RX power | ✅ verified 2026-04-21 — `MDigUser.template:L6970`
| `sd_sm_lost_lock_flag_rst` | ✓ | bo | SERDES SM lost lock flag reset |
| `sd_sm_stringent_lock` | ✓ | bo | SERDES stringent lock mode |
| `sd_line_loopback_en` | ✓ | bo | SERDES line loopback enable |
| `sd_local_loopback_en` | ✓ | bo | SERDES local loopback enable |
| `code_date_RBV` | — | stringin | Firmware compile date |
| `fw_type_RBV` | — | mbbi | Firmware type |
| `mjr_code_revision_RBV` | — | longin | Major firmware revision |
| `min_code_revision_RBV` | — | longin | Minor firmware revision |
| `pcb_revision_RBV` | — | longin | PCB revision |
| `geo_addr_RBV` | — | longin | Geographic VME address |
| `status_RBV` | — | longin | Board status register |
| `serdes_lock_RBV` | — | bi | SERDES locked |
| `serdes_sm_locked_RBV` | — | bi | SERDES SM locked |
| `serdes_sm_lost_lock_flag_RBV` | — | bi | SERDES SM lost lock flag |
| `serdes_phase_value_RBV` | — | longin | SERDES phase value |
| `fbus_serdes_sm_locked_RBV` | — | bi | Front-bus SERDES locked |
| `fbus_throttle_RBV` | — | bi | Front-bus throttle active |
| `phase_value_RBV` | — | longin | Phase value |
| `phase_errors_RBV` | — | longin | Phase error count |
| `ph_checking_RBV` | — | bi | Phase checking active |
| `ph_hunting_up_RBV` | — | bi | Phase hunting up |
| `ph_hunting_down_RBV` | — | bi | Phase hunting down |
| `ph_success_RBV` | — | bi | Phase success |
| `ph_failure_RBV` | — | bi | Phase failure |
| `ts_err_count_RBV` | — | longin | Timestamp error count |
| `PU_TIME_ERR_RBV` | — | bi | Pileup time error |
| `int_FIFO_PROG_ERR_RBV` | — | bi | Internal FIFO prog error |
| `int_FIFO_PROG_FLG_RBV` | — | bi | Internal FIFO prog flag |
| `int_fifo_prog_flag_RBV` | — | bi | Internal FIFO prog flag (alt) |
| `fifo_almost_full_RBV` | — | bi | FIFO almost full |
| `fifo_almost_empty_RBV` | — | bi | FIFO almost empty |
| `fifo_half_full_RBV` | — | bi | FIFO half full |
| `fifo_fulla_RBV` | — | bi | FIFO A full |
| `fifo_fullb_RBV` | — | bi | FIFO B full |
| `fifo_emptya_RBV` | — | bi | FIFO A empty |
| `fifo_emptyb_RBV` | — | bi | FIFO B empty |
| `fifo_depth_RBV` | — | longin | FIFO depth |
| `acq_dcm_lock_RBV` | — | bi | Acq DCM locked |
| `acq_dcm_reset_RBV` | — | bi | Acq DCM reset |
| `acq_dcm_clock_stopped_RBV` | — | bi | Acq DCM clock stopped |
| `acq_dcm_ctrl_status_RBV` | — | longin | Acq DCM control status |
| `acq_ph_shift_overflow_RBV` | — | bi | Acq phase shift overflow |
| `adc_dcm_lock_RBV` | — | bi | ADC DCM locked |
| `adc_dcm_reset_RBV` | — | bi | ADC DCM reset |
| `adc_dcm_clock_stopped_RBV` | — | bi | ADC DCM clock stopped |
| `adc_dcm_ctrl_status_RBV` | — | longin | ADC DCM control status |
| `adc_ph_shift_overflow_RBV` | — | bi | ADC phase shift overflow |

### 1b. Per-Channel Registers (append N = 0–9)

Pattern: `VME{CC}:{BOARD}:{register}N` and `VME{CC}:{BOARD}:{register}N_RBV`

| Register | RBV? | Type | Nature | Description |
|----------|------|------|--------|-------------|
| `channel_enableN` | ✓ | bo | Config | Enable channel N |
| `coarse_thresholdN` | ✓ | ao | Config | Coarse discriminator threshold |
| `led_thresholdN` | ✓ | ao | Config | LED (low-energy) threshold |
| `CFD_fractionN` | ✓ | ao | Config | CFD fraction (0–1 scale) |
| `preamp_reset_delayN` | ✓ | ao | Config | Preamp reset delay |
| `preamp_reset_delay_enN` | ✓ | bo | Config | Enable preamp reset delay |
| `disc_widthN` | ✓ | ao | Config | Discriminator width |
| `coarse_widthN` | ✓ | ao | Config | Coarse discriminator width |
| `trigger_polarityN` | ✓ | bo | Config | Trigger polarity (0=pos, 1=neg) |
| `pileup_modeN` | ✓ | mbbo | Config | Pileup handling mode |
| `pileup_extension_enableN` | ✓ | bo | Config | Enable pileup extension |
| `pileup_waveform_only_modeN` | ✓ | bo | Config | Pileup waveform-only mode |
| `event_extension_modeN` | ✓ | mbbo | Config | Event extension mode |
| `enable_dec_pauseN` | ✓ | bo | Config | Enable decimation pause |
| `downsample_factorN` | ✓ | mbbo | Config | Downsampling factor |
| `write_flagsN` | ✓ | mbbo | Config | Write flags (event fields to include) |
| `raw_data_delayN` | ✓ | ao | Config | Raw waveform delay (samples) |
| `raw_data_lengthN` | ✓ | ao | Config | Raw waveform length (samples) |
| `k_windowN` | ✓ | ao | Config | K window (energy filter) |
| `k0_windowN` | ✓ | ao | Config | K0 window (baseline) |
| `m_windowN` | ✓ | ao | Config | M window (peaking time) |
| `d_windowN` | ✓ | ao | Config | D window (gap) |
| `d3_windowN` | ✓ | ao | Config | D3 window |
| `p1_windowN` | ✓ | ao | Config | P1 window |
| `p2_windowN` | ✓ | ao | Config | P2 window |
| `P2_modeN` | ✓ | mbbo | Config | P2 mode |
| `trig_ts_modeN` | ✓ | mbbo | Config | Trigger timestamp mode |
| `ahit_count_modeN` | ✓ | mbbo | Config | Accepted hit counter mode |
| `ahit_countN_RBV` | — | longin | Monitor | Accepted hit count readback |
| `disc_count_modeN` | ✓ | mbbo | Config | Discriminator counter mode |
| `disc_countN_RBV` | — | longin | Monitor | Discriminator count readback |
| `event_count_modeN` | ✓ | mbbo | Config | Event counter mode |
| `accepted_event_countN_RBV` | — | longin | Monitor | Accepted event count readback |
| `dropped_event_count_modeN` | ✓ | mbbo | Config | Dropped event counter mode |
| `dropped_event_countN_RBV` | — | longin | Monitor | Dropped event count readback |
| `hilo_count_modeN` | ✓ | mbbo | Config | Hi-lo counter mode |
| `hihilolo_count_modeN` | ✓ | mbbo | Config | HiHi-LoLo counter mode |
| `HILO_EDGE_LEVEL_SELN` | ✓ | mbbo | Config | Hi-lo edge/level select |
| `counter_resetN` | ✓ | bo | Reset | Reset this channel counter |
| `overflow_flag_chanN_RBV` | — | bi | Monitor | Overflow flag for channel N |
| `MultiplexWordSelectN` | ✓ | mbbo | Config | Multiplex word select |
| `cfd_esum_modeN` | ✓ | mbbo | Config | CFD energy sum mode |
| `ext_disc_selN` | ✓ | mbbo | Config | External discriminator select |
| `ext_disc_srcN` | ✓ | mbbo | Config | External discriminator source |
| `Early_pre_m_selN` | ✓ | mbbo | Config | Early pre-M select (timing) |
| `MARK_EXTENDED_AS_ACCEPTEDN` | ✓ | bo | Config | Mark extended events as accepted |
| `Phase_offsetN_RBV` | — | longin | Monitor | Phase offset readback |
| `led_green_stateN_RBV` | — | bi | Monitor | Green LED state |
| `led_red_stateN_RBV` | — | bi | Monitor | Red LED state |

---

## 2. RTRG PVs — RTR1–RTR4 (RTrigUser.template)

Pattern: `VME{CC}:{RTR1|RTR2|RTR3|RTR4}:{register}`
~1,080 PVs per RTRG instance (RTrigUser: 897 + RTrigRegisters: 183). ✅ verified 2026-04-19 — `record` count in each template + boot/vme66.cmd
RTRG boards: VME03=RTR1, VME06=RTR2, VME09=RTR3, VME12=RTR4

### 2a. Board-wide Controls

| Register | RBV? | Nature | Description |
|----------|------|--------|-------------|
| `Threshold` | ✓ | Config | Global multiplicity threshold |
| `ASSERTION_DELAY` | ✓ | Config | Trigger assertion delay |
| `OVERLAP_DELAY` | ✓ | Config | Overlap delay |
| `PULSE_COUNTDOWN` | ✓ | Config | Pulse countdown |
| `THROTTLE_WIDTH` | ✓ | Config | Throttle pulse width |
| `THROTTLE_FILTER_TIME` | ✓ | Config | Throttle filter time |
| `THROTTLE_TIME_RANGE` | ✓ | Config | Throttle time range |
| `NIM_THROTTLE_SELECT` | ✓ | Config | NIM throttle select |
| `ENABLE_VETO` | ✓ | Config | Enable veto input |
| `FORCE_THRTL_ON` | — | Diag | Force throttle on (test) |
| `FORCE_THRTL_OFF` | — | Diag | Force throttle off (test) |
| `ENBL_DISCBIT_DELAY` | ✓ | Config | Enable discriminator bit delay |
| `LOAD_DISCBIT_DELAYS` | — | Reset | Load discriminator bit delays (one-shot) |
| `BIT_5_OFFSET` | ✓ | Config | Bit 5 offset |
| `DIAG_THROTTLE_TYPE` | ✓ | Diag | Diagnostic throttle type |
| `CLEAR_DIAG_COUNTERS` | — | Reset | Clear diagnostic counters |
| `X_SELECT` | ✓ | Config | X multiplicity select |
| `Y_SELECT` | ✓ | Config | Y multiplicity select |
| `FS_SEL` | ✓ | Config | Fast strobe select |
| `ClkSrc` | ✓ | Config | Clock source |
| `LEDControl` | ✓ | Config | LED control |
| `NIMSrc1` | ✓ | Config | NIM output source 1 |
| `NIMSrc2` | ✓ | Config | NIM output source 2 |
| `PEHLRU` | ✓ | Config | PE mask H/L/R/U links |
| `PEEFG` | ✓ | Config | PE mask E/F/G links |
| `PEABCD` | ✓ | Config | PE mask A/B/C/D links |
| `RESET_LINK_INIT` | — | Reset | Reset link initialization |
| `SM_LOST_LOCK_RESET` | — | Reset | SM lost lock reset |
| `STRINGENT_LOCK` | ✓ | Config | Stringent lock mode |
| `LOCK_ACK` | — | Reset | Lock acknowledge (one-shot) |
| `LOCK_RETRY` | — | Reset | Lock retry (one-shot) |

### 2b. Per-Link Registers (suffix = link letter A–H, L, R, U)

| Register | RBV? | Nature | Description |
|----------|------|-------------|
| `LINK_{x}` | ✓ | Enable link x |
| `ILM_{x}` | ✓ | Inhibit link mask for link x |
| `RPwr_{x}` | ✓ | RX power for link x |
| `TPwr_{x}` | ✓ | TX power for link x |
| `SLiL_{x}` | ✓ | SERDES line loopback for link x |
| `SLoL_{x}` | ✓ | SERDES local loopback for link x |
| `LostLockRst{L簊R簊U}` | — | Lost lock reset for L/R/U links |
| `LRUCtl{NN}` | ✓ | L/R/U control word |
| `conn_{a–d}_mask` | ✓ | Connector A–D discriminator mask |
| `{A–H}_3_0_DIR` | ✓ | Link direction bits 3–0 |
| `{A–H}_7_4_DIR` | ✓ | Link direction bits 7–4 |
| `Link_A-H_R_U_TX_DCBAL` | ✓ | TX DC balance for A–H/R/U |
| `LinkL_DCbal` | ✓ | DC balance for Link L |
| `PrE_{0–2}` | ✓ | Pre-emphasis settings |
| `FIFOReset{NN}` | — | Reset FIFO NN (00–15) |
| `LED{4–12}` | ✔ | LED control bits 4–12 |
| `CF{1–8}_MODE` | ✓ | Crate filter mode for crate 1–8 |
| `CF{1–8}_MODE_WE` | ✓ | Crate filter mode write-enable |

### 2c. Per-Link Per-Channel Registers (link A–H, channel 0–9)

| Register | RBV? | Nature | Description |
|----------|------|-------------|
| `DISCRIMINATOR_DELAY_{x}_N` | ✓ | Discrim delay for link x, ch N |
| `XMAP_{x}_N` | ✓ | X-multiplicity map for link x, ch N |
| `YMAP_{x}_N` | ✓ | Y-multiplicity map for link x, ch N |

---

## 3. MTRG PVs (MTrigUser.template)

Pattern: `VME10:MTRG:{register}` (single MTRG in VME10)
~7,711 PVs total (mostly RAM table entries) ✅ verified 2026-04-19 — `record` count in MTrigUser.template + MTrigRegisters.template + boot/vme99.cmd

### 3a. Trigger / Threshold Controls

| Register | RBV? | Nature | Description |
|----------|------|-------------|
| `Threshold` | ✓ | Global multiplicity threshold |
| `COINC_OVERLAP_DELAY` | ✓ | Coincidence overlap delay |
| `L_OVERLAP_DELAY` | ✓ | Link L overlap delay |
| `R_OVERLAP_DELAY` | ✓ | Link R overlap delay |
| `U_OVERLAP_DELAY` | ✓ | Link U overlap delay |
| `MYRIAD_OVERLAP_DELAY` | ✓ | MγRIAD overlap delay |
| `EN_SUM_X` | ✓ | Enable X-sum trigger |
| `EN_SUM_Y` | ✓ | Enable Y-sum trigger |
| `EN_SUM_XY` | ✓ | Enable XY coincidence trigger |
| `ALGO_5_SELECT` | ✓ | Algorithm 5 select |
| `EN_ALGO5` | ✓ | Enable algorithm 5 |
| `L_COINC_MASK` | ✓ | Link L coincidence mask |
| `R_COINC_MASK` | ✓ | Link R coincidence mask |
| `U_COINC_MASK` | ✓ | Link U coincidence mask |
| `MANUAL_TRIGGER` | — | Issue manual trigger |
| `SOFTWARE_VETO` | ✓ | Software veto |
| `ENBL_SYNC_RESET` | ✓ | Enable sync reset |
| `ENBL_THROTTLE_VETO` | ✓ | Enable throttle veto |
| `ENBL_NIM_VETO` | ✓ | Enable NIM veto |
| `ENBL_MON7_VETO` | ✓ | Enable monitor-7 veto |
| `EN_RAM_VETO` | ✓ | Enable RAM-based veto |
| `EN_REM_MSTR_VETO` | ✓ | Enable remote master veto |
| `EN_NIM_AUX` | ✓ | Enable NIM auxiliary trigger input |
| `EN_MAN_AUX` | ✓ | Enable manual auxiliary trigger |
| `AUXPolaritySelect` | ✓ | Auxiliary trigger polarity |
| `AuxTrig_Width` | ✓ | Auxiliary trigger width |
| `EN_DATA_ALWAYS` | ✓ | Enable data always mode |
| `TRIG_A_PRESCALE_ENBL` | ✓ | Enable prescale for RTR A |
| `TRIG_A_PRESCALE_FACTOR` | ✓ | Prescale factor for RTR A |
| `TRIG_{A–H}_PRESCALE_ENBL` | ✓ | (pattern, one per RTR A–H) |
| `TRIG_{A–H}_PRESCALE_FACTOR` | ✓ | (pattern, one per RTR A–H) |

### 3b. TAC-II / TDC Controls

| Register | RBV? | Nature | Description |
|----------|------|-------------|
| `NUM_TDC_WORDS` | ✓ | Number of TDC words per event |
| `NUM_TRIG_WORDS` | ✓ | Number of trigger words |
| `SKIP_TDC_DATA` | ✓ | Skip TDC data |
| `SEND_FRAME_12` | ✔ | Send frame 12 (TAC start) |
| `SEND_FRAME_14` | ✔ | Send frame 14 |
| `SEND_FRAME_16` | ✔ | Send frame 16 |
| `SEND_FRAME_17` | ✔ | Send frame 17 |
| `RST_F12_COUNT` | — | Reset frame 12 counter |
| `RST_F14_COUNT` | — | Reset frame 14 counter |
| `RST_F16_COUNT` | — | Reset frame 16 counter |
| `RST_F17_COUNT` | — | Reset frame 17 counter |
| `FRAME2_SAMP_CTL` | ✓ | Frame 2 sample control |
| `TS_SAMP_PHASE` | ✓ | Timestamp sample phase |
| `MANUAL_LATCH_TIMESTAMP` | — | Manually latch timestamp |

### 3c. Link Controls (per-link L/R/U or A–H)

| Register | RBV? | Nature | Description |
|----------|------|-------------|
| `EN_LINK_L` / `EN_LINK_R` | ✓ | Enable Link L/R |
| `EN_MYRIAD_LINK_U` | ✓ | Enable MγRIAD Link U |
| `LINK_L_CMD_FORMAT` | ✓ | Link L command format |
| `LINK_L_IS_TRIGGER_TYPE` | ✓ | Link L is trigger type |
| `LINK_L_PROPAGATE_F{N}` | ✓ | Propagate frame N on Link L |
| `LINK_R_PROPAGATE_F{N}` | ✓ | Propagate frame N on Link R |
| `LINK_U_PROPAGATE_F{N}` | ✓ | Propagate frame N on Link U |
| `LINK_{L簊R簊U}_STRINGENT` | ✓ | Stringent lock mode |
| `L_RT_TS_MODE` / `R_RT_TS_MODE` / `U_RT_TS_MODE` | ✓ | Real-time TS mode |
| `MYR_TS_MODE` | ✓ | MγRIAD timestamp mode |
| `MYR_TRIGGER_TYPE_SELECT` | ✓ | MγRIAD trigger type select |
| `MYR_OTHER_TRIG_MASK` | ✓ | MγRIAD other trigger mask |
| `ILM_{A–H,L,R,U}` | ✓ | Inhibit link mask per link |
| `RPwr_{x}` / `TPwr_{x}` | ✓ | RX/TX power per link |
| `SLiL_{x}` / `SLoL_{x}` | ✓ | SERDES loopback per link |
| `EN_{x}_TX_DCBAL` | ✓ | TX DC balance per link |
| `XLM_{A–H}` / `YLM_{A–H}` | ✓ | X/Y link mask per RTR link |
| `EN_NIM_VETO_{A–H}` | ✓ | NIM veto enable per RTR |
| `EN_RAM_VETO_{A–H}` | ✓ | RAM veto enable per RTR |
| `EN_THROTTLE_VETO_{A–H}` | ✓ | Throttle veto per RTR |
| `EN_REMTRIG_VETO_{A–H}` | ✓ | Remote trigger veto per RTR |
| `EN_TRIG_HOLDOFF_{A–H}` | ✓ | Trigger hold-off per RTR |
| `Rtr{1–8}ThrottleReq` | ✓ | Throttle request to RTR 1–8 |

### 3d. Trigger Monitor / Rate Counters

| Register | RBV? | Nature | Description |
|----------|------|-------------|
| `TRIG_MON_SEL` | ✓ | Trigger monitor select |
| `TRIG_MON_COLL_RESET` | — | Reset trigger monitor collector |
| `TRIGMON_ENBL_ACK{1–8}` | ✓ | Enable trigger monitor ACK 1–8 |
| `TRIGMON_ENBL_TEST` | ✓ | Enable trigger monitor test |
| `TRIGMON_ENBL_VME_CLK` | ✓ | Enable VME clock monitor |
| `Trigger_rate_counter_mode` | ✓ | Trigger rate counter mode |
| `Rate_clock_source_select` | ✓ | Rate clock source |
| `CLEAR_RATE_COUNTERS` | — | Clear rate counters |
| `CLEAR_ENCODER_CNTR` | — | Clear encoder counter |
| `CLEAR_DIAG_COUNTERS` | — | Clear diagnostic counters |
| `COUNTER_ROLL_999` | ✓ | Counter roll at 999 mode |
| `F12_RESET_CNTR{1–5}` | ✓ | Frame 12 reset counter 1–5 |
| `FM7Sel15` | ✓ | Frame monitor 7 select 15 |

### 3e. RAM Tables (pattern — bulk PV count)

| Pattern | Count | Description |
|---------|-------|-------------|
| `VETO_RAM_O{A–D}_B{0–15}` | 64 | Output veto RAM (odd rows A–D, bits 0–15) |
| `VETO_RAM_P{A–D}_B{0–15}` | 64 | Output veto RAM (pair rows A–D) |
| `TRIG_RAM_*` | ~200 | Trigger decision RAM |
| `SWEEP_RAM_*` | ~200 | Sweep RAM |
| `CF{0–8}_F12RESET` | 9 | Crate filter frame-12 reset |
| `MF{0–8}_F12RESET` | 9 | Module filter frame-12 reset |
| `CFIFO{1–8}_FORCE` | 8 | Force crate FIFO |
| `COINC_TRIG_MASK_A/B{1–8}` | 16 | Coincidence trigger mask |
| `FIFOReset{00–15}` | 16 | FIFO reset 00–15 |

---

## 4. Collector Box PVs (CollectorBox_RevA/db/)

Pattern: `GS{N}_{register}` where N = GS hole number (1–110)
EPICS CA port: part of collector box soft IOC on each Raspberry Pi
~1,431 PVs per detector ✅ verified 2026-04-16 (see `knowledgeBase/collectorbox_PVs.md` for full list)

### 4a. SlopeBox / Detector Identification (SlopeBox.db)

| Register | Nature | Description |
|----------|-------------|
| `GS{N}_SBX_Present` | SBX dongle present flag |
| `GS{N}_Dig_Channel` | Digitizer channel (0–9) this detector is connected to |
| `GS{N}_Dig_Index` | Digitizer board index |
| `GS{N}_VME_Index` | VME crate index |
| `GS{N}_Ge_ID` | Germanium detector serial ID |
| `GS{N}_Ge_Type` | Ge detector type |
| `GS{N}_Ge_Prefix` | Ge PV prefix |
| `GS{N}_Ge_MCA_Resolution` | MCA resolution |
| `GS{N}_Ge_MCA_GR` | MCA gain ratio |
| `GS{N}_Ge_MCA_Depletion_Voltage` | Depletion voltage |
| `GS{N}_Ge_MCA_Reset_Period` | Reset period (ms) |
| `GS{N}_Pi_ID` | Raspberry Pi ID for this detector |
| `True_GS{N}_to_VME_GS` | Mapping: true GS hole → VME_GS number |
| `VME_GS{M}_to_True_GS` | Mapping: VME_GS → true GS hole |

### 4b. HV Controls (HV_STEP.db + Pickoff.db)

| Register | Nature | Description |
|----------|-------------|
| `MOD{N}_DS_GEHV` | Ge HV demand setpoint (from IOC config) |
| `MOD{N}_DV_GEHV` | Ge HV demand readback |
| `GS{N}_GE_HV_DEMAND_VOLTS` | Ge HV demand in volts |
| `GS{N}_GE_HV_STEP_SIZE` | HV ramp step size |
| `GS{N}_GE_HV_HYSTERESIS` | HV hysteresis |
| `GS{N}_GE_HV_ABSMAX` | Absolute maximum HV limit |
| `GS{N}_MANUAL_GE_HV_DEMAND` | Manual HV demand override |
| `GS{N}_DIRECT_MANUAL_GE_HV_DEMAND` | Direct manual HV demand |
| `GS{N}_GE_HV_CTRL` | Ge HV control word |
| `GS{N}_CHAINED_GE_HV_DEMAND` | Chained HV demand (from SBX) |
| `GS{N}_BGO_HV{0–13}` | BGO HV for each segment 0–13 |
| `GS{N}_BGO_HV_CTRL` | BGO HV control word |
| `GS{N}_SlopeBoxBGO_HV_On` | BGO HV on flag |
| `GS{N}_SlopeBoxBGOInterlock` | BGO HV interlock flag |
| `GS{N}_SlopeBoxGe_HV_On` | Ge HV on flag |

### 4c. Temperature / Power Monitoring (SlopeBox.db + Pickoff.db)

| Register | Nature | Description |
|----------|-------------|
| `GS{N}_Conv_Temp` | Converted temperature |
| `GS{N}_Conv_Resistance` | Converted resistance (PT100/PT500) |
| `GS{N}_Conv_24V` | Converted 24V rail |
| `GS{N}_Conv_5V` | Converted 5V rail |
| `GS{N}_Conv_plus12V` / `Conv_minus12V` | +/-12V rails |
| `GS{N}_Conv_GeHV` | Converted Ge HV readback |
| `GS{N}_Conv_BGO400` / `Conv_BGO450` | BGO HV readback (400V/450V range) |
| `GS{N}_SlopeBoxTemperatureRaw` | SBX temperature raw ADC |
| `GS{N}_SlopeBoxTempHigh` | SBX temp high alarm |
| `GS{N}_LocalTempHigh` / `LocalTempLow` | Local (Pi PCB) temp limits |
| `GS{N}_RemoteTempHigh` / `RemoteTempLow` | Remote (collector box) temp limits |
| `GS{N}_RemoteDiodeError` | Remote diode error flag |
| `GS{N}_FanFault` | Fan fault flag |

### 4d. Ctrl FPGA (CtrlFPGA.db)

| Register | Nature | Description |
|----------|-------------|
| `GS{N}_ctl_master_reset` | Master reset |
| `GS{N}_ctl_serial_reset` | Serial reset |
| `GS{N}_ctl_reset_startup_rom` | Reset startup ROM |
| `GS{N}_ctl_reset_cmd_fifo` | Reset command FIFO |
| `GS{N}_ctl_transactor_go` | Start transactor |
| `GS{N}_ADC_scanner_reset` | ADC scanner reset |
| `GS{N}_ADC_transactor_reset` | ADC transactor reset |
| `GS{N}_ADC_transactor_fifo_rst` | ADC transactor FIFO reset |
| `GS{N}_MON_ADC_RESET` | Monitor ADC reset |
| `GS{N}_CTL_FPGA_LED` | Control FPGA LED |
| `GS{N}_NIM_OUT1` / `NIM_OUT2` | NIM output 1/2 |
| `GS{N}_DPRAM_Banksel` | DPRAM bank select |

### 4e. BGO Pickoff (Pickoff.db — key registers)

| Register | Nature | Description |
|----------|-------------|
| `GS{N}_BGO_Threshold` | BGO discriminator threshold |
| `GS{N}_BGOMultiplicityThresh` | BGO multiplicity threshold |
| `GS{N}_BGO_DiscbitMask` | BGO discbit mask |
| `GS{N}_BGO_DiscbitCntMode` | BGO discbit count mode |
| `GS{N}_BGOSumAttenuation` | BGO sum signal attenuation |
| `GS{N}_BGOpSelect` | BGO pattern select |
| `GS{N}_BGOpMuxMode` | BGO pattern mux mode |
| `GS{N}_BGOpStaticAddress` | BGO pattern static address |
| `GS{N}_BGOpSelectionDwell` | BGO pattern selection dwell |
| `GS{N}_BGOpattern_DCOffset` | BGO pattern DC offset |
| `GS{N}_BGOsum_DCOffset` | BGO sum DC offset |
| `GS{N}_BGOSum_DiscbitMask` | BGO sum discbit mask |
| `GS{N}_BGO_OSERDES_DataCtl` | BGO OSERDES data control |
| `GS{N}_Ge_Threshold` | Ge signal threshold (on pickoff) |
| `GS{N}_GeCenterGain` | Ge center segment gain |
| `GS{N}_GeCenterTimeConstant` | Ge center time constant |
| `GS{N}_GeCenter_DCOffset` | Ge center DC offset |
| `GS{N}_GeSide_DCOffset` | Ge side DC offset |
| `GS{N}_GeSideB_Offset` | Ge side B offset |
| `GS{N}_GeSideInputSelect` | Ge side input select |

### 4f. RAM Table Patterns (Dongle/Preamp — bulk)

| Pattern | Count/det | Description |
|---------|-----------|-------------|
| `GS{N}_Dongle_P{0–8}_B{0–15}` | 144 | GS_ID dongle I2C ROM data |
| `GS{N}_Preamp_P{0–8}_B{0–15}` | 144 | Preamp I2C ROM data |
| `GS{N}_Preamp_digpot_B{0–15}` | 16 | Preamp digital pot bits |

---

## 5. Global / Other PVs

### 5a. System-wide (from boot/vme*.cmd OTHER section)

| Register | Nature | Description |
|----------|-------------|
| `DAQC{NN}_CV_CRATENUM` | Crate number PV (N=01–12) |
| `DAQC{NN}_CV_InLoop{1}` | In-loop status flag |
| `inLoop_Running` | Global DAQ loop running flag |

### 5b. Useful CA Lookup Commands

```bash
# All MDIG1 PVs in VME01
cainfo "VME01:MDIG1:*"

# trigger_mux_select across all crates + boards
for c in 01 02 03 04 05 06 07 08 09 10 11 12; do
  for b in MDIG1 MDIG2 SDIG1 SDIG2; do
    caget VME${c}:${b}:trigger_mux_select 2>/dev/null
  done
done

# All Ge HV demands
for n in $(seq 1 110); do caget GS${n}_GE_HV_DEMAND_VOLTS; done

# BGO HV for GS hole 1
for i in $(seq 0 13); do caget GS1_BGO_HV${i}; done
```

---

## 6. PV Count Summary

| Subsystem | PVs/instance | Instances | ~Total |
|-----------|-------------|-----------|--------|
| MDIG | 1,743 | 22 | ~38,350 |
| SDIG | 1,743 | 22 | ~38,350 |
| RTRG | 1,080 | 4 | ~4,320 |
| MTRG | 7,711 | 1 | ~7,711 |
| Collector Box | 1,431 | 110 | ~158,070 |
| Global/OTHER | ~700 | 1 | ~700 |

> **Note:** The ~88,700 VME IOC PV count is based on verified template record counts (2026-04-19).
> Adding collector box PVs brings the true total to ~247,000 across the full DGS system.
> (Older raw dumps showed ~57,000 — that figure was based on undercounted template records.)

_Source: `DGS_tools_pack/ioc/db/*.template`, `collectorboxpi/CollectorBox_RevA/db/*.db`_
_Last updated: 2026-04-19_

## Cross-References

- `knowledgeBase/VME_registers.md` — VME register addresses; maps to PV names via asyn driver
- `knowledgeBase/ioc.md` — IOC boot scripts and DB loading; how these PVs are instantiated
- `knowledgeBase/EPICS.md` — EPICS record types used in this PV list
- `knowledgeBase/EPICS_asyn.md` — asyn driver: how caput/caget maps to VME register reads/writes
- `knowledgeBase/snapshot_pv.md` — snapshot_pv tools for saving/restoring PV values
