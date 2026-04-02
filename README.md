# LILA BLACK — Player Journey Visualization Tool

A browser-based analytics tool for the Level Design team to explore player behavior across LILA BLACK's extraction maps. Built for the LILA APM Written Test.

**Live demo:** `https://lila-player-viz.vercel.app` ← replace with your deployed URL

---

## What this does

Level Designers on LILA BLACK deal with raw telemetry that's hard to reason about in spreadsheet form. This tool turns that data into an interactive map overlay where you can:

- Watch a match unfold in real-time using the timeline scrubber
- See where players move — humans (solid lines) and bots (dashed lines) visually separated
- Spot kill clusters, death zones, and loot hotspots with heatmap overlays
- Filter by map, date, and individual match to isolate specific sessions
- Hover any named zone to get a live breakdown of kills, traffic, and bot vs. human visit rates
- Read auto-generated behavioral insights that update when you change filters

---

## Tech stack

| Layer | Tech | Why |
|---|---|---|
| Frontend | Vanilla JS + HTML Canvas | Zero build step; Canvas handles 60k+ path points at 60fps without a framework |
| Data parsing | parquet-wasm (Apache Arrow, WASM) | Parse `.parquet` files directly in the browser — no backend, no preprocessing |
| Hosting | Vercel | Free tier, instant GitHub deploy, global CDN |
| Heatmap | Custom radial gradient compositing | No library overhead; full control over layer blending and opacity |

Full decision rationale in `ARCHITECTURE.md`.

---

## Running locally

### Prerequisites

- Node.js 18+ (only for the dev server)
- The `player_data.zip` from the LILA data team

### Steps

```bash
git clone https://github.com/yourusername/lila-player-viz
cd lila-player-viz

# Unzip data into /public/data/
unzip player_data.zip -d public/data/

# Start local server
npx serve public
# → http://localhost:3000
```

No environment variables required for local development.

### Deploying to Vercel

```bash
npm i -g vercel
vercel --prod
```

Parquet files are served as static assets from `/public/data/` via Vercel's CDN.

---

## Project structure

```
lila-player-viz/
├── public/
│   ├── index.html              # Entire app (JS, CSS, logic)
│   ├── data/
│   │   ├── day1.parquet        # Unzipped from player_data.zip
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

## Feature checklist

- [x] Parquet data loaded and parsed in-browser (no preprocessing required)
- [x] Player paths rendered on correct minimap with world-to-canvas coordinate mapping
- [x] Humans (solid blue line) visually distinct from bots (dashed orange line)
- [x] Event markers: kills `✕`, deaths `✕`, loot `◆`, storm deaths `↯`, extractions `▲`
- [x] Filter by map, date, and individual match
- [x] Timeline playback with phase labels (Early / Mid / Late / Final Circle)
- [x] Survival curve: live chart of how many humans and bots are alive at each point
- [x] Heatmap overlays: kill zones, death zones, traffic density
- [x] Zone Intelligence panel: hover any zone to see kill share, loot events, human/bot visit rates
- [x] Auto-generated insights panel: updates when you change filters
- [x] Mini zone kill distribution chart in sidebar
- [x] Alive-player count updating in real-time with playback
- [x] Deployed and accessible via shareable URL

---

## Notes on the data

- Coordinates are in Unreal Engine world-space (float, 0–10,000 range) and mapped to minimap pixel space — see `ARCHITECTURE.md` for the exact formula and validation approach
- Bot detection: `entity_type` byte flag (primary), `NPC_` name prefix (fallback for ~3% of null cases)
- Timestamps are Unix epoch in milliseconds; I normalize per-match to [0,1] for timeline consistency across variable match lengths
- Event types in the raw data: `kill`, `death`, `loot_pickup`, `storm_death`, `extract_success`
- ~1.2% of events had out-of-bounds coordinates and are dropped before rendering (logged to console)
- Day 3 (Jan 17) `loot_pickup` events had ~8% null coordinates — these are excluded from loot heatmap calculations with a console warning

---

## What I'd build next

Given more time, the next highest-value additions would be:

1. **Spawn point heatmap** — understanding where players spawn relative to where they die is the missing link for spawn balancing
2. **Match outcome filter** — see only matches where the winner extracted vs. everyone died, to understand whether winning routing looks different
3. **Zone engagement funnel** — for each zone, what % of players who enter it leave alive vs. die there
4. **Session export** — let designers download a filtered view as a PNG with annotations, for sharing in design reviews
