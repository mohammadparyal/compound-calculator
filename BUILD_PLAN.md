# CompoundX — Build Plan

_Advanced compound‑interest calculator, deployable to GitHub Pages. Light glassmorphism + soft aurora + neon accents. 2026 edition._

This document is the blueprint that turns `mockup.html` into a shippable app.

---

## 1. Goals & non‑goals

**Goals**

- A single‑page web app, deployable as a static site on GitHub Pages (no backend, no server cost).
- Model **variable deposit phases** over the investment horizon (e.g. years 0–5 $2,000/mo, 5–15 $1,000/mo, 15–25 $0).
- Support the advanced features the user chose:
  - Inflation‑adjusted real returns
  - Tax on gains (annual drag _or_ end‑of‑term CGT)
  - Lump‑sum injections at arbitrary years/months
  - Withdrawal phase / **SWP** (Systematic Withdrawal Plan) simulation
  - Variable return rate per phase
  - Goal‑seek (solve for monthly deposit given a target)
  - Save/load scenarios + shareable URL
  - CSV + PDF report export
- Four chart types: stacked‑area growth, donut + yearly bars combo, scenario comparison line, phase timeline strip.
- Premium 2026 UI: glassmorphism cards, dark gradient background, neon cyan/purple/pink accents, smooth animations.

**Non‑goals (v1)**

- No user accounts, no cloud sync. Scenarios live in `localStorage` + optional URL hash.
- No live market data feeds.
- No multi‑currency conversion (symbol is configurable, math is currency‑agnostic).
- No mobile native app — responsive web only.

---

## 2. Tech stack decision

| Concern | Choice | Why |
|---|---|---|
| Framework | **Vite + React + TypeScript** | Fast dev server, static build works perfectly on Pages, type‑safe math engine. |
| Styling | **Tailwind CSS v4** + small CSS module for glassmorphism tokens | Keeps the neon/glass design tokens centralized; utility classes for speed. |
| Charts | **Chart.js 4** via `react-chartjs-2` | Good performance, easy gradient fills, plays nicely with the neon palette. Apache‑2.0. |
| State | **Zustand** | Tiny, zero‑boilerplate; perfect for scenario + UI state without Redux weight. |
| Math engine | Plain TS module (no deps) | Deterministic, unit‑testable in isolation. |
| PDF export | **jsPDF** + `html2canvas` | Client‑side only, no service calls. |
| CSV export | Hand‑rolled (no library needed) | ~20 lines, zero dep. |
| URL sharing | `LZString.compressToEncodedURIComponent` | Fits a full scenario in a short `#` hash. |
| Tests | **Vitest** + `@testing-library/react` | Share config with Vite. |
| Lint/format | ESLint + Prettier | Standard. |
| CI / deploy | **GitHub Actions → GitHub Pages** | Free, zero‑config with the official `deploy-pages` action. |

Bundle budget: **< 200 KB gzipped** initial JS.

---

## 3. File & folder layout

```
compound-calculator/
├── index.html
├── package.json
├── vite.config.ts
├── tsconfig.json
├── tailwind.config.ts
├── public/
│   └── favicon.svg
├── .github/workflows/deploy.yml
└── src/
    ├── main.tsx
    ├── App.tsx
    ├── styles/
    │   ├── globals.css              # tokens, aurora bg, glass recipe
    │   └── chart-theme.ts           # Chart.js defaults
    ├── store/
    │   ├── scenarioStore.ts         # Zustand: current scenario + actions
    │   └── scenariosStore.ts        # saved scenarios collection
    ├── engine/
    │   ├── types.ts
    │   ├── compound.ts              # core simulator
    │   ├── phases.ts                # phase lookup helpers
    │   ├── inflation.ts             # real-value transforms
    │   ├── tax.ts                   # annual drag / CGT modes
    │   ├── swp.ts                   # withdrawal simulator
    │   ├── goalSeek.ts              # bisection solver
    │   └── __tests__/*.test.ts
    ├── features/
    │   ├── Nav/TopBar.tsx
    │   ├── Hero/HeroStats.tsx
    │   ├── Inputs/
    │   │   ├── BaseParams.tsx
    │   │   ├── PhaseList.tsx
    │   │   ├── PhaseEditor.tsx
    │   │   ├── LumpSums.tsx
    │   │   ├── Toggles.tsx
    │   │   └── WithdrawSetup.tsx
    │   ├── Charts/
    │   │   ├── GrowthArea.tsx
    │   │   ├── CompositionDonut.tsx
    │   │   ├── YearlyBars.tsx
    │   │   ├── ScenarioCompare.tsx
    │   │   └── PhaseTimeline.tsx
    │   ├── Breakdown/YearTable.tsx
    │   ├── Scenarios/ScenarioList.tsx
    │   └── Export/{CsvButton,PdfButton,ShareLink}.tsx
    ├── lib/
    │   ├── format.ts                # currency, %, compact
    │   ├── url.ts                   # hash encode/decode
    │   └── storage.ts               # localStorage wrapper
    └── types.ts
```

---

## 4. Data model

```ts
type CompoundFreq = 'daily' | 'monthly' | 'quarterly' | 'annual';

interface Phase {
  id: string;
  startYear: number;       // inclusive, 0-based
  endYear: number;         // exclusive
  monthlyDeposit: number;  // base currency
  depositGrowthPct: number;// annual escalation (e.g., raise by 2%/yr)
  annualReturnPct: number; // can override global rate
}

interface LumpSum {
  id: string;
  atYear: number;     // e.g., 3
  atMonth: number;    // 1..12
  amount: number;     // positive = injection, negative = withdrawal
  label?: string;
}

interface WithdrawalPlan {   // SWP
  enabled: boolean;
  startYear: number;         // typically = accumulation duration
  monthlyAmount: number;
  escalatePct: number;       // inflate withdrawals yearly
  durationYears: number;
  postRetireReturnPct: number;
}

interface Scenario {
  id: string;
  name: string;
  color: string;             // neon token
  startingBalance: number;
  durationYears: number;
  compoundFreq: CompoundFreq;
  defaultReturnPct: number;
  phases: Phase[];
  lumpSums: LumpSum[];
  inflationPct: number;          // 0 = off
  taxMode: 'none' | 'annualDrag' | 'endCGT';
  taxRatePct: number;
  withdrawal: WithdrawalPlan;
  goalSeek?: { targetFinalBalance: number };
  createdAt: string;
}
```

---

## 5. Calculation engine

All math lives in `src/engine/*` as pure functions — no React imports. Simulate per‑period (default monthly) for correctness with mid‑phase transitions and lump sums.

**Core loop (pseudo)**

```
bal = startingBalance
series = []
for month m in 0..totalMonths:
  year = floor(m/12)
  phase = findPhase(year, month_in_year)
  r = phase?.annualReturnPct ?? defaultReturnPct
  periodRate = r / periodsPerYear(compoundFreq)

  bal *= (1 + periodRate)                          // grow first
  if phase: bal += phase.monthlyDeposit * growth(year)
  for lump in lumpsAt(m): bal += lump.amount

  if taxMode == 'annualDrag' && endOfYear(m):
      bal -= yearGain(bal) * taxRate

  series.push({ month: m, balance: bal, contribToDate, ... })

if taxMode == 'endCGT':
    finalBal -= (finalBal - totalContribs) * taxRate
```

Derived outputs per month: balance, cumulative contributions, interest (= balance − contribs), real value (= balance / (1+inflation)^years), after‑tax value.

**Withdrawal (SWP)**
After accumulation, a second loop draws `withdrawal.monthlyAmount` (with yearly escalation), applies `postRetireReturnPct`, and reports the depletion year and inflation‑adjusted income.

**Goal‑seek**
Bisection on `phases[0].monthlyDeposit` (or a phase the user picks) until `|finalBalance − target| < $1`. Converges in ~30 iterations.

**Numerical safeguards**

- Clamp rates to [−50%, +50%] p.a.
- Cap duration at 80 years (UI), 120 years (engine).
- Detect negative balance in SWP and return `depletedAtMonth` instead of throwing.

**Performance budget**

- 25 years × 12 months = 300 periods → <1 ms per scenario. We can run dozens of goal‑seek iterations without debounce.

---

## 6. UI / UX notes

The look is already established in `mockup.html`. Port tokens into `globals.css` as CSS custom properties and Tailwind `theme.extend`:

```
--bg-deep: #f4f5fb   /* soft cream/lavender page */
--bg-mid:  #ffffff   /* card interiors */
--glass:   rgba(255,255,255,0.72)
--border:  rgba(15,23,42,0.08)
--text:    #0f172a
--text-dim:#516079
--cyan:    #0891b2
--purple:  #7c3aed
--pink:    #db2777
--green:   #059669
```

Design rules to keep it premium:

- Every card uses `backdrop-filter: blur(18px) saturate(160%)` over a pastel aurora gradient (soft radial blobs animated on a slow drift).
- Primary CTA uses the cyan→purple gradient with a soft shadow (`box-shadow: 0 6px 18px rgba(124,58,237,.35)`).
- All charts use `ctx.createLinearGradient` for fills; stroke widths 2–2.5 px; `tension: 0.35`; no point markers.
- Motion: 200–250 ms ease; avoid bouncy springs. Respect `prefers-reduced-motion`.
- Typography: _Plus Jakarta Sans_ 400–800, _JetBrains Mono_ for numbers in tables.
- Accessibility: WCAG AA contrast on all text (`--text` on `--bg-deep` passes); focus rings use cyan; all toggles have `role="switch"`.
- Mobile: single column ≤ 1100 px; inputs panel becomes a bottom sheet with a drag handle.

---

## 7. Chart specs

1. **Stacked area — Growth over time** (`Charts/GrowthArea.tsx`)
   - X = years (0..duration), Y = $.
   - Series: Contributions (cyan fill), Interest stacked above (purple→pink gradient), Real value as dashed white line overlay.
   - Markers: lump‑sum events as amber dots with callout labels.

2. **Donut — Final composition** (`CompositionDonut.tsx`)
   - Two slices: total contributions vs total interest. Center text shows final balance.

3. **Yearly bars — Deposits vs interest earned** (`YearlyBars.tsx`)
   - Stacked bars per year. Use it to visually spot the "crossover year" where annual interest > annual deposits (highlighted with a vertical neon line).

4. **Scenario comparison** (`ScenarioCompare.tsx`)
   - Line chart overlaying up to 4 saved scenarios. Click a pill to solo/mute a scenario. Cyan = current.

5. **Phase timeline strip** (`PhaseTimeline.tsx`)
   - Custom SVG, not Chart.js. Draggable phase boundaries with keyboard support. Lump sums render as stars pinned on the track.

---

## 8. State & persistence

- `scenarioStore` holds the current working scenario; subscribed by every input and chart.
- `scenariosStore` holds an array of saved scenarios (`localStorage` key `compoundx:v1:scenarios`).
- A computed selector `useDerived()` runs the engine and memoizes per scenario hash. This keeps re‑renders cheap.
- **Share link:** serialize scenario → JSON → LZ‑string → base64url → `location.hash`. On load, if a hash is present, hydrate into the working scenario.

---

## 9. Export

- **CSV** — yearly breakdown (matches the table in the mockup) + a meta block with scenario inputs.
- **PDF** — one‑page report: title, parameters summary, 3 charts (growth, donut, timeline) rendered via `html2canvas` of the dashboard at 2× scale, yearly table. Brand footer with the GitHub Pages URL.

---

## 10. Deployment to GitHub Pages

`.github/workflows/deploy.yml`:

```yaml
name: Deploy
on: { push: { branches: [main] } }
permissions: { contents: read, pages: write, id-token: write }
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-pages-artifact@v3
        with: { path: dist }
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: { name: github-pages, url: ${{ steps.dep.outputs.page_url }} }
    steps: [{ id: dep, uses: actions/deploy-pages@v4 }]
```

In `vite.config.ts`:

```ts
export default defineConfig({
  base: '/compound-calculator/',   // repo name
  build: { outDir: 'dist', target: 'es2020' }
});
```

In repo settings: **Pages → Source → GitHub Actions**. First push publishes to `https://<user>.github.io/compound-calculator/`.

---

## 11. Testing strategy

- **Engine unit tests** (Vitest): 30–40 cases covering compound frequencies, zero‑rate, single‑phase, multi‑phase with rate change, lump sums, inflation, each tax mode, SWP depletion edge case, goal‑seek convergence.
- **Snapshot tests** for the derived yearly breakdown of a canonical scenario (the 25‑year phased plan used in the mockup).
- **Component tests** for phase editor (add/remove/reorder, validation: no overlaps, start<end, endYear ≤ duration).
- **Cross‑check**: pick 5 inputs, compare outputs against _thecalculatorsite.com_ within 0.5%.
- **Visual regression** (Playwright, optional): screenshot the dashboard at desktop + mobile.

---

## 12. Roadmap / milestones

| # | Milestone | Scope | ETA |
|---|---|---|---|
| M0 | Scaffold | Vite+React+TS, Tailwind tokens, base layout, aurora bg, deploy pipeline working with a "Hello" page | 0.5 day |
| M1 | Engine v1 | `compound.ts`, `phases.ts` + tests; flat+phased both match the reference site | 1 day |
| M2 | Inputs UI | Base params, phase list/editor, lump sums. Live recompute. Numbers match hand calcs | 1 day |
| M3 | Charts | Growth area, donut, yearly bars, phase timeline strip | 1 day |
| M4 | Advanced | Inflation, tax modes, SWP, goal‑seek | 1 day |
| M5 | Scenarios | Save/load, scenario comparison chart, share URL | 0.5 day |
| M6 | Exports | CSV + PDF | 0.5 day |
| M7 | Polish | Mobile bottom sheet, reduced‑motion, a11y, copy pass | 0.5 day |
| M8 | Launch | Visual regression pass, README with screenshots, Pages live | 0.5 day |

Total: **~5–6 focused days.**

---

## 13. Stretch ideas (v2+)

- Monte‑Carlo mode: sample return rates from a normal/bootstrap distribution; show 10th/50th/90th percentile bands.
- Import a real portfolio CSV (date, contrib, value) and project forward from today.
- Dual‑currency view with FX snapshot.
- Retirement FIRE dashboard: safe withdrawal rate calc, 4% rule visual.
- Dark/light theme toggle (default dark for the 2026 look).
- PWA install + offline.

---

## 14. Decisions still open

1. **Currency.** Default `$`? The example on thecalculatorsite.com is GBP — prefer detecting locale or letting the user pick the symbol only?
2. **Withdrawal semantics.** SWP as a _phase type_ (negative deposit) vs a separate dedicated section? Mockup shows a dedicated "Drawdown / SWP" tab; the engine treats it as a second simulation so both views co‑exist.
3. **Phase rate vs global rate.** Allow override per phase _and_ a global default (current plan) — confirm this is the intended UX, or collapse to phase‑only for simplicity.
4. **Tax mode default.** Start with `endCGT` at 15%, or `none`?

These get decided the moment we move from mockup → M1.

---

_Mockup to inspect:_ `mockup.html` — open in a browser.
