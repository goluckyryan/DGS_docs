# Digitizer Connector Pinouts

**Source:** Digitizer-Specification-RevA-v2.0.pdf (GRETINA/DGS Digitizer board)

---

## 1. RJ45 — SER/DES (TTCL) Interface

The RJ45 connector on the digitizer is **not Ethernet-compatible**. It connects to the Router Trigger (RTRG) via a custom shielded RJ45-to-2mm hard Metric cable.

Carries: TTCL commands (receive), fast event data (transmit), and the 50 MHz system clock (recovered by SER/DES).

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

> ⚠️ Some signals have inverted logic (marked in original schematic with \*). Check schematic `31Y334-Schematic-10ChanDigitizer-Rev4.2.pdf` for exact polarity.

**SER/DES IC:** National Semiconductor DS92LV18TVV (LVDS, 1 Gbps)

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

### Key Signal: AUX_DIN[10] — External Discriminator Input

`AUX_DIN[10]` is the **MSbit** of the 11-bit Auxiliary I/O bus (`AUX_DIN[10:0]`), located on **pins 32/33**.

This is the designated **front-panel external discriminator input** used by the ANL digitizer firmware:

- When a channel's external discriminator source is set to "front panel" (option 3 in the external discriminator source matrix), it reads `AUX_DIN[10]`
- Used in `ExtTTL` trigger mode (`trigger_mux_select = ExtTTL`) — a TTL/RS485 pulse here triggers all channels simultaneously, latching DSP results (energy, CFD timestamps) into header memory
- ⚠️ If Aux I/O is configured with bit 10 as an **output**, it cannot be used as an external discriminator input

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

> 🔌 **MTRG/RTRG connector pinouts** are in a separate file: `MTRG_connectors.md`

---

## References

- `Digitizer-Specification-RevA-v2.0.pdf` — Section 2.2.7 (SER/DES), 2.2.8 (Auxiliary I/O)
- `ANL Digitizer Firmware for Experts.pdf` — Section on external discriminator source matrix
- `31Y334-Schematic-10ChanDigitizer-Rev4.2.pdf` — full schematic (pinout verification)
- `20160418 trig command link.pdf` — TTCL spec, connector details

---

*Created: 2026-04-05*
