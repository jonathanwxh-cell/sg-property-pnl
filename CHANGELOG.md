# Changelog & QA log

Every change made to this calculator, newest first, framed as the **rounds of testing** that drove
them. Each round = a QA pass that found issues, fixed them, and re-verified in a real browser. For the
deep technical detail behind a round, follow its **HANDOFF.md §** link; the correctness invariants that
must never regress are consolidated in **[HANDOFF.md §4](./HANDOFF.md)**.

> **How this was tested.** There is no automated test suite (it's one self-contained HTML file). Every
> round is verified by driving the real page in a headless browser: fill the story inputs, read the
> rendered DOM/values, and run the reconciliation check (the cost-breakdown table must sum **exactly**
> to Net P&L). From Round 5 on, fixes are exercised with **real click/type events**, not by calling JS
> functions directly — an earlier round's "passing" direct-call tests had masked bugs that only appear
> under real interaction (see Round 5).

## Rounds at a glance

| Round | Date | Focus | Outcome |
|------:|------|-------|---------|
| **5** | 2026‑07‑02/03 | Real‑interaction re‑fixes + mobile/a11y polish | 4 re‑reported issues fixed for real + native `<h2>`/touch sliders/back‑pill |
| **4** | 2026‑07‑02 | Input‑edge fixes | negatives block · occupancy>100 · field append · breakdown CTA |
| **3** | 2026‑07‑02 | Robustness / tax / a11y (9 bugs) | `1e308` hang · SC+foreigner ABSD refund · invalid‑input blocking · a11y |
| **2b** | 2026‑07‑02 | ABSD spouse remission (Section A) | first‑home, ≥1 SC spouse → 0% at stamping (IRAS‑verified) |
| **2** | 2026‑07‑02 | Second correctness pass (8 findings) | SSD anniversary boundary · pre‑2018 BSD · LTV tenure cap · input validation |
| **1b** | 2026‑07‑02 | Scenario analysis feature | sliders persist · downside break‑even · sensitivity table |
| **1** | 2026‑07‑02 | First correctness pass (8 findings) | ABSD 16 Dec 2021 · remission timing · occupancy yield · valuation break‑even · SSD cliff |
| **0b** | 2026‑07‑02 | UI redesign | form → **editable‑story** narrative with live results |
| **0** | 2026‑07‑02 | Baseline import + handoff docs | fixed & modernized calculator brought into this repo |

---

## Round 5 — Real-interaction re-fixes + mobile/a11y polish
`1fa6411` · 2026‑07‑02/03 · detail: **HANDOFF.md §18** (+ §4.8 invariants)

The owner re-reported four earlier fixes as *still broken in real use*. They were — or were only
half-fixed. The root lesson: the previous round had verified some fixes by calling JS functions
directly, which masked bugs that only surface under real clicks/keystrokes. This round re-tested
everything with real browser events on **both** localhost and the live reference URL.

- **Occupancy now visibly clamps to 0–100 in the field** (`150 → 100`, `−20 → 0`). Round 4 only
  *warned* while the calc silently used a capped 100 — so the field could show `150` but compute/export
  `100`. Removed the now-moot warning.
- **`#bsdHint` refreshes on purchase-date change.** It was already date-aware but wasn't wired to the
  date picker, so changing the date left the top chip stale (`2018‑02‑19` showed `S$69,600` instead of
  the pre-2018 `S$54,600`). Verified it flips `S$69,600 ↔ S$54,600` on the date change.
- **Sale-field append fixed for real (clear-on-focus).** Round 4's *select-on-focus* did not survive a
  real click's `mouseup` or `formatMoney`'s per-keystroke caret rewrites — typing gave
  `1,800,000,700,000` (append) or `0` (mid-type corruption). Replaced with **clear the untouched example
  on focus** (restore on an un-typed blur). Typing `700000` now yields `700,000`, confirmed with real
  keystrokes.
- **"See the full breakdown" confirmed expanding** via a real hit-tested click (nothing overlays the
  button): `#breakdownBody` none→block, 19 rows, scrolls into view. The "still broken" report was a
  **stale cached/deployed copy** — the live URL now serves the working fix.
- **Mobile / a11y:** section labels are native `<h2 class="section-title">` (×10) under the single
  `<h1>` (CSS reset keeps the look); range sliders use a 26px thumb / 8px track for touch; the hub
  wrapper's floating back-control became a 40px "‹ All apps" pill (that control lives in the deploy
  wrapper repo, not here).

## Round 4 — Input-edge fixes
`3cd0a0d` · 2026‑07‑02 · detail: **HANDOFF.md §17**

- Occupancy > 100% warned instead of silently computing (superseded by Round 5's visible clamp).
- **Negative amounts block the calc** (`_negField` in the invalid guard) — a negative sale no longer
  computes as a S$0 sale; the break-even cushion copy guards its `/ sp` divide so a S$0 sale can't print
  `+Infinity`.
- Prefilled fields select-on-focus (superseded by Round 5's clear-on-focus).
- **"See the full breakdown"** CTA expands the collapsed cost-breakdown table + scrolls (was scroll-only).

## Round 3 — Robustness, tax & accessibility (9 bugs)
`8674a43` · 2026‑07‑02 · detail: **HANDOFF.md §16**

- **`1e308`/absurd fee no longer hangs** — `num()` clamps to `[0, 1e12]`, tenure ≤ 35 yr, amortisation
  loop hard-capped.
- **SC + foreign-spouse ABSD refund path** implemented (model the couple at the higher-rate profile).
- **Invalid `badInput` blocks** the calc; results hidden, `lastReport` nulled, chart destroyed, stale
  DOM blanked → nothing stale is shown or exported.
- Negative money shown (leading `−`), not silently flipped positive.
- **S$0 sale is valid** (total loss / gift); **underwater sale** shows the cash top-up needed.
- Scenario LTV re-syncs to the main LTV after it changes.
- A11y: narrow `#srSummary` live region instead of a broad `aria-live`; heading semantics.

## Round 2b — ABSD spouse remission, Section A
`7130536` · 2026‑07‑02 · detail: **HANDOFF.md §15**

First matrimonial home bought together, neither owns other property, at least one **Singapore-Citizen**
spouse (SC + foreigner qualifies) → **full remission at stamping, 0% ABSD, no bridging**. Rule verified
against IRAS before implementing.

## Round 2 — Second correctness pass (8 findings)
`7ae2aab` · 2026‑07‑02 · detail: **HANDOFF.md §14**

- **SSD anniversary boundary** made exclusive (strict `<`): a sale *on* the Nth anniversary counts as
  "more than N years" (was overcharging by a day).
- **Pre-20 Feb 2018 BSD** regime added (3% top) — `S$2M` ⇒ `54,600 / 64,600 / 69,600` across the three
  regimes.
- **MAS LTV reduced to 55/25/15 when loan tenure > 30 years** (was disclosed but not enforced).
- Invalid numeric fields no longer drive the calc.

## Round 1b — Scenario analysis + sensitivity
`a8fe3a2` · 2026‑07‑02 · detail: **HANDOFF.md §13**

Scenario sliders (LTV, rate delta, exit price delta) persist across base-case edits; added a **downside
break-even** and a **sensitivity table**. Core math refactored into a pure `computePnl(inputs)` shared
by the base render and the scenario panel.

## Round 1 — First correctness pass (8 findings)
`8d38292` · 2026‑07‑02 · detail: **HANDOFF.md §12**

- **ABSD 16 Dec 2021 regime** added; remission modelled as **pay-now / refund-later** (Section B).
- **Net rental yield made occupancy-aware** (gross stays at full occupancy).
- Break-even corrected when SSD is fixed on a higher sale **valuation**.
- Negative-rate / zero-tenure guards; **SSD day-level cliff** surfaced to the user.
- **Net P&L = interest only** — no principal add-back — so the breakdown reconciles exactly (this was
  the flagship ~2× profit-overstatement bug).

## Round 0b — UI redesign (editable story)
`c2cb969`, `465d7f3` · 2026‑07‑02

Rebuilt the form-heavy calculator as an **editable plain-English "story"** (narrative inputs with a
live result and visual), then a two-column sticky-results layout — "less like a calculator."

## Round 0 — Baseline import + handoff docs
`cad480e`, `5172bb1`, `095299b` · 2026‑07‑02

The fixed & modernized calculator was brought into this repo (`jonathanwxh-cell/sg-property-pnl`) as the
canonical single `index.html`, with `AGENTS.md` (agent brief), `HANDOFF.md` (depth), and a cited
known-good live reference. Original project: [`AngsumalinX/sg-property-pnl`](https://github.com/AngsumalinX/sg-property-pnl).
