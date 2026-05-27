# Mode: fashion-board-ingest — Fashion Workplace Job Ingestion

Scan for new fashion industry jobs and push them into the Fashion Workplace staging queue via MCP.
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

Confirmed working sources (from scan history):
- `site:fashionunited.nl/modevacatures` — returns individual job listing URLs
- `site:fashionunited.de/jobs-in-der-mode` — returns individual job listing URLs
- `site:linkedin.com/jobs` — returns individual job view URLs when role/location specific
- `site:stepstone.de` — returns individual job listing URLs
- `site:fashionsolution.nl` — returns individual job listing URLs
- `site:jobs.lever.co` — returns individual job listing URLs
- `site:job-boards.greenhouse.io` — returns individual job listing URLs

## Phase 2 — Ingest to Fashion Workplace staging queue

Process unchecked URLs from `data/pipeline.md` in batches of up to 50.
Cap per run: **50 URLs** — remaining stay in pipeline.md for the next run.
**Per-company cap: max 8 jobs per company per run.** When iterating through pipeline.md, track how many jobs have been queued per `companyName`. Once a company hits 8, skip further entries from that company for this run (leave them as `- [ ]` for the next run). This prevents any single company from flooding the review queue.

### Step 2a — Extract structured data

For each URL, call `mcp__fashionworkplace__prefill_from_url`.

If prefill fails or returns null title: use **scan-metadata fallback** — submit with only the fields available from `pipeline.md` (title, companyName, sourceUrl, country inferred from URL domain). Log as `ingested_fallback` in scan-history.tsv.

**Source reliability (confirmed from testing):**
| Source | prefill result | strategy |
|---|---|---|
| `fashionunited.nl/modevacatures` | ✅ full data (occasional 500 error) | prefill; fallback if 500 |
| `fashionunited.de/jobs-in-der-mode` | ✅ full data | prefill |
| `nl.linkedin.com/jobs/view/` | ✅ full data | prefill |
| `job-boards.greenhouse.io` | ✅ full data + salary | prefill |
| `jobs.ashbyhq.com` | ✅ usually works | prefill |
| `fashionsolution.nl` | ✅ full data | prefill |
| `stepstone.de` | ❌ always times out | scan-metadata fallback only |
| `jobs.lever.co` | ❌ returns empty `{job:{}}` | scan-metadata fallback only |

### Step 2b — Map fields to create_scraped_jobs schema

Prefill field paths use dot notation (`data.job.title`, etc.).

| prefill field | create_scraped_jobs field | notes |
|---|---|---|
| `data.job.title` | title | required |
| the URL passed to prefill | sourceUrl | required |
| — | source | always `"career-ops"` |
| `data.organization.name` | companyName | |
| `data.organization.url` | companyUrl | skip if it resolves to a LinkedIn page |
| `data.city` | city | |
| `data.country` (ISO 2) | country | convert to ISO alpha-3 (NL→NLD, GB→GBR, US→USA, DE→DEU, FR→FRA, BE→BEL) |
| `data.locationType` | locationType | remote / hybrid / onsite |
| `data.job.type` | jobType | full_time / part_time / contract / freelance |
| `data.job.seniority` | seniority | array — values: entry-level, mid-level, senior, manager |
| `data.salary.min` | salaryMin | integer |
| `data.salary.max` | salaryMax | integer |
| `data.salary.currency` | salaryCurrency | ISO 4217 (USD, EUR, GBP) |
| `data.job.appLinkOrEmail` | applicationUrl | only if value starts with `http`; if it contains `@` use `applicationEmail` instead; if it's a LinkedIn login redirect, omit |
| `data.job.appLinkOrEmail` | applicationEmail | only if value contains `@` (email address) |

**Note on description:** `prefill_from_url` returns `data.job.description` as a Tiptap/ProseMirror JSON document, not HTML. Do not pass it to `descriptionHtml`. The admin can view the full posting via `sourceUrl`.

### Step 2c — Push to staging queue

Call `mcp__fashionworkplace__create_scraped_jobs` with the batch (max 50 jobs).
The tool deduplicates by `sourceUrl` — safe to re-submit previously seen URLs.

### Step 2d — Mark pipeline.md entries

For each successfully submitted URL, mark the pipeline.md checkbox: `- [x] {url} | ...`

### Step 2e — Log to scan-history.tsv

For each submitted job, append a row with status `ingested`.
For each skipped job, append with status `skipped_prefill_error`.

## ISO 3166-1 alpha-3 conversion reference

| alpha-2 | alpha-3 | Country |
|---|---|---|
| NL | NLD | Netherlands |
| GB | GBR | United Kingdom |
| US | USA | United States |
| DE | DEU | Germany |
| FR | FRA | France |
| BE | BEL | Belgium |
| IT | ITA | Italy |
| ES | ESP | Spain |
| CA | CAN | Canada |
| AU | AUS | Australia |
| DK | DNK | Denmark |
| SE | SWE | Sweden |
| NO | NOR | Norway |
| PT | PRT | Portugal |
| HK | HKG | Hong Kong |
| IN | IND | India |
| CN | CHN | China |
| MX | MEX | Mexico |
| TR | TUR | Turkey |

If country is unknown or not in this list: omit the field.

### Step 2f — Commit and push state

After every run (whether any jobs were ingested or not), commit the updated files so the next run has accurate dedup history:

```bash
git add data/pipeline.md data/scan-history.tsv
git commit -m "ingest: {YYYY-MM-DD} — {N} jobs pushed, {N} pending"
git push
```

This is required for cloud-scheduled runs — each run starts from a fresh git clone, so state that is not committed is lost.

## Run summary output

After each run, print:

```
Fashion Workplace Ingest — {YYYY-MM-DD}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
API scan:     {N} new URLs (tracked_companies)
Web queries:  {N} new URLs (search_queries)
Total new:    {N} URLs discovered

Ingested:     {N} jobs pushed to review queue
Skipped:      {N} (prefill failed / no title)
Duplicates:   {N} (already seen)

Pending in pipeline.md: {N} remaining for next run
Review queue: https://fashionworkplace.com/admin/scraped-jobs
```

## Scheduled run behaviour

- Run both phases fully without prompting
- Never call `approve_scraped_job` or `create_job_from_prefill` — staging only
- If `node scan.mjs` fails, continue with WebSearch queries
- If a batch `create_scraped_jobs` call fails, retry once; if it fails again, log and continue with next batch
- Always write the run summary at the end
