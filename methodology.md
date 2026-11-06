# pdb-browser — Methodology: tract cohorts, similarity, and validation

> **Status:** Research draft v0.1 (2026-11-06). This is the companion to `requirements.md` §6
> (profile and similarity). It reviews prior art, recommends a v1 approach, and
> defines how we will know the cohorts are any good. Fact checks are logged in
> [`verification_log.md`](verification_log.md) (`V-xx` ids). Items marked **[verify]**
> were found in search summaries but not yet read in full.

## 0. Problem framing

- **Data shape:** about 85k 2020-vintage tracts (about 72–84k with meaningful
  population) × about 1,000 candidate variables (PDB about 500, ACS tables the PDB
  omits, and federal or other supplemental sources). Block groups: about 242k ×
  fewer, noisier variables.
- **Three related outputs.** Each needs its own validation.
  1. **Similarity** (soft, per-query): "which tracts are most like this one, on
     these features?" This is kNN in a transformed feature space.
  2. **Cohorts / typology** (hard, global): a nested classification (about 8–12
     supergroups → about 30–60 groups) with names and descriptions. This is a
     geodemographic classification in the UK Output Area Classification (OAC) /
     Esri Tapestry tradition.
  3. **Trajectories** (temporal): cohorts of tracts that *changed alike* over time.
     Examples: "stable affluent suburb" or "rapidly gentrifying inner-ring".
- **Main risks at this scale:**
  - **Measurement noise.** ACS tract margins of error (MOEs) are large. Spielman
    reports that about 99% of tracts carry a ≥10% MOE on typical estimates.
  - **Redundancy.** About 1,000 variables collapse to perhaps 10–20 real dimensions.
  - **Domain imbalance.** Whichever topic has the most variables dominates the distance.
  - **Urbanicity dominance.** Density and rurality swamp everything else nationally.
  - **Stigmatizing labels.**

## 1. Prior art

### 1.1 Geodemographic classifications (closest analogues)
| Work | Unit / scale | Method | What we take from it |
|---|---|---|---|
| **Spielman & Singleton (2015)**, *Annals AAG*, "Studying neighborhoods using uncertain data from the ACS". Open US tract classification (Liverpool / ICPSR 100235). | All US tracts, ACS 2006–10 | Multivariate geodemographic typology framed as the *answer* to ACS MOEs: composite profiles are more reliable than single variables. Variables organized as Concepts → Domains → Measures. | The core argument for this project. An open, reproducible US precedent and a structure for variable domains. Their uncertainty treatment (assignment confidence) is reused in §4.3. |
| **UK Output Area Classification 2011 / 2021** (Gale et al. 2016; Wyszomierski et al., *Geographical Journal*, GeoDS + ONS) | About 189k–232k output areas | Domain-balanced variable selection that drops highly correlated pairs. Transform (IHS/log) → range standardize → **k-means**, hierarchical by re-clustering within each parent (supergroup → group → subgroup). Pen portraits for labels. Open code and specs. | The most transparent end-to-end recipe: variable selection, transforms, nested k-means, labeling. Liverpool also published a **PCA-based automated variable selection** framework. |
| **Esri Tapestry (2025 overhaul; 2026 vintage)** | US block groups and tracts | k-means with Euclidean distance, multistage seeding (not random). 60 segments in 12 LifeMode groups, plus an explicit **"unclassified"** segment. | Proprietary, but the published methodology confirms that k-means with good seeding is still the industry standard. The "unclassified" bin for outliers is worth copying. |
| Claritas PRIZM, Experian Mosaic | Household / BG | Proprietary, consumer-data enhanced | Benchmark only. They show the value of non-census (behavioral) inputs. |
| **Kolak et al. (2020)**, *JAMA Netw Open*, SDOH typology | 71,901 tracts | Dimension reduction → **4 indices** (advantage, isolation, opportunity, immigrant cohesion; 71% of variance) → **k-means → 7 types**. Externally validated against premature mortality (Chicago, spatial regression). | A two-stage *reduce then cluster* design that yields interpretable axes. A model for external validation. Public data at SDOHAtlas.github.io. |

### 1.2 Composite indices (one dimension, but useful inputs and benchmarks)
| Index | Notes |
|---|---|
| **Census Low Response Score** (in the PDB) | OLS-predicted tract non-response (0–100). A ready-made "hard-to-count" axis from the source dataset itself. |
| **CDC/ATSDR SVI** | 4 themes, percentile-ranked, tract level. Useful benchmark and covariate. |
| **Area Deprivation Index** (Singh; UW Neighborhood Atlas) | BG-level deprivation. A benchmark. |
| **Child Opportunity Index 3.0** (diversitydatakids) | 44 indicators, 3 domains, 14 subdomains. Built at **block** level and aggregated up. Weights come from **how well each indicator predicts intergenerational-mobility outcomes**. This is the main precedent for *outcome-informed weighting* (§3.4). |
| Census Community Resilience Estimates | Small-area estimation. **Still published:** the 2024 CRE was released 2026-01-29. Only the *Equity Supplement* is listed as discontinued (V-07). |
| USDA RUCA / NCHS urban-rural | Urbanicity strata (§3.6). |

### 1.3 Neighborhood change and trajectories
| Work | Method | Takeaway |
|---|---|---|
| **Delmelle (2015, 2016; 2025 with Delmelle)** | Cluster each decade with **pooled k-means**, then build each tract's label sequence, then run **sequence analysis** (optimal matching, TraMineR) and cluster the sequences. Five-decade, multi-city studies; NYC 1980–2020 gives 14 pathways. | The canonical recipe for trajectory typologies. |
| **geosnap** (PySAL; Knaap, Kang, Rey, Wolf). SciPy 2024. Latest v0.17. | Python: tract harmonization across boundary vintages, cluster/regionalize, **Markov and spatial-Markov transitions** (`giddy`), sequence analysis, LINCS (local indicator of neighborhood change). | The reference implementation. Adopt it directly for the temporal layer. |
| Sequence-analysis robustness study (50 metros, NSF PAR) | Compared optimal-matching cost settings | Trajectory typologies are **sensitive to cost choices**, so sensitivity analysis is required (§4.6). |
| **LTDB** (Brown S4) | 1970–2020 tract data harmonized to **2010** boundaries. 2020 data limited to race/ethnicity. | A long history at the price of interpolation error ("spatial stationarity" assumption). Many rural tracts are missing 1970/80. |
| **NHGIS** time series and geographic crosswalks; Census 2010↔2020 tract relationship files; Drexel UHC 2020→2010 crosswalk | Areal and population-weighted crosswalks | Harmonization options (§5.2). |

### 1.4 Representation learning
- **Autoencoders vs PCA on census data.** A 2026 study on 408 variables × 188,880
  England & Wales output areas reports **about 19% lower reconstruction error than PCA** at
  operational dimensionality (Liverpool/Leicester conference paper, Zenodo 2026-04-04;
  abstract-level check only, peer-review status unknown, V-08). Tooling: `DeepGeoDemo` (YAML-driven
  deep-embedding geodemographics).
- **Imagery-derived context.** Singleton's convolutional autoencoder on Sentinel-2
  produced a 7-group "Postcode Context Classification" of Great Britain. Pre-trained CNN
  features (ResNet → PCA) and **global satellite embedding datasets** (for example
  Google DeepMind's AlphaEarth Foundations: 64-dim, 10 m, annual layers 2017–2024 in
  Earth Engine as `GOOGLE/SATELLITE_EMBEDDING/V1/ANNUAL`, V-06; check the license
  before redistributing derivatives) give
  per-pixel built-environment vectors that can be mean-pooled to tracts.
- **Assessment.** Learned embeddings help *similarity search* and can add
  built-form signal that census data lacks. They hurt *interpretability* of named
  cohorts. Use them as an optional feature block (§6), not the backbone.

## 2. Recommended v1 approach (summary)

```
~1,000 raw vars
  → (A) curate & convert to rates/shares, attach MOEs              → ~250 candidates
  → (B) domain assignment + redundancy pruning (|r|>0.85, PCA-loading selection) → ~60–120 features
  → (C) transform: shares→logit/CLR, skewed→log1p/IHS, winsorize, robust-scale
  → (D) domain-balanced weighting (each domain sums to equal weight)
  → (E1) similarity: kNN in weighted space (+ optional domain-PCA space)
  → (E2) cohorts: domain PCA scores → k-means (k≈8–12) → nested k-means within each
         supergroup (≈4–6 each); GMM fit on the same space for soft membership;
         "unclassified" bin for low-confidence / outlier tracts
  → (E3) trajectories: pooled cohorts across non-overlapping periods → sequences
         → Markov transitions + optimal-matching clusters (geosnap)
  → (F) validation suite (§4) gates every release
```

Rationale:
- k-means with nested re-clustering is the method every successful public
  classification uses (OAC, Tapestry, Kolak). It is fast at 85k rows and easy to explain.
- Clustering on **domain-level PCA scores**, not raw features, de-noises the input,
  removes redundancy, and keeps each axis interpretable (the Kolak design).
- HDBSCAN-on-UMAP and autoencoder embeddings are **comparators**, run in the
  experiment harness. They are not the default.

## 3. Design decisions in detail

### 3.1 Variable curation (A–B)
- **Rates, not counts.** Raw counts encode tract size, and size would dominate the
  first principal component. Convert every count to a share using its PDB
  denominator. Keep **population density** and **total population** as explicit,
  separately weighted features.
- **Domains** (start from the OAC / Spielman structure, adjusted for US sources):
  1. Demographic structure: age bands, household type
  2. Race/ethnicity, nativity, language
  3. Education
  4. Income, poverty, assistance
  5. Employment and occupation/industry
  6. Housing: tenure, stock age, structure type, value/rent, burden, vacancy
  7. Mobility and commuting: mode, time, vehicles
  8. Health and insurance (+ PLACES)
  9. Digital access
  10. Census operations: self-response, LRS
  11. Built environment / urbanicity: density, land cover, walkability
  12. Environment and hazard: EJ, NRI
- **Redundancy pruning.** Within a domain, drop one of each pair with |Spearman r| >
  0.85, keeping the more reliable variable (lower median CV). Then apply the Liverpool
  PCA-loading selection, which keeps the variables that load highest on the retained
  components.
- **Reliability screen.** Drop variables whose **median tract CV > 40%** (for example
  fine-grained age × sex × race cells). Keep them for browsing, but leave them out of
  clustering.
- **Race/ethnicity is a design choice.** Including it reproduces segregation
  patterns, which is informative. Excluding it gives "race-blind" cohorts, which suit
  equity comparisons. The second use is the balt311 analysis: compare outcomes
  *across* race *within* cohorts. Ship both feature presets, label them
  clearly, and default the equity lens to race-excluded cohorts.

### 3.2 Transforms (C)
- Shares near 0/1: **logit** (with epsilon) or the inverse hyperbolic sine.
- Compositional blocks (race shares, tenure, age bands, mode share): **CLR**,
  matching balt311's service-mix treatment. This avoids the closure problem, where
  shares sum to 1 and become spuriously correlated.
- Dollars and densities: log1p or IHS.
- Winsorize at p0.5/p99.5, then **robust scale** (median / IQR). OAC used range
  standardization; we use robust scaling so a handful of extreme tracts
  (prisons, campuses, airports) don't compress everyone else.
- **Special-population tracts.** Group quarters share > 50%, population < 200, or
  water/park tracts are flagged up front. They get an "unclassified / special"
  cohort instead of distorting centroids.

### 3.3 Dimensionality reduction
- **Per-domain PCA**, keeping components to about 80% within-domain variance (1–4 each),
  then concatenate the domain scores, about 25–35 dims total. This gives interpretable axes
  and equal domain voice by construction (scale each domain's block to unit
  total variance).
- Comparators: global PCA, factor analysis (varimax) as in the Kolak indices,
  a shallow autoencoder (`DeepGeoDemo`), and UMAP for visualization only. **Never cluster on
  2-D UMAP for production cohorts**, because it distorts density and global distances.

### 3.4 Weighting
- **Default: equal domain weights.** Transparent and defensible.
- **User weights** (requirements R-SIM-1): sliders per domain for similarity queries.
- **Outcome-informed weights (experimental, COI-style).** Learn feature weights
  that maximize the prediction of held-out outcomes (life expectancy, PLACES
  outcomes, Opportunity Atlas mobility) using metric learning or permutation importance. These
  answer "similar *with respect to* health" and are offered as an
  outcome-specific preset, never as the default. The default must not optimize for
  any single outcome.

### 3.5 Clustering (E2)
- **k-means**, k-means++ with ≥50 restarts, or OAC/Tapestry-style multistage
  seeding (Ward on a sample → centroids → k-means on everything).
- **Choosing k.** No single statistic decides. Shortlist k with elbow, Calinski–Harabasz,
  and silhouette (on 10–20k subsamples), then pick the value that passes **stability**
  (§4.2) and **interpretability** review. Usability ceiling: ≤12 supergroups,
  ≤60 groups.
- **Nested hierarchy.** Re-cluster within each supergroup (OAC), which guarantees nesting.
- **Soft membership.** A Gaussian mixture (diagonal or tied covariance) on the same
  space, plus k-means distance ratios, gives each tract a membership confidence.
  Low confidence or a large distance to every centroid sends the tract to the "unclassified" bin (Tapestry).
- **Comparators** in the experiment harness: HDBSCAN (density-based; finds
  outliers natively), Ward agglomerative on a subsample, spectral clustering on a kNN graph.

### 3.6 Spatial and urbanicity considerations
- **National vs stratified.** One national typology is mostly an
  urban/suburban/rural sort. Offer (a) a national typology and (b) **within-stratum
  typologies** (RUCA or NCHS urban-rural classes) for "similar *rural* tracts".
- **Spatial context features.** Add queen-neighbor spatial lags of key domain scores
  (requirements R-SIM-5) as an optional block. Without them, an affluent tract
  next to concentrated poverty is treated the same as one inside an affluent
  region.
- **Regionalization** (contiguous cohorts, spopt SKATER / max-p) is a separate
  product for *building neighborhoods*, not for finding peers.

### 3.7 Facet architecture (feature cohorts)
Facets are first-class objects, so users can run quick similarity searches within
one feature set and still cluster across sets.
- **Registry** (`config/facets.yaml`): age and household, race/ethnicity (opt-in),
  income and poverty, education and work, housing stock, **cost of living** (rent,
  value, burden + BEA regional price parities, HUD FMR/SAFMR, optional family-budget
  sources, copied down from county/metro), **urbanity** (density, RUCA, footprints,
  land cover, walkability), commuting, health, environment/hazard, digital,
  civic/response, jobs/flows.
  - Each variable has one primary facet, so nothing is double-counted.
  - Facet coherence is checked by clustering the variables themselves. Variables
    that track another facet more closely, and facet pairs with r > 0.8, are flagged.
- **Per facet:**
  - an embedding of 2–8 dims (transform → PCA/FA → unit total variance);
  - **distances converted to percentiles**, so "close on income" and "close on
    urbanity" are comparable despite different intrinsic dimensionality;
  - a facet cohort (k-means with k = 4–8, or percentile bands if ALT-26 says
    the facet is one continuous dimension);
  - a facet-specific external benchmark (urbanity vs NCHS/RUCA; income vs IRS
    SOI; cost of living vs RPP/FMR; demographics vs decennial counts);
  - a per-facet MOE stability test.
- **Rapid similarity:** 85k × ~12 facets × ~5 dims ≈ 5M values, so it is shipped to
  the browser and searched exhaustively in milliseconds. The combined distance
  √(Σ w_f·d_f²) is recomputed live, which supports:
  - **match / ignore / differ** settings per facet (contrast queries such as
    "same income, but rural");
  - **conditional** queries (filter by one facet's cohort, rank by another);
  - **adjusted** facets (e.g. income relative to cost of living).
- **Across facets:**
  - the **facet signature** tuple (a browsable faceted typology);
  - equal-weight concatenation → k-means (the §2 baseline);
  - LCA on facet labels (meta-cohorts);
  - multi-view methods (SNF/MOFA, ALT-37);
  - a **facet relationship** report (correlation between facet embeddings, mutual
    information between facet labels; "decoupled" tracts such as high income with
    low cost of living).

## 4. Validation framework

No single score validates a geodemographic classification, so we use a battery of
checks. Each release produces a `validation_report.json` and an HTML summary,
and the thresholds below are release gates. They start as working defaults and
get tuned in Phase 3.

### 4.1 Internal validity (compactness and separation)
- Silhouette (subsampled, repeated), Calinski–Harabasz, Davies–Bouldin, and
  within-cluster sum of squares by cluster.
- Per-cluster **size bounds**: no supergroup < 1% or > 25% of tracts.
- Use these to *compare candidate k and methods*. They are not pass/fail on their own:
  real geodemographic data has no crisp clusters, and silhouettes of 0.1–0.25 are normal.

### 4.2 Stability (does the structure replicate?)
- **Bootstrap Jaccard** (Hennig `clusterboot` logic, reimplemented in Python): for
  ≥100 bootstrap / 80%-subsample refits, record each cluster's best-match Jaccard.
  **Gate: mean Jaccard ≥ 0.75 per supergroup.** At ≤ 0.5 the cluster is "dissolved"
  and must be merged or dropped. Hennig's caveat applies: inflexible methods can be
  stable but wrong, so stability is necessary, not sufficient.
- **Split-half geographic replication.** Fit on a random half of states, assign the
  other half to the nearest centroids, refit there independently, and compare with the
  adjusted Rand index (ARI). Low ARI means the typology is region-specific.
- **Seed / initialization stability:** ARI across 20 random seeds ≥ 0.9.
- **Hyperparameter sensitivity:** ARI vs. the baseline when dropping a domain, swapping
  logit for raw shares, or switching robust scaling to z-scores. Report which choices
  move the result.

### 4.3 Measurement-uncertainty validity (ACS MOEs)
- **Monte Carlo assignment.** For R = 100 draws, perturb each estimate by
  N(0, SE), where SE = MOE / 1.645 (truncated to valid ranges; for derived shares use
  the Census approximation formulas for ratio and proportion SEs). Re-transform, then
  **re-assign** to the fixed centroids without refitting. The per-tract **assignment
  probability** is the share of draws that land in the modal cohort.
  - Report the distribution. **Gate: ≥ 80% of tracts with assignment probability ≥ 0.7
    at supergroup level.** Show the probability in the UI next to the cohort label
    (Spielman & Singleton's argument made operational).
  - Run the same exercise for kNN: **neighbor-set overlap** (Jaccard of top-10 neighbors)
    under perturbation. This shows whether "similar tracts" is a stable answer.
- Report MOE-driven instability by tract population, because small tracts will dominate
  the unstable set.

### 4.4 External / predictive validity (does it mean anything?)
Hold out outcome variables that are **not** clustering inputs. Candidates:
USALEEP life expectancy, CDC PLACES measures, Opportunity Atlas mobility,
eviction rates, balt311 service metrics, election turnout, 2020 self-response
(if LRS inputs are excluded).
- **Variance explained (η²)** of each outcome by cohort label, compared with baselines:
  county, state, income decile alone, SVI decile, and a random partition with the same
  sizes. Cohorts should beat income-decile-only on most outcomes, not just one.
- **kNN predictive check** (validates *similarity* directly): predict each held-out
  outcome as the mean over a tract's k nearest neighbors, **excluding its own county**
  so spatial autocorrelation can't leak in. Compare the error against same-county
  neighbors, random tracts, and income-matched tracts. If "similar" tracts don't share
  outcomes better than income-matched ones, the feature set isn't adding value.
- **Benchmark agreement:** ARI / normalized mutual information against Kolak's 7 types and the SVI and
  ADI quintiles. Moderate agreement is expected and healthy. Near-1 agreement means we
  rebuilt an index; near-0 is a warning.

### 4.4b Within-city validation with patchy municipal data (311 and similar)

Municipal operational data (311, code enforcement, permits) covers only the
cohort cities (balt311 has 15). It is still the most independent external check
available. The national typology never sees it, and balt311's Areas tab already
shows that tract service-request mix separates tracts meaningfully. Patchy
coverage limits **where** we can validate, not **whether** we can: cohorts absent
from every covered city are reported as *untested*, not as failed.

**Role rule:** for the national typology, municipal data is **validation only,
never an input**. If a scoped within-city similarity mode uses 311 as a
feature, that mode must be validated with different held-out outcomes
(PLACES, eviction, code enforcement) or whole held-out cities. The same data
never serves as both input and validation for one model.

**Procedure**
1. **Stratify by city, then meta-analyze.** Run every test within one city.
   Pool the per-city effect sizes with each city weighted by its tract count,
   and report how much they vary across cities. Never pool raw 311 values
   nationally: taxonomies, reporting culture, and closure workflows differ by
   city (see the balt311 `pct_same_day_close` contamination flags).
2. **Avoid crosswalking request taxonomies.** Use distance-based tests on each city's own
   service-type mix:
   - CLR-transform the SRType shares (the balt311 Areas treatment).
   - Run **PERMANOVA** to ask whether same-cohort tracts have more similar mixes than
     different-cohort tracts in the same city.
   - Report pseudo-R² per city.

   Scalar delivery metrics (median days to close, closure rate) can be compared
   directly with within-city η².
3. **Separate reporting propensity from need.** Request *volume* (per 1k) mixes
   conditions with propensity to report. Propensity varies with income and
   demographics, so volume-based validation largely rediscovers income.
   - Prefer service **mix** and **mix-adjusted delivery** metrics (balt311 Tab 6
     direct standardization) over raw volume.
   - **Mandatory baseline:** an income-decile-only partition within the same
     city. Cohorts count as validated on 311 only if they beat it, and also beat
     a random partition with the same sizes and the city's own geographic units
     (e.g. Baltimore CSAs).
4. **Control for spatial autocorrelation.** Both cohorts and 311 outcomes are
   spatially clustered, so naive tests overstate agreement.
   - Use **spatially constrained permutation** nulls (e.g. Moran spectral
     randomization or toroidal-shift-style rotations) instead of free shuffles.
   - Add a **distance-matched check**: do cohort-mates resemble each other more than
     non-cohort tracts at the same distance?
   - In the kNN predictive check (§4.4), "exclude own county" does not apply
     inside one city. Exclude tracts within **2 queen-adjacency rings** instead.
5. **Handle sparse tracts.**
   - Suppress cells with n<5 (balt311 `_MIN_GEO_SRTYPE_N`).
   - Shrink tract rates and mixes toward the city mean with empirical Bayes.
   - Weight tracts by request count in all statistics.
   - Report the share of tracts and cohorts excluded for sparsity.
6. **Leave-one-city-out transfer.** Learn the cohort-to-311-profile relationship
   in N−1 cities, then predict the held-out city's tract delivery metrics and
   harmonized high-level category shares. Report the improvement over the
   income-only model. This tests whether cohorts carry meaning *across* cities,
   which the balt311 cross-city work depends on.

**Gates (initial):**
- Cohorts beat the income-decile baseline on mix PERMANOVA pseudo-R² in
  ≥ 2/3 of covered cities.
- The spatially constrained permutation p < 0.05 in the meta-analysis.
- Results are reported per cohort with an *untested* flag where city coverage is
  thin (< 10 tracts across all cities).

### 4.5 Spatial diagnostics
- **Join-count statistics / Moran's I on cohort labels.** Some spatial clustering
  is expected. An extreme value suggests urbanicity or region dominates, and favors
  the stratified variant (§3.6).
- Map review for implausible "salt-and-pepper" patterns in dense cores. Use them to
  check for MOE noise (cross-reference §4.3 probabilities).

### 4.6 Temporal validity
- **Pooled-centroid consistency.** Fit on 2014–18, assign 2019–23 tracts to the same
  centroids, and compare with an independent 2019–23 fit (ARI). Also check that the year-over-year
  transition matrix is diagonal-heavy (neighborhoods change slowly). If 30%+ of tracts
  switch supergroups between adjacent periods, the result is noise, not change.
- **Trajectory robustness:** re-run sequence clustering under ≥3 optimal-matching cost
  settings (constant, transition-rate-based, feature-distance-based) and report
  agreement (prior art shows this is a real sensitivity).
- **Back-cast plausibility:** known gentrification and decline case studies (in literature or
  locally verified, e.g. Baltimore via BNIA) should land in the expected
  trajectories.

### 4.7 Face validity, labeling, and harm review
- **Pen portraits** for each cohort, generated from the top ± z-scored features vs.
  the national mean (as in OAC). An LLM may draft them from the stats table, but a human
  reviews every one.
- **Naming rules:** describe conditions, not people. Use no racial or ethnic terms in
  names, and no pejoratives ("struggling", "blue collar"). Run a local expert review on 2–3
  known cities (Baltimore first, using the balt311 context).
- **Disparity audit:** for the race-excluded preset, report the racial
  composition of each cohort so users can see how much segregation the
  "race-blind" features still encode.
- **Use statement:** cohorts describe areas, not people (ecological fallacy).
  They are not for eligibility, pricing, or enforcement targeting.

## 5. Historical (time-series) data

### 5.1 Benefits
1. **Noise reduction / reliability.** Averaging or smoothing a tract across periods
   shrinks the ACS sampling error that dominates single-vintage tract estimates. This
   is the strongest practical argument, given §4.3.
2. **Trajectory similarity.** "Similar today *and* got here the same way" is a
   much stronger peer than a cross-sectional match. Two tracts with identical 2023
   income can be on opposite trajectories (gentrifying vs declining).
3. **Change typologies** answer a different, high-value question for planners
   (gentrification, displacement risk, sustained decline).
4. **Validation.** A temporal hold-out is the most honest predictive check (§4.6).
5. **Stability of cohort definitions.** Pooled fitting gives one cohort
   vocabulary across years, so labels mean the same thing over time.

### 5.2 Pitfalls and rules
- **Overlapping ACS 5-year windows are not independent.** The Census Bureau
  discourages comparing overlapping periods: the difference only reflects the
  non-shared years. Each 5-year estimate is roughly the mean of five annual values.
  Adjacent vintages share about 80% of their sample, so the series is a lagged,
  damped rolling average with strongly correlated errors: about 15 vintages carry
  about 3 independent observations' worth of information. Use them in three layers
  ("display yes, inference no"):
  1. **Display layer: every vintage as published** (2005–09 … 2020–24, about 16
     points). Plot each at its window midpoint with an MOE ribbon, labeled
     "5-year rolling estimate".
     - Inflate dollars to a common year (each vintage is in its end-year dollars).
     - **Boundary vintages (V-01, verified):** 5-year releases 2005–09 … 2015–19
       (11 vintages) are on **2010 tracts**. **2016–20 (released 2022-03-17) was the
       first on 2020 tracts**, and every later release is too. Harmonize the 11 older
       vintages with the Census relationship files or the NHGIS 2010→2020 crosswalks.
     - 2016–20 was published with standard methodology (the Census Bureau judged
       the 2020 disruption much smaller over 5 years). Keep a soft COVID-era flag on
       2016–20 and 2017–21 only.
  2. **Annualized modeled layer** (for annual trajectories in visuals, and for
     PECOTA comp paths):
     - A per-tract state-space model (Kalman smoother). A latent annual process
       (local linear trend) is observed through 5-year averaging, with observation
       errors correlated by window overlap (approximately the overlap fraction;
       the Census Bureau does not publish the covariance) and MOE-derived variances.
     - **Annual anchors:** county/place ACS 1-year (populations ≥ 65k),
       Population Estimates Program, decennial 2010/2020, and tract-level annual
       administrative series.
       - **ACS 1-year 2020 gap (V-09):** standard 2020 1-year products were never
         published. Only 54 experimental tables exist, at national and state level
         only. The model must treat 2020 as a missing anchor year.
       - **Tract series:** LODES 8 (2020 blocks, 2002–2022 for most states, V-10) and
         HMDA.
       - **FHFA annual tract HPI** (V-02): verified to exist but *developmental*.
         The research release covers about 63k **2010** tracts for 1989–2021, so it
         needs a crosswalk and has coverage gaps, and the current file vintage is
         unconfirmed.
       - IRS SOI by ZIP, via the HUD crosswalk.
       - **USPS vacancy (V-04):** access is restricted to registered government and
         nonprofit users. Use it only if the project qualifies, as an internal model
         input, and never redistribute it in the public app.
     - Output is annual estimates with uncertainty bands, labeled "modeled".
     - **Validation:** aggregate modeled tracts in large counties and compare
       against the held-out published county 1-year ACS.
  3. **Inference layer.** Change tests, trajectory clustering, and backtests use
     **non-overlapping windows** (2009–13, 2014–18, 2019–23, then 2024–28), or the
     layer-2 model with the overlap covariance built in. They never treat raw
     overlapping vintages as independent observations. Decennial 1990/2000/2010/2020
     supplies long history. Do not difference annual PDB vintages directly.
- **Boundary changes.** Tracts are redrawn every decade. Options, in order of preference:
  1. Census **2010↔2020 tract relationship files**, weighted by 2020 block population,
     for the 2010s → 2020 geography.
  2. **NHGIS** geographic crosswalks and standardized time-series tables for
     1990/2000/2010.
  3. **LTDB** (1970–2020 on 2010 boundaries) for long history. It carries known
     interpolation error and thin 2020 coverage.
  4. geosnap's harmonization utilities (areal or dasymetric) to keep everything in one tool.
- **Definitional drift.** Race categories (2000 multi-race, 2020 coding changes),
  income in nominal dollars (CPI-adjust to the latest year), ACS question changes, and
  2020 Decennial **differential privacy noise** at block level. Block-level time
  series are not usable for anything but coarse counts.
- **Variable availability shrinks with history.** Long histories (1970+) cover about 20
  core variables. Define a **"core temporal" feature preset** available in every
  period, separate from the rich cross-sectional preset.

### 5.3 Temporal methods (v1 → later)
| Method | Output | Notes |
|---|---|---|
| **Pooled typology** (fit k-means on all tract-periods stacked) | The same cohort vocabulary in every period | geosnap default. Prerequisite for the rest. |
| **Markov / spatial-Markov transitions** (`giddy`) | Transition probabilities between cohorts, conditioned on neighbors' cohort | "What do tracts like this usually become?" |
| **Change-vector features** (Δ domain scores between periods) | Appended to similarity features | The cheapest way to add trajectory to kNN. |
| **Sequence analysis** (optimal matching → clustering; TraMineR via R or Python ports) | Trajectory typology (Delmelle) | Requires sensitivity analysis (§4.6). |
| **Multivariate time-series similarity** (DTW on domain scores) | Trajectory-aware kNN | On non-overlapping periods (3–5 points), plain Euclidean on stacked vectors is enough. DTW/shape matching becomes worthwhile on the layer-2 annualized series (§5.2), but report it with the modeled uncertainty. |
| **Higher-frequency non-ACS series** (annual LODES, HMDA, FHFA tract HPI, IRS SOI by ZIP, Zillow monthly, 311; USPS vacancy only if eligible, V-04) | Recent-momentum features | Genuinely independent annual data. Good for detecting change *since* the last ACS window. |

### 5.4 Long-term history (1940–2020): tiers, sources, and uses

The MIT Libraries historic-census guide and the sources it points to (NHGIS, LTDB,
Social Explorer, Geolytics NCDB), plus open research datasets, allow tract
trajectories well beyond the ACS era. Coverage and comparability fall off with each
earlier decade, so history comes in **tiers**, each with explicit quality flags.
Checks are logged in `verification_log.md` as V-15 to V-21.

| Tier | Years | Coverage | Variables | Source(s) | Boundary basis | Access |
|---|---|---|---|---|---|---|
| **H0** | 1990, 2000, 2010, 2020 | All tracts | Short-form counts (population, race/ethnicity, age, housing units, occupancy, tenure) | **NHGIS geographically standardized time series**, block-based interpolation with **lower and upper bounds** per value (V-15) | 2010 tracts → NHGIS 2010→2020 crosswalk | Free (IPUMS account) |
| **H0-LF** | 1990, 2000 | All tracts | Long-form (income, education, occupation, poverty) | NHGIS nominal tract tables + NHGIS crosswalks; or the LTDB sample file | Nominal → 2010 → 2020 | Free |
| **H1** | 1970, 1980 (to 2010) | Tracted areas only. Much of rural America was untracted in 1970/80. | About 20 core full-count and sample variables | **LTDB** (Brown S4), 2010 tracts (V-16); 2020 data limited to race | 2010 tracts → 2020 | Free |
| **H1-alt** | 1970–2020 | As above | Broader variable set | **Geolytics NCDB 2020**: all decades normalized to **2020 tracts** (V-17) | 2020 tracts natively | **Commercial** (about $1.5k) or institutional (MIT and other libraries). No redistribution. Use only as an internal cross-check. |
| **H2** | 1940, 1950, 1960 | **Major cities and surroundings only** (1940/50: limited tracted cities; 1960: "select areas") | Population, race, housing, some socioeconomic variables | NHGIS nominal tract tables + **historical tract boundaries (1910–2000)** (V-18). Social Explorer 1940–60 is *not* on consistent boundaries. | **No standard crosswalk before 1990.** Build our own areal/dasymetric crosswalk, weighted by H3 housing estimates. | Free (NHGIS) |
| **H3** (all tracts, long-run built form) | 1940–2010 by decade, plus 2019 | **All continental tracts** (72,539) | **Housing units** and **urbanization year** (density + land-cover threshold) | **HHUUD10** (Markley et al. 2022, *Scientific Data*; "maximum reabsorption" interpolation that out-performs LTDB in out-of-sample validation) (V-19) | 2010 tracts → 2020 | Free (OSF / GitHub) |
| **Legacy covariates** | 1930s | Graded metros only (about 13.5k 2020 tracts) | **HOLC Historic Redlining Indicator** (1–4, area-weighted grades) | Meier & Mitchell / NCRC via openICPSR 141121 V3, on 2000/2010/**2020** tracts (V-20) | Native 2020 | Free |
| *(deferred)* | 1940, 1950 full count | Enumeration-district level | Microdata | IPUMS full count. **Restricted access** for ED-level geography; no public 1950 ED shapefiles (V-21). | ED → tract is unbuilt | Restricted |

**Rules**
- **One canonical 2020-tract panel.** Every tract-year row records
  `harmonization_path` (e.g. `ltdb2010→nhgis_xw2020`), `source_tier`, and an
  **interpolation-uncertainty bound**. NHGIS supplies bounds directly. For LTDB and
  our own H2 crosswalks, derive the bound from the area or housing share that
  crosses boundaries. Two-step paths (nominal → 2010 → 2020) widen the bound.
- **Coverage flags.** Each record also stores `tracted_share`, the share of today's
  tract that was tracted in that year. Earlier decades are *missing*, not zero,
  where untracted, and are never imputed silently.
- **Variable comparability.** Category definitions drift: race (1980 Hispanic
  origin, 2000 multi-race), occupation codes (not harmonized across decades),
  and income (nominal, so deflate). Keep a **core long-run preset**: population,
  housing units, occupancy/vacancy, tenure, age structure, broad race/ethnicity,
  and from 1970 onward median income (deflated) and BA+ share.
- **Weighting by quality.** Similarity and trajectory models down-weight
  tract-years by uncertainty bound and coverage. Earlier decades inform the
  *shape* of a trajectory more than its exact levels.

**What long history adds**
1. **Neighborhood "age" for PECOTA** (research_plan.md R2). HHUUD10's
   **urbanization year** gives every tract a birth year, so trajectories can be
   aligned by **years since urbanization**, just as PECOTA aligns players by age.
   Life-cycle ("aging") curves are then estimated from 80 years of housing growth
   across all 72k tracts, not from three ACS periods.
2. **Long "careers" for comps.** Tracts with H0/H1 history can be matched on
   1970–2020 trajectory shape (core preset). Comps whose decades of history
   resemble the target's are much stronger than single-snapshot matches.
3. **Longer backtests.** Extra out-of-sample folds on the core preset: 1980→1990
   predicting 2000, and 1990→2000 predicting 2010. Era adjustment matters more here
   (1970s urban decline vs 2000s growth), mirroring PECOTA's run-environment
   adjustment.
4. **Trajectory typologies.** Delmelle-style sequences over 1970–2020, decennial.
   This is the classic neighborhood-change literature design, now possible nationally
   for tracted areas.
5. **Legacy context.** The redlining indicator and early composition explain
   present-day gaps. Treat them as a *contextual and equity* facet ("History and
   legacy"), together with urbanization year and the housing growth path.

**Responsible use**
- Do not let historical redlining grades act as a *predictor of decline* in the
  public forecast. That would re-encode the original harm.
- Test whether including them changes forecasts. If they are used, show them as
  explanatory context, and gate them like the R2.6 Decline tiles.

## 6. Supplemental sources that add independent signal for clustering

Prioritize sources that measure what the PDB/ACS **cannot**, rather than more
re-cuts of the ACS. Independent signal is what improves similarity beyond income
and race.
| Signal | Source | Join (requirements §5.2) | Why it helps |
|---|---|---|---|
| Built form and land use | NLCD land cover / impervious surface / tree canopy. Overture/Microsoft building footprints (count, size, density). EPA Smart Location DB. | raster zonal stats / point-in-polygon / direct BG | Separates dense-old-urban from new-dense-suburban areas, which demographics alone conflate |
| Satellite or learned embeddings | Global annual satellite embeddings (AlphaEarth, verified: 64-dim, 10 m, 2017–2024, V-06), Sentinel-2 CNN features | zonal mean → tract | Dense built-environment signal. An optional block for similarity, with low interpretability. |
| Amenities | Overture / OSM POIs: grocery, pharmacy, schools, transit stops, bars, places of worship | point-in-polygon → per-capita and access | "Lived environment". Overture is CDLA, OSM is ODbL. |
| Jobs and functional ties | **LEHD LODES** (workplace jobs by sector; origin-destination flows) | block → tract | Workplace character. **Flow-graph embeddings** (node2vec on commute flows) give *relational* similarity: places that play the same role in their metro. |
| Health outcomes | CDC PLACES, USALEEP | direct | Better held out for validation than used as inputs (§4.4). |
| Economic mobility | Opportunity Atlas | direct (2010 tracts → crosswalk) | Same: validation, or the outcome-weighted preset. |
| Housing market | HMDA, FHFA tract HPI (developmental), Zillow (license), USPS vacancy (restricted, V-04) | direct / crosswalk | Market momentum and investment flows. |
| Hazard and environment | FEMA NRI, EJ indices (EJScreen archived copy, V-05), flood zones | direct / areal | Exposure dimension. |
| Municipal operations | 311, code enforcement, permits (balt311 adapters) | point-in-polygon | **Validation only** for the national typology (within-city procedure in §4.4b). Optional input only for scoped within-city similarity, which is then validated on other outcomes. |

**Coverage rule:** a feature enters the *national* typology only if it covers
≥ 95% of tracts with consistent definitions. Municipal and partial-coverage sources
feed *scoped* similarity, which runs within one city or region.

## 7. Experiment plan (Phase 3 of requirements.md)

> **Sequencing:** see [`research_plan.md`](research_plan.md). R1 is the MVP (facets +
> the E0 k-means baseline). R2 is the tract-level PECOTA (comps-based forecasting,
> with a worked example and backtest). R3 is alternatives + consensus, judged partly
> by whether they improve R2 forecast skill. E1–E6 below run in R1 (E0–E1) and R3
> (E2–E6).

> The wider backlog of candidate methods is in
> [`methodology_alternatives.md`](methodology_alternatives.md). It covers
> dimensionality reduction (PCA/FA/MDS/Isomap/LLE/NMF/…), out-of-domain
> approaches from sports, business, biology and physics, input-perturbation
> experiments, and consensus clustering, each with an `ALT-xx` id.

1. **E0 Baseline.** PDB-only, rates, equal domain weights, per-domain PCA → k-means
   (k = 6…16). Run the full validation battery. This is the reference point.
2. **E1 Variable sets.** Race-included vs race-excluded, with vs without
   built-environment and LODES blocks. Measure the change in external validity (§4.4) and stability.
3. **E2 Representations.** Domain PCA vs global PCA vs factor analysis vs autoencoder
   vs satellite-embedding concat. Metrics: kNN predictive check and MOE neighbor
   stability.
4. **E3 Algorithms.** k-means vs GMM vs HDBSCAN(UMAP-10D) vs Ward. Metrics: stability,
   external validity, size balance, and interpretability review.
5. **E4 Stratification.** National vs RUCA-stratified typologies.
6. **E5 Temporal.** Pooled typology on 2009–13 / 2014–18 / 2019–23 (2020-boundary
   harmonized). Metrics: transition diagonal share, then sequence clusters and
   cost-setting sensitivity.
7. **E6 Outcome-weighted preset.** Metric learning on held-out outcomes, with
   cross-validation by state to avoid leakage.

Tooling: `scikit-learn`, `hdbscan` / sklearn's HDBSCAN, `umap-learn`,
`geosnap` + `giddy` + `libpysal` + `spopt`, `tobler` (harmonization),
optional `torch` for autoencoders, and an R bridge (TraMineR, `fpc`) or Python
ports for sequence analysis and clusterboot. Track every experiment as a
config + metrics row in `experiments/` (Parquet), so results are reproducible
and comparable.

## 8. Open methodological questions

1. Is 2020 Decennial differential-privacy noise material at tract level for
   small-population tracts? Quantify it before using DHC variables as features.
2. National vs metro-stratified default: which matches user intent ("similar
   neighborhoods" usually means within comparable metro types)?
3. Pen-portrait generation. How much can an LLM draft before human review becomes
   the bottleneck, and what review rubric applies?
4. BG-level cohorts: does the §4.3 Monte Carlo show BG assignment is too
   unstable? If so, BG users inherit their tract's cohort, with BG-level similarity only.
5. Trajectory horizon: 3 non-overlapping ACS periods (rich variables) vs 4–6
   decennial points (thin variables). Probably both, as separate presets.

## References (starting set)

- Spielman & Singleton (2015), *Annals AAG* 105(5):1003–1025. Open classification:
  [ICPSR 100235](https://www.openicpsr.org/openicpsr/project/100235/version/V5/view),
  [Liverpool project page](https://liverpool.ac.uk/geographic-data-science/research/understandingthemorphologyofcities/north-american-geodemographics).
- Wyszomierski et al. (*Geographical Journal*; sources disagree on 2023 vs 2024), 2021/22 UK OAC:
  [paper](https://livrepository.liverpool.ac.uk/3177556/),
  [GeoDS dataset](https://data.geods.ac.uk/dataset/uk-oac),
  [ONS page](https://www.ons.gov.uk/methodology/geography/geographicalproducts/areaclassifications/2021and2022residentialbasedareaclassifications);
  [PCA-based variable selection](https://livrepository.liverpool.ac.uk/3046998/).
- Esri, [ArcGIS Tapestry methodology](https://doc.arcgis.com/en/esri-demographics/latest/esri-demographics/tapestry-methodology.htm)
  and [2025 methodology statement](https://content.esri.com/esri_content_doc/dbl/us/arcgis_tapestry/2025arcgistapestrymethodologystatement.pdf).
- Kolak et al. (2020), *JAMA Netw Open*, doi:10.1001/jamanetworkopen.2019.19928;
  [SDOH Atlas](https://sdohatlas.github.io/).
- Census, [Low Response Score](https://www.census.gov/newsroom/blogs/research-matters/2016/05/identifying-hard-to-survey-populations-using-low-response-scores-by-census-tract.html);
  [Comparing ACS data](https://www.census.gov/programs-surveys/acs/guidance/comparing-acs-data.2019.html);
  [non-overlapping 5-year periods](https://www.census.gov/library/stories/2020/12/changes-in-americas-demographic-landscape-in-distinct-5-year-periods.html).
- diversitydatakids, [COI 3.0 technical documentation](https://www.diversitydatakids.org/sites/default/files/file/COI30_TechDoc_20240313.pdf).
- Delmelle (2016), *Annals AAG* 106(1):36–56; Delmelle & Delmelle (2025),
  [REGION 12(2)](https://openjournals.wu-wien.ac.at/ojs/index.php/region/article/view/566),
  [tutorial](https://ericdelmelle.github.io/sequencePaper); sequence-method
  sensitivity ([NSF PAR](https://par.nsf.gov/servlets/purl/10207911)).
- geosnap: [repo](https://github.com/oturns/geosnap),
  [dynamics guide](https://oturns.github.io/geosnap-guide/dynamics.html),
  [SciPy 2024 paper](https://proceedings.scipy.org/articles/FVWM4182.pdf).
- LTDB: [Brown S4](https://s4.ad.brown.edu/Projects/Diversity/Researcher/LTDB.htm);
  [Drexel UHC 2020→2010 crosswalk](https://drexel.edu/uhc/news-events/news/2025/february/census-tract-crosswalk).
- Hennig (2007, 2008), cluster stability:
  [fpc::clusterboot](https://search.r-project.org/CRAN/refmans/fpc/html/clusterboot.html).
- Deep geodemographics: [DeepGeoDemo](https://deepgeodemo.readthedocs.io/);
  [autoencoder vs PCA, E&W OAs](https://zenodo.org/records/19420685) (conference paper, abstract verified);
  [Geodemographics from satellite images](https://aag.secure-platform.com/aag2023/solicitations/39/sessiongallery/5942/application/21363).
