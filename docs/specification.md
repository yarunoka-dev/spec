# Yrnk Schedule DSL Specification (version 1.0)

A DSL for describing calendar-aware schedules in JSON. The name of the
language is **Yrnk** (short for Yarunoka) — the project is Yarunoka, the
notation is Yrnk. The authoritative definition of the syntax is the set of
JSON Schemas under [`schema/`](../schema/) (JSON Schema draft 2020-12);
this document is the specification of the language including its semantics.
Language implementations are copies of this authority, and their agreement
is guaranteed by tests.

A Yrnk document is a **description of a set of points in time** and knows
nothing about execution. "Should this fire", "last run at", and
"catch-up" do not exist in this language's vocabulary — they are the
caller's concern, expressed through the questions the caller asks.

## Document model

A document is a JSON object with two layers:

- **Reading directives** — `version` and `timezone` declare *how to
  interpret* the document before any content is read
- **Content** — `calendar` (the definitions) and `schedules` (the
  expressions)

```json
{
  "version": "1.0",
  "timezone": "Asia/Tokyo",
  "calendar": {
    "holidays": ["2026-01-01", "2026-01-12"],
    "custom": { "founding-day": ["2026-10-01"] }
  },
  "schedules": [
    {"days": ["founding-day"], "allday": true}
  ]
}
```

| Key | Required | Meaning |
|---|---|---|
| `version` | ✓ | The spec version this document is written against, as an `"x.y"` string. Implementations must reject versions they do not know rather than silently interpreting them |
| `timezone` | ✓ | The timezone in which every schedule is interpreted. **The document is authoritative** — a document carried anywhere means the same thing |
| `calendar` | | The definitions part (see below) |
| `schedules` | ✓ | The list of schedules. **The list is an OR of complete schedules** (a bare object is not allowed) |

- Unknown keys are an error (closed set — the same rule applies at the
  document, calendar, schedule, and times levels)
- `timezone` is an IANA name (`Asia/Tokyo`) or a fixed offset (`+09:00`).
  Zones with daylight-saving transitions are allowed. Wall-clock times that
  fall on a transition are resolved **per RFC 5545 §3.3.5** — a time that
  does not exist (the spring-forward gap) is interpreted with the offset in
  effect before the transition, which pushes it forward in real time; a
  time that occurs twice (the fall-back overlap) counts only as its first
  occurrence
- **An array is an enumeration; omission means "all"**. There is no scalar
  sugar — `"days": "mon"` and `"months": 2` are invalid; a single element
  is still written as an array (so that the same value never has two
  spellings, and round-tripping is the identity)
- The whole DSL denotes a set, so when several schedules produce the same
  point the OR contains it once, and a firing decision sees it once

## Versioning

The spec version is an `"x.y"` string, and the `version` field of a
document declares which spec version it is written against.

- **y is raised** for changes that keep compatibility: additions to the
  closed sets (new vocabulary, new atoms, new fields) that leave the
  meaning of every existing document unchanged. A document written against
  1.y is accepted by an implementation of 1.y′ (y′ > y) with the same
  meaning
- **x is raised** for breaking changes. Compatibility with documents
  written against a lower major version is not guaranteed
- An implementation must reject a document whose declared version it does
  not know

The first public version is 1.0.

## calendar — the definitions

The calendar is the set of date and time-window definitions that schedules
refer to. Its top-level keys are a closed set of **reserved keys** (the
built-in definitions); under `custom` is an **open namespace** (the user's
own named date lists). A calendar contains wall-clock dates and times
only, and its content does not depend on the document timezone.

```json
"calendar": {
  "holidays": ["2026-01-01", "2026-01-12"],
  "business_holidays": [],
  "business_days": [],
  "workweek": ["mon", "tue", "wed", "thu", "fri"],
  "business_hours": [["09:00", "12:00"], ["13:00", "18:00"]],
  "custom": {
    "founding-day": ["2026-10-01"],
    "garbage-day": "garbage-days"
  }
}
```

- **The built-in definitions are special**: `holidays` /
  `business_holidays` / `business_days` / `workweek` carry the layer-model
  semantics (below), and `business_hours` is the window list behind the
  `business_hour` vocabulary. `custom` entries take no part in the layers;
  a custom name is a flat "membership in a set" and nothing more
- **Custom values are date lists only** (windows cannot be named — date
  sets can be large and dynamic, which is a real need for naming, while a
  window list is short enough to write inline in a schedule; the only
  shared windows are the built-in `business_hours`). Custom key names must
  not collide with reserved words and must not look like literals (digits
  only, date-shaped, time-shaped)
- **Resolver name references**: wherever a date list is expected, a string
  **resolver name** may be written instead (`"holidays": "yasumi-jp"`).
  The runtime registers a name-to-function mapping, and a reference to an
  unregistered name is a parse-time error. The two are distinguished by
  shape (a `YYYY-MM-DD`-shaped string is a date; any other string is a
  resolver name). This is the mechanism for feeding dynamic data (a
  database, holiday computation) into a document while the document keeps
  the *intent* — what the dates are resolved by

## Schedule

```json
{"days": [25], "shift": ["prev", "or_same", "business_day"], "times": ["10:00"]}
```

The fields are `years` / `months` / `days` (the date axes), `shift` / `if`
(the date modifiers), `times` | `allday` | `every` (the time part —
**exactly one is required**), and `from` / `until` (the validity range).

The algebra has three tiers: within an axis (an array) = OR, between
fields (juxtaposition) = AND, between schedules (the `schedules` list)
= OR.

### from / until — validity range

A boundary that clips the schedule's set of points to the half-open
interval **[from, until)**. It is **not a recurrence condition** — it
never interferes with how the daily points are laid out (the times grid,
the matching days); points outside the range simply do not exist.

```jsonc
// an hourly task starting 10:00 on 7/15
{"from": "2026-07-15 10:00", "every": [1, "hour"]}

// every Monday 10:00 from 8/1 onward
{"from": "2026-08-01 00:00", "days": ["mon"], "times": ["10:00"]}

// limited period (all of July)
{"from": "2026-07-01 00:00", "until": "2026-08-01 00:00", "times": ["09:00"]}
```

- The value has exactly one form: `"YYYY-MM-DD HH:MM"` (zero-padded, a
  single space, no seconds). A date-only `"YYYY-MM-DD"` is invalid — if
  omission meant 00:00, the same instant would have two spellings, so a
  range starting at the top of a day writes `00:00` explicitly. `24:00`
  does not exist in this position (write the next day's `00:00`; the
  interval being half-open makes that mean exactly "through that day")
- Interpretation uses the document `timezone`. A wall time that does not
  exist (the DST gap) resolves by the same rule as scheduled points
  (RFC 5545 §3.3.5)
- A point at `from` is included; a point at `until` is not (the same
  half-open convention as `between`)
- Each is independent (only `from`, only `until`, or both). With both,
  from < until is required. **Vocabulary that counts** (the
  `["every", N, "day"]` atom and the interval `every`) requires `from`
  (there is no way to start counting without it); otherwise both are
  optional
- An `allday` point is a point at the start of its day (00:00), so with
  `from: "2026-07-14 12:00"` the allday point of 7/14 is out of range. The
  clipping rule is uniform and has no exceptions

### Date axes

- `years` — integers (1–9999). A one-off event (a specific day of a
  specific year) is expressible
- `months` — integers (1–12)
- `days` — an enumeration of the atoms below

| Atom | Example | Meaning |
|---|---|---|
| Number (1–31) | `25` | the 25th of every month |
| Day name | `"mon"` | every Monday |
| Calendar vocabulary | `"holiday"` | the five layer-model words (below) |
| Ordinal tuple | `["3rd", "mon"]` / `["last", "fri"]` | the third Monday / last Friday of the month |
| End of month | `"last_day_of_month"` | the only month boundary that moves, so it is the one special word |
| Day-cycle tuple | `["every", 2, "day"]` | every N days (below) |
| Custom name | `"founding-day"` | a reference to a date list in `calendar.custom` |

- The ordinals are the six words `1st`–`5th`, `last`. In a month without a
  fifth week, a `5th` tuple simply does not match. A tuple is always
  written as an element of the enumeration (`{"days": [["3rd", "mon"]]}`)
- A date literal (`"2026-10-01"`) cannot be written directly in `days`.
  Give the date a name under `custom` and refer to it
- There are no negative day numbers (`-1` = end of month). The day before
  the end of the month is written `{"if": ["next", "last_day_of_month"]}`

### The day-cycle tuple — every N days

```jsonc
// every 2 days at 03:00 (7/14, 7/16, 7/18, …)
{"from": "2026-07-14 00:00", "days": [["every", 2, "day"]], "times": ["03:00"]}
```

- The matching days are **every Nth day counting the date of the
  schedule's `from` as day one** ({the from day, +N, +2N, …}). Days before
  the `from` date never match. Because it counts, **`from` is required**
- `from` is a validity start, not a recurrence condition — the day the
  validity starts is merely day one of the count, and the firing times are
  decided by `times` (the time part of `from` only clips the range: with
  `from` at 7/14 12:00 and `times` 03:00, 7/14 03:00 is out of range and
  the first point is 7/16 03:00)
- The count is an integer ≥ 1 with no upper bound (`["every", 1, "day"]`
  = every day from the `from` date). The unit is fixed and explicit:
  `"day"` (as with the times `every`, the unit is never sometimes-written,
  sometimes-not). A future year cycle would have a syntactic home as
  `["every", 2, "year"]`
- `years` / `months` / `if` only **filter** the matching days; the count is
  not reset (excluded days do not shift the cycle). `shift` moves each
  matching day as a base day
- Allowed only as an element of the `days` enumeration (not as a `shift`
  landing condition or an `if` condition)

### Calendar vocabulary — the layer model

A business day is not a weekday. Weekdays are determined by the calendar
alone; business days are decided by an organization. The decision consults
the calendar's layers top-down with early return:

```text
business_days      top layer: "we work this day" — overrides everything below
business_holidays  the organization's own closures (often built on holidays)
holidays           public holidays; closed by default
workweek           bottom layer: the weekly pattern that sets the default (omitted = Mon–Fri)
```

- `weekday` / `weekend` / `holiday` are **questions to a single layer**
  (putting a holiday into `business_days` makes it a working day, but it
  is still a holiday). `business_day` / `business_holiday` are **questions
  to the stacked conclusion** (an ordinary Saturday is in none of the
  lists, yet by the weekly pattern it matches `business_holiday`)
- The lists are independent and may overlap; overlaps are settled by the
  layer priority
- A document that uses `holiday` requires the `holidays` definition; one
  that uses `business_day` / `business_holiday` requires all three layers
  (`holidays` / `business_holidays` / `business_days`). **Using them
  undefined is a parse-time error** (never a silent "no match"). An
  explicit empty list is a legitimate statement that there are no such
  days
- Choosing vocabulary: when the meaning of a schedule must not depend on
  the reader's definition data (say, garbage collection that stops on
  public holidays), write it with the calendar/holiday words rather than
  the business words

### shift — rounding

Takes each base day selected by the `days` condition and moves it in a
fixed direction until a landing condition holds. The date version of
numeric rounding (floor / ceil).

```text
shift: [direction, "or_same"?, landing condition]
    direction: "prev" | "next"; landing condition: a day atom
    2 elements = exclusive (strictly before / after); 3 with "or_same" = inclusive
```

```jsonc
// payday: the 25th, moved earlier if it falls on a non-business day
{"days": [25], "shift": ["prev", "or_same", "business_day"], "times": ["10:00"]}
```

- Omitting `or_same` is **not a default — it is the other meaning** (the
  same distinction as java.time's `previous` / `previousOrSame`).
  Forgetting it in the payday rule produces the quiet bug "rings on the
  24th exactly in the months where the 25th is a business day"
- Consecutive non-matching days land on the same day and collapse into a
  single match (Sat/Sun/Mon of a three-day weekend → all Friday)
- `years` / `months` / `days` select the **base day only**. The landing
  day is not bound by them and may move into an adjacent month or year (a
  December base day landing in January of the next year still counts)
- The landing condition is searched up to 366 days in the given direction.
  If nothing is found within 366 days, that base day produces no points —
  it is not a parse error for the document

### if — filtering by the day itself or a neighbour

**shift moves the day; if filters without moving.**

```text
if: [direction?, "not"?, condition]
    direction: "prev" | "next" (omitted = the day itself); condition: a day atom
```

```json
{"days": ["business_day"], "if": ["next", "business_holiday"]}   // last working day before a break
{"days": ["mon"], "if": ["not", "holiday"]}                      // skip holidays (don't move)
{"days": [13], "if": ["fri"]}                                    // Friday the 13th
{"if": ["next", "last_day_of_month"]}                            // the day before the end of the month
```

Combined with `shift`, **`if` filters the base days first, then `shift`
moves what remains**.

### times / allday — the time part

- **A list = an enumeration of fixed times**: `{"times": ["09:00", "12:00"]}`
- **An object = a clock grid**: `{"times": {"every": [1, "hour"], "between": ["08:00", "20:00"]}}`
- **`"allday": true`** = a day-level event that carries no time

Semantics:

- `every` is a **clock grid** — a description of a row of clock positions
  ("on the hour", …); a late execution never moves future points. The
  count is an integer ≥ 1; the unit is `"hour" | "minute" | "second"`
  (singular, fixed). The upper bounds are one day's worth: 24 hour /
  1440 minute / 86400 second
- `between` is the **half-open interval [start, end)**. "Every hour from
  8:00 to 20:00" is the 12 points 8:00–19:00 and **20:00 does not ring**.
  The value is a window pair or `"business_hour"` (the window list of
  `calendar.business_hours`; multiple windows, e.g. with a lunch break,
  are allowed)
- **The grid anchors at the start of the window** (`["08:30", …]` gives
  8:30, 9:30, …). Omitted `between` = the whole day [00:00, 24:00). The
  grid is laid out per day and per window, carrying nothing over from the
  previous day or window: a 7-hour grid over the whole day is 00:00,
  07:00, 14:00, 21:00 every day, starting again at 00:00 the next day;
  with `business_hours` of 09:00–12:00 and 13:00–18:00, an hourly grid
  anchors at 09:00 and at 13:00
- The grid is enumerated on the local wall clock, then each point is
  resolved to an instant per RFC 5545 §3.3.5. When the DST gap folds
  several wall times onto one instant, the set contains that one point;
  a wall time that occurs twice in the overlap counts only as its first
  occurrence
- A document that uses `between: "business_hour"` requires
  `calendar.business_hours`; using it undefined is a parse-time error
- `"24:00"` is a token allowed only as a window end. Windows that cross
  midnight (start ≥ end) cannot be written
- Times are **zero-padded HH:MM, fixed** (`"0:00"` is invalid; seconds
  cannot be written)
- An `allday` point is, in implementation terms, **a point at the start of
  its day (00:00)** plus an allday flag. The difference from
  `times: ["00:00"]` is that the intent "any time that day" is preserved

### every (directly on a schedule) — an interval sequence from `from`

The third form, for intervals that do not decompose into days and times
(every 36 hours = 129600 seconds = 1.5 days, …). Mutually exclusive with
`times` / `allday`; written directly on the schedule.

```jsonc
// every 7 hours anchored at 7/17 10:00: 10:00, 17:00, 00:00 next day, 07:00, …
{"from": "2026-07-17 10:00", "every": [7, "hour"]}

// every 36 hours
{"from": "2026-07-14 00:00", "every": [36, "hour"]}
```

- The points are **from + k × interval** (k = 0, 1, 2, …); `from` is the
  first point. The row is laid out by wall-clock arithmetic and each point
  resolved per RFC 5545 §3.3.5. **It keeps counting across days — unlike
  the clock grid there is no per-day re-anchoring**
- **`from` is required** (a sequence has no definition without its
  anchor). `until` is optional and clips [from, until) as everywhere else
- The unit is `"hour" | "minute" | "second"` (singular, fixed); the count
  is an integer ≥ 1 with **no upper bound** — the grid's one-day cap is a
  consequence of its per-day re-anchoring semantics and does not apply to
  a from-anchored sequence
- The unit `"day"` is invalid here. Whole-day cycles belong to the
  calendar vocabulary (the `["every", N, "day"]` atom × `times`), and
  `from` + `every` 48 hour is **not** a substitute for "every 2 days at
  03:00" (it chains the ringing time to the time of `from`, so the
  validity start and the firing time can no longer be chosen
  independently)
- Cannot be combined with `days` / `months` / `years` / `shift` / `if`
  (it is a sequence of points, not a product of matching days × times, so
  it does not take the date axes). If a real need appears, it will be
  considered later in the closed-set manner
- Because the arithmetic is on the wall clock, real elapsed time deviates
  at DST transitions (a 36-hour sequence is really 35/37 hours on a
  transition day, keeping the wall-clock row intact). "Exactly every N
  seconds of real time" remains unsupported (the same line as the
  unsupported relative intervals)

## Deliberately unsupported

Closed sets can be widened compatibly, so these are added if and when a
real need appears.

- **Year cycles** (true biennial). Unlike the day cycle
  (`["every", N, "day"]`), years do not fold into a day count because
  their lengths differ. A strict fortnight is `["every", 14, "day"]`; the
  everyday sense of "biweekly" is usually expressible as "1st and 3rd
  Friday"
- **Relative intervals** (N seconds since the last run). That is
  throttling, not a schedule (the caller's concern). The last run time
  appears only in the caller's question interval, never in the definition
  of the points
- **Computed dates anchored to a fixed date** ("20 years after that
  day"). Write the folded result; preserving the provenance is the
  producer's responsibility
- **Windows that cross midnight**, **per-weekday business hours**,
  **user-defined window names**, **definition macros** (names referring
  to names)

## Constraints beyond the schema

The authoritative syntax is the JSON Schemas, but the following cannot be
expressed there and are validated by implementations at parse time:

- Resolvability of custom references and resolver names (undefined or
  unregistered is an error)
- Presence of the calendar entries required by the calendar vocabulary in
  use
- start < end for every time window, and non-overlap between windows
  (half-open, so touching is legal)
- Every date literal is a real date (`2026-02-30` is well-formed but
  invalid); the date part of `from` / `until` likewise
- from < until
- Presence of `from` in a schedule that uses `["every", N, "day"]`
  (cross-field constraints are outside the schema's reach; the `from` of
  the interval `every` is required by the schema as well)
- Existence of the timezone

## Examples

```jsonc
// the third Monday of every month at 10:00
{"days": [["3rd", "mon"]], "times": ["10:00"]}

// Mon–Fri, hourly from 8:00 to 20:00 (20:00 does not ring — half-open)
{"days": ["mon", "tue", "wed", "thu", "fri"],
 "times": {"every": [1, "hour"], "between": ["08:00", "20:00"]}}

// 8:00 on the day before a break (= the last business day before closed days)
{"days": ["business_day"], "if": ["next", "business_holiday"], "times": ["08:00"]}

// every 600 seconds
{"times": {"every": [600, "second"]}}

// payday: the 25th, moved earlier on non-business days
{"days": [25], "shift": ["prev", "or_same", "business_day"], "times": ["10:00"]}

// last business day of the month at 17:00
{"days": ["last_day_of_month"], "shift": ["prev", "or_same", "business_day"], "times": ["17:00"]}

// garbage collection: 1st and 3rd Friday, skipped on holidays
{"days": [["1st", "fri"], ["3rd", "fri"]], "if": ["not", "holiday"], "times": ["07:30"]}

// water bill: the 27th of even months, moved later on non-business days
{"months": [2, 4, 6, 8, 10, 12], "days": [27],
 "shift": ["next", "or_same", "business_day"], "times": ["09:00"]}

// a golden wedding (one specific day)
{"years": [2043], "months": [6], "days": [15], "allday": true}

// every 2 days at 03:00 (anchored 7/14)
{"from": "2026-07-14 00:00", "days": [["every", 2, "day"]], "times": ["03:00"]}

// medication every other day, morning and evening
{"from": "2026-07-14 00:00", "days": [["every", 2, "day"]], "times": ["08:00", "20:00"]}

// an hourly task starting 7/15 at 10:00
{"from": "2026-07-15 10:00", "every": [1, "hour"]}

// every 36 hours
{"from": "2026-07-14 00:00", "every": [36, "hour"]}

// 8:00 on working days, 10:00 on days off (the schedules list = OR)
[{"days": ["business_day"], "times": ["08:00"]},
 {"days": ["business_holiday"], "times": ["10:00"]}]
```
