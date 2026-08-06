---
title: Reference
description: The Yrnk language in tables — keys, fields, atoms, vocabulary, forms, and literals.
sidebar:
  order: 4
---

This page restates the language as lookup tables. The
[specification](../specification/) is the normative text; where wording
differs, the specification governs.

## Document keys

| Key | Required | Value |
|---|---|---|
| `version` | ✓ | The spec version the document is written against, as an `"x.y"` string |
| `timezone` | ✓ | An IANA Time Zone Database name (`"Asia/Tokyo"`, `"UTC"`); fixed offsets are invalid |
| `resolvers` | | Non-empty list of names the host must bind; omitted when there are none |
| `calendar` | | The definitions (below) |
| `schedules` | ✓ | Non-empty list of schedules, combined with OR |

Unknown keys are an error at every level — document, calendar,
schedule, and the times object.

## Calendar keys

| Key | Value | Role |
|---|---|---|
| `business_days` | date list or name | Top layer: days the organization works, overriding everything below |
| `business_holidays` | date list or name | The organization's own closures |
| `holidays` | date list or name | Public holidays |
| `workweek` | non-empty list of day names | Bottom layer: the weekly default (omitted = `mon`–`fri`) |
| `business_hours` | non-empty list of windows | The window list behind `"business_hour"` |
| `date_sets` | object: name → date list | The open namespace of the document's own named date lists (no layer semantics) |

- A date-list position accepts exactly two forms: an array of date
  literals, or a name (a `date_sets` entry, or a name the host binds to
  a resolver). A date-shaped string matches neither form
- A date list may be empty — an explicit empty list states that there
  are no such days

## Calendar vocabulary

| Word | Asks |
|---|---|
| `weekday` | day-of-week ∈ Mon–Fri — fixed; `workweek` plays no part |
| `weekend` | day-of-week ∈ Sat–Sun — fixed |
| `holiday` | date ∈ `holidays`, that list alone |
| `business_day` | the stacked conclusion (procedure below) |
| `business_holiday` | the exact complement of `business_day` |

```text
business_day(date):
    if date ∈ business_days:      true     — the top layer wins
    if date ∈ business_holidays:  false
    if date ∈ holidays:           false
    otherwise:                    day-of-week(date) ∈ workweek
                                  (workweek omitted = mon–fri)
```

`holiday` requires the `holidays` definition; `business_day` /
`business_holiday` require all three of `holidays` /
`business_holidays` / `business_days`. Using a word without its
required definitions is a document validation error.

## Schedule fields

| Field | Value | Notes |
|---|---|---|
| `years` | non-empty list of integers 1–9999 | Date axis |
| `months` | non-empty list of integers 1–12 | Date axis |
| `days` | non-empty list of day atoms | Date axis |
| `if` | filter tuple | Filters base days without moving them; applied before `shift` |
| `shift` | shift tuple | Moves each base day until a landing condition holds |
| `times` | list or grid object | Time part |
| `allday` | `true` | Time part: a day-level occurrence |
| `every` | `[count, unit]` | Time part: an interval sequence anchored at `from` |
| `from` | `"YYYY-MM-DD HH:MM"` | Validity start, inclusive |
| `until` | `"YYYY-MM-DD HH:MM"` | Validity end, exclusive |

- **Exactly one of `times` / `allday` / `every` is required**
- A schedule with a top-level `every` takes no `years` / `months` /
  `days` / `shift` / `if`, and requires `from`
- `from` is also required by the `["every", N, "day"]` day atom
- An omitted date axis means no restriction on that axis
- The algebra: within an axis's array — OR; between fields — AND;
  between schedules — OR

## Day atoms

| Atom | Example | Selects |
|---|---|---|
| Number 1–31 | `25` | That day of the month (a day the month does not have simply does not match) |
| Day name | `"mon"` | That weekday |
| Calendar word | `"holiday"` | The five vocabulary words above |
| Ordinal tuple | `["3rd", "mon"]` | The Nth / last such weekday of the month |
| End of month | `"last_day_of_month"` | The last day of the month |
| Day-cycle tuple | `["every", 2, "day"]` | Every Nth day, counting the date of `from` as day one |
| Name | `"founding-day"` | Membership in the named date set |

- Ordinals: `"1st"` `"2nd"` `"3rd"` `"4th"` `"5th"` `"last"`
- Day names: `"mon"` `"tue"` `"wed"` `"thu"` `"fri"` `"sat"` `"sun"`
- The day-cycle count is an integer ≥ 1; the unit is the fixed `"day"`.
  The tuple is allowed only as an element of `days`, not as a `shift`
  landing condition or an `if` condition
- A date literal cannot be written in `days` — give the date a name
  under `date_sets` and refer to it
- There are no negative day numbers

## shift

```text
["prev" | "next", atom]                exclusive: strictly before / after
["prev" | "next", "or_same", atom]     inclusive: the base day itself is tested first
```

- The landing condition is a day atom (not a day-cycle tuple)
- Maximum displacement: 366 calendar days. A base day whose landing
  condition never holds within that range produces no occurrences
- Consecutive base days may land on the same day and collapse into a
  single match
- The landing day is not bound by the date axes and may move into an
  adjacent month or year

## if

```text
[atom]                            the day itself matches
["not", atom]                     the day itself does not match
["prev" | "next", atom]           the neighbouring day matches
["prev" | "next", "not", atom]    the neighbouring day does not match
```

`if` filters the base days first; `shift` then moves what remains.

## times

| Form | Example | Meaning |
|---|---|---|
| List | `["09:00", "12:00"]` | Non-empty enumeration of fixed times |
| Grid | `{"every": [1, "hour"], "between": ["08:00", "20:00"]}` | Clock positions, laid out per day and per window |
| All-day | `"allday": true` | A day-level occurrence; time does not apply to it |

The grid object:

- `every` — `[count, unit]`. Unit and maximum count: `"hour"` 24,
  `"minute"` 1,440, `"second"` 86,400 (one day's worth in each unit);
  the count is an integer ≥ 1
- `between` — a window (half-open `[start, end)`), or
  `"business_hour"` (the window list of `calendar.business_hours`,
  which must then be defined). Omitted = the whole day [00:00, 24:00)
- The grid anchors at the start of each window and carries nothing over
  from the previous day or window

## every (directly on a schedule)

`[count, unit]` — the points from + k × interval (k = 0, 1, 2, …),
counting across days with no per-day re-anchoring.

- Unit: `"hour" | "minute" | "second"`; the count is an integer ≥ 1
  with no upper bound. The unit `"day"` is invalid here — whole-day
  cycles are the `["every", N, "day"]` atom combined with `times`
- `from` is required; `until` is optional

## Literal forms

| Kind | Form | Notes |
|---|---|---|
| Date | `"YYYY-MM-DD"` | Proleptic Gregorian; years 1–9999; must be a real date |
| Time | `"HH:MM"` | Zero-padded; no seconds; `"24:00"` is a token allowed only as a window end |
| Window | `["HH:MM", "HH:MM"]` | Half-open [start, end); start < end; windows must not overlap (touching is legal); cannot cross midnight |
| Date-time | `"YYYY-MM-DD HH:MM"` | The `from` / `until` form; zero-padded, a single space (U+0020), no seconds, no `24:00` |
| Timezone | IANA name | `"Asia/Tokyo"`, `"UTC"`; fixed offsets (`"+09:00"`) are rejected |
| Version | `"x.y"` | |
| Day of week | `"mon"`–`"sun"` | |

## Names

A name — a `date_sets` key, a declared resolver, or the reference
written in a date-list position or in `days` — must be a non-empty
string that is **not**: digits only, time-shaped (`HH:MM`), date-shaped
(`YYYY-MM-DD`), or a reserved word. All names share one namespace: a
name must not be both a `date_sets` key and a declared resolver, and a
name that is used but neither defined nor declared is a document
validation error.

The reserved words:

```text
weekday weekend holiday business_day business_holiday business_hour
mon tue wed thu fri sat sun
1st 2nd 3rd 4th 5th last
last_day_of_month
not prev next or_same
hour minute second day
version timezone resolvers calendar schedules
years months days shift if times allday every between from until
holidays business_holidays business_days workweek business_hours date_sets
```

## Validation beyond the schema

In addition to structural JSON Schema validation, implementations
validate:

- every used name resolves — a `date_sets` entry or a declared
  resolver; no name is both; every declared name is bound by the host
- the calendar entries required by the calendar vocabulary in use are
  present
- start < end for every window, and windows do not overlap
- every date literal is a real date (`2026-02-30` is well-formed but
  invalid)
- the resolved instant of `from` is strictly earlier than the resolved
  instant of `until`
- `from` is present in a schedule that uses `["every", N, "day"]`
- the timezone name exists in the IANA Time Zone Database as available
  to the implementation; fixed-offset strings are rejected

## Deliberately unsupported

Year cycles · relative intervals ("N seconds since the last run") ·
computed dates anchored to a fixed date · anything that crosses the
date plane and the time plane · windows that cross midnight ·
per-weekday business hours · user-defined window names · definition
macros. The reasoning is in the specification's
[Deliberately unsupported](../specification/#deliberately-unsupported)
section.
