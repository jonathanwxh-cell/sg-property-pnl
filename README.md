# Singapore Property P&L Calculator

Estimate the **net profit or loss of a Singapore residential property transaction** — after
Buyer's/Additional Buyer's/Seller's Stamp Duty (BSD/ABSD/SSD), mortgage interest, and transaction
costs — and compare it against STI ETF, S&P 500, and 6-month T-bill benchmarks.

It's a **single self-contained HTML file** ([`index.html`](./index.html)): no build step, no
framework, inline CSS + vanilla JS. Open it in a browser and it works. Dark/light aware, responsive.

**Live demo / reference implementation:** https://agent-deployed-applications-alyosha.zocomputer.io/sg-property-pnl
— a deployed, known-good build of this code. Compare against it to confirm the intended UI, behaviour, and numbers.

---

## ⚠️ Before you edit — read the handoff docs

This is a fixed & modernized version of the calculator (originally `AngsumalinX/sg-property-pnl`).
Several **tax/finance correctness bugs were fixed** and must not be reintroduced.

- **[`AGENTS.md`](./AGENTS.md)** — the 30-second brief that AI coding agents load automatically.
- **[`HANDOFF.md`](./HANDOFF.md)** — the full change log, architecture, correctness invariants,
  the external benchmark dependency, deployment, open tasks, and regression checks.

**`index.html` is the single source of truth.** (The original repo kept dated snapshot files; those
remain there for history — this repo is intentionally a single clean file.)

## Quick facts

- **Tax rules are hardcoded** as of the latest revisions (SSD reflects the **4 Jul 2025** regime; BSD
  tops at 6% above $3M; ABSD as of 27 Apr 2023) — verify against IRAS/MAS before relying on them.
- **Benchmarks** are fetched live from an external endpoint (`sg-benchmarks.alyoechosys.dev`) with a
  built-in fallback to fixed long-run estimates if it's unreachable. See HANDOFF.md §3 to point it
  elsewhere or run fully offline.
- No backend required — the report copy/email feature is fully client-side.

## Credit

Original project by [AngsumalinX](https://github.com/AngsumalinX/sg-property-pnl).
