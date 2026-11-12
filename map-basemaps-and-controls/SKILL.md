---
name: map-basemaps-and-controls
description: Use when building or changing a web map application. Covers choosing basemap tile layers and attribution, adding map controls (zoom, minimap, scale, coordinate readout, geolocate, place search, right-click menu), and keeping the map fast with large datasets (thousands of markers, long paths, many labels). Library-agnostic, with Leaflet examples.
---

# Map basemaps, controls and large datasets

Use this skill for any browser map app. Examples use Leaflet; the rules apply equally to MapLibre, OpenLayers and similar libraries.

## 1. Ground rules

- Default projection is Web Mercator (EPSG:3857). Only use tile services that publish in it, or reproject deliberately.
- Every tile request tells the tile provider which area the user is viewing. Treat that as a disclosure (see the `local-data-storage-and-privacy` skill). Never put user data in tile URLs.
- Every basemap needs correct attribution, visible on the map at all times.
- Verify tile URLs before relying on them. Services move, change zoom limits or add keys. The app must detect failing tiles and tell the user (section 2.6).
- Keep all basemap definitions in one data table (key, name, group, icon, URL, attribution, zoom limits). The UI is generated from the table, so adding a layer is a one-line change.

## 2. Basemaps

### 2.1 Definition shape

```js
const PROVIDERS = {
  osm: { name: 'OpenStreetMap', group: 'Standard maps', icon: 'fa-map',
         url: 'https://tile.openstreetmap.org/{z}/{x}/{y}.png',
         attribution: '© OpenStreetMap contributors', maxZoom: 19 },
  // maxNativeZoom: highest zoom the server really has. Above it the library stretches tiles (overzoom).
  // subdomains: only for URLs containing {s}.
  // wms: { layers, version, format, transparent } for WMS sources.
};
```

Always set `maxNativeZoom` when a service stops before the map's `maxZoom`; otherwise the user sees grey tiles when zooming in. Include a "No base layer" entry (blank background) for overlay-only work.

### 2.2 Catalogue of commonly used sources

Tile order differs: standard XYZ is `{z}/{x}/{y}`; ArcGIS REST services are `{z}/{y}/{x}`. Getting this wrong gives scrambled or empty tiles.

| Group | Name | URL template | Max zoom | Notes |
|---|---|---|---|---|
| Standard | OpenStreetMap | `https://tile.openstreetmap.org/{z}/{x}/{y}.png` | 19 | Community service. Light use only, no bulk download or prefetch, must identify the app via Referer. Use a commercial provider for heavy use. |
| Standard | OSM Humanitarian | `https://{s}.tile.openstreetmap.fr/hot/{z}/{x}/{y}.png` (a,b,c) | 19 | Same fair-use rules. |
| Standard | Esri World Street Map | `https://server.arcgisonline.com/ArcGIS/rest/services/World_Street_Map/MapServer/tile/{z}/{y}/{x}` | 19 | Attribution "© Esri". |
| Satellite | Esri World Imagery | `https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}` | 19 | Good default satellite layer. Detail varies by region. |
| Satellite | USGS Imagery | `https://basemap.nationalmap.gov/arcgis/rest/services/USGSImageryOnly/MapServer/tile/{z}/{y}/{x}` | 19 | US coverage only. |
| Satellite | NASA Blue Marble | `https://map1.vis.earthdata.nasa.gov/wmts-webmerc/BlueMarble_ShadedRelief_Bathymetry/default/GoogleMapsCompatible_Level8/{z}/{y}/{x}.jpg` | 8 (native 8) | Global overview only. |
| Satellite | NASA night lights | same host, layer `VIIRS_CityLights_2012` | 8 (native 8) | Overview only. |
| Terrain | OpenTopoMap | `https://{s}.tile.opentopomap.org/{z}/{x}/{y}.png` (a,b,c) | 17 | Contours and relief. Needs its attribution text. |
| Terrain | Esri World Topo | `https://server.arcgisonline.com/ArcGIS/rest/services/World_Topo_Map/MapServer/tile/{z}/{y}/{x}` | 19 | |
| Terrain | USGS Topo | `https://basemap.nationalmap.gov/arcgis/rest/services/USGSTopo/MapServer/tile/{z}/{y}/{x}` | 19 | US only. |
| Terrain | CyclOSM | `https://{s}.tile-cyclosm.openstreetmap.fr/cyclosm/{z}/{x}/{y}.png` (a,b,c) | 20 | Cycling-oriented. |
| Minimal | Esri Light Gray | `https://server.arcgisonline.com/ArcGIS/rest/services/Canvas/World_Light_Gray_Base/MapServer/tile/{z}/{y}/{x}` | 16 | Best for minimaps and data-heavy overlays. |
| Minimal | Esri Dark Gray | `https://server.arcgisonline.com/ArcGIS/rest/services/Canvas/World_Dark_Gray_Base/MapServer/tile/{z}/{y}/{x}` | 16 | Pair with dark theme. |
| Minimal | Carto Positron / Dark Matter | `https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png` (`dark_all` for dark) | 20 | Check current terms before commercial use. |
| Artistic | Esri National Geographic | `https://server.arcgisonline.com/ArcGIS/rest/services/NatGeo_World_Map/MapServer/tile/{z}/{y}/{x}` | 16 | |

Keyed providers (MapTiler, Stadia, Mapbox, Thunderforest and similar) put an API key in the URL. Keep the key in one config constant, restrict it to the app's domain in the provider dashboard, and remember it is not secret once shipped to a browser.

### 2.3 Regional and government sources

National and state mapping agencies often publish excellent aerial, topographic and hillshade tiles, commonly as ArcGIS REST caches:

```
https://<host>/arcgis/rest/services/<Folder>/<Service>/MapServer/tile/{z}/{y}/{x}
```

Rules for adding one:

1. Open `<service URL>?f=json` and read `tileInfo`. The spatial reference must be 102100 or 3857, otherwise it cannot be used as plain tiles.
2. Note the highest level in `tileInfo.lods`. Set `maxNativeZoom` to it.
3. Read the licence. Many are attribution-required and non-commercial or no-derivatives. Put the exact required attribution string in the definition, and tell the user about any licence restriction.
4. Dated aerial series (one service per survey year) work well as a group, newest first, so users can compare years.
5. Group regional layers under the agency's name so they are easy to find.

### 2.4 WMS sources

For sources that only offer WMS (hydrographic charts, geology, cadastre), use the library's WMS tile layer in the same Mercator projection:

```js
L.tileLayer.wms(url, { layers: 'LayerName', version: '1.3.0', format: 'image/png',
                       transparent: true, attribution, maxZoom: 19 });
```

### 2.5 Layer picker UI

- Group by `group`, with a small heading per group, a radio-style row per layer and a check mark on the active one.
- Remember the last choice in localStorage (a small preference, not user data) and restore it on load. Fall back to a sensible default (satellite or standard street) if the saved key no longer exists.
- Show a toast "Map: <name>" on switch.
- Keep the user's own overlays above the basemap: after switching, bring the data layers to front.
- Optional: put terrain heatmaps or other raster overlays in their own pane with a z-index between the basemap (200) and vectors (400+), non-interactive.

### 2.6 Failure detection

Count `tileload` and `tileerror` events per layer. If, say, four errors occur and no tile has ever loaded, show one toast: "<name>: no tiles came back. Check your connection or zoom level." Mark layers whose URL could not be verified as `unverified` and mention that in the toast. Never fail silently with a blank map.

### 2.7 Caching

Rely on the browser HTTP cache. Do not prefetch or bulk-download tiles unless the provider's terms explicitly allow offline use. If an in-memory cache is added, cap it (LRU, about 1,000 tiles) and revoke object URLs on eviction.

## 3. Controls

Layout positions are in the `app-shell-layout` skill. This section defines behaviour.

### 3.1 Zoom buttons

Custom buttons, bottom-right above the status bar, not the library default. Slide left when a docked properties panel opens so they are never covered. Keep keyboard `+` and `-`. Disable double-click zoom if double-click is used to finish drawing.

### 3.2 Minimap (overview map)

- A second, non-interactive map (no drag, wheel, double-click, keyboard or touch zoom), about 160 x 120 px, bottom-left above the status bar.
- Basemap: the light gray minimal layer, max zoom 16.
- Zoom: main zoom minus 4, minimum 2. Re-centre it when the main view leaves its bounds.
- A rectangle shows the main viewport. Update it on `move` (live) and re-sync on `moveend`.
- Click on the minimap recentres the main map, keeping the current zoom.
- Header strip shows a ground-scale label (width of the minimap in km or m, rounded to 1/2/5 x 10^n) and a close button.
- **Closing must be reversible.** Provide at least two ways to bring it back: a status-bar toggle button and a command-palette entry. Persist the on/off preference. Create the minimap lazily the first time it is turned on.
- Hide it below about 900 px viewport width.
- **Do not cluster points on the minimap.** Draw plain small circle markers and cap the count (about 300). If there are more, thin by repeatedly removing one point of the currently closest pair (section 4.6). Show "Showing N of M" in a tooltip. Recompute only when the data changes (debounced about 150 ms), not on every pan.
- Only include data that is currently in the visible dataset: exclude hidden, archived, deleted and session-only items.

### 3.3 Scale and zoom readout

Show `z12 · 1:35,000` in the status bar. Metres per pixel = `156543.03392 * cos(lat) / 2^zoom`; ratio = mpp * 3779.5276 (96 dpi). Optionally add the library's metric scale bar.

### 3.4 Coordinate readout

- Status bar left: latitude, longitude to 5 decimals under the pointer; falls back to map centre (prefixed "Centre") when the pointer leaves the map.
- Click copies; the `C` key copies the pointer position.
- Right-click menu on the map: Copy coordinates, Copy as DMS, Copy as lng, lat, Copy elevation (if terrain is loaded), Add marker here.
- Update through one `requestAnimationFrame`-coalesced paint, never on every mousemove.

```js
function toDMS(v, pos, neg) { const h = v >= 0 ? pos : neg; v = Math.abs(v);
  let d = Math.floor(v), m = Math.floor((v - d) * 60), s = Math.round(((v - d) * 60 - m) * 6000) / 100;
  if (s >= 60) { s = 0; m++; } if (m >= 60) { m = 0; d++; }
  return `${d}°${String(m).padStart(2,'0')}′${s.toFixed(2).padStart(5,'0')}″${h}`; }
```

### 3.5 Geolocate

One button in the header. Call `getCurrentPosition` once per click (never `watchPosition` unless asked). Show an accuracy circle and a popup with the accuracy. Handle denial and insecure-context errors with a toast. Never store or transmit the position. Requires HTTPS or localhost.

### 3.6 Place and coordinate search

- The search box is an icon that expands in the header. `/` focuses it. `Escape` collapses it.
- **Parse coordinates first, locally**: decimal pairs (`-42.88, 147.33`), hemisphere letters (`42.88S 147.33E`), DMS (`42°53′10″S 147°19′30″E`). If only the swapped order is valid, accept it and tell the user. Anything containing words is a place name.
- Place names need a geocoder, which sends the typed text to a third party. Disclose this next to the box, and make the feature switchable off for sensitive apps.
- Nominatim policy: at most 1 request per second, cache results, identify the app, show "© OpenStreetMap contributors", and do not use it for search-as-you-type. Use a separate suggestion service (for example Photon, or a self-hosted instance) for as-you-type suggestions with a 300 ms debounce, 3-character minimum, and abort of in-flight requests. Enter runs the full search.
- Results: combobox with listbox semantics, arrow-key navigation, Enter to choose, a footer with attribution. Choosing flies to the result bounds, drops a temporary marker with a popup (Copy, Add marker).

### 3.6a Other controls worth including

Drawing tool column (icon-only with tooltips), a status banner while drawing ("Click points · Finish or double-click to end" with Finish/Cancel), live length or area badge near the cursor, keyboard `Enter` to finish, `Esc` to cancel, and optional snap-to-marker with a visible snap ring.

### 3.7 Map resize

Leaflet does not notice container resizes caused by panels. Attach a `ResizeObserver` to the map element and call `invalidateSize({pan:false})` inside a `requestAnimationFrame`.

## 4. Large datasets

Design for 10,000+ markers and 100,000+ path vertices from the start. Measure before and after each change.

### 4.1 Know the cost

The costs, from most to least expensive per item: DOM markers, bound popups and handlers, SVG path elements, text labels, SVG path vertices, canvas vertices. Track and display a "complexity score" (a temporary, heuristic number): markers in the DOM + labels + SVG paths x 3 + drawn vertices / 20. Rough comfort levels on a laptop: under 1,500 fine; 1,500 to 5,000 noticeable; above 5,000 laggy. Validate on the target machine.

### 4.2 Techniques, in order of payoff

1. **Cull markers to the viewport.** Keep markers in a data structure, but only put those inside the padded view bounds (pad about 0.5) on the map. Update on `moveend` and throttled `move` (about 180 ms). Markers being edited or dragged are pinned and never culled.
2. **Canvas rendering for vectors.** Use a canvas renderer for paths and polygons instead of one SVG element each. It must be chosen when the map is created, so expose it as a setting that reloads the page.
3. **Simplify when zoomed out.** Skip vertices closer than a pixel tolerance (Leaflet `smoothFactor`, default 1, try 3). Data is never altered, only drawing.
4. **Optional clustering.** Make it a setting, off by default. Load the clustering library only when first enabled. Route only plain markers to it, and pin edited markers out of it. Expose cluster radius (default 60 px) and "stop clustering at zoom" (default 16).
5. **Labels.** Build labels only for items in view, reuse existing label elements (cache by item id plus a signature of style and text), coalesce refreshes to one per animation frame, and cap the count (about 600, nearest the centre win). Tell the user once when capped. Support a min and max zoom range for labels.
6. **Vertex handles when editing.** For long paths (say at least 200 vertices) only draw handles for vertices inside the view and none at all when more than a limit (about 400) would show; toast "Zoom in to edit vertices". Window the vertex list with fixed row heights.
7. **Virtualised lists.** The side-panel list renders only the visible window of rows (fixed row height plus top/bottom spacers) once a folder exceeds about 200 items.
8. **Lazy popups and handlers.** Bind popups as functions that build content on open. Never create DOM for hidden UI up front.
9. **Batch changes.** For bulk show/hide/delete/move/import: suspend list re-rendering, apply all changes, refresh once, save once. Show a busy overlay with a label when the job involves more than about 150 items. Let the browser paint the spinner before starting heavy work (`requestAnimationFrame` then `setTimeout 0`).
10. **Chunk imports.** Parse files in chunks that yield to the event loop, show a progress bar, and consider a Web Worker for very large files.
11. **Debounce persistence** (about 500 ms) and avoid serialising the whole dataset on every small change (see the `undo-history` skill).
12. **Beyond about 50,000 points**, switch to canvas circle markers, GeoJSON tiling, or a WebGL layer (MapLibre, deck.gl) rather than tuning DOM markers.

### 4.3 Settings group

Add a "Large datasets" settings section with: canvas rendering (reload), simplify when zoomed out plus tolerance, vertex handles only in view plus thresholds, only draw markers in view, max labels, clustering plus radius and stop zoom. Persist in localStorage. Each toggle needs a one-line plain-language explanation.

### 4.4 Diagnostics (temporary, behind a status-bar button)

- Show the complexity score in the status bar, coloured green/amber/red.
- Clicking copies a plain-text report: counts by type, markers/paths/labels/DOM size on screen, vertex totals, setting values, timings of the last render, label pass, layer rebuild and save (wrap those functions with a timer), frame times during the last pan (sample `requestAnimationFrame` deltas between `movestart` and `moveend`: average, p90, worst), history size, heap use if available, browser and device pixel ratio.
- **For sensitive data the report must contain counts and timings only.** Leave out names, descriptions, file names and coordinates. Make map centre/bounds an explicit opt-in.

### 4.5 Stress-test harness (temporary)

A dialog that generates synthetic data into one clearly named folder (for example "STRESS TEST (temporary)"): N markers, M paths x V vertices, P polygons x Q vertices, spread over a region or the current view. Requirements:

- The folder is flagged and excluded from persistence, undo snapshots and exports, and disappears on reload.
- A benchmark closes the dialog (a blurred backdrop skews frame time), then at zoom levels 7, 9, 11, 13 and 15 pans 12 times, timing each step including two animation frames, and reports average, p90 and worst plus DOM marker and SVG path counts. 33 ms per step is the 60 Hz floor.
- "Delete test data" removes everything it made. "Copy results" adds the diagnostics report.

### 4.6 Thinning to a point cap (closest-pair removal)

Goal: reduce N points to at most K, removing one point of the closest pair each time, without O(N^2) work.

1. Project to a flat plane: x = lng x cos(mean lat), y = lat.
2. Put every point in a uniform grid with cell size about sqrt(area / N) x 1.5, with a floor of span/2000.
3. For each point find its nearest neighbour by scanning grid rings outward until the best distance is no larger than (ring x cell). Record `nn[i]` and `dist[i]`. Push `(dist, i)` into a min-heap and keep a reverse list `rev[j]` of points whose nearest is j.
4. Repeat while more than K remain: pop the smallest entry; skip it if the point is dead or its distance is stale; mark the point dead; for every k in `rev[i]` whose nearest was i and is alive, find a new nearest and push it.
5. Return the survivors.

Each removal drops one member of the globally closest pair, so the result stays evenly spread. It runs in roughly O(N log N) and handles hundreds of thousands of points.

## 5. Decisions to ask the user

- Which basemaps, and is any regional agency layer needed (and what is its licence)?
- Is any network search allowed, or must the app be fully offline apart from tiles?
- Expected maximum counts of markers, paths and vertices?
- Is clustering acceptable, or must every point stay individually visible?

## 6. Acceptance checklist

- [ ] Every basemap shows attribution; ArcGIS layers use `{z}/{y}/{x}`.
- [ ] Zooming past a service's native zoom overzooms instead of going blank.
- [ ] A dead tile URL produces a visible message.
- [ ] Minimap can be closed and restored, never clusters, never exceeds its point cap, and ignores hidden or archived items.
- [ ] Panel resizes never leave the map with a wrong size.
- [ ] 20,000 markers and 100 paths of 200 vertices pan smoothly with culling on.
- [ ] Diagnostics report contains no user content by default.
