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
| `test-plan-get-memberships.md` | GET | `/memberships` | 63 |
| `test-plan-post-membership.md` | POST | `/memberships` | 47 |
| `test-plan-get-count.md` | GET | `/memberships/count` | 37 |
| `test-plan-get-membership.md` | GET | `/memberships/{membershipId}` | 35 |
| `test-plan-put-membership.md` | PUT | `/memberships/{membershipId}` | 46 |
| `test-plan-delete-membership.md` | DELETE | `/memberships/{membershipId}` | 21 |
| **Total** | | | **249** |

---

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
| Required Fields | POST — required field presence, types, and formats |
| Optional Fields | POST — optional field behaviour and defaults |
| Field Validation | POST — invalid values, type mismatches, enum violations |
| Request Body: Accepted Fields | PUT — fields the endpoint will accept and apply |
| Request Body: Rejected Fields | PUT — fields the endpoint rejects with `400` |
| Response Structure | Field presence, types, enum values, contract compliance |
| Pagination | `$limit`, `$cursorNext`, `$cursorPrev` behaviour |
| Filtering | Query parameter filter operators |
| Field Selection | `$select` behaviour |
| Query Parameters | General query parameter handling |
| Business Rules | Domain constraints enforced at the API layer |
| Idempotency | Behaviour of repeated identical requests |
| Edge Cases | Boundary values, unexpected input combinations |

---

## Bug Classification

Bugs found during execution are collected in a **Bugs Found** table at the end of each plan. Severity maps to the priority of the failing test case:

| Severity | Meaning |
|---|---|
| Critical | Security or data isolation failure; core flow broken |
| High | Functional behaviour incorrect; meaningful use blocked or data at risk |
| Medium | Incorrect but not immediately harmful; validation or contract gap |
| Low | Minor deviation from standards or cosmetic issue |

