# CLAUDE.md

Agent instructions for this repo live in **[AGENTS.md](./AGENTS.md)** — read it first, then the fuller
**[HANDOFF.md](./HANDOFF.md)**, before editing anything.

**TL;DR:** this is a **single self-contained `index.html`** (no build, no framework, vanilla ES5-style
browser JS). Keep it one file, add libraries only via CDN, and **preserve all element `id`s and result
class names** (the JS depends on them). Do **not** regress the Singapore tax/finance invariants:

- SSD **4 Jul 2025** regime (16/12/8/4 over 4 years; regime by purchase date; calendar-anniversary tiers)
- BSD **6% top above $3M** (no 7% band)
- **Net P&L = interest only** — no principal add-back — so the breakdown table sums exactly to the total
- LTV **75/45/35** by property count; duties on **max(price, valuation)**; yields at **full occupancy**;
  ABSD **FTA-national → Singapore-citizen** treatment

Verify with the browser-console reconciliation check in AGENTS.md ("Run & verify locally") **before and
after** any change. There is no test suite. Benchmarks come from an external endpoint with an automatic
fixed-estimate fallback (HANDOFF.md §3).
