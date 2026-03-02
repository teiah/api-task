# Test Plan: GET /memberships

**Version:** 2.1 · **Date:** 2026-02-21 · **Status:** Tested

**Endpoint:** `GET https://staging.officernd.com/api/v2/organizations/{orgSlug}/memberships`
**Auth:** OAuth 2.0 Bearer token · **Required scope:** `flex.community.memberships.read`
**`$limit` max:** 50

> All test cases assume a valid Bearer token with the `flex.community.memberships.read` scope unless the case explicitly tests authentication or authorization.


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
   - [TC-07 — Edge Cases](#tc-07--edge-cases)
3. [Bugs Found](#bugs-found)

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
| TC-01-04 | High | An expired Bearer token | Send the request with the expired token | `401 Unauthorized`; same generic error schema as TC-01-02; no expiry-specific message ✅ |
| TC-01-05 | Critical | Token without `flex.community.memberships.read` scope | Send a valid request with the restricted token | `403 Forbidden`; response identifies the missing permission ⚠️ **BUG: returns `401 Unauthorized` instead of `403 Forbidden`** |
| TC-01-06 | Critical | Token for Org A; `orgSlug` for Org B | `GET /{orgSlug_B}/memberships` using Org A's token | `403 Forbidden` or `404 Not Found`; no Org B data returned ⚠️ **BUG: returns `500` with `"Organization not found"`** |

---

### TC-02 — Path Parameters

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-02-01 | High | A slug confirmed not to exist | `GET /does-not-exist-xyz/memberships` | `404 Not Found` ⚠️ **BUG: returns `500` with `"Organization not found"`** |

---

### TC-03 — Response Structure

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-03-01a | Critical | Org with ≥1 membership | Make a valid request; check top-level keys | Body contains exactly: `rangeStart`, `rangeEnd`, `results`. `cursorNext` and `cursorPrev` are **absent** when all results fit in one page; present when paginating ✅ |
| TC-03-01b | Critical | Org with ≥1 membership | Check the type of each top-level field | `rangeStart` number; `rangeEnd` number; `results` array; `cursorNext` string (when present); `cursorPrev` string (when present) ✅ |
| TC-03-01c | Critical | Org with ≥1 membership | Compare `rangeStart`, `rangeEnd`, and `results.length` | `rangeStart` is the **1-indexed** position of the first result on this page; `rangeEnd` equals `rangeStart + results.length - 1` ✅ |
| TC-03-01d | Critical | Org with > default page size memberships | Request first page with `$limit`; inspect `cursorNext` | `cursorNext` is a non-empty string when more records exist; absent when all records fit on the page ✅ |
| TC-03-02 | High | Org with ≥1 membership covering all enum states and conditional field variants | Apply MembershipResultDto verification to each object in `results` — see [GET single test plan TC-03-01b – TC-03-10](test-plan-get-membership.md#tc-03--response-structure---verify-membershipresultdto) | Every item passes all field-presence, type, timestamp-format, enum, and conditional-field checks ✅ |
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
| TC-04-08 | Medium | Org with ≥1 membership | `GET ...?$limit=-5` | `400 Bad Request` ⚠️ **BUG: returns `200` with 3 results and `cursorNext` — result count does not match `abs(-5)=5`; negative value not rejected** |
| TC-04-09 | Medium | — | `GET ...?$limit=abc` | `400 Bad Request`; message: `"$limit must be a number conforming to the specified constraints"` ✅ |
| TC-04-10 | High | — | Pass a tampered or random string as `$cursorNext` | `400` or `422` ⚠️ **BUG: returns `500` with JSON parse error message** |
| TC-04-11 | Low | Two valid cursor values from a prior request | Supply both `$cursorNext` and `$cursorPrev` in one request | `400 Bad Request` ⚠️ **BUG: returns `504 Gateway Timeout`** |
| TC-04-12 | Critical | — | `GET ...?$limit=-2` | `400 Bad Request` ⚠️ **BUG: returns `500 Internal Server Error`; `"Cannot read properties of undefined (reading '_id')"` — server crash** |
| TC-04-13 | High | Org with ≥1 membership | `GET ...?$limit=-1` | `400 Bad Request` ⚠️ **BUG: returns `200` with all results; no pagination cap applied — behaves as "no limit"** |

---

### TC-05 — Filtering

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-05-01 | High | Org with memberships linked to ≥2 distinct companies; known target `companyId` | `GET ...?company=<companyId>` | `200 OK`; every result has `company` equal to `<companyId>` and `isPersonal=false` |
| TC-05-02 | High | Org with ≥1 membership | `GET ...?company=aaaaaaaaaaaaaaaaaaaaaaaa` | `200 OK`; `results: []` |
| TC-05-03 | Medium | Org with personal memberships linked to ≥2 distinct members; known target `memberId` | `GET ...?member=<memberId>` | `200 OK`; every result has `member` equal to `<memberId>` and `isPersonal=true` |
| TC-05-04 | Medium | Org with ≥1 membership | `GET ...?member=aaaaaaaaaaaaaaaaaaaaaaaa` | `200 OK`; `results: []` |
| TC-05-05 | Medium | Org with memberships for a known company and a known member | `GET ...?company=<companyId>&member=<memberId>` | `200 OK`; results contain only memberships matching both filters *(document observed behaviour if AND/OR semantics are unclear)* |

---

### TC-06 — Field Selection

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-06-01 | High | Org with ≥1 membership | `GET ...?$select=_id,createdAt` | Each object contains **only** `_id` and `createdAt` ✅ |
| TC-06-02 | Medium | Org with ≥1 membership | `GET ...?$select=_id` | Each object contains only `_id` ✅ |
| TC-06-03 | Medium | Org with ≥1 membership | `GET ...?$select=nonExistentField` | `200 OK`; field silently ignored; each object contains only `_id` *(falls back to `_id` as default minimum field)* ✅ |
| TC-06-04 | Low | Org with ≥1 membership | `GET ...?$select=` | `400 Bad Request` ✅ |

---

### TC-07 — Edge Cases

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-07-01 | Low | — | `GET ...?unknownParam=value` | `400 Bad Request`; `"property unknownParam should not exist"` *(API strictly rejects any undocumented param)* ✅ |
| TC-07-02 | Medium | Org with ≥5 memberships | `GET ...?$limit=5&$select=_id` | `200 OK`; both params honoured; each result contains only `_id` ✅ |

---

## Bugs Found

| TC ID | Severity | Description |
|---|---|---|
| TC-01-04 | Low | Expired token returns the same generic `401 Unauthorized` error as an invalid token — response does not indicate expiry; clients cannot distinguish the cause |
| TC-01-05 | High | Token with missing scope returns `401 Unauthorized` instead of `403 Forbidden` — `401` implies unauthenticated; the token is valid but lacks authorisation |
| TC-01-06 | Critical | Cross-org request returns `500` instead of `403`/`404` — organization existence is revealed and status code is incorrect |
| TC-02-01 | High | Non-existent `orgSlug` returns `500` instead of `404` |
| TC-04-08 | Medium | `$limit=-5` returns `200` with 3 results instead of `400` — result count doesn't match `abs(-5)=5`; negative values not validated |
| TC-04-12 | Critical | `$limit=-2` causes `500 Internal Server Error` — `"Cannot read properties of undefined (reading '_id')"` — server crash on negative limit |
| TC-04-13 | High | `$limit=-1` returns `200` with all results instead of `400` — no pagination cap applied; behaves as "no limit" |
| TC-04-10 | High | Invalid `$cursorNext` value returns `500` with a raw JSON parse error message instead of `400`/`422` |
| TC-04-11 | High | Supplying both `$cursorNext` and `$cursorPrev` returns `504 Gateway Timeout` |

