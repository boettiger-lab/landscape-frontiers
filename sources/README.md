# Sources

The **article PDFs are AAAS-copyrighted** and are deliberately **not** committed to this public repo
(only the Dryad *data* is CC0, not the article). Retrieve them from the publisher:

## Paper
- **Polasky, S. et al. (2026)** "Landscape efficiency frontiers for biodiversity, climate mitigation, and net economic value." *Science* 392, 1069. **doi:[10.1126/science.aea9058](https://doi.org/10.1126/science.aea9058)**
  - Main text, Supplementary Materials, and Reproducibility Checklist available from the Science article page (subscription/institutional access).

## Data, code, results — Dryad (CC0-1.0, freely reusable)
- **Input data:** doi:[10.5061/dryad.qjq2bvqw5](https://doi.org/10.5061/dryad.qjq2bvqw5) — `inputs_for_publication.zip` (8.5 GB): per-pixel economic / biodiversity / carbon / scenario GeoTIFFs.
- **Results:** doi:[10.5061/dryad.b2rbnzsvt](https://doi.org/10.5061/dryad.b2rbnzsvt) — per-country Pareto frontiers (`solutions.h5`).
- **Code:** doi:[10.5061/dryad.9s4mw6mxn](https://doi.org/10.5061/dryad.9s4mw6mxn) — Python `natures_frontiers` + Julia `NaturesFrontiers.jl` (the weighted-sum optimizer).

Download past Dryad's Anubis bot wall: the solver script is in `data-workflows` (`catalog/nci-frontiers/scripts/`).

## Captured CC0 materials (small — committed here for easy resume)

These are the small, redistributable (CC0-1.0) pieces we actually use, captured so a fresh session
needn't re-download them:

- **`cc0-code/`** — the full `natures_frontiers` (Python) + `NaturesFrontiers.jl` codebase from the code
  deposit. Key files for the biodiversity reproduction: `natures_frontiers/wbnci/biodiversity.py` (the
  6-sub-index model), `data_prep_scripts/speciesRichness.py` (per-taxon pool rasterization), `scenario_creation.py`.
- **`cc0-inputs/`** — `predicts_July_2022.csv` (LULC→class/intensity crosswalk), `predicts2_July_2022.csv`
  (relative richness/abundance response), `carbon_table_2022-08-10.csv` (zone×land-use carbon density), and
  the code/results deposit READMEs + `country_batches.csv`.
- **`cc0-validation/`** — per-country results summaries (`Brazil_value_table_summary.csv`,
  `Brazil_merged_summary_table.csv`) used to confirm the transition-cost zeros and as a stage-3 validation fixture.

## NOT captured (too large — re-download via the Anubis scripts + DOIs above)

- The 8.5 GB input `inputs_for_publication.zip` and the per-country **result** zips (7–10 GB each) from the
  results deposit — including each country's `InputRasters/` (GlobalLayers `band1/14/24`, `ecoMaps`, `flii`,
  `plantation`) and `ScenarioMaps/` (the 13 scenario LULC maps). Stage-3 reproduction re-fetches the needed
  per-country packages with `data-workflows/catalog/nci-frontiers/scripts/anubis_fetch.py`.

## Issue tracking

- **App-side tracker:** this repo's issues (canonical entry point).
- **Data/build tracker:** `boettiger-lab/data-workflows#334` — the full checkpoint trail, locked scope, and
  reproduction plan. The biodiversity reproduction spec lives in `research/assessment.md`.
