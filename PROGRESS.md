# PROGRESS.md

Context reference for QddFi (household finance tracker). Read before making changes — durable architecture, known bugs, and operational gotchas only. This file does not track session-to-session status; check `git log` for current state.

## 1. Project overview

QddFi: household finance tracker (budgeting, transaction ledger, CPF contribution tracking, income tax estimate, insurance/contract/loan tracking, dashboards) for a couple to log together.

**Stack — single file, no build step:**
- `index.html` (~6,860 lines): all markup, styles, app code.
- React 18 + ReactDOM via `<script>` CDN (unpkg, production builds), no JSX — `React.createElement(...)`, aliased `rc(...)`.
- Recharts 2.12.0, SheetJS `xlsx`, also CDN `<script>`.
- Firebase v8 "classic" SDK (not v9+ — avoids ES-module loading issues from plain `<script>` tag).
- Firestore only backend, no server code. Offline persistence enabled (`enablePersistence`).
- Auth: Google Sign-in via Firebase Auth, gated by hardcoded `ALLOWED_EMAILS` (`eng.teck.93@gmail.com`, `siruchew@gmail.com`) — intentional personal/trial setup. Firestore security rules mirror the same allowlist, already use `{householdId}` as wildcard path.
- Hosting: GitHub Pages, repo `https://github.com/qdfi0/QddFi.git`, live at `https://qdfi0.github.io/QddFi/`.

**Firestore data model:**
- Household document ID is user-chosen/generated (join-code system, see §2) — not hardcoded.
- Per household doc: `records`, `ledgerStart`, `assumptions`, `budgets`, `contracts`, `policies`, `recurring`, `familyMembers`, `categories` (`{expense, income}`), `accounts` (`{liquidCash, investments, property}`), `openingBalances`, `dismissedHints`, `cpfSettings`, `taxSettings`.
- Transactions: separate subcollection `households/{id}/transactions`, one doc per transaction — enables concurrent writes from both household members without whole-document overwrite (`persistAll`) clobbering.

## 2. Architecture — locked decisions

**Ledger model:** double-entry-like, simplified for the user. `sealedRecords` = read-only historical months. `ledgerStart` = date from which months are **derived**, not stored: `deriveRecord()` replays every transaction dated in that month against a running balance. `computeBalances(registry, opening, txns, asOf)` — `registry` is a flattened list of every account (bank, investment, `cpfPerson1.oa/sa/ma`, `cpfPerson2.oa/sa/ma`, property/loan), each keyed like `liquidCash.bankAccount1`. New households bootstrap directly into the live ledger (`ledgerStart = today`, `sealedRecords: []`) — no backfill period.

**Household identity (join-code system):** `LEDGER_DOC`/`TXN_COL` are `let`, resolved at runtime by `resolveHousehold(name)` (index.html:95) — not hardcoded consts. Code stored in `localStorage["householdName"]`. Household creation: `generateHouseholdCode()` (index.html:89). Sanitization: `normalizeHouseholdName()` (index.html:83). Switching household = full page reload (avoids manual Firestore listener cleanup). No migration layer for the old hardcoded `households/etsy` doc — deliberately abandoned, don't build one without checking with the user first.

**Reset feature:** Settings → Household → "Reset household data" deletes the `transactions` subcollection and overwrites the household doc with `blankHouseholdDoc()` (single shared source of truth for a blank household).

**Key renames (fully migrated, flagging in case anything was missed):** `uob/dbsEtsy/dbsEt/syReserve`→`bankAccount1..4`, `endowusEt/endowusSy/moomooEt/moomooSy`→`investmentAccount1..4`, `cpfEt/cpfSy`→`cpfPerson1/cpfPerson2`, `payrollEt`→`primaryIncome`, `piano`→`secondaryIncome`, plus matching `cpfSettings`/`assumptions` field renames.

**Tab navigation (sidebar drawer):** Only `dashboard` and `daily` render as always-visible top-nav buttons (`PRIMARY_TABS`, declared just above `function App()`). Every other tab (`OTHER_TABS`: budgets, contracts, insurance, recurring, loans, cpf, tax, forecast, history, settings) lives behind a "☰ More" toggle (`sidebarOpen` state in `App`) that opens a fixed-position right-side drawer; clicking an item sets `tab` and closes the drawer. Adding a new secondary tab = append to `OTHER_TABS`, no other wiring needed. The `tab === "x" && ...` routing itself is unchanged — this was a nav-chrome-only change.

**Daily ledger quick-add (redesign shipped):** Account field is always a plain `<select>` (`AccountSelect`) — the old chip/pill quick-picks and "More…" toggle are gone. "Who" only renders when `type === "income"` (silently defaults to `"Joint"` otherwise) since CPF/tax attribution is the only thing that needs it. Categories can be pinned via a `pinned` boolean, set from Settings → Categories ("Pin to top" toggle next to Archive); pinned categories sort first in the ledger's category tile grid (`sortedCatList` in `TransactionsTab`). The From/To/Account/Search filter row collapses behind a "🔎 Filters" pill (`showFilters` state). The on-screen tap keypad (`.qty-keypad` CSS class) is hidden by media query on desktop/wide viewports (`@media (max-width: 640px), (pointer: coarse)`) since the Amount `<input>` already takes typed keyboard input directly — kept only for touch/mobile widths.

**Live balances (moved to Dashboard):** The account-balance breakdown + "Adjust" (reconcile) flow moved out of `TransactionsTab` into `App`'s dashboard render, as a "💰 Live balances · net $X" pill (`showLiveBalances` state) sitting above the Liquid cash KPI card. Expanding it shows the same grouped-by-account breakdown as before. `ReconcileDialog` is now driven from `App`-level state (`reconcileTarget`) rather than living inside `TransactionsTab`.

## 3. Known bugs — confirmed, not yet fixed

Fix order: #2 first (clearer scope), then #1 (needs user input first), then get end-to-end verification on the live authenticated site (sign-in, real transactions, confirm CPF/tax numbers).

1. **AW ceiling uses YTD OW instead of full-year OW.** `computeCpfContribution()`, index.html:893: `const awCeiling = Math.max(CPF_ANNUAL_CEILING - annualOwToDateExclThisMonth - owSubject, 0);`. Intended behavior per user: AW ceiling is fixed from latest YA CPF data; YTD income tracks against it to check if it'll be met. **Reconcile with user before coding** — original bug framing may not match intended fix.
2. **Tax CPF relief hardcodes flat 20% employee rate, ignores bonus.** index.html:3642: `const cpfReliefAmt = taxSettings.includeCpfRelief ? Math.min(0.20 * Math.min(cpfSettings.person1Salary, CPF_OW_CEILING) * 12, 37740) : 0;`. Should use age-banded CPF tiers (reuse existing `contribBand`/`allocBand` logic) and include AW, not just OW. Peg to latest YA tax-relief rules (currently referencing YA2026); needs annual maintenance.
3. **Self-employed MediSave flat-rate bug — not present in `index.html`.** Confirmed via full-file search; no self-employed MediSave logic exists here. No action needed unless it resurfaces in a file the user provides.

## 4. In-progress / not yet integrated

- **`ledger-rates.js` / `ledger-core.js` / test suite do not exist** — confirmed never actually created. Bugs in §3 must be fixed directly in `index.html`, not merged from elsewhere.

## 5. Testing approach (no build step, no test runner)

No dev server, bundler, or test suite — static file only.

1. **Bracket-balance check before opening a browser**, after any large text removal/replacement. Python script scans the last `<script>` block, treats `"`/`'`/`` ` ``-delimited spans as opaque, sums `( { [` vs `) } ]`. Compare against same scan on last known-good commit (`git show <sha>:index.html`) — should roughly match, but this scanner has known false positives (regex literals get misread as string delimiters) and the mismatch count can shift with unrelated edits even when the code is valid. Treat it as a coarse smoke test, not proof — the `debug.html` run in step 2 (checking `read_console_messages` for real parse/runtime errors, not just permission-denied noise) is the actual authority on whether the file is syntactically sound.
2. **Temporary `debug.html` inside the project folder** (`cp index.html debug.html`), skip auth by changing the final render line from `ReactDOM.createRoot(...).render(React.createElement(LoginGate, null, React.createElement(App, null)))` to `ReactDOM.createRoot(...).render(React.createElement(App, null))` (Firebase Auth requires http/https, won't work over `file://`).
   - Files opened via `file://` **outside** the project folder render as inert static snapshots in the Claude Browser tool — copy must be inside the project dir for a scriptable page.
   - `localStorage.setItem("householdName", "<fresh-name>")` to skip the household chooser — always use a name not used earlier in the same browser session (`localStorage` persists across `navigate()`; `localStorage.clear()` doesn't reliably force a fresh React mount — use a new value or new tab).
   - Unauthenticated Firestore calls fail with `permission-denied`, which the app handles gracefully (local/seed fallback + banner) — fine for UI/logic testing, can't verify real authenticated read/write.
   - **Always `rm -f debug.html` before finishing** — never commit it.
   - `navigate()`/`screenshot()` on `file://` pages can be flaky (stale tab state, wrong origin). Prefer querying the live DOM directly (`document.getElementById('root').innerText`, `localStorage.getItem(...)`, `[...document.querySelectorAll('button')].find(...).click()`) over trusting a screenshot. Rapid synchronous `.click()` calls in the same tick don't reliably simulate distinct user actions with React re-renders between them — insert `setTimeout` (~50ms) between dependent simulated clicks.
3. **Real production debugging**: app's `window.onerror` shows only generic "Script error." for cross-origin CDN script exceptions (Firebase, React, etc.). Use Safari Web Inspector → JavaScript Console (Settings → Advanced → enable "Show features for web developers" → Develop → Show JavaScript Console) for the real file/line/message.

## 6. Fragile / easy to break

- **Never run `git config` yourself.** If git identity is missing, ask the user to run it themselves **from inside the project directory** (`cd /Users/ethan/Documents/QddFi` first) — running from home directory fails semi-silently (`fatal: not in a git directory`).
- **`git push` doesn't work from the Bash-tool environment** — no credential helper available, even on the same Mac where GitHub Desktop works. Route pushes through GitHub Desktop, verify after via `git fetch origin main && git log --oneline -3 origin/main` compared against local `HEAD` — don't trust a verbal "done."
- **Single ~6,860-line file, zero build step, zero linting.** Syntax errors only surface at runtime, often just as "Script error." (see §5.3). Large text-block removals are highest risk — always run the bracket-balance check (§5.1) immediately after.
- **GitHub Pages caching** — after a confirmed push, the live page can still serve stale content briefly. Check the commit landed before debugging application logic for a reported "I don't see the change."
- **No migration layer exists for pre-rename Firestore data.** Don't build one without checking with the user first — abandoning old-schema data was deliberate.
- **Firebase config, API keys, `ALLOWED_EMAILS` are intentionally hardcoded**, not a bug. Don't harden/env-var-ize without being asked — fine for current personal-trial phase.
