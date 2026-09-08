---
name: scrape
description: >
  Finds new job postings matching your profile via installed portal-search CLIs
  (LinkedIn, freehire, and any skills added with /add-portal), then Tavily search
  (when configured) or built-in WebSearch for the rest of the market, plus a curated
  company watchlist mode. Deduplicates across runs. Triggers on: job scrape, find jobs,
  search jobs, new jobs, job search, scrape jobs, /scrape
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(bun --version), Bash(bun run .agents/skills/*/cli/src/cli.ts *), WebFetch, WebSearch, Agent, AskUserQuestion, mcp__tavily-mcp__tavily_search, mcp__tavily-mcp__tavily_extract, mcp__tavily-mcp__tavily_research
---

# Job Scraper

---

## How It Works

This skill searches job sites using targeted queries based on your profile, deduplicates against previously seen jobs and the application tracker, and presents new matches with a quick fit assessment.

**Search backend (augment-with-fallback):** if the **Tavily MCP tools** (`mcp__tavily-mcp__tavily_search`, `mcp__tavily-mcp__tavily_extract`) are present in your current tool list, prefer them — they give better international results and cleaner page extraction. If they are **not** available (no `TAVILY_API_KEY`, MCP server failed to start, or Tavily not installed), fall back to the built-in **WebSearch** / **WebFetch** tools. The skill works fully on the built-ins alone; Tavily is an optional quality upgrade. See `../../../.mcp.json` and the README's Tavily section for setup.

> **How to tell which backend is active:** check whether `mcp__tavily-mcp__tavily_search` appears in the tools available to you this session. If yes → use Tavily. If no → use WebSearch/WebFetch. Do not assume; a present `.mcp.json` does not guarantee the tool loaded (a missing/invalid key makes the server fail silently).

It is **country-agnostic**: the sites and query terms come from `search-queries.md`, which you localize for your market via `/setup --section search`. Universal sources (LinkedIn Jobs, Indeed, Google Jobs) work in any country with no setup; local boards are added per market.

> **Note on portal CLIs:** portal-search CLIs live under `.agents/skills/`. Step 1b runs every
> **enabled** one first — for this fork `linkedin-search` and `freehire-search`; the Danish demo
> boards ship `enabled: false`. Sources with no CLI (jobs.ch, jobscout24.ch, Indeed, Google Jobs,
> company career pages) are covered by the Step 1c Tavily/WebSearch pass. Mind the LinkedIn CLI's
> keep-volume-low ToS caveat in its SKILL.md.

## Invocation

The user triggers this skill by saying things like:
- "Find new jobs"
- "Scrape for jobs"
- "Any new positions?"
- "/scrape"

Optional arguments:
- A focus area, e.g. "/scrape data science" or "/scrape geophysics"
- "broad" to run all search categories, e.g. "/scrape broad"
- "watchlist" to sweep the curated company career pages in `watchlist.md` instead of running keyword searches, e.g. "/scrape watchlist" (see "Watchlist mode" below)

---

## Execution Steps

### Step 0: Load State

1. Read `job_scraper/seen_jobs.json` (create if missing - start with `{"seen": {}}`)
2. Read `job_search_tracker.csv` to extract already-applied companies+roles
3. Read `search-queries.md` (this directory) for the search strategy

### Step 1: Search

Read `search-queries.md` (this directory) for the search strategy. By default, run the top 3 priority query categories. If the user said "broad", run all categories. If the user specified a focus area (e.g. "AI enablement"), prioritize queries from that category.

**Use the installed CLI tools as the primary search mechanism, then cover the rest of the market with the Tavily/WebSearch pass.** For this fork the CLI pass is `linkedin-search` and `freehire-search` (the Danish demo boards ship `enabled: false`); the Swiss boards in `search-queries.md` have no CLI and are always searched through Step 1c.

#### 1a. Check bun availability

```bash
bun --version
```

If this fails (bun not installed), skip to **1c** for all portals and note the fallback in the Step 5 output.

#### 1b. Run CLI tools (primary — run these in parallel where possible)

Discover all installed portal CLI skills by reading every `SKILL.md` found under `.agents/skills/*/SKILL.md`. Each file documents that portal's exact CLI flags and usage examples. **Use each portal's own documented interface — do not guess flags.** This approach automatically includes any new portals added via `/add-portal` without requiring changes to this file.

**Honor the `enabled` toggle.** A portal is enabled unless its `SKILL.md` frontmatter sets `enabled: false` (a missing key means enabled — the default). Skip each disabled portal and record it for the Step 5 summary. A fork can thus keep a portal installed but sit out a run without deleting its directory.

For each **enabled** portal skill:

1. Read its `SKILL.md` to find the correct `bun run …` invocation and supported flags.
2. Translate the query terms from `search-queries.md` into that portal's flag format (e.g. `--key`, `--search-string`, `--query`, filter codes — whatever the portal's SKILL.md specifies).
3. Scope to the last 14 days using the portal's supported recency **filter** flag (`--jobage`, `--since <YYYY-MM-DD>`, etc. — as documented per portal). A portal with **no recency flag** (jobdanmark offers none) still gets scoped: every portal's search output carries a `date` field, so filter client-side — drop results whose `date` is older than 14 days after the call returns, and never invent a flag the portal's SKILL.md does not document (the CLIs reject unknown flags). `--order PublicationDate` is a sort, and a sort is not a filter — pairing it with a `--limit` is a defensible approximation on a portal that offers nothing better (jobnet), but apply the client-side date filter on top all the same.
4. Cap results to ~20 per call using the portal's limit flag.
5. Use `--format json` for machine-readable output.

Run all portal CLI calls in parallel where possible using the Agent tool. Collect all `results` arrays into a single pool for Step 2, keeping each result tagged with its source portal skill (for Step 2 `detail` lookups).

If a CLI tool exits with a non-zero code, log the error message and continue — do not abort the whole search.

#### 1c. WebSearch fallback

Use this pass for:
- Sources listed in `search-queries.md` that do **not** have a corresponding directory under `.agents/skills/` — for this fork that means jobs.ch, jobscout24.ch, Indeed, Google Jobs and company career pages
- Any portal whose CLI fails at runtime
- When bun is unavailable (Step 1a failed)

For each search:
- **If `mcp__tavily-mcp__tavily_search` is available:** use it. Pass your localized query. To steer results to your market, set Tavily's geographic/topic parameters where the tool supports them (e.g. a `country` parameter set to your market, and a news/general `topic`) — only use parameter names/values the tool's schema actually accepts; do not invent enum values. Otherwise just embed the city/country in the query string as usual.
- **Otherwise:** use the built-in `WebSearch` with the same queries.
- Draw the queries and sites from `search-queries.md` (universal sources like linkedin.com/jobs and indeed.com work anywhere; local boards are listed there per market).
- Target your configured geographic area.
- Look for postings from the last 14 days.

Tag each Step 1c result as WebSearch-sourced, keeping the portal tag when the pass stands in for an installed portal whose CLI failed. Step 4 persists this as the entry's `source`, and Step 5 reports which portals ran on the fallback this run.

### Step 2: Fetch & Parse

For each promising result from Step 1:

**From CLI results:** Search output already includes title, company, location, date,
and URL. For jobs worth a deeper look, fetch full detail with that portal's `detail`
command (see its SKILL.md — do not guess flags) to extract **key requirements**,
**application deadline**, and a brief description snippet.

**Closed-at-source detection:** `linkedin-search detail` also returns `isActive`.
`false` means the posting page itself renders LinkedIn's "No longer accepting
applications" banner — the job died between being indexed and being fetched (expired
LinkedIn URLs redirect to *similar live jobs*, so a search hit can be a ghost). Mark
such a job, never silently drop it: write its entry to `seen_jobs.json` in Step 4 with
`"status": "expired"` and leave it out of the Step 5 presentation — an absent entry
looks identical to a job never seen, and the recorded status is what makes a later
ghost report self-triaging. `isActive: true` is only the absence of that banner, not
proof the posting is open; deadlines and dead URLs remain `/rank`'s job.

**From Tavily/WebSearch results:** Retrieve the posting page with
`mcp__tavily-mcp__tavily_extract` when it is available (cleaner content), otherwise
`WebFetch`, and extract the same fields manually. If it returns HTTP 403, retry with browser headers via curl per
`.claude/skills/job-application-assistant/09-web-research.md` before giving up — most
bank and corporate sites reject WebFetch's user agent while serving browsers normally.

**Store a URL that actually resolves to the posting.** A listing-page URL with a
`#fragment` appended (`.../jobs/ciso/#ikerian`) is not a posting: it fetches fine and
returns unrelated job titles, which makes every later `/rank` and `/apply` run fail on
that entry. When WebSearch only yields a listing page, search the employer's own careers
site for the role and store that URL instead, or drop the candidate rather than saving a
fragment link.

> **Fetchability note:** Individual **LinkedIn** job pages (`linkedin.com/jobs/view/...`) are **login-gated** — both `WebFetch` and `tavily_extract` return only a teaser, not the full description (it is an auth wall, not a rendering problem, so Tavily does not bypass it). Use LinkedIn for *discovery* (titles, companies, locations, deep links from Step 1), but for the full posting prefer **fetchable sources**: Indeed, the company's own career page, or local boards. If only a gated URL is available and the user wants to apply, the title/company/location from the search result is enough to present the match; the full description can be pulled later by pasting the posting text into `/apply` (which supports pasted text when a URL is blocked).

For every candidate:
- Skip if the URL or company+title combo already exists in `seen_jobs.json`
- Skip if the company+role already appears in `job_search_tracker.csv`

### Step 2.5: Mass-Posting Detection (within this run)

A distribution pattern worth flagging to the user as a caution signal, not as an accusation against the employer - it describes how a listing is being distributed, not a verdict on whether the company is legitimate. It alone proves nothing is wrong (companies do legitimately hire the same role across several cities); flag it so the user can factor it in when deciding whether to invest time, don't downgrade fit or silently exclude the result because of it.

If two or more results in this run's pool (from the same company, or sharing the same req/job ID visible in the URL or title) have substantially the same description and differ only in city/location/title, don't present them as separate rows. Consolidate into a single row and note the spread, e.g. "posted identically across 6 cities (BR, MX, GT)".

### Step 3: Quick Fit Assessment

For each new job, do a rapid fit check (NOT the full evaluation from `04-job-evaluation.md` - just a quick signal):

- **High match**: Role directly involves your core skills
- **Medium match**: Role is adjacent to your experience
- **Low match**: Role requires significant skills you lack

**Language override:** before assigning a match level, check the posting against `04-job-evaluation.md`'s Language Gate (a required language you haven't declared at all in your CLAUDE.md Languages table). A required language that's entirely undeclared overrides skill fit: mark it **Low** regardless of how well the skills align, and name it in the highlight bullets so it isn't buried under an otherwise-good-looking match. A **declared** language at a requirement that reads higher than your declared level is *not* an override — score fit normally, but add a red-flag bullet under that job's highlights (Step 5) quoting the posting's requirement next to your declared level, so the gap is visible without being auto-downgraded.

### Step 4: Deduplicate & Store

1. Add ALL fetched jobs (new and skipped) to `seen_jobs.json` with structure:
```json
{
  "seen": {
    "<url_or_company_title_key>": {
      "title": "...",
      "company": "...",
      "url": "...",
      "first_seen": "YYYY-MM-DD",
      "posted_date": "YYYY-MM-DD" | null,
      "deadline": "YYYY-MM-DD" | null,
      "fit": "high/medium/low",
      "status": "new/skipped/ranked/expired/drafted/applied/discarded",
      "portal": "<source portal skill, site or board, e.g. linkedin-search, jobs.ch, watchlist:<company>>",
      "source": "cli/websearch"
    }
  }
}
```

The `portal` field records which source site or board the result came from (search backend, `site:` domain, or watchlist company). Entries written before this field existed lack it; do not backfill.

The `source` field records which mechanism produced the entry: `cli` for Step 1b portal-CLI output, `websearch` for the Step 1c fallback. This is what keeps a ghost-job report diagnosable after the run's summary is gone: a stored entry whose URL later resolves to nothing (or to a different job) reads very differently depending on whether it came from live CLI output or from a search index that can be weeks stale - and a presented job with no entry here at all points at fabrication, which Rule 1 forbids. Entries written before this field existed lack it; never backfill it - the mechanism was not recorded.

`/rank` extends this schema additively: ranked entries also carry `rank_score` (0–100 overall score), `rank_verdict` (fit band, e.g. "strong fit"), `rank_date` (ISO date of ranking), the veto fields `location_verdict` and `language_gate` (both PASS/FAIL/FLAG) with `language_note` (the quoted requirement explaining a non-PASS), and `strengths`/`gaps` (1-3 verbatim bullets each, copied from the scoring agent's findings). The `status` field is set to `"ranked"`. Do not drop any of these fields when re-writing entries. Entries ranked before `strengths`/`gaps` existed simply lack them; readers tolerate their absence and never backfill by guessing. Entries ranked before the verdict rename may carry a legacy PASS/FAIL/FLAG string in `location` - read that as the verdict when `location_verdict` is absent; in fresh entries `location` is always a place, never a verdict.

`deadline` is a base field rather than a `/rank` extension: Step 2's detail fetch already extracts the application deadline, so it is written when the job is first seen and refreshed by `/rank` Step 4 when a scoring agent returns a different value. `null` means the posting states no deadline; a missing key means the entry predates this field - **never infer a deadline** from either, and never backfill by guessing.

`posted_date` is the posting's own publication date, taken from the `date` field Step 2's contract already guarantees on every portal CLI's search output. Step 1b uses that date to scope the run to the last 14 days and then drops it, so nothing downstream can distinguish a posting published yesterday from one published two years ago - `first_seen` is when this scraper first saw the entry, not when the employer posted it. Persisting it makes Step 1b's window auditable after the run and gives `/rank` a freshness signal to weigh, instead of rediscovering the date and recording it in prose that nothing reads. That gap landed for real: a freehire-search posting dated 2024-05-13 was scraped and ranked Strong Fit at position 1 of 133, its own scoring note observing the listing "may be long stale" with nothing able to act on it. `null` means the portal returned no date for that result (the CLIs emit `date: null` when a listing omits it); a missing key means the entry predates this field - **never infer a posting date** from either, and never backfill by guessing.

2. Only present jobs NOT already in the seen list or tracker.

### Step 4.5: Generate Referral Contact Links (High & Medium Fit Only)

For every job from this run with `fit` of **high** or **medium** (skip low-fit jobs),
build two LinkedIn people-search URLs so the user can find a recruiter or team member to
reach out to for a referral or a warm intro. This is deliberately a link-generation step,
not an automated lookup: no scraping, no third-party API, zero runtime dependencies or
credentials required.

**A. Recruiters / Talent Acquisition (the referral path)**
```
https://www.linkedin.com/search/results/people/?keywords=<url-encoded "<Company Name> recruiter">&origin=GLOBAL_SEARCH_HEADER
```

**B. Role/team peers (informational-outreach / warm-intro path)**
```
https://www.linkedin.com/search/results/people/?keywords=<url-encoded "<Company Name> <role keyword>">&origin=GLOBAL_SEARCH_HEADER
```
Use a short keyword drawn from the posting's title for `<role keyword>` - e.g. a posting
titled "AI Program Manager" becomes `"<Company Name> AI Program Manager"`.

Both links are for the user to open and browse themselves - never fetch or scrape the
LinkedIn people-search result pages programmatically. Never fabricate contacts or claim a
specific person was found; these are search links, not results.

### Step 4.75: Portal Health Check

Scraper-based portal CLIs rot silently: when a portal changes its markup, the parser usually exits 0 with zero results or with null/garbled fields, and the Step 1c fallback never fires because it only triggers on hard failure. This step catches that from evidence the run already holds.

**Free pass (no extra requests).** For each enabled portal that ran in Step 1b:

- **Degraded scan:** inspect the results it returned this run. Flags: `company` null or empty on every result, empty titles, undecoded entities (`&amp;`) or HTML fragments in titles, URLs that do not point at the portal. Any of these means the parser is half-working and `/scrape` is silently collecting junk.
- **Yield history:** if the portal returned zero results across all of this run's queries, check whether `seen_jobs.json` holds prior entries from it (via the `portal` field, or by matching URL domains for entries predating the field). A portal that produced jobs on earlier runs and produces nothing now is suspect - the same queries worked before.

**Escalation (bounded, on suspicion only).** A suspect portal gets **one** sentinel probe: run its documented `search` with the example query from its own SKILL.md (that query provably worked when the skill was registered), the portal's limit flag capped at 3, `--format json`. If that returns nothing, retry **once** with a single common word. Only then is the verdict **broken**. A 429 or block page is **never** evidence of breakage - record the portal as **inconclusive (rate-limited)**, back off, and do not retry.

**Verdicts.** Healthy portals get silence - no table, no line. Anything else surfaces in the Step 5 summary as a health line.

**Probe-only mode (`/scrape health`).** Skip Steps 1-4 and this step's free pass (there is no fresh run to scan); instead probe every installed portal directly - enabled ones by default, a disabled one only when named explicitly (e.g. `/scrape health jobnet`). Each portal gets the sentinel probe above, the degraded criteria applied to whatever it returns, and - since the user explicitly asked for diagnosis - one `detail` fetch on the first result of each healthy portal (description must be readable decoded text; a failure downgrades to degraded). Report all statuses in this mode, including healthy. Volume stays bounded: one search, at most one retry, at most one detail per portal.

### Step 5: Present Results

Present new jobs in a table sorted by fit (high first). If a watchlist company's career page could not be fetched this run, note it below the table so a silently rotting source stays visible. When Step 1b skipped
portals (`enabled: false`), report them with the `skipped (disabled):` line below
so opting one out stays visible rather than silent; omit the line when nothing
was skipped. When any portal's results came from the Step 1c fallback this run
(bun unavailable, or its CLI failed at runtime), report it with the
`fallback (websearch):` line - fallback results come from a search index that
can be stale, so the reader should know which rows carry that caveat; omit the
line when every portal ran its CLI. When Step 4.75 found a portal degraded, broken, or inconclusive,
add one `health:` line per suspect portal (healthy portals get no line); after
the report, offer to set that portal's `enabled: false` so `/scrape` stops
running it (and covers it via the Step 1c fallback) until it is fixed - only
edit the toggle with the user's confirmation, and never edit anything else in
the skill.

```
## New Job Matches - YYYY-MM-DD

Found X new positions (Y high, Z medium, W low match).

skipped (disabled): <portal-name>, <portal-name>

fallback (websearch): <portal-name>, <portal-name>

health: <portal-name> - degraded (company null on all 12 results); parsing anchors in .agents/skills/<portal-name>/url-reference.md
health: <portal-name> - broken (0 results for the SKILL.md test query and a broader retry); parsing anchors in .agents/skills/<portal-name>/url-reference.md

| # | Fit | Title | Company | Location | Deadline | URL |
|---|-----|-------|---------|----------|----------|-----|
| 1 | High | ... | ... | ... | ... | [Link](...) |

If Step 2.5 flagged a mass-posting pattern, note it in the Title cell (e.g. "Frontend Developer (posted in 6 cities)") rather than burying it. Do the same for a declared-language-insufficient-level flag from the Language Gate (e.g. "Backend Engineer ⚠ fluent English required") - both are signals the user should see at a glance, not just in the detail highlights below.

### High-Match Highlights
For each high-match job, add 2-3 bullet points:
- Why it matches your profile
- Key requirements to check
- Any red flags (including mass-posting signals from Step 2.5)

### Contacts
For each high/medium-fit job from Step 4.5, add a short contacts block with the two
LinkedIn search links:
- Recruiters/TA search link, for the referral path
- Role/team-peer search link, for the warm-intro / informational-outreach path
```

After presenting, ask:
> "Want me to evaluate any of these in detail? Just give me the number(s)."

If the user picks a number, invoke the **job-application-assistant** skill workflow (fit evaluation first, then CV + cover letter if approved).

### Step 5b: Write the markdown report (ALWAYS — do not skip)

The chat presentation in Step 5 is ephemeral. Every run must also persist a ranked report file so the
user has a durable, openable artifact. Use the **Write** tool (this skill has no Bash) to write:

```
job_scraper/job_matches_<YYYY-MM-DD>.md
```

Overwrite the same-day file if it already exists (a re-run supersedes the earlier snapshot). Build the
report from the **full `seen_jobs.json`** (the source of truth), not just this run's new hits — so the
file is always the complete current ranking, ordered by fit (high → medium → low) then company. Structure:

```markdown
# Job Matches — <market> watchlist + market scrape

_Generated <YYYY-MM-DD> · ranked by genuine fit × growth × earning · <active exclusions, e.g. German-language roles excluded>._

**<N> tracked** — <H> high · <M> medium · <L> low

## 🟢 High fit (<H>)
| Fit | Role | Company | Location | Source | Link |
|-----|------|---------|----------|--------|------|
| high | … | … | … | … | [link](…) |

## 🟡 Medium fit (<M>)
| … (same columns) |

## ⚪ Low fit (<L>)
_IC / research / intern / sales / language-excluded — recorded for dedup, weak fit._
- **<Company>** — <Title> (<Location>)<⚠️ flag if language-excluded> · [link](…)
```

Render any `language_flag` / exclusion note as a `⚠️` marker so excluded roles are visibly tagged. After
writing, tell the user the path. (Note: `job_scraper/*.md` is gitignored by default — it's a local
artifact; only force-add it if the user wants it committed.)

If the run found many new jobs (roughly 8+), also suggest `/rank` - it batch-scores all new postings against the full fit framework and returns a ranked shortlist, which beats eyeballing a long table. (`/rank` sets the `ranked` and `expired` status values in `seen_jobs.json`; treat both as already-seen for dedup purposes, alongside this fork's `drafted`, `applied`, and `discarded` statuses. When updating `seen_jobs.json`, always preserve fields you don't recognize — `/rank` and this skill share the file.)

### Step 6: Update Tracker (Optional)

If the user decides to apply to any job, the tracker row is written by **job-application-assistant Step 3b**, which Step 5 already routes into - do not add a second row here. Only when the user says they applied to something outside that path, add a row using the header and the match-then-update rule in `/outcome` Step 1.

---

## Watchlist mode (`/scrape watchlist`)

When invoked with `watchlist`, sweep the curated employers in `watchlist.md` (this directory)
directly, instead of running the keyword searches in `search-queries.md`. This monitors specific
target companies' career pages for new postings. Steps 0, 3, 4, 5, 5b, 6 (state, fit, dedup, present,
write report, tracker) are unchanged — only the search/fetch changes:

1. Read `watchlist.md`. Note the **target location filter** (the Zurich-area town list) at the top.
2. Process each company according to its **tier**:
   - **Tier 1 (Public JSON API):** `WebFetch` (or `tavily_extract`) the company's `Fetch URL`. It returns JSON. Parse the postings, then keep only those whose location field matches the location filter. **Match both `Zurich` and `Zürich` explicitly** — a substring test for `urich` silently misses the umlaut spelling (`ü` breaks the `u-r-i-c-h` run), which several boards use. Lowercase and test for `zurich` OR `zürich` plus the suburb names. Extract title, location, URL, and posting date from the JSON fields named in `watchlist.md`. This is the most reliable tier — prefer it.
   - **Tier 2 (Workday):** do NOT try to `WebFetch` the board URL (it's a JS shell, and the real API needs a POST). Instead run `WebSearch` scoped to the careers domain, e.g. `site:roche.wd3.myworkdayjobs.com Zürich <your role keywords>`. Fetch any promising individual job URLs that are static.
   - **Tier 3 (SPA / other):** run `WebSearch` like `<company> careers Zurich <your role keywords>` (and use the Careers URL for context). For the two rows marked ✅ static HTML (IBM Research, Disney Research), `WebFetch`/`tavily_extract` the careers URL directly.
3. For every candidate posting, apply the **location filter** strictly — discard anything not in the Zurich area. Many of these boards are global, so most postings will not match; that is expected.
4. Deduplicate against `seen_jobs.json` and `job_search_tracker.csv` exactly as in Step 4, then run the Step 3 fit check and present via Step 5. In the results table, add the **company** as the source so it's clear which watchlist employer each hit came from.
5. Be efficient: a watchlist sweep can touch 20+ companies. Run the Tier-1 fetches in parallel (Agent tool or parallel calls), and for Tier 2/3 prioritize the companies most aligned with the user's focus area if one was given (e.g. "/scrape watchlist data science").

> If a Tier-1 `Fetch URL` returns an error or empty list, fall back to that company's careers
> page (Tier 3 row, or the "conflicting signals" note) and flag it so the user can fix the slug
> in `watchlist.md`. Never silently drop a company — report which ones failed to fetch.

---

## Important Rules

1. **Never fabricate job postings.** Only present jobs found via actual Tavily/WebSearch/WebFetch results.
2. **Respect deduplication.** Always check seen_jobs.json AND job_search_tracker.csv before presenting. Treat `ranked`, `expired`, `drafted`, `applied`, and `discarded` statuses as already-seen.
3. **Focus on configured geographic area.** Skip jobs that require relocation or are clearly outside commute range.
4. **Only open positions.** Skip postings with expired deadlines or those marked as closed.
5. **Be efficient with fetching.** Don't run `detail`, `tavily_extract` or `WebFetch` on every search hit — pre-filter by title and snippet, then fetch only promising matches. Tavily credits are limited (free tier ~1,000/month), so lean on search snippets.
6. **Parallel searches.** Run portal CLI searches in parallel; use the Step 1c Tavily/WebSearch pass for the sources the CLIs don't cover.
7. **Always write the report file** (Step 5b) — every run produces `job_scraper/job_matches_<YYYY-MM-DD>.md` from the full `seen_jobs.json`, even if there are no new hits. The chat table is not a substitute; the user needs the file.
8. **No automated people lookups.** Referral contacts (Step 4.5) are LinkedIn search links only - never fetch or scrape LinkedIn people-search result pages programmatically.
9. **Health checks are bounded and honest.** Step 4.75 spends at most one probe, one retry, and (in `health` mode) one detail fetch per portal - a diagnosis, not a crawl. A rate-limit is never evidence of breakage. Health verdicts come only from observed CLI output; a portal that could not be tested is reported as inconclusive, never guessed. The `enabled` toggle is the only thing the health check may edit, and only with confirmation.
10. **Flag distribution patterns, never accuse.** The mass-posting signal (Step 2.5) describes how a listing is being distributed, not a claim that the employer is a scam. Never name a company as fraudulent or untrustworthy - present the observation and let the user decide.
