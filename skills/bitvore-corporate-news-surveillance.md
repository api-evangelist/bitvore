---
name: Surveil a company for material business events
description: Resolve a company to a Bitvore bvId, pull its Cellenus corporate news and signals over a date range, and fetch the enriched record for anything material.
api: openapi/bitvore-corporate-openapi.yml
operations:
  - handleOrgLookupUsingGET_1
  - handleOrgMatchingUsingPOST_1
  - handleGetOrgUsingGET_2
  - handleNewsSearchUsingGET
  - handleCorpAdvancedSearchUsingPOST_1
  - handleGetEnrichedRecordUsingGET
  - handleSentimentScoreQueryUsingGET_1
generated: '2026-08-07'
method: generated
source: openapi/_original/bitvore-corporate-swagger.json + https://developer.bitvore.com/v2/getting-started
---

# Surveil a company for material business events

Authenticate first — see `bitvore-authenticate.md`. This flow needs a **Corporate** dataset key.

## 1. Resolve the company to a bvId

Never guess a `bvId`. Resolve it.

- One company by name or ticker: `handleOrgLookupUsingGET_1` — `GET /v2/corp/organizations/lookup`
- A list of companies at once: `handleOrgMatchingUsingPOST_1` — `POST /v2/corp/organizations/match`
- Confirm the record and pull alternate identifiers: `handleGetOrgUsingGET_2` —
  `GET /v2/corp/organizations/{bvId}`

The organization record carries CUSIP, ISIN, SEDOL and FIGI **only if your licence includes them**.
If those fields are absent, that is an entitlement result, not an error — do not retry.

## 2. Pull the news

`handleNewsSearchUsingGET` — `GET /v2/corp/news`

Required: `startDate` and `endDate`, both ISO 8601 datetimes. These are **not optional** in v2.
Useful filters: `bvIds`, `portfolioId`, `signals`, `significance`, `articleType`, `dateType`, `tz`.

```
GET /v2/corp/news?bvIds=b00001ab7&startDate=2026-07-01T00:00:00Z&endDate=2026-08-01T00:00:00Z&significance=HIGH
X-BV-APIKEY: <APIKEY>
```

For a structured query too large or complex for a query string, use
`handleCorpAdvancedSearchUsingPOST_1` — `POST /v2/corp/news`. POST **requires** the header form of
the key; the `key=` query parameter will not be accepted.

Multi-value parameters are plural in v2 (`bvIds`, `signals`). The singular v1 names only work on the
legacy unprefixed paths.

## 3. Page through the results

Responses are `CorpNewsResultsResponse`:

```json
{ "success": true, "response": [ ... ], "returned": 10, "total": 100 }
```

Page with `pageNo` + `pageSize`. Keep requesting until the accumulated `returned` reaches `total`.
There is no cursor and no `Link` header. The v1 `offset` parameter was removed in v2.

## 4. Read the signals

Each article carries `signals` (e.g. `MergeAcquisition.AcquisitionComplete`), `significance`
(`HIGH`/…), `sentiment` (-1.0 to 1.0), `matchedOrgs`, `referencedOrgs`, `locations`, `publishedAt`
and `availableAt`. The signal vocabulary is published at
https://developer.bitvore.com/v2/docs/api-reference/signals.

Two things that will bite an agent:
- **Themes are dead.** Any Theme-related field returns an empty list as of the 2023-09-01 release.
  Do not treat an empty Theme list as a failure.
- **Full article content is not returned.** It was removed on 2021-12-24 for upstream licensing
  reasons. Use `sourceUrl` if you need the original.
- **ESG sentiment requires additional credentials** beyond a base Corporate key.

## 5. Enrich anything material

For an article `key` worth acting on, fetch the enriched record:

`handleGetEnrichedRecordUsingGET` — `GET /v2/corp/news/byKey/{recordKey}/enriched`

For a batch of keys, use `handleGetEnrichedRecordUsingPOST` — `POST /v2/corp/news/byKey/enriched`.

## 6. Optional — trend the score

`handleSentimentScoreQueryUsingGET_1` — `GET /v2/corp/sentimentscores` returns trended sentiment,
growth and risk scores over a date range, which is the right signal for "is this deteriorating"
rather than "what happened today".

## Rules

- `startDate`/`endDate` are required wherever a date range exists. A missing date is a `400`
  `Invalid parameter` with a `ReasonResponse`, not a default window.
- Check `success` on every response before reading `response`. HTTP status is correct as of v2, but
  the envelope is the contract.
- No idempotency key exists. Retries are safe on GET only.
- No published rate limit. Throttle conservatively and treat sustained `500`s as backpressure.
