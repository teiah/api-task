# Test Plan: POST /memberships

**Version:** 1.0 · **Date:** 2026-02-21 · **Status:** Tested

**Endpoint:** `POST https://staging.officernd.com/api/v2/organizations/{orgSlug}/memberships`
**Auth:** OAuth 2.0 Bearer token · **Required scope:** `flex.community.memberships.create`

> All test cases assume a valid Bearer token with the `flex.community.memberships.create` scope and `Content-Type: application/json` unless the case explicitly tests authentication or headers.

> **Test execution note:** TC-01-04 and TC-01-05 could not be executed — they require an expired token and a restricted-scope token respectively.

---

## Contents

1. [Scope](#1-scope)
2. [Request Body Reference](#2-request-body-reference)
3. [Test Cases](#3-test-cases)
   - [TC-01 — Authentication & Authorization](#tc-01--authentication--authorization)
   - [TC-02 — Path Parameters](#tc-02--path-parameters)
   - [TC-03 — Required Fields](#tc-03--required-fields)
   - [TC-04 — Optional Fields](#tc-04--optional-fields)
   - [TC-05 — Field Validation](#tc-05--field-validation)
   - [TC-06 — Response Structure](#tc-06--response-structure)
   - [TC-07 — Edge Cases](#tc-07--edge-cases)
   - [TC-08 — Security](#tc-08--security)
4. [Risk & Coverage Summary](#4-risk--coverage-summary)

---

## 1. Scope

| In Scope | Out of Scope |
|---|---|
| Functional correctness of the POST endpoint | GET / PUT / DELETE membership endpoints |
| Authentication and authorization enforcement | UI layer |
| Request body validation (required fields, types, enums, formats) | Third-party OAuth provider internals |
| Response schema validation | Load / stress testing at infrastructure scale |
| Error response format and HTTP status codes | Data seeding or teardown automation |
| Basic security checks | |

---

## 2. Request Body Reference

### Required Fields

| Field | Type | Notes |
|---|---|---|
| `name` | string | Non-empty |
| `startDate` | string | ISO 8601 format (`YYYY-MM-DDTHH:mm:ss.sssZ`) |
| `location` | string | Valid ObjectId; must reference an existing location |
| `plan` | string | Valid ObjectId; must reference an existing plan |
| `company` | string | Valid ObjectId; required when `isPersonal` is `false` or absent |
| `member` | string | Valid ObjectId; required when `isPersonal` is `true` |

### Optional Fields

| Field | Type | Enum / Constraints |
|---|---|---|
| `isPersonal` | boolean | Default: `false` |
| `type` | string | `month_to_month`, `fixed` |
| `endDate` | string | ISO 8601; required when `type` is `fixed`; must be after `startDate` |
| `intervalLength` | string | `once`, `hour`, `day`, `month` |
| `properties` | object | Free-form key-value pairs |

### Server-Managed Fields (not accepted in request body)

`_id`, `status`, `calculatedStatus`, `intervalCount`, `isLocked`, `price`, `discountAmount`, `calculatedDiscountAmount`, `createdAt`, `createdBy`, `modifiedAt`, `modifiedBy`

---

## 3. Test Cases

---

### TC-01 — Authentication & Authorization

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-01-01 | Critical | Valid request body | `POST /memberships` with valid Bearer token and minimal valid body | `201 Created`; body contains `_id` and all expected fields ✅ |
| TC-01-02 | Critical | — | Send request with no `Authorization` header | `401 Unauthorized`; body: `{"statusCode":401,"message":"Unauthorized access","error":"Unauthorized"}` ✅ |
| TC-01-03 | High | — | Send `Authorization: Bearer badtoken` | `401 Unauthorized`; same error schema as TC-01-02 ✅ |
| TC-01-04 | High | An expired Bearer token | Send request with expired token | `401 Unauthorized`; error indicates expiry; no stack trace *(not executed)* |
| TC-01-05 | Critical | Token without `flex.community.memberships.create` scope | Send valid body with restricted token | `403 Forbidden`; response identifies missing permission *(not executed)* |
| TC-01-06 | Critical | Valid token; `orgSlug` belonging to a different org | `POST /organizations/some-other-org/memberships` with valid body | `403 Forbidden` or `404 Not Found` ⚠️ **BUG: returns `500` with `"Organization not found"`** |

---

### TC-02 — Path Parameters

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-02-01 | High | A slug confirmed not to exist | `POST /organizations/does-not-exist/memberships` with valid body | `404 Not Found` *(not executed separately — consistent with GET behaviour which returns `500`)* |
| TC-02-02 | Medium | — | `POST /organizations//memberships` (empty slug) | `401 Unauthorized` *(routes to a different path without org context)* |

---

### TC-03 — Required Fields

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-03-01 | Critical | — | Send body with all five required fields: `name`, `startDate`, `location`, `plan`, `company` | `201 Created`; `_id` present in response ✅ |
| TC-03-02 | Critical | — | Omit `name` | `400 Bad Request`; message: `"name must be a string"` ✅ |
| TC-03-03 | Critical | — | Omit `startDate` | `400 Bad Request`; message: `"startDate must be a valid ISO 8601 date string"` ✅ |
| TC-03-04 | Critical | — | Omit `location` | `400 Bad Request`; message: `"location must be a string"` ✅ |
| TC-03-05 | Critical | — | Omit `plan` | `400 Bad Request`; message: `"plan must be a string"` ✅ |
| TC-03-06 | Critical | — | Omit `company` when `isPersonal` is `false` or absent | `400 Bad Request`; message: `"A company field must be provided if the isPersonal field is set to false."` ✅ |
| TC-03-07 | Critical | — | Omit `member` when `isPersonal` is `true` | `400 Bad Request`; message: `"A member field must be provided if the isPersonal field is set to true."` ✅ |
| TC-03-08 | Critical | — | Send completely empty body `{}` | `400 Bad Request`; all required field errors returned in a single `message` array ✅ |

---

### TC-04 — Optional Fields

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-04-01 | High | — | Send valid body without any optional fields | `201 Created`; server applies defaults: `isPersonal=false`, `type=month_to_month`, `intervalLength=month`, `intervalCount=1`, `isLocked=false`, `price=0` ✅ |
| TC-04-02 | High | — | Include `"type": "fixed"` and a valid `endDate` after `startDate` | `201 Created`; `type=fixed`; `endDate` matches value sent ✅ |
| TC-04-03 | High | — | Include `"type": "fixed"` without `endDate` | `400 Bad Request`; message: `"End date is required for fixed memberships"` ✅ |
| TC-04-04 | Medium | — | Include `"intervalLength": "once"` | `201 Created`; `intervalLength=once` in response ✅ |
| TC-04-05 | Medium | — | Include `"intervalLength": "hour"` | `201 Created`; `intervalLength=hour` in response |
| TC-04-06 | Medium | — | Include `"intervalLength": "day"` | `201 Created`; `intervalLength=day` in response |
| TC-04-07 | Medium | — | Include `"endDate"` set before `startDate` | `400 Bad Request` ⚠️ **BUG: returns `500` with `"Membership startDate should be before the endDate"`** |
| TC-04-08 | Medium | — | Include `"properties": {"key1": "value1"}` | `201 Created`; `properties` in response equals `{"key1": "value1"}` ✅ |

---

### TC-05 — Field Validation

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-05-01 | High | — | Set `name` to an empty string `""` | `400 Bad Request` ⚠️ **BUG: returns `500` with `"Path 'name' is required."` — should be `400`** |
| TC-05-02 | High | — | Set `name` to a number (e.g. `123`) | `400 Bad Request`; message: `"name must be a string"` ✅ |
| TC-05-03 | High | — | Set `startDate` to `"21-02-2026"` (not ISO 8601) | `400 Bad Request`; message: `"startDate must be a valid ISO 8601 date string"` ✅ |
| TC-05-04 | High | — | Set `location` to a non-ObjectId string | `500 Internal Server Error` ⚠️ **BUG: returns `500` with BSON cast error — should be `400`** |
| TC-05-05 | High | — | Set `plan` to a non-ObjectId string | `500 Internal Server Error` ⚠️ **BUG: returns `500` with `"Specified plan was not found"` — should be `400`** |
| TC-05-06 | High | — | Set `plan` to a valid ObjectId format that does not exist | `500 Internal Server Error` ⚠️ **BUG: returns `500` with `"Specified plan was not found"` — should be `404`** |
| TC-05-07 | High | — | Set `location` to a valid ObjectId format that does not exist | `500 Internal Server Error` ⚠️ **BUG: returns `500` with `"Specified office was not found"` — should be `404`** |
| TC-05-08 | High | — | Set `intervalLength` to `"weekly"` (invalid enum) | `400 Bad Request`; message: `"intervalLength must be one of the following values: once, hour, day, month"` ✅ |
| TC-05-09 | High | — | Set `type` to `"daily"` (invalid enum) | `400 Bad Request`; message: `"type must be one of the following values: month_to_month, fixed"` ✅ |
| TC-05-10 | Medium | — | Include `"status"` field in body (server-managed field) | `400 Bad Request`; message: `"property status should not exist"` ✅ |
| TC-05-11 | Medium | — | Include any unknown field (e.g. `"unknownField": "value"`) | `400 Bad Request`; message: `"property unknownField should not exist"` ✅ |
| TC-05-12 | Medium | — | Send request without `Content-Type: application/json` header | `400 Bad Request`; body is not parsed as JSON — field-level validation errors returned ✅ |

---

### TC-06 — Response Structure

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-06-01 | Critical | Valid request body | Send a valid POST; inspect all top-level keys in the `201` response | Response contains: `_id`, `name`, `isPersonal`, `status`, `calculatedStatus`, `type`, `location`, `plan`, `company`, `startDate`, `intervalLength`, `intervalCount`, `isLocked`, `price`, `discountAmount`, `calculatedDiscountAmount`, `createdAt`, `createdBy`, `modifiedAt`, `modifiedBy` ✅ |
| TC-06-02 | Critical | Valid request body | Check field types in the `201` response | `_id` non-empty string; `name` string; `isPersonal` boolean; `isLocked` boolean; `price` number; `createdAt` and `modifiedAt` ISO 8601 strings ✅ |
| TC-06-03 | High | Valid request body | Check `status` and `calculatedStatus` in the `201` response | `status` is `approved`; `calculatedStatus` is `active` for a membership with a past `startDate`; neither field is null or absent ✅ |
| TC-06-04 | High | Valid request body | Check that `createdAt` and `modifiedAt` are equal and close to the request time | Both timestamps match and fall within a few seconds of the POST request time ✅ |
| TC-06-05 | Medium | Valid request body | Inspect response headers | `Content-Type: application/json; charset=utf-8` ✅ |

---

### TC-07 — Edge Cases

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-07-01 | High | — | Send two identical POST requests in quick succession | Two separate `201` responses each with a distinct `_id` — POST is **not idempotent** ✅ |
| TC-07-02 | Medium | — | Send a very long string as `name` (e.g. 10 000 characters) | `201 Created` or `400 Bad Request` with a max-length error *(document observed behaviour)* |
| TC-07-03 | Medium | — | Send `startDate` with a past date (e.g. `2018-01-01`) | `201 Created`; `calculatedStatus` reflects actual state based on dates ✅ *(endDate=2027 test confirmed past startDate is accepted)* |

---

### TC-08 — Security

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-08-01 | Critical | Conditions to trigger `401`, `500`, and `400` | Trigger each error type; inspect response bodies | No stack traces or internal paths in error bodies ✅; error schema: `{statusCode, message, error?, timestamp, path}` |
| TC-08-02 | High | — | Set `name` to an injection string: `"' OR 1=1 --"` | `201 Created`; value stored as a plain string; no query manipulation or data leakage ✅ |
| TC-08-03 | Medium | — | Inspect response headers from a successful `201` | ⚠️ `X-Powered-By: Express` header present — reveals server framework |

---

## 4. Risk & Coverage Summary

### Coverage Matrix

| Area | # Cases | Priorities |
|---|---|---|
| Authentication & Authorization | 6 | Critical, High |
| Path Parameters | 2 | High, Medium |
| Required Fields | 8 | Critical |
| Optional Fields | 8 | High, Medium |
| Field Validation | 12 | High, Medium |
| Response Structure | 5 | Critical, High, Medium |
| Edge Cases | 3 | High, Medium |
| Security | 3 | Critical, High, Medium |
| **Total** | **47** | |

### Bugs Found During Execution

| TC ID | Severity | Description |
|---|---|---|
| TC-01-06 | Critical | Wrong org returns `500` instead of `403`/`404` |
| TC-04-07 | High | `endDate` before `startDate` returns `500` instead of `400`/`422` |
| TC-05-01 | High | Empty `name` string returns `500` instead of `400` |
| TC-05-04 | High | Non-ObjectId `location` string returns `500` BSON cast error instead of `400` |
| TC-05-05 | High | Non-ObjectId `plan` string returns `500` instead of `400` |
| TC-05-06 | High | Non-existent `plan` ObjectId returns `500` instead of `404` |
| TC-05-07 | High | Non-existent `location` ObjectId returns `500` instead of `404` |
| TC-08-03 | Low | `X-Powered-By: Express` header exposed in all responses |

### Key Risks

| Risk | Likelihood | Impact | Mitigated By |
|---|---|---|---|
| Unauthenticated membership creation | Low | Critical | TC-01-02, TC-01-03 |
| Cross-org membership creation | Low | Critical | TC-01-06 *(bug — returns 500)* |
| Invalid reference IDs causing 500s | High | High | TC-05-04 through TC-05-07 *(bugs confirmed)* |
| Duplicate memberships created unintentionally | Medium | Medium | TC-07-01 *(POST is not idempotent — expected)* |
| Sensitive data in error responses | Low | High | TC-08-01 *(passed)* |
