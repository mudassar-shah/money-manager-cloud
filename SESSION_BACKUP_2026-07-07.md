# Money Manager — Session Backup (2026-07-07)

This is a full backup/recap of a long working session on the Money Manager app, written so it can be pasted into a fresh Claude conversation later if a bug shows up and there's no chat history to refer back to. It explains what exists, what was fixed, why, and what's still open.

> **Superseded on 2026-07-30 — hosting only.** This document is a historical record and its Netlify references (the `peppy-lily-416823.netlify.app` URL in Section 1, and the deploy notes in Section 7) are no longer current. The Cloud copy now lives at **<https://mudassar-shah.github.io/money-manager-cloud/>** on GitHub Pages, and deploying means uploading/pushing to the `mudassar-shah/money-manager-cloud` repo rather than dragging a folder. Everything else here — the bugs, the fixes, the reasoning, the three-copies structure — is still accurate. See `DOCUMENTATION.md` Section 1.1 for the current hosting setup. The paths in Section 1 are also under `D:\`, not the `E:\Claude\…` shown in older docs.

**If you're starting a new chat with a bug report**: paste this file plus `DOCUMENTATION.md` (technical reference, kept up to date with every change — has the authoritative field lists, formulas, and Change Log) and `USER_MANUAL.md` (plain-language how-to guide). Those two files are the living source of truth; this document is the narrative of how they got that way.

---

## 1. What this software is

A self-hosted personal + business money manager, built as vanilla HTML/CSS/JS with no build step and no framework — chosen specifically for full data ownership and zero cloud dependency by default.

**Three copies**, meant to be logically identical except where noted:
| Copy | Location | Storage |
|---|---|---|
| **Cloud** | `money-manager-cloud/index.html` — live on Netlify at `peppy-lily-416823.netlify.app` | Browser localStorage + optional Google Sheets sync |
| **PC** | `money-manager/index.html` + `app.js` + `style.css` | Browser localStorage only, no sync |
| **iPhone** | `money-manager/Money Manager (iPhone).html` (single file) | Browser localStorage only, no sync |

The Cloud copy is the one actually used day to day (via Netlify, in Safari/Chrome on phone and desktop). The PC/iPhone copies are non-synced fallbacks.

## 2. Core features

- Personal and Business "ledgers" fully isolated from each other (own accounts, categories, transactions; Dashboard/Budgets/Yearly Report are personal-only, Business tab is business-only).
- Accounts (bank/cash/savings/credit card), each with an opening balance and a running computed balance.
- Categories with one level of subcategories, drag-free reordering (↑/↓ buttons), and a Personal/Business toggle.
- Transactions with a Personal/Business toggle (not a per-item picker — see Section 4), multi-select delete, and a "Move to Account" bulk action for fixing mis-entered transactions.
- Budgets per category per month, with rollup from subcategories into their parent's total.
- Reconciliation — enter your real bank balance, the app adds one adjustment transaction to close the gap, rather than you editing old entries.
- Yearly Report on Pakistan's fiscal year (1 July–30 June).
- Cloud Sync via the user's own Google Sheet + own Google OAuth Client ID (Cloud copy only) — fully private per person, nothing shared just because the code is identical.
- Mobile-friendly layout with an installable "app" experience (Add to Home Screen / Install app) — see Section 6.

## 3. Documentation discipline ("Rule Zero")

Early in this project a real bug (Cloud Sync silently dropping subcategory/order data) came from changing code without first checking it against a written contract of what fields exist and where they need to survive. Because of that, every change since has followed a fixed process, written into `DOCUMENTATION.md`:

1. Update the documentation first (describe the change before making it).
2. Make the code change in whichever copies are affected — if it's shared logic, apply it to all three.
3. Test it (there's a Test Checklist in Section 8 of `DOCUMENTATION.md`; any Cloud Sync schema change requires a mandatory round-trip test).
4. Update the Change Log.

This is why `DOCUMENTATION.md`'s Change Log is the single most complete record of what happened and why — read it in full if you need exact detail on any specific fix below.

## 4. Bugs found and fixed this session (chronological)

All of these were found via a deliberate, exhaustive audit (formula-by-formula, tab-by-tab) plus direct verification against the live Cloud site's real data — not guesses. Each was fixed in all three copies and independently re-verified with test data (add transaction → check number → confirm) before being marked done.

1. **`renderYearlyView` missing bucket filter** — the Yearly Report's headline income/expense/net/savings-rate numbers had no personal/business filter, so Business transactions leaked into personal totals.
2. **`renderFyBar` missing bucket filter** — the Yearly Report's 12-month bar chart had its own separate, entirely unfiltered transaction query — same leak, different code path.
3. **Budget rollup gap** — budgeting a *parent* category (e.g. "Food") never counted spending logged against its *subcategory* (e.g. "Groceries"); the budget would sit at 0% used forever. Fixed with a `categoryAndChildIds()` helper that's now used everywhere a budget or filter needs to include subcategory spending.
4. **Reconciliation bucket leak** — reconciling an account created an adjustment transaction with no bucket tag, always defaulting to "personal" — so reconciling a *business* account would silently push that adjustment into the personal Dashboard instead of Business tab. Fixed, plus added a self-healing repair (`repairReconciliationBuckets`) that auto-corrects any bad record on load.
5. **Yearly Report account leak** — the "Account Summary — Opening vs Closing" table at the bottom of the Yearly Report listed *all* accounts including business ones, violating the personal-only rule for that report. Found by visually checking the live site (the business account "Jazz cash" was showing up there). Fixed by filtering to personal accounts only.
6. **Donut chart legend long-name wrap** — reported directly by the user from the live Dashboard: a long category name like "House Hold & food › Groceries / Ration" wrapped to two lines and pushed the amount/percentage onto its own broken line too. This was the exact same underlying issue as an earlier fix to the Budget Progress list, just never applied to this widget. Fixed the same way: truncate with an ellipsis, full name on hover, amount stays on one line.
7. **Service worker would have frozen installed users on stale code** — caught *before* it caused a real problem: the original PWA service worker cached the app shell cache-first, which would have meant that after "Add to Home Screen," future code updates would never reach the installed icon even with internet access, with no obvious symptom. Changed to network-first for the HTML shell (cache is only a fallback when offline); static assets (icons, manifest) stay cache-first since they don't change.

Checked at the time and confirmed **not** to have corrupted any real data: the budget rollup gap and reconciliation bucket leak were both fixed before the user had ever actually hit those specific conditions in their real data (no budgets were set on parent categories yet, no reconciliations had been logged yet).

## 5. Features added this session

- **Business tab isolation was extended and hardened** (it already existed going into this session, but items 1, 2, 4, 5 above were all specifically about places where that isolation was leaking).
- **Transactions tab Personal/Business toggle** — replaced a "Ledger" dropdown (a per-transaction picker in the Add form, plus an "All Ledgers/Personal/Business" filter) with a page-level toggle matching how the Categories tab already worked. New transactions now always take the bucket of whichever toggle is active; there's no more "all ledgers mixed together" view and no way to forget to switch a dropdown before logging an entry.
- **Bulk "Move to Account"** — select several transactions and move them all to a different account in one action. Fixes the exact real scenario the user hit: entering a batch of transactions under "Cash" when they should have been under a real bank account. Only `accountId` changes; date/type/category/amount/description/bucket are untouched, and both accounts' balances update automatically and correctly since balances are always computed live from which transactions point at which account.
- **Category filter rollup on the Transactions tab** — picking a parent category (e.g. "Transport") in the filter now shows entries from its subcategories too (e.g. "Transport › Fuel Car"), not just entries logged directly against the parent. Same underlying fix as the budget rollup (item 3 above), applied to a different filter.
- **Mobile-responsive layout + installable app** — see Section 6, the biggest piece of work this session.
- **`USER_MANUAL.md`** — a new, separate plain-language guide (no jargon) covering every feature above from a "how do I..." angle, with a Quick Answers FAQ at the bottom. `DOCUMENTATION.md` stays the technical reference; this is the one to hand to a non-technical user (including a friend using their own copy).

## 6. The mobile / installable-app work

**What was asked for**: the user wanted the app usable "as an app" on their iPhone instead of through a browser, and initially asked about publishing it to TestFlight.

**Why TestFlight/native app wasn't pursued**: building any installable iOS file (TestFlight, sideloaded, or App Store) requires Xcode, which only runs on macOS — this session runs on Windows, so that's a hard tooling blocker regardless of Apple account status. Also clarified a terminology mix-up: "APK" is Android's package format, not Apple's (Apple's is "IPA") — but that distinction didn't matter much since the Mac/Xcode requirement blocks *any* native iOS build here either way. Real Android APKs are technically more feasible on Windows (Android Studio does run on Windows) but require a much bigger, separate toolchain setup (Android SDK, signing keys, wrapping the web app in a native shell) that wasn't pursued once the actual root complaint turned out to be fixable more directly (see below).

**What was actually wrong and what was built instead**: the real complaint ("stretches like PC, some things don't show properly on iPhone Safari") turned out to be that the Cloud copy's layout was desktop-first — the sidebar rendered as a ~600px-tall block that had to be scrolled past before any page content appeared on a phone-width screen. That's a fixable responsive-design problem, not something that needs a native app:

- Added a slim fixed top bar with a hamburger (☰) button, visible only below 700px width.
- The sidebar becomes an off-canvas slide-out drawer (`position: fixed`, slides in from the left) instead of stacking above the content.
- Page content is now visible immediately under the top bar, no scrolling past navigation first.
- Made this installable as a real "app icon" experience via the browser's own built-in feature (no App Store involved): added `manifest.json`, a service worker (`sw.js`), and three PNG icons (`icon-192.png`, `icon-512.png`, `apple-touch-icon.png` — a purple rounded square with "Rs" on it, generated via Python/PIL after a canvas/base64 approach proved unreliable to transcribe by hand).
- On iPhone: Safari → Share → **Add to Home Screen**. On Android: Chrome → ⋮ menu → **Install app**. Either way you get a real home-screen icon that opens full-screen with no browser bar, and can keep working offline once loaded.
- The service worker deliberately never intercepts any request to a different origin (so Google Sheets sync and Google Sign-In are completely unaffected), and uses network-first for the main page specifically so future code updates reach an already-installed icon automatically instead of getting stuck on a stale cached version forever (item 7 in Section 4 — a bug caught before it ever shipped).

## 7. Netlify deploy notes (things that actually went wrong during this session)

- **Netlify Drop sometimes only uploads the single file you drag, not the whole folder.** This happened once here: after adding `manifest.json`/`sw.js`/the three icon PNGs, a redeploy only picked up `index.html` and left the five new files 404ing on the live site (confirmed by fetching each URL directly and seeing Netlify's error page HTML come back instead of the real file). Fixed by re-dragging the **entire folder** — all 6 files needed to be included in the same drop. If a future deploy seems to have "lost" a feature, check whether every file in the folder actually made it up (`fetch('manifest.json')` etc. from the browser console, checking for a 200 vs. a 404-styled HTML response) before assuming the code itself is broken.
- **Netlify's "credits" (Free plan, 300/month) are unrelated to normal app usage.** They track things like build/deploy activity on Netlify's side, not visits to the finished app. The credit drop seen during this session came from the unusually high number of manual redeploys done while iterating (10+ in a single day), not from real day-to-day use. Signing in / syncing via Google Sheets happens entirely between the browser and Google's servers — Netlify is not involved in that at all, so syncing never touches Netlify credits regardless of how often it's done.

## 8. Currently open / worth knowing about

- Nothing was left mid-fix — every item in Section 4 was verified fixed and (as of the end of this session) confirmed live on the Cloud site.
- **Known limitation, not a bug**: if a budget is ever set on *both* a parent category and one of its own subcategories for the same month, spending on the subcategory gets counted in both budgets' totals (each row is individually correct, but summing all Total rows would double-count that overlap). Documented in `DOCUMENTATION.md` Section 3.5 — pick one level (parent or subcategory) per category tree, not both.
- PC and iPhone copies still don't sync with each other or with Cloud automatically — Export Backup / Import Backup is the manual bridge between them, as documented in `USER_MANUAL.md` Section 11.
- If a friend wants their own fully independent Cloud copy: they deploy the same code to their own Netlify, with their **own** Google Sheet and their own Google Cloud OAuth Client ID — full steps in `DOCUMENTATION.md` Section 9 and `USER_MANUAL.md` Section 13.

## 9. How to use this if a bug shows up later

1. Open the relevant file (`money-manager-cloud/index.html` for the live site, or the PC/iPhone equivalents) and find the function involved — `DOCUMENTATION.md` Section 3 has the exact formula/logic for every calculation, and Section 2 has the exact field list for every data type.
2. Check the Change Log in `DOCUMENTATION.md` — if it's a regression, there's a good chance a past entry explains the intended behavior and why it was built that way.
3. Follow Rule Zero (Section 1 above) for the fix: update the doc first, then the code, in all three copies, then test, then log it.
4. If it's data-shaped (wrong numbers, missing entries), test on the live real data directly if possible — that's what caught most of the real bugs this session, rather than only testing with synthetic data.
