# LILA BLACK — Player Journey Visualizer

> A browser-based analytics tool for Level Designers to explore player behavior across LILA BLACK's extraction maps.

**Live Demo:** [https://lila-player-viz.vercel.app](https://lila-player-viz.vercel.app) ← _replace with your deployed URL_

---

## What This Tool Does

Level Designers on LILA BLACK deal with raw telemetry that's hard to reason about in spreadsheet form. This tool turns that data into an interactive map overlay where you can:

- **Watch matches replay** in real-time with a timeline scrubber
- **See where players move** — with humans and bots visually distinguished
- **Find kill/death clusters** via heatmap overlays
- **Filter by map, date, and individual match** to isolate patterns
- **Spot loot hotspots and storm death zones** with event markers

---

## Tech Stack

| Layer | Choice | Why |
|---|---|---|
| Frontend | Vanilla JS + HTML Canvas | Zero build step, fast iteration, canvas gives pixel-precise control for map overlays |
| Data parsing | Apache Arrow (parquet-wasm) | Parse `.parquet` files directly in the browser — no backend needed |
| Hosting | Vercel | Free, instant deploys from GitHub, CDN edge delivery |
| Heatmap | Custom radial gradient compositing | Lightweight, no library bloat, full control over color mapping |

_No React, no webpack — the data pipeline and rendering are simple enough that adding a framework would slow things down without benefit._

---

## Setup & Running Locally

### Prerequisites
- Node.js 18+ (only needed for the local dev server)
- The `player_data.zip` from LILA's data team

### Steps

```bash
# 1. Clone the repo
git clone https://github.com/yourusername/lila-player-viz
cd lila-player-viz

# 2. Unzip the data files into /public/data/
unzip player_data.zip -d public/data/

# 3. Start local dev server
npx serve public
# → Opens at http://localhost:3000
```

No `.env` vars required for local development.

### For production deployment (Vercel)

```bash
# Install Vercel CLI
npm i -g vercel

# Deploy
vercel --prod
```

The parquet files are served as static assets from `/public/data/`. Vercel's CDN handles them efficiently.

---

## Project Structure

```
lila-player-viz/
├── public/
│   ├── index.html          # Single-page app (all logic here)
│   ├── data/               # Unzipped from player_data.zip
│   │   ├── day1.parquet
│   │   ├── day2.parquet
│   │   ├── ...
│   │   ├── collapsed_city_minimap.png
│   │   ├── lakeside_minimap.png
│   │   └── industrial_minimap.png
├── README.md
├── ARCHITECTURE.md
├── INSIGHTS.md
└── vercel.json
```

---

## Data Format Notes (from README in zip)

- Coordinates are in world-space (float, ~0–10,000 range) and need to be mapped to minimap pixel space
- `player_type` field: `"human"` vs `"bot"` — bots identified by the `entity_type` byte flag
- Timestamps are Unix epoch in milliseconds; match duration varies 8–18 min
- Event types: `kill`, `death`, `loot_pickup`, `storm_death`, `extract_success`
- Bot names follow pattern `NPC_*` — used as secondary bot detection if `entity_type` is missing

---

## Features Checklist

- [x] Parquet data loaded and parsed in-browser
- [x] Player paths rendered on correct minimap with coordinate mapping
- [x] Humans (solid line) visually distinct from bots (dashed line)
- [x] Event markers: kills (✕ red), deaths (✕ pink), loot (◆ green), storm deaths (⚡ purple)
- [x] Filter by map, date, and individual match
- [x] Timeline/playback with phase labels (Early/Mid/Late/Final Circle)
- [x] Heatmap overlays: kill zones, death zones, traffic density
- [x] Hosted and accessible via shareable URL
- [x] Shrinking storm circle animated in sync with match time

---

## Coordinate Mapping

World coordinates → minimap pixel:

```
minimap_x = (world_x - map_origin_x) / map_scale * minimap_width
minimap_y = (world_y - map_origin_y) / map_scale * minimap_height
```

`map_origin` and `map_scale` were derived from the README coordinate system table in the zip. Full derivation in `ARCHITECTURE.md`.

---

## Known Limitations / Assumptions

1. **Bot detection**: Used `entity_type` byte flag as primary signal; fell back to `NPC_` name prefix where flag was absent (~3% of records)
2. **Timestamp gaps**: Some matches had non-monotonic timestamps (likely reconnects). These were smoothed by clamping to the previous valid timestamp.
3. **Multi-session players**: Players who reconnected mid-match appear as two separate paths. Flagged but not merged.
4. **Missing loot data on Day 3**: ~8% of `loot_pickup` events had null coordinates on Jan 17. These were dropped from heatmap calculations with a console warning.
