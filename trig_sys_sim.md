# Trigger System Simulation Testbench (`FPGA/others/Trig_sys_sim/`)

Stability: C3 - Structural / stable

_Source: `DGS_tools_pack/FPGA/others/Trig_sys_sim/` — 6 VHDL source files + ISim compiled artifacts._  
_Created: 2026-04-24. These are development/debug testbench files; not part of the production FPGA build tree._

---

## Purpose

A Xilinx ISim VHDL testbench that simulates two MTRG boards wired together via a fake SERDES link, exercised through a simulated VME register bus:
- LOCAL_MASTER: BUILD_TYPE=4 (DGS Master Trigger) ✅ verified 2026-04-24 — regio_tb.vhd:L66
- REMOTE_MASTER: BUILD_TYPE=5 (DSSD Master Trigger) ✅ verified 2026-04-24 — regio_tb.vhd:L251

Used during development to test:
- VME register read/write cycles (strobe/ack handshake)
- SERDES link initialization sequence (MISC_CTL1 dance)
- Manual trigger generation via `PULSED_CONTROL1`
- Inter-trigger propagation via `LINK_L_PROPAGATION_CONTROL_REG`
- DC balance enable/disable (MISC_CTL2 bits 15/14)

---

## File Inventory

| File | Lines | Role |
|------|-------|------|
| `regio_tb.vhd` | 795 | Top-level testbench: instantiates both MTRGs, clocks, VME bus mux, and stimulus process |
| `MstrTrig_pkg.vhd` | 490 | Package: `trigger_top` component declaration, VME register address constants, `Mtrig_record` I/O record type |
| `crate_def_tb.vhd` | 402 | Alternative testbench: uses `CRATE_REC` + `CRATE_DEF_TB` entity; simpler port-based approach |
| `MyRIAD_pkg.vhd` | 396 | Package: `MYRIAD` component declaration (IDT FIFOs A/B, SERDES, VME interface) — stubbed-out in testbench |
| `bus_pkg.vhd` | 127 | Package: `CRATE_REC` type, `clk_gen()`, `bus_write()`, `bus_read()` procedures |
| `bus_trans.vhd` | 31 | Thin BUS_TRANS entity (placeholder/stub) |
| `Work13_4/` | — | ISim compiled artifacts (`.prj`, `.exe`, `.wdb`, `.cmd`, logs) — not VHDL source |

---

## Simulated Hardware Configuration

```
LOCAL_MASTER  (slot 0, DGS MTRG, BUILD_TYPE=4)
    LOGIC_CLOCK = 50 MHz
    VME_CLOCK   = 50 MHz
    LINKR_TX  ──79ns──►  REMOTE_MASTER.LINKL_RX

REMOTE_MASTER (slot 1, DGS MTRG, BUILD_TYPE=4)
    LOGIC_CLOCK = LOCAL + 34ns offset (same frequency, phase-shifted)
    VME_CLOCK   = LOCAL + 34ns offset
```

- Two full `trigger_top` instantiations (entity from MTRG firmware, BUILD_TYPE=4)
- `SIMULATION_FLAG=1` passed to both — disables hardware-specific primitives
- SERDES link delay modeled as VHDL `transport` delay (79 ns LOCAL→REMOTE, 34 ns clock offset)
- MyRIAD board slot (#2) is commented out in `regio_tb.vhd`; present in `crate_def_tb.vhd` framework

---

## VME Bus Simulation

`MstrTrig_pkg.vhd` defines procedures for the stimulus process:

| Procedure | Signature | Description |
|-----------|-----------|-------------|
| `clk_gen` | `(clk, FREQ)` | Infinite clock loop at given frequency | ✅ verified 2026-04-24 — bus_pkg.vhd:L23 |
| `bus_write` | `(addr, data, dly, timeout, address, databus, strobe, ack)` | Assert addr, assert data after `dly` ns, raise strobe, wait for ack (or timeout), drop strobe | ✅ verified 2026-04-24 — bus_pkg.vhd:L24 |
| `bus_read` | `(addr, data, dly, timeout, address, databus, strobe, ack)` | Assert addr, raise strobe, latch `databus` on ack rising edge, return as integer | ✅ verified 2026-04-24 — bus_pkg.vhd:L25 |

Bus mux (`BUS_MUX` process, 1 GHz oversample clock):
- `SLOT` variable selects which board gets the strobe
- All boards share the address bus; only the selected board sees `VME_STB`
- Write: `LOC_DATA` driven from stimulus; Read: `LOC_DATA` captured on `LACK_OUT` rising edge

---

## VME Register Address Constants (from `MstrTrig_pkg.vhd`)

These match the production MTRG register map (see also `MTRG_registers.md`):

| Name | Address | Notes |
|------|---------|-------|
| `LINK_LOCKED` | 0x0100 | |
| `LINK_DEN` | 0x0104 | |
| `LINK_REN` | 0x0108 | |
| `LINK_SYNC` | 0x010C | |
| `GITMO_LOCK_COUNT` | 0x0110 | |
| `TIMESTAMP_A/B/C` | 0x0114–0x011C | |
| `MSM_STATE` | 0x0120 | |
| `RC_STATE` | 0x0124 | |
| `MISC_STAT` | 0x0128 | |
| `Diagnostic_A–H` | 0x012C–0x0148 | |
| `DIAG_STAT` | 0x014C | |
| `MISC_STAT2` / `THROTTLE_STAT` | 0x0150 | Dual-name (version dependent) |
| `CODE_DATE` | 0x0158 | |
| `CODE_REVISION` | 0x015C | |
| `MON1_FIFO – MON8_FIFO` | 0x0160–0x017C | |
| `CHAN1_FIFO – CHAN8_FIFO` | 0x0180–0x019C | |
| `MON_FIFO_STAT` | 0x01A0 | |
| `CHAN_FIFO_STAT` | 0x01A4 | |
| `GITMO_STAT1/2` | 0x01D4–0x01D8 | |
| `GITMO_ERROR_COUNT` | 0x01DC | |
| `INPUT_LINK_MASK` | 0x0800 | |
| `LED_REGISTER` | 0x0804 | |
| `SKEW_CTL_A/B/C` | 0x0808–0x0810 | |
| `MISC_CLK_CTL` | 0x0814 | |
| `AUX_IO_CTL` | 0x0818 | |
| `AUX_IO_DATA` | 0x081C | |
| `AUX_INPUT_SELECT` | 0x0820 | |
| `AUX_TRIGGER_WIDTH` | 0x0824 | |
| `SERDES_TPOWER` | 0x0828 | |
| `SERDES_RPOWER` | 0x082C | |
| `SERDES_LOCAL_LE` | 0x0830 | |
| `SERDES_LINE_LE` | 0x0834 | |
| `LVDS_PREEMPHASIS` | 0x0838 | |
| `LINK_LRU_CTL` | 0x083C | |
| `MISC_CTL1` | 0x0840 | Bit 0 = LOCK_RETRY (link-init retry), bit 1 = LOCK_ACK (lock acknowledge), bit 2 = RESET_LINK_INIT (active high), bit 6 = IMP_SYNC (imperative sync) ✅ verified 2026-04-24 — Generated_top.vhd:L1852-1858 |
| `MISC_CTL2` | 0x0844 | Per Generated_top.vhd:L1875-1877: bits 11/12/13=EN_LINKL/R/U_TX_DCBAL, bit 15=EN_RTR_DCBAL. Testbench stimulus comments describe bit 15 as 'TX DC balance' and bit 14 as 'RX DC balance' — bit 14 is **not** assigned in production firmware; this is a testbench comment discrepancy. ✅ verified 2026-04-24 — Generated_top.vhd:L1875-1878 + regio_tb.vhd:L734-748 |
| `GENERIC_TEST_FIFO` | 0x0848 | |
| `DIAG_PIN_CTL_REG` | 0x084C | |
| `TRIG_MASK` | 0x0850 | Bit 0 = manual trigger, bit 5 = Link-L remote (bit 6 = Link-R remote) |
| `TRIG_DIST_MASK` | 0x0854 | |
| `SYNC_RST_RT_FIFO_MASK` / `LOCAL_FRAME_12_DATA_0` | 0x0858 | Dual-name |
| `SYNC_RST_RT_DIAG_MASK` / `LOCAL_FRAME_12_DATA_1` | 0x085C | Dual-name |
| `SERDES_MULT_THRESH` / `LOCAL_FRAME_12_DATA_2` | 0x0860 | GRETINA / DGS |
| `TW_ETHRESH_CTL` / `LOCAL_FRAME_12_DATA_3` | 0x0864 | GRETINA / DGS |
| `TW_ETHRESH_LOW` / `LOCAL_FRAME_12_DATA_4` | 0x0868 | GRETINA / DGS |
| `TW_ETHRESH_HIGH` / `LOCAL_FRAME_14_DATA_0` | 0x086C | GRETINA / DGS |
| `LINK_L_PROPAGATION_CONTROL_REG` | 0x08D0 | |
| `LINK_R_PROPAGATION_CONTROL_REG` | 0x08D4 | |
| `LINK_U_PROPAGATION_CONTROL_REG` | 0x08D8 | |
| `PULSED_CONTROL1` | 0x08E0 | Bit 15 = whack manual trigger |
| `PULSED_CONTROL2` | 0x08E4 | |
| `FIFO_RESETS` | 0x08F0 | |
| `ASYNC_CMD_FIFO` | 0x08F4 | |
| `AUX_CMD_FIFO` | 0x08F8 | |
| `DEBUG_CMD_FIFO` | 0x08FC | |
| `CPLD_MULT_SUM` | 0x6000 | |
| `RAW_DISC_BITS` | 0x8000 | |
| `DISC_BIT_MASK` | 0xA000 | |
| `FS_THRESH` | 0xE000 | |
| `FS_PARTIAL_A–D` | 0xA010–0xA01C | |
| `FS_TOTAL_MULTIPLICITY` | 0xA004 | |

---

## Stimulus Sequence (regio_tb.vhd BUS_ACTIVITY process)

The testbench exercises a specific init + manual trigger sequence:

1. **Slot 0 (LOCAL_MASTER) init:**
   - Write `INPUT_LINK_MASK = 0x0000` (unmask all links)
   - Read back + write to `0x0848` (verify R/W cycle)
   - Assert `LINK_LOCK = 0x000` (all links "locked" to firmware)
   - Write `MISC_CTL1 = 0x0004` (bit 2 = RESET_LINK_INIT set; link-init in reset) ✅ verified 2026-04-24 — regio_tb.vhd:L593 comment + Generated_top.vhd:L1854
   - Write `MISC_CTL1 = 0x0000` (release link-init reset) ✅ verified 2026-04-24 — regio_tb.vhd:L598
   - Write `MISC_CTL1 = 0x0002` (bit 1 = LOCK_ACK assert) ✅ verified 2026-04-24 — regio_tb.vhd:L603, Generated_top.vhd:L1853
   - Write `MISC_CTL1 = 0x0000` (release LOCK_ACK) ✅ verified 2026-04-24 — regio_tb.vhd:L608
   - Write `LINK_LRU_CTL = 0x8000` (enable L/R/U data output)

2. **Slot 1 (REMOTE_MASTER) init:** Same sequence as slot 0.

3. **REMOTE_MASTER propagation disabled:**
   - Write `LINK_L_PROPAGATION_CONTROL_REG = 0x0000`

4. **Slot 0 manual triggers:**
   - Write `TRIG_MASK = 0x0001` (enable manual trigger)
   - Write `PULSED_CONTROL1 = 0x8000` × 2 (fire manual trigger twice)
   - Read `Diagnostic_A` (expect readback = 2)

5. **Slot 1 propagation re-enabled + trigger test:**
   - Write `LINK_L_PROPAGATION_CONTROL_REG = 0x0002` (propagate Frame 3 triggers)
   - Write `TRIG_MASK = 0x0020` (enable Link-L remote trigger, bit 5)

6. **More Slot 0 triggers** and Slot 1 local+remote trigger enable (`TRIG_MASK = 0x0041`):

7. **DC balance test:**
   - Write `MISC_CTL2 = 0x8000` on Slot 0 (enable TX DC balance)
   - Write `MISC_CTL2 = 0x4000` on Slot 1 (enable RX DC balance)
   - Verify link recovers

8. **Rapid-fire triggers** (Slot 0): 4 × `PULSED_CONTROL1` at 800/600/400 ns intervals

---

## BUILD_TYPE Firmware Code Map (from `MstrTrig_pkg.vhd`)

| Code | Firmware Type |
|------|--------------|
| 0 | Proto |
| 1 | GRETINA Router |
| 2 | GRETINA Master Trigger |
| 3 | GRETINA Data Generator |
| 4 | **DGS Master Trigger** ← testbench uses this |
| 5 | DSSD Master Trigger |
| 6 | DGS Router |
| 7 | DSSD Router |
| 8 | DGS Data Generator |
| 9 | DSSD Data Generator |
| A | Digitizer Tester |
| B | MyRIAD Trigger Expansion Module |
| C | DGS Digitizer |
| D | DSSD Digitizer |
| E | (unused) |
| F | VME FPGA |

---

## Notes

- `crate_def_tb.vhd` is an alternate structural testbench (entity `CRATE_DEF_TB`) — port-based, with `oversample_clk` and `LOCAL_MASTER`/`REMOTE_MASTER` as top-level ports. Likely intended for integration with an external stimulus generator.
- `MyRIAD_pkg.vhd` defines the full `MYRIAD` component interface (dual IDT FIFOs A/B, SERDES link, VME 16-bit bus) but the MYRIAD slot (slot 2) is commented out in both testbenches.
- `bus_trans.vhd` is a stub entity with no architecture body — placeholder for a bus transactor that was never implemented.
- The ISim `.exe` files in `Work13_4/` correspond to compiled versions of `CRATE_DEF_TB` and `REGIO_TB`.

---

## See Also

- `knowledgeBase/vhdl/MTRG_registers.md` — production MTRG register map (same addresses used in regio_tb.vhd stimulus)
- `knowledgeBase/vhdl/MTRG_Generated_top.md` — `trigger_top` entity (what's being simulated)
- `knowledgeBase/fpga.md` — FPGA firmware overview and build type table
- `knowledgeBase/myriad.md` — MyRIAD module documentation
- `knowledgeBase/deep_fpga_building.md` — FPGA build toolchain (ISE/Vivado); Firmware_Tags archive; original brief simulation note (stub points here)
- `knowledgeBase/deep_fpga_MTRG_MAIN.md` — production ISE MTRG Main FPGA internals; same `trigger_top` component
