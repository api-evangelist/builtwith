---
name: Bulk enrich a large domain list
description: Submit thousands of domains as an asynchronous job, poll it to completion, and retrieve the one-shot result without losing it.
api: openapi/builtwith-rest-api-openapi.json
operations: [whoami_v1, domain_bulk_submit_v22, domain_bulk_status_v22, domain_bulk_result_v22, usage_v2]
generated: '2026-08-14'
method: generated
source: openapi/builtwith-rest-api-openapi.json, https://api.builtwith.com/llms.txt
---

# Bulk enrich a large domain list

Use this instead of looping `domain_lookup_v22` whenever you have more domains than the
per-call batch ceiling.

## Steps

1. `whoami_v1` — `GET /whoamiv1/api.json`. Free. Read
   `account.max_batch_size.domain_bulk_submit` (5000 at the time of writing) and
   `credits.remaining`. Do not hardcode the ceiling; it is per account.
2. `domain_bulk_submit_v22` — `POST /v23/domain/bulk` with
   `Content-Type: application/json` and a body of:

   ```json
   {
     "lookups": ["example.com", "builtwith.com"],
     "options": { "noMeta": false, "noPii": true, "hideText": false, "hideDL": false, "liveOnly": false }
   }
   ```

   - A small batch returns the ordinary Domain API result **synchronously** (200). The
     threshold is reported back as `sync_max`.
   - A large batch returns **202** with `{job_id, status: "queued", count, sync_max}`.
3. `domain_bulk_status_v22` — `GET /v23/domain/bulk/{job_id}`. Poll until `status` is
   `completed`. The response carries `created_utc`, `completed_utc`, `count` and a
   `result_url`.
4. `domain_bulk_result_v22` — `GET /v23/domain/bulk/{job_id}/result`. Returns the Domain API
   JSON payload.

## Rules

- **Results are deleted after the first successful read.** Persist the response before you
  parse it. There is no second fetch and no replay.
- Handle both outcomes of step 2. A 200 means you already have the answer and must not poll;
  a 202 means you must.
- Set `noPii: true` in `options` unless contacts are required — it is the bulk equivalent of
  `NOPII=yes`.
- Poll with backoff and respect the 8-concurrent / 10-per-second ceiling; a bulk job does not
  exempt you from rate limits.
- If the job errors, `status` becomes `error`; re-submit rather than re-polling.
- Check `X-API-CREDITS-REMAINING` before submitting — a 5000-domain job that runs out of
  credits mid-flight is more expensive to recover from than a `usage_v2` call.
