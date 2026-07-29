# TieOut

**Clean, balanced books from a pile of bank and credit-card statements, imported into QuickBooks right the first time.**

[Open the actual app](https://tieoutapp.com)

TieOut is a catch-up bookkeeping product. Drop in every account for a client's messy period and TieOut converts the statements, matches transfers, removes feed-seam duplicates, collapses double-counted processor payouts, proves every month ties out, and exports an import-safe cleanup package.

> Others convert statements. TieOut turns a client's whole backlog into books that balance.

This repository is the public product and technical specification site for TieOut. The production app lives at `https://tieoutapp.com`; the private implementation repository remains separate.

## Product Description

Catch-up bookkeeping means downloading months of statements across a bunch of accounts and retyping every transaction. The typing is the easy part. The mess is everything after:

- money moved between the client's own accounts looks like income and expense;
- the bank feed and uploaded statement history create duplicates;
- Stripe, PayPal, or Square payouts can be counted twice;
- one missing month throws off every balance after it.

TieOut handles that cleanup layer and hands back a clean, source-backed package ready for QuickBooks.

## What TieOut Does

1. Reads statements across every account and period.
2. Matches transfers between the client's own accounts.
3. Removes duplicates, including overlap with already-booked QuickBooks rows.
4. Catches double-counted sales from payment processors.
5. Checks period coverage and carry-forward continuity.
6. Exports a QuickBooks import CSV, Excel workbooks, review CSV, PDF report, source index, and metadata.

## Product Surface

The public site in this repo uses real screenshots from the running TieOut app:

- public converter;
- cleanups list;
- cleanup overview/workflow;
- review queue;
- source proof and source proof modal;
- coverage gate;
- books check;
- export package;
- billing;
- bank support.

## Technical Architecture

TieOut is organized around two cleanly separated engines:

```text
PDFs / markdown
  -> Statement Engine
  -> StatementObject[]
  -> CanonicalTxn[]
  -> Ledger Engine
  -> Engagement { matches, exceptions, exportReady }
  -> gated QuickBooks CSV
```

The Statement Engine extracts one document into a typed, source-backed `StatementObject`. The Ledger Engine reconciles all statements in an engagement and emits proposed matches, blocking exceptions, tie-out status, and export readiness.

Core principles:

- deterministic-first, LLM-bounded;
- every match is a proposal;
- provenance and confidence on every atom;
- idempotent pipeline;
- conservative export gate.

## Technical Docs

- [Product description](docs/PRODUCT.md)
- [Engine spec](docs/ENGINE_SPEC.md)
- [Architecture](docs/ENGINE.md)
- [Data model](docs/DATA_MODEL.md)
- [Reconciliation](docs/RECONCILIATION.md)
- [Ledger quality plan](docs/LEDGER_QUALITY_PLAN.md)
- [Statement scorecard](results/BASELINE_SCORECARD.md)
- [Ledger scorecard](results/LEDGER_SCORECARD.md)
- [Real-substrate ledger scorecard](results/LEDGER_REAL_SUBSTRATE_SCORECARD.md)

## GitHub Pages

This repository is designed to serve its root `index.html` through GitHub Pages. The page links all app actions to:

```text
https://tieoutapp.com
```

Do not use this repository as the production application repo. The implementation stays private.
