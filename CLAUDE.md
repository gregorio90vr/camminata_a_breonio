# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Interactive hiking map for the **Garibaldi della Lessinia** trail (Strada Bresciana, Verona → Breonio, via Valpolicella) in the Lessinia mountains, Veneto. Single-page app built on **Leaflet.js**, with a real trail-following route, an elevation profile, and a linked map/chart selection.

UI text is in **Italian** — keep new user-facing strings in Italian.

## Running

Open `index.html` in a browser. No build, no server, no dependencies beyond Leaflet from CDN.

The app **works fully offline** (aside from map tiles). This is deliberate: it is meant to be usable on the mountain. Do not introduce a runtime dependency on a routing or elevation API.

When verifying a route change, **hard-reload** (Ctrl+F5) — a cached `index.html` will show the old track and looks exactly like a routing bug.

## Architecture

Everything lives in `index.html`, in three `<script>` blocks that **must stay in this order**:

1. **`TRACK`** — the generated route: `[lon, lat, quota_m]` × 1432. Marked as generated; do not hand-edit.
2. **`stops`** — the 11 waypoints: name, coordinates, time, notes, `kind`. Purely descriptive metadata.
3. **Main logic** — everything else.

### The key data relationship

`stops` holds *metadata only*. Every number shown in the UI is **derived from `TRACK` at load time** — nothing is hardcoded:

- `cumDist[]` — cumulative distance (haversine)
- `eleSmooth[]` — elevation smoothed over a ±4 moving average, used **only** for ascent/descent. Raw SRTM elevation oscillates and inflates the total; smoothing brings it in line with BRouter's own filtered figure.
- `cumAsc[]` / `cumDesc[]` — cumulative positive and negative elevation, computed on the smoothed series
- `stopIndex[]` — for each waypoint, the nearest point on `TRACK`; this is what splits the route into clickable legs

Displayed elevations and the chart use **raw** `TRACK[i][2]`, not the smoothed series — smoothing would flatten the summits.

If you add or move a waypoint you **must regenerate `TRACK`**, otherwise the route will not pass through it.

### Selection model

A single `sel = { mode, a, b }` object is the source of truth, shared by map, chart and readout. `mode` is `'none' | 'point' | 'range'`; `setNone()` / `setPoint(i)` / `setRange(i,j)` are the only writers, each calling `render()` (rAF-throttled) → `draw()` → `drawMap()` + `drawCursor()`.

Gestures, all funnelling into those three setters:

| Gesto | Effetto |
|---|---|
| Click su tratta o marker | punto |
| Click sul grafico | 1° click fissa l'ancora, 2° chiude l'intervallo |
| Trascinamento sul grafico | intervallo in un gesto (soglia `DRAG_PX`) |
| Passaggio del mouse | anteprima; con l'ancora attiva mostra l'intervallo |
| ✕ o click sullo sfondo | azzera |

`anchor` holds the pinned first point and is what distinguishes a committed selection from a hover preview — hover must never overwrite it. The five readout tiles are relabelled per mode by `setRo()`; they are generic (`ro1`…`ro5`), so don't reintroduce semantic ids.

**Leaflet event bubbling**: the hit polylines and markers set `bubblingMouseEvents: false`, and the decorative lines set `interactive: false`. Without this, a click on the route bubbles up to the map, whose handler calls `resetSel()` and instantly wipes the selection just made. Keep these flags if you add layers.

### Elevation chart

Hand-built SVG (no charting library — `leaflet-elevation` would pull in d3 for something a few hundred lines does). Redrawn on resize and on panel expand; `plot` holds the current scales `X`/`Y` plus the mutable elements. Both the "already walked" portion and the selected range are revealed by setting the width of a `<clipPath>` rect rather than rebuilding paths.

## Regenerating the route

Routing uses **BRouter** (`hiking-beta` profile) — public, no API key, and it returns elevation inside the coordinates, so no separate elevation API is needed.

```bash
LL="10.899294,45.446218|10.847057,45.474841|10.867012,45.523208|10.849613,45.534736|10.8562451,45.5459113|10.8710519,45.5698093|10.8664438,45.5844161|10.883385,45.597134|10.8906472,45.6157991|10.8977744,45.6248394|10.903341,45.623874"

curl -s "https://brouter.de/brouter?lonlats=${LL}&profile=hiking-beta&alternativeidx=0&format=geojson" -o br.json

# → array JS compatto, 6 punti per riga
grep -oE '\[10\.[0-9]+, 45\.[0-9]+, [0-9.]+\]' br.json | tr -d '[]' \
 | awk -F', ' '{printf "[%.5f,%.5f,%d],", $1, $2, ($3+0.5); if (NR%6==0) printf "\n"} END{printf "\n"}' \
 | sed '$ s/,$//'
```

`lonlats` is `lon,lat` — the reverse of the `lat, lng` used in `stops`.

**The `$` in `sed '$ s/,$//'` is load-bearing**: it strips the trailing comma from the last line only. Without it every line loses its comma and the array silently becomes `][` — a syntax error.

`TOTAL_TIME` (28741 s) is BRouter's `total-time` and is **hardcoded** — update it when regenerating. Remaining time is apportioned by `effort = distance + 8 × ascent`, which keeps steep sections from reading as fast as flat ones. The same model generates the per-waypoint `time` values (plus a 45 min lunch stop from Monte Pastello onward).

Rounding to 5 decimals costs well under 0.1% of total distance and saves ~40% of the file size.

## Route data (current)

| | |
|---|---|
| Distanza | 34,2 km |
| Dislivello + | 1.380 m |
| Discesa − | 625 m |
| Tempo | ~7h59 |
| Quota | 79 → 1.119 m (Monte Pastello) |
| Tappe | 11 |

Sanity check that must always hold: `salita − discesa = quota finale − quota iniziale` (1.380 − 625 = 755 = 849 − 95).

### Waypoint coordinates are authoritative

All summit and village coordinates come from **OSM (Nominatim + Overpass)**, not estimates. This matters: the original hand-entered values were given to 3 decimals and were badly wrong — Cavalo was **1,7 km** off, so the route bypassed the village entirely, and Monte Pastelletto was ~700 m off, resolving to 791 m instead of the real 1.031 m summit. "Monte Solane" turned out not to exist as a peak at all; the real landmark is the *Santuario di Monte Solane*.

Before adding a waypoint by hand, look it up:

```bash
curl -s -H "User-Agent: camminata-breonio/1.0" \
  "https://nominatim.openstreetmap.org/search?q=<luogo>&format=json&limit=3"
```

For summits, Overpass with `node["natural"="peak"](bbox)` also returns `ele`.

After changing waypoints, verify each one resolves close to the track and that indices stay **monotonic** — a waypoint whose nearest track point comes before the previous one produces an empty, silently missing leg.

## Map layers

OpenTopoMap (default), Esri World Imagery, OSM — all keyless.
