# Bug Summary

**Project:** OfficeRnD Flex Memberships API
**Environment:** `https://staging.officernd.com/api/v2/organizations/{orgSlug}/memberships`

Several bugs appeared consistently across multiple endpoints, indicating systemic issues rather than isolated defects.

---

## Present in all six endpoints

| Bug | Expected | Actual |
|---|---|---|
| Expired token returns same generic `401` as invalid token | `401` with expiry-specific message | Generic `{"statusCode":401,"message":"Unauthorized access","error":"Unauthorized"}` — clients cannot distinguish cause |
| Token with missing scope returns `401` instead of `403` | `403 Forbidden` | `401 Unauthorized` — token is valid but lacks authorisation; incorrect status code |
| Wrong `orgSlug` (non-existent org) | `404 Not Found` | `500 Internal Server Error` with `"Organization not found"` |

---

## Present in all single-resource endpoints (GET, PUT, DELETE)

| Bug | Expected | Actual |
|---|---|---|
| Non-ObjectId `membershipId` | `400 Bad Request` | `404 Not Found` — format not validated before lookup |
| `POST` on `/{membershipId}` path | `405 Method Not Allowed` | `404 Not Found` with `"Cannot POST ..."` |

---

## Business logic surfaced as 500

Multiple endpoints return `500 Internal Server Error` for predictable business rule violations. These should return `4xx` responses:

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

---

## Endpoint-specific findings

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
