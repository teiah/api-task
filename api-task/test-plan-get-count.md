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
   - [TC-05 — HTTP Methods](#tc-05--http-methods)
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

---

### TC-03 — Response Structure

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-03-01 | Critical | Org with ≥1 membership | Send valid request; inspect top-level response fields and types | `200 OK`; body contains `total` (number, required, non-negative integer); body may contain `groups` (array of objects, each with `key` (string, required) and `count` (number, required)); no other fields present ✅ *(`groups` absent when no grouping parameter is in effect)* |
| TC-03-02 | High | Org with ≥1 membership | Verify the `total` field value against `GET /memberships` with no `$limit` | `total` equals `rangeEnd` from the list response ✅ *(both returned `30` for the test org)* |
| TC-03-03 | High | Org with zero memberships | `GET /{orgSlug}/memberships/count` on an empty org | `200 OK`; `total` is `0`; `groups` absent or empty array |
| TC-03-04 | High | Org with memberships in multiple states | Send a request that triggers grouping; verify each object in `groups` | Each object in `groups` contains exactly: `key` (string, non-empty) and `count` (number, non-negative integer); sum of all `count` values equals `total` *(not executed — could not determine the parameter that triggers `groups`)* |
| TC-03-05 | Medium | — | Inspect response headers from a valid request | `Content-Type: application/json; charset=utf-8`; rate-limit headers present: `x-ratelimit-limit-read-minute`, `x-ratelimit-remaining-read-minute`, `x-ratelimit-reset-read-minute`, `x-ratelimit-limit-read-day`, `x-ratelimit-remaining-read-day`, `x-ratelimit-reset-read-day` ✅ |

---

### TC-04 — Query Parameters

> **Finding — filters:** The [OfficeRnD query documentation](https://developer.officernd.com/docs/building-queries) defines basic equality filtering as `fieldName=value` (e.g. `?status=approved`). TC-04-02 through TC-04-05 use this correct format. The endpoint returns the same unfiltered `total` for every filter value, including contradictory combinations, which means the filters are silently ignored. This is a bug regardless of intent: if filtering is supported, the result should be filtered; if it is not supported, the endpoint should reject the parameter with `400` rather than silently accept it.
>
> **Finding — inapplicable params:** `$limit` and `$select` are pagination/projection controls that have no meaning on a count endpoint. Silently ignoring them is acceptable behaviour.

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-04-01 | High | — | `GET .../count?unknownParam=value` | `400 Bad Request`; unrecognised parameter rejected ⚠️ **BUG: silently ignored; returns `200` with full unfiltered total — inconsistent with GET list which rejects unknown params** |
| TC-04-02 | High | Org with a mix of `approved` and `not_approved` memberships | `GET .../count?status=approved` (valid equality filter per API query docs) | `200 OK`; `total` equals the count of `approved` memberships only ⚠️ **BUG: filter silently ignored; returns full unfiltered total regardless of value** |
| TC-04-03 | High | Org with memberships in multiple calculated states | `GET .../count?calculatedStatus=active` (valid equality filter per API query docs) | `200 OK`; `total` equals the count of `active` memberships only ⚠️ **BUG: filter silently ignored; returns full unfiltered total regardless of value** |
| TC-04-04 | High | Org with both personal and company memberships | `GET .../count?isPersonal=true` (valid equality filter per API query docs) | `200 OK`; `total` equals the count of personal memberships only ⚠️ **BUG: filter silently ignored; returns full unfiltered total regardless of value** |
| TC-04-05 | High | Org with both `fixed` and `month_to_month` memberships | `GET .../count?type=fixed` (valid equality filter per API query docs) | `200 OK`; `total` equals the count of `fixed` memberships only ⚠️ **BUG: filter silently ignored; returns full unfiltered total regardless of value** |
| TC-04-06 | Medium | — | `GET .../count?status=invalid_enum_value` | `400 Bad Request`; invalid enum value rejected ⚠️ **BUG: silently ignored; returns `200` with full total** |
| TC-04-07 | Medium | — | `GET .../count?$limit=1` | `200 OK`; `$limit` silently ignored; full total returned *(acceptable — `$limit` is a pagination control and does not apply to a count endpoint)* ✅ |

---

### TC-05 — HTTP Methods

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-05-01 | Medium | — | Send `POST /memberships/count` with valid token | `405 Method Not Allowed` ⚠️ **BUG: returns `404 Not Found` with `"Cannot POST ..."` instead of `405`** |
| TC-05-02 | Low | — | Send `DELETE /memberships/count` with valid token | `405 Method Not Allowed` *(not executed — consistent behaviour expected with TC-05-01)* |

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
| TC-05-01 | Low | `POST` on count endpoint returns `404` instead of `405 Method Not Allowed` |
