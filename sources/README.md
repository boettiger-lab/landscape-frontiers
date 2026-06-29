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
