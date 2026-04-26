# DGS/DFMA EPICS Implementation — Tools & Workflow

Stability: C2 - Active / semi-stable

_How the DGS EPICS PV database is generated, maintained, and accessed. Sources: wiki pages `/gsdaq/The_DGS/DFMA_EPICS_Implementation`, `/gsdaq/Database_generation_and_maintenance`, `/gsdaq/Engineer_access_to_the_system_from_LabWindows`. Fetched 2026-04-25._

---

## Table of Contents

- [1. PV Database Generation — Excel → Python Workflow](#1-pv-database-generation--excel--python-workflow)
- [2. GUI Interfaces](#2-gui-interfaces)
  - [GammaWare (LabWindows)](#gammaWare-labwindows)
  - [Carlware (EDM)](#carlware-edm)
  - [CSS Interfaces (Tim Madden)](#css-interfaces-tim-madden)
- [3. Remote Access via Sonata](#3-remote-access-via-sonata)
- [4. VME Peek/Poke via IOC Console](#4-vme-peekpoke-via-ioc-console)
- [5. Key Machines](#5-key-machines)
- [See Also](#see-also)

---

## 1. PV Database Generation — Excel → Python Workflow

Developed by **Madden, Oberling, and Anderson**.

- Firmware developers fill in an **Excel spreadsheet** defining **every bit in every register**.
- Columns specify: which bits get EPICS PVs, what GUI control type (bit / multi-bit / integer).
- A **Python script** parses a CSV export of the spreadsheet and auto-generates the EPICS database source.
- This same approach also regenerates the **CSS GUI** when firmware changes.

### Spreadsheet Rules (from Tim Madden, 2013-03-28)
- `()` in a register name is **illegal** (C treats it as a function call) — use `_xx_` instead.
- If a register name **ends in a number**, Python treats it as a digitizer channel and processes it differently → add a trailing `_` to prevent this.

**Benefit:** When firmware changes, updating the spreadsheet and re-running the script keeps both the PV database and the GUI in sync with minimal manual effort.

---

## 2. GUI Interfaces

### GammaWare (LabWindows)

- A **LabWindows/CVI** program (National Instruments) providing low-level register/bit access.
- Runs on **WDGS** — a Windows 7 laptop in the Gammasphere data room.
- Used for diagnosis: JTAG access, Chipscope, Xilinx ISE 14.7, and register-level inspection.
- Also includes **IMPACT** (Xilinx programming tool).

### Carlware (EDM)

- EDM-based GUI, originally written by Carl Lionberger for GRETINA, adopted by DGS.
- Launched with the command: **`dgscommander`** (DGS) ✅ verified 2026-04-25 — DGS_SVN 20230818_edm EDM launcher script; also confirmed in `dgsSupport.db` comments and `nfs_layout.md`. **`dfmacommander`** (DFMA/DSSD) — ⚠️ unverified - source needed (wiki claim; no DFMA launcher found in local repo).
- DFMA version has **5 Routers** under the trigger button; DGS has **4 Router Triggers** (RTR1–RTR4, in VME03/06/09/12) + 1 Master Trigger (MTRG in VME10). ✅ verified 2026-04-25 — `ioc/boot/vme03.cmd`, `vme06.cmd`, `vme09.cmd`, `vme12.cmd` each contain `asynTrigRouterConfig1("RTRn",4,7)`; `ioc.md` slot table (verified 2026-04-24) confirms 4 RTRGs total. **Wiki claim of "3 Routers" is incorrect.**
- Navigating to a specific register requires multiple nested windows (e.g., trigger → router → SERDES → link controls → transmit power).
- **Limitations:**
  - Many trigger registers (including diagnostic FIFOs) are **not accessible** from Carlware.
  - No generic VME peek/poke for registers outside the defined interface.
  - Still has known small bugs.
- LBNL continues to update Carlware for GRETINA, but DGS and DFMA versions **diverge**.
- Mike Carpenter has scripts that handle standard initialization via Carlware.

### CSS Interfaces (Tim Madden)

- Control System Studio (CSS) GUIs — functionally equivalent to Carlware.
- Auto-generated from the Excel register maps via Python scripts (same workflow as PV database).
- **Advantage:** updating firmware → update spreadsheet → regenerate both DB and CSS GUI.

---

## 3. Remote Access via Sonata

To reach the Gammasphere data room ("onenet" subnet) remotely:

1. Run an **X windows package** on your local machine (e.g., Cygwin, Putty + Xming).
2. SSH to gateway: `ssh -Y sonata.phy.anl.gov` (ANL domain password required).
3. From Sonata:
   - **Windows PC (WDGS):** `rdesktop wdgs -g 1280x1024` → login as `topoadmin`
   - **DGS Linux (dgs1):** `ssh -Y dgs@dgs1` (standard Gammasphere password)
   - **DFMA/DSSD Linux (nat2):** `ssh -Y dgs@nat2` (same standard password)

**Note:** The "standard Gammasphere password" is shared across all onenet machines.

---

## 4. VME Peek/Poke via IOC Console

From Carlware's **Terminals** button, you can access console ports of individual IOC processor boards.

| Command | Function |
|---------|----------|
| `d`     | Display (read) memory locations |
| `m`     | Modify (write) memory locations |
| `reboot`| Reboot the IOC |

✅ verified 2026-04-25 — `IOC_cmd.md:L410,L460,L498-499,L570` documents `d`, `m`, and `reboot` as standard VxWorks MVME5500 shell commands with usage examples. `d` takes address/count/width args; `m` opens address for interactive modification; `reboot` hard-reboots (Ctrl-X is keyboard alias). All confirmed from DGS IOC documentation.

- The **Soft IOC** is a virtual IOC running in software on the Linux machine.
- If the Soft IOC is not running, many EPICS features will not work.
- Direct EPICS PV access from the Linux prompt: `caput` / `caget` commands.

---

## 5. Key Machines

| Machine | OS | Purpose |
|---------|-----|---------|
| **WDGS** | Windows 7 | GammaWare, JTAG, Chipscope, Xilinx ISE 14.7; located on relay rack in data room |
| **dgs1** | Linux | DGS Linux host; runs dgsSoftIOC, Carlware/CSS |
| **nat2** | Linux | DFMA/DSSD Linux host |
| **sonata.phy.anl.gov** | Linux | ANL gateway into onenet subnet |

---

## See Also

- `EPICS.md` — General EPICS framework, record types, Channel Access
- `EPICS_DB_templates.md` — DGS-specific DB template details (dgsGlobals, fanout structure)
- `EPICS_RTrig_templates.md` — RTrig register template PVs
- `IOC_cmd.md` — IOC startup scripts and commands
- `ANLDAQ.md` — DAQ GUI (PyQt6) that reads and displays the PVs generated by this workflow
- `ANLDAQ_commander.md` — commander.py run control GUI; top-level consumer of DGS PVs
