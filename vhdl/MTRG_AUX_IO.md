# MTRG AUX_IO.VHD — Auxiliary I/O & Target Wheel Interface

**Source:** `FPGA/MTRG/Firmware/MAIN_FPGA/trunk/Source/AUX_IO.VHD` (786 lines)  
**Entity:** `aux_io`  
**Author:** John Anderson (ANL)  
**Stability:** C3 - Structural / stable  
**Last analyzed:** 2026-04-24

---

## Overview

`aux_io.vhd` implements the MTRG auxiliary I/O port control logic. It multiplexes two 8-bit bidirectional I/O ports (AUX_PORT_A, AUX_PORT_B) and two NIM outputs between multiple signal sources, provides target wheel encoder address decoding and filtering/sliding, and includes an SSI (Synchronous Serial Interface) receiver state machine for reading absolute encoder values.

---

## Ports

| Port | Dir | Width | Description |
|------|-----|-------|-------------|
| CLK | in | 1 | 50 MHz master clock |
| RST | in | 1 | Active-high master reset |
| RESET_SSI_MACH | in | 1 | Reset specific to SSI state machine |
| SSI_MACH_GO | in | 1 | Enables SSI machine (setup complete) |
| SSI_CTL_REG | in | 16 | SSI control register from VME |
| SSI_ENCODE_TIME | in | 16 | SSI encode period timeout (VME) |
| SSI_STATE_MON | out | 3 | SSI state machine monitor (for ILA) |
| SSI_COUNT_MON | out | 4 | SSI monitor counters |
| AUX_IO_CTL_REG | in | 16 | AUX I/O direction/mux control (VME) |
| AUX_IO_DATA_REG | in | 16 | Manual data value for AUX output pins (VME) |
| xAUX_PORT_A_IN | in | 8 | Raw AUX port A inputs from I/O buffer |
| xAUX_PORT_B_IN | in | 8 | Raw AUX port B inputs from I/O buffer |
| TS_EDGE_FLAG_SEL | in | 3 | Selects which TS edge flag to route to NIM_OUT2 |
| TS_EDGE_FLAGS | in | 8 | One-tick-wide flags for timestamp bit 0→1 transitions |
| SELECTED_SLOW_CLOCK | in | 1 | Decade-programmable clock signal |
| TARGET_WHEEL_AUX_CTL_REG | in | 16 | Target wheel control register (VME) |
| AUX_INPUT_STATE | out | 16 | Sampled value of AUX I/O as inputs (readback) |
| ENCODER_ADDRESS | out | 10 | 10-bit encoder address for target wheel RAMs |
| ENCODER_EXTRA | out | 1 | Extra encoder bit (not in address) |
| AUX_TRIG_WIDTH_REG | in | 16 | Trigger pulse width control |
| TRIG_FROM_TRIG_RAM | in | 1 | Target wheel trigger decode |
| SWEEP_FROM_SWEEP_RAM | in | 1 | Target wheel beam sweep decode |
| SWEEP_RAM_ADDRESS | in | 10 | Sweep RAM address (multiplexed back into AUX_INPUT_STATE) |
| VETO_FROM_VETO_RAM | in | 1 | Target wheel trigger veto decode |
| SLOW_CLOCK | in | 1 | Timestamp bit 16 (~655.36 µs period) |
| FAST_STROBE_IN | in | 1 | Fast strobe from CPLD |
| TRIGGER_DECISION | in | 1 | High whenever a trigger decision is made |
| SYNC_COMMAND | in | 1 | SYNC command issued by master state machine |
| REMOTE_SYNC_FLAG | in | 1 | Sync from remote monarch |
| LATCHED_IMPERATIVE_FLAG | in | 1 | Set if this is an Imperative Sync |
| DIAGNOSTIC_BITS_IN | in | 18 | Diagnostic signal bus (17/16=NIM_OUT 2/1, 15..8=AUX_A, 7..0=AUX_B) |
| DIAG_NIM_IN | in | 2 | Manually-controlled NIM output bits |
| BUF_SYSCLK | in | 1 | Buffered system clock (added 2025-06-28 for TDC) |
| AUX_A_PORT_OUT | out | 8 | AUX_A output pins |
| AUX_B_PORT_OUT | out | 8 | AUX_B output pins |
| NIM_OUT1 | out | 1 | NIM output 1 |
| NIM_OUT2 | out | 1 | NIM output 2 |

---

## AUX_IO_CTL_REG Bit Map

Controls which signal is routed to each AUX output pin group and configures input direction:

| Bits | Pins | Mux Selection |
|------|------|---------------|
| 1:0 | AUX_A[3:0] | 00=FAST_STROBE, 01=ANY_TRIG_PULSE, 10=DATA_REG[3:0], 11=DIAGNOSTIC | ✅ verified 2026-04-24 — AUX_IO.VHD:L163–181 |
| 2 | — | AUX_A[3:0] input enable (set=inputs active) |
| 4:3 | AUX_A[7:4] | 00=FAST_STROBE, 01=ANY_TRIG_PULSE, 10=DATA_REG[7:4], 11=DIAGNOSTIC | ✅ verified 2026-04-24 — AUX_IO.VHD:L185–202 |
| 5 | — | AUX_A[7:4] input enable |
| 7:6 | AUX_B[3:0] | 00=FAST_STROBE, 01=ANY_TRIG_PULSE, 10=DATA_REG[11:8], 11=DIAGNOSTIC (B1=ISYNC_PULSE) | ✅ verified 2026-04-24 — AUX_IO.VHD:L203–220 (B1 uses ISYNC_PULSE in mode 11, others use DIAGNOSTIC_BITS_IN) |
| 8 | — | AUX_B[3:0] input enable |
| 10:9 | AUX_B[7:4] | 00=FAST_STROBE, 01=ANY_TRIG_PULSE, 10=DATA_REG[15:12], 11=DIAGNOSTIC | ✅ verified 2026-04-24 — AUX_IO.VHD:L221–234 |
| 11 | — | AUX_B[7:4] input enable |
| 13:12 | NIM_OUT1 | 00=RAM_MUX_OUT1, 01=ANY_TRIG_PULSE, 10=SYNC_PULSE, 11=BUF_SYSCLK (was FAST_STROBE before 2025-06-28) |
| 15:14 | NIM_OUT2 | 00=RAM_MUX_OUT2, 01=ANY_TRIG_PULSE, 10=ISYNC_PULSE, 11=REMOTE_SYNC_FLAG | ✅ verified 2026-04-24 — AUX_IO.VHD:L248–252 |

### Notes
- **AUX_B[1]** in mode `11` outputs `ISYNC_PULSE` rather than DIAGNOSTIC, providing an "ECL clock+sync" output option for external detectors when combined with a clock copy on AUX_B[0].
- NIM_OUT1 bit[13:12]=`11` was changed from FAST_STROBE to BUF_SYSCLK on 2025-06-28 for TDC support. ✅ verified 2026-04-24 — AUX_IO.VHD:L244–249 (comment `--modified 20250628 for TDC`; old FAST_STROBE commented out, BUF_SYSCLK active)

---

## TARGET_WHEEL_AUX_CTL_REG Bit Map

| Bits | Description |
|------|-------------|
| 2:0 | NIM_OUT1 sub-mux (RAM_MUX_OUT1): 000=TRIG_RAM, 001=VETO_RAM, 010=SWEEP_RAM, 011=ENCODER_EXTRA, 100=FILTERING_IN_PROGRESS, 101=DIAG_NIM_IN[0], 110=ENCODER_CHANGE_PULSE, 111=SELECTED_SLOW_CLOCK | ✅ verified 2026-04-24 — AUX_IO.VHD:L141–150 |
| 5:3 | NIM_OUT2 sub-mux (RAM_MUX_OUT2): 000=TRIG_RAM, 001=VETO_RAM, 010=SWEEP_RAM, 011=BEAM_SWEEP_OUT, 100=FILTERING_IN_PROGRESS, 101=DIAG_NIM_IN[1], 110=ENCODER_CHANGE_PULSE, 111=SEL_TS_EDGE_FLAG | ✅ verified 2026-04-24 — AUX_IO.VHD:L151–160 (note: mode 111 is SEL_TS_EDGE_FLAG; SELECTED_SLOW_CLOCK is commented out) |
| 6 | FILTERED_ENCODER_CHANGE pulse enable for non-zero encoder addresses (0=only pulse on zero, 1=pulse on any change) |
| 7 | Encoder bit polarity: 0=inverted (default, matches analog GS/FMA direction), 1=non-inverted |
| 15:8 | FILTER_COUNTER initial value (parallel encoder debounce depth, or BEAM_SWEEP pulse width in SLOW_CLOCK ticks) |

---

## Processes

### STRETCHA — ANY_TRIG_PULSE
- Generates a stretched pulse on `ANY_TRIG_PULSE` whenever `TRIGGER_DECISION` is asserted
- Pulse width = `AUX_TRIG_WIDTH_REG[3:0]` clock ticks (at 50 MHz → up to 15 × 20 ns = 300 ns)  
  ✅ verified 2026-04-24 — AUX_IO.VHD:L262–278 (`ANY_TRIG_COUNT <= AUX_TRIG_WIDTH_REG(3 downto 0)` on TRIGGER_DECISION)

### STRETCHB — SYNC_PULSE / ISYNC_PULSE
- On `SYNC_COMMAND`: sets `SYNC_COUNT = 10` (decimal), holds `SYNC_PULSE` high for 10 ticks (~200 ns)
- `ISYNC_PULSE` is only set if `LATCHED_IMPERATIVE_FLAG` is also set at the time of the sync command  
  ✅ verified 2026-04-24 — AUX_IO.VHD:L280–299 (`SYNC_COUNT <= "1010"` = 10; `ISYNC_PULSE <= LATCHED_IMPERATIVE_FLAG`)

### STRETCHC — ENCODER_CHANGE_PULSE
- Same pulse-stretcher pattern applied to `FILTERED_ENCODER_CHANGE`
- Pulse width = `AUX_TRIG_WIDTH_REG[3:0]` ticks (shared with STRETCHA)

### SWEEP_PROC — BEAM_SWEEP_OUT
Mode selected by `AUX_TRIG_WIDTH_REG[7:6]`:
- `00`: Level from SWEEP_RAM (raw)
- `01`: ENCODER_EXTRA bit
- `10`: Rising-edge-triggered pulse from SWEEP_RAM, width = `AUX_TRIG_WIDTH_REG[15:8]` SLOW_CLOCK ticks (×655.36 µs)
- `11`: Rising-edge-triggered pulse from ENCODER_EXTRA, same width encoding

### AUX_READ_PROC — Encoder Address & Input Sampling
- **AUX_INPUT_STATE** readback (16-bit):
  - bits 15:13 = AUX_A_INPUT[7:5] (unused in encoder addressing)
  - bits 12:11 = AUX_B_INPUT[1:0] (ECL bits)
  - bit 10 = ENCODER_EXTRA
  - bits 9:0 = SWEEP_RAM_ADDRESS (which may equal CURRENT_ENCODER_ADDRESS)
- **Encoder bit mapping** (AUX_B[3..7] → addr[9..5], AUX_A[0..4] → addr[4..0]):
  - Default (bit7=0): bits **inverted** so rotation direction matches analog GS/FMA setup (per 20161207 note)
  - Alternate (bit7=1): bits non-inverted
- **SSI override** (SSI_CTL_REG[15]=1): encoder address comes from `LATCHED_SSI_DATA`, not parallel pins; 3-bit shift selector `SSI_CTL_REG[10:8]` picks which 10-bit window of the 16-bit SSI data to use as encoder address
- **Input direction**: controlled by AUX_IO_CTL_REG bits 2, 5, 8, 11 (each enables a 4-bit group for input)

### AUX_READ_FILTER — Encoder Address Debounce/Slide FSM
State machine: `SETUP → WAIT_DIFFERENT → FILTER/SLIDE → SLIDE_DELAY`

**Parallel encoder mode** (SSI_CTL_REG[15]=0) — debounce filter:
- `WAIT_DIFFERENT`: watches for input change; on change → `FILTER`
- `FILTER`: counts `FILTER_COUNTER` (= TARGET_WHEEL_AUX_CTL_REG[15:8]) stable samples before accepting new encoder address
  - Any further change restarts the counter
  - On stable completion: asserts new `ENCODER_ADDRESS`, fires `FILTERED_ENCODER_CHANGE` (if address=0 or bit6=1)
  
**Serial encoder mode** (SSI_CTL_REG[15]=1) — sliding transition:
- `WAIT_DIFFERENT`: on change → `SLIDE`
- `SLIDE`: increments `SLIDING_ENCODER_ADDRESS` one step at a time toward `CURRENT_ENCODER_ADDRESS`, updating `ENCODER_ADDRESS` at each step
- `SLIDE_DELAY`: 99-tick delay (~2 µs at 50 MHz) between steps to avoid overflowing trigger logic ✅ verified 2026-04-24 — AUX_IO.VHD:L568 (`FILTER_COUNTER <= X"63"` = 99 ticks × 20 ns = 1.98 µs)
- Maximum target wheel speed: 6000 RPM × 1024 codes/rev → one code every ~9.7 µs; 2 µs/step is safe

### SSI_PORT_SEL_PROC — SSI Pin Selection
- `SSI_CTL_REG[7:4]` selects which AUX_B/AUX_A pin pair carries SSI clock and data
- 16 options map to paddle card connector positions (J34–J37) and front-panel/header positions
- Bit 7 set → inverts both data and clock signals

### SSI_RECEIVER — SSI State Machine
States: `IDLE → WAIT_RISE_EDGE → WAIT_ENCODE_TIME → WAIT_FIRST_FALL → DATA_CLK_HIGH → DATA_CLK_LOW → DONE`

SSI protocol (absolute encoder, e.g., Heidenhain):
- After end of data transfer, SSI clock idles high for ~20 µs (encode time)
- Next falling edge = start of new transaction, MSB first
- `SSI_ENCODE_TIME` (VME-programmable 16-bit counter) sets the timeout to distinguish idle-high from data-high
- `SSI_CTL_REG[3:0]` = transaction length (max 15 bits; data collected MSB-first into `SSI_DATA_VALUE`)
- Data latched into `LATCHED_SSI_DATA` on `DONE` for stable readout; machine loops back to `WAIT_RISE_EDGE`
- `RESET_SSI_MACH` or RST → machine returns to `IDLE`

---

## AUX_INPUT_STATE Register (read-back, 16-bit)

| Bits | Source |
|------|--------|
| 15:13 | AUX_A_INPUT[7:5] |
| 12:11 | AUX_B_INPUT[1:0] (ECL bits) |
| 10 | ENCODER_EXTRA |
| 9:0 | SWEEP_RAM_ADDRESS (= current encoder address when parallel mode; note: assumption from 20161207 comment) |

---

## Key Design Notes

1. **Polarity inversion** (20161207): default encoder bit polarity is inverted so that the same physical wheel rotation increments address on both digital and analog Gammasphere/FMA setups. ✅ verified 2026-04-24 — AUX_IO.VHD:L393–402 (comment: "after testing at DFMA it was determined that the bits should be inverted"; `TARGET_WHEEL_AUX_CTL_REG(7)='0'` = inverted default)
2. **NIM_OUT1 mode 11** changed from FAST_STROBE to BUF_SYSCLK on 2025-06-28 to support TDC readout.
3. **AUX_B[1] ECL clock+sync**: when AUX_IO_CTL_REG[7:6]=11, AUX_B[0]=MCLK copy (DIAGNOSTIC_BITS_IN[8]) and AUX_B[1]=ISYNC_PULSE; together they provide an ECL "clock and imperative sync" output for external detector interfaces.
4. **SLIDE mode risk**: if jump in encoder value is large, SLIDE loop runs N × 2 µs; target wheel max speed means this can't overrun trigger logic in practice.
5. This module is instantiated in `top.vhd`; its output `ENCODER_ADDRESS` feeds the three lookup RAMs (TRIG/VETO/SWEEP RAMs) in `registers.vhd` for target wheel trigger/veto/sweep control.

---

## See Also
- [MTRG_top.md](MTRG_top.md) — top-level instantiation
- [MTRG_registers.md](MTRG_registers.md) — VME register map; includes TRIG/VETO/SWEEP RAMs that consume ENCODER_ADDRESS
- [deep_fpga_MTRG_MAIN.md](../deep_fpga_MTRG_MAIN.md) — high-level MTRG firmware overview
