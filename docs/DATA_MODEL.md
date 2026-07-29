# TieOut — Data Model

The unit of work is the **engagement** (a client's period being caught up), never a single statement.

```
Organization → User(s)
Organization → Client → Engagement → { StatementFiles[] , CanonicalTxns[] , Matches[] , Exceptions[] , Exports[] }
```

## Core atoms

### StatementObject (output of the Statement Engine, per document)
```ts
StatementObject {
  statement_file_id, engagement_id, account_id,
  account_type: "checking"|"savings"|"credit_card"|"loan"|"processor",
  bank_name, account_suffix, period_start, period_end, currency,
  summary: { opening_balance, closing_balance, total_credits, total_debits,
             fees?, interest?, /* card: previous_balance, new_balance, payments, purchases… */ },
  rows: TransactionRow[],
  tieout: { equation, computed_closing, reported_closing, difference, closes: boolean },
  extraction: { tier: "specialist"|"generic"|"llm", profile_id?, engine_version },
  confidence,
}
```

### TransactionRow → CanonicalTxn
Rows are normalized into one canonical stream the Ledger Engine operates on:
```ts
CanonicalTxn {
  id, engagement_id, account_id, account_type,
  date, posting_date?, description, check_number?,
  displayed_amount,               // what was printed, preserved
  signed_amount,                  // effect on balance (see RECONCILIATION.md)
  txn_type?,                      // purchase|payment|fee|interest|cash_advance|credit|deposit|withdrawal
  transfer_eligible, flow?,       // outflow|inflow|paydown|draw  (for the transfer matcher)
  running_balance?,
  source: { statement_id, page, bbox, raw_text },
  confidence: { date, description, amount, overall },
  extraction_method,              // text_layer|ocr|specialist|generic|llm|manual
  match_id?, category?, status,   // status: active|excluded|needs_review
  user_edited, created_at, updated_at,
}
```

### Match (the atom the exporter reads)
```ts
Match {
  id, engagement_id,
  type: "transfer"|"duplicate"|"processor",
  member_txn_ids: [...],
  resolution: "net_to_transfer"|"drop"|"collapse"|"exclude_seam",
  confidence, reasons: [...],
  status: "proposed"|"confirmed"|"rejected",
  created_by, created_at,
}
```
Export never mutates transactions — it reads `Match` records to decide what to emit, how to label transfers, what to drop, and what to exclude past the seam.

### Exception
```ts
Exception {
  id, engagement_id, type, severity,
  message, suspected_page?, suspected_txn_ids?, suggested_fix?,
  review_required: boolean, resolved, resolved_by, resolved_at,
}
```

### LedgerState (output of the Ledger Engine)
```ts
LedgerState {
  engagement_id, transactions: CanonicalTxn[], matches: Match[], exceptions: Exception[],
  coverage: { [account_id]: { months: {[period]: tieout_status} , gaps: [...] } },
  engagement_tieout: { carry_forward_ok: boolean, breaks: [{account_id, period}] },
  export_set: CanonicalTxn[],     // post-dedup, post-seam, transfer-labeled
}
```

## Persisted tables
`organizations`, `users`, `clients`, **`engagements`**, `statement_files`, `statement_pages`, `extraction_jobs`, `statement_accounts`, `transaction_rows`, **`matches`**, **`validation_runs`**, **`extraction_exceptions`**, `export_packages`, plus `llmCalls`, `orgCostBudgets`, `apiKeys`, `webhookEndpoints`, billing.

**New/adapted for TieOut:** `engagements` (batch grouping + the QBO seam config), `statement_accounts` as first-class rows, typed `matches` for transfers/duplicates/processors, typed `validation_runs` / `extraction_exceptions`, per-row bbox in provenance, page-based metering.

## Invariants
- Every `CanonicalTxn` has a resolvable `source` (statement + page + bbox).
- Nothing is deleted; exclusion is a `status` + a reversible `Match`.
- `displayed_amount` is never overwritten by normalization.
- The pipeline is a pure function of (StatementObjects + confirmed Matches + seam) → same input, same LedgerState.
