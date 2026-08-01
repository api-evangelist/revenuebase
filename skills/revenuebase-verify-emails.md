---
name: revenuebase-verify-emails
description: Verify email deliverability with RevenueBase — one address in real time, or thousands via an asynchronous batch job.
generated: '2026-07-20'
method: generated
api: Revenuebase API v2
base_url: https://api.revenuebase.ai
auth: API key in the x-key header
operations:
- verify_email_v2_email_verify_post
- verify_email_batch_v2_email_verify_batch_post
- list_jobs_v2_jobs_get
- get_job_v2_jobs__process_id__get
- download_job_v2_jobs__process_id__download_get
- cancel_job_v2_jobs__process_id__cancel_post
- get_usage_v2_account_balance_get
source: openapi/revenuebase-openapi.json
---

# Verify emails with RevenueBase

Authenticate every request with your API key in the `x-key` header (not
`Authorization`, not `x-api-key`). A `Valid` or `Invalid` result deducts 1 credit.

## Single email (real time)

1. `POST /v2/email/verify` (`verify_email_v2_email_verify_post`) with body
   `{"email": "user@example.com"}`. Add `?metadata=true` to also get MX presence,
   provider, security gateway, and failure reason.
2. Read `status` from the response: `Valid`, `Invalid`, or `Unknown`.
3. Handle errors: `402` means insufficient credits — top up and retry. This
   endpoint is rate limited to 5 requests/second; on `429` read `X-RateLimit-Reset`
   before retrying.

## Batch (asynchronous)

1. `POST /v2/email/verify/batch` (`verify_email_batch_v2_email_verify_batch_post`)
   as `multipart/form-data` with a `.csv` or `.json` file. A `400 invalid_extension`
   means the file type is wrong. On success you receive a `process_id`.
2. Poll `GET /v2/jobs/{process_id}` (`get_job_v2_jobs__process_id__get`) until
   `current_status` is `COMPLETED` (other states: `QUEUED`, `PROCESSING`, `ERROR`,
   `CANCELLING`, `CANCELLED`). Use `GET /v2/jobs` (`list_jobs_v2_jobs_get`) to see
   all active jobs.
3. When complete, `GET /v2/jobs/{process_id}/download`
   (`download_job_v2_jobs__process_id__download_get`) streams the output file
   (filename in the `Content-Disposition` header). `400 not_completed` means the
   job is not finished yet.
4. To stop a job, `POST /v2/jobs/{process_id}/cancel`
   (`cancel_job_v2_jobs__process_id__cancel_post`). `400 not_cancellable` means it
   already finished; `400 already_cancelled` means it is already cancelling.

## Notes

- Check remaining credits with `GET /v2/account/balance`
  (`get_usage_v2_account_balance_get`) before large batches.
- All emails already in the RevenueBase dataset are pre-verified valid — you only
  need this API for addresses you bring yourself.
