# Updating Sleep Clearance

How to retune the app's numbers and push the change live. The workflow is
always: **edit `index.html` locally → commit → push**. Never edit through
GitHub's web editor — it creates a second history that has to be painfully
reconciled with the one on this Mac.

## Where the numbers live

All tunables sit near the top of `index.html`, inside the `<script>` block,
under the **CONFIGURATION** banner. Each has a comment with reference values
next to it.

## Drink strength

One number per drink type, in "standard drinks" (14 g ethanol each).
Formula for computing your own: `oz × (ABV ÷ 100) ÷ 0.6` — e.g. a 12 oz
IPA at 6.5% is `12 × 0.065 ÷ 0.6 ≈ 1.3`.

**`BEER_UNITS`** (12 oz bottles/cans):

| What you're drinking | Value |
|---|---|
| 12 oz lager at 5% ABV | `1.0` |
| 12 oz IPA at 6.5% ABV | `1.3` ← current |
| 12 oz DIPA at 8% ABV | `1.6` |

**`PINT_UNITS`** (16 oz pours):

| What you're drinking | Value |
|---|---|
| 16 oz lager at 5% ABV | `1.3` |
| 16 oz IPA at 6.5% ABV | `1.7` ← current |
| 16 oz DIPA at 8% ABV | `2.1` |

**`WINE_UNITS`**:

| What you're drinking | Value |
|---|---|
| 5 oz red/white at 12% ABV | `1.0` ← current |
| 5 oz bold red at 14.5% ABV | `1.2` |
| 6 oz pour at 13% ABV | `1.3` |
| 8 oz pour at 13% ABV | `1.7` |

## Clearance rate

**`MINUTES_PER_STANDARD_DRINK`** — how long your body takes to clear one
standard drink.

| Body type | Minutes |
|---|---|
| Larger body (210+ lb) | `70–80` ← current: `80` |
| Average male | `84–90` |
| Smaller body / female | `90–100` |

Higher number = slower clearance = fewer drinks allowed. To be conservative
(better sleep), bump it up.

## Naltrexone

**`NALTREXONE_LEAD_MINUTES = 60`** — how long after logging a dose the app
considers it active. Matches the Sinclair-method instruction "take it 60
minutes before drinking." The dose stays active until the 4 AM rollover.

## Other knobs

- **`BEDTIME_HOUR` / `BEDTIME_MINUTE`** — currently 23:30. The clearance
  queue is budgeted against this.
- **`CUTOFF_HOUR` / `CUTOFF_MINUTE`** — currently 20:30. After this the
  verdict is a hard NO (buttons still work, to record lapses).
- **`BUDGET_THRESHOLD = 5`** — "N left" only appears on a drink button once
  its budget drops to this. It nudges restraint without bragging about
  headroom.
- **`UNDO_WINDOW_SECONDS = 60`** — how long the undo button lingers after a
  log.
- **`SHORTCUT_NAME = "Drink"`** — the Apple Shortcut that appends each log
  line to `sleep_log.txt` in iCloud. Set to `""` to disable.

## Publishing a change

From Terminal, in the project folder:

```bash
cd "/Users/pmather/Dev/Sleep tracker"
git add -A
git commit -m "Tune pint strength for 8% DIPAs"   # describe what you changed
git push
```

What each step does: `add` chooses what goes in the snapshot, `commit`
saves the snapshot on this Mac only, `push` uploads it to GitHub. Only the
push makes it live — GitHub Pages rebuilds within a minute or two.

(Or just ask Claude Code to make the change; it will walk through the same
steps.)

## Seeing the change on the phone

iOS caches home-screen web apps aggressively:

1. Fully close the app (swipe up, flick the app card away).
2. Reopen from the home screen icon.

If it still looks stale, open the Pages URL in Safari directly to confirm
the deploy landed, then relaunch the home-screen version.

## Sanity-checking after a tune

One drink adds `UNITS × MINUTES_PER_STANDARD_DRINK` minutes to the queue.
At current values a pint adds `1.7 × 80 = 136` minutes. If you log a drink
and "Queue Clears" doesn't roughly match that math, the edit likely
introduced a typo.

## If something breaks

Git keeps every version, so nothing is ever lost:

- **See what changed:** `git log --oneline` lists snapshots;
  `git diff` shows uncommitted edits.
- **Throw away uncommitted edits:** `git checkout -- index.html` puts the
  file back to the last commit.
- **Undo a bad commit that's already pushed:** ask Claude Code, or
  `git revert <commit-id>` followed by `git push` — this creates a new
  commit that reverses the bad one (history stays intact).
- There are also dated snapshots in `archive/` from before each significant
  change.
