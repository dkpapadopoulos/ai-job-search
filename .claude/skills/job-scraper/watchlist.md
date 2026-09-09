# Company Career-Page Watchlist

<!-- Localize this for your market. This file is swept by `/scrape watchlist`.
     The example below is a Zurich, Switzerland list (tech + pharma/life-sciences),
     verified 2026-06-15. Replace with your own target employers + city. -->

A curated list of employers to monitor directly for new postings, instead of relying
only on aggregators. Each company is tagged with a **fetch tier** that tells `/scrape`
*how* to read it (see the skill's "Watchlist mode" section).

**Target location filter:** `Zürich` / `Zurich`, plus Zurich-canton suburbs that show up
in postings: **Opfikon, Glattbrugg, Glattpark, Schlieren, Stäfa, Dübendorf, Winterthur**
(Winterthur ~20 min by train — borderline, keep). Treat all of these as "Zurich area".

> ⚠️ **Umlaut trap (verified bug):** match **both** `Zurich` and `Zürich` explicitly. A bare
> substring test for `urich` matches `Zurich` but NOT `Zürich` (the `ü` breaks the run `u-r-i-c-h`).
> Several boards (EthonAI, Lakera, Anthropic) label roles `Zürich` and were silently counted as 0
> until this was fixed. When filtering, lowercase and test for `zurich` OR `zürich` (plus the suburb
> names), never a single accent-stripped token.

---

## Tier 1 — Public JSON API (most reliable; fetch the API URL directly)

These ATS platforms serve all postings as unauthenticated JSON. `WebFetch` / `tavily_extract`
the URL, parse the JSON, then keep only jobs whose location matches the Zurich filter above.

> ⚠️ **GET vs POST:** every ATS below is a plain **GET** except **Workday**, which serves the same
> unauthenticated JSON only over **POST** (`WebFetch`/`tavily_extract` are GET-only and will fail on it).
> Workday rows are marked `POST` and must be fetched with a curl/Bash call — see the Workday bullet.
- **Greenhouse:** `https://boards-api.greenhouse.io/v1/boards/<slug>/jobs?content=true` → `jobs[].location.name`, `jobs[].absolute_url`. No server-side location filter — filter client-side.
- **Lever:** `https://api.lever.co/v0/postings/<slug>?mode=json` → fields `text`, `hostedUrl`, `categories.location`. **Do NOT use the `&location=` param** — it is exact-string and misses `Zürich, Switzerland`/suburb labels (e.g. it returned 1 ANYbotics job vs 9 when filtering client-side). Fetch the whole board, filter client-side.
- **Ashby:** `https://api.ashbyhq.com/posting-api/job-board/<slug>` → filter client-side on `jobs[].address.postalAddress.addressLocality` (and `jobs[].location`).
- **SmartRecruiters:** `https://api.smartrecruiters.com/v1/companies/<slug>/postings?limit=100&country=ch` → `content[].location.city`. (API can be slow; allow a longer timeout.)
- **Recruitee:** `https://<slug>.recruitee.com/api/offers` → filter client-side on `offers[].city`.
- **Workable:** `https://apply.workable.com/api/v1/widget/accounts/<slug>?details=true` → `jobs[].location.city`.
- **Personio:** `https://<slug>.jobs.personio.com/xml` (XML; common for Swiss/DE scale-ups) → each `<position>` has `<name>` (title), `<office>` (location), `<id>`; job URL is `https://<slug>.jobs.personio.com/job/<id>`. Filter client-side on `<office>`. (Slug is often the brand name but not always, e.g. MoonLake = `moonlaketx`; a wrong slug 307-redirects, a right one 200s.)
- **Workday (POST):** `POST https://<tenant>.myworkdayjobs.com/wday/cxs/<account>/<site>/jobs` with header `Content-Type: application/json` and body `{"appliedFacets":{},"limit":20,"offset":0,"searchText":"Zurich"}` → `jobPostings[].title`, `jobPostings[].locationsText`, `jobPostings[].externalPath`. Build the view URL as `https://<tenant>.myworkdayjobs.com/<site>` + `externalPath`. Page via `offset` if `total` > `limit`. GET-only tools can't read this — use curl/Bash.

Counts below are live as of the **last probe: 2026-06-15** (`total` = whole board, `ZH` = Zurich-area
after client-side filter). They drift — treat as a freshness hint, not a guarantee. Rows with ZH=0
are real Zurich employers whose open roles cycle; keep monitoring them.

| Company | Sector | ATS | Fetch URL | Last probe (umlaut-fixed) |
|---|---|---|---|---|
| Takeda | pharma | Workday `POST` | `https://takeda.wd3.myworkdayjobs.com/wday/cxs/takeda/External/jobs` (body `searchText:"Zurich"`) | 8/8 ⭐ (POST; labels `Zurich, Switzerland`) |
| Accenture | consulting/IT | Workday `POST` | `https://accenture.wd103.myworkdayjobs.com/wday/cxs/accenture/AccentureCareers/jobs` (body `searchText:"Zurich"`) | 9 hits 2026-07-21 ⭐ (POST; `locationsText` is null → filter client-side on `externalPath` containing `/Zurich/`, drops London etc. false matches; view URL = `https://accenture.wd103.myworkdayjobs.com/AccentureCareers` + `externalPath`) |
| DeepJudge | legal AI | Ashby | `https://api.ashbyhq.com/posting-api/job-board/deepjudge` | 12/19 ⭐ |
| Climeworks | cleantech | Recruitee | `https://climeworks.recruitee.com/api/offers` | 10/11 ⭐ |
| Avaloq | fintech | SmartRecruiters | `https://api.smartrecruiters.com/v1/companies/Avaloq1/postings?limit=100&country=ch` | 10/130 ⭐ |
| ANYbotics | robotics | Lever | `https://api.lever.co/v0/postings/anybotics?mode=json` | 9/12 ⭐ |
| EthonAI | industrial AI | Recruitee | `https://ethonai.recruitee.com/api/offers` | 6/9 ⭐ (labels `Zürich`) |
| GetYourGuide | tech | Greenhouse | `https://boards-api.greenhouse.io/v1/boards/getyourguide/jobs?content=true` | 6/59 |
| Lakera | AI security | Ashby | `https://api.ashbyhq.com/posting-api/job-board/lakera.ai` | 5/34 (labels `Zürich`) |
| Anthropic | AI | Greenhouse | `https://boards-api.greenhouse.io/v1/boards/anthropic/jobs?content=true` | 4/378 (labels `Zürich`) |
| Scandit | tech (CV) | Greenhouse | `https://boards-api.greenhouse.io/v1/boards/scandit/jobs?content=true` | 3/16 |
| Netcetera | IT | Workable | `https://apply.workable.com/api/v1/widget/accounts/netcetera?details=true` | 2/19 |
| Google DeepMind | AI research | Greenhouse | `https://boards-api.greenhouse.io/v1/boards/deepmind/jobs?content=true` | 1/33 |
| Mistral AI | AI | Lever | `https://api.lever.co/v0/postings/mistral?mode=json` | 1/171 |
| DeepL | AI | Ashby | `https://api.ashbyhq.com/posting-api/job-board/DeepL` | 1/23 |
| RepRisk | data/AI | SmartRecruiters | `https://api.smartrecruiters.com/v1/companies/RepRiskAG/postings?limit=100&country=ch` | 1/13 |
| Zurich Instruments | quantum/measurement | Lever | `https://api.lever.co/v0/postings/zurichinstruments?mode=json` | 1/10 |
| Palantir | tech | Lever | `https://api.lever.co/v0/postings/palantir?mode=json` | 0/236 — Zurich roles cycle |

> Resolved on probe (2026-07-21), ATS-fingerprinting the newly-added employers: **Accenture** promoted to
> Tier 1 (it runs on **Workday** — `accenture.wd103.myworkdayjobs.com/AccentureCareers`; clean CXS `POST`,
> 9 Zurich hits). **SAP** kept in Tier 3 but upgraded to ✅ fetchable — its SuccessFactors search page
> server-renders the location-filtered results, so `WebFetch` + parse works (no blind WebSearch needed).
> **Deloitte** is **Avature** (`apply.deloitte.com`), JS-rendered with no clean JSON → stays search-driven.
> **Hitachi** and **J&J** are both **Phenom** behind Akamai bot protection: the site *and* the Phenom
> `/api/jobs` endpoint return 403 to curl/WebFetch, so both stay search-driven (no fetchable endpoint).
>
> Resolved on probe (2026-06-16): **Takeda** promoted from Tier 3 to Tier 1 — its Workday CXS `POST`
> endpoint returns clean JSON (8 Zurich roles, incl. Change Enablement Partner) and is more reliable than
> the old Avature search. It's POST-only, so fetch via curl/Bash, not WebFetch. **Caveat (verified
> 2026-06-16):** the `searchText:"Zurich"` body is a *full-text* filter, not a location filter, and it
> only isolates Zurich roles cleanly on tenants (like Takeda) that label `locationsText` with the plain
> city. On tenants that show `"N Locations"` (Roche, Novartis, CSL, Temenos…) it under-counts badly, so
> those stay in Tier 2 — a reliable POST sweep there needs the per-tenant **Switzerland location-facet
> GUID** in `appliedFacets` (the facet values aren't returned by the basic list call; discovery TODO).
>
> Resolved on probe (2026-06-15): **Scandit** is Greenhouse (Workable board is empty). **Adnovum**'s
> Greenhouse slug 404s → it is on SuccessFactors, see Tier 3. **EthonAI/Lakera/Anthropic** jumped from
> 0 once the umlaut bug was fixed (they label roles `Zürich`). Dropped as dead/noisy: beekeeper (GH 404),
> speechify (1600+ jobs, mostly remote false-matches), quantco/SMG (0 and marginal).

---

## Tier 2 — Workday (public but needs POST; `/scrape` falls back to site-scoped search)

Workday exposes jobs only via `POST .../wday/cxs/<tenant>/<site>/jobs`, which `WebFetch` (GET-only)
can't call. For these, `/scrape` uses `WebSearch` scoped to the careers domain instead, e.g.
`site:roche.wd3.myworkdayjobs.com Zürich <role>`. (A future scripted version could POST the CXS API directly.)

| Company | Sector | Workday board URL |
|---|---|---|
| Roche (Innovation Center, Schlieren) | pharma | `https://roche.wd3.myworkdayjobs.com/en-US/roche-ext` |
| Novartis (Basel; commutable/hybrid) | pharma | `https://novartis.wd3.myworkdayjobs.com/en-US/Novartis_Careers` |
| CSL Vifor (Glattbrugg) | pharma | `https://csl.wd1.myworkdayjobs.com/CSL_External` |
| Swisscom | tech/telecom | `https://swisscom.wd103.myworkdayjobs.com/en-US/SwisscomExternalCareers` |
| Zühlke (Schlieren HQ) | tech consulting | `https://zuehlke.wd3.myworkdayjobs.com/Zuhlke-Careers` |
| NVIDIA | AI/hardware | `https://nvidia.wd5.myworkdayjobs.com/NVIDIAExternalCareerSite` |
| Sulzer (Winterthur) | industrial | `https://sulzer.wd502.myworkdayjobs.com/SulzerJobs` |
| Julius Baer | finance/tech | `https://juliusbaer.wd3.myworkdayjobs.com/Technology` |
| Temenos | banking software | `https://temenos.wd103.myworkdayjobs.com/Temenoscareers` |

---

## Tier 3 — Proprietary SPA / other (search-driven; some directly fetchable)

These run their own JS-rendered career sites with no public GET API. `/scrape` discovers
roles via `WebSearch` (company + "careers" + Zurich + role). Three are plain static HTML and
**can** be fetched directly (marked ✅ fetchable).

| Company | Sector | Careers URL | Direct-fetchable? |
|---|---|---|---|
| IBM Research Zurich | AI research | `https://www.zurich.ibm.com/careers/` | ✅ static HTML |
| Disney Research\|Studios | AI research | `https://studios.disneyresearch.com/open-positions/` | ✅ static HTML |
| Google | tech | `https://www.google.com/about/careers/applications/jobs/results?location=Zurich%2C+Switzerland` | ❌ SPA — search only |
| Microsoft | tech/AI | `https://jobs.careers.microsoft.com/global/en/search?lc=Zurich%2C%20Switzerland` | ❌ SPA |
| Meta | tech/AI | `https://www.metacareers.com/locations/zurich/` | ❌ SPA |
| Apple (Zurich Vision Lab) | AI | `https://jobs.apple.com/en-us/search?location=zurich-ZUR` | ❌ SPA |
| ABB | industrial/automation | `https://careers.abb/global/en/abb-switzerland-careers` | ❌ SPA |
| Adnovum | IT/security | `https://careers.adnovum.com/search/?locationsearch=zurich` | ❌ SuccessFactors (GH slug is dead) |
| SAP (Regensdorf ZH) | enterprise software | `https://jobs.sap.com/search/?q=&locationsearch=Zurich` | ✅ static HTML — SuccessFactors **server-renders** the location-filtered results; `WebFetch`/`tavily_extract` and parse `jobTitle-link` (title) + `jobLocation` (e.g. `Zürich-Flughafen, Zurich, CH`). Verified 2026-07-21. |
| Johnson & Johnson (Zug/Schaffhausen) ⚠️ | pharma/medtech | `https://www.careers.jnj.com/en/jobs` | ❌ Phenom, Akamai bot-gated (403 to curl/WebFetch **and** its `/api/jobs`) — `site:careers.jnj.com (Zug OR Zürich) <role>`; Zug ~40 min = borderline commute |
| Hitachi (group: Digital, Vantara, Rail, GlobalLogic) | tech/industrial | `https://careers.hitachi.com/search/jobs/in/zh-zurich` | ❌ Phenom, Akamai bot-gated (403 to curl/WebFetch **and** its `/api/jobs`) — `site:careers.hitachi.com Zürich <role>` (distinct from **Hitachi Energy** at hitachienergy.com) |
| Deloitte (Zurich) | consulting/advisory | `https://apply.deloitte.com` (Avature portal; landing `deloitte.com/ch/en/careers/job-search.html`) | ❌ Avature SPA, no clean JSON — `Deloitte Switzerland careers Zürich <role>` |
| u-blox (Thalwil ZH) ⚠️ | semiconductors / IoT | `https://www.u-blox.com/en/job-openings` | ❌ Salesforce-recruiting SPA (`fs-4627.my.salesforce-sites.com`, JS-rendered) — `site:jobs.ch u-blox <role>` / LinkedIn. ⚠️ deep-tech hardware/embedded eng shop → PM/program/ops roles only, weak fit. |
| MSD / Merck & Co. (The Circle, Zurich + Lucerne HQ) ⚠️ | pharma | `https://jobs.merck.com` (also `https://www.msd.ch/en/`) | ❌ Phenom, **geo-blocked** to datacenter IPs (`jobs.merck.com/api/...` returns "not available in your country") — `site:jobs.merck.com Switzerland <role>`, `MSD Switzerland careers (Zurich OR Lucerne) <role>` / LinkedIn |

> SAP, Accenture and J&J are **large global boards** — Zurich-area roles are a small fraction of
> the total, so most postings will not match the location filter (expected). J&J's nearest offices
> are **Zug and Schaffhausen** (no Zurich office), so treat its hits as borderline-commute per the
> location filter, not core Zurich.

> **MSD location:** Swiss HQ is the new "One Roof" building in **Lucerne** (Rösslimatt, ~750 staff,
> opened May 2026); R&D centre in **Schachen (LU)**; plus an **office at The Circle, Zurich Airport**
> (Opfikon-adjacent). Treat **The Circle / Zurich** roles as in-area; **Lucerne / Kriens / Schachen**
> roles are borderline (~50-60 min, relocation-worthy only). Pharma target sector.

> Dropped from Tier 3 as weak/low-fit: **Oracle Labs** (board mid-migration) and **Sonova**
> (niche medtech SuccessFactors). Re-add if a relevant role appears.

---

## Finance / insurance (Zurich; mostly search-only)

Large Zurich finance/insurance employers. None expose a clean public JSON API, so `/scrape`
treats these like Tier 2/3 (scoped search + fetch static job pages). Avaloq (fintech) is already
in Tier 1 above.

| Company | Sector | Careers URL | How to read |
|---|---|---|---|
| UBS | banking/tech | `https://jobs.ubs.com/TGnewUI/Search/Home/Home?partnerid=25008&siteid=5012` | Taleo SPA — `site:jobs.ubs.com Zürich <role>` search |
| Swiss Re | insurance/data | `https://careers.swissre.com/` | search `Swiss Re careers Zürich <role>` |
| Zurich Insurance | insurance/tech | `https://careers.zurich.com/digital-and-technology` | search `Zurich Insurance careers Zürich <role>` |
| Julius Baer | wealth/tech | `https://juliusbaer.wd3.myworkdayjobs.com/Technology` | Workday (Tier 2) |
| Temenos | banking software | `https://temenos.wd103.myworkdayjobs.com/Temenoscareers` | Workday (Tier 2) |
| Sygnum Bank ⭐ | digital-asset bank / fintech | `https://www.sygnum.com/working-at-sygnum-careers` | JS-rendered custom ATS (not SR/Personio/Teamtailor — probed 2026-07-21) — search `Sygnum Bank careers Zürich <role>` + LinkedIn/jobs.ch. **Strong fit**: Zurich hub, product/ops/compliance/risk/leadership roles (e.g. Head of Corporate Clients, Head Digital Wealth, Product Manager Tokenization). |

---

## Recently-funded scaleups / startups (monitor for ops / PM / CoS / BizOps leadership only)

Added 2026-07-21 from a "Top Swiss startup funding rounds (last 6 months)" list. These are mostly
**research biotechs** — these are research biotechs, so unless you are a scientist **skip clinical/scientific/lab roles** and keep only
ops, PM, program, strategy, BizOps, finance-ops and Chief-of-Staff-type leadership. Apply the usual
Zurich-area location filter (many roles are elsewhere).

| Company | Location (HQ) | Sector | ATS / fetch | Fit / notes |
|---|---|---|---|---|
| terralayr | Zug (ZG) | clean-energy / grid storage | Personio ⭐ `https://terralayr.jobs.personio.com/xml` | Best fit here: open **Chief of Staff**, CEO Office, Strategy & Commercial, Operations. **But** `<office>` shows most roles are **Berlin / Hamburg / London** (German energy market) — filter `<office>` for Zug/CH/remote; expect few to pass. Public site `trlyr.com/careers`. |
| MoonLake Therapeutics | Zug (ZG) ⚠️ | biotech (inflammation; NASDAQ: MLTX) | Personio `https://moonlaketx.jobs.personio.com/xml` (8 roles 2026-07-21) | Zug = borderline commute. Clinical/commercial drug-launch roles; keep only ops/BizOps/finance-ops. |
| LimmaTech Biologics | Schlieren (ZH) | biotech (vaccines / AMR) | search-driven — `https://lmtbio.com/careers` (real domain is **lmtbio.com**, not limmatech.com) + `jobs.ch/en/companies/33079-limmatech-biologics-ag` | Core Zurich-area but tiny board (1 clinical PM role 2026-07-21); mostly scientific → low fit. |

> Excluded from this list as out of Zurich commute range: **Memo Therapeutics** (Reinach BL — Basel area)
> and **Kandou AI** (Lausanne VD — Romandie, French-speaking). Re-add only if they open a remote-CH or
> Zurich role. (Only 5 of the source list's 20 rows were legible in the screenshot — the other 15 are TBD.)

---

## Excluded (no confirmed Zurich-area office — verified 2026-06-15)

Lonza (Basel/Visp/Stein), Straumann (Basel), Bühler (Uzwil), Bristol Myers Squibb
(Zug/Boudry). Novartis is Basel-only but kept in Tier 2 as commutable/hybrid.
**J&J** (Zug/Schaffhausen) was moved from Excluded to Tier 3 on request (2026-07-21) — tracked as
borderline-commute, not core Zurich.

---

## Re-probing freshness

The Tier-1 "Last probe" counts go stale. To refresh them (and catch dead slugs), re-run a quick
GET against each Tier-1 `Fetch URL` and recount Zurich matches — the same check used to build this
list on 2026-06-15. Slugs most worth re-checking periodically: any that drift to repeated errors.
