# 🧪 Advanced Quality Architecture - Playwright & MCP

![Playwright Tests](https://github.com/gustavaom7/playwright-typescript-e2e-framework/actions/workflows/playwright.yml/badge.svg?branch=main)
[![Quality](https://img.shields.io/badge/Quality-Assurance-orange)](https://github.com/gustavaom7/playwright-typescript-e2e-framework)
[![MCP](https://img.shields.io/badge/MCP-Playwright-blueviolet)](https://github.com/gustavaom7/playwright-typescript-e2e-framework/blob/main/.mcp.json)
![Performance (k6)](https://github.com/gustavaom7/playwright-typescript-e2e-framework/actions/workflows/performance.yml/badge.svg?branch=main)

Professional E2E automation suite developed with **Playwright** and **TypeScript** against [saucedemo.com](https://www.saucedemo.com/), extended with an **AI-driven workflow** (Playwright MCP + Claude Code skills) for test generation and defect triage, **k6** browser performance checks and **Slack** reporting. See [docs/ai-workflow.md](docs/ai-workflow.md).

---

## 🚀 Key Features & Engineering Patterns

### 🖥️ UI Automation (E2E)
* **Page Object Model (POM):** one class per screen (`LoginPage`, `InventoryPage`, `CartPage`, `CheckoutPage`, `ItemDetailPage`), all extending a shared `BasePage`.
* **Session Reuse:** `auth.setup.ts` logs in once and persists storage state (`playwright/.auth/user.json`), so every spec starts pre-authenticated instead of repeating the UI login flow.
* **Fixture-driven composition:** `pages.fixture.ts` extends Playwright's `test` with one fixture per page object; `test-data.ts` centralizes user accounts, products, and checkout data.
* **Cross-browser coverage:** desktop scenarios run against **Chromium, Firefox, and WebKit**.
* **Projects by scenario nature:** `public` (anonymous), `auth` (shared session, parallel), `session` (logout, runs after `auth`) and `mobile-*` (Pixel 5 / iPhone 12 on real Chromium/WebKit engines).
* **Tag filtering per project:** `TAG=@smoke` / `SEM_TAG=@slow` env vars, since `--grep` does not filter dependency projects.
* **Known-bug regression suite:** `tests/public/quirks.spec.ts` pins down real SauceDemo bugs tied to specific seeded users (`problem_user`, `error_user`, `performance_glitch_user`, `visual_user`).

### 🔬 Beyond Happy-Path E2E
* **Network resilience (`page.route`):** product images aborted, slow image responses and unreachable third-party font hosts — the core shopping flow must keep working. Also asserts zero failed asset requests and zero console errors on load.
* **Accessibility (axe-core):** WCAG 2.x A/AA audit of login, login-error, inventory, cart and checkout. Full violation detail is attached to the report as JSON. Baseline is 0 violations, so the gate defaults to 0; relax it with `A11Y_MAX=<n>`.
* **Data-driven scenarios:** invalid-credential, checkout-validation and sort-order cases are tables of data driving one test body each (7 login cases, 4 checkout cases, 4 sort orders).
* **Performance budgets:** Navigation Timing of the inventory page against budgets named by intent (`utils/budgets.ts`), scalable with `TIMEOUT_FACTOR` for slower environments. Timings are attached to the report even when passing.
* **Visual regression (Chromium only, baseline not yet enabled):** `tests/authenticated/visual-regression.spec.ts` uses Playwright's built-in `toHaveScreenshot()` — no external service, no account, no extra dependency — on 5 flows (inventory, sorted inventory, cart, checkout step one, checkout complete). See "Visual regression" under CI/CD Workflow for why the baseline isn't committed yet and how to turn it on.

### 🤖 AI-Driven Testing (MCP + Claude Code)
* **Playwright MCP:** `.mcp.json` wires up the official `@playwright/mcp` server, giving Claude Code live control of a real browser against saucedemo.com.
* **AI-driven test generation:** the `explore-and-generate-tests` skill (`.claude/skills/`) explores a page through MCP, writes a page map (`docs/page-maps/`), then drafts a Page Object and spec that follow this repo's conventions and runs lint, typecheck and the new spec before reporting.
* **Human in the loop:** generated tests are drafts. They are reviewed before merge, and anything the agent could not confirm is marked `test.fixme` instead of guessed.
* **Automated defect triage:** `scripts/triage-failures.ts` reads the Playwright JSON report, merges the same failure across browsers, classifies it (functional, accessibility, performance, visual, network, flaky), ranks it with an explicit severity rubric, flags likely common root causes and writes Jira-ready drafts. It is deterministic on purpose, so it is reproducible and testable.
* **AI-assisted defect management:** the `bug-report` skill turns those drafts into final reports: it confirms the bug reproduces, describes the evidence and rewrites the steps as user-level steps.
* **See it without running anything:** `npm run triage:example` and [`docs/examples/bug-reports/`](docs/examples/bug-reports/) (generated from a synthetic fixture).

### ⚡ Performance (k6) and Slack
* **k6 browser test:** `perf/k6/inventory-browser.js` measures Core Web Vitals (LCP, FCP, CLS) of the login-to-inventory flow in headless Chromium and fails when a limit in `perf/k6/thresholds.json` is breached.
* **One source of truth:** the same thresholds file drives the k6 gate and the summary/Slack report, so they can never disagree.
* **Slack:** the performance workflow and the failure triage post a summary through an optional `SLACK_WEBHOOK_URL` secret; without it they just skip the notification.

### ⚙️ DevOps & CI/CD
* **GitHub Actions:** lint, typecheck, and the full cross-browser regression suite run on every push/PR.
* **Dependency caching:** Playwright browser binaries are cached by version to speed up runs.
* **Live report on GitHub Pages:** the HTML report of the latest `main` run is published at https://gustavaom7.github.io/playwright-typescript-e2e-framework/ (requires Pages source set to *GitHub Actions*).
* **Cost-aware matrix:** push/PR run Chromium + Chrome mobile for a fast signal; a nightly schedule (and manual dispatch) runs every browser and mobile project.
* **Automated Reporting:** HTML report uploaded as a build artifact on every run, even on failure.
* **Failure triage in CI:** every run turns failures into ranked bug drafts (uploaded as the `bug-report-drafts` artifact and written to the job summary).
* **Scheduled performance run:** `.github/workflows/performance.yml` runs the k6 browser test daily and can be started manually.

---

## 🏗️ Project Structure

```text
playwright/
├── .github/workflows/ # CI/CD (playwright.yml, visual-regression.yml, performance.yml)
├── .claude/skills/ # Claude Code skills: explore-and-generate-tests, bug-report
├── .mcp.json # Playwright MCP server config
├── docs/ # ai-workflow.md and example bug reports
├── perf/k6/ # k6 browser test and shared thresholds
├── scripts/ # triage-failures.ts, k6-summary.ts and a sample fixture
├── pages/ # Page Object Model
│ ├── base.ts # Shared navigate() helper
│ ├── LoginPage.ts
│ ├── InventoryPage.ts
│ ├── CartPage.ts
│ ├── CheckoutPage.ts
│ └── ItemDetailPage.ts
├── fixtures/
│ ├── pages.fixture.ts # Injects page objects as Playwright fixtures
│ └── test-data.ts # Shared users, products, checkout data
├── tests/
│ ├── auth.setup.ts # Logs in once, persists storage state
│ ├── public/ # Anonymous: login, seeded-user bugs (quirks)
│ ├── authenticated/ # Catalog, cart, checkout, visual regression (shared session)
│ ├── session/ # Logout, runs after the authenticated projects
│ └── mobile/ # Pixel 5 / iPhone 12 flows
├── utils/
│ ├── a11y.ts # axe audit helper (A11Y_MAX gate + JSON attachment)
│ └── budgets.ts # Performance budgets, scaled by TIMEOUT_FACTOR
├── playwright.config.ts # Projects: setup -> {browser}-public / -auth / -session, mobile-*
└── package.json # Scripts and dependencies
```

## 🚦 Local Execution

1. **Installation**

```bash
npm install
npx playwright install --with-deps
```

2. **Running Tests**

**Full suite, headless (all browsers):** `npx playwright test`

**Headed mode:** `npx playwright test --headed`

**Interactive UI mode:** `npx playwright test --ui`

**Single project:** `npx playwright test --project=chromium-auth`

**By tag:** `TAG=@smoke npx playwright test` (tags: `@smoke`, `@quirk`, `@slow`, `@network`, `@a11y`, `@perf`, `@visual`)

**Skip slow tests:** `SEM_TAG=@slow npx playwright test`

**Single file:** `npx playwright test tests/authenticated/cart-and-checkout.spec.ts`

**Lint:** `npm run lint`

**Typecheck:** `npm run typecheck`

**Open last HTML report:** `npx playwright show-report`

3. **Exploring with MCP**

`.mcp.json` already configures the `@playwright/mcp` server. In Claude Code, ask for the `explore-and-generate-tests` skill on a page or flow: it explores, maps and drafts the tests for review.

4. **Triage failures**

```bash
npm test                  # writes test-results/results.json
npm run triage            # bug-reports/BUG-*.md, summary.md, slack-payload.json
npm run triage:example    # same, from a committed sample report
```

5. **Performance (needs [k6](https://grafana.com/docs/k6/latest/set-up/install-k6/) and Chrome)**

```bash
k6 run perf/k6/inventory-browser.js --summary-export=perf/k6/summary.json
npm run k6:summary
```

## 📊 CI/CD Workflow

The automation runs on **Ubuntu-latest** via **GitHub Actions**:

**Trigger:** every push/PR to `main`/`master`.

**Execution:** `npm ci` → lint → typecheck → install/cache Playwright browsers → `npm test` across Chromium, Firefox, and WebKit.

**Artifacts:** uploads the Playwright HTML report for review, even if tests fail.

### 🖼️ Visual regression

`tests/authenticated/visual-regression.spec.ts` exists and is wired to run, but its baseline screenshots are **deliberately not committed yet**. Generating a first baseline without a human reviewing it would freeze whatever the page looks like today as "correct" — including any real bug already on the page. So right now:

* `playwright.yml` runs both the push/PR step and the nightly all-browsers step with `SEM_TAG='@visual'`, which excludes these tests from the badge you see above.
* A separate workflow, `.github/workflows/visual-regression.yml`, is manual-only (`workflow_dispatch`) with an `update_baselines` checkbox.

To turn visual regression on for real:

1. Run the **Visual Regression** workflow from the Actions tab with `update_baselines` checked. It runs on the same `ubuntu-latest` runner as the rest of CI, which matters — screenshots taken on a local macOS/Windows machine render fonts differently and will not match what CI produces.
2. Download the `visual-baselines` artifact from that run and look at every image. This step is the whole point — it's the human review that stops a bug from being baked in as "correct".
3. Commit the reviewed PNGs to `tests/authenticated/visual-regression.spec.ts-snapshots/`.
4. Remove both `SEM_TAG: '@visual'` blocks from `playwright.yml` so these tests join the regular push/PR and nightly runs.

## 👤 Author

**Gustavo Mesquita** - Senior QA Automation Engineer (SDET)

- [LinkedIn](https://www.linkedin.com/in/qa-gustavo-mesquita/)
- [GitHub](https://github.com/gustavaom7)

_Built with Playwright, TypeScript and AI agents (Playwright MCP + Claude Code)._

---

## ⚠️ Known Limitations (what does NOT work, and why)

* **No API login.** SauceDemo has no authentication endpoint, so the session is built through the UI once in `auth.setup.ts`. Login-via-API is deliberately not used.
* **No API/network mocking of business data.** The app is a static SPA with no backend calls, so network tests break assets (images, fonts) rather than mocking responses.
* **Deep links answer HTTP 404.** `GET /inventory.html` returns 404 and relies on a fallback page to render the app. A `@quirk` test pins this; tests that check status codes of documents must account for it.
* **The route changes before the view renders.** Assertions right after navigation can race with the render. Locators shared across views are scoped to their container and list reads wait for the inventory first.
* **`--grep` does not filter dependency projects.** Use the `TAG` / `SEM_TAG` environment variables.
* **Firefox has no mobile emulation** (`isMobile` is unsupported), so mobile runs on Chromium (Pixel 5) and WebKit (iPhone 12).
* **Quirk tests pass while the bug exists.** `tests/public/quirks.spec.ts` pins documented SauceDemo bugs; a failing quirk test means the site fixed something, not that the suite broke.
* **Visual regression has tests, but no reviewed baseline yet**, so it doesn't gate CI. See "Visual regression" above for exactly why and how to change that.
* **The k6 workflow has not run in CI yet.** Run it once (Actions -> Performance (k6) -> Run workflow) and tune `perf/k6/thresholds.json` to what the site really delivers before trusting the gate.
* **The skills are instructions, not guarantees.** Generated tests and bug reports depend on the model and are drafts until reviewed.
* **Example bug reports come from a synthetic fixture**, not from real failures of the suite.
