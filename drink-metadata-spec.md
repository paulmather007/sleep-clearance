# Spec — Pass drink type + overshoot to the Sleep Journal

**Status:** design agreed, not yet implemented.
**Date:** 2026-06-04.
**Scope:** two repos — `Sleep tracker/` (the web app + the Apple Shortcut) and
`Sleep Journal/` (the macOS ingest + DB + UI). No third-party services; iCloud
Drive remains the only courier, exactly as today.

## Goal

Carry two new facts from the iPhone web app, through the existing Apple Shortcut
and `sleep_log.txt` in iCloud Drive, into the Mac journal:

1. **Drink type** — `Beer` / `Pint` / `Wine` (currently a single generic label).
2. **Overshoot** — whether this drink pushes alcohol clearance past bedtime
   (11:30 PM). Computed silently; **never shown in the web app UI**.

Decisions taken (2026-06-04):
- Overshoot is stored in a **new dedicated column** `entries.overshoot` (0/1).
- Overshoot **is shown** in the Mac journal timeline as a subtle marker.
- The **web app builds the full log line**; the Shortcut just appends it.

---

## 1. The wire format (`sleep_log.txt` line grammar)

Today (`Sleep Journal/app/ingest_shortcuts.py:23`, splits on the first `|` only):

```
<ISO 8601 timestamp> | <label name>
```

New grammar — a third, optional field:

```
<local ISO 8601 timestamp> | <Type> [ | <flags> ]
```

- **Field 1 — timestamp.** MUST be **local** time, either naive
  (`2026-06-04T20:15:00`) or with a local offset (`2026-06-04T20:15:00-04:00`).
  **Do NOT send UTC `Z`** — see Pitfalls.
- **Field 2 — Type.** One of `Beer`, `Pint`, `Wine`. Becomes the journal label.
- **Field 3 — flags (optional).** Currently only the literal token `overshoot`.
  Absent or empty ⇒ not an overshoot. Single token for now; if more flags are
  ever needed, make it comma-separated (`overshoot,foo`) — parser should tolerate
  that shape already.

Examples:

```
2026-06-04T20:15:00|Pint|overshoot
2026-06-04T20:40:00|Beer
2026-06-04T21:05:00|Wine|overshoot
```

**Backward compatibility:** old two-field lines (`<ts>|Drink`) and any existing
`Drink`-labelled rows in `sleep.db` keep working unchanged — type defaults to the
label as written, overshoot defaults to 0.

---

## 2. Web app changes — `Sleep tracker/index.html`

All edits live in `logDrink(type)` (currently lines 327–354). Archive
`index.html` to `archive/index-YYYY-MM-DD-drink-metadata.html` first, per project
convention.

### 2a. Compute the overshoot flag

`logDrink` already sets `systemZeroTime` to the post-drink clearance time
(line 333). Bedtime is already computed elsewhere (lines 234–235). So:

```js
const bedtime = new Date(now);
bedtime.setHours(BEDTIME_HOUR, BEDTIME_MINUTE, 0, 0);
const overshoot = systemZeroTime > bedtime;   // this drink finishes after 11:30
```

(`systemZeroTime` here is the value *after* the `new Date(baseTime + ms)`
assignment on line 333.) Nothing about this is surfaced in the UI.

### 2b. Build a local ISO timestamp (no UTC)

`new Date().toISOString()` returns UTC — must not be used for the log line. Add a
small helper:

```js
function localISO(d) {
  const p = n => String(n).padStart(2, '0');
  return `${d.getFullYear()}-${p(d.getMonth()+1)}-${p(d.getDate())}` +
         `T${p(d.getHours())}:${p(d.getMinutes())}:${p(d.getSeconds())}`;
}
```

(Naive local time. The journal parser localizes a naive timestamp via
`astimezone()`, so this round-trips correctly.)

### 2c. Pass the full line as the Shortcut's input

Replace the no-input call (line 352):

```js
if (SHORTCUT_NAME) {
  const TYPE_LABEL = { beer: 'Beer', pint: 'Pint', wine: 'Wine' };
  let line = `${localISO(now)}|${TYPE_LABEL[type]}`;
  if (overshoot) line += '|overshoot';
  window.location.href =
    `shortcuts://run-shortcut?name=${encodeURIComponent(SHORTCUT_NAME)}` +
    `&input=text&text=${encodeURIComponent(line)}`;
}
```

The `input=text&text=…` form hands `text` to the Shortcut as its Shortcut Input.
`encodeURIComponent` handles the `|` (→ `%7C`).

### 2d. Document the new tunable

Add `TYPE_LABEL` (or reuse the existing type strings) near the CONFIGURATION
banner if it should be tweakable. The mapping is the only new constant.

**Out of scope / known limitation:** undo (lines 360+) reverses local state only.
The Shortcut has already fired and written the line to iCloud by then; undo cannot
retract it. This matches today's behaviour and is not addressed here.

---

## 3. Apple Shortcut changes (`Drink`)

The Shortcut becomes a dumb sink:

1. **Receive** Shortcut Input (text) — the fully-formed line.
2. **Append** `Shortcut Input` + a newline to
   `…/iCloud~is~workflow~my~workflows/Documents/sleep_log.txt`.

It no longer needs to compute the timestamp or know the drink type. If the current
Shortcut generates its own timestamp/label, remove that; just append the input.
Keep one Shortcut (no per-type duplication).

### Confirm it still writes silently — test in isolation first

Adding `&input=text&text=…` does **not** introduce a per-write iOS confirmation:
it's the same `shortcuts://run-shortcut` scheme (the Safari→Shortcuts grant is
keyed to source + shortcut name, not the query params), and "Append to File" is an
action the Shortcut already runs and is already granted. The one caveat is that
**editing/rebuilding the Shortcut may show a single one-time "Allow access to
[file]?" prompt on its first run afterwards** — a re-grant, not recurring.

iOS prompting is version-dependent, so verify before wiring up the web app:

1. Edit the Shortcut to append `Shortcut Input`.
2. Trigger it **once manually** (or from a test link) and clear any one-time grant.
3. Confirm the write is silent — *then* do the §2 web app change.

---

## 4. Sleep Journal changes (`Sleep Journal/` repo)

Four touch points. File references are to the current tree.

### 4a. Schema + migration — `app/db.py`

Add the column to the `entries` CREATE TABLE (around line 54):

```sql
overshoot INTEGER NOT NULL DEFAULT 0,
```

`init_db()` runs `CREATE TABLE IF NOT EXISTS`, which will **not** alter an existing
table, so add an explicit migration for the live `sleep.db`. Minimal approach in
`init_db()` (or a tiny `_migrate()` helper) before/after the `executescript`:

```python
cols = [r[1] for r in conn.execute("PRAGMA table_info(entries)")]
if "overshoot" not in cols:
    conn.execute("ALTER TABLE entries ADD COLUMN overshoot INTEGER NOT NULL DEFAULT 0")
```

Back up `sleep.db` first (the repo already keeps `sleep.db.bak.*` files).

### 4b. Entry writer — `app/entries.py`

`create_entry` / `update_entry_full` build explicit column lists. Add `overshoot`
to both INSERT/UPDATE and read `data.get('overshoot', 0)`. Default 0 keeps every
existing caller (manual entry form, HAE ingest) working untouched.

### 4c. Parser + dedup — `app/ingest_shortcuts.py`

**`_parse_line` (line 23):** stop splitting on the first pipe only. Parse up to
three fields:

```python
parts = [p.strip() for p in raw.split("|")]
# parts[0]=ts, parts[1]=type/label, parts[2:]=flags (optional)
```

Validate ≥2 non-empty fields (ts + type). Derive
`overshoot = 1 if any flag == "overshoot" else 0` from `parts[2:]`. Return the
overshoot value alongside `(iso_time, label_name)`.

**`_entry_exists` (line 40):** the **shape predicate stays valid** — shortcut rows
still have `text_body IS NULL AND value IS NULL` (overshoot is its own column).
Just extend the **key** so a normal vs. overshoot drink aren't confused on
re-ingest:

```sql
... AND label_id IS ?
    AND text_body IS NULL
    AND value IS NULL
    AND overshoot IS ?
```

(pass `overshoot` as the extra param.) This preserves the iCloud-resurrection
idempotency guarantee.

**`ingest_shortcuts_log` (line 107+):** thread `overshoot` from the parsed tuple
into the `_entry_exists(...)` check and the `create_entry({...})` call.

### 4d. Timeline display — `app/ui/timeline.py`

`render_entry` (line 364) builds `header_text = f"{time_str}  {label_name}"`.
Append a subtle marker when overshoot:

```python
marker = " ⚠" if entry.get("overshoot") else ""
header_text = f"{time_str}  {label_name}{marker}"
```

`get_all_entries` does `SELECT e.*`, so `overshoot` is already in the row dict — no
query change needed. (Decide marker glyph/text during implementation; `⚠` or
`(overshoot)` both fit.)

---

## 5. Pitfalls / gotchas (carry into implementation)

1. **UTC timestamps display wrong.** The parser preserves any tz you send; a UTC
   `Z` time shows the wrong HH:MM in the timeline. Web app MUST send local time
   (see §2b).
2. **Parser previously swallowed extra pipes** (`split("|", 1)`). The 3-field
   change is mandatory or `overshoot` ends up glued onto the label name.
3. **Idempotency is load-bearing.** iCloud resurrects the truncated log and
   re-feeds lines; the dedup must include `overshoot` in its key (§4c) or the
   re-ingest guard weakens.
4. **Label proliferation.** Switching the generic `Drink` label to
   `Beer`/`Pint`/`Wine` means historical `Drink` rows stay under `Drink` while new
   ones split three ways. Acceptable; optionally backfill/rename later (not in
   scope).
5. **Migration on a live DB.** `CREATE TABLE IF NOT EXISTS` won't add the column;
   the explicit `ALTER TABLE` in §4a is required. Back up `sleep.db` first.
6. **Undo can't un-send.** See §2d.

---

## 6. Test plan

Web app (manual, browser):
- Log each type below cutoff with low queue → line is `<localISO>|<Type>` (no flag).
- Log enough to push clearance past 11:30 → line gains `|overshoot`.
- Confirm `text=` is percent-encoded (`%7C`).

Journal (`tests/`, using the `temp_db` fixture — never the real `sleep.db`):
- `_parse_line` for 2-field, 3-field-overshoot, 3-field-empty-flag, and malformed
  lines.
- Ingest writes `label_id` = correct type and `overshoot` = correct 0/1.
- **Idempotency:** re-ingesting the same overshoot line inserts nothing; a
  same-timestamp/same-type line that flips overshoot is treated as distinct.
- Migration: open a pre-migration DB copy → column added, existing rows default 0.
- Timeline: an overshoot entry renders the marker; a normal one doesn't.

---

## 7. Implementation order (suggested)

1. Journal first (parser + schema/migration + dedup + entries writer) with tests —
   it can accept the new line format while staying backward-compatible.
2. UI marker.
3. Web app `logDrink` change (+ archive copy).
4. Update the Apple Shortcut to append raw input.
5. End-to-end: log a drink on the phone → reload journal on Mac → verify type +
   marker.

Cross-repo note: §2 lives in this repo (`Sleep tracker/`); §4 lives in
`Sleep Journal/`. Commit them separately.
