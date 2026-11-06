# pdb-browser — Requirements

> **Status:** Draft v0.1 (2026-11-06). Living document: update it as decisions land,
> and move settled "why" discussions to a `context-archive.md` (same convention as
> the `plutometer` repo).

## 1. Mission

An interactive, free-to-host browser for the **U.S. Census Planning Database (PDB)**
at **tract** and **block-group** level. It lets a user:

1. **Browse.** Find any tract or block group by map click, address, or name, and read all
   ~500 PDB variables in an organized, documented profile.
2. **Join.** Attach supplemental data (municipal, state, federal, private) to the
   same geographies through a small, repeatable join layer: point-in-polygon,
   areal interpolation, and crosswalks.
3. **Compare.** Build a full **multi-source profile** of each geography and find
   **similar places**, nationally or within a region, on a user-chosen feature set.

Audiences:
- **Analysts and advocates** who want a "what is this neighborhood like, and what
  is it like?" answer without writing code.
- **Municipal staff and researchers** (for example, the balt311 use case) who need a
  clean tract-level covariate table to join to their own operational data.
- **Developers.** The processed outputs are a reusable, documented data product
  (Parquet and PMTiles), not only a UI.

## 2. Lessons carried over from the reference repos

| From | What we reuse | What we avoid |
|---|---|---|
| `balt311-service-equity` | PDB `GEOID` joins byte-for-byte to TIGER `GEOID` with no reformatting. Point→tract spatial join for municipal event data (311). Population-weighted rollup to custom regions (CSA via BNIA crosswalk). Overlap scores and PCA/RobustScaler embeddings for "where does this area sit". Staged headless pipeline (`--stage`). Per-city adapter pattern (ArcGIS/Socrata/Carto/CKAN). Small-cell suppression (n<5). | Recomputing heavy analytics on every Streamlit rerun (it took a multipage `st.navigation` refactor to fix). ACS-vintage drift between sources (PDB 2024 = ACS 2018–22). |
| `plutometer` | National tract coverage. Geometry/attribute split (`tract.fgb` geometry+GEOID only, `tract_values.parquet` attributes). Viewport bbox reads via `pyogrio` to stay under ~1 GB RAM. Precomputed adjacency (queen contiguity). The Connecticut planning-region GEOID crosswalk. A data-integrity test that no state is all-zero. | `st_folium` remount/zoom feedback loops (`known-issues.md`): any map whose Python state rebuilds the map on pan/zoom. Here, map interaction should stay client-side, in JS. |

## 3. Core data: Census Planning Database

### 3.1 Vintages
- **Required:** latest PDB at time of build. **Target the 2026 PDB.** Its Census
  API dataset page confirms 2020–2024 ACS 5-year estimates, tract self-response
  rates, 2020 Census operational variables and the LRS (endpoint
  `api.census.gov/data/2026/pdb/tract`). The 2025 PDB (2019–2023 ACS) is the
  fallback.
  - The CSV filename (`pdb2026tr.csv`?) is unconfirmed until the Phase 0 spike.
  - **Vintage label ≠ release year ≠ ACS period:** the 2024 PDB (2018–22 ACS) was
    only released 2026-01-22. The pipeline must take `--vintage` and read the ACS
    period and variable suffixes (`_ACS_20_24`, …) from metadata, never hard-coding
    them.
  - Verify the row and column count of each download. Example: the 2023 tract file
    should be 85,395 rows × 524 columns.
  - See `verification_log.md` V-03.
- **Should:** keep at least two vintages so users can see changes over time. Variable names change
  suffix per vintage, so the pipeline normalizes to a **stable variable key**
  (`med_hhd_inc_acs`) plus a `vintage` and `acs_period` column.
- **Won't (v1):** 2010-geography PDBs. These are the vintages built on 2015–19 or
  earlier ACS, i.e. through the 2021 PDB; the ACS switched to 2020 tracts with
  2016–20, per V-01. They need the 2010→2020 tract relationship file; defer to a
  later phase.

### 3.2 Sources
| File | Grain | Approx size |
|---|---|---|
| `https://www2.census.gov/adrm/PDB/{V}/pdb{V}tr.csv` | Tract, ~85k rows × ~530 cols | ~190 MB |
| `https://www2.census.gov/adrm/PDB/{V}/pdb{V}bg.csv` | Block group, ~242k rows × ~385 cols | ~500 MB |
| `https://api.census.gov/data/{V}/pdb/{tract,blockgroup}` | API (key optional for low volume) | — |
| PDB documentation PDF + API `variables.json` | Data dictionary | — |

### 3.3 Requirements
- **R-PDB-1.** Ingest the tract and BG CSVs as `dtype=str` first, then coerce numerics
  explicitly. Preserve leading zeros in `GEOID`, `State`, `County`, `Tract`.
- **R-PDB-2.** Build a **data dictionary** table (`variables.parquet`) from the API
  `variables.json` and documentation. It holds variable key, label, topic group,
  source (ACS/Census/derived), unit (count / percent / dollars / score),
  `has_moe`, and a denominator variable for counts. Every UI label comes from here.
- **R-PDB-3.** Pair each count with its percent, and each estimate with its MOE where the PDB
  provides one. **Unverified:** whether the PDB ships a MOE column for every ACS
  variable (its documentation has an MOE-calculation appendix). If not, fetch
  MOEs for clustering variables from the ACS detailed-table API. This is a Phase 0
  spike check (V-14). Surface the **coefficient of variation** and flag
  unreliable estimates (CV > 30% amber, > 50% red) in the profile view.
- **R-PDB-4.** Topic groups for browsing: Population & age; Race/ethnicity;
  Language & nativity; Education; Income & poverty; Employment; Housing &
  tenure; Households & families; Health insurance; Internet/computer access;
  Mobility; Census operations (mail return, self-response, LRS).
- **R-PDB-5.** Apply the plutometer **Connecticut fix** wherever a non-PDB source uses
  a different CT county scheme (old FIPS 09001–09015 vs planning regions
  09110–09190). Add a test asserting no state is entirely null for a core variable.
- **R-PDB-6.** Suppress or grey out geographies with tiny populations (<100 people or
  <50 households: water tracts, institutional tracts). Never drop them silently.

## 4. Geography layer

- **R-GEO-1.** Boundaries: Census **cartographic boundary** files (`cb_{yr}_us_tract_500k`,
  `cb_{yr}_{st}_bg_500k`) for display. Use TIGER/Line full-resolution files only for spatial
  joins where shoreline precision matters.
- **R-GEO-2.** Store geometry once and attributes separately (the plutometer split):
  - `geo/tract.fgb`, `geo/bg/{state}.fgb`: GEOID + geometry (FlatGeobuf, spatially indexed)
  - `tiles/tract.pmtiles`, `tiles/bg.pmtiles`: vector tiles for the browser, GEOID
    plus a **small** set of styling attributes. Full attributes are fetched on demand.
- **R-GEO-3.** Hierarchy columns on every row: state, county, county subdivision,
  place (principal city), CBSA/metro, congressional district, and ZCTA (by
  majority-area relationship file). These drive the "filter to my city/region"
  scope and the crosswalk joins below.
- **R-GEO-4.** Precompute **queen adjacency** for tracts and BGs (plutometer
  `STRtree` approach) for spatial-lag features, contiguous-region tools, and
  "neighbors of" panels. Build it from full-precision geometry, not simplified (see
  plutometer's documented false-negative issue).
- **R-GEO-5.** **Blocks.** The PDB stops at block group. Block-level detail is
  limited to 2020 Decennial P.L. 94-171 / DHC counts (population, race, housing
  units, occupancy). v1 supports block data only as an **optional drill-down**
  (counts only, single county at a time) and as an **interpolation weight**
  for areal joins (§5.3). Block-level similarity analysis is out of scope
  for v1: ACS is not published there and the noise would swamp the signal.

## 5. Supplemental data join framework

The key extensibility requirement: adding a dataset should mean writing a small
**source spec** (YAML or Python adapter), not editing the app.

### 5.1 Source spec (one per dataset)
```yaml
id: hud_chas_cost_burden
title: Housing cost burden (HUD CHAS)
tier: federal            # municipal | state | federal | private
license: public-domain   # SPDX id or "proprietary-display-only"
vintage: 2017-2021
native_geography: tract  # tract | bg | block | point | polygon | county | zcta | place
join: direct             # direct | point_in_polygon | areal_interpolation | crosswalk | broadcast
geoid_vintage: 2020      # 2010 inputs trigger the 2010→2020 relationship-file crosswalk
fetch: {type: http_csv, url: ...}   # http_csv | arcgis | socrata | ckan | carto | census_api | bigquery | s3_parquet
variables:
  - {key: cost_burden_pct, label: "% households cost-burdened", unit: percent, higher_is: worse}
coverage: national       # national | state:24 | place:2404000 ...
refresh: annual
```

### 5.2 Join types (all must be implemented in a shared `joins.py`)
| Type | Use for | Method |
|---|---|---|
| `direct` | Already keyed by 2020 tract/BG GEOID | Left join on GEOID. Report the match rate. |
| `point_in_polygon` | Events/facilities: 311, crime, permits, code violations, schools, clinics | `geopandas.sjoin(predicate="within")`, then aggregate (count, rate per 1k pop, median, share by category). Reuse balt311's ingest-then-aggregate pattern and n<5 suppression. |
| `areal_interpolation` | Polygons not aligned to tracts: school districts, flood zones, zoning, parks, BNIA CSAs, council districts | Area-weighted by default. **Population-weighted via 2020 blocks** when the variable is people-based (dasymetric). Use `tobler` or an equivalent. |
| `crosswalk` | ZIP/ZCTA, county, place, 2010 tracts | HUD USPS ZIP crosswalk (residential ratio; free with registration / API key; 2020 tracts from 2023 Q1 onward). Census 2010→2020 tract relationship file. Population-weighted. |
| `broadcast` | County/state/metro-level values (unemployment rate, state policy flags) | Copy the parent value to every child tract, marked `granularity=county` in the UI. |

- **R-JOIN-1.** Every join writes a **join report**: rows in, rows matched, unmatched
  sample, % of tracts covered, and method. The UI shows coverage next to each variable.
- **R-JOIN-2.** Every joined variable carries provenance (`source_id`, `vintage`,
  `join`, `native_geography`). The profile view shows it on hover.
- **R-JOIN-3.** Joined outputs are written as `joined/{source_id}.parquet`
  (GEOID × variables). They are never merged into one mega-table on disk. The
  app assembles columns lazily (DuckDB).
- **R-JOIN-4.** **Bring your own data.** In the browser, a user can upload a CSV
  with a GEOID column, or lat/lon points (≤ ~200k rows), and join it client-side for
  this session only. Nothing is uploaded to a server.
- **R-JOIN-5.** Licensing gate: `proprietary-display-only` sources are never
  written into downloadable exports or committed tiles.

### 5.3 Candidate supplemental sources (prioritized)

**Federal (P1, national, free)**
| Source | Geography | Adds |
|---|---|---|
| ACS 5-year API (newer than the PDB's vintage) | tract/BG | Fresher estimates and any table the PDB omits (e.g. B25070 rent burden, B08303 commute time) |
| CDC **PLACES** (2025 release confirmed available: 40 measures, 2023 BRFSS) | tract | Model-based health outcomes (diabetes, asthma, obesity, depression, …) |
| CDC/ATSDR **Social Vulnerability Index** | tract | SVI themes and percentiles |
| FEMA **National Risk Index** | tract | Hazard risk, expected annual loss, resilience |
| EPA **EJScreen** (**removed by EPA 2025-02-05**; use the Public Environmental Data Partners reconstruction or another archived copy, labeled historical) / CEQ CEJST | tract/BG | Environmental burden (PM2.5, diesel, proximity to hazards) |
| USDA **Food Access Research Atlas** | tract | Low-income / low-access flags |
| HUD **CHAS**, **LIHTC**, **Opportunity Zones**, **Qualified Census Tracts** | tract / point | Housing burden, subsidized housing, designations |
| HMDA (CFPB) | tract | Mortgage applications, denials, by race |
| FHFA annual HPI (**developmental**; research release on 2010 tracts, ~63k tracts, 1989–2021) | tract → crosswalk | Repeat-sales house price change |
| BLS LAUS / **LEHD LODES 8** (2020 blocks; 2002–2022 for most states, with some state gaps) | county / block→tract | Unemployment. Jobs by workplace/residence, commute flows. |
| FCC **Broadband Data Collection** | location→block→tract | Served / unserved share |
| EPA **Smart Location Database** / **Walkability Index** | BG | Density, transit access, walkability |
| NCES **EDGE** school locations, district boundaries | point / polygon | Schools, district joins |
| Opportunity Insights **Opportunity Atlas** | tract | Intergenerational mobility outcomes |
| HRSA HPSA / MUA designations | polygon | Health-professional shortage areas |

**State (P2, adapter per state, starting with MD)**
- State open-data portals (Socrata/ArcGIS): school performance, licensed facilities,
  crash data, environmental permits. MD iMAP is the first target (balt311 home state).
- State legislative districts (Census SLDU/SLDL) via `areal_interpolation`.

**Municipal (P2, adapter per city, reusing balt311 adapters)**
- 311 service requests (balt311 `cities/` adapters: ArcGIS, Socrata, Carto, CKAN;
  15-city cohort) → point-in-polygon → requests per 1k, median days to close, top type.
- Crime incidents, building permits, code violations, vacant-property registries,
  tree canopy, parcels/assessments (aggregate to tract medians).
- Custom planning geographies (Baltimore CSA/NSA via BNIA crosswalk) as an
  alternate **roll-up** level, population-weighted like balt311.

**Private / nonprofit (P3, license check required for each)**
- Zillow ZHVI / ZORI (ZIP→tract via HUD crosswalk; display-only per terms).
- Eviction Lab (tract), Urban Institute data catalog, PolicyMap-style indicators
  (licensed, display-only), Overture Maps / OpenStreetMap POIs (amenity counts
  per tract: grocery, pharmacy, transit stops; ODbL/CDLA attribution).
- Walk Score and similar: only if the license permits caching; otherwise skip.

## 6. Tract profile and similarity analysis

> Method details, prior art, and the validation battery are in
> [`methodology.md`](methodology.md). Where they conflict, methodology.md supersedes
> the specifics below.

### 6.1 Profile
- **R-PROF-1.** A profile page or panel per geography: header (name, county, place,
  population, area, density), then topic sections. Each variable shows its
  **value**, its **national, state, and metro percentile**, a small distribution strip
  (where this place falls), its MOE/CV badge, and its source.
- **R-PROF-2.** "Neighbors" strip: adjacent tracts' values for the same variables.
- **R-PROF-3.** Shareable URL per geography and view (`?geoid=24510160100&vars=…`).
- **R-PROF-4.** Export the profile as CSV and as a printable HTML/PDF one-pager.

### 6.2 Feature store
- **R-FEAT-1.** A versioned **feature matrix** `features/{level}_{vintage}.parquet`:
  GEOID × curated features (~40–80 at launch, a subset of PDB plus P1 federal sources).
  Each feature has a transform recipe:
  - shares/rates: as-is (0–1). Compositions (race, tenure, age bands): CLR or
    ILR transform (balt311 used CLR for service-mix shares).
  - skewed counts and dollars: log1p, then **RobustScaler** (balt311 demographic PCA
    precedent).
  - all: winsorize at p1/p99. Missing values: impute with the county median and add a
    missing-indicator flag. Drop features with >20% missing in scope.
- **R-FEAT-2.** Feature **presets** (formalized as facets in methodology.md §3.7): "Demographic" (race, age, nativity,
  language), "Economic" (income, poverty, employment, education), "Housing"
  (tenure, value, rent, burden, vacancy, age of stock), "Health & environment"
  (PLACES, EJ, NRI), "Census operations" (LRS, response), plus "Custom" (pick
  any variables).

### 6.3 Similarity
- **R-SIM-1.** **Find similar.** Given a geography, a feature preset or custom set, and a
  scope (same metro / same state / national / exclude same county), return the
  top-k most similar geographies. Metric: cosine or Euclidean in standardized
  space (user-selectable), with optional per-feature weights.
- **R-SIM-2.** **Explain similarity.** For each match, show the features with the
  smallest and largest standardized differences ("similar because…, differs
  in…"). This matters more than the score itself.
- **R-SIM-3.** **Embedding map.** A 2-D projection (PCA by default; UMAP optional,
  precomputed) of all geographies in scope, colored by any variable, with the
  selected tract and its matches highlighted. This is a generalization of balt311's
  Areas tab (shared coordinate space, quadrant labels, labeled exemplars).
- **R-SIM-4.** **Typology clusters.** Precomputed k-means / Gaussian mixture
  typologies (k ≈ 8–12 nationally) per preset, with auto-generated cluster
  descriptions (top distinguishing features vs. national mean). Display them as a
  categorical map layer.
- **R-SIM-5.** **Spatially-aware option.** Add spatial-lag features (mean of
  queen neighbors) so "similar" can mean "similar place in a similar
  surrounding". Optionally add regionalization (contiguous clusters, e.g.
  `spopt` max-p / SKATER) for custom neighborhood building.
- **R-SIM-6.** **Reliability weighting.** Optionally down-weight features with high CV
  for the source tract. A similarity built on ±60% MOEs should say so.
- **R-SIM-7.** Performance: national tract kNN (85k × ≤100 dims) must return in
  <1 s. Precompute a nearest-neighbor index per preset (`pynndescent` / sklearn
  BallTree, or exact brute force in-browser via DuckDB-WASM / typed arrays,
  which is fast enough at 85k × 50). Custom feature sets compute on demand within
  the active scope.

### 6.4 Equity and comparison lenses (from balt311)
- **R-EQ-1.** For any joined outcome variable (e.g. 311 days-to-close, eviction
  rate), compare its distribution across demographic splits (majority-group,
  above/below median income) with the balt311 **overlap score**
  (`1 − 2|P(A>B) − 0.5|`, ≥3 per group) and box/strip charts.
- **R-EQ-2.** Optional mix-adjustment hook: let an outcome be stratified by a
  category column before comparison (balt311 Tab 6 pattern).

## 7. UX requirements

- **R-UX-1. Map-first browse.** Nationwide tract choropleth with zoom-dependent
  level (state/county overview → tract → BG at high zoom). Variable picker grouped
  by topic, with search over the data dictionary. Legend with quantile or
  equal-interval classes, diverging when a variable has a meaningful midpoint.
- **R-UX-2. Search.** Address geocoding (Census Geocoder, free, no key) → tract/BG,
  plus place/county name search.
- **R-UX-3. Table view.** Sortable and filterable table of geographies in the current
  scope (≤ ~5k rows rendered, virtualized), with column chooser and CSV/Parquet export.
- **R-UX-4. Linked views.** Map selection, table selection, embedding scatter,
  and profile stay in sync.
- **R-UX-5. Mobile.** Usable at phone width: map on top, controls below (plutometer
  layout lesson), no horizontal scroll.
- **R-UX-6. Accessibility.** Colorblind-safe palettes (viridis/cividis-style
  sequential, a validated diverging palette), keyboard navigation of the table, and
  text alternatives (the table) for every map.
- **R-UX-7. Caveats inline.** MOE badges, join coverage, vintage mismatch warnings
  ("PDB ACS 2019–23 vs PLACES 2022"), and "model-based estimate" tags for PLACES and
  similar sources.

## 8. Architecture and hosting

### 8.1 Options considered
| Option | Hosting | Map rendering | Strengths | Weaknesses |
|---|---|---|---|---|
| **A. Static site: MapLibre GL + PMTiles + DuckDB-WASM / Parquet** | GitHub Pages or Cloudflare Pages (free); large files on **Cloudflare R2** (free 10 GB-month Standard storage, 1M Class A / 10M Class B ops per month, no egress fees) or Hugging Face Datasets | Client-side vector tiles, GPU. 85k tracts are smooth. | Free, no server, no cold starts, no RAM cap. Pan/zoom never round-trips to Python (avoids the plutometer `st_folium` issues). Scales to BG level nationally. Shareable URLs. | Needs JS. Heavy analytics (custom kNN) runs in-browser. GitHub Pages blocks files ≥100 MB, so tiles and Parquet go to R2/HF. |
| **B. Streamlit (Community Cloud)** | Free, auto-deploy from `main` | `pydeck` (deck.gl) or folium | Python-only, fast to build, matches both reference repos | ~1 GB RAM cap, sleeps when idle, full rerun per interaction, map state loops (plutometer `known-issues.md`). National BG layer is not feasible. |
| **C. Observable Framework** | Static build → GitHub Pages / Cloudflare | MapLibre/deck.gl or Plot | Data loaders run Python at build time and ship Parquet. Reactive JS. Good charts (Observable Plot). | Newer toolchain. Same static-file limits as A. |
| **D. Panel / Shiny for Python via Pyodide (WASM)** | Static | deck.gl / ipyleaflet | Python in browser | Slow first load (pyodide + pandas ~30 MB), weak map performance |
| **E. Hugging Face Spaces (Streamlit/Gradio/Docker)** | Free CPU Space (16 GB RAM) | any | More RAM than Streamlit Cloud. Can co-locate datasets. | Sleeps when idle. Same rerun model if Streamlit. |
| **F. Evidence.dev / Datasette-Lite** | Static | limited | Good for the table/SQL browse | Weak custom maps and similarity UI |

### 8.2 Decision (proposed)
- **Primary: Option A/C hybrid.** A static site built with **Observable
  Framework** (or plain Vite + TypeScript if Framework proves limiting), using:
  - **MapLibre GL JS** + `pmtiles` protocol. Free basemap: OpenFreeMap or
    Protomaps basemap PMTiles, so no Mapbox token is needed.
  - **DuckDB-WASM** querying remote **Parquet** over HTTP range requests (feature
    matrix, joined sources, dictionary), with row-group pruning by state.
  - **Observable Plot** (or Vega-Lite) for distributions, strips, and scatter.
  - Pre-built tiles, Parquet, and kNN indexes from a Python pipeline in GitHub Actions.
- **Secondary: Streamlit "analyst workbench"** (Option B or E) for heavy or
  experimental work: custom regionalization, re-fitting embeddings,
  BYO-data joins too large for the browser. It reads the **same** Parquet outputs, so
  there is one data product and two front ends. Use `pydeck` with a scoped bbox, not
  `st_folium`.
- **Revisit** if in-browser kNN on national BG (242k rows) proves too slow. If so, ship
  precomputed top-50 neighbors per preset as Parquet.

### 8.3 Data budget
| Artifact | Target size | Host |
|---|---|---|
| `tract.pmtiles` (z0–12, GEOID + ~5 attrs) | ≤ 150 MB | R2 / HF |
| `bg.pmtiles` (z8–14) | ≤ 400 MB | R2 / HF |
| `pdb_tract_{V}.parquet` (zstd, row groups by state) | ~40–60 MB | R2 / HF |
| `features/*.parquet`, `variables.parquet`, small joins | < 50 MB each | repo or R2 |
| App bundle | < 2 MB | GitHub Pages |
Commit to git only what stays under ~50 MB per file. Larger artifacts are published by CI
to object storage, with a `manifest.json` (URL, sha256, vintage, build date) committed
to the repo.

## 9. Pipeline

Headless, stage-based, idempotent (mirrors `balt311/scripts/pipeline.py` and
`plutometer/scripts/prepare_data.py`):

```bash
python -m pdbb.pipeline --vintage 2025 --stage pdb          # download + normalize PDB, build dictionary
python -m pdbb.pipeline --vintage 2025 --stage geo          # boundaries, hierarchy, adjacency
python -m pdbb.pipeline --stage join --source cdc_places    # one supplemental source (or --all-sources)
python -m pdbb.pipeline --vintage 2025 --stage features     # feature matrix + presets
python -m pdbb.pipeline --vintage 2025 --stage similarity   # embeddings, clusters, kNN indexes
python -m pdbb.pipeline --vintage 2025 --stage tiles        # tippecanoe → PMTiles
python -m pdbb.pipeline --vintage 2025 --stage publish      # upload large artifacts, write manifest.json
```

- **R-PIPE-1.** Raw downloads go to a gitignored `data/raw/` cache and are reused across runs.
- **R-PIPE-2.** GitHub Actions: `pipeline.yml` (`workflow_dispatch`, inputs: vintage,
  stages, sources), with a commit or publish after each stage so a late failure keeps
  earlier output (plutometer pattern). Per-source jobs run in a matrix (balt311
  `peer_city_parallel.yml` pattern). Secrets: `CENSUS_API_KEY`,
  `SOCRATA_APP_TOKEN` (optional), R2 credentials.
- **R-PIPE-3.** Tooling: Python 3.11+, `pandas`/`pyarrow`, `geopandas` + `pyogrio`,
  `duckdb` (+ spatial extension), `shapely` 2, `tobler`, `libpysal`/`spopt`,
  `scikit-learn`, `pynndescent`/`umap-learn` (optional), `tippecanoe` (apt/binary),
  `pmtiles` CLI.
- **R-PIPE-4.** Reuse code from the reference repos as a vendored or shared module
  where practical: `balt311.cities.*` adapters (ArcGIS/Socrata/Carto/CKAN
  clients), `balt311.tiger.fetch_city_tracts`, `equity_stats.overlap_score`,
  plutometer's CT crosswalk and adjacency builder.

## 10. Quality, testing, documentation

- **R-QA-1.** Unit tests (no network): join methods on synthetic grids (area
  weights sum to 1, point counts conserved, crosswalk weights per source GEOID sum to
  1), feature transforms, kNN self-match = rank 1, CT crosswalk.
- **R-QA-2.** Data-integrity tests on built outputs: GEOID uniqueness, row counts
  match the PDB, no state entirely null for core variables, percent fields in [0, 100],
  join coverage ≥ the threshold declared in the source spec.
- **R-QA-3.** Front-end smoke test (Playwright): map loads, click → profile,
  find-similar returns k results, export works. Run in CI against the built static site.
- **R-QA-4.** Docs: `README.md` (run/deploy), `data_catalog.md` (every source,
  URL, license, vintage, join method: the balt311 `data_catalog.md` /
  `geo_reference.md` convention), `CLAUDE.md` (developer context), and in-app
  "About the data" and methodology pages.
- **R-QA-5.** Privacy and ethics: aggregate only (no individual-level records in outputs).
  Apply n<5 suppression on joined event counts. Add a methodology note warning against
  using similarity or typology output for individual eligibility or targeting decisions.

## 11. Phasing

| Phase | Scope | Exit criteria |
|---|---|---|
| **0. Spike** | Validate the PDB 2025/2026 URLs and variables. One-state (MD) prototype: PMTiles + MapLibre + DuckDB-WASM on GitHub Pages. | Click a tract → see PDB values, on free hosting, < 3 s load. |
| **1. Browse MVP** | National tract PDB browse: map, variable picker, dictionary, profile with percentiles and MOE flags, table + export, address search. | Public URL. All ~85k tracts. Mobile usable. |
| **2. Join framework** | Source-spec + `joins.py` (all 5 join types). P1 federal sources (PLACES, SVI, NRI, Food Atlas, CHAS/QCT, LODES). BYO-CSV upload. | ≥6 sources live with join reports. |
| **3. Similarity** | Feature store + facets (methodology.md §3.7), find-similar with explanations, embedding scatter, typology clusters. Then **forecast mode**: tract-level PECOTA comps, cards, and backtest (research_plan.md R2). | kNN < 1 s national. Explanations shown. R2 backtest gates pass. |
| **4. Municipal/state** | balt311 adapters → 311 per-tract metrics for the cohort cities. MD state sources. Custom roll-ups (CSA). Equity lens. | Baltimore reproduces balt311 tract metrics within tolerance. |
| **5. Block groups and blocks** | BG layer nationally. Block drill-down (decennial counts) per county. Dasymetric interpolation. | BG browse + similarity live. |
| **6. Analyst workbench** | Streamlit/HF Space for regionalization, custom embeddings, large BYO joins. | Shares the data product with the static site. |

## 12. Open questions

1. ~~Is the 2026 PDB published?~~ **Resolved (V-03):** yes, via the API, with
   2020–24 ACS. Remaining: the CSV filenames and the variable-list diff vs 2025,
   checked in the Phase 0 spike.
2. Observable Framework vs. plain Vite/TS for the static front end. Decide after the
   Phase 0 spike.
3. R2 vs Hugging Face Datasets for large artifacts. R2 gives better range-request
   latency and needs a Cloudflare account. HF is zero-config and public.
4. ~~EJScreen's federal status~~ **Resolved (V-05):** EPA removed it on
   2025-02-05. Use the PEDP reconstruction or another archived copy and label it
   historical. A challenge (Sierra Club v. EPA) was reportedly dismissed for lack of
   standing in 2026-03 (single source).
5. Which private sources clear license review for caching vs. display-only?
6. Is BG-level similarity useful given ACS MOEs, or should BG be browse-only?
