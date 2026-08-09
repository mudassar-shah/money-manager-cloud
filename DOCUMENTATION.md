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
| **Cloud** | `D:\money-manager-cloud\index.html` (single file) | Browser localStorage + optional Google Sheets sync | Primary, live on GitHub Pages, actively used |
| **PC** | `D:\money-manager\index.html` + `app.js` + `style.css` | Browser localStorage only | Offline backup copy, no sync |
| **iPhone** | `D:\money-manager\Money Manager (iPhone).html` (single file) | Browser localStorage only | Offline backup copy, no sync |

All three share **identical business logic and formulas**. Only the Cloud copy has the extra Cloud Sync section (Google Sheets read/write). When a formula or business rule changes, it must change in all three, worded identically enough that this document applies to all of them without exception.

### 1.1 Where the Cloud copy is hosted and how it goes live

**Live URL: <https://mudassar-shah.github.io/money-manager-cloud/>** — served by **GitHub Pages** from the repository `mudassar-shah/money-manager-cloud` (branch `main`).

Deployment is: push/upload the files to that repository, and GitHub Pages publishes them automatically within a minute or so. There is no build step — the repo contains the finished static files.

**Hosting moved off Netlify on 2026-07-30.** The old URL (`peppy-lily-416823.netlify.app`) is no longer maintained; Netlify's free plan bills "credits" against deploy activity, and a day of iterating could burn through them, whereas GitHub Pages has no such limit for a static site like this. Change Log entries dated before 2026-07-30 refer to Netlify redeploys — that is accurate history, not a live instruction. Anything from that date on means GitHub Pages.

Two things about the move that were verified rather than assumed:

- **GitHub Pages serves from a subpath** (`/money-manager-cloud/`), not a domain root. Every asset reference in the app was already relative — `<link rel="manifest" href="manifest.json">`, `href="apple-touch-icon.png"`, `navigator.serviceWorker.register("sw.js")`, `manifest.json`'s `"start_url": "./index.html"` / `"scope": "./"` / relative icon `src`s, and `sw.js`'s `SHELL_FILES` of `./…` paths. So the PWA, icons, install prompt and offline caching all work unchanged on a subpath. **Do not change any of these to absolute (`/…`) paths** — that would break the site on GitHub Pages while appearing fine on a root domain.
- **Cloud Sync needs the new origin whitelisted in Google Cloud Console.** Sign-in uses Google Identity Services (`google.accounts.oauth2.initTokenClient`), which validates the page's origin against the OAuth 2.0 Client ID's **Authorized JavaScript origins**. `https://mudassar-shah.github.io` must be listed there or "Sign in with Google" fails with an origin mismatch and Cloud Sync cannot connect. Note that an origin is scheme + host only — do **not** include the `/money-manager-cloud/` path.

---

## 2. Data Model

### 2.1 Category
```
{ id, name, type: "income" | "expense", color, parentId: string|null, order: number, bucket: "personal" | "business" }
```
- `parentId` — the id of another category, making this a subcategory. `null`/absent means top-level.
- `order` — integer position among siblings (same `type` + `parentId` + `bucket`). Lower = higher in the list. Assigned automatically on creation (`nextCategoryOrder`), changed by the ↑/↓ buttons (`moveCategory`).
- `bucket` — which ledger this category belongs to. Personal and Business categories are never mixed in the same list or dropdown.
- `excludeFromTotals` — boolean. When true, transactions in this category are skipped by every income/expense figure but still counted in every balance (Section 3.12). Used for money that **moves without being earned or spent** — chiefly loans given and repaid. Absent/false on ordinary categories.

**Invariant — must never be violated:** A category that already has its own subcategories (i.e. some other category's `parentId` points to it) cannot itself be given a `parentId`. If this happens, its own children become invisible (the UI only renders two levels deep). This is blocked at entry (`catForm` submit handler) and auto-repaired on every load (`repairOrphanedParents`) as a safety net, but the block is the real fix — the repair only exists in case it slips through anyway.

### 2.2 Account
```
{ id, name, type: "bank" | "cash" | "savings" | "credit_card", openingBalance, openingDate, color, bucket: "personal" | "business", excludeFromBalance?: true }
```
- `excludeFromBalance` — boolean. When true this account's balance is left out of `totalNetWorth` (Section 3.13), while every transaction logged against it still counts in full toward income/expense reporting. Used for a "previously paid" holding account. Absent/false on ordinary accounts.

### 2.3 Transaction
```
{ id, type: "income" | "expense", bucket: "personal" | "business", date, accountId, categoryId, amount, description, isReconciliation?: true, transferId?: string }
```
- `isReconciliation` is only set on auto-generated adjustment transactions created by the Reconcile feature (Section 3.6).
- `ref` — free-text deal/invoice code (e.g. `INV-101`) used by the **goods** business to group one sale with all of its costs (Section 3.11). Empty/absent on ordinary transactions.
- `panel` — free-text IPTV panel name (e.g. `Premium`, `Cheap`) identifying **which separate credit stock** a transaction belongs to (Section 3.11). Credits never move between panels.
- `credits` — number of credits. On a business **expense** with a `panel`, this is credits *bought* (making that row a credit purchase, and `amount / credits` its cost per credit). On a business **income** with a `panel`, this is credits *used* by that sale. Absent/0 on everything else.
- `transferId` — set on **both halves** of a transfer (Section 3.10). Empty/absent means an ordinary transaction. A non-empty value means this row is one half of a money movement between two of your own accounts, and the value is shared by exactly the two rows forming that pair. Presence of a value is the *only* test for "is this a transfer" — there is deliberately no separate boolean flag, so the two can never disagree with each other.
- `bucket` — like Category (2.1), the Transactions tab shows one ledger at a time via a Personal/Business toggle; a new transaction always takes the bucket of the currently active toggle, not a per-transaction picker.
- **Bulk move account**: the Transactions tab's multi-select bar has a "Move to Account" action for fixing mis-entered transactions (e.g. logged against Cash instead of a bank account). It changes only `accountId` on the selected transactions — every other field (`type`, `date`, `categoryId`, `amount`, `description`, `bucket`) is left untouched. Because both the source and target account balances are always computed live from `accountId` matches (Section 3.1), moving a transaction automatically and correctly shifts its effect from the old account's balance to the new one — no separate adjustment or reconciliation entry is created or needed. The target-account dropdown is scoped to the currently active ledger (`transactionsLedger`), so a transaction can never be moved into an account belonging to the other bucket. Any reconciliation history (Section 2.5) tied to the *old* account is left as-is — it's a historical log of what was checked at the time, not a live calculation, so moving transactions after the fact does not retroactively change it.
- **Category filter rollup**: the Transactions tab's Category filter dropdown rolls up like the Budget rollup (Section 3.5) — selecting a *parent* category (e.g. "Transport") shows transactions logged against that parent **and** all its direct subcategories, via `categoryAndChildIds()`. Selecting a *subcategory* directly (e.g. "Transport › Fuel Car") shows only that subcategory's own transactions, since a subcategory has no children of its own (one level of nesting only, Section 2.1).
- **Add Transaction form — sticky Type/Date/Account/Category**: after saving a transaction (add or edit), the form no longer resets Type, Date, Account, or Category back to defaults — they stay exactly as last set, so entering several transactions in a row for the same account/category/date/type doesn't require reselecting them each time. Only **Amount** and **Description** clear after save, since those are unique per transaction. Implemented by no longer calling the form's native `reset()`; the Account/Category `<select>` elements are still repopulated fresh (to reflect any category/account added meanwhile) but their `.value` is explicitly restored to what was just used, immediately after that repopulation.
- **Type-to-filter on both category dropdowns**: with ~79 categories, picking one meant scrolling the whole `<select>`. Both category dropdowns on the Transactions tab now have a small text box that filters which options the dropdown contains: `txCategorySearch` above the Add Transaction form's `txCategory`, and `filterCategorySearch` beside the `filterCategory` filter. Typing `groc` narrows 79 options to one and selects it automatically; typing `transport` shows Transport **and** all its subcategories.
  - **Matching rule**: a category is shown if its own name matches, **or** if its parent's name matches — a parent match keeps that parent's whole group, the same rollup idea used by the Budget rollup and the Category filter (Sections 3.5, 2.3). An `<optgroup>` with no surviving children is dropped entirely, and a parent's own "(General)" option appears only when the parent itself matches. Matching is case-insensitive substring, on the category's own `name` — not on the `Parent › Child` display string, so a query cannot match across the `›` separator.
  - **The `<select>` remains the single source of truth.** Only *which options it contains* changes; the element, its id and its `.value` semantics are untouched. This was deliberate: `txCategory`'s value is read or written in **six** places (the `txForm` submit handler, `editTransaction`, the post-save sticky-field restore, the `txType` change handler, `renderTransactionsView`, and initial load), and the sticky-field logic is order-sensitive about when it re-applies `.value` after a repopulation. A custom picker would have had to touch all six; filtering the option list touches none. It also keeps the native iOS picker, which is easier to tap than any custom replacement.
  - **Selection handling** (`applyCategorySearch`): the previously selected id is preserved if it still matches; otherwise, if exactly one option remains it is auto-selected; otherwise the browser's normal "first option" behaviour applies. A hint (`N of M categories`, or `No category matches`) appears only while a query is active.
  - **When the filter is cleared automatically**: after saving a transaction (so the next entry starts from the full list, while the sticky Type/Date/Account/Category selection is still restored — the search box is cleared *before* `renderTransactionsView()` runs, so the repopulation is unfiltered), when clicking **Edit** on a transaction (otherwise an active filter could hide the very category being edited), when switching the Personal/Business ledger, and via the filter row's existing **Clear** button.
  - **Business ledger needs no separate work.** These are the only two category dropdowns in the app — the Business tab has no category picker at all, only its spending chart. Business transactions are entered through this same form with the ledger toggle flipped, so both search boxes are automatically scoped to whichever ledger is active, via the `bucket` argument already threaded through `populateCategorySelect`.
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
{ currency: <main currency code>, rates: { [code]: <PKR per 1 unit> }, displayCurrencies: [code, …] }
```
- `currency` — the **main currency**: the one every cross-account total is expressed in. Not a per-item label (Section 3.15).
- `rates` — manually maintained, always **"how many PKR is 1 unit of this currency"**. PKR is fixed at 1 and never stored. Anchoring to PKR rather than to the main currency keeps the meaning unambiguous even after the main currency changes.
- `displayCurrencies` — which currencies the "also worth" conversion lines show.

Supported codes: `PKR, USD, AED, SAR, QAR, GBP, EUR, INR`.

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
**Ledger toggle (added 2026-08-08).** The Yearly Report is no longer personal-only. It carries a Personal/Business toggle (`yearlyLedger`) exactly like the Categories and Transactions tabs, showing **one ledger at a time** — which is the pattern Section 4 rule 2 approves; that rule forbids *mixing* ledgers on one screen, not switching between them. Every FY query takes its bucket from `yearlyLedger` instead of a hardcoded `"personal"`, and the account summary filters accounts the same way.

This is the one part of the app where a bucket filter has previously been got wrong three separate times (Change Log, 2026-07-07) — `renderYearlyView`, `renderFyBar` and `renderFyAccounts` each leaked business data into personal totals through independent code paths. **The filter must therefore be threaded explicitly through every FY function; none may fall back to a default.** Test 8.22 exists to catch a regression here.

Business-mode differences:
- **Net Profit** replaces "Net Savings" and **Profit Margin** replaces "Savings Rate" (same arithmetic, business-appropriate labels), matching the Business tab's own naming (3.8).
- The **Annual Budget vs Actual** card is hidden entirely — business budgeting does not exist (Section 7).
- An **Annual Profit by Deal** card appears instead, applying Section 3.11's grouping and FIFO credit costing across all 12 months of the fiscal year rather than one month.
- The account summary lists business accounts only.

Both ledgers use the **same 1 July – 30 June fiscal year** and the same unlimited prev/next navigation — confirmed with the user.

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

### 3.9a Dates and "today" — always the device's local date, never UTC

```
todayISO() = the current date on the device the app is open on, as "YYYY-MM-DD",
             built from local date parts:
             `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())}`
```

**This must never be `new Date().toISOString().slice(0, 10)`.** `toISOString()` converts to UTC first, which silently reports a *different calendar day* than the device shows for part of every day — and for a user in Pakistan (UTC+5), that part is midnight to 5am every single day.

**The bug this fixes (found 2026-08-01).** The user opened the app on 1 August and the Dashboard displayed **July**. `todayISO()` was reading UTC, where it was still 31 July, so `monthKey()` sliced `"2026-07-31"` down to `"2026-07"`. The same function was also the default for the Add Transaction date, meaning **a transaction entered before 5am was dated into the previous month** — filed under the wrong month's Income/Expenses and the wrong month's budget, with no warning. On 1 July before 5am the Yearly Report default (`fiscalYearStartOf`) resolved to the *previous fiscal year*, i.e. off by a full year. Note the codebase already disagreed with itself: `shiftMonth` had always used local `getFullYear()`/`getMonth()`.

The failure is symmetrical and affects any timezone, not just Pakistan's: **east of UTC** (Pakistan, India, UAE) the app reports *yesterday* during local early morning; **west of UTC** (US, Canada) it reports *tomorrow* during local late evening. Using local date parts is correct everywhere, needs no configuration, and follows the device if it moves timezone — there is deliberately no timezone setting, and none should be added.

**Everything that depends on this** — all of these were wrong by one day for part of every day: the Dashboard / Business / Accounts / Budgets default month, the Yearly Report's default fiscal year, the Add Transaction and Transfer date defaults, the Reconcile statement-date default, account opening-date defaults, and the Export Backup filename.

**Known remaining limitation (deliberately not fixed).** Each tab's month (`dashMonth`, `bizMonth`, `accountsMonth`, `budgetMonth`, `fyStartMonth`) is evaluated **once at page load** and never re-evaluated. An installed app left open across midnight on the 1st keeps showing the old month until reloaded. This was considered and left alone on purpose: re-evaluating on tab switch would fight the user whenever they had deliberately navigated to a different month, throwing away where they were. A reload is the intended fix, and correct behaviour on a fresh load is what actually matters.

### 3.10 Transfers between your own accounts

**The problem this solves.** Withdrawing cash from a bank account is not income and not an expense — it is the same money in a different place. Before this feature the only way to record it was two ordinary transactions (an expense from the bank, an income into Cash), which made account balances correct but inflated the Dashboard's **Income** *and* **Expenses** by the transferred amount, distorted **Savings Rate**, added a fake slice to the **spending donut**, and consumed any **budget** on the withdrawal's category. Real example from live data (August 2026): Income read Rs 214,553 against real income of Rs 194,553, and a Rs 20,000 "Cash Withdrawal" showed as the 3rd largest expense category at 18.0%.

```
A transfer is a pair of ordinary transactions sharing one transferId:
  expense half: accountId = source,      amount = X, transferId = T
  income  half: accountId = destination, amount = X, transferId = T
Both halves always share: the same transferId, date, amount, description and bucket.
isTransferTx(t) = !!t.transferId
```

**The split — which figures exclude transfers, and which must not.** This distinction is the whole correctness of the feature and must never be collapsed into "exclude everywhere":

| Excludes transfers (`&& !t.transferId`) | Why |
|---|---|
| Dashboard Income / Expenses (`renderDashboard`) | and therefore Net This Month and Savings Rate, which derive from them |
| `renderDonutChart` (Dashboard + Business, filtered inside the shared function) | a transfer is not spending |
| `renderBarChart` (Dashboard + Business 6-month chart) | same reason |
| `renderBudgetProgress`, `renderBudgetsView` | a withdrawal must not consume a budget |
| Business tab Income / Expense / Net Profit | same as Dashboard |
| Yearly Report: headline totals, donut, 12-month bar, annual budgets | same as Dashboard |

| **Keeps** transfers (deliberately unchanged) | Why — breaking this is a real bug |
|---|---|
| `accountNet` / all balance functions (3.1) | a transfer genuinely moves money; excluding it corrupts every balance |
| Accounts tab carry-forward table inflow/outflow | `closing = opening + inflow − outflow` is computed *from* these two figures — excluding transfers makes the row stop adding up |
| Yearly Report "Account Summary — Opening vs Closing" inflow/outflow | opening/closing come from `accountBalanceBefore`, which includes transfers; excluding them from inflow/outflow would make the row fail to reconcile |
| Transactions table and Recent Transactions lists | these are lists of what happened, not totals — transfers stay visible, marked with a `⇄` badge |

**Creating one.** The Transactions tab has a "Transfer Between Accounts" form (From account, To account, Amount, Date, optional Description). One submit creates both halves, guaranteeing they share a date — hand-entered pairs could and did drift apart by a day, which silently misstates `accountBalanceAsOf` (breaking Reconcile) and, if a pair straddles 30 June / 1 July, lands the two halves in *different fiscal years*.

**Categories.** Both halves need a `categoryId` because the field is required everywhere. A pair of `Transfer` categories (one income, one expense) is auto-created per bucket on first use, exactly mirroring how `Balance Adjustment` is auto-created by Reconcile (3.6). Note that once a row carries a `transferId`, its category no longer affects any reported figure — so pre-existing pairs keep whatever categories they were entered under (e.g. the user's own "Cash Withdrawal" / "Cash Deposite") and remain fully correct.

**Same ledger only, on this form.** Both halves take the bucket of the active `transactionsLedger`, and the From/To dropdowns are scoped to it. A Personal ↔ Business movement is **blocked here** and handled by a separate form (Section 3.14) — the two are kept apart deliberately, at the user's request, so the everyday same-ledger transfer they use constantly is never disturbed by the rare cross-ledger one.

**Converting pairs entered by hand.** The Transactions multi-select bar has a **Mark as Transfer** action. Select exactly two rows; it validates they are one income + one expense, equal amounts, different accounts, same bucket, then assigns both a shared new `transferId`. Because the flag lives on the transactions themselves, this retroactively corrects every report for every past month with no other edits. If the two selected rows are *already* a transfer pair, the same button offers to remove the marking instead, so a mistake is always reversible. Dates are reported in the confirmation dialog if they differ but are never silently changed — altering a date is a data edit the user did not ask for.

**Editing and deleting.** Deleting either half deletes both (the confirmation names the other half) — leaving one half behind would silently corrupt an account balance, which Section 4 rule 3's "never silently orphan" principle forbids. Editing a transfer half through the ordinary Edit button is **blocked** with a message directing the user to delete and re-enter, because editing one half's amount or account would desynchronise the pair. As a defensive measure the `txForm` submit handler preserves `transferId` on any record it saves, since it rebuilds the record object from scratch and would otherwise strip the field.

### 3.11 Business profitability — deal grouping and credit costing

**The problem.** The Business tab could only ever answer "how much cash moved this month" (3.8). It could not answer "did *that* sale make money", because nothing linked a sale to its costs. The user runs two different businesses through this one ledger, and each needs a different mechanism:

| | **Goods** | **IPTV** |
|---|---|---|
| Bought | per order, from one or more vendors | credits in bulk, ahead of any sale |
| Costs | vendor payment(s) + commission + courier | the credits only |
| Invoiced? | yes, one invoice per customer | no invoice at all |
| Mechanism | `ref` grouping | `panel` + `credits` FIFO costing |

**Commission is a fixed negotiated amount, never a percentage** — confirmed with the user; it can be most of the sale (10,000 commission on a 12,000 deal), so a deal can legitimately be near-breakeven or a loss. Invoiced amount always equals the amount received (agreed before invoicing, so no partial payments), which is why **no receivables/partly-paid tracking exists or is needed**.

#### 3.11a Goods deals — grouping by `ref`
```
dealIncome(ref) = Σ amount of business income transactions with that ref
dealCosts(ref)  = Σ amount of business expense transactions with that ref
dealProfit(ref) = dealIncome − dealCosts − creditCost(the deal's sale, if any)
margin(ref)     = dealProfit / dealIncome × 100     (blank if dealIncome is 0)
```
One sale, any number of cost rows — two vendors, commission and courier all simply share the `ref`. **Grouping is by code, never by date**, so a cost paid in one month and a payment received in another still land on the same deal; this is what makes the user's late-payment case work. A deal with costs but no income yields a negative profit and is shown as a loss rather than being invisible.

#### 3.11b IPTV credits — separate stocks per panel, oldest-first costing
The user has **two physically separate vendor panels**; a premium credit cannot be spent on a cheap user. So `panel` partitions credits into independent stocks, and **no costing ever crosses a panel boundary**.

Within one purchase every credit costs the same (`amount / credits`), but the price drifts between purchases (observed range 50–260 per credit, set by the market). The user tops up when roughly 10 credits remain, so a panel routinely holds a few old credits at the old price alongside new ones at the new price.

```
For one panel, in date order (ties broken by id for stability):
  purchases = business expenses with that panel and credits > 0   → unitCost = amount / credits
  sales     = business income  with that panel and credits > 0
  Walk sales oldest-first, consuming purchases oldest-first (FIFO):
    creditCost(sale) = Σ (creditsTakenFromPurchase × thatPurchase.unitCost)
  A single sale may span two purchases and is charged a blended cost.
```
**The user picks only the panel and the credit count — never a specific purchase.** Credits are fungible within a panel (they are just a balance in the vendor's panel), so asking which purchase a sale drew from would be inventing information the user does not have. FIFO removes that question.

**Credits per sale is the user's own number, and the app deliberately knows nothing about months or bonus tiers.** Cheap panel: 1 month = 1 credit, 6 months = 5 credits, 12 months = 10 credits. Premium: always 1 credit per month. Encoding those tiers would mean maintaining the vendor's commercial rules in the app; instead the volume bonus *is* the smaller number the user types, and it surfaces automatically as a higher margin. A hint listing the tiers sits next to the Credits box so it doesn't have to be memorised.

**Shortfall must be reported, never hidden.** If a panel's recorded sales consume more credits than its recorded purchases supplied, the excess is costed at **0** and the affected deals plus the Credit Stock card carry an explicit warning. Silently costing missing credits at zero would overstate profit — exactly the class of quiet misstatement Section 6 exists to prevent.

```
Stock per purchase: left = bought − used(by FIFO),  valueLeft = left × unitCost
Stock per panel:    Σ left,  Σ valueLeft
```

#### 3.11c Scope, and the two profit numbers that must never be forced to agree
- **Profit by Deal** lists deals whose **sale** falls in the selected Business month, but pulls in costs **from any month** — otherwise a deal whose vendor was paid in a different month would show a false profit.
- **Credit Stock** is **not** month-scoped. It is a running position, so it always reflects every purchase and sale to date.
- Business income with neither a `ref` nor `credits` cannot have a cost attributed to it. Such rows are **counted and reported** beneath the deal list rather than silently omitted.
- **`Net Profit` (3.8) and deal profit will not match, and neither is wrong.** Net Profit is cash that moved this month; in a month where credits are bought in bulk it looks poor because the money genuinely left. Deal profit is what each sale earned. They answer different questions and are deliberately kept in separate cards; no attempt is made to reconcile them.
- Everything here is **read-only and additive**. `Income`, `Expenses`, `Net Profit`, `Total Business Balance`, the donut, the bar chart, all account balances, and every personal figure are untouched — the three new fields are labels that feed two new cards and alter no existing calculation. Transfers (3.10) are excluded from these views too, since a transfer is not a sale.

### 3.12 Excluded categories — money that moves without being earned or spent

**The problem.** The user lends money from UBL and is repaid into UBL. Both legs were counted, so Income *and* Expenses were each overstated by the loan amount (real case: Rs 6,000 across two loans in July 2026) while the UBL balance stayed correct.

**Why the transfer mechanism (3.10) cannot solve it.** `Mark as Transfer` requires the two halves to be on **different accounts**, and a loan leaves and returns to the same account. That guard is correct and must not be relaxed. More fundamentally, loans do not behave like transfers:

| | Transfer | Loan |
|---|---|---|
| Timing | both halves at the same moment | repayment may come weeks later |
| Completeness | always exactly two halves | may be partial, or never repaid |
| If unrepaid | n/a | the money is still **owed to you**, not spent |

Pairing would therefore count an unrepaid loan as an expense, which is wrong, and would break entirely on a partial repayment.

**The mechanism: a flag on the category, not on the transactions.**
```
excludeFromTotals(categoryId) = the category's own flag
A transaction is skipped by income/expense reporting iff its category has the flag set.
```
Because the flag lives on the category, it is **retroactive with no data migration** — ticking it corrects every past month instantly, since the user had already filed these under dedicated categories ("Office › Loan Given", "Loan return"). No pairing, no matching, and partial or late repayments need no special handling.

**The same split as transfers applies, and for the same reason** (3.10):

| Excludes flagged categories | Keeps them |
|---|---|
| Dashboard Income / Expenses / Net / Savings Rate | `accountNet` and all balance functions — the money genuinely moved |
| Spending donut and 6-month bar (both ledgers) | Accounts carry-forward inflow/outflow — `closing = opening + inflow − outflow` |
| Budget progress and the Budgets tab | Yearly Report account summary inflow/outflow |
| Business tab totals; Yearly Report totals, donut, bar, annual budgets | Transaction lists — the rows stay visible |

**Money still out on loan** (Accounts tab, beneath Total Balance):
```
outstanding = Σ(flagged personal EXPENSE amounts) − Σ(flagged personal INCOME amounts)
```
i.e. lent out minus returned. Shown only when non-zero, so it stays invisible for users who never lend. A negative result (more returned than lent, e.g. a loan predating the app) is shown as **owed by you** rather than as a negative number.

**Bad debt is a deliberate manual decision.** If a loan is never repaid it eventually becomes a real loss, but only the user can decide when. The fix is to re-categorise that one transaction to an ordinary expense category; the app never makes that judgement automatically.

**Cloud Sync safety (verified before implementation, at the user's request).** `readAllFromSheet` maps columns **by name from the sheet's own header row**, not by position, so adding a column cannot shift or scramble existing data, and a sheet **without** the new column simply yields `undefined` → `false`, which is the correct default. `pushAllToSheet` clears `A:Z` before rewriting, so no stale column survives. The boolean is stored exactly like `isReconciliation`, a pattern already proven in production since July. Test 8.23 covers the backward-compatibility case explicitly.

### 3.13 Accounts excluded from Total Balance — spending that predates tracking

**The problem.** Tracking began in **July 2026** for both ledgers, but life and the business did not. A year of school fees was paid in January; business costs were settled months earlier. The user wants those costs to appear as expenses **in the month they relate to**, so the monthly picture is honest — but the money is already gone, and each real account's July opening balance *already reflects* that prior spending. Logging the expense against UBL or Cash would therefore subtract it a second time.

The workaround is a holding account ("Pervious Paid"). It works, except its balance drifts ever more negative and drags down Total Balance, which is not a real loss — it is spending that happened before the app existed.

**Confirmed with the user: the original lump-sum payments were never recorded**, because there is no pre-July data at all. So there is no double-counting; the negative balance is simply an artefact of the tracking start date.

```
totalNetWorth(bucket) = Σ accountCurrentBalance(a.id)
                        for accounts where bucket matches AND NOT a.excludeFromBalance
```

**The exact mirror of Section 3.12**, and the two must not be confused:

| | Excluded **category** (3.12) | Excluded **account** (3.13) |
|---|---|---|
| Counts in Income / Expenses? | **No** | **Yes** — that is the entire point |
| Counts in Total Balance? | **Yes** | **No** |
| Typical use | loans given and repaid | costs paid before tracking began |

**Deliberately narrow scope.** Only `totalNetWorth` changes — the two places it is used, "Total Balance (Personal)" on the Accounts tab and "Total Business Balance" on the Business tab. Because that function is already bucket-filtered, **one flag serves both ledgers** with no extra work. Everything else keeps showing the account: it stays in the account list with its running balance (carrying a "not in total" badge), in the Dashboard/Business accounts overview, in the monthly carry-forward table, and in the Yearly Report account summary. Hiding it from those would destroy the record the user is deliberately keeping. `accountCurrentBalance` itself is untouched, so Reconcile and every per-account figure behave exactly as before.

**An alternative that avoids the flag**, noted for completeness: fund the holding account with a **Transfer** (3.10) from the real account at the time of payment, so it starts positive and drains to zero as the expense is consumed, never going negative and showing the unconsumed prepayment as a genuine asset. This is the tidier model but only works when the original payment falls inside the tracking period — which, here, it does not.

### 3.14 Business ↔ Personal transfers

Money genuinely moves between the user's own business and personal accounts: profit taken out, own money put in, or — the case they described most — **borrowing** from the business to cover something personal and repaying it later.

**Treated as a pure transfer on both sides.** Both halves are excluded from every income/expense figure, in both ledgers; both account balances update in full. Nothing is earned and nothing is spent, because it is the same person's money in both places.

```
Business → Personal 30,000:
  business account −30,000   personal account +30,000
  Business Income/Expenses/Net Profit   unchanged
  Personal Income/Expenses/Savings Rate unchanged
```

Mechanically it reuses Section 3.10 exactly — one shared `transferId`, both halves excluded via `countsInTotals`, linked deletion — with one difference: **each half takes the bucket of its own account**, not of any active toggle. That is what makes it cross-ledger.

**Why not treat a withdrawal as personal income?** Considered and rejected with the user. It would be defensible for a permanent owner's draw, but wrong for borrowing, which is the dominant case here. Making it a plain transfer gives one rule to remember and keeps both ledgers' figures clean. **The user retains control per movement**: use this form and it doesn't touch income/expense; enter it as two ordinary transactions instead and it counts normally. The known trade-off, stated to the user: if profit is permanently drawn out via this form, personal balance rises with no income explaining it and Savings Rate flatters slightly.

**A separate form, deliberately.** It lives on the **Accounts** tab, not with the Transactions transfer form. The user asked for this explicitly: cross-ledger movement is rare ("once in a blue moon") while same-ledger transfers are constant, and merging them would have meant changing a form used daily to serve an occasional case. The everyday form is therefore left completely untouched. This form is the mirror image: it **requires** the two accounts to be in different ledgers and refuses a same-ledger pair, pointing the user back to the Transactions tab.

**Section 4 rule 2 is not violated.** Each half sits wholly within its own ledger's reports — a business half never appears in a personal figure, or vice versa. Only the `transferId` link spans the two, and since both halves are excluded from income/expense anyway, they surface solely in balances and transaction lists.

`Mark as Transfer` (3.10) also accepts cross-ledger pairs, so movements already entered by hand can be corrected retroactively.

### 3.15 Multi-currency — currency belongs to the account

**The problem.** The user plans to move abroad (Dubai, Saudi, possibly UK) and wants to keep using this app for life. Before this change the app had **one global currency setting used only to choose a symbol**: every amount was a bare number with no currency attached. Switching the setting from PKR to AED would have relabelled Rs 194,553 as "AED 194,553" and then **added rupees to dirhams** in every total — a silently wrong figure, which is worse than an obvious error because it looks authoritative.

**The model: currency lives on the account.** You never spend dirhams out of a rupee account, so a transaction inherits its currency from its account and needs no currency of its own. Moving abroad means **adding an AED account**, not changing a setting. All Pakistani history stays in rupees, permanently.

```
accountCurrency(accountId) = account.currency, defaulting to the main currency
Per-account balances (3.1) are ALREADY correct — each works on one account, so one currency.
Only CROSS-account sums need conversion.
```

**Conversion.**
```
rates[code]           = how many PKR 1 unit of that currency is worth   (PKR ≡ 1)
convertAmount(a,f,t)  = a × rates[f] / rates[t]      null if either rate is missing
txAmountInMain(t)     = convertAmount(t.amount, accountCurrency(t.accountId), mainCurrency)
```
Rates are **entered manually**, with an optional "get latest" button. Deliberately not fetched automatically: the app otherwise talks to nobody but Google, an outside rate service could change or disappear, and it would break offline use. A manually-set rate is ample for "roughly how much do I have in dollars".

**A missing rate is never guessed.** `convertAmount` returns `null`, the amount is **excluded** from the total, and a visible warning names the currency. Treating an unknown rate as 1:1 would produce a confident, badly wrong number — the exact failure this whole section exists to prevent.

**What changed, and what deliberately did not:**

| Unchanged | Converted to main currency |
|---|---|
| `accountNet`, `accountCurrentBalance`, `accountBalanceAsOf`, `accountBalanceBefore` — each is single-account, so already single-currency | `totalNetWorth`, Dashboard Income/Expenses/Net/Savings Rate, both budget functions, the donut, both bar charts, Business tab totals, all Yearly Report totals, deal profit, credit costing, outstanding loans |
| Account list, accounts overview, carry-forward table, Yearly Report account summary, transaction rows — all display in **their own account's currency** | |

`fmtMoney(n, code)` gained an optional currency argument defaulting to the main currency, so all 54 existing call sites kept working unchanged; only the per-account displays were switched to pass an explicit code.

**Cross-currency transfers (3.10, 3.14).** When the two accounts share a currency the form is unchanged — one amount. When they differ, a **second amount box** appears: what left, and what actually arrived. The user types the received figure from their bank, because a bank's real rate and fees never match a published rate; the saved rate only pre-fills it as a suggestion. The two halves therefore hold **different amounts**, which is why `Mark as Transfer` no longer requires equal amounts when the accounts' currencies differ. No fee transaction is derived — whether the spread is a "loss" depends on which rate you consider true, so that judgement is left to the user.

**Conversion display.** An "also worth" line in the user's chosen currencies appears beneath:
- **Total Balance** (Accounts) and **Total Business Balance** (Business)
- The **Dashboard's monthly** Income, Expenses and Net This Month
- The **Business tab's monthly** Income, Expenses and Net Profit
- The **Yearly Report's** Total Income, Total Expenses and Net

The purpose the user stated is planning a move: *"what amount of USD does my family actually need for expenses"*. Converting the **monthly** figures answers that directly.

**Savings Rate and Profit Margin are never converted** — they are percentages.

**Net worth deliberately stays off the Dashboard.** Only *month-scoped* figures are converted there. Section 3.3a removed Total Balance from the Dashboard as a privacy decision, and adding a converted net-worth line would silently reverse it. This was raised with the user, who confirmed they wanted the monthly figures, not the balance.

**Migration.** Every existing account is set to PKR, so on the day of release **every figure must be byte-identical to before**. That equality is the release test (8.26) and is only available while all data is single-currency — which is precisely why this was done before the user moves abroad.

---

## 4. Business Rules / Invariants (must hold at all times)

1. A category with subcategories cannot itself have a `parentId` (Section 2.1).
2. Every category, account, and transaction has a `bucket` — `"personal"` or `"business"` — and personal/business data never appear in each other's reports. The **Dashboard and Budgets are personal-only**; the **Business tab is business-only**; the **Yearly Report shows one ledger at a time via its own toggle** (Section 3.7). What is forbidden is *mixing* two ledgers in one figure or one list, never offering a switch between them. The **Categories** and **Transactions** tabs each show only one ledger at a time via a Personal/Business toggle (`categoriesLedger` / `transactionsLedger`) — there is no "all ledgers" mixed view for either, and new items always take the bucket of whichever toggle is currently active (no per-item ledger picker in the Add form).
3. Deleting a category or account is blocked if any transaction or budget still references it — the app never silently orphans a transaction.
4. `order` values are unique within a sibling group; `ensureCategoryOrder` repairs any category missing an `order` value on every load.
5. Every field that exists in the local data model and is used anywhere in the app **must** also exist in the Cloud Sync schema (Section 5) if it needs to survive syncing. This is the single most important rule in this document — see Section 6.
6. A `transferId` never exists on a lone transaction — it is always present on exactly two rows (one income, one expense, equal amounts, different accounts, same bucket, same date). A single orphaned half silently corrupts an account balance, so deletion of either half always removes both (Section 3.10).
7. Transfers are excluded from every *income/expense* figure but included in every *balance* and *account inflow/outflow* figure. The Accounts tab and Yearly Report account tables compute closing balances from their own inflow/outflow sums, so excluding transfers there would make those rows stop adding up — see the two tables in Section 3.10 before touching any total.
8. Any transaction/category created automatically by the app on your behalf (reconciliation adjustments, Section 3.6; transfer halves and their `Transfer` categories, Section 3.10) must be tagged with the same `bucket` as whatever triggered it — auto-created records are just as capable of leaking across the personal/business boundary as manually entered ones, and are easier to miss during testing because the user didn't type them in directly.

---

## 5. Cloud Sync Schema (Cloud copy only)

Your Google Sheet has one tab per entity. The exact columns, in order, that get written and read:

| Sheet tab | Columns |
|---|---|
| `Categories` | `id, name, type, color, parentId, order, bucket, excludeFromTotals` |
| `Accounts` | `id, name, type, openingBalance, openingDate, color, bucket, excludeFromBalance, currency` |
| `Transactions` | `id, type, date, accountId, categoryId, amount, description, isReconciliation, bucket, transferId, ref, panel, credits` |
| `Budgets` | `id, categoryId, month, amount` |
| `Reconciliations` | `id, accountId, date, actualBalance, computedBefore, diff` |
| `Settings` | `key, value` — now **multiple rows**: `currency`, `rates` (JSON), `displayCurrencies` (comma-separated). Previously only a single `currency` row was written or read. |

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
- **Business tab has no budgeting.** Budgets are personal-only by design, and the Yearly Report's "Annual Budget vs Actual" card is hidden in Business mode for the same reason. (The Yearly Report itself covers both ledgers as of 2026-08-08 — Section 3.7.)
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

**8.26 Multi-currency (Section 3.15)** — the release test is *equality*, and it is only possible while all data is still single-currency. Run it first and treat any difference as a failure:
- **With every account set to PKR, confirm every figure on every tab is byte-identical to before the change** — Dashboard, Accounts (incl. Total Balance and carry-forward), Transactions, Budgets, Yearly Report (both ledgers), Business (incl. deals and credit stock). Not one number may move.
- **Backward compatibility**: feed `coerceTypesFromSheet` an Accounts sheet with the **old 8-column header** (no `currency`) and confirm every account loads intact with currency defaulting to the main currency. Feed a Settings sheet containing **only the old single `currency` row** and confirm it still loads, with rates empty and no crash.
- **Round-trip (8.7)** for `Accounts.currency` and for all three Settings rows, including rates surviving as numbers through JSON.
- Add an AED account and confirm: its own balance shows in **AED**; its transactions show in AED in the list; but Dashboard Income/Expenses and Total Balance convert it into PKR at the saved rate.
- **Missing rate**: add an account in a currency with no rate set, give it a transaction, and confirm the amount is **excluded** from totals and a warning naming that currency is shown — never silently counted as 1:1.
- Confirm per-account balances, the carry-forward table and Reconcile are unaffected by any rate change (they must never convert).
- Change a rate and confirm only the converted totals move, never a per-account balance.
- **Cross-currency transfer**: PKR → AED with different sent/received amounts; confirm each account moves by its own amount, that both halves share one `transferId`, that deleting either removes both and restores both balances, and that no income/expense figure moves.
- Confirm the transfer form shows **one** amount box for same-currency accounts and **two** when they differ, and that `Mark as Transfer` accepts unequal amounts only when the currencies differ.
- Confirm the "also worth" lines on Dashboard and Accounts match `total × rate`, and that Business conversions stay separate from personal.

**8.25 Business ↔ Personal transfers (Section 3.14)** — the risk is leaking a figure across the ledger boundary, so test both sides' totals explicitly:
- Move an amount from a business account to a personal one. Confirm the business account's balance falls by exactly that amount and the personal account's rises by exactly that amount.
- **Confirm all six headline figures are unchanged**: Business Income, Business Expenses, Business Net Profit, Personal Income, Personal Expenses, Personal Savings Rate. Not one of them may move.
- Confirm **Total Balance (Personal)** rises and **Total Business Balance** falls by the amount — these *should* move, since real money moved.
- Confirm the reverse direction (personal → business) behaves symmetrically.
- Confirm both halves carry the same `transferId`, the same date and the same amount, and that **each half's `bucket` matches its own account** — not a toggle.
- Confirm the business half appears only in business views and the personal half only in personal views (Section 4 rule 2 holds).
- Confirm deleting either half deletes both, leaving both balances correct.
- Confirm the form **refuses** two accounts in the same ledger, and refuses From == To.
- Confirm the everyday Transactions-tab transfer form is **completely unaffected** and still refuses cross-ledger pairs.
- Confirm `Mark as Transfer` now accepts a hand-entered cross-ledger pair and that doing so removes both from income/expense without moving any balance.
- Confirm a borrow-then-repay round trip (business → personal, later personal → business, same amount) returns both balances to their starting values and leaves every income/expense figure untouched throughout.

**8.24 Accounts excluded from Total Balance (Section 3.13)** — the risk is scope creep: this must change **one** figure and nothing else.
- **Backward compatibility first** (live data is Sheets-only): feed `coerceTypesFromSheet` an Accounts set with the **old 7-column header**. Confirm every account loads with `openingBalance` as a number, `openingDate`, `color` and `bucket` intact, and the absent flag becoming `false` — never `undefined` or `NaN`.
- Flag a personal account and confirm **Total Balance (Personal)** drops by exactly that account's balance; flag a business account and confirm **Total Business Balance** does the same. One flag must serve both ledgers.
- **Confirm nothing else moves**: Dashboard Income/Expenses, the donut, budgets, the Yearly Report totals, and every *per-account* balance must be byte-identical before and after flagging.
- Confirm expenses logged against a flagged account **still count** in Income/Expenses and still appear in the spending donut — this is the whole purpose and the opposite of 8.23.
- Confirm the flagged account still appears in the account list with its own balance and a "not in total" badge, in the Dashboard/Business accounts overview, in the carry-forward table, and in the Yearly Report account summary.
- Confirm Reconcile on a flagged account still computes from its own balance correctly (`accountCurrentBalance` must be untouched).
- Confirm a flagged account with a **negative** balance no longer drags Total Balance down, which is the reported symptom.
- **Round-trip (8.7)**: flag an account, push, read back, confirm the flag survives as a boolean and no other account field changed value or type.
- Confirm unflagging restores the original Total Balance exactly.

**8.23 Excluded categories (Section 3.12)** — the user's live data exists only in Google Sheets, so backward compatibility is the priority test, not an afterthought:
- **Backward compatibility (do this first)**: feed `coerceTypesFromSheet` a Categories set with the **old 7-column header** (no `excludeFromTotals`). Confirm every category still loads with `id`, `name`, `type`, `color`, `parentId`, `order` and `bucket` intact, and that the missing flag becomes `false` — never `undefined`, `NaN` or the string `"undefined"`.
- Confirm a sheet row with the flag blank (`""`) also yields `false`, and one with `TRUE`/`true` yields `true`.
- **Round-trip (8.7)**: flag a category, push, read back, and confirm the flag survives as a **boolean** and that no other category field changed value or type.
- Tick the flag on an expense category and an income category; confirm Dashboard Income and Expenses each drop by exactly those categories' totals, that Savings Rate recalculates, and that the categories vanish from the spending donut.
- **Confirm every account balance is byte-for-byte unchanged** before and after ticking — this is the whole point of the feature.
- Confirm the Accounts carry-forward table still satisfies `closing = opening + inflow − outflow` with flagged transactions included there, and likewise on the Yearly Report account summary.
- Confirm a budget set on a flagged category is unaffected in the sense that flagged spending no longer consumes any budget.
- Confirm flagged transactions still appear normally in the Transactions list and Recent Transactions (they are hidden from totals, not from history).
- **Outstanding line**: with a loan out and unreturned, confirm the Accounts tab shows the correct outstanding figure; repay it fully and confirm the line disappears rather than showing zero; confirm it counts personal only and ignores business.
- Confirm unticking restores the original figures exactly.

**8.22 Yearly Report ledger toggle (Section 3.7)** — this is the exact area where a missing bucket filter has caused three separate real leaks, so test isolation before anything else:
- With both personal and business data in the same fiscal year, confirm the **Personal** view's Total Income, Total Expenses, Net Savings, Savings Rate, category donut, 12-month bar and account summary are **numerically identical to what they were before the toggle existed** — no business figure may move any of them.
- Switch to **Business** and confirm every one of those figures changes to business-only, that the account summary lists business accounts only, and that no personal account or category appears anywhere.
- Confirm the two ledgers' Total Income figures **sum to** the all-ledger total for that year — if they don't, something is being double-counted or dropped.
- Confirm **Net Savings → Net Profit** and **Savings Rate → Profit Margin** relabel in Business mode and revert in Personal.
- Confirm **Annual Budget vs Actual is hidden** in Business mode and returns in Personal, and that **Annual Profit by Deal** appears only in Business.
- Confirm Annual Profit by Deal spans all 12 FY months (a deal in month 1 and another in month 12 both appear) and that its credit costing matches Section 3.11's FIFO — same blended-cost check as 8.21.
- Confirm the toggle survives prev/next FY navigation (it must not reset to Personal), and that navigating to a year with no data shows empty states rather than errors in either ledger.
- Confirm transfers (3.10) are excluded from both ledgers' income/expense totals here, as everywhere else.

**8.21 Business deal grouping and credit costing (Section 3.11)** — the FIFO walk is the risky part; test the boundaries, not just the happy path:
- **Goods deal**: one sale with two vendor costs, a commission and a courier charge all sharing a `ref`; confirm profit = income − all four costs and margin = profit/income. Add a cost row dated in a *different month* and confirm it still counts toward that deal.
- Confirm a `ref` with costs but **no** income shows a negative profit (loss), not a blank row or a crash.
- Confirm commission larger than the sale yields a loss shown as negative, since commission is a negotiated amount with no upper bound.
- **FIFO within one panel**: purchase 100 credits at 170, then 50 at 180. A 10-credit sale must cost 1,700. Once only 10 old credits remain, a 15-credit sale must cost 10×170 + 5×180 = **2,600** — the blended figure is the specific regression to watch.
- **Panel isolation**: with credits in both Premium and Cheap, confirm a Premium sale is never costed from Cheap credits and vice versa, and that each panel's stock totals are independent.
- **Ordering**: confirm costing is by transaction **date**, not entry order — enter an earlier-dated purchase *after* a later one and confirm the earlier one is consumed first.
- **Shortfall**: sell more credits than were ever purchased in a panel; confirm the excess is costed at 0, the deal is flagged, and the Credit Stock card shows the warning rather than quietly overstating profit.
- **Divide-by-zero**: a credit purchase with `credits` of 0 or blank must be ignored as a purchase, not produce `Infinity`/`NaN` anywhere.
- **Stock arithmetic**: for every purchase confirm `left = bought − used` and `valueLeft = left × unitCost`, and that panel totals equal the sum of their purchases. Confirm the card is **not** month-scoped (switching Business months must not change it).
- **Scope**: confirm the deal list is filtered to sales in the selected month, that business income with neither `ref` nor `credits` is reported as a count beneath the list rather than dropped, and that transfer halves never appear as deals.
- **Isolation**: confirm Income, Expenses, Net Profit, Total Business Balance, the donut, the bar chart and every personal figure are byte-for-byte unchanged by adding these fields.
- **Round-trip (8.7)**: sync and confirm `ref`, `panel` and `credits` all survive with correct types — `credits` must come back as a **number**, and a blank must not become `NaN` or `"0"`.

**8.20 Category type-to-filter (Section 2.3)** — the risk here is not the filtering itself but the six existing readers/writers of `txCategory.value`, so test those explicitly:
- Type a subcategory fragment (`groc`) and confirm the dropdown narrows to that one category and selects it automatically; save a transaction and confirm it is filed against that exact category.
- Type a parent name (`transport`) and confirm the parent **and** all its subcategories are listed, with the parent shown as "(General)".
- Type a fragment matching a subcategory whose parent does *not* match (`fuel`) and confirm only that subcategory appears, inside its parent's group heading, with **no** "(General)" option for the parent.
- Type nonsense (`zzz`) and confirm the hint reads "No category matches" and saving is refused by the existing validation rather than saving a blank category.
- Confirm the hint appears only while a query is present and disappears when the box is emptied, and that emptying the box restores all categories.
- **Sticky fields**: with a filter active, save a transaction, then confirm the search box is now empty, the full list is restored, **and** the just-used category is still selected (this is the ordering trap — the box must be cleared before `renderTransactionsView()` repopulates).
- **Edit**: filter to hide a category, then click **Edit** on a transaction using a *different* category; confirm the search box clears and the correct category loads and is visible.
- **Ledger switch**: with a filter active, switch Personal↔Business and confirm the box clears and the list shows only the new ledger's categories.
- **Filter dropdown**: type in the filter row's category box and confirm the transactions table re-renders immediately (the programmatic `.value` change fires no `input` event, so the re-render must be called explicitly), and that the **Clear** button resets it.
- Change the Type between Income/Expense with a filter active and confirm the filter is re-applied against the new type's categories rather than being lost.

**8.19 No horizontal overflow on phones (Section 10.4a)** — the failure mode is a page slightly wider than the screen, which is easy to miss by eye, so measure it rather than looking:
- At a 375px viewport with realistic long category names present, confirm `document.documentElement.scrollWidth` equals `clientWidth` (no sideways scroll) on the Dashboard, Business tab and Yearly Report — every tab containing a `.grid-2`.
- Confirm this query returns an empty list on each of those tabs: every element whose `getBoundingClientRect().right` exceeds the viewport width.
- Confirm a Dashboard stat card and the "Spending by Category" card report the **same** width — unequal widths are the visible symptom of overflow somewhere on the page.
- Grep each copy for `grid-template-columns: 1fr` — there should be **no** bare `1fr`; all must be `minmax(0, 1fr)`.

**8.18 Local-date handling (Section 3.9a)** — the regression this guards against is invisible for 19 hours a day, so test it by simulating the timezone rather than waiting for 2am:
- In the browser console, confirm `todayISO()` matches the date your device's clock shows, and that it equals `new Date().toLocaleDateString("en-CA")` (which is also local, `YYYY-MM-DD`).
- Confirm `todayISO()` does **not** equal `new Date().toISOString().slice(0, 10)` when the local and UTC dates genuinely differ — e.g. set the OS clock to 1 August, 02:00, Pakistan time and confirm `todayISO()` returns `2026-08-01`, not `2026-07-31`.
- With the clock still at 1 August 02:00 PKT, reload and confirm the Dashboard, Business, Accounts and Budgets tabs all open on **August**, and that the Add Transaction date field shows `2026-08-01`. Save a transaction and confirm it lands in August's Income/Expenses, not July's.
- Set the clock to 1 July, 02:00 PKT and confirm the Yearly Report opens on the FY starting July of that same year, not the previous one.
- Repeat the first check with the OS timezone set to a zone *west* of UTC (e.g. New York) at 23:00 local, confirming `todayISO()` still reports the local date and not the next day's.
- Grep each copy for `toISOString` and confirm the only remaining use is unrelated to calendar dates (there should be **no** `toISOString().slice(0, 10)` anywhere).

**8.17 Transfers (Section 3.10)** — the split between excluded and included figures is the whole point of this feature, so test both sides:
- Note the Dashboard's Income, Expenses, Net This Month and Savings Rate, and an account's balance. Make a transfer between two accounts. Confirm: **Income and Expenses are both unchanged**, Net This Month is unchanged, Savings Rate is unchanged, the source account's balance dropped by the amount, the destination's rose by the same amount, and no new slice appeared in the spending donut.
- Confirm the Accounts tab carry-forward table still satisfies `closing = opening + inflow − outflow` for **both** affected accounts (the transfer must appear in inflow/outflow there — if it doesn't, the row won't add up). Same check on the Yearly Report's "Account Summary — Opening vs Closing", where opening/closing come from `accountBalanceBefore`.
- Set a budget on the category a withdrawal would normally use; make a transfer; confirm the budget's spent figure did **not** move.
- Confirm both halves show in the Transactions list with a `⇄` badge, on the same date.
- Try to transfer From and To the same account (must be blocked), and confirm the From/To dropdowns only ever list accounts in the active ledger.
- **Mark as Transfer**: select a hand-entered pair (one income + one expense, equal amounts, different accounts), mark it, and confirm the Dashboard's Income and Expenses each drop by exactly that amount while every account balance stays identical. Select the same pair again and confirm it offers to *remove* the marking, and that doing so restores the original figures.
- Confirm Mark as Transfer refuses: a selection of 1 or 3+ rows, two rows of the same type, two rows with different amounts, and two rows on the same account.
- **Deletion**: delete one half and confirm the confirmation dialog names the other half and that both are removed, leaving both account balances correct. Confirm clicking Edit on a transfer half is blocked with an explanatory message.
- **Round-trip (8.7)**: sync, confirm `transferId` survives on both halves with the same value, and that the pair is still excluded from Income/Expenses after a pull. A blank `transferId` must come back as a normal transaction, not a transfer.

---

## 9. Setting Up Your Own Cloud Instance (for a friend, or a second device you want fully synced)

See the separate setup guide (Google Cloud Console + Google Sheet steps) for the full walkthrough. Summary:

1. Deploy the same code — create a GitHub repository, upload these files to it, and turn on GitHub Pages (Settings → Pages → deploy from branch `main`, root). Or just reuse the existing URL: the code itself has no user-specific data baked in, so the same deployed site works for anyone with their own Client ID + Sheet ID (see Section 1.1).
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

### 10.4a Horizontal overflow on phones — `1fr` vs `minmax(0, 1fr)`

**Symptom (reported 2026-08-01, from an iPhone screenshot).** On the installed PWA the four Dashboard stat cards and the dark top bar appeared to stop short of the right edge, leaving a band of empty page background beside them, while the "Spending by Category" card extended further right than they did. It read as "the stat cards aren't expanding to fit the screen."

**Actual cause — the stat cards were innocent.** The page itself was wider than the screen, so correctly-sized elements *looked* short next to an oversized one. Measured at a 375px viewport: `document.scrollWidth` was **480px**, overflowing by 105px, with **40** elements extending past the right edge.

The source was one CSS value. `.grid-2` (the two-column chart row holding the donut and the 6-month bar chart) had:

```
.grid-2 { grid-template-columns: minmax(0, 1fr) minmax(0, 1fr); }   /* desktop — correct */
@media (max-width: 900px) { .grid-2 { grid-template-columns: 1fr; } }  /* mobile — the bug */
```

A bare `1fr` is shorthand for `minmax(auto, 1fr)`, and that `auto` **minimum** means the track can never be narrower than its content's min-content width. The chart cards' min-content exceeded the available 343px, so instead of the content shrinking, the *track* grew to 464px and burst out of its own 343px grid box, dragging the whole document to 480px. The desktop rule already had this right; only the mobile override reintroduced it.

**Fix:** `minmax(0, 1fr)` in the mobile override too — a zero minimum lets the track shrink and forces the content to fit. **Never write a bare `1fr` in a `grid-template-columns` in this project**; always `minmax(0, 1fr)`. The same trap applies to flex items, whose default `min-width: auto` behaves identically.

**A second, independent instance of the same bug — the Budgets tab.** Auditing every `1fr` after the fix above turned up `.budget-list-item { grid-template-columns: 1fr auto; }`, which was initially assumed safe and then measured: the Budgets tab was **587px wide at a 375px viewport** with 33 overflowing elements — worse than the Dashboard. The `auto` amount-input column (110px) plus the min-content width of a long name like "Life Style & Social › Subscriptions (Apps / Software)" exceeded the 301px row, and the `1fr` track's auto minimum grew rather than letting the name shrink. Fixed the same way (`minmax(0, 1fr) auto`), which alone brought it to 375px with zero offenders — no ellipsis or `min-width: 0` rules were needed, and each was tested separately and confirmed unnecessary. The lesson: this trap applies to `1fr` in **any** track list, not just single-column mobile overrides. `.grid-3` and `.stat-cards` mobile overrides were converted at the same time for consistency, even though neither was overflowing yet.

Verified across **all seven tabs** (Dashboard, Transactions, Budgets, Yearly Report, Business, Accounts, Categories) at a 375px viewport with realistic long category names and budgets present: every tab reports a 375px page width and zero overflowing elements, excluding the transactions table, which scrolls inside its own `.table-wrap` by design.

**What was ruled out by measurement rather than assumption**, so it isn't retried later:
- `.main { min-width: 0 }` — tested alone, changed nothing (still 480px / 40 overflowing). Not added.
- The 6-month bar chart — already responsive (`width="100%"` plus a `viewBox`), not a fixed-width offender.
- Restacking `.donut-wrap` vertically on phones — unnecessary once the track can shrink; the donut (140px) + gap (22px) + legend minimum (160px) = 322px, which fits inside 343px, and `flex-wrap: wrap` already handles narrower cases.

Verified after the change at a 375px viewport: `scrollWidth` 375 (equal to the viewport, no horizontal scroll), the `.grid-2` track 343px, **0** overflowing elements anywhere in `body`, and a stat card and the donut card both measuring exactly 343px — the equal widths the user was asking for.

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
- **2026-07-30 (Transfers between your own accounts)** — See Section 3.10 for the full design. User reported the Dashboard's Income reading Rs 214,553 in August 2026 against real income of Rs 194,553: withdrawing Rs 20,000 cash from UBL had to be entered as two ordinary transactions (an expense from UBL, an income into Cash), so it inflated **both** Income and Expenses, dragged Savings Rate down from 53.2% to 48.3%, and put a Rs 20,000 "Cash Withdrawal" slice at 18.0% — the 3rd largest — in the spending donut. Account balances were always correct, and Net This Month was too (+20,000 −20,000 nets to zero), which is why it went unnoticed; only the gross figures lied.
  Implemented a single new `transferId` field on Transaction (Section 2.3) — presence alone marks a transfer, deliberately not a separate boolean, so the two can never disagree. A "Transfer Between Accounts" form on the Transactions tab creates both halves in one action with a guaranteed shared date, and a **Mark as Transfer** bulk action converts pairs already entered by hand, which retroactively corrects every past month with no other edits (the same button unmarks an existing pair, so mistakes are reversible). Deleting either half deletes both, and editing a half is blocked — both to prevent an orphaned half silently corrupting an account balance (Section 4 rules 6/7).
  **The critical design point**: this is *not* "exclude transfers everywhere." 8 sites exclude them (Dashboard income/expense/donut/bar, both budget functions, Business tab, Yearly Report headline/donut/bar/budgets) while 2 sites deliberately **keep** counting them — the Accounts tab carry-forward table and the Yearly Report's "Account Summary — Opening vs Closing", both of which compute `closing = opening + inflow − outflow` from their own inflow/outflow sums. Excluding transfers there would have made those rows stop adding up; this was caught by reading the call sites before editing rather than pattern-matching the filter onto all 10.
  Cloud Sync updated on **both** sides per Sections 5/6 — `SHEET_TABS.Transactions` and the hand-written `coerceTypesFromSheet` mapping. Also fixed a latent pre-existing bug found while doing this: the `txForm` submit handler rebuilds its record object from scratch, so editing a reconciliation adjustment silently stripped its `isReconciliation` flag; both that and `transferId` are now preserved through an edit.
  Verified live in the browser (Cloud copy) with 47 assertions across the whole Section 8.17 checklist, all passing: the exclusion side (Income/Expenses/Net/Savings Rate/donut all unmoved by a transfer), the inclusion side (both account tables still reconcile exactly, UBL outflow correctly 91,004 + 20,000 = 111,004), budgets untouched by a transfer, Mark as Transfer dropping Income and Expenses by exactly the amount while leaving every balance byte-identical, unmark restoring the originals, all six validation rejections (1 row, 3 rows, same type, mismatched amounts, same account, From==To), blocked edit, linked single and bulk delete, ledger scoping making a cross-ledger transfer structurally impossible, and the **mandatory 8.7 round-trip** — `transferId` survives write→sheet→read with pairs still paired, blank values coming back as ordinary transactions, business-bucket transfers preserved, and the exclusion still holding after a simulated pull. Zero console errors.
  Mirrored identically into the PC (`D:\money-manager\index.html` + `app.js`) and iPhone (`D:\money-manager\Money Manager (iPhone).html`) copies — no CSS changes were needed in any copy (reuses the existing `.hint`/`.tx-form`/`.form-row` classes plus one inline-styled badge). Neither has Cloud Sync, so no schema work there. Unlike previous sessions, the iPhone copy **was** live-tested this time by temporarily copying it into the Cloud project folder (the browser only executes `file://` JS inside that folder) — 12 core assertions passed there including both account tables reconciling and linked delete; the temp file was then deleted. Since the iPhone copy and `app.js` received byte-identical JS edits, that run validates the PC copy's logic too; the PC copy's own `index.html` markup was verified by grep only. Note `python`/`node` are still unavailable here (the `python` on PATH is a Microsoft Store stub), so serving the PC copy's multi-file structure directly remains impossible.
  **All three copies verified consistent**: 8 exclusions, 1 transfer form handler, 1 Mark as Transfer handler, 1 edit guard, 1 badge, and both KEEP sites unmodified in each.
- **Action required**: on the live site, select the existing Rs 20,000 pair (the UBL "Cash Withdrawal" expense and the Cash "Cash Deposite" income, both dated 2026-08-02) and use **Mark as Transfer** — August's Income should drop from Rs 214,553 to Rs 194,553, Expenses by Rs 20,000, and Savings Rate rise to about 53.2%, with both account balances unchanged.
- **2026-07-30 (hosting moved from Netlify to GitHub Pages)** — The Cloud copy is now live at **<https://mudassar-shah.github.io/money-manager-cloud/>**, served by GitHub Pages from the `mudassar-shah/money-manager-cloud` repository (branch `main`). Netlify is no longer used: its free plan meters "credits" against deploy activity, and iterating on the app in a single day could consume a large share of the monthly allowance, while GitHub Pages has no equivalent limit for a static site. The old `peppy-lily-416823.netlify.app` URL is abandoned. Deployment is now "upload/push to the repo and it publishes itself" instead of a manual folder-drag — which also removes the recurring partial-upload hazard noted in the session backup, where Netlify Drop once uploaded only the dragged file and left `manifest.json`, `sw.js` and the three icons 404ing on the live site.
  Section 1.1 added to record the URL, the hosting mechanism and two things that were checked rather than assumed: (a) GitHub Pages serves from a **subpath** (`/money-manager-cloud/`) rather than a domain root, and every asset reference in the app is already relative (`manifest.json`, `apple-touch-icon.png`, `register("sw.js")`, the manifest's `./` start_url/scope/icons, and `sw.js`'s `./` SHELL_FILES), so the PWA, install prompt, icons and offline caching all work unchanged — these must never be converted to absolute `/…` paths, which would break on a subpath while looking correct on a root domain; and (b) Cloud Sync requires `https://mudassar-shah.github.io` to be added to the OAuth 2.0 Client ID's **Authorized JavaScript origins** in Google Cloud Console, since sign-in uses `google.accounts.oauth2.initTokenClient` which validates the page origin — without it, "Sign in with Google" fails on an origin mismatch. **No application code was changed for this migration.** Also corrected the Section 1 table, which still listed the three copies under stale `E:\Claude\…` paths; they are and have been under `D:\`. Change Log entries dated before 2026-07-30 refer to Netlify redeploys as accurate history, not live instructions.
- **2026-08-09 (multi-currency — currency belongs to the account)** — The user plans to move to Dubai or Saudi (possibly the UK later) and wants to keep using this app for life. Before this, one global setting chose a symbol and nothing more, so switching it would have relabelled rupees as dirhams **and added the two together** in every total. Implemented Section 3.15: currency on the **account**, conversion only where sums span accounts.
  - **Measured before building**, at the user's request: **54** display points and **22** cross-account sums across **15** functions. The four per-account balance functions needed no change — each already works on one account, so one currency.
  - `fmtMoney(n, code)` gained an optional currency defaulting to the main one, so all 54 existing call sites kept working untouched; only the 6 per-account displays were switched to pass an explicit code. 18 sums were converted via `txAmountInMain`; the 4 per-account inflow/outflow sums were **deliberately left alone** and verified still reading `t.amount`.
  - **A missing rate is never guessed.** `convertAmount` returns null, the amount is excluded, and a warning names the currency — chosen over a 1:1 fallback, which would hand the user a confident wrong number.
  - Rates are manual (anchored as "PKR per 1 unit", so they stay unambiguous if the main currency changes), with a **user-initiated** "get latest" button. Never automatic: the app otherwise talks to nobody but Google, and a failed fetch leaves typed rates untouched.
  - Cross-currency transfers gained a second amount box, shown only when the two accounts' currencies differ, pre-filled from the rate but overridden by what the bank actually delivered. `Mark as Transfer` now only requires equal amounts when the currencies match.
  - **The release test was equality, and it passed with zero differences.** The pre-change `index.html` was extracted from commit `0c7f158`, the identical dataset run through both, and every rendered figure deep-compared across Dashboard, Accounts (total, list, carry-forward, outstanding), Transactions, Budgets, both Yearly Report ledgers and the Business tab including deals and credit stock: **0 differences**. That test is only possible while all data is single-currency, which is exactly why this was done before the user moves.
  - Multi-currency behaviour verified: with no AED rate the AED money is excluded and warned; with a rate, Total Balance becomes 150,000 + 15,000×76 and Dashboard Income 100,000 + 5,000×76, while the AED account still shows **AED 15,000** and UBL **Rs 150,000**; changing a rate moves only converted totals, never a per-account balance; a PKR→AED transfer took 76,000 out and put 990 in with one shared `transferId` and no income/expense movement; deleting either half restored both. Sync verified against the user's **current sheet shape** — Accounts without `currency` and a Settings tab holding only the single old `currency` row — everything intact and every account defaulting to PKR; round-trip preserves currency, rates as numbers and display currencies; a corrupt rates cell and negative/zero rates are rejected without crashing.
  - Two console `ReferenceError`s appeared mid-build and were investigated rather than dismissed; both came from the preview auto-reloading between a call being added and its function being added. Proven stale by confirming every function is defined, `loadData()` runs with all accounts stamped, and all seven views render cleanly.
  - **Not yet mirrored** into the PC and iPhone copies — see the Action required note below.
- **Action required**: the PC (`index.html`/`app.js`) and iPhone copies are **behind** on the multi-currency change. They remain fully working on their own data, but no longer match the Cloud copy, so Section 1's "identical logic" rule is temporarily broken. Mirror before making further shared-logic changes.
- **2026-08-09 (Business ↔ Personal transfers)** — The user takes money out of the business, puts money in, and — the case they emphasised — **borrows** from the business and repays later. Cross-ledger movement had been proposed on 2026-08-01 and explicitly declined at the time; the need became real, so it was revisited (Section 3.14).
  - **Treated as a pure transfer on both sides**: both halves excluded from every income/expense figure in both ledgers, both balances updating in full. The "count the personal side as income" model was reconsidered and rejected *with* the user — defensible for a permanent owner's draw, wrong for borrowing, which dominates here. The user keeps control per movement: use the form and it doesn't touch income/expense; enter two ordinary transactions instead and it counts normally. Trade-off stated to them: permanently drawing profit via this form flatters Savings Rate.
  - **A separate form on the Accounts tab, at the user's explicit request.** They asked that the everyday Transactions-tab transfer form be left exactly as it is — it is used constantly for same-ledger moves, while cross-ledger is "once in a blue moon", and merging them would have meant changing a daily-use form to serve a rare case. The two are now mirror images: each refuses the other's job and points the user to the right place. The everyday form was verified untouched, still listing only the active ledger's accounts.
  - Mechanically reuses Section 3.10 — one shared `transferId`, `countsInTotals` exclusion, linked deletion — with the defining difference that **each half takes the bucket of its own account** rather than any active toggle. `Mark as Transfer` now also accepts cross-ledger pairs so hand-entered movements can be corrected retroactively.
  - Verified: business 135,000 → 105,000 and personal 130,000 → 160,000 on a 30,000 transfer, with **all six headline figures byte-identical** (Personal Income/Expenses/Savings Rate, Business Income/Expenses/Net Profit); Total Balance and Total Business Balance moved as they should, since real money moved; a **borrow-and-repay round trip returned both balances exactly to their starting values** while leaving every income/expense figure untouched throughout; both halves shared one `transferId`, date and amount, with each bucket matching its own account and one half on each side; linked deletion removed both halves and **fully restored both balances** with the confirmation naming the other account; same-ledger and same-account attempts were refused and created no rows; `Mark as Transfer` on a hand-entered cross-ledger pair dropped Personal Income by 15,000 and Business Expenses by 15,000 while moving no balance. Zero errors.
  - **One test assertion initially failed and was traced to the test, not the code** — it asserted post-deletion balances as though both transfer pairs had been removed when only one had. Re-verified from a clean single-transfer state, where deletion restored balances exactly.
  - Mirrored into the PC and iPhone copies; all markers reconcile and the old same-ledger guard is gone from all three. The iPhone copy was **functionally executed**, all 8 checks passing, then deleted from the upload folder.
- **2026-08-08 (accounts excluded from Total Balance — spending that predates tracking)** — Tracking began July 2026 for both ledgers, but a year of school fees was paid in January and the business ran for months before. The user logs those costs against a "Pervious Paid" holding account so each month's expense is honest, but that account drifts negative and drags Total Balance down — which is not a real loss, just an artefact of the start date. **Confirmed with the user that the original lump sums were never recorded** (there is no pre-July data at all), so there is no double-counting. Added `excludeFromBalance` on **Account** (Section 3.13).
  - **The exact mirror of 3.12**, and the two must not be confused: an excluded *category* is skipped by income/expense but kept in balances; an excluded *account* is skipped by Total Balance but its transactions still count fully as income/expense — which is the entire point.
  - **Deliberately narrow**: only `totalNetWorth` changed, so exactly two figures move — "Total Balance (Personal)" and "Total Business Balance". Because that function is already bucket-filtered, **one flag serves both ledgers** with no extra work. `accountCurrentBalance` is untouched, so per-account balances, the carry-forward table, the Yearly Report account summary and Reconcile all behave identically. The account stays visible everywhere with a "not in total" badge — hiding it would destroy the record the user is deliberately keeping.
  - **Caught during implementation**: accounts have **no edit form** (only add/delete, and delete is blocked once transactions exist), so an existing holding account could never have been flagged. Added an "Exclude from total" / "Include in total" toggle directly on the account card, alongside a tickbox on the Add Account form for new ones.
  - Verified: an **old 7-column Accounts sheet** loads with `openingBalance` numeric, `openingDate`, `color` and `bucket` intact and the absent flag becoming `false` as a real boolean. Behaviourally, Total Balance went 141,982 → **150,000** and Total Business Balance 35,000 → **40,000**, while Income, Expenses (Rs 8,018 both sides), the spending donut, the Yearly Report, every per-account balance and the whole carry-forward table were **byte-identical before and after**; the account stayed listed, the badge rendered, and unflagging restored the originals exactly. Round-trip passed on all four accounts with every field surviving. Zero errors.
  - Mirrored into the PC and iPhone copies; marker counts reconcile with the two Cloud-only differences confirmed to be the sync schema and sync reader lines. The iPhone copy was **functionally executed**, all 10 checks passing, then deleted from the upload folder.
- **2026-08-08 (excluded categories — loans given and repaid)** — User lends money from UBL and is repaid into UBL; both legs were counted, overstating July Income *and* Expenses by Rs 6,000 each while the balance stayed correct. Verified first that the transfer mechanism (3.10) **cannot** solve this: `Mark as Transfer` requires two *different* accounts and a loan returns to the same one, and pairing would wrongly treat an unrepaid loan as an expense and break outright on a partial repayment. Added `excludeFromTotals` on **Category** — a tickbox "Don't count as income or expense" (Section 3.12).
  - Because the flag lives on the category, it is **retroactive with no data migration**: the user had already filed these under dedicated categories, so ticking two boxes corrects July and every earlier month instantly. Partial and late repayments need no special handling; an unrepaid loan correctly stays out of expenses because the money is still owed, not spent.
  - The nine reporting filters that previously read `!isTransferTx(t)` now share one predicate, **`countsInTotals(t)`**, so the transfer rule and the exclusion rule can never drift apart. Balance functions deliberately still count both.
  - Also added the **"Lent out, not yet returned"** line on the Accounts tab (flagged personal money out minus back), hidden when it nets to zero so it never appears for someone who doesn't lend, and relabelled if negative rather than showing a negative amount. Flagged categories carry a "not counted" badge in the category list.
  - **Sync safety was audited before writing any code, at the user's explicit request** — their live data exists only in Google Sheets. `readAllFromSheet` maps columns **by name from the sheet's own header row**, so adding a column cannot shift or scramble existing data; `pushAllToSheet` clears `A:Z` before rewriting so no stale column survives; the boolean uses the same string-compare as `isReconciliation`, proven in production since July.
  - Verified: an **old 7-column Categories sheet** (exactly the user's current one) loads with every field intact — `parentId`, numeric `order`, `bucket` — and the absent flag becomes `false` as a real boolean, never `undefined`/`NaN`; blank/`TRUE`/`true`/`FALSE` cells coerce correctly; a full round-trip preserves the flag with **every other field identical**. Behaviourally: Income 106,000 → 100,000 and Expenses 26,000 → 20,000, the Yearly Report follows, the loan leaves the donut, **every account balance and the entire carry-forward table are byte-identical before and after**, a partial repayment correctly shows Rs 3,000 outstanding, unticking restores the original figures **exactly**, and flagged rows stay visible in the transactions list. Zero errors.
  - **A real bug was caught by these tests and fixed**: hiding the outstanding block returned early without clearing its value, leaving a stale money figure inside a hidden element — the kind of thing that resurfaces later looking authoritative.
  - Mirrored into the PC and iPhone copies; marker counts reconcile, with the two Cloud-only differences confirmed to be exactly the sync schema line and the sync reader line. The iPhone copy was **functionally executed**, all 9 checks passing, then deleted from the upload folder.
- **2026-08-08 (Yearly Report covers both ledgers)** — User asked whether a yearly report existed for the business and confirmed it should use the **same 1 July – 30 June fiscal year** as personal, with the same unlimited year navigation. Verified first that it genuinely did not exist: every FY function was hardcoded to `"personal"` and the page had no ledger toggle at all. Added a Personal/Business toggle (`yearlyLedger`) matching the Categories and Transactions pattern — Section 4 rule 2 forbids *mixing* ledgers in one figure, not offering a switch, so the rule was clarified rather than broken (Section 3.7).
  - Business mode relabels **Net Savings → Net Profit** and **Savings Rate → Profit Margin** (same arithmetic, Business-tab naming), hides **Annual Budget vs Actual** since business budgeting doesn't exist, and shows a new **Annual Profit by Sale/Deal** card in its place — Section 3.11's grouping and FIFO credit costing applied across all 12 FY months instead of one.
  - `buildDeals` was generalised to accept either a month string or a **Set of months**, and the deal-table renderer extracted into one shared `renderDealTable` used by both the monthly and annual cards, so the two can never drift apart.
  - **This is the single most leak-prone area in the app** — `renderYearlyView`, `renderFyBar` and `renderFyAccounts` each leaked business data into personal totals through independent code paths on 2026-07-07. The filter is therefore threaded explicitly into every FY query with no function defaulting to a bucket; test 8.22 exists solely to catch a regression.
  - Verified against a dataset holding personal and business data in the *same* fiscal year: Personal reads 350,000 / 50,000 with only personal accounts and categories; Business reads 50,000 / 29,000 with only business ones; **the two sum to 400,000, the true all-ledger total**, proving nothing is double-counted or dropped. Labels and cards swap correctly, the deal card spans month 1 *and* month 12 of the FY, FIFO costing carries over (50 credits × 170 = 8,500, 71.7%), a sale dated the next July is correctly excluded, the toggle survives prev/next navigation, and empty years render without error. Every render path was exercised under try/catch with **zero** errors.
  - Two `ReferenceError`s appeared in the console during this work (`personalAccounts`, `renderBizDeals`) and were investigated rather than dismissed: both cited a line inside the Cloud Sync section, and were traced to the preview auto-reloading *between* two halves of a rename. Proven stale by confirming the final file contains no such reference, that the whole script parses (the last function in the file is defined), and that `renderFyAccounts` runs cleanly on both ledgers.
  - Mirrored into the PC (`index.html` + `app.js`) and iPhone copies, with the refactored business block re-spliced from the Cloud copy so all three stay byte-identical; marker counts reconcile. The iPhone copy was **functionally executed** — all 10 checks re-run and passing, then deleted from the upload folder.
- **2026-08-01 (business profitability: deal grouping + credit FIFO costing)** — The Business tab could only answer "how much cash moved this month"; it could not answer "did *that* sale make money", because nothing linked a sale to its costs. Established through a long clarification with the user that **two different businesses** run through this one ledger, each needing a different mechanism (Section 3.11). Added three fields — `ref`, `panel`, `credits` — and two read-only cards, **Profit by Sale/Deal** and **Credit Stock**.
  - **Goods**: one invoice, several vendors, plus a negotiated commission and courier charges, all grouped by a shared `ref`. Grouping is by code not date, so a cost paid in a different month still lands on the right deal. Commission is a **fixed negotiated amount, never a percentage** (confirmed — it can exceed the sale, so losses are real and are shown as negative).
  - **IPTV**: two physically separate vendor panels, so `panel` partitions credits into independent stocks that never cross. Each purchase keeps its own `amount / credits` unit cost (observed 50–260, market-driven), and sales consume **oldest credits first**, so a sale can span two purchases and gets a blended cost. The user picks only the panel and the credit count — never a specific purchase — because credits are fungible inside a panel and asking would invent information they don't have.
  - **The app deliberately knows nothing about months or the vendor's bonus tiers** (cheap: 6mo=5 credits, 12mo=10; premium: 1/month). Encoding those would mean maintaining someone else's commercial rules; instead the bonus *is* the smaller number typed, and it surfaces as a higher margin. A hint next to the Credits box lists the tiers.
  - **Shortfall is reported, never hidden**: credits sold beyond what was purchased are costed at 0 and flagged on both cards, rather than quietly overstating profit.
  - **Net Profit (3.8) and deal profit will not agree, and neither is wrong** — cash-moved-this-month versus what-each-sale-earned. Kept in separate cards with no attempt to reconcile them.
  - Verified in-browser against a purpose-built dataset covering every boundary, all passing: the blended span (10×170 + 5×180 = **2,600**); **date ordering** (a later-dated purchase entered *first* in the data, earlier one still consumed first); panel isolation (Premium costed at 250, never touching Cheap); a goods deal picking up a cost dated the *following* month (12,100 total, 39.5%); commission exceeding the sale yielding a **−1,000 loss**; a zero-credit purchase ignored with no `Infinity`/`NaN`; shortfall of 999 credits costed at 0 and warned on both cards; stock card confirmed **not** month-scoped; unattributed sales counted and reported; and the existing Expenses figure unchanged (30,100 → 30,100). Mandatory **8.7 round-trip** passed for all three fields across four representative row shapes, with `credits` returning as a real number and blanks as `0`, never `NaN` or `"0"`. Zero console errors; rendered output visually checked.
  - Mirrored into the PC copy (`index.html` markup + `app.js`) and the iPhone copy, with the shared JS **spliced from the Cloud copy so all three are byte-identical**. Marker counts reconcile (`cloud = app.js + index.html`, iPhone = cloud). No CSS was needed — existing `.hint`, `.cat-hint`, `.tx-table`, `.table-wrap`, `.cat-badge` and `.budget-total-row` classes were reused. Cloud Sync columns are Cloud-only by design (Section 1), correctly absent from the other two. The iPhone copy was **functionally executed**, not merely diffed — temp-copied into the Cloud folder to escape the `file://` sandbox, the full FIFO/deal suite re-run and passing, ledger row visibility confirmed to toggle, then deleted so it cannot reach an upload.
- **2026-08-01 (type-to-filter on both category dropdowns)** — User pointed out that picking a category meant scrolling the entire ~79-item dropdown "from outer to lower", and asked for a search box on the Add Transaction picker *and* the Transactions filter picker. Added `txCategorySearch` and `filterCategorySearch` (Section 2.3, test 8.20): typing narrows the dropdown, a parent-name match keeps that parent's whole group, and a single remaining match is auto-selected. A hint shows `N of M categories`, or `No category matches`.
  - **Deliberately kept the native `<select>` as the source of truth** and only changed which options it contains. `txCategory.value` is read or written in six places (submit handler, `editTransaction`, the post-save sticky-field restore, the `txType` change handler, `renderTransactionsView`, initial load), and the sticky-field logic is order-sensitive about when it re-applies `.value`. A custom combobox would have had to touch all six; option-list filtering touched none, and it keeps the native iOS picker.
  - **The one real trap** was ordering: the search box must be cleared *before* `renderTransactionsView()` re-populates after a save, otherwise the sticky category can't be restored from a filtered list. Also cleared on **Edit** (an active filter could hide the category being edited), on ledger switch, and by the filter row's **Clear** button. The filter dropdown's re-render is called explicitly, since setting `.value` in code fires no `input` event.
  - **No separate Business work needed** — confirmed these are the only two category dropdowns in the app (the Business tab has no category picker, only its chart), and business transactions use this same form with the ledger toggle flipped, so both boxes are already ledger-scoped via the existing `bucket` argument.
  - Verified in-browser against a padded 74-expense-category list, 8 assertion groups, all passing: `groc` → 1 option auto-selected with hint "1 of 74 categories"; `transport` → parent plus all subs with "(General)"; `fuel` → only the subcategory, with **no** "(General)" for its non-matching parent; `zzz` → 0 options and "No category matches"; clearing restores all 74 and blanks the hint; the save-ordering trap (saved against the right category, box cleared, full list back, sticky selection intact); Edit clearing the filter and loading the right category; type-switch preserving the filter and re-applying it to income categories. Filter dropdown separately verified: auto-select re-renders the table to 1 row, parent selection still rolls up to 2 rows, **Clear** restores all 76 options, and the ledger switch clears both boxes. Broadening a query from `fuel` to `transport` correctly *keeps* the existing selection rather than resetting it. Zero console errors.
  - Mirrored into the PC copy (`index.html` markup, `app.js` logic, `style.css`) and the iPhone copy. Marker counts reconcile across all three (Cloud's 8 = PC's `app.js` 7 + `index.html` 1 = iPhone's 8). The iPhone copy was **functionally executed**, not just diffed — temp-copied into the Cloud folder to get around the `file://` sandbox, all 8 groups re-run and passing with zero console errors, then deleted so it can't reach an upload.
- **2026-08-01 (mobile horizontal overflow — `1fr` → `minmax(0, 1fr)`)** — User sent an iPhone screenshot of the installed PWA showing the four Dashboard stat cards and the top bar stopping short of the right edge with a band of empty background beside them, while the "Spending by Category" card ran past them. Diagnosed by measurement rather than eye: the stat cards were correct and the *page* was too wide — 480px at a 375px viewport, with 40 elements past the right edge. Cause was a single CSS value, `.grid-2 { grid-template-columns: 1fr }` in the ≤900px override, where a bare `1fr` means `minmax(auto, 1fr)` and the `auto` minimum forbids the track shrinking below its content, so the track grew to 464px and burst its own 343px grid box. The desktop rule already used `minmax(0, 1fr)` correctly; only the mobile override regressed it. Full write-up in Section 10.4a, test in 8.19.
  - Auditing the remaining `1fr` declarations found a **second, worse instance**: `.budget-list-item { grid-template-columns: 1fr auto }` left the Budgets tab at **587px** with 33 offenders. Fixed identically. `.grid-3` and `.stat-cards` mobile overrides converted too, pre-emptively.
  - Ruled out by testing each in isolation, so they aren't retried: `.main { min-width: 0 }` (no effect at all), restacking `.donut-wrap` on phones (unnecessary once the track can shrink), and adding ellipsis/`min-width: 0` rules to the budget rows (unnecessary). The 6-month bar chart was already responsive. Final change is four `1fr` → `minmax(0, …)` substitutions and nothing else.
  - Verified across all seven tabs at 375px with realistic long category names and budgets: 375px page width and zero overflowing elements on every one, and a stat card and the donut card measuring an identical 343px. Applied to all three copies (Cloud `index.html`, PC `style.css`, iPhone HTML).
- **2026-08-01 (dates follow the device's clock, not UTC)** — User reported the Dashboard opening on **July** while it was already 1 August, and asked whether a setting controlled the default month. There is no such setting, and none is wanted: the default already *intended* to be the current month (`dashMonth = monthKey(todayISO())`). The fault was one level down in `todayISO()`, which was `new Date().toISOString().slice(0, 10)` — `toISOString()` converts to UTC first, so for a user at UTC+5 it reported *yesterday* every day between midnight and 5am. `monthKey()` then sliced `"2026-07-31"` to `"2026-07"`. Fixed to build the string from local date parts (`getFullYear`/`getMonth`/`getDate`), matching what `shiftMonth` had always done — the codebase had been contradicting itself. Full write-up in Section 3.9a, test in 8.18.
  - **The default month was the mildest symptom.** The same function was the default for the **Add Transaction date**, so a transaction entered before 5am was silently dated into the previous month — landing in the wrong month's Income/Expenses and the wrong month's budget. On 1 July before 5am, `fiscalYearStartOf(monthKey(todayISO()))` resolved to the **previous fiscal year**, putting the Yearly Report a full year out. Also affected: the Transfer date default, the Reconcile statement-date default, account opening-date defaults, and the Export Backup filename.
  - Verified in-browser in a **UTC+5 environment** (`getTimezoneOffset()` = −300, matching Pakistan), comparing old and new implementations against explicit local wall-clock times: at 1 Aug 00:30 and 02:00 local the fix yields `2026-08` where the old code yielded `2026-07`; at 1 Jul 02:00 local it yields `2026-07` where the old code yielded `2026-06`; at 31 Jul 23:30 local both agree, confirming the fix is a no-op whenever local and UTC already match. After the fix `todayISO()` returned `2026-08-01`, equal to both the device date and `toLocaleDateString("en-CA")`; `dashMonth`/`bizMonth`/`accountsMonth`/`budgetMonth` were all `2026-08`, `fyStartMonth` `2026-07`, and the Add Transaction and Transfer date fields both `2026-08-01`. Zero console errors. Confirmed no `toISOString().slice` remains in any of the three copies.
  - Applied identically to all three copies (Cloud `index.html`, PC `app.js`, iPhone HTML). Deliberately **not** fixed: each tab's month is still evaluated only once at page load, so an installed app left open across midnight on the 1st shows the old month until reloaded — re-evaluating on tab switch would discard a month the user had deliberately navigated to (reasoning in Section 3.9a).
- **2026-07-07 (service worker: network-first for the app shell)** — Before the PWA install setup had even been deployed once, caught a real forward-looking problem: the original service worker cached the main HTML page cache-first, which would have kept serving an old cached version of the app forever after future redeploys, even with internet access, once someone had "Add to Home Screen" installed it — silently freezing them on stale code with no obvious way to tell. Fixed by splitting the fetch strategy: the app shell (`.html` / navigations) is now network-first (falls back to cache only when offline), while static assets (icons, manifest) stay cache-first for speed since they rarely change. Section 10.3 updated. Verified the URL-matching logic directly (`index.html`/`/` → app shell, `manifest.json`/`icon-192.png`/`sw.js` → static asset) and confirmed the page still renders correctly with the updated worker active and registered, zero console errors, in both Cloud and PC/iPhone copies.
