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
1. **SSD** (`getSsdInfo`): 4 Jul 2025 regime = 16/12/8/4 over **4 years**; regime chosen by purchase
   date; tiers decided by **calendar anniversary** (not a day count).
2. **BSD** (`getBsdRate`): residential tops at **6% above $3M** — no 7% band.
3. **Net P&L** (`computePnl`): financing cost is **interest only** — do NOT add `loan`/subtract
   `outstandingLoan` (that double-counts principal and overstated profit ~2×). The breakdown table +
   email report must **sum exactly to Net P&L** (cash basis).
4. **LTV** capped 75/45/35 by property count; **duties on max(price, valuation)**; **yields at full
   occupancy**; **ABSD** FTA-national → Singapore-citizen treatment.

## Run & verify locally (no build, no test suite)
- Serve the folder and open `index.html`:
  ```bash
  python3 -m http.server 8000   # then open http://localhost:8000/index.html
  ```
  (or just open the file directly). It works **offline** — benchmarks fall back to fixed estimates if
  the endpoint is unreachable.
- **There are no automated tests.** After any logic change, run the manual checks in HANDOFF.md §9.
  Fastest regression check — open the page, enter Purchase `1500000` / Sale `1800000` / 5-year hold,
  click **Calculate P&L**, then paste this into the browser console (expect `reconciles: true`, no errors):
  ```js
  (function(){var r=document.querySelectorAll('#breakdownTable tbody tr'),s=0,g=null;
  function p(t){t=t.trim();if(t==='—'||t==='')return 0;var k=t[0]==='-'?-1:1;return k*Number(t.replace(/[^0-9.]/g,''));}
  r.forEach(function(tr){var d=tr.querySelectorAll('td');if(d.length!==2)return;
  if(tr.classList.contains('grand')){g=p(d[1].textContent);return;}if(tr.classList.contains('total'))return;s+=p(d[1].textContent);});
  console.log({sum:Math.round(s),grand:g,reconciles:Math.abs(s-g)<2});})();
  ```

## First steps for a new agent
1. Read `HANDOFF.md` end to end.
2. Open `index.html`, Calculate with the sample inputs above, confirm it renders and the reconciliation
   check passes — that's your baseline.
3. Make your change; re-run the §9 checks; confirm no console errors and the breakdown still reconciles.

## External dependency
Live benchmarks come from `https://sg-benchmarks.alyoechosys.dev/benchmarks?years=N` (not in this repo;
CORS-open; cached). If unreachable the app **falls back to fixed estimates** automatically. Change the
`BENCHMARK_API` constant (or set it to `''`) to point elsewhere / go fully offline — see HANDOFF.md §3.
