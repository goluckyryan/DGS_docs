# MTRG GITMO Interface: GITMO_RCV_MACH + GITMO_TRIGGER

Stability: C3 - Structural / stable

Source files (MTRG MAIN_FPGA trunk/Source/):
- `GITMO_RCV_MACH.vhd` — 372 lines, JTA 2011-09-03
- `GITMO_TRIGGER.vhd` — 313 lines, JTA 2011-09-03 (minor update 2015-10)

**GITMO** = "Gammasphere In-Test-Mode Operation" — a VXI-based interface module that
bridges the legacy Gammasphere electronics backplane signals onto the MTRG's Link L SERDES
port. When GITMO mode is active, the MTRG can be driven by a legacy Gammasphere VXI trigger
system rather than the DGS digital trigger algorithm.

Last reviewed: 2026-04-24

---

## Table of Contents
1. [Background: Gammasphere Trigger Flow](#background)
2. [GITMO_RCV_MACH — Link L Receiver](#gitmo_rcv_mach)
   - [Ports](#rcv-ports)
   - [5-Word Frame Format](#frame-format)
   - [State Machine](#rcv-sm)
3. [GITMO_TRIGGER — Trigger Algorithm](#gitmo_trigger)
   - [Ports](#trig-ports)
   - [State Machine](#trig-sm)
   - [Trigger Type Code](#trig-type)
4. [Integration in Generated_top.vhd](#integration)

---

## Background: Gammasphere Trigger Flow {#background}

From JTA comments (preserved in both files, citing J. Weizeorick):

| Signal | VXI Backplane | GITMO word bit |
|--------|---------------|----------------|
| ZECLTRIG0 (Master Trigger) | ECL TRIG0 | bit 12 |
| RDY/BSY (ADC conversion busy) | open-collector ORed per crate | bit 11 |
| EOE (End Of Event — abort) | TTLTRIG2 | bit 10 |
| TOKEN_BUSY (readout started) | TTLTRIG1 | bit 9 |
| RUN | TTLTRIG0 | bit 8 |

**Event flow:**
1. Master Trigger Module fires → ECLTRIG0 asserted → DGS sees `TRIG0_FROM_VXI`
2. Hit Ge modules pull RDY/BSY low (ADC conversion in progress)
3. At late-trigger time: Master Trigger Module checks digital multiplicity
4. If passed: releases RDY/BSY → Master Readout Module starts Token → `TOKEN_BUSY` asserted
5a. If TOKEN_BUSY: event accepted — issue DGS trigger
5b. If EOE (abort/inhibit): skip event

Note: As of 2012-01-28, per Mike Carpenter, the "late trigger + NIM pattern match" states
were disabled. Only the early TRIG0 → delay countdown → ISSUE_TRIGGER path is active.

✅ verified 2026-04-24 — GITMO_TRIGGER.vhd (20180507 tag): NIM_TRIG and TOKEN_RCVD states commented out with datestamp "20120128: per request of Mike Carpenter" (L248-259)

---

## GITMO_RCV_MACH — Link L Receiver {#gitmo_rcv_mach}

Locks onto the data stream from the GITMO module over SERDES Link L.
Runs on `LINKL_RCLK` (SERDES recovered clock), not the board-wide CLK.

### Ports {#rcv-ports}

| Port | Dir | Width | Description |
|------|-----|-------|-------------|
| `CLK` | in | 1 | Board-wide clock |
| `RST` | in | 1 | Active-high power-on reset |
| `LINKL_RCLK` | in | 1 | Recovered clock directly from SERDES chip |
| `LINKL_RX` | in | 16 | Raw data from Link L (GITMO) |
| `RECEIVED_GITMO_DATA_OUT` | out | 16 | Re-latched (on LINKL_RCLK) copy of LINKL_RX |
| `LINKL_LOCK` | in | 1 | SERDES LOCK* signal (active LOW = locked) |
| `CLK_10_FLAG` | out | 1 | 10 MHz clock flag bit (from bit 15 of every word) |
| `MACHINE_LOCKED` | out | 1 | 1 = machine is locked and data is valid |
| `TRIG0_FROM_VXI` | out | 1 | ECL TRIG0 (Master Trigger) — bit 12 |
| `RDY_BSY_FROM_VXI` | out | 1 | RDY/BSY signal — bit 11 |
| `GS_EOE` | out | 1 | End-Of-Event (abort) — bit 10 |
| `GS_TOKEN_BUSY` | out | 1 | Token Busy (readout started) — bit 9 |
| `GS_RUN` | out | 1 | Run enable — bit 8 |
| `NIM_STAT` | out | 8 | NIM input states (from word 01, bits 7:0) |
| `ECL_STAT` | out | 8 | ECL input states (from word 02, bits 7:0) |
| `FERA_STAT` | out | 8 | FERA control states (from word 03, bits 7:0) |
| `FRAME_COUNT_ERROR` | out | 1 | Frame counter mismatch detected (MBO addition) |
| `EOF_ERROR` | out | 1 | End-of-frame 0xA5 marker missing |

### 5-Word Frame Format {#frame-format}

The GITMO continuously transmits groups of 5 words at SERDES rate.

**Common bits in every word (bits 15:8):**

| Bit | Signal |
|-----|--------|
| 15 | 10 MHz clock flag |
| 14:13 | Two-bit ordinal sync counter (must = `"00"` for frame lock) |
| 12 | ECLTRIG0 (Master Trigger / TRIG0_FROM_VXI) |
| 11 | RDY/BSY |
| 10 | TTLTRIG2 = EOE |
| 9 | TTLTRIG1 = TOKEN_BUSY |
| 8 | TTLTRIG0 = RUN |

✅ verified 2026-04-24 — GITMO_RCV_MACH.vhd (20180507 tag): LINKL_RX[12]=TRIG0, [11]=RDY_BSY, [10]=GS_EOE, [9]=GS_TOKEN_BUSY, [8]=GS_RUN (L243-248); bits[14:13]="00" required for MACHINE_LOCKED (L252-253)

**Per-word payload (bits 7:0):**

| Word | Bits 7:0 |
|------|----------|
| W01 | NIM input states (8 bits) |
| W02 | ECL input states (8 bits) |
| W03 | FERA control states (8 bits) |
| W04 | Frame counter (1–20, wrapping) |
| W05 | Fixed `0xA5` (end-of-frame marker) |

Bits 14:13 must be `"00"` in each word for lock to be maintained. Any mismatch → PRELOCK1.

✅ verified 2026-04-24 — GITMO_RCV_MACH.vhd (20180507 tag): W01=NIM_STAT[7:0] (L241), W02=ECL_STAT (L261), W03=FERA_STAT (L277), W04=frame counter (L303-330), W05=0xA5 end-of-frame (L335-347); frame counter wraps at 20 back to 1 (L309,322)

### State Machine {#rcv-sm}

States: `PRELOCK1`, `PRELOCK2`, `W01`, `W02`, `W03`, `W04`, `W05`

Runs on `LINKL_RCLK` edge.

✅ verified 2026-04-24 — GITMO_RCV_MACH.vhd (20180507 tag): state type declared L133; PRELOCK1 waits for LINKL_LOCK='0' (L207-211); PRELOCK2 waits for [14:13]="00" AND [7:0]=0xA5 (L226-231); PRELOCK2 returns to PRELOCK1 if LINKL_LOCK='1' (L224); W01-W05 all update common bits on every state (L239-349)

```
PRELOCK1: Wait for SERDES LOCK* = '0' (locked)
          → PRELOCK2

PRELOCK2: Wait for RECEIVED_GITMO_DATA[14:13]="00" AND [7:0]=0xA5
          (i.e., find the W05 end-of-frame marker to achieve word alignment)
          → W01 on match
          → PRELOCK1 if LINKL_LOCK goes high (SERDES lost)

W01:      Latch NIM_STAT, common status bits, CLK_10_FLAG
          Assert MACHINE_LOCKED if bits[14:13]="00"
          → W02 on success, → PRELOCK1 on mismatch

W02:      Latch ECL_STAT + common bits
          → W03 on success, → PRELOCK1 on mismatch

W03:      Latch FERA_STAT + common bits
          → W04 on success, → PRELOCK1 on mismatch

W04:      Latch common bits; validate frame counter (MBO addition):
          - First valid: capture counter, set NEXT_FRAME_VALID
          - Subsequent: compare against expected; FRAME_COUNT_ERROR if mismatch
          - Frame counter wraps at 20 (counts 1..20)
          → W05 on success, → PRELOCK1 on mismatch

W05:      Check for [14:13]="00" AND [7:0]=0xA5
          → W01 on success (continuous loop), → PRELOCK1 on mismatch
          EOF_ERROR asserted on mismatch
```

The machine outputs `TRIG0_FROM_VXI`, `RDY_BSY_FROM_VXI`, `GS_EOE`, `GS_TOKEN_BUSY`, `GS_RUN`
on **every** word state (W01–W05), continuously tracking the latest values.

---

## GITMO_TRIGGER — Trigger Algorithm {#gitmo_trigger}

Wraps the standard `trig_algo_support` base with a GITMO-specific 5-state front-end FSM.
When enabled and GITMO data is locked, waits for ECLTRIG0 from the GITMO, counts down an
optional delay, then issues a DGS trigger.

### Ports {#trig-ports}

Standard `trig_algo_support` interface ports (TRIGGER_ENABLE, TRIGGER_VETO, TRIGGER_PRESCALE,
TRIG_FIFO_RE/ACK/OUT, EVENT_AVAILABLE, ALGO_THROTTLE_REQUEST, trigger monitoring FIFO) plus:

| Port | Dir | Description |
|------|-----|-------------|
| `GITMO_DATA_LOCK` | in | Lock flag from GITMO_RCV_MACH |
| `TRIG0_FROM_VXI` | in | Early trigger from VXI (ECL TRIG0) |
| `RDY_BSY_FROM_VXI` | in | RDY/BSY busy signal |
| `GS_EOE` | in | End-of-event (abort signal) |
| `GS_TOKEN_BUSY` | in | Token busy (readout underway) |
| `GS_RUN` | in | Run enable bit |
| `NIM_STAT` | in | 8 NIM input states |
| `ECL_STAT` | in | 8 ECL input states |
| `FERA_STAT` | in | 8 FERA control states |
| `GITMO_NIM_MASK_REG` | in | NIM pattern trigger mask (VME register; unused since 2012) |
| `GITMO_TRIG_DELAY_REG` | in | Delay count in 20 ns increments before issuing trigger |

### State Machine {#trig-sm}

States: `IDLE`, `WAIT_TRIG0`, `BUSY`, `ISSUE_TRIGGER`, `WAIT_NO_TRIG0`

Reset condition: `TRIGGER_ENABLE='0'` OR `RST='1'` OR `GITMO_DATA_LOCK='0'` (asynchronous)

✅ verified 2026-04-24 — GITMO_TRIGGER.vhd (20180507 tag): state type declared L183; async reset on all three conditions L185; IDLE→WAIT_TRIG0 one-shot L191; WAIT_TRIG0 latches TS+delay on TRIG0='1' L204-207; BUSY: GS_EOE→WAIT_NO_TRIG0 L220, delay=0→ISSUE_TRIGGER L222, else count down L225; ISSUE_TRIGGER asserts TRIGGER_OCCURRED='1' L232; WAIT_NO_TRIG0 waits for TRIG0='0' L237-239

```
IDLE:           → WAIT_TRIG0 (one-cycle pass-through entry)

WAIT_TRIG0:     Poll TRIG0_FROM_VXI
                On TRIG0='1': latch TIME_STAMP_BUS, load GITMO_TRIG_DELAY_REG → BUSY
                Otherwise: clear timestamp, stay

BUSY:           Count down GITMO_TRIG_DELAY (20 ns per count)
                GS_EOE='1': abort → WAIT_NO_TRIG0
                GITMO_TRIG_DELAY=0: → ISSUE_TRIGGER

ISSUE_TRIGGER:  Assert TRIGGER_OCCURRED='1' for one cycle
                → WAIT_NO_TRIG0

WAIT_NO_TRIG0:  Wait for TRIG0_FROM_VXI to deassert (prevents double-triggering)
                TRIG0='1': stay
                TRIG0='0': → WAIT_TRIG0
```

Note: `GS_TOKEN_BUSY`, `RDY_BSY_FROM_VXI`, `NIM_STAT`, `ECL_STAT`, `FERA_STAT` and
`GITMO_NIM_MASK_REG` are all connected as ports but the active FSM does **not** use them.
The NIM_TRIG and TOKEN_RCVD states were removed 2012-01-28 per M. Carpenter's request.

### Trigger Type Code {#trig-type}

```
TRIGGER_TYPE_CODE => X"56"    -- 0x56 = GITMO trigger
```

This is passed to `trig_algo_support` and written into the TRIG FIFO.

✅ verified 2026-04-24 — GITMO_TRIGGER.vhd (20180507 tag): `TRIGGER_TYPE_CODE => X"56"` in trig_algo_support port map (L271)  
The `LATCHED_TIME_STAMP_BUS` is declared in the entity but **not connected** to
`trig_algo_support` — the support base uses its own `TIME_STAMP_BUS` input directly.

✅ verified 2026-04-24 — GITMO_TRIGGER.vhd (20180507 tag): LATCHED_TIME_STAMP_BUS signal declared L178, latched in WAIT_TRIG0 state L204, but port map connects TIME_STAMP_BUS directly (L272) — local latch unused in output path
The local latch in `GITMO_TRIGGER` captures the timestamp at TRIG0 for potential use
but it is not routed out (leftover from an earlier design revision).

---

## Integration in Generated_top.vhd {#integration}

From `MTRG_Generated_top.md` and `deep_fpga_MTRG_MAIN.md`:

- **Trigger slot 6A** = `TRIG_LOGIC6A : GITMO_TRIGGER` (top.vhd:L2778)
- **Trigger slot 6B** = `TRIG_LOGIC6B : REMOTE_MASTER_TRIG_SUPPORT`
- Mux selection: `LINK_L_IS_TRIGGER_TYPE <= TRIG_ALGO_MUX_SEL_REG(0)` — selects GITMO vs remote trigger for Link L
- VME register bit 15 of register 0x8000: `L_SM_LOCKED_RBV` — reflects GITMO_RCV_MACH lock in GITMO mode (top.vhd: updated 2011-10-05)

`GITMO_RCV_MACH` provides decoded status to:
- `GITMO_TRIGGER` (TRIG0, RDY_BSY, EOE, TOKEN_BUSY, RUN, NIM_STAT, ECL_STAT, FERA_STAT)
- Optionally to AUX_IO for NIM output monitoring

---

## See Also
- [`MTRG_Generated_top.md`](MTRG_Generated_top.md) — trigger slot assignments
- [`MTRG_trig_algo_support.md`](MTRG_trig_algo_support.md) — shared algorithm base used by GITMO_TRIGGER
- [`deep_fpga_MTRG_MAIN.md`](../deep_fpga_MTRG_MAIN.md) — MTRG top-level architecture, VME register table
- [`deep_fpga_MTRG_VME.md`](../deep_fpga_MTRG_VME.md) — VME register details including L_SM_LOCKED_RBV
