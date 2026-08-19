---
name: Profile a domain's technology stack
description: Establish account limits, then take a domain from a free counts check to a full technology profile and its recent technology changes, without wasting credits.
api: openapi/builtwith-rest-api-openapi.json
operations: [whoami_v1, usage_v2, free_v1, domain_lookup_v22, change_v1]
generated: '2026-08-14'
method: generated
source: openapi/builtwith-rest-api-openapi.json, https://api.builtwith.com/llms.txt, conventions/builtwith-conventions.yml
---

# Profile a domain's technology stack

Use this when you are asked what a website is built with, or when you need a stack profile
before deciding whether a domain is worth deeper research.

## Before you call anything

Authenticate with `Authorization: API {key}` (preferred) or `?KEY={guid}`. HTTPS only. A
`bw-` device token from Agent Device-Code Authorization works anywhere a key works.

1. `whoami_v1` — `GET /whoamiv1/api.json`. Free, no credits. Read
   `account.max_batch_size.domain_lookup` (how many domains you may pass in one `LOOKUP`),
   `rate_limits`, `credits.costs` and `privacy.flags_supported`.
2. `usage_v2` — `GET /usagev2/api.json`. Free. Confirm `remaining` before you spend.

Never skip step 1. The batch ceiling, the per-endpoint credit cost and whether `NOPII` is
required on this account are all account-specific and all published there.

## Steps

3. `free_v1` — `GET /free1/api.json?LOOKUP={domain}`. Cheap orientation: technology group
   counts (`live` / `dead`) plus first/last index timestamps. Rate limited to 1 request per
   second. Use it to decide whether a full profile is worth a credit.
4. `domain_lookup_v22` — `GET /v23/api.json?LOOKUP={domain}` (the spec documents `/v22/`;
   `llms.txt` names `/v23/` as current and keeps `/v22/` available). Returns
   `Results[].Result.Paths[].Technologies[]` with `Name`, `Tag`, `Categories`,
   `FirstDetected`, `LastDetected`, `Live`, plus `Meta` firmographics and `Spend`.
   - Up to 16 comma-separated root domains per call — batch rather than loop.
   - For throughput, add `HIDETEXT=yes&NOMETA=yes&NOPII=yes&NOATTR=yes`.
   - Add `NOPII=yes` whenever you do not need contact data. Emails, names and telephones come
     back by default.
   - `TRUST=yes` costs extra credits — only ask for it when trust is the question.
5. `change_v1` — `GET /change1/api.json?LOOKUP={domain}&SINCE=last+month`. Returns
   `Changes.events[]` with `type` (`technology_added` / `technology_removed`), `importance`
   and an AI-written `why_this_matters`. Credits are consumed only for domains that actually
   have changes.

## Rules

- Timestamps are Unix **milliseconds** on the Domain API and Unix **seconds** on the Free API.
  Do not mix them.
- `LOOKUP` takes root domains only. A URL with a path returns error `-8`.
- Errors arrive as `{"Errors":[{"Code":-N,"Message":"..."}]}` inside a 200 body **and** as
  non-200 statuses. Handle both. Full registry: `errors/builtwith-error-codes.yml`.
  - `-2` malformed key, `-3` out of credits, `-8` invalid root domain, `-99` server error.
- On `429` read `retryAfterSeconds` from the body and back off. Ceilings are 8 concurrent and
  10 requests/second; the live values are in `X-RATELIMIT-LIMIT-CONCURRENT` and
  `X-RATELIMIT-LIMIT-PERSECOND`.
- Watch `X-API-CREDITS-REMAINING` on every response rather than re-polling `usage_v2`.
- Some domains are on BuiltWith's ignore list — check
  `https://api.builtwith.com/ignoresv1/api.json` before reporting "no data" as a finding.
