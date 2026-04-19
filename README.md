# CompoundX — Advanced Compound Interest Calculator

A fancy, 2026‑styled compound interest calculator with variable deposit phases, inflation & tax adjustments, lump sums, systematic withdrawal (SWP), goal‑seek, scenario comparison, CSV export, and shareable URLs.

**Single static HTML file — no build, no backend.** Works on any static host, including GitHub Pages.

## Files

| File | Purpose |
|---|---|
| `index.html` | The actual working app (this is what GitHub Pages serves) |
| `mockup.html` | The original visual mockup (optional, for reference) |
| `BUILD_PLAN.md` | Longer‑term architecture plan if you ever want to split into a React app |
| `README.md` | This file |

## Deploy to GitHub Pages (5 minutes)

### Step 1 — Put the files in a repo

```bash
# from this folder
git init
git add index.html mockup.html BUILD_PLAN.md README.md
git commit -m "Initial compound calculator"
git branch -M main
# create a public repo on github.com first (e.g. "compound-calculator"), then:
git remote add origin https://github.com/<YOUR_USERNAME>/compound-calculator.git
git push -u origin main
```

### Step 2 — Turn on Pages

1. On GitHub, go to your repo → **Settings** → **Pages** (in the left sidebar).
2. Under **Source**, pick **Deploy from a branch**.
3. Branch: **main**, folder: **/ (root)**. Click **Save**.
4. Wait ~30 seconds. GitHub will show a green banner with your live URL, e.g. `https://<YOUR_USERNAME>.github.io/compound-calculator/`.

That's it — your calculator is live. Every `git push` to `main` auto‑redeploys.

### Optional — custom domain

In **Settings → Pages → Custom domain**, add your domain and create a `CNAME` DNS record pointing at `<YOUR_USERNAME>.github.io`. GitHub handles HTTPS automatically.

## Features

- **Variable deposit phases** — add/remove phases, each with its own monthly deposit, return rate, and yearly escalation.
- **Compound frequencies** — monthly, daily, quarterly, annual.
- **Lump sums / bonuses** — one‑time deposits at any year.
- **Inflation adjustment** — nominal + real (today's money) values shown.
- **Tax on gains** — end‑of‑term capital gains, or annual drag.
- **Goal‑seek** — solve for the phase‑1 monthly deposit that hits a target.
- **SWP (drawdown)** — simulate monthly withdrawals after accumulation; see depletion year or surplus.
- **Save & compare scenarios** — persisted in `localStorage`, overlaid on the comparison chart.
- **Share link** — scenario encoded in a URL hash (LZ‑compressed), one‑click copy.
- **CSV export** — full yearly breakdown.
- **PDF export** — uses the browser's print dialog (Ctrl/Cmd+P → "Save as PDF").
- **Keyboard friendly** — all inputs native; no layout thrash.

## Tech stack

- Pure HTML/CSS/JS — no framework.
- [Chart.js 4](https://www.chartjs.org/) via CDN for charts.
- [lz‑string](https://github.com/pieroxy/lz-string) via CDN for URL‑safe scenario compression.
- Google Fonts: Plus Jakarta Sans + JetBrains Mono.

## Local development

Open `index.html` directly in any browser — no server required. All state is in `localStorage` under keys prefixed `compoundx:v1:`.

## License

MIT — do whatever you like.
