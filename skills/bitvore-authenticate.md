---
name: Authenticate against the Bitvore Cellenus API
description: Obtain and present Bitvore credentials correctly — API key in query or header, or an OAuth 2.0 client-credentials bearer token — before calling any Cellenus endpoint.
api: openapi/_original/bitvore-security-openapi.yml
operations:
  - issueAccessTokenUsingPOST
generated: '2026-08-07'
method: generated
source: https://developer.bitvore.com/v2/docs/security
---

# Authenticate against the Bitvore Cellenus API

Base URL: `https://api.bitvore.com/`

Bitvore is **not self-serve**. There is no signup form that issues a key. A trial key is requested from
`products@bitvore.com`; a licensed key from `sales@bitvore.com`. Do not attempt to register a key
programmatically — no such endpoint exists. If you do not have a key, stop and report that.

The API key also carries product scope. Corporate (`/v2/corp/`) and Municipal (`/v2/muni/`) are
separately licensed and issue **separate keys**. A Corporate key will 401 against a Muni path.

## Choose one of three forms

**1. Query parameter — GET requests and dataset exports only.**

```
GET https://api.bitvore.com/v2/corp/news?bvId=b00001ab7&key=<APIKEY>
```

**2. HTTP header — GET and POST. REQUIRED for POST.**

```
POST https://api.bitvore.com/v2/corp/news
X-BV-APIKEY: <APIKEY>
Content-Type: application/json
```

Prefer this form. Never put the key in a query string on a request you will log or share.

**3. OAuth 2.0 client credentials — so the key is not sent on every call.**

Call `issueAccessTokenUsingPOST`:

```
POST https://api.bitvore.com/oauth/accesstoken
Content-Type: application/x-www-form-urlencoded
Accept: application/json

grant_type=client_credentials&client_id=<REGISTERED_USERNAME>&client_secret=<APIKEY>
```

The `client_secret` **is** the API key. Response is an `AccessTokenGrantResponse`:

```json
{ "access_token": "...", "expires_in": 1800, "token_type": "Bearer" }
```

Present it as a bearer token (RFC 6750):

```
Authorization: Bearer <access_token>
```

## Token handling rules

- **No refresh token is issued** by the client_credentials grant, even though the response schema has a
  `refresh_token` field. Do not build a refresh loop. When the token expires, re-run the grant with the
  client id and API key.
- `expires_in` is in seconds (the documented example is 1800). Re-issue on expiry, or proactively at
  ~80% of the TTL. Cache the token; do not request one per API call.
- The declared oauth2 scope is a single `Global` scope ("Includes all Bitvore APIs"). There is no
  fine-grained scope model — entitlement is enforced by which key you authenticated with.

## Failure handling

Failures come back in the vendor envelope, not RFC 9457:

```json
{ "success": false, "reason": "access_denied", "reasonSupport": "You are not authorized for this resource.", "response": "UNAUTHORIZED" }
```

- `401` with `reason: access_denied` — key is wrong, expired, or licensed for the other product.
  Do not retry with the same key.
- Token-grant failures return an `OAuthError` (`{ error, error_description }`) with `400` or `500`.
- `500` returns `reason: internal_error`. Retry with backoff; there is no idempotency key, so only
  retry idempotent GETs automatically. See `errors/bitvore-problem-types.yml`.
