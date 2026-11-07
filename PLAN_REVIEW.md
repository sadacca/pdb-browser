# pdb-browser — Plan review (2026-11-07)

A review of `requirements.md`, `methodology.md`, `methodology_alternatives.md`,
`research_plan.md` and `verification_log.md`, written while turning them into
`TASKS.md`. Findings are ordered by how much they change what gets built next.

## Verdict

The product direction is sound. The pieces fit together: a static MapLibre +
PMTiles + DuckDB-WASM site, one Python pipeline producing a documented data
product, facets as the unit of similarity, and PECOTA-style comps judged by an
external backtest. The verification log is unusually careful.

The main risk is **sequencing, not design**. There are about 2,000 lines of
research plan and zero lines of code. The research scope (panel harmonization,
annualized state-space models, 1940–2020 history, PECOTA, a 40-method alternatives
backlog) is several person-years. The browse MVP is a few weeks. Ship Phases 0–1
before adding more research text.

## 1. Blocking issues (fix before or during M0–M1)

**1.1 Phase 0 can't run where the plan was written.** census.gov is blocked
from the research containers. I re-checked from this session: `www2.census.gov`
and `api.census.gov` both fail. **Fix (built into TASKS):** run the spike and the
pipeline as GitHub Actions `workflow_dispatch` jobs (T-010, T-042). All unit
tests run on committed synthetic fixtures (T-020), so development never waits on
the network.

**1.2 The reference repos aren't available.** R-PIPE-4 says to vendor code from
`balt311-service-equity` and `plutometer` (city adapters, CT crosswalk, adjacency
builder, overlap score). Neither repo is in this project's GitHub scope. TASKS
treats these as **reimplementations from the description** (T-032, T-033), and
defers the municipal adapters (Phase 4) until access is granted.
*Decision needed:* add those repos to the session, or accept the rewrites.

**1.3 Four overlapping taxonomies for "kinds of variables".**
| Where | Name | Count |
|---|---|---|
| requirements R-PDB-4 | topic groups | 12 |
| requirements R-FEAT-2 | presets | 5 + custom |
| methodology §3.1 | domains | 12 |
| methodology §3.7 / research_plan R1.1 | facets | 13 |

Each list draws its boundaries a little differently. For example, "cost of
living" and "jobs/flows" are only facets, and "Census operations" vs "civic/response" are near-duplicates.
An implementer will build two or three of these by accident.
**Recommendation:** keep only two concepts.
- **Topic groups** organize *browsing*: a UI grouping in the dictionary.
- **Facets** organize *similarity and cohorts*.

A "preset" is then just a named set of facet weights, and "domains" is retired
as a synonym for facets. TASKS already follows this (T-023, T-080).

**1.4 Spec conflicts between requirements and methodology.** methodology.md says
it supersedes requirements, but the conflicts are not marked in requirements.md:
- Winsorizing: p1/p99 (R-FEAT-1) vs p0.5/p99.5 (§3.2).
- Shares: "as-is" (R-FEAT-1) vs logit/IHS (§3.2).
- Distance: cosine/Euclidean with per-feature weights (R-SIM-1) vs
  facet-block distance `√Σ w_f d_f²` with percentile-converted facet distances
  (§3.7).
- Small-area thresholds: <100 people / <50 households (R-PDB-6) vs <200 people
  or GQ > 50% (§3.2).

**Recommendation:** edit R-FEAT-1 and R-SIM-1 to point to methodology.md. Keep
both thresholds, on purpose: `low_pop` greys out a geography in the browser, and
`special` excludes it from model fitting (T-025, T-082).

## 2. Scope and sequencing

**2.1 R1's exit criteria are too heavy for an MVP.** research_plan R1 calls
itself "MVP", but its exit criteria include building the full harmonized
2005–2024 panel and passing the annualized-model county holdout. Neither is needed for
cross-sectional find-similar. Both are prerequisites for R2.
**Recommendation:** split them out:
- **R1-exit** is cross-sectional: facets, typology, find-similar < 1 s, and the
  validation battery §4.1–4.4.
- **R2-entry** is the panel and annual series.

Without that split, the first user-visible similarity feature waits behind the
hardest data engineering in the plan.

**2.2 Defer the long-history tiers and the alternatives backlog.**
- Long history (H0–H3, NCDB, ICPSR, HHUUD10) is valuable for neighbourhood age,
  but only once a basic comps engine exists to consume it.
- The 40+ ALT entries are a good idea bank. Treat them as a backlog, not a plan,
  until R2's backtest exists to judge them (research_plan's own logic).

**2.3 The Streamlit workbench (Phase 6) duplicates effort.** If the static site
and a notebook-friendly data product (Parquet + manifest) exist, most "analyst
workbench" needs are met by DuckDB in a notebook. Drop it from the roadmap unless
a real user asks.

## 3. Method risks (R2 PECOTA)

**3.1 Era-adjustment leakage.** R2.0 says to remove national and metro change
per period, then "add back the current era's expectation". For an honest
backtest, that expectation has to be **forecast as of 2014–18**: Tract-Marcel
drift, or a national trend extrapolation. It must not be the realized 2014–18 →
2019–23 national change. The realized COVID-era shift is exactly what the
as-of forecaster didn't know. Add this explicitly to the R2.2 anti-leakage rule.

**3.2 One ACS fold is thin evidence.** The only rich-variable backtest is
2014–18 → 2019–23, and that period is unusual (COVID, 2020 response problems).
The decennial and LTDB folds cover thin variables only. Pass or fail on the
gates should be read as one observation. Report skill per fold, with a bootstrap over
metros, and treat the gates as necessary, not sufficient.

**3.3 ACS noise will inflate the event rates.** "Upgrade = metro-relative income
percentile rises ≥ 10 points" can happen through sampling error alone for
small tracts: a ±30% MOE on median income moves a tract many percentiles.
Compute a **noise-only base rate**: the share of tracts crossing the threshold
under MOE-perturbed resampling with no true change. Show event rates net of it,
or calibrate against it. This fits the existing "MOE-perturbed truth" line in R2.2.

**3.4 The MOE question (V-14) gates more than the CV badges.** These all assume
MOEs:
- MOE-aware shrinkage (R2 baseline);
- MOE-weighted distance;
- the §4.3 stability test;
- the noise floor in 3.3.

If the PDB lacks per-variable MOEs, the ACS detailed-table API fallback becomes
a real pipeline stage, not a footnote. T-010 measures this first.

## 4. Engineering notes

- **Browser CORS and range requests.** DuckDB-WASM on remote Parquet and PMTiles
  both need `Range` requests, and CORS must expose `Content-Range`/`Accept-Ranges`.
  On R2 this is a bucket CORS policy: add it to the publish runbook. HF Datasets
  `resolve/` URLs redirect, which DuckDB handles, but check latency in the spike.
- **Census Geocoder from the browser** may not allow CORS. T-067 checks this and
  falls back to JSONP, then to name search.
- **Mapping any of ~500 variables.** Tiles carry ~5 attributes, so mapping
  an arbitrary variable needs a client join: MapLibre `feature-state`, filled from DuckDB
  per viewport or per state. This deserves a small design spike after T-070. It
  is listed as an undecomposed epic rather than guessed at.
- **R2 operation counts.** Each range read is a Class B operation (V-11). A
  national choropleth session can make hundreds. 10M a month is fine for a hobby
  site, but put a usage alarm on the account.
- **Percentiles at three scopes** for 500 variables are too big to precompute for
  ~935 CBSAs. TASKS precomputes national and state quantile tables, and computes
  metro percentiles live (D-10).

## 5. Responsible use (kept; two additions)

The R2.6 and R-QA-5 requirements are good: neutral rate names, fans not points,
use restrictions, and the race facet opt-in. Two additions:
1. Make the Decline/Thinning tiles **off by default in public builds**, until the
   noise floor (3.3) and calibration gates are met. Don't leave this at "consider gating".
2. Keep k-means cohort labels and descriptions as `reviewed: false`, unpublished,
   until a human has read them. Auto-descriptions of clusters that track race are
   exactly where harmful labels creep in. TASKS T-080/T-084 add this flag.

## 6. Decisions the owner should confirm

1. Front end: **Vite + TS** (locked as D-2 in TASKS) instead of Observable
   Framework. Simpler toolchain for an implementing model; this can be revisited after Phase 1.
2. Large-file host: R2 or HF. TASKS keeps this abstract behind `manifest.json`,
   so it can be decided at T-041.
3. Access to `balt311-service-equity` and `plutometer` (1.2).
4. Adopt the two-concept taxonomy (1.3), and mark the requirements sections that
   methodology supersedes (1.4).
5. Split R1 exit / R2 entry (2.1).
