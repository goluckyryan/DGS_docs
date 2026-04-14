# DGS Connector Pinouts

_Created: 2026-04-05. Last updated: 2026-04-06._

---


**Source:** Digitizer-Specification-RevA-v2.0.pdf (GRETINA/DGS Digitizer board)

---

## 1. RJ45 — SER/DES (TTCL) Interface

The RJ45 connector on the digitizer is **not Ethernet-compatible**. It connects to the Router Trigger (RTRG) via a custom shielded RJ45-to-2mm hard Metric cable.

Carries: TTCL commands (receive), fast event data (transmit), and the 50 MHz system clock (recovered by SER/DES).

```
  RJ45 (viewed from front, tab down)
  ┌──────────────────────┐
  │ 1  2  3  4  5  6  7  8 │
  └──────────────────────┘
  Pin 1 = left, Pin 8 = right
```

| Pin | Signal | Description |
|-----|--------|-------------|
| 1 | Auxiliary In0 + | Aux input 0, positive |
| 2 | Auxiliary In0 − | Aux input 0, negative |
| 3 | SerDes TX Out + | Fast data out to trigger, positive |
| 4 | Auxiliary In1 + | Aux input 1, positive |
| 5 | Auxiliary In1 − | Aux input 1, negative |
| 6 | SerDes TX Out − | Fast data out to trigger, negative |
| 7 | SerDes RX Out + | TTCL in from trigger, positive |
| 8 | SerDes RX Out − | TTCL in from trigger, negative |

✅ verified 2026-04-11 — `Digitizer-Specification-RevA-v2 0.pdf` p.48 (Table: RJ-45 Connector Pin Assignments — exact match)

> ⚠️ Some signals have inverted logic (marked in original schematic with \*). Check schematic `31Y334-Schematic-10ChanDigitizer-Rev4.2.pdf` for exact polarity.

**SER/DES IC:** National Semiconductor DS92LV18TVV (LVDS, 1 Gbps) ✅ verified 2026-04-08 — `DGS_SVN/dgs/FromT/l2.txt:L62` (BOM entry: qty=2, mfr=National, DigiKey=DS92LV18TVV-ND)

---

## 2. Auxiliary I/O — 36-Pin Front Panel Header

**Connector type:** Standard 0.1" × 0.1" (2.54 mm) pitch header, 3 columns × 12 rows = 36 pins  
**Signal standard:** RS485 differential pairs (high-speed)  
**Direction:** Configurable in increments of 2 signal pairs (inputs or outputs)

### I/O Configuration Options

| Inputs | Outputs |
|--------|---------|
| 11 | 0 |
| 10 | 1 |
| 8 | 2 |
| 6 | 4 |
| 4 | 6 |
| 2 | 8 |
| 1 | 10 |
| 0 | 11 |

### Full Pinout

```
  36-pin Header (3 columns × 12 rows, 2.54mm pitch)
  Viewed from front panel (pin 1 = top-left)

  Col: [1=GND] [2=Sig+] [3=Sig−]
  ┌────────┬──────────────┬──────────────┐
  │ Pin 1  │   Pin 2      │   Pin 3      │  Row 1: GND | Aux0 +  | Aux0 −
  │ Pin 4  │   Pin 5      │   Pin 6      │  Row 2: GND | Aux1 +  | Aux1 −
  │ Pin 7  │   Pin 8      │   Pin 9      │  Row 3: GND | Aux2 +  | Aux2 −
  │ Pin 10 │   Pin 11     │   Pin 12     │  Row 4: GND | Aux3 +  | Aux3 −
  │ Pin 13 │   Pin 14     │   Pin 15     │  Row 5: GND | Aux4 +  | Aux4 −
  │ Pin 16 │   Pin 17     │   Pin 18     │  Row 6: GND | Aux5 +  | Aux5 −
  │ Pin 19 │   Pin 20     │   Pin 21     │  Row 7: GND | Aux6 +  | Aux6 −
  │ Pin 22 │   Pin 23     │   Pin 24     │  Row 8: GND | Aux7 +  | Aux7 −
  │ Pin 25 │   Pin 26     │   Pin 27     │  Row 9: GND | Aux8 +  | Aux8 −
  │ Pin 28 │   Pin 29     │   Pin 30     │  Row10: GND | Aux9 +  | Aux9 −
  │ Pin 31 │  *Pin 32*    │  *Pin 33*    │  Row11: GND |*Aux10+* |*Aux10−* ← AUX_DIN[10] ExtTTL
  │ Pin 34 │   Pin 35     │   Pin 36     │  Row12: GND | ClkIn + | ClkIn −
  └────────┴──────────────┴──────────────┘
```

| Pin | Function | Pin | Function | Pin | Function |
|-----|----------|-----|----------|-----|----------|
| 1 | GND | 2 | Aux0 I/O + | 3 | Aux0 I/O − |
| 4 | GND | 5 | Aux1 I/O + | 6 | Aux1 I/O − |
| 7 | GND | 8 | Aux2 I/O + | 9 | Aux2 I/O − |
| 10 | GND | 11 | Aux3 I/O + | 12 | Aux3 I/O − |
| 13 | GND | 14 | Aux4 I/O + | 15 | Aux4 I/O − |
| 16 | GND | 17 | Aux5 I/O + | 18 | Aux5 I/O − |
| 19 | GND | 20 | Aux6 I/O + | 21 | Aux6 I/O − |
| 22 | GND | 23 | Aux7 I/O + | 24 | Aux7 I/O − |
| 25 | GND | 26 | Aux8 I/O + | 27 | Aux8 I/O − |
| 28 | GND | 29 | Aux9 I/O + | 30 | Aux9 I/O − |
| 31 | GND | 32 | **Aux10 I/O +** | 33 | **Aux10 I/O −** |
| 34 | GND | 35 | Ext Clock In + | 36 | Ext Clock In − |

### Key Signal: AUX_DIN[10] — External Discriminator Input (Deprecated in Production)

`AUX_DIN[10]` is the **MSbit** of the 11-bit Auxiliary I/O bus (`AUX_DIN[10:0]`), located on **pins 32/33**. ✅ verified 2026-04-08 — `Digitizer.vhd:L63` (`AUX_DIN : in std_logic_vector(10 downto 0)`)

**Historical use:** Originally designated as the front-panel external discriminator input — `reg_bit_slices = "010"` would slave a channel to `AUX_DIN(10)`, allowing an external RS485 pulse on pins 32/33 to trigger all channels simultaneously.

**⚠️ Disabled as of 2022-09-30:** The AUX_DIN(10) path was commented out in `Digitizer.vhd` (src mode `"010"`) because the **digitizer fanout board** (added July 2022) physically covers the Aux I/O pins, making front-panel access impossible. ✅ verified 2026-04-08 — `Digitizer.vhd:L994` (comment: "The digitizer fanout board covers the AUX I/O pins so use of the AUX I/O as an external discriminator is now valueless.")

Current `"010"` mode instead routes **BGO pattern/sum discriminator bits** from the front bus (not the Aux I/O pin).

- ⚠️ If Aux I/O is configured with bit 10 as an **output**, it cannot be used as an external discriminator input (moot in current hardware config)

### Dedicated External Clock Input (pins 35/36)

Separate from the 16 I/O signals. Can substitute for the onboard 100 MHz clock as the ADC sample clock source.

---

## 3. Front Bus Ribbon Cable — Inter-Digitizer (Intra-Crate)

Connects Master digitizer to Slave digitizer(s) within the same VME crate.

- Carries: `FB_LED` (discriminator propagation), front bus bits [17:0] including discriminator patterns
- In Slave mode: bit 17 carries the Master ch 0 discriminator bit → all Slave channels slave to Master ch 0
- No dedicated pinout documented here yet — see schematic `31Y334-Schematic-10ChanDigitizer-Rev4.2.pdf`

---

## 4. IEC 61076-4-101 Connector — Router Trigger (RTRG) Side

On the **Router Trigger** (not the digitizer itself), the TTCL + Data links use a hard-metric 2mm pitch connector.

**Pinout (25-row, 5-column, a–e):**

| Column | Signal |
|--------|--------|
| a | Data In + |
| b | Data In − |
| c | GND / Shield |
| d | TTCL Out + |
| e | TTCL Out − |

Each row = one digitizer connection. Signals are LVDS differential pairs.

> 🔌 **MTRG/RTRG connector pinouts** are in a separate file: `connectors.md`

---

## References

- `Digitizer-Specification-RevA-v2.0.pdf` — Section 2.2.7 (SER/DES), 2.2.8 (Auxiliary I/O)
- `ANL Digitizer Firmware for Experts.pdf` — Section on external discriminator source matrix
- `31Y334-Schematic-10ChanDigitizer-Rev4.2.pdf` — full schematic (pinout verification)
- `20160418 trig command link.pdf` — TTCL spec, connector details

---

*Created: 2026-04-05*

---


**Source:** Trigger user manual 20140901.pdf  
**Boards:** Master Trigger (MTRG) and Router Trigger (RTRG) — same physical board, different firmware.

---

## 1. SERDES Link Connector — 125-pin, 2mm × 2mm Hard Metric

Provides all 11 SERDES links (A–H, L, R, U). Mates with **Trigger Paddle Cards** (ANL-designed; two types: copper/Cat5 and fiber-optic).

Each link uses **2 rows (one "wafer")** of the connector. Per-link signal layout:

```
  125-pin Hard Metric connector — per-link layout (2 rows per link)
  Columns: a   b   c   d   e
           │   │   │   │   │
  Row N:  [a+][b+][GND][d+][e+]   a=SERDES_IN+  b=SERDES_OUT+  d=Discrim+  e=Throttle+
  Row N+1:[a−][b−][GND][d−][e−]   a=SERDES_IN−  b=SERDES_OUT−  d=Discrim−  e=Throttle−

  ⚠️ Links L and R: columns d and e absent (SERDES pairs only)
```

| Column | Signal | Direction (relative to trigger board) |
|--------|--------|---------------------------------------|
| a+/a− | SERDES IN (receives TTCL/data from downstream) | Input |
| b+/b− | SERDES OUT (sends TTCL/data to downstream) | Output |
| c | GND | — |
| d+/d− | LVDS IN — Discriminator bit (fast multiplicity, routed to CPLD) | Input |
| e+/e− | LVDS IN — Throttle request (from digitizer FIFO half-full flag) | Input |

> ⚠️ Links **L and R** do **not** have the extra LVDS lines (D/E columns) — SERDES pairs only. ✅ verified 2026-04-11 — `MTRG/top.vhd`: FAST_STROBE is a single signal from the external CPLD (not per-link); discriminator/throttle inputs only exist for links A–H via CPLD ribbon cables. Links L/R/U are SERDES-only inter-trigger connections. Trigger user manual §2.1 confirms the same figure (no d/e columns shown for L/R).

### Link Assignments

| Link | Row # | Typical Use |
|------|-------|-------------|
| A | 1–2 | Master→Router or Router→Digitizer |
| B | 3–4 | Master→Router or Router→Digitizer |
| C | 5–6 | Master→Router or Router→Digitizer |
| D | 7–8 | Master→Router or Router→Digitizer |
| E | 9–10 | Master→Router or Router→Digitizer |
| F | 11–12 | Master→Router or Router→Digitizer |
| G | 13–14 | Master→Router or Router→Digitizer |
| H | 15–16 | Master→Router or Router→Digitizer |
| L | 17–18 | Clock source input; GITMO (DGS) or Master-to-Master sync |
| R | 19–20 | Cross-system or diagnostic (MγRIAD, Master-to-Master) |
| U | 21–22 | Cross-system or diagnostic (MγRIAD, Master-to-Master) |

### Special Property of Link L

The SERDES recovered clock (RCLK) from link L feeds a clock MUX. Firmware can select it as the main FPGA logic clock instead of the onboard oscillator:
- **Routers:** slave their clock to the Master via link L → entire system runs synchronously
- **Masters:** can be synchronized to each other (e.g. DGS ↔ DFMA) via cross-Master link L connections

### Rev D Boards (post-2013)

Three pins on the SERDES connector optionally supply **+3.3V** to power fiber-optic Trigger Paddle Cards. Earlier boards require external power.

---

## 2. Master-to-Router Cabling

- **Cable:** 2 twisted pairs (or Cat5 patch cord), mates with Trigger Paddle Card
- **Master side:** any of links A–H
- **Router side:** link L (for clock recovery + command receipt)
- **Pair 1:** TTCL commands (Master SERDES OUT → Router SERDES IN)
- **Pair 2:** Data back (Router SERDES OUT → Master SERDES IN)
- Up to **8 Routers per Master** (links A–H); links L, R, U of Master reserved for expansion/cross-system

---

## 3. NIM I/O — Front Panel (Master Trigger only)

**Physical location:** Bottom of the MTRG front panel, just above the ejector handle. Standard LEMO coax connectors.

**Source:** Trigger user manual 20140901.pdf (Figure 1 board photo) + Master Trigger Registers Master Document.pdf (Section AUX_IO_CTL_REGISTER, AUX_INPUT_SELECT REGISTER)

### Physical Layout (viewed from front, looking into board)

```
+------------------+
|  NIM OUT 2  | NIM OUT 1  |   ← top row (outputs)
+------------------+
|  NIM IN  2  | NIM IN  1  |   ← bottom row (inputs)
+------------------+
```

> NIM numbering confirmed via `MTrigUser.template`: `NIMSrc1` uses `asynMask(...,0x00003000,12)` (bits 13:12) and `NIMSrc2` uses `asynMask(...,0x0000C000,14)` (bits 15:14). Similarly `EN_NIM1_DELAY`=bit 9, `EN_NIM2_DELAY`=bit 10 in `reg_MISC_CTL2`. NIM1 is consistently the lower register field → lower-numbered output. Left/right physical ordering (which LEMO jack is #1 vs #2 on the front panel) is not stated in available documentation — verify against PCB silkscreen. ✅ verified 2026-04-07 — MTrigUser.template (bit masks for NIMSrc1/2 and EN_NIM1/2_DELAY)

**Full front panel layout (top to bottom):**
1. Status LEDs
2. SERDES link connector (125-pin, 2mm hard metric)
3. CPLD ribbon cable connectors (SUM A/B/C/D, FAST STROBE)
4. Auxiliary I/O header
5. **NIM I/O → 4× LEMO connectors, 2×2 grid** (here)
6. Ejector handle

Two NIM inputs and two NIM outputs:

| Signal | Name | Direction | DGS Firmware Function |
|--------|------|-----------|------------------------|
| NIM In 1 | `NIM_IN1` | Input | Auxiliary trigger source (selectable via `EN_NIM_AUX`) |
| NIM In 2 | `NIM_IN2` | Input | **TAC-II TDC input — connect RF clock here** for timing measurements; also hard-wired as trigger veto source (`ENBL_NIM_VETO`) |
| NIM Out 1 | `NIMSrc1` | Output | Trigger output — source selectable via `NIMSrc1` + `NIM1_SubSelect` PVs |
| NIM Out 2 | `NIMSrc2` | Output | Trigger output — source selectable via `NIMSrc2` + `NIM2_SubSelect` PVs |

**NIM output source options:**

`NIMSrc1` (bits 13:12 of `reg_AUX_IO_CTL`): `SubSrc` (0), `AnyTrig` (1), `Sync` (2), `FastStrobe` (3) ✅ verified 2026-04-11 — `MTrigUser.template:L82-90`

`NIMSrc2` (bits 15:14 of `reg_AUX_IO_CTL`): `SubSrc` (0), `AnyTrig` (1), `ImpSync` (2), `RemoteSync` (3) ✅ verified 2026-04-11 — `MTrigUser.template:L91-99`

> ⚠️ **Previous doc listed incorrect values** (`AUX/NIM`, `SumX`, `SumY`, etc.) — those were from an older firmware version. Corrected 2026-04-11 against current `MTrigUser.template`.

**NIM output sub-select** (`NIM1_SubSelect`/`NIM2_SubSelect` PV):
- `TrigRam` (0), `VetoRam` (1), `SweepRam` (2), `EncXtra` (3), `FiltInProg` (4), `Diag` (5), `EncChng` (6), `SelSlowClk` (7)

**Key EPICS PVs (on `VME99:MTRG:`):**

| PV | Function |
|----|----------|
| `EN_NIM_AUX` | Enable NIM In 1 as auxiliary trigger input |
| `EN_NIM1_DELAY` | Enable delay on NIM In 1 |
| `EN_NIM2_DELAY` | Enable delay on NIM In 2 |
| `ENBL_NIM_VETO` | Enable NIM input as veto |
| `EN_NIM_VETO_A`–`H` | Enable NIM veto for Router links A–H |
| `NIMSrc1`, `NIMSrc2` | Select source for NIM Out 1/2 |
| `NIM1_SubSelect`, `NIM2_SubSelect` | Second-rank mux for NIM Out 1/2 |
| `xNIM_IN1_RBV` | Readback: current state of NIM In 1 |
| `DLYD_TDC_IN_NIM_IN2_RBV` | Readback: delayed TDC input from NIM In 2 |

**Rev D NIM upgrade:** Fast NIM receivers allow TDC use — measures time between NIM leading edge and next 50 MHz clock edge to **< 300 ps** (single shot, no averaging). Used in DGS MTRG firmware for the TAC-II TDC implementation.

**In exp2008_Chiara:** `EN_NIM1_DELAY` is set to `N` (disabled) in `basic_settings.sh`.

---

## 4. CPLD Ribbon Cable Connectors

- **Five 8-bit LVTTL ribbon cable connectors** on the front panel, to/from the on-board CPLD
- Used for fast "any channel hit" or multiplicity signals between trigger modules (MTRG ↔ RTRG fast strobe path)
- Separate from SERDES links — this is the **CPLD-to-CPLD fast trigger cable** path
- Enables the < 1 µs fast multiplicity strobe (FAST_STRB) used in DGS trigger algorithm

---

## 5. Auxiliary I/O Header (Trigger Module)

- **Top 11 rows:** RS485/TTL-compatible signals, selectable as inputs or outputs in **groups of 4**
- **Bottom 2 rows:** Differential ECL outputs
- General purpose — exact function defined by firmware and register settings

---

## 6. ECL Differential Outputs

Two differential ECL output signals provided alongside the Auxiliary I/O connector. Firmware-defined; typically used for fast timing outputs to legacy electronics.

---

## References

| Document | Path | Sections Used |
|----------|------|---------------|
| Trigger user manual | `/home/ryan/DGS_tools_pack/DGS_docs/DGS_System_Documentation/Modules/Trigger user manual 20140901.pdf` | §1.5 (General I/O), §2 (Module Photo, Connector Descriptions), §2.1 (SERDES connector), §2.1.1 (Link L), §2.2 (Master-to-Router cabling) |
| Master Trigger Registers | `/home/ryan/DGS_tools_pack/DGS_docs/DGS_System_Documentation/Firmware/Master_Trigger/GRETINA Master Trigger/Master Trigger Registers Master Document.pdf` | AUX_IO_CTL_REGISTER (0x0818), AUX_INPUT_SELECT (0x0820), Table 5 (NIM OUT encoding), MISC_STAT (0x0128) |
| DGS trigger firmware user guide | `/home/ryan/DGS_tools_pack/DGS_docs/DGS_System_Documentation/Modules/DGS trigger system firmware user guide.pdf` | §1.3 (NIM I/O), §1.3.1 (Rev D NIM upgrade) |
| CPLD sum logic | `/home/ryan/DGS_tools_pack/DGS_docs/DGS_System_Documentation/Firmware/Master_Trigger/CPLD_sum_logic.pdf` | CPLD fast strobe logic |
| MTrigUser.template (EPICS) | `/home/ryan/DGS_tools_pack/ANLDAQ/ioc/db/MTrigUser.template` | NIM PV definitions: EN_NIM_AUX, ENBL_NIM_VETO, EN_NIM1_DELAY, EN_NIM2_DELAY, NIMSrc1/2, NIM1/2_SubSelect |

**Related memory docs:**
- `deep_fpga_MTRG_MAIN.md` — MTRG main FPGA firmware (TAC-II TDC uses NIM_IN2)
- `deep_fpga_MTRG_CPLD.md` — CPLD fast multiplicity logic
- `connectors.md` — Digitizer-side connector pinouts (RJ45, Aux I/O)

---

*Created: 2026-04-05*

## Cross-References

- `knowledgeBase/deep_fpga_DIG.md` — DIG firmware: signal flow from RJ45 inputs through ADC pipeline
- `knowledgeBase/deep_fpga_MTRG_MAIN.md` — MTRG Main FPGA: 125-pin SERDES connector, NIM I/O, Aux I/O, TAC-II (NIM IN 2)
- `knowledgeBase/deep_fpga_RTRG.md` — RTRG firmware: SERDES links A–H to DIGs, Link L to MTRG
- `knowledgeBase/myriad.md` — MγRIAD NIM I/O (8 in/4 out) and ECL connectors; connected to MTRG Link U
- `knowledgeBase/reference_index.md` — Hardware drawings index: schematic PDFs for DIG, MTRG, RTRG, SBX
