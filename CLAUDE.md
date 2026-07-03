# CLAUDE.md

Agent instructions for this repo live in **[AGENTS.md](./AGENTS.md)** — read it first, then the fuller
**[HANDOFF.md](./HANDOFF.md)**, before editing anything.

**TL;DR:** this is a **single self-contained `index.html`** (no build, no framework, vanilla ES5-style
browser JS). Keep it one file, add libraries only via CDN, and **preserve all element `id`s and result
class names** (the JS depends on them). Do **not** regress the Singapore tax/finance invariants:

- SSD **4 Jul 2025** regime (16/12/8/4 over 4 years; regime by purchase date; calendar-anniversary tiers,
  boundary **exclusive** — a sale ON the Nth anniversary is "more than N years", strict `<`, not `<=`);
  historical regimes retained incl. the **2010 progressive** ones — `getSsdInfo(pd, sd, base)` returns an exact
  amount; **no SSD before 20 Feb 2010**
- BSD three regimes: pre-20 Feb 2018 **3% top**, 20 Feb 2018–14 Feb 2023 **4% top**, ≥15 Feb 2023 **6% >$3M** (no 7% band)
- **Net P&L = interest only** — no principal add-back — so the breakdown table sums exactly to the total
- LTV **75/45/35** by property count (**55/25/15 when tenure > 30 yrs**); duties on **max(price, valuation)**;
  **gross** yield at full occupancy but **net** yield occupancy-adjusted; ABSD **FTA-national → Singapore-citizen**
  and **all regimes incl. 16 Dec 2021**; both ABSD married-couple remissions need only **≥1 SC spouse**
  (SC+foreigner qualifies) — **replacement** (2nd home) pay-now/refund-later, **spouse** (1st home) full 0%;
  break-even handles valuation-based SSD & shows "S$0 or below"; IRR floors at **-100%** on wipe-out
- **Robustness:** `num()` clamps [0,1e12] + **interest rates clamped to [0,100]% in `buildAmort`** (else a 1e308 rate overflows `Math.pow` to NaN) + tenure ≤35yr + amortisation loop capped (no `1e308` hang);
  invalid (`badInput`) / negative inputs **block** calc+export (negatives shown, not flipped); **S$0 sale valid**;
  underwater sale shows a cash top-up; **occupancy + loan tenure visibly clamp in the field** (0–100 / max 35);
  **money fields clear on every focus + `formatMoney` caps 10 digits, parses decimals as $-and-cents (Excel paste
  "1500000.00"→1,500,000) + k/m shorthand** (no absurd concatenation); **>1e12 warns**; **duty chips refresh on
  price change + clear on blank**; **ABSD chip shows Section B bridging cash**; **"hold N yr" = exact anniversary**;
  **BSD/ABSD chips refresh on date change too**;
  a11y = **native `<h2>` titles** under one `<h1>` + **touch-sized sliders (26px thumb / 8px track)** + narrow
  `#srSummary` live region
- Full change history: **HANDOFF.md §11–§18** (§4 is the canonical invariants)

Verify with the browser-console reconciliation check in AGENTS.md ("Run & verify locally") **before and
after** any change. There is no test suite. Benchmarks come from an external endpoint with an automatic
fixed-estimate fallback (HANDOFF.md §3).
