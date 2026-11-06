# pdb-browser — Research plan

> **Status:** Draft v0.1 (2026-11-06). This sequences the method research in
> `methodology.md` and `methodology_alternatives.md` into three phases:
>
> 1. **Phase R1, MVP:** k-means baseline and facets.
> 2. **Phase R2, tract-level PECOTA:** the flagship worked example.
> 3. **Phase R3, alternatives and consensus.**
>
> "R" phases are research phases. They map onto the product phases in
> `requirements.md` §11 as noted under each phase.

**Why this order.** R1 produces the representations and cohorts. R2 turns them
into a forecasting system with a **hard, external scoring rule**: did the comps
predict what actually happened next? R3 then judges every alternative method
partly by whether it **improves R2's forecast skill**. That gives the
method-comparison work an objective yardstick instead of a beauty contest of
maps.

---

## Phase R1 — MVP: facets, k-means baseline, find-similar

*Maps to requirements.md Phases 1 and 3 (browse MVP + similarity).*

### R1.1 Facet architecture (methodology.md §3.7)
- Facet registry in config: age and household, race/ethnicity (opt-in), income
  and poverty, education and work, housing stock, cost of living, urbanity,
  commuting, health, environment/hazard, digital, civic/response, jobs/flows.
  Each variable has exactly one primary facet.
- Per facet:
  - an embedding of 2–8 dims (transform, then PCA/FA, then scale to unit total
    variance);
  - distances converted to percentiles;
  - a facet cohort (k-means with k = 4–8, or percentile bands if the
    taxometric test says the facet is one continuous dimension);
  - a facet-specific external benchmark.
- Combined: the equal-weight concatenation, run through k-means as the national
  typology (methodology.md §2, E0), plus the facet signature tuple for each tract.

### R1.2 Find-similar MVP
- Brute-force in-browser kNN over facet embeddings, with live weight sliders. Each
  facet can be set to match, ignore, or differ, and results can be restricted to a
  facet cohort.
- Results come with a facet × match heatmap explanation.

### R1.3 Panel construction (needed by R2, started here)
Three layers (methodology.md §5.2, "display yes, inference no"):
1. **Display: every ACS 5-year vintage** 2005–09 … 2020–24 (about 16 points per
   tract). The 11 vintages 2005–09 … 2015–19 are on 2010 tracts and are harmonized
   to 2020 tracts (Census relationship files weighted by 2020 block population, or
   NHGIS crosswalks). From 2016–20 onward, vintages are natively on 2020 tracts
   (V-01). Dollars are CPI-adjusted to a common year, MOEs are stored, and 2016–20
   and 2017–21 get a soft COVID-era flag. Plotted at window midpoints
   and labeled "5-year rolling".
2. **Annualized modeled series.**
   - A state-space smoother per tract and headline indicator, observed through
     5-year averaging with overlap-correlated errors.
   - Anchored by county/place 1-year ACS (no standard 2020 release: a missing
     anchor year, V-09), Population Estimates Program, decennial counts, and annual
     tract administrative series:
     - LODES 8 (2002–2022);
     - HMDA;
     - FHFA tract HPI (developmental, 2010 tracts, V-02);
     - IRS SOI by ZIP;
     - USPS vacancy, only if the project is eligible for restricted access, and
       never published (V-04).
   - Validated against held-out county 1-year ACS.
   - Stored as `panel/tract_annual_modeled.parquet` with uncertainty bands.
3. **Inference: non-overlapping windows** 2009–13, 2014–18, 2019–23, plus decennial
   1990/2000/2010/2020 for the "core temporal" preset, and optionally LTDB
   1970–2020 for long histories.

### R1 exit criteria
- Validation battery v1 passes: methodology.md §4.1–4.4, plus the §4.4b 311 checks
  in Baltimore and at least 2 other cohort cities.
- Facet cohorts are named and reviewed.
- The find-similar query returns in under 1 s nationally.
- The harmonized panel is built, and its row counts and spot checks are documented.
- The modeled annual series passes the county 1-year holdout check (80% intervals
  cover 75–85% of held-out county values).

---

## Phase R2 — Tract-level PECOTA (flagship worked example)

*Maps to requirements.md Phase 3 extension, "forecast mode". It is ALT-12
promoted to the second step after the k-means baseline.*

### R2.0 What PECOTA does, and how each piece maps to tracts

PECOTA (Nate Silver, Baseball Prospectus, 2003) forecasts a player from the later
careers of his most similar historical players ("comps"), rather than from one
regression. Its published "player card" made the method legible to a general
audience. That communication model is what we want to copy.

| PECOTA element | What it does in baseball | Tract analogue |
|---|---|---|
| **Player** / **season** | Unit / time step | Tract / **annual "seasons" for display** (the modeled annual series, plus raw rolling vintages). The **forecast step is a non-overlapping 5-year window** (decennial points for long history). |
| **Age** and **aging curves** | Players peak and decline on typical curves | **Neighborhood life cycle** (Hoover & Vernon's stages: development → transition → downgrading → thinning → renewal). Proxies: median year built, years since the tract's population took off, housing-stock turnover. Life-cycle curves are the "aging curves". |
| **Position, body type** | Constrain comparable careers | **Facet signature** (urbanity, housing-stock type, metro type and size), used as hard or soft match constraints |
| **League and park translations** (Davenport) | Put stats from different contexts on one scale | **Regional translation.** Deflate dollars by BEA regional price parities. Express key stats as **metro-relative percentiles**, so a Cleveland tract can comp to a Seattle tract in relative terms. |
| **Run environment / era adjustment** | League-wide offense shifts by year | **Era adjustment.** Remove national and metro change in each period (post-recession 2009–13→2014–18 vs COVID-era 2014–18→2019–23) before transferring comps' changes, then add back the current era's expectation. |
| **Similarity score** (0–100) | How close each comp is | Facet-weighted distance converted to 0–100. Shown per comp with a per-facet breakdown. |
| **Comps list** (top 3 on the card; many used internally) | Story and forecast basis | Top 3 shown by name and map; top ~50–200 used, weighted by similarity. **Comps come from other counties by default.** |
| **Baseline** | Weighted recent stats, regressed to the mean | Weighted recent periods, MOE-aware **shrinkage** toward the metro/cohort mean (noisy small tracts shrink more) |
| **Projection** | Baseline + comps' subsequent changes | Baseline + the similarity-weighted distribution of comps' era-adjusted next-period changes, per facet and per headline indicator |
| **Percentile forecasts** (10th…90th) | Range of outcomes | Fan of 10/25/50/75/90 for each indicator: real median household income, median rent, home value, population, poverty rate, BA+ share, vacancy, renter share |
| **Breakout / Improve / Collapse / Attrition rates** | Probabilities of big moves | **Upgrade / Improve / Decline / Thinning rates.** Upgrade = metro-relative income percentile rises ≥ 10 points; Improve = any rise; Decline = falls ≥ 10 points; Thinning = population falls ≥ 10%. Turnover (high in-mover share combined with a composition shift) is optional. Neutral names; see R2.6. |
| **Similarity index / comparability** | How typical the player is; low means wider error | Mean similarity of the top-k comps. "Unusual tract" flag, with intervals widened. |
| **Upside** | Expected value above a baseline | Optional: the mean positive deviation, i.e. the upside tail of the fan |
| **Marcel** (Tango's minimal benchmark) | Any forecast system must beat "Marcel the Monkey" | **Tract-Marcel.** Weighted last two periods + shrinkage to the metro mean + the average life-cycle drift. **Mandatory baseline.** |

### R2.1 Build steps
1. **Translations.** Regional price-parity deflation, metro-relative percentiles,
   and era de-meaning per period.
2. **Tract-Marcel baseline.** Its backtest score is the bar to beat.
3. **Comps engine.**
   - Match a target's state at period *t* (optionally with its trajectory
     *t−1 → t*) to candidate tracts' states at an **earlier** period *s*, whose
     *s → s+1* outcomes are already known.
   - Facet-weighted, MOE-aware distance (down-weight noisy facets for small tracts).
   - Constraints: exclude the same county, require the same urbanity stratum,
     and cap comps from any single metro so one city can't dominate.
4. **Projection engine.** Similarity-weighted empirical distribution of comps' changes
   (kernel-weighted quantiles), added to the shrunken baseline. Rates are the
   weighted share of comps crossing each threshold.
5. **Comparability:** a similarity index, with intervals widened when the comps
   are weak.
6. **Card renderer and visualizations** (R2.3).

### R2.2 Backtest protocol (strictly "as-of"; no peeking)
- **Fit era:**
  - Comps come from transitions 2000→2010 (decennial core) and 2009–13→2014–18 (ACS).
  - Targets are tracts at 2014–18.
  - Forecasts of 2019–23 are scored against the actual 2019–23 values.
- **Second fold**, where long history allows: comps from 1990→2000 forecast
  2000→2010. Use the core temporal variables only.
- **Anti-leakage rule (overlapping vintages).**
  - The forecast target is always the **next non-overlapping window** (five years
    ahead), never an adjacent overlapping vintage. Predicting 2015–19 from 2014–18
    is nearly free, because the two share 4 of 5 years of sample.
  - Predictors may use only vintages **published by the as-of date**, including
    the annualized-model fit on those vintages only (re-fit per as-of date; no
    smoothing with future data).
  - Annual points between the as-of window and the target appear in graphics only.
    They are not scored as separate forecasts.
- **Point accuracy:** MAE/RMSE per indicator vs Tract-Marcel, linear regression,
  gradient boosting (a supervised ML benchmark), and the R1 cohort Markov model
  (transition probabilities between k-means cohorts).
- **Distribution quality:**
  - CRPS (a combined score for whole forecast distributions);
  - interval coverage (80% bands should cover 75–85%);
  - PIT histograms, which should be flat.
- **Event quality:** Brier scores and reliability diagrams for the
  Upgrade/Decline/Thinning rates.
- **Error sources:**
  - Score separately by urbanity, region, tract size, and comparability decile.
  - ACS sampling error alone sets a floor on achievable accuracy for small tracts.
    Report scores against MOE-perturbed "truth" too.
- **Gates:**
  - Beat Tract-Marcel on CRPS for ≥ 5 of 8 headline indicators.
  - 80% interval coverage within 75–85%.
  - Event-rate calibration slope within 0.8–1.2.

### R2.3 Visualizations to replicate (the "tract card" and companions)
1. **Tract card** (one page, printable, PECOTA-card layout):
   - Header: tract name, NSA/neighborhood, metro, facet signature (the
     "position"), life-cycle stage (the "age").
   - **History table:** one row per period. Projected rows are italicized, with
     the 50th-percentile projection.
   - **Percentile table:** 10/25/50/75/90 for each headline indicator.
   - **Four rate tiles:** Upgrade · Improve · Decline · Thinning.
   - **Comparability** gauge.
   - **Top 3 comps:** name, city, similarity score, and a per-facet similarity bar.
2. **Comp trajectory "spaghetti" plot.** The target's history as a bold line. Each
   comp's path is aligned at the match point (like age-aligned careers), with
   opacity set by similarity, and a weighted fan of 10–90% bands over it.
   - Lines use the **annualized modeled series**, with raw 5-year rolling vintages
     as faint dots at window midpoints.
   - The forecast fan is anchored at the non-overlapping 5-year horizon.
   - Intermediate years are interpolated from comps' annual paths and visibly
     styled as modeled.
3. **Life-cycle ("aging") curves:** typical change by life-cycle stage and urbanity,
   with the target's position marked.
4. **Comp map:** a national map of comps colored by similarity, with an inset of
   the target tract.
5. **Facet radar / parallel coordinates:** the target vs its top comps, in metro-relative percentiles.
6. **Backtest dashboard:** calibration plot, coverage by decile,
   PECOTA-tract vs Marcel skill by indicator, and a "best and worst calls" gallery.
7. **"What it said then vs what happened"** for a single tract: the 2014–18
   card's fan with the actual 2019–23 value overlaid.

### R2.4 Worked example and storytelling
A narrative explainer, published as an artifact/page and in the repo. It should
be readable by a non-technical audience, the way PECOTA cards and their essays
were:
1. **"Meet the tract."** A Baltimore tract, chosen to tie into balt311:
   - the BNIA neighborhood context;
   - its history across periods;
   - its facet signature in plain English.
2. **"Who does it look like?"** Its top comps across the country, why they
   match (facet bars), and where they differ.
3. **"What happened to them next?"** A spaghetti plot walk-through: some comps
   upgraded, most held steady, some thinned.
4. **"So what's likely here?"** The fan chart and rate tiles, with the
   comparability caveat.
5. **"Would this have worked before?"** The same tract's 2014–18 card against the
   actual 2019–23 outcome, then the national backtest scorecard vs Marcel.
6. **"What this is not."** Uncertainty, the ecological fallacy, era shocks
   (COVID), and the use restrictions in R2.6.

Supporting cases, chosen by criteria rather than in advance:
- one **stable, typical** tract (high comparability, narrow fan);
- one **unusual** tract (low comparability, wide fan);
- one **rural/small-town** tract;
- one **backtest miss**, analyzed honestly. Why the comps failed is part of the story.

### R2.5 Optional extensions
- **Within-city 311 projection** for the balt311 cohort cities: what service-request
  mix do comps in other cohort cities have? This combines with methodology.md §4.4b.
- **Long-history comps (1970–2020, LTDB):** "career-length" matching using the
  core temporal variables.

### R2.6 Responsible-use requirements (blocking for publication)
- Forecasts describe *areas*, not people. They carry wide uncertainty and are
  shown with it, always as fans, never single numbers alone.
- Neutral rate names (Upgrade/Decline/Thinning), never "gentrifying",
  "dying", or similar labels.
- **Self-fulfilling-prophecy risk:**
  - Decline probabilities could steer disinvestment, much like redlining.
  - Publicly, show rates in context with the comparability caveat, and consider
    gating the Decline/Thinning tiles behind the methodology explainer.
  - The use statement prohibits lending, insurance, pricing, and enforcement
    targeting.
- Human review of every published worked example.

### R2 exit criteria
- The R2.2 backtest gates pass.
- Tract cards render for all tracts.
- The worked example is published.
- The responsible-use review is complete.

---

## Phase R3 — Alternative approaches and consensus relationships

*Maps to requirements.md Phase 3 continuation and Phase 6 (analyst workbench).*

### R3.1 Run the alternatives backlog
Run in the order of methodology_alternatives.md §E: T1 set → spikes → exploratory,
plus the input-perturbation experiments (ALT-P1…P9). Each run logs to `experiments/`.

### R3.2 Three yardsticks for every method
1. The **methodology.md §4 battery** (stability, MOE robustness, external
   validity, within-city 311).
2. **PECOTA skill.** Swap the method's representation or similarity into the R2
   comps engine and re-run the R2.2 backtest. Does it beat the R1 facet-PCA comps on
   CRPS and calibration? This is the decisive, external test.
3. **Interpretability review:** can the output be named and explained on a tract card?

### R3.3 Consensus relationships
- **Method agreement atlas:**
  - A matrix of agreement between methods (adjusted Rand index / normalized mutual
    information for cohort methods, top-k neighbor overlap for similarity methods).
  - Hierarchically clustered, to show which method families agree.
- **Consensus cohorts** (ALT-E1): co-association across methods × perturbations;
  per-tract agreement as confidence.
- **Consensus comps:**
  - Comps that several representations agree on are the "robust comps".
  - Test whether forecasts from consensus comps beat any single method's comps
    (an ensemble, like averaging forecast systems).
- **Cross-structure relationships:**
  - Do PECOTA comps share facet signatures and k-means cohorts with their target?
  - How much does trajectory matching change the comps relative to
    cross-sectional matching?
  - Where do cohort boundaries cut through dense comp networks? Those are
    candidates for merging cohorts.
- **Deliverable:** a "consensus report" artifact. It covers the agreement atlas,
  the stable vs contested tracts map, which methods improve forecasts, and a
  recommendation of what to promote into the product.

### R3 exit criteria
- At least the T1 set has been evaluated on all three yardsticks.
- The consensus typology and robust comps are published (or explicitly rejected,
  with reasons).
- The product defaults have been updated based on the evidence.

---

## Cross-phase artifacts
| Artifact | Phase | Form |
|---|---|---|
| Facet registry + facet cohorts | R1 | `config/facets.yaml`, Parquet |
| Harmonized panel | R1 | `panel/tract_{period}.parquet` (2020 tracts) |
| Validation report | R1–R3 | `validation_report.json` + HTML |
| Comps + projections | R2 | `pecota/comps_{asof}.parquet`, `pecota/projections_{asof}.parquet` |
| Tract cards | R2 | In-app view + printable HTML |
| Worked example | R2 | Published explainer page |
| Backtest dashboard | R2 | Published page |
| Experiment log | R3 | `experiments/*.parquet` |
| Consensus report | R3 | Published page |
