# CompoundX — Advanced Compound Interest Calculator

**Live site:** https://mohammadparyal.github.io/compound-calculator/

A 2026‑styled compound interest calculator with variable deposit phases (weekly / biweekly / monthly / daily cadence), inflation & tax adjustments, lump sums, three systematic‑withdrawal strategies (fixed, % rule, custom phases), goal‑seek, saved portfolios with overwrite protection, CSV export, and shareable URLs.

Single static HTML file — no build, no backend. Works on any static host.

## Files

| File | Purpose |
|---|---|
| `index.html` | The app (this is what GitHub Pages serves) |
| `mockup.html` | Original visual mockup, kept for reference |
| `README.md` | This file |

## Features

- Variable deposit phases with per‑phase cadence, return, and escalation.
- Compound frequencies: monthly, daily, quarterly, annual.
- Lump sums / bonuses at any year.
- Inflation adjustment — nominal and real (today's money) side‑by‑side.
- Tax on gains — end‑of‑term CGT or annual drag.
- Goal‑seek — solve for the monthly deposit that hits a target.
- SWP drawdown — fixed amount, % rule (of initial / of balance), or custom retirement phases (go‑go / slow‑go / no‑go).
- Saved portfolios — live badge, explicit overwrite prompt on name collision, no silent auto‑save while editing.
- Share link — scenario encoded in a URL hash (LZ‑compressed).
- CSV export and monthly/yearly breakdown tables.
- Multi‑currency (USD, EUR, GBP, CAD, AUD, AED, SAR, INR, PKR) with Lakh/Crore short forms.

## Tech stack

- Pure HTML/CSS/JS — no framework.
- [Chart.js 4](https://www.chartjs.org/) via CDN.
- [lz‑string](https://github.com/pieroxy/lz-string) via CDN.
- Google Fonts: Plus Jakarta Sans + JetBrains Mono.

## Local development

Open `index.html` directly in any browser — no server required. State is in `localStorage` under keys prefixed `compoundx:v1:`.

## License

MIT — do whatever you like.
