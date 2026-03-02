# Test Plan: PUT /memberships/{membershipId}

**Version:** 1.0 · **Date:** 2026-02-21 · **Status:** Tested

**Endpoint:** `PUT https://staging.officernd.com/api/v2/organizations/{orgSlug}/memberships/{membershipId}`
**Auth:** OAuth 2.0 Bearer token · **Required scope:** `flex.community.memberships.update`
**Documentation:** https://developer.officernd.com/reference/membershipscontroller_updateitem.md

> All test cases assume a valid Bearer token with the `flex.community.memberships.update` scope unless the case explicitly tests authentication or authorization.


> **Accepted fields:** Only `startDate`, `endDate`, and `price` are accepted in the request body. Every other field — including `name`, `type`, `plan`, `location`, `company`, `isPersonal`, `intervalLength`, `properties`, `status`, and all server-managed fields — returns `400 "property X should not exist"`. An empty body `{}` is accepted as a no-op.

---

## Contents

1. [Scope](#1-scope)
2. [Test Cases](#2-test-cases)
   - [TC-01 — Authentication & Authorization](#tc-01--authentication--authorization)
   - [TC-02 — Path Parameters](#tc-02--path-parameters)
   - [TC-03 — Request Body: Accepted Fields](#tc-03--request-body-accepted-fields)
   - [TC-04 — Request Body: Rejected Fields](#tc-04--request-body-rejected-fields)
   - [TC-05 — Response Structure](#tc-05--response-structure)
   - [TC-06 — Edge Cases](#tc-06--edge-cases)
3. [Bugs Found](#bugs-found)

---

## 1. Scope

| In Scope | Out of Scope |
|---|---|
| Functional correctness of the PUT endpoint | Other membership endpoints |
| Authentication and authorization enforcement | UI layer |
| Request body field acceptance and rejection | Third-party OAuth provider internals |
| Response schema validation | Load / stress testing at infrastructure scale |
| Error response format and HTTP status codes | |
| Basic security checks | |

---

## 2. Test Cases

---

### TC-01 — Authentication & Authorization

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-01-01 | Critical | A valid `membershipId` | `PUT /{orgSlug}/memberships/{membershipId}` with valid Bearer token and valid body | `200 OK`; body is the updated membership object ✅ |
| TC-01-02 | Critical | — | Send request with no `Authorization` header | `401 Unauthorized`; body: `{"statusCode":401,"message":"Unauthorized access","error":"Unauthorized"}` ✅ |
| TC-01-03 | High | — | Send `Authorization: Bearer not_a_real_token` | `401 Unauthorized`; same error schema as TC-01-02 ✅ |
| TC-01-04 | High | An expired Bearer token | Send request with expired token | `401 Unauthorized`; same generic error schema as TC-01-02; no expiry-specific message ✅ |
| TC-01-05 | Critical | Token without `flex.community.memberships.update` scope | Send valid body with restricted token | `403 Forbidden`; response identifies the missing permission ⚠️ **BUG: returns `401 Unauthorized` instead of `403 Forbidden`** |
| TC-01-06 | Critical | `orgSlug` belonging to a different org | `PUT /organizations/some-other-org/memberships/{membershipId}` with valid body | `403 Forbidden` or `404 Not Found` ⚠️ **BUG: returns `500` with `"Organization not found"`** |

---

### TC-02 — Path Parameters

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-02-01 | Critical | A valid ObjectId that does not match any membership | `PUT .../memberships/aaaaaaaaaaaaaaaaaaaaaaaa` with valid body | `404 Not Found`; body: `{"statusCode":404,"message":"Item with Id (aaaaaaaaaaaaaaaaaaaaaaaa)","error":"Not Found"}` ✅ |
| TC-02-02 | High | — | `PUT /organizations/does-not-exist-xyz/memberships/{membershipId}` | `404 Not Found` ⚠️ **BUG: returns `500` with `"Organization not found"`** |

---

### TC-03 — Request Body: Accepted Fields

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-03-01 | Critical | A membership with no invoices | `PUT .../memberships/{id}` with `{"startDate": "2019-01-01T00:00:00.000Z"}` | `200 OK`; response reflects updated `startDate`; `modifiedAt` and `modifiedBy` updated ✅ *(note: `startDate` update fails with `500` on invoiced memberships — see TC-03-06)* |
| TC-03-02 | High | A valid `membershipId` | `PUT .../memberships/{id}` with `{"endDate": "2027-12-31T00:00:00.000Z"}` | `200 OK`; response reflects updated `endDate` ✅ |
| TC-03-03 | High | A valid `membershipId` | `PUT .../memberships/{id}` with `{"endDate": null}` | `200 OK`; `endDate` cleared (set to `null`) in the response ✅ |
| TC-03-04 | High | A valid `membershipId` | `PUT .../memberships/{id}` with `{"price": 200}` | `200 OK`; `price` and `discountedPrice` updated in the response ✅ |
| TC-03-05 | Medium | A valid `membershipId` | `PUT .../memberships/{id}` with `{"price": 0}` | `200 OK`; `price` set to `0`; `discountedPrice` also `0` ✅ |
| TC-03-06 | High | A membership with at least one invoice | `PUT .../memberships/{id}` with `{"startDate": "2019-01-01T00:00:00.000Z"}` | `422 Unprocessable Entity` or `409 Conflict`; message indicates the membership is invoiced ⚠️ **BUG: returns `500` with `"Cannot update membership's start date when the membership is invoiced"` — business logic error surfaced as 500** |
| TC-03-07 | High | A valid `membershipId` with `startDate` before intended `endDate` | `PUT .../memberships/{id}` with `{"endDate": "2010-01-01T00:00:00.000Z"}` (before existing `startDate`) | `400 Bad Request`; message indicates `endDate` must be after `startDate` ⚠️ **BUG: returns `500` with `"Membership startDate should be before the endDate"` — validation error surfaced as 500** |
| TC-03-08 | Medium | A valid `membershipId` | `PUT .../memberships/{id}` with `{}` (empty body) | `200 OK`; membership unchanged; `modifiedAt` not updated ✅ |
| TC-03-09 | Medium | A valid `membershipId` | `PUT .../memberships/{id}` with `{"startDate": "not-a-date"}` | `400 Bad Request`; message: `"startDate must be a valid ISO 8601 date string"` ⚠️ **BUG: returns `500` with `"Cannot update membership's start date when the membership is invoiced"` — ISO validation bypassed; business rule evaluated first** |
| TC-03-10 | Medium | A valid `membershipId` | `PUT .../memberships/{id}` with all three accepted fields: `{"startDate": "…", "endDate": "…", "price": 200}` | `200 OK`; all three fields updated in response *(not executed on invoiced membership — startDate fails; test with non-invoiced membership)* |
| TC-03-11 | Medium | A valid `membershipId` | `PUT .../memberships/{id}` with `{"endDate": "not-a-date"}` | `400 Bad Request`; message: `"endDate must be a valid ISO 8601 date string"` *(verify whether the same 500 bug seen in TC-03-09 applies here too)* |
| TC-03-12 | Medium | A valid `membershipId` | `PUT .../memberships/{id}` with `{"startDate": null}` | `400 Bad Request`; `startDate` cannot be nulled; or `200 OK` with `startDate` cleared *(document observed behaviour; compare with TC-03-03 where `endDate: null` succeeds)* |
| TC-03-13 | Medium | A valid `membershipId` | `PUT .../memberships/{id}` with `{"price": -1}` | `400 Bad Request`; message indicates price must be non-negative; or `200 OK` if the API permits negative prices *(document observed behaviour)* |

---

### TC-04 — Request Body: Rejected Fields

> All fields below return `400 Bad Request` with `"property <field> should not exist"`.

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-04-01 | Critical | — | Include `"name": "New Name"` in body | `400 Bad Request`; `"property name should not exist"` ✅ |
| TC-04-02 | Critical | — | Include `"type": "fixed"` in body | `400 Bad Request`; `"property type should not exist"` ✅ |
| TC-04-03 | Critical | — | Include `"plan": "<objectId>"` in body | `400 Bad Request`; `"property plan should not exist"` ✅ |
| TC-04-04 | Critical | — | Include `"location": "<objectId>"` in body | `400 Bad Request`; `"property location should not exist"` ✅ |
| TC-04-05 | High | — | Include `"company": "<objectId>"` in body | `400 Bad Request`; `"property company should not exist"` ✅ |
| TC-04-06 | High | — | Include `"isPersonal": true` in body | `400 Bad Request`; `"property isPersonal should not exist"` ✅ |
| TC-04-07 | High | — | Include `"intervalLength": "day"` in body | `400 Bad Request`; `"property intervalLength should not exist"` ✅ |
| TC-04-08 | High | — | Include `"status": "approved"` in body | `400 Bad Request`; `"property status should not exist"` ✅ |
| TC-04-09 | High | — | Include `"properties": {"key": "value"}` in body | `400 Bad Request`; `"property properties should not exist"` ✅ |
| TC-04-10 | Medium | — | Include `"_id": "<objectId>"` in body | `400 Bad Request`; `"property _id should not exist"` ✅ |
| TC-04-11 | Medium | — | Include `"isLocked": true` in body | `400 Bad Request`; `"property isLocked should not exist"` ✅ |
| TC-04-12 | Medium | — | Include `"intervalCount": 3` in body | `400 Bad Request`; `"property intervalCount should not exist"` ✅ |
| TC-04-13 | Medium | — | Include `"discountAmount": 10` in body | `400 Bad Request`; `"property discountAmount should not exist"` ✅ |
| TC-04-14 | Medium | — | Include `"deposit": 100` in body | `400 Bad Request`; `"property deposit should not exist"` ✅ |
| TC-04-15 | Medium | — | Include `"createdAt": "2024-01-01T00:00:00.000Z"` in body | `400 Bad Request`; `"property createdAt should not exist"` ✅ |
| TC-04-16 | Medium | — | Include `"modifiedAt": "2024-01-01T00:00:00.000Z"` in body | `400 Bad Request`; `"property modifiedAt should not exist"` ✅ |
| TC-04-17 | Medium | — | Include `"calculatedStatus": "expired"` in body | `400 Bad Request`; `"property calculatedStatus should not exist"` ✅ |
| TC-04-18 | Low | — | Include `"unknownField": "value"` in body | `400 Bad Request`; `"property unknownField should not exist"` ✅ |
| TC-04-19 | Medium | — | Include multiple rejected fields in a single body | `400 Bad Request`; `message` is an array listing every rejected field ✅ |

---

### TC-05 — Response Structure

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-05-01 | Critical | A valid `membershipId` | Send a valid PUT; apply MembershipResultDto verification to the `200` response — see [GET single test plan TC-03-01b – TC-03-10](test-plan-get-membership.md#tc-03--response-structure---verify-membershipresultdto) | All field-presence, type, timestamp-format, enum, and conditional-field checks pass ✅. Note: `endDate` may be `null` when explicitly cleared via PUT (see TC-03-03) — treat `null` as a valid value alongside the absent/ISO 8601 cases in TC-03-04a and TC-03-04b |
| TC-05-02 | High | A valid `membershipId` | Send a PUT that changes `price`; compare `modifiedAt` before and after | `modifiedAt` is updated to the current timestamp; `modifiedBy` reflects the authenticated user's ID ✅ |
| TC-05-03 | Medium | A valid `membershipId` | Send `PUT` with `{}` (no changes); compare `modifiedAt` before and after | `modifiedAt` is **not** updated — the server does not create a revision when no data changes ✅ |
| TC-05-04 | Medium | — | Inspect response headers from a valid request | `Content-Type: application/json; charset=utf-8` ✅ |

---

### TC-06 — Edge Cases

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-06-01 | Medium | — | Send `PUT` with a malformed JSON body (e.g. `{startDate:}`) and `Content-Type: application/json` | `400 Bad Request`; message indicates invalid JSON; no stack trace leaked |
| TC-06-02 | Low | A valid `membershipId` with `startDate` set | `PUT .../memberships/{id}` with `{"endDate": "<same ISO value as current startDate>"}` | `200 OK`; equal `startDate` and `endDate` are permitted ✅ |

---

## Bugs Found

| TC ID | Severity | Description |
|---|---|---|
| TC-01-04 | Low | Expired token returns the same generic `401 Unauthorized` error as an invalid token — response does not indicate expiry; clients cannot distinguish the cause |
| TC-01-05 | High | Token with missing scope returns `401 Unauthorized` instead of `403 Forbidden` — `401` implies unauthenticated; the token is valid but lacks authorisation |
| TC-01-06 | Critical | Wrong org returns `500` instead of `403`/`404` |
| TC-02-02 | High | Non-existent `orgSlug` returns `500` instead of `404` |
| TC-03-06 | High | Updating `startDate` on an invoiced membership returns `500` instead of `409`/`422` — business logic error surfaced as internal server error |
| TC-03-07 | High | `endDate` before `startDate` returns `500` instead of `400`/`422` — validation error surfaced as internal server error |
| TC-03-09 | Medium | Invalid `startDate` format returns `500` (business rule evaluated before format validation) instead of `400` |
