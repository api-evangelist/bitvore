---
name: Find municipal bond news by CUSIP, location or sector
description: Query the Cellenus Municipal dataset for news and ESG events tied to bond issuers, by CUSIP, state/city, FIPS code, sector, or a saved CUSIP portfolio.
api: openapi/bitvore-muni-openapi.yml
operations:
  - handleCusipMuniSearchUsingGET
  - handleLocationMuniSearchUsingGET
  - handleFipsMuniSearchUsingGET
  - handleMuniAdvancedSearchUsingPOST_1
  - handlePortfolioMuniSearchUsingGET
  - handleEconSearchUsingGET_2
  - handleFipsEconSearchUsingGET_1
generated: '2026-08-07'
method: generated
source: openapi/_original/bitvore-muni-swagger.json + https://developer.bitvore.com/v2/getting-started
---

# Find municipal bond news by CUSIP, location or sector

Needs a **Municipal** dataset key — a Corporate key will 401 here. See `bitvore-authenticate.md`.

Base path: `https://api.bitvore.com/v2/muni/`

## Pick the right entry point

There is no single multi-parameter search. v2 split the legacy `/muninews` call into
action-specific endpoints — choose by what you actually hold:

| You have | Operation | Path |
|---|---|---|
| A CUSIP | `handleCusipMuniSearchUsingGET` | `GET /v2/muni/news` (`cusip`) |
| A state / county / city | `handleLocationMuniSearchUsingGET` | `GET /v2/muni/news/bylocation` |
| A FIPS code | `handleFipsMuniSearchUsingGET` | `GET /v2/muni/news/byfips` |
| A saved CUSIP portfolio | `handlePortfolioMuniSearchUsingGET` | `GET /v2/muni/news/byportfolio` |
| A structured multi-criteria query | `handleMuniAdvancedSearchUsingPOST_1` | `POST /v2/muni/news` |

```
GET https://api.bitvore.com/v2/muni/news?state=ca&sector=education&startDate=2026-07-01T00:00:00Z&endDate=2026-08-01T00:00:00Z
X-BV-APIKEY: <APIKEY>
```

`POST /v2/muni/news` **requires** the `X-BV-APIKEY` header form. The `key=` query parameter is
GET-only.

## What comes back

`MuniNewsResponse` — a paged envelope:

```json
{
  "success": true, "returned": 10, "total": 100,
  "response": [{
    "key": "0700...", "title": "...", "sourceUrl": "...", "excerpt": "...",
    "matchedCusips": ["00037CSW1"],
    "locations": ["/Orange County/California/United States", "Los Angeles//California/United States"],
    "sectors": ["Utilities"], "significance": "HIGH",
    "publishedAt": "2026-06-29T21:41:19.109Z", "availableAt": "2026-06-29T21:41:21.109Z"
  }]
}
```

Location strings are slash-delimited `city/county/state/country` with **empty segments preserved** —
`Los Angeles//California/United States` has no county. Split on `/` and keep the empties; do not
collapse them or the fields shift.

`publishedAt` is when the source published; `availableAt` is when Bitvore made it queryable. For
incremental polling, window on `availableAt` semantics via `dateType`, not on `publishedAt`, or you
will miss late-arriving articles.

## Economic context

The Municipal surface also carries US economic news for a geography, which is the macro backdrop to
an issuer story:

- `handleEconSearchUsingGET_2` — `GET /v2/muni/econnews/bylocation`
- `handleFipsEconSearchUsingGET_1` — `GET /v2/muni/econnews/byfips`

## Rules

- `startDate`/`endDate` are required ISO 8601 datetimes on every date-ranged query.
- Page with `pageNo`/`pageSize`; accumulate until `returned` totals `total`.
- Multi-value params are plural in v2.
- Full article text is not returned — follow `sourceUrl`.
- Check `success` before reading `response`; failures carry `reason` / `reasonSupport`
  (see `errors/bitvore-problem-types.yml`).
