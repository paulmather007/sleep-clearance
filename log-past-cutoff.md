# Interface change — allow logging past the cutoff

## Why

In a moderation context the most important data is the *overshoot* — the drink
logged after the 8:30 PM cutoff or after the budget is empty. The current app
refuses to log in exactly those moments (buttons go dead, "Cut off" / "No room"),
which trains the habit of *not logging when you go over*. That censors the dataset
flowing to the macOS analysis app, making real drinking look tidier than it is.

The fix separates the app's two jobs:

- **Advice** — "should I have a drink?" → the YES/NO verdict. Stays honest.
- **Record** — "log what actually happened" → the buttons. Always open.

## What changes

1. **Logging buttons always function.** Drop the disabled state entirely. Past
   cutoff or out of budget, tapping 🍺 / 🍺 Pint / 🍷 still logs the drink,
   extends the queue, increments the count, and fires the Drink shortcut. The
   data always flows.

2. **Totals keep updating.** "Drinks Today" and the breakdown row count
   over-limit drinks the same as any other — no special-casing.

## What stays exactly as-is (no added emphasis)

- **Verdict copy and color.** Big rose `NO`, the rose panel, and the reason line
  (`"Past 8:30 PM hard cutoff."` / `"Budget empty. Another drink disrupts sleep."`)
  are unchanged. The honest answer to "Can I have a drink?" is still NO.
- **No new warnings, emoji, colors, or scolding** anywhere. The count never turns
  red, never celebrates, never reacts.
- Tone is held flat. Honesty lives in the verdict; the log stays guilt-free.

## Net effect

A state is *removed*, not added. The verdict carries all the honesty; the buttons
just stop blocking. Past-cutoff logging works silently, so the downstream analysis
finally sees the complete picture.
