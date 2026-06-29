# Interactive platform design: a "Landscape Efficiency Frontier" geo-agent app

**Goal.** Let a user pick *any* area — draw a polygon, type a country/region, or click a hex — and see
**where that area sits in the biodiversity × carbon × net-economic-value trade-off space**, and how far it
is from its efficiency frontier. The paper gives *static, national* frontiers; this turns them into a
**live, any-AOI, any-weighting** exploration that complements the paper's message ("large simultaneous gains
are possible") by letting people feel the trade-offs for places they care about.

## Why the geo-agent framework fits

A geo-agent app (like `../global-30x30`) is just three files over the shared runtime:
`system-prompt.md` (persona + canned SQL recipes), `layers-input.json` (which collections load, the map
view, **`draw_enabled`**, welcome examples), and the CDN runtime (MapLibre map + the duckdb-geo MCP `query`
tool + map styling/filter tools + geocoder + **draw tool** + sidebar). So most of what we need is
*configuration*, not new code:

| Capability needed | Framework support |
|---|---|
| Pick a country/region | ✅ geocoder + Overture `ISO3` filter / SQL |
| **Draw an arbitrary AOI** | ✅ `draw_enabled: true` — drawn polygon → `ST_Intersects` / H3 prefilter in `query` |
| Click a hex / zoom to scale | ✅ map interaction; H3 res 5↔8 rollups already in the layers |
| Zonal stats over the AOI (bio, carbon, econ totals) | ✅ duckdb-geo `query` (the POCs already do this) |
| **Run the optimization for the AOI** | ✅ it's a separable per-parcel `arg_max … GROUP BY` — one SQL query (POC 6), fast enough to run live |
| Recolor the map by the chosen land-use solution / by an objective | ✅ `set_style` / `add_layer` on a computed hex tile (`register_hex_tiles`) |
| **Plot the frontier curve + "you are here" point** | ❌ **no built-in charting** — the one real gap (see below) |

## The core interaction loop

1. **Select** — draw / geocode / click → an AOI resolved to a set of H3 cells.
2. **Score** — `query` returns the AOI's *current* (biodiversity, carbon, economic) totals and its
   achievable maxima.
3. **Optimize** — sweep objective weights; for each weight the separable per-parcel argmax (POC 6) picks a
   land-use per cell and sums to a frontier point. Returns the frontier + the "current" point's position.
4. **Show** —
   - **Map**: recolor the AOI's hexes by the chosen land use under the current weighting (restore / keep / intensify / graze), or by any single objective.
   - **Frontier plot**: the trade-off curve with the AOI's *current* point marked and the *gap* to the frontier (← the charting gap).
   - **Headline**: agent-written, e.g. *"This watershed could increase carbon storage 22% with no loss of economic value, or raise economic value 40% while keeping 90% of its biodiversity."*

## The interactive moves that go beyond the paper

- **Any AOI, not just nations.** Draw a watershed, a concession, a protected-area buffer, an ecoregion — the paper only published country frontiers.
- **A weight slider as the headline control.** Drag bio↔carbon↔economy; the map *re-solves live* (cheap separable argmax) and recolors by the new optimal land-use allocation, while the headline numbers update. This is the "dig into the message" experience — *watch* where and how much conservation you trade for economic value.
- **Co-benefit / low-opportunity-cost finder.** "Highlight cells that are high biodiversity **and** high carbon **and** low economic value **and** unprotected" — the win-win restore targets (POC 5 found ~10% of land).
- **Compare two AOIs' frontiers** side by side (e.g. Gabon vs Indonesia).
- **Scale drill-down.** Global → country (res 5) → drawn parcel, using the H3 parent columns.

## The charting gap — three options (cheapest → best)

1. **Agent-rendered artifact (interim).** The agent emits the frontier as a small self-contained HTML/JS scatter (Observable Plot inline) shown in the sidebar/chat. Zero framework change; lower polish.
2. **Map-only proxy (works today).** Skip the chart: deliver the frontier as a `query` table + recolor the map by the optimal solution; the "where you fall" message is text + map. Honest and immediately shippable.
3. **A `FrontierPanel` component (best).** A modest addition to the geo-agent template: a docked panel that takes the SQL frontier result and renders the curve + "you are here" + a draggable weight control wired back to a re-`query`. This is the only real front-end work; it's small and reusable.

Recommendation: ship with **#2** on day one (pure config over the existing runtime), add **#1** for the curve, and propose **#3** upstream to `geo-agent-template` as a reusable widget if the pattern proves out.

## What it takes to build (config-first)

Fork `geo-agent-template` → `landscape-frontiers`:
- **`layers-input.json`**: load `nci-frontiers` (economic + FLII + transition costs), `iucn-richness-2025`, `wwf-ecoregions-2017`, `irrecoverable-carbon`, `wdpa`, `overture-divisions-countries`; `draw_enabled: true`; global view; welcome examples below.
- **`system-prompt.md`**: frame the three objectives + the IUCN I–IV constraint; embed the **canned recipes** as copy-paste SQL — (a) AOI zonal scorecard, (b) the weight-sweep frontier (POC 6), (c) the co-benefit finder (POC 5), (d) recolor-by-solution via `register_hex_tiles`. Document the per-layer reducers (densities = AVG, transition costs = SUM) and the res-5 join grid so the agent never mis-aggregates.
- **Welcome examples**: *"Draw an area and show its biodiversity–carbon–economy frontier"*, *"Where could my country gain the most carbon at zero economic cost?"*, *"Recolor the map by the balanced-weight land-use plan"*, *"Find unprotected win-win restoration cells in the Amazon."*

## Fidelity scales with the ingest

- **Today (2-objective + simplified econ):** works now from the catalog (biodiversity × carbon) plus the point-binned econ POC — enough to demo the loop.
- **First-pass ingest (in flight, #335):** 13 hex layers → a real 3-objective scorecard + a 2-alternative (natural vs best-production) frontier per AOI.
- **Full ingest (pass 2):** all 14 transition costs + the per-alternative crop/grazing/forestry value tables → the faithful **13-alternative** frontier the paper runs — same separable argmax, just more alternatives per cell. No new app work, only more layers.

The optimization never leaves SQL, so the whole thing stays inside the duckdb-geo MCP the app already
speaks — the platform is mostly a system-prompt + a frontier widget away.
