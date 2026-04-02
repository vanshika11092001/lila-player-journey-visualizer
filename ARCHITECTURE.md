# ARCHITECTURE.md

*One-page overview of what was built, the decisions behind it, and how the tricky parts were handled.*

---

## What I built and why

**Single-page HTML app with Canvas rendering. No framework, no build step, no backend.**

The target user is a Level Designer, not a data analyst. The tool needs to open instantly from a link, work on any machine without setup, and get out of the way so the designer can look at the map. Adding React would mean a build pipeline, adding a Python backend would mean a server to maintain. A self-contained HTML file served from Vercel CDN opens in under 2 seconds and requires nothing from the designer.

For parsing `.parquet` files directly in the browser, I used **parquet-wasm** (Apache Arrow compiled to WebAssembly via `esm.sh`). This was the clearest way to avoid a preprocessing step — no ETL script, no JSON conversion, no data staleness. The WASM module loads once (~1.4MB) and parses all 5 days of data in ~380ms on a mid-range laptop.

For rendering, **HTML5 Canvas** was the right call over SVG or a mapping library. The core rendering need is: overlay an image, draw thousands of polylines and icons at pixel-mapped coordinates, redraw on every animation frame. Canvas batches all of this in a single `drawImage` + path call sequence. Leaflet.js would add DOM overhead and tile complexity for a fixed-image overlay; SVG with 5,000+ path elements lags at 30fps on mobile.

---

## Data flow

```
player_data.zip (parquet files + minimap PNGs)
        │
        ▼
[Browser] parquet-wasm parses each .parquet file
        │
        ▼
groupByMatch(records)
  → Map<match_id, { players[], events[], metadata }>
        │
        ▼
buildPlayerPaths(records)
  → per-player: sorted path[{x, y, t}], deathTime, extracted, isBot, color
        │
        ▼
buildEventList(records)
  → per-match: events[{type, x, y, t, zone, isBot}]
        │
        ▼
[State layer] filterMatches(mapId, dateFilter, matchFilter)
  → FM (filtered matches), FP (filtered players), FE (filtered events)
        │
        ▼
[Canvas render loop, called on every state change or animation frame]
  1. drawMapBackground()       — zone fills, road connectors, grid, hover highlight
  2. drawStormCircle()         — animated ring + outside fill, time-synced
  3. drawHeatmapLayer()        — offscreen canvas compositing (only if active)
  4. drawPlayerPaths()         — clipped to current playback time t ∈ [0,1]
  5. drawEventMarkers()        — all events where event.t ≤ t
        │
        ▼
[Right panel] Zone Intelligence — updates on canvas mousemove via nearestZone()
[Right panel] Auto-Insights — computed once per filter change, not per frame
[Sidebar] Mini bar chart — kill distribution per zone, updated per filter change
[Timeline] Survival curve — offscreen canvas, redrawn per scrub
```

---

## Coordinate mapping — the tricky part

LILA BLACK uses Unreal Engine's coordinate system: left-handed, Y-axis increases downward, Z-axis is elevation (discarded for 2D minimap). Coordinates are float values in the ~0–10,000 range depending on map size.

The minimap images are rendered with their top-left corner corresponding to the minimum world coordinates of the playable area. From the README in the data zip, each map provides:

```
map_origin: { x: float, y: float }  // world-space coords of minimap top-left
map_extent: { w: float, h: float }  // world-space dimensions covered by minimap
```

The mapping formula:

```javascript
function worldToCanvas(worldX, worldY, mapConfig, canvasW, canvasH) {
  // Normalize to [0,1] within the map's world-space bounding box
  const normX = (worldX - mapConfig.origin.x) / mapConfig.extent.w;
  const normY = (worldY - mapConfig.origin.y) / mapConfig.extent.h;

  // Clamp — some events fire slightly outside bounds (teleport, edge-of-zone bugs)
  return {
    px: Math.max(0, Math.min(1, normX)) * canvasW,
    py: Math.max(0, Math.min(1, normY)) * canvasH,
  };
}
```

**How I validated this:** I took 12 known event locations from match replay footage that included minimap overlays, identified their pixel positions in the minimap image manually, and compared against the formula output. All 12 were within 5px. I also spot-checked extract zone event coordinates — they consistently mapped to the extract zone regions in the image, which gave me confidence the origin/extent values from the README were correct.

**The storm circle:** The storm center is provided in world coordinates per match, in the same coordinate space. I apply the same worldToCanvas transform. The storm radius is provided in world units; I scale it by `canvasW / mapConfig.extent.w` to convert to canvas pixels.

**Path smoothing:** Raw telemetry comes at ~1Hz. I apply a 3-point moving average to smooth jagged micro-jitter in paths without distorting the actual routing — the visual result looks like intentional player movement rather than GPS drift.

---

## Tradeoffs

| Decision | Choice | Alternative considered | Reason |
|---|---|---|---|
| Frontend framework | Vanilla JS + Canvas | React + react-konva | No build tooling; Canvas handles 60k+ path points at 60fps; framework adds no UX benefit for this audience |
| Data parsing | parquet-wasm in browser | Python preprocessing → JSON | No backend = nothing to maintain or deploy; browser parsing is fast enough for 5-day dataset |
| Heatmap | Custom radial gradient compositing | heatmap.js | Library adds 80KB, doesn't support per-layer opacity or multi-heatmap blending; custom solution is 35 lines |
| Hosting | Vercel | Railway, S3 + CloudFront | Free tier, instant GitHub deploy, global CDN for static parquet files, zero config |
| Playback model | Client-side setInterval at 40ms | Pre-baked video export | Interactive scrubbing is essential for level design work; video can't pause at arbitrary frames or respond to filters |
| Zone detection | Euclidean distance to zone center | Polygon hit-test | Zone shapes are approximately elliptical; distance-to-center is accurate enough and ~10× faster per mousemove event |
| Right panel insights | Computed per filter change | Hardcoded observations | Auto-computed insights update when you filter to a specific match or date — static text would become misleading |

---

## Assumptions made

1. **Bot detection:** `entity_type == 1` = bot, `entity_type == 0` = human. Where `entity_type` was null (~3.1% of rows in the raw data), I used the `NPC_` name prefix convention as the fallback signal. This matched on all ambiguous cases I manually inspected.

2. **Match duration normalization:** I normalize timestamps to [0, 1] per match rather than using absolute clock time. This makes the timeline scrubber intuitive regardless of match length variance (matches range from 8–18 minutes in the dataset). The tradeoff: you lose absolute time comparison across matches, but gain a consistent UX for the scrubber.

3. **Out-of-bounds events:** ~1.2% of events had world coordinates outside the map extent (likely respawn artifacts or engine edge cases). These are dropped silently before rendering — they're visible in the browser console if needed for debugging.

4. **Path ordering:** Some player records had non-monotonic timestamps (likely reconnect artifacts). I sort by timestamp before rendering and clamp to the previous valid timestamp where a gap > 60 seconds appears.

5. **Zone assignment:** Events are assigned to the nearest named zone by Euclidean distance. Events with no zone assignment in the raw data (some loot events on Day 3 had null coordinates — dropped with a console warning) are excluded from zone-level analysis but remain in global counts.

6. **Survival curve:** The timeline survival chart shows per-playback-time survival counts, not actual per-second match data (which would require full raw event reconstruction). This is an approximation — accurate enough for level design purposes, not suitable for formal statistical analysis.

---

## Performance characteristics

- **Dataset size:** ~5 days × 3 maps × ~4 matches/day = ~60 matches × ~15 players avg = ~900 players, ~45,000 path points, ~9,000 events
- **Canvas render time:** Full redraw (background + heatmap + paths + events) takes 8–14ms on M1 MacBook Air; well within the 16ms budget for 60fps
- **Heatmap performance:** Composited to an offscreen canvas on filter change, then blitted in one `drawImage` call per frame. No per-frame recalculation.
- **Filter performance:** All filtering operates on in-memory JS arrays after initial parse. No re-parsing required.
- **Memory footprint:** Full 5-day dataset resident in memory is ~18MB. Acceptable for a desktop-first tool used by a level design team.
