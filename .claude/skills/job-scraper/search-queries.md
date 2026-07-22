# Search Queries for Job Scraper

<!-- Zurich, Switzerland edition. Personalize with `/setup --section search`:
     replace the [YOUR_...] placeholders with your own target roles and skills. -->

## Installed portal CLIs (manual opt-in only)

Upstream ships portal-search CLIs under `.agents/skills/` (`linkedin-search`, `freehire-search`, Danish boards). **This fork's `/scrape` does not run them** — WebSearch/Tavily with the `site:` queries below is the backend (see the "Note on portal CLIs" in `SKILL.md`). Use a CLI only when explicitly asked.

## Search Sites

The scraper uses built-in **WebSearch** (or Tavily) with `site:` filters. Sources:

Universal (work everywhere):
- **linkedin.com/jobs** — filter by Zurich / Switzerland in the query (discovery only; pages are login-gated)
- **indeed.com** / **ch.indeed.com** — global + Swiss aggregator
- **Google Jobs** — via a plain `[role] jobs Zurich` search
- Company career pages — see `watchlist.md` for the curated Zurich employer list (preferred for direct monitoring via `/scrape watchlist`)

Swiss boards (Zurich market):
- **jobs.ch** — largest Swiss general job board
- **jobup.ch** — French-speaking Switzerland sister site (skip unless targeting Romandie)
- **jobscout24.ch** — Swiss general board

## Query Templates

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
Flag rather than silently drop: mark language-excluded roles with a `language_flag` in
`seen_jobs.json` so they stay visible in dedup.

## Recency

Only surface postings from the **last 14 days** (stale postings are usually filled or
closed). Mark older `seen_jobs.json` entries `expired` rather than deleting them.
