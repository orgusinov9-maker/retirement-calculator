# Retirement Calculator — Project Notes

Single self-contained HTML file (`index.html`) — Hebrew RTL UI, no build step, no backend.
All data lives only in the browser's `localStorage` (key `retirementCalcStateV1`) — nothing is ever sent to a server.

## Live site
- **URL:** https://orgusinov9-maker.github.io/retirement-calculator/
- **GitHub repo:** https://github.com/orgusinov9-maker/retirement-calculator (public — required for free GitHub Pages; no secrets in the code, so this is fine)
- **GitHub account used:** `orgusinov9-maker` (authenticated via `gh` CLI, installed at `C:\Program Files\GitHub CLI\gh.exe`, not yet on PATH in new shells — call it by full path or re-open shell)
- Local git identity used for commits: name "Or", email `orgusinov9@gmail.com`
- HTTPS enforced automatically by GitHub Pages.

## How to deploy an update
```
cd C:\Users\Or\retirement-calculator
git add index.html
git commit -m "..."
git push
```
Site updates live within ~1 minute of push (GitHub Pages auto-builds on push to `master`).

## Feature history (in order built)
1. **Initial build** — took a base HTML calculator and added: persistent state (localStorage, no reset on reload), multiple funds support, developer-facing controls for all cash-flow amounts, multiple real-estate assets, per-asset tax calculations, mortgage amortization schedule, hover "i" tooltip explanations for every term (via `data-tip-key` + `TIPS` dictionary + `tipify()`).
2. **Bug fix round 1** (user-reported):
   - Fund delete button didn't work (`renderFunds()` was missing the `[data-del]` click listener that `renderProperties()` already had) — fixed.
   - Removed the raw "JSON mode" developer editor from the Advanced tab entirely (textarea + apply/refresh buttons + CSS) per user request.
   - Self-review turned up and fixed: `addProperty` was hardcoding sale/rent tax defaults instead of reading `state.advanced.defaultSaleTax`/`defaultRentTax`; mortgage schedule preview went stale after editing mortgage fields after opening it (added `refreshSched()`); cleaned up dead/confusing code in `model()` (no functional change).
3. **"Add the suggestions" round** — 4 improvements:
   - Empty-state messages when assets/funds/properties tables are empty.
   - Soft validation warnings box (`computeWarnings()`) — flags e.g. retirement age ≤ current age, withdrawal rate outside 0–15%, extreme inflation, negative net return, negative property equity at retirement.
   - Print/export button (`window.print()` + `@media print` CSS) to export/print results.
   - Mortgage schedules that run past retirement age now collapse the post-retirement rows into a `<details>` toggle instead of always showing the full table.
   - This round exposed and required fixing a real bug: `assetData()` crashed (`Cannot set properties of null`) when the assets table was empty, because it assumed every table row has a `.asset-weight` cell — the empty-state placeholder row doesn't. This silently froze the rest of `calculate()` (KPIs/warnings/charts/schedules) whenever assets were empty. Fixed with a null-guard.
4. **Deployment to GitHub Pages** (this session):
   - Installed GitHub CLI via winget, authenticated via `gh auth login --web` (user completed the device-code flow in their own browser).
   - Created the public repo and pushed, enabled GitHub Pages via `gh api POST .../pages`.
   - Security hardening added: `<meta name="robots" content="noindex,nofollow">` (keeps it out of Google search — see "Open question" below) and a strict CSP meta tag (`default-src 'none'; style-src 'unsafe-inline'; script-src 'unsafe-inline'; img-src data:; base-uri 'none'; form-action 'none'`) as defense-in-depth, even though the app makes zero external requests already.
5. **Mobile/UX polish round** (user-reported, this session):
   - Charts (`#capitalChart`, `#allocationChart`) were getting squished horizontally on narrow/mobile viewports because their container (`.chart`) had a fixed `height:250px` while width was fluid, and the SVGs used `preserveAspectRatio="none"`. Fixed by giving each chart's container `aspect-ratio` matching its own SVG `viewBox` (720/250 and 420/250 respectively) instead of a fixed height — verified via Playwright at 320px/375px/1280px viewports, exact ratio held with 0% deviation.
   - The "i" info tooltips (`.info` → child `.tip`) were getting clipped at the screen edge because they were `position:absolute`, always horizontally centered under the icon at a fixed 230px width — this overflowed off-screen for icons near an edge, on both mobile and desktop. Fixed by switching `.tip` to `position:fixed` with a JS helper (`positionTip()`, wired to delegated `mouseover`/`focusin` on `.info`) that measures the icon's position and clamps the tooltip within `[8px, window.innerWidth - width - 8px]`, flipping above/below the icon depending on available space. Verified across 7+ icons in different tabs/viewports — all stayed fully on-screen.
   - Both fixes verified with zero console/page errors via background Playwright test agents before pushing.

6. **Per-asset return/tax + allocation UX rebuild** (this session):
   - The core model previously blended the whole portfolio into one weighted-average return and one global tax rate (`state.portfolioTaxRate`) before compounding — so per-asset return/tax fields existed for weighting/display only, not for actual growth math. Rebuilt `model()` to compound and tax each asset in `state.assets` individually (monthly contribution split by each asset's current value-weight), then sum. A `portfolioReturn` override (used by the goal-solver's "required weighted return" mode) is now translated internally into a uniform percentage-point shift applied to every asset's own rate, preserving the external solve-for-return UX with the new per-asset math underneath.
   - Added a per-asset `taxRate` field (each asset now has its own capital-gains tax rate, like funds already did). Removed the single global `state.portfolioTaxRate` field; replaced with `state.advanced.defaultAssetTax` (used to prefill new assets), matching the existing `defaultRentTax`/`defaultSaleTax` pattern for properties. Migration on load: old saves get `defaultAssetTax` and per-asset `taxRate` backfilled from the legacy `portfolioTaxRate`, then the legacy field is deleted — verified via a simulated old-format `localStorage` state that migrated cleanly with no console errors.
   - Converted the asset allocation editor from a compact `<table>` (no labels/tooltips, was the site's one interaction pattern inconsistent with funds/properties) into a card list matching the funds/properties design — each asset is now a card with labeled fields (value, return, tax rate) and info tooltips, plus a live per-card weight readout. This also fixed a latent fragility: the old table matched each row to its asset by DOM position (the same bug class already fixed once for the empty-state case, see round 2) — the new cards match by `data-id` instead.
   - Added the results breakdown listing each asset individually (name, net value, tax pill) instead of one aggregate "תיק השקעות" line, and added a warning for any individual asset whose own net return (after cost drag) is negative.
   - Verified via chrome-devtools: per-asset tax edits move the total tax by exactly that asset's tax amount; add/delete asset cards work and weights recompute; the "required weighted return" solver still converges to the goal under the new shift-based model; mobile viewport (375px) renders the new cards correctly with on-screen tooltips; old-format localStorage migrates without errors; `node --check` on the extracted script passes.

7. **Advanced-tab removal, zeroed starting data, 3% default withdrawal, asset preset picker** (this session):
   - Removed the "מתקדם" (Advanced) tab entirely (tab button + `<section id="advanced">` + its `bindSimple` call) per user request — it exposed developer-only settings (compounding frequency, contribution timing, default tax rates, rounding) that aren't meant for end users. The underlying `state.advanced` object and its fields (`compoundingPerYear`, `contribTiming`, `defaultRentTax`, `defaultSaleTax`, `defaultAssetTax`, `roundDecimals`) still exist and are used internally by `model()`/`futureValueWithComp()`/`addProperty`/asset-add logic — only the UI to edit them was removed; they're now fixed at their `defaultState()` values (12, 'end', 10, 25, 25, 2).
   - Default annual withdrawal rate (`state.basic.withdrawal`) changed from 4% to 3% in `defaultState()`.
   - `defaultState()`'s seed `assets` and `funds` now start with `value:0` / `initial:0`+`monthly:0` respectively (properties were already 0) — only each item's `ret`/`taxRate` (and fund `include`/`taxFree`) keep meaningful defaults. This makes the allocation/funds/properties tabs open empty for the user to fill in, while return/tax assumptions are still pre-populated.
   - Added a preset picker for the allocation tab: clicking "+ הוספת אפיק" toggles a checkbox-list card (`#assetPicker`) populated from a new `ASSET_PRESETS` array (13 common Israeli investment channels — index funds, individual markets, bonds, commodities, gold, forex, cash, REIT, crypto — each with a sensible default annual return). Confirming adds one asset per checked box (value 0, preset return, current `defaultAssetTax`); cancel/confirm both clear and hide the picker. Replaces the old behavior of adding one blank "אפיק חדש" per click.
   - Verified by launching the file directly in Edge (`explorer.exe` on the `.html` path) and using Windows desktop automation (screenshot/click/scroll, since the chrome-devtools MCP server was disconnected this session): confirmed only 5 tabs remain, reset-to-default shows 0 ₪ everywhere with 3% withdrawal, allocation cards show value 0 with return/tax intact, and the preset picker adds checked items as new cards correctly. `node --check` on the extracted script also passed.

## Open question (not yet decided)
- **Google `noindex` tag:** currently the site is excluded from search engine indexing (added as a "why not" precaution, not in response to any actual exposure found — the site has no data leaving the browser regardless, so `noindex` is not a real security control, just discoverability). User was asked whether to keep it or remove it — **no answer given yet as of this note**. If asked to remove, delete this line from the `<head>`:
  ```html
  <meta name="robots" content="noindex,nofollow">
  ```

## Notes on working style for this project
- User does hands-on QA questions after each round (asks pointed follow-up bug reports rather than a full re-review request) — treat their bug reports as precise and investigate the exact mechanism before fixing.
- Always verify changes with a background Playwright agent (multiple viewport sizes, check `errors: []` for console/page errors) before pushing to the live site — this workflow caught 2 real regressions across past rounds that would otherwise have shipped.
- `node --check` on the extracted inline `<script>` block is used as a fast syntax gate before every Playwright run.
