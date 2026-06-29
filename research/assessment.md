# Can we replicate Polasky et al. 2026 with the duckdb-geo catalog?

**Paper:** Polasky, Hawthorne, Chaplin-Kramer, Smith, Gerber, Mamun, Ruckelshaus, Russ et al.,
*"Landscape efficiency frontiers for biodiversity, climate mitigation, and net economic value,"*
**Science 392, 1069 (2026)**, DOI 10.1126/science.aea9058.

Based on the Supplementary Materials (Materials & Methods, Tables S4–S12) and the Reproducibility
Checklist. Assessed against the 157-collection public STAC catalog served through `mcp__duckdb-geo__query`.

---

## TL;DR

**Yes — the whole paper is reproducible and explorable.** The authors released the full dataset on Dryad
under **CC0-1.0** (public domain): the per-pixel input layers for *all three* objectives — including the
economic revenue surfaces — plus the optimization code and the complete results. So the hard-to-derive
parts (crop/grazing/forestry economics from GTAP + InVEST) need not be rebuilt; they are downloadable.

| Component | Reproduce it? | How |
|---|---|---|
| **Biodiversity objective** | ✅ (mostly) | 4 of 6 sub-indices already in our catalog (`iucn-richness-2025`, `wwf-ecoregions-2017`); **forest intactness (FLII) ships in the CC0 deposit**; **KBA is withheld there for licensing** (canonical source needed) |
| **Climate-mitigation objective** | ✅ | carbon in catalog (`irrecoverable-carbon`); paper's exact carbon-by-ecoregion + methane in the input deposit |
| **Net economic value objective** | ✅ | per-pixel crop/grazing/forestry revenue + transition-cost GeoTIFFs in the CC0 input deposit |
| **Optimization (Pareto frontier)** | ✅ | **not MILP** — separable weighted-sum argmax; a DuckDB `arg_max … GROUP BY parcel` query + a thin notebook, or the authors' 308 KB Julia code |

The single most important alignment is structural: the paper's spatial unit — **"8 000-ha hexagonal land
parcels"** — is essentially an H3 cell (≈ resolution 5–6), so its pixel→parcel aggregation model and our hex
pipeline are the same shape. The path of least resistance is to ingest the 8.5 GB of CC0 input GeoTIFFs
through our `raster-workflow` into H3 hex, join them in DuckDB, and run the (trivially separable)
optimization. Our existing catalog adds value on top by supplying **independent, newer** biodiversity,
carbon, and protected-area layers to refresh and cross-check the paper's frozen 2019-vintage inputs.

---

## 1. What the paper does

It assembles ~1.85 billion 300 m (10 arc-sec) pixels globally, scores **13 land-use/land-management
alternatives** per pixel on **three objectives** — climate mitigation, biodiversity, net economic value —
aggregates pixels into **~8 000-ha hexagonal parcels**, and runs an optimization that, for each randomly
weighted combination of the three objectives, picks the best alternative per parcel. Sweeping the weights
traces a **Pareto efficiency frontier** per country (146 countries). Engine = InVEST + custom extensions.

The three objectives decompose into:

- **Climate mitigation** = above- + below-ground biomass carbon (Spawn et al. 2020), differentiated by ecoregion, minus 20 yr of livestock methane, in CO₂e.
- **Biodiversity** = the **max** of six normalized sub-indices: (i) vertebrate species richness, (ii) threatened/endangered habitat, (iii) endemic/range-restricted habitat, (iv) rare-ecoregion habitat, (v) forest intactness, (vi) Key Biodiversity Areas.
- **Net economic value** = crop + grazing + forestry revenue − production − transport − transition costs.

All three are constrained by protected areas (no production in **IUCN categories I–IV**), urban areas,
slope, soil, and irrigation-sustainability masks.

---

## 2. The released data (Dryad, CC0-1.0)

The Reproducibility Checklist states *"No new materials were created"* and gives a Data Availability
Statement with three Dryad deposits, all **CC0-1.0** — public domain, freely mirrorable:

| Deposit | DOI | Size | Contents |
|---|---|---|---|
| **Input data** | `10.5061/dryad.qjq2bvqw5` | **8.5 GB** | Per-pixel **300 m GeoTIFFs**. Economic (CC0, USD/ha): crop revenue current/irrigated/rainfed (no-palm + palm), forestry returns, grazing revenue; grazing methane (kg/ha/yr); **14 transition-cost surfaces (USD/pixel)**. Biodiversity: **FLII (forest intactness) provided**; carbon-by-ecoregion lookup + ecoregion rasters; ESA land cover (current + potential); a `pixel_area_km2` raster; PREDICTS lookups; all suitability/slope/sustainable-irrigation/PA masks. **⚠️ IUCN richness and KBA rasters are *withheld* for licensing (described only)** — only analogous public sources linked. |
| **Results** | `10.5061/dryad.b2rbnzsvt` | **110 GB** | Optimization outputs — per-country Pareto frontiers + summary statistics (`solutions.h5` per country). |
| **Code** | `10.5061/dryad.9s4mw6mxn` | **308 KB** | Python `natures_frontiers` (preprocess → per-decision-unit value tables; postprocess → plots/maps) + Julia `NaturesFrontiers.jl` optimizer. |

Three escalating ways to engage:

- **Path A — analyze the published results.** No recomputation; read/visualize the 110 GB of frontiers directly.
- **Path B — ingest the inputs and re-run the optimization.** Download the 8.5 GB CC0 GeoTIFFs → `raster-workflow` → H3 hex. The per-decision-unit value aggregation (`preprocess.py`) is exactly our pixel→hex sum; the optimization is a per-parcel weighted argmax + Pareto filter in DuckDB + a thin notebook (§5), or the authors' Julia code as-is.
- **Path C — re-derive from primary sources.** Still out of reach (GTAP, InVEST, ORCHIDEE-GM) and unnecessary given A and B.

The CC0 dedication covers the economic layers too: GTAP's own database is license-gated, but the authors'
*derived* per-pixel net-value rasters are their work product, released CC0 — redistributing those is clean.

---

## 3. Data crosswalk: paper input → our existing catalog

This is what we already hold independently — useful both as a no-download Tier-1 path and as **newer**
layers to refresh the paper's frozen inputs. Legend: ✅ exact/near-exact · 🟡 strong proxy · 🟠 partial · ❌ absent (→ but in the CC0 input deposit).

### Biodiversity objective

| Paper sub-index (source) | Catalog holding | Status |
|---|---|---|
| (i) Vertebrate species richness (IUCN range maps + PREDICTS) | `iucn-richness-2025` `*_sr` layers (amphib/bird/mammal/reptile/combined), hex res 5 | ✅ (IUCN 2025 vs paper's 2019; we lack the PREDICTS land-use modulation) |
| (ii) Threatened/endangered habitat (IUCN Red List) | `iucn-richness-2025` `*_thr_sr`; `iucn-taxonomy-2025` | ✅ |
| (iii) Endemic / range-restricted (inverse-range-weighted) | `iucn-richness-2025` `combined_rwr`, `combined_thr_rwr` (Σ inverse range size) | ✅ — this *is* the paper's inverse-range weighting |
| (iv) Rare-ecoregion habitat (Dinerstein 2017, rarity = 1/area) | `wwf-ecoregions-2017` (847 ecoregions, 14 biomes) | ✅ — same dataset (see §4) |
| (v) Forest intactness (Grantham FLII 2020) | — | ❌ → **`flii.tif` provided CC0** in the input deposit (0–10000 index); canonical product trackable — #334 |
| (vi) Key Biodiversity Areas | — | ❌ — **withheld from the deposit for licensing**; need the canonical KBA source (license-gated) — #334 |
| land-cover basemap (ESA CCI 300 m) | `cgls-lc100-2019` (Copernicus 100 m); `nlcd-2024` (CONUS) | 🟡 (ESA CCI tracked in #161; in CC0 deposit as the modified map) |
| *(bonus, beyond paper)* | `ncp-biodiversity` (Chaplin-Kramer 2019 — a co-author), `mobi-species-richness-all`, `plant-richness`, `freshwater-species-richness`, `gbif-derived` | 🟡 |

**4 of 6 sub-indices reproducible from our catalog alone**; the other 2 from the CC0 deposit.

### Climate-mitigation objective

| Paper input | Catalog holding | Status |
|---|---|---|
| Above+below biomass carbon (Spawn 2020), ecoregion-differentiated | `irrecoverable-carbon` — Noon/CI v2, ~300 m, hex res 9, area-integrated to Mg C totals per cell, multi-year | 🟡 (built on Spawn + soil; different framing, validated, SUM-able; paper's exact carbon-by-ecoregion table is in the CC0 deposit) |
| Livestock methane (20-yr) | — | ❌ (in CC0 input deposit) |

### Net economic value objective

| Paper input (Table S4) | Catalog | Status |
|---|---|---|
| Crop revenue surfaces; grazing revenue; forestry returns; transition costs | — | ❌ → **published as per-pixel CC0 GeoTIFFs in the input deposit** |
| Upstream models (GTAP 10, InVEST, ORCHIDEE-GM, FAOSTAT, World Bank yields, Weiss travel-time, Hoskins pasture) | — | ❌ (re-derivation out of scope; outputs available CC0) |
| Closest we hold | `fmmp-2022` (CA farmland), `ghs-pop-2020`, RAP rangeland (US) | 🟠 |

### Constraints / masks

| Paper constraint | Catalog | Status |
|---|---|---|
| No production in **IUCN cat I–IV** protected areas | `wdpa` with `IUCN_CAT`, hex res 8/9 globally | ✅ Exact — `WHERE IUCN_CAT IN ('Ia','Ib','II','III','IV')` |
| Biomes for restoration target (Dinerstein 14 biomes) | `wwf-ecoregions-2017` `BIOME_NAME`/`BIOME_NUM` | ✅ |
| Urban exclusion | `cgls-lc100-2019` urban class; `ghs-pop` | 🟡 |
| Slope / soil / irrigation-sustainability | — | ❌ (suitability masks in CC0 deposit) |

---

## 4. The methodological fit: H3 *is* their "hexagonal parcel"

The paper:

> *"We group pixels into hexagonal land parcels (8 000 hectares at the equator) for optimization … A parcel's score for an objective is the sum of its constituent pixels' values."*

That is exactly what our pipeline produces: every layer above is aggregated from ~300 m / 10 km source
pixels into H3 cells by area-weighted sum/mean/max. 8 000 ha sits between **H3 resolution 5** (≈ 25 200 ha)
and **resolution 6** (≈ 3 600 ha). Our layers are hexed at res 5 (IUCN richness), 7 (NCP), 8 (ecoregions,
WDPA-coarse), and 9 (carbon, WDPA-fine), and all carry parent-resolution columns, so a common analysis grid
is one `h3_cell_to_parent()` / `GROUP BY h5` away. The authors' own `preprocess.py` performs this same
pixel→decision-unit aggregation before optimizing.

**Worked example — reproducing biodiversity sub-index (iv)** "rare-ecoregion rarity = inverse of global
ecoregion area", directly from our hexed WWF/Dinerstein ecoregions (H3 cell count as the equal-area proxy):

```sql
WITH eco AS (
  SELECT ECO_NAME, BIOME_NAME, REALM, COUNT(DISTINCT h8) AS n_cells
  FROM read_parquet('s3://public-ecoregion/ecoregion/hex/h0=*/data_0.parquet')
  GROUP BY ECO_NAME, BIOME_NAME, REALM)
SELECT ECO_NAME, n_cells, ROUND(1.0/n_cells * 1e6, 3) AS rarity_index
FROM eco ORDER BY rarity_index DESC LIMIT 15;
```

Result (rarest first — small oceanic-island endemism hotspots, exactly as the rarity logic predicts):

| ECO_NAME | n_cells | rarity_index (×1e6) |
|---|---|---|
| St. Peter and St. Paul Rocks | 8 | 125 000 |
| Malpelo Island xeric scrub | 10 | 100 000 |
| San Félix-San Ambrosio Islands temperate forests | 11 | 90 909 |
| Trindade-Martin Vaz Islands tropical forests | 14 | 71 429 |
| Lord Howe Island subtropical forests | 17 | 58 824 |
| … | … | … |

The same pattern extends to species/threatened/range-weighted richness (`iucn-richness-2025`), the carbon
objective (`SUM(carbon)` over `irrecoverable-carbon` per parcel), and the IUCN I–IV constraint on `wdpa`.

---

## 5. The optimization is not MILP — and DuckDB can do it

The released Julia code (`NaturesFrontiers.jl`) *"performs multiple linear optimization problems by randomly
sampling weight vectors from the positive unit sphere … solves many weighted-sum problems in parallel."*
That is **linear weighted-sum scalarization**, not mixed-integer programming — and it is computationally
trivial because the problem is **separable**:

- objectives are **additive over parcels** (a country score is a sum of parcel scores), and
- there are **no coupling constraints** across parcels — no area target, no budget. Every constraint (protected areas, slope, soil, irrigation) is *per-parcel feasibility*, only pruning each parcel's ≤14 allowed activities; nothing links one parcel's choice to another's.

So for any fixed weight `w`, `max Σ_parcels (w·score) = Σ_parcels max_activity (w·score)` — an independent
**per-parcel argmax**, which is one SQL statement:

```sql
-- one Pareto-supported landscape for one weight vector w = (w_bio, w_clim, w_econ)
SELECT parcel_id,
       arg_max(activity, w_bio*bio + w_clim*clim + w_econ*econ) AS chosen_activity,
       max(              w_bio*bio + w_clim*clim + w_econ*econ) AS parcel_score
FROM parcel_activity_scores            -- one row per (parcel, allowed activity)
GROUP BY parcel_id;
```

The frontier = loop that over many random `w`, sum the chosen parcels to a national (bio, clim, econ)
triple, keep the non-dominated points — both the weight sweep and the Pareto filter are a few lines of
numpy (or themselves SQL). Weighted-sum recovers only *supported* (convex-hull) efficient points, but
summing thousands of independent discrete choices makes the achievable set ~convex (Shapley–Folkman), so the
hull essentially *is* the frontier — which is why the simple method suffices. A coupling constraint (e.g.
"hit biodiversity target T at least economic cost") would have made it a genuine combinatorial/MILP problem
(à la Marxan); the authors avoided that by tracing the whole frontier with unconstrained scalarization.

**The optimization was never the hard part.** The per-parcel × per-activity *score table* is — and that is
exactly what the CC0 input deposit provides.

---

## 6. Recommendation

Two complementary moves:

1. **Quick win (from the existing catalog, days):** a **biodiversity × carbon** analysis with the exact IUCN I–IV constraint — *"how much biodiversity value / irrecoverable carbon sits outside IUCN I–IV protection, and where do the two co-occur?"* — at native resolution, in SQL, using inputs *newer* than the paper's. No external dependency.

2. **Full replication (a short project):** ingest the **8.5 GB CC0 input GeoTIFFs** as H3 hex via `raster-workflow` (reducer per layer: revenue/carbon = area-integral like our carbon build; biodiversity indices = `mean`/`max`; land cover = `mode`). Then the separable optimization is a notebook over DuckDB. This puts the paper's *entire* three-objective dataset — economics included — into the catalog as queryable, joinable layers, and lets the geo-agent answer efficiency-frontier questions interactively.

Our catalog's distinct contribution beyond replication is the **independent, refreshed** biodiversity,
carbon, and protected-area layers (IUCN 2025, multi-year Noon/CI carbon, current WDPA) for cross-checking
and updating the frozen 2019-vintage inputs.

### Ingest candidates → tracked in **#334**
- **Forest Landscape Integrity Index** (Grantham 2020) — the paper's `flii.tif` is in the CC0 deposit; canonical upstream product also fine for general catalog use.
- **Terrestrial KBA** — **not** in the deposit (licensing); needs the canonical (license-gated) KBA source.
- **ESA CCI Land Cover 300 m** — already tracked in **#161** (item 8).
- **The Dryad economic + scenario layers** (crop/grazing/forestry revenue, transition costs, suitability masks) — CC0, ingestable if pursuing full 3-objective replication.
