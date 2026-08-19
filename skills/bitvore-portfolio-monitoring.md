---
name: Load a portfolio into Bitvore and monitor it
description: Create a Cellenus portfolio (organizations or CUSIPs), load its members by API or CSV upload, then query news matched against that portfolio.
api: openapi/_original/bitvore-corporate-openapi.yml
operations:
  - handleAddPortfolioUsingPOST_1
  - handleUploadPortfolioItemsUsingPOST_1
  - handleAddPortfolioItemsUsingPOST_1
  - handleGetPortfolioItemsUsingGET_1
  - handlePortfolioNewsSearchUsingGET
  - handleGetPortfoliosUsingGET_1
  - handleModifyPortfolioUsingPUT_1
  - handleDeletePortfolioUsingDELETE_1
  - handleAddPortfolioUsingPOST_2
  - handlePortfolioMuniSearchUsingGET
generated: '2026-08-07'
method: generated
source: openapi/_original/bitvore-corporate-swagger.json + openapi/_original/bitvore-muni-swagger.json + https://developer.bitvore.com/v2/getting-started
---

# Load a portfolio into Bitvore and monitor it

A portfolio is the primary way to run Cellenus against a book you care about instead of the whole
universe. There are two parallel portfolio surfaces and they are **not interchangeable**:

| Book | Path | Create op | idScheme |
|---|---|---|---|
| Companies | `/v2/corp/portfolios` | `handleAddPortfolioUsingPOST_1` | `bvid`, or a custom foreign-id scheme |
| Municipal bonds | `/v2/muni/portfolios` | `handleAddPortfolioUsingPOST_2` | `cusip` |

Each needs the key for its own licensed product. Authenticate first — see `bitvore-authenticate.md`.

## 1. Create the portfolio

```
POST https://api.bitvore.com/v2/corp/portfolios
X-BV-APIKEY: <APIKEY>
Content-Type: application/json

{ "name": "SP500Portfolio", "idScheme": "bvid" }
```

The response envelope returns the generated portfolio id you need for every subsequent call:

```json
{ "success": true, "count": 1, "response": [ { "id": "123", "name": "SP500Portfolio", "idScheme": "bvid" } ] }
```

Capture `response[0].id`. Do not assume it.

## 2. Load the members

Two ways, pick by size.

**Bulk CSV upload** — `handleUploadPortfolioItemsUsingPOST_1`,
`POST /v2/corp/portfolios/{portfolioId}/itemUploads`, `multipart/form-data`, field `file`:

```
curl -X POST -F "file=@sp500.csv" \
  -H "Content-Type: multipart/form-data" -H "X-BV-APIKey: <APIKEY>" \
  "https://api.bitvore.com/v2/corp/portfolios/123/itemUploads"
```

Sample portfolios (S&P 500 and others) are published at
https://developer.bitvore.com/v2/docs/api-reference/portfolios.

**Incremental add** — `handleAddPortfolioItemsUsingPOST_1`,
`POST /v2/corp/portfolios/{portfolioId}/items` with a JSON array of items. The Muni equivalent takes
CUSIPs directly: `POST /v2/muni/portfolios/{portfolioId}/items` with `[{ "id": "00037CSW1" }]`.

To add or replace by identifier value rather than internal item id, use the `/items/ids` variants
(`handleAddPortfolioItemsByValuesUsingPOST`, `handleReplacePortfolioItemsByValuesUsingPUT`).

Verify with `handleGetPortfolioItemsUsingGET_1` — `GET /v2/corp/portfolios/{portfolioId}/items` —
before relying on the portfolio. A create that succeeded does not mean the upload landed.

## 3. Carry your own identifiers

Since the 2021-10-01 release, custom foreign-id → bvId mappings live **on the portfolio** via
`idScheme`. This replaces the account-wide v1 Id Map API (`/idapi`), which allowed only one mapping
per account. Set your own scheme at creation and each portfolio carries its own mapping — no second
resolution call per company.

## 4. Query news for the portfolio

Corporate: `handlePortfolioNewsSearchUsingGET` — `GET /v2/corp/news/byportfolio?portfolioId=123`
(or `GET /v2/corp/news?portfolioId=123`).

Municipal: `handlePortfolioMuniSearchUsingGET` — `GET /v2/muni/news/byportfolio?portfolioId=123`.

`startDate` and `endDate` are required. Page with `pageNo`/`pageSize`; stop when accumulated
`returned` reaches `total`.

## 5. Maintain

- List: `handleGetPortfoliosUsingGET_1` — `GET /v2/corp/portfolios`
- Rename: `handleModifyPortfolioUsingPUT_1` — `PUT /v2/corp/portfolios/{portfolioId}`
- Replace the whole membership: `handleReplacePortfolioItemsUsingPUT_1` — `PUT .../items`
- Remove one: `handleDeletePortfolioItemsUsingDELETE_3` — `DELETE .../items/{id}`
- Delete the portfolio: `handleDeletePortfolioUsingDELETE_1` — `DELETE /v2/corp/portfolios/{portfolioId}`

## Rules

- **Writes are not idempotent.** Bitvore publishes no idempotency key. A retried `POST /portfolios`
  creates a second portfolio. On a timeout, `GET /v2/corp/portfolios` and reconcile by name before
  retrying.
- `404` with `reason` "Portfolio with given Id not found" means the id is wrong or belongs to the
  other product surface (corp id used against muni, or vice versa).
- `PUT .../items` **replaces** the membership. It does not merge.
