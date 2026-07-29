# TieOut — Product Description

## One-liner
TieOut turns a pile of bank and credit-card statements into clean books that import into QuickBooks right the first time — with the transfers, duplicates, and double-counted sales already sorted out.

## Short description
Catch-up bookkeeping means downloading months of statements across a bunch of accounts and retyping every transaction. The typing is the easy part. The mess is everything after: money moved between the client's own accounts looks like income, the bank feed and the uploaded statements create duplicates, and Stripe or PayPal payouts get counted twice. TieOut handles all of that and hands back one clean, balanced file ready to drop into QuickBooks.

## The problem, in plain terms
Today a bookkeeper converts each statement to a spreadsheet, then spends hours stitching them together by hand:
- A $4,000 move from checking to savings shows up as a $4,000 "expense" in one account and $4,000 of "income" in the other — so the books say the client made money they didn't.
- The bank feed already pulled the last 90 days, so importing the older statements duplicates the overlapping weeks.
- A Stripe payout hits the bank *and* shows on the Stripe report, so sales get counted twice.
- One missing month throws off every balance after it — and you don't find out until the end.

Every existing converter stops right before this part and leaves it to the bookkeeper.

## What TieOut does
1. **Reads all the statements at once** — every account, every month, plus Stripe/PayPal/Square, in one batch.
2. **Matches transfers between the client's own accounts** so money moved around is labeled a transfer, not income or an expense.
3. **Removes duplicates** — including the overlap between what's already in QuickBooks and what's being uploaded.
4. **Catches double-counted sales** from payment processors.
5. **Checks that every month connects to the next** and flags the exact spot if a statement is missing or a balance doesn't line up.
6. **Exports a clean file** (QuickBooks, Excel, or CSV) that imports without creating a mess — plus a reconciliation report, every row traceable back to the original PDF.

## Core differentiator (said simply)
Other tools convert statements. TieOut does the cleanup *around* the conversion — the part that actually eats the day. You get books that balance, not just a spreadsheet of rows you still have to untangle.

The conversion is the **ticket to entry**; the cleanup is **why you win**. Convert is a step *inside* the product, never the pitch. (Same way Stripe is "payments," not "an HTTP API for card networks.")

## Who it's for
Bookkeepers, accountants, and small firms doing catch-up or cleanup work — anyone handed months (or years) of statements across several accounts who needs them turned into clean, tax-ready books fast.

- **Primary:** solo catch-up/cleanup bookkeepers.
- **Then:** tax preparers reconstructing prior-year activity; small firms onboarding messy clients.
- **Not the wedge:** lenders/underwriting (Ocrolus territory — a different product); enterprise close (DualEntry/HighRadius territory).

## Product principles
- Never silently export bad numbers.
- Prefer `needs review` over false confidence.
- Every row and every match traces back to the source PDF.
- Nothing is changed silently — automated actions are proposals the user confirms.
- Make repair faster than manual re-entry.
- The bank/card format library compounds: every statement handled makes the next one better.
