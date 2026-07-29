# TieOut — Reconciliation & Matching

The Ledger Engine's four passes. All matchers are **deterministic, confidence-scored, and produce proposals only** (never silent mutation). Text similarity may feed a score; it never decides.

## Unified tie-out equation
Because specialists sign each row by its **effect on the account balance**, one equation works for both account types:

```
opening_balance + Σ(signed_amount) = closing_balance      (tolerance 0.01)
```

- **Bank (asset):** deposits `+`, withdrawals `−`.
- **Credit card (liability):** purchases/fees/interest/cash-advances `+`; payments/credits/returns `−`.

The account-type complexity is isolated in the specialist (see the per-bank specialist profiles); the reconciliation math stays uniform.

## ① Transfer matching (cross-account)
Candidate = two transfer-eligible rows in **different accounts**, same `|amount|`, within a date window (default 0–5 days).

Scoring (deterministic):
- amount exact-match (required) · date proximity · description signals ("transfer", "online banking", account-number fragments) · round-number penalty (avoids coincidental $500s).
- score ≥ HIGH → auto-propose; MID → review queue; LOW → ignore.

**Account-type-aware pairing** (the subtlety): pure "opposite signed_amount" fails for a card paydown (bank `−X`, card `−X` are same sign). Tag each transfer-eligible row with a `flow` and pair compatible flows:

| Pair | Rule |
|---|---|
| asset ↔ asset (checking→savings) | `outflow(X)` ↔ `inflow(X)` |
| asset ↔ liability (checking→card paydown) | `outflow(X)` ↔ `paydown(X)` |

Resolution: both legs reclassified to a **Transfer** (balance-sheet), removed from income/expense. On export, they are labeled, not double-booked.

## ② Duplicate removal (two kinds)
- **Within data:** month-boundary repeats, re-uploaded overlapping PDFs → composite key `(account, date, amount, description)`.
- **The seam:** any extracted row on/after the cutover date that already exists in QuickBooks (fuzzy match on date+amount+description) → flagged **"already in QBO — exclude from export."** Export only the gap. Kills the #1 QBO import nightmare.

## ③ Processor payout matching
Stripe/PayPal/Square payout of `$X` on date D in the processor report ↔ bank deposit of `$X` labeled "STRIPE TRANSFER" on D±(1–2). Match → collapse the relationship so revenue isn't counted once as the payout and again as the deposit.

## ④ Cross-month carry-forward (engagement tie-out)
Order each account's statements by period; assert `ending_balance[m] == opening_balance[m+1]` across the whole engagement. Where the chain breaks, flag the **exact account + month**. This proves nothing is missing across the period — what makes catch-up "stick" (backlogs hide in the balance sheet, not the P&L).

## Statuses & exceptions
Per-statement and per-engagement status: `reconciled`, `reconciled_with_warnings`, `mismatch`, `missing_summary_fields`, `missing_transactions`, `low_confidence`, `unsupported`, `manual_review_required`.

Typed exceptions (each: severity, message, suspected page/rows, suggested_fix, review-required flag): missing/duplicate transaction candidate, amount/date OCR ambiguity, running-balance break, summary-field conflict, unsupported layout, multi-account ambiguity, credit/debit sign uncertainty, unmatched transfer, coverage gap.

## Export gating
Export is gated on unresolved exceptions: *"3 unconfirmed transfers, 1 missing month — review before export."* Export reads the `Match` records to decide what to emit, how to label transfers, what to drop, and what to exclude past the seam. Stable transaction IDs (FITID) so QuickBooks won't re-duplicate.

## What the matchers must NEVER do
- Auto-apply a below-threshold match.
- Use an LLM to decide a match or to produce a number.
- Silently drop or merge a row without a reversible `Match` record.
- Report `reconciled` when the tolerance check fails.
