# Test Plan: GET /memberships/{membershipId}

**Version:** 1.0 · **Date:** 2026-02-21 · **Status:** Tested

**Endpoint:** `GET https://staging.officernd.com/api/v2/organizations/{orgSlug}/memberships/{membershipId}`
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
   - [TC-05 — Consistency](#tc-05--consistency)
   - [TC-06 — Security](#tc-06--security)
   - [TC-07 — HTTP Methods](#tc-07--http-methods)
3. [Bugs Found](#bugs-found)

---

## 1. Scope

| In Scope | Out of Scope |
|---|---|
| Functional correctness of the GET single membership endpoint | Other membership endpoints |
| Authentication and authorization enforcement | UI layer |
| Response schema validation | Third-party OAuth provider internals |
| Path parameter validation | Load / stress testing at infrastructure scale |
| Response consistency with the GET list endpoint | |
| Basic security checks | |

---

## 2. Test Cases

---

### TC-01 — Authentication & Authorization

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-01-01 | Critical | A valid `membershipId` | `GET /{orgSlug}/memberships/{membershipId}` with valid Bearer token | `200 OK`; body is a single membership object containing `_id` equal to the requested ID ✅ |
| TC-01-02 | Critical | — | Send request with no `Authorization` header | `401 Unauthorized`; body: `{"statusCode":401,"message":"Unauthorized access","error":"Unauthorized"}` ✅ |
| TC-01-03 | High | — | Send `Authorization: Bearer not_a_real_token` | `401 Unauthorized`; same error schema as TC-01-02 ✅ |
| TC-01-04 | High | An expired Bearer token | Send request with expired token | `401 Unauthorized`; error indicates expiry; no stack trace *(not executed)* |
| TC-01-05 | Critical | Token without `flex.community.memberships.read` scope | Send request with restricted token | `403 Forbidden`; response identifies the missing permission *(not executed)* |
| TC-01-06 | Critical | `orgSlug` belonging to a different org | `GET /organizations/some-other-org/memberships/{membershipId}` with valid token | `403 Forbidden` or `404 Not Found` ⚠️ **BUG: returns `500` with `"Organization not found"`** |

---

### TC-02 — Path Parameters

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-02-01 | Critical | A valid ObjectId that does not belong to any membership | `GET .../memberships/aaaaaaaaaaaaaaaaaaaaaaaa` | `404 Not Found`; body: `{"statusCode":404,"message":"Item with Id (aaaaaaaaaaaaaaaaaaaaaaaa) not found","error":"Not Found"}` ✅ *(returned 404; error message truncated to `"Item with Id (aaaaaaaaaaaaaaaaaaaaaaaa)"` — missing "not found" suffix)* |
| TC-02-02 | High | — | `GET .../memberships/notanid` (non-ObjectId string) | `400 Bad Request`; invalid ID format rejected ⚠️ **BUG: returns `404 Not Found` — non-ObjectId string treated as a lookup miss rather than a validation error** |
| TC-02-03 | High | — | `GET .../memberships/../../etc` or other path traversal strings | `302` redirect to `/login` or `400 Bad Request`; no data exposed *(not executed — consistent redirect behaviour expected based on GET list results)* |
| TC-02-04 | Medium | — | `GET /organizations/does-not-exist-xyz/memberships/{membershipId}` | `404 Not Found` ⚠️ **BUG: returns `500` with `"Organization not found"`** |
| TC-02-05 | Medium | — | Omit `membershipId` entirely: `GET .../memberships/` | Routes to the GET list endpoint; `200 OK` with paginated results ✅ *(expected routing behaviour — handled by the list handler)* |

---

### TC-03 — Response Structure

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-03-01 | Critical | A valid `membershipId` | Send valid request; verify response is a direct object (not wrapped in an array or `results` key) and check all fields and types | Body is a flat object containing: `_id` (string, ObjectId, equals requested ID); `name` (string); `isPersonal` (boolean); `status` (string: `approved` or `not_approved`); `calculatedStatus` (string: `not_started`, `active`, `expired`, or `not_approved`); `type` (string: `month_to_month` or `fixed`); `location` (string, ObjectId); `plan` (string, ObjectId); `startDate` (string, ISO 8601); `intervalLength` (string: `once`, `hour`, `day`, or `month`); `intervalCount` (number); `isLocked` (boolean); `price` (number); `discountAmount` (number); `calculatedDiscountAmount` (number); `discountedPrice` (number); `createdAt` (string, ISO 8601); `createdBy` (string, ObjectId); `modifiedAt` (string, ISO 8601); `modifiedBy` (string, ObjectId) ✅ |
| TC-03-02 | High | A membership where `isPersonal` is `false` | Verify `company` field | `company` (string, ObjectId) is present; `member` is absent ✅ |
| TC-03-03 | High | A membership where `isPersonal` is `true` | Verify `member` field | `member` (string, ObjectId) is present; `company` is absent ✅ |
| TC-03-04 | High | A membership with `type` = `fixed` | Verify `endDate` field | `endDate` (string, ISO 8601) is present and after `startDate` *(not executed — no fixed-type membership available with a non-null endDate; a membership with `endDate: null` was observed)* |
| TC-03-05 | Medium | A membership with a non-empty `properties` object | Verify `properties` field | `properties` (object) is present and contains the expected key-value pairs ✅ *(confirmed: `{"key1": "value1"}` returned correctly)* |
| TC-03-06 | Medium | A membership with an empty `properties` object | Verify `properties` field when empty | `properties` is present as `{}` ⚠️ **BUG: `properties` is absent in the single-item response when empty, but always present as `{}` in the GET list response — inconsistency between endpoints** |
| TC-03-07 | Medium | A membership with a `deposit` value | Verify `deposit` field | `deposit` (number) is present ✅ |
| TC-03-08 | Medium | — | Inspect response headers | `Content-Type: application/json; charset=utf-8` ✅ |

---

### TC-04 — Query Parameters

> ⚠️ **Finding:** No query parameters are documented for this endpoint. In testing, unknown parameters and `$select` were both silently ignored — the full membership object was returned regardless. This is inconsistent with the GET list endpoint, which strictly rejects unknown parameters with `400`.

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-04-01 | Medium | A valid `membershipId` | `GET .../memberships/{id}?unknownParam=value` | `400 Bad Request`; unrecognised parameter rejected ⚠️ **BUG: silently ignored; full membership returned — inconsistent with GET list** |
| TC-04-02 | Low | A valid `membershipId` | `GET .../memberships/{id}?$select=_id,name` | `400 Bad Request`; `$select` not supported on this endpoint ⚠️ **BUG: silently ignored; full membership returned** |

---

### TC-05 — Consistency

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-05-01 | High | A `membershipId` present in both list and single-item responses | Fetch `GET /memberships` (no `$limit`) and `GET /memberships/{id}`; compare the object for the same `_id` field by field | All field values are identical between the two responses ✅; exception: `properties` is absent in single-item when empty but present as `{}` in list — see TC-03-06 ⚠️ |
| TC-05-02 | Medium | — | Send the same valid request twice in quick succession | Both responses are identical (same field values, same `modifiedAt`) ✅ |

---

### TC-06 — Security

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-06-01 | Critical | Conditions to trigger `401`, `404`, and `500` | Trigger each error type; inspect response bodies | No stack traces or internal paths in any error body ✅; error schema: `{statusCode, message, error?, timestamp, path}` |
| TC-06-02 | Medium | — | Inspect response headers from a valid request | ⚠️ `x-powered-by: Express` header present — reveals server framework |

---

### TC-07 — HTTP Methods

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-07-01 | Medium | — | Send `POST /memberships/{id}` with valid token | `405 Method Not Allowed` ⚠️ **BUG: returns `404 Not Found` with `"Cannot POST ..."` instead of `405`** |

---

## Bugs Found

| TC ID | Severity | Description |
|---|---|---|
| TC-01-06 | Critical | Wrong org returns `500` instead of `403`/`404` |
| TC-02-02 | Medium | Non-ObjectId `membershipId` returns `404` instead of `400` — invalid format not caught at the validation layer |
| TC-02-01 | Low | `404` error message for a non-existent ID is truncated: `"Item with Id (xxx)"` — missing the "not found" suffix |
| TC-02-04 | High | Non-existent `orgSlug` returns `500` instead of `404` |
| TC-03-06 | Medium | `properties` field absent in single-item response when empty (`{}`); always present in GET list response — inconsistency between endpoints |
| TC-04-01 | Low | Unknown query params silently ignored instead of rejected with `400` — inconsistent with GET list |
| TC-04-02 | Low | `$select` silently ignored; field selection not supported on single-item endpoint |
| TC-06-02 | Low | `x-powered-by: Express` header exposed in all responses |
| TC-07-01 | Low | `POST` on single-item path returns `404` instead of `405 Method Not Allowed` |
