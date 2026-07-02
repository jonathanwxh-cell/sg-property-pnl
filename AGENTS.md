# AGENTS.md — read this before editing

AI agents and new contributors: **read [`HANDOFF.md`](./HANDOFF.md) fully before making changes.**
It documents the architecture, every recent fix, and the correctness invariants. This file is the
30-second version.

## What this is
A single self-contained HTML calculator (no build step, vanilla JS + inline CSS) that estimates the
net profit/loss of a Singapore residential property transaction (stamp duties, mortgage, costs) and
compares it to STI / S&P 500 / T-bill benchmarks.

## Canonical file
**`2026-06-26-sg-property-pnl-clarity-pass.html`** is the current source of truth. The other dated
`*.html` files are historical snapshots — **do not edit them.** (First good task: copy the canonical
file to `index.html` and archive the rest.)

## Do NOT regress these (details + why in HANDOFF.md §4–§6)
1. **SSD** (`getSsdInfo`): 4 Jul 2025 regime = 16/12/8/4 over **4 years**; regime chosen by purchase
   date; tiers by **calendar anniversary**.
2. **BSD** (`getBsdRate`): residential tops at **6% above $3M** — no 7% band.
3. **Net P&L** (`computePnl`): financing cost is **interest only** — do NOT add `loan`/subtract
   `outstandingLoan` (that double-counts principal). The breakdown table + email report must **sum
   exactly to Net P&L** (cash basis).
4. **LTV** capped 75/45/35 by property count; **duties on higher of price or valuation**; **yields at
   full occupancy**; **ABSD** FTA-national → citizen treatment.
5. All motion gated behind `@media (prefers-reduced-motion: reduce)`; contrast ≥ 4.5:1.

## External dependency
Live benchmarks come from `https://sg-benchmarks.alyoechosys.dev/benchmarks?years=N` (not in this
repo; CORS-open; cached). If unreachable the app **falls back to fixed estimates** automatically.
Change `BENCHMARK_API` (or set it to `''`) to point elsewhere / go fully offline.

## Verify after changes
See HANDOFF.md §9. Quickest check: after `calculate()`, the `#breakdownTable` line items must sum to
the Net P&L in the `tr.grand` row.
