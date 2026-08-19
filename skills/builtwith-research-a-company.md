---
name: Research a company from its name to its web estate
description: Resolve a company name to a domain, profile it, expand to every related site through shared identifiers, and check whether the domain is trustworthy.
api: openapi/builtwith-rest-api-openapi.json
operations: [company_to_url_v3, domain_lookup_v22, relationships_v4, tags_v1, redirects_v1, trust_v1]
generated: '2026-08-14'
method: generated
source: openapi/builtwith-rest-api-openapi.json, https://api.builtwith.com/llms.txt, data-model/builtwith-data-model.yml
---

# Research a company from its name to its web estate

Use this for account research, ownership mapping, and "is this a real business" checks.

## Steps

1. `company_to_url_v3` — `GET /ctu3/api.json?COMPANY={name}`. Returns candidate
   `{Domain, CompanyName, Spend, Country, Socials}` records. Pick by country and spend; do
   not assume the first row is right.
2. `domain_lookup_v22` — `GET /v23/api.json?LOOKUP={domain}` for the stack, `Meta`
   firmographics and `Spend`. Add `NOPII=yes` unless you need contacts.
3. `relationships_v4` — `GET /rv4/api.json?LOOKUP={domain}`. Returns `Relationships[]` where
   each `Identifiers[]` entry (analytics id, GTM container, ad publisher id) carries
   `Matches[]` — the other domains that used the same identifier, with `Overlap` telling you
   whether they used it at the same time.
   - Page with `OFFSET={next_skip}` while `more_results` is true. Page size is 500.
   - Add `IP=yes` to include IP-address identifiers.
4. `tags_v1` — `GET /tag1/api.json?LOOKUP=IP-1.2.3.4`. Expand a specific identifier you found
   in step 3 into its full domain set. Prefix the value with its type (`IP-`, `CA-PUB-`, …);
   `TYPES=true` returns the list of valid attribute types.
5. `redirects_v1` — `GET /redirect1/api.json?LOOKUP={domain}`. `Inbound` and `Outbound`
   redirect history reveals rebrands and retired domains.
6. `trust_v1` — `GET /trustv1/api.json?LOOKUP={domain}` (llms.txt documents `/trustv2/` as
   current). Returns `Assessment.TrustLevel` — one of `Unverified`, `RestrictedContent`,
   `HighRisk`, `Caution`, `VerificationRecommended`, `Neutral`, `Trusted` — with `Reasons`,
   `ContentSafety` flags and a `BusinessProfile`.

## Rules

- `Overlap: true` is the strong ownership signal. Two domains that used one analytics id in
  different eras are usually an agency reusing a snippet, not one owner.
- Relationship expansion fans out fast. Cap the graph at one or two hops and record which
  identifier justified each edge.
- Contact fields under `Meta` are personal data. Request `NOPII=yes` by default and only drop
  it when the task genuinely requires contacts.
- `-7` means the endpoint takes a single lookup — split multi-domain calls.
- Errors: `errors/builtwith-error-codes.yml`. Rate limits and credit headers:
  `rate-limits/builtwith-rate-limits.yml`.
