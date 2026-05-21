# GS_model — Gammasphere 3D Detector Viewer

**Location:** `~/GS_model/`  
**Status:** ✅ Complete (2026-05-20)  
**Run:** `python3 ~/GS_model/viewer/server.py` → http://192.168.203.132:8765

---

## Overview

Interactive 3D visualization of all 110 Gammasphere HPGe detector positions. Built with Three.js (3D sphere) and a pure-Python HTTP server (no external dependencies). Supports live temperature coloring via EPICS caget or InfluxDB.

---

## Files

| File | Description |
|------|-------------|
| `gs_geometry.json` | Canonical geometry: GS#, θ, φ, x, y, z (cm) for all 110 detectors |
| `gs_mapping.json` | GS# → VME/digitizer/channel mapping (cached from EPICS) |
| `gs_hose_map.json` | LN2 hose map: hose label → GS# |
| `gs_temp_high.json` | High-temp threshold overrides per detector |
| `build_geometry.py` | Generates `gs_geometry.json` from `gsang.h` data |
| `viewer/server.py` | HTTP server: serves static files + `/api/temps/epics` + `/api/temps/influx` + `/api/mapping` |
| `viewer/index.html` | Three.js 3D sphere viewer |
| `viewer/flat.html` | Flat hemisphere unfolding view (north/south disks) |
| `viewer/rings.html` | Ring layout view |
| `viewer/OrbitControls.js` | Three.js orbit controls (bundled) |
| `viewer/three.min.js` | Three.js library (bundled) |

---

## Server API

The server (`server.py`) runs on port **8765** and exposes:

| Endpoint | Description |
|----------|-------------|
| `GET /` | Serves `index.html` |
| `GET /api/temps/influx` | Latest HPGe temps from InfluxDB (`HPGeTemp` db on DCS2) |
| `GET /api/temps/epics` | Live temps via `caget MOD###_DV_TEMP` for all 110 detectors |
| `GET /api/mapping` | GS# → VME/dig/channel mapping JSON |

Response format for temps:
```json
{"1": 85.2, "2": 84.1, ..., "110": 91.0}
```
Keys are GS# as strings (1–110). Missing detectors are omitted.

### InfluxDB connection
- Host: `http://192.168.203.56:8181`
- DB: `HPGeTemp`, table: `Temperature`
- Token: embedded in `server.py` (read-only)
- Query: last value per `gsid` within 1 hour

### EPICS connection
- Binary: `DGS_tools_pack/ANLDAQ/EPICS/base-7.0/bin/linux-aarch64/caget`
- PVs: `MOD001_DV_TEMP` … `MOD110_DV_TEMP`
- `EPICS_CA_ADDR_LIST=192.168.203.78`, port 5064

---

## Viewer Features

### index.html (3D sphere)
- All 110 detectors placed at correct (θ, φ) positions on a sphere (r = 24.965 cm)
- **Color modes:** Temperature (K) · Ring · GS Number
- **Hover tooltip:** GS#, θ, φ, ring, temperature
- **Live temp button:** fetches from `/api/temps/epics` or `/api/temps/influx`
- Beam axis shown in yellow (Z direction)
- Adjustable opacity
- Optional GS# labels

### flat.html (hemisphere unfolding)
- Unfolds north (odd GS#) and south (even GS#) hemispheres into flat disks
- North disk: pole = -X out of plane, display up = +Y, display right = +Z
- South disk: pole = +X out of plane, display up = +Y, display right = -Z

### rings.html (ring layout)
- Detectors arranged by ring number

---

## Coordinate System (from gsang.h)

Right-hand coordinate system:
- **Z** = beam axis, pointing out (θ = 0°)
- **Y** = vertical, pointing up (-Y is gravity)
- **X** = horizontal (X×Y = Z)
- **θ** = polar angle from Z (beam) axis
- **φ** = azimuthal angle in X-Y plane: 0°/360° → -Y (down), 90° → +X, 180° → +Y, 270° → -X
- **r** = 24.965 cm (target → front face of Ge crystal)

Axis colors in viewer: 🟡 Yellow = Z (beam), 🟢 Green = Y (up), 🔴 Red = X

---

## Data Sources

- Geometry: `DGS_tools_pack/gebsort/gsang.h`
- θ backup: `DGS_tools_pack/dgs_analysis/armory/parquet_pysort/angtheta.dat`
- Mapping: EPICS PVs (cached in `gs_mapping.json`)
- Temps: EPICS caget or InfluxDB HPGeTemp

---

## See Also

- [lnfill.md](lnfill.md) — LN2 fill system (hose map used by `gs_hose_map.json`)
- [influxdb_grafana.md](influxdb_grafana.md) — InfluxDB on DCS2 (temp data source)
- [overview_DGS.md](overview_DGS.md) — DGS system overview, detector numbering
