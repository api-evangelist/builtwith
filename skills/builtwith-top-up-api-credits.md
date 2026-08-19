---
name: Top up account API credits safely
description: Check balance and spending limits, then charge the account's saved Stripe method for more credits using a required idempotency key so a retry cannot double-charge.
api: openapi/builtwith-agent-stripe-topup-openapi.json
operations: [getStripeTopUpBalance, getStripeTopUpConfiguration, purchaseAccountCreditsWithStripe]
generated: '2026-08-14'
method: generated
source: openapi/builtwith-agent-stripe-topup-openapi.json, https://api.builtwith.com/llms.txt, conventions/builtwith-conventions.yml
---

# Top up account API credits safely

This is the only mutating flow BuiltWith publishes. Treat it accordingly.

## Credential

`Authorization: Bearer YOUR_AGENT_BILLING_KEY`. This is a **separately scoped** credential
obtained from `https://payments.builtwith.com/agent-payment-api-config`. A general BuiltWith
API key and a temporary `bw-` device token **cannot** purchase. The legacy `?KEY=` query
parameter is deprecated on this service.

Two host forms serve the same operations:

- `https://payments.builtwith.com/v1/billing/*` (primary)
- `https://api.builtwith.com/mppx/*` (path aliases; requires manual enablement, discovery at
  `https://api.builtwith.com/mppx/openapi.json`)

## Steps

1. `getStripeTopUpBalance` — `GET /mppx/api-discovery`. Returns `credits_total`,
   `credits_used`, `credits_available`.
2. `getStripeTopUpConfiguration` — `GET /mppx/api-configuration`. Returns spending limits and
   this UTC calendar month's purchase status. Check it **before** purchasing; monthly limits
   are enforced on UTC months, not rolling windows.
3. `purchaseAccountCreditsWithStripe` — `POST /mppx/api-purchase` with body `{"credits":2000}`
   and a required `Idempotency-Key` header.
   - Quantities must be fixed increments of **2,000** credits.
   - `Idempotency-Key` must be 8–200 printable characters.
   - A replayed request returns 200 with `Idempotency-Replayed: true` and the original
     response body.

## Rules

- **Generate one idempotency key per intended purchase and reuse it only for retries of that
  exact purchase.** A new key on a retry is a second charge.
- `409` means the key is in flight or conflicts with an earlier request under the same key.
  Do **not** mint a new key and retry — poll `getStripeTopUpBalance` and reconcile first.
- `402` is a Stripe failure or no chargeable payment method on file — a human problem, not a
  retryable one.
- `403` means the billing credential is missing, the account is suspended, or spending
  permission was denied.
- `401` returns a `WWW-Authenticate` header; the credential is wrong or not the billing key.
- Every 4xx shares one `components/schemas/Error` envelope, so branch on status plus the
  error body, not on message text.
- This buys **account** credits. It cannot fund an x402 `creditKey` — those are a separate
  ledger (see `skills/builtwith-pay-per-call-with-x402.md`).
