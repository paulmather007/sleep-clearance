# Sleep Clearance — Development Context

## Purpose

Single-user mobile web app that answers "Can I have a drink right now without disrupting sleep?" Logs beers, pints, and wines, tracks the alcohol-clearance "queue" against a fixed bedtime, and refuses logs after a hard cutoff. Personal use only (Paul).

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
No build step. The app lives on Paul's iPhone home screen as a saved web page (PWA-style via the `apple-mobile-web-app-*` meta tags), served from GitHub Pages — see Deployment.

## Deployment (GitHub Pages)

- **Repo:** `https://github.com/paulmather007/sleep-clearance` (public). GitHub Pages builds from `main` / root → live at `https://paulmather007.github.io/sleep-clearance/`.
- **Local git is the source of truth.** Edit locally, then `git add -A && git commit -m "…" && git push`. Pages rebuilds within ~1 min; reload on the phone.
- **Do NOT edit via GitHub's web editor** — it diverges from local and forces a reconcile. (The repo was bootstrapped from a web-only history; local and remote were force-pushed into one history once. They share history now, so plain `git push` works.)
- Auth is via the `gh` CLI token in macOS keychain — no token juggling on push.

## Core Model

- **Bedtime**: 11:30 PM. **Hard cutoff**: 8:30 PM (no logs accepted after).
- **Standard drink** = 0.6 fl oz / 14 g ethanol.
- **`MINUTES_PER_STANDARD_DRINK = 80`** — Paul's metabolic clearance rate. Tune for body size.
- **Per-type units** (configurable, in standard-drink equivalents):
  - `BEER_UNITS = 1.3` (≈ 12 oz IPA @ 6.5%)
  - `PINT_UNITS = 1.7` (≈ 16 oz IPA @ 6.5%)
  - `WINE_UNITS = 1.0` (≈ 5 oz wine @ 12%)
  - Each has a comment block in `index.html` listing common volume × ABV values. The user prefers a single tweakable number per type with reference values, **not** separate volume + ABV variables.
  - `MS_PER_TYPE = { beer, pint, wine }` maps each stored type string to its clearance cost; `logDrink`/undo index into it rather than branching per type.
- **`systemZeroTime`** — the running "queue clears" timestamp. Each logged drink pushes it later by `units × MINUTES_PER_STANDARD_DRINK`. Budget for a given type = `floor((bedtime − max(now, systemZeroTime)) / ms_per_that_type)`.

## UI Rules

- Big `YES` / `NO` verdict reflects the **most permissive** state: YES if any of beer/pint/wine still fits before bedtime, NO only when none does.
- Three side-by-side log buttons in a `grid-cols-3` row (`🍺 Beer`, `🍺 Pint`, `🍷 Wine`). Each button independently:
  - Shows `N left` **only** when that type's budget ≤ `BUDGET_THRESHOLD` (5). Above the threshold the sub-text is blank — the threshold exists to nudge restraint, not to brag about headroom.
  - Shows `No room` if that type alone won't fit, or `Cut off` after 8:30 PM. Buttons themselves always remain active and enabled (never greyed out or disabled) to encourage recording overshoot drinks.
- Drinks Today row shows total; Breakdown row shows `🍺 N · 🍺16 N · 🍷 N` (the `🍺16` glyph distinguishes pints from 12 oz beers).
- Undo button appears for `UNDO_WINDOW_SECONDS = 60` after any log; remembers the drink type and subtracts the right amount.

## Apple Shortcut Integration

Logging a drink fires `shortcuts://run-shortcut?name=Drink&input=text&text=<line>` via `window.location.href`, where `<line>` is the full `sleep_log.txt` line the web app builds: `<localISO>|<Type>[|overshoot]` (`Type` ∈ `Beer`/`Pint`/`Wine`; `|overshoot` appended when the drink's clearance finishes after bedtime). The single `Drink` shortcut is now a dumb sink — it just appends `Shortcut Input` to `sleep_log.txt` in iCloud Drive; it no longer computes the timestamp or type. The timestamp MUST be **local** time (never UTC `Z`) or the journal timeline shows the wrong HH:MM — `logDrink()` builds it via the `localISO()` helper and the `TYPE_LABEL` map. Set `SHORTCUT_NAME = ""` to disable. Full design + the consuming Sleep Journal changes: `drink-metadata-spec.md`.

## Persistence

`localStorage` keys (all prefixed `sleepTracker_`):
- `zeroTime` — ISO timestamp of queue clearance
- `lastDrink`, `lastType` — for undo (`lastType` is `'beer'`, `'pint'`, or `'wine'`)
- `beerCount`, `pintCount`, `wineCount`, `countDate` — daily counters; rollover at 4 AM local (`todayKey()`)

When adding new state, follow the same prefix and rollover pattern.

## Editing Conventions

- Keep it dependency-free: no build tools, no npm. Tailwind via CDN is fine.
- Keep the tunables (`BEER_UNITS`, `PINT_UNITS`, `WINE_UNITS`, `MINUTES_PER_STANDARD_DRINK`, bedtime/cutoff hours, `BUDGET_THRESHOLD`, `UNDO_WINDOW_SECONDS`, `SHORTCUT_NAME`) at the top of the `<script>` block under the `CONFIGURATION` banner. Comment ranges/examples next to each.
- Before any non-trivial change, copy the current `index.html` to `archive/index-YYYY-MM-DD-<reason>.html`.
- Test in a browser after edits — the app is small enough that manual click-through covers it.
