# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an interactive hiking map for the Garibaldi della Lessinia trail in the Lessinia Mountains (Veneto region, Italy). It's a single-page HTML application that visualizes a 28 km hiking route with elevation, timing, and waypoint information.

The application uses **Leaflet.js** for map rendering with **OpenTopoMap** tiles as the base layer. All code is self-contained in a single `index.html` file.

## Running the Application

Simply open `index.html` in a web browser. No build step, server, or dependencies are required beyond the CDN-hosted Leaflet library.

## Project Structure

- **index.html** — The entire application:
  - **\<style\>** (lines 9–108) — CSS for header, map container, markers, and popup styling
  - **\<script\>** (lines 125–238) — JavaScript:
    - `stops` array — defines waypoints with coordinates, elevation, timing, and description
    - Map initialization and layer setup
    - Route line (polyline) rendering
    - Marker and popup creation for each stop

## Key Customization Points

### Editing the Trail Route
The `stops` array defines each waypoint. Each stop object has:
- `name` — waypoint name/description
- `lat`, `lng` — coordinates (WGS84)
- `quota` — elevation in meters (display only)
- `time` — time stamp or timing info
- `notes` — detailed description shown in popup
- `peak` — boolean; `true` colors the marker red for summits/endpoints

To modify the trail: update the `stops` array, adjust header stats (distance, elevation, time), and the initial map center in `map.setView([45.55, 10.88], 12)`.

### Styling
The color scheme uses greens for navigation elements and earth tones for accents:
- Primary green: `#2d5738` (dark)
- Accent gold: `#e8b34f`
- Peak/endpoint red: `#c0392b`
- Background: `#1a2f23`

Route line uses a dashed red style. Adjust these hex values in the `<style>` section for rebranding.

### Map Tiles
The base map layer is OpenTopoMap (lines 201–204). To switch tile providers, replace the URL pattern and attribution.

## Data Format

All route data is hardcoded in the JavaScript `stops` array. For future iterations, consider exporting this to a JSON file or external data source if:
- Multiple trails are needed
- Real-time GPS tracking is added
- Elevation profiles become dynamic

## Browser Compatibility

Uses modern CSS (flexbox, gradients) and Leaflet v1.9.4. Works on modern browsers and mobile (responsive design handles screens ≤600px with adjusted layout).
