# Research & provenance

The analysis behind this app — can we reproduce Polasky et al. 2026 with cloud-native H3 layers, and what
do the trade-offs look like?

- **`assessment.md`** — feasibility assessment: data crosswalk (paper inputs → our STAC catalog), the CC0 Dryad release, the H3-parcel methodological fit, why the optimization is *not* MILP (separable weighted-sum argmax), and recommendations.
- **`poc-results.md`** — six runnable proof-of-concept results over the duckdb-geo MCP: layer-combination, combine-rule, and resolution sensitivity of the biodiversity composite; within-country vs global normalization; the biodiversity × carbon × economic three-objective trade-off; and a separable Pareto-frontier sweep.

See also `../DESIGN.md` (interactive-platform design + roadmap) and `../README.md` (the app).

## Data generation lives in `data-workflows`

The actual ingest of the CC0 Dryad layers into H3 hex (`public-nci-frontiers`) is owned by
[`boettiger-lab/data-workflows`](https://github.com/boettiger-lab/data-workflows) per its dataset-import
process — recipes, staging jobs, STAC, and the `cng-datasets` pipeline (issues #334/#335). This repo
consumes those published layers; it does not generate them.

## Sources
Paper + CC0 data DOIs in `sources/README.md` (the article PDFs are AAAS-copyrighted and intentionally not committed here).
