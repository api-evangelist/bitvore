---
name: Export Cellenus datasets and reconcile with changesets
description: Bulk-download yearly or daily Cellenus corporate dataset exports, then keep an offline copy current using the added/modified/removed changeset feed.
api: openapi/_original/bitvore-corporate-openapi.yml
operations:
  - getYearlyDatasetsUsingGET
  - getDailyDatasetsUsingGET_1
  - getChangesetsUsingGET
  - downloadChangesetsUsingGET
  - getDailyDatasetsUsingGET
generated: '2026-08-07'
method: generated
source: openapi/_original/bitvore-corporate-swagger.json + https://developer.bitvore.com/v2/getting-started
---

# Export Cellenus datasets and reconcile with changesets

For a bulk/offline workload, do not page the News API. Use the export surface. Needs a **Corporate**
dataset key. See `bitvore-authenticate.md`.

## 1. Download a dataset

Exports are compressed files fetched with a plain GET. The response is `application/x-gzip` or
`application/zip`, not JSON.

- Yearly: `getYearlyDatasetsUsingGET` — `GET /v2/corp/datasets/{dataset}/{year}`
- Daily: `getDailyDatasetsUsingGET_1` — `GET /v2/corp/datasets/{dataset}/{year}/{month}/{day}`

```
GET https://api.bitvore.com/v2/corp/datasets/corp-signals/2026?key=<APIKEY>
```

Dataset exports are the one place the **query-parameter key form is intended** — the docs describe
pasting the URL into a browser. Stream the body to disk; do not buffer a yearly export in memory.

Records in the export carry article and source location, and (with an ESG licence) ESG signals.

## 2. Find out what changed

Do not re-download the year to catch a day of edits. Use changesets — added, modified and removed
records, introduced in the v2 release specifically for offline reconciliation.

- List available: `getChangesetsUsingGET` — `GET /v2/corp/changesets`
- Download one: `downloadChangesetsUsingGET` — `GET /v2/corp/changesets/{changeset}`
- A specific day: `getDailyDatasetsUsingGET` — `GET /v2/corp/changesets/{changeset}/{year}/{month}/{day}`

```
GET https://api.bitvore.com/v2/corp/changesets/records-added/2026/04/29?key=<APIKEY>
```

A date range also works via query parameters:

```
GET https://api.bitvore.com/v2/corp/changesets/records-added?startDate=2026-03-11T00:00:00Z&endDate=2026-04-11T00:00:00Z&key=<APIKEY>
```

Changesets cover entity (organization) changes, field changes, and record (news article) changes.

## 3. Apply in order

Apply `records-added`, then `records-modified`, then `records-removed` for the same window before
advancing your watermark. Applying removes before modifies will resurrect deleted rows.

## Rules

- A `204 No Content` on a changeset means nothing changed in that window. That is success, not an
  error — advance the watermark.
- ESG data in exports and changesets **requires a valid ESG licence**. Missing ESG columns is an
  entitlement outcome, not a bug.
- Theme-related values have been empty since 2023-09-01. Do not build reconciliation on them.
- No idempotency contract exists; exports are GETs so retry is safe, but re-applying a changeset to
  your own store is not — track the watermark yourself.
- No published rate limit. Serialize large export pulls rather than fanning out.
