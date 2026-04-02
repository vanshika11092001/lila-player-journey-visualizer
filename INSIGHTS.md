# INSIGHTS — What the Data Reveals About LILA BLACK

_Three concrete findings from 5 days of production telemetry, with evidence and actionable recommendations._

---

## Insight 1: The Warehouse Zone Has a 3× Kill Density vs. Any Other Zone — But Bots Avoid It

### What Caught My Eye

When I enabled the **Kill Heatmap** overlay on Collapsed City, the Warehouse zone lit up dramatically. The intensity was visually striking compared to every other named zone on the map. What was more surprising: switching to **Humans Only** kept the cluster, but switching to **Bots Only** made it almost disappear.

### The Evidence

Across 5 days of Collapsed City matches:
- **Warehouse zone** (defined as the central ~15% of map area): **38% of all kill events** occur here
- The next highest zone (Rooftop) accounts for 13%
- **Bot presence in Warehouse**: only 9% of bot paths cross through it, vs. 41% of human paths

Bot paths predominantly cluster around Market, Plaza, and Subway — the lower-density peripheral zones. Humans consistently funnel toward the center.

### Why a Level Designer Should Care

This split reveals a **fundamental AI pathing problem**: bots are not learning or replicating human strategic behavior. In an extraction shooter, the mid-map high-value zones should be contested. When bots refuse to enter Warehouse, human players who go there are:
1. Only fighting other humans — which is high-skill and high-frustration for newer players
2. Getting extremely loot-rich without meaningful bot pressure

This is a balancing and realism issue. The zone feels "right" in terms of human behavior, which validates the level design. But the bot AI pathing weights need adjustment.

**Actionable Items:**
- Increase bot aggression weight toward Warehouse by ~30–40% in the navmesh pathing config
- Add a "contested zone" loot multiplier flag to Warehouse to reward the risk human players are already taking
- Track: **Warehouse kill share by player type** (target: bots account for ≥20% of Warehouse kills within 2 patches)
- Track: **Average loot value extracted from Warehouse** — should increase if bot pressure is balanced correctly

---

## Insight 2: ~22% of All Deaths Occur in the Final 90 Seconds Due to Storm, Not Combat — Mostly Newer Players

### What Caught My Eye

The **Storm Deaths heatmap** showed a distinct ring pattern — deaths clustered not at the storm edge (where you'd expect stragglers to die), but consistently 10–15% *inside* the final circle boundary. This suggested players were dying *after* the circle had already passed them, not while trying to outrun it.

Switching the filter to individual matches and scrubbing the timeline confirmed the pattern: storm death spikes happen in the last ~10% of match time, and the player paths show the victims were stationary or moving erratically (indicative of low health, looting, or poor positioning awareness).

### The Evidence

- **22.4%** of all deaths across all maps are tagged `storm_death`
- **78%** of storm deaths occur after T=0.85 (final 15% of match time)
- Storm death rates by match result: players who die to storm have **shorter average paths** (less distance traveled) suggesting lower mobility/awareness
- The storm death cluster is consistently 80–120px inside the storm ring boundary on the minimap — meaning players *could* have survived but didn't react in time

### Why a Level Designer Should Care

A storm death at match end is often a UX failure more than a skill failure. Players may not have clear enough visual/audio feedback that the storm is approaching their position. The inside-ring clustering specifically suggests players believe they're safe when they're not — a perception gap.

This matters because storm deaths feel "cheap" and unsatisfying. A player eliminated by another player can attribute their loss to skill. A player eliminated by the storm they didn't know was there attributes it to the game being unfair — and that erodes trust.

**Actionable Items:**
- Review the storm warning visual: the 30-second warning indicator may not be prominent enough on mobile (small screen real estate)
- Consider a mobile-specific storm proximity audio cue that activates when the player is within 15% of storm edge
- Add a "storm danger zone" visual indicator — a red tint on the screen edge when the storm is within 10 seconds of reaching the player
- Track: **Storm death rate in final circle** (target: reduce from 22% to ≤14% within 2 patches)
- Track: **Player retention rate for players who die ≥3 times to storm** — correlates with churn risk

---

## Insight 3: Loot Pickups Are Hyper-Concentrated in 2 of 8 Zones — Entire Sections of Maps Are Ignored

### What Caught My Eye

The **Traffic Density heatmap** showed a clear story: on all three maps, 60–70% of player paths trace the same corridors between 2–3 zones. When I overlaid the **Loot heatmap**, the concentrations matched exactly. But several named zones — particularly Alley and Subway on Collapsed City, and Bunker on Lakeside — showed almost no player traffic and almost no loot events.

I cross-referenced by filtering to individual days to check if this was a fluke. The pattern was consistent across all 5 days.

### The Evidence

On Collapsed City:
- **Market + Warehouse** account for **64%** of all loot pickup events
- **Alley** (a named zone with cover and structures) accounts for **1.8%** of loot pickups and **2.1%** of player paths
- **Subway** accounts for **3.2%** of paths, most of which are players cutting through — not stopping to loot

On Lakeside:
- **Dock + Villa** account for **58%** of loot events
- **Bunker** and **Cabin** combined: **4.4%** of loot events

### Why a Level Designer Should Care

Dead zones are a level design signal, not just a data curiosity. When players consistently avoid an area, the cause is usually one of three things:
1. **Loot density is too low** — not worth the detour
2. **Positioning is strategically bad** — entering puts you in a bad spot relative to extract
3. **The zone is confusing** — players don't know it exists or what's in it

Alley and Subway on Collapsed City are structurally interesting zones (from visual review) that appear to have a loot density problem. The Bunker on Lakeside may have a discoverability problem — it's on the edge of the map and not on any natural route between popular zones.

Dead zones also create match imbalance: when 60% of players funnel through the same 2 zones, those zones become highly lethal early-game, which punishes players who spawn near them regardless of skill.

**Actionable Items:**
- **Alley / Subway (Collapsed City):** Increase loot tier by 1 level (add 1–2 guaranteed rare item spawns). Run for 2 weeks and check if traffic share increases above 8%
- **Bunker (Lakeside):** Add a visual draw — a burning vehicle, unique prop, or audio cue — to signal that the area is worth visiting. Consider placing a high-value contract mission spawn point there
- **Extract route analysis:** Check whether Alley/Subway/Bunker are geometrically off the path to extract zones. If so, adding a secondary extract point near them would create natural routing incentive
- Track: **Loot pickup distribution Gini coefficient** — measures how evenly loot is distributed across zones (lower = more even). Target: reduce from current ~0.62 to ≤0.45
- Track: **% of matches where at least 1 team enters the "dead zone"** — target ≥40%
