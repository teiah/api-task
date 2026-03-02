# Test Plan: GET /memberships/{membershipId}

**Version:** 1.0 · **Date:** 2026-02-21 · **Status:** Tested

**Endpoint:** `GET https://staging.officernd.com/api/v2/organizations/{orgSlug}/memberships/{membershipId}`
**Auth:** OAuth 2.0 Bearer token · **Required scope:** `flex.community.memberships.read`
**Documentation:** https://developer.officernd.com/reference/membershipscontroller_getitem.md

> All test cases assume a valid Bearer token with the `flex.community.memberships.read` scope unless the case explicitly tests authentication or authorization.


---

## Contents

1. [Scope](#1-scope)
2. [Test Cases](#2-test-cases)
   - [TC-01 — Authentication & Authorization](#tc-01--authentication--authorization)
   - [TC-02 — Path Parameters](#tc-02--path-parameters)
   - [TC-03 — Response Structure - Verify MembershipResultDto](#tc-03--response-structure---verify-membershipresultdto)
   - [TC-04 — Query Parameters](#tc-04--query-parameters)

3. [Bugs Found](#bugs-found)

---

## 1. Scope

| In Scope | Out of Scope |
|---|---|
| Functional correctness of the GET single membership endpoint | Other membership endpoints |
| Authentication and authorization enforcement | UI layer |
| Response schema validation | Third-party OAuth provider internals |
| Path parameter validation | Load / stress testing at infrastructure scale |

---

## 2. Test Cases

---

### TC-01 — Authentication & Authorization

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-01-01 | Critical | A valid `membershipId` | `GET /{orgSlug}/memberships/{membershipId}` with valid Bearer token | `200 OK`; body is a single membership object containing `_id` equal to the requested ID ✅ |
| TC-01-02 | Critical | — | Send request with no `Authorization` header | `401 Unauthorized`; body: `{"statusCode":401,"message":"Unauthorized access","error":"Unauthorized"}` ✅ |
| TC-01-03 | High | — | Send `Authorization: Bearer not_a_real_token` | `401 Unauthorized`; same error schema as TC-01-02 ✅ |
| TC-01-04 | High | An expired Bearer token | Send request with expired token | `401 Unauthorized`; same generic error schema as TC-01-02; no expiry-specific message ✅ |
| TC-01-05 | Critical | Token without `flex.community.memberships.read` scope | Send request with restricted token | `403 Forbidden`; response identifies the missing permission ⚠️ **BUG: returns `401 Unauthorized` instead of `403 Forbidden`** |
| TC-01-06 | Critical | `orgSlug` belonging to a different org | `GET /organizations/some-other-org/memberships/{membershipId}` with valid token | `403 Forbidden` or `404 Not Found` ⚠️ **BUG: returns `500` with `"Organization not found"`** |

---

### TC-02 — Path Parameters

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-02-01 | Critical | A valid ObjectId that does not belong to any membership | `GET .../memberships/aaaaaaaaaaaaaaaaaaaaaaaa` | `404 Not Found`; body: `{"statusCode":404,"message":"Item with Id (aaaaaaaaaaaaaaaaaaaaaaaa) not found","error":"Not Found"}` ✅ *(returned 404; error message truncated to `"Item with Id (aaaaaaaaaaaaaaaaaaaaaaaa)"` — missing "not found" suffix)* |
| TC-02-02 | Medium | — | `GET /organizations/does-not-exist-xyz/memberships/{membershipId}` | `404 Not Found` ⚠️ **BUG: returns `500` with `"Organization not found"`** |

---

### TC-03 — Response Structure - Verify MembershipResultDto

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-03-01a | Critical | A valid `membershipId` | Send valid request; inspect top-level response shape | Body is a flat JSON object (not an array and not wrapped in a `results` key); `_id` equals the requested `membershipId` ✅ |
| TC-03-01b | Critical | A valid `membershipId` | Check presence of all required fields | All of the following are present: `_id`, `name`, `status`, `calculatedStatus`, `type`, `startDate`, `intervalLength`, `intervalCount`, `plan`, `location`, `isPersonal`, `isLocked`, `price`, `deposit`, `discountAmount`, `calculatedDiscountAmount`, `discountedPrice`, `createdAt`, `createdBy`, `modifiedAt`, `modifiedBy` ✅ |
| TC-03-01c | High | A valid `membershipId` | Inspect each field's JavaScript type | Required: `_id` string (ObjectId); `name` string; `isPersonal` boolean; `isLocked` boolean; `status` string; `calculatedStatus` string; `type` string; `startDate` string (ISO 8601); `intervalLength` string; `intervalCount` number; `plan` string (ObjectId); `location` string (ObjectId); `price` number; `deposit` number; `discountAmount` number; `calculatedDiscountAmount` number; `discountedPrice` number; `createdAt` string (ISO 8601); `createdBy` string (ObjectId); `modifiedAt` string (ISO 8601); `modifiedBy` string (ObjectId). Optional when present: `properties` object; `company` string (ObjectId); `member` string (ObjectId); `endDate` string (ISO 8601); `discount` string (ObjectId); `contract` string; `source` string ✅ |
| TC-03-01d | High | A valid `membershipId` with `type=fixed`; separately one with `type=month_to_month` | Extract `createdAt`, `modifiedAt`, `startDate`, and `endDate` (when present) | All match `YYYY-MM-DDTHH:mm:ss.sssZ`; never a plain date or Unix epoch ✅ |
| TC-03-01e | High | A membership with `status=approved`; separately one with `status=not_approved` | Extract `status` from each response | Value is exactly `approved` or `not_approved`; never absent, null, or empty ✅ |
| TC-03-01f | High | Memberships in each of the four `calculatedStatus` states; request each separately | Extract `calculatedStatus` | Value is exactly `not_started`, `active`, `expired`, or `not_approved`; never absent, null, or empty ✅ |
| TC-03-01g | High | A membership with `type=month_to_month`; separately one with `type=fixed` | Extract `type` | Value is exactly `month_to_month` or `fixed`; never absent, null, or empty ✅ |
| TC-03-01h | High | Memberships covering each `intervalLength` value; request each separately | Extract `intervalLength` | Value is exactly `once`, `hour`, `day`, or `month`; never absent, null, or empty ✅ |
| TC-03-02 | High | A membership where `isPersonal` is `false` | Verify `company` field | `company` (string, ObjectId) is present; `member` is absent ✅ |
| TC-03-03 | High | A membership where `isPersonal` is `true` | Verify `member` field | `member` (string, ObjectId) is present; `company` is absent ✅ |
| TC-03-04a | High | A membership with `type` = `fixed` | Verify `endDate` field | `endDate` (string, ISO 8601) is present and after `startDate` *(not executed — no fixed-type membership available with a non-null endDate; a membership with `endDate: null` was observed)* |
| TC-03-04b | High | A membership with `type` = `month_to_month` | Verify `endDate` field | `endDate` is absent from the response ✅ |
| TC-03-05 | Medium | A membership with a non-empty `properties` object | Verify `properties` field | `properties` (object) is present and contains the expected key-value pairs ✅ *(confirmed: `{"key1": "value1"}` returned correctly)* |
| TC-03-06 | Medium | A membership with an empty `properties` object | Verify `properties` field when empty | `properties` is present as `{}` ⚠️ **BUG: `properties` is absent in the single-item response when empty, but always present as `{}` in the GET list response — inconsistency between endpoints** |
| TC-03-07 | Medium | — | Inspect response headers | `Content-Type: application/json; charset=utf-8` ✅ |
| TC-03-08 | Medium | A membership with a discount definition applied | Verify `discount` field | `discount` (string, ObjectId) is present and references the discount definition *(not executed — requires a membership with a linked discount)* |
| TC-03-09 | Medium | A membership associated with a contract | Verify `contract` field | `contract` (string) is present *(not executed — requires a membership with a linked contract)* |
| TC-03-10 | Low | A membership with a known `source` value | Verify `source` field | `source` (string) is present *(not executed — requires a membership created via a channel that sets source)* |

---

### TC-04 — Query Parameters

> ⚠️ **Finding:** No query parameters are documented for this endpoint. In testing, unknown parameters and `$select` were both silently ignored — the full membership object was returned regardless. This is inconsistent with the GET list endpoint, which strictly rejects unknown parameters with `400`.

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-04-01 | Medium | A valid `membershipId` | `GET .../memberships/{id}?unknownParam=value` | `400 Bad Request`; unrecognised parameter rejected ⚠️ **BUG: silently ignored; full membership returned — inconsistent with GET list** |
| TC-04-02 | Low | A valid `membershipId` | `GET .../memberships/{id}?$select=_id,name` | `400 Bad Request`; `$select` not supported on this endpoint ⚠️ **BUG: silently ignored; full membership returned** |

---

## Bugs Found

| TC ID | Severity | Description |
|---|---|---|
| TC-01-04 | Low | Expired token returns the same generic `401 Unauthorized` error as an invalid token — response does not indicate expiry; clients cannot distinguish the cause |
| TC-01-05 | High | Token with missing scope returns `401 Unauthorized` instead of `403 Forbidden` — `401` implies unauthenticated; the token is valid but lacks authorisation |
| TC-01-06 | Critical | Wrong org returns `500` instead of `403`/`404` |
| TC-02-01 | Low | `404` error message for a non-existent ID is truncated: `"Item with Id (xxx)"` — missing the "not found" suffix |
| TC-02-02 | High | Non-existent `orgSlug` returns `500` instead of `404` |
| TC-03-06 | Medium | `properties` field absent in single-item response when empty (`{}`); always present in GET list response — inconsistency between endpoints |
| TC-04-01 | Low | Unknown query params silently ignored instead of rejected with `400` — inconsistent with GET list |
| TC-04-02 | Low | `$select` silently ignored; field selection not supported on single-item endpoint |

