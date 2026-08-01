---
name: revenuebase-resolve-organizations
description: Resolve a company name to a canonical RevenueBase record, or discover net-new organizations by keyword using semantic search.
generated: '2026-07-20'
method: generated
api: Revenuebase API v2
base_url: https://api.revenuebase.ai
auth: API key in the x-key header
operations:
- resolve_company_v2_organization_resolve_post
- discover_company_v2_organization_discover_post
- get_usage_v2_account_balance_get
source: openapi/revenuebase-openapi.json
---

# Resolve and discover organizations with RevenueBase

Authenticate with your API key in the `x-key` header. Each call is metered in
credits; check `GET /v2/account/balance` (`get_usage_v2_account_balance_get`)
first. A `402` means insufficient credits.

## Resolve a known company name

1. `POST /v2/organization/resolve` (`resolve_company_v2_organization_resolve_post`)
   with body `{"company_name": "Acme Inc", "result_count": 3}`. Optionally narrow
   with `headquarters_city`, `headquarters_state`, `headquarters_country`.
2. `result_count` must be between 1 and 10 (`400` if out of range). Deducts 1
   credit per request.
3. The response `companies[]` are ranked by `similar_score`; each carries an
   `rbid` (join key), `company_name`, `about_us`, and headquarters fields. Use the
   `rbid` to join against Person/Insight data.

## Discover net-new organizations by keyword

1. `POST /v2/organization/discover` (`discover_company_v2_organization_discover_post`)
   with body `{"keyword": "developer tools for API governance", "result_count": 500,
   "min_similarity_score": 0.5}`.
2. `result_count` may be 1–10000; `min_similarity_score` 0–0.95. Credits are
   deducted only for results at or above `min_similarity_score`, so set the
   threshold to control spend.
3. Iterate on the keyword and threshold to trade recall against precision and cost.

## Notes

- Join keys are `rbid`, `rbid_org`, and `linkedin_url` — never email.
- Insights are organization-level only; there are no contact-level insights.
