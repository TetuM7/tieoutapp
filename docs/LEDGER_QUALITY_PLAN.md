# Ledger Engine — Quality & Corpus Plan (build/implementation)

> **Build status (2026-07-14).** All buildable-now code artifacts are shipped and
> green (engine 56 tests, api 12 tests, both typechecks clean):
> - **LE-0** — `packages/engine/eval/generate-engagement.ts` (deterministic, seeded;
>   easy/medium/hard tiers incl. a 1-source/2-destination disambiguation stressor).
> - **LE-2** — `packages/engine/eval/run-ledger-corpus.ts` + committed
>   `LEDGER_SCORECARD.md`; **300** engagements, 38k distractor rows, **false
>   cross-account = 0**. `pnpm --filter @tieout/engine eval:ledger`.
> - **LE-1** — `apps/api/src/ledger-eval-harness.ts` + `LEDGER_REAL_SUBSTRATE_SCORECARD.md`;
>   real extracted rows as distractors, injected labels, same scorer as LE-2.
> - **LE-3 (code artifact)** — `packages/engine/eval/ingest-qbo-gl.ts`: GL-diff label
>   deriver with a human-review step, proven by a generate→book→re-derive round-trip.
>   The *operational* parts (recruit partners, DPA, ingest real books) remain.
> - **LE-4 (wiring)** — `apps/api/src/active-learning.ts` + `EngagementService`:
>   every confirm/reject persists `{proposal, humanDecision, features}`; a re-score
>   reports per-matcher acceptance + threshold hints. Compounds once live.
>
> A shared scorer (`packages/engine/eval/score.ts`, exposed as `@tieout/engine/eval`)
> gives LE-0/LE-1/LE-3 one definition of recall/precision/the false-cross gate, so
> every substrate's scorecard is directly comparable.


**The question:** why isn't there a corpus-grade Ledger eval yet, and how do we ship
a quality product without real multi-account engagements + human-verified
reconciliations?

**Short answer:** it's a **data + partnership** problem, not a coding one. The
Statement Engine could be measured against 281 *public* single-account statements
because those exist and an ending balance is objectively checkable. A Ledger
engagement is (a) inherently **private** (one entity's transactions across several of
their own accounts) and (b) has no objective single answer — "correct" is what a
**human bookkeeper decided** (which legs are transfers, which rows are dups, which
processor payouts collapse, which month is missing). That ground truth only exists
inside real, consented bookkeeping engagements — none of which live in a dev
environment. So the sequence is: get the engines correct on public data (done) →
build a quantitative bar with generated + real-substrate evals (bridge) → acquire
real labeled engagements from design partners (gold) → learn continuously from
production. This plan is that sequence.

---

## Why we can ship BEFORE the eval is corpus-grade (the trust model)

The product is **propose-and-confirm, non-destructive, hard-gated** (D-003/D-006):
- Every match is a **proposal a human confirms/rejects** — a missed transfer is caught
  by the bookkeeper; a wrong proposal is rejected; nothing is booked silently.
- The **hard no-silent-wrong-booking gate**: the engine never nets/drops/collapses
  a row without a reversible `Match` record, and export is blocked until review.
- So quality degrades **gracefully**: below-perfect recall costs the bookkeeper a
  little manual work, not a wrong ledger. That is precisely why you ship to design
  partners *first* — their corrections become the corpus.

The eval's job is not a gate on shipping; it is (1) a **regression bar** so quality
only goes up, and (2) the **evidence** that lets us eventually claim "reconciled"
autonomously. Both can start on generated/real-substrate data today.

---

## Four data sources (ranked by fidelity)

| # | Source | Fidelity | Cost | Availability |
|---|---|---|---|---|
| A | **Design-partner completed books** — a real engagement's statements + the bookkeeper's finished QuickBooks GL; labels derived by diffing | ⭐⭐⭐⭐⭐ real + human-verified | partnerships + DPA + ingestion | the gold corpus; requires pilots |
| B | **Realistic generator** — synthetic multi-account engagements with relationships planted by construction (exact ground truth) | ⭐⭐⭐ structure real, txns synthetic | build once | **now**, at scale (100s) |
| C | **Real statements + injected relationships** — real extracted rows as distractors, constructed transfers/dups/payouts | ⭐⭐⭐⭐ real noise, constructed links | small | **now** (`ledger-eval.test.ts` is the seed) |
| D | **Production active-learning** — every confirmed/rejected proposal in the live product becomes a labeled example | ⭐⭐⭐⭐⭐ real + human-verified, self-growing | wiring | after launch |

The strategy uses **B+C now** to develop against a real quantitative bar, then **A**
to validate for production, then **D** to compound forever.

---

## Phased implementation

### LE-0 · Realistic engagement generator (buildable now) — the quantitative bar
Build `packages/engine/eval/generate-engagement.ts` producing labeled engagements:
- **Entity model:** household or small business → 2–5 accounts (checking, savings,
  1–2 cards, a processor) over 3–12 months.
- **Transaction synthesis:** sample from realistic distributions — recurring payroll,
  rent, subscriptions, a vendor tail (real merchant-name list), card spend, interest;
  amounts/cadence calibrated against stats from our 770 real statements.
- **Planted relationships (exact ground truth):** N internal transfers (checking↔savings,
  card paydowns), M re-uploaded duplicates, K processor payouts (Stripe/PayPal/Square
  gross→fee→net→deposit), plus deliberate **carry-forward breaks** and **missing months**.
- **Adversarial noise:** coincidental equal amounts, near-window non-transfers,
  round-number bait — so precision is actually stressed.
- **Output:** `{ engagement: StatementInput[], processorReports, qboBooked, groundTruth }`
  where groundTruth enumerates every true relationship + expected exceptions.
- **Scale:** generate 200–500 engagements across parameterized difficulty.

### LE-1 · Scale the real-substrate eval (extends today's seed)
Generalize `apps/api/src/ledger-eval.test.ts` into a harness that, for many
configurations, draws **real extracted statement rows** as the distractor pool and
injects the LE-0 relationship set. Real transaction density + descriptions +
amounts; constructed links. This is the honest middle ground until source A exists.

### LE-2 · Scorecard + hard gates (mirror `run-corpus`)
`packages/engine/eval/run-ledger-corpus.ts` → a committed `LEDGER_SCORECARD.md`:
- **Per-matcher precision/recall:** transfers, duplicates, processor, carry-forward.
- **HARD GATE — false cross-account match rate = 0** (net/collapse an unrelated pair):
  the Ledger analog of false-reconciled-0.
- **Coverage:** carry-forward break / missing-month detection recall.
- **Grading:** report at HIGH/MID/LOW confidence bands; auto-propose vs review-queue.
- Wire into CI as a regression gate, exactly like the Statement Engine's eval.

### LE-3 · Design-partner gold corpus (the real thing) — the milestone
This is the answer to "real engagements + human-verified reconciliations." It is an
**operational** workstream, not a code file:
1. **Recruit 3–5 catch-up bookkeepers** as design partners.
2. **Consent + DPA + data handling** (SOC-2 path, on-our-infra open-source models per
   D-010 = no third-party egress — this is a *sales* advantage, not just compliance).
3. **Ingest a completed engagement:** their raw statements (PDF) + the finished
   QuickBooks **General Ledger export** (the booked truth).
4. **Derive labels by GL-diff** (`packages/engine/eval/ingest-qbo-gl.ts`):
   - A booked **Transfer** category on two rows netting to zero → a transfer label.
   - Rows in our extraction absent from the GL past the cutover → seam/duplicate labels.
   - Processor deposits booked once vs the payout report → collapse labels.
   - Month-boundary balance continuity in the GL → carry-forward truth.
   - **Human review step:** the partner confirms the derived labels (1–2 hrs/engagement)
     → this is the human-verified ground truth.
5. **Target:** 20–50 real engagements → the first corpus-grade Ledger eval. Run LE-2's
   scorecard against it; the delta vs the generator reveals distribution gaps to fix.

### LE-4 · Production active learning (compounds forever)
Every confirm/reject in the live product is a label. Wire the `EngagementService` to
persist `{proposal, humanDecision, features}` → a growing real corpus that (a)
re-scores the matchers weekly and (b) feeds threshold/feature tuning. This is how the
corpus outgrows the design partners and how "autonomous reconciled" gets earned.

---

## Metrics & release gates

| Metric | Bar to auto-propose | Bar to claim autonomous "reconciled" |
|---|---|---|
| False cross-account match rate | **0** (hard, always) | **0** |
| Transfer recall | ≥ 0.9 @ HIGH conf | ≥ 0.98 on the design-partner corpus |
| Duplicate/seam recall | ≥ 0.9 | ≥ 0.98 |
| Processor recall | ≥ 0.9 | ≥ 0.98 |
| Carry-forward break detection | ≥ 0.95 | ≥ 0.99 |
| Per-engagement export correctness (post-confirm) | n/a (human-gated) | GL matches on ≥ 0.98 |

Ship to design partners at the left column (safe because propose-and-confirm). Claim
autonomous reconciliation only at the right column, proven on source A + D.

---

## Sequencing / what to build first

1. **LE-0 generator** + **LE-2 scorecard/gates** — pure code, buildable now; gives the
   regression bar and a `LEDGER_SCORECARD.md` this week.
2. **LE-1** scale-up of the real-substrate eval — extends today's passing seed.
3. **LE-3 design-partner program** in parallel (recruiting + DPA + `ingest-qbo-gl.ts`)
   — the long pole, because it's people + consent, not code.
4. **LE-4** once the product has live confirmations.

The engine is already correct on constructed + real-substrate data (D-052/D-056, false
cross-match 0). LE-0/LE-2 convert that into a defensible quantitative bar; LE-3 is the
real corpus that only a real product with real users can produce — which is why it's a
launch milestone, not a dev-environment task.
