# Tax Liability Model (2026)

A single-file, dependency-free HTML calculator for modeling federal + California
personal income tax liability — built for scenario planning around W-2 income,
RSU vesting, ISO/NSO option exercises, capital gains (short- and long-term), and
charitable giving.

## Usage

Open `index.html` directly in any browser — no build step, no server, no
dependencies. Everything (tax engine, UI, and the marginal-rate chart) is
self-contained in that one file.

## What it covers

- Federal: ordinary brackets, LTCG stacking, AMT (including the OBBBA-driven
  2026 exemption phase-out changes), NIIT, Additional Medicare Tax, the SALT
  cap phase-down, and the new "2/37" itemized deduction limitation.
- California: ordinary brackets (no LTCG preference), Mental Health Services
  Tax, California AMT, and CA's own (non-conforming) itemized deduction rules.
- Multiple income "events" (several W-2 sources, RSU vests, ISO/NSO exercises,
  and capital gains lots with shares/cost basis/sale price) that can each be
  added or removed independently.
- Capital gain/loss netting between short- and long-term buckets, plus the
  $3,000/year ordinary-income capital loss deduction.
- A marginal-rate table and chart showing how the rate on the next dollar of
  each income type changes as you add more of it — including detection of the
  federal AMT exemption phase-out "hump" that OBBBA made significantly harsher
  starting in 2026.
- Save/load: the "Save scenario" / "Load scenario" buttons export and import
  the full input set as a JSON file, so distinct scenarios can be kept as
  separate files and compared. A `scenarios/` folder in this repo is meant
  to hold them (and is gitignored, since scenario files contain your actual
  financial inputs). In Chrome/Edge, point the save/open dialog at
  `scenarios/` once and it'll default there automatically on future
  saves/loads; Firefox/Safari fall back to a normal download instead.

## Sources & important caveats

Federal figures are confirmed 2026 numbers from IRS Rev. Proc. 2025-32 and
IR-2025-103. California figures are confirmed **2025** FTB numbers (2026
brackets hadn't been published by the FTB as of when this was built) with an
optional inflation-adjustment input to approximate 2026. Every source and
simplifying assumption is documented in-app under "Sources & assumptions."

**This is a planning/scenario-comparison tool, not tax advice.** Several
areas are deliberately simplified (no above-the-line adjustments, no mortgage
interest, no QBI, no multi-year AMT credit tracking, etc. — see the in-app
assumptions list). Verify anything load-bearing with a CPA before acting on it.
