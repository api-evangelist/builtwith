---
name: Call BuiltWith with no account using x402
description: Pay per HTTP call in Base USDC — read the 402 challenge, sign it, retry the identical request — or buy reusable prepaid lookup units through the MCP server.
api: openapi/builtwith-x402-pay-per-call-openapi.json
operations: [lookupDomainTechnologies, findRelatedWebsites, getDomainTechnologyChanges, findCompanyDomains, getDomainTrust, askForWebsites, purchaseBasicListPass, purchaseProListPass]
generated: '2026-08-14'
method: generated
source: openapi/builtwith-x402-pay-per-call-openapi.json, https://api.builtwith.com/.well-known/x402, https://api.builtwith.com/llms.txt
---

# Call BuiltWith with no account using x402

Use this when the agent holds Base USDC and has no BuiltWith account or API key.

## Read the descriptor first — never hardcode payment values

`GET https://api.builtwith.com/.well-known/x402` is authoritative for the facilitator,
network, asset, payment address, tool inventory, pricing tiers and List passes. At the time
of writing it declares x402 v2 on Base mainnet (`eip155:8453`), USDC with 6 decimals, and a
Coinbase CDP facilitator — but the descriptor wins over any value written here.

## One-off HTTP calls

1. Send the intended request to the route you want. All routes live under
   `https://api.builtwith.com/agent/` and carry `security: []` — no key, no account.
   - `lookupDomainTechnologies` — `GET /agent/domain?domain={d}&liveOnly={bool}`
   - `findRelatedWebsites` — `POST /agent/relationships`
   - `getDomainTechnologyChanges` — `POST /agent/changes`
   - `findCompanyDomains` — `POST /agent/company-domains`
   - `findDomainsByTag`, `recommendTechnologies`, `getDomainRedirects`, `getDomainKeywords`,
     `getDomainTrust`, `getCompanyIdentifiers`, `searchTechnologies`, `askForWebsites`
2. The server answers **402** with a base64 `PAYMENT-REQUIRED` header carrying the x402 v2
   payment requirements.
3. Sign the requirement with Base USDC and **retry the identical request** with a
   `PAYMENT-SIGNATURE` header.
4. On success you get **200** with the BuiltWith result and a `PAYMENT-RESPONSE` settlement
   header.

Each call is a fixed `$0.0495` USDC (`x-payment-info.price` on every operation).

## Reusable prepaid units (better for repeated lookups)

Through the MCP server at `https://api.builtwith.com/mcp`:

1. `x402-pricing` — public config and batch tiers. No payment. Optional `credits` quantity
   returns a quote.
2. `x402-credit-purchase` — minimum 2000 units, plus the `payer` wallet. Sign the returned
   payment requirement and retry with the payment payload in MCP request metadata at
   `_meta["x402/payment"]`.
3. Store the returned `creditKey` and pass it to every `x402-*` lookup tool
   (`x402-domain-lookup`, `x402-change-api`, `x402-trust-api`, …).
4. `x402-credit-balance` — purchased / used / pending / available for a `creditKey`. Free.
5. Top up by passing the existing `creditKey` back to `x402-credit-purchase`. The payer
   wallet must match the wallet that created the key.

For list-shaped work, `purchaseBasicListPass` / `purchaseProListPass` (or the
`x402-list-pass-purchase` tool) buy a 30-day pass; `x402-list-api` and
`x402-keywords-search-api` then require both `payer` and `passToken`.

## Rules

- Lookup units do not expire, and a failed BuiltWith call releases the reserved unit.
- **400 and 502 are not settled** — a validation failure or an upstream failure does not
  charge the verified payment. Do not treat them as spend.
- **503** means x402 is not configured on that server; stop retrying.
- A `creditKey` lives in a separate x402 ledger, not a BuiltWith account. It is not the same
  currency as account API credits, and the Stripe top-up API cannot fund it (see
  `skills/builtwith-top-up-api-credits.md`).
- Retry the *identical* request after signing. A changed body or query invalidates the
  challenge.
