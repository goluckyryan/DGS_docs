# MTRG sum_hits_XY.vhd — XY Coincidence Trigger Algorithm
Stability: C3 - Structural / stable

Source: `FPGA/MTRG/Firmware/MAIN_FPGA/trunk/Source/sum_hits_XY.vhd`  
Lines: 158 ✅ verified 2026-04-24 — file confirmed 158 lines

---

## Table of Contents

- [Purpose](#purpose)
- [Entity](#entity)
- [Architecture](#architecture)
  - [Trigger Algorithm FSM](#trigger-algorithm-fsm)
  - [Generic Support Layer](#generic-support-layer)
- [Key Differences vs sum_hits_x.vhd](#key-differences-vs-sum_hits_xvhd)
- [See Also](#see-also)

---

## Purpose

`sum_hits_xy` implements the **X+Y coincidence trigger algorithm** for the Master Trigger (MTRG). It fires a trigger only when **both** the global X-plane hit sum and the global Y-plane hit sum simultaneously exceed their respective thresholds.

This is a stricter variant of the individual-axis triggers (`sum_hits_x.vhd`, `sum_hits_y.vhd`) and is used when spatial coincidence in both detector planes is required.

---

## Entity

```vhdl
entity sum_hits_xy is
Port (
    CLK : in std_logic;     -- board-wide 50 MHz
    RST : in std_logic;     -- global reset
    -- Algorithm-specific inputs
    GLOBAL_X_TOTAL    : in std_logic_vector(15 downto 0);  -- sum of X-plane hits from all Routers
    GLOBAL_Y_TOTAL    : in std_logic_vector(15 downto 0);  -- sum of Y-plane hits from all Routers
    SUM_OF_X_THRESH   : in std_logic_vector(15 downto 0);  -- VME-configurable X threshold
    SUM_OF_Y_THRESH   : in std_logic_vector(15 downto 0);  -- VME-configurable Y threshold
    TRIG_TYPE         : in std_logic_vector(7 downto 0);   -- trigger type code to issue
    -- Generic algorithm I/O (passed to trig_algo_support)
    TIME_STAMP_BUS    : in std_logic_vector(47 downto 0);
    TRIG_DIST_MASK    : in std_logic_vector(7 downto 0);
    TRIGGER_ENABLE    : in std_logic;
    TRIGGER_VETO      : in std_logic;
    TRIGGER_HOLDOFF   : in std_logic_vector(15 downto 0);  -- added 20251022
    TRIG_HOLDOFF_ENBL : in std_logic;                      -- added 20251022
    TRIGGER_PRESCALE  : in std_logic_vector(15 downto 0);
    TRIG_FIFO_RE      : in std_logic;
    TRIG_FIFO_ACK     : in std_logic;
    -- Trigger monitor FIFO I/O (standard interface)
    TRIG_MON_FIFO_RD_CLK      : in  std_logic;
    TRIG_MON_FIFO_RE          : in  std_logic;
    LOCAL_TRIG_MON_DET_DATA   : in  std_logic_vector(15 downto 0);
    LOCAL_TRIG_MON_XTRA_DATA  : in  std_logic_vector(15 downto 0);
    TRIG_MON_ACK              : in  std_logic;
    TRIG_MON_FIFO_DATA_OUT    : out std_logic_vector(15 downto 0);
    TRIG_MON_FIFO_FULL        : out std_logic;
    TRIG_MON_FIFO_EMPTY       : out std_logic;
    -- Output signals (generic)
    TRIG_FIFO_OUT             : out std_logic_vector(15 downto 0);
    DIAG_TRIG_FIFO_EMPTY      : out std_logic;
    DIAG_TRIG_FIFO_FULL       : out std_logic;
    DIAG_TRIG_FIFO_WE         : out std_logic;
    DIAG_EVENT_COUNT          : out std_logic_vector(7 downto 0);
    DIAG_PRESCALE_ENABLE      : out std_logic;
    DIAG_HOLDOFF_IN_PROGRESS  : out std_logic;  -- added 20251022
    RAW_NONVETOED_TRIG_ACK    : out std_logic;
    ENABLED_TRIG_ACK          : out std_logic;
    ENABLED_NONVETOED_TRIG_ACK: out std_logic;
    EVENT_AVAILABLE           : out std_logic;
    MON_EVENT_AVAILABLE       : out std_logic;
    ALGO_THROTTLE_REQUEST     : out std_logic
);
end sum_hits_xy;
```

---

## Architecture

### Trigger Algorithm FSM

2-state FSM (`SUM_STATES`) ✅ verified 2026-04-24 — sum_hits_XY.vhd:L70-71 (`type SUM_STATES is (WAIT_TRIG, WAIT_FALL)`):

| State | Transition Condition | Action |
|-------|---------------------|--------|
| `WAIT_TRIG` | `GLOBAL_X_TOTAL > SUM_OF_X_THRESH AND GLOBAL_Y_TOTAL > SUM_OF_Y_THRESH` | `TRIGGER_OCCURRED = '1'`, capture `TRIG_TYPE`, go to `WAIT_FALL` ✅ verified 2026-04-24 — L85-90 |
| `WAIT_FALL` | Both totals drop to threshold or below (≤ thresh re-arms, since `>` test fails at equality) | Return to `WAIT_TRIG`; `TRIGGER_OCCURRED = '0'` always in this state ✅ verified 2026-04-24 — L90-96 |

Key behavior:
- Trigger fires on the **rising edge** of both totals exceeding threshold simultaneously
- **Hysteresis**: stays in `WAIT_FALL` until both totals drop ≤ their thresholds (prevents double-triggering on a single event)
- Note (from code comment, L85): `TRIGGER_ENABLE` is now processed inside `trig_algo_support`, not locally (changed 2015-09-16) ✅ verified 2026-04-24 — L84 comment `--20150916: TRIGGER_ENABLE is now processed in trig_algo_support.`
- Note: a local `TRIGGER_TYPE` signal captures `TRIG_TYPE` in WAIT_TRIG (L88), but the port map to `trig_algo_support` passes `TRIG_TYPE` directly — the captured signal is unused in the port map ✅ verified 2026-04-24 — L72 (`signal TRIGGER_TYPE : std_logic_vector(7 downto 0)`) vs L110 (`TRIGGER_TYPE_CODE => TRIG_TYPE`)

### Generic Support Layer

All standard trigger bookkeeping (FIFO management, prescaler, holdoff, veto, monitor FIFO) is delegated to a `trig_algo_support` instance (U2). See [`MTRG_trig_algo_support.md`](./MTRG_trig_algo_support.md).

---

## Key Differences vs sum_hits_x.vhd

`sum_hits_x` triggers when the X-plane total alone exceeds its threshold. `sum_hits_xy` requires **both planes** to exceed simultaneously — providing a 2D spatial coincidence requirement.

---

## See Also

- [`MTRG_trig_algo_support.md`](./MTRG_trig_algo_support.md) — generic trigger infrastructure used by all MTRG algorithms
- [`MTRG_sum_hits_X.md`](./MTRG_sum_hits_X.md) — single-axis X variant
- [`MTRG_support_modules.md`](./MTRG_support_modules.md) — supporting infrastructure
- [`deep_fpga_MTRG_MAIN.md`](../deep_fpga_MTRG_MAIN.md) — MTRG top-level architecture
- [`fpga.md`](../fpga.md) — VHDL index

---

*Analysis date: 2026-04-24*
