# InfluxDB & Grafana on DCS2

_DCS2 (192.168.203.56) runs InfluxDB 3 and Grafana for DGS monitoring._

---

## Infrastructure

| Component | Version | Port | Process owner |
|-----------|---------|------|--------------|
| InfluxDB 3 (influxdb3) | latest (as of 2026-04) | 8181 (HTTP) | `adminrt` | ✅ verified 2026-04-09 — `curl http://192.168.203.56:8181/health` returns `{"error":"the request was not authenticated"}` (port live, auth required)
| Grafana | 12.3.1 | 3000 (HTTP) | `grafana` |

Both run as persistent systemd services on DCS2 (Rocky Linux 8).

**InfluxDB data dir:** `/home/phy/adminrt/.influxdb/data/` (no read access from `dcsu`)
**Grafana DB:** `/var/lib/grafana/grafana.db` (SQLite, no read access from `dcsu`)

---

## InfluxDB 3

### API Endpoint
```
http://192.168.203.56:8181
```
All requests require a Bearer token in the `Authorization` header.

### Known Databases

| Database | What's in it | Who writes |
|----------|-------------|-----------|
| `HPGeTemp` | Detector temperatures (all 110 GS holes), Pi board temp | lnFill `SaveTemp.sh` → `StoreDetTemps.py` | ✅ verified 2026-04-09 — `StoreDetTemps.py:L57` (curl write to 192.168.203.56:8181/api/v3/write_lp?db=HPGeTemp)
| `DGS` | DGS PV snapshots (line protocol from `dumpPVs.py`) | `snapshot_pv/dumpPVs.py` (commented-out code, not yet active) |

### Write API (InfluxDB line protocol)
```bash
# Write line protocol data
curl "http://192.168.203.56:8181/api/v3/write_lp?db=HPGeTemp" \
  --header "Authorization: Bearer <token>" \
  --data-binary @data.txt
```

### Query API (SQL)
```bash
# SQL query (InfluxDB 3 uses SQL, not InfluxQL)
# IMPORTANT: the "db" field is required — omitting it returns a 400 error
# Table name "Temperature" must be double-quoted (case-sensitive)
curl "http://192.168.203.56:8181/api/v3/query_sql" \
  --header "Authorization: Bearer <token>" \
  --header "Accept: application/json" \
  --header "Content-Type: application/json" \
  -d '{"db": "HPGeTemp", "q": "SELECT gsid, value FROM \"Temperature\" WHERE time >= now() - interval '\''30 minutes'\'' ORDER BY value DESC LIMIT 10"}'
```

Returns a JSON array of row objects. Example response:
```json
[{"gsid": "041", "value": 94.64}, {"gsid": "043", "value": 94.52}, ...]
```

### Reading Detector Temperature via EPICS CA (Alternative)

Each SBX (Slope Box) Pi publishes live detector temperatures as EPICS PVs.
This is a fast alternative to InfluxDB for spot-checking a single detector.

```bash
# Set environment (onenet subnet, SBX Pi CA ports are 5080/5081)
export EPICS_CA_ADDR_LIST=192.168.203.0/24   # or specific Pi IP
export EPICS_CA_AUTO_ADDR_LIST=NO
export EPICS_CA_SERVER_PORT=5080

# Read temperature for detector 033 (sbxh3)
caget Det033:SlopeBox:Temp

# Or point directly at sbxh3 IP
export EPICS_CA_ADDR_LIST=192.168.203.164
caget Det033:SlopeBox:Temp
```

**When to use which:**
- **InfluxDB query** — bulk reads across all detectors, historical trends, programmatic access
- **caget** — quick live spot-check of a single detector; no token needed on the local subnet

### Authentication
- Token file: `/home/phy/dcsu/lnFill/influx.token` ✅ verified 2026-04-09 — SSH DCS2: file exists, 123 bytes, `export INFLUXDB_WRITE_TOKEN=...`
- Format: `export INFLUXDB_WRITE_TOKEN="apiv3_..."`
- The `dcsu` write token is scoped to **write-only** — cannot list databases or run queries
- A separate read/admin token is needed for queries (held by `adminrt`)
- **Read-only token (spark-ca9f / formerly pi5-dgs):** stored at `~/workspace/secrets/influx3_read.token` on spark-ca9f (DGX Spark — General DGS host as of 2026-04-15). Do NOT expose this token. Parse it with: `grep '^Token:' ~/workspace/secrets/influx3_read.token | awk '{print $2}'`

### Line Protocol Format
```
measurement[,tag1=val1,tag2=val2] field1=val1[,field2=val2] [timestamp_ns]
```
Examples from DGS:
```
# Detector temperature (HPGeTemp db)
Temperature,gsid=005,en=1 value=87.3

# PV snapshot (DGS db)  
VME01:MDIG1:coarse_threshold0 value=500i 1743890400000000000
GS5_GE_HV_DEMAND_VOLTS value=3000.0 1743890400000000000
```

**Type suffixes:** `i` = integer, no suffix = float, `"..."` = string

---

## What Gets Written to InfluxDB

### 1. Detector Temperatures (`HPGeTemp` database)

**Script:** `/home/phy/dcsu/lnFill/SaveTemp.sh` → `templog/StoreDetTemps.py`
**Triggered by:** cron job on DCS2 (runs periodically)
**Data written:**
- `Temperature,gsid=NNN,en=0/1 value=<temp_C>` for each of 110 GS holes ✅ verified 2026-04-09 — `StoreDetTemps.py:L51` (`influx_entry = "Temperature,gsid="+str(i).zfill(3)+",en="...`)
- `pi_Temp value=<temp_C>` — Raspberry Pi board temperature ✅ verified 2026-04-09 — `StoreDetTemps.py:L41`

**PVs read:** `MOD001_DV_TEMP` + `MOD001_DV_EN` … `MOD110_DV_TEMP` + `MOD110_DV_EN` (from collector box softIOC via pyepics). If `DV_TEMP > 520 K`, the detector is not connected — value stored as 0. ✅ verified 2026-04-13 — `StoreDetTemps.py:L95` (`if detTemp[gsid] > 520: detTemp[gsid] = 0 # not connected`)
**Also logs to:** `templog/templog_YYYYMMDD.csv` (CSV backup on DCS2)

### 2. DGS PV Snapshots (`DGS` database)

**Script:** `snapshot_pv/dumpPVs.py` with `WriteInflux()` function
**Status:** Code exists but is **commented out** — currently writes to files only
**When activated:** would push all PV snapshots to Influx at run start

---

## Grafana

**URL:** `http://192.168.203.56:3000`
**Version:** 12.3.1 (Enterprise build) ✅ verified 2026-04-06 — `/api/health` returns `{"version":"12.3.1","database":"ok"}`
**Commit:** `0d1a5b4420a5e4579b91c239ecb240ea2b247fba` (core); `2a9691bd01b56d649d9fd494d0166752f0891473` (enterprise)
**Authentication:** admin credentials required (not accessible from `dcsu` account — 401 on `/api/datasources`)

### Data Sources (inferred)
Grafana connects to InfluxDB 3 at `http://localhost:8181`. Expected datasources:
- `HPGeTemp` database → temperature trend panels
- `DGS` database → PV history panels (when activated)

### Provisioning
Grafana provisioning dirs exist at `/etc/grafana/provisioning/` (access-control, alerting, dashboards, datasources, plugins) but contain **only the default `sample.yaml` placeholders** — no active provisioned datasources or dashboards. ✅ verified 2026-04-08 — SSH to DCS2: `datasources/sample.yaml` only, `dashboards/sample.yaml` only.

**Implication:** All datasources and dashboards are configured manually via the Grafana UI and stored in `/var/lib/grafana/grafana.db` (SQLite). No `/var/lib/grafana/dashboards/` directory exists.

### Dashboards
Dashboards are stored in `grafana.db` (no read access from `dcsu`). Based on what's being written to InfluxDB, expected dashboards include:
- Detector temperature heatmap / trend (110 GS holes from `HPGeTemp` db)
- LN2 fill status and tank levels
- Possibly: PV history browser (when `DGS` db is activated)

> 💡 **Planned:** Custom Gammasphere detector map panel — all 110 GS holes as two polar projections (N+S hemisphere), colored by temperature from `HPGeTemp`. Design notes: `workspace/grafana_gammasphere_panel.md`. Geometry data: `knowledgeBase/gammasphere_geometry.md`.

---

## WriteDiscordMessage.py

Helper used by lnFill scripts. Has two functions:
- `WriteDiscordMessage(msg)` — posts to Discord via webhook
- `WriteInflux(msg)` — writes a single line protocol message to `HPGeTemp` db

Token and webhook loaded from files in the `lnFill/` directory: `discord.WebHook` (webhook URL) and `influx.token` (InfluxDB write token). ✅ verified 2026-04-12 — `WriteDiscordMessage.py:L6,L17`

---

## Adding DGS PV Data to InfluxDB (Future)

The `dumpPVs.py` has `WriteInflux()` ready but commented out. To enable:
1. Create an `influx.token` file in `snapshot_pv/` with a write token for the `DGS` db
2. Uncomment the `WriteInflux()` call at the end of `dumpPVs.py`
3. InfluxDB will receive all PV snapshots in line protocol format
4. Add a Grafana datasource pointing to the `DGS` db
5. Build panels: threshold trends, HV history, trigger rate history, etc.

---

## Quick Reference

```bash
# Source the write token (on DCS2 as dcsu)
source ~/lnFill/influx.token

# Write a test point
echo 'test_measurement value=42.0' | \
  curl -s "http://192.168.203.56:8181/api/v3/write_lp?db=HPGeTemp" \
  --header "Authorization: Bearer $INFLUXDB_WRITE_TOKEN" \
  --data-binary @-

# Check Grafana is alive
curl -s http://192.168.203.56:3000/api/health
```

_Source: DCS2 exploration via SSH (dcsu@DCS2.onenet). Created: 2026-04-05_

## Cross-References

- `knowledgeBase/lnfill.md` — LN2 fill system; writes temperature data to HPGeTemp InfluxDB database
- `knowledgeBase/expMemory_2008_Chiara.md` — Active experiment log; references monitoring dashboards
- `knowledgeBase/nfs_layout.md` — DCS2 filesystem layout; InfluxDB/Grafana run on DCS2 (192.168.203.56)
