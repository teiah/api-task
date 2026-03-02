# Test Plan: DELETE /memberships/{membershipId}

**Version:** 1.0 · **Date:** 2026-02-21 · **Status:** Tested

**Endpoint:** `DELETE https://staging.officernd.com/api/v2/organizations/{orgSlug}/memberships/{membershipId}`
**Auth:** OAuth 2.0 Bearer token · **Required scope:** `flex.community.memberships.delete`
**Documentation:** https://developer.officernd.com/reference/membershipscontroller_deleteitem.md

> All test cases assume a valid Bearer token with the `flex.community.memberships.delete` scope unless the case explicitly tests authentication or authorization.

> **Test execution note:** TC-03-01 was executed against a freshly created membership (`DELETE-TEST`) to avoid destroying a membership with associated invoices.

---

## Contents

1. [Scope](#1-scope)
2. [Test Cases](#2-test-cases)
   - [TC-01 — Authentication & Authorization](#tc-01--authentication--authorization)
   - [TC-02 — Path Parameters](#tc-02--path-parameters)
   - [TC-03 — Response Structure](#tc-03--response-structure)
   - [TC-04 — Business Rules](#tc-04--business-rules)
   - [TC-05 — Idempotency](#tc-05--idempotency)
3. [Bugs Found](#bugs-found)

---

## 1. Scope

| In Scope | Out of Scope |
|---|---|
| Functional correctness of the DELETE endpoint | Other membership endpoints |
| Authentication and authorization enforcement | UI layer |
| Response schema validation | Third-party OAuth provider internals |
| Business rule enforcement (invoiced memberships) | Load / stress testing at infrastructure scale |
| Idempotency behaviour | |
| Basic security checks | |

---

## 2. Test Cases

---

### TC-01 — Authentication & Authorization

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-01-01 | Critical | A non-invoiced membership | `DELETE /{orgSlug}/memberships/{membershipId}` with valid Bearer token | `200 OK`; body is the membership object as it existed before deletion ✅ |
| TC-01-02 | Critical | — | Send request with no `Authorization` header | `401 Unauthorized`; body: `{"statusCode":401,"message":"Unauthorized access","error":"Unauthorized"}` ✅ |
| TC-01-03 | High | — | Send `Authorization: Bearer not_a_real_token` | `401 Unauthorized`; same error schema as TC-01-02 ✅ |
| TC-01-04 | High | An expired Bearer token | Send request with an expired token | `401 Unauthorized`; error indicates expiry; no stack trace *(not executed)* |
| TC-01-05 | Critical | Token without `flex.community.memberships.delete` scope | Send request with restricted token | `403 Forbidden`; response identifies the missing permission *(not executed)* |
| TC-01-06 | Critical | `orgSlug` belonging to a different org | `DELETE /organizations/some-other-org/memberships/{membershipId}` with valid token | `403 Forbidden` or `404 Not Found` ⚠️ **BUG: returns `500` with `"Organization not found"`** |

---

### TC-02 — Path Parameters

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-02-01 | Critical | A valid ObjectId that does not match any membership | `DELETE .../memberships/aaaaaaaaaaaaaaaaaaaaaaaa` | `404 Not Found`; body: `{"statusCode":404,"message":"Item with Id (aaaaaaaaaaaaaaaaaaaaaaaa)","error":"Not Found"}` ✅ |
| TC-02-02 | High | — | `DELETE /organizations/does-not-exist-xyz/memberships/{membershipId}` | `404 Not Found` ⚠️ **BUG: returns `500` with `"Organization not found"`** |

---

### TC-03 — Response Structure

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-03-01 | Critical | A non-invoiced membership | Send a valid DELETE; apply MembershipResultDto verification to the `200` response — see [GET single test plan TC-03-01b – TC-03-10](test-plan-get-membership.md#tc-03--response-structure---verify-membershipresultdto) | Response is the membership as it existed just before deletion and passes all field-presence, type, timestamp-format, enum, and conditional-field checks ✅. Note: `properties` is always present in the DELETE response even when empty (`{}`) — this differs from the TC-03-06 behaviour in GET single and is documented in TC-03-03 and TC-03-04 below |
| TC-03-02 | High | A non-invoiced membership | After receiving the `200` response, immediately `GET` the same `membershipId` | `404 Not Found`; the membership no longer exists *(implied by idempotency test TC-05-01 — second DELETE returns 404)* |
| TC-03-03 | Medium | A non-invoiced membership with `properties: {}` | Send a valid DELETE; inspect `properties` field in response | `properties` is present as `{}` even when empty ✅ *(note: this is inconsistent with GET single, which omits `properties` when empty — see TC-03-04)* |
| TC-03-04 | Medium | — | Compare `properties` field behaviour between DELETE response and GET single response for the same membership | DELETE response always includes `properties` (even `{}`); GET single omits `properties` when empty ⚠️ **BUG: inconsistent `properties` field presence between endpoints** |
| TC-03-05 | Medium | — | Inspect response headers from a valid request | `Content-Type: application/json; charset=utf-8` ✅ |

---

### TC-04 — Business Rules

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-04-01 | Critical | A membership that has at least one associated invoice | `DELETE .../memberships/{invoicedMembershipId}` | `409 Conflict` or `422 Unprocessable Entity`; message indicates membership has invoices and cannot be deleted ⚠️ **BUG: returns `500` with `"Cannot delete invoiced membership"` — business rule violation surfaced as internal server error** |
| TC-04-02 | Medium | A membership with `isLocked: true` | `DELETE .../memberships/{lockedMembershipId}` | `409 Conflict` or `403 Forbidden`; message indicates the membership is locked and cannot be deleted; or `200 OK` if locking does not protect against deletion *(document observed behaviour)* |

---

### TC-05 — Idempotency

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-05-01 | High | A non-invoiced membership | Send `DELETE .../memberships/{id}` twice using the same `membershipId` | First request: `200 OK`; second request: `404 Not Found` — DELETE is **not idempotent** ✅ |

---

## Bugs Found

| TC ID | Severity | Description |
|---|---|---|
| TC-01-06 | Critical | Wrong org returns `500` instead of `403`/`404` |
| TC-02-02 | High | Non-existent `orgSlug` returns `500` instead of `404` |
| TC-03-04 | Low | `properties` always present in DELETE response (including `{}`); absent in GET single when empty — inconsistency between endpoints |
| TC-04-01 | High | Deleting an invoiced membership returns `500` instead of `409`/`422` — business rule violation surfaced as internal server error |
