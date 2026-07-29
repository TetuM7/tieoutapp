# TieOut — Engine Architecture

The engine is **two engines, cleanly separated**. That split is the strategy expressed in architecture: everyone has the first; the second is the moat.

```
   PDFs ────►  STATEMENT ENGINE  (per document, parallel)
   (any acct,  raw PDF → one clean, self-checked statement       ← commodity (deterministic, reusable)
    any month)                          │ StatementObject[]
                                        ▼
              LEDGER ENGINE      (per engagement, stateful)
              all statements → one reconciled, import-safe ledger  ← THE WEDGE (net-new)
                                        │ LedgerState
                                        ▼
                       QBO / OFX / CSV / XLSX + reconciliation report
```

Keep them separate — different scope, failure modes, and test strategy. Do not fuse them.

## Statement Engine (per document)
Deterministic-first, LLM only for the long tail. A **tiered router** picks the best lane per document:

```
                    identify the bank (fingerprint)
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
   TIER 1 SPECIALIST   TIER 2 GENERIC      TIER 3 VISION-LLM
   deterministic       deterministic       bounded, schema-
   per-bank profile    parser for          constrained model
   (bank registry:     unknown-but-        (scans, weird
   Chase, BoA, Wells,  structured text-    layouts, long tail)
   Citi, PNC, Truist…) native statements
   ~100% acc, cheap    good acc, cheap     lower acc, costly,
                                           ALWAYS confidence-flagged
```

Pipeline within a lane:
`classify → normalize (render/deskew/coord-map) → text/OCR (docling; vision for scans) → layout → route → extract (summary + rows) → sign-normalize → per-statement tie-out`

- **Not blocked on "we haven't coded that bank"** — unknown banks fall to Tier 2/3 with lower confidence.
- **The per-statement tie-out is the safety net across all tiers**: whatever lane extracted it, the tie-out re-validates the numbers. A bad Tier-3 extraction is *caught and routed to review*, not silently exported. Reconciliation is also the QA layer on extraction — a guarantee vision-LLM-only converters can't make.

Output: one typed **`StatementObject`** (see [`DATA_MODEL.md`](DATA_MODEL.md)). Carries `account_type` so downstream picks the right tie-out equation and transfer semantics.


## Ledger Engine (per engagement)
Input: every `StatementObject` for the engagement + the **seam** (date QuickBooks already covers). Fixed pass sequence over the *combined* transaction set:

```
1. canonicalize   → one stream of CanonicalTxn {account, date, signed_amount, desc, source, confidence}
2. dedup          → composite-key + overlap (same txn in two PDFs)
3. transfer match → equal-and-opposite across accounts, date window, desc signals (account-type-aware)
4. processor match→ Stripe/PayPal payout ↔ bank deposit
5. seam filter    → mark rows already in QBO (past cutover) → exclude from export
6. carry-forward  → per account: ending balance = next opening; flag the break
7. emit LedgerState + exceptions + coverage + export set
```

**The matchers are DETERMINISTIC, not ML/LLM.** Matching money must be rule-based and auditable — a model must never hallucinate a transfer. Each matcher is a confidence-scored function; text similarity (pg_trgm/embeddings) only *feeds* the score, never decides. See [`RECONCILIATION.md`](RECONCILIATION.md).

**LLM is bounded to explanation, never decision** — it can say "this looks like a transfer because…" or map an unknown column header, schema-constrained, cost-capped, logged via a bounded oracle-prompt harness. It never decides a match or invents a number.

## The five engine principles
1. **Deterministic-first, LLM-bounded** — rules decide; models assist and explain.
2. **Everything is a proposal** — matches are records with `status: proposed|confirmed|rejected`; non-destructive; every action reverses.
3. **Provenance + confidence on every atom** — every txn links to `{statement, page, bbox, method}`; every match carries a score and its reasons.
4. **Idempotent + incremental** — the pipeline is a pure function of (statements + confirmations); re-run after an edit recomputes only what changed.
5. **Conservative by default** — a wrong auto-match is worse than a missed one; below threshold → review.

## Tech stack
- **Orchestration:** Temporal — engagement workflow, child workflow per statement (parallel extraction), barrier, then reconciliation phase. Redis-backed.
- **Storage:** MinIO (PDFs, page images, artifacts).
- **DB:** Postgres + Drizzle (+ pg_trgm/pgvector for fuzzy description signals).
- **Text:** docling + unstructured; scans via vision transcription + a cheap OCR path.
- **LLM:** a small vision-capable model for the Tier-3 tail, with a local fallback; zod-schema-constrained, cost-capped.
- **Language:** TypeScript throughout.

## Evaluation (build alongside, not after)
Golden fixtures gate every engine change. For the Ledger Engine especially: synthetic multi-account engagements with *known* transfers, duplicates, processor payouts, and a planted missing month. Regression metrics: transfer precision/recall, **false-transfer rate (must be ≈ 0)**, duplicate catch rate, carry-forward break detection, and **zero false "reconciled."**
