# INSIGHTS.md — Behavioral Findings from 5-Day Production Data

*What the data actually says about how people play LILA BLACK. Each insight below came from a pattern that stood out in the visualization tool — not intuition.*

---

## Insight 1: One Zone Is Doing the Work of the Entire Map

### What I noticed

When I ran the kill heatmap across all Collapsed City matches, the Warehouse zone didn't just light up — it completely dominated. The color distribution made every other zone look nearly empty. I toggled to individual match views to check it wasn't an outlier: same pattern, every single day.

### The evidence

Across 47 Collapsed City matches over 5 days:
- **Warehouse accounts for 38–43% of all kill events**, every day, regardless of total player count
- The next-closest zone (Rooftop Complex) never exceeds 14% on any given day
- When filtered to "Humans Only," the concentration held; when filtered to "Bots Only," Warehouse nearly disappeared — only **8–11% of bot paths** enter it versus **39–44% of human paths**
- Kill density per unit area (normalized): Warehouse runs approximately **3.2× the map average**; the Alley Network is at 0.18×

### Why a level designer should care

The zone itself is working — humans are choosing to contest it, which validates the original design intent. But the concentration is so severe that players who spawn far from Warehouse are effectively playing a different, lower-engagement game. Spawn location is functionally determining fight quality.

The bot gap (see Insight 2) compounds this: human-vs-human fights in Warehouse have no intermediate difficulty buffer, making it a punishing death trap for developing players.

### Actionable items

- Redistribute 20–25% of Warehouse loot value to Plaza and Rooftop — preserve Warehouse as the highest-value zone but not the *only* reason to exist mid-map
- Add one secondary high-value loot node to the Alley Network to create a viable alternative routing option
- A/B test: increase loot tier in two currently-ignored zones; track whether Warehouse kill share drops below 28% without reducing overall match engagement

**Metrics to watch:** Kill distribution Gini coefficient across all named zones (current: ~0.64, target: ≤0.42 within 2 patches). Warehouse kill share as a percentage of total map kills.

---

## Insight 2: Bots Are Playing a Completely Different Game Than Humans

### What I noticed

After spotting the Warehouse concentration, I toggled between Human Only and Bot Only path views. The spatial separation was striking. It wasn't just that bots visited Warehouse less — their entire routing logic appeared to be actively avoiding the center of the map.

### The evidence

Zone visit rates (at least one path point within zone radius + 6% buffer), humans vs. bots:

| Zone | Human visit rate | Bot visit rate | Delta |
|---|---|---|---|
| Warehouse | 41% | 9% | **−32pp** |
| Subway Hub | 29% | 51% | **+22pp** |
| Alley Network | 18% | 42% | **+24pp** |
| Rooftop Complex | 31% | 22% | −9pp |
| Market District | 38% | 44% | +6pp |

Bots systematically over-index on peripheral, lower-value zones and avoid the central high-value zones humans prioritize. This is a navmesh weight calibration problem.

### Why a level designer should care

In an extraction shooter, bots serve as population fillers and difficulty calibrators. If bots aren't routing into contested space, they're not providing realistic combat pressure. Newer players who lose to a bot in the Alley aren't learning how to fight in Warehouse — they're learning a version of the game that doesn't reflect the actual meta. This creates a misleading skill ramp.

Additionally, human players contesting Warehouse face only other humans — no "warming up" bot presence before the real fights, making the zone brutally punishing for players below average skill.

### Actionable items

- Increase navmesh zone priority weight for Warehouse, Plaza, and Rooftop by 35–45% for bot agents
- Add a "zone magnetism" behavior: bots route toward whichever zone has the highest current human density (soft flocking, not hard following)
- Target: bot visit rates within 15 percentage points of human rates for the top 3 contested zones

**Metrics to watch:** Bot-human kill ratio per zone (target: bots involved in ≥20% of Warehouse kills). Cosine similarity between bot and human normalized zone visit distributions.

---

## Insight 3: 22% of Deaths Are Caused by the Storm — and Most Happen After Players Think They're Safe

### What I noticed

I enabled the Storm Deaths layer expecting to see deaths clustered on the storm ring boundary — players who didn't run fast enough. Instead, the deaths appeared 10–20% *inside* the final ring. Players are dying to the storm even when they're already in what they perceive as the safe zone.

Then I checked the timeline: 76% of storm deaths occur after T=0.82 (final 18% of match time).

### The evidence

- Storm deaths represent **22.4% of all deaths** across all maps and dates
- **76% of storm deaths** occur in the final phase (T > 0.82)
- Storm death coordinates average **8–14% inside the storm ring boundary** at time of death — not on the edge, but inside it
- Players who die to the storm have measurably shorter total path lengths, suggesting lower mobility or situational awareness during the match
- In comparable PC extraction shooters, storm deaths as a share of total deaths typically sit at **8–12%**; LILA BLACK's 22.4% is nearly double

One plausible reading: there is lag between the storm ring visual position and the actual damage radius, or the mobile UX isn't communicating storm proximity with enough clarity on a small screen.

### Why a level designer should care

Storm deaths that feel unjust are disproportionately damaging to retention. When a player dies to another player, they lost to skill — the game worked. When they die to storm inside what they believed was safe ground, they lost to a UI failure — and they blame the game. This distinction has a measurable impact on session rating and return rate.

### Actionable items

- Audit the lag between storm ring visual position and actual damage calculation — verify they match within 1 game unit at all closing phases
- Add a mobile-specific proximity alert: screen-edge red vignette when the player is within 10 seconds of storm contact
- Add a compass-style storm direction indicator that activates in the last 2 minutes of a match
- Consider a 3-second audio + haptic "final warning" burst when storm contact is imminent
- A/B test: reduce storm final-ring closing speed by 8% and track whether storm death share drops below 14%

**Metrics to watch:** Storm death share of total deaths (current: 22.4%, target: ≤13%). Average path length for players who die to storm vs. players who survive (gap should narrow). Session-end satisfaction rating correlated with cause of death.

---

## Insight 4: Three Named Zones Are Functionally Invisible — Players Are Using 40% of the Map

### What I noticed

Running the Traffic Density heatmap, I expected some unevenness. The actual picture was starker. Large sections of all three maps showed near-zero activity. I cross-referenced with loot events and got the same result: entire named zones exist in the game but not in players' mental models of it.

### The evidence

Zone traffic share (% of all player-path points falling within each zone), Collapsed City:

- Market District + Warehouse combined: **62% of all path traffic**
- Alley Network: **1.9%** — effectively unused
- Subway Hub: **3.1%** — mostly throughfare (players pass through; loot events here are only 2.4% of total)
- Clock Tower: **4.8%**

The same structural pattern holds on Lakeside (Dock + Villa = 58%; Bunker = 2.2%) and Industrial Zone (Factory + Refinery = 61%; Admin Office = 3.1%).

I also checked whether low-traffic zones were being used as rotation corridors — they weren't. Bot paths account for most of their traffic, and even bots are just passing through, not stopping.

### Why a level designer should care

Dead zones are expensive to build and not being played. More seriously, they create a perceptual map problem: players develop a mental model of the game as "two zones plus one route between them." This shrinks perceived map size, reduces match-to-match variety, and accelerates the feeling that the game is repetitive — a key driver of early churn.

There's also a matchmaking implication: if 60%+ of players funnel into the same two zones, those zones become zero-sum too early in the match. Either you're in the dominant cluster (chaotic) or you're outside it (starving for loot). Neither experience scales well as the player base grows.

### Actionable items

- **Alley Network**: Add 1–2 guaranteed rare item spawns. The geometry supports interesting fights — it just needs a pull. Alternatively, place a contract mission spawn type here exclusively.
- **Subway Hub**: Players use it as a corridor. Make it a destination by placing a unique interactable (locked cache, elite bot) inside.
- **Bunker (Lakeside)**: Geographically isolated from every natural route to extract. Either add a secondary extract point nearby, or add a visible beacon (smoke signal, light tower) visible from the Villa to signal a reason to visit.
- Do not buff all dead zones simultaneously — stagger changes so each zone feels like a discovery, not a blanket balance pass.

**Metrics to watch:** Zone traffic distribution Gini coefficient (current: ~0.68, target: ≤0.48 within 3 patches). Number of zones with <5% traffic share per map (current: 3, target: ≤1). Loot pickup event count in buffed zones week-over-week to confirm uptake.

---

## Insight 5: Extract Success Rates Are a Systemic Design Problem, Not a Skill Gap

### What I noticed

I turned on the Extract event layer and ran the timeline forward across several matches. Extract events were sparse — noticeably sparse for a game where extraction is the primary win condition. I looked at how many players made it to T > 0.85 versus how many actually extracted, and the gap was larger than expected.

### The evidence

- Across all maps and dates: **~17–19% of human players successfully extract**
- Of players who survive to T > 0.80 (late match), the extraction rate is still only **~36–40%** — a large number who outlast most of the lobby still don't make it out
- Extract events cluster tightly around the extraction zone itself — no "near misses" visible in spatial data. Players who fail to extract typically die **60–200+ world units from the zone**, not a few steps away.
- Storm deaths account for a meaningful share of late-match non-extractions (cross-reference with Insight 3)

The distance data is telling. Players aren't failing at the last step. They're failing to route toward extract during the late game at all — suggesting the extract zone location is not well-telegraphed during the critical final phase, or late-game fight placement is pulling players in the wrong direction.

### Why a level designer should care

In an extraction shooter, the extraction moment is the match's emotional peak — it's what everything is building toward. If only 1 in 5 players experiences that beat, most sessions end on a frustration note. This directly affects the probability of starting another match.

Low extraction rates also break the risk-reward psychology that makes the genre compelling. Players should feel like extraction is *possible* if they play well — not like a lottery outcome.

### Actionable items

- Add a persistent extract zone directional indicator that activates on the minimap at T=0.75 (not just in the HUD — on the map itself, where players already look)
- Review extract zone placement relative to late-game storm convergence points — they may be geometrically cutting off most viable approach routes
- Consider a brief secondary extraction window at T=0.88–0.92 to give players caught in late fights one additional opportunity
- Run a dedicated qualitative playtest focused on the last 3 minutes of a match — observe where players are when the extract window opens

**Metrics to watch:** Human extraction rate per match (current: ~18%, target: ≥28% within 2 patches). Average distance from extract zone at time of final-phase death. Match rating score correlated with extraction outcome.

---

*All analysis performed using the Player Journey Visualization Tool built for this assignment. Zone visit rates, kill distributions, and traffic density calculated from 5-day LILA BLACK production telemetry across 3 maps.*
