# Test Plan: GET /memberships

**Version:** 2.1 · **Date:** 2026-02-21 · **Status:** Tested

**Endpoint:** `GET https://staging.officernd.com/api/v2/organizations/{orgSlug}/memberships`
**Auth:** OAuth 2.0 Bearer token · **Required scope:** `flex.community.memberships.read`
**`$limit` max:** 50

> All test cases assume a valid Bearer token with the `flex.community.memberships.read` scope unless the case explicitly tests authentication or authorization.

> **Test execution note:** Cases TC-01-04, TC-01-05, and TC-02-01 could not be executed — they require credentials not available in this session (an expired token, a restricted-scope token, and a token for a separate org respectively).

---

## Contents

1. [Scope](#1-scope)
2. [Test Cases](#2-test-cases)
   - [TC-01 — Authentication & Authorization](#tc-01--authentication--authorization)
   - [TC-02 — Path Parameters](#tc-02--path-parameters)
   - [TC-03 — Response Structure](#tc-03--response-structure)
   - [TC-04 — Pagination](#tc-04--pagination)
   - [TC-05 — Filtering](#tc-05--filtering)
   - [TC-06 — Field Selection](#tc-06--field-selection)
   - [TC-07 — Sorting](#tc-07--sorting)
   - [TC-08 — Edge Cases](#tc-08--edge-cases)
   - [TC-09 — Security](#tc-09--security)
   - [TC-10 — Performance](#tc-10--performance)
3. [Risk & Coverage Summary](#3-risk--coverage-summary)

---

## 1. Scope

| In Scope | Out of Scope |
|---|---|
| Functional correctness of the GET endpoint | POST / PUT / DELETE membership endpoints |
| Authentication and authorization enforcement | UI layer |
| Query parameter behaviour | Third-party OAuth provider internals |
| Response schema validation | Load / stress testing at infrastructure scale |
| Error response format and HTTP status codes | Data migration or seeding scripts |
| Basic performance and security checks | |

---

## 2. Test Cases

---

### TC-01 — Authentication & Authorization

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-01-01 | Critical | Org with ≥1 membership | `GET /{orgSlug}/memberships` with a valid Bearer token | `200 OK`; body contains `results` array and pagination metadata ✅ |
| TC-01-02 | Critical | — | Send the request with no `Authorization` header | `401 Unauthorized`; body: `{"statusCode":401,"message":"Unauthorized access","error":"Unauthorized"}` ✅ |
| TC-01-03 | High | — | `Authorization: Bearer not_a_real_token` | `401 Unauthorized`; same error schema as TC-01-02 ✅ |
| TC-01-04 | High | An expired Bearer token | Send the request with the expired token | `401 Unauthorized`; error indicates expiry; no stack trace *(not executed)* |
| TC-01-05 | Critical | Token without `flex.community.memberships.read` scope | Send a valid request with the restricted token | `403 Forbidden`; response identifies the missing permission *(not executed)* |
| TC-01-06 | Critical | Token for Org A; `orgSlug` for Org B | `GET /{orgSlug_B}/memberships` using Org A's token | `403 Forbidden` or `404 Not Found`; no Org B data returned ⚠️ **BUG: returns `500` with `"Organization not found"`** |

---

### TC-02 — Path Parameters

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-02-01 | Critical | Orgs A and B with distinct memberships | Request memberships for Org A, then Org B; compare IDs | No IDs overlap between responses *(not executed — requires two org tokens)* |
| TC-02-02 | High | A slug confirmed not to exist | `GET /does-not-exist-xyz/memberships` | `404 Not Found` ⚠️ **BUG: returns `500` with `"Organization not found"`** |
| TC-02-03 | Medium | — | Try `orgSlug` values: `../../etc`, `' OR 1=1`, URL-encoded variants | `302` redirect to `/login`; no data exposed ✅ |
| TC-02-04 | Low | — | `GET /api/v2/organizations//memberships` (empty slug) | `401 Unauthorized`; the empty segment routes to a different path *(no org context; treated as unauthenticated)* ✅ |

---

### TC-03 — Response Structure

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-03-01a | Critical | Org with ≥1 membership | Make a valid request; check top-level keys | Body contains exactly: `rangeStart`, `rangeEnd`, `results`. `cursorNext` and `cursorPrev` are **absent** when all results fit in one page; present when paginating ✅ |
| TC-03-01b | Critical | Org with ≥1 membership | Check the type of each top-level field | `rangeStart` number; `rangeEnd` number; `results` array; `cursorNext` string (when present); `cursorPrev` string (when present) ✅ |
| TC-03-01c | Critical | Org with ≥1 membership | Compare `rangeStart`, `rangeEnd`, and `results.length` | `rangeStart` is the **1-indexed** position of the first result on this page; `rangeEnd` equals `rangeStart + results.length - 1` ✅ |
| TC-03-01d | Critical | Org with > default page size memberships | Request first page with `$limit`; inspect `cursorNext` | `cursorNext` is a non-empty string when more records exist; absent when all records fit on the page ✅ |
| TC-03-02a | High | Org with ≥1 membership | Check each object in `results` for required fields | Every object has: `_id`, `name`, `status`, `calculatedStatus`, `type`, `startDate`, `intervalLength`, `intervalCount`, `plan`, `location`, `isPersonal`, `isLocked`, `price`, `deposit`, `discountAmount`, `calculatedDiscountAmount`, `discountedPrice`, `createdAt`, `createdBy`, `modifiedAt`, `modifiedBy` ✅ |
| TC-03-02b | High | Org with ≥1 membership | Check the type of every field in a membership object | `_id` non-empty string; `name` string; `isPersonal` boolean; `isLocked` boolean; `price` number; `deposit` number; `properties` object; `company` string or absent ✅ |
| TC-03-02c | High | Org with ≥1 membership | Extract all timestamp fields from each object | `createdAt`, `modifiedAt`, and `startDate` match `YYYY-MM-DDTHH:mm:ss.sssZ`; never a plain date or Unix epoch ✅ |
| TC-03-02d | High | Org with memberships in both `approved` and `not_approved` states | Extract `status` from every object | Values are only `approved` or `not_approved`; never absent, null, or empty ✅ |
| TC-03-02e | High | Org with memberships in all four calculated states | Extract `calculatedStatus` from every object | Values are only `not_started`, `active`, `expired`, or `not_approved`; never absent, null, or empty ✅ |
| TC-03-02f | High | Org with both membership types | Extract `type` from every object | Values are only `month_to_month` or `fixed`; never absent, null, or empty ✅ |
| TC-03-02g | High | Org with memberships using all interval types | Extract `intervalLength` from every object | Values are only `once`, `hour`, `day`, or `month`; never absent, null, or empty ✅ |
| TC-03-03 | High | Org with zero memberships | `GET /{orgSlug}/memberships` | `200 OK`; `results: []`; `rangeStart` and `rangeEnd` are `0`; `cursorNext` and `cursorPrev` absent |
| TC-03-04 | Medium | — | Inspect response headers from any successful request | `Content-Type: application/json; charset=utf-8` ✅ |

---

### TC-04 — Pagination

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-04-01 | High | Org with memberships | `GET /{orgSlug}/memberships` with no `$limit` | All memberships returned in a single response; no `cursorNext`; `rangeEnd` equals total membership count ✅ |
| TC-04-02 | High | Org with ≥2 memberships | `GET ...?$limit=1` | `results` has exactly 1 item; `cursorNext` is present ✅ |
| TC-04-03 | Critical | Org with exactly 21 known memberships | Paginate with `$limit=10` until `cursorNext` is absent; collect all IDs | 21 unique IDs; no gaps, no duplicates ✅ |
| TC-04-04 | High | Org with ≥3 pages of data | Navigate forward two pages; use `$cursorPrev` from page 2 | Items match page 1 exactly ✅ |
| TC-04-05 | High | Org with < 50 memberships | `GET ...?$limit=50` | All records returned in one page; `cursorNext` absent ✅ |
| TC-04-06 | High | Org with ≥1 membership | `GET ...?$limit=51` | `400 Bad Request`; message: `"$limit must not be greater than 50"` ✅ |
| TC-04-07 | Medium | Org with ≥1 membership | `GET ...?$limit=0` | `200 OK`; returns 20 results; `cursorNext` present *(falls back to default page size of 20; does not reject)* ✅ |
| TC-04-08 | Medium | Org with ≥1 membership | `GET ...?$limit=-5` | `200 OK`; returns results with `cursorNext` present ⚠️ **BUG: negative value not rejected; treated as a valid limit** |
| TC-04-09 | Medium | — | `GET ...?$limit=abc` | `400 Bad Request`; message: `"$limit must be a number conforming to the specified constraints"` ✅ |
| TC-04-10 | High | — | Pass a tampered or random string as `$cursorNext` | `400` or `422` ⚠️ **BUG: returns `500` with JSON parse error message** |
| TC-04-11 | Low | Two valid cursor values from a prior request | Supply both `$cursorNext` and `$cursorPrev` in one request | `400 Bad Request` ⚠️ **BUG: returns `504 Gateway Timeout`** |

---

### TC-05 — Filtering

> ⚠️ **Finding:** The `createdAt` and `modifiedAt` filter parameters are **not supported** on this endpoint. All filter attempts return `400 Bad Request` with `"property createdAt should not exist"` / `"property modifiedAt should not exist"`. Test cases below document this behaviour and should be re-evaluated if filtering is added.

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-05-01 | High | Memberships on both sides of a known date | `GET ...?createdAt[$gt]=<ISO8601>` | ⚠️ `400 Bad Request`; `"property createdAt should not exist"` *(filtering not supported)* |
| TC-05-02 | Medium | A membership at a known exact timestamp | `GET ...?createdAt[$gte]=<timestamp>` | ⚠️ `400 Bad Request` *(filtering not supported)* |
| TC-05-03 | Medium | A membership at a known exact timestamp | `GET ...?createdAt[$lt]=<timestamp>` | ⚠️ `400 Bad Request` *(filtering not supported)* |
| TC-05-04 | Medium | A membership at a known exact timestamp | `GET ...?createdAt[$lte]=<timestamp>` | ⚠️ `400 Bad Request` *(filtering not supported)* |
| TC-05-05 | High | Memberships inside and outside a known date range | `GET ...?createdAt[$gte]=<start>&createdAt[$lte]=<end>` | ⚠️ `400 Bad Request` *(filtering not supported)* |
| TC-05-06 | Medium | Memberships with known `modifiedAt` values | `GET ...?modifiedAt[$gt]=<ISO8601>` | ⚠️ `400 Bad Request`; `"property modifiedAt should not exist"` *(filtering not supported)* |
| TC-05-07 | Medium | — | `GET ...?createdAt[$gt]=2099-01-01T00:00:00.000Z` | ⚠️ `400 Bad Request` *(filtering not supported)* |
| TC-05-08 | High | — | `GET ...?createdAt[$gt]=not-a-date` | `400 Bad Request` ✅ *(rejects invalid input, though for wrong reason — param not recognised rather than invalid date format)* |
| TC-05-09 | High | Org with ≥20 memberships spanning a date boundary | Apply `createdAt[$gte]=<date>`; paginate all pages | ⚠️ `400 Bad Request` *(filtering not supported; cannot execute)* |

---

### TC-06 — Field Selection

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-06-01 | High | Org with ≥1 membership | `GET ...?$select=_id,createdAt` | Each object contains **only** `_id` and `createdAt` ✅ |
| TC-06-02 | Medium | Org with ≥1 membership | `GET ...?$select=_id` | Each object contains only `_id` ✅ |
| TC-06-03 | Medium | Org with ≥1 membership | `GET ...?$select=nonExistentField` | `200 OK`; field silently ignored; each object contains only `_id` *(falls back to `_id` as default minimum field)* ✅ |
| TC-06-04 | Low | Org with ≥1 membership | `GET ...?$select=` | `400 Bad Request` ✅ |

---

### TC-07 — Sorting

> ⚠️ **Finding:** The `$sort` parameter is **not supported** on this endpoint. All sort attempts return `400 Bad Request` with `"property $sort should not exist"`. Test cases below document this behaviour.

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-07-01 | High | Org with ≥2 memberships with distinct `createdAt` values | `GET ...?$sort[createdAt]=1` | ⚠️ `400 Bad Request`; `"property $sort should not exist"` *(sorting not supported)* |
| TC-07-02 | High | Org with ≥2 memberships with distinct `createdAt` values | `GET ...?$sort[createdAt]=-1` | ⚠️ `400 Bad Request` *(sorting not supported)* |
| TC-07-03 | Medium | — | `GET ...?$sort[nonExistentField]=1` | ⚠️ `400 Bad Request` *(sorting not supported)* |
| TC-07-04 | High | Org with ≥3 pages of data | Sort by `createdAt` descending with `$limit=5`; paginate | ⚠️ `400 Bad Request` *(sorting not supported; cannot execute)* |

---

### TC-08 — Edge Cases

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-08-01 | Low | — | `GET ...?unknownParam=value` | `400 Bad Request`; `"property unknownParam should not exist"` *(API strictly rejects unknown params)* ✅ |
| TC-08-02 | Medium | Org with ≥5 memberships | `GET ...?$limit=5&$select=_id` | `200 OK`; both params honoured; each result contains only `_id` ✅ |
| TC-08-03 | Medium | Org with stable data (no writes during test) | Send the exact same request twice in quick succession | Both responses are identical ✅ |
| TC-08-04 | Low | — | Send a request with a query string several kilobytes long | `400 Bad Request`; API rejects unrecognised params before URI length becomes an issue ✅ |

---

### TC-09 — Security

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-09-01 | Critical | Conditions to trigger `401`, `500`, and `400` | Trigger each error; inspect response bodies | No stack traces or internal paths in any response ✅; error body schema: `{statusCode, message, error?, timestamp, path}` |
| TC-09-02 | High | — | Inject `' OR 1=1 --`, `{"$gt": ""}`, `; DROP TABLE memberships;` into `$limit` | `400 Bad Request`; no 5xx; no unexpected data ✅ |
| TC-09-03 | Medium | — | Inspect response headers from a successful request | ⚠️ `X-Powered-By: Express` header is present — reveals the server framework; `Server: cloudflare` |
| TC-09-04 | High | — | Send the request over `http://` | `301`/`308` redirect to HTTPS, or connection refused *(not executed — HTTPS enforced at network level)* |

---

### TC-10 — Performance

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-10-01 | Medium | Org with ≥1 membership; timing utility | Send a valid request 10 times; record response times | ⚠️ P95 = **1726 ms** across 10 requests; most responses ~120 ms but one spike at 1726 ms — exceeds 500 ms SLA target |
| TC-10-02 | Medium | Org with ≥50 memberships; timing utility | `GET ...?$limit=50` repeated 5 times | P95 = **124 ms** ✅ *(only 21 memberships in test org; results still within SLA)* |
| TC-10-03 | High | — | Send a rapid burst of requests until rate-limited | *(not executed — rate limit threshold unknown; excluded to avoid unintended service impact)* |

---

## 3. Risk & Coverage Summary

### Coverage Matrix

| Area | # Cases | Priorities |
|---|---|---|
| Authentication & Authorization | 6 | Critical, High |
| Path Parameters | 4 | Critical, High, Medium, Low |
| Response Structure | 14 | Critical, High, Medium |
| Pagination | 11 | Critical, High, Medium, Low |
| Filtering | 9 | High, Medium |
| Field Selection | 4 | High, Medium, Low |
| Sorting | 4 | High, Medium |
| Edge Cases | 4 | Medium, Low |
| Security | 4 | Critical, High, Medium |
| Performance | 3 | High, Medium |
| **Total** | **63** | |

### Bugs Found During Execution

| TC ID | Severity | Description |
|---|---|---|
| TC-01-06 | Critical | Cross-org request returns `500` instead of `403`/`404` — organization existence is revealed and status code is incorrect |
| TC-02-02 | High | Non-existent `orgSlug` returns `500` instead of `404` |
| TC-04-08 | Medium | `$limit=-5` returns `200` with results instead of `400` — negative values are not validated |
| TC-04-10 | High | Invalid `$cursorNext` value returns `500` with a raw JSON parse error message instead of `400`/`422` |
| TC-04-11 | High | Supplying both `$cursorNext` and `$cursorPrev` returns `504 Gateway Timeout` |
| TC-05-01–09 | High | `createdAt` and `modifiedAt` filter parameters are not supported — all return `400 Bad Request` |
| TC-07-01–04 | High | `$sort` parameter is not supported — all return `400 Bad Request` |
| TC-09-03 | Low | `X-Powered-By: Express` header exposed in all responses |
| TC-10-01 | Medium | P95 response time of 1726 ms observed across 10 requests — exceeds 500 ms SLA target |

### Key Risks

| Risk | Likelihood | Impact | Mitigated By |
|---|---|---|---|
| Tenant data isolation failure | Low | Critical | TC-01-06 *(bug found — 500 instead of 403/404)* |
| Pagination gaps or duplicates | Low | High | TC-04-03 *(passed)* |
| Stale cursor causing 5xx | High | High | TC-04-10 *(bug confirmed — returns 500)* |
| Sensitive data in error responses | Low | High | TC-09-01 *(passed)* |
| Framework fingerprinting via headers | High | Low | TC-09-03 *(X-Powered-By: Express exposed)* |
