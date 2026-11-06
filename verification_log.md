# pdb-browser — Verification log

> Fact checks on the core assumptions in `requirements.md`, `methodology.md`,
> `research_plan.md`, and `methodology_alternatives.md`. Last pass: 2026-11-06.
> Method: web search of primary sources (census.gov, fhfa.gov, huduser.gov, data.cdc.gov,
> Earth Engine catalog), plus reputable secondary sources where noted. Direct
> downloads from census.gov were blocked in the research environment, so
> file-level checks (V-03 filename, V-14) are deferred to the Phase 0 spike.
>
> Status key:
> - **✅ Verified:** confirmed by a primary source.
> - **⚠️ Corrected:** the doc assumption was wrong and has been fixed.
> - **🟡 Partial:** confirmed with caveats or secondary sources only.
> - **⏳ Open:** needs a file-level or hands-on check.

| ID | Assumption | Status | Finding | Doc impact |
|---|---|---|---|---|
| V-01 | Which ACS 5-year vintage first used 2020 tracts | ✅ | **2016–2020** (released 2022-03-17) was the first 5-year release on 2020 Census geography: blocks, BGs, tracts, places, ZCTAs. 2015–19 and earlier use 2010 geography. For 5-year data, the end year sets the boundary vintage. | methodology.md §5.2, research_plan.md R1.3: 11 vintages need 2010→2020 harmonization |
| V-02 | FHFA publishes an annual tract-level house price index | 🟡 | **Exists:** FHFA's annual index covers county, ZIP and **tract** levels from all-transactions repeat sales, and FHFA labels it **developmental**. Its research release (WP21-01) has about 63,122 tracts on **2010** definitions, annual 1989–2021. The current production file vintage and its 2020-tract availability are unconfirmed. | methodology.md §5.2 / §6, requirements.md §5.3: flagged as developmental and 2010-tract |
| V-03 | A 2026 PDB exists | ✅ / ⏳ | The Census API dataset page for the 2026 PDB lists **2020–2024 ACS 5-year** estimates, tract self-response, 2020 operational variables and the LRS. **Naming gotcha:** the 2024 PDB (2018–22 ACS) was released 2026-01-22. The CSV filename is unconfirmed (⏳). | requirements.md §3.1, open question 1 resolved |
| V-04 | USPS vacancy data is an openly usable tract series | ⚠️ | **Restricted.** Under HUD's agreement with USPS, access is limited to registered **government and nonprofit** users. It is quarterly, on 2020 tracts back to 2010 Q1. Older "public" descriptions predate the current agreement. | Demoted to "internal input only if eligible; never redistributed" everywhere it appeared |
| V-05 | EJScreen is unavailable from EPA | ✅ | EPA removed EJScreen on **2025-02-05**. The Public Environmental Data Partners (PEDP) reconstruction launched 2025-02-07 and was still active in later reporting. Social Explorer offers 2016–24 archives to account holders. Sierra Club v. EPA was reportedly dismissed for lack of standing on 2026-03-13 (single source, 🟡). | requirements.md §5.3, open question 4 resolved |
| V-06 | AlphaEarth satellite embeddings are available | ✅ | Google DeepMind's AlphaEarth Foundations **Satellite Embedding** dataset: **64-dim**, **10 m**, annual, `GOOGLE/SATELLITE_EMBEDDING/V1/ANNUAL`, covering **2017–2024** per the English Earth Engine catalog. The license and terms for redistributing derived tract aggregates are ⏳. | methodology.md §1.4, §6 |
| V-07 | Census Community Resilience Estimates were removed | ⚠️ | **Wrong.** The 2024 CRE was released 2026-01-29. Only the **CRE Equity Supplement** is listed as discontinued (Data Rescue Project). There was never a 2020 CRE. | methodology.md §1.2 corrected |
| V-08 | Autoencoder vs PCA result (about 19% lower reconstruction error) | 🟡 | Confirmed at abstract level: the conference paper "Capturing Non-linear Neighbourhood Structure: An Autoencoder Approach to Census-based Dimensionality Reduction" (Liverpool/Leicester, Zenodo, 2026-04-04) reports 408 variables × 188,880 E&W Output Areas and 19.4% lower error. The full text is unread and peer-review status is unknown. | methodology.md §1.4, references |
| V-09 | County/place ACS 1-year estimates exist annually (anchors for the annualized model) | ⚠️ | **Gap in 2020:** standard 2020 ACS 1-year products were **not released**. There are 54 experimental tables at national and state level only, which the Census Bureau advises against comparing with standard estimates. Standard 1-year products for areas of 65k+ exist for the other years. The 2016–20 5-year used standard methods. | methodology.md §5.2 and research_plan.md R1.3: 2020 is a missing anchor year; the COVID flag is softened |
| V-10 | LODES is an annual tract-capable series on 2020 geography | ✅ | LODES 8 is on **2020 blocks**, with historical years recast. Most states have **2002–2022**, with some state and year gaps (originally AK 2017–20, AR and MS 2019–20). The LODES 8.4 technical documentation is dated 2025-12. | Anchors and supplemental source list |
| V-11 | Cloudflare R2 free tier | ✅ | 10 GB-month Standard storage, 1M Class A and 10M Class B operations per month, **zero egress**. Beyond that: $0.015/GB-month, plus operation charges with no hard cap. Infrequent Access is excluded from the free tier. | requirements.md §8.1 detail added. Watch operation counts from range requests (each PMTiles/Parquet range read is a Class B op). |
| V-12 | CDC PLACES tract data is still available | ✅ | The 2025 release (data.cdc.gov, catalog last modified 2025-12-04) has 40 tract measures from 2023 BRFSS (2022 for biennial measures). ZCTA data for one release had a posting issue. | requirements.md §5.3 |
| V-13 | HUD ZIP crosswalk is free; Opportunity Atlas is on 2010 tracts | ✅ / 🟡 | The HUD-USPS crosswalk is **free with registration / API key**, quarterly, with a residential ratio, and on 2020 tracts from **2023 Q1** (2010 geographies earlier). The Opportunity Atlas uses 2000/2010 Census-linked data and publishes a 2010↔2020 tract crosswalk, which implies 2010 tract keys (inferred, 🟡). | requirements.md §5.2 |
| V-14 | The PDB tract file carries a MOE for each ACS estimate | ⏳ | **Not confirmed.** The PDB documentation includes an MOE-calculation appendix, but the column-level MOE coverage and naming were not found. **This is core to methodology.md §4.3**, so a fallback is specified: pull MOEs for the clustering variables from the ACS detailed-table API. | requirements.md R-PDB-3; **Phase 0 spike check** |

## Other assumptions spot-checked (no change needed)
- **Non-overlapping ACS comparisons:** Census guidance is explicit (see the
  methodology.md references).
- **2020–2024 ACS 5-year release date:** scheduled 2026-01-29 per a Census tip sheet.
- **LTDB 2020 coverage:** limited to race/ethnicity on 2010 boundaries.
- **GitHub Pages** blocks files ≥ 100 MB.
- **2025 PDB content:** 2019–23 ACS on 2020 geographies.

## Still unverified (lower priority; check before user-facing use)
- Exact OAC 2021/22 paper year (sources say 2023 and 2024), its final variable
  count, and k.
- Tapestry 2026 vintage methodology changes vs 2025.
- Citations in methodology_alternatives.md (written from background knowledge;
  that file's header says so).
- BEA Regional Price Parities geography (state / metro / nonmetro remainder) and
  release lag. HUD SAFMR coverage.
- Overture and OSM license terms for redistributing derived per-tract counts.
- Exact FHFA production tract-HPI file and its update cadence (V-02 follow-up).

## Phase 0 spike checklist (file-level, needs network to census.gov)
1. Download the 2026 PDB tract CSV. Confirm the filename, rows/columns, ACS suffix
   `_ACS_20_24`, and **MOE column coverage (V-14)**.
2. Diff the variable list vs the 2025 PDB.
3. Pull one FHFA tract HPI file. Confirm the vintage, tract count, and 2010 vs 2020 keys.
4. Check the AlphaEarth dataset license in the Earth Engine catalog.
5. Confirm whether HUD USPS vacancy access is available to this project (eligibility).

## Sources
- ACS geography: [Geography Boundaries by Year (2020)](https://www.census.gov/programs-surveys/acs/geography-acs/geography-boundaries-by-year.2020.html), [Hawaii SDC 2016–2020 release note](https://census.hawaii.gov/?p=9978), [Census 5-year webinar deck](https://www.census.gov/content/dam/Census/newsroom/press-kits/2022/20220310-presentation-acs-5year-webinar.pdf), [Esri: 2016–2020 updated boundaries](https://www.esri.com/arcgis-blog/products/arcgis-living-atlas/mapping/acs-2016-2020-updated-boundaries)
- FHFA: [HPI FAQs](https://www.fhfa.gov/faqs/hpi), [WP21-01](https://www.fhfa.gov/PolicyProgramsResearch/Research/Pages/wp2101.aspx), [WP21-01 FAQ](https://fhfa.gov/sites/default/files/2023-03/wp2101-faqs.pdf)
- PDB: [Planning Database 2026 (API)](https://census.gov/data/developers/data-sets/planning-database.2026.html), [2024 PDB press release](https://www.census.gov/newsroom/press-releases/2026/2024-planning-database.html), [2023 PDB](https://www.census.gov/topics/research/guidance/planning-databases/2023.html), [Census tip sheet Jan 2026](https://www.census.gov/newsroom/tip-sheets/2026/tp26-01.html), [2018 Tract PDB documentation](https://census.gov/content/dam/Census/topics/research/2018_Tract_PDB_Documentation_V4.pdf)
- USPS vacancy: [HUD USPS data login](https://huduser.gov/apps/public/usps/login), [HUD USPS home](https://www.huduser.gov/apps/public/usps/home)
- EJScreen: [EDGI: EPA removes EJScreen](https://envirodatagov.org/?p=11456), [SEJ](https://www.sej.org/node/49426), [screening-tools.com mirror](https://screening-tools.com/epa-ejscreen), [Harvard EELP tracker](https://eelp.law.harvard.edu/?p=8168), [Social Explorer archive](https://home.socialexplorer.com/post/the-ejscreen-tool-social-explorer-provides-access-to-archived-environmental-justice-data)
- AlphaEarth: [DeepMind announcement](https://deepmind.google/blog/alphaearth-foundations-helps-map-our-planet-in-unprecedented-detail/?hl=es), [Earth Engine catalog](https://developers.google.com/earth-engine/datasets/catalog/GOOGLE_SATELLITE_EMBEDDING_V1_ANNUAL)
- CRE: [Census CRE](https://www.census.gov/data/experimental-data-products/cre.html), [DRP: Equity Supplement](https://portal.datarescueproject.org/datasets/community-resilience-estimates-equity-supplement/), [Census CRE webinar 2025](https://www.census.gov/data/academy/webinars/2025/cre-data-and-tools.html)
- Autoencoder paper: [Zenodo 19420685](https://zenodo.org/records/19420685)
- 2020 ACS 1-year: [Census press release (experimental)](https://www.census.gov/newsroom/press-releases/2021/experimental-2020-acs-1-year-data.html), [Pandemic impact blog](https://www.census.gov/newsroom/blogs/random-samplings/2021/10/pandemic-impact-on-2020-acs-1-year-data.html)
- LODES: [LODES 8.4 tech doc](https://lehd.ces.census.gov/data/lodes/LODES8/LODESTechDoc8.4.pdf), [lehdr README](https://www.stats.bris.ac.uk/R/web/packages/lehdr/readme/README.html)
- R2: [Cloudflare R2](https://www.cloudflare.com/en-gb/r2/), [Upstash R2 pricing 2026](https://upstash.com/kb/cloudflare-r2-pricing-in-2026-storage-operations-and-real-monthly-bills)
- PLACES: [data.gov 2025 tract release](https://catalog.data.gov/dataset/places-census-tract-data-gis-friendly-format-2025-release-4c3af), [CDCPLACES 1.2.0](https://www.r-bloggers.com/2026/02/cdcplaces-1-2-0/)
- HUD crosswalk: [HUD USPS crosswalk](https://huduser.gov/portal/node/7785), [Planetizen: crosswalk API](https://www.planetizen.com/node/102739)
- Opportunity Atlas: [Opportunity Insights](https://opportunityinsights.org/paper/the-opportunity-atlas/)
