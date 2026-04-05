# Gammasphere Detector Map — Custom Grafana Panel

_Feature idea from Ryan, 2026-04-05_

---

## Concept

A custom Grafana panel showing the full Gammasphere array as **two flat polar projections** (north + south hemisphere), where each of the 110 detector holes is rendered as a labeled circle at its correct geometric position, colored by temperature when data is available.

---

## Visual Layout

```
[ North Hemisphere ]        [ South Hemisphere ]
   (θ < 90°)                   (θ > 90°)

     Ring 1 (17.3°)               Ring 10 (119.9°)
        ○                               ○ ○
      ○ ○ ○                           ○ ○ ○ ○
    ○ ○ ● ○ ○           ...        ○ ○ ● ○ ○ ○
      ○ ○ ○                           ○ ○ ○ ○
        ○                               ○ ○

  GS label inside each circle
  Color = temperature (colorscale)
  Gray = no data / hole unused
```

---

## Data Source

- **InfluxDB 3** on DCS2 (`http://192.168.203.56:8181`)
- **Database:** `HPGeTemp`
- **Measurement:** `Temperature`
- **Tags:** `gsid` (zero-padded 3-digit, e.g. `005`), `en` (enabled flag)
- **Query example (SQL):**
  ```sql
  SELECT gsid, value 
  FROM Temperature 
  WHERE time > now() - INTERVAL '10 minutes'
  ORDER BY time DESC
  ```
- If no recent data for a hole → render as blank/gray

---

## Geometry

Source: `gammasphere_geometry.md` — 110 holes, 17 rings, θ angles defined per ring.

**Polar projection formula:**
- For each hole: `r = θ / 90°` (normalized radius; 0 = pole, 1 = equator)
- `φ` = azimuthal angle from geometry file
- `x = r × cos(φ)`, `y = r × sin(φ)`
- North: θ from 17.3° to ~80°; South: θ from ~100° to 162.7°
- South projection: mirror or rotate so both hemispheres display left-to-right

**GS hole → ring assignment:** See `gammasphere_geometry.md` for the full table.

---

## Implementation Options

### Option A: Grafana Canvas Panel (no plugin, quickest)
- Grafana's built-in Canvas panel supports custom element placement
- Place circle elements at computed (x, y) positions
- Color each circle by temperature value via field override
- **Limitation:** Canvas positions are static JSON — hard to compute 110 positions dynamically

### Option B: Custom Grafana Plugin (recommended)
- Write a React-based panel plugin
- Render SVG: two `<svg>` elements (north + south), each with 55-ish `<circle>` elements
- Position circles using precomputed (x, y) from geometry
- Color by temperature using a D3 colorscale (blue → red, or custom)
- Tooltip on hover: GS number, temperature, enabled status
- **Stack:** Node.js + `@grafana/create-plugin`, TypeScript/React, D3 for colorscale
- Plugin installed to Grafana's plugin directory (`/var/lib/grafana/plugins/`)

### Option C: Standalone Web App (simpler to develop, separate from Grafana)
- Pure HTML/JS/SVG page served on DCS2
- Queries InfluxDB REST API directly
- Auto-refreshes every N seconds
- Embeddable in Grafana via iframe panel

---

## Color Scale (suggestion)

| Temperature | Color |
|-------------|-------|
| < 80 K (cold, good) | Blue |
| 80–100 K (normal) | Green |
| 100–120 K (warm) | Yellow |
| > 120 K (hot, alarm) | Red |
| No data | Gray |

---

## TODO (see MEMORY.md)

1. Get the full (GS_hole → θ, φ) table from `gammasphere_geometry.md`
2. Compute (x, y) for all 110 holes in both projections
3. Decide: Canvas panel vs custom plugin vs standalone page
4. Implement and test with live InfluxDB data
5. Deploy to Grafana on DCS2

---

_Status: Planning / Not started_
_Created: 2026-04-05_
