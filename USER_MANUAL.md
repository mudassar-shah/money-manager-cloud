# Money Manager — User Manual

A simple guide to using the app. No technical knowledge needed. (If you're looking for the technical/developer documentation instead, see `DOCUMENTATION.md`.)

---

## 1. The Three Versions

| Version | Where it lives | When to use it |
|---|---|---|
| **Cloud** | A website (Netlify link) | Use this if you want your data to sync between your PC and phone, or if you want to access it from any browser. Needs a one-time Google sign-in setup to sync. |
| **PC** | A folder of files on your computer | Use this if you only ever use one computer and don't need syncing. Works fully offline. |
| **iPhone** | A single file you open in Safari | Use this on your phone. Does not sync with the other versions on its own — see Section 11 for moving data between versions. |

All three look and work the same way. The instructions below apply to all of them, with a few Cloud-only notes marked clearly.

---

## 2. Installing as an App on Your Phone

The Cloud version (and the standalone iPhone file) can be "installed" so it behaves like a real app — its own icon, opens full-screen with no browser bar, and keeps working even with no internet connection. This isn't through the App Store or Play Store; it's a built-in browser feature.

**On iPhone (Safari):**
1. Open the site in Safari (not Chrome — this only works in Safari on iPhone).
2. Tap the **Share** button (the square with an arrow pointing up).
3. Scroll down and tap **Add to Home Screen**.
4. Tap **Add**.

**On Android (Chrome):**
1. Open the site in Chrome.
2. Tap the **⋮** menu in the top right.
3. Tap **Install app** (or **Add to Home screen**).
4. Confirm.

Either way, you'll now have a "Money Manager" icon on your home screen. Tapping it opens the app full-screen, exactly like any other app on your phone — the layout is also designed to fit a phone screen properly (a menu button in the top corner opens the navigation instead of it taking up the whole screen).

---

## 3. Getting Started

### Add your accounts first
Before entering any transactions, go to the **Accounts** tab and add each account you use — for example "Cash," "UBL," or any bank account. For each one, enter:
- **Name** — whatever you want to call it.
- **Ledger** — Personal or Business.
- **Type** — Bank, Cash, Savings, or Credit Card.
- **Opening Balance** — how much was in it on the date you're starting to track it. If you're not sure, you can start at 0 and fix it later (see Section 9).
- **As Of** — the date that opening balance is accurate for.

### Set up your categories
Go to the **Categories** tab. Some common categories (Food, Transport, Rent, etc.) are already there. You can:
- Add a new category.
- Add a **subcategory** under an existing one (e.g. "Fuel" under "Transport") by picking a Parent Category when adding.
- Use **Add Multiple Subcategories** to quickly add several subcategories under one category at once, one per line.
- Use the **Personal / Business** toggle at the top to switch which set of categories you're managing.

---

## 4. Adding a Transaction

Go to the **Transactions** tab.

1. At the top, make sure the **Personal / Business** toggle is set to the right ledger — this decides which set of accounts and categories you'll see, and which ledger the new transaction belongs to.
2. Fill in: Type (Income/Expense), Date, Account, Category, Amount, and an optional Description.
3. Click **Add Transaction**.

The toggle stays where you leave it, so if you're entering several business transactions in a row, you don't need to reset it each time.

---

## 4a. Moving Money Between Your Own Accounts (Transfers)

When you take cash out of your bank, or move money from one account to another, that is **not** income and **not** an expense — it's the same money in a different place. Use the **Transfer Between Accounts** form on the Transactions tab for this:

1. **From Account** — where the money is leaving (e.g. UBL).
2. **To Account** — where it's going (e.g. Cash).
3. Enter the **Amount** and the **Date**, and a description if you like.
4. Click **Add Transfer**.

Both account balances update straight away — UBL goes down, Cash goes up — but the transfer is **not** counted as income or expense anywhere. Your Dashboard's Income, Expenses and Savings Rate stay honest, it doesn't appear as spending in your Spending by Category chart, and it doesn't eat into any budget.

**Why this matters:** if you record a withdrawal the old way — an expense from the bank and an income into Cash — the app has no way to know they're the same money, so your Income *and* your Expenses both get inflated by that amount. Your balances would still be right, but the Dashboard would tell you you earned more than you did.

Transfers show in your transaction list with a small **⇄ Transfer** tag, so you can always see them.

### Fixing withdrawals you already entered the old way

If you've been entering these by hand as two separate entries, you don't need to delete and redo them:

1. On the Transactions tab, tick the checkbox on **both** entries — the one leaving the first account, and the matching one arriving in the second.
2. Click **Mark as Transfer** in the bar that appears.

That's it. Those two entries stop counting as income and expense — in that month and in your Yearly Report — while both account balances stay exactly as they are. It works for any month, however long ago. If you ever mark the wrong pair, select the same two again and the button offers to undo it.

The app will refuse if the two don't make sense as a transfer — if they're not one income and one expense, if the amounts don't match, or if they're both on the same account — and it will tell you which.

**A few things to know:**
- Deleting one half of a transfer deletes both. This is on purpose: leaving one half behind would make one of your account balances wrong.
- You can't edit one half on its own for the same reason. Delete it and re-enter it with the Transfer form.
- Transfers work within Personal, and within Business, but not between the two — those ledgers are kept completely separate by design.

---

## 5. Fixing Mistakes

### Editing one transaction
Click **Edit** next to any transaction in the list. Its details load into the form at the top — change anything and click **Add Transaction** again to save it.

### Deleting transactions
Tick the checkbox next to one or more transactions, then click **Delete Selected** in the bar that appears.

### Moved money to the wrong account by mistake
If you entered a batch of transactions under the wrong account (for example, you meant to use "UBL" but accidentally used "Cash" for several entries):

1. Tick the checkboxes for all the transactions that need fixing.
2. In the bar that appears, pick the correct account from the dropdown next to **Move to Account**.
3. Click **Move to Account**.

This only changes which account those transactions are linked to — the date, amount, category, and everything else about them stays exactly the same. Both accounts' balances update automatically to reflect the move.

### Deleting or editing categories
Same idea — tick the checkboxes on the Categories tab and use the bulk actions bar. A category can't be deleted if it's still used by a transaction or budget, or if it has subcategories under it — you'll get a warning telling you why.

---

## 6. Budgets

Go to the **Budgets** tab.

1. Pick the month using the arrows at the top.
2. Type an amount into any category's box to set (or change) its budget for that month.
3. The bar under each category shows how much you've spent against that budget, and turns amber/red as you get close to or go over it.
4. The **Total** row at the bottom adds up all your budgeted categories together.

If you set a budget on a main category like "Food," spending logged under any of its subcategories (like "Groceries") automatically counts toward that budget too — you don't need to budget each subcategory separately unless you want more detailed tracking.

---

## 7. Reconciling — When Your Bank Balance Doesn't Match

If the balance shown in the app doesn't match what your bank or wallet actually shows, don't edit your old entries. Instead:

1. Go to the **Accounts** tab.
2. Click **Reconcile** next to the account.
3. Enter today's date and the **real** balance from your bank/wallet right now.
4. Click **Save Reconciliation**.

The app works out the difference and adds a single adjustment entry to close the gap — you don't need to calculate anything yourself. For example, if the app shows Rs 500 but your bank shows Rs 1,000, it adds one Rs 500 "income" entry labeled "Balance Adjustment," and your balance becomes exactly Rs 1,000. If your bank showed *less* than the app, it would add an expense adjustment instead, to bring the balance down.

None of your existing entries are changed or deleted — this just adds one new entry to correct the total.

**If the account has no transactions on it yet at all**, it's simpler to just delete the account and re-add it with the correct opening balance, instead of reconciling.

---

## 8. Yearly Report

Go to the **Yearly Report** tab. This shows a full financial year at a time, running **1 July to 30 June** (Pakistan's fiscal year), with:
- Total income, expenses, and savings for the year.
- Spending broken down by category.
- A month-by-month chart.
- How each budget performed across the whole year.
- Each account's opening and closing balance for the year.

Use the arrows next to the year label to move to a previous or future year. This tab only ever shows **personal** data — your business numbers live separately in the Business tab.

---

## 9. The Business Tab

If you run a side business (like reselling IPTV), the **Business** tab tracks it completely separately from your personal finances — its own income, expenses, accounts, and monthly view. To add a business transaction, just switch the Personal/Business toggle on the Transactions tab to "Business" before adding it. Nothing you enter there ever mixes into your personal Dashboard, Budgets, or Yearly Report, and vice versa.

---

## 10. Backup and Restore

At the bottom of the sidebar:
- **Export Backup** — downloads a copy of everything (all your accounts, categories, transactions, budgets) as a file. Do this regularly, especially before making any big changes.
- **Import Backup** — loads a previously exported backup file. This replaces whatever is currently in the app, so only do this if you're sure.

---

## 11. Moving Data Between Devices

- **PC and iPhone versions do not sync with each other automatically.** To move data from one to the other, use Export Backup on one and Import Backup on the other.
- **The Cloud version can sync automatically** through Google Sheets, once you've connected it (see Section 12) — so if you use the Cloud version on both your PC and phone's browser, they'll stay in sync without manual exporting.

---

## 12. Cloud Sync (Cloud version only)

This lets your data follow you between devices automatically, using your own private Google Sheet as storage.

1. Go to the **Cloud Sync** tab.
2. Enter your **Google Client ID** and **Google Sheet ID** (a one-time setup — ask whoever set this up for you if you're not sure what these are).
3. Click **Sign in with Google** and use your own Google account.
4. Click **Sync Now** any time to push/pull the latest data, or just let it sync automatically in the background.

Only the Google account you signed in with can access your data — nobody else, even if they have the same website link.

---

## 13. Letting Someone Else Use This

- **To let a friend use the PC version:** just give them the folder of files. They open it in any browser — no installation, no account. Their data is entirely separate from yours from the start.
- **To let a friend use their own Cloud version:** they'd need their own Google Sheet and their own Google sign-in setup (their own Client ID), entered into their own copy of the site. This keeps their data completely private and separate from yours, even though the app looks identical.

---

## Quick Answers to Common Questions

**"How do I get this as an app on my phone instead of using it in the browser?"**
Add to Home Screen on iPhone Safari, or Install app on Android Chrome (Section 2). It's not in the App Store, but it opens and feels like a real app once installed.

**"I entered transactions under the wrong account — how do I fix it?"**
Select them with the checkboxes and use "Move to Account" (Section 5).

**"My real bank balance doesn't match what's shown here."**
Use Reconcile, don't edit old entries (Section 7).

**"I picked a main category like 'Transport' in a filter and nothing showed up."**
This is now fixed — picking a main category shows all entries under its subcategories too, not just entries logged directly against the main category itself.

**"I want a budget on 'Food' to also count spending on 'Groceries' (a subcategory of Food)."**
It already does — no extra setup needed (Section 6).

**"I took cash out of my bank and now my Income looks higher than what I actually earned."**
That's what the Transfer form is for (Section 4a). Use **Mark as Transfer** on the two entries you already made to fix it, including for past months.

**"How do I stop my business entries from mixing with my personal ones?"**
They never do — the app keeps them fully separate by design, as long as you use the Personal/Business toggle correctly when adding things.
