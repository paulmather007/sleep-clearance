# iPhone test — does passing text input to the Shortcut cause a prompt?

**Goal:** confirm that adding `&input=text&text=…` to the `shortcuts://run-shortcut`
call does **not** make iOS show a confirmation every time you log a drink — *before*
we rewrite the real app. This is the gate on the `drink-metadata-spec.md` rewrite.

Nothing here touches the live `Drink` shortcut or `sleep_log.txt`. We use a throwaway
shortcut (`SleepTest`) writing to a throwaway file (`sleeptest_log.txt`).

**Pass condition:** Button B (with input) is *as silent as* Button A (baseline), and
the full line lands intact in the test file. If B prompts when A doesn't, the test
**fails** and we adjust before touching the real pipeline.

---

## Step 1 — Deploy `test.html` (on the Mac)

From the `Sleep tracker` repo:

```bash
git add test.html TESTING-shortcut-prompt.md
git commit -m "Add isolated test page for Shortcut input-prompt behaviour"
git push
```

Wait ~1 min for GitHub Pages to rebuild. The page will be live at:

**https://paulmather007.github.io/sleep-clearance/test.html**

> Same repo / same origin as the real app on purpose — the iOS "wants to run"
> grant is keyed to the source, so the test must launch from the same place.

---

## Step 2 — Build the `SleepTest` Shortcut (on the iPhone)

Open the **Shortcuts** app → **+** (new shortcut).

1. Tap the shortcut's name at the top → **Rename** → type exactly `SleepTest`
   (must match the name in `test.html`).
2. Add action **Append to File** (search "Append") — this is the only action needed:
   - **Append:** tap the variable slot → choose **Shortcut Input**.
     - If "Shortcut Input" isn't offered, tap **Select Variable** →
       **Shortcut Input**.
   - **File:** tap **File** → navigate to **iCloud Drive → Shortcuts** (create the
     folder if prompted) → set the filename to `sleeptest_log.txt`.
     - If the picker needs an existing file, first create an empty
       `sleeptest_log.txt` in Files (iCloud Drive → Shortcuts), then pick it.
   - Make sure **"New Line"** is **on** (so each append is its own line) — tap the
     action's expand arrow if you don't see the toggle.
3. Confirm there's **no** other action (no "Show Result"/notification) — we want it
   silent.
4. Tap the shortcut's settings (top, the (i) or the name → Details) and confirm
   **"Ask Before Running"** is **OFF** if such a toggle is shown. (For URL-scheme
   runs this may not appear; that's fine.)
5. **Done** to save.

> Editing/creating the shortcut can cause a **single one-time** "Allow access to
> sleeptest_log.txt?" prompt the first time it runs. That's a one-time re-grant,
> **not** the per-log prompt we're worried about — allow it and continue.

### Quick sanity check
In Shortcuts, tap **SleepTest** to run it manually once. Allow any one-time file
prompt. Then in **Files → iCloud Drive → Shortcuts**, open `sleeptest_log.txt` and
confirm a line was appended. (Run from the app, the appended text will be empty/blank
since there's no input — that's expected; we're just clearing the file grant here.)

---

## Step 3 — Add the test page to the Home Screen (on the iPhone)

So it launches in the same standalone PWA context as the real app:

1. Open **Safari** → go to
   `https://paulmather007.github.io/sleep-clearance/test.html`.
2. **Share** → **Add to Home Screen** → name it e.g. "ShortcutTest" → **Add**.
3. Launch it **from the Home Screen icon** (not the Safari tab) for the real test.

---

## Step 4 — Run the test

Do this in order and **watch closely for any popup, banner, or "Run Shortcut?"
sheet**:

1. **Tap Button A (Baseline)** first.
   - The first tap may show a one-time "[page] wants to run SleepTest" — **allow /
     Always Allow** it. This establishes the baseline grant (same as your real app
     already has).
   - Tap A **2–3 more times**. Confirm it's now **silent** (jumps to Shortcuts, runs,
     bounces back — no prompt).
2. **Tap Button B (With text input)** 2–3 times.
   - **The key question:** does B show *any* prompt that A didn't? Note exactly what
     you see, if anything.
3. Open **Files → iCloud Drive → Shortcuts → `sleeptest_log.txt`**.
   - Confirm the B taps appended the full line intact:
     `2026-06-04T20:15:00|Pint|overshoot`
   - (The pipes should be real `|` characters in the file — iOS decodes the
     `%7C` from the URL.)

The on-screen **Fired URLs** log on the test page shows the exact URL each button
sent, so you can copy it back to me if anything's odd.

---

## Step 5 — Report back

Tell me:

- **Did Button B prompt when A didn't?** (yes/no — and what the prompt said)
- **Did the full `…|Pint|overshoot` line land intact** in `sleeptest_log.txt`?

Then:

- **All silent + line intact →** we proceed with the `drink-metadata-spec.md`
  rewrite as written.
- **B prompts every time →** we stop and pick an alternative (e.g. clipboard
  hand-off) before touching the real app.

---

## Step 6 — Clean up (after we've decided)

- Delete the **ShortcutTest** Home Screen icon (long-press → Remove).
- Delete the **SleepTest** shortcut (long-press in Shortcuts → Delete).
- Delete `sleeptest_log.txt` from iCloud Drive.
- On the Mac, remove the test files and push:
  ```bash
  git rm test.html TESTING-shortcut-prompt.md
  git commit -m "Remove Shortcut input-prompt test scaffolding"
  git push
  ```
