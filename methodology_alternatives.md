# pdb-browser — Alternative methodologies backlog

> **Status:** Living backlog, v0.1 (2026-11-06). Companion to `methodology.md`.
> The k-means baseline (methodology.md §2) stays the reference implementation.
> Everything here is a **candidate experiment**. Each one gets an `ALT-xx` id, is
> run through the same validation battery (methodology.md §4, including the §4.4b
> within-city 311 checks), and is logged in `experiments/`.
>
> Citations are written from background knowledge, not re-verified in this pass. Check
> each before it goes into user-facing methodology text.

## How to read an entry

Each method lists:
- **Origin:** the field it comes from.
- **Idea:** the method in one line.
- **Tract mapping:** what plays the role of "item" and "feature" for us.
- **Buys:** what it adds over k-means on PCA.
- **Watch:** risks and scale at about 85k tracts × about 1,000 variables.
- **Priority:**
  - **T1:** cheap and likely useful; run in Phase 3.
  - **T2:** worth a spike.
  - **T3:** exploratory or long shot.

---

## A. Dimensionality-reduction layer (before any clustering)

The question is whether to cluster in a reduced space, and which one. The baseline
is per-domain PCA (methodology.md §3.3). Alternatives:

| ID | Method | Structure kept | Out-of-sample for new tracts / vintages? | Scale at 85k | Use for |
|---|---|---|---|---|---|
| ALT-01 | **PCA**, per domain or global | Global linear variance | Yes (projection) | Trivial | Baseline clustering input |
| ALT-02 | **Factor analysis** (varimax/promax) | Shared variance; rotated, interpretable factors | Yes | Trivial | Named axes (as in Kolak's 4 indices); explicit noise term per variable suits MOE-heavy data |
| ALT-03 | **Classical MDS** | Same as PCA on Euclidean distance | Landmark-MDS only | n×n infeasible, so use landmark/pivot MDS | Only worthwhile with a **non-Euclidean** distance (Gower for mixed types, Wasserstein for distributions, ALT-21) |
| ALT-04 | **Non-metric MDS** (+ Bray–Curtis, ecology standard) | Rank order of distances | No (re-fit / landmark) | Landmark + subsample | Compositional blocks (race, occupation, 311 mix) |
| ALT-05 | **Isomap** | Geodesic (manifold) distance | Landmark Isomap | Landmark (~5k) | Tests whether tracts lie on a continuum (urban → rural gradient) |
| ALT-06 | **LLE / Hessian LLE** | Local linear patches | Weak | kNN-sparse, OK to about 85k but fragile | Diagnostic only; unstable with noisy inputs |
| ALT-07 | **Laplacian eigenmaps / diffusion maps** | Local connectivity at multiple diffusion scales | Nyström extension | Sparse kNN graph, OK | Diffusion distance is robust to noise; a natural partner for graph clustering (ALT-30) |
| ALT-08 | **NMF** | Parts-based, non-negative additive "ingredients" | Yes (fixed basis) | OK | Shares and counts. Each tract becomes a *recipe* of a few parts. Interpretable. |
| ALT-09 | **ICA** | Statistically independent sources | Yes | OK | Separating overlapping signals (e.g. a "university" signal vs a "poverty" signal in renter share) |
| ALT-10 | **Autoencoder / VAE** | Non-linear, learned | Yes (encoder) | GPU optional | Reconstruction-error comparison vs PCA; a VAE gives uncertainty |
| ALT-11 | **UMAP / t-SNE / PHATE** | Local neighborhoods (PHATE: branches and trajectories) | UMAP yes; t-SNE no | OK | **Visualization only.** Never the final clustering space in 2-D. PHATE comes from single-cell biology and is built to show continua and branching. |

**Selection criteria for the DR layer:**
- **Neighborhood preservation:** trustworthiness and continuity at k = 10 and 50 (do a tract's neighbors survive?).
- **kNN predictive check** (methodology.md §4.4).
- **MOE stability** (§4.3).
- **Out-of-sample transform:** required for new PDB vintages and bring-your-own data.

---

## B. Out-of-domain categorization approaches

### B1. Sports analytics

**ALT-12 Player comparables ("comps") and similarity scores**
> **Promoted:** this is the second step after the k-means baseline. The full
> design, backtest, and worked example are in `research_plan.md` Phase R2. The
> entry below is kept for context.

- **Origin:** Bill James's baseball similarity scores (start at 1000, subtract fixed
  points per unit difference in each stat); Silver's PECOTA projections from
  nearest-neighbor career trajectories.
- **Tract mapping:** a tract is a player, and its domain scores are its stat line.
  Comps are the k nearest tracts. *PECOTA-style:* match on trajectory to date,
  then forecast the next period from how the comps evolved.
- **Buys:**
  - A transparent, hand-weighted distance that anyone can audit.
  - Analog forecasting of neighborhood change, validated on the real next ACS
    period (methodology.md §4.6).
- **Watch:** comps inherit the bias of the weights; needs the temporal panel.
- **Priority:** **T1.** This is essentially our find-similar feature, plus a
  forecast mode.

**ALT-13 Positionless archetypes via topological data analysis (TDA)**
- **Origin:** NBA "13 positions" (Alagappan, MIT Sloan 2012, using Mapper) and
  later GMM/k-means role clustering on per-possession stats.
- **Tract mapping:** traditional "positions" correspond to urban/suburban/rural or
  RUCA codes; "roles" are data-driven archetypes found within each position.
  Per-possession normalization is the analogue of per-capita rates.
- **Buys:** shows that standard categories hide finer roles; motivates
  within-stratum typologies (methodology.md §3.6).
- **Priority:** T2. Implement via ALT-27 (Mapper).

**ALT-14 Pairwise ratings (Elo / Bradley–Terry) for human similarity judgments**
- **Origin:** chess and sports ratings; triplet-comparison psychophysics.
- **Tract mapping:**
  - Local experts answer "Is tract A more like B or like C?"
  - Fit a learned metric (triplet embedding / metric learning) to their answers.
  - Bradley–Terry can also rank tracts on a fuzzy construct ("walkable feel").
- **Buys:** a human-grounded ground truth for validating *similarity*. Feeds
  the face-validity work in methodology.md §4.7 with numbers.
- **Watch:** needs a judging UI and recruited raters. Expert bias.
- **Priority:** T3. A natural fit for an in-app "rate these matches" feedback loop.

**ALT-15 Role detection in passing / tracking networks**
- **Origin:** soccer pass networks; role assignment from player-tracking data.
- **Tract mapping:**
  - LEHD LODES commute flows form the network.
  - A tract's role comes from its network position (job hub, bedroom community,
    pass-through), not its attributes.
  - Methods: node2vec or structural role discovery (RolX, struc2vec).
- **Buys:** **relational** similarity. Two tracts can be demographically
  different but play the same metro role. This signal is independent of the census.
- **Priority:** T2.

### B2. Business, finance, and marketing

**ALT-16 Text-based dynamic peer sets (no fixed categories)**
- **Origin:** Hoberg & Phillips text-based network industries (cosine similarity of
  10-K product descriptions). Each firm gets its own peer set, which changes
  over time and need not be transitive. It replaced static SIC/NAICS codes.
- **Tract mapping:** drop global clusters for some uses. Each tract's "cohort" is
  everything within a similarity threshold, recomputed per vintage. A tract can be
  a peer of two groups that are not peers of each other.
- **Buys:**
  - Matches how users actually ask ("places like mine").
  - Avoids forcing tracts on a continuum into boxes.
  - Pairs with the static typology for labels.
- **Priority:** **T1.** Cheap: it is thresholded kNN. The design question is mainly
  in the UI.

**ALT-17 Style box / faceted grid typology**
- **Origin:** Morningstar 3×3 fund style box (size × value/growth); also
  library science faceted classification (Ranganathan).
- **Tract mapping:**
  - Pick 2–3 interpretable axes (e.g. the domain factors economic position ×
    density × household stage).
  - Cut each axis into terciles.
  - A cohort is a tuple: "High-income · Dense · Young-household".
- **Buys:**
  - Maximum interpretability.
  - Natural facet filters in the browser.
  - No clustering instability.
  - An honest choice if taxometrics (ALT-26) says the structure is dimensional.
- **Watch:** cell sizes are uneven; arbitrary cut points.
- **Priority:** **T1.** Also a strong baseline that k-means must beat.

**ALT-18 Factor-exposure profiles (Fama–French style)**
- **Origin:** asset pricing. Describe each asset by its loadings on a few common
  factors, plus an idiosyncratic residual.
- **Tract mapping:** regress each tract's outcome profile (or its standardized
  features) on national factors. The residual describes what is distinctive about
  this tract beyond its factors.
- **Buys:** the "distinctiveness" panel in the profile view; flags tracts poorly
  explained by any cohort.
- **Priority:** T2.

**ALT-19 Latent class / latent profile analysis (market research segmentation)**
- **Origin:** survey segmentation; psychometrics.
- **Tract mapping:**
  - Latent profile analysis on continuous domain scores is essentially a Gaussian
    mixture with model selection by BIC.
  - LCA on discretized indicators (tercile flags, designations such as Qualified
    Census Tract or food desert).
- **Buys:**
  - Principled k selection (BIC/entropy).
  - Posterior class probabilities (the methodology.md §4.3 confidence for free).
  - Handles mixed variable types.
- **Priority:** **T1** (as GMM, already a comparator). LCA is T2.

**ALT-20 Collaborative filtering / matrix factorization**
- **Origin:** recommender systems ("customers who bought…").
- **Tract mapping:** a tract × category matrix (311 service types, POI categories,
  occupation × industry cells) treated as implicit feedback. Factorize it (ALS,
  Poisson MF) to get latent tract vectors; item–item similarity gives "tracts like
  this".
- **Buys:**
  - Handles sparse and missing cells natively. That is the patchy municipal-data
    problem.
  - Can fill in categories a city doesn't publish.
- **Watch:** within city only (taxonomy differences; methodology.md §4.4b).
- **Priority:** T2.

### B3. Biology and ecology

**ALT-21 Single-cell RNA-seq pipeline (the closest structural analogue)**
- **Origin:** single-cell genomics. Millions of cells × about 20k genes:
  1. Select highly variable genes.
  2. PCA to about 50 dims.
  3. Build a kNN graph.
  4. Leiden/Louvain community detection.
  5. UMAP for display.
  6. Find marker genes per cluster.
  7. Annotate by hand.
- **Tract mapping:** cells are tracts; genes are variables.
- **Buys:** a mature, scalable toolchain (`scanpy`). Mapped piece by piece:
  - **Graph clustering** (Leiden) finds clusters of uneven size and shape that
    k-means misses.
  - **Marker-gene tests** are automatic pen-portrait features.
  - **Batch correction** (Harmony, ComBat) removes *state/region batch effects*
    such as cost-of-living-inflated incomes and rents. "Similar" then means similar
    *relative to regional context*, which could be a user toggle.
- **Priority:** **T1.** Run as the main non-k-means comparator.

**ALT-22 Trajectory inference / pseudotime**
- **Origin:** single-cell developmental biology (Monocle, PAGA, diffusion
  pseudotime, RNA velocity). Infer a developmental ordering from one
  cross-sectional snapshot.
- **Tract mapping:**
  - Infer "neighborhood-change pseudotime" (e.g. a gentrification continuum) from
    the cross-section alone.
  - **Validate against the real ACS time series** (methodology.md §5): does
    pseudotime order predict observed change direction?
  - **PAGA** gives a coarse graph of which cohorts connect to which.
- **Buys:**
  - Trajectory-like insight for variables with no history.
  - A testable hypothesis generator.
- **Priority:** T2.

**ALT-23 Community-ecology ordination and indicator species**
- **Origin:** vegetation science.
- **Tract mapping:** composition blocks (occupation mix, 311 type mix, POI mix,
  household types) are species assemblages; census context variables are the
  environment.
  - **Canonical correspondence analysis (CCA) / redundancy analysis (RDA):**
    how much of the composition is explained by the environment.
  - **Bray–Curtis + NMDS:** composition distance.
  - **Indicator values (IndVal, Dufrêne & Legendre):** picks the variables that are
    both specific to and frequent in a cohort, giving rigorous pen-portrait labels.
- **Buys:** decades of method for compositional, zero-heavy data. IndVal is
  directly useful for naming cohorts.
- **Priority:** **T1** for IndVal (cheap, labeling). CCA is T2.

**ALT-24 Admixture / mixed membership**
- **Origin:** population genetics (STRUCTURE, ADMIXTURE); the same mathematics as
  topic models (LDA).
- **Tract mapping:**
  - Each tract is a mixture of K "ancestral" neighborhood types (e.g. 60%
    "inner-ring renter" + 40% "student").
  - LDA works directly on **count tables** (people per category), so tracts with
    few counts are naturally treated as less certain. A partial fix for MOE.
  - Effective counts can be approximated from MOEs (n_eff ≈ p(1−p)/SE²).
- **Buys:**
  - Honest representation of transitional and mixed tracts.
  - Soft membership by design.
- **Priority:** **T1** (LDA/NMF on count blocks).

**ALT-25 Archetypes and Pareto task inference**
- **Origin:** archetypal analysis (Cutler & Breiman); Pareto-front task inference
  in evolutionary biology (Shoval et al. 2012). Data that lie in a polytope have
  vertices that are "specialists" at competing tasks.
- **Tract mapping:**
  - Find 3–6 extreme archetype tracts (e.g. the job-center CBD, the family
    exurb, the student district, the retirement community).
  - Every tract becomes a convex combination of them.
  - Test whether the data fill a low-dimensional polytope (a trade-off structure).
- **Buys:** archetypes are *real, nameable places*, which is great for
  communication ("this tract is 70% like Tract X in Arlington"). Complements centroids, which are averages and not real places.
- **Priority:** T2.

**ALT-26 Taxometric analysis (are there types at all?)**
- **Origin:** clinical psychology (Meehl; MAXCOV/MAMBAC). A test of whether a latent
  construct is categorical (taxon) or dimensional. It grew out of the MBTI-vs-Big-Five
  debate: forcing types onto continua misleads.
- **Tract mapping:** run on the domain factors. If the structure is dimensional,
  communicate positions on axes (ALT-17 style box), with clusters as convenience
  labels only.
- **Buys:** an honest answer to "are cohorts real or just cut points?". Should
  shape how the UI presents cohorts.
- **Priority:** **T1.** Cheap, and it determines the framing.

**ALT-27 Phylogenetics-style trees with bootstrap support**
- **Origin:** systematics. Neighbor-joining or UPGMA trees with a Felsenstein
  bootstrap support value on every branch.
- **Tract mapping:**
  - Hierarchical clustering of cohort centroids (or a subsample of tracts).
  - Bootstrap over **variables** (resample features) and over tracts.
  - Report support per merge.
- **Buys:**
  - The nested supergroup → group hierarchy with a confidence on each node.
  - Directly addresses "shuffling input variables".
- **Priority:** **T1.** It is how we present the stability results from methodology.md §4.2.

### B4. Physics, mathematics, and the physical sciences

**ALT-28 Spatially coupled clustering (Potts / hidden Markov random field)**
- **Origin:** statistical mechanics (Ising/Potts spin models); image segmentation.
- **Tract mapping:**
  - Cohort labels are "spins"; the adjacency graph supplies the coupling.
  - The energy combines feature fit with a penalty for disagreeing with neighbors.
  - The coupling strength is a tunable "temperature".
- **Buys:**
  - Denoises the salt-and-pepper MOE noise in dense cores (methodology.md §4.5).
  - At high coupling, becomes regionalization.
- **Watch:** smooths away real sharp boundaries (e.g. a highway dividing
  neighborhoods). Validate against known boundaries.
- **Priority:** T2.

**ALT-29 Topological data analysis: Mapper and persistent homology**
- **Origin:** applied topology. Mapper found a breast-cancer subtype (Nicolau
  et al. 2011) and the NBA positions (ALT-13).
- **Tract mapping:**
  - Mapper with a lens (e.g. the first two factors, or density) gives a graph
    summary showing **branches, flares, and loops**.
  - Persistent homology tests whether those features are robust.
- **Buys:**
  - Finds continua and side-branches (e.g. a distinct "college town" flare off the
    main urban–rural spine) that partitions hide.
  - Good for exploration and for the embedding view.
- **Priority:** T2 (`kmapper`, `giotto-tda`).

**ALT-30 Graph community detection (modularity, stochastic block models)**
- **Origin:** network physics (Newman–Girvan modularity, Leiden, stochastic block
  models with Bayesian inference, e.g. Peixoto's `graph-tool`).
- **Tract mapping:** kNN similarity graph → communities. A nested SBM gives a
  hierarchy *and* picks the number of groups by minimum description length.
- **Buys:** principled k (MDL); hierarchy for free; handles non-convex clusters.
- **Priority:** **T1** via Leiden (shared with ALT-21). Nested SBM is T2.

**ALT-31 Optimal transport (Wasserstein) distances on full distributions**
- **Origin:** physics and mathematics (earth mover's distance); widely used in ML.
- **Tract mapping:**
  - Compare tracts on full **distributions**, not summary scalars: income-bracket
    histograms (B19001), age pyramids, rent brackets, commute-time bins.
  - The ordered brackets make 1-D Wasserstein cheap and exact.
- **Buys:**
  - Two tracts with the same median income but different spreads (bimodal
    gentrifying vs uniform) become distinguishable.
  - Feeds ALT-03 MDS or graph methods.
- **Priority:** **T1** for 1-D Wasserstein on bracket tables. Cheap and a clear
  information gain.

**ALT-32 Regime detection and phase diagrams / tipping points**
- **Origin:** phase transitions; climate "weather regimes" (k-means on EOFs, the
  same recipe as ours); economics of neighborhood tipping (Card, Mas & Rothstein 2008).
- **Tract mapping:** look for thresholds where transition probabilities between
  cohorts (methodology.md §5.3 Markov) jump, e.g. a vacancy or composition level
  beyond which decline accelerates.
- **Buys:** turns the trajectory layer into an early-warning tool.
- **Priority:** T3 (needs the temporal panel).

**ALT-33 Analog forecasting**
- **Origin:** meteorology (Lorenz's analogues): forecast by finding past states
  that resemble the present.
- **Tract mapping:** the same as ALT-12 PECOTA comps, from a different field. Merge
  the two when implementing.
- **Priority:** see ALT-12.

**ALT-34 Template fitting and morphological classification**
- **Origin:** astronomy. Stellar spectral classes by template matching; galaxy
  morphology from crowdsourcing (Galaxy Zoo) plus ML; the Hertzsprung–Russell
  diagram, where two axes reveal a "main sequence".
- **Tract mapping:**
  - **Templates:** define hand-built archetype profiles (expert-specified) and
    assign each tract by best fit. A supervised counterpart to clustering.
  - **HR-diagram analogue:** look for a "main sequence" in a 2-axis plot such as
    income × density; tracts off it are the interesting ones.
  - **Galaxy-Zoo-style crowdsourcing:** collect labels to train a classifier
    (pairs with ALT-14).
- **Priority:** T3.

**ALT-35 Information bottleneck clustering**
- **Origin:** information theory (Tishby et al.).
- **Tract mapping:** compress tracts into K clusters that keep the most
  information about a chosen *relevance* variable (held-out outcomes).
  This is the principled version of methodology.md §3.4's outcome-weighted preset.
- **Watch:** leakage; the relevance outcomes can't then be used for validation.
- **Priority:** T3.

### B5. Machine learning and remote sensing (self-supervised)

**ALT-36 Contrastive learning with MOE draws as augmentations**
- **Origin:** computer vision (SimCLR); remote sensing (Tile2Vec: nearby tiles
  should embed nearby).
- **Tract mapping:** positive pairs include:
  - Two **MOE-perturbed draws** of the same tract, which teaches invariance to
    sampling noise.
  - Optionally, adjacent tracts (Tile2Vec), which adds spatial smoothness.
  - Optionally, the same tract in consecutive non-overlapping periods.
- **Buys:** an embedding that is *by construction* robust to ACS uncertainty.
  The most direct ML answer to methodology.md §4.3.
- **Watch:** needs a GPU for comfort, but it is a small MLP on 85k rows, so CPU
  is feasible. Less interpretable.
- **Priority:** T2.

**ALT-37 Multi-view integration: similarity network fusion and MOFA**
- **Origin:** multi-omics medicine. Similarity network fusion (Wang et al. 2014)
  fuses per-data-type patient similarity networks. MOFA (multi-omics factor
  analysis) learns factors shared across, or specific to, data types.
- **Tract mapping:** the data blocks are census, built environment, health,
  environment/hazard, jobs/flows, and municipal.
  - SNF builds a kNN graph per block, fuses them, then clusters the fused graph.
  - MOFA reports which factors are shared across blocks and which are block-specific.
- **Buys:**
  - Integrates heterogeneous sources without hand-weighting.
  - Shows which sources add *new* structure, which answers the methodology.md §6
    question with numbers.
- **Watch:** SNF is n×n, so use sparse kNN graphs or a per-region fit.
- **Priority:** T2.

---

## C. Input-perturbation ("shuffling") experiments

Run on every candidate method, not just k-means:

| ID | Perturbation | Answers |
|---|---|---|
| ALT-P1 | **Domain ablation:** drop one domain at a time | Which domains drive the structure; is the result dominated by one? |
| ALT-P2 | **Random subspace ensembles:** many runs on random 50–70% feature subsets | Feature-robust cohorts; input to consensus (ALT-E1) |
| ALT-P3 | **Variable bootstrap** (phylogenetic-style; ALT-27) | Support values per cohort and merge |
| ALT-P4 | **Substitute-variable swaps:** replace each variable with its nearest correlated proxy | Sensitivity to curation choices |
| ALT-P5 | **MOE Monte Carlo** (methodology.md §4.3) | Sensitivity to sampling noise |
| ALT-P6 | **Permutation null:** shuffle each column independently (destroys covariance, keeps marginals) | Is there structure beyond the marginals? Silhouette and stability must clearly beat this. |
| ALT-P7 | **Knockoff features** (model-X knockoffs) | Which variables carry real signal for supervised presets (ALT-35, §3.4) |
| ALT-P8 | **Vintage swap:** fit on one PDB vintage, apply to another | Temporal transferability |
| ALT-P9 | **Region holdout:** fit without one Census division | Geographic transferability (methodology.md §4.2 split-half) |

## D. Combining the menagerie

**ALT-E1 Consensus / evidence-accumulation clustering**
- **Origin:** Monti et al. 2003 (cancer subtyping); Fred & Jain 2005.
- **Method:**
  1. For each pair of tracts, record the share of runs (across methods ×
     perturbations) that put them together. This is a co-association matrix.
  2. Cluster that matrix.
  3. The consensus CDF / PAC score picks k.
- **Buys:**
  - One robust typology from many methods.
  - Per-tract "agreement" doubles as a confidence measure.
- **Watch:** the matrix is 85k² dense, which is too big. Run on a stratified
  subsample of about 10k and assign the rest via kNN, or store only kNN-sparse
  co-associations.
- **Priority:** **T1** once ≥3 methods are running.

---

## E. Suggested order

1. **Run first (Phase 3, cheap, decision-shaping):**
   - ALT-26 taxometrics: types or dimensions?
   - ALT-17 style box as a baseline.
   - ALT-01/02 PCA vs FA.
   - ALT-21/30 Leiden on a kNN graph.
   - ALT-19 GMM.
   - ALT-24 LDA/NMF on count blocks.
   - ALT-31 Wasserstein on bracket tables.
   - ALT-23 IndVal labeling.
   - ALT-27 variable bootstrap.
   - ALT-P1/P2/P6.
   - ALT-E1 consensus.
   - ALT-12/16: comps and dynamic peers, which are mostly UI work on top of kNN.
2. **Spikes:** ALT-15 LODES roles, ALT-22 pseudotime vs real time series, ALT-25
   archetypes, ALT-28 Potts smoothing, ALT-29 Mapper, ALT-36 contrastive, ALT-37
   SNF/MOFA, ALT-07 diffusion maps, ALT-20 matrix factorization (within city).
3. **Exploratory:** ALT-14 human triplets, ALT-32 tipping points, ALT-34 templates,
   ALT-35 information bottleneck.

Every run is logged as `experiments/{ALT-id}_{date}.parquet`, holding the config plus
the methodology.md §4 metrics, so methods are compared on identical gates rather than
on how good their maps look.

## F. Tooling map

| Need | Python |
|---|---|
| PCA/FA/ICA/NMF/GMM/LDA, manifold learners, kNN | `scikit-learn`, `factor_analyzer` |
| UMAP / PHATE / diffusion maps | `umap-learn`, `phate`, `pydiffmap` |
| Single-cell-style pipeline, Leiden, marker tests, Harmony, PAGA | `scanpy`, `leidenalg`, `harmonypy` |
| Stochastic block models | `graph-tool` (conda) |
| TDA | `kmapper`, `giotto-tda`, `ripser` |
| Optimal transport | `POT`, `scipy.stats.wasserstein_distance` |
| Archetypes | `archetypes` / custom NNLS |
| Taxometrics | R `RTaxometrics` via `rpy2` |
| Indicator species | R `indicspecies` via `rpy2`, or a short custom implementation |
| Node embeddings / roles | `node2vec`, `karateclub` |
| Contrastive / autoencoders | `torch` (small MLPs, CPU OK) |
| Multi-view | `snfpy`, `mofapy2` |
| Consensus clustering | custom (sparse co-association) |
| Potts / HMRF | custom, or `pygco` graph cuts |
