# OfficeRnD Memberships API — Test Plan Project

## Project Purpose

This project produces a comprehensive test plan for the **OfficeRnD Flex Memberships API**. The goal is maximum test coverage across all six membership endpoints, presented in clear, minimal markdown.

Every test plan document in this project must:
- Cover all six endpoints (see below)
- Follow the table-based style established in `test-plan-get-memberships-v2.md`
- Assign a **Priority** (Critical / High / Medium / Low) to every test case
- Include a **Bugs Found** section at the end (omit if no bugs were observed)
- Never send live requests to the API — all cases are design-only

---

## API Documentation

| Endpoint | Method | Documentation |
|---|---|---|
| Get Memberships (list) | GET | https://developer.officernd.com/reference/membershipscontroller_getitems |
| Add Membership | POST | https://developer.officernd.com/reference/membershipscontroller_additem |
| Get Membership Count | GET | https://developer.officernd.com/reference/membershipscontroller_count |
| Get Membership (single) | GET | https://developer.officernd.com/reference/membershipscontroller_getitem |
| Update Membership | PUT | https://developer.officernd.com/reference/membershipscontroller_updateitem |
| Delete Membership | DELETE | https://developer.officernd.com/reference/membershipscontroller_deleteitem |

**Base URL:** `https://app.officernd.com/api/v2/organizations/{orgSlug}/memberships`
**Auth:** OAuth 2.0 Bearer token. Required scope varies per endpoint — check individual documentation pages.
**`$limit` max:** 50

---

## Test Case Style Guide

### Structure

Test cases are grouped into categories. Each category is a `###` section heading containing a single horizontal table. Every row is one test case.

```markdown
### <Category Name>

| ID | Priority | Prerequisites | Steps | Expected Result |
|---|---|---|---|---|
| TC-XX-YY | Critical | Org with ≥1 membership | `GET /{orgSlug}/memberships` with valid token | `200 OK`; `results` array present |
```

**Global prerequisites note** — add the following blockquote at the top of every test plan's Test Cases section:

```markdown
> All test cases assume a valid Bearer token with the required scope unless the case explicitly tests authentication or authorization.
```

This removes the need to repeat "Valid Bearer token" in every Prerequisites cell. Prerequisites cells should state only data or state requirements (e.g. "Org with ≥2 memberships", "An expired token").

### Column Definitions

| Column | Guidance |
|---|---|
| **ID** | `TC-XX-YY` where XX is the category number and YY is the case number within it. Sub-cases use a letter suffix: `TC-03-01a`, `TC-03-01b`. |
| **Priority** | Critical / High / Medium / Low — see definitions below |
| **Prerequisites** | Data or state required before the test runs. Use `—` when none are needed beyond the global token assumption. Keep concise. |
| **Steps** | What to send or do. Prefer the literal URL pattern or parameter (e.g. `GET ...?$limit=1`). |
| **Expected Result** | HTTP status code first, then a precise description. For ambiguous cases, list both acceptable outcomes and add *(document observed behaviour)*. |

### Priority Definitions

| Priority | When to use |
|---|---|
| **Critical** | Security, data isolation, core happy path — must pass before any release |
| **High** | Important functional behaviour; failure blocks meaningful use |
| **Medium** | Non-default but documented behaviour; edge cases with real-world impact |
| **Low** | Unlikely edge cases or exploratory observations |

### Category Conventions

Categories are used as `###` section headings. Use these names consistently across all test plan files:

- `Authentication & Authorization` — token presence, format, expiry, scope, tenant isolation
- `Path Parameters` — `orgSlug`, `membershipId` validation
- `Request Body` — payload validation for POST / PUT
- `Response Structure` — response field types, required fields, enum values, contract compliance
- `Pagination` — `$limit`, `$cursorNext`, `$cursorPrev`
- `Filtering` — `createdAt`, `modifiedAt` operators
- `Field Selection` — `$select`
- `Sorting` — `$sort`
- `Edge Cases` — boundary values, unexpected input combinations, idempotency
- `Security` — injection, header leakage, HTTPS enforcement
- `Performance` — response time, rate limiting

---

## Test Plan Files

| File | Endpoint | Status |
|---|---|---|
| `test-plan-get-memberships-v2.md` | GET list | Draft |
| `test-plan-post-membership.md` | POST | — |
| `test-plan-get-count.md` | GET count | — |
| `test-plan-get-membership.md` | GET single | — |
| `test-plan-put-membership.md` | PUT | — |
| `test-plan-delete-membership.md` | DELETE | — |

---

## Commit Message Convention

- Start with a verb followed by a colon: `add:`, `update:`, `fix:`, `delete:`, `refactor:`, `rename:`, etc.
- Keep the message short
- Use lowercase only

**Example:** `add: test cases for pagination edge cases`

---

## General Testing Notes

- **Do not cross-test endpoints.** Each test plan covers only the behaviour of its own endpoint. Do not verify a response by comparing it against output from a different endpoint.

- **Path traversal on `membershipId`** (e.g. `../../etc`) is not a test case. The API sits behind auth middleware that redirects or rejects before routing — this is not specific to the memberships endpoint and adds no meaningful coverage.

- **Non-ObjectId `membershipId` path parameter** (e.g. `notanid`) is not a separate test case. The API returns `404` for both a valid-format but missing ObjectId and a non-ObjectId string — the behaviour is identical, so there is no added value in testing them separately. Only test the valid-but-missing ObjectId case (`aaaaaaaaaaaaaaaaaaaaaaaa`).



- **Do not send live requests** to the API while writing or reviewing test plans.
- All date values must use **ISO 8601** format: `YYYY-MM-DDTHH:mm:ss.sssZ`.
- Pagination tests require a minimum of **55 seed memberships** to exercise multi-page behaviour meaningfully given the `$limit` max of 50.
- Schema validation test cases must cover all **enum fields** with their exact allowed values. For the membership object these are:
  - `status`: `approved`, `not_approved`
  - `calculatedStatus`: `not_started`, `active`, `expired`, `not_approved`
  - `type`: `month_to_month`, `fixed`
  - `intervalLength`: `once`, `hour`, `day`, `month`
- Each plan must cover the upper boundary of `$limit` (50) and the over-limit case (51).
