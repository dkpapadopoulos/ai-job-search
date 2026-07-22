---
name: scrape
description: >
  Searches job sites for new positions matching your profile, using Tavily search
  (when configured) or built-in WebSearch, plus a curated company watchlist mode.
  Deduplicates across runs. Triggers on: job scrape, find jobs, search jobs, new jobs,
  job search, scrape jobs, /scrape
allowed-tools: Read, Write, Edit, Glob, Grep, WebFetch, WebSearch, Agent, AskUserQuestion, mcp__tavily-mcp__tavily_search, mcp__tavily-mcp__tavily_extract, mcp__tavily-mcp__tavily_research
---

# Job Scraper

---

## How It Works

This skill searches job sites using targeted queries based on your profile, deduplicates against previously seen jobs and the application tracker, and presents new matches with a quick fit assessment.

**Search backend (augment-with-fallback):** if the **Tavily MCP tools** (`mcp__tavily-mcp__tavily_search`, `mcp__tavily-mcp__tavily_extract`) are present in your current tool list, prefer them — they give better international results and cleaner page extraction. If they are **not** available (no `TAVILY_API_KEY`, MCP server failed to start, or Tavily not installed), fall back to the built-in **WebSearch** / **WebFetch** tools. The skill works fully on the built-ins alone; Tavily is an optional quality upgrade. See `../../../.mcp.json` and the README's Tavily section for setup.

> **How to tell which backend is active:** check whether `mcp__tavily-mcp__tavily_search` appears in the tools available to you this session. If yes → use Tavily. If no → use WebSearch/WebFetch. Do not assume; a present `.mcp.json` does not guarantee the tool loaded (a missing/invalid key makes the server fail silently).

It is **country-agnostic**: the sites and query terms come from `search-queries.md`, which you localize for your market via `/setup --section search`. Universal sources (LinkedIn Jobs, Indeed, Google Jobs) work in any country with no setup; local boards are added per market.

> **Note on portal CLIs:** upstream ships portal-search CLIs under `.agents/skills/` (LinkedIn, freehire, Danish boards). This fork does **not** use them as the default backend — WebSearch/Tavily covers the Zurich market via `site:` queries. They remain installed for explicit, manual opt-in use only (mind the LinkedIn CLI's keep-volume-low ToS caveat in its SKILL.md).

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

For each search:
- **If `mcp__tavily-mcp__tavily_search` is available:** use it. Pass your localized query. To steer results to your market, set Tavily's geographic/topic parameters where the tool supports them (e.g. a `country` parameter set to your market, and a news/general `topic`) — only use parameter names/values the tool's schema actually accepts; do not invent enum values. Otherwise just embed the city/country in the query string as usual.
- **Otherwise:** use the built-in `WebSearch` with the same queries.
- Either way, draw the queries and sites from `search-queries.md` (universal sources like linkedin.com/jobs and indeed.com work anywhere; local boards are added there per market).
- Target your configured geographic area.
- Look for postings from the last 14 days.

### Step 2: Fetch & Parse

For each promising result from Step 1:
- Retrieve the job posting page: use `mcp__tavily-mcp__tavily_extract` if it is available (cleaner content), otherwise `WebFetch`.
- Extract: **job title**, **company**, **location**, **posting date** (or "recent"), **URL**, **key requirements** (brief), **application deadline** (if listed)
- Skip if the URL or company+title combo already exists in `seen_jobs.json`
- Skip if the company+role already appears in `job_search_tracker.csv`

> **Fetchability note:** Individual **LinkedIn** job pages (`linkedin.com/jobs/view/...`) are **login-gated** — both `WebFetch` and `tavily_extract` return only a teaser, not the full description (it is an auth wall, not a rendering problem, so Tavily does not bypass it). Use LinkedIn for *discovery* (titles, companies, locations, deep links from Step 1), but for the full posting prefer **fetchable sources**: Indeed, the company's own career page, or local boards. If only a gated URL is available and the user wants to apply, the title/company/location from the search result is enough to present the match; the full description can be pulled later by pasting the posting text into `/apply` (which supports pasted text when a URL is blocked).

### Step 3: Quick Fit Assessment

For each new job, do a rapid fit check (NOT the full evaluation from `04-job-evaluation.md` - just a quick signal):

- **High match**: Role directly involves your core skills
- **Medium match**: Role is adjacent to your experience
- **Low match**: Role requires significant skills you lack

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
      "fit": "high/medium/low",
      "status": "new/skipped/evaluated/ranked/expired/drafted/applied/discarded",
      "portal": "<source site or board, e.g. linkedin, jobs.ch, watchlist:<company>>"
    }
  }
}
```

The `portal` field records which source site or board the result came from (search backend, `site:` domain, or watchlist company). Entries written before this field existed lack it; do not backfill.

`/rank` extends this schema additively: ranked entries also carry `rank_score` (0–100 overall score), `rank_verdict` (fit band, e.g. "strong fit"), and `rank_date` (ISO date of ranking). The `status` field is set to `"ranked"`. Do not drop any of these fields when re-writing entries.

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

### Step 5: Present Results

Present new jobs in a table sorted by fit (high first). If a watchlist company's
career page could not be fetched this run, note it below the table so a silently
rotting source stays visible.

```
## New Job Matches - YYYY-MM-DD

Found X new positions (Y high, Z medium, W low match).

| # | Fit | Title | Company | Location | Deadline | URL |
|---|-----|-------|---------|----------|----------|-----|
| 1 | High | ... | ... | ... | ... | [Link](...) |

### High-Match Highlights
For each high-match job, add 2-3 bullet points:
- Why it matches your profile
- Key requirements to check
- Any red flags

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

If the user decides to apply to any job, add a row to `job_search_tracker.csv`.

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
5. **Be efficient with fetching.** Don't fetch every search result - use titles and snippets to pre-filter before calling tavily_extract/WebFetch. Tavily credits are limited (free tier ~1,000/month), so prefer search snippets and fetch only promising results.
6. **Parallel searches.** Use the Agent tool or parallel search calls to speed up the search phase.
7. **Always write the report file** (Step 5b) — every run produces `job_scraper/job_matches_<YYYY-MM-DD>.md` from the full `seen_jobs.json`, even if there are no new hits. The chat table is not a substitute; the user needs the file.
8. **No automated people lookups.** Referral contacts are LinkedIn search links only - never fetch or scrape LinkedIn people-search result pages programmatically.
