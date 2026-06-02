# Sleep Clearance — Development Context

## Purpose

Single-user mobile web app that answers "Can I have a drink right now without disrupting sleep?" Logs beers and wines, tracks the alcohol-clearance "queue" against a fixed bedtime, and refuses logs after a hard cutoff. Personal use only (Paul).

## Files

- **`index.html`** — the live app. Self-contained: HTML + Tailwind (CDN) + vanilla JS, no build step.
- **`Tracker.html`** — earlier prototype, kept for reference. Do not edit.
- **`updating-sleep-clearance.md`** — design notes / change history.
- **`archive/`** — dated backups of `index.html` before significant changes. Always copy here before a non-trivial edit.

## Run / Test

Open `index.html` directly in a browser, or serve the directory:
```bash
cd "Sleep tracker" && python3 -m http.server 8000
```
No build, no deploy pipeline. The app is meant to live on the user's iPhone home screen as a saved web page (PWA-style via the `apple-mobile-web-app-*` meta tags).

## Core Model

- **Bedtime**: 11:30 PM. **Hard cutoff**: 8:30 PM (no logs accepted after).
- **Standard drink** = 0.6 fl oz / 14 g ethanol.
- **`MINUTES_PER_STANDARD_DRINK = 70`** — Paul's metabolic clearance rate. Tune for body size.
- **Per-type units** (configurable, in standard-drink equivalents):
  - `BEER_UNITS = 1.3` (≈ 12 oz IPA @ 6.5%)
  - `WINE_UNITS = 1.0` (≈ 5 oz wine @ 12%)
  - Each has a comment block in `index.html` listing common volume × ABV values. The user prefers a single tweakable number per type with reference values, **not** separate volume + ABV variables.
- **`systemZeroTime`** — the running "queue clears" timestamp. Each logged drink pushes it later by `units × MINUTES_PER_STANDARD_DRINK`. Budget for a given type = `floor((bedtime − max(now, systemZeroTime)) / ms_per_that_type)`.

## UI Rules

- Big `YES` / `NO` verdict reflects the **most permissive** state: YES if either beer or wine still fits before bedtime, NO only when neither does.
- Two side-by-side log buttons (`🍺 Beer`, `🍷 Wine`). Each button independently:
  - Shows `N left` **only** when that type's budget ≤ `BUDGET_THRESHOLD` (5). Above the threshold the sub-text is blank — the threshold exists to nudge restraint, not to brag about headroom.
  - Greys to `No room` if that type alone won't fit, or `Cut off` after 8:30 PM.
- Drinks Today row shows total; Breakdown row shows `🍺 N · 🍷 N`.
- Undo button appears for `UNDO_WINDOW_SECONDS = 60` after any log; remembers the drink type and subtracts the right amount.

## Apple Shortcut Integration

Logging either drink fires `shortcuts://run-shortcut?name=Drink` via `window.location.href`. Single shortcut handles both types — type is **not** passed to the shortcut. Set `SHORTCUT_NAME = ""` to disable.

## Persistence

`localStorage` keys (all prefixed `sleepTracker_`):
- `zeroTime` — ISO timestamp of queue clearance
- `lastDrink`, `lastType` — for undo (`lastType` is `'beer'` or `'wine'`)
- `beerCount`, `wineCount`, `countDate` — daily counters; rollover at 4 AM local (`todayKey()`)

When adding new state, follow the same prefix and rollover pattern.

## Editing Conventions

- Keep it dependency-free: no build tools, no npm. Tailwind via CDN is fine.
- Keep the tunables (`BEER_UNITS`, `WINE_UNITS`, `MINUTES_PER_STANDARD_DRINK`, bedtime/cutoff hours, `BUDGET_THRESHOLD`, `UNDO_WINDOW_SECONDS`, `SHORTCUT_NAME`) at the top of the `<script>` block under the `CONFIGURATION` banner. Comment ranges/examples next to each.
- Before any non-trivial change, copy the current `index.html` to `archive/index-YYYY-MM-DD-<reason>.html`.
- Test in a browser after edits — the app is small enough that manual click-through covers it.
