# SBX — Slope Box Extension

**Full name:** Slope Box Extension (SBX)  
**Also called:** Pickoff Card (in some contexts the pickoff is a sub-board of the SBX)  
**Source:** `DGS_SVN/dgs/SlopeBoxExtension/`, `DGS_SVN/dgs/SlopeBoxInterface/`, [wiki: The Slope Box Extension](https://wiki.anl.gov/gsdaq/The_Slope_Box_Extension)

---

## Slope Box (parent module)

The **Slope Box** is the interface between the detector and the control/monitoring system:
- Takes 48VDC power from the SBX power board
- Generates **Ge bias voltage** (programmable via EPICS DAC; typical range 3000–4800V, most common 3500V or 4000V per detector) ✅ verified 2026-04-06 — [wiki: The Slope Box](https://wiki.anl.gov/gsdaq/The_Slope_Box); range confirmed 2026-04-07 — `DGS_SVN/dgs/Documentation/North_db.csv` + `South_db.csv` (2017 install records, 82 of 110 holes: min=3000V, max=4800V, mode=4000V)
- Generates **BGO bias voltage** (~550–800V, programmable) ✅ verified 2026-04-06 — [wiki: The Slope_Box](https://wiki.anl.gov/gsdaq/The_Slope_Box)
- Has a multiplexed ADC that measures: detector temperature, actual HV, power supply values
- Connectors: preamp signal output, BGO ring signal, BGO backplug signal, HV output
- Secured to detector via a metal belt on its underside
- Slope box function unchanged after upgrade — only VXI→SBX/Collector Box replacement changed how it's controlled

*Source: [wiki: The Slope Box](https://wiki.anl.gov/gsdaq/The_Slope_Box)*

---

## Purpose

> **Hardware note:** The SBX and Pickoff Card are **pure analog PCBs** — no FPGA, no firmware. All intelligence is in the Raspberry Pi IOC.

The SBX sits between the **Slope Box** and the **Collector Box** (or directly to the digitizer for small systems). It replaces the old VXI system entirely:
- Converts single-ended analog signals from the Slope Box → **differential signals** for the Collector Box / digitizers
- Drives a **DVI-I cable** carrying: analog signals + power + communication interface → Collector Box
- Power board: provides all detector power from a single **48VDC** source
- Enables a single detector + SBX to run from one **PoE port** (standalone operation)
- Optional **Raspberry Pi** inside SBX for standalone detector operation (EDM screen for PVs)
- Eliminates extensive VXI cabling; moves all analog signal conditioning directly to the detector

*Source: [wiki: The Slope Box Extension](https://wiki.anl.gov/gsdaq/The_Slope_Box_Extension)*
- Provides **BGO pattern discrimination** (individual BGO segment hits)
- Provides **BGO sum** signal
- Handles **Pickoff** routing of signals to specific digitizer channels
- Hosts the **GS_ID dongle** — identifies which GS hole this detector occupies
- Provides **48V power distribution** from Collector Box to Slope Box electronics

---

## Signal Processing

### Input signals (from Slope Box):
- Ge Center (preamp output)
- Ge Side A / Side B
- BGO segment signals (up to 6 sides + 1 back plug = 7 BGOs)

### Output signals (to Collector Box / Digitizers):
| Signal | Description |
|--------|-------------|
| Ge Center | Primary gamma-ray energy contact |
| Ge Side | Segmented detector side contact (A or B) |
| BGO Sum | Analog sum of all 7 BGO signals — used for Compton suppression |
| BGO Pattern | Discriminated individual BGO segment bits — used for Electric Honeycomb (in development) |

### BGO Pattern channel
- Uses two ranks of analog switches + discriminators
- Generates multiplexed BGO segment discriminator pattern
- Time-sliced through MUX — firmware can stop and monitor single BGO signal
- Rate and source of MUX sync controllable via `MISC_CTL_REG` (should be moved to named register per dev notes)

---

## Pickoff Card

Sub-board within SBX. **Pure analog PCB** — no FPGA, no firmware. It is a hardwired patch panel that maps the 4 conditioned signals per detector to specific DIG input channels:

| Signal | Description |
|--------|-------------|
| Ge Center | Primary gamma-ray energy → DIG channel N |
| Ge Side A/B | Segmented contact → DIG channel N+1 |
| BGO Sum | Analog sum of 7 BGOs → DIG channel N+2 |
| BGO Pattern | Discriminated BGO bits → DIG channel N+3 |

The routing is **fixed at fabrication** (hardwired traces), not dynamic. The correct Pickoff card variant must match the physical installation (which GS hole, which Collector Box slot). Also provides BGO HV demand control via DAC (see address map below).

### BGO HV demand map (from dev notes)
| Address | BGO Ring/Pin |
|---------|-------------|
| 32 | Ring 4w, pin 5 |
| 33 | Ring 4, pin 4 |
| 34 | Ring 3, pin 5 |
| 35 | Ring 3, pin 4 |
| 36 | Ring 2, pin 5 |
| 37 | Ring 2, pin 4 |
| 38 | Ring 1, pin 5 |
| 39 | Unknown (doesn't affect ring — possibly 2nd connection to ring 1?) |
| 40 | Ring 6, pin 5 |
| 41 | Ring 6, pin 4 |
| 42 | Ring 5, pin 5 |
| 43 | Ring 5, pin 4 |
| 44/45 | Likely back plug (unconfirmed) |

✅ verified 2026-04-13 — `SlopeBoxInterface/PickoffCard/bgo notes.txt:L1-19` (addr 32–43 all confirmed; addr 39 note: "doesn't affect the ring" and may be 2nd connection to ring 1 per same source)

### BGO threshold calibration notes
- Address 19 = BGO differential offset; address 18 = BGOp threshold (documentation was backwards, fixed) ✅ verified 2026-04-09 — `SlopeBoxInterface/PickoffCard/bgo notes.txt`: "address 19 is the BGO differential offset and 18 is the BGOp threshold. correct in circuit but documentation was backwards, fixed."
- BGOp threshold noise window: ~4 DAC counts
- DAC step size: 5V/1024 = 4.88 mV → through 4.7k/6.9k divider (68.1%) → **3.32 mV per step** ✅ verified 2026-04-09 — `SlopeBoxInterface/PickoffCard/bgo notes.txt`: "Each DAC count is 5V/1024, or 4.88mV; goes through 4.7k/6.9k divider (68.1%) so is step size of 3.32mV."
- Noise window: ~13 mV (~4 DAC counts × 3.32 mV) ✅ verified 2026-04-09 — same source: "noise window is about 13mV" (4 DAC counts)

### Nominal BGO HV operating point
- **180 DAC units per tube** — established from `SVN/dgs/sbxscreens/Std_Test.sh` (JTA, 2021-04-02), the cross-test commissioning script
- BGO HV sweep range used during tuning: **0 → 250 DAC units** (from `slopebox_scripts/BGO_Sweep_test`)
- Ge/BGO HV both switched off first; BGO HV0-13 all set to 180 then BGO HV supply enabled; Ge HV enabled last
- DC offset DACs (GeCenter, GeSide, BGOsum, BGOpattern): **150 DAC units** nominal
- Ge threshold: **600 DAC units** nominal
- `PARST_AutoClampDwell`: **10000** (preamp reset clamp dwell time)

---

## GS_ID Dongle

A small board in the SBX that identifies the **GS hole number** for this detector position.
- Read by the Raspberry Pi IOC on the Collector Box via SPI
- `DNG_ID` in scan file = dongle-reported GS hole number = `DetNbr` in EPICS PVs
- Located in: `DGS_SVN/dgs/SlopeBoxExtension/GS_ID/`
- Schematic: `GS_ID.pdf`, PCB: `SlopeBoxtExt.brd`

---

## Hardware Versions

| Version | Directory | Notes |
|---------|-----------|-------|
| SBXa | `RaspberryPi/SBXa_20211107` | Earlier revision |
| SBXc3 | `RaspberryPi/sbxc3_src_20220325` | Main production version |
| SBXL | `RaspberryPi/SBXL` | Later revision |
| SBXw | `RaspberryPi/sbxw_20211108` | Another variant |

---

## EPICS IOC (Raspberry Pi)

### Full GS System
The SBX is controlled via the **Collector Box Raspberry Pi** soft IOC (one Pi per Collector Box, handles up to 28 detectors):
- **SPI1 hardware interface** via `bcm2835` library at ~50 MHz clock
- **GPIO 12** (J8 pin 32) used as ADC scanner enable/reset control line
  - `RESET_ADC_SCANNER()` → GPIO HIGH
  - `ENABLE_ADC_SCANNER()` → GPIO LOW
- Source: `collectorboxpi/CollectorBox_RevA/CollectorApp/src/spi.h`, `initTrace.c`

### Small Systems (DUO, DXA)
- Each SBX has its **own dedicated Raspberry Pi** (sbxh3 @ .164, sbxcc @ .158 for DuoGe)
- Same SPI + GPIO principle, but Pi talks directly to SBX/Pickoff hardware — **no Collector Box**
- Software is an **older version**, not present in `DGS_tools_pack` — pending exploration of sbxh3/sbxcc when online
- Source: `DGS_SVN/dgs/SlopeBoxExtension/RaspberryPi/` (SBXa, SBXc3, SBXL, SBXw variants)

### Common EPICS DB files (full GS)
- `Pickoff.db` — 448 records per detector
- `PickoffDiagCtl.db` — 40 diagnostic control records
- `Pickoff_reg.db` — 264 register-level records
- `GS${DetNbr}_SBX_Present` PV — indicates whether an SBX is connected for this GS hole

---

## Preamp Reset Kill (PRK) — SBX interaction

The SBX differentiator converts Ge preamp reset spikes into large opposite-polarity signals. The digitizer firmware's Preamp Reset Kill (PRK) detects these via an opposite-polarity discriminator and disables the main discriminator during the reset:
- **Firmware PRK holdoff** = `PREAMP_RESET_DELAY` (8-bit, from `reg_led_threshold[23:16]`) × 512 clock cycles × 10 ns = up to 1.31 ms; testbench default = 0x25 (37) = **~189 µs** ✅ verified 2026-04-06 — `thresh_disc.vhd:L661`, `one_chan_tb.vhd:L360`
- **PRK enable** = `reg_channel_control[3]` (bit 3) ✅ verified 2026-04-06 — `Digitizer.vhd:L1203`
- Reset rate: every few ms to ~100s of ms (depends on radiation damage)
- PRK dead time: not significant unless detector needs annealing
- **SBX GeCenter clamp time** (SBX Extension FPGA, `PARST_ANALOG_SWITCH_CTL` process, 100 MHz clock): `PARST_SWITCH_COUNT` default = `X"1388"` = 5000; loaded as `5000 & '0'` = 10000 counts at 100 MHz = **100 µs** default clamp duration. Register is 16-bit: bit 15 = enable flag, bits[14:0] = count (× 2 × 10 ns, max ~655 µs). ✅ verified 2026-04-06 — `SlopeboxInt_TopLevel_RevC.vhd:L552,L3208–3237`. Prior claim of "200–250 µs" was incorrect — that figure is not in the firmware source.

---

## I2C DPRAM Opcode Engine (`SlopeBoxExtension/I2C_Tools/I2C_Compiler/`)

The Ctrl FPGA and Stripe FPGA (SBX) use a **DPRAM-based I2C state machine** to drive on-board I2C devices (ADCs, DACs, power monitors, etc.). The `I2C_Compiler` is a C tool that compiles CSV-format I2C transaction sequences into 32-bit DPRAM opcodes loaded into the FPGA's I2C engine ROM.

### 32-bit Opcode Format

Each DPRAM word is 32 bits, split as UPPER_16 (bits[31:16]) and LOWER_16 (bits[15:0]):

**UPPER_16:**
| Bits | Field | Description |
|------|-------|-------------|
| [15] | CONT | Continue — 1 = more words follow for this transaction |
| [14:12] | OPCODE | Operation type (see table below) |
| [11:8] | — | Reserved/unused |
| [7:0] | DPRAM_ADDR | Address in DPRAM where read-back data is saved |

**LOWER_16 (Opcode 0b000 — I2C data word):**
| Bits | Field | Description |
|------|-------|-------------|
| [15:8] | CTRL_BYTE | Control flags (see below) |
| [7:0] | I2C_BYTE | I2C data byte (device addr+R/W, register addr, or data) |

**CTRL_BYTE bit fields (Opcode 000):**
| Bit | Flag | Meaning |
|-----|------|---------|
| [7] | DONE | Last word of transaction |
| [6] | RPTS | Repeated start condition |
| [5] | NACK | NACK expected (don't retry) |
| [4] | READ | Read transaction (1=read, 0=write) |
| [3] | SAVE | Save result to DPRAM at DPRAM_ADDR |
| [2] | EXTD | Extended (multi-byte) |
| [1] | LOOP | Loop mode |
| [0] | MACK | Master ACK |

**LOWER_16 (Opcodes 001–101 — delay/jump):** holds 16-bit delay or jump address.

**Opcodes:**
| Code | Meaning |
|------|---------|
| 0b000 | I2C transaction word |
| 0b001 | Start delay |
| 0b010 | End delay |
| 0b011 | Delay |
| 0b100 | Delay variant |
| 0b101 | Jump (relative or absolute) |
| 0b110 | Reserved |
| 0b111 | Reserved |

### CSV Input Format
The compiler reads `input.csv` with columns: `Start_Delay`, `End_Delay`, `Device_Address` (hex), `Read_or_Write`, `Reg_Address_or_Command` (hex), `Num_Reg_or_Cmd_Bytes`, `Write_Data` (hex), `Num_of_Data_Bytes`, `DPRAM_Write_Address`, `Force_Special_SAVE_SEQ` (bin), `Force_Special_Opcode` (bin).

Outputs: `output.csv` (one hex 32-bit word per line) + `listing.csv` (annotated debug listing).

The same I2C engine + opcode format is used in **both the SBX Stripe FPGA and the Collector Box Ctrl FPGA** — the flag constants in `collectorboxpi/CollectorBox_RevA/include/CollectorSupport.h` and `PickoffSupport.h` (e.g. `Collector_I2C_DONE=0x8000`, etc.) map directly to the CTRL_BYTE bits above.

*Source: `DGS_SVN/dgs/SlopeBoxExtension/I2C_Tools/I2C_Compiler/main.c` (1026 lines, J. Anderson) — explored 2026-04-12*

---

## Related Files
- `collectorboxpi.md` — Pi IOC that controls the SBX
- `collectorbox_PVs.md` — full PV list including Pickoff records
- `DIG_firmware_expert.md` — Preamp Reset Kill details
- `gammasphere_geometry.md` — GS hole numbering (what GS_ID dongle reports)

---

## SVN Location
`DGS_tools_pack/DGS_SVN/dgs/SlopeBoxExtension/`  
`DGS_tools_pack/DGS_SVN/dgs/SlopeBoxInterface/`

---

*Created: 2026-04-05 (from SVN code reading)*

## Cross-References

- `dgs/collector_fpga.md` — Pickoff card FPGAs (SBX Interface + Extension) that interface to SBX
- `dgs/collectorboxpi.md` — Raspberry Pi soft IOC; controls BGO HV via pickoff card
- `dgs/collectorbox_devicesupport.md` — EPICS device support; SPI protocol to pickoff card
- `dgs/collectorbox_PVs.md` — Full PV list including BGO HV, SBX presence, slope box PVs
- `dgs/hardware_architecture.md` — Signal chain overview: detector → slope box → pickoff → collector FPGA → digitizer
- `dgs/gammasphere_geometry.md` — GS hole numbering system used for GS_ID dongle addressing
