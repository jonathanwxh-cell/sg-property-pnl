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
**§4 is the current, canonical statement of the tax/finance invariants you must not regress**; **§11–§18
are the dated change history** behind them (read §4 as truth — if a §11–§18 note ever conflicts, §4 wins).

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

### 4.1 Seller's Stamp Duty — `getSsdInfo(pd, sd, base)`
- **Regime chosen by PURCHASE date**; tiers by **calendar anniversary**, NOT a 365.25-day count.
- **The anniversary boundary is EXCLUSIVE:** `within(n)` is `sDate < anniv(n)`. Disposing **on** the Nth
  anniversary counts as "held **more than** N years" (the lower/zero tier), per IRAS — e.g. bought
  21 Mar 2023, sold on/after 21 Mar 2026 ⇒ **no SSD**. A tier's last chargeable day is the anniversary − 1
  (returned as `boundary`, with `nextRate`, for the day-level cliff display). **Do NOT revert to `<=`.**
- **On/after 4 Jul 2025:** 4-year holding — ≤1yr 16 / ≤2yr 12 / ≤3yr 8 / ≤4yr 4 / >4yr 0.
- 11 Mar 2017 – 3 Jul 2025: 3-year — 12 / 8 / 4 / 0. 14 Jan 2011 – 10 Mar 2017: 4-year 16 / 12 / 8 / 4.
- **2010 progressive regimes** (before 14 Jan 2011): SSD is a **BSD-style progressive amount** — 1% first
  $180k, 2% next $180k, 3% remainder — NOT a flat %. **30 Aug 2010 – 13 Jan 2011:** 3-year taper (full /
  two-thirds / one-third by holding year). **20 Feb 2010 – 29 Aug 2010:** within 1 year only, full
  progressive. **Before 20 Feb 2010:** SSD did not exist (0%). `getSsdInfo` takes the SSD `base` (higher of
  sale price/valuation) and returns the exact `amount` plus an effective `rate` (so the flat-rate display and
  break-even stay unchanged). Check: **$1.8M bought 30 Aug 2010, sold 29 Aug 2012 ⇒ S$32,400** (two-thirds of
  the $48,600 full progressive; ~1.8% effective — the within-1-year case is $48,600 progressive, not $54,000 flat).
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
- **Replacement remission** (`remission`, "Section B"): a **married couple with a Singapore-Citizen spouse
  buying their 2nd property** while selling the first within 6 months → rate 0. Qualifies **whatever the
  higher-rate profile** (SC+foreigner included — model the couple as the higher-rate spouse, e.g. FG); the
  checkbox asserts the SC spouse and shows for any non-entity 2nd-property buyer. Modelled **pay-now /
  reclaim-later** — ABSD paid upfront (counts in peak/bridging cash), shown refunded; Net P&L excludes it.
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
- Loan tenure is clamped to **≤ 35 years** (MAS max) and `buildAmort` hard-caps the schedule at 600 months,
  so an absurd/pasted tenure (e.g. `1e308`) can't spin an infinite amortisation loop and hang the page.

### 4.7 Rental yields — `computePnl`
- **Gross yield** uses full contractual annual rent (100% occupancy) — the market convention.
- **Net yield is occupancy-adjusted** (occupancy-scaled effective rent) so it tracks the P&L cash flow.
  Occupancy is clamped to **0–100%** and 0% is honoured (do not treat a 0 as "unset").

### 4.8 Input safety & display (do not regress)
- `num()` clamps every numeric field to **[0, 1e12]** (floors negatives; caps absurd/`1e308` values so they
  can't overflow to Infinity). **Interest rates get a tighter clamp** — `buildAmort` caps `fixedRate`/`floatRate`
  to **[0, 100]%**, because even a 1e12% rate overflows `Math.pow(1+r, n)` to Infinity and leaks NaN into every
  figure (Net P&L, interest, balance, the copied report); a rate over 100% also warns. A field the browser
  can't parse (`validity.badInput`) **blocks the calc
  entirely** — `#results` hidden, `lastReport` nulled, empty-state explains — so an invalid field never
  silently computes or exports with a defaulted value. A sub-1-year loan tenure is clamped to 1 year.
- **Negative amounts block the calc** — `_negField` is in the invalid guard, so a negative in ANY field
  (price, sale, fee) hides results with the "amount is negative" message; a money chip keeps a visible
  leading `-` (not silently flipped positive). **Occupancy is visibly clamped to 0–100 in the field itself**
  (typing 150 snaps the field to 100, −20 to 0), so the calc/export can never use a value the UI doesn't show.
  The break-even cushion copy guards a `/ sp` divide so a S$0 sale never prints `+Infinity`.
- A **sale price of 0 is valid** (total loss / gift) — the guard requires the field *provided*, not truthy;
  the **purchase price** must still be > 0.
- When inputs are missing/invalid, `calculate()` hides `#results`, **nulls `lastReport`, destroys the chart,
  and blanks the stale result/breakdown/chart text** — nothing stale is shown or exported (copy/email guard null).
- **Underwater sale** (`#underwaterNote`): if proceeds after selling costs don't cover the outstanding loan,
  it shows the **cash top-up** needed at completion.
- A break-even ≤ 0 shows **"S$0 or below"** / "profitable at any price" (never a misleading positive).
- Scenario **rate/sale-price** sliders persist across base edits; the scenario **LTV re-syncs** to the main
  LTV (it's absolute). No "stale base" warning (results are live).
- **A11y & touch:** section titles are **native `<h2 class="section-title">`** under the page's single `<h1>`
  (a CSS margin/size reset keeps them reading as labels, not big headings); range sliders use a **26px thumb /
  8px track** for touch; `#results` has no broad `aria-live` — a small `#srSummary` (`role="status"`) announces
  a one-line Net-P&L summary on recalc.
- **Prefilled example fields clear their untouched example on focus** (`purchasePrice`, `salePrice`,
  `holdYearsQuick`, `fixedRate`, `loanTenure`) so the first keystroke starts a fresh number and can never
  concatenate the example with typed digits (no "1,800,000,700,000"); an un-typed focus restores the example on
  blur. This replaced select-on-focus, which didn't survive a real click's mouseup or `formatMoney`'s caret rewrites.
- The top **BSD/ABSD hint chips recompute on every price, valuation AND purchase-date change**, so they always
  match the date-aware duty the calc uses (a 2018-02-19 purchase shows the pre-2018 3%-top BSD = S$54,600 on S$2M).

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

---

## 16. Robustness & accessibility — third QA pass (2026-07-02)

1. **No hang on huge inputs.** `num()` clamps to [0, 1e12]; loan tenure clamps to ≤ 35 (with a warning);
   `buildAmort` hard-caps the schedule at 600 months. A pasted `1e308` tenure previously made `totalM`
   Infinity and spun the amortisation loop forever, hanging the page.
2. **SC + foreigner refund path.** Section B replacement remission now applies to **any couple with a SC
   spouse** (modelled as the higher-rate profile, e.g. foreigner), not just SC/PR profiles — the checkbox
   shows for a 2nd property whenever the buyer isn't an entity. (See §4.3.)
3. **Invalid native-number inputs block.** A `validity.badInput` field now hides results and nulls
   `lastReport` (blocking export), instead of computing/exporting with a silent default.
4. **Negative money is visible.** `formatMoney` keeps a leading `-`, so a negative price is shown and
   treated as invalid (0 → guard) + flagged, instead of silently becoming positive.
5. **Scenario LTV re-syncs** to the main LTV on every recalc (the rate & sale-price deltas still persist).
6. **Underwater warning** (`#underwaterNote`): shows the cash top-up needed when sale proceeds don't cover
   the outstanding loan.
7. **S$0 sale modelable** (the guard requires the sale field *provided*, not truthy); invalid states also
   blank the stale results / breakdown / chart DOM.
8. **Mobile break-even label** moved above the bar (it was overlapping the coloured fill on narrow screens).
9. **Accessibility:** removed the broad `aria-live` on `#results`; added a concise `#srSummary`
   (`role="status"`) live region announcing the Net-P&L one-liner; section titles are `role="heading"`.

---

## 17. Input-edge fixes — fourth QA pass (2026-07-02)

1. **Occupancy > 100%** warns ("treated as 100%") instead of silently continuing (the math clamps to 100).
2. **Negative amounts block** the calc (`_negField` added to the invalid guard) — a negative sale no
   longer computes as a S$0 sale, and the break-even cushion copy can't divide by 0 to print `+Infinity`.
   The S$0-sale cushion line also guards the divide (shows the break-even amount without a %).
3. **Prefilled fields select-on-focus** (`purchasePrice`, `salePrice`, `holdYearsQuick`, `fixedRate`,
   `loanTenure`) so the first keystroke replaces the example instead of appending — clicking "1,800,000"
   and typing no longer yields "1,800,000,700,000".
4. **"See the full breakdown"** CTA now calls `showFullBreakdown()`: it computes, **expands the collapsed
   cost-breakdown table**, and scrolls to it (previously it only scrolled and left the table collapsed).

---

## 18. Correctness & UX re-fixes — fifth QA pass (2026-07-02)

Owner re-reported that four earlier fixes were still visibly broken in real use (they were, or were only
half-fixed), plus a mobile/a11y polish pass. All re-verified with **real keystroke/click interaction** (the
earlier passes had tested some of these by calling functions directly, which masked the actual UI bug):

1. **Occupancy now visibly clamps to 0–100 in the field.** The fourth-pass "warn but keep calculating at a
   capped 100%" was insufficient — the field could still *show* 150 while the calc/export silently used 100.
   `updateRentalCalc()` now writes the clamped value back to `#occupancy` (150 → 100, −20 → 0), so what you
   see is what's computed; the now-moot ">100% warning" was removed.
2. **Historical `#bsdHint` now matches the date-aware BSD.** `updateBsdHint()` was date-aware but only wired to
   price/valuation input, not the date picker, so changing the purchase date left the top chip stale
   (2018-02-19 showed S$69,600 instead of S$54,600). `purchaseDate`'s `onchange` now runs
   `updateAbsdHint(); updateBsdHint()`. Verified the chip flips S$69,600 ↔ S$54,600 on the date change.
3. **Sale-field append fixed for real (clear-on-focus).** Fourth-pass select-on-focus did not survive a real
   click's mouseup or `formatMoney`'s per-keystroke caret rewrites — typing into the prefilled sale field gave
   "1,800,000,700,000" (append) or "0" (mid-type corruption from a deferred select). Replaced with **clear the
   untouched example on focus** (restore on an un-typed blur). Typing 700000 now yields "700,000"; confirmed
   with real `pressSequentially` keystrokes, not a direct value set.
4. **"See the full breakdown" confirmed expanding.** Re-tested with a real hit-tested pointer click (verified
   nothing overlays the button): `#breakdownBody` goes none → block with 19 rows and scrolls into view. The
   owner's "still broken" was a stale cached/deployed copy — the live reference URL now serves the working fix.
5. **Mobile / a11y polish.** Section labels are now **native `<h2 class="section-title">`** (10 of them) under
   the single `<h1>` (a CSS reset keeps their look unchanged); range sliders got a **26px thumb / 8px track**
   for touch. The hub wrapper's floating back-control (in `agent-deployed-applications/src/apps/sg-property-pnl/
   index.tsx`, NOT this repo) became a **40px blurred "‹ All apps" pill** with a legible label and safe-area
   insets, replacing the tiny 12px chip.

Verified in-browser on both localhost and the live reference URL: 10 `<h2>` section titles / 0 leftover
`role="heading"` divs, 8px slider track, occupancy 150→100 & −20→0, `bsdHint` S$69,600↔S$54,600 on date flip,
sale field → "700,000", breakdown expands to 19 rows, no console errors, no mobile horizontal overflow, and the
cost breakdown still reconciles exactly to Net P&L.

---

## 19. Historical SSD regimes + interest-rate overflow — sixth QA pass (2026-07-03)

Two findings from an external "round-8" critical-verify script (`fb447c3`):

1. **`fixedRate` / `floatRate` = `1e308` leaked `NaN`** into Net P&L, interest and loan balance — in the
   visible UI *and* the copied report. `num()`'s [0, 1e12] clamp is too loose for a *rate*: even 1e12% makes
   `Math.pow(1 + r, n)` overflow to Infinity, and Infinity / Infinity = NaN. Fixed by clamping the annual rates
   to **[0, 100]% inside `buildAmort`** (so both the base render and the scenario panel are covered), plus a
   "rate above 100% is treated as 100%" input warning. Real mortgage rates are single digits, so 100% is an
   unreachable ceiling, not a real limit.
2. **Pre-2011 SSD was a wrong flat-3% / 1-year stub.** Added the two 2010 progressive regimes IRAS actually
   applied: **20 Feb 2010 – 29 Aug 2010** (SSD only if sold within 1 year; progressive 1% / 2% / 3% on the
   $180k / $180k / remainder bands) and **30 Aug 2010 – 13 Jan 2011** (up to 3 years, tapering full / two-thirds
   / one-third by year). **Before 20 Feb 2010 there was no SSD.** `getSsdInfo(pd, sd, base)` now takes the SSD
   base and returns the exact progressive `amount` plus an effective `rate` (flat-rate display and break-even
   unchanged). Verified: $1.8M bought 30 Aug 2010 → sold 29 Aug 2012 = **S$32,400** (2/3 of the $48,600 full
   progressive; the within-1-year case is the progressive $48,600, not a flat $54,000); pre-20-Feb-2010 and
   >3-year holds are 0; the Jan 2011 / Mar 2017 / Jul 2025 flat regimes are unchanged.

Verified in-browser on localhost + the live reference URL, function-level and full end-to-end (including the
copied report): no `NaN` with 1e308 rates, correct SSD across all six regimes, and the breakdown still reconciles.
