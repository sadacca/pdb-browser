# pdb-browser — Critical review of the plan

> **Status:** Review v0.1 (2026-11-06). Reviews `requirements.md`, `methodology.md`,
> `methodology_alternatives.md`, `research_plan.md` and `verification_log.md`.
> It focuses on the stated mission: a **common resource for analyzing public data,
> hosted on free cloud services**. Severity key: **🔴 blocking** (fix before
> Phase 0/1), **🟠 important** (fix before a public launch), **🟡 worth tracking**.
> New facts are tagged `RV-xx` so they can be added to `verification_log.md`.

## 0. Summary verdict

The **architecture is the right call**. A static site using MapLibre, PMTiles and
Parquet fetched with HTTP range requests is the only free-tier design that scales
nationally without a server, cold starts or a RAM cap. The plan's reasons for
rejecting Streamlit are well grounded in the reference repos. The data and
methodology work is unusually careful about ACS margins of error, overlapping
vintages and boundary changes.

The plan's weaknesses are in four other areas:

1. **"Free" is assumed, not engineered.** The plan does not handle:
   - R2's r2.dev rate limit;
   - the fact that R2 has no spending cap;
   - Cloudflare's 512 MB cache limit;
   - Hugging Face's per-IP rate limits;
   - the GitHub Actions disk and cache ceilings.

   None of these is fatal, but each needs a design response (§1).
2. **"Common resource" is mostly undefined.** The plan has no governance, no
   account ownership plan, no contribution path for source specs, no citation or
   DOI, no stable data contract, no hosting fallback and no sustainability plan
   (§2).
3. **The research layer contradicts the "public data" premise in places.** Some
   flagship inputs cannot be redistributed:
   - NHGIS and LTDB terms;
   - Geolytics NCDB;
   - USPS vacancy;
   - Zillow.

   So outputs built from them cannot be reproduced by the public (§3).
4. **Scope and sequencing risk.** The research plan's R1 exit criteria (including a
   Kalman-smoothed annual panel and a harmonized 2010→2020 history) are attached to
   the *browse MVP*. That conflicts with the requirements doc, which defers
   2010-geography data (§4).

---

## 1. Free-tier hosting: does the plan actually stay free and up?

### 1.1 Object storage (R2 vs Hugging Face): the main open risk

| Issue | Severity | Detail | Recommendation |
|---|---|---|---|
| **r2.dev is not a production endpoint** (RV-01) | 🔴 | Cloudflare documents public `r2.dev` access as rate limited and for development only. Over the limit it returns 429s and may throttle bandwidth. Caching, WAF rules and bot management need a **custom domain** on the bucket. | Budget for a domain (about $10/yr, the one unavoidable cost), add it to Cloudflare, and serve the bucket from `data.<domain>` with cache rules. State this in §8. |
| **No hard spending cap on R2** | 🔴 | V-11 notes billing beyond the free tier has no hard cap. R2 also needs a payment method on file, so "free" really means "free until a traffic spike". Every PMTiles tile or Parquet range read that **misses the cache** is a Class B operation. | (a) Serve through the custom domain with "cache everything" plus long TTLs on **immutable, versioned paths**, so most reads never reach R2. (b) Run a scheduled Action that reads R2 operation counts from the Cloudflare analytics API and alerts at 50% and 80% of the free tier. (c) Write down a kill switch: point `manifest.json` at the HF mirror. |
| **Cloudflare free cache limit is 512 MB per object** (RV-02) | 🟠 | `bg.pmtiles` is budgeted at ≤ 400 MB, which is close. A national BG archive at z8–14 could plausibly exceed it. Objects over the limit are never cached, so every BG tile read becomes a billed R2 operation. | Make "every published object < 512 MB" a CI assertion. Shard BG tiles by state or region (`bg/{st}.pmtiles`), which also makes partial rebuilds cheaper. |
| **Hugging Face per-IP rate limits** (RV-03) | 🟡 | The HF Hub documents per-IP limits for anonymous users in 5-minute windows, about 3,000 `/resolve/` requests. Limits are per visitor, so ordinary use is fine. Map panning plus DuckDB range reads are bursty, though, and HF says the numbers can change. Hosting web-map tiles is also not HF Datasets' core use case. | Use HF as the **zero-billing mirror and fallback**, not the primary tile host. Test PMTiles with HF's `/resolve` redirects and CORS in the Phase 0 spike. |
| **No alternative considered: Source Cooperative** (RV-04) | 🟠 | Radiant Earth's Source Cooperative is a hosting service built for exactly this: public, cloud-native geospatial data (PMTiles, GeoParquet) on S3, with range requests and stable URLs. Protomaps and others already host there. Its current free-tier terms need checking; the 2023 terms were sponsored by AWS and Microsoft. | Add it to open question 3 as a candidate primary or mirror host. If accepted, it removes R2's billing risk and adds visibility in the open-data community. Also consider the **AWS Open Data Sponsorship Program** once the dataset is established. |
| **No archival host** | 🟠 | R2 and HF are serving layers, not archives. A common resource needs citable, frozen releases. | Upload each versioned release (the Parquet files plus `manifest.json`) to **Zenodo** to get a DOI. Zenodo's per-record quota is tens of GB. Don't serve from it; cite it. |

### 1.2 Front end and in-browser compute

| Issue | Severity | Detail | Recommendation |
|---|---|---|---|
| **The "app bundle < 2 MB" target conflicts with DuckDB-WASM** | 🔴 | The DuckDB-WASM binary alone is several MB compressed. Adding MapLibre and the Plot/Vega libraries makes the < 2 MB target and the Phase 0 "< 3 s load" exit criterion unrealistic on a cold mobile connection. | Split the budget into a **first-paint budget** (MapLibre, PMTiles and the app shell, ≤ 1.5 MB, with the choropleth drawn from tile attributes) and **lazily loaded DuckDB** for tables, profiles and kNN. For simple reads, consider `hyparquet` or an Arrow IPC file instead of DuckDB. Measure on throttled 4G in Playwright. |
| **Profile on click may read MBs** | 🟠 | R-GEO-2 says full attributes are fetched on demand, from Parquet with row groups by state. One tract's ~530 columns means reading every column chunk of that state's row group: hundreds of range requests or several MB per click, each a Class B operation on a cache miss. | Publish a **profile-optimized artifact** in addition: per-county JSON or Parquet shards, sorted by GEOID with small row groups and page indexes. Keep the wide state-grouped Parquet for analysts. Add a test that counts bytes and requests per click. |
| **The facet kNN payload is large for phones** | 🟡 | 85k × ~60 floats as float32 is about 20 MB, and BG (242k) is about 58 MB. iOS Safari WASM and tab memory limits are tight. | Quantize the embeddings to int8 or float16 (2–4× smaller; check recall@k against float32). Load them only when the user opens "Find similar". Keep the plan's fallback of shipping precomputed top-50 neighbors for BG. |
| **Basemap choice has hidden size and terms issues** | 🟡 | A self-hosted Protomaps US basemap at useful zooms is tens of GB, beyond R2's 10 GB free storage. OpenFreeMap is keyless but has fair-use terms and no SLA. | Default to OpenFreeMap, keep a low-zoom Protomaps extract (≤ z10, a few GB at most) as a fallback, and make the style URL configurable. |
| **Census Geocoder from the browser** | 🟡 | It isn't verified whether the geocoder supports CORS, what its rate limits are, or how reliable it is. Sending a user's address to a third party also needs a privacy note. | Phase 0 check: call it from the browser. Fallbacks are a build-time place and county name index (local, private) and Nominatim or Photon under their usage policies. State in the privacy notice that address lookups go to the Census Bureau. |
| **Observable Framework maintenance** (RV-05) | 🟡 | The repo was still being updated in 2026, but a 2026 issue reports dependency security alerts going unaddressed, and Observable's own effort appears to be moving to `notebook-kit`. | Lean toward **plain Vite + TypeScript** for the product, with Framework at most for the explainer and backtest pages. Decide this in Phase 0 (open question 2). |

### 1.3 Build pipeline on free CI

| Issue | Severity | Detail | Recommendation |
|---|---|---|---|
| **The raw cache doesn't persist on runners** | 🔴 | R-PIPE-1 assumes `data/raw/` is reused across runs. GitHub-hosted runners are ephemeral, and the Actions cache is capped at 10 GB per repo with eviction. Raw inputs (PDB CSVs about 0.7 GB, TIGER/CB, 2020 blocks, LODES, NHGIS extracts) will exceed that. | Store raw inputs in a `raw/` prefix in object storage, keyed by URL and sha256. This also protects against upstream removals; EJScreen is the precedent. Note that it adds storage and operations to the R2 budget. Never publish restricted raw files (§3). |
| **Runner disk and job time** | 🟠 | Public-repo runners give about 14 GB of free disk and a 6-hour job limit. Steps that need care: national tippecanoe for BGs, dasymetric `tobler` with about 8M 2020 blocks, the R2 PECOTA backtests with several folds, MOE Monte Carlo and R3 consensus runs. | Split stages by state in matrix jobs (as the plan already does for sources), write intermediate results to object storage between jobs, and keep R3 experiments off CI. Run them locally, on Colab or on a HF Space, and only *log* their outputs to `experiments/`. |
| **Census and agency hosts may block CI** | 🟠 | The verification log already records census.gov downloads blocked in one environment. Federal hosts sometimes rate limit or block cloud IP ranges. | Retry with backoff, pin sha256 checksums, use the raw-input mirror above, and fall back to the Census API for the PDB. |
| **Scheduled workflows go stale** | 🟡 | GitHub disables scheduled workflows in public repos after 60 days without repository activity. A "refresh: annual" source would silently stop. | Use a `workflow_dispatch` run per release, plus a monthly freshness check that opens an issue when a source's upstream `Last-Modified` changes. |

### 1.4 The "analyst workbench" (Phase 6) is the wrong free-tier shape

- 🟠 **Contradiction with R-JOIN-4.** The workbench exists for "BYO-data joins
  too large for the browser". Server-side uploads break the "nothing is uploaded"
  promise and add data handling and abuse exposure. Streamlit Community Cloud's
  RAM cap rules out national BG work anyway.
- **Recommendation:** replace Phase 6 with a **`pip install`-able `pdbb` client
  plus notebooks** (Colab, Binder, local) that read the same published Parquet over
  HTTP with DuckDB. Users then bring their own compute and their own data, nothing
  needs hosting, and it serves the "developers" audience directly. Keep a HF Space
  only as an optional demo.
- Consider also publishing the tract tables to a **BigQuery public dataset** or
  documenting `bq` access. Analysts can then join them to existing
  `bigquery-public-data` ACS tables with SQL under the free sandbox quota.

---

## 2. Gaps in the "common resource" part of the mission

The plan specifies a product thoroughly but a *shared resource* hardly at all.

| Gap | Severity | Recommendation |
|---|---|---|
| **Ownership and bus factor.** Cloudflare, HF and the domain are implicitly personal accounts. | 🔴 | Put the repo in a GitHub **organization**, use an HF **organization**, hold the Cloudflare account under a shared role-based email, and give at least two people admin access. Document a "rebuild from zero" runbook. |
| **Data contract and versioning.** `manifest.json` exists, but nothing defines schema stability, semantic versions, deprecation or URL permanence. | 🔴 | Use immutable paths `/{release}/{artifact}` plus a `latest` pointer, give `manifest.json` a JSON schema, and adopt semver for the data release: column removal or renaming = major. Also keep a published CHANGELOG. In-flight PMTiles readers break if an object changes under the same URL (ETag mismatch), so never overwrite in place. |
| **Contribution path.** Adding a source is designed as "write a source spec", but there is no defined way for *others* to contribute one. | 🟠 | Add a JSON schema for source specs, CI that validates a spec and runs its join on a fixture, a PR template that requires the license and the join report, CODEOWNERS for review, and a "source request" issue template. This is what makes it a common resource rather than one person's tool. |
| **Citation and attribution.** | 🟠 | Add `CITATION.cff`, Zenodo DOIs per release, and per-source attribution strings generated from the spec (`license`, `attribution`) into the app's footer and exports. ODbL and CDLA (OSM, Overture) and NHGIS have specific citation requirements. |
| **Sustainability and cost governance.** | 🟠 | Name the one real cost (the domain) and the worst-case R2 bill at 10× expected traffic. Set a policy: if free limits are hit, degrade by switching to the HF mirror and disabling BG tiles, instead of paying. Optionally use a fiscal sponsor or a "sponsor this project" page. |
| **Usage measurement and feedback.** There are no analytics, and no way to know whether the resource is used or what fails. | 🟡 | Use Cloudflare Web Analytics (free, cookieless), an in-app "report a data issue" link that opens a prefilled GitHub issue, and a public status note in the README. |
| **Abuse and hotlinking.** Other apps can hotlink the bucket and spend your Class B operations. | 🟡 | Hotlinking is fine (it's a common resource), but keep it cache-friendly. Cloudflare's free plan allows one rate-limiting rule; use it on uncached paths. Document "please mirror bulk downloads". |
| **User research.** The audiences are listed, but there is no validation step. | 🟡 | Before Phase 3, run 5–8 interviews or usability sessions with the named audiences (municipal staff, advocates). Find out whether they want similarity and forecasts at all, or mainly a clean, documented covariate table. |

---

## 3. Assumptions about "public data" that don't hold

| Assumption | Severity | Problem | Recommendation |
|---|---|---|---|
| **NHGIS outputs can be published** (RV-06) | 🔴 | The NHGIS terms grant a limited license and do **not** permit redistribution without permission. H0 of the long-history layer (NHGIS standardized 1990–2020) and the 1940–60 nominal tables are planned as published panel layers. LTDB requires registration with its own terms. | Add a `redistribution` field to the source spec: `public`, `derived-only`, `internal-only` or `display-only`. Extend R-JOIN-5 into a full gate. Ask NHGIS for permission early, or publish only derived model outputs and point users to NHGIS for the inputs. Where possible, prefer the Census relationship files, which are public domain, for 2010→2020 harmonization. |
| **Restricted inputs in a reproducible public model** | 🟠 | The plan makes Geolytics NCDB the "preferred H1 source", lists USPS vacancy as a model input, and lists Social Explorer. Outputs that depend on them (R2 forecasts) cannot be reproduced by the public, which undercuts the resource. | Define a **public-reproducible build** that excludes restricted inputs and is the default for everything published. A restricted build can be used for internal validation only, and its results are labeled as such. Report how much skill the restricted inputs actually add. |
| **Every federal source stays online** | 🟠 | EJScreen's removal (V-05) shows that federal sources can disappear. The plan reacts to it case by case. | Treat this as a general requirement: mirror every public-domain raw input (§1.3), record its retrieval date and checksum, and label vanished sources "historical". This is arguably one of the resource's most valuable functions. |
| **Private sources: display-only is enforceable on a static site** | 🟡 | Anything sent to a browser can be downloaded. A "display-only" layer in PMTiles is effectively redistributed. | Exclude display-only sources from the static site entirely, unless the license explicitly allows serving tiles. |
| **Overture and OSM derived counts** | 🟡 | ODbL share-alike applies to derived *databases*. Per-tract POI counts may need to be released under ODbL. | Keep OSM-derived variables in a separate artifact with an ODbL label, so the core product stays public domain or CC-BY. |

---

## 4. Technical plan: consistency, sequencing and scope

| Finding | Severity | Detail | Recommendation |
|---|---|---|---|
| **2010-geography deferral conflicts with R1 and R2** | 🔴 | requirements.md §3.1 puts 2010-geography PDBs under "Won't (v1)". But research_plan R1.3 needs 11 ACS vintages harmonized from 2010 to 2020 tracts. R2's fit era (2009–13 → 2014–18) and the decennial 2000→2010 folds are all on 2010 or older geography. | Decide explicitly. Either move the 2010→2020 crosswalk into Phase 2 (join framework), where the `crosswalk` join type already lives, or drop the R2 fit era to 2016–20 onward. The latter isn't viable, because there are no non-overlapping 2020-geography transitions until 2024–28. **So the crosswalk is a Phase 2 dependency.** |
| **Research gates are attached to the product MVP** | 🔴 | research_plan says R1 maps to requirements Phases 1 and 3. Its exit criteria include the annualized Kalman panel and the county 1-year holdout, which are research, not browsing. | Decouple them. The **product Phase 1 ships browse, dictionary, profile and export** with no research dependency. R1 can then proceed in parallel on the same data product. This gets a public, useful resource out early. |
| **The scope is very large for a free, small-team resource** | 🟠 | 5 join types, 15+ federal sources, municipal adapters, a long-history layer back to 1940, PECOTA forecasting, a 40-method alternatives backlog and a consensus atlas. There is no estimate of effort or of who does the work. | Define a **"v1 common resource"** cut: national tract browse + dictionary + 6 federal joins + find-similar + documented Parquet. Move PECOTA and R3 to a separate "research track" milestone that publishes to the same data product but doesn't block it. |
| **Forecasts as a public resource** | 🟠 | R2.6 covers harm well. But publishing per-tract Decline and Thinning probabilities on a free, open, hotlinkable data host means anyone can bulk-download them, whatever the UI gates. | Decide whether the forecast outputs are **published data** or **explainer-only**. If published, the use restrictions need a license (for example CC-BY with a use statement; note that real use restrictions make the data not open). A use statement shown only in the UI doesn't travel with a Parquet file. |
| **MOE availability is still unknown (V-14)** | 🟠 | The reliability badges, R-SIM-6 and the §4.3 validation all depend on MOEs. If the PDB lacks them, the fallback (the ACS detailed-table API) changes the pipeline's volume and its API-key and rate-limit needs. | Keep this as the first Phase 0 check and size the fallback: number of variables × 85k tracts × API calls, against the Census API's limits. |
| **Percentile scopes multiply artifacts** | 🟡 | R-PROF-1 requires national, state and metro percentiles for ~500+ variables. That's 3× the columns, or computing them in the browser. | Precompute national percentiles. Compute state and metro percentiles in the browser from the column (85k values sort in milliseconds). |
| **Block-level DP noise** | 🟡 | Dasymetric weights from 2020 blocks carry differential-privacy noise (methodology §8 Q1). For interpolation weights this is usually acceptable, but small-area joins can inherit odd weights. | Use block *housing units* (invariant or less noisy under the DP design) as the default weight where suitable, and document the choice. |
| **No performance or cost tests in QA** | 🟡 | R-QA-3 tests function but not bytes, requests or load time. | Add Playwright checks for first-paint bytes, requests per profile click, and time to kNN result on a throttled profile. They protect both user experience and the Class B budget. |

---

## 5. What the plan gets right (keep these)

- The geometry and attribute split, and Parquet + PMTiles as the **data product**
  with the UI as a client of it. This is the core of a good common resource.
- **Client-only BYO data.** It's private by construction and costs nothing to host.
- Join reports, provenance on every variable, coverage shown in the UI.
- "Display yes, inference no" for overlapping ACS vintages, and the as-of
  anti-leakage rule in the backtest. Most neighborhood-change tools get this wrong.
- The Tract-Marcel benchmark and the gate-based evaluation. It holds methods to an
  external yardstick.
- The verification log as a living artifact. Extend it with the `RV-xx` items below.

---

## 6. Recommended changes, in priority order

1. **Hosting design (§8):** a custom domain in front of R2, immutable versioned
   paths, a < 512 MB per-object CI assertion, an R2 operations alert, and the HF
   mirror as kill switch. Evaluate Source Cooperative as primary or mirror. Use
   Zenodo DOIs for releases.
2. **Redistribution gate:** add `redistribution` to the source spec, define the
   public-reproducible build, and resolve the NHGIS and LTDB terms before planning
   published long-history layers.
3. **Re-sequence:** product Phase 1 has no research dependency. Move the
   2010→2020 crosswalk into Phase 2. Run PECOTA and R3 as a parallel research track.
4. **Front-end budgets:** a first-paint budget separate from lazily loaded DuckDB,
   a profile-optimized artifact, and quantized embeddings. Add byte and request
   tests to QA.
5. **Replace the Phase 6 workbench** with a Python client and notebooks.
6. **Governance:** an org-owned repo and accounts, a source-spec schema and
   contribution CI, `CITATION.cff`, a data semver and CHANGELOG, and analytics.
7. **Phase 0 additions:** test PMTiles through HF `/resolve` redirects, Census
   Geocoder CORS from the browser, Observable Framework vs Vite, and DuckDB-WASM
   cold-start on throttled mobile.

## 7. New verification items

| ID | Claim | Status | Source |
|---|---|---|---|
| RV-01 | r2.dev public access is rate limited and for development only. Caching and WAF need a custom domain. | ✅ (Cloudflare docs) | [R2 limits](https://developers.cloudflare.com/r2/platform/limits), [R2 public buckets](https://developers.cloudflare.com/r2/buckets/public-buckets/index.md) |
| RV-02 | Cloudflare's Free plan caches objects up to 512 MB. | 🟡 (doc mirrors and third-party sources; confirm on cloudflare.com) | [Workers limits (docs mirror)](https://cloudflare-docs-2tg.pages.dev/workers/platform/limits) |
| RV-03 | HF Hub anonymous limits are per IP per 5 minutes: about 3,000 resolver, 500 API and 100 page requests. Subject to change. | ✅ (HF docs, 2025 snapshot) | [HF Hub rate limits](https://huggingface.co/docs/hub/rate-limits) |
| RV-04 | Source Cooperative hosts public PMTiles and GeoParquet (for example Protomaps OSM). Free-tier terms unconfirmed after 2023. | 🟡 | [What is Source Cooperative?](https://radiant.earth/blog/2023/10/what-is-source-cooperative/), [Protomaps on Source](https://source.coop/protomaps/openstreetmap/tiles) |
| RV-05 | Observable Framework: repo updated in 2026, but there are user reports of unaddressed dependency alerts. `notebook-kit` is newer. | 🟡 | [framework#2045](https://github.com/observablehq/framework/issues/2045), [observablehq org](https://github.com/observablehq) |
| RV-06 | NHGIS terms do not permit redistribution without permission, and require citation. | ✅ | [NHGIS terms of use](https://www.nhgis.org/ipums-nhgis-terms-use), [NHGIS citation](https://www.nhgis.org/research/citation) |
| RV-07 | GitHub Actions public-repo runners: about 14 GB free disk, a 6 h job limit, a 10 GB Actions cache per repo, and scheduled workflows disabled after 60 days of repository inactivity. | ⏳ (from background knowledge; confirm in GitHub docs) | — |
| RV-08 | Census Geocoder: CORS support and rate limits for browser calls. | ⏳ (Phase 0) | — |
