# landscape-frontiers — geo-agent app

Interactive explorer for the **biodiversity · climate · economy trade-offs** of Polasky et al. 2026
(*Science*, "Landscape efficiency frontiers"). Draw an area or name a country → see where it sits in the
three-objective space and how far it is from its efficiency frontier. Built on the
[`geo-agent`](https://github.com/boettiger-lab/geo-agent) runtime (fork of `geo-agent-template`).

## Files
- `index.html` — HTML shell, loads the geo-agent runtime + CSS from CDN (pinned `@v3.17.1`).
- `layers-input.json` — collections (carbon, ecoregions, WDPA, Overture admin), map view, `draw_enabled`, welcome examples. The objective surfaces (biodiversity composite, economic value, the optimal-land-use solution) are **computed on demand** via the duckdb-geo MCP and rendered with `register_hex_tiles` — not static toggles (the `nci-frontiers` economic layers are hex-parquet, no PMTiles).
- `system-prompt.md` — frontier framing + the canned SQL recipes (AOI scorecard, weight-sweep frontier, co-benefit finder, recolor-by-solution) + aggregation rules.

## Status / dependencies
- Depends on the `nci-frontiers` hex layers (data-workflows #335) — economic revenue (res 5), forestry/FLII + transition costs (res 8). **First-pass hexing in progress**; the app's context layers (carbon, ecoregions, WDPA) and the biodiversity/carbon objectives work today; the economic axis lights up as those layers publish.
- Fidelity: the live frontier is a **2-alternative** (natural vs best-production) approximation; the faithful 13-alternative frontier needs the full transition-cost set (pass-2 ingest) — same separable SQL, more alternatives.

## Roadmap (see DESIGN.md)
- **Charting** (frontier curve + "you are here"): tracked upstream at geo-agent (opt-in generic chart primitive). Until then, frontier is delivered as a table + map recolor.
- **Live weight slider** (drag bio↔carbon↔economy → re-solve + recolor live): best as a generic geo-agent *reactive-parameter* control (relates to geo-agent #147 temporal slider); a frontier-specific extension can bolt on in the interim.

## Deploy

Two deployment targets, both supported from this repo:

### Option A — GitHub Pages (bring-your-own LLM key)
Static hosting, no server. The `llm` block in `layers-input.json` (`user_provided: true`,
OpenRouter default) means each visitor enters **their own API key** in the in-app settings panel — stored
in the browser only, never on a server. The duckdb-geo MCP it queries is the public NRP endpoint.
- `.github/workflows/gh-pages.yml` deploys the repo to Pages on every push to `main`.
- One-time: **Settings → Pages → Source → GitHub Actions**. Live at `https://boettiger-lab.github.io/landscape-frontiers/`.

### Option B — NRP / Kubernetes (lab LLM proxy, no user key)
`k8s/` serves the static files via nginx and reverse-proxies `/api/llm` to the lab LLM proxy (the
`open-llm-proxy-secrets` key lives only in the nginx config, never in the browser); `config.json` is
injected at boot from `k8s/configmap.yaml` and overrides the BYO `llm` block. Live at
`https://landscape-frontiers.nrp-nautilus.io`.
```bash
kubectl apply -f k8s/      # configmap + deployment + service + ingress (namespace: biodiversity)
```

## Local preview
```bash
python -m http.server 8000   # open http://localhost:8000, enter an API key in settings
```
