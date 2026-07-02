# CLAUDE.md

Agent instructions for this repo live in **[AGENTS.md](./AGENTS.md)** — read it first, then the fuller
**[HANDOFF.md](./HANDOFF.md)**, before editing anything.

**TL;DR:** this is a **single self-contained `index.html`** (no build, no framework, vanilla ES5-style
browser JS). Keep it one file, add libraries only via CDN, and **preserve all element `id`s and result
class names** (the JS depends on them). Do **not** regress the Singapore tax/finance invariants:

- SSD **4 Jul 2025** regime (16/12/8/4 over 4 years; regime by purchase date; calendar-anniversary tiers,
  boundary **exclusive** — a sale ON the Nth anniversary is "more than N years", strict `<`, not `<=`)
- BSD three regimes: pre-20 Feb 2018 **3% top**, 20 Feb 2018–14 Feb 2023 **4% top**, ≥15 Feb 2023 **6% >$3M** (no 7% band)
- **Net P&L = interest only** — no principal add-back — so the breakdown table sums exactly to the total
- LTV **75/45/35** by property count (**55/25/15 when tenure > 30 yrs**); duties on **max(price, valuation)**;
  **gross** yield at full occupancy but **net** yield occupancy-adjusted; ABSD **FTA-national → Singapore-citizen**
  and **all regimes incl. 16 Dec 2021**; ABSD **replacement remission** (2nd home) is pay-now/refund-later and
  the **spouse remission** (1st home, ≥1 SC spouse) is a full 0% remission (SC+foreigner qualifies); break-even
  handles valuation-based SSD & shows "S$0 or below" when non-positive; IRR floors at **-100%** on equity wipe-out
- Full change history: **HANDOFF.md §11–§15** (§4 is the canonical invariants)

Verify with the browser-console reconciliation check in AGENTS.md ("Run & verify locally") **before and
after** any change. There is no test suite. Benchmarks come from an external endpoint with an automatic
fixed-estimate fallback (HANDOFF.md §3).
