# Search Queries for Job Scraper

<!-- Zurich, Switzerland edition. Personalize with `/setup --section search`:
     replace the [YOUR_...] placeholders with your own target roles and skills. -->

## Installed portal CLIs (primary for `/scrape`)

Portal-search CLIs live under `.agents/skills/`. `/scrape` Step 1b runs every **enabled** one
first, then covers the rest of this market through the WebSearch/Tavily `site:` queries below.
For this fork that means `linkedin-search` and `freehire-search` run as CLIs; the Danish demo
boards ship `enabled: false` and stay off unless you enable them. Swiss boards (jobs.ch,
jobscout24.ch) have no CLI, so they are searched through the queries below — the same path the
watchlist uses.

**Language scope:** write every query category in every language listed in your CLAUDE.md Languages table (typically 1-2, sometimes more). A posting requiring a language you have *not* declared, as a job condition, is excluded before scoring; a posting requiring a *higher level* than you declared in a language you *do* work in is flagged for your own judgment, not excluded — see `04-job-evaluation.md`'s Language Gate, the single source of truth for this rule. Translate each category's keywords rather than machine-translating word-for-word (e.g. "Frontend Developer" -> "Desarrollador Frontend", not a literal word-for-word translation) if you work in more than one language.

## Search Sites

Everything a portal CLI does not cover is searched with **Tavily** (when configured) or built-in **WebSearch**, using `site:` filters. Sources:

Universal (work everywhere):
- **linkedin.com/jobs** — covered by the `linkedin-search` CLI in Step 1b; the `site:` query below is the fallback when that CLI is unavailable or fails (discovery only either way; posting pages are login-gated)
- **indeed.com** / **ch.indeed.com** — global + Swiss aggregator
- **Google Jobs** — via a plain `[role] jobs Zurich` search
- Company career pages — see `watchlist.md` for the curated Zurich employer list (preferred for direct monitoring via `/scrape watchlist`)

Swiss boards (Zurich market):
- **jobs.ch** — largest Swiss general job board
- **jobup.ch** — French-speaking Switzerland sister site (skip unless targeting Romandie)
- **jobscout24.ch** — Swiss general board

## Query Templates

**Organize by function, not job title.** The same underlying work carries different titles across companies and markets (a "Data Scientist" role at one employer may be posted as "Insights Analyst" or "Data Consultant" at another). Name each priority category after the function it covers, and list several plausible job titles as query variants within that category rather than betting an entire priority tier on one exact title string.

Queries are grouped by priority and combined with Zurich/Switzerland location terms.
Rank your own categories by **fit × growth × earning potential** via `/setup --section search`.

### Priority 1: [YOUR_PRIMARY_ROLE_FAMILY]

```
site:linkedin.com/jobs "[YOUR_PRIMARY_JOB_TITLE]" Zurich Switzerland
site:linkedin.com/jobs ("[YOUR_PRIMARY_JOB_TITLE]" OR "[YOUR_ADJACENT_TITLE]") Zurich
"[YOUR_PRIMARY_JOB_TITLE]" jobs Zurich OR remote Switzerland
site:jobs.ch "[YOUR_PRIMARY_JOB_TITLE]" Zürich
```

### Priority 2: [YOUR_SECONDARY_ROLE_FAMILY]

```
site:linkedin.com/jobs ("[YOUR_DOMAIN_KEYWORD_1]" OR "[YOUR_DOMAIN_KEYWORD_2]") Zurich
"[YOUR_SECONDARY_JOB_TITLE]" jobs Zurich Switzerland
site:jobs.ch ("[YOUR_DOMAIN_KEYWORD_1]") Zürich
```

### Priority 3: [YOUR_TERTIARY_ROLE_FAMILY]

```
site:linkedin.com/jobs "[YOUR_TERTIARY_JOB_TITLE]" Zurich
"[YOUR_TERTIARY_JOB_TITLE]" jobs Zurich OR remote Switzerland
```

### Priority 4: Broader [YOUR_BROADER_FIELD]

Wider net, run by `/scrape broad` rather than a default run.

```
site:linkedin.com/jobs "[YOUR_BROADER_JOB_TITLE]" Zurich Switzerland
site:jobs.ch "[YOUR_BROADER_JOB_TITLE]" Zürich
"[YOUR_BROADER_JOB_TITLE]" OR "[YOUR_BROADER_KEYWORD]" jobs Zurich
```

<!-- Worked example (a program-management-flavored Zurich search):
site:linkedin.com/jobs "Program Manager" Zurich Switzerland
site:linkedin.com/jobs ("Technical Program Manager" OR "Engineering Program Manager") Zurich
site:jobs.ch "Program Manager" Zürich
"Head of Operations" OR "Operational Excellence" jobs Zurich
-->

## Language Filter

Zurich postings are frequently written in German. If your working language is English,
consider excluding German-only postings at evaluation time (set this in your profile's
language preferences — e.g. "skip postings that require German/Swiss-German fluency").
Flag rather than silently drop: record the outcome in `seen_jobs.json` with the canonical
fields `language_gate` (PASS/FAIL/FLAG) and `language_note` (the quoted requirement), as
documented in Step 4's schema, so excluded roles stay visible in dedup.

## Recency

Only surface postings from the **last 14 days**, or with an application deadline that has
not yet passed (stale postings are usually filled or closed). If a posting date cannot be
determined, include it but flag as "date unknown".

Age alone is **not** `expired`. That status has a precise meaning downstream - the posting
was confirmed closed at source, or its deadline has passed (see `/rank`'s rule, and Step 2's
closed-at-source detection). Since `SKILL.md` Rule 2 treats `expired` as already-seen, marking
a merely-old but still-open posting `expired` would suppress it from every future run. Leave
such entries in `seen_jobs.json` with the status they already carry, and never delete them.

## Location Filter

When evaluating results, verify the job location is within reasonable commute distance from your home. Define acceptable areas:
- Zurich city and surrounding areas
- [ACCEPTABLE_AREA_1]
- [ACCEPTABLE_AREA_2]
- [BORDERLINE_AREA] (borderline - ~X min by transit)
- [TOO_FAR_AREA] (too far)

## Adapting Queries

If the user specifies a focus area, select queries from the matching category and also generate 2-3 custom queries for that focus. For example:
- "/scrape [focus_area]" -> relevant category queries + custom focus-specific queries
