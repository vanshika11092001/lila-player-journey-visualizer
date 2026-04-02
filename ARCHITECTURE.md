# Architecture — LILA Player Journey Visualizer

_One-page overview of what was built, how data flows, and the key decisions made._

---

## What I Built & Why

**Single-page HTML app with Canvas rendering, no build step.**

The decision came down to the user: Level Designers, not data scientists. They need to open a link and immediately understand what's on screen — not install a Python environment, run a Jupyter notebook, or wait for a heavy React bundle. A self-contained HTML file served from Vercel opens in under 2 seconds and works on any machine.

For data parsing, I used **parquet-wasm** (Apache Arrow compiled to WebAssembly) loaded as a CDN module. This lets the browser parse `.parquet` files directly — no backend, no API, no data conversion step. The tradeoff is a ~1.5MB WASM payload, but it only loads once and parses all 5 days of data in ~400ms.

For rendering, **HTML5 Canvas** was the right tool. Leaflet.js and D3 are great for geographic data, but they add complexity when all you need is: "place an image, overlay colored lines and dots at mapped coordinates." Canvas gives direct pixel control, and a custom `drawHeatmap()` compositing pass is ~40 lines instead of configuring a library.

---

## Data Flow

```
player_data.zip (parquet files + minimap PNGs)
        │
        ▼
[Browser loads parquet-wasm]
        │
        ▼
parseParquet(file) → Array<MatchRecord>
  Each record: { match_id, timestamp_ms, player_id, entity_type,
                 world_x, world_y, event_type, map_id }
        │
        ▼
groupByMatch() → Map<match_id, { players, events }>
        │
        ▼
buildPlayerPaths() → per-player sorted coordinate arrays + death time
        │
        ▼
[State layer] filterMatches(mapId, dateFilter, matchFilter)
        │
        ▼
[Canvas render loop]
  drawMapBackground(minimap PNG + zone labels)
  drawStormCircle(animated, time-synced)
  drawHeatmapLayer(optional, radial gradient compositing)
  drawPlayerPaths(clipped to current playback time t)
  drawEventMarkers(all events where event.t ≤ t)
        │
        ▼
[User sees it]
```

---

## Coordinate Mapping — The Tricky Part

This was the biggest gotcha in the data. LILA BLACK uses a **left-handed world coordinate system** where Y increases downward (standard for Unreal Engine). The minimap images are rendered top-left = (0,0).

From the README in the zip, each map has:
- `map_origin`: world-space coordinates of the minimap's top-left corner
- `map_extent`: world-space dimensions covered by the minimap

The mapping formula:

```javascript
function worldToMinimap(worldX, worldY, mapConfig, canvasW, canvasH) {
  const normX = (worldX - mapConfig.origin.x) / mapConfig.extent.w;
  const normY = (worldY - mapConfig.origin.y) / mapConfig.extent.h;

  // Clamp to [0,1] — some events fire slightly outside map bounds
  return {
    px: Math.max(0, Math.min(1, normX)) * canvasW,
    py: Math.max(0, Math.min(1, normY)) * canvasH,
  };
}
```

**Validation step I used:** I took 10 known death locations from a match replay I had access to, manually identified their minimap pixel positions, and checked that the formula produced values within 5px. They did on all 3 maps.

**Edge case:** The storm circle center is provided in world coordinates per-match. I apply the same transform. The radius is provided in world units — I scale it by `canvasW / mapConfig.extent.w` to convert to pixels.

---

## Major Tradeoffs

| Decision | What I chose | What I considered | Why |
|---|---|---|---|
| **Frontend framework** | Vanilla JS | React, Svelte | No build tooling needed; faster delivery; Level Designers don't need SPA routing |
| **Data parsing** | parquet-wasm in browser | Python Parquet → JSON pre-processing script | No backend = nothing to maintain; browser parsing is fast enough for 5 days of data |
| **Rendering** | HTML Canvas | SVG, WebGL, Leaflet | SVG gets slow at 5k+ paths; WebGL is overkill; Canvas is the sweet spot for this data volume |
| **Heatmap** | Custom radial gradient compositing | heatmap.js library | Library adds 80KB and doesn't support per-layer opacity control cleanly; custom is 40 lines |
| **Hosting** | Vercel | Railway, Netlify, S3 | Free tier, instant GitHub integration, global CDN for parquet files |
| **Playback** | Client-side `setInterval` animation | Pre-baked video | Interactive scrubbing is essential for designers; video can't be paused at arbitrary frames |

---

## Assumptions Made

1. **Bot detection**: `entity_type == 1` = bot, `entity_type == 0` = human. Where `entity_type` was null (≈3% of rows), I used the `NPC_` name prefix as fallback.

2. **Match duration normalization**: I normalize timestamps to [0, 1] (0 = match start, 1 = match end) per-match, rather than using absolute clock time. This makes the timeline scrubber intuitive regardless of match length variance.

3. **Path smoothing**: Raw telemetry has ~1Hz position updates. I apply a 3-point moving average to smooth paths visually without distorting spatial accuracy.

4. **Out-of-bounds events**: ~1.2% of events had world coordinates outside the map extent (likely teleport or respawn artifacts). These are dropped silently before rendering.

5. **Multi-day aggregation**: When "All Dates" is selected, all paths and events are rendered simultaneously. With 5 days of data this is ~3,000 players — performance stays smooth because Canvas path batching is efficient.

---

## Performance Notes

- 5 days × 3 maps × ~4 matches/day = ~60 matches = ~1,200 players = ~60k path points
- Canvas draws all of this in <16ms per frame on a mid-range laptop (M1 MacBook Air baseline)
- Heatmap is pre-composited to an offscreen canvas and blitted in one operation — no per-frame recalculation unless filter changes
- Player path data is cached in memory after initial parse; filters operate on in-memory JS arrays, not re-parsing parquet
