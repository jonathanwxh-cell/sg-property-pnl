# AGENTS.md — read this before editing

AI agents and new contributors: **read [`HANDOFF.md`](./HANDOFF.md) fully before making changes.**
It documents the architecture, every recent fix, and the correctness invariants. This file is the
operating brief; HANDOFF.md is the depth.

## What this is
A single self-contained HTML calculator (`index.html`) — no build step, vanilla JS + inline CSS —
that estimates the net profit/loss of a Singapore residential property transaction (stamp duties,
mortgage, costs) and compares it to STI ETF / S&P 500 / T-bill benchmarks.

## Canonical file
**`index.html` is the single source of truth.** (The original repo, `AngsumalinX/sg-property-pnl`,
kept dated `YYYY-MM-DD-*.html` snapshots; those stay there for history. Do not recreate that pattern.)

## Reference implementation (known-good)
A deployed, working build of this exact code is live at
**https://agent-deployed-applications-alyosha.zocomputer.io/sg-property-pnl**. Use it as your reference:
open it to see the intended UI, behaviour, and numbers, and **compare your local version against it after
changes** — same inputs should produce the same output and layout. (It's an external deployment that may
change or move, so treat it as a convenience reference, not a permanent contract.)

## Working constraints — how to change this codebase (do not violate)
1. **Keep it ONE self-contained file.** No build step, no framework (React/Vue/Svelte/etc.), no
   bundler, no `npm install`. If you genuinely need a library, add it via a CDN `<script>`/`<link>` —
   that's how Chart.js, Tabler icons, and Google Fonts are already loaded.
2. **Plain browser JS.** ES5-style `var` + `function` declarations, no modules/JSX/TypeScript. It runs
   directly in the browser with no transpile — match the existing style.
3. **Preserve element `id`s and the result HTML class names.** The JS reads inputs by `id` and rebuilds
   results via `innerHTML` using specific classes (`.metric`, `.metric-val`, `.breakdown-table tr.grand`,
   `.bmark-card`, `#metricGrid`, `#breakdownTable`, …). Renaming them silently breaks the calculator.
4. **`computePnl(inputs)` is a pure function** used by BOTH the base-case render and the scenario panel.
   Keep it pure; a change there updates both — and you must re-run the reconciliation check (below).
5. **Keep the light + dark themes** (CSS custom properties in `:root` + the `prefers-color-scheme`
   block) and keep every animation behind `@media (prefers-reduced-motion: reduce)`.

## Do NOT regress (details + rationale in HANDOFF.md §4–§6)
1. **SSD** (`getSsdInfo(pd, sd, base)`): 4 Jul 2025 regime = 16/12/8/4 over **4 years**; regime by purchase
   date; tiers by **calendar anniversary** with an **exclusive boundary** — a sale ON the Nth anniversary
   counts as "more than N years" (strict `<`; e.g. sold on the 4-yr anniversary ⇒ 0%). Do NOT revert to `<=`.
   Historical regimes retained incl. the **2010 progressive** ones (1%/2%/3% bands, tapering full/⅔/⅓ for
   30 Aug 2010–13 Jan 2011; 1-year-only for 20 Feb–29 Aug 2010; **no SSD before 20 Feb 2010**) — they return an
   exact `amount` off the SSD `base`, so keep passing it. Check: $1.8M, 30 Aug 2010→29 Aug 2012 ⇒ S$32,400.
2. **BSD** (`getBsdRate`): three regimes — before 20 Feb 2018 **3% top**; 20 Feb 2018–14 Feb 2023 **4%**
   top; ≥15 Feb 2023 **6% above $3M** (no 7% band). Check: S$2M ⇒ 54,600 / 64,600 / 69,600.
3. **Net P&L** (`computePnl`): financing cost is **interest only** — do NOT add `loan`/subtract
   `outstandingLoan` (that double-counts principal and overstated profit ~2×). The breakdown table +
   email report must **sum exactly to Net P&L** (cash basis).
4. **LTV** capped 75/45/35 by property count, **reduced to 55/25/15 when loan tenure > 30 years**;
   **duties on max(price, valuation)**; **gross yield at full occupancy but net yield occupancy-adjusted**
   (0% occupancy honoured); **ABSD** FTA-national → Singapore-citizen treatment, and ABSD covers **every
   regime incl. 16 Dec 2021** (SC 2nd 17 / 3rd 25; PR 2nd 25 / 3rd 30; FG 30; Entity 35).
5. **ABSD remissions** — both need only a **Singapore-Citizen spouse** (SC+foreigner qualifies; model the
   couple as the higher-rate profile): *(Section A, `#absdSpouseRemit`)* **1st** home together, neither owns
   other property → **full remission at stamping, 0% ABSD, no bridging**. *(Section B, `#absdRemission`)*
   **2nd** home, sell the first within 6 months → **pay-now / refund-later** (paid upfront as bridging,
   shown refunded, excluded from Net P&L). Section B shows for any non-entity 2nd-property buyer.
6. **Robustness:** **break-even** accounts for SSD fixed on a higher sale valuation (shows "S$0 or below"
   when non-positive); **IRR floors at -100%** on equity wipe-out; **loan tenure capped ≤ 35yr and the
   amortisation loop hard-capped** (an absurd/`1e308` tenure can't hang the page); `num()` clamps to
   [0, 1e12]; **`validity.badInput` and negative amounts BLOCK** the calc (results hidden, `lastReport`
   nulled → no stale export), negatives shown not silently flipped; **S$0 sale is valid**; an **underwater
   sale** shows the cash top-up (`#underwaterNote`). **Occupancy and loan tenure visibly clamp in the field**
   (0–100 / max 35); **money fields clear on every focus and `formatMoney` caps them at 10 digits, parses a
   decimal as dollars-and-cents (a "1500000.00" paste is 1,500,000 not 150,000,000) and honours k/m shorthand
   ("1.65m" → 1,650,000)**, so re-editing replaces and no absurd value like "1,800,000,700,000" can form; a value above the `num()` 1e12
   cap **warns**; the top **BSD/ABSD hint chips refresh on purchase-date change too** (not just price/valuation).
   A11y: **native `<h2>` section titles** under the single `<h1>`, **26px slider thumbs / 8px tracks** for
   touch, and a narrow `#srSummary` live region (no broad `aria-live`). Yields are labelled **scheduled**
   (gross, 100% occupancy) vs **effective** (net, occupancy-adjusted); the copied report includes an
   **ASSUMPTIONS & INPUTS** block for auditability. Duty **hint chips refresh on any price/valuation/date change
   and clear on a blank price**; the ABSD chip shows remission **bridging cash** (Section B pays upfront, then
   reclaims); **"hold N years" maps to the exact calendar anniversary**; commission fields accept **fractional %**
   (`step="any"`).

## Run & verify locally (no build, no test suite)
- Serve the folder and open `index.html`:
  ```bash
  python3 -m http.server 8000   # then open http://localhost:8000/index.html
  ```
  (or just open the file directly). It works **offline** — benchmarks fall back to fixed estimates if
  the endpoint is unreachable.
- **There are no automated tests.** After any logic change, run the manual checks in HANDOFF.md §9.
  Fastest regression check — open the page, enter Purchase `1500000` / Sale `1800000` / 5-year hold,
  the results render live as you fill the story (there is no Calculate button), then paste this into the browser console (expect `reconciles: true`, no errors):
  ```js
  (function(){var r=document.querySelectorAll('#breakdownTable tbody tr'),s=0,g=null;
  function p(t){t=t.trim();if(t==='—'||t==='')return 0;var k=t[0]==='-'?-1:1;return k*Number(t.replace(/[^0-9.]/g,''));}
  r.forEach(function(tr){var d=tr.querySelectorAll('td');if(d.length!==2)return;
  if(tr.classList.contains('grand')){g=p(d[1].textContent);return;}if(tr.classList.contains('total'))return;s+=p(d[1].textContent);});
  console.log({sum:Math.round(s),grand:g,reconciles:Math.abs(s-g)<2});})();
  ```

## First steps for a new agent
1. Skim **`CHANGELOG.md`** for the rounds of testing so far (what's been fixed and why), then read
   `HANDOFF.md` end to end for the depth.
2. Open `index.html`, Calculate with the sample inputs above, confirm it renders and the reconciliation
   check passes — that's your baseline.
3. Make your change; re-run the §9 checks; confirm no console errors and the breakdown still reconciles.

## External dependency
Live benchmarks come from `https://sg-benchmarks.alyoechosys.dev/benchmarks?years=N` (not in this repo;
CORS-open; cached). If unreachable the app **falls back to fixed estimates** automatically. Change the
`BENCHMARK_API` constant (or set it to `''`) to point elsewhere / go fully offline — see HANDOFF.md §3.
