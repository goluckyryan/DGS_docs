# Gammasphere Detector Geometry

Stability: C3 - Structural / stable

**Source:** `DGS_tools_pack/dgs_analysis/armory/parquet_pysort/angtheta.dat`

Gammasphere has **110 detector positions** (GS holes 1–110), arranged in **17 rings** at fixed polar angles θ from the beam axis. GS numbers are fixed — they identify the physical position in the array, not the crystal installed there. ✅ verified 2026-04-18 — `dgs_analysis/armory/parquet_pysort/angtheta.dat` (110 lines, 17 unique θ values: 17.3°–162.7°)

✅ verified 2026-04-07 — `angtheta.dat`: 110 lines, 17 unique θ values, line N = GS hole N. Ring 1 (17.3°) = GS001–004,006 ✓; Ring 9 (90.0°) = GS049,051,053–058,060,062 ✓; all ring counts and angles match.

---

## Table of Contents
- [Ring Structure](#ring-structure)
- [Full GS Hole → θ Angle Map](#full-gs-hole--angle-map)
- [Key Properties](#key-properties)
- [Collector Box → GS Hole Assignment](#collector-box--gs-hole-assignment)
- [Detector Installation Database (May 2017)](#detector-installation-database-may-2017)
- [Related Files](#related-files)
- [Cross-References](#cross-references)

---

## Ring Structure

| Ring | θ (deg) | # Detectors | GS Holes |
|------|---------|-------------|----------|
| 1 | 17.3° | 5 | GS001, GS002, GS003, GS004, GS006 | ✅ verified 2026-04-14 — `angtheta.dat` lines 1–6
| 2 | 31.7° | 5 | GS005, GS007, GS008, GS009, GS010 | ✅ verified 2026-04-14 — `angtheta.dat` lines 5,7–10
| 3 | 37.4° | 5 | GS011, GS012, GS013, GS014, GS016 | ✅ verified 2026-04-18 — `angtheta.dat` lines 11,12,13,14,16 all = 37.4
| 4 | 50.1° | 10 | GS015, GS017, GS018, GS019, GS020, GS021, GS022, GS023, GS024, GS026 | ✅ verified 2026-04-20 — `angtheta.dat` lines 15,17–24,26 all = 50.1
| 5 | 58.3° | 5 | GS025, GS027, GS028, GS030, GS032 | ✅ verified 2026-04-18 — `angtheta.dat` lines 25,27,28,30,32 all = 58.3
| 6 | 69.8° | 10 | GS029, GS031, GS033, GS034, GS035, GS036, GS037, GS038, GS040, GS042 | ✅ verified 2026-04-14 — `angtheta.dat` lines 29,31,33–38,40,42 all = 69.8
| 7 | 79.2° | 5 | GS039, GS041, GS044, GS046, GS048 | ✅ verified 2026-04-18 — `angtheta.dat` lines 39,41,44,46,48 all = 79.2
| 8 | 80.7° | 5 | GS043, GS045, GS047, GS050, GS052 | ✅ verified 2026-04-14 — `angtheta.dat` lines 43,45,47,50,52 all = 80.7
| 9 | 90.0° | 10 | GS049, GS051, GS053, GS054, GS055, GS056, GS057, GS058, GS060, GS062 | ✅ verified 2026-04-20 — `angtheta.dat` lines 49,51,53–58,60,62 all = 90.0
| 10 | 99.3° | 5 | GS059, GS061, GS064, GS066, GS068 | ✅ verified 2026-04-20 — `angtheta.dat` lines 59,61,64,66,68 all = 99.3
| 11 | 100.8° | 5 | GS063, GS065, GS067, GS070, GS072 | ✅ verified 2026-04-20 — `angtheta.dat` lines 63,65,67,70,72 all = 100.8
| 12 | 110.2° | 10 | GS069, GS071, GS073, GS074, GS075, GS076, GS077, GS078, GS080, GS082 | ✅ verified 2026-04-20 — `angtheta.dat` lines 69,71,73–78,80,82 all = 110.2
| 13 | 121.7° | 5 | GS079, GS081, GS083, GS084, GS086 | ✅ verified 2026-04-20 — `angtheta.dat` lines 79,81,83,84,86 all = 121.7
| 14 | 129.9° | 10 | GS085, GS087, GS088, GS089, GS090, GS091, GS092, GS093, GS094, GS096 | ✅ verified 2026-04-20 — `angtheta.dat` lines 85,87–94,96 all = 129.9
| 15 | 142.6° | 5 | GS095, GS097, GS098, GS099, GS100 | ✅ verified 2026-04-21 — `angtheta.dat` lines 95,97–100 all = 142.6
| 16 | 148.3° | 5 | GS101, GS102, GS103, GS104, GS106 | ✅ verified 2026-04-20 — `angtheta.dat` lines 101–104,106 all = 148.3
| 17 | 162.7° | 5 | GS105, GS107, GS108, GS109, GS110 | ✅ verified 2026-04-21 — `angtheta.dat` lines 105,107–110 all = 162.7

**Total: 110 detectors, 17 rings** ✅ verified 2026-04-18 — `angtheta.dat` (110 entries, 17 distinct polar angles)

---

## Full GS Hole → θ Angle Map

| GS | θ | GS | θ | GS | θ | GS | θ | GS | θ |
|----|---|----|---|----|---|----|---|----|---|
| 001 | 17.3° | 023 | 50.1° | 045 | 80.7° | 067 | 100.8° | 089 | 129.9° |
| 002 | 17.3° | 024 | 50.1° | 046 | 79.2° | 068 | 99.3° | 090 | 129.9° |
| 003 | 17.3° | 025 | 58.3° | 047 | 80.7° | 069 | 110.2° | 091 | 129.9° |
| 004 | 17.3° | 026 | 50.1° | 048 | 79.2° | 070 | 100.8° | 092 | 129.9° |
| 005 | 31.7° | 027 | 58.3° | 049 | 90.0° | 071 | 110.2° | 093 | 129.9° |
| 006 | 17.3° | 028 | 58.3° | 050 | 80.7° | 072 | 100.8° | 094 | 129.9° |
| 007 | 31.7° | 029 | 69.8° | 051 | 90.0° | 073 | 110.2° | 095 | 142.6° |
| 008 | 31.7° | 030 | 58.3° | 052 | 80.7° | 074 | 110.2° | 096 | 129.9° |
| 009 | 31.7° | 031 | 69.8° | 053 | 90.0° | 075 | 110.2° | 097 | 142.6° |
| 010 | 31.7° | 032 | 58.3° | 054 | 90.0° | 076 | 110.2° | 098 | 142.6° |
| 011 | 37.4° | 033 | 69.8° | 055 | 90.0° | 077 | 110.2° | 099 | 142.6° |
| 012 | 37.4° | 034 | 69.8° | 056 | 90.0° | 078 | 110.2° | 100 | 142.6° |
| 013 | 37.4° | 035 | 69.8° | 057 | 90.0° | 079 | 121.7° | 101 | 148.3° |
| 014 | 37.4° | 036 | 69.8° | 058 | 90.0° | 080 | 110.2° | 102 | 148.3° |
| 015 | 50.1° | 037 | 69.8° | 059 | 99.3° | 081 | 121.7° | 103 | 148.3° |
| 016 | 37.4° | 038 | 69.8° | 060 | 90.0° | 082 | 110.2° | 104 | 148.3° |
| 017 | 50.1° | 039 | 79.2° | 061 | 99.3° | 083 | 121.7° | 105 | 162.7° |
| 018 | 50.1° | 040 | 69.8° | 062 | 90.0° | 084 | 121.7° | 106 | 148.3° |
| 019 | 50.1° | 041 | 79.2° | 063 | 100.8° | 085 | 129.9° | 107 | 162.7° |
| 020 | 50.1° | 042 | 69.8° | 064 | 99.3° | 086 | 121.7° | 108 | 162.7° |
| 021 | 50.1° | 043 | 80.7° | 065 | 100.8° | 087 | 129.9° | 109 | 162.7° |
| 022 | 50.1° | 044 | 79.2° | 066 | 99.3° | 088 | 129.9° | 110 | 162.7° |

---

## Key Properties

- **Total detectors:** 110
- **Total rings:** 17
- **Beam axis:** θ = 0° (forward) / 180° (backward)
- **Equatorial ring:** θ = 90° (10 detectors, ring 9) — highest coverage at 90°
- **Forward hemisphere:** rings 1–8 (θ < 90°) — 50 detectors
- **Backward hemisphere:** rings 10–17 (θ > 90°) — 50 detectors
- **Symmetric:** array is symmetric about 90° (ring angles mirror: 17.3↔162.7, 31.7↔148.3, etc.)
- **Ring sizes:** alternating 5 and 10 detectors — rings 4, 6, 9, 12, 14 have 10 detectors each

---

## Collector Box → GS Hole Assignment

| Collector Box | Pi | GS Holes Owned |
|---------------|----|----------------|
| 201 | pi0 (South-East) | GS 2,4,6,...,60,70 (even GS 2–60 + GS 70 — 31 detectors) |
| 202 | pi1 (South-West) | GS 62,64,66,...,110 (even GS 62–110 — 25 detectors) |
| 203 | pi2 (North-East) | GS 1,3,5,...,59 (odd) — old piserver, not in repo |
| 204 | pi3 (North-West) | GS 61,63,...,109 (odd) — old piserver, not in repo |

✅ verified 2026-04-08 — `collectorboxpi/CollectorBox_RevA/iocBoot/iocCollectorApp/st_20{1,2,3,4}.cmd` (grep DetNbr). Location from `collectorboxpi/README.md:L257-259` + NFS piserver README.

---

## Detector Installation Database (May 2017)

_Source: `DGS_tools_pack/DGS_SVN/dgs/Documentation/North_db.csv` + `South_db.csv`_

These CSVs record which physical crystals were installed in which GS holes during the May 2017 Gammasphere commissioning. Columns: **GS #** (hole), **Hoze #** (housing ID, e.g. A26/B15), **Det/Ge #** (Ge crystal serial), **HV** (bias voltage in V), **SB #** (slope box number), **SB pos** (position on slope box: T/B/L/R/TL/TR/BL/BR), **BGO #** (BGO crystal number), **BGO Install Date**.

Key observations:
- Holes 1, 2, 3, 4, 5, 6 have no Ge or HV entry — likely forward/backward beam holes or permanently empty positions
- GS 53 (north) and GS 58 (south) are explicitly marked `EMPTY` ✅ verified 2026-04-11 — `North_db.csv:L53` + `South_db.csv:L58`
- HV ranges from 3000–4800 V (typically 3500–4000 V per crystal) ✅ verified 2026-04-11 — `North_db.csv` + `South_db.csv`: min=3000V, max=4800V
- Two crystals noted as ANL serials (`ANL00` at GS 95, `ANL16` at GS 98) rather than numeric serials
- BGO #s run 1–113 (not contiguous); SBX positions use compass notation (TL=top-left, BR=bottom-right, etc.)
- Install dates cluster around May 15–31, 2017

> ⚠️ This database reflects the 2017 configuration. Crystal assignments may have changed due to repairs/replacements. Cross-reference with current EPICS PVs (`GS${N}_SBX_Present`, slope box telemetry) for live status.

**North hemisphere summary (GS odd 1–109):** 55 positions; ~43 with Ge crystals installed
**South hemisphere summary (GS even 2–110):** 55 positions; ~43 with Ge crystals installed

---

## Related Files

- `angtheta.dat` — one angle per line, line N = GS hole N: `DGS_tools_pack/dgs_analysis/armory/parquet_pysort/angtheta.dat`
- `map.dat` — maps DAQ channel IDs to GS holes and detector type (GE/BGO/AUX/SIDE): `DGS_tools_pack/dgs_analysis/armory/parquet_pysort/map.dat`
- `collectorbox_PVs.md` — EPICS PV list for collector box; explains GS/MOD/VME_GS/Ge_ID numbering

---

*Created: 2026-04-05*

## Cross-References

- `knowledgeBase/collectorboxpi.md` — Collector box Pi IOC; detector assignments per box (SE/SW/NE/NW)
- `knowledgeBase/collectorbox_PVs.md` — PV naming uses GS hole number (e.g. `GS001_...` through `GS110_...`)
- `knowledgeBase/influxdb_grafana.md` — Temperature monitoring per GS hole (`gsid=NNN` tag in InfluxDB)
- `knowledgeBase/sbx.md` — Slope Box Extension; GS_ID dongle encodes hole number for BGO HV addressing
- `knowledgeBase/hardware_architecture.md` — System topology; 110 GS holes × 4 signals (1 Ge + 3 BGO)
