---
name: Find and qualify websites using a technology
description: Resolve a fuzzy technology name to its canonical form for free, list the websites using it with attribute filters, page through the results, then enrich the shortlist.
api: openapi/builtwith-rest-api-openapi.json
operations: [trends_v6, vector_search_v1, lists_v12, ask_v1, domain_lookup_v22]
generated: '2026-08-14'
method: generated
source: openapi/builtwith-rest-api-openapi.json, https://api.builtwith.com/llms.txt, conventions/builtwith-conventions.yml
---

# Find and qualify websites using a technology

Use this for "who uses X", competitive displacement lists, and technographic prospecting.

## Validate the technology name first — it is free

`lists_v12` charges credits and rejects a name it does not recognise, so never guess.

1. `trends_v6` — `GET /trends/v6/api.json?TECH={name}`. Free. A valid name returns
   `Tech.name` (the canonical form), `tag`, `categories`, `description` and `coverage` counts
   across the top 10K / 100K / 1M / whole internet. An unknown name returns
   `{"Errors":[{"Code":-8}]}`.
2. If validation fails, `vector_search_v1` — `GET /vector/v1/api.json?QUERY={free text}`.
   Semantic search across technologies and categories; costs 1 credit, `LIMIT` defaults to 10
   (max 100). Take `Results[].Name` and re-validate it with `trends_v6`.

Use `Tech.name` verbatim in the next step, with spaces replaced by dashes.

## List the sites

3. `lists_v12` — `GET /lists12/api.json?TECH={canonical}`. Returns `Results[]` with `D`
   (domain), `S` (monthly tech spend), `FD`/`LD` (detection window) and, with `META=yes`,
   names, titles and social links.
   - Combine technologies: `OTHERTECHS=Google-Analytics,Meta-Pixel` (max 16).
   - Geography: `COUNTRY=US,CA`.
   - Recency: `SINCE=30+days+ago` (mutually exclusive with `ALL=yes`).
   - Numeric filters use `value|OPERATOR` where the operator is `EQ`, `LT`, `LTE`, `GT` or
     `GTE`, defaulting to `GTE`: `SPEND=100|GT`, `REVENUE=100000|GT`, `EMPLOYEES=50|GTE`,
     `FOLLOWERS=10000|GTE`, `PAGERANK=1000000|LT`, plus `SKU`, `SITEMAP`, `BWRANK`, `TRANCO`,
     `MAJESTIC`, `BWS`, `ECAT` and the AI attributes `AIM`, `AIO`, `AIR`, `AIV`.
   - Attribute filters are **ANDed**. Every one you add narrows the set.
4. Page with `OFFSET={previous NextOffset}`. Stop when `NextOffset` is `END`. Do not
   construct offsets yourself — they are opaque.

## Natural-language alternative

If the criteria are prose rather than a technology name, use `ask_v1` —
`GET /ask1/api.json?QUERY=Magento%20websites%20in%20Spain`. 1 credit, returns a sample by
default; add `COMMIT=true` to run the full report (up to 1000 results). Page with
`NEXTOFFSET`. Ask results use Lists attributes but never return `LOS`.

## Enrich the shortlist

5. `domain_lookup_v22` — batch up to 16 shortlisted domains per call for the full stack and
   firmographics. See `skills/builtwith-profile-a-domain.md`.

## Rules

- Validate with `trends_v6` before every new `TECH` value. It is the only free way to avoid
  spending a credit on a typo.
- `ALL=yes` and `SINCE` cannot be combined.
- Add `NOPII=yes` on the enrichment step unless contact data is genuinely required.
- `-4` means the technology endpoint did not recognise the name; `-5` means the plan's
  technology-search allowance is exhausted (Basic allows 2) — that is a billing problem, not a
  retryable error.
