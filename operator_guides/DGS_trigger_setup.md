# DGS Trigger Setup — Operator Walkthrough

Stability: C1 - Active / Living Document

**Audience:** Operators and physicists setting up the DGS/Gammasphere DAQ for an experiment.
**System:** Full DGS (Gammasphere) — **not** DuoGe. Uses fixed production FPGA firmware.
**Goal:** Get from "signals connected" to "GeC data with BGO Compton suppression flowing to disk."
**Style:** Step-by-step, data-flow order. Each step shows what to set and why.

> For hardware internals → [`../engineer_guides/README.md`](../engineer_guides/README.md)
> For system overview → [`../overview_DGS.md`](../overview_DGS.md)
> For trigger firmware details → [`../260E_trigger_scheme.md`](../260E_trigger_scheme.md), [`../260E_MTRG_scheme.md`](../260E_MTRG_scheme.md)

---

## Production Firmware Versions (DGS System)

| Board | VME ID | Code Date | Code Revision | Notes |
|-------|--------|-----------|---------------|-------|
| MTRG  | 0xf13  | 0x1022    | **0x04A8**    | TAC2 + Trigger Hold-Off |
| RTRG  | 0xf13  | 0x0414    | **0x260E**    | Current production router |
| MDIG  | 0xf13  | 20250704  | **0x4CD8**    | Master digitizer |
| SDIG  | 0xf13  | 20250704  | **0x4CD8**    | Slave digitizer |

Verify after IOC boot:
```bash
caget VME10:MTRG:CODE_REVISION_RBV   # expect 0x04A8
caget VME03:RTR1:CODE_REVISION_RBV   # expect 0x260E
caget VME01:MDIG1:CODE_REVISION_RBV  # expect 0x4CD8
```

---

## Signal Flow Overview

```
Ge detector + BGO shield (7 segments)
       │
       ▼
   Slope Box (SBX)              ← gain, offset, time constant (set via Collector Box Pi)
       │  4 signals per detector:
       │  GeC (Ge Center), BGO Sum, GeSide, BGO Pattern
       ▼
  Collector Box                 ← routes signals, BGO/Ge HV control
       │
       ▼
  MDIG1 ch 5  ← GeC signal     ← ADC + discriminator (MDIG = BUS_LEFT board)
  MDIG1 ch 0  ← BGO Sum signal ← ADC + discriminator
       │ discriminator bits sent via SERDES to RTRG
       ▼
     RTRG (firmware 0x260E)     ← Ge/BGO pairing, clean/dirty classification, multiplicity
       │ multiplicity counts sent via SERDES to MTRG
       ▼
     MTRG (firmware 0x04A8)     ← trigger decision (threshold on clean Ge multiplicity)
       │ accept/reject back to RTRG → DIG
       ▼
  DIG FIFO → VME backplane → tcpReceiver (port 9001) → host PC → disk
```

**Channel assignment (standard DGS):**
- **MDIG1** (BUS_LEFT/Center+Sum board): ch 0–4 = BGO Sum, ch 5–9 = GeC
- **SDIG1** (BUS_RIGHT/Side+Pattern board): ch 0–4 = BGO Pattern, ch 5–9 = GeSide

---

## Prerequisites

- Hardware connected: Ge detector → SBX → Collector Box → MDIG1 (GeC→ch5, BGO→ch0)
- IOC booted on your VME crate, EPICS CA accessible
- Source environment: `source /global/EPICS_env.sh` (sets CA ports=5064/5065, addr list)
- Trigger links initialized: run `trig_setup_Stage1.sh` through `trig_setup_Stage5.sh`
  → see [`../trig_setup_scripts.md`](../trig_setup_scripts.md) for the full link init sequence
- Replace `VMExx` with your crate number (e.g. `VME01`) and `RTRx` with your router (e.g. `RTR1`)

---

## Step 1 — Digitizer: Set Discriminator Thresholds

The DIG generates discriminator bits that tell the RTRG "this channel had a hit."

### GeC channel (MDIG1 ch 5)

```bash
# LED threshold — minimum pulse height to register a GeC hit (ADC counts above baseline)
caput VMExx:MDIG1:led_threshold5   200     # typical starting value; tune per experiment

# Discriminator mode: LED (leading edge) is simplest; CFD for better timing resolution
caput VMExx:MDIG1:disc_mode5       0       # 0=LED, 1=CFD

# Enable the channel
caput VMExx:MDIG1:channel_enable5  1
```

### BGO Sum channel (MDIG1 ch 0)

```bash
# BGO threshold — lower is OK, BGO just needs to fire reliably above noise
caput VMExx:MDIG1:led_threshold0   100     # typical starting value
caput VMExx:MDIG1:disc_mode0       0       # LED mode
caput VMExx:MDIG1:channel_enable0  1
```

> **Verify readback:** `caget VMExx:MDIG1:led_threshold5_RBV` should match what you set.

---

## Step 2 — Digitizer: DIG-Level BGO Veto (optional — RTRG veto is primary)

The RTRG handles Compton suppression automatically (Step 3). This step adds a secondary
DIG-level veto that **erases** the GeC event from the readout FIFO if BGO fires.

For GeC ch 5: route BGO Sum discriminator as the external discriminator source.

```bash
# Route BGO Sum as external discriminator source for ch 5
# ext_disc_src5 options: Ch9=0, GeSide=1, BGOp=2, Timestamp=3, VME=4, BGOsum=5, ExtFromTrig=6
caput VMExx:MDIG1:ext_disc_src5    5       # 5 = BGOsum

# Set mode: how ext disc combines with internal disc
# Options: Disc ONLY=0, Disc OR Ext=1, Disc AND Ext=2, Ext ONLY=3
# For veto: use "Disc AND Ext" (require both) or handle at RTRG level only
caput VMExx:MDIG1:ext_disc_sel5    0       # 0=Disc ONLY (disable DIG-level ext disc if using RTRG veto)

# Enable veto on ch 5
caput VMExx:MDIG1:veto_enable5     1

# Set DIG holdoff (suppress re-triggering on same channel after a hit)
caput VMExx:MDIG1:holdoff_time5    <ns>    # typical: 2000–5000 ns
```

> **Note:** For standard DGS with RTRG veto active (Step 3), `ext_disc_sel5=0` (Disc ONLY) is fine —
> the RTRG handles clean/dirty classification. Only enable DIG-level veto if you want hard suppression at the FIFO level.

---

## Step 3 — RTRG: Ge/BGO Pairing and Compton Suppression

The RTRG firmware (0x260E) automatically pairs **Ge bits [9:5] ↔ BGO bits [4:0]** from each DIG.
For your detector on MDIG1: **BGO ch-0 (bit 0) ↔ GeC ch-5 (bit 5)** = pair index 0.

The `disc_mach` FSM classifies each event:
- **CLEAN** = Ge fired, BGO silent within overlap window → full-energy deposit
- **DIRTY** = both fired within window → Compton scatter
- **BGO_ONLY** = BGO fired, Ge did not → shield-only event

### Set the coincidence overlap window

```bash
# TSCATTER_DELAY_REG:
#   bits [6:0]  = overlap window (10 ns ticks; how long after Ge to wait for BGO)
#   bits [14:8] = assertion window (10 ns ticks; how long CLEAN/DIRTY flags stay asserted)
# Example: overlap = 10 ticks = 100 ns, assertion = 20 ticks = 200 ns → 0x140A
caput VMExx:RTRx:TSCATTER_DELAY_REG   0x140A
```

### Enable clean/dirty classification with delay alignment

```bash
# CLEAN_DIRTY register:
#   bit 15 = 1 → use DPRAM per-bit delay alignment (recommended — corrects Ge/BGO skew)
#   bits [3:0]: 0001=count CLEAN hits for trigger plane, 0010=count DIRTY
caput VMExx:RTRx:CLEAN_DIRTY          0x8001    # delay ON + clean-hit counting
```

### Map GeC ch-5 to the trigger multiplicity plane

```bash
# Y_PLANE_MAP: bitmask of which Ge channels contribute to trigger multiplicity count
# Bit 5 = ch 5 = 0x0020. For 2 detectors on ch5+ch6: 0x0060, etc.
caput VMExx:RTRx:Y_PLANE_MAP          0x0020    # 1 detector on ch5
```

### Enable RTRG-level veto (suppresses DIRTY events from triggering)

```bash
caput VMExx:RTRx:ENABLE_VETO          1
```

> **Result:** Only CLEAN Ge hits (no BGO coincidence within overlap window) count toward
> the Y-plane multiplicity sent to MTRG. DIRTY events are flagged but not counted → Compton suppressed.

---

## Step 4 — MTRG: Trigger Decision

The MTRG (firmware 0x04A8) receives clean-Ge multiplicity from all RTRGs and fires a global
trigger when the threshold is met.

### Set trigger multiplicity threshold

```bash
# Threshold: minimum number of CLEAN Ge hits to fire a trigger
# For 1-detector setup: threshold = 1
caput VME10:MTRG:Threshold            1         # VME10 always has the MTRG in full DGS

# Enable trigger algorithm 5 (standard DGS multiplicity trigger)
caput VME10:MTRG:ALGO_5_SELECT        1         # 1 = enable algo 5

# Enable trigger holdoff (prevent retriggering on same event)
caput VME10:MTRG:EN_TRIG_HOLDOFF_A   1         # per-algorithm holdoff enable
```

> **Verify:** `caget VME10:MTRG:Threshold_RBV` and watch `VME10:MTRG:TRIGGER_RATE` for counts.

---

## Step 5 — Run Control: Start DAQ

```bash
# Enable data save to disk
caput Online_CS_SaveData              1         # 1 = save

# Start run (big red button)
caput Online_CS_StartStop             1         # 1 = Start
```

> Monitor:
> - `DAQC<NN>_OL_DataRate0` — data rate on your crate (should be non-zero)
> - `DAQC<NN>_OL_TotalBufsLost` — should stay at 0
> - `VME10:MTRG:TRIGGER_RATE` — should reflect beam-on rate

---

## Step 6 — Verify Compton Suppression is Working

In gebsort or parquet analysis, Compton-suppressed spectra should show:
- **Reduced Compton continuum** below full-energy peaks
- Full-energy peaks intact
- Single/double escape peaks unchanged

**If Compton suppression is NOT working:**
1. Check BGO threshold (Step 1) — may be too high; BGO not firing
2. Check overlap window (Step 3) — may be too short; BGO arriving late
3. Verify `ENABLE_VETO=1` on RTRG (Step 3)
4. Verify `CLEAN_DIRTY` bit 15 = 1 (delay alignment on)
5. Check BGO signal is actually reaching ch 0 (scope check at DIG input)
6. Check `Y_PLANE_MAP` has the right bits set for your Ge channels

---

## Quick Reference — Key PVs

| PV | Description | Typical Value |
|----|-------------|---------------|
| `VMExx:MDIG1:led_threshold5` | GeC discriminator threshold | 200 |
| `VMExx:MDIG1:led_threshold0` | BGO Sum discriminator threshold | 100 |
| `VMExx:MDIG1:channel_enable5` | Enable GeC channel | 1 |
| `VMExx:MDIG1:channel_enable0` | Enable BGO channel | 1 |
| `VMExx:MDIG1:ext_disc_src5` | External disc source for ch5 | 5 (BGOsum) |
| `VMExx:MDIG1:veto_enable5` | DIG-level veto on GeC | 1 |
| `VMExx:RTRx:TSCATTER_DELAY_REG` | Ge/BGO overlap + assertion window | 0x140A |
| `VMExx:RTRx:CLEAN_DIRTY` | Clean/dirty mode + delay alignment | 0x8001 |
| `VMExx:RTRx:Y_PLANE_MAP` | Which Ge channels count for trigger | 0x0020 (ch5 only) |
| `VMExx:RTRx:ENABLE_VETO` | RTRG Compton veto enable | 1 |
| `VME10:MTRG:Threshold` | Minimum clean-Ge multiplicity to trigger | 1 |
| `VME10:MTRG:ALGO_5_SELECT` | Enable trigger algorithm 5 | 1 |
| `VME10:MTRG:EN_TRIG_HOLDOFF_A` | Enable trigger holdoff | 1 |
| `Online_CS_StartStop` | Run start/stop | 1=Start |
| `Online_CS_SaveData` | Data save enable | 1=Save |

---

*Created: 2026-05-02 | Firmware: MTRG=0x04A8, RTRG=0x260E, DIG=0x4CD8 | Status: Draft — needs Ryan review*
