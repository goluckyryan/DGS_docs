# DIG Firmware — Selected Module Analysis (Part 2: DC Balance, FIFOs, VME)

Stability: C3 - Structural / stable

_Analysis of DIG production VHDL modules: DC balance, waveform FIFOs, decimation, event headers, channel FIFO readout, VME interface, and register map._  
_For signal chain, SERDES, timestamp, trigger mux, and per-channel readout FSMs → see [deep_fpga_DIG_modules.md](deep_fpga_DIG_modules.md)._  
_Source tag: `MAIN_FPGA_TAGS/20180507/MAIN_FPGA/BuildBranches/DGS/Source/`_  
_Authors: John Anderson (JTA), Michael Oberling (MBO)_

## Table of Contents

- [dc_balance_mach.vhd — SERDES DC Balance Machine](#dc_balance_machvhd--serdes-dc-balance-machine)
- [disparity_lookup.vhd — 8-bit Disparity Lookup Table](#disparity_lookupvhd--8-bit-disparity-lookup-table)
- [event_data_fifo.vhd — Per-Channel Event Waveform FIFO](#event_data_fifovhd--per-channel-event-waveform-fifo)
- [decimator.vhd — Waveform Decimation / Averaging](#decimatorvhd--waveform-decimation--averaging)
- [Event_Header_FIFO.vhd — Per-Channel Event Header FIFO](#event_header_fifovhd--per-channel-event-header-fifo)
- [Channel_FIFO_Readout_Mach.vhd](#channel_fifo_readout_machvhd-365-l-mbo-2013-12-03--dgs-production)
- [Channel_FIFO_Readout_Mach_Rework_WIP.vhd](#channel_fifo_readout_mach_rework_wipvhd-232-l-mbo--wip--never-deployed-in-production)
- [Lvme.vhd — VME Interface](#lvmevhd-190-l-anl-hep--vme-interface)
- [Registers.vhd — DIG VME Register Map](#registersvhd-391-l-mbo--dig-vme-register-map)
- [dp_srl_template.vhd — PEHQ SRL Delay](#dp_srl_templatevhd-51-l--pehq-srl-delay)

---

## dc_balance_mach.vhd — SERDES DC Balance Machine

**Entity:** `dc_balance_mach`  
**Author:** Michael Oberling  
**Date:** 2013-09-12  
**Lines:** 133  
**Clock domains:** CLK (50 MHz) + CLK_2X (100 MHz — used for the interleaved 8-bit disparity lookup)

### Role

Applies a polarity-invert DC balance scheme to the 18-bit SERDES TX data. Works in tandem with `disparity_lookup.vhd`. The `enable` input gates the feature on/off; when disabled the input data passes through unmodified.

### Algorithm — 7-Stage Pipeline

| Stage | Clock | Action |
|-------|-------|--------|
| 1 — DISP_LOOKUP_MUX | CLK_2X | Mux 8-bit slice from 18-bit word: **low half** [8:1] when CLK='1' (rising CLK_2X), **high half** [16:9] when CLK='0' ✅ verified 2026-04-24 — dc_balance_mach.vhd:L56-60 (if clk='1' → bits[8:1]; else → bits[16:9]) |
| 2 — LOOKUP | comb | `disparity_lookup` ROM: 256-entry table maps 8-bit slice → 4-bit signed partial disparity |
| 3 — PARTIAL_DISP_SHIFTER | CLK_2X | Pipeline register — shifts partial_disparity(2:0) by one each CLK_2X cycle |
| 4 — DISP_ADDER | CLK | Sign-extend (4→5 bit) and add the two partial disparities → 5-bit `data_disparity` |
| 5 — TRUE_COMP_CALC | CLK | Register `true_data_disparity = data_disparity`; compute `comp_data_disparity = NOT data_disparity` (i.e., inverted polarity contribution) |
| 6 — TRUE_COMP_COMPARE | CLK (+ reset) | Compare MSB of `running_disparity` vs. MSB of `true_data_disparity`; if equal → select complemented polarity, else select true; accumulate into 8-bit `running_disparity`; drive `data_polarity` |
| 7 — OUTPUT_LATCH | CLK | Delay 18-bit input by 4 clocks (8-entry SRL); apply inversion to bits [17:1] based on `data_polarity`; bit [0] carries `data_polarity` itself |

### Key Details

- **Input word:** `unbalanced_data[17:0]` — bit 0 is discarded / not used for disparity measurement (comment in code).
- **Output word:** `balanced_data[17:0]` — bits [17:1] = (possibly inverted) data; bit [0] = polarity flag.
- **Disparity accumulator:** 8-bit signed running sum. When the accumulated disparity has the same sign as the word being added (MSBs match), the complemented version is sent to keep the running value near zero.
- **Latency:** ~6 clock cycles from input to output (pipeline stages 1–7 add up to one CLK_2X + five CLK cycles; the 4-clock data delay in stage 7 aligns the data with the polarity decision).
- **Disable path:** When `enable='0'`, `balanced_data <= unbalanced_data` (pass-through, registered at CLK).

---

## disparity_lookup.vhd — 8-bit Disparity Lookup Table

**Entity:** `disparity_lookup`  
**Author:** Michael Oberling  
**Date:** 2013-09-12  
**Lines:** 55  
**Combinational (no clock)**

### Role

Instantiated inside `dc_balance_mach`. Implements a 256-entry ROM that returns the signed 4-bit disparity (excess of 1s over 0s, biased) for any 8-bit input byte. The ROM is populated at compile time using the `disp_calc` function from `Digitizer_pkg.vhd`.

### Details

- **Input:** `data[7:0]` — 8-bit word.
- **Output:** `disparity[3:0]` — 4-bit signed disparity value.
- **Implementation:** purely combinational ROM lookup (`lookup(conv_integer(data))`).
- **ROM initialization:** `disp_calc(X"00")` through `disp_calc(X"FF")` — all 256 entries, ordered by byte value. The `disp_calc` function computes `(number_of_ones – number_of_zeros)` in some signed encoding defined in the package.

---

## event_data_fifo.vhd — Per-Channel Event Waveform FIFO

**Entity:** `event_data_fifo`  
**Author:** Michael Oberling  
**Date:** 2014-06-20  
**Lines:** 113  
**Clock domain:** CLK (100 MHz)

### Role

The "accordion" FIFO for the per-channel waveform data path. Accumulates 16-bit waveform words from the `decimator` and presents them as 32-bit pairs to `event_packer`. Called an accordion FIFO because it buffers data to allow time for header injection between consecutive event readouts without losing waveform continuity.

### Signal Interface

| Signal | Dir | Description |
|--------|-----|-------------|
| `event_data_in[15:0]` | in | Raw 16-bit waveform word from decimator output |
| `event_data_write_en` | in | Strobe — one per waveform sample |
| `event_data_read_en` | in | Read enable from `event_packer` |
| `dec_enable` | in | When '0', write_toggle resets to '1' (resync for next event) |
| `event_data_out[1:0][15:0]` | out | 2× 16-bit (= 32-bit) output from FIFO, low half `[0]` and high half `[1]` |
| `event_data_empty` | out | FIFO empty flag |
| `event_data_almost_empty` | out | FIFO almost-empty flag |

### Write Logic

The underlying Xilinx FIFO is 32 bits wide. Two 16-bit samples are packed before each FIFO write:

```
write cycle 1: shift new sample into DIN[15:0], no FIFO write (write_toggle=1→0)
write cycle 2: shift again into DIN[31:16], write to FIFO (write_toggle=0→1)
```

- `write_toggle` resets to '1' when `dec_enable='0'`, ensuring alignment is always correct at the start of a new event.
- If `dec_enable='0'` and no write is occurring, `write_toggle` holds at '1' and `wr_en` holds '0'.

### Read Logic

```
event_data_fifo_rd_en = event_data_fifo_full OR event_data_read_en
```

The FIFO self-drains when full (overflow prevention). Normal readout is driven by `event_packer`.

### FIFO Instance

Uses `event_fifo` Xilinx core (32-bit wide, FWFT mode implied by the almost_empty flag use).

---

## decimator.vhd — Waveform Decimation / Averaging

**Entity:** `decimator`  
**Author:** Michael Oberling  
**Lines:** 205  
**Date added:** updated 2016-03-04 (dec_pause), 2016-03-07 (timing mark(1)), note about future bit-14 flag in data  
**Clock domain:** CLK (100 MHz)

### Role

Averages ADC waveform samples by accumulating and dividing (via right-shift), reducing data rate during non-peak portions of the waveform. Supports decimation factors 1×–128× and a special PAUSE mode that outputs full-speed undecimated data while continuing the accumulation.

### Ports

| Port | Width | Description |
|------|-------|-------------|
| `dec_enable` | 1 | Averaging begins synchronously on rising edge; exits state machine on falling edge |
| `dec_pause` | 1 | 2016-03-04: separate on/off control; transitions synchronized to accumulation boundary |
| `dec_factor[2:0]` | 3 | Decimation factor: 0=1×, 1=2×, 2=4×, 3=8×, 4=16×, 5=32×, 6=64×, 7=128× |
| `adc_data[13:0]` | 14 | Raw 14-bit ADC input |
| `timing_mark_in[1:0]` | 2 | Timing mark flags from channel logic |
| `dec_data[15:0]` | 16 | Output: decimated value, always 16-bit precision |
| `dec_timing_mark[1:0]` | 2 | Timing-aligned flag output (via `flag_queue` components) |
| `dec_data_valid` | 1 | Pulses once per output sample |

### State Machine: 3 States

| State | Behaviour |
|-------|-----------|
| `DISABLE` | `accumulate_counter=1`, `accumulate_strobe='1'` (reset accumulator), `dec_data_valid=0`. Exits to ENABLED on `delayed_enable='1'`. |
| `ENABLED` | Count accumulate_counter; when `counter(dec_factor)='1'`, roll over, strobe accumulator, assert `dec_data_valid`. If `latched_dec_pause='1'` at rollover → transition to PAUSE. |
| `PAUSE` | Full-speed output (`dec_data_valid='1'` every clock), `sampled_dec_pause='1'` (selects unscaled output mux). Continues accumulating. At accumulation rollover: if `latched_dec_pause='0'` → return to ENABLED. |
✅ verified 2026-04-24 — decimator.vhd:L91-148 (DISABLE: counter=1, strobe='1', exits when delayed_enable='1' L102; ENABLED: rollover on counter(dec_factor)='1' L110, latched_dec_pause gates PAUSE entry L113; PAUSE: early_dec_data_valid='1' every clock L132, sampled_dec_pause='1' L131, returns ENABLED when latched_dec_pause='0' L139)

### Output Scaling (Right-Shift by dec_factor)

```
PAUSE mode:  dec_data = last_unscaled_sum[13:0] & "00"       (no averaging)
dec_factor 0: dec_data = last_sum[13:0] & "00"              (no averaging, 14-bit → 16-bit)
dec_factor 1: dec_data = last_sum[14:0] & '0'              (2× average)
dec_factor 2: dec_data = last_sum[15:0]                     (4× average)
dec_factor 3: dec_data = last_sum[16:1]                     (8× average)
dec_factor 4: dec_data = last_sum[17:2]                     (16× average)
dec_factor 5: dec_data = last_sum[18:3]                     (32× average)
dec_factor 6: dec_data = last_sum[19:4]                     (64× average)
dec_factor 7: dec_data = last_sum[20:5]                     (128× average)
```
✅ verified 2026-04-24 — decimator.vhd:L69-78 (concurrent signal assignment; PAUSE uses last_unscaled_sum(13 downto 0)&"00"; factors 0-6 explicit when-else; factor 7 = last_sum(20 downto 5) as else default; redundant line commented out at L77)

All outputs are 16-bit, so the effective precision scales with averaging depth.

### Accumulator

21-bit `sum` accumulates 14-bit ADC samples (`"0000000" & adc_data`). On `accumulate_strobe='1'`, `last_sum <= sum` and `sum` resets to current sample.
✅ verified 2026-04-24 — decimator.vhd:L56 (signal sum std_logic_vector(20 downto 0)); L157-163 (accumulator process: last_unscaled_sum latches "0000000"&adc_data each clock; on accumulate_strobe: last_sum<=sum, sum resets to current sample; else sum+=adc_data)

### Timing Marks

- `timing_mark_in[0]` is passed into a `flag_queue` component and re-emitted synchronized to `dec_data_valid` — ensures timing marks survive decimation.
- `timing_mark_in[1]` is always 0 from channel logic (noted 2016-03-03); `dec_timing_mark[1]` is instead the time-aligned `pause_change_state` signal, marking decimation-mode boundaries in the waveform output.
- Comment (2016-03-07): `dec_timing_mark[1]` (or bit 14 of the data) could be used by readout software to identify decimated vs. non-decimated segments.

---

## Event_Header_FIFO.vhd — Per-Channel Event Header FIFO

**Entity:** `event_header_fifo`  
**Author:** Michael Oberling  
**Lines:** 704  
**Generic:** `channel_id : integer` — 4-bit channel identifier embedded in header word 1  
**Clock domain:** CLK (100 MHz)

### Role

Manages per-channel event metadata. Buffers 14-word event headers in a 36-bit-wide ×514-deep FIFO (Xilinx `fifo_36x514_comclk_pfiport_fwft`). Exposes a pre-header word to the readout controller for timing decisions, and a sequential header word stream to `Channel_Readout_Mach` for inclusion in the accepted-event data stream. Supports both LED and CFD header formats.

### Key Constants

| Constant | Value | Meaning |
|----------|-------|---------|
| `cHEADER_LENGTH` | 28 | Total header length (in 32-bit words) reported in the header |
| `cSTOP_HEADER_INDEX` | 13 | Last word index written/read (words 0–13 = 14 total FIFO entries) |
| `cREPORTED_HEADER_SIZE` | 26 | `cHEADER_LENGTH – 2` |
| `cEVENT_HEADER_FIFO_PROG_EMPTY_THRESH` | 375 | = 25 events (prog_empty threshold) |
| `cEVENT_HEADER_FIFO_PROG_FULL_THRESH` | 495 | = 33 events (prog_full threshold) |
| `cEVENT_DELIMITER` | `0xAAAAAAAA` | Fixed delimiter word (word 0 of every header in the output stream) |

✅ verified 2026-04-24 — Event_Header_FIFO.vhd:L85-90 (cHEADER_LENGTH=28, cSTOP_HEADER_INDEX=13, cREPORTED_HEADER_SIZE=cHEADER_LENGTH-2=26, PROG_EMPTY_THRESH=375, PROG_FULL_THRESH=495, cEVENT_DELIMITER=X"AAAAAAAA")

| `cHEADER_TYPE_LED` | 5 | Header type code for LED events ✅ verified 2026-04-24 — Event_Header_FIFO.vhd:L94 (to_unsigned(5,4)) |
| `cHEADER_TYPE_CFD` | 6 | Header type code for CFD events ✅ verified 2026-04-24 — Event_Header_FIFO.vhd:L95 (to_unsigned(6,4)) |

### Header Format — Words Written to FIFO (14 words × 32 bits)

Word 0 is an **internal pre-header** (never forwarded to accepted-event FIFO):
```
[31]    = xextended_event
[30]    = pileup_flag
[29:24] = 0
[23:0]  = ts_of_discbit[23:0]  (lower 24 bits of disc timestamp — for readout timing)
```

Words 1–13 are the **actual header** (forwarded to accepted-event FIFO). Word 0 in the output stream is replaced by `0xAAAAAAAA`.

#### LED Header (cfd_mode='0') — Words 1–13

| Word | Bits | Content |
|------|------|---------|
| 1 | [31:27] | Geo address |
| 1 | [26:16] | Packet length (filled at readout time by READ_EVENT_WORD_1 state) |
| 1 | [15:4] | UserDef (user_package_data) |
| 1 | [3:0] | Channel ID |
| 2 | [31:0] | Event timestamp [31:0] |
| 3 | [31:26] | Header length (cHEADER_LENGTH) |
| 3 | [25:23] | Event type (filled at readout by READ_EVENT_WORD_3) |
| 3 | [22] | '0' |
| 3 | [21] | TRIG_TS_MODE (0=use TRIG_ARRIVAL_TS in word 10, 1=use TRIGGER_MUX_TS) |
| 3 | [20] | PEQ_BYPASS |
| 3 | [19:16] | Header type = cHEADER_TYPE_LED (5) |
_✅ corrected 2026-04-24 — Event_Header_FIFO.vhd:L288-291 (bit22='0', bit21=TRIG_TS_MODE, bit20=PEQ_BYPASS, bits[19:16]=cHEADER_TYPE_LED; previous KB had [22:20]="000" and header type=3, both wrong)_
| 3 | [15:0] | Event timestamp [47:32] |
| 4 | [31:16] | ts_of_last_discbit[15:0] |
| 4 | [15] | pileup_flag |
| 4 | [14] | pileup_waveform_only |
| 4 | [13] | general_error_flag (filled at readout by READ_EVENT_WORD_4) |
| 4 | [12] | sync_error_flag |
| 4 | [11] | '0' |
| 4 | [10] | offset_flag (filled at readout) |
| 4 | [9] | peak_valid_flag |
| 4 | [8] | external_disc_flag |
| 4 | [7] | '0' |
| 4 | [6] | veto_flag (filled at readout by READ_EVENT_WORD_4) |
| 4 | [5] | NOT write_flags (WF bit) |
| 4 | [4] | pu_time_error_flag |
| 4 | [3:0] | '0' |
| 5 | [31:0] | ts_of_last_discbit[47:16] |
| 6 | [31:24] | 0x00 |
| 6 | [23:0] | sampled_baseline[23:0] |
| 7 | [31:16] | TRIG_MON_DET_DATA[15:0] (detector state from F2 frame) |
| 7 | [15:0] | TRIG_MON_XTRA_DATA[15:0] (extra state from F2 frame) |
_✅ corrected 2026-04-24 — Event_Header_FIFO.vhd:L315-316 (header(7)(31:16)<=TRIG_MON_DET_DATA; header(7)(15:0)<=TRIG_MON_XTRA_DATA; previous KB had 0x00000000, wrong)_
| 8 | [31:24] | post_rise_energy[7:0] |
| 8 | [23:0] | pre_rise_energy[23:0] |
| 9 | [31:16] | ts_of_peak[15:0] |
| 9 | [15:0] | post_rise_energy[23:8] |
| 10 | [31:16] | Trigger timestamp (filled at readout) |
| 10 | [15:14] | '00' |
| 10 | [13:0] | last_peak_sample |
| 11 | [31:30] | '00' |
| 11 | [29:16] | post_rise_enter_sample |
| 11 | [15:14] | '00' |
| 11 | [13:0] | post_rise_leave_sample |
| 12 | [31:30] | '00' |
| 12 | [29:16] | pre_rise_enter_sample |
| 12 | [15:14] | '00' |
| 12 | [13:0] | pre_rise_leave_sample |
| 13 | [31:30] | '00' |
| 13 | [29:16] | base_sample |
| 13 | [15:14] | '00' |
| 13 | [13:0] | peak_sample |

#### CFD Header (cfd_mode='1') — Differences from LED

| Word | Bits | LED content | CFD content |
|------|------|-------------|-------------|
| 3 | [19:16] | cHEADER_TYPE_LED (5) | cHEADER_TYPE_CFD (6) |
| 4 | [15] | pileup_flag | event_data.pileup_flag |
| 4 | [11] | '0' | cfd_valid_flag |
| 4 | [7] | '0' | tsm_flag |
| 4 | [3:0] | '0' | TRIG_MON_DET_DATA[15:12] ✅ corrected 2026-04-24 — L397 |
| 5 | [31:30] | ts_of_last_discbit[47:32] upper 2b | TRIG_MON_DET_DATA[11:10] ✅ corrected 2026-04-24 — L400 |
| 5 | [29:16] | ts_of_last_discbit[45:32] (14b) | cfd_sample(0)[13:0] ✅ verified 2026-04-24 — L401 |
| 5 | [15:14] | ts_of_last_discbit[31:30] (2b) | TRIG_MON_DET_DATA[9:8] ✅ corrected 2026-04-24 — L402 |
| 5 | [13:0] | ts_of_last_discbit[29:16] (14b) | ts_of_last_discbit[29:16] ✅ verified 2026-04-24 — L403 |
| 7 | [31:0] | 0x00000000 | {00, cfd_sample(2)[13:0], 00, cfd_sample(1)[13:0]} |

### Write State Machine (2 states)

| State | Action |
|-------|--------|
| `IDLE` | Waits for `accepted_hit='1'` OR `extended_event='1'` AND `prog_full='0'`. Latches `xextended_event`, `pileup_flag`, resets `header_word_sel=0`. |
| `WRITE` | Writes words 0–13 sequentially (14 clocks). On each clock: `header_word_sel` increments; bit 32 of FIFO word = stop bit (set on word 13). Returns to IDLE when word 13 done. |

- **Stop bit:** Bit 32 of the 36-bit FIFO entry is '1' on the last header word (index 13), used by the read SM to detect end-of-header.
- **Throttle:** `local_throttle='1'` when `prog_empty='0'` (≥25 events queued). This emergency throttle gates the readout controller to prevent overrun with large decimation ratios.
- **Drop detection:** `event_header_dropped='1'` if a new event arrives while write SM ≠ IDLE or FIFO is prog_full — non-recoverable, requires reset.

### Read State Machine (6 states)

| State | Action |
|-------|--------|
| `WAIT_FOR_WRITE` | Waits until FIFO non-empty. |
| `READ_PRE_HEADER_WORD` | Reads word 0 (internal pre-header): latches `next_event_extended_flag[31]`, `next_event_pileup_flag[30]`, `next_event_timestamp[23:0]`. Sets `next_event_data_ready='1'`. Outputs `0xAAAAAAAA` as first visible header word. → IDLE |
| `IDLE` | Waits for `read_event` or `drop_event` from readout controller. |
| `FLUSH_EVENT` | Drains words 1–13 into bit-bucket. After word 13: if FIFO not almost-empty → READ_PRE_HEADER_WORD, else → WAIT_FOR_WRITE. |
| `READ_EVENT_WORD_N` | Standard word readout. Detects special words (1, 3) and routes to patching states. Stop bit → end of header. |
| `READ_EVENT_WORD_1` | Patches packet_length into word 1 `[26:16]` before forwarding. → READ_EVENT_WORD_N |
| `READ_EVENT_WORD_3` | Injects `current_event_type` into word 3 `[25:23]`. → READ_EVENT_WORD_4 |
| `READ_EVENT_WORD_4` | Injects `veto_flag` into bit [6], `offset_flag` into bit [10], `general_error_flag` into bit [13] of word 4. → READ_EVENT_WORD_N |

### Late-Injection Fields

Three header fields are not known at write time and are patched at readout by the read SM:

| Field | Word | Bits | Source |
|-------|------|------|--------|
| Packet length | 1 | [26:16] | `next_event_waveform_length + cREPORTED_HEADER_SIZE` |
| Event type | 3 | [25:23] | `current_event_type` (from event decision FIFO) |
| Veto flag | 4 | [6] | `veto_flag` |
| Offset flag | 4 | [10] | `offset_flag` |
| General error | 4 | [13] | `general_error_flag` |

### Reset Handling

Write SM uses `control_reset` directly. Read SM uses a 16-clock **SRL16-delayed** version (`delayed_control_reset`) to ensure any in-progress readout completes before the FIFO is cleared.

---

---

## Channel_FIFO_Readout_Mach.vhd (365 L, MBO 2013-12-03 / DGS production)

**Source:** `BuildBranches/DGS/Source/Channel_FIFO_Readout_Mach.vhd`  
**Entity:** `channel_FIFO_readout_mach`  
**Role:** Arbitrates readout from up to 10 per-channel accepted-event FIFOs into a single dual-clock collection FIFO, handling packet boundaries and backpressure.

### Interface Summary

| Port | Width | Direction | Description |
|------|-------|-----------|-------------|
| `reset_fifo` | 1 | in | Synchronous reset (changed from async 2023-07-24 JTA) |
| `clk_dsp` | 1 | in | Write-side clock |
| `acptd_event_fifo_rd_en` | 10 | out | One-hot per-channel read enables |
| `acptd_event_fifo_dout` | 10×36 | in | Per-channel 36-bit data (bit 32 = end-of-packet/stop bit) |
| `acptd_event_fifo_almost_empty` | 10 | in | Per-channel almost-empty flags |
| `data_available` | 10 | in | Per-channel packet-ready flags |
| `coll_fifo_read_clk` | 1 | in | Collection FIFO read clock (separate domain) |
| `coll_fifo_read_enable` | 1 | in | Collection FIFO read enable |
| `coll_fifo_data` | 32 | out | Collection FIFO output (bits 35:32 stripped — carried as `coll_fifo_data_extra`) |
| ~~`EVENT_BOUNDARY_FLAG`~~ | — | — | ⚠️ corrected 2026-04-24: this port does **not exist** in the entity; `coll_fifo_data_extra[35:32]` is an internal signal only — bit 32 is not exposed as an output port ✅ verified 2026-04-24 — Channel_FIFO_Readout_Mach.vhd entity ports (L36-54): no EVENT_BOUNDARY_FLAG; internal signal at L78 |
| `coll_fifo_empty/almost_empty/full` | 1 each | out | Collection FIFO status |
| `reg_ch_fifo_readout_status` | 8 | out | Diagnostic status register |

### State Machine (7 states)

| State | Behaviour |
|-------|-----------|
| **IDLE** | Waits for any `data_available` bit; resets `channel_select=0`, all enables low; transitions to SCAN on any data |
| **SCAN** | Scans channels 0–9 in order; skips channels without data; pauses (stays SCAN) if collection FIFO full/almost-full; jumps to XFER_EVENT when a data-ready channel is found; returns to IDLE when all 9 channels exhausted |
| **XFER_EVENT** | Streams words from current channel into collection FIFO; pauses to XFER_PAUSED if collection FIFO almost-full or channel almost-empty; transitions to XFER_LAST_WORD when stop-bit (bit 32) detected; stops write/read enables simultaneously |
| **XFER_LAST_WORD** | Writes the latched final word (WR_EN=1, read_en_0=0); increments to next channel then returns to SCAN, or wraps to IDLE if channel_select=9 |
| **XFER_PAUSED** | Holds all enables low; waits for `data_available_mux_out=1` AND `coll_fifo_almost_full=0`; transitions to RESUME_XFER |
| **RESUME_XFER** | Re-asserts WR_EN=1, read_en_0=1 for one clock (pipeline prime) then goes to XFER_EVENT |
| **XFER_PAUSED_ON_LAST_WORD** | Special case: collection FIFO went almost-full exactly on last word; holds until `int_coll_fifo_full=0`, then XFER_LAST_WORD |

**Key design points:**
- `read_en_0` is the master enable; actual `acptd_event_fifo_rd_en[9:0]` is one-hot gated by `channel_select`
- Stop bit: `acptd_event_fifo_dout[32]` signals end-of-packet (set by `event_packer` nibble 0x1 logic)
- Bit 32 propagates through collection FIFO's upper bits; read side strips lower 32 bits to `coll_fifo_data`; `coll_fifo_data_extra[35:32]` holds the upper nibble internally but is **not exported** as a port ⚠️ corrected 2026-04-24 — no EVENT_BOUNDARY_FLAG port exists
- Collection FIFO: **`fifo_36x1025_sepclk_pfiport_fwft`** (1025 deep, separate clocks, FWFT mode); prog-empty=0xFF (255), prog-full=0x300 ✅ verified 2026-04-24 — Channel_FIFO_Readout_Mach.vhd:L338-339 (prog_empty_thresh="0011111111"=0xFF, prog_full_thresh="1100000000"=0x300); upgraded from 36×513 (comment L328)
- Status register bits: [0]=coll_fifo_read_enable, [1]=WR_EN, [3]=read_en_0, [4]=channel_data[32], [5]=coll_fifo_empty, [6]=coll_fifo_almost_full, [7]=coll_fifo_full; all driven through explicit `BUF` primitives added 2016-03-10 to ease timing

---

## Channel_FIFO_Readout_Mach_Rework_WIP.vhd (232 L, MBO — WIP / never deployed in production)

**Source:** `BuildBranches/DGS/Source/Channel_FIFO_Readout_Mach_Rework_WIP.vhd`  
**Entity:** `channel_FIFO_readout_mach` (same name — drop-in replacement intent)  
**Role:** Redesigned readout arbiter with a simpler, cleaner 6-state FSM; replaces complex pause/resume logic with explicit increment-and-wait pipeline. **Not used in production** — named WIP.

### Key Differences vs. Production Version

| Aspect | Production (365 L) | WIP Rework (232 L) |
|--------|--------------------|--------------------|
| States | 7 (IDLE/SCAN/XFER_EVENT/XFER_LAST_WORD/XFER_PAUSED/RESUME_XFER/XFER_PAUSED_ON_LAST_WORD) | 6 (FIND_NEXT_CHANNEL/CHANNEL_INC/CHANNEL_INC_WAIT/PUSH_CHANNEL/PUSH_WAIT_0/PUSH_WAIT_1) |
| Reset | Synchronous only (2023 JTA change) | Asynchronous (`if RESET='1'` in sensitivity) |
| Channel scan | Linear 0→9, returns to IDLE at 9 | Wraps around (channel 9 → channel 0); no IDLE state — always scanning |
| Collection FIFO | 36×1025 FWFT sep-clk | 36×513 FWFT sep-clk (smaller) |
| `EVENT_BOUNDARY_FLAG` | ⚠️ corrected 2026-04-24: not a port in production version; `coll_fifo_data_extra` is internal only | Not present |
| Stop-bit handling | Dedicated XFER_LAST_WORD + XFER_PAUSED_ON_LAST_WORD states | Inline in PUSH_CHANNEL: WR_EN=0 + go to CHANNEL_INC |
| `xDATA_AVAILABLE_MUX_OUT` | Not present | Registered copy of DATA_AVAILABLE_MUX_OUT (1-cycle delay) used in PUSH_WAIT_1 |
| ILA/BUF diagnostics | Yes (explicit BUF + 32-bit ILA port) | No |

**Conclusion:** The WIP rework simplifies the FSM and removes the ILA overhead, but was never integrated into production builds. All DGS build branches still use the original 7-state version.

---

## Lvme.vhd (190 L, ANL HEP — VME Interface)

**Source:** `MAIN_FPGA_ISE11/Source/Lvme.vhd`  
**Entity:** `COMP_LVME`  
**Author:** ANL High Energy Physics Division (original); MBO changes 2014  
**Role:** VME bus interface for the DIG FPGA. Bridges the VME address/data bus to the internal register file (`COMP_REGISTERS`) and the board-level FIFO, issuing `reg_write`, `fifo_vme_write`, `fifo_vme_oe`, and `fifo_vme_read` control strobes.

### Port Summary

| Port | Dir | Width | Description |
|------|-----|-------|-------------|
| `lvme_addr_in` | in | 32 | VME address bus |
| `lvme_ack` | out | 1 | DTACK (data transfer acknowledge) |
| `lvme_csel` | in | 3 | Chip select: `"001"` = register/main-FPGA, `csel[2]`=FIFO OE, `csel[1]`=FIFO read |
| `lvme_data_io` | inout | 32 | VME data bus (tristate) |
| `lvme_rnw` | in | 1 | Read/not-write |
| `lvme_spare_io` | inout | 5 | Spare; bit 0 used for system-restore indirect address latch |
| `lvme_strobe` | in | 1 | VME DS (data strobe) |
| `fifo_vme_{data,oe,read,write}` | out | – | FIFO control/data path |
| `reg_{address,data_in,data_out,write}` | – | – | Register file interface |

### State Machine (8 states)
✅ verified 2026-04-24 — Lvme.vhd:L60 (type declaration: ST_IDLE/ST_READ/ST_WRITE/ST_CLEAR/ST_ACK_DLY1/ST_ACK_DLY2/ST_ACK_DLY3/ST_ACK); L124-180 (FSM logic: RNW→ST_READ, write→ST_WRITE, ST_WRITE→ST_CLEAR on reg/ST_ACK on FIFO, ST_CLEAR→ST_ACK_DLY1, DLY1-3 chain, ST_ACK holds until strobe='0')

| State | Next | Action |
|-------|------|--------|
| `ST_IDLE` | ST_READ / ST_WRITE | Latch `lvme_addr_in[19:0]`; advance on `strobe='1' AND csel="001"` |
| `ST_READ` | ST_ACK_DLY1 | Register reads require pipeline delay (JTA 2012-11-08) |
| `ST_WRITE` | ST_CLEAR (reg) / ST_ACK (FIFO) | Register: assert `reg_write`; FIFO: assert `fifo_vme_write` and skip delay |
| `ST_CLEAR` | ST_ACK_DLY1 | Deassert `reg_write` |
| `ST_ACK_DLY1–3` | ST_ACK | 3-cycle pipeline to ensure DTACK asserts **after** data is valid |
| `ST_ACK` | ST_IDLE | Assert `lvme_ack='1'`; hold until `strobe='0'` |

### Address Decode
✅ verified 2026-04-24 — Lvme.vhd:L86-89 (addr[19:12]=X"00"→reg path; else→lvme_data_out<=X"BAD"&lvme_addr_lat[19:0]); L144 (WRITE: addr[19:12]=X"00"→reg, else FIFO); L150 (fifo_vme_write='1' → ST_ACK directly, no delay)

- `addr[19:12] = 0x00` → register file access (12-bit address `addr[11:0]`)
- `addr[19:12] ≠ 0x00` → returns `0xBAD_xxxxx` (bad-address marker) on reads; FIFO write on writes
- `lvme_csel[2]` → FIFO output enable (pass-through, async)
- `lvme_csel[1]` → FIFO read enable (pass-through, async)
- FIFO writes bypass delay states — faster cycle time than register writes

### System Restore (MBO 2014-06-10)

- `lvme_spare_io[0]='1'` → latch `lvme_data_in` into `lvme_indirect_addr` (32-bit indirect address for restore)
- Enables restoring VME state after power cycle without bus arbitration

### Notes
✅ verified 2026-04-24 — Lvme.vhd:L79 (spare_io<="ZZZZZ"); L96 (lvme_data_dir combinatorial, async); L106-107 (spare_io(0)='1'→latch data into indirect_addr); L78 (data_io driven from lvme_data_out when dir='1' else 32×'Z')

- `lvme_spare_io` is driven `"ZZZZZ"` (hi-Z) on output; only `spare_io[0]` is used as an input
- `lvme_data_dir` is combinatorial (2014 MBO change to async): asserts when `strobe='1' AND rnw='1' AND csel="001"`
- Register data read path has one additional pipeline stage (`reg_data_out → clk → lvme_data_out`) added by MBO to ease routing
- Reset is asynchronous (active high); all outputs deassert and FSM returns to `ST_IDLE`

---

## Registers.vhd (391 L, MBO — DIG VME Register Map)

**Source:** `MAIN_FPGA_ISE11/Source/Registers.vhd`  
**Entity:** `COMP_REGISTERS`  
**Author:** Michael Oberling (MBO); various additions by JTA 2015–2016  
**Role:** Complete VME register file for the DIG FPGA. Instantiates `REGISTER_LOGIC` (BRAM-backed generic register array) and maps all 199 register entries to named per-channel and board-wide control/status ports.

### Register Configuration Array

All 199 registers are defined in a constant `R : tREGISTER_CONFIG_ARRAY(1 to 199)`. Each entry contains:
`(Address, Reset_Value, Read_Mask, Write_Mask, Fan-In_Group)`

- **Fan-In Group 0–9:** per-channel registers (group = channel index); resampled at 100 MHz (ADC_CLK)
- **Fan-In Group 15:** board-wide registers (not per-channel); left at 50 MHz domain
- **`reg_out_50MHz`** → driven by `REGISTER_LOGIC`; **`reg_out`** = `reg_out_50MHz` resampled to ADC_CLK

### Register Address Map (selected)

| Address | Name | Width | Default | Notes |
|---------|------|-------|---------|-------|
| `0x000` | `board_id` | 32 | 0 | R/O |
| `0x004` | `programming_done` | 27 | 0 | bit 27 W, bits 26:0 R/O; FPGA programming status |
| `0x008` | `external_disc_src` | 32 | 0 | External discriminator source select |
| `0x00C` | `vme_ext_delay` | 16 | 0xFF | VME external discriminator delay (added 2016-04-25) |
| `0x020` | `hardware_status` | 32 | 0 | R/O |
| `0x024` | `user_package_data` | 12 | 0 | |
| `0x028` | `win_comp_min` | 16 | `0b1111111000000000` | Window comparator minimum |
| `0x02C` | `win_comp_max` | 16 | `0b0000001000000000` | Window comparator maximum |
| `0x040–0x064` | `channel_control[0:9]` | 32 | 0 | Per-channel control word (MBO rename) |
| `0x080–0x0A4` | `led_threshold[0:9]` | 24 | `0x000564` | `[23:16]`=hysteresis(8b), `[13:0]`=threshold(14b); default threshold=100 |
| `0x0C0–0x0E4` | `cfd_fraction[0:9]` | 13 | `0x1800` | CFD fraction (MBO rename from `cfd_fraction`) |
| `0x100–0x124` | `raw_data_length[0:9]` | 10 | 25 | Waveform capture length in samples |
| `0x140–0x164` | `raw_data_window[0:9]` | 11 | 50 | Waveform capture window (extended to 11 bits 2016-03-04) |
| `0x180–0x1A4` | `d_window[0:9]` | 7 | 10 | D delay window; 50 MHz domain |
| `0x1C0–0x1E4` | `k_window[0:9]` | 14 | 100 | K integration window (mask `0x3FFF`; extended 2016-04-20 for K0 depth) |
| `0x200–0x224` | `m_window[0:9]` | 10 | 200 | M integration window |
| `0x240–0x264` | `d2_window[0:9]` | 7 | 23 | D2 delay window |
| `0x280–0x2A4` | `disc_width[0:9]` | 12 | 0 | Discriminator width; mask `0x3F3F` (two 6-bit fields) |
| `0x2C0–0x2E4` | `baseline_start[0:9]` | 14 | 1000 | Baseline tracker start value |
| `0x300–0x324` | `p1_window[0:9]` | 4 | 1 | P1 delay (50 MHz domain) |
| `0x400` | `dac` | 8 | 0 | DAC control |
| `0x404` | `p2_window` | 10 | 2 | P2 delay global (50 MHz domain) |
| `0x408` | `ila_config` | 8 | `0xD0` | ILA config; bits 7:4 co-opted for FIFO prog-full setting (2016-05-20) |
| `0x40C` | `channel_pulsed_control` | 32 | 0 | W/O (read mask 0); 50 MHz domain |
| `0x410` | `diag_mux_control` | 2 | 0 | Diagnostic mux |
| `0x414` | `holdoff_control` | 13 | 2198 | Renamed from `peak_sensitivity` (JTA 2015-04-08); bits [12]=0, [11:9]=4 (pkSens), [8:0]=150 (holdoff) |
| `0x418` | `baseline_delay` | 11+10 | `0x219` | Bits [25:16] R/O status (`regin_baseline_status`); bits [10:8]=speed, [7:0]=delay |
| `0x41C` | `diag_channel_input` | 32 | 0 | Diagnostic channel input select |
| `0x420` | `external_disc_mode` | 32 | 0 | External discriminator mode (MBO rename) |
| `0x424` | `rj45_spare_dout_control` | 32 | `0x21300000` | RJ45 spare output control; 50 MHz domain |
| `0x428` | `led_status` | 32 | 0 | R/O LED status |
| `0x430` | `led_control` | 32 | 0 | LED control; 50 MHz domain |
| `0x434` | `decimate_holdoff` | 8 | 100 | On/off decimation holdoff timeout (JTA 2016-03-06) |
| `0x484–0x490` | `lat_timestamp` / `live_timestamp` | 48 | 0 | R/O; `[31:0]` at base, `[47:32]` at +4 |
| `0x494` | `veto_gate_width` | 16 | 0 | Veto gate width |
| `0x500` | `master_logic_status` | 27 | `0x50` | Bits [26:13] and [11:0] from 100 MHz; bit [12] from 50 MHz (reset bit must not be relatched — MBO) |
| `0x504` | `trigger_config` | 2 | 0 | Trigger configuration |
| `0x508` | `phase_errors` | 32 | 0 | R/O SERDES phase error count |
| `0x50C` | `phase_value` | 32 | 0 | R/O current SERDES phase value |
| `0x510–0x518` | `phase_offset[0:2]` | 32 | 0 | R/O phase offset per SERDES lane |
| `0x51C` | `serdes_phase_value` | 32 | 0 | R/O SERDES phase value |
| `0x600` | `code_revision` | 32 | 0 | R/O firmware revision |
| `0x604` | `code_date` | 32 | 0 | R/O firmware date |
| `0x608` | `ts_err_count_ctrl` | 3 | 0 | TS error count control (bits [18:16]) |
| `0x60C` | `ts_err_count` | 32 | 0 | R/O timestamp error count |
| `0x700–0x724` | `dropped_event_count[0:9]` | 32 | 0 | R/O per-channel dropped event counters |
| `0x740–0x764` | `accepted_event_count[0:9]` | 32 | 0 | R/O per-channel accepted event counters |
| `0x780–0x7A4` | `ahit_count[0:9]` | 32 | 0 | R/O per-channel accepted-hit counters |
| `0x7C0–0x7E4` | `disc_count[0:9]` | 32 | 0 | R/O per-channel discriminator fire counters |
| `0x848` | `sd_config` | 21 | `0x11` | SERDES config (mask `0x000107FF`); 50 MHz domain |

### Clock Domain Handling

- `REGISTER_LOGIC` runs at 50 MHz (`clk`); output is `reg_out_50MHz`
- `RESAMPLE` process: every 100 MHz rising edge (`ADC_CLK`) copies `reg_out_50MHz → reg_out`
- Per-channel registers (groups 0–9) are driven from `reg_out` (100 MHz domain) for timing with the ADC pipeline
- Board-wide registers (group 15) use `reg_out_50MHz` directly — avoids re-latching the reset bit in `master_logic_status`
- The one exception: `master_logic_status[12]` (reset bit) taken from 50 MHz; all other bits from 100 MHz

### Notes

- Array size extended to 199 entries on 2016-04-25 for `vme_ext_delay` addition
- `reg_decimate_holdoff` added 2016-03-06 (JTA) at `0x434` for on/off decimation control
- `k_window` mask extended to 14 bits (`0x3FFF`) in 2016-04-20 for K0 depth feature
- `raw_data_window` extended to 11 bits in 2016-03-04 (up from 10 bits)
- `holdoff_control` renamed from `peak_sensitivity` (JTA 2015-04-15); default encodes pkSens=4 in bits [11:9] and holdoff=150 in bits [8:0]

---

## dp_srl_template.vhd (51 L — PEHQ SRL Delay)

**Source:** `MAIN_FPGA_ISE11/Source/dp_srl_template.vhd`  
**Entity:** `PEHQ`  
**Role:** Wrapper for the Pending Event History Queue (`PEHQ_SRL_DELAY`) SRL-based shift-register delay. Implements a 324-bit wide, up-to-16-deep FIFO using Xilinx SRL primitives. Used by `jta_channel.vhd` as the per-channel PEHQ buffer.

### Ports

| Port | Dir | Width | Description |
|------|-----|-------|-------------|
| `clk` | in | 1 | Common clock for both read and write sides |
| `rst` | in | 1 | Active-high async reset |
| `wr_en` | in | 1 | Write enable (side A) |
| `rd_en` | in | 1 | Read enable (side B) |
| `din` | in | 324 | Data to write |
| `dout` | out | 324 | Data to read |

### Internal Operation

- **Address register `a[3:0]`:** tracks occupancy (depth 0–15)
  - `wr_en=1, rd_en=0` → `a = a + 1` (enqueue)
  - `wr_en=0, rd_en=1` → `a = a - 1` (dequeue)
  - Both or neither → `a` unchanged
- **`clk_en`:** registered copy of `wr_en` (1-cycle delay before driving `PEHQ_SRL_DELAY.clk_en`)
- **`PEHQ_SRL_DELAY`:** lower-level SRL primitive instance; receives `clk`, `clk_en`, `a`, `din`, drives `dout`
- Reset is **asynchronous** (`rst` in sensitivity list); `a` resets to 0

### Notes

- The 324-bit data width matches the PEHQ entry format defined in `jta_channel.vhd` (full per-channel event descriptor)
- The SRL delay naturally implements a LIFO if `a` is held fixed; incrementing `a` on write and decrementing on read gives FIFO semantics with last-written word always at the SRL output
- `dp_srl_template.vhd` is distinct from `jta_dpram_template.vhd` (BRAM-based); this file uses SRL shift registers (lower latency, higher density for small queues)
- Uses `Digitizer_pkg.all` for type definitions

---

## See Also

- [deep_fpga_DIG_modules.md](deep_fpga_DIG_modules.md) — Part 1: signal chain, SERDES_TX, event_packer, pileup, SERDES_RX, Timestamp, Trigger_Mux, Channel_Readout_Controller/Mach
- [deep_fpga_DIG.md](deep_fpga_DIG.md) — top-level DIG architecture, source file table, SERDES RX format
- [deep_fpga_DIG_channel.md](deep_fpga_DIG_channel.md) — per-channel signal processing, LED/CFD discriminators, Trigger Rondel
- [deep_fpga_DIG_eventpacket.md](deep_fpga_DIG_eventpacket.md) — full event packet format
- [fpga.md](fpga.md) — FPGA overview and cross-reference index
