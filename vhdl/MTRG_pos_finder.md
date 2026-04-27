# MTRG pos_finder.vhd — Local TDC Position Lookup
Stability: C3 - Structural / stable

Source: `~/FPGA_svn2git/MTRG_git/MAIN_FPGA/trunk/Source/pos_finder.vhd`  
Author: Katelyn Kufahl, 2013-06-28  
Lines: 605

---

## Table of Contents

- [Purpose](#purpose)
- [Entity](#entity)
- [Architecture](#architecture)
  - [ROM Selection (Generic)](#rom-selection-generic)
  - [Output Packing](#output-packing)
  - [Registered Output (1-cycle latency)](#registered-output-1-cycle-latency)
- [How It Fits Into the TDC Pipeline](#how-it-fits-into-the-tdc-pipeline)
- [Overlap / Redundancy Design Note](#overlap--redundancy-design-note)
- [Companion Component Declaration](#companion-component-declaration)
- [See Also](#see-also)

---

## Purpose

`pos_finder` is a single pipelined stage of the DGS TDC (Time-to-Digital Converter) thermometer-code decoder. It converts an 11- or 12-bit **slice** of presumably thermometer-coded TDC data into the binary position of where the 0→1 (or 1→0) edge transition occurred within that slice. Multiple `pos_finder` instances are pipelined in `vernier_pos_finder` to decode the full 64-bit TDC word in segments.

---

## Entity

```vhdl
entity pos_finder is
generic ( 
    rom_size : integer    -- 11 (small) or 12 (large)
);
port (
    clk_tdc   : in  std_logic;
    din       : in  std_logic_vector((rom_size-1) downto 0);  -- thermometer slice
    edge_pos  : out std_logic_vector(3 downto 0);             -- 4-bit edge position result
    valid_out : out std_logic                                  -- '1' if a valid edge was found in this slice
);
end pos_finder;
```

---

## Architecture

### ROM Selection (Generic)

Two ROM tables are compiled into the entity:

| Generic | Table | Size | Notes |
|---------|-------|------|-------|
| `rom_size = 11` | `alt_small_lookup_bram` | 2048 × 5-bit | Active small ROM (11-bit input) |
| `rom_size = 12` | `large_lookup_bram` | 4096 × 5-bit | Large ROM (12-bit input) |

A second `small_lookup_bram` table exists but is commented out (`--lookup_bram <= small_lookup_bram`); the alternate table `alt_small_lookup_bram` is the active one for `rom_size=11`. ✅ verified 2026-04-24 — pos_finder.vhd:L587-594 (IF_SMALL_ROM generate: `lookup_bram <= alt_small_lookup_bram`; L589 comment shows `small_lookup_bram` is commented out; IF_LARGE_ROM generate: L592-594)

### Output Packing

The 5-bit ROM output is split as:
- `dout[4:1]` → `edge_pos` (4-bit binary position of the edge within the slice)
- `dout[0]` → `valid_out` (1 if this slice contains a valid edge)

✅ verified 2026-04-24 — pos_finder.vhd:L596-597 (`edge_pos <= dout(4 downto 1)`; `valid_out <= dout(0)`)

**Position encoding:** The ROM encodes position values as 5-bit binary (e.g., `"11000"` = 24, `"10111"` = 23, `"00001"` = 1). `"00000"` means no valid edge (valid_out = 0). ✅ verified 2026-04-24 — pos_finder.vhd:L63-66 (ROM entries: `"11000"` at index 7, `"10111"` at indexes 14-15, `"00000"` at 0-6 confirmed as no-edge values)

### Registered Output (1-cycle latency)

```vhdl
edge_finder : process (clk_tdc)
begin
    if(clk_tdc'event and clk_tdc = '1') then
        dout <= lookup_bram(conv_integer(din));
    end if;
end process edge_finder;
```

One clock cycle of pipeline latency per `pos_finder` instance. ✅ verified 2026-04-24 — pos_finder.vhd:L599-604 (edge_finder process body matches exactly)

---

## How It Fits Into the TDC Pipeline

The 64-bit TDC word is split into overlapping 11- or 12-bit slices. Each slice is fed into one `pos_finder` instance (pipelined stages in `vernier_pos_finder`). The ROM lookup detects the thermometer edge position within that slice:
- If the edge falls in this slice: `valid_out = '1'`, `edge_pos` = binary position (0–15 for 11-bit, 0–15 for 12-bit)
- If no edge in this slice: `valid_out = '0'`, `edge_pos` = undefined / ignored

The pipelined `vernier_pos_finder` assembles the final 6-bit TDC position from whichever stage reports a valid edge.

---

## Overlap / Redundancy Design Note

From the header comment: slices are overlapped by a couple of bits to ensure the edge is never missed at a slice boundary. This explains why multiple position values can appear in the ROM — the highest-priority (most significant) matching slice wins in the overall pipeline.

---

## Companion Component Declaration

Declared in `trigger_comp_defs.vhd` (L228–L237): ✅ verified 2026-04-24 — trigger_comp_defs.vhd:L228,L238 (component pos_finder … end component pos_finder; port list matches entity exactly)
```vhdl
component pos_finder is
generic ( rom_size : integer );
port (
    clk_tdc   : in  std_logic;
    din       : in  std_logic_vector((rom_size-1) downto 0);
    edge_pos  : out std_logic_vector(3 downto 0);
    valid_out : out std_logic
);
end component pos_finder;
```

---

## See Also

- [`MTRG_support_modules.md`](./MTRG_support_modules.md) — `vernier_pos_finder` (parent instantiator), `jta_odelay`, `jta_vernier_pos_finder`
- [`MTRG_calc_total_sum.md`](./MTRG_calc_total_sum.md) — downstream consumer of TDC position data
- [`deep_fpga_MTRG_MAIN.md`](../deep_fpga_MTRG_MAIN.md) — MTRG top-level architecture overview
- [`fpga.md`](../fpga.md) — VHDL index

---

*Analysis date: 2026-04-24*
