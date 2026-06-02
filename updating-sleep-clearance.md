# Updating Sleep Clearance

How to tweak the two calibration variables and push the change live.

## The two variables

Both live near the top of `index.html`, inside the `<script>` block, in a section labeled **CONFIGURATION**. They look like this:

```javascript
const DRINK_UNITS = 1.3;
const MINUTES_PER_STANDARD_DRINK = 84;
```

**`DRINK_UNITS`** — how many "standard drinks" one tap of the button represents.
A US standard drink = 14g of pure ethanol = a 12 oz beer at 5% ABV. Reference values:

| What you're drinking | Value to use |
|---|---|
| 12 oz beer at 5% ABV | `1.0` |
| 12 oz IPA at 6.5% ABV | `1.3` ← current |
| 16 oz pint at 6.5% ABV | `1.7` |
| 5 oz wine at 12% ABV | `1.0` |
| 1.5 oz spirit at 40% ABV | `1.0` |

Formula if you want to compute your own: `oz × (ABV ÷ 100) × 1.315`

**`MINUTES_PER_STANDARD_DRINK`** — how long your liver takes to clear ONE standard drink.

| Body type | Minutes |
|---|---|
| 210 lb male, average enzymes | `84` ← current |
| Larger body (240+ lb) | `70–80` |
| Smaller body / female | `90–100` |

Lower number = faster clearance = more drinks allowed. If you want to be conservative (better sleep), bump the number up.

## Method 1: Edit on GitHub directly (easiest)

You don't need to download anything. Just edit in the browser.

1. Go to your repo: `github.com/<yourusername>/sleep-clearance`
2. Click the **`index.html`** filename in the file list
3. Click the **pencil icon** at the top right of the file view ("Edit this file")
4. Use Cmd+F to find `DRINK_UNITS` — change the number
5. Use Cmd+F to find `MINUTES_PER_STANDARD_DRINK` — change the number
6. Scroll to the bottom of the page
7. In the "Commit changes" box:
   - Leave the auto-generated message or write something like "Tune for pints"
   - Leave "Commit directly to the main branch" selected
   - Click the green **Commit changes** button

GitHub Pages redeploys automatically. Wait ~30 seconds, then reopen the app on your phone (close and relaunch from the home screen). The new values are live.

## Method 2: Edit locally, then upload

If you'd rather edit in a real text editor:

1. In the repo, click `index.html` → click the **download icon** (or "Raw" → save the page)
2. Open the file in any text editor (TextEdit works, but use **Format → Make Plain Text** first; better: VS Code, BBEdit, Sublime)
3. Edit the two variables, save
4. Back in your repo, click `index.html` → pencil icon → **delete all the contents** and paste in your new version
5. Commit changes

Or, faster: from the repo home page, click **Add file → Upload files**, drag the new `index.html` in (it'll replace the existing one), commit.

## Verifying the change took effect

The app caches aggressively on iOS. To force it to pull the new version:

1. Close the app fully (swipe up from home screen, swipe the app card away)
2. Reopen from the home screen icon

If you're really paranoid, open the GitHub Pages URL in Safari directly (not the home screen app) and confirm the change is there. Then relaunch the home screen version.

## Sanity-checking your numbers

After a change, do a mental calculation: at the current values, one drink adds **`DRINK_UNITS × MINUTES_PER_STANDARD_DRINK`** minutes to the queue. With the defaults that's `1.3 × 84 = 109 minutes`, just under two hours per beer.

If after editing you tap "+ Log 1 Drink" and the queue clears at a time that doesn't roughly match that math, something's wrong — most likely you accidentally added a typo (a stray letter, missing semicolon). Easiest fix: revert by editing again and putting the old values back.

## If something breaks

GitHub keeps every version. If a bad edit makes the app stop working:

1. Go to the repo → click `index.html`
2. Click **History** (top right of the file view)
3. Find the last version that worked, click it
4. Click the **`...`** menu (top right) → **View at this point in history**
5. Click the pencil to edit, then commit — this restores the old version as the new latest

You literally cannot permanently break it. Worst case is 5 minutes of confusion.
