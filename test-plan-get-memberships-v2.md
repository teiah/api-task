# Test Plan: GET /api/v2/organizations/{orgSlug}/memberships

**Version:** 2.0 · **Date:** 2026-02-21 · **Status:** Draft

---

**Endpoint**
```
GET https://app.officernd.com/api/v2/organizations/{orgSlug}/memberships
```
**Auth:** OAuth 2.0 Bearer token · **Required scope:** `flex.community.memberships.read`

---

## Table of Contents

1. [Overview](#1-overview)
2. [Scope](#2-scope)
3. [Test Objectives](#3-test-objectives)
4. [Test Environment](#4-test-environment)
5. [Test Cases](#5-test-cases)
   - [TC-01 — Authentication & Authorization](#tc-01--authentication--authorization)
   - [TC-02 — Path Parameters](#tc-02--path-parameters)
   - [TC-03 — Response Structure](#tc-03--response-structure)
   - [TC-04 — Pagination](#tc-04--pagination)
   - [TC-05 — Filtering](#tc-05--filtering)
   - [TC-06 — Field Selection](#tc-06--field-selection-select)
   - [TC-07 — Sorting](#tc-07--sorting-sort)
   - [TC-08 — Edge Cases & Boundary Values](#tc-08--edge-cases--boundary-values)
   - [TC-09 — Security](#tc-09--security)
   - [TC-10 — Performance](#tc-10--performance)
6. [Risk & Coverage Summary](#6-risk--coverage-summary)

---

## 1. Overview

This document describes the test plan for the **Get Memberships** endpoint of the OfficeRnD Flex API. The endpoint retrieves a paginated list of membership records belonging to an organization.

---

## 2. Scope

| In Scope | Out of Scope |
|---|---|
| Functional correctness of the GET endpoint | POST / PUT / DELETE membership endpoints |
| Authentication and authorization enforcement | UI layer |
| Query parameter behaviour (pagination, filtering, sorting, field selection) | Third-party OAuth provider internals |
| Response schema validation | Load / stress testing at infrastructure scale |
| Error response format and HTTP status codes | Data migration or seeding scripts |
| Basic performance and security checks | |

---

## 3. Test Objectives

- Verify the endpoint returns correct data for a valid, authenticated request.
- Verify that authentication failures are handled correctly and consistently.
- Verify that path parameter validation rejects invalid or missing `orgSlug` values.
- Verify pagination cursors behave correctly and produce consistent, non-overlapping pages.
- Verify each supported query parameter (`$limit`, `$cursorNext`, `$cursorPrev`, `$select`, `$sort`, `createdAt`, `modifiedAt`) works independently and in combination.
- Verify that malformed, unexpected, or boundary-value inputs produce appropriate error responses.
- Verify the response schema matches the documented contract.
- Verify no sensitive data is leaked in responses or error messages.

---

## 4. Test Environment

| Environment | Purpose |
|---|---|
| **Development / Sandbox** | Primary environment for functional and negative test cases |
| **Staging** | Regression and performance checks against production-like data |

**Tools:** REST client (e.g. Postman, `curl`, or pytest + `httpx`) · JSON Schema validator · Response-time monitoring utility

---

## 5. Test Cases

---

### TC-01 — Authentication & Authorization

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-01-01 | Critical | Valid Bearer token with `flex.community.memberships.read` scope; `orgSlug` for an org with ≥1 membership | `GET /api/v2/organizations/{orgSlug}/memberships` with `Authorization: Bearer <valid_token>` | `200 OK`; response body contains `results` array and pagination metadata |
| TC-01-02 | Critical | None | Send the request with no `Authorization` header | `401 Unauthorized`; body includes a machine-readable error code and human-readable message |
| TC-01-03 | High | None | Send `Authorization: Bearer not_a_real_token` | `401 Unauthorized`; error body matches TC-01-02 schema |
| TC-01-04 | High | A correctly-formed but expired Bearer token | Send the request with the expired token | `401 Unauthorized`; error indicates expiry; no stack trace |
| TC-01-05 | Critical | Valid Bearer token that does **not** include `flex.community.memberships.read` | Send a valid request using the restricted token | `403 Forbidden`; response identifies the missing permission *(flag as bug if `401` is returned)* |
| TC-01-06 | Critical | Valid token for Org A; known `orgSlug` for Org B | `GET /api/v2/organizations/{orgSlug_B}/memberships` using Org A's token | `403 Forbidden` or `404 Not Found`; zero Org B memberships returned under any circumstance *(flag as bug if `401` is returned)* |

---

### TC-02 — Path Parameters

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-02-01 | Critical | Two organizations (A and B) each with distinct memberships; a valid token scoped to each | Request memberships for Org A; note returned IDs. Request memberships for Org B; note returned IDs. | No IDs appear in both responses; results are fully isolated by org |
| TC-02-02 | High | A slug confirmed not to exist in the system; valid Bearer token | `GET /api/v2/organizations/does-not-exist-xyz/memberships` | `404 Not Found`; descriptive error body *(flag as bug if `500` is returned)* |
| TC-02-03 | Medium | Valid Bearer token | Try `orgSlug` values containing `../../etc`, `' OR 1=1`, and URL-encoded variants | `400 Bad Request` or `404 Not Found`; no 500; no stack trace or data in response |
| TC-02-04 | Low | Valid Bearer token | `GET /api/v2/organizations//memberships` (empty slug segment) | `400 Bad Request` or `404 Not Found` *(flag as bug if `500` is returned)* |

---

### TC-03 — Response Structure

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-03-01a | Critical | Valid Bearer token; `orgSlug` for an org with ≥1 membership | Make a valid request; inspect the top-level keys of the response body | Response contains **exactly** these top-level keys: `rangeStart`, `rangeEnd`, `cursorNext`, `cursorPrev`, `results`. No undocumented keys are present at the top level |
| TC-03-01b | Critical | Valid Bearer token; `orgSlug` for an org with ≥1 membership | Inspect the type of each top-level field in the response | `rangeStart` is a number; `rangeEnd` is a number; `cursorNext` is a string or `null`; `cursorPrev` is a string or `null`; `results` is an array |
| TC-03-01c | Critical | Valid Bearer token; `orgSlug` for an org with ≥1 membership | Compare `rangeStart`, `rangeEnd`, and `results` against each other | `rangeStart` is `0`; `rangeEnd` equals `results.length - 1`; `rangeEnd >= rangeStart` |
| TC-03-01d | Critical | Valid Bearer token; `orgSlug` for an org with more memberships than the default page size | Request the first page; check `cursorNext` | `cursorNext` is a non-empty string when more records exist beyond the current page; `cursorNext` is `null` or absent when the page contains all remaining records |
| TC-03-02a | High | Valid Bearer token; `orgSlug` for an org with ≥1 membership | Inspect each object in `results`; check for required field presence | Every membership object contains all required fields: `_id` (string), `name` (string), `status` (string), `createdAt` (string), `modifiedAt` (string) |
| TC-03-02b | High | Valid Bearer token; `orgSlug` for an org with ≥1 membership | Inspect the type of every field in a membership object | `_id` is a non-empty string; `name` is a string; `description` is a string or absent; `startDate` is a string or absent; `status` is a string; `properties` is an object or absent; `createdAt` is a string; `modifiedAt` is a string |
| TC-03-02c | High | Valid Bearer token; `orgSlug` for an org with ≥1 membership | Extract `createdAt`, `modifiedAt`, and `startDate` (where present) from each membership object; validate each value | Every timestamp field matches ISO 8601 format (`YYYY-MM-DDTHH:mm:ss.sssZ`); no timestamp is a plain date string, a Unix epoch integer, or `null` |
| TC-03-02d | High | Valid Bearer token; org with memberships in both `approved` and `not_approved` states | Extract the `status` field from every object in `results` | Every value is one of `approved`, `not_approved`; field is never absent, `null`, or an empty string |
| TC-03-02e | High | Valid Bearer token; org with memberships covering all calculated states (not yet started, currently active, expired, and not approved) | Extract the `calculatedStatus` field from every object in `results` | Every value is one of `not_started`, `active`, `expired`, `not_approved`; field is never absent, `null`, or an empty string |
| TC-03-02f | High | Valid Bearer token; org with both `month_to_month` and `fixed` memberships | Extract the `type` field from every object in `results` | Every value is one of `month_to_month`, `fixed`; field is never absent, `null`, or an empty string |
| TC-03-02g | High | Valid Bearer token; org with memberships using each interval type (`once`, `hour`, `day`, `month`) | Extract the `intervalLength` field from every object in `results` | Every value is one of `once`, `hour`, `day`, `month`; field is never absent, `null`, or an empty string |
| TC-03-03 | High | Valid Bearer token; `orgSlug` for an org confirmed to have zero memberships | Request memberships for that org | `200 OK`; `results: []`; `rangeStart` and `rangeEnd` are `0`; `cursorNext` and `cursorPrev` are `null` or absent |
| TC-03-04 | Medium | Valid Bearer token; any valid `orgSlug` | Inspect the response headers of any successful request | `Content-Type: application/json` (with optional `; charset=utf-8`) |

---

### TC-04 — Pagination

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-04-01 | High | Valid Bearer token; org with more memberships than the documented default page size | `GET /memberships` without `$limit`; record `results.length` | Count equals the documented default limit; `cursorNext` is populated |
| TC-04-02 | High | Valid Bearer token; org with ≥2 memberships | `GET ...?$limit=1` | `results` has exactly 1 item; `cursorNext` is present |
| TC-04-03 | Critical | Valid Bearer token; org with exactly 25 known memberships | Paginate all pages with `$limit=10`; collect every returned ID until `cursorNext` is null | Total unique IDs = 25; no duplicates; no missing IDs |
| TC-04-04 | High | Valid Bearer token; org with ≥3 pages of data | Navigate forward two pages; use `$cursorPrev` from page 2 to go back | Items returned match page 1 exactly |
| TC-04-05 | Medium | Valid Bearer token; org with fewer memberships than the maximum allowed limit | Set `$limit` to a number greater than the total membership count | All records returned in `results`; `cursorNext` is `null` or absent |
| TC-04-06 | Medium | Valid Bearer token; any valid `orgSlug` | `GET ...?$limit=0` | `400 Bad Request`, or default page size returned *(document observed behaviour)* |
| TC-04-07 | Medium | Valid Bearer token; any valid `orgSlug` | `GET ...?$limit=-5` | `400 Bad Request`; no data returned |
| TC-04-08 | Medium | Valid Bearer token; any valid `orgSlug` | `GET ...?$limit=abc` | `400 Bad Request`; error identifies the invalid parameter type |
| TC-04-09 | High | Valid Bearer token; any valid `orgSlug` | Pass a randomly-generated or tampered string as `$cursorNext` | `400 Bad Request` or `422 Unprocessable Entity`; no 5xx |
| TC-04-10 | Low | Valid Bearer token; two known valid cursor values from a prior paginated request | Include both `$cursorNext` and `$cursorPrev` in one request | `400 Bad Request`, or one parameter takes precedence *(document observed behaviour)* |

---

### TC-05 — Filtering

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-05-01 | High | Valid Bearer token; memberships exist on both sides of a known date boundary | `GET ...?createdAt[$gt]=<ISO8601_date>` | All returned memberships have `createdAt` strictly after the given date; memberships on or before are excluded |
| TC-05-02 | Medium | Valid Bearer token; a membership created at a known exact timestamp | `GET ...?createdAt[$gte]=<that_exact_timestamp>` | The boundary membership is **present** in `results` |
| TC-05-03 | Medium | Valid Bearer token; a membership created at a known exact timestamp | `GET ...?createdAt[$lt]=<that_exact_timestamp>` | The boundary membership is **absent** from `results` |
| TC-05-04 | Medium | Valid Bearer token; a membership created at a known exact timestamp | `GET ...?createdAt[$lte]=<that_exact_timestamp>` | The boundary membership is **present** in `results` |
| TC-05-05 | High | Valid Bearer token; memberships both inside and outside a known date range | `GET ...?createdAt[$gte]=<start>&createdAt[$lte]=<end>` | All returned memberships fall within [start, end]; memberships outside the range are absent |
| TC-05-06 | Medium | Valid Bearer token; memberships with known `modifiedAt` timestamps on both sides of a boundary | `GET ...?modifiedAt[$gt]=<ISO8601_date>` | All returned memberships have `modifiedAt` strictly after the given date |
| TC-05-07 | Medium | Valid Bearer token; any valid `orgSlug` | `GET ...?createdAt[$gt]=2099-01-01T00:00:00.000Z` | `200 OK`; `results: []` |
| TC-05-08 | High | Valid Bearer token; any valid `orgSlug` | `GET ...?createdAt[$gt]=not-a-date` | `400 Bad Request`; error identifies the invalid date format |
| TC-05-09 | High | Valid Bearer token; org with ≥20 memberships spanning a known date boundary | Apply `createdAt[$gte]=<date>` and paginate all pages using `$cursorNext`; collect every returned ID | No items outside the date range appear on any page; no duplicate or missing IDs within the filtered set |

---

### TC-06 — Field Selection (`$select`)

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-06-01 | High | Valid Bearer token; org with ≥1 membership | `GET ...?$select=_id,createdAt` | Each object in `results` contains **only** `_id` and `createdAt`; no other fields present |
| TC-06-02 | Medium | Valid Bearer token; org with ≥1 membership | `GET ...?$select=_id` | Each result object contains only `_id` |
| TC-06-03 | Medium | Valid Bearer token; org with ≥1 membership | `GET ...?$select=nonExistentField` | `400 Bad Request`, or field silently ignored *(document observed behaviour)* |
| TC-06-04 | Low | Valid Bearer token; org with ≥1 membership | `GET ...?$select=` | `400 Bad Request`, or parameter ignored and full objects returned *(document observed behaviour)* |

---

### TC-07 — Sorting (`$sort`)

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-07-01 | High | Valid Bearer token; org with ≥2 memberships with distinct `createdAt` timestamps | `GET ...?$sort[createdAt]=1` | `createdAt` values in `results` are in non-decreasing order |
| TC-07-02 | High | Valid Bearer token; org with ≥2 memberships with distinct `createdAt` timestamps | `GET ...?$sort[createdAt]=-1` | `createdAt` values in `results` are in non-increasing order |
| TC-07-03 | Medium | Valid Bearer token; any valid `orgSlug` | `GET ...?$sort[nonExistentField]=1` | `400 Bad Request`, or parameter silently ignored *(document observed behaviour)* |
| TC-07-04 | High | Valid Bearer token; org with ≥3 pages of data | Sort descending by `createdAt` with `$limit=5`; paginate all pages; collect all `createdAt` values in sequence | The combined sequence across all pages is strictly non-increasing; ordering does not reset between pages |

---

### TC-08 — Edge Cases & Boundary Values

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-08-01 | Low | Valid Bearer token; any valid `orgSlug` | `GET ...?unknownParam=value` | `200 OK` with normal results, or `400 Bad Request` *(document observed behaviour)* |
| TC-08-02 | Medium | Valid Bearer token; any valid `orgSlug` | `GET ...?$limit=999999999` | Results capped at the maximum allowed value, or `400 Bad Request` with a max-limit error |
| TC-08-03 | Medium | Valid Bearer token; org with ≥5 memberships spanning a date range | `GET ...?$limit=5&$select=_id&$sort[createdAt]=-1&createdAt[$gte]=<date>&modifiedAt[$lte]=<date>` | `200 OK`; all parameters honoured simultaneously; no parameter silently overrides another |
| TC-08-04 | Medium | Valid Bearer token; org with stable membership data (no writes during the test) | Send the exact same GET request twice in quick succession | Both responses are identical (same IDs, same order, same metadata) |
| TC-08-05 | Low | Valid Bearer token; any valid `orgSlug` | Construct a query string several kilobytes long (e.g. many repeated unknown params) and send the request | `400 Bad Request` or `414 URI Too Long`; no 5xx |

---

### TC-09 — Security

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-09-01 | Critical | Tokens and conditions sufficient to trigger `401`, `403`, `404`, and `400` responses | Trigger each error type; inspect all response bodies and headers | No stack traces, internal file paths, database identifiers, or secret tokens present in any error response |
| TC-09-02 | High | Valid Bearer token; any valid `orgSlug` | Inject `' OR 1=1 --`, `{"$gt": ""}`, and `; DROP TABLE memberships;` into query parameter values | `400 Bad Request`; no 5xx; no unexpected data in the response body |
| TC-09-03 | Medium | Valid Bearer token; any valid `orgSlug` | Inspect all response headers from a successful request | No `X-Powered-By` header; no `Server` header exposing version numbers or stack details |
| TC-09-04 | High | None | Send the request using `http://` instead of `https://` | `301` or `308` redirect to HTTPS, or connection refused; request is never served over plain HTTP |

---

### TC-10 — Performance

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-10-01 | Medium | Valid Bearer token; org with ≥1 membership; response-time monitoring utility | Send a valid request 10 times; record response times | P95 < 500 ms *(adjust threshold to agreed SLA)* |
| TC-10-02 | Medium | Valid Bearer token; org with memberships ≥ the maximum page size; response-time monitoring utility | Send a request with `$limit` set to the maximum allowed value, repeated 5 times | P95 within the agreed SLA; no timeout errors |
| TC-10-03 | High | Valid Bearer token; knowledge of the rate-limit threshold | Send a rapid burst of requests until a rate limit response is received | `429 Too Many Requests`; `Retry-After` header or equivalent guidance present in response |

---

## 6. Risk & Coverage Summary

### Coverage Matrix

| Area | # Cases | Priority Coverage |
|---|---|---|
| Authentication & Authorization | 6 | Critical, High |
| Path Parameters | 4 | Critical, High, Medium, Low |
| Response Structure | 4 | Critical, High, Medium |
| Pagination | 10 | Critical, High, Medium, Low |
| Filtering | 9 | High, Medium |
| Field Selection | 4 | High, Medium, Low |
| Sorting | 4 | High, Medium |
| Edge Cases | 5 | Medium, Low |
| Security | 4 | Critical, High, Medium |
| Performance | 3 | High, Medium |
| **Total** | **53** | |

### Key Risks

| Risk | Likelihood | Impact | Mitigated By |
|---|---|---|---|
| Tenant data isolation failure | Low | Critical | TC-01-06 |
| Pagination gaps or duplicates | Medium | High | TC-04-03, TC-05-09 |
| Inconsistent filter boundary behaviour | Medium | Medium | TC-05-02, TC-05-03, TC-05-04 |
| Injection via query parameters or path | Low | Critical | TC-09-02, TC-02-03 |
| Stale cursor producing a 5xx | Medium | High | TC-04-09 |
| Sensitive data in error messages | Low | High | TC-09-01 |

### Known Issues (Flagged for Verification)

| TC ID | Issue |
|---|---|
| TC-01-05 | Original expected `401`; corrected to `403 Forbidden` — an authenticated but insufficiently-scoped request should be forbidden, not treated as unauthenticated |
| TC-01-06 | Original expected `401`; corrected to `403 Forbidden` or `404 Not Found` — cross-tenant access is an authorization failure, not an authentication failure |
| TC-02-02 | Original expected `500`; corrected to `404 Not Found` — a non-existent slug is a client error; flag as a bug if `500` is observed |
| TC-02-04 | Original expected `500`; corrected to `400` or `404` — same reasoning as TC-02-02 |
