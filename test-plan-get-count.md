# Test Plan: GET /memberships/count

**Version:** 1.0 · **Date:** 2026-02-21 · **Status:** Tested

**Endpoint:** `GET https://staging.officernd.com/api/v2/organizations/{orgSlug}/memberships/count`
**Auth:** OAuth 2.0 Bearer token · **Required scope:** `flex.community.memberships.read`

> All test cases assume a valid Bearer token with the `flex.community.memberships.read` scope unless the case explicitly tests authentication or authorization.

> **Test execution note:** TC-01-04 and TC-01-05 could not be executed — they require an expired token and a restricted-scope token respectively.

---

## Contents

1. [Scope](#1-scope)
2. [Test Cases](#2-test-cases)
   - [TC-01 — Authentication & Authorization](#tc-01--authentication--authorization)
   - [TC-02 — Path Parameters](#tc-02--path-parameters)
   - [TC-03 — Response Structure](#tc-03--response-structure)
   - [TC-04 — Query Parameters](#tc-04--query-parameters)
   - [TC-05 — Security](#tc-05--security)
   - [TC-06 — HTTP Methods](#tc-06--http-methods)
3. [Bugs Found](#bugs-found)

---

## 1. Scope

| In Scope | Out of Scope |
|---|---|
| Functional correctness of the GET count endpoint | Other membership endpoints |
| Authentication and authorization enforcement | UI layer |
| Response schema validation | Third-party OAuth provider internals |
| Query parameter behaviour | Load / stress testing at infrastructure scale |
| Error response format and HTTP status codes | |
| Basic security checks | |

---

## 2. Test Cases

---

### TC-01 — Authentication & Authorization

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-01-01 | Critical | Org with ≥1 membership | `GET /{orgSlug}/memberships/count` with valid Bearer token | `200 OK`; body: `{"total": <number>}` ✅ |
| TC-01-02 | Critical | — | Send request with no `Authorization` header | `401 Unauthorized`; body: `{"statusCode":401,"message":"Unauthorized access","error":"Unauthorized"}` ✅ |
| TC-01-03 | High | — | Send `Authorization: Bearer not_a_real_token` | `401 Unauthorized`; same error schema as TC-01-02 ✅ |
| TC-01-04 | High | An expired Bearer token | Send request with expired token | `401 Unauthorized`; error indicates expiry; no stack trace *(not executed)* |
| TC-01-05 | Critical | Token without `flex.community.memberships.read` scope | Send request with restricted token | `403 Forbidden`; response identifies the missing permission *(not executed)* |
| TC-01-06 | Critical | `orgSlug` belonging to a different org | `GET /organizations/some-other-org/memberships/count` with valid token | `403 Forbidden` or `404 Not Found` ⚠️ **BUG: returns `500` with `"Organization not found"`** |

---

### TC-02 — Path Parameters

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-02-01 | High | A slug confirmed not to exist | `GET /organizations/does-not-exist-xyz/memberships/count` | `404 Not Found` ⚠️ **BUG: returns `500` with `"Organization not found"`** |
| TC-02-02 | Medium | — | Try `orgSlug` values: `../../etc`, `' OR 1=1`, URL-encoded variants | `302` redirect to `/login`; no data exposed *(consistent with GET list behaviour)* |
| TC-02-03 | Low | — | `GET /organizations//memberships/count` (empty slug) | `401 Unauthorized`; empty segment routes to a different path without org context *(consistent with GET list behaviour)* |

---

### TC-03 — Response Structure

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-03-01 | Critical | Org with ≥1 membership | Send valid request; inspect response body | Body contains exactly one field: `total` (number, non-negative integer); no other fields present ✅ |
| TC-03-02 | High | Org with a known number of memberships | Compare `total` in count response against `rangeEnd` from `GET /memberships` with no `$limit` | Values are equal ✅ *(both returned `30` for the test org)* |
| TC-03-03 | High | Org with zero memberships | `GET /{orgSlug}/memberships/count` on an empty org | `200 OK`; `{"total": 0}` |
| TC-03-04 | Medium | — | Inspect response headers from a valid request | `Content-Type: application/json; charset=utf-8`; rate-limit headers present: `x-ratelimit-limit-read-minute`, `x-ratelimit-remaining-read-minute`, `x-ratelimit-reset-read-minute`, `x-ratelimit-limit-read-day`, `x-ratelimit-remaining-read-day`, `x-ratelimit-reset-read-day` ✅ |

---

### TC-04 — Query Parameters

> ⚠️ **Finding:** The count endpoint silently ignores **all** query parameters — including filter fields (`status`, `calculatedStatus`, `isPersonal`, `type`), pagination controls (`$limit`), field selection (`$select`), and unknown keys. Every request returns the same unfiltered `total` regardless of parameters passed. Test cases below document this behaviour.

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-04-01 | High | — | `GET .../count?unknownParam=value` | `200 OK`; `{"total": <same unfiltered total>}` *(unknown param silently ignored — no `400` as seen on the GET list endpoint)* ⚠️ **BUG: query params are not validated; filters have no effect** |
| TC-04-02 | High | Org with a mix of `approved` and `not_approved` memberships | `GET .../count?status=approved` | `200 OK`; expected filtered count of approved memberships ⚠️ **BUG: param silently ignored; returns full unfiltered total** |
| TC-04-03 | High | Org with memberships in multiple calculated states | `GET .../count?calculatedStatus=active` | `200 OK`; expected count of active memberships ⚠️ **BUG: param silently ignored; returns full unfiltered total** |
| TC-04-04 | High | Org with both personal and company memberships | `GET .../count?isPersonal=true` | `200 OK`; expected count of personal memberships ⚠️ **BUG: param silently ignored; returns full unfiltered total** |
| TC-04-05 | High | Org with both `fixed` and `month_to_month` memberships | `GET .../count?type=fixed` | `200 OK`; expected count of fixed memberships ⚠️ **BUG: param silently ignored; returns full unfiltered total** |
| TC-04-06 | Medium | — | `GET .../count?status=invalid_enum_value` | `400 Bad Request`; invalid enum value rejected ⚠️ **BUG: param silently ignored; returns `200` with full total** |
| TC-04-07 | Medium | — | `GET .../count?$limit=1` | `400 Bad Request` or ignored with note; `$limit` is not applicable to a count endpoint | `200 OK`; `$limit` silently ignored; returns full total *(not a bug — `$limit` does not apply to count)* ✅ |

---

### TC-05 — Security

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-05-01 | Critical | Conditions to trigger `401` and `500` | Trigger each error type; inspect response bodies | No stack traces or internal paths in error bodies ✅; error schema: `{statusCode, message, error?, timestamp, path}` |
| TC-05-02 | High | — | Pass URL-encoded injection strings as query param values: `' OR 1=1 --`, `{"$gt":""}` | `200 OK`; params silently ignored; `total` unchanged; no data leakage ✅ |
| TC-05-03 | Medium | — | Inspect response headers from a valid request | ⚠️ `x-powered-by: Express` header present — reveals server framework |

---

### TC-06 — HTTP Methods

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-06-01 | Medium | — | Send `POST /memberships/count` with valid token | `405 Method Not Allowed` ⚠️ **BUG: returns `404 Not Found` with `"Cannot POST ..."` instead of `405`** |
| TC-06-02 | Low | — | Send `DELETE /memberships/count` with valid token | `405 Method Not Allowed` *(not executed — consistent behaviour expected with TC-06-01)* |

---

## Bugs Found

| TC ID | Severity | Description |
|---|---|---|
| TC-01-06 | Critical | Wrong org returns `500` instead of `403`/`404` |
| TC-02-01 | High | Non-existent `orgSlug` returns `500` instead of `404` |
| TC-04-01 | High | Unknown query params silently ignored instead of rejected with `400` (inconsistent with GET list behaviour) |
| TC-04-02 | High | `status` filter silently ignored; count is always the unfiltered total |
| TC-04-03 | High | `calculatedStatus` filter silently ignored; count is always the unfiltered total |
| TC-04-04 | High | `isPersonal` filter silently ignored; count is always the unfiltered total |
| TC-04-05 | High | `type` filter silently ignored; count is always the unfiltered total |
| TC-04-06 | Medium | Invalid enum value in query param returns `200` instead of `400` |
| TC-05-03 | Low | `x-powered-by: Express` header exposed in all responses |
| TC-06-01 | Low | `POST` on count endpoint returns `404` instead of `405 Method Not Allowed` |
