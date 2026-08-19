---
name: Revenuebase
description: Use when building B2B data pipelines, verifying email addresses, enriching contact lists, querying verified contact and company data via Snowflake or S3, or searching for prospects in Gigasheet. Agents should reach for this skill when users need to access 400M+ verified B2B contacts, validate email deliverability, integrate data feeds into data warehouses, or build GTM workflows on top of clean B2B data.
metadata:
    mintlify-proj: revenuebase
    version: "1.0"
---

# RevenueBase Skill

## Product summary

RevenueBase is a B2B data platform delivering 400M+ verified contacts and 65M+ companies with 97%+ email deliverability. It provides three core capabilities: flat-rate data feeds (monthly refreshes to Snowflake or S3), per-outcome email verification and enrichment APIs, and a spreadsheet interface (Gigasheet) for no-code searching and filtering.

**Key files and paths:**
- API base URL: `https://api.revenuebase.ai/v2`
- Snowflake tables: `RELEASES.RELEASE.PER_LATEST`, `ORG_LATEST`, `INSIGHTS_LATEST`, `VELOCITY_BASE_UNLIMITED_LATEST`, `VELOCITY_ENHANCED_UNLIMITED_LATEST`
- S3 delivery: date-stamped folders with `/per/`, `/org/`, `/insights/`, `/velocity_base_unlimited/` subfolders
- Authentication: `x-key` header (not `Authorization`)
- Primary docs: https://docs.revenuebase.ai

## When to use

Reach for this skill when:
- **Verifying email addresses** — validating single emails in real time or batch-processing lists of 10+ addresses
- **Enriching contact lists** — adding verified work emails, job titles, company data, or firmographic attributes to existing records
- **Building data pipelines** — querying B2B data in Snowflake or S3 for analytics, segmentation, or outbound campaigns
- **Searching for prospects** — using Gigasheet to filter by company size, industry, job title, seniority, geography, or tech stack without SQL
- **Assessing data freshness** — understanding when records were last verified and whether they're suitable for outbound or analytics use
- **Resolving organization names** — matching company names to canonical records or discovering companies by keyword

## Quick reference

### Core identifiers

| ID | Entity | Scope | Use case |
|---|---|---|---|
| `RBID_PER` | Person | Global | Deduplication, career tracking, identity resolution |
| `RBID_ORG` | Organization | Global | Company-level firmographics, insights, joining |
| `RBID_PAO` | Contact | Per company | Outbound campaigns, work email lookup, enrichment |

### API authentication

```bash
# Set environment variable
export REVENUEBASE_API_KEY=your_key_here

# Include in every request
curl https://api.revenuebase.ai/v2/account/balance \
  -H "x-key: $REVENUEBASE_API_KEY"
```

**Common auth errors:**
- `401 Unauthorized` — Check header is `x-key` (not `Authorization`), key has no trailing spaces, key hasn't been revoked
- `403 Forbidden` — Valid key but no permission for this endpoint; check account tier

### Email verification endpoints

| Task | Endpoint | Response | Best for |
|---|---|---|---|
| Verify one email | `POST /v2/email/verify` | Immediate (Valid/Invalid/Unknown) | Form validation, live enrichment, single lookups |
| Verify batch | `POST /v2/email/verify/batch` | Async (returns `process_id`) | Cleaning lists of 10+ addresses, pre-send scrubbing |
| Check job status | `GET /v2/jobs/{process_id}` | Current status (QUEUED/PROCESSING/COMPLETED) | Polling batch jobs |
| Download results | `GET /v2/jobs/{process_id}/download` | Processed file with status column | Retrieving batch results |

### Data tables and joins

| Table | Primary key | Use when |
|---|---|---|
| `PER_LATEST` | `RBID_PAO` | Building prospect lists, outbound campaigns, contact enrichment |
| `ORG_LATEST` | `RBID` | Joining firmographic data (headcount, revenue, industry) to contacts |
| `INSIGHTS_LATEST` | `RBID_ORG` | Adding buying signals, hiring activity, growth indicators |
| `VELOCITY_BASE_UNLIMITED_LATEST` | `RBID_PAO` | Contact + organization data in one table (no insights) |
| `VELOCITY_ENHANCED_UNLIMITED_LATEST` | `RBID_PAO` | Contact + organization + insights in one table |
| `HISTORICAL_EXPERIENCE` | `LINKEDIN_URL` | Job history and career progression |

### Snowflake query template

```sql
-- High-confidence outbound: recently verified emails
SELECT *
FROM RELEASES.RELEASE.PER_LATEST
WHERE EMAIL_LAST_VERIFIED_AT >= DATEADD(day, -60, CURRENT_DATE());

-- Enrichment with firmographics
SELECT p.*, o.HEADCOUNT, o.REVENUE, o.INDUSTRY
FROM RELEASES.RELEASE.PER_LATEST p
LEFT JOIN RELEASES.RELEASE.ORG_LATEST o ON p.RBID_ORG = o.RBID;

-- Pre-joined table (simpler)
SELECT *
FROM RELEASES.RELEASE.VELOCITY_ENHANCED_UNLIMITED_LATEST
WHERE EMAIL_LAST_VERIFIED_AT >= DATEADD(day, -60, CURRENT_DATE());
```

### Rate limits and backoff

- **Single email verification:** 5 requests per second
- **Batch jobs:** No per-request limit; process asynchronously
- **Rate limit response:** `429 Too Many Requests`
- **Backoff strategy:** Exponential backoff (1s, 2s, 4s, 8s, 16s)
- **Track usage:** Read `X-RateLimit-Remaining` and `X-RateLimit-Reset` headers

## Decision guidance

### When to use real-time vs batch email verification

| Scenario | Use real-time | Use batch |
|---|---|---|
| Validating form submission | ✓ | — |
| Live CRM sync | ✓ | — |
| Cleaning a list of 10+ emails | — | ✓ |
| Pre-send scrubbing | — | ✓ |
| One-off lookups | ✓ | — |

### When to use Snowflake vs S3 vs Gigasheet

| Use case | Snowflake | S3 | Gigasheet |
|---|---|---|---|
| SQL queries, analytics | ✓ | — | — |
| Data warehouse integration | ✓ | ✓ | — |
| No-code searching/filtering | — | — | ✓ |
| Exporting to CSV | — | — | ✓ |
| Enriching your own lists | — | — | ✓ |
| Building pipelines | ✓ | ✓ | — |

### When to use base tables vs pre-joined tables

| Need | Use base tables | Use pre-joined |
|---|---|---|
| Contact + organization only | — | `VELOCITY_BASE_UNLIMITED_LATEST` |
| Contact + organization + insights | — | `VELOCITY_ENHANCED_UNLIMITED_LATEST` |
| Custom joins or filtering | `PER_LATEST` + `ORG_LATEST` | — |
| Historical job data | `HISTORICAL_EXPERIENCE` | — |

### When to filter by `updated_at` vs `email_last_verified_at`

| Goal | Filter | Threshold |
|---|---|---|
| High-confidence outbound (email campaigns) | `EMAIL_LAST_VERIFIED_AT` | Within 60 days |
| Enrichment and analytics | `UPDATED_AT` | Within 90 days (higher confidence); older records acceptable |
| AI agent pipelines | Both | Pass both fields to agent for programmatic decisions |

## Workflow

### Verify a single email address

1. **Get your API key** — Go to app.revenuebase.ai → Developers → copy Production API Key
2. **Set environment variable** — `export REVENUEBASE_API_KEY=your_key`
3. **Make the request** — `POST /v2/email/verify` with `{"email": "address@company.com"}`
4. **Check the response** — Status is `Valid`, `Invalid`, or `Unknown`; `Valid` and `Invalid` deduct 1 credit each
5. **Add metadata if needed** — Append `?metadata=true` to get MX record, email provider, security gateway, and failure reason

### Batch verify a list of emails

1. **Prepare your file** — CSV or JSON with email addresses (one per row or in an `email` field)
2. **Upload the batch** — `POST /v2/email/verify/batch` with file as multipart form data; get back `process_id`
3. **Poll for status** — `GET /v2/jobs/{process_id}` every 5–10 seconds until `current_status` is `COMPLETED`
4. **Download results** — `GET /v2/jobs/{process_id}/download` returns your file with a `status` column added
5. **Handle rate limits** — If you get `429`, wait for `X-RateLimit-Reset` header or use exponential backoff

### Query data in Snowflake

1. **Set up integration** — Go to RevenueBase dashboard → Integrations → enter your Snowflake Account ID
2. **Wait for provisioning** — Allow up to 10 minutes for data sharing
3. **Verify access** — Query `SELECT * FROM RELEASES.RELEASE.PER_LATEST LIMIT 100;`
4. **Choose your table** — Use `VELOCITY_ENHANCED_UNLIMITED_LATEST` for most use cases (pre-joined contact + org + insights)
5. **Filter by freshness** — For outbound, add `WHERE EMAIL_LAST_VERIFIED_AT >= DATEADD(day, -60, CURRENT_DATE())`
6. **Join if needed** — Use `LEFT JOIN` to include contacts without organization records; use `INNER JOIN` for org-only results

### Search and export in Gigasheet

1. **Access Gigasheet** — Included free with your RevenueBase subscription
2. **Choose your dataset** — Person + Organization or Person + Organization + Insights
3. **Apply filters** — Use 150+ available filters (company size, industry, job title, seniority, geography, tech stack)
4. **Save your view** — Views auto-update monthly with new data
5. **Export results** — Download as CSV or export directly to HubSpot
6. **Enrich your list** — Upload your own contacts to match against RevenueBase data

### Enrich a contact list

1. **Prepare your list** — CSV with names, companies, or email addresses
2. **Upload to Gigasheet** — Use the enrichment workflow
3. **Match against RevenueBase** — System matches your records to verified contacts
4. **Add fields** — Select which RevenueBase fields to append (email, phone, job title, company data)
5. **Download enriched list** — Export as CSV with new columns populated

## Common gotchas

- **Wrong auth header** — Use `x-key`, not `Authorization` or `x-api-key`. This is the #1 cause of 401 errors.
- **Trailing spaces in API key** — Copy the full key with no leading or trailing whitespace.
- **Using v1 endpoints** — v1 was retired July 7, 2026. All requests must use `/v2/` paths. v1 requests return `410 Gone`.
- **Forgetting to poll batch jobs** — Batch email verification is async. You must poll `GET /v2/jobs/{process_id}` until `COMPLETED`, then download results.
- **Using INNER JOIN instead of LEFT JOIN** — If you join `PER_LATEST` to `ORG_LATEST` with `INNER JOIN`, you'll lose contacts without organization records. Use `LEFT JOIN` unless you specifically want org-only results.
- **Not filtering by email freshness** — For outbound campaigns, always filter to `EMAIL_LAST_VERIFIED_AT >= DATEADD(day, -60, CURRENT_DATE())`. All emails in the dataset are verified as valid, but older verifications may be stale.
- **Confusing RBID_PER with RBID_PAO** — `RBID_PER` is the person (stable across all companies they work at); `RBID_PAO` is the contact (person at a specific company). Use `RBID_PAO` for outbound; use `RBID_PER` for deduplication.
- **Sparse profiles** — Some records have minimal data (name + company only). These are sourced the same way but the underlying source contains less detail. Prioritize records with verified emails; sparse profiles without emails should be treated with lower confidence.
- **S3 access via AWS Console** — Use AWS CLI with `--request-payer requester` flag, not the AWS Console web UI. The Console doesn't support request-payer and will return access errors.
- **Exceeding rate limits** — Single email verification is limited to 5 requests/second. Use batch verification for lists. Implement exponential backoff on `429` responses.
- **Stale records** — Records not verified for 1 year are deprecated and removed from the dataset. Check `updated_at` and `email_last_verified_at` to assess freshness before using in high-stakes workflows.

## Verification checklist

Before submitting work with RevenueBase:

- [ ] **API key is correct** — Verify `x-key` header is present, key has no trailing spaces, and key hasn't been revoked
- [ ] **Using v2 endpoints** — All paths start with `/v2/`, not `/v1/`
- [ ] **Batch jobs are complete** — Polled `GET /v2/jobs/{process_id}` until `current_status` is `COMPLETED` before downloading
- [ ] **Freshness filters applied** — For outbound, filtered to `EMAIL_LAST_VERIFIED_AT >= DATEADD(day, -60, CURRENT_DATE())`
- [ ] **Correct join type** — Used `LEFT JOIN` to preserve contacts without org records; used `INNER JOIN` only when org data is required
- [ ] **Correct identifiers used** — Using `RBID_PAO` for contacts, `RBID_ORG` for organizations, `RBID_PER` for deduplication
- [ ] **Rate limits respected** — Implemented exponential backoff for `429` responses; batch verification for lists of 10+
- [ ] **Snowflake paths correct** — Tables are under `RELEASES.RELEASE.*`, not other schemas
- [ ] **S3 access uses CLI** — Using `aws s3 cp` with `--request-payer requester`, not AWS Console
- [ ] **Metadata fields understood** — Know what `updated_at`, `email_last_verified_at`, and `revenuebase_contact_verification_summary` mean before filtering on them

## Resources

**Comprehensive navigation:** https://docs.revenuebase.ai/llms.txt

**Critical documentation pages:**
1. [Data Overview](https://docs.revenuebase.ai/docs/data-feeds/overview) — Core identifiers, table index, schema conventions, delivery paths
2. [API Overview & Authentication](https://docs.revenuebase.ai/api-reference/v2/overview) — v2 endpoints, authentication, rate limits
3. [Data Freshness & Quality](https://docs.revenuebase.ai/docs/data-features/data-freshness/data-freshness) — Verification cycles, staleness, quality filters, recommended thresholds

---

> For additional documentation and navigation, see: https://docs.revenuebase.ai/llms.txt