# Handoff & Change Log — Singapore Property P&L Calculator

> Purpose: give a new contributor (human or AI agent) with **no prior context** everything
> needed to safely continue this project. Read this top to bottom before editing.

---

## 0. TL;DR

A single, self-contained HTML calculator that estimates the **net profit/loss of a Singapore
residential property transaction** after stamp duties, mortgage, and costs, and compares it to
STI / S&P 500 / T-bill benchmarks. No build step, no framework — one HTML file with inline CSS + vanilla JS.

A recent working session (1) fixed several **correctness bugs in the tax/P&L logic**, (2) replaced a
client-side Twelve Data API-key feature with a **hosted benchmark endpoint**, (3) modernized the
**UI**, and (4) fixed the **save/email** feature. This document records all of it. Sections 3–6 contain
**invariants you must not regress.**

---

## 1. Canonical file (READ FIRST — easy to get wrong)

This repo stores **dated snapshots** (`YYYY-MM-DD-sg-property-pnl-*.html`), one per past change.
**The current, canonical file is:**

```
2026-06-26-sg-property-pnl-clarity-pass.html
```

All the fixes below live in that file. The other dated `.html` files are **historical** — do not edit them.

**Recommended first task:** collapse this to a single `index.html` (copy the canonical file to
`index.html`, move the dated ones into an `/archive` folder) so there is one obvious source of truth.
The `README.md` is currently empty and should describe the project + point at the canonical file.

Everything is referenced below **by function / CSS-section name**, not line number, because line
numbers drift — `grep` for the name.

---

## 2. What the app is (architecture)

- **One file**, no build/bundler. Run it by serving the folder (`python3 -m http.server 8000`, then
  open `index.html`) or just opening the file directly. Works **offline** (benchmarks fall back to fixed
  estimates). **There is no test suite** — verify manually via §9.
- **Vanilla JS**, no framework. Core functions: `getBsdRate`, `getAbsdRate`, `getSsdInfo`,
  `buildAmort`, `computePnl` (pure P&L math), `calculate` (reads the form → renders results),
  `buildBenchmark` / `buildLayman`, `fetchLiveBenchmarks`, `buildReportText` / `copyReport` / `emailReport`.
- **External libs via CDN:** Chart.js (benchmark bar chart), Tabler icons, Google Fonts (Inter + JetBrains Mono).
- **`computePnl(inputs)` is a pure function** shared by both the base-case render and the scenario
  panel — keep it pure so the two never drift.

---

## 3. External dependency ⚠️ (the one thing that isn't self-contained)

The benchmark section fetches live returns from a **hosted endpoint that is NOT part of this repo**:

```
GET https://sg-benchmarks.alyoechosys.dev/benchmarks?years=<1..10>
→ { "sti": {cagr, years, asof} | null,
    "spy": {cagr, years, asof} | null,
    "tbill": {rate, asof, tenor} | null }
```

- Defined in the JS as `var BENCHMARK_API = 'https://sg-benchmarks.alyoechosys.dev/benchmarks';`
- STI ETF (ES3) from SGX in SGD; S&P 500 (SPY) FX-adjusted to SGD; 6-month T-bill from MAS auctions.
- CORS is open (`*`), responses cached ~6h.
- **Graceful fallback:** if the endpoint is unreachable, `fetchLiveBenchmarks` returns `null` and the
  UI shows fixed long-run estimates (STI ~6.5%, S&P ~10%, T-bill ~1.5%). The app never breaks if it's down.

**This endpoint is maintained outside this repo/owner.** If you want the app fully independent:
either (a) stand up your own equivalent endpoint and change `BENCHMARK_API`, or (b) set
`BENCHMARK_API = ''` and have `fetchLiveBenchmarks` short-circuit to `null` so it always uses the
fixed estimates. A config flag for this is a good small task (see §8).

---

## 4. Tax / finance correctness — INVARIANTS, do not regress

These were verified against IRAS/MAS/MOF (sources in §10). Bugs here are silent and expensive.

### 4.1 Seller's Stamp Duty — `getSsdInfo(pd, sd)`
- **Regime is chosen by PURCHASE date**, and tiers are decided by **calendar anniversary** (dispose on
  or before the Nth anniversary = "held ≤ N years"), NOT a 365.25-day count (avoids exact-anniversary /
  leap-year off-by-one).
- **Purchases on/after 4 Jul 2025:** 4-year holding — **≤1yr 16%, ≤2yr 12%, ≤3yr 8%, ≤4yr 4%, >4yr 0%.**
- Purchases 11 Mar 2017 – 3 Jul 2025: 3-year — 12 / 8 / 4 / then 0.
- Earlier regimes retained. SSD is charged on the **higher of sale price or market valuation** (see 4.5).

### 4.2 Buyer's Stamp Duty — `getBsdRate(price, date)`
- Residential top rate is **6% on the portion above $3M** — there is **no 7% band**. Bands (post 15 Feb 2023):
  1% / 2% / 3% / 4% / 5% (1.5–3M) / **6% (>3M)**.

### 4.3 Additional Buyer's Stamp Duty — `getAbsdRate(bt, pc, date, remission, fta)`
- Current (27 Apr 2023) rates: SC 0/20/30, PR 5/30/35, Foreigner 60 (flat), Entity 65 (flat).
- **FTA nationals** (US nationals; nationals/PRs of Iceland, Liechtenstein, Norway, Switzerland) get
  **Singapore-Citizen treatment** — the `fta` flag maps a Foreigner to SC rates.
- **Remission** (rate → 0) only for a **married couple with ≥1 SC/PR buying their 2nd property** while
  selling the first within 6 months. Gated on effective buyer type.

### 4.4 Net P&L — `computePnl` (the flagship bug that was fixed)
- `netPnl = exitProceeds − totalEntryCost − totalInterest − totalPropTax − totalMaint + totalRentalNet`.
- **Do NOT add back `loan` / subtract `outstandingLoan` in netPnl.** A prior version did
  (`- outstandingLoan + loan`), which double-counted principal repayment and overstated profit by the
  amount amortized (e.g. showed +S$279k when the true figure was +S$112k). Financing cost = **interest only**;
  principal is a return of capital recovered at sale.
- **`breakEven` uses the same basis** (no loan terms) — keep them consistent.
- **Breakdown table & email report MUST reconcile:** the visible line items sum **exactly** to Net P&L.
  They are presented on a **cash basis**: Down payment (equity) + duties/fees, then interest +
  principal repaid + holding costs, then sale proceeds − selling costs − loan payoff. If you change one
  side, keep the sum equal to `netPnl` (regression check in §9).

### 4.5 Duties on higher-of-price-or-valuation — `computePnl`
- BSD & ABSD are computed on `max(purchasePrice, purchaseValuation)`; SSD on `max(salePrice, saleValuation)`.
- The optional valuation inputs are `#purchaseValuation` / `#saleValuation` (blank ⇒ use the price).

### 4.6 LTV cap by property count — `ltvCap()` / `updateLtvCap()`
- MAS limits: **75% (1st loan), 45% (2nd), 35% (3rd+)**; the UI caps the slider/input by the buyer's
  property-count selection and clamps the value. (Tenure >30yr or age >65 lowers these further to
  55/25/15 — noted in the hint, not modelled, since age isn't collected.)

### 4.7 Rental yields — `computePnl`
- **Gross/net yields use full contractual annual rent (100% occupancy)** — the market convention.
  Occupancy affects the P&L cash flow only, not the headline yield. Occupancy is clamped to ≤100%.

> ⚠️ SG stamp-duty rules change by policy. Re-verify §4 against IRAS/MAS (§10) periodically; the rates
> and dates are hardcoded intentionally (there is no real-time rates API).

---

## 5. UI / design system

- Design tokens are CSS custom properties in `:root` (+ a `prefers-color-scheme: dark` block).
  **The variable *names* are legacy (`--color-text-*`, `--color-background-*`, `--border-radius-*`) but
  the *values* are modern** — if you add components, reuse these vars so light/dark stays consistent.
  Modern additions: `--font-mono`, `--accent`, `--accent-soft`, `--accent-ring`, `--shadow*`.
- Fonts: **Inter** (UI) + **JetBrains Mono** (all figures use `font-variant-numeric: tabular-nums`).
- Net P&L is a **full-width hero** (`#metricGrid .metric:first-child`); other metrics flow below.
- Motion: results **rise-in** with staggered metric cards; **all motion is gated behind
  `@media (prefers-reduced-motion: reduce)`** — keep any new animation gated too.
- Contrast was checked (body/label ≥ 4.5:1 in both themes). Keep it there.

---

## 6. Save / email feature — `copyReport()`, `emailReport()`, `buildReportText()`

- **100% client-side (no backend)** — works on any static host.
- **`copyReport()`** is the reliable path: `navigator.clipboard` with an `execCommand` fallback, and if
  both are blocked it **reveals the report in a selectable `<textarea id="reportFallback">`**.
- **`emailReport()`** validates the address, then opens a `mailto:` via a transient `<a>` (not
  `window.location`). If no mail client exists, nothing opens — hence Copy is offered alongside.
- `buildReportText()` includes summary + full cash-basis breakdown + **benchmark comparison**
  (captured into `lastBenchmarkText` by `buildLayman`). Status messages use `#emailStatus` (aria-live).

---

## 7. Deployment

**Currently deployed** (outside this repo) to a Zo "app hub" at:
`https://agent-deployed-applications-alyosha.zocomputer.io/sg-property-pnl`
— as an iframe sub-app wrapping the static HTML. That deployment is not required for this repo; the
file is fully standalone and can be hosted on GitHub Pages / Netlify / any static host as `index.html`.

**Pending:** the fixes in this session have not yet been committed to this GitHub repo. When committing,
match the repo's existing author identity (`git log` shows `Karen Xing`).

---

## 8. Open items / suggested next tasks

1. **Consolidate to `index.html`** and archive the dated snapshots (§1); write a real `README.md`.
2. **Decide benchmark-endpoint ownership** (§3): add a `BENCHMARK_API` config flag or self-host; the app
   already falls back to fixed estimates, so this is non-blocking.
3. (Optional) surface `#purchaseValuation` / FTA / LTV notes more prominently if users miss them.
4. Periodic: re-verify §4 tax rates against IRAS/MAS (they change by Budget/cooling measures).

---

## 9. Regression checks (run these after any logic change)

- **Breakdown reconciles:** after `calculate()`, sum the amount column of `#breakdownTable` (excluding the
  subtotal `tr.total` and grand `tr.grand` rows) — it must equal the Net P&L shown in `tr.grand`. Paste
  this in the browser console after clicking Calculate (expect `reconciles: true`, no errors):
  ```js
  (function(){var r=document.querySelectorAll('#breakdownTable tbody tr'),s=0,g=null;
  function p(t){t=t.trim();if(t==='—'||t==='')return 0;var k=t[0]==='-'?-1:1;return k*Number(t.replace(/[^0-9.]/g,''));}
  r.forEach(function(tr){var d=tr.querySelectorAll('td');if(d.length!==2)return;
  if(tr.classList.contains('grand')){g=p(d[1].textContent);return;}if(tr.classList.contains('total'))return;s+=p(d[1].textContent);});
  console.log({sum:Math.round(s),grand:g,reconciles:Math.abs(s-g)<2});})();
  ```
- **SSD boundaries** (purchase on/after 2025-07-04): 1yr→16, 2yr→12, 3yr→8, 4yr→4, 4yr+1day→0.
- **BSD** at $6.5M = S$329,600 (6% top, not 7%). At $1.5M = S$44,600.
- **LTV**: switching buyer to 2nd property caps LTV at 45%; 3rd+ at 35%.
- **FTA**: `getAbsdRate('FG','2',date,false,true)` returns 20 (SC-2nd), not 60.
- **Benchmark endpoint**: `GET .../benchmarks?years=5` returns `{sti,spy,tbill}`; blocking it must fall
  back to fixed estimates without a console error.
- **Compare against the reference implementation:** a known-good deployed build is live at
  https://agent-deployed-applications-alyosha.zocomputer.io/sg-property-pnl — same inputs should produce
  the same numbers and the same layout there. Use it to sanity-check UI or behaviour after a change.
  (External deployment; may change/move — a convenience reference, not a contract.)

---

## 10. Authoritative sources (verified 2026-07)

- SSD 4 Jul 2025 change — MAS: https://www.mas.gov.sg/news/media-releases/2025/extension-of-the-holding-period-of-sellers-stamp-duty-and-higher-ssd-rates
- SSD rates — IRAS: https://www.iras.gov.sg/taxes/stamp-duty/for-property/selling-or-disposing-property/seller's-stamp-duty-(ssd)-for-residential-property
- BSD rates — IRAS: https://www.iras.gov.sg/taxes/stamp-duty/for-property/buying-or-acquiring-property/buyer's-stamp-duty-(bsd)  ·  MOF (6% top): https://www.mof.gov.sg/news-resources/newsroom/buyer-s-stamp-duty-bsd-rates-to-be-raised-for-higher-value-properties/
- ABSD rates + FTA remission — IRAS: https://www.iras.gov.sg/taxes/stamp-duty/for-property/buying-or-acquiring-property/additional-buyer's-stamp-duty-(absd)
- LTV limits — MAS: https://www.mas.gov.sg/publications/macroprudential-policies-in-singapore

---

## 11. UI layout & interaction (redesigned 2026-07-02)

The page is a **two-column layout** (`.app-layout`): inputs on the left, a **sticky results panel**
on the right (`.col-results`). Below 900px it collapses to one column and the results un-stick. Do
not reintroduce a single-column "long form, results dumped at the bottom" flow — that was the old design.

- **Instant / live calculation.** Results recompute automatically as the user edits any input
  (debounced `liveRecalc()` bound to the `.col-inputs` container). The "Calculate P&L" button still
  works and scrolls to the results on mobile, but results are **no longer gated on it** — do not make
  the button mandatory again.
- **Empty state.** Before any meaningful input, `#emptyState` is shown and `#results` is hidden; the
  first real input reveals the results panel. Keep both elements and the toggle.
- **Segmented controls** replace three dropdowns (buyer type, property count, price basis). Each is a
  `.segmented` button group with `data-for="<selectId>"`. The real `<select>` still exists (visually
  hidden) and **remains the source of truth**: a button click sets the select's `value` and dispatches
  a `change` event, and `refreshAllSegmented()` mirrors state via `aria-pressed` and disables the
  property-count group for Entity buyers. All original `<select>` / input `id`s are unchanged, so the
  calculation logic is untouched. **If you add an option, add it to BOTH the hidden `<select>` and the
  button group** (and give the button a short label; long labels are truncated — e.g. "Foreigner" is
  shown as "Foreign" with the full name in `title`).
- Everything else — finance logic, element ids, and the breakdown reconciliation invariant — is exactly
  as documented in §4–§9. The regression check in §9 still passes.
