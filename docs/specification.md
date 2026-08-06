---
title: Specification
description: The Yrnk schedule DSL, version 1.0 — the normative definition of the language, its syntax and its semantics.
sidebar:
  order: 3
---

Yrnk is a JSON DSL for describing calendar-aware schedules. **Yrnk** is
short for Yarunoka: Yarunoka is the project, and Yrnk is the notation. The
JSON Schemas under
[`schema/`](https://github.com/yarunoka-dev/spec/tree/1.0/schema)
(JSON Schema draft 2020-12)
define the structural syntax; this document defines the language — its
syntax and its semantics. Implementations must conform to both, and their
agreement is verified by tests.

A Yrnk document is a **description of a set of occurrences** — points in
time, or whole days — and knows nothing about execution. "Should this fire", "last run at", and
"catch-up" do not exist in this language's vocabulary — they are the
caller's concern, expressed through the queries the caller asks.

## Conventions

How this document writes time boundaries. In interval notation, a
square bracket includes the endpoint and a parenthesis excludes it. In
words, the boundary words carry the same distinctions and are used with
exactly these meanings throughout:

| In words | In notation | Start | End |
|---|---|---|---|
| from a through b | [a, b] | included | included |
| from a until b | [a, b) | included | excluded |
| after a, through b | (a, b] | excluded | included |
| after a, before b | (a, b) | excluded | excluded |

## Document model

A document is a JSON object with three layers:

- **Reading directives** — `version` and `timezone` declare *how to
  interpret* the document before any content is read
- **External requirements** — `resolvers` declares what the document
  leaves to its host. The only optional layer: a document that resolves
  every name itself has nothing to declare
- **Content** — `calendar` (the definitions) and `schedules` (the
  expressions)

```json
{
  "version": "1.0",
  "timezone": "Asia/Tokyo",
  "resolvers": ["yasumi-jp"],
  "calendar": {
    "holidays": "yasumi-jp",
    "date_sets": { "founding-day": ["2026-10-01"] }
  },
  "schedules": [
    {"days": ["founding-day"], "allday": true}
  ]
}
```

| Key | Required | Meaning |
|---|---|---|
| `version` | ✓ | The spec version this document is written against, as an `"x.y"` string. Implementations must reject versions they do not know rather than silently interpreting them |
| `timezone` | ✓ | The timezone in which every schedule is interpreted. **The document, not the host's default timezone or locale, is authoritative** (resolver-backed names additionally depend on the supplied bindings — see the resolvers section — and the wall-to-instant mapping follows the implementation's tz database) |
| `resolvers` | | The names this document leaves to its host to resolve (see below). Omitted when there are none |
| `calendar` | | The definitions part (see below) |
| `schedules` | ✓ | The list of schedules. **The list is an OR of complete schedules** (a bare object is not allowed) |

- Unknown keys are an error (closed set — the same rule applies at the
  document, calendar, schedule, and times levels)
- `timezone` is an **IANA Time Zone Database name** (`Asia/Tokyo`,
  `UTC`). Fixed offsets (`+09:00`) are not accepted — a document anchored
  to UTC writes `"UTC"`. Whether a name exists is checked against the
  implementation's tz database. Zones with daylight-saving transitions
  are allowed. Wall-clock times that
  fall on a transition are resolved **per RFC 5545 §3.3.5** — a time that
  does not exist (a gap: the spring-forward hour, or the whole day a zone
  skips when it moves across the date line) is interpreted with the offset
  in effect before the transition, which pushes it forward in real time; a
  time that occurs twice (the fall-back overlap) counts only as its first
  occurrence
- Arrays appear in two roles, distinguished by position. **List positions**
  (`years` / `months` / `days`, a `times` list, `workweek`,
  `business_hours`, date lists, `resolvers`, `schedules`) hold
  enumerations: the array
  enumerates members, and for the date axes omission means no restriction
  on that axis. There is no scalar sugar — `"days": "mon"` and
  `"months": 2` are invalid; a single member is still written as a
  one-element array (so that the same value never has two spellings, and
  round-tripping is the identity). **Tuple positions** (`shift`, `if`, the
  ordinal and day-cycle tuples, both `every` forms, a window) hold
  fixed-arity tuples whose elements are read by position, as defined in
  their sections
- Dates follow the **proleptic Gregorian calendar**; years run 1–9999
- Enumerations reject duplicate members. The date axes, a `times` list,
  `workweek`, `business_hours`, and `resolvers` must be non-empty when
  present, and `schedules` must be non-empty. Date lists **may** be empty
  — an explicit empty list is the statement that there are no such days
- The whole DSL denotes a set of **occurrences**. An occurrence is either
  **timed** (an instant) or **all-day** (a whole day; time does not apply
  to it). The two kinds never merge: an all-day occurrence and a timed
  occurrence at 00:00 of the same day are distinct elements. Within a
  kind, when several schedules produce the same occurrence the OR
  contains it once, and a firing decision sees it once
- An all-day occurrence stands at the **start of its day** on the document
  timezone's clock. The day is chosen on the calendar; its start then
  resolves like any other wall-clock point: a midnight that occurs twice
  (the fall-back overlap) stands at its first occurrence, and where a
  zone skips a day outright the push crosses midnight and the occurrence
  stands on the day that follows — that resulting day is the one it
  denotes. Two calendar days whose starts resolve to the same instant are
  therefore one occurrence

## Versioning

The spec version is an `"x.y"` string, and the `version` field of a
document declares which spec version it is written against.

- **y is raised** for backward-compatible changes: additions to closed
  sets (new vocabulary, atoms, or fields) that leave the meaning of every
  existing document unchanged. An implementation of 1.y′ accepts documents
  written against 1.y, where y′ > y, with the same meaning
- **x is raised** for breaking changes. Compatibility with documents
  written against a lower major version is not guaranteed
- An implementation must reject a document whose declared version it does
  not know

The first public version is 1.0.

## resolvers — what the document leaves to its host

A name the document does not define under `calendar.date_sets` is
resolved outside the document, by whatever the host binds to that name — a
**resolver**. `resolvers` is where the document says which names those
are.

```json
"resolvers": ["yasumi-jp", "company-closures"]
```

- **Every name that is used and not defined must be declared here.** A
  name that appears in the document, is not a `date_sets` entry, and is
  not listed is a document validation error; the language never leaves a
  name silently to the host. The list is therefore complete — what a host
  must bind is exactly what it says. That completeness is the point:
  nothing in the document means anything until its names resolve, so the
  requirement has to be readable from the document alone, before the
  bindings that reading it is meant to prepare exist
- **A declared name need not be used.** The declaration states what the
  document depends on, and a document may name a dependency it has not
  yet written a schedule for. An unused declaration costs the host a
  binding that is never asked for, and nothing else
- The value is an enumeration of names — non-empty, no duplicates. A
  document that leaves nothing to its host **omits the key** rather than
  writing an empty list, so that "requires nothing" has a single spelling
- A declared name is spelled like any other name — no reserved words,
  nothing shaped like a literal (see the calendar section) — and must not
  collide with a `date_sets` key

**This is how dynamic data enters a document** — a database, a holiday
computation — while the document keeps the *intent*: what the dates are
resolved by, rather than what they happened to be when it was written. A
document that declares resolvers is portable only together with its
bindings. The host's locale or default timezone never affects
interpretation, but a resolver-backed name does depend on what the host
binds it to, which is why the document states the dependency rather than
leaving it to be discovered.

How a host materializes a resolver — the call signature, and whether a
relevant range is communicated — is implementation API, outside this
language. Implementations validate that what a resolver yields is a list
of date literals (`YYYY-MM-DD`); a resolver that fails at call time is a
host-side runtime error, not a document validation error.

## calendar — the definitions

The calendar is the set of date and time-window definitions that schedules
refer to. Its top-level keys are a closed set of **reserved keys** (the
built-in definitions); under `date_sets` is an **open namespace** (the
user's own named date lists). A calendar contains wall-clock dates and times
only and defines no timezone of its own: the timezone is declared at the
document level, and a calendar — like the schedules — is written on that
premise.

```json
"calendar": {
  "holidays": ["2026-01-01", "2026-01-12"],
  "business_holidays": [],
  "business_days": [],
  "workweek": ["mon", "tue", "wed", "thu", "fri"],
  "business_hours": [["09:00", "12:00"], ["13:00", "18:00"]],
  "date_sets": {
    "founding-day": ["2026-10-01"],
    "closing-day": ["2026-12-29", "2026-12-30"]
  }
}
```

- **The built-in definitions are special**: `holidays` /
  `business_holidays` / `business_days` / `workweek` carry the layer-model
  semantics (below), and `business_hours` is the window list behind the
  `business_hour` vocabulary. `date_sets` entries take no part in the
  layers; such a name is a flat "membership in a set" and nothing more
- **A `date_sets` value is a list of date literals** — nothing else. This
  is where the document *contains* the dates it names; a name whose dates
  come from elsewhere is not written here but left to a resolver, so the
  entry never stands for another name. Windows cannot be named at all: a
  date set can be large and dynamic, so naming one is a real need, but a
  window is a short literal written in place where it is used, so an
  aliasing mechanism would buy nothing. The only shared window list is
  the built-in `business_hours`. A `date_sets` key follows the rules every
  name follows: it must not collide with reserved words and must not
  look like a literal (digits only, date-shaped, time-shaped)
- **Name references**: wherever a date list is expected, a **name** may be
  written instead of the array (`"holidays": "yasumi-jp"`). A date-list
  position accepts exactly two forms: an array of date literals, or a
  name. Names must not be date-shaped, so a bare date-shaped string
  (`"holidays": "2026-01-01"`) matches neither form and is invalid — it is
  not read as a one-date list. This is a distinction of kind, not scalar
  sugar
- **A name denotes a date set, and all names share one namespace.** A name
  is resolved one of two ways: **inside** the document, by a `date_sets`
  entry that carries the date list itself, or **outside** it, by a binding
  the host supplies for that name (a **resolver**). Which of the two a
  name is makes no difference to where it may be written — every date-list
  position takes a name of either kind, and so does the `days` axis. A
  `date_sets` key and a resolver name must therefore never be the same
  string. A name that is neither a `date_sets` entry nor declared under
  `resolvers` is a document validation error

## Schedule

```json
{"days": [25], "shift": ["prev", "or_same", "business_day"], "times": ["10:00"]}
```

The fields are `years` / `months` / `days` (the date axes), `shift` / `if`
(the date modifiers), `times` | `allday` | `every` (the time part —
**exactly one is required**), and `from` / `until` (the validity range).

The algebra has three tiers: alternatives within a date-axis array are
combined with OR, fields within one schedule are combined with AND, and
complete schedules within the `schedules` list are combined with OR.

A schedule that uses `times` or `allday` is a product of two orthogonal
planes: the **date plane** (`years` / `months` / `days`, filtered by
`if`, moved by `shift`) and the **time plane** (`times` | `allday`).
Days are selected with no knowledge of times, and times are laid on the
selected days without moving them. A schedule with a top-level `every`
is the third form — a from-anchored sequence of points that takes no
date axes at all (see its section).

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
  single space — U+0020 — no seconds). A date-only `"YYYY-MM-DD"` is invalid — if
  omission meant 00:00, the same instant would have two spellings, so a
  range starting at the top of a day writes `00:00` explicitly. `24:00`
  does not exist in this position (write the next day's `00:00`; the
  interval being half-open makes that mean exactly "through that day")
- Interpretation uses the document `timezone`. A boundary on a transition
  resolves by the same rule as scheduled points (RFC 5545 §3.3.5): a wall
  time that does not exist (a gap) is pushed forward, and one that occurs
  twice (the fall-back overlap) stands at its first occurrence
- A point at `from` is included; a point at `until` is not (the same
  half-open convention as `between`)
- Each is independent (only `from`, only `until`, or both). With both,
  the resolved instant of `from` must be strictly earlier than the
  resolved instant of `until`. **Vocabulary that counts** (the
  `["every", N, "day"]` atom and the interval `every`) requires `from`
  (there is no way to start counting without it); otherwise both are
  optional
- A timed occurrence is clipped by its instant. An all-day occurrence is
  clipped by its day: it survives when that day overlaps the range at all,
  so with `from: "2026-07-14 12:00"` the all-day occurrence of 7/14 stays
  — the day is still partly inside. A day carries no time, so asking
  whether it starts before a boundary is asking a question it cannot
  answer

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
| Name | `"founding-day"` | a reference to a date set — a `calendar.date_sets` entry, or a name the host binds to a resolver |

- A day number a month does not have **simply does not match** — `[31]`
  in a 30-day month produces no occurrence, exactly as a `5th` tuple in a
  month without a fifth week does not. Evaluation is a predicate over the
  days a month really has, so a nonexistent day never rolls over: a date
  library asked for April 31st may quietly answer May 1st, and that day
  is not an occurrence
- A day number is therefore not a spelling of the end of the month —
  `[31]` reaches only the months that have a 31st. The end of the month
  is written `last_day_of_month`
- The ordinals are the six words `1st`–`5th`, `last`. In a month without a
  fifth week, a `5th` tuple simply does not match. A tuple is always
  written as an element of the enumeration (`{"days": [["3rd", "mon"]]}`)
- A date literal (`"2026-10-01"`) cannot be written directly in `days`.
  Give the date a name under `date_sets` and refer to it
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

- `weekday` / `weekend` ask the **fixed calendar** and consult no
  definition: weekday is always Monday–Friday and weekend always
  Saturday–Sunday, regardless of `workweek` (with a Tue–Sat workweek,
  Monday is still a weekday — and also a business holiday). `holiday`
  asks the `holidays` list **alone** (putting a holiday into
  `business_days` makes it a working day, but it is still a holiday)
- `business_day` / `business_holiday` are **questions to the stacked
  conclusion** (an ordinary Saturday is in none of the lists, yet by the
  weekly pattern it matches `business_holiday`). `business_holiday` is
  the exact complement of `business_day`: every day is exactly one of
  the two. The decision procedure:

```text
business_day(date):
    if date ∈ business_days:      true     — the top layer wins
    if date ∈ business_holidays:  false
    if date ∈ holidays:           false
    otherwise:                    day-of-week(date) ∈ workweek
                                  (workweek omitted = mon–fri)

business_holiday(date) = not business_day(date)
weekday(date)          = day-of-week(date) ∈ {mon … fri}   (workweek plays no part)
weekend(date)          = day-of-week(date) ∈ {sat, sun}
holiday(date)          = date ∈ holidays
```
- The lists are independent and may overlap; overlaps are settled by the
  layer priority
- A document that uses `holiday` requires the `holidays` definition; one
  that uses `business_day` / `business_holiday` requires all three layers
  (`holidays` / `business_holidays` / `business_days`). **Using them
  without the required definitions is a document validation error** (never
  a silent "no match"). An
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
// (if the 25th is a business day, that day itself)
{"days": [25], "shift": ["prev", "or_same", "business_day"], "times": ["10:00"]}
```

- Omitting `or_same` is **not a default — it is the other meaning** (the
  same distinction as java.time's `previous` / `previousOrSame`).
  Forgetting it in the payday rule produces the quiet bug of an occurrence
  on the 24th exactly in the months where the 25th is a business day
- Consecutive non-matching days land on the same day and collapse into a
  single match (Sat/Sun/Mon of a three-day weekend → all Friday)
- `years` / `months` / `days` select the **base day only**. The landing
  day is not bound by them and may move into an adjacent month or year (a
  December base day landing in January of the next year still counts)
- The maximum displacement of a shift is **366 calendar days**; with
  `or_same`, displacement zero (the base day itself) is tested first. A
  base day whose landing condition never holds within that range produces
  no occurrences — it does not invalidate the document

### if — filtering by the day itself or a neighbour

**shift moves the day; if filters without moving.**

```text
if: [direction?, "not"?, condition]
    direction: "prev" | "next" (omitted = the day itself); condition: a day atom
```

```jsonc
// last working day before a break
{"days": ["business_day"], "if": ["next", "business_holiday"], "times": ["09:00"]}

// skip holidays without moving the occurrence
{"days": ["mon"], "if": ["not", "holiday"], "times": ["09:00"]}

// Friday the 13th
{"days": [13], "if": ["fri"], "times": ["09:00"]}

// the day before the end of the month
{"if": ["next", "last_day_of_month"], "times": ["09:00"]}
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
  (singular, fixed). The maximum count is 24 for `"hour"`, 1,440 for
  `"minute"`, and 86,400 for `"second"` — one day's worth in each unit
- `between` is the **half-open interval [start, end)**. "Every hour from
  8:00 until 20:00" is the 12 points 8:00–19:00 and **20:00 is excluded**.
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
  resolved to an instant per RFC 5545 §3.3.5. When a gap folds
  several wall times onto one instant, the set contains that one point;
  a wall time that occurs twice in the overlap counts only as its first
  occurrence
- A document that uses `between: "business_hour"` requires
  `calendar.business_hours`; using it without that definition is a
  document validation error
- `"24:00"` is a token allowed only as a window end. Windows that cross
  midnight (start ≥ end) cannot be written
- Times are **zero-padded HH:MM, fixed** (`"0:00"` is invalid; seconds
  cannot be written)
- An `allday` occurrence is a **day-level occurrence: time does not apply
  to it**. It matches on the day alone and ignores the time. It is not
  `times: ["00:00"]` — a timed occurrence at 00:00 and
  an all-day occurrence of the same day are distinct occurrences and never
  collapse into one. Because it carries no time, every question about a
  range answers for it by **whether its day overlaps that range**, never
  by comparing a time within the day

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

## Evaluation model

The sections above define each construct; this section fixes the order in
which they combine. The order defines the **observable result** only — an
implementation may compute it however it likes, as long as the outcome is
identical.

For each schedule:

1. **Base-day selection** — the date axes select days: `years` AND
   `months` AND `days`, with the enumeration inside each axis combined by
   OR. A day-cycle atom counts its days from the date of the schedule's
   `from` (day one)
2. **`if` filters** — base days whose condition does not hold are
   removed; nothing moves
3. **`shift` moves** — each remaining base day moves in the given
   direction until the landing condition holds (with a maximum
   displacement of 366 calendar days; a base day that finds nothing
   produces no occurrences). Consecutive
   base days may land on the same day and collapse
4. **Time generation** — on each resulting day, `times` lays out its
   fixed times or its grid (anchored per day and per window), or `allday`
   produces the day-level occurrence. A schedule with a top-level `every`
   skips steps 1–4 and generates the from + k × interval sequence instead
5. **Wall-to-instant resolution** — every wall-clock point resolves to an
   instant per RFC 5545 §3.3.5 (gap: pushed forward; overlap: first
   occurrence only)
6. **Range clipping** — `from` / `until` resolve to instants by the same
   rule. A timed occurrence survives if its instant lies in
   [from, until); the comparison is between **instants**, never
   wall-clock values. An all-day occurrence survives if its **day
   overlaps** [from, until) — that is, if the range holds any instant of
   that day. A boundary written on a transition resolves like any other
   wall-clock point — a nonexistent wall time (a gap) forward, a repeated
   one (the fall-back overlap) to its first occurrence; no special
   clipping rule applies to either
7. **Union across schedules** — the document's set is the union of the
   schedules' occurrences, with duplicates collapsing within each kind
   (timed / all-day) as defined in the document model

A judgment at a point asks: is the given instant an occurrence? For a
timed occurrence the answer is instant equality — the given instant,
ignoring anything finer than a second (no scheduled point is finer),
equals the occurrence's instant. The comparison is between instants,
never wall-clock values. An all-day occurrence matches on the day alone:
the answer is yes for every instant whose local date — read in the
document timezone — is that day.

A judgment over a period asks: is there a scheduled point after the
previous run, through now? A point exactly at the start does not count —
it was the previous judgment's "now", already counted; a point exactly
at now counts in this judgment, not the next. Each judgment's "now"
becomes the next one's start, so every timed point is seen exactly once.
Within a period, an all-day occurrence counts when its day overlaps the
period — the same reading as everywhere else. A period that lies inside
one day therefore answers yes for that day's all-day occurrence, however
late in the day it is asked: a day is due for as long as it lasts. That
does not make it a timed occurrence at 00:00.

An enumeration asks: which occurrences lie from a through b? The answer
is the occurrence set cut to [a, b] — every timed occurrence whose
instant lies in the closed interval, and every all-day occurrence whose
day overlaps it. Timed occurrences are
answered as instants, all-day occurrences as dates; the two kinds stay
distinct, as everywhere, and the answer is in ascending order, an
all-day occurrence taking the start of its day as its place in the order.
When that coincides with a timed occurrence (a timed point at the day's
00:00), the all-day occurrence comes first: the day precedes the points
within it. Unlike
the period judgment — whose start is excluded because that instant was
the previous judgment's "now" — an enumeration has no previous window:
the caller names two instants, and both are part of what it names.
Adjacent windows that share a boundary instant therefore both contain a
point exactly on it; a caller that means to exclude a boundary moves it.

## Deliberately unsupported

Closed sets can be widened compatibly, so these are added if and when a
real need appears.

- **Year cycles** (true biennial). Unlike the day cycle
  (`["every", N, "day"]`), years do not fold into a day count because
  their lengths differ. A strict fortnight is `["every", 14, "day"]`.
  Many real-world "every other week" rules are in fact month-anchored —
  "1st and 3rd Friday" — which is a **different rule** (a month with five
  Fridays breaks the 14-day cadence) and is written as that form
- **Relative intervals** (N seconds since the last run). That is
  throttling, not a schedule (the caller's concern). The last run time
  appears only in the caller's question interval, never in the definition
  of the points
- **Computed dates anchored to a fixed date** ("20 years after that
  day"). Write the folded result; preserving the provenance is the
  producer's responsibility
- **Anything that crosses the date plane and the time plane**. The two
  planes are orthogonal by construction: no day selection can depend on a
  time, and no time can depend on how a day was selected ("two hours
  after the same time on the previous business day", "move to the next
  day when the time falls outside business hours" are not expressible).
  The orthogonality is a design decision, not an omission
- **Windows that cross midnight**, **per-weekday business hours**,
  **user-defined window names**, **definition macros** (names referring
  to names)

## Constraints beyond the schema

The following constraints are validated by implementations in addition to
structural JSON Schema validation. Some may be expressible in JSON Schema,
but they are defined here as semantic validation rules:

- Resolvability of every name: a name used in the document is either a
  `date_sets` entry or declared under `resolvers`, no name is both, and
  every declared name is bound by the host
- Presence of the calendar entries required by the calendar vocabulary in
  use
- start < end for every time window, and non-overlap between windows
  (half-open, so touching is legal)
- Every date literal is a real date (`2026-02-30` is well-formed but
  invalid); the date part of `from` / `until` likewise
- The resolved instant of `from` is strictly earlier than the resolved
  instant of `until`
- Presence of `from` in a schedule that uses `["every", N, "day"]`
  (cross-field constraints are outside the schema's reach; the `from` of
  the interval `every` is required by the schema as well)
- Existence of the timezone name in the IANA Time Zone Database (as
  available to the implementation); fixed-offset strings are rejected

## Examples

Unless noted otherwise, the following values are complete schedule
objects, not complete Yrnk documents.

```jsonc
// the third Monday of every month at 10:00
{"days": [["3rd", "mon"]], "times": ["10:00"]}

// Mon–Fri, hourly from 8:00 until 20:00
{"days": ["mon", "tue", "wed", "thu", "fri"],
 "times": {"every": [1, "hour"], "between": ["08:00", "20:00"]}}

// 8:00 on the day before a break (= the last business day before closed days)
{"days": ["business_day"], "if": ["next", "business_holiday"], "times": ["08:00"]}

// every 600 seconds
{"times": {"every": [600, "second"]}}

// payday: the 25th, moved to the previous business day on days off
{"days": [25], "shift": ["prev", "or_same", "business_day"], "times": ["10:00"]}

// last business day of the month at 17:00
{"days": ["last_day_of_month"], "shift": ["prev", "or_same", "business_day"], "times": ["17:00"]}

// garbage collection: 1st and 3rd Friday, skipped on holidays
{"days": [["1st", "fri"], ["3rd", "fri"]], "if": ["not", "holiday"], "times": ["07:30"]}

// water bill: the 27th of even months, moved to the next business day on days off
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

// value of `schedules`: 8:00 on working days, 10:00 on days off
[{"days": ["business_day"], "times": ["08:00"]},
 {"days": ["business_holiday"], "times": ["10:00"]}]
```
