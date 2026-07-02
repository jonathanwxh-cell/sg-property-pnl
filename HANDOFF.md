# Handoff & Change Log — Singapore Property P&L Calculator

> Purpose: give a new contributor (human or AI agent) with **no prior context** everything
> needed to safely continue this project. Read this top to bottom before editing.

---

## 0. TL;DR

A single, self-contained HTML calculator that estimates the **net profit/loss of a Singapore
residential property transaction** after stamp duties, mortgage, and costs, and compares it to
STI / S&P 500 / T-bill benchmarks. No build step, no framework — one HTML file with inline CSS + vanilla JS.

Working sessions have since (1) fixed **correctness bugs in the tax/P&L logic** (two QA passes),
(2) replaced a client-side Twelve Data API-key feature with a **hosted benchmark endpoint**, (3) rebuilt
the **UI as an editable "story"** with a live result + sensitivity table, and (4) fixed **save/email**.
**§4 is the current, canonical statement of the tax/finance invariants you must not regress**; **§11–§15
are the dated change history** behind them (read §4 as truth — if a §11–§15 note ever conflicts, §4 wins).

---

## 1. Canonical file (READ FIRST — easy to get wrong)

**`index.html` is the single, canonical file** — the entire calculator (HTML + inline CSS + vanilla JS)
lives in it. This repo does **not** keep dated snapshots; do not reintroduce that pattern. (The original
`AngsumalinX/sg-property-pnl` repo kept `YYYY-MM-DD-*.html` snapshots — that history stays there.)

A live, known-good build is deployed at
**https://agent-deployed-applications-alyosha.zocomputer.io/sg-property-pnl** — compare against it after
any change (same inputs ⇒ same numbers + layout).

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
- **Regime chosen by PURCHASE date**; tiers by **calendar anniversary**, NOT a 365.25-day count.
- **The anniversary boundary is EXCLUSIVE:** `within(n)` is `sDate < anniv(n)`. Disposing **on** the Nth
  anniversary counts as "held **more than** N years" (the lower/zero tier), per IRAS — e.g. bought
  21 Mar 2023, sold on/after 21 Mar 2026 ⇒ **no SSD**. A tier's last chargeable day is the anniversary − 1
  (returned as `boundary`, with `nextRate`, for the day-level cliff display). **Do NOT revert to `<=`.**
- **On/after 4 Jul 2025:** 4-year holding — ≤1yr 16 / ≤2yr 12 / ≤3yr 8 / ≤4yr 4 / >4yr 0.
- 11 Mar 2017 – 3 Jul 2025: 3-year — 12 / 8 / 4 / 0. Earlier regimes (Jan 2011, pre-2011) retained.
- SSD is charged on the **higher of sale price or market valuation** (see 4.5).

### 4.2 Buyer's Stamp Duty — `getBsdRate(price, date)`
Three regimes by purchase date:
- **≥ 15 Feb 2023:** 1 / 2 / 3 / 4 / 5 (1.5–3M) / **6% (>3M)** — top **6%, no 7% band**.
- **20 Feb 2018 – 14 Feb 2023:** 1 / 2 / 3 / **4% (>1M)**.
- **Before 20 Feb 2018:** 1 / 2 / **3% top** (the 4% band started 20 Feb 2018).
- Check: S$2M ⇒ S$54,600 (pre-2018) / S$64,600 (2018–2023) / S$69,600 (2023+).

### 4.3 Additional Buyer's Stamp Duty — `getAbsdRate(bt, pc, date, remission, fta, spouseRemitFirst)`
- Rates by regime (SC / PR / FG / Entity; 1st-property rate is 0 for SC, 5 for PR):
  - **≥ 27 Apr 2023:** SC 0/20/30, PR 5/30/35, FG 60 (flat), Entity 65 (flat).
  - **16 Dec 2021 – 26 Apr 2023:** SC 0/17/25, PR 5/25/30, FG 30, Entity 35. *(Do not skip this regime.)*
  - **6 Jul 2018 – 15 Dec 2021:** SC 0/12/15, PR 5/15/15, FG 20, Entity 25. Earlier regimes retained.
- **FTA nationals** (US; nationals/PRs of Iceland, Liechtenstein, Norway, Switzerland) get
  **Singapore-Citizen treatment** — the `fta` flag maps a Foreigner to SC rates.
- **Replacement remission** (`remission`, "Section B"): a **married couple with ≥1 SC/PR buying their 2nd
  property** while selling the first within 6 months → rate 0. Modelled **pay-now / reclaim-later** — ABSD
  is paid upfront (counts in peak/bridging cash) and shown refunded; Net P&L excludes it.
- **Spouse remission** (`spouseRemitFirst`, "Section A"): a **married couple with ≥1 SC spouse buying
  their 1st property** jointly, neither owning any other residential property → rate 0, **whatever the
  other spouse's nationality** (SC+foreigner qualifies). **Full remission at stamping** — no ABSD paid, no
  bridging, no refund line. Model the couple as the higher-rate spouse (PR/FG). See §15.

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
- MAS limits by property count: **75 / 45 / 35** (1st / 2nd / 3rd+); the UI caps the slider/input and
  clamps the value.
- **ENFORCED for long tenure:** loan tenure **> 30 years** drops the cap to **55 / 25 / 15** (editing
  `#loanTenure` re-runs `updateLtvCap()` to re-clamp). Age > 65 at maturity would also reduce it but is
  **not** modelled (age isn't collected).

### 4.7 Rental yields — `computePnl`
- **Gross yield** uses full contractual annual rent (100% occupancy) — the market convention.
- **Net yield is occupancy-adjusted** (occupancy-scaled effective rent) so it tracks the P&L cash flow.
  Occupancy is clamped to **0–100%** and 0% is honoured (do not treat a 0 as "unset").

### 4.8 Input safety & display (do not regress)
- Invalid (`validity.badInput`) and negative numeric fields surface a warning in `#inputWarn` rather than
  silently defaulting/zeroing; a sub-1-year loan tenure is clamped to 1 year; money chips are positive-only.
- When required inputs are missing/invalid, `calculate()` hides `#results`, **nulls `lastReport`, and
  destroys the benchmark chart** — nothing stale can be exported (copy/email guard null) or left hidden.
- A break-even ≤ 0 shows **"S$0 or below"** / "profitable at any price" (never a misleading positive).
- Scenario sliders persist across base-case edits; there is no "stale base" warning (results are live).

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

*(The earlier "consolidate to `index.html` + write a README" task is done — the repo is a single
`index.html` with `README.md` / `AGENTS.md` / `CLAUDE.md` / this file.)*

1. **Decide benchmark-endpoint ownership** (§3): point `BENCHMARK_API` at your own host or self-host it;
   the app already falls back to fixed estimates, so this is non-blocking.
2. **Rates are hardcoded** (no live rates API) — after any SG Budget / cooling-measure change, re-verify
   the §4 rates & dates against IRAS/MAS (§10) and update `getSsdInfo` / `getBsdRate` / `getAbsdRate`.
3. (Optional) remission / FTA eligibility is **asserted via checkboxes, not enforced** — a future version
   could collect the underlying facts (spouse nationality, other-property ownership) to reduce misuse.
4. (Optional, bolder) the scenario **rate × sale-price heatmap** sketched at the end of §13.

---

## 9. Regression checks (run these after any logic change)

- **Breakdown reconciles:** after `calculate()`, sum the amount column of `#breakdownTable` (excluding the
  subtotal `tr.total` and grand `tr.grand` rows) — it must equal the Net P&L shown in `tr.grand`. Paste
  this in the browser console once results are showing — they update live as you edit; there is no
  Calculate button (expect `reconciles: true`, no errors):
  ```js
  (function(){var r=document.querySelectorAll('#breakdownTable tbody tr'),s=0,g=null;
  function p(t){t=t.trim();if(t==='—'||t==='')return 0;var k=t[0]==='-'?-1:1;return k*Number(t.replace(/[^0-9.]/g,''));}
  r.forEach(function(tr){var d=tr.querySelectorAll('td');if(d.length!==2)return;
  if(tr.classList.contains('grand')){g=p(d[1].textContent);return;}if(tr.classList.contains('total'))return;s+=p(d[1].textContent);});
  console.log({sum:Math.round(s),grand:g,reconciles:Math.abs(s-g)<2});})();
  ```
- **SSD boundaries** (purchase on/after 2025-07-04): a sale the **day before** the Nth anniversary is in
  tier N; **on/after** the anniversary drops a tier. e.g. bought 2 Jul 2026: sell 1 Jul 2030 → 4%, sell
  **2 Jul 2030 (4th anniversary) → 0%** (getSsdInfo uses strict `<`; see §4.1).
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

## 11. UI: the editable-story layout (redesigned 2026-07-02)

The inputs are presented as an **editable plain-English sentence** (`.story`), not a form:
"As **a Singapore Citizen**, I'm buying **my 1st property** for **S$1,500,000**. I'll **live in it**,
hold it **5** years, then sell at **S$1,800,000**." Each bold/underlined value is an inline **chip**
(`.chip`) that is the *real* input control. A live result panel sits alongside (sticky on desktop, and
stacked below on mobile). Do not turn this back into a labelled grid of fields.

- **The chips ARE the inputs** (same element `id`s the JS has always used):
  - `#buyerType`, `#propCount`, `#usagePurpose` are inline `<select>`s (`.chip-select`), auto-sized to
    the selected option by `fitSelect()` so they read as part of the sentence.
  - `#purchasePrice` and `#salePrice` are **comma-formatted text inputs** (`type="text"`,
    `inputmode="numeric"`). `formatMoney()` reformats them with thousands separators on input, and
    **every place that reads their value strips commas** (`.replace(/,/g,'')`) — the core parser
    `num()` in `calculate()`, plus `recalcLoan`/`updateBsdHint`/`updateAbsdHint`/`updateRentalCalc`.
    If you add another money chip, keep both halves of that contract.
  - `#holdYearsQuick` is a small chip that writes `#saleDate` (= purchase date + N years) via
    `applyHoldYears()`; editing the exact dates in the drawer syncs it back via `syncHoldYearsField()`.
- **Everything else lives in the `<details class="more-options">` "Fine-tune the details" drawer**:
  exact dates, price basis (lump vs PSF), LTV/loan/tenure/rates, fees, valuations, purpose note, rental. It is **expanded by default** (the details element carries the `open` attribute) so every input is visible on first load; remove `open` to restore collapsed-by-default progressive disclosure.
  `#purchaseLumpField` / `#saleLumpField` are now short pointer notes (kept so `togglePriceBasis()`
  can still show/hide them when switching to PSF).
- **Prefilled worked example**: on load the story is filled (SC / 1st / S$1.5M / 5yr / S$1.8M / 2.75% /
  25yr) and `calculate()` runs so a live result shows immediately. An empty `#purchasePrice` (or
  `#salePrice`) hides the result via the existing empty-state guard.
- The **segmented button controls** from the previous revision were removed; the leftover `.segmented`
  CSS/JS is vestigial (its `querySelectorAll('.segmented')` simply matches nothing) and can be deleted.

**New do-not-regress (added this revision):**
5. **IRR is floored at -100%** in `computePnl` when ending equity (`totalCashIn + netPnl`) is `<= 0`.
   A loss larger than the cash invested makes `Math.pow(negativeBase, 1/years)` return **NaN**, which
   previously crashed `buildLayman` (`Cannot read 'gain' of undefined`). Keep the `endingEquity > 0`
   guard, and keep `buildLayman`'s empty-filter guard (`lostList[0] ? … : 0`).

Everything in §4–§9 (tax rules, element ids, and the breakdown reconciliation invariant) is unchanged;
the §9 regression check still passes for the prefilled example and after edits.


---

## 12. Correctness & modelling fixes (2026-07-02 QA pass)

Eight issues from a QA review were fixed. Do not regress:

1. ABSD - 16 Dec 2021 regime added. getAbsdRate previously jumped from Jul 2018 straight to Apr 2023,
   undercharging 16 Dec 2021 to 26 Apr 2023 purchases. Regime columns are now
   [Apr 2023+, 16 Dec 2021, 6 Jul 2018, 12 Jan 2013, 8 Dec 2011]. e.g. SC 2nd on 2023-04-26 = 17%
   (S$272k on S$1.6m), PR 2nd = 25% (S$400k).
2. ABSD remission = pay-now / refund-later. computePnl returns absdFullRate, absdPaid (charged upfront),
   absdRefund, effective absd (0 when remitted) and peakUpfrontCash. The breakdown shows ABSD paid + a
   refund line and a "Peak upfront cash (incl. refundable ABSD)" subtotal; Net P&L is unchanged (paid +
   refund net to zero) and the table still reconciles.
3. Net rental yield is occupancy-adjusted (uses rentalGrossPa); gross yield stays at full contractual
   occupancy (market convention).
4. Break-even handles valuation-based SSD. When the sale valuation exceeds the break-even price, SSD is
   fixed (valuation x rate) not price-scaled; both regimes are solved and the consistent one is used.
5. Input guards. Negative rates are floored at 0% and a sub-1-year loan tenure is clamped to 1 year
   (a mortgage cannot be 0 years); both surface a warning in #inputWarn so bad input is not silently rosy.
6. IRR floored at -100% when ending equity <= 0 (see section 11) - prevents a NaN that crashed buildLayman.
7. SSD day-level cliff surfaced. getSsdInfo returns boundary/nextRate; the SSD note states the exact date
   the tier ends, and the holding-period metric shows days + the cliff date.
8. Live recalc covers valuation fields (they live inside .col-inputs), so purchase/sale valuation edits
   update the P&L immediately - no stale results.

---

## 13. Scenario analysis + sensitivity (2026-07-02)

The "Scenario analysis" panel was tested (revised figures verified exact against computePnl across LTV,
rate and sale-price sliders, singly and combined) and improved:

- Sliders persist across base edits. initScenarioPanel() seeds the three sliders (LTV, rate delta,
  sale-price delta) once via the scenReady flag, then keeps the user's positions and re-applies the
  deltas to the new base. Previously every keystroke reset them. resetScenario() still returns to base.
- Removed the vestigial stale-warning (#scenStaleWarning + currentFormSnapshot/lastFormSnapshot):
  results are live, so there is no "stale base" to warn about, and its copy referenced the removed
  Calculate button.
- Downside break-even cushion (#downsideBreakeven): shows how far the sale price can fall (to the
  break-even price, as a %) before Net P&L goes negative - the single most useful risk number.
- Sensitivity table (#sensitivityTable, renderSensitivity()): pre-baked one-at-a-time shocks
  (rates +1/+2/+3pp, sale -5/-10/-20%, bigger downpayment), each with Net P&L + annualised return,
  colour-coded, base row highlighted. Base-driven: recomputed from lastInputs on every base recalc,
  independent of the sliders.

Everything routes through the same computePnl(), so the reconciliation invariant and all tax rules apply
unchanged.

---

## 14. Correctness fixes — second QA pass (2026-07-02)

1. SSD anniversary boundary. getSsdInfo now uses within(n) = sDate < anniv(n) (strict). IRAS treats a
   sale ON the Nth anniversary as "more than N years": e.g. bought 21 Mar 2023, sold on/after
   21 Mar 2026 => no SSD. A tier's last chargeable day is the anniversary minus one (the boundary field).
2. BSD pre-20 Feb 2018. getBsdRate adds the pre-2018 regime: 1/2/3% only (the 4% band started
   20 Feb 2018). S$2m before 20 Feb 2018 = S$54,600; 20 Feb 2018-14 Feb 2023 = S$64,600; 15 Feb 2023+ = S$69,600.
3. MAS LTV for long tenure. ltvCap() returns 55/25/15 (by property count) when loan tenure > 30 years,
   else 75/45/35. Editing loanTenure re-runs updateLtvCap() to re-clamp. (Age > 65 at maturity still
   not modelled - age isn't collected.)
4. Input validation. calculate() flags fields the browser can't parse (validity.badInput) and negative
   amounts via #inputWarn instead of silently defaulting/zeroing. Money chips stay positive-only.
5. Stale state on invalid input. When the guard hides #results, lastReport is nulled and the benchmark
   chart destroyed, so nothing stale can be exported (copy/email already guard null) or linger hidden.
6. Valuation refreshes the chip. purchaseValuation oninput now also calls updateAbsdHint(), so the
   prominent ABSD/BSD chip reflects duty on the higher of price or valuation, not just the drawer hint.
7. Negative break-even. A non-positive break-even shows "S$0 or below" (metric) and "profitable even at
   a S$0 sale price" (cushion) instead of a misleading positive number; the bar ratio is guarded.
8. 0% occupancy. updateRentalCalc() uses a NaN-safe parse so 0% occupancy is honoured in the helper text
   and effective-rent figure (previously || 90 silently turned 0 into 90).

RESOLVED in section 15 (implemented after confirming against IRAS). ABSD remission for a married couple where one spouse is
a foreigner buying their first matrimonial home. Sources conflict on whether a SC+foreigner couple
qualifies; not added because wrongly showing 0% instead of 60% ABSD would be a serious undercharge.
Confirm the exact IRAS provision (Stamp Duties (Spouses) (Remission of ABSD) Rules) before implementing.


---

## 15. ABSD spouse remission (Section A) - implemented (2026-07-02)

Verified against IRAS "Remission of ABSD for a Married Couple" (Stamp Duties (Spouses) (Remission of
ABSD) Rules), Section A: FULL ABSD remission for a married couple buying their FIRST residential property
jointly where at least one spouse is a Singapore Citizen and neither spouse owns any other residential
property - regardless of the other spouse's nationality (so SC + foreigner qualifies). It is a full
remission at stamping (no ABSD paid, no bridging cash) - unlike the Section B replacement refund.

Implementation:
- getAbsdRate(bt, pc, date, remission, fta, spouseRemitFirst) returns 0 when spouseRemitFirst and it is
  a 1st property.
- New checkbox #absdSpouseRemit, shown by updateAbsdHint only for a 1st property with a PR or foreigner
  profile (a citizen's 1st property is already 0%). Model the couple as the higher-rate spouse (PR/FG).
- computePnl: when spouseRemit, absdPaid = absd = absdRefund = 0 (no bridging, no refund line); the
  breakdown shows "ABSD (X%, remitted - spouse relief)". Net P&L equals the citizen first-property case
  and the table reconciles.

Also fixed here: the ABSD "rates effective" chip badge now includes the 16 Dec 2021 regime.
