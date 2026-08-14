# Instructions

## Project Purpose

This scraper extracts job listings from Metro Digital Romania Careers (Attrax/HTML) and imports them to peviitor.ro.

Target: https://careers.metro.digital/jobs

## Model Schemas

The job and company models are defined in:
- `job-model.md` - Job model schema
- `company-model.md` - Company model schema

## Important

These models are **dynamic** and can change over time. They are based on the official Peviitor Core schemas which may be updated.

## How to Keep Models Updated

When working on this scraper:

1. **Check for updates** in the Peviitor Core repository:
   - Repository: https://github.com/peviitor-ro/peviitor_core
   - Main file: README.md (contains Job and Company model schemas)

2. **When to update**:
   - Before starting new development work
   - If field requirements or validations have changed
   - If new fields have been added

3. **How to update**:
   - Fetch the latest README.md from peviitor_core main branch
   - Compare with current job-model.md and company-model.md
   - Update local files if there are differences
   - Update `scraper/index.js` mapping logic if field requirements changed

## Technologies

- **Node.js & JavaScript** - For scraping and data extraction
- **Peviitor API** - For data storage and indexing (no direct SOLR access)
- **ANAF + CUIScan + CUIFirma** - For company validation

## Workflow Steps

1. **Start with brand** - We know the brand ("METRO DIGITAL")
2. **Search in DemoANAF** - Find company by brand, get CIF from search results
3. **Get company details from ANAF** - Using CIF, fetch full company data from ANAF
4. **Validate with Peviitor** - Verify company exists in Peviitor, get group/brand info
5. **Check existing jobs in API** - Query by CIF via the Peviitor API to see what jobs already exist
6. **Check company status** - If ANAF status = "inactive" → DELETE existing jobs from API and STOP
7. **Save company.json** - Save all ANAF + Peviitor data for backup
8. **Scrape new jobs** - Extract jobs from Metro Digital Careers (HTML, `.attrax-vacancy-tile`)
9. **Transform for SOLR** - Validate and fix job data:
   - location: Only Romanian cities allowed
   - tags: lowercase, no diacritics
   - company: uppercase
10. **Upsert to API** - Import/update jobs via the Peviitor API
11. **Verify URLs** - Check existing job URLs still work, delete 404s

## Running the Scraper

```bash
# Run the full scraper workflow (single command)
node scraper/index.js

# Test mode (one page only)
node scraper/index.js --test
```

> **Important**: Scraper does NOT delete jobs from other sources (ANOFM, etc). It only upserts Metro Digital Careers jobs. Existing jobs are preserved.

## Full Workflow (automatic)

When running `node scraper/index.js`, the following steps happen automatically:

1. **Check existing jobs count** - Query API by CIF (read-only)
2. **Validate company via ANAF** - Check company exists and is active
3. **Scrape jobs** - Extract jobs from Metro Digital Careers HTML (Romania only)
4. **Transform for SOLR** - Fix locations (only Romanian cities), normalize fields
5. **Upsert to API** - Add/update jobs via the Peviitor API
6. **Delete stale jobs** - Remove jobs no longer present on the site
7. **Show Summary** - Log job counts

## Workflow Flowchart

```
scraper/config/company.json (single source of truth: id, company, brand, URLs)
    │
    ▼
scraper/index.js
    │
    ▼
querySOLR(id) - via Peviitor API, just count, don't delete
    │
    ▼
scraper/company.js (validate company)
    ├── load cache (tmp/company.json → company.json)
    │   └── if fresh (<7 days), skip ANAF entirely
    ├── ANAF API ──► get company name + CIF (only if cache stale/missing)
    ├── Peviitor API ──► validate company model
    └── API ──► check existing jobs count
    │
    ▼ (if active)
scrape careers.metro.digital/jobs (HTML, Attrax)
    │
    ▼
transformJobsForSOLR()
    ├── Filter: keep only Romanian locations
    │         (Bucharest, Cluj-Napoca, etc)
    ├── Fallback: "România" for unknown
    └── Format: lowercase tags, uppercase company
    │
    ▼
upsertJobs() - via Peviitor API, duplicates handled by URL
    │
    ▼
generateJobsMarkdown() → docs/jobs.md
    └── committed to repo by CI → available on GitHub Pages
```

## File Responsibilities

| File | Role |
|------|------|
| `scraper/config/company.json` | **Single source of truth** for company identity (id, company, brand, URLs) |
| `scraper/config/company.js` | ESM wrapper that loads `scraper/config/company.json` for Node code |
| `scraper/config/scraper.json` | Scraper-specific config (API base URL) |
| `scraper/index.js` | Main entry point - full workflow: validate company → scrape → transform → upsert → generate docs/jobs.md |
| `scraper/company.js` | Validates company via ANAF + Peviitor; caches in root `company.json` (7-day TTL) and `tmp/company.json` |
| `scraper/api.js` | Peviitor API module - query, delete, upsert jobs + standalone commands |
| `scraper/anaf.js` | ANAF core module - searchCompany(brand) and getCompanyFromANAF(cif) with CUIScan/CUIFirma fallbacks |
| `scraper/markdown-generator.js` | Generates `docs/jobs.md` with company info and all scraped jobs |
| `scraper/job-validator.js` | Shared validation primitives: `validateByHead`, `validateByContent`, `DEFAULT_EXPIRED_KEYWORDS` |
| `scraper/validate-jobs.js` | Manual deep validator (content-aware); thin CLI wrapper over `scraper/job-validator.js` |
| `scraper/demoanaf.js` | CLI entry point for ANAF module (thin wrapper around `scraper/anaf.js`) |
| `tests/validate-metro-digital-jobs.js` | CI fast validator (HEAD only); thin CLI over `scraper/job-validator.js` + `scraper/api.js` |
| `tests/unit/index.test.js` | Unit tests for parseHtmlJobs, mapToJobModel, transformJobsForSOLR |
| `tests/unit/company.test.js` | Unit tests for validateAndGetCompany and fallback caching |
| `tests/unit/api.test.js` | Unit tests for API query, upsert, delete operations |
| `tests/unit/demoanaf.test.js` | Unit tests for ANAF search and company retrieval |
| `tests/integration/workflow.test.js` | Live integration tests - ANAF + Peviitor API |
| `tests/e2e/scraper.test.js` | End-to-end tests with real Metro Digital HTML |
| `tests/consistency/public.test.js` | Verifies repo is public on GitHub |
| `tests/consistency/repo.test.js` | Verifies branch, Pages, workflow files |
| `tests/consistency/topics.test.js` | Verifies required repo topics |
| `tests/consistency/workflow-naming.test.js` | Validates workflow naming conventions |

## API Endpoints

- **DemoANAF Search**: `https://demoanaf.ro/api/search?q=BRAND` - Search companies by name/brand
- **DemoANAF Company**: `https://demoanaf.ro/api/company/:cui` - Get company details by CIF
- **Peviitor API**: `https://api.peviitor.ro/v1` - jobs + company operations (no auth needed)

## Rate Limiting & Politeness

The scraper is intentionally slow to be a good citizen:

| Setting | Value | Where |
|---------|-------|-------|
| Delay between pages | 1000 ms | `scraper/index.js` — `sleep(1000)` in `scrapeAllListings()` |
| Max pages | 5 | `scraper/index.js` — `MAX_PAGES` in `scrapeAllListings()` |
| Request timeout | 15000 ms | `scraper/index.js` — `TIMEOUT` constant |
| ANAF sources | demoanaf → cuiscan, no retries | `scraper/anaf.js` |
| Concurrency | 1 (sequential) | No `Promise.all` for paginated fetches |
| User-Agent | `job_seeker_ro_spider` | Identifies the scraper in server logs |

Derived scrapers should keep these defaults unless the target site explicitly permits otherwise.

## Environment Variables

| Variable | Description |
|----------|-------------|
| `GITHUB_REPOSITORY` | Used by consistency tests — format: `owner/repo` |
| `GITHUB_TOKEN` | GitHub API token for consistency tests |

No `SOLR_AUTH` needed — all operations go through the Peviitor API (public, no auth).

## Standalone Commands

```bash
# Verify jobs in API by CIF
node scraper/api.js <CIF>

# Get company details from ANAF by CIF
node scraper/demoanaf.js <CIF>

# Search companies in ANAF by brand
node scraper/demoanaf.js search <brand>

# Validate job URLs from API by CIF (check active/expired)
node scraper/validate-jobs.js <CIF>

# Validate a single job URL
node scraper/validate-jobs.js --url <url>

# Delete expired jobs from API by CIF
node scraper/validate-jobs.js <CIF> --delete
```

## Testing

This project requires multiple levels of testing:

1. **Unit Tests** - Test individual modules (`scraper/api.js`, `scraper/company.js`) in isolation
2. **Integration Tests** - Test API interactions (ANAF, Peviitor API) in `/tests/integration` folder
3. **E2E Tests** - Test full workflow in `/tests/e2e` folder

Run tests:
```bash
npm test
```

## Temporary Files

All temporary/scratch files must be placed in `tmp/` inside the project root (never outside the project). The `tmp/` directory is in `.gitignore` and will not be committed.

## Technical Debt / Completed

- [x] Extract demoanaf.js to separate module (#2)
- [x] Write Unit Tests for all modules (#3)
- [x] Write Integration Tests in separate folder (#4)
- [x] Write E2E automated tests in separate folder (#5)
- [x] Move to Peviitor API (no direct SOLR access)
- [x] Write Unit/Component/E2E tests for index.js
