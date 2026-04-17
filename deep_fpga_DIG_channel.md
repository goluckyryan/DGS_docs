# DIG Firmware — Per-Channel Signal Processing

_Split from `deep_fpga_DIG.md` on 2026-04-10 (file exceeded 1200 lines)._
_Source: `DGS_tools_pack/raw_FPGA/Dig*/` — `jta_channel.vhd`, `thresh_disc.vhd`, `Digitizer.vhd`. PDF: `ANL Digitizer Firmware for Experts.pdf`._

---

## Per-Channel Signal Processing: LED and CFD Modes

Each of the 10 channels runs an identical 100 MHz pipeline implemented in `jta_channel.vhd`. The pipeline has two discriminator modes selectable per channel: **LED** (Leading-Edge Discriminator, threshold-based) and **CFD** (Constant Fraction Discriminator, zero-crossing-based). Both modes share the same upstream delay chain and filters; they differ in how they derive the discriminator signal and fire the event timestamp.

### Common Signal Path — Delay Chain and Filtering

The raw 14-bit ADC sample passes through a series of programmable delay buffers before reaching the discriminators. All delays are in 100 MHz clock cycles (10 ns steps).

```
ADC_DATA[13:0]  (14-bit, 100 MHz)
    │
    ▼ P1 delay  (reg_p1_window, default 1 cycle) ✅ verified 2026-04-10 — Registers.vhd:L216 (to_std_logic_vector(1,32))
    │
    ▼ P2 delay  (reg_p2_window, default 2 cycles) ✅ verified 2026-04-10 — Registers.vhd:L227 (to_std_logic_vector(2,32))
    │
    ▼ M delay   (reg_m_window[9:0], default 200 cycles = 2 µs) ✅ verified 2026-04-10 — Registers.vhd:L176 (to_std_logic_vector(200,32))
    │   X_M  ← pre-event buffer; holds signal before the pulse arrives
    │
    ▼ K0 delay  (lower bits of reg_k_window)
    │   X_M_K0
    │
    ▼ K delay   (upper bits of reg_k_window, default 100 cycles) ✅ verified 2026-04-10 — Registers.vhd:L166 (to_std_logic_vector(100,32))
    │   X_M_K0_K
    │
    ▼ D delay   (reg_d_window[6:0], default 10 cycles) ✅ verified 2026-04-10 — Registers.vhd:L156 (to_std_logic_vector(10,32))
    │   X_M_K0_K_D
    │
    ▼ D3 delay  (reg_d3_window[6:0], default 23 cycles) ✅ verified 2026-04-06 — Registers.vhd:L186 (to_std_logic_vector(23,32))
    │   X_M_K0_K_D_D3  ← used for baseline tracking input
    │
    ▼ TRIPLE_FILTER  (triple_filter.vhd)
    │   Cascaded moving-average filter: 3× (1-2-1) stages
    │   Smooths the signal for cleaner threshold comparison
    │   Produces two taps: PROMPT (at K0) and DELAYED (at K0+K)
    │
    ▼ Baseline subtraction
        FILTERED_SIGNAL − BASELINE_VALUE  →  discriminator inputs
```

**Triple filter:** Each stage is a (1-2-1) moving average. Three cascaded stages produce an effective kernel of [1,8,28,56,70,56,28,8,1] / 256, reducing high-frequency noise without significantly broadening the pulse.

**Baseline tracker** (`baseline_tracker.vhd`): Estimates the DC baseline by accumulating a running difference `X(n) − X(n−T)` over a 1024-sample (10.24 µs) window. ✅ verified 2026-04-10 — jta_channel.vhd:L1393 ("Accumulate 1024 samples") + L1924 ("10.24 usec prior to the pre-rise sum") It holds off updates for a programmable time after every discriminator fire (`reg_baseline_delay`) to avoid pulling the baseline onto the pulse tail.

---

### LED Mode — Leading-Edge Threshold Discriminator

In LED mode (`CFD_MODE = '0'`), the discriminator fires as soon as the filtered, baseline-subtracted signal crosses a fixed threshold. This gives a coarse timestamp tied to the signal's leading edge.

```
THRESH_DISC_PROMPT  = triple_filter output at tap X_M_K0_K  (earlier)
THRESH_DISC_DELAYED = triple_filter output at tap X_M_K0_K_D (D cycles later)

Both taps − BASELINE_VALUE
    │
    ▼ thresh_disc.vhd
    Compare THRESH_DISC_DELAYED > DISCRIMINATOR_THRESHOLD
    ─── AND ───
    Compare THRESH_DISC_PROMPT  > DISCRIMINATOR_THRESHOLD
    │
    ▼
THRESH_DISC_FLAG  (one-shot pulse)
    │
    └─→ Opens PEQ entry, starts energy integration, latches 48-bit timestamp
```

The two-tap comparison (PROMPT and DELAYED both above threshold) acts as a simple coincidence filter that suppresses single-sample noise spikes. The threshold value is set by `reg_led_threshold`.

**Timing:** The discriminator flag is asserted exactly **5 clock cycles (50 ns)** after the signal crosses threshold. ✅ verified 2026-04-14 — `thresh_disc.vhd:L259-260` (MBO comment 2014-09-12: "5 clocks from input to disc flag"; one pipeline delay added to PROMPT_INPUT to match CFD discriminator timing).

---

### CFD Mode — Constant Fraction Discriminator

In CFD mode (`CFD_MODE = '1'`), the discriminator fires at the zero-crossing of `(fraction × prompt_signal) − delayed_signal`. Because the zero-crossing position on the pulse shape is independent of amplitude, CFD gives significantly better timing resolution than LED for pulses of varying heights.

```
CFD_PROMPT  = triple_filter tap at X_M_K0_K    (same as LED prompt)
CFD_DELAYED = triple_filter tap at X_M_K0_K_D  (D cycles later)

Step 1 — Pre-trigger (thresh_disc.vhd fires first as a gate):
    THRESH_DISC_FLAG fires on leading edge (same as LED but used only as a gate)
    → triggers CFD_SAMPLE_ZERO: latches LOCAL_ZERO = current CFD_SUBTRACTION value
    → after K cycles, asserts CFD_PRE_TRIGGER

Step 2 — Fraction multiply (MULT17×17, 34-bit result):
    FRACTIONAL_PROMPT = CFD_PROMPT × CFD_FRACTION >> 13
    (CFD_FRACTION register encodes the fraction as N/8192;
     e.g. reg_cfd_fraction = 0x0C00 ≈ 75% of full scale)

Step 3 — CFD subtraction:
    CFD_SUBTRACTION = FRACTIONAL_PROMPT − CFD_DELAYED

Step 4 — Zero-crossing detection (cfd_disc.vhd):
    LOCAL_DIFFERENCE = CFD_SUBTRACTION − LOCAL_ZERO
    Track sign of LOCAL_DIFFERENCE each clock cycle
    When sign flips → CFD_DISC_FLAG asserted, 48-bit timestamp latched
    Three CFD_SAMPLES captured around the crossing for interpolation
```

The zero-crossing tracks the point on the pulse where `fraction × amplitude = delayed_amplitude`, which moves in time but not in amplitude — giving the amplitude-independent timestamp.

**Key difference from LED:** The timestamp latched in CFD mode is the zero-crossing time, not the threshold-crossing time. This typically improves coincidence timing resolution from ~10 ns (LED) to ~1.7–2.5 ns (CFD) for germanium detectors. ✅ verified 2026-04-14 — `DIG_firmware_expert.md:L100` ("~1.7 ns (1σ) for large signals, ~2.5 ns for small signals at 800–1000 ns rise time" — from ANL Digitizer Firmware for Experts PDF)

---

### Mode Selection

| Register | Address (Ch 0) | Bit | Function |
|----------|---------------|-----|----------|
| `reg_channel_control` | `0x040` | `CFD_MODE` | `0` = LED, `1` = CFD |

Channels 1–9 use addresses `0x044` through `0x064` (4-byte spacing). The `CFD_MODE` bit is distributed as four copies (`xCFD_MODE[3:0]`, with KEEP attribute) inside `jta_channel.vhd` to avoid long-path timing issues.

---

### After Discrimination — PEQ and Energy Integration

When a discriminator fires (LED or CFD), the channel opens a slot in the **Pending Event Queue (PEQ)** — a 16-deep FIFO. The event remains pending until the trigger decision arrives from the Router (~2–4 µs later, within the ~20 µs TRIG_DELAY window). During that time, three energy sums are accumulated:

```
Discriminator fire
    │
    ├─ Latch 48-bit timestamp
    ├─ Open PEQ entry
    │
    ├─→ PRE_RISE integration
    │     Accumulates M cycles of samples before the pulse peak
    │     Duration: reg_m_window[9:0] clock cycles
    │
    ├─→ POST_RISE integration
    │     Accumulates samples from peak onwards
    │     Starts at PEAK_FLAG (peak-finding algorithm in thresh_disc.vhd)
    │
    └─→ P2 integration  (tail sum)
          Accumulates after POST_RISE for additional baseline/tail correction
          Duration: reg_p2_window[9:0] clock cycles

Trigger decision arrives (~2–4 µs later):
    Accepted → pack (timestamp + PRE_RISE + POST_RISE + P2 + pileup flags)
               into 36-bit external FIFO for VME readout
    Rejected → discard PEQ entry silently
```

In CFD mode with `CFD_ESUM_MODE = '1'`, the energy integration start is deferred to `THRESH_DISC_FLAG_DELAYED` (the LED crossing) rather than the CFD zero-crossing, so energy always integrates the same portion of the pulse regardless of discriminator mode.

---

### Pileup Detection

The **pileup processor** (`pileup_processor.vhd`) tracks how many events are in-flight (discriminator fired but not yet readout-complete). It uses a 4-bit counter and an 8-state machine:

```
States: IDLE → ONE_HIT → MANY_HIT → OVERFLOW
        (each with ACC or REJ variant)

Counter increments: on each THRESH_DISC_FLAG
Counter decrements: on PILE_RELEASE_DLYD (end-of-event holdoff pulse)

PILEUP_DISABLE register:
    0 → reject second and subsequent pileup hits (standard spectroscopy)
    1 → accept all hits (pileup recording mode)

Outputs per event:
    ACCEPTED_HIT   — first hit (or any hit in accept-all mode)
    EXTENDED_EVENT — subsequent pileup hits (accept-all mode only)
    PILEUP_FLAG    — level: counter > 0
    OVERFLOW_FLAG  — counter saturated at 15
    PU_TOO_SHORT   — pileup interval shorter than retrigger holdoff; event invalid
```

The holdoff time (`reg_holdoff_control[8:0]`) controls both the minimum inter-event spacing and the peak-finding window; it is shared between the threshold discriminator and the pileup counter.

---

### VME Registers for Discriminator Configuration

All addresses are per-channel. Channel 0 uses the base address shown; channels 1–9 add `4 × channel_number`.

**Discriminator mode and thresholds:**

| Register | Ch 0 Address | Bits | Description |
|----------|-------------|------|-------------|
| `reg_channel_control` | `0x040` | `CFD_MODE` bit | `0` = LED, `1` = CFD |
| `reg_led_threshold` | `0x080` | `[13:0]` | Threshold in ADC counts (both LED and CFD pre-gate) |
| `reg_cfd_fraction` | `0x0C0` | `[12:0]` | CFD fraction encoded as N/8192 (e.g. `0x0C00` ≈ 75%) |
| `reg_external_disc_mode` | `0x420` | 2 bits/ch | `00`=normal, `01`=OR with external, `10`=AND, `11`=external only |

**Delay chain:**

| Register | Ch 0 Address | Bits | Description |
|----------|-------------|------|-------------|
| `reg_p1_window` | `0x300` | `[3:0]` | P1 delay (cycles) |
| `reg_p2_window` | `0x404` | `[9:0]` | P2 delay and tail-sum window (cycles) |
| `reg_m_window` | `0x200` | `[9:0]` | M delay = pre-event buffer depth (cycles) |
| `reg_k_window` | `0x1C0` | `[13:0]` | K0+K combined delay (cycles) |
| `reg_d_window` | `0x180` | `[6:0]` | D delay — sets CFD fraction delay (cycles) |
| `reg_d3_window` | `0x240` | `[6:0]` | D3 delay — baseline tracker input offset (cycles) |

**Baseline tracking:**

| Register | Ch 0 Address | Bits | Description |
|----------|-------------|------|-------------|
| `reg_baseline_start` | `0x2C0` | `[13:0]` | Initial baseline value (ADC counts) |
| `reg_baseline_delay` | `0x418` | `[7:0]` | Holdoff after disc fire before resuming tracking (× 10.24 µs) |
| `reg_baseline_delay` | `0x418` | `[10:8]` | Baseline update step size (tracking speed) |

**Holdoff and peak finding:**

| Register | Address | Bits | Description |
|----------|---------|------|-------------|
| `reg_holdoff_control` | `0x414` | `[8:0]` | Retrigger holdoff duration (cycles × 10 ns) |
| `reg_holdoff_control` | `0x414` | `[11:9]` | Peak sensitivity (controls peak-finding rate) |
| `reg_disc_width` | `0x280` | `[7:0]` | Discriminator output pulse width (cycles) |

**Waveform capture:**

| Register | Ch 0 Address | Bits | Description |
|----------|-------------|------|-------------|
| `reg_raw_data_length` | `0x100` | `[9:0]` | Number of waveform samples to capture |
| `reg_raw_data_window` | `0x140` | `[10:0]` | Capture window relative to discriminator fire (samples) |

**Diagnostic counters (read-only):**

| Register | Ch 0 Address | Description |
|----------|-------------|-------------|
| `reg_disc_count` | `0x7C0` | Total discriminator fires |
| `reg_accepted_event_count` | `0x740` | Events accepted by trigger |
| `reg_dropped_event_count` | `0x700` | Events dropped (FIFO full or vetoed) |
| `reg_ahit_count` | `0x780` | Accepted-hit pulses from pileup processor |

---

## VME FPGA

**Location:** `VME_FPGA_ANL/`

| Field | Value |
|-------|-------|
| Part | xc3s400 (Spartan-3) |
| Package | fg320 |
| Speed Grade | -5 |
| Tool | ISE 13.4 |
| Project | `VME_FPGA_ANL/Work11/vme_A32_D32.xise` |
| Top Entity | `vme_top` |

Same architecture as the MTRG VME FPGA: acts as A32/D32 VME slave, programs the main FPGA from external flash, and bridges host VME commands to the main FPGA.

### Source Files
| File | Description |
|------|-------------|
| `TOP.VHD` | Top-level entity (`vme_top`) |
| `vme_addr_decode.vhd` | VME address space decoder |
| `external_bus_controller.vhd` | Flash/FPGA bus multiplexer |
| `configuration_controller.vhd` | FPGA programming sequencer |
| `register_block.vhd` | Status and control registers |
| `register_block_FlashHi.vhd` | Upper flash address register block |

### Bitfiles
| File | Description |
|------|-------------|
| `Work11/vme_top.bit` | Standard VME FPGA bitfile |
| `Work11/vme_top_usehi.bit` | Variant using upper flash address |
| `Work11/20230928.mcs` | MCS flash image (Sept 2023) |
| `Work11/20230928_usehi.mcs` | MCS flash image, upper flash variant |

### Clock Select Register (`clk_select`)

**VME address:** `0x0910` bits[1:0] in `register_block.vhd`

Controls which clock the digitizer uses as its system clock. The two bits (`sysclk_sel[1:0]`) drive physical output pins `sysclk_sel0_out` (B9) and `sysclk_sel1_out` (B10) off-chip to a hardware clock mux on the digitizer PCB.

**Important:** the register bits are **inverted** before the output pins (`sysclk_sel0_out <= NOT sysclk_sel0`) to match the original LBL digitizer board design. EPICS values are correct end-to-end — the inversion is transparent to software.

Default at reset: `sysclk_sel0=1, sysclk_sel1=0` → OSC (local oscillator). ✅ verified 2026-04-17 — `VME_FPGA_ANL/Source/register_block.vhd:L165-166` (`sysclk_sel0 <= '1'; sysclk_sel1 <= '0'; -- initialize to use internal clock`)

| `clk_select` EPICS value | `sel[1:0]` | Meaning |
|---|---|---|
| 0 | `00` | S/D — SERDES derived (link clock from Router) |
| 1 | `01` | **OSC** — local on-board oscillator (default) |
| 2 | `10` | S/D — same as 0 |
| 3 | `11` | AUX — auxiliary clock input |

✅ verified 2026-04-17 — `MDigUserVME.template` (`clk_select` mbbo: ZRST=S/D, ONST=OSC, TWST=S/D, THST=AUX); `register_block.vhd:L351-352` (write to 0x0910 sets sysclk_sel[1:0] from VME_data_in[1:0])

**EPICS PV:** `VME$(CRATE):$(BOARD):clk_select` (mbbo, `MDigUserVME.template` / `SDigUserVME.template`)

**Usage in `link_sys.py`:**
- Stage 4A: `clk_select=1` (OSC) — initialize DIGs on independent local clock first
- Stage 4E: `clk_select=0` (S/D) — switch DIGs to Router-derived link clock for full timestamp sync

## Main FPGA Bitfiles

| Branch | Bitfile | Description |
|--------|---------|-------------|
| DGS | `DGS/Work/BUS_LEFT.bit` | Production — front bus sender role |
| DGS | `DGS/Work/BUS_RIGHT.bit` | Production — front bus receiver role |
| Majorana | `Majorana/Work/digitizer.bit` | MAJORANA experiment variant |
| DGS_QUAD_M_SUMS | `DGS_QUAD_M_SUMS/Work/FB_SENDER.bit` / `FB_RCVR.bit` | Quad M-sum, sender/receiver |
| SumOverRise | `SumOverRise/Work/FB_SENDER.bit` / `FB_RCVR.bit` | Sum-over-rise, sender/receiver |
| DGSBramTest | `DGSBramTest/Work/MSTR_digitizer.bit` | BRAM test |
| — | `tag_4975_mod_fifo_digitizer.bit` (root) | Tagged release build |
| — | `Walter_Release_MDIG_6194/MSTR_digitizer-6194.bin` | Release candidate v6194 |

Note: DGS branch produces two bitfiles (`BUS_LEFT` / `BUS_RIGHT`) because the `FRONT_BUS_LEFT` generic changes the front bus direction — two digitizer modules are paired, one sender and one receiver.

## IP Cores

Located in each branch's `Cores/` directory:

| Core | Description |
|------|-------------|
| `chipscope_icon` | ChipScope controller |
| `chipscope_ila` | ChipScope logic analyzer |
| `fifo_16x1023_async` | 16-bit, 1K deep async FIFO |
| `fifo_16x64K_async` | 16-bit, 64K deep async FIFO |
| `BRAM_1024X16_REGSHADOW` | Block RAM register shadow |

---

## See Also

- `knowledgeBase/deep_fpga_DIG.md` — DIG firmware overview: Spartan-3 architecture, ADC pipeline, event packet format, master/slave config, FIFO readout (this file is a continuation of that)
- `knowledgeBase/fpga.md` — System-level overview: trigger hierarchy, signal flow, PEQ explanation, end-to-end timeline
- `knowledgeBase/DIG_firmware_expert.md` — Operator-level guide: all 8 readout modes, register summary, discriminator config
- `knowledgeBase/deep_fpga_RTRG.md` — Router firmware: multiplicity aggregation, throttle, VME register map
- `knowledgeBase/deep_fpga_MTRG_MAIN.md` — Master trigger firmware: trigger algorithms, TAC-II TDC, 20-frame commands
- `knowledgeBase/ttcl.md` — TTCL spec: full frame-by-frame breakdown of the 20-frame command structure DIG receives
- `knowledgeBase/ANLDAQ.md` — DAQ software: `class_DIG.h` decodes DIG packet format documented here
- `knowledgeBase/connectors.md` — DIG connector pinouts: RJ45 SERDES, 36-pin Aux I/O, RTRG IEC cable
- `knowledgeBase/deep_fpga_building.md` — Build toolchain: ISE 14.7 on Ubuntu 24.04, Docker/Podman approach
- `knowledgeBase/preamp_reset_readme.md` — Detailed explanation of preamp reset (PRK) detection logic, blanking timing, BGO veto gate, and PARST timestamp fields

---
*Source: `DGS_tools_pack/raw_FPGA/Dig*/` — VHDL source. PDF: `ANL Digitizer Firmware for Experts.pdf`. Created: 2026-04-05.*
