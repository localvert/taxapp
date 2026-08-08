# CLAUDE.md — 2026 Tax Liability Model

Context for working on this repo. This was originally scoped in a Claude Desktop
project called "Tax Planning"; this file is the durable, Claude Code-facing
version of that context (see `HANDOFF.md` for the original full handoff note).

## What this is

A single-file, dependency-free HTML calculator modeling combined **federal +
California** personal income tax liability for tax year 2026. Built for
scenario planning around W-2 income, RSU vesting, ISO/NSO option exercises,
short/long-term capital gains, and charitable giving — with particular focus
on getting the **AMT exemption phase-out mechanics right**, since that's the
biggest driver of high marginal rates in 2026.

- **File:** `index.html` — the entire app. No build step, no `package.json`,
  no dependencies. Open directly in a browser.
- **Docs:** `README.md` has user-facing usage notes; `HANDOFF.md` has the
  original full project handoff.
- **Do not confuse with:** `code/tax-planner/` — an unrelated, untouched
  default Vite+React+TypeScript scaffold that predates this project. Nothing
  from this app lives there; leave it alone unless separately instructed.

## Why this app exists

Built to support real personal tax planning for an employee at a major tech
company (single filer, CA resident) navigating a year with RSU vesting, ISO
exercises, and a large planned equity sale. Key facts that drove feature
priorities:

- ISO exercises create an AMT preference item with no regular-tax impact —
  the interaction between ISO spread and a large capital gain sale, run
  through the 2026 AMT exemption phase-out, was the original motivating
  question.
- OBBBA (signed 2025) changed 2026 AMT phase-out mechanics materially: the
  phase-out **rate doubled from 25% to 50%**, and the phase-out **threshold
  dropped** back near pre-TCJA levels. This creates a "hump" band where
  marginal rates spike well above the stated 26%/28% AMT rates or 15%/20%
  LTCG rates, because eroding the exemption effectively multiplies the
  taxable base.
- DAF stock donations, MFJ-vs-single comparison, and multi-lot capital gains
  (mixing gains and losses) are real planning levers — hence the app supports
  multiple independent income "events" rather than single aggregate fields.

## Architecture

Everything lives in one `<script>` tag inside `index.html`, in this order:

1. **`CONST`** — all tax parameter tables (brackets, exemptions, thresholds,
   rates) for federal 2026 and CA 2025 (see "Data sources" below).
2. **`calculate(inputs)`** — the pure tax engine. Takes a flat inputs object
   (`w2Income`, `rsuIncome`, `isoPreference`, `nsoOrdinaryIncome`, `stcg`,
   `ltcg`, `cashCharity`, `stockCharity`, `propertyTax`, `filingStatus`,
   `deductionMethod`, `caInflationPct`) and returns a nested result object
   (`income`, `federal`, `ca`, `payrollInfo`, `totals`). No DOM access — fully
   testable in isolation.
3. **`state`** — the UI's source of truth: filing status, deduction method,
   CA inflation %, and four **arrays** of events (`w2Sources`, `rsuVests`,
   `optionEvents`, `capGainEvents`) plus flat charitable/property-tax fields.
4. **`aggregateInputs()`** — collapses `state`'s event arrays into the flat
   shape `calculate()` expects (sums W-2 sources, sums RSU vests, splits
   option events into ISO-preference vs. NSO-ordinary-income by type,
   computes each capital gain event's `shares × (salePrice − costBasis)` and
   splits into ST/LT buckets).
5. **Rendering helpers** (`money`, `pct`, `pct1`, `esc`, `bracketTableHTML`)
   and **dynamic list templates** (`simpleRowHTML`, `optionRowHTML`,
   `capGainRowHTML`) for the "+"-button repeatable input rows.
6. **Chart functions** — `computeMarginalSeries()` generates
   segment-marginal-rate data points by re-running `calculate()` at ~40 steps
   across a user-chosen range; `buildMarginalChartSVG()` hand-rolls an SVG
   line chart (no external charting library, so the file stays fully
   self-contained and works offline); `initChartHover()` wires up a custom
   mousemove tooltip.
7. **`renderOutputs()`** — the main render function, rebuilds the headline
   stats, Federal/CA "tax form waterfall" breakdown, payroll-tax info panel,
   and marginal-rate table+chart on every input change.
8. **Save/load** — `saveConfig()`/`loadConfig()`/`loadConfigFromFile()`
   export/import the full `state` object as a JSON file (deliberately *not*
   localStorage, so scenarios are portable real files, not browser-tied).
   Where the File System Access API is available (Chrome/Edge), both use a
   picker tagged with a shared `SCENARIO_PICKER_ID` so the browser remembers
   the last folder used — pointed at `scenarios/` once, it defaults there on
   every later save/load. Firefox/Safari fall back to a classic
   `<a download>` / `<input type="file">` flow (normal Downloads folder).
   `scenarios/` is gitignored (see `.gitignore`) since scenario files
   contain real financial inputs.
9. Final section wires up all event listeners (list add/remove via event
   delegation, modal open/close, save/load buttons, marginal-table row
   clicks → chart).

## Data sources (as of build time, Aug 2026)

**Federal — confirmed 2026 figures**, sourced to IRS Rev. Proc. 2025-32 and
IR-2025-103:
- Ordinary brackets, standard deduction ($16,100 single / $32,200 MFJ), LTCG
  breakpoints — directly from Rev. Proc. 2025-32.
- AMT exemption ($90,100 single / $140,200 MFJ), phase-out threshold ($500K /
  $1M), **50% phase-out rate** (OBBBA §70107 amending IRC §55(d) — doubled
  from the historical 25%), 26%/28% breakpoint ($244,500).
- NIIT (3.8%, $200K/$250K) and Additional Medicare Tax (0.9%, $200K/$250K) —
  fixed statutory thresholds, not inflation-indexed.
- SALT cap ($40,400, phases down 30¢/$ starting $505,000 MAGI to a $10,000
  floor) — OBBBA amendment to IRC §164(b)(6).
- New "2/37" itemized deduction limitation (IRC §68, replaces repealed
  Pease) for filers above the 37%-bracket threshold.
- New 0.5%-of-AGI charitable floor (IRC §170(b)(1)(L)) — **flagged
  assumption**: this model applies the floor against cash contributions
  before stock; the statute's exact ordering wasn't explicit in publicly
  available guidance at build time.

**California — confirmed 2025 figures, 2026 not yet published by FTB at
build time.** A "CA inflation adjustment %" input lets the user scale 2025
brackets/exemptions/deductions to approximate 2026; defaults to 0 (shows
2025 figures as-is). Notable CA-specific quirks already handled correctly:
- CA's static IRC conformity date (Jan 1, 2015) means it does **not** pick up
  OBBBA changes automatically: CA's AMT phase-out rate stayed at **25%** (not
  50%), CA's cash-charity AGI limit stayed at **50%** (not the federal 60%),
  and CA does **not** apply the new 0.5% charitable floor or the "2/37"
  limitation.
- CA taxes all capital gains as ordinary income (no LTCG preference), has its
  own flat-7% AMT with a milder phase-out, a fixed (non-inflation-indexed)
  $1,000,000 Mental Health Services Tax threshold, and its own un-repealed
  Pease-style high-income itemized deduction limitation.
- CA SDI: 1.3% for 2026, no wage cap (confirmed current, unlike the bracket
  figures).
- **Action item for whoever picks this up:** check ftb.ca.gov for the
  official 2026 Personal Income Tax Booklet (usually posted Nov/Dec) and
  update `CONST.ca.base2025` (rename appropriately) once available.

Every one of these citations is also reproduced in-app under the "ⓘ Sources
& assumptions" button, split into Federal / California / Simplifying-
assumptions sections.

## Feature set (current state)

- Filing status toggle (Single / MFJ), deduction method toggle (Auto / Force
  standard / Force itemized).
- Repeatable input events via "+" buttons: multiple W-2 sources, multiple
  RSU vests, multiple option-exercise events (ISO or NSO per event, via
  dropdown), multiple capital-gain events (ST or LT per event, entered as
  shares × cost basis × sale price, with a live per-event gain readout).
- Capital gain/loss netting between ST and LT buckets per IRC §1211/§1212,
  with the $3,000/year ordinary-income capital loss deduction and
  carryforward flagging.
- Federal and CA sections each render as a "tax form waterfall" (total
  income → AGI → deduction → taxable income → tax → AMT/NIIT/Additional-
  Medicare or MHST/AMT add-ons → total), plus expandable accordions with full
  bracket-by-bracket math for anyone who wants to hand-check it.
- AMT exemption phase-out ("hump") detection with a collapsible warning
  callout explaining the mechanism in plain language when the user's AMTI
  falls in that band.
- Marginal-rate table (Federal / CA / Combined, for W-2, STCG, LTCG, ISO,
  NSO) — click any row to open an SVG line chart plotting Federal rate / CA
  rate / Combined rate (left axis, %) and cumulative extra $ owed (right
  axis, $) against additional income, out to a user-chosen ceiling. Hover
  shows exact values plus total income at that point.
- Save/load scenario as a named JSON file (round-trip tested to be exact),
  defaulting to the `scenarios/` folder in Chrome/Edge once pointed there.
- Payroll-tax info panel (SS/Medicare/SDI) kept clearly separate from the
  income-tax totals, with explicit "why this isn't double-counted"
  explanation (Additional Medicare Tax is the one item that's genuinely part
  of the Federal total and lives only there).

## Explicitly out of scope / simplified

Don't "fix" these without discussing — they're deliberate:

- No above-the-line adjustments (401(k), HSA, etc.) — all income inputs are
  treated as already net of pre-tax deferrals.
- No mortgage interest deduction modeling.
- No QBI deduction, no child tax credit / other credits, no multi-year AMT
  credit (Form 8801 / FTB 3510) carryforward tracking.
- No same-year ISO disqualifying-disposition scenario.
- Federal SALT deduction "tax paid" is proxied by this model's own computed
  CA liability for the same year (plus a property-tax input), not actual
  withholding/estimated payments.

## How to verify changes

There's no test framework set up. The approach used during development:

1. Extract the `<script>...</script>` contents to a standalone `.js` file,
   add `module.exports = { calculate, CONST, ... }`, and run targeted
   `node -e` scenarios, hand-verifying bracket math against the `CONST`
   tables (every dollar figure in this app was cross-checked by hand at
   least once this way).
2. For UI/interaction changes, load the full HTML into `jsdom`
   (`npm install jsdom`), use `runScripts: 'dangerously'`, and simulate
   `input`/`change`/`click` events to drive the app headlessly, then assert
   on `document.getElementById(...)` contents. This caught a real bug once
   (the "+Add capital gain event" button was pushing the old data shape
   after a field-rename) — worth keeping this kind of check for any change
   to the dynamic list add/remove logic.
3. There is no visual regression testing; changes to the SVG chart rendering
   should be sanity-checked in an actual browser.

## Ideas discussed but not yet built

No commitment either way — just visibility:

- A 5th chart series showing the *cumulative average* rate (total extra $ ÷
  x), to sit alongside the four existing marginal-rate lines and make the
  marginal-vs-average distinction visually explicit rather than requiring
  mental math.

## This is a planning tool, not tax advice

Verify anything load-bearing with a CPA before acting on it. Keep the
in-app "Sources & assumptions" panel in sync with any change to `CONST` or
to the calculation logic.
