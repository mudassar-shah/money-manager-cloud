# Money Manager — User Manual

A simple guide to using the app. No technical knowledge needed. (If you're looking for the technical/developer documentation instead, see `DOCUMENTATION.md`.)

---

## 1. The Three Versions

| Version | Where it lives | When to use it |
|---|---|---|
| **Cloud** | **<https://mudassar-shah.github.io/money-manager-cloud/>** | Use this if you want your data to sync between your PC and phone, or if you want to access it from any browser. Needs a one-time Google sign-in setup to sync. |
| **PC** | A folder of files on your computer | Use this if you only ever use one computer and don't need syncing. Works fully offline. |
| **iPhone** | A single file you open in Safari | Use this on your phone. Does not sync with the other versions on its own — see Section 11 for moving data between versions. |

All three look and work the same way. The instructions below apply to all of them, with a few Cloud-only notes marked clearly.

---

## 2. Installing as an App on Your Phone

The Cloud version (and the standalone iPhone file) can be "installed" so it behaves like a real app — its own icon, opens full-screen with no browser bar, and keeps working even with no internet connection. This isn't through the App Store or Play Store; it's a built-in browser feature.

**On iPhone (Safari):**
1. Open <https://mudassar-shah.github.io/money-manager-cloud/> in Safari (not Chrome — this only works in Safari on iPhone).
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

Use the arrows next to the year label to move to a previous or future year — as far back or forward as you like.

**There's now a Personal / Business toggle at the top of this tab.** It works exactly like the one on the Transactions tab: you see one ledger at a time, never the two mixed together.

Switch it to **Business** and you get the same yearly view for your business, on the same 1 July – 30 June year:

- Total income, expenses and **Net Profit** for the year (labelled "Profit Margin" instead of "Savings Rate")
- Business spending by category
- Month-by-month chart
- **Annual Profit by Sale / Deal** — every goods deal and IPTV sale for the whole year, with each one's real cost and margin
- Opening and closing balances for your business accounts

There's no budget section in Business mode, because business budgeting doesn't exist in the app.

---

## 9. The Business Tab

If you run a side business (like reselling IPTV), the **Business** tab tracks it completely separately from your personal finances — its own income, expenses, accounts, and monthly view. To add a business transaction, just switch the Personal/Business toggle on the Transactions tab to "Business" before adding it. Nothing you enter there ever mixes into your personal Dashboard, Budgets, or Yearly Report, and vice versa.

---

## 9a. Business Profit — per sale, not just per month

The Business tab always showed your monthly totals. It now also shows **what each individual sale actually earned**. Two new cards at the bottom of the Business tab do this, and they work differently for your two businesses.

When the Personal/Business toggle is set to **Business**, the Add Transaction form shows one extra row: **Ref / Order #**, **Panel**, and **Credits**. Personal entry is unchanged. You never fill all three in — each business uses its own boxes.

### For goods deals — use Ref

Put the same code on the sale **and** every cost belonging to it:

- Sale to customer, Ref `INV-101` → +20,000
- Vendor A, Ref `INV-101` → −6,000
- Vendor B, Ref `INV-101` → −5,000
- Commission, Ref `INV-101` → −1,000
- Courier, Ref `INV-101` → −100

The **Profit by Sale/Deal** card then shows that deal earned **7,900** (39.5%).

Any number of costs works — two vendors, three, commission, courier. And it doesn't matter if you pay a vendor in a different month; the code keeps them together. If the customer never pays, the deal shows a **loss in red** instead of disappearing.

### For IPTV — use Panel and Credits

**When you buy credits**, enter the payment as a business expense and fill in **Panel** (e.g. `Premium` or `Cheap`) and **Credits** (how many you bought). The app works out the cost per credit itself.

**When you make a sale**, fill in the same **Panel** and how many **Credits** it used:

| Customer buys | Cheap | Premium |
|---|---|---|
| 1 month | 1 | 1 |
| 2 months | 2 | 2 |
| 3 months | 3 | 3 |
| 6 months | **5** | 6 |
| 12 months | **10** | 12 |

That's the only number you type — the same hint appears under the Credits box so you don't have to remember it.

The app spends your **oldest credits first**. So if you have 10 old credits at 170 and 50 new ones at 180, a 15-credit sale costs 10×170 + 5×180 = **2,600**. It splits it automatically. Premium and Cheap credits are kept completely separate and never mix.

The **Credit Stock** card shows how many credits you have left in each panel and what they're worth — something you couldn't see before at all.

### Two profit numbers, both correct

**Net Profit** at the top is cash that moved this month. In a month where you buy a lot of credits it looks poor, because that money really did leave your account.

**Profit by Sale/Deal** is what each sale actually earned.

They won't match, and that's fine — they answer different questions. Don't try to reconcile them.

If a sale has no Ref and no Credits, the app can't work out its cost, so it's listed as a note under the card rather than being quietly left out. And if you sell more credits than you've recorded buying, you'll get a warning that the profit is overstated until you enter the missing purchase.

---

## 9b. Lending Money to People

When you lend someone money and they pay you back, **neither side is income or an expense**. Lending isn't spending — that money is still yours, just in someone else's pocket for a while. Getting it back isn't earning.

Before, both sides were counted, so your Income and Expenses were both too high.

### The fix — tick a box, once

1. Go to **Categories**
2. Click **Edit** on your "Loan Given" category
3. Tick **"Don't count as income or expense"** → Save
4. Do the same for "Loan return"

That's it. You never touch it again, and every new loan you record is handled automatically.

**It fixes your old entries too.** As soon as you tick the box, every past month is corrected — nothing to re-enter.

### What changes and what doesn't

| | |
|---|---|
| Income and Expenses | corrected ✅ |
| Savings Rate | corrected ✅ |
| Loans in the spending chart | gone ✅ |
| **Your account balances** | **completely unchanged** — the money really did leave |
| Your loan entries | still there in the list, untouched |

Flagged categories show a small **"not counted"** label in the Categories list, so you can always see which ones are set.

### Knowing who still owes you

If money you've lent hasn't come back yet, the **Accounts** tab shows a line under Total Balance:

> **Lent out, not yet returned: Rs 3,000**

It appears only when something is actually outstanding, and disappears once everyone has paid you back.

### If someone never pays you back

At some point an unpaid loan stops being a loan and becomes a real loss. Only you can decide when. When you do, just **edit that one transaction and change its category** to a normal expense — then it counts properly as money lost.

### This works for anything of the same shape

Not just loans — any money that passes through your account without being earned or spent. Money you're holding for someone else, a deposit you'll get back, cash you're passing on. Give it its own category and tick the box.

---

## 9c. Things You Paid For Before You Started Using This App

You started tracking in July. But you paid a whole year's school fees back in January, and your business was running for months before that too.

You still want each month's fee to show as an expense in **that** month — otherwise your monthly picture is wrong. But that money is long gone, and it can't come out of UBL or Cash now, because the balance you entered for those accounts in July **already** reflects that earlier spending.

### The setup

Make an account called something like **"Previously Paid"**, and log those expenses against it. Each month's fee then appears properly in your expenses, without touching your real accounts.

The only snag was that this account keeps going more and more negative, and that dragged down your Total Balance — even though it isn't a real loss.

### The fix — one click

On the **Accounts** tab, each account card now has an **"Exclude from total"** button.

Click it on your "Previously Paid" account and it stops being counted in Total Balance. Click again ("Include in total") to undo. New accounts can also be set up this way with a tickbox on the Add Account form.

Flagged accounts show a small **"not in total"** label so you can always tell.

### What changes

| | |
|---|---|
| **Total Balance** | no longer dragged down ✅ |
| **Your expenses** | **still counted in full** — that's the whole point |
| Spending chart, budgets, Yearly Report | unchanged |
| The account itself | still listed, still shows its own balance |
| Reconcile | works exactly as before |

### This works for the business too

Same button, same behaviour. Flag a business "Previously Paid" account and **Total Business Balance** stops counting it, while the expenses still show in your business figures.

### Don't confuse this with the loan tickbox

They do opposite jobs:

| | Loan **category** (Section 9b) | Previously-paid **account** (this one) |
|---|---|---|
| Counts as an expense? | **No** | **Yes** |
| Counts in Total Balance? | **Yes** | **No** |

---

## 9d. Moving Money Between Business and Personal

Sometimes you take money out of the business, sometimes you put your own money in, and sometimes you just **borrow** from the business to cover something and pay it back later.

There's a card on the **Accounts** tab called **Transfer Between Business and Personal**.

```
From:  Jazz cash (Business)
To:    UBL (Personal)
Amount: 30,000        Date: today
```

### What it does

Say your business account has 100,000 and your personal UBL has 50,000, and you move 30,000:

| | Before | After |
|---|---|---|
| Business account | 100,000 | **70,000** ⬇ |
| Personal UBL | 50,000 | **80,000** ⬆ |
| Business Income / Expenses / Net Profit | | **no change** |
| Personal Income / Expenses / Savings Rate | | **no change** |

**The rule: the money moves, the balances follow — but nobody earned anything and nobody spent anything.** It's your own money on both sides.

### Borrowing works naturally

Borrow from the business today, and repay it next month with a transfer the other way. Both balances simply return to where they were, and nothing ever pretended to be income or an expense.

### Two separate forms — don't mix them up

| Moving money… | Use |
|---|---|
| Between two **personal** accounts (UBL → Cash) | Transfer form on the **Transactions** tab |
| Between two **business** accounts | Same form, with the toggle on Business |
| **Business ↔ Personal** | This card on the **Accounts** tab |

Each form refuses the other's job and tells you where to go, so you can't record it in the wrong place.

### If you already entered one by hand

Select the two rows on the Transactions tab and click **Mark as Transfer** — it now accepts business-to-personal pairs too. Both stop counting as income and expense, and no balance moves.

### One thing to be aware of

If you take profit out of the business permanently and always record it as a transfer, your personal Income won't include it — so your Savings Rate will look better than it really is. If you'd rather a withdrawal counted as personal income, just record it as an ordinary income transaction instead of using this form. You choose, each time.

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

**"I have too many categories — do I have to scroll the whole list every time?"**
No. There's a small **Type to filter** box just above the Category dropdown on the Transactions tab. Type a few letters and the list shrinks to what matches. Typing `groc` narrows it to "Grocery" and picks it for you, so you can go straight to the amount.

Typing a main category name like `transport` shows Transport *and* everything under it. Typing `fuel` jumps straight to that one subcategory. The small grey text tells you how many matched.

There's the same box next to the **Category filter** further down the page, for finding past entries. It clears itself after you save a transaction, and the **Clear** button resets the filter one.

This works for Business too — it's the same form, just with the Personal/Business toggle switched over.

**"Why did the Dashboard open on last month instead of this month?"**
This was a bug, fixed on 1 August 2026. The app was reading a world clock (UTC) instead of your own device's clock, so in Pakistan it thought the new month hadn't started yet until 5am. It now always uses the date your device shows — Pakistan, UK, US, anywhere — and follows your device automatically if you travel. There's no setting to change; it just works.

The same bug also meant a transaction you entered between midnight and 5am was dated to the *previous* day, which could file it under the wrong month. That's fixed too. If you entered anything in the early hours before this fix, it's worth checking those dates are right.

One thing to know: if you leave the app open overnight on the last day of a month, it keeps showing the old month until you close and reopen it. That's normal — a reload fixes it.

**"How do I stop my business entries from mixing with my personal ones?"**
They never do — the app keeps them fully separate by design, as long as you use the Personal/Business toggle correctly when adding things.
