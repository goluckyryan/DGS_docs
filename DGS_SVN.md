# DGS_SVN — Legacy SVN Repository Mirror

## What It Is

A mirror/checkout of the DGS legacy **SVN (Subversion)** repository. This is the historical source of DGS code before the migration to Git.

Contents:
- `dgs/` — Main DGS SVN tree (large, many subdirectories)
- `findFile.sh` — Utility script to search within the SVN tree

---

## Contents of `dgs/`

The SVN tree contains a broad historical archive of DGS development:

| Directory | Contents |
|-----------|----------|
| `17pc030-GretinaTRGT` | Gretina trigger target hardware |
| `17pc031-HeliosPreampPower` | Helios preamp power supply |
| `20180921`, `20230818_edm` | Dated snapshots / EDM displays |
| `con6_20220728` | con6 (Solaris host) snapshot |
| `con6_EPICS_base` | EPICS base from con6 era |
| `con6_work` | con6 working files |
| `daq_system_tags` | DAQ system version tags |
| `Data_Generator` | Data generator / test tools |
| `DB_backup_20240205` | Database backup (Feb 2024) |
| `Detector_Repair` | Detector repair records/tools |
| `devel8`, `dgs_devel8` | Development branch 8 |
| `devel_tracker` | Development tracking |
| `DGS1_clean_folders` | DGS1 (old system) clean directory tree |
| `DGS1_total_backup` | Full DGS1 backup |
| `dgsext` | DGS extensions |
| `DGSFiberExpander` | Fiber expander hardware |
| `dgsSoftIOC` | Soft IOC (pre-Git version) |
| `ArdisiaDocuments.zip` | Ardisia documentation |

---

## `findFile.sh`

A utility to locate files within the large SVN tree by name pattern.

```sh
./findFile.sh <pattern>
```

---

## Usage / Status

This is primarily an **archive** and **reference** — not actively developed. The active development has migrated to Git repositories (fpga/, ioc/, vxworks/, etc.).

Useful for:
- Finding historical implementations
- Recovering old configuration files
- Understanding the DGS1 → DGS2 migration
- Looking up Solaris/con6-era build environments

---

## Connections to Other Subsystems

- **vxworks/** — `migration.md` references the con6/Solaris-to-Ubuntu migration this SVN contains
- **fpga/** — FPGA firmware may have early versions in the SVN tree
- **ioc/** — pre-Git IOC files are in `dgsSoftIOC/`
