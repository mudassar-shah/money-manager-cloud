# Money Manager — Technical Documentation

This is the source of truth for how this software works. It exists so that every future change is made deliberately, not by guesswork — especially because this handles real financial data.

## Rule zero: the process for making any change

1. **Update this document first** — describe the change (what field/formula/behavior is changing and why) in the relevant section below.
2. **Make the code change** — in whichever of the three copies is affected (see "The Three Copies" below). If the change touches shared logic, apply it to all three.
3. **Run the relevant tests from Section 8 (Test Checklist)** before considering the change done. If the change touches Cloud Sync, the round-trip test in 8.7 is mandatory — this is non-negotiable, because skipping it is exactly what caused the subcategory/order data-loss bug on 2026-07-06/07.
4. **Update the "Change Log" at the bottom of this document** with the date and a one-line summary.

Do not skip step 1. The lesson from this project so far: every real bug came from changing code without first checking it against a written contract of "what fields exist and where do they need to survive."

---

## 1. The Three Copies

| Copy | Location | Storage | Use case |
|---|---|---|---|
| **Cloud** | `E:\Claude\money-manager-cloud\index.html` (single file) | Browser localStorage + optional Google Sheets sync | Primary, live on Netlify, actively used |
| **PC** | `E:\Claude\money-manager\index.html` + `app.js` + `style.css` | Browser localStorage only | Offline backup copy, no sync |
| **iPhone** | `E:\Claude\money-manager\Money Manager (iPhone).html` (single file) | Browser localStorage only | Offline backup copy, no sync |

All three share **identical business logic and formulas**. Only the Cloud copy has the extra Cloud Sync section (Google Sheets read/write). When a formula or business rule changes, it must change in all three, worded identically enough that this document applies to all of them without exception.

---

## 2. Data Model

### 2.1 Category
```
{ id, name, type: "income" | "expense", color, parentId: string|null, order: number, bucket: "personal" | "business" }
```
- `parentId` — the id of another category, making this a subcategory. `null`/absent means top-level.
- `order` — integer position among siblings (same `type` + `parentId` + `bucket`). Lower = higher in the list. Assigned automatically on creation (`nextCategoryOrder`), changed by the ↑/↓ buttons (`moveCategory`).
- `bucket` — which ledger this category belongs to. Personal and Business categories are never mixed in the same list or dropdown.

**Invariant — must never be violated:** A category that already has its own subcategories (i.e. some other category's `parentId` points to it) cannot itself be given a `parentId`. If this happens, its own children become invisible (the UI only renders two levels deep). This is blocked at entry (`catForm` submit handler) and auto-repaired on every load (`repairOrphanedParents`) as a safety net, but the block is the real fix — the repair only exists in case it slips through anyway.

### 2.2 Account
```
{ id, name, type: "bank" | "cash" | "savings" | "credit_card", openingBalance, openingDate, color, bucket: "personal" | "business" }
```

### 2.3 Transaction
```
{ id, type: "income" | "expense", bucket: "personal" | "business", date, accountId, categoryId, amount, description, isReconciliation?: true }
```
- `isReconciliation` is only set on auto-generated adjustment transactions created by the Reconcile feature (Section 3.6).
- `bucket` — like Category (2.1), the Transactions tab shows one ledger at a time via a Personal/Business toggle; a new transaction always takes the bucket of the currently active toggle, not a per-transaction picker.
- **Bulk move account**: the Transactions tab's multi-select bar has a "Move to Account" action for fixing mis-entered transactions (e.g. logged against Cash instead of a bank account). It changes only `accountId` on the selected transactions — every other field (`type`, `date`, `categoryId`, `amount`, `description`, `bucket`) is left untouched. Because both the source and target account balances are always computed live from `accountId` matches (Section 3.1), moving a transaction automatically and correctly shifts its effect from the old account's balance to the new one — no separate adjustment or reconciliation entry is created or needed. The target-account dropdown is scoped to the currently active ledger (`transactionsLedger`), so a transaction can never be moved into an account belonging to the other bucket. Any reconciliation history (Section 2.5) tied to the *old* account is left as-is — it's a historical log of what was checked at the time, not a live calculation, so moving transactions after the fact does not retroactively change it.
- **Category filter rollup**: the Transactions tab's Category filter dropdown rolls up like the Budget rollup (Section 3.5) — selecting a *parent* category (e.g. "Transport") shows transactions logged against that parent **and** all its direct subcategories, via `categoryAndChildIds()`. Selecting a *subcategory* directly (e.g. "Transport › Fuel Car") shows only that subcategory's own transactions, since a subcategory has no children of its own (one level of nesting only, Section 2.1).
- **Add Transaction form — sticky Type/Date/Account/Category**: after saving a transaction (add or edit), the form no longer resets Type, Date, Account, or Category back to defaults — they stay exactly as last set, so entering several transactions in a row for the same account/category/date/type doesn't require reselecting them each time. Only **Amount** and **Description** clear after save, since those are unique per transaction. Implemented by no longer calling the form's native `reset()`; the Account/Category `<select>` elements are still repopulated fresh (to reflect any category/account added meanwhile) but their `.value` is explicitly restored to what was just used, immediately after that repopulation.
- **Search box matches amount and category too**: the Transactions tab's free-text search box (`filterSearch`) checks the query as a substring against the transaction's `description`, its `amount` (as a plain number string, so typing `480` matches an amount of 480), and its category's display name (via `categoryDisplayName()`, so a parent category name like "Transport" also matches subcategory transactions like "Transport › Fuel Car", since the display string contains both). This is separate from — and in addition to — the exact-match Account/Category/Type filter dropdowns next to it, which are unchanged.

### 2.4 Budget
```
{ id, categoryId, month: "YYYY-MM", amount }
```
- Budgets are **always personal**. The Budgets tab and Yearly Report only ever list personal expense categories (`getOrderedCategoriesFlat("expense", "personal")`). There is currently no business budgeting feature.
- **Copy Budgets from another month**: the Budgets tab has a "Copy from" dropdown + "Copy Budgets" button (`copyBudgetsFromMonth`), so you don't have to retype every category's amount each time you move to a new month. The dropdown lists every month that already has at least one budget, excluding the month currently being viewed, sorted newest-first, and defaults to the previous month if it has budgets (otherwise the most recent available month). On click, a `confirm()` dialog states the source/target month labels and how many budgets will be copied; on confirmation, for every budget in the source month: if the current month already has a budget for that same `categoryId`, its `amount` is overwritten; otherwise a new budget row is created for the current month. Budgets in the current month for categories **not** present in the source month are left untouched — this is a merge/upsert, never a wipe of the current month. Nothing else (transactions, spending totals, other months' budgets) is touched.
- **Undo Copy**: immediately before a copy is applied, the current month's existing budgets are snapshotted into an in-memory variable (`lastBudgetCopy`, not persisted to `localStorage` or Cloud Sync — it only lives for the current browser session/tab). Once a copy completes, an "Undo Copy" button appears next to "Copy Budgets"; clicking it (after its own `confirm()`) replaces the current month's budgets with that snapshot exactly as they were right before the copy, then clears the snapshot. It only ever undoes the single most recent copy for the month it was taken on — switching to a different month, doing another copy, or reloading the page all make it unavailable again (the button hides itself via the same `lastBudgetCopy.month === budgetMonth` check that reveals it).

### 2.5 Reconciliation
```
{ id, accountId, date, actualBalance, computedBefore, diff }
```
- A historical log entry, written every time you use "Reconcile" on an account — kept even when `diff` is 0 (no adjustment needed), so you have a record of when you last checked.

### 2.6 Settings
```
{ currency: "PKR" | "USD" | "EUR" | "GBP" | "INR" | "AED" }
```

---

## 3. Formulas — exact definitions

### 3.1 Account balance
```
accountCurrentBalance(accountId) = account.openingBalance + Σ(transaction.amount, income = +, expense = −)  [ALL transactions for that account, regardless of date]
accountBalanceAsOf(accountId, date) = same, but only transactions with date <= given date
accountBalanceBefore(accountId, date) = same, but only transactions with date < given date  (used as "opening balance" for a period)
```

### 3.2 Total Net Worth / Total Business Balance
```
totalNetWorth(bucket) = Σ accountCurrentBalance(a.id)  for every account where account.bucket === bucket
```
The personal figure (`totalNetWorth("personal")`) is shown as **"Total Balance (Personal)"** on the **Accounts tab** (Section 3.3a) — not on the Dashboard. The business figure (`totalNetWorth("business")`) is shown as "Total Business Balance" on the Business tab (Section 3.8), same as always. These never overlap.

### 3.3 Dashboard monthly figures (personal only)
```
monthTx = all transactions where monthKey(date) === selected month AND bucket === "personal"
Income  = Σ amount where type === "income"
Expense = Σ amount where type === "expense"
Net This Month = Income − Expense
Savings Rate = (Income − Expense) / Income × 100   (0% if Income is 0)
```
The Dashboard shows exactly these four stats (Income, Expenses, Net This Month, Savings Rate) — all month-scoped, none of them reveal your real running account balance. **"Total Balance" is intentionally not on the Dashboard** (see 3.3a) — this was a deliberate privacy choice, not an oversight.

### 3.3a Total Balance moved to the Accounts tab (privacy)
The Dashboard is the first screen anyone sees when opening the app, so a user asked to remove the real all-time net worth figure from it — someone glancing at the phone/screen would otherwise see the actual total balance without doing anything. "Total Balance (Personal)" (`totalNetWorth("personal")`) now lives on the **Accounts tab** instead, at the top of the "Your Accounts" card, above the individual account list — requiring a deliberate tab switch to see it, rather than being visible by default. This is a privacy deterrent against a casual glance, not real security: the Accounts tab has no PIN/lock (a PIN-based lock was considered and explicitly declined as unnecessary complexity), and the Dashboard's own "Accounts" preview widget and "Recent Transactions" list still show real per-account/per-transaction amounts, unchanged. The Business tab's "Total Business Balance" was intentionally left as-is on the Business tab itself, since the user considers Business a separate, lower-sensitivity context.

### 3.4 Spending-by-category donut (Dashboard, Business tab, Yearly Report)
```
For each category with at least one expense transaction in the period:
  categoryTotal = Σ amount of that category's expense transactions in the period
  percentage = categoryTotal / (Σ all categoryTotals in the period) × 100
```
This is a **share of that period's total expenses** — not a share of income, and not compared against any budget. Subcategories are their own slice; they are never merged into their parent's total.

**Long category names**: the legend row (color dot + name + amount/percentage) truncates a long name with an ellipsis and shows the full name on hover (`title` attribute) rather than letting it wrap — the same pattern used for the Budget Progress list (Section 3.5). Without this, a long name like "House Hold & food › Groceries / Ration" would wrap to two lines and push the amount/percentage onto its own wrapped line too, since the row has no width to grow into.

### 3.5 Budget progress (per category, and the Total row)
```
spent = Σ expense transactions in (categoryId OR any of its direct subcategories) for the selected month
percentage = min(100, spent / budgetAmount × 100)
color = red if spent > budgetAmount, amber if percentage > 80%, else green

Total row:
  totalBudget = Σ budgetAmount across all budgeted categories that month
  totalSpent  = Σ spent across those same categories (rolled up as above)
  remaining   = totalBudget − totalSpent   (shown as "Remaining: X" if ≥ 0, "Over by X" if negative)
```
The Total row only includes categories that actually have a budget set that month — it is not "all expenses," it is "all budgeted expenses vs. their budgets."

**Rollup rule:** if you budget a parent category (e.g. "Food"), spending logged against its subcategories (e.g. "Groceries") counts toward that budget too, via `categoryAndChildIds(catId)`. If you instead budget the subcategory directly, only that subcategory's own spending counts. **Do not set a budget on both a parent and its own subcategory for the same month** — spending on the subcategory would then be counted in both budgets' totals (double-counted in the sum of all Total rows, though each individual row is still internally correct). Pick one level per category tree.

### 3.6 Reconciliation
```
computed = accountBalanceAsOf(accountId, statementDate)
diff = actualBalance − computed
if |diff| >= 0.01:
    create a transaction: type = income if diff > 0 else expense, amount = |diff|,
    category = "Balance Adjustment" (auto-created the first time, per type), isReconciliation = true
```
This is the only mechanism that creates a transaction automatically. It exists to correct drift between what the app calculated and your actual bank/cash balance.

The adjustment transaction (and its "Balance Adjustment" category) always take the **same `bucket` as the account being reconciled** — reconciling a business account creates a business-bucket adjustment, so it shows up in the Business tab, never in the personal Dashboard/Yearly Report, and vice versa. `repairReconciliationBuckets()` runs on every `loadData()` (and after every Cloud Sync pull) to self-heal any reconciliation transaction whose bucket doesn't match its account, in case one was ever saved incorrectly.

**Important subtlety, verified by testing (Section 8.5):** the adjustment is dated *at the statement date you reconciled for*, not "today." So `accountBalanceAsOf(accountId, statementDate)` will equal your entered actual balance immediately after — but `accountCurrentBalance` (all-time) will only equal it too if there are no transactions dated *after* the statement date. If you reconcile a past date while newer transactions already exist, the all-time balance correctly continues to include those newer transactions on top of the correction. This is expected, not a bug — reconciling "as of 10 July" does not claim anything about your balance today if you've logged transactions since.

### 3.7 Fiscal Year (Yearly Report)
```
Fiscal year runs 1 July – 30 June (Pakistan-style).
fiscalYearStartOf(monthKey) = "{year}-07" where year is the current year if month >= July, else previous year.
The 12 months of that FY = that start month + the next 11 months.
All FY totals (income, expense, net, savings rate, category donut, monthly bar chart, annual budget vs actual,
account opening/closing) are computed by filtering to those exact 12 months — always personal only, including
the "Account Summary — Opening vs Closing" table, which lists personal accounts only.
```
Account opening balance for the FY = `accountBalanceBefore(accountId, "{FYstart}-01")`.
Account closing balance for the FY = `accountBalanceBefore(accountId, "{month after FY end}-01")` (i.e. before 1 July of the following year).

### 3.8 Business tab
Identical formulas to Dashboard (3.3) and the donut/bar charts (3.4), but every filter uses `bucket === "business"` instead of `"personal"`, and there is no savings-rate stat (replaced with "Net Profit" = Income − Expense) and no budget section.

### 3.9 Category ordering
```
Categories are grouped into "sibling groups": same type + same parentId + same bucket.
Within a group, `order` values are unique integers starting at 0.
Move Up/Down (moveCategory) swaps the `order` value with the adjacent sibling in that same group.
New categories get order = (max existing order in their group) + 1  (nextCategoryOrder) — i.e. added to the bottom.
```

---

## 4. Business Rules / Invariants (must hold at all times)

1. A category with subcategories cannot itself have a `parentId` (Section 2.1).
2. Every category, account, and transaction has a `bucket` — `"personal"` or `"business"` — and personal/business data never appear in each other's reports (Dashboard, Budgets, Yearly Report are personal-only; Business tab is business-only). The **Categories** and **Transactions** tabs each show only one ledger at a time via a Personal/Business toggle (`categoriesLedger` / `transactionsLedger`) — there is no "all ledgers" mixed view for either, and new items always take the bucket of whichever toggle is currently active (no per-item ledger picker in the Add form).
3. Deleting a category or account is blocked if any transaction or budget still references it — the app never silently orphans a transaction.
4. `order` values are unique within a sibling group; `ensureCategoryOrder` repairs any category missing an `order` value on every load.
5. Every field that exists in the local data model and is used anywhere in the app **must** also exist in the Cloud Sync schema (Section 5) if it needs to survive syncing. This is the single most important rule in this document — see Section 6.
6. Any transaction/category created automatically by the app on your behalf (currently: only reconciliation adjustments, Section 3.6) must be tagged with the same `bucket` as whatever triggered it — auto-created records are just as capable of leaking across the personal/business boundary as manually entered ones, and are easier to miss during testing because the user didn't type them in directly.

---

## 5. Cloud Sync Schema (Cloud copy only)

Your Google Sheet has one tab per entity. The exact columns, in order, that get written and read:

| Sheet tab | Columns |
|---|---|
| `Categories` | `id, name, type, color, parentId, order, bucket` |
| `Accounts` | `id, name, type, openingBalance, openingDate, color, bucket` |
| `Transactions` | `id, type, date, accountId, categoryId, amount, description, isReconciliation, bucket` |
| `Budgets` | `id, categoryId, month, amount` |
| `Reconciliations` | `id, accountId, date, actualBalance, computedBefore, diff` |
| `Settings` | `key, value` |

How it works:
- **Writing** (`buildRowsForTab`) is fully generic — it writes whatever columns are listed in `SHEET_TABS` for that tab, reading each column name directly off the in-memory object. Adding a column to `SHEET_TABS` is enough to make it get written.
- **Reading back** (`coerceTypesFromSheet`) is **not generic** — each tab has a hand-written mapping that must explicitly name every column you want back, with the right type conversion (numbers via `parseFloat`/`Number`, booleans via string comparison, missing/blank values defaulted sensibly). **Adding a column to `SHEET_TABS` alone does nothing on the read side — you must also update `coerceTypesFromSheet` by hand**, or that field will silently come back empty every time, even though it looks like it's syncing.

This exact gap — a field added to the write side but not the read side — is what caused the subcategory/order data loss. Section 8.7 is the mandatory test for this class of bug.

---

## 6. Why fields go missing after sync (root cause, and how to never repeat it)

When a new field is added to categories, accounts, or transactions (like `parentId`, `order`, or `bucket` were, in that order, across this project's history):
1. It's easy to update the object shape and every place that *uses* the field locally, and forget the Cloud Sync layer entirely, because the app works perfectly in the browser without ever touching Sheets.
2. The bug is invisible until an actual sync round-trip happens — which may be minutes or days later — by which point it looks like "my data disappeared" rather than "a column was never wired up."
3. **The fix going forward:** any time a field is added anywhere in the data model, immediately open Section 5 of this document, add it to the table, then update both `SHEET_TABS` and `coerceTypesFromSheet`, then run the Section 8.7 test before doing anything else with that feature.

---

## 7. Known Limitations (by design, not bugs)

- **PC and iPhone copies never sync with the Cloud copy or each other.** Each is a fully separate local dataset. Moving data between them requires Export Backup → Import Backup manually.
- **No true multi-device conflict resolution.** If you edit on two devices without syncing in between, importing/pulling one will fully overwrite the other — there is no merge.
- **Business tab has no budgeting.** Budgets and Yearly Report are personal-only by design.
- **Cloud Sync requires you to have completed the Google Cloud Console + Google Sheet setup once** (Client ID + Sheet ID entered in the Cloud Sync tab). This is per-person — see Section 9 for how someone else would set up their own.

---

## 8. Test Checklist (run after any change to the relevant area)

**8.1 Account balances** — add an opening balance, add income/expense transactions dated before/after a given date, confirm `accountCurrentBalance`, `accountBalanceAsOf`, `accountBalanceBefore` all match hand-calculated totals.

**8.2 Dashboard** — confirm Income/Expense/Savings Rate/Net This Month match the sum of that month's personal transactions only (Net This Month = Income − Expense exactly); confirm a Business transaction does NOT appear in these numbers; confirm there is no "Total Balance" card on the Dashboard at all (moved to Accounts tab, Section 3.3a).

**8.3 Budgets** — set a budget, log expenses under and over it, confirm the progress bar color (green/amber/red) and the Total row's Remaining/Over-by math.

**8.4 Yearly Report** — confirm the 12 months shown always run July–June regardless of which month "today" is; confirm prev/next FY navigation shifts by exactly 12 months; confirm account opening balance for a FY equals the previous FY's closing balance.

**8.5 Reconciliation** — reconcile an account with a deliberate mismatch, confirm the adjustment transaction's amount equals `|actualBalance − computedBefore|` and its type (income/expense) matches the sign of the difference; reconcile with no mismatch and confirm no transaction is created but the reconciliation record is still saved.

**8.6 Category structure** — create a subcategory, try to give a category-with-children a parent (must be blocked with an alert), reorder with ↑/↓ and confirm the order persists after a page reload.

**8.7 Cloud Sync round-trip (mandatory after ANY data model change)** — for every field on Categories, Accounts, and Transactions, simulate `buildRowsForTab` → row/object conversion → `coerceTypesFromSheet`, and confirm every field survives with the correct type. Do this for both a personal and a business record. Never skip this.

**8.8 Business tab isolation** — add a business transaction, confirm it changes Business tab numbers but leaves Dashboard, Budgets, and Yearly Report numbers exactly unchanged.

**8.9 Multi-select delete** — select several transactions/categories including one that's in use, confirm the in-use one is skipped and reported, and the others are deleted.

**8.10 Bulk move account** — select several transactions on one account, move them to a different account (same ledger), confirm: the old account's balance decreases by exactly the moved transactions' net effect, the new account's balance increases by the same amount, `accountId` is the only field that changed on each transaction (type/date/category/amount/description/bucket all untouched), and the target-account dropdown only ever lists accounts in the currently active ledger.

**8.11 Copy Budgets from another month** — set budgets in one month, move to a month with no budgets and confirm the "Copy from" row is hidden; go back and confirm it reappears once budgets exist elsewhere; copy from the source month and confirm every budgeted category's amount appears in the target month while un-budgeted categories stay blank; set a different amount on one category in the target month, copy again, and confirm it gets overwritten to match the source; confirm the dropdown defaults to the previous month when it has budgets, or the most recent available month otherwise.

**8.12 Undo Copy** — confirm "Undo Copy" is hidden before any copy has happened; note a category's current amount, copy budgets into that month (overwriting it), confirm "Undo Copy" appears and clicking it (after confirming the prompt) restores that exact pre-copy amount and hides the button again; after a fresh copy, navigate to a different month and confirm "Undo Copy" is hidden there, then navigate back to the copied-into month and confirm it reappears (state persists per-month for the session); reload the page and confirm "Undo Copy" no longer appears anywhere (the snapshot is in-memory only, not persisted).

**8.13 Sticky Add Transaction fields** — add an Income transaction dated in the past against a specific account/category, confirm the form still shows that same Type/Date/Account/Category afterward (Amount/Description cleared); add another transaction without changing anything but Amount, confirm it saves against the same Type/Date/Account/Category; edit an existing transaction, change its account, save, and confirm the form now shows that new account (not the pre-edit one) ready for the next entry; switch the Personal/Business ledger toggle and confirm the form fully resets to today's date/first account/first category as before (this reset behavior is unchanged, only the post-save behavior changed).

**8.14 Search box matches amount/category** — add a transaction with a distinctive amount (e.g. 480) and one with a description containing unrelated text, type the amount into the search box and confirm only the matching-amount transaction(s) show; clear it and type a category name (including a parent category that has subcategories) and confirm transactions in that category and its subcategories show; clear it and type text that only appears in a description and confirm that still works too; confirm the separate Account/Category/Type filter dropdowns are unaffected and still do exact-match filtering as before.

**8.15 Net This Month stat** — add income and expense transactions in the same month, confirm "Net This Month" equals Income − Expense exactly; switch months and confirm it recalculates per month; confirm the Business tab's 4-card stat grid is unaffected (still `repeat(4, 1fr)` on desktop) and its "Net Profit" figure is untouched.

**8.16 Total Balance moved to Accounts** — confirm the Dashboard has exactly 4 stat cards (Income, Expenses, Net This Month, Savings Rate) and no Total Balance card anywhere on it; confirm the Accounts tab's "Your Accounts" card shows "Total Balance (Personal)" matching the sum of personal accounts' `accountCurrentBalance`, and that it excludes any business accounts even though the account list below it shows both; confirm switching Dashboard months does not affect the Accounts tab figure at all (it's not month-scoped); confirm the Business tab's "Total Business Balance" is untouched.

---

## 9. Setting Up Your Own Cloud Instance (for a friend, or a second device you want fully synced)

See the separate setup guide (Google Cloud Console + Google Sheet steps) for the full walkthrough. Summary:

1. Deploy the same code (drag the `money-manager-cloud` folder onto Netlify, or reuse the existing URL — the code itself has no user-specific data baked in).
2. Create your **own** Google Sheet (blank — the app creates the needed tabs automatically on first connect).
3. Create your **own** Google Cloud project + OAuth Client ID (this is what makes "Sign in with Google" work, and must be done once per person — you cannot reuse someone else's Client ID and still have your own private data, since the Client ID's Testing-mode test user list controls exactly who can sign in).
4. In the app's Cloud Sync tab, enter your own Client ID and Sheet ID, then Sign in with Google using the same account you set as a test user.

Two different people, each with their own Client ID + Sheet ID entered into the same app, have **completely independent, private data** — nothing is shared just because the app code is the same.

---

## 10. Mobile Layout & Installable App (PWA)

### 10.1 The problem being solved
The Cloud copy's layout was built desktop-first: a fixed sidebar next to the content, using a 900px breakpoint that only stacked the sidebar full-width above the content on narrow screens. On an actual phone (~375–430px wide), that meant the *entire* sidebar — brand, all 8 nav buttons, and the footer (Export/Import/currency/Reset) — rendered as a tall vertical block (~600px+) that had to be scrolled past before any actual page content (Dashboard, Transactions, etc.) came into view. This is the root cause of "it stretches / doesn't look right on iPhone."

### 10.2 The fix — off-canvas mobile drawer
Below a `700px` breakpoint:
- A slim fixed top bar appears (`.mobile-topbar`, hidden above 700px) with a hamburger button (`☰`) and the app name.
- `.sidebar` becomes a fixed, off-canvas panel positioned just off the left edge of the screen (`left: -280px`), sliding into view (`left: 0`) only when opened.
- A semi-transparent backdrop (`.sidebar-backdrop`) appears behind the open drawer; tapping it (or tapping any nav button) closes the drawer.
- `main` content sits directly under the slim top bar and is visible immediately — no more scrolling past the sidebar.
- `toggleMobileMenu()` opens/closes the drawer; it's also called automatically whenever a nav button is clicked, so navigating always closes the drawer behind you.

This is pure CSS (media query + `position: fixed` + a `transform`/`left` transition) plus a small amount of JS state (`mobileMenuOpen`) — no new dependencies, and it has zero effect on the desktop layout since it only activates below the breakpoint.

### 10.3 Installable app (Add to Home Screen / Install app)
The Cloud copy already had the basic iOS meta tags (`apple-mobile-web-app-capable`, viewport). To make it properly installable on both iPhone (Safari → Add to Home Screen) and Android (Chrome → Install app), this adds:
- `manifest.json` — declares the app name, colors, `display: "standalone"` (so it opens without any browser UI), and icon references. Required for Android's "Install app" prompt to appear at all.
- App icons (192×192 and 512×512 PNG for the manifest, 180×180 for `apple-touch-icon`) — a simple rounded square matching the existing sidebar brand mark (₨ on the accent color background), generated once and saved as static files.
- A minimal service worker (`sw.js`) that caches the app shell — this is one of the technical requirements Chrome checks before it will offer the "Install app" prompt, and it also means the app keeps working (on cached data) with no internet connection. The main HTML page uses a **network-first** strategy (try the network, fall back to the cached copy only if there's no connection) specifically so that redeploying the Cloud copy is picked up automatically the next time the installed app is opened with internet access — you never need to manually clear the cache or bump a version number for a code update to reach an already-installed icon. Static assets that rarely change (icons, manifest) stay cache-first for speed.
- None of this touches `localStorage`, Cloud Sync, or any calculation — it is purely about how the page is packaged and presented once installed.

### 10.4 Known scope
This mobile layout work applies to the **Cloud** copy first, since that's the one used in a phone's browser day-to-day. The same CSS/JS was mirrored into the PC and iPhone copies for consistency (per the "identical logic across all three" rule in Section 1) — on PC this has no visible effect since it only activates below 700px width; the standalone iPhone file gains the same drawer and install support as a fallback if Cloud Sync isn't set up.

---

## Change Log

- **2026-07-07** — Added Business ledger (bucket field on categories/accounts/transactions), Business tab, Personal/Business category toggle. Added `bucket`, `parentId`, `order` to Cloud Sync schema (read + write) after discovering they were missing (see Section 6). Added subcategory-with-children guard and auto-repair. Added category drag-free reordering (↑/↓ buttons). Added Yearly Report, Budget Total row, multi-select delete, category name truncation with tooltip. This document created.
- **2026-07-07 (exhaustive formula audit)** — Ran a full pass of Section 8 against all three copies plus the live Cloud data. Found and fixed 4 real bugs, all applied to Cloud/PC/iPhone identically and verified by direct testing:
  1. `renderYearlyView`'s headline income/expense/net/savings-rate were computed from a transaction query with no `bucket` filter — Business transactions leaked into the personal Yearly Report totals. Fixed by adding the personal-only filter.
  2. `renderFyBar` (the Yearly Report's 12-month bar chart) had its own separate, entirely unfiltered transaction query — same leak, independently. Fixed the same way.
  3. **Budget rollup gap**: `renderBudgetsView`, `renderBudgetProgress`, and `renderFyBudgets` all computed "spent" via an exact `categoryId` match, so a budget set on a *parent* category never counted spending logged against its *subcategory* — the budget would show 0% used forever even while you were actively overspending. Fixed via a new `categoryAndChildIds()` helper that rolls up subcategory spending into the parent's budget total (Section 3.5). Checked the live data — no budgets were currently set on parent categories, so this had not corrupted any real numbers yet, but would have as soon as a budget was set at the parent level.
  4. **Reconciliation bucket leak**: reconciling an account created an adjustment transaction (and its "Balance Adjustment" category) with no `bucket` field at all, so it always defaulted to personal — reconciling a *business* account would silently push that adjustment into the personal Dashboard/Yearly Report instead of the Business tab. Fixed by tagging the adjustment with the reconciled account's own bucket (Section 3.6), and added a self-healing `repairReconciliationBuckets()` that runs on every load/sync-pull in case a bad record already exists. Checked the live data — zero reconciliations had been logged yet, so this had not affected any real numbers either.
  - Also independently verified against the live Cloud site's real data (79 categories, 36 transactions, 32 budgets, 2 accounts, one business account "Jazz cash"): net worth, monthly income/expense, Business tab figures, and the Budget Progress Total row were all cross-checked against an independent manual recomputation from the raw transaction list — all matched exactly, and no orphaned-parent or duplicate-id issues were found in the live category structure.
  - **Action required**: this file (`money-manager-cloud/index.html`) has these fixes applied locally but the live Netlify site is still running the *previous* deploy — redeploy to apply them there.
- **2026-07-07 (post-redeploy full re-check)** — After the 4 fixes above were redeployed live, re-ran a check of every tab and function against the live real data (79 categories, 36 transactions, 32 budgets, 2 accounts) via the connected browser session: Dashboard, Business tab, Budgets tab, Yearly Report, Categories tab, Transactions tab (including bucket/type/account filters), Accounts tab, and Cloud Sync tab. Found one more real bug:
  5. **Yearly Report account leak**: `renderFyAccounts` (the "Account Summary — Opening vs Closing" table at the bottom of the Yearly Report) mapped over `db.accounts` with no bucket filter, so the business account ("Jazz cash") appeared in the personal-only Yearly Report — violating the Section 4 rule 2 invariant. Fixed by filtering to personal accounts only, in all three copies, and verified locally (business account correctly excluded from the table after the fix). Section 3.7 updated to state this table is personal-only.
  - Every other formula was independently cross-checked against a manual recomputation from the raw transaction list, all matching exactly: net worth, monthly income/expense, Business tab figures, Budget Progress Total row, donut chart percentages (sum to ~100% modulo expected per-slice rounding), category/account/budget referential integrity (no dangling references, no duplicate IDs, no orphaned parents). All render functions execute with zero JS errors.
  - Redeployed and confirmed live on 2026-07-07 (this fix + Transactions ledger toggle + Bulk move account + Category filter rollup, items below, all went out together).
- **2026-07-07 (Transactions ledger toggle)** — Replaced the Transactions tab's "Ledger" dropdown (a per-transaction picker in the Add form, plus an "All Ledgers/Personal/Business" filter) with a Personal/Business toggle at the top of the tab, matching the Categories tab's existing pattern (Section 4 rule 2). New transactions now always take the bucket of whichever toggle is active — there is no more per-transaction ledger picker, and no "all ledgers mixed together" view. This removes the main way a transaction could be logged under the wrong ledger by mistake (forgetting to change a dropdown), since the currently active toggle is now the only source of truth for which ledger you're adding to. Implemented in all three copies (`transactionsLedger` state + `setTransactionsLedger()`, mirroring `categoriesLedger`/`setCategoriesLedger()`); the filter/account/category dropdowns in the Transactions tab are now also scoped to the active ledger. Verified in each copy: added a personal transaction, switched to Business, confirmed it disappeared from view; added a business transaction, confirmed it didn't appear when switching back to Personal; edited an existing transaction and confirmed its bucket was preserved correctly. Zero console errors in any copy.
- **2026-07-07 (Bulk move account)** — Added a "Move to Account" bulk action to the Transactions tab's multi-select bar (alongside "Delete Selected"), for fixing transactions logged against the wrong account (e.g. entered under Cash when they should've been under a bank account). Changes only `accountId` on the selected transactions; every other field is untouched, and the target-account dropdown is scoped to the active ledger so a transaction can never cross into the other bucket's accounts (Section 2.3). Implemented in all three copies and verified: moved 3 expense transactions (Rs 1,000/2,000/3,000) from a Cash account to a UBL account, confirmed Cash's balance went from -6,000 to 0 and UBL's went from 0 to -6,000 (exact net effect of the move), confirmed type/date/category/amount/bucket/isReconciliation were unchanged on all 3, and confirmed the target dropdown only lists accounts in the currently active ledger. Zero console errors in any copy.
- **2026-07-07 (Category filter rollup)** — Fixed the Transactions tab's Category filter: selecting a *parent* category (e.g. "Transport") previously showed nothing unless transactions were logged against that exact category id, even though real spending was logged against its subcategories (e.g. "Transport › Fuel Car") — the same exact-match bug as the earlier Budget rollup fix (item 3 above), just in a different filter. Fixed using the same `categoryAndChildIds()` helper (Section 2.3). Selecting a subcategory directly still shows only that subcategory's own transactions. Verified in all three copies: created a "Fuel Car" subcategory under "Transport," logged one expense directly against "Transport" and one against "Fuel Car," confirmed the parent filter shows both while the subcategory filter shows only its own. (The earlier "can't change the account" report turned out to be a miscommunication, not a bug — confirmed by the user as working fine; see Section 7 note removed below.)
- **2026-07-07 (confirmed live)** — User redeployed `money-manager-cloud/index.html` to Netlify and confirmed everything is working correctly on the live site: Yearly Report account leak fix, Transactions ledger toggle, Bulk move account, and Category filter rollup are all live and verified against real data. [USER_MANUAL.md](USER_MANUAL.md) created as a plain-language, non-technical guide to using the app (separate from this technical document).
- **2026-07-07 (mobile layout + installable PWA)** — See Section 10 for the full design writeup. Summary: the Cloud copy's layout previously forced the entire sidebar (~600px+) to render above the page content on phone-width screens, which is what made it "not mobile friendly" in Safari. Replaced with a slim top bar + off-canvas slide-out menu (`toggleMobileMenu()`/`closeMobileMenu()`) below 700px width, so page content is visible immediately. Also added `manifest.json`, three PNG icons (`icon-192.png`, `icon-512.png`, `apple-touch-icon.png`, generated via Python/PIL since no browser-based export was reliable for this), a minimal service worker (`sw.js`, same-origin GET requests only — explicitly never intercepts Google Sheets/Sign-in calls), and the `<link rel="manifest">`/theme-color meta tags needed for iPhone's "Add to Home Screen" and Android Chrome's "Install app" to work properly (opens full-screen, own icon, works offline). Mirrored into the PC copy (CSS/JS only — no PWA files, since PC isn't meant to be "installed") and the iPhone copy (full CSS/JS/PWA parity, as a non-synced fallback). Verified in both Cloud and PC/iPhone preview instances: sidebar correctly off-canvas at 375px width, page content visible immediately below the top bar without scrolling past the sidebar, manifest/icons/service worker all fetch with 200 and the service worker registers successfully, transactions table correctly scrolls horizontally inside its own container rather than breaking the page layout. Zero console errors in any copy.
- **Action required**: none of this mobile/PWA work is deployed yet — needs a Netlify redeploy, after which the user should open the live site in iPhone Safari and use "Add to Home Screen" (or "Install app" in Android Chrome) to get the app-like icon and full-screen experience.
- **2026-07-07 (donut legend long-name wrap)** — User reported real long category names (e.g. "House Hold & food › Groceries / Ration") "not showing properly" on the live Dashboard's Spending by Category legend — the same underlying issue as the earlier Budget Progress fix, just never applied to this widget: `.legend-left` had no width constraint, so a long name wrapped to two lines and pushed the amount/percentage onto its own wrapped line too, since neither had `flex-shrink`/`white-space:nowrap`. Fixed with the identical pattern already used for Budget Progress (Section 3.5): the category name is now wrapped in `.legend-name` (`white-space:nowrap; overflow:hidden; text-overflow:ellipsis`, full name in a `title` tooltip) and the amount in `.legend-amount` (`flex-shrink:0; white-space:nowrap`) — Section 3.4 updated. Applied to both places this legend template appears (Dashboard/Business tab and Yearly Report) in all three copies. Verified at a realistic narrow width: the row stays a single 17px-tall line, the name truncates with ellipsis while the full name is available via hover, and the amount/percentage remains fully visible on the same line. Zero console errors in any copy.
- **Action required**: this fix is applied locally but not yet redeployed.
- **2026-07-18 (Copy Budgets from another month)** — Added a "Copy from" dropdown + "Copy Budgets" button to the Budgets tab (Section 2.4, 8.11) so re-entering the same budget amounts every month isn't necessary. Only additive: a new `copyBudgetsFromMonth()` function plus a `populateBudgetCopyOptions()` render step; `setBudget`, the spending/progress math, and every other tab are untouched. Also added a small scoped CSS override (`#budgetCopyRow select, #budgetCopyRow .ghost-btn`) so the new button/dropdown are legible on the light card background — the shared `.ghost-btn`/`.bulk-actions-bar` classes themselves were left unchanged since they're used elsewhere (e.g. Transactions tab bulk bar) and are already low-contrast there too; not in scope to fix as part of this change. Verified live in the browser for the **Cloud** copy: set two budgets in July 2026, moved to August 2026, confirmed the copy row appeared defaulting to "July 2026," copied and got both budgeted categories with the right amounts while un-budgeted categories stayed blank, confirmed the copy row is hidden on a month with no other budgeted months anywhere, and confirmed the data survives a page reload. Zero console errors. Mirrored the identical change (same function bodies, same element IDs) into the **PC** copy (`D:\money-manager\index.html` / `app.js` / `style.css`) and the **iPhone** copy (`D:\money-manager\Money Manager (iPhone).html`) once those folders were made accessible in this session — these two could not be exercised live in-browser here (no Python/Node available in this environment to serve the PC copy's multi-file structure, and the browser tool sandboxes local files outside the Cloud copy's project folder), so double-check the Copy Budgets button once on the actual PC/iPhone devices per Section 8.11.
- **Action required**: redeploy `money-manager-cloud/index.html` to Netlify for this to go live at the real URL.
- **User-confirmed 2026-07-18**: Copy Budgets tested and working on the PC copy on the user's actual device.
- **2026-07-18 (Undo Copy)** — Added an "Undo Copy" button next to "Copy Budgets" (Section 2.4, 8.12): the user pointed out that an accidental copy into a month with many existing entries had no way back. Right before a copy runs, the target month's existing budgets are snapshotted into an in-memory `lastBudgetCopy` variable (never written to `localStorage` or synced — session-only). The button appears only when `lastBudgetCopy.month === budgetMonth`, and clicking it (behind its own `confirm()`) replaces the current month's budgets with that snapshot and clears it — a single-level, same-session undo, not a full history. Implemented identically in all three copies (Cloud/PC/iPhone). Verified live in the browser (Cloud copy): manually changed a category's budget after a prior copy, ran a new copy to overwrite it, confirmed "Undo Copy" appeared, clicked it, and confirmed the exact pre-copy value came back and the button disappeared again; confirmed the button is hidden on a different month and reappears when navigating back to the month the copy happened in; zero console errors. Not live-tested for PC/iPhone in this session (same environment limitation as the original Copy Budgets change — no local server available here) but the code is identical to the verified Cloud copy.
- **Action required**: redeploy `money-manager-cloud/index.html` to Netlify, and spot-check "Undo Copy" once on the PC/iPhone devices directly.
- **2026-07-18 (sticky Add Transaction fields + smarter search)** — Two changes (Section 2.3, 8.13, 8.14):
  1. **Sticky Type/Date/Account/Category**: the Add Transaction form used to call the form's native `reset()` after every save, snapping Type/Date/Account/Category back to defaults — annoying when entering several transactions in a row for the same account/category/date. Replaced with explicitly clearing only `txAmount`/`txDescription`, then re-applying `record.accountId`/`record.categoryId` to the (freshly repopulated) Account/Category selects after `renderTransactionsView()` runs — ordering matters here since `renderTransactionsView()` itself rebuilds those same dropdowns from scratch and would otherwise wipe the restored selection. Type and Date needed no special handling — simply not calling `reset()` was enough to leave them as-is. The Personal/Business ledger-toggle's own full-reset behavior (`setTransactionsLedger`) was intentionally left untouched, since switching ledgers is a real context switch where resetting to defaults still makes sense.
  2. **Search box matches amount and category, not just description**: `renderTransactionsTable`'s search filter (previously `(t.description||"").toLowerCase().includes(search)` only) now also checks `String(t.amount)` and the category's display name via `categoryDisplayName()` — so typing `480` finds a ₹480 transaction, and typing a category name (including a parent whose subcategories should match, e.g. "Transport") finds those too. The separate exact-match Account/Category/Type filter dropdowns are unchanged.
  Verified live in the browser (Cloud copy): added an Income/UBL Bank/Freelance/2026-06-15/₹480 transaction, confirmed the form still showed exactly that Type/Date/Account/Category afterward (only Amount/Description cleared); added a second Expense/Groceries/₹50 transaction; searched `480` and got only the first transaction, searched `groceries` and got only the second, searched a description-only word and it still worked; switched the ledger toggle and confirmed the form still fully resets in that case. Zero console errors. Mirrored identically into the PC (`app.js`) and iPhone copies — not live-tested for those two in this session (same local-server limitation noted in the earlier entries), code is identical to the verified Cloud copy.
- **Action required**: redeploy `money-manager-cloud/index.html` to Netlify, and spot-check both of these on the PC/iPhone devices directly.
- **User-confirmed 2026-07-19**: Copy Budgets, Undo Copy, sticky Add Transaction fields, and the smarter search box all tested and confirmed working live on the real Netlify site.
- **2026-07-19 (Net This Month stat)** — User observed that the Dashboard's "Total Balance" didn't change when switching months and expected a per-month balance like Income − Expense (e.g. ₹1000 income − ₹700 expense = ₹300 for that month). Clarified before changing anything: "Total Balance" (Section 3.2, `totalNetWorth`) is intentionally all-time/all-account and already labeled "All time" — what was actually missing was a month-scoped net figure, the same concept as the Business tab's existing "Net Profit" stat (Section 3.8). User chose to add a new "Net This Month" stat card rather than replace "Total Balance", keeping the real running total visible. Implemented: a 5th stat card (`statNet` = `monthIncome − monthExpense`, Section 3.3) added to the Dashboard only, right between Expenses and Savings Rate. The Dashboard's `.stat-cards` container got a new `id="dashStatCards"` and a scoped CSS override (`repeat(5, 1fr)` desktop, matching the shared `.stat-cards` class's existing `repeat(2, 1fr)`/`1fr` responsive breakpoints via `.stat-cards, #dashStatCards` selectors) so the Business tab's separate 4-card `.stat-cards` grid is completely unaffected. Verified live in the browser (Cloud copy): confirmed the Business tab's stat grid is still exactly `repeat(4, 1fr)` and untouched; added a ₹1,000 income and ₹700 expense transaction for the same month and confirmed Net This Month showed exactly ₹300, matching Total Balance (which also happened to be ₹300 since these were the only transactions) while Savings Rate correctly showed 30.0%; confirmed the 5 cards lay out cleanly at both ~524px (2-column) and 1280px (5-column) widths. Zero console errors. Mirrored identically into the PC (`index.html`/`app.js`/`style.css`) and iPhone copies — not live-tested for those two in this session (same local-server limitation as prior entries), code is identical to the verified Cloud copy.
- **Action required**: redeploy `money-manager-cloud/index.html` to Netlify, and spot-check the new "Net This Month" card on the PC/iPhone devices directly.
- **2026-07-19 (Total Balance moved to Accounts, privacy)** — Follow-up discussion after the Net This Month change: user raised that showing the real "Total Balance" on the Dashboard is a privacy concern, since Dashboard is the first screen visible to anyone who picks up the device — they don't want their actual net worth visible without a deliberate action. Explored a PIN-lock option for the Accounts tab (whole-tab lock, 5-minute unlock window, PIN synced via Google Sheets) but the user decided against it as unnecessary complexity, and asked for the simpler fix only: move "Total Balance" off the Dashboard entirely and onto the Accounts tab (Section 3.2, 3.3, 3.3a). Implemented: removed the `statBalance` card from the Dashboard (`.stat-cards` reverted to its plain 4-card grid — Income, Expenses, Net This Month, Savings Rate — the `#dashStatCards` id/CSS override added for the 5-card layout was removed since it's no longer needed); added a "Total Balance (Personal)" stat block to the Accounts tab's "Your Accounts" card, directly above the account list, using the same `totalNetWorth("personal")` value as before (`accountsTotalBalance` element, populated in `renderAccountsView()`). No new CSS was needed for the Accounts placement — it reuses the existing `.stat-label`/`.stat-value`/`.stat-sub` typography classes directly (not the `.stat-cards` grid or a nested `.card`), since it sits inside a card that already exists. Labeled "(Personal)" explicitly because the account list below it mixes personal and business accounts (shown with badges), while the total itself is personal-only, matching the Dashboard's original scope and Section 4 rule 2. Verified live in the browser (Cloud copy): confirmed the Dashboard has exactly 4 stat cards with no Total Balance anywhere on it; confirmed the Accounts tab shows "Total Balance (Personal): Rs 300.00" matching the sole Cash account's own balance; confirmed the Business tab's "Total Business Balance" card and 4-card grid are completely untouched; zero console errors; confirmed no dangling `statBalance`/`dashStatCards` references remain anywhere in the file. Mirrored identically into the PC (`index.html`/`app.js`/`style.css`) and iPhone copies, including removing their `#dashStatCards` CSS the same way — not live-tested for those two in this session (same local-server limitation as prior entries).
- **Action required**: redeploy `money-manager-cloud/index.html` to Netlify, and spot-check the Accounts tab's new Total Balance placement on the PC/iPhone devices directly.
- **2026-07-07 (service worker: network-first for the app shell)** — Before the PWA install setup had even been deployed once, caught a real forward-looking problem: the original service worker cached the main HTML page cache-first, which would have kept serving an old cached version of the app forever after future redeploys, even with internet access, once someone had "Add to Home Screen" installed it — silently freezing them on stale code with no obvious way to tell. Fixed by splitting the fetch strategy: the app shell (`.html` / navigations) is now network-first (falls back to cache only when offline), while static assets (icons, manifest) stay cache-first for speed since they rarely change. Section 10.3 updated. Verified the URL-matching logic directly (`index.html`/`/` → app shell, `manifest.json`/`icon-192.png`/`sw.js` → static asset) and confirmed the page still renders correctly with the updated worker active and registered, zero console errors, in both Cloud and PC/iPhone copies.
