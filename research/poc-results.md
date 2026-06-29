# Proof-of-concept: parametric sensitivity probing of the biodiversity objective

**Goal.** Show that we can reproduce the paper's conservation-value construction from layers *already in
the catalog*, and cheaply vary (a) **which data layers** are combined, (b) the **combine rule**, and
(c) the **resolution** of the decision unit, to see how much the result moves — plus a headline
biodiversity × carbon × protected-area result.

All queries run through `mcp__duckdb-geo__query` (cluster DuckDB, internal S3). Each returned in seconds —
no ingest, no cluster job. Run date 2026-06-29. Layers used (all already H3-hexed):

- `s3://public-iucn/hex/combined_sr/...`      — vertebrate species richness (paper sub-index i), res 5
- `s3://public-iucn/hex/combined_thr_sr/...`  — threatened-species richness (sub-index ii), res 5
- `s3://public-iucn/hex/combined_rwr/...`     — range-weighted (endemic) richness (sub-index iii), res 5
- `s3://public-carbon/irrecoverable-carbon-2024/hex/...` — irrecoverable carbon total Mg C, res 9 (has h5)
- `s3://public-wdpa/wdpa/hex/...`             — protected areas with `IUCN_CAT`, res 8/9

**Method note.** The paper normalizes each sub-index 0–1 *within each country* and takes the per-pixel
`MAX`. These POCs normalize **globally** (min-max) and work at the hex level (no country join), which is
enough to probe sensitivity. The global-normalization choice is itself instructive — see POC 1.

---

## POC 1 — Layer-combination & combine-rule sensitivity (fixed res 5)

Build the composite four ways, all globally min-max normalized to [0,1]:

| Config | Definition |
|---|---|
| A | richness only |
| B | `MAX`(richness, threatened) |
| C | `MAX`(richness, threatened, range-weighted) — the paper's max-of-normalized rule, 3 sub-indices |
| D | `MEAN`(richness, threatened, range-weighted) — alternative combine rule |

Compared by Pearson correlation of the score and **top-decile hotspot overlap (Jaccard)**.

| Comparison | corr | top-decile Jaccard |
|---|---|---|
| richness-only (A) vs paper-3 (C) | 0.95 | 0.943 |
| max-2 (B) vs paper-3 (C) | 1.00 | 0.999 |
| max (C) vs mean (D) | 0.967 | 0.896 |

**Finding.** Under *global* normalization the composite is dominated by species richness: adding the
threatened and endemic layers barely moves the global hotspot map (Jaccard 0.94–0.999). The **combine rule
(max vs mean) changes more of the top decile (~10%) than adding layers does.** This is exactly the
"swamping by broad richness gradients" the paper cites as its reason for normalizing *within country* and
taking the max — our global-norm POC makes that failure mode visible, and is itself a sensitivity result:
**the normalization scope matters more than the layer menu.** (Next iteration: redo with within-country
normalization via an Overture/country join to see the layers separate.)

<details><summary>SQL</summary>

```sql
WITH base AS (
  SELECT s.h5, s.combined_sr AS sr,
         COALESCE(t.combined_thr_sr,0) AS thr, COALESCE(r.combined_rwr,0) AS rwr
  FROM read_parquet('s3://public-iucn/hex/combined_sr/h0=*/data_0.parquet') s
  LEFT JOIN read_parquet('s3://public-iucn/hex/combined_thr_sr/h0=*/data_0.parquet') t USING(h5)
  LEFT JOIN read_parquet('s3://public-iucn/hex/combined_rwr/h0=*/data_0.parquet')   r USING(h5)
),
mm AS (SELECT MIN(sr) sr0,MAX(sr) sr1,MIN(thr) th0,MAX(thr) th1,MIN(rwr) rw0,MAX(rwr) rw1 FROM base),
cfg AS (
  SELECT (sr-sr0)/(sr1-sr0) AS A,
         greatest((sr-sr0)/(sr1-sr0),(thr-th0)/(th1-th0)) AS B,
         greatest((sr-sr0)/(sr1-sr0),(thr-th0)/(th1-th0),(rwr-rw0)/(rw1-rw0)) AS C,
         ((sr-sr0)/(sr1-sr0)+(thr-th0)/(th1-th0)+(rwr-rw0)/(rw1-rw0))/3.0 AS D
  FROM base, mm),
q AS (SELECT quantile_cont(A,0.9) qA,quantile_cont(B,0.9) qB,quantile_cont(C,0.9) qC,quantile_cont(D,0.9) qD FROM cfg),
f AS (SELECT A,B,C,D, A>=qA tA,B>=qB tB,C>=qC tC,D>=qD tD FROM cfg,q)
SELECT corr(A,C),corr(B,C),corr(C,D),
       SUM((tA AND tC)::INT)::DOUBLE/SUM((tA OR tC)::INT),
       SUM((tB AND tC)::INT)::DOUBLE/SUM((tB OR tC)::INT),
       SUM((tC AND tD)::INT)::DOUBLE/SUM((tC OR tD)::INT)
FROM f;
```
</details>

---

## POC 2 — Resolution (parcel-size) sensitivity

Recompute composite C natively at H3 res 5, 4, 3 (coarser = larger decision unit; sub-indices rolled up
with `MAX`, the hotspot-preserving reducer). Ask: of the fine-scale (res-5) top-decile hotspots, what
fraction remain top-decile when the parcel is coarsened?

| Resolution | cells | fraction of res-5 hotspots still top-decile |
|---|---|---|
| res 5 (native, ~8.5 km cell) | 198,596 hotspot of 1.98M | — |
| res 4 (~22 km) | 283,241 | **0.941** |
| res 3 (~59 km) | 40,550 | **0.854** |

**Finding.** Coarsening the parcel progressively loses fine-scale hotspots — ~6% gone at res 4, ~15% at
res 3. The paper's 8 000-ha parcel (≈ res 5–6) sits right where this loss is still small; coarser decision
units would meaningfully blur the conservation map. Resolution is a real, quantifiable knob.

---

## POC 3 — Biodiversity × carbon co-benefits outside protected areas (res 5)

Join the biodiversity composite (C), irrecoverable-carbon total per cell (`SUM(carbon)` rolled to res 5),
and IUCN cat I–IV protection (WDPA `h8` → res-5 parent via `h3_cell_to_parent`). Top decile = top 10%.

| Metric | Value |
|---|---|
| Land cells (with both biodiversity & carbon) | 1,980,600 |
| Cells overlapping an IUCN I–IV PA | 7.7% |
| Top-decile **biodiversity** cells that are **unprotected** | **81.6%** |
| Top-decile **carbon** cells that are **unprotected** | **69.9%** |
| Cells top-decile in **both** bio & carbon | 98,394 |
| …of those, **unprotected** | **76.2%** |

**Finding.** Most high-value land is outside strict protection: ~82% of biodiversity hotspots, ~70% of
irrecoverable-carbon hotspots, and ~76% of the ~98k cells that are hotspots for *both* objectives sit
outside IUCN I–IV areas. This is the paper's central tension (large unrealized conservation gains on
unprotected land) reproduced as a single SQL query over existing catalog layers, with the exact
protected-area constraint the paper uses.

<details><summary>SQL</summary>

```sql
WITH bio AS (
  SELECT s.h5, s.combined_sr AS sr, COALESCE(t.combined_thr_sr,0) thr, COALESCE(r.combined_rwr,0) rwr
  FROM read_parquet('s3://public-iucn/hex/combined_sr/h0=*/data_0.parquet') s
  LEFT JOIN read_parquet('s3://public-iucn/hex/combined_thr_sr/h0=*/data_0.parquet') t USING(h5)
  LEFT JOIN read_parquet('s3://public-iucn/hex/combined_rwr/h0=*/data_0.parquet')   r USING(h5)),
mm AS (SELECT MIN(sr) a0,MAX(sr) a1,MIN(thr) b0,MAX(thr) b1,MIN(rwr) c0,MAX(rwr) c1 FROM bio),
biocomp AS (SELECT h5, greatest((sr-a0)/(a1-a0),(thr-b0)/(b1-b0),(rwr-c0)/(c1-c0)) bioC FROM bio,mm),
carb AS (SELECT h5, SUM(carbon) carbon FROM read_parquet('s3://public-carbon/irrecoverable-carbon-2024/hex/h0=*/data_0.parquet') GROUP BY h5),
pa AS (SELECT DISTINCT h3_cell_to_parent(h8,5) h5 FROM read_parquet('s3://public-wdpa/wdpa/hex/h0=*/data_0.parquet') WHERE IUCN_CAT IN ('Ia','Ib','II','III','IV')),
j AS (SELECT b.h5,b.bioC,c.carbon,(p.h5 IS NOT NULL) protected
      FROM biocomp b JOIN carb c USING(h5) LEFT JOIN pa p USING(h5)),
q AS (SELECT quantile_cont(bioC,0.9) qb, quantile_cont(carbon,0.9) qc FROM j)
SELECT 100.0*AVG(protected::INT),
       100.0*AVG(CASE WHEN bioC>=qb THEN (NOT protected)::INT END),
       100.0*AVG(CASE WHEN carbon>=qc THEN (NOT protected)::INT END),
       100.0*AVG(CASE WHEN bioC>=qb AND carbon>=qc THEN (NOT protected)::INT END)
FROM j,q;
```
</details>

---

## POC 4 — Within-country normalization (the paper's actual method)

POC 1 normalized globally and found the layer menu barely mattered. The paper instead normalizes each
sub-index **within each country**. Repeating the sensitivity test with within-country min-max normalization
(each res-5 cell assigned its majority country via the Overture countries hex, `h3_cell_to_parent(h8,5)`):

| Comparison | corr | top-decile Jaccard | (global-norm POC 1 Jaccard) |
|---|---|---|---|
| richness-only vs paper-3 (max) | 0.948 | **0.759** | 0.943 |
| max vs mean combine rule | 0.934 | **0.523** | 0.896 |

597,922 land cells across 222 countries.

**Finding.** Within-country normalization makes the result **much more sensitive** to the methodological
choices: the layer menu now swings ~24% of the top-decile hotspots (vs ~6% globally), and the combine rule
(max vs mean) swings ~48% (vs ~10% globally). Under global normalization the tropical richness gradient
swamps everything so nothing else moves the map; within-country, *which layers* and *how they combine*
genuinely determine which places rank highest. This both **confirms the paper's rationale** for
within-country max-of-normalized and shows the parametric probe is sensitive enough to be worth running.

<details><summary>SQL</summary>

```sql
WITH cc AS (SELECT h3_cell_to_parent(h8,5) AS h5, country
            FROM read_parquet('s3://public-overturemaps/2026-02-18.0/countries/hex/h0=*/data_0.parquet')),
cell_country AS (SELECT h5, arg_max(country,cnt) country
                 FROM (SELECT h5,country,COUNT(*) cnt FROM cc GROUP BY 1,2) GROUP BY h5),
bio AS (SELECT s.h5, s.combined_sr sr, COALESCE(t.combined_thr_sr,0) thr, COALESCE(r.combined_rwr,0) rwr
        FROM read_parquet('s3://public-iucn/hex/combined_sr/h0=*/data_0.parquet') s
        LEFT JOIN read_parquet('s3://public-iucn/hex/combined_thr_sr/h0=*/data_0.parquet') t USING(h5)
        LEFT JOIN read_parquet('s3://public-iucn/hex/combined_rwr/h0=*/data_0.parquet') r USING(h5)),
j AS (SELECT b.*, c.country FROM bio b JOIN cell_country c USING(h5)),
norm AS (SELECT h5,country,
   COALESCE((sr -MIN(sr) OVER w)/NULLIF(MAX(sr) OVER w-MIN(sr) OVER w,0),0) nsr,
   COALESCE((thr-MIN(thr)OVER w)/NULLIF(MAX(thr)OVER w-MIN(thr)OVER w,0),0) nthr,
   COALESCE((rwr-MIN(rwr)OVER w)/NULLIF(MAX(rwr)OVER w-MIN(rwr)OVER w,0),0) nrwr
   FROM j WINDOW w AS (PARTITION BY country)),
cfg AS (SELECT nsr A, greatest(nsr,nthr,nrwr) C, (nsr+nthr+nrwr)/3.0 D FROM norm),
q AS (SELECT quantile_cont(A,0.9) qA,quantile_cont(C,0.9) qC,quantile_cont(D,0.9) qD FROM cfg),
f AS (SELECT A,C,D,A>=qA tA,C>=qC tC,D>=qD tD FROM cfg,q)
SELECT corr(A,C),corr(C,D),
       SUM((tA AND tC)::INT)::DOUBLE/SUM((tA OR tC)::INT),
       SUM((tC AND tD)::INT)::DOUBLE/SUM((tC OR tD)::INT) FROM f;
```
</details>

---

## POC 5 — The third objective: biodiversity × carbon × net economic value

Ingested the CC0 Dryad economic rasters (5-arcmin / ~10 km — *not* 300 m; matches our res-5 grid).
Current production value per cell = `greatest(crop_current, palm_current, grazing, 0)` USD/ha (point-binned
to H3 res 5 via the MCP, nodata/sentinels excluded). Joined to the biodiversity composite (POC 1 config C)
and irrecoverable carbon, over 505,893 land cells:

| Objective pair | Pearson corr | High-in-both (top-decile overlap) |
|---|---|---|
| biodiversity × economic | 0.29 | 2.1% (hard-conflict cells) |
| **carbon × economic** | **−0.02** | **0.6%** |
| biodiversity × carbon | 0.34 | (co-benefit) |

Plus: **10.2%** of land is high-conservation-value, low-economic-value, *and* unprotected — low-opportunity-cost
restoration/protection wins. **Finding:** irrecoverable carbon and economic value are spatially
near-independent (corr ≈ 0), so protecting carbon costs almost no production — the paper's headline that
large climate gains are achievable without economic loss, reproduced directly.

## POC 6 — The separable optimization (Pareto frontier)

The paper's optimizer = weighted-sum scalarization with per-parcel argmax (no coupling constraints; §5 of
`assessment.md`). Demonstrated with a 2-alternative choice per cell — **natural** (keep bio + carbon,
econ = 0) vs **production** (gain econ, lose bio + carbon) — sweeping objective weights and summing the
chosen cells. Objectives min-max normalized; totals as % of each objective's achievable maximum:

| weighting (bio/carbon/econ) | % land kept natural | biodiversity | carbon | economic value |
|---|---|---|---|---|
| conservation (1/0/0, 0/1/0, .5/.5/0) | 100% | 100% | 100% | 0% |
| conservation-lean (.4/.4/.2) | 99.9% | 99.9% | 100% | 0.9% |
| **balanced (.34/.33/.33)** | 98.1% | 98.3% | 99.7% | **11.6%** |
| **economic-lean (.1/.1/.8)** | 63.8% | 61.2% | 75.7% | **85.1%** |
| all-economic (0/0/1) | 1.7% | 0.8% | 0.3% | 100% |

**Finding.** The frontier is sharply non-linear: because high-economic and high-conservation cells barely
overlap, the economic-lean solution **captures 85% of producible value while retaining 61% of biodiversity
and 76% of carbon** — converting only the ~36% of land that is low conservation value. Large simultaneous
gains across all three objectives are feasible — the paper's central result, reproduced as `arg_max`
over weight vectors in pure DuckDB.

**POC caveats (honest scope).** Economic surface = best of current crop/palm/grazing only (forestry +
14 transition-cost surfaces not yet folded in); 5-arcmin layers point-binned (not exact-extract) to res 5;
2 alternatives, not the paper's 13; global (not within-country) normalization; conversion assumed to zero
out bio + carbon. These are POC simplifications — the faithful version uses `raster-workflow` exact-extract
hexing of all layers (the data-workflows ingest recipe (`catalog/nci-frontiers/`)) and the 13-alternative value tables (or the authors' Julia code).
The point: the full three-objective frontier machinery runs end-to-end on our stack **today**.

## What this establishes for the broader effort

- The parametric workflow the report proposes is **real and fast**: vary layers / combine-rule / resolution = edit a `SELECT`; each probe is a sub-minute MCP query, no ingest.
- The biodiversity and climate objectives are fully probeable **today** from the catalog.
- Two natural next steps: (1) **within-country normalization** (join an Overture/GADM country layer) to reproduce the paper's actual normalization and let the sub-indices separate; (2) **ingest the CC0 Dryad economic rasters** to add the third objective and run the full separable optimization (see `assessment.md` §5–6).
