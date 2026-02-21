# Testing Approach and Structure

**Project:** OfficeRnD Flex Memberships API
**Environment:** `https://staging.officernd.com/api/v2/organizations/{orgSlug}/memberships`
**Date:** 2026-02-21

---

## Overview

This document describes the approach, structure, and conventions used across the six test plan files in this project. Each plan covers one endpoint of the OfficeRnD Flex Memberships API and was both designed and executed against the staging environment during the same session.

---

## Endpoints Covered

| File | Method | Endpoint | Test Cases |
|---|---|---|---|
| `test-plan-get-memberships-v2.md` | GET | `/memberships` | 63 |
| `test-plan-post-membership.md` | POST | `/memberships` | 47 |
| `test-plan-get-count.md` | GET | `/memberships/count` | 37 |
| `test-plan-get-membership.md` | GET | `/memberships/{membershipId}` | 35 |
| `test-plan-put-membership.md` | PUT | `/memberships/{membershipId}` | 46 |
| `test-plan-delete-membership.md` | DELETE | `/memberships/{membershipId}` | 21 |
| **Total** | | | **249** |

---

## Testing Approach

### Execution Against Staging

All test cases were executed live against the staging environment using short Python scripts (`urllib.request`). Responses — including HTTP status codes, response bodies, and headers — were captured and used to set expected results directly in the test plan tables. Cases that could not be executed (e.g. expired tokens, restricted-scope tokens) are marked `*(not executed)*` with the reason stated.

This approach means every expected result in these plans is grounded in observed behaviour, not assumption. Where the observed behaviour differed from what the API specification or HTTP standards require, the case is marked with ⚠️ **BUG** and included in the **Bugs Found** section.

### State Management

For mutating endpoints (POST, PUT, DELETE), test execution followed this pattern:

- **POST**: Created memberships were left in the staging org. The org already contained seed data and the additional memberships do not affect other test plans.
- **PUT**: Each change was restored to the original field values immediately after the test case, using a follow-up PUT. The membership used (`69973aa8b93b7f45ab1360cf`) was confirmed to be in its original state after testing.
- **DELETE**: A dedicated throwaway membership (`DELETE-TEST`) was created via POST immediately before deletion tests. The original membership was not deleted — it has associated invoices and the API correctly prevents its deletion.

### Cases Not Executed

Some cases require credentials or infrastructure not available during testing:

| Reason | Affected cases |
|---|---|
| Expired Bearer token | TC-01-04 in every plan |
| Token with restricted scope | TC-01-05 in every plan |
| Empty org (zero memberships) | TC-03-03 in GET list and GET count |

---

## Test Case Structure

Each test plan organises cases into categories. Each category is a `###` heading containing a single horizontal table.

```
| ID | Priority | Prerequisites | Steps | Expected Result |
```

**ID** — `TC-XX-YY` where `XX` is the category number and `YY` is the case number. Sub-cases use a letter suffix (`TC-03-01a`).

**Priority** — one of four levels applied to every case:

| Priority | When to use |
|---|---|
| Critical | Security, data isolation, core happy path — must pass before any release |
| High | Important functional behaviour; failure blocks meaningful use |
| Medium | Non-default but documented behaviour; edge cases with real-world impact |
| Low | Unlikely edge cases or exploratory observations |

**Prerequisites** — data or state required before the test runs, beyond the global token assumption. Set to `—` when none are needed.

**Steps** — the literal request to send (URL pattern, method, headers, body). Kept concise and unambiguous.

**Expected Result** — HTTP status code first, followed by a precise description of the response body or behaviour. Observed results are appended inline using ✅ (passed) or ⚠️ **BUG** (unexpected behaviour).

A global blockquote at the top of each plan's Test Cases section removes the need to repeat token requirements in every row:

> All test cases assume a valid Bearer token with the required scope unless the case explicitly tests authentication or authorization.

---

## Category Conventions

The same category names are used across all plans for consistency:

| Category | What it covers |
|---|---|
| Authentication & Authorization | Token presence, format, expiry, scope enforcement, tenant isolation |
| Path Parameters | `orgSlug` and `membershipId` validation |
| Request Body | Payload field acceptance and rejection for POST and PUT |
| Response Structure | Field presence, types, enum values, contract compliance |
| Pagination | `$limit`, `$cursorNext`, `$cursorPrev` behaviour |
| Filtering | Query parameter filter operators |
| Field Selection | `$select` behaviour |
| Sorting | `$sort` behaviour |
| Query Parameters | General query parameter handling (count endpoint) |
| Business Rules | Domain constraints enforced at the API layer |
| Idempotency | Behaviour of repeated identical requests |
| Consistency | Response parity between related endpoints |
| Security | Injection, header leakage, error body content |
| Performance | Response time, rate limiting |
| HTTP Methods | Correct handling of unsupported methods |

---

## Bug Classification

Bugs found during execution are collected in a **Bugs Found** table at the end of each plan. Severity maps to the priority of the failing test case:

| Severity | Meaning |
|---|---|
| Critical | Security or data isolation failure; core flow broken |
| High | Functional behaviour incorrect; meaningful use blocked or data at risk |
| Medium | Incorrect but not immediately harmful; validation or contract gap |
| Low | Minor deviation from standards or cosmetic issue |

---

## Cross-Cutting Findings

Several bugs appeared consistently across all six endpoints, indicating systemic issues rather than isolated defects.

### Present in all six endpoints

| Bug | Expected | Actual |
|---|---|---|
| Wrong `orgSlug` (non-existent org) | `404 Not Found` | `500 Internal Server Error` with `"Organization not found"` |
| `x-powered-by: Express` response header | Header absent or suppressed | Always present — reveals server framework |

### Present in all single-resource endpoints (GET, PUT, DELETE)

| Bug | Expected | Actual |
|---|---|---|
| Non-ObjectId `membershipId` | `400 Bad Request` | `404 Not Found` — format not validated before lookup |
| `POST` on `/{membershipId}` path | `405 Method Not Allowed` | `404 Not Found` with `"Cannot POST ..."` |

### Business logic surfaced as 500

Multiple endpoints return `500 Internal Server Error` for conditions that are predictable business rule violations. These should return `4xx` responses:

| Endpoint | Condition | Actual | Expected |
|---|---|---|---|
| POST | `endDate` before `startDate` | `500` | `400`/`422` |
| POST | `name` is empty string `""` | `500` | `400` |
| POST | `location` is non-ObjectId string | `500` | `400` |
| POST | `plan` is non-ObjectId string | `500` | `400` |
| POST | `plan` ObjectId does not exist | `500` | `404` |
| POST | `location` ObjectId does not exist | `500` | `404` |
| PUT | `startDate` updated on invoiced membership | `500` | `409`/`422` |
| PUT | `endDate` before `startDate` | `500` | `400`/`422` |
| PUT | Invalid `startDate` format (on invoiced membership) | `500` | `400` |
| DELETE | Membership has associated invoices | `500` | `409`/`422` |

### Endpoint-specific findings

| Endpoint | Finding |
|---|---|
| GET list | `$sort` and `createdAt`/`modifiedAt` filter parameters not supported — return `400` |
| GET list | `$limit=-5` accepted as valid — negative values not rejected |
| GET list | Invalid `$cursorNext` returns `500` instead of `400`/`422` |
| GET list | Supplying both `$cursorNext` and `$cursorPrev` returns `504 Gateway Timeout` |
| GET count | All query parameters (including field filters) silently ignored — count always returns unfiltered total |
| GET count | `POST` on `/count` path returns `404` instead of `405` |
| GET single | `properties` absent from response when empty `{}`; always present as `{}` in GET list |
| PUT | Only `startDate`, `endDate`, and `price` accepted — all other fields including `name`, `type`, `plan`, `location`, `company`, `isPersonal`, `intervalLength`, and `properties` return `400 "property X should not exist"` |
| DELETE | Response includes `properties: {}` even when empty; GET single omits it — inconsistency |
| DELETE | DELETE is not idempotent — second call on the same ID returns `404` |
