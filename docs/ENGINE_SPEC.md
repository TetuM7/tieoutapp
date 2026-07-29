# TieOut Engine — Spec Summary

A single reference for the whole engine as it exists in the codebase today. TieOut is
**two cleanly-separated engines** in a TypeScript pnpm monorepo:

- `@tieout/statement-engine` — per-document extraction (`packages/statement-engine`)
- `@tieout/engine` — per-engagement reconciliation + the eval infrastructure (`packages/engine`)
- `@tieout/api` — the application core over both engines (`apps/api`)

This summarizes the *implemented* system and flags where the documented target
architecture (`ENGINE.md`) is not yet in this repo. For deeper design rationale see
[`ENGINE.md`](ENGINE.md), [`DATA_MODEL.md`](DATA_MODEL.md), [`RECONCILIATION.md`](RECONCILIATION.md),
and [`LEDGER_QUALITY_PLAN.md`](LEDGER_QUALITY_PLAN.md).

---

## 0. Shape of the system

```
PDFs/markdown ─► STATEMENT ENGINE (per document, stateless)
 (any bank,       raw doc → 1 self-checked StatementObject
  any month)                    │  StatementObject[]
                                ▼
                fromStatementObject()  ── adapter → CanonicalTxn
                                │  StatementInput[]
                                ▼
                LEDGER ENGINE (per engagement, stateful graph)
                all statements → 1 reconciled, import-safe ledger
                                │  Engagement { matches, exceptions, exportReady }
                                ▼
                       gated export → QuickBooks CSV
```

**Five engine principles** (`ENGINE.md`):

1. **Deterministic-first, LLM-bounded** — rules decide; models only assist/explain.
2. **Everything is a proposal** — matches carry `status: proposed|confirmed|rejected`; non-destructive; every action reverses.
3. **Provenance + confidence on every atom** — every txn links to `{statement, page, bbox, method}`; every match carries a score and its reasons.
4. **Idempotent + incremental** — the pipeline is a pure function of (statements + confirmations); re-run recomputes deterministically.
5. **Conservative by default** — a wrong auto-match is worse than a missed one; below threshold → review.

---

## 1. Statement Engine — `extractStatement(input) → StatementObject`

**Input:** `{ markdown?, filename?, fileBuffer? (PDF) }`. **Job:** turn one bank or
credit-card statement into a typed, source-backed, tie-out-graded object.

### Tiered router (fingerprints the bank, picks a lane per document)

| Tier | Lane | Accuracy / cost |
|---|---|---|
| 1 | **Specialist** — deterministic per-bank profile (Chase, BoA, Wells, Citi, Capital One, US Bank) | ~100%, cheap |
| 2 | **Generic** — deterministic parser for unknown-but-structured text-native statements | good, cheap |
| 3 | **Vision-LLM** — schema-constrained, cost-capped, **always** confidence-flagged | lower acc, long tail |

Unknown banks are **not blocked** — they fall to Tier 2/3 with lower confidence.

### Per-lane pipeline

```
classify → normalize (render/deskew/coord-map) → text/OCR (docling; vision/tesseract for scans)
        → layout → route → extract (summary + rows) → sign-normalize → per-statement tie-out
```

**The per-statement tie-out is the cross-tier safety net.** Whatever lane extracted it,
`computedClosing = opening + Σcredits − Σdebits` is re-validated against the stated
closing (`tieout.closes`). A bad extraction (including a bad Tier-3 one) is *caught and
routed to review*, never silently exported — reconciliation is also the QA layer on
extraction.

### `StatementObject` output (`contract/types.ts`)

- **`account`** `{ institution, accountType, accountNumberLast4, currency, productLine? }` — each value is a `FieldConfidence<T>` `{ value, confidence, status: high|medium|low|abstained }`.
- **`summary`** `{ openingBalance, closingBalance, totalCredits, totalDebits, fees?, interest?, periodStart, periodEnd }` — all `FieldConfidence`.
- **`rows[]`** `StatementTransactionRow` `{ index, date, postingDate?, description, checkNumber?, displayedAmount, signedAmount, runningBalance?, txnType, flow?, source{ statementId, page, bbox?, rawText }, confidence{ date, amount, description, overall }, extractionMethod }`.
- **`tieout`** `{ closes, computedClosing }`, **`status`** (`reconciled | reconciled_with_warnings | mismatch | low_confidence | …`), **`exceptions[]`**, **`audit`** `{ tier, method }`.

**`flow`** is the key handoff to the Ledger transfer matcher:
`outflow | inflow` (deposit-account transfer legs), `paydown | cash_advance` (card),
`purchase | fee | interest` (not transfer-eligible).

### Corpus / eval

281+ real public statements across 6 banks under `extraction/banks/*/__corpus__`;
`run-corpus.ts` → `BASELINE_SCORECARD.md`. **Hard gate: false-reconciled = 0** (never
report `reconciled` to a closing balance that disagrees with ground truth).

---

## 2. The adapter — `fromStatementObject(so, ctx) → StatementInput`

Bridges the two engines. Structurally typed against a local `StatementObjectLike` (so
`@tieout/engine` stays decoupled from the extractor's module system). It:

- maps each `StatementTransactionRow` → a **`CanonicalTxn`**;
- derives the ledger transfer role: prefers the Statement Engine's `flow`, falling back
  to keyword + direction (`TRANSFER_RE`) for untagged deposit-account transfers;
- sets `transferEligible = flow !== undefined` and stamps a globally-unique
  `id = "${statementId}#${row.index}"`.

---

## 3. Ledger Engine — the data model

- **`CanonicalTxn`** — the atom the matchers operate on:
  `{ id, engagementId, accountId, accountType, date, description, displayedAmount, signedAmount, txnType?, transferEligible, flow?, runningBalance?, source?, matchId?, status: active|excluded|needs_review }`.
  `signedAmount` is the effect on the account balance (deposits +, withdrawals −; card charges +, payments −).
- **`Match`** — a proposed relationship (never a silent mutation):
  `{ id, engagementId, type, memberTxnIds[], resolution, confidence 0..1, reasons[], status }`.
  - `type`: `transfer | duplicate | processor`
  - `resolution`: `net_to_transfer | drop | collapse | exclude_seam`
  - `status`: `proposed | confirmed | rejected`
- **`Engagement`** — the graph the passes transform:
  `{ id, accounts[], statements[] (StatementPeriod), transactions[] (flat pool), matches[], tieOut, exceptions[], exportReady }`.
- **`LedgerException`** — typed, each `reviewRequired` blocks export (D-003). Codes:
  `STATEMENT_NOT_RECONCILED, CARRY_FORWARD_BREAK, COVERAGE_GAP, OVERLAPPING_PERIOD, UNMATCHED_TRANSFER, DUPLICATE_CANDIDATE, MULTI_ACCOUNT_AMBIGUITY`.

---

## 4. Ledger Engine — `reconcileEngagement(id, statements, opts) → Engagement`

Fixed, non-destructive pass order. Every pass is a pure immutable transform that returns
a new engagement and only **adds** `Match`/exception records; matchers skip
already-matched rows, so the order is safe to replay (idempotent).

```
buildEngagement → transfer → carry-forward → dedup → [seam?] → [processor?] → prune
```

Every matcher is **DETERMINISTIC — arithmetic, not ML/LLM.** Text similarity only
*feeds* a score; it never decides a match.

| Pass | Module | Algorithm | Emits |
|---|---|---|---|
| **L0** build | `engagement.ts` | Assemble N statements → graph; enforce unique statement/txn ids; order periods; flag non-reconciled statements | `Engagement` |
| **L1** transfer | `transfer-match.ts` + `transfer-pass.ts` | Pair transfer-eligible legs in **different** accounts: equal amount (ε 0.005) + date window (≤ 5 d) + flow-compatible (source `outflow`/`draw` ↔ dest `inflow`/`paydown`). Score = 0.5 base + proximity (≤ 0.25) + transfer keyword (+0.2) + shared account-number fragment (+0.15) − round-number-no-corroboration penalty (−0.1). **Greedy** by score; each leg used once. Thresholds: high 0.7 (auto-propose), review 0.4 (below → ignored). A transfer-eligible leg with no partner → `UNMATCHED_TRANSFER` | `transfer` / `net_to_transfer` |
| **L2** carry-forward | `carry-forward-pass.ts` | Per account, ordered periods: `closing[m] == opening[m+1]` (tol 0.01) else `CARRY_FORWARD_BREAK`; coverage `start[m+1] ≈ end[m] + 1 d` (else `COVERAGE_GAP`); earlier start than prior end → `OVERLAPPING_PERIOD`. Sets `tieOut.carryForwardComplete` | exceptions only |
| **L3** dedup | `dedup-pass.ts` | Group by `(accountId, date, |signedAmount|, normalized desc)`; groups ≥ 2 → propose dropping the extras (keep first) | `duplicate` / `drop` |
| **L3b** seam *(opt, needs QBO input)* | `dedup-pass.ts` | Rows on/after `cutoverDate` that match an already-booked QuickBooks `bookedEntry` (amount + date window + desc similarity, one-to-one) → exclude from export | `duplicate` / `exclude_seam` |
| **L4** processor *(opt, needs report)* | `processor-pass.ts` | An unmatched incoming deposit matching a payout report (net amount + date window + processor named in the description) → collapse so revenue isn't double-counted | `processor` / `collapse` |
| prune | `reconcile.ts` | Drop stale `UNMATCHED_TRANSFER` flags for legs a later pass (dedup/processor) claimed | — |

**Alternate entry:** `reconcileFromStatements(id, [{statement, ctx}], opts)` adapts raw
`StatementObject`s (runs `fromStatementObject`) then reconciles.

### The export gate — `computeExportReady`

Ready **iff** no exception still requires review **and** no match is still merely
`proposed` (every proposal must be human-confirmed or rejected first). Shared by every
pass so the verdict stays consistent as records accumulate.

---

## 5. Export — `buildExport(engagement)` → `toQuickBooksCsv(rows)`

Refuses to emit while anything is unresolved — returns the blocking list + a
human-readable reason (`"Review before export: 3 unconfirmed matches, 1 coverage gap."`).
When the gate clears, it applies each **confirmed** match's resolution:

| Resolution | Effect on export |
|---|---|
| `net_to_transfer` | both legs labeled **Transfer** (not double-booked income/expense) |
| `drop` | keep the first duplicate, drop the rest |
| `exclude_seam` | omit (already in QuickBooks) |
| `collapse` | one **Processor Settlement** revenue row |

A `rejected` match is ignored (its legs export normally). Every row carries a stable
**FITID** so QuickBooks won't re-duplicate on re-import. CSV columns:
`Date, Description, Amount, Category, FITID`. Nothing here mutates the engagement.

---

## 6. Application core — `@tieout/api`

- **`EngagementService`** (over a pluggable `EngagementStore`):
  `create → addStatement → reconcile → reviewQueue → confirm/reject → export/exportCsv`.
  `reviewQueue(id)` = pending proposals + blocking exceptions + tie-out + `exportReady` —
  what a reviewer sees. Adding a statement invalidates the last reconcile (stale until
  re-run).
- **`store.ts`** — in-memory `Map` implementing `EngagementStore` (the persistence seam;
  a real Postgres/Drizzle store implements the same interface). Also holds the LE-4
  decision log.
- **`http.ts`** — thin, dependency-free JSON router over Node `http`; one route per
  service call; service errors → 400; `GET …/export.csv` → `text/csv` or **409** when gated.
- **`upload.ts`** — wires `extractStatement` in front of `addStatement`, so a raw
  document flows the full stack in one call.

---

## 7. Evaluation infrastructure — `@tieout/engine/eval`

A shared, relationship-agnostic scorer (`score.ts`) gives every substrate **one
definition** of recall / precision and the **hard gate**, so all scorecards are directly
comparable. See [`LEDGER_QUALITY_PLAN.md`](LEDGER_QUALITY_PLAN.md).

| Phase | Artifact | What it does |
|---|---|---|
| **LE-0** | `generate-engagement.ts` | Deterministic seeded synthetic engagements; every relationship planted by construction — transfers, duplicates, processor payouts, carry-forward breaks, missing months, plus adversarial noise (out-of-window / flow-incompatible / coincidental-equal-amount / 1-source-2-destination disambiguation) |
| **LE-2** | `run-ledger-corpus.ts` → `LEDGER_SCORECARD.md` | Reconciles a **300-engagement** corpus (38k distractor rows) and scores it: 100% recall/precision, **false cross-account = 0** |
| **LE-1** | `apps/api/ledger-eval-harness.ts` → `LEDGER_REAL_SUBSTRATE_SCORECARD.md` | Real extracted statement rows as the distractor pool + injected labels; same scorer, false cross-account = 0 |
| **LE-3** | `ingest-qbo-gl.ts` | Derives labels by diffing a QuickBooks General Ledger against our extraction (Transfer pairs, seam/duplicate absences, processor collapses, carry-forward continuity) + a human-review step; proven by a generate → book → re-derive round-trip |
| **LE-4** | `apps/api/active-learning.ts` + `EngagementService` | Every confirm/reject persists `{ engagementId, matchId, decision, features }`; `matcherAcceptance()` re-scores per matcher (acceptance rate + threshold hints). Thresholds stay human-owned (D-003) |

**The two hard gates (CI regression bars):** Statement Engine **false-reconciled = 0**;
Ledger Engine **false cross-account match = 0**.

---

## 8. Status & boundaries

- **Implemented & green:** both engines, the adapter, the app core (in-memory store),
  full eval LE-0 → LE-4. Engine **56** tests, api **12** tests, both typechecks clean.
- **Documented target, not yet in this repo** (`ENGINE.md`): Temporal orchestration,
  Postgres/Drizzle + pgvector, MinIO artifact storage, the production LLM/OCR wiring.
  The current store is in-memory and extraction runs on markdown/PDF fixtures.
- **Operational, not code:** LE-3's design-partner gold corpus (recruit partners, DPA,
  ingest real completed books) and LE-4's compounding (needs live production
  confirmations) are launch/partnership milestones — the code is ready for real data to
  swap straight in.

---

## 9. Module map

```
packages/statement-engine/           @tieout/statement-engine
  src/orchestrator/extract-statement  → extractStatement()
  src/contract/                       StatementObject types
  src/extraction/banks/*/__corpus__   281+ real fixtures + manifests
  src/eval/run-corpus + BASELINE_SCORECARD.md

packages/engine/                     @tieout/engine  (Ledger Engine)
  src/types.ts                        CanonicalTxn, Match, exceptions
  src/ledger/engagement.ts            L0 graph + export gate
  src/ledger/transfer-match|pass.ts   L1 transfer matcher
  src/ledger/carry-forward-pass.ts    L2 continuity/coverage
  src/ledger/dedup-pass.ts            L3 dedup + seam
  src/ledger/processor-pass.ts        L4 processor collapse
  src/ledger/reconcile.ts             orchestrator (single entry point)
  src/ledger/export.ts                gated QuickBooks export
  src/ledger/from-statement.ts        StatementObject → StatementInput adapter
  eval/                              @tieout/engine/eval  (LE-0..LE-4 + shared scorer)
    score.ts · generate-engagement.ts · run-ledger-corpus.ts · ingest-qbo-gl.ts
    LEDGER_SCORECARD.md

apps/api/                            @tieout/api  (application core)
  src/service.ts                      EngagementService
  src/store.ts                        EngagementStore (in-memory) + decision log
  src/http.ts                         JSON API over node:http
  src/upload.ts                       extract → addStatement
  src/active-learning.ts              LE-4 feature capture + re-score
  src/ledger-eval-harness.ts          LE-1 real-substrate harness
```
