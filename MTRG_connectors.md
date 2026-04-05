# MTRG/RTRG Connector Pinouts

**Source:** Trigger user manual 20140901.pdf  
**Boards:** Master Trigger (MTRG) and Router Trigger (RTRG) — same physical board, different firmware.

---

## 1. SERDES Link Connector — 125-pin, 2mm × 2mm Hard Metric

Provides all 11 SERDES links (A–H, L, R, U). Mates with **Trigger Paddle Cards** (ANL-designed; two types: copper/Cat5 and fiber-optic).

Each link uses **2 rows (one "wafer")** of the connector. Per-link signal layout:

| Column | Signal | Direction (relative to trigger board) |
|--------|--------|---------------------------------------|
| A+ / A− | SERDES IN (receives TTCL/data from downstream) | Input |
| B+ / B− | SERDES OUT (sends TTCL/data to downstream) | Output |
| C | GND | — |
| D+ / D− | LVDS IN — Discriminator bit (fast multiplicity, routed to CPLD) | Input |
| E+ / E− | LVDS IN — Throttle request (from digitizer FIFO half-full flag) | Input |

> ⚠️ Links **L and R** do **not** have the extra LVDS lines (D/E columns) — SERDES pairs only.

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

> ⚠️ Left/right ordering within each row: inferred from register bit ordering (bits 15:14 = #2, bits 13:12 = #1). **Verify against PCB silkscreen** for ground truth — not explicitly stated in text in the manuals.

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

**NIM output source options** (`NIMSrc1`/`NIMSrc2` PV):
- `AUX/NIM` (0), `SumX` (1), `SumY` (2), `SumXY` (3), `CPLD FS` (4), `RemMstr(L)` (5), `RemMstr(R)` (6), `MyRIAD(U)` (7)

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
- `raw_fpga_MTRG_MAIN.md` — MTRG main FPGA firmware (TAC-II TDC uses NIM_IN2)
- `raw_fpga_MTRG_CPLD.md` — CPLD fast multiplicity logic
- `digitizer_connectors.md` — Digitizer-side connector pinouts (RJ45, Aux I/O)

---

*Created: 2026-04-05*
