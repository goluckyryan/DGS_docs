# MTRG — Master Trigger Firmware

The Master Trigger (MTRG) is the central trigger decision-maker for the DGS (Digital Gamma-ray Spectrometer) system. It collects detector data from up to 8 Routers via high-speed serial links, runs configurable trigger algorithms, and distributes synchronized trigger decisions back to all Routers every 2 µs (one 20-frame cycle at 50 MHz).

The hardware contains three programmable devices:

| Device | Chip | Role |
|--------|------|------|
| Main FPGA | Virtex-4 XC4VLX80 (ISE) / Kintex UltraScale XCK060 (Vivado) | Trigger logic, algorithms, SERDES links |
| VME FPGA | Spartan-3 XC3S400 | VME slave, main FPGA configuration |
| CPLD | XC9500XL XC95144XL | Fast strobe multiplicity threshold |

> 🔌 **Front panel connectors & pinouts:** see `connectors.md` — covers the 125-pin hard metric SERDES connector (links A–H/L/R/U), NIM I/O, CPLD ribbon cables, and Aux I/O header.

## Table of Contents

- [Firmware Modules](#firmware-modules)
- [Repository Layout](#repository-layout)

## Firmware Modules

- [deep_fpga_MTRG_MAIN.md](deep_fpga_MTRG_MAIN.md) — ISE-based main FPGA (production, Virtex-4)
- [deep_fpga_MTRG_VIVADO.md](deep_fpga_MTRG_VIVADO.md) — Vivado-based main FPGA (Kintex UltraScale)
- [deep_fpga_MTRG_VME.md](deep_fpga_MTRG_VME.md) — VME interface and configuration FPGA (Spartan-3)
- [deep_fpga_MTRG_CPLD.md](deep_fpga_MTRG_CPLD.md) — Fast strobe CPLD (XC9500XL)

## Repository Layout

```
MTRG/
├── Firmware/
│   ├── MAIN_FPGA/trunk/          # ISE 13.4 project (production)
│   ├── VIVADO_MAIN_FPGA/trunk/   # Vivado 2018.3 project
│   ├── VME_FPGA/A32D32_VME_FPGA/ # Spartan-3 VME controller
│   ├── DGS_CPLD/                 # XC9500XL CPLD
│   └── offshoots/                # Archived branch variants
└── Gretina Trigger/              # Legacy GRETINA hardware reference
```

---
*Source: `DGS_tools_pack/raw_FPGA/` + `DGS_tools_pack/fpga/` (git repos on gitlab.phy.anl.gov/dgs-tools-pack). Created: 2026-04-05.*
