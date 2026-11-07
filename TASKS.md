# pdb-browser — Implementation task list

> **Audience:** an implementing model or developer working one task at a time.
> **Source of truth for "what":** `requirements.md`, `methodology.md`, `research_plan.md`.
> This file is the "how, in what order". Plan critique and the reasons behind the
> locked decisions are in `PLAN_REVIEW.md`.
>
> **Scope:** Phase 0 (spike) through Phase 2 (join framework), plus the R1 similarity
> baseline. Later phases are listed at the end as epics. They are **not yet broken
> into tasks** because they depend on spike results and need design judgment.

---

## How to work this list (read first)

1. **One task per commit** (or PR). Use the task ID in the message: `T-021: ingest PDB CSV`.
2. **Do the tasks in order within a milestone.** `Depends:` lists cross-milestone prerequisites.
3. **A task is done only when every "Done when" check passes.** Run them and paste the
   output into the commit body or PR. Then tick the box here in the same commit.
4. **Tests never touch the network.** Use the fixtures in `tests/fixtures/`. Real
   downloads happen only in the GitHub Actions workflows (`spike.yml`, `pipeline.yml`).
   census.gov is blocked from the dev container.
5. **Do not change a Locked Decision (below).** If one seems wrong, stop, add a
   `> BLOCKED:` note under the task, and move on to an independent task.
6. **Do not edit the research docs** (`requirements.md`, `methodology*.md`,
   `research_plan.md`, `verification_log.md`). The exceptions are T-011, and any
   task that says otherwise.
7. **Never hard-code a vintage suffix** (`_ACS_19_23`) or an ACS period in code.
   They come from `config/vintages.yaml` or are parsed from column names.
8. Keep functions small and pure where possible. Put I/O at the stage edges
   (`pdbb/stages/*.py`), and logic in modules that tests can call directly.
9. If something is ambiguous and not covered here, choose the simplest option, write
   it down in `CLAUDE.md` under "Decisions made during implementation", and continue.

## Locked decisions

| ID | Decision |
|---|---|
| D-1 | Python ≥ 3.11 package `pdbb` in `src/pdbb/`, built with `pyproject.toml` (hatchling). Dev tools: `pytest`, `ruff`. |
| D-2 | Front end: **Vite + TypeScript, no UI framework**, in `web/`. Libraries: `maplibre-gl`, `pmtiles`, `@duckdb/duckdb-wasm`, `@observablehq/plot`. Tests: `vitest` (unit) and `@playwright/test` (smoke). This settles requirements.md open question 2 for now. |
| D-3 | Data layout. `data/raw/` holds the download cache and `data/out/` holds build outputs; both are gitignored. Output subfolders: `pdb/`, `geo/`, `tiles/`, `features/`, `similarity/`, `joined/`, `reports/`. `manifest.json` is written to `data/out/` and copied to `web/public/` at deploy. |
| D-4 | **Stable variable key:** lowercase the raw name, then replace a trailing `_ACS_YY_YY` with `_acs`, and a trailing `_CEN_YYYY` / `_DEC_YYYY` with `_cen`. Keep a `pct_` prefix as-is. MOE columns keep their `moe_` prefix or `_moe` suffix in the same normalized form. ID columns (`GEOID`, `State`, `County`, `Tract`, `Block_group`, names) map to fixed keys: `geoid`, `state`, `county`, `tract`, `block_group`, `state_name`, `county_name`. |
| D-5 | GEOIDs are **strings** everywhere: 11 characters for tracts, 12 for BGs. Every reader passes `dtype=str` or the Arrow `string` type. |
| D-6 | Geometry CRS: store EPSG:4326. Compute area and adjacency in EPSG:5070 (CONUS Albers). Alaska, Hawaii, and Puerto Rico are fine in 5070 for adjacency and area at tract scale. |
| D-7 | Large-file host: abstract it behind `manifest.json` URLs, so the web app never hard-codes R2 or HF. Dev serves files from `data/out/` through Vite's `publicDir`/proxy. |
| D-8 | Similarity MVP distance: per-facet PCA blocks, each scaled to unit total variance. Combined distance `sqrt(Σ_f w_f · ||x_f − y_f||²)`. `w_f ∈ {0 = ignore, 1 = match}` for the MVP. Percentile-converted distances and "differ" mode come later (R1.2 follow-up). |
| D-9 | The race/ethnicity facet is `opt_in: true` and **off** in the default preset and the default typology. |
| D-10 | Percentiles: the pipeline precomputes **national and state** quantile tables (101 quantiles per variable per scope). The browser computes **metro** percentiles live, within the metro subset. |

---

## M0 — Scaffold

- [ ] **T-001 Repo scaffold.** Create `pyproject.toml` (package `pdbb`, extras `[dev]`, `[geo]`, `[ml]`), `src/pdbb/__init__.py`, `tests/test_smoke.py` (imports `pdbb`), `.gitignore` (`data/raw/`, `data/out/`, `node_modules/`, `web/dist/`, `.venv/`), a `Makefile` with `install`, `test`, `lint` targets, and a `README.md` stub (purpose and how to run tests).
  Core deps: `pandas`, `pyarrow`, `pyyaml`, `requests`, `duckdb`. `[geo]`: `geopandas`, `pyogrio`, `shapely>=2`. `[ml]`: `scikit-learn`, `numpy`.
  *Done when:* `pip install -e '.[dev,geo,ml]' && pytest -q && ruff check .` all pass.
- [ ] **T-002 `CLAUDE.md`.** Add developer context: the Locked Decisions table (copied), the directory map, how to run tests, the "no network in tests" rule, and an empty "Decisions made during implementation" section.
  *Done when:* the file exists, and its decisions table matches this file's.
- [ ] **T-003 CI.** Add `.github/workflows/ci.yml`, triggered on push and pull_request. It sets up Python 3.11, installs `.[dev,geo,ml]`, and runs `ruff check .` and `pytest -q`. The web job is added in T-060.
  *Done when:* the workflow YAML parses (`python -c "import yaml,sys; yaml.safe_load(open('.github/workflows/ci.yml'))"`) and the commands it runs pass locally.
- [ ] **T-004 Paths and config.** Add `src/pdbb/paths.py`, with a `Paths(data_dir)` dataclass that has a method per artifact (e.g. `pdb_parquet(level, vintage)`, `dictionary(vintage)`, `report(name)`). Add `config/vintages.yaml`, keyed by vintage (`2025`, `2026`), with fields: `acs_period` (e.g. `"2019-2023"`), `tract_csv_url`, `bg_csv_url`, `api_variables_url`, `expected_tract_rows` (nullable), `expected_tract_cols` (nullable), `confirmed: false`. Add `src/pdbb/config.py` with `load_vintage(v)` that validates the required fields.
  *Done when:* tests cover a valid load, a missing field (raises a clear error), and an unknown vintage.
- [ ] **T-005 Download cache.** `src/pdbb/fetch.py`: `fetch(url, dest, *, force=False) -> Path`. It reuses `dest` if present, streams to `dest.part` then renames, writes a `dest.sha256` sidecar, and retries 4× with backoff (2, 4, 8, 16 s; injectable `sleep` for tests). Also `unzip_member(zip_path, pattern, out_dir)`.
  *Done when:* tests pass using a monkeypatched `requests.Session.get`, covering cache hit, cache miss, retry-then-success, and a partial file never left as `dest`.
- [ ] **T-006 Pipeline CLI.** `src/pdbb/pipeline.py`, run as `python -m pdbb.pipeline --vintage 2025 --stage pdb [--data-dir data] [--level tract|bg] [--source ID]`. It holds a stage registry (`pdb`, `geo`, `features`, `similarity`, `tiles`, `publish`, `join`), with `--list-stages` and `--stage all` (registry order). Each stage lives in `src/pdbb/stages/<name>.py` as `run(ctx)` and initially raises `NotImplementedError`. `ctx` carries `paths`, `vintage_cfg`, `level`, `source`, and a logger.
  *Done when:* tests cover argument parsing, unknown-stage error, `--list-stages` output, and that a stub stage's `NotImplementedError` gives exit code 2 with a clear message.

## M0b — Phase 0 spike (needs network: runs in GitHub Actions)

- [ ] **T-010 Spike script and workflow.** `scripts/spike_pdb.py`:
  - For each vintage in `vintages.yaml`, and also for candidate filenames (`pdb{V}tr.csv`, `pdb{V}trv{1..3}_us.csv`, `pdb{V}bgv{1..3}_us.csv`, and the directory listing at `https://www2.census.gov/adrm/PDB/{V}/` if one is served), record the HTTP status.
  - For the first working tract and BG CSV, stream it and record row count, column count, the first 60 headers, every distinct `_ACS_YY_YY` suffix, and the count of columns containing `MOE` (case-insensitive). Also report the share of ACS estimate columns with a matching MOE column (V-14).
  - Fetch the API `variables.json` and record its variable count.
  - Write `spike_report.md` and `spike_report.json`.

  Put the header analysis in a pure function, `analyze_header(cols) -> dict`, in `src/pdbb/spike.py`. Add `.github/workflows/spike.yml` (`workflow_dispatch`) that runs the script and uploads both reports as an artifact.
  *Done when:* unit tests for `analyze_header` pass on a fixture header list, and the workflow YAML parses. **A human then triggers the workflow.**
- [ ] **T-011 Record spike results** (after a human has run T-010). Update `config/vintages.yaml` with the confirmed URLs, expected rows and columns, and `confirmed: true`. Update `verification_log.md` V-03 and V-14 with the findings. If MOE coverage is incomplete, open a `> FOLLOW-UP:` note under T-024.
  *Done when:* config is updated and the log rows are marked ✅ or ⚠️ with numbers.

> Until T-011 lands, every M1 task works against the **synthetic fixture** only. That is fine, and it is the point.

## M1 — PDB stage (tract first; BG reuses it with `--level bg`)

- [ ] **T-020 Synthetic fixtures.** `tests/fixtures/make_fixtures.py` generates and commits:
  - `pdb_tract_fixture.csv`: 30 tracts across states `01`, `09` (CT), and `24` (MD), 3 counties each. Include:
    - ID columns per D-4;
    - `Tot_Population_ACS_19_23` and its MOE;
    - `Tot_Housing_Units_ACS_19_23`;
    - `Med_HHD_Inc_ACS_19_23` as strings like `"$63,750"`, with one blank value;
    - `pct_Renter_Occp_HU_ACS_19_23` with one value of `"101.2"` (for the integrity check to catch);
    - one decennial column `Tot_Population_CEN_2020`;
    - two tracts with population < 100.
  - `pdb_variables_fixture.json`: a minimal API `variables.json` shape, `{"variables": {name: {"label": …, "concept": …}}}`.

  Generate deterministically (fixed seed).
  *Done when:* running the script reproduces byte-identical files.
- [ ] **T-021 Ingest and coerce.** `src/pdbb/pdb/ingest.py`: `read_pdb_csv(path, level) -> DataFrame`. It reads everything as strings, renames columns to stable keys (T-022), and coerces numerics by stripping `$`, `,`, `%`, and whitespace, with empty → NA. A coercion failure goes to a report (column, raw value, count), not an exception. It asserts GEOID length per D-5 and uniqueness. `write_pdb_parquet(df, path)` sorts by GEOID, uses zstd compression, and writes one row group per state.
  *Depends:* T-022.
  *Done when:* fixture tests pass for leading zeros preserved, `"$63,750"` → 63750.0, blank → NA, one row group per state (check with `pyarrow.parquet.ParquetFile(...).metadata`), and duplicate GEOIDs raising.
- [ ] **T-022 Stable keys.** `src/pdbb/pdb/keys.py`: `stable_key(raw) -> KeyInfo(key, kind ∈ {id, acs, cen, moe, other}, period)`, implementing D-4. Also `build_key_map(columns, vintage) -> DataFrame` with columns `raw_name`, `key`, `kind`, `period`, `vintage`. If two raw names collide on one key, raise and list both.
  *Done when:* table-driven tests cover at least 15 realistic names (including MOE variants and IDs) and the collision error.
- [ ] **T-023 Data dictionary.** `src/pdbb/pdb/dictionary.py` builds `variables.parquet`, one row per key. Columns: `key`, `label` (from API JSON, falling back to the raw name), `topic_group`, `source` (`ACS`/`Census`/`derived`/`id`), `unit` (`percent` if `pct_`; `dollars` if the name matches `Med_.*(Inc|Val|Rent)|Avg_.*(Inc|Val)`; `score` for LRS/response; else `count`), `has_moe` (**computed from the columns actually present**), `moe_key`, `denominator_key` (nullable), and `higher_is` (nullable).
  - `topic_group` comes from regex rules in `config/topic_rules.yaml` (the R-PDB-4 groups). Anything unmatched gets `Other`.
  - `denominator_key` comes from `config/denominators.yaml` (explicit map; start with ≥10 common counts).

  *Done when:* fixture tests pass, and the stage writes `reports/dictionary_unmatched_{V}.csv` listing every key in `Other`.
- [ ] **T-024 MOE → CV.** `src/pdbb/pdb/reliability.py`: `cv(est, moe) = (moe / 1.645) / est`, with NA when `est <= 0` or MOE is missing. `reliability_flag(cv)` returns `ok` (≤ 0.30), `amber` (0.30–0.50), or `red` (> 0.50). The stage writes `pdb/cv_tract_{V}.parquet` (GEOID × cv per `has_moe` key, float32).
  *Done when:* unit tests cover the boundary values 0.30 and 0.50, est = 0, and missing MOE.
- [ ] **T-025 Small-population flag.** Add `low_pop` (bool) = population < 100 or households < 50 (household key from config). Rows are **never dropped**.
  *Done when:* a test confirms the 2 fixture tracts are flagged and the row count is unchanged.
- [ ] **T-026 Integrity checks.** `src/pdbb/checks.py`. Each check returns `CheckResult(name, ok, detail)`. Checks:
  - GEOID unique;
  - row count = `expected_*_rows` (skip with a warning if null);
  - no state entirely null for each core variable (list in config);
  - `pct_*` values in [0, 100] (report offenders, don't clip).

  Write `reports/pdb_{level}_{V}.json`. The stage fails (non-zero exit) only on GEOID uniqueness or a row-count mismatch; the other checks warn.
  *Done when:* tests show the fixture's `101.2` reported, and an all-null state reported.
- [ ] **T-027 Quantile tables (D-10).** `src/pdbb/pdb/quantiles.py` writes `pdb/quantiles_{level}_{V}.parquet`, long format: `scope_type ∈ {national, state}`, `scope_id`, `key`, `q` (0–100), `value`. Exclude `low_pop` rows. Add a helper `percentile_of(value, qtable)`, plus a TS twin in T-066 (both must agree on the shared test vectors in `tests/fixtures/percentile_vectors.json`).
  *Done when:* tests pass, including ties and NA handling.
- [ ] **T-028 Wire `pdb` stage.** `stages/pdb.py`: fetch (T-005) → ingest → dictionary → CV → flags → checks → quantiles → write. Add `--input-csv` and `--input-variables` overrides, so it runs end-to-end on fixtures.
  *Done when:* `python -m pdbb.pipeline --vintage 2025 --stage pdb --input-csv tests/fixtures/pdb_tract_fixture.csv --input-variables tests/fixtures/pdb_variables_fixture.json --data-dir /tmp/x` produces every expected file. An integration test asserts this.

## M2 — Geo stage

- [ ] **T-030 Boundaries.** `src/pdbb/geo/boundaries.py`: download `cb_{yr}_us_tract_500k.zip` (display) and convert it to `geo/tract.fgb` (GEOID + geometry, EPSG:4326, spatial index on). `yr` comes from vintages.yaml (`boundary_year`; add the field).
  *Done when:* a test builds a 3×3 synthetic grid GeoDataFrame, writes it as a shapefile zip in `tmp_path`, runs the converter, and reads the FGB back with 9 rows and GEOIDs as strings.
- [ ] **T-031 Hierarchy.** `src/pdbb/geo/hierarchy.py`: `geo/tract_hierarchy.parquet` with `geoid`, `state`, `county` (5-char), `cbsa`, `cbsa_name`, `cbsa_type` (metro/micro/none). CBSA comes from the Census CBSA delineation file (county → CBSA; URL in config). Place, county subdivision, congressional district, and ZCTA are **out of scope here** (follow-up).
  *Done when:* a test with a 3-row fixture delineation file passes, and counties outside any CBSA get `cbsa = None`, `cbsa_type = 'none'`.
- [ ] **T-032 Queen adjacency.** `src/pdbb/geo/adjacency.py`: `queen_adjacency(gdf) -> DataFrame[geoid, neighbor]`. It uses `shapely.STRtree` in EPSG:5070 with predicate `touches` or `intersects`, excluding self, and is symmetric. Build it from **full-resolution TIGER** geometry, not cb_500k (requirements R-GEO-4); add the TIGER URL to config.
  *Done when:* on a 3×3 grid, the centre has 8 neighbours, an edge 5, and a corner 3. Also test symmetry, and that two polygons touching at a single point count as neighbours.
- [ ] **T-033 Connecticut crosswalk.** `src/pdbb/geo/ct.py`: a table mapping old CT county FIPS (09001–09015) to planning regions (09110–09190), loaded from `config/ct_county_crosswalk.csv`. Source it from the Census CT changes page in T-034's CI run, or hand-enter it with a citation. Add `to_planning_region(geoid)` for tract GEOIDs. Note that plutometer's version can't be vendored: that repo isn't available to this project, so this is a reimplementation.
  *Done when:* tests pass for one mapping per old county, and non-CT GEOIDs are unchanged.
- [ ] **T-034 Wire `geo` stage.** Fetch → boundaries → hierarchy → adjacency. Write `reports/geo_{V}.json` covering tract count, GEOIDs in PDB but not in geometry (and the reverse), and isolated tracts (0 neighbours).
  *Done when:* an integration test on fixtures (with `--input-*` overrides like T-028) passes.

## M3 — Tiles and publish

- [ ] **T-040 Tiles stage.** `src/pdbb/stages/tiles.py`:
  - join `geo/tract.fgb` with ≤ 6 styling attributes (`config/tile_attrs.yaml`; default: population, median household income, % renter, LRS, `low_pop`);
  - write newline-delimited GeoJSON;
  - run `tippecanoe -o tract.pmtiles -l tracts -Z0 -z12 --coalesce-densest-as-needed --detect-shared-borders --use-attribute-for-id=…` (do not use the id flag if GEOID has a leading zero; keep GEOID as a string property).

  Put the command in `build_tippecanoe_cmd(...) -> list[str]`, and fail with a clear message if `tippecanoe` is missing.
  *Done when:* the command builder has a unit test. An integration test runs only when `shutil.which('tippecanoe')` (otherwise `pytest.skip`) and checks the PMTiles header magic bytes.
- [ ] **T-041 Publish stage.** `src/pdbb/stages/publish.py`:
  - list files in `data/out` and compute their sha256 and size;
  - write `manifest.json` (`{build_date, vintage, files: {logical_name: {url, sha256, bytes}}}`);
  - upload with `boto3` to an S3-compatible endpoint (`R2_ENDPOINT`, `R2_BUCKET`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`).

  **Default is `--dry-run`**; real upload needs `--upload`. Files over 50 MB must not be written into the repo.
  *Done when:* a dry-run test writes a correct manifest for a tmp dir.
- [ ] **T-042 `pipeline.yml` workflow.** `workflow_dispatch` with inputs `vintage`, `stages` (comma list), and `upload` (bool). It installs tippecanoe (build from the `felt/tippecanoe` release tag if apt lacks it) and caches `data/raw/` with `actions/cache`, keyed by vintage. It runs the stages in order, uploads `data/out/reports/` as an artifact on every run (even on failure), and runs publish only if `upload` is true.
  *Done when:* the YAML parses, and a human has triggered one successful `pdb,geo,tiles` run (tick the box only then).

## M4 — Web browse MVP (Phase 1)

> Development uses outputs built from fixtures (T-028/T-034/T-040), served from
> `data/out`. Make `make dev-data` build them.

- [ ] **T-060 Web scaffold.** `web/` with Vite + TS (strict), ESLint, `vitest`. Scripts: `dev`, `build`, `test`, `lint`. Add `VITE_MANIFEST_URL` (default `/manifest.json`). Extend `ci.yml` with a web job: `npm ci && npm run lint && npm test && npm run build`.
  *Done when:* those four commands pass and `web/dist` is under 2 MB (excluding wasm files fetched from the CDN).
- [ ] **T-061 Data access layer.** `web/src/data/`:
  - `manifest.ts` loads and validates the manifest;
  - `db.ts` lazily initializes DuckDB-WASM (bundles from jsDelivr per its docs);
  - `queries.ts` provides `getTract(geoid)`, `getColumn(key, scope)`, `getDictionary()`, and `getQuantiles(key, scopeType, scopeId)`.

  Remote Parquet is read via `read_parquet('<url>')`, with HTTP range requests.
  *Done when:* unit tests cover query-string builders (pure functions). Query execution is covered by T-070.
- [ ] **T-062 URL state.** `web/src/state/url.ts`: `parse(search) -> {geoid?, var?, vars?, scope?}` and `serialize(state)`; round-trip tests. A tiny pub/sub store (`store.ts`) holds `selectedGeoid`, `activeVar`, and `scope`, and syncs to the URL with `history.replaceState`.
  *Done when:* round-trip tests pass, and invalid GEOIDs are dropped.
- [ ] **T-063 Map.** `web/src/map/`: MapLibre with an OpenFreeMap style URL (no token), `pmtiles` protocol registered, and a tract fill layer coloured by `activeVar` (only the T-040 tile attributes for now). Click sets `selectedGeoid`, with a highlight outline. Hover shows a tooltip with name and value.
  *Done when:* it builds; behaviour is verified in T-070.
- [ ] **T-064 Classification and legend.** `web/src/map/classify.ts` provides `quantileBreaks(values, n)`, `equalBreaks(min, max, n)`, and a diverging option when the dictionary gives a midpoint. Palettes: viridis (sequential) and a validated colourblind-safe diverging palette (e.g. ColorBrewer PuOr). The legend is a DOM component.
  *Done when:* unit tests cover breaks (ties, NA, n > unique values).
- [ ] **T-065 Variable picker.** Variables grouped by `topic_group`, with substring search over key and label, keyboard navigable. For now, only tile attributes can be mapped; others are marked "profile only". Mapping any variable is a follow-up: feature-state join from DuckDB by viewport.
  *Done when:* unit tests cover search filtering.
- [ ] **T-066 Profile panel.**
  - Header: name, county, CBSA, population, area, density.
  - Then topic sections. Each variable shows its value with unit formatting, national and state percentile (from quantile tables via `percentileOf`, the TS twin of T-027), metro percentile computed live, and a CV badge (amber/red).
  - `low_pop` tracts show a grey banner.
  - Export the profile as CSV.

  *Done when:* `percentileOf` passes the shared `percentile_vectors.json`, and the formatters have unit tests.
- [ ] **T-067 Address search.** Use the Census Geocoder `geographies/onelineaddress` (benchmark `Public_AR_Current`, vintage `Current_Current`) → tract GEOID. **Check CORS first.** If browser fetch is blocked, use its `format=jsonp` with a callback. If that also fails, write `> BLOCKED:` and fall back to county/place name search over the hierarchy table.
  *Done when:* response parsing has unit tests on a saved fixture response.
- [ ] **T-068 Table view.** Rows = tracts in the current scope (state by default; ≤ 5k rendered, virtualized with a hand-written windowing list, no library). Columns are sortable, with a column chooser and CSV export. Row click sets `selectedGeoid` (linked view).
  *Done when:* unit tests cover sorting and CSV escaping.
- [ ] **T-069 Layout, mobile and a11y.** CSS grid: map + side panel on desktop; map on top with panels below under 700 px, and no horizontal scroll. Visible focus states, ARIA labels on controls, and the table as the text alternative to the map.
  *Done when:* T-070 includes a 390 px viewport check with `scrollWidth <= innerWidth`.
- [ ] **T-070 Playwright smoke test.** Use `web/tests/smoke.spec.ts` against `vite preview` with fixture data. Check that:
  - the map canvas renders;
  - clicking a known tract (or setting `?geoid=` directly) opens the profile with the correct population;
  - the table renders rows;
  - CSV export downloads;
  - the mobile width has no horizontal scroll.

  Add it to CI (fixture tiles need tippecanoe in CI; reuse the T-042 install step).
  *Done when:* it passes in CI.
- [ ] **T-071 Deploy.** `.github/workflows/pages.yml` builds `web/` and deploys to GitHub Pages on push to `main`. The manifest URL points to the published object store.
  *Done when:* the workflow parses; a human confirms the public URL (Phase 1 exit).

## M5 — Features and similarity baseline (R1)

- [ ] **T-080 Facet registry.** Add `config/facets.yaml` (schema: `facets: [{id, label, opt_in, variables: [key…], transform_hint}]`) and `src/pdbb/features/facets.py` to load and validate it. Validation rules:
  - every key exists in the dictionary;
  - **each key belongs to exactly one facet**;
  - the race facet has `opt_in: true` (D-9).

  The first draft of the facet contents is generated from `topic_group` and **flagged `reviewed: false`**. A human reviews it before T-084 results are published.
  *Done when:* validation tests pass, including the duplicate-key error.
- [ ] **T-081 Transforms.** `src/pdbb/features/transforms.py`, as pure numpy/pandas functions, each with tests:
  - `logit(p, eps=1e-3)` on 0–1 shares (percent ÷ 100 first);
  - `clr(block, eps)`;
  - `log1p`;
  - `winsorize(x, 0.005, 0.995)`;
  - `robust_scale` (median/IQR, IQR = 0 → 1);
  - `impute_county_median(x, county)` + `missing_flag`;
  - `drop_sparse(df, max_missing=0.20)`.

  Methodology.md supersedes requirements.md where they differ (winsor p0.5/p99.5; shares go through logit, not as-is).
  *Done when:* tests cover known values, CLR rows summing to 0, and IQR = 0.
- [ ] **T-082 Special-population flag.** `special = low_pop or group-quarters share > 0.5 or population < 200`. Special rows are excluded from fitting (PCA, k-means), get projected embeddings, and get cohort `special`.
  *Done when:* tests pass.
- [ ] **T-083 Facet embeddings.** `src/pdbb/features/embed.py`: per facet, transform → PCA keeping components to ≥ 80% variance, clipped to 2–8 → scale the block to unit total variance. Fit on non-special rows. Output `features/facet_embed_{level}_{V}.parquet` (geoid + `f{facet}_{i}` float32) and `features/facet_loadings_{V}.json`. Fixed seed.
  *Done when:* tests check shapes, total variance per block ≈ 1 (tolerance 1e-6), determinism across two runs, and that special rows are still present.
- [ ] **T-084 Baseline typology (E0).** `src/pdbb/similarity/typology.py`: k-means (k-means++, `n_init=50`, seed) on the concatenated default facets (race excluded), for k in a configured range. For each k, record silhouette (on a 20k subsample, 5 repeats), Calinski–Harabasz, and Davies–Bouldin. The chosen k comes from config (a human picks it after reading the report). Output labels, centroids, and auto descriptions (top 5 features by |centroid z − 0|, from pre-PCA scaled features).
  *Done when:* on synthetic Gaussian blobs (k = 4), `k = 4` recovers the blobs with ARI > 0.9.
- [ ] **T-085 Stability report.** Bootstrap ARI (20 resamples, 80%), plus MOE-perturbation ARI (draw `est + N(0, (moe/1.645)²)` for `has_moe` features, 20 draws; skip with a note if MOEs are missing). Write `reports/validation_{V}.json`.
  *Done when:* tests on synthetic data pass, with high ARI for well-separated blobs.
- [ ] **T-086 kNN precompute.** Top-50 neighbours per tract for the default preset (brute force, sklearn `NearestNeighbors`, Euclidean on the concatenated blocks) → `similarity/knn_default_{V}.parquet` (geoid, rank, neighbor, dist).
  *Done when:* every tract's rank-1 result is itself at distance 0, or the self row is excluded, as documented; and a hand-computed 5-point example matches.
- [ ] **T-087 Web find-similar.** `web/src/similarity/`:
  - load facet embeddings as a `Float32Array` (via DuckDB → Arrow);
  - compute `knn(target, weights, scopeMask, k)` by brute force (D-8);
  - UI: a facet toggle per facet (match/ignore), scope (national / state / metro / exclude own county), and k = 10;
  - explanation: per-facet distance bars for each match (smallest = "similar because", largest = "differs in").

  *Done when:* unit tests check against a hand-computed example, and a vitest benchmark of 85,000 × 60 random float32 finishes in < 1 s on CI (log the time).
- [ ] **T-088 Embedding scatter.** A 2-D PCA of the concatenated default embedding, precomputed in the pipeline (`similarity/pca2_{V}.parquet`) and drawn with canvas (not SVG: 85k points). The selected tract and its matches are highlighted, and clicking a point selects it.
  *Done when:* it builds; a smoke test click selects a tract.
- [ ] **T-089 Typology layer.** Cohort ID added to the tiles as an attribute, with a categorical palette (≤ 12 colours, colourblind-checked) and a legend using the auto descriptions.
  *Done when:* it builds; the smoke test toggles the layer.

## M6 — Join framework (Phase 2)

- [ ] **T-090 Source spec schema.** `src/pdbb/joins/spec.py`: a dataclass schema for requirements.md §5.1, with enum validation (`tier`, `join`, `native_geography`, `license`), and `load_specs(dir='sources/')`.
  *Done when:* tests cover a valid spec, a bad enum, and a missing field.
- [ ] **T-091 `direct` join.** `joins.direct(src_df, geoid_col, universe_geoids) -> (df, JoinReport)`. `JoinReport` holds rows_in, rows_matched, unmatched_sample (≤ 20), coverage_pct, and method; it is written to `reports/join_{source}.json`.
  *Done when:* tests cover match rate and leading-zero mismatches surfacing in the unmatched sample.
- [ ] **T-092 `point_in_polygon`.** `geopandas.sjoin(predicate='within')` → aggregations: count, rate per 1k population, median of a field, share by category. Counts in 1–4 are suppressed (set NA plus a `suppressed` flag); 0 stays 0.
  *Done when:* tests on a grid check that points are conserved (sum of counts = points inside), points on a boundary land in exactly one cell (document the rule), and suppression works.
- [ ] **T-093 `areal_interpolation` (area-weighted).** Overlay in EPSG:5070. Weights = intersection area / source area. Extensive variables are summed with weights; intensive ones are weighted by target overlap share. Population-weighted (blocks) is a follow-up.
  *Done when:* tests check that the weights for each source polygon sum to 1 (for full coverage) and that an extensive total is conserved.
- [ ] **T-094 `crosswalk`.** Generic `apply_crosswalk(df, xwalk[src, tgt, weight], extensive|intensive)`, plus loaders for the HUD ZIP→tract crosswalk (residential ratio) and the Census 2010→2020 tract relationship file (weights from the population or area columns).
  *Done when:* tests check that weights per source sum to 1 (validated, with a warning if not), and conservation of totals.
- [ ] **T-095 `broadcast`.** Copy the parent value to children by GEOID prefix (state = 2, county = 5) or by CBSA from hierarchy. Set `granularity` metadata.
  *Done when:* tests pass.
- [ ] **T-096 Provenance and license gate.** Joined outputs go to `joined/{source_id}.parquet`, with provenance in Parquet key-value metadata, and a `variables_joined.parquet` dictionary is appended (source_id, vintage, join, native_geography, coverage). Export and tile builders call `assert_redistributable(keys)`, which raises on any `proprietary-display-only` source.
  *Done when:* tests show the tile builder refuses a proprietary key.
- [ ] **T-097..T-099 First sources** (one task each; these need network, so validate in `pipeline.yml`): `sources/cdc_places.yaml` (data.cdc.gov Socrata CSV, 2025 release), `sources/cdc_svi.yaml`, `sources/fema_nri.yaml`. Each includes a minimal 5-row fixture plus a test of the parse/normalize step.
  *Done when:* fixture tests pass, and a human-triggered CI run has a coverage report of at least 95% of tracts (SVI and NRI) and at least 90% (PLACES).
- [ ] **T-100 Joined variables in the web app.** Merge the joined dictionary into the picker and profile, with a provenance tooltip and a coverage % next to each variable. Add a "Model-based estimate" tag for PLACES and a vintage-mismatch warning when the source vintage differs from the PDB ACS period.
  *Done when:* unit tests cover the vintage-mismatch logic.

---

## Not yet decomposed (needs design review and spike results first)

These need judgment calls that a task list shouldn't pre-empt. Break each into
tasks only after its prerequisites land and a reviewer signs off.

| Epic | Why it waits | Prerequisite |
|---|---|---|
| Map any variable (feature-state join by viewport) | Choose between DuckDB-per-viewport and per-state attribute shards after measuring T-061 latency | T-070 |
| BYO-data upload (R-JOIN-4) | Client-side point-in-polygon needs a geometry source in the browser (FGB by bbox) | M4, M6 |
| Block groups nationally (Phase 5) | Tile size budget; the BG similarity question (requirements open Q6, methodology Q4) | T-085 results on BG |
| Hierarchy: place, county subdivision, congressional district, ZCTA | Relationship-file choices | T-031 |
| Population-weighted (dasymetric) areal interpolation | Block-level 2020 data volume | T-093 |
| Municipal/state adapters, 311 (Phase 4) | balt311 code is not in this session's repos; needs access or a rewrite decision | — |
| Harmonized ACS panel 2009–2024 (R1.3 layers 1 and 3) | 2010→2020 weighting choice; this is the R2 prerequisite | M5 |
| Annualized state-space series (R1.3 layer 2) | A research task, not an implementation task | Panel |
| Long-history tiers H0–H3 | Licence and access checks (NCDB, ICPSR) | Panel |
| Tract-PECOTA (R2) and backtest | Needs the panel; the method has open choices | Panel + M5 |
| Alternatives and consensus (R3) | Judged against R2 | R2 |
| Streamlit analyst workbench (Phase 6) | Lowest priority | — |
