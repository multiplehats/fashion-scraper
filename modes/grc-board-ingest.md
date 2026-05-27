# Mode: grc-board-ingest — GRC Jobs Board Ingestion

Scan for new GRC & Legal jobs (NL-focused) and push them into the GRC Jobs staging queue via MCP.
Jobs land in the review queue and are NOT published until an admin approves them.

## Overview

Two phases, fully automated — no human input required mid-run:

1. **Scan** — zero-token API scan + WebSearch queries discover new job URLs
2. **Ingest** — URLs are extracted via `prefill_from_url`, batched, and pushed to the staging queue via `create_scraped_jobs`

## Phase 1 — Scan

### Step 1a — Zero-token API scan (Greenhouse / Ashby / Lever)

```bash
node scan.mjs
```

Hits all `tracked_companies` in `portals.yml` directly via their ATS APIs. Writes new URLs to `data/pipeline.md`. Deduplicates against `data/scan-history.tsv`.

### Step 1b — WebSearch queries (Level 3)

Run each `search_query` with `enabled: true` from `portals.yml` using WebSearch.
For each result URL:
- Apply title filter from `portals.yml` (positive must match, negative must not)
- Deduplicate against `data/scan-history.tsv` and `data/pipeline.md`
- Append new URLs to `data/pipeline.md` as `- [ ] {url} | {company} | {title}`
- Record in `data/scan-history.tsv`

## Phase 2 — Ingest to GRC Jobs staging queue

Process unchecked URLs from `data/pipeline.md` in batches of up to 50.
Cap per run: **50 URLs** — remaining stay in pipeline.md for the next run.
**Per-company cap: max 8 jobs per company per run.** Track per-company count; once a company hits 8, skip further entries for that company this run.

### Step 2a — Extract structured data

For each URL, call `mcp__grcjobs__prefill_from_url`.

If prefill fails or returns null title: use **scan-metadata fallback** — submit with only the fields available from `pipeline.md` (title, companyName, sourceUrl, country inferred from URL domain). Log as `ingested_fallback` in scan-history.tsv.

**Source reliability guide:**
| Source | prefill result | strategy |
|---|---|---|
| `job-boards.greenhouse.io` | ✅ full data + salary | prefill |
| `jobs.ashbyhq.com` | ✅ usually works | prefill |
| `nl.linkedin.com/jobs/view/` | ✅ full data | prefill |
| `nationalevacaturebank.nl` | ✅ full data | prefill |
| `werkenbijgemeenten.nl` | ✅ full data | prefill |
| `jobs.lever.co` | ❌ returns empty `{job:{}}` | scan-metadata fallback only |
| `werken.nl` | test on first run | prefill; fallback if error |

### Step 2b — Map fields to create_scraped_jobs schema

| prefill field | create_scraped_jobs field | notes |
|---|---|---|
| `data.job.title` | title | required |
| the URL passed to prefill | sourceUrl | required |
| — | source | always `"grc-scraper"` |
| `data.organization.name` | companyName | |
| `data.organization.url` | companyUrl | skip if resolves to LinkedIn |
| `data.city` | city | |
| `data.country` (ISO 2) | country | convert to ISO alpha-3 (NL→NLD, BE→BEL, DE→DEU, GB→GBR) |
| `data.locationType` | locationType | remote / hybrid / onsite |
| `data.job.type` | jobType | full_time / part_time / contract / freelance |
| `data.job.seniority` | seniority | array — values: entry-level, mid-level, senior, manager |
| `data.salary.min` | salaryMin | integer |
| `data.salary.max` | salaryMax | integer |
| `data.salary.currency` | salaryCurrency | ISO 4217 (EUR, GBP, USD) |
| `data.job.appLinkOrEmail` | applicationUrl | only if starts with `http`; skip LinkedIn login redirects |
| `data.job.appLinkOrEmail` | applicationEmail | only if contains `@` |

**Note:** Do not pass `data.job.description` (Tiptap JSON) to `descriptionHtml`. Admin views full posting via `sourceUrl`.

### Step 2c — Push to staging queue

Call `mcp__grcjobs__create_scraped_jobs` with the batch (max 50 jobs).
The tool deduplicates by `sourceUrl` — safe to re-submit previously seen URLs.

### Step 2d — Mark pipeline.md entries

For each successfully submitted URL: `- [x] {url} | ...`

### Step 2e — Log to scan-history.tsv

Append a row per job: `ingested` for success, `skipped_prefill_error` for failures.

### Step 2f — Commit and push state

```bash
git add data/pipeline.md data/scan-history.tsv
git commit -m "ingest: {YYYY-MM-DD} — {N} jobs pushed, {N} pending"
git push
```

Required for cloud-scheduled runs — each run starts from a fresh clone.

## ISO 3166-1 alpha-3 conversion reference

| alpha-2 | alpha-3 | Country |
|---|---|---|
| NL | NLD | Netherlands |
| BE | BEL | Belgium |
| DE | DEU | Germany |
| GB | GBR | United Kingdom |
| FR | FRA | France |
| LU | LUX | Luxembourg |
| CH | CHE | Switzerland |
| US | USA | United States |

If country unknown: omit the field.

## Run summary output

```
GRC Jobs Ingest — {YYYY-MM-DD}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
API scan:     {N} new URLs (tracked_companies)
Web queries:  {N} new URLs (search_queries)
Total new:    {N} URLs discovered

Ingested:     {N} jobs pushed to review queue
Skipped:      {N} (prefill failed / no title / wrong role)
Duplicates:   {N} (already seen)

Pending in pipeline.md: {N} remaining for next run
Review queue: https://[grc-jobs-domain]/admin/scraped-jobs
```

## Scheduled run behaviour

- Run both phases fully without prompting
- Never auto-publish — staging only
- If `node scan.mjs` fails, continue with WebSearch queries
- If a batch call fails, retry once; if it fails again, log and continue
- Always write the run summary at the end
