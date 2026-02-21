# OfficeRnD UI Test — Playwright + TypeScript

## Project Purpose

Implement a minimal Playwright + TypeScript test framework that executes exactly one end-to-end test case against the OfficeRnD staging UI. Build only the components, classes, and methods required to run that test — nothing more.

---

## Test Environment

| Item | Value |
|---|---|
| URL | `https://staging.officernd.com/login` |
| Email | `teodoravhristova@gmail.com` |
| Password | `P@ssw0rd` |

---

## Test Case

**TC-UI-01 — Filter memberships by name and verify result count**

| Step | Action |
|---|---|
| 1 | Navigate to `https://staging.officernd.com/login` |
| 2 | Log in with the credentials above |
| 3 | In the left-hand menu select **Operations**, then **Memberships** from the submenu |
| 4 | Apply a filter by **Name** with the value `zara` |
| 5 | Assert that exactly **2 results** are displayed in the memberships grid |

---

## Implementation Requirements

### Framework constraints

- Use **Playwright** with **TypeScript**
- Use the **Page Object Model** pattern — one page object per distinct page or component
- Create **only** the page objects, locators, and helper methods needed to execute TC-UI-01; do not build reusable utilities or abstractions for hypothetical future tests
- Credentials must be stored in a `.env` file and read via `process.env`; never hard-coded in test or page object files
- The `.env` file must be listed in `.gitignore`

### Required files (minimum set)

```
ui-task/
├── .env                        # EMAIL and PASSWORD (git-ignored)
├── .gitignore
├── package.json
├── playwright.config.ts
├── tsconfig.json
├── pages/
│   ├── LoginPage.ts
│   ├── MembershipsPage.ts
│   └── NavigationMenu.ts       # or inline into MembershipsPage if simpler
└── tests/
    └── memberships-filter.spec.ts
```

### Locator strategy

Prefer locators in this order:
1. `getByRole` / `getByLabel` / `getByPlaceholder` (ARIA-based)
2. `getByText`
3. `data-testid` attribute
4. CSS selector as a last resort

### Assertions

Use Playwright's built-in `expect` with auto-retry where possible (e.g. `toHaveCount`, `toBeVisible`).

---

## General Notes

- Target browser: **Chromium** (default)
- Run command: `npx playwright test`
- Do not add linting, CI configuration, or reporting beyond Playwright's built-in HTML reporter unless explicitly requested
