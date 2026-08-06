---
title: Concepts
description: Where Yarunoka comes from, and why the Yrnk language is shaped the way it is.
sidebar:
  order: 2
---

## The question in the name

Yarunoka is the Japanese question **やるのか？** (*yaru no ka?*) —
roughly "so, do we do it?". That is the question the whole project
exists to answer: here is a schedule, here is a moment — do we do it?
**Yarunoka** is the project; **Yrnk** is the notation it defines.

## Calendar rules, not clock rules

Real-world schedules are calendar rules. "Payday is the 25th — moved up
to the previous business day when it falls on a day off." "Collection
day is the first and third Friday, skipped on holidays." "The poller
runs every 90 minutes, but only within business hours." None of these
is a clock rule: they lean on what a *business day*, a *holiday*, an
*ordinal weekday* mean, and those meanings belong to calendars and
organizations, not to arithmetic on timestamps.

Cron expressions and plain timestamps cannot carry such rules, so the
rules end up as application code — a shift rule here, a holiday lookup
there. Code in that position is hard to store, hard to display back to
the person who asked for the rule, and impossible for them to edit
safely. Yarunoka began inside a larger application whose schedules were
exactly these ordinary calendar rules, and nothing about them was
specific to that application — so the language was extracted and
specified on its own.

## A description of a set of occurrences

A Yrnk document is a pure description of a **set of occurrences** —
points in time, or whole days. It is data, not behavior: nothing in the
language can start a job, remember a last run, or catch up on missed
work. "Should this fire", "last ran at", "run it twice to make up" are
simply not words this language has.

That line is deliberate, and it is what keeps an engine **pure**. A
document describes a set of points; answers appear only when a caller
asks a **query**. The evaluation model defines three: is this instant
an occurrence (a judgment at a point)? was there an occurrence after
the last run, through now (a judgment over a period)? which occurrences
lie in this range (an enumeration)? Everything stateful — firing,
retries, throttling — stays on the caller's side of the line, where it
belongs.

## Definitions and expressions

A business day is not a weekday. Weekdays are determined by the
calendar alone; business days are decided by an organization — its
holidays, its own closures, its weekly pattern. So a document is split
in two:

- the **calendar** holds the definitions — holidays, the organization's
  own closures and extra working days, the weekly pattern, business
  hours, and named date sets
- the **schedules** are pure expressions — they refer to that
  vocabulary by name and carry no definitions of their own

The same schedule — `{"days": [25], "shift": ["prev", "or_same",
"business_day"], "times": ["10:00"]}` — means the right thing in every
organization, because *whose* business days it runs on is the
calendar's decision. The vocabulary is answered by stacked layers
(`business_days` over `business_holidays` over `holidays` over the
`workweek`), so the one question "is this a working day?" has one place
to be decided.

## The document is authoritative

A document declares how to read it. The timezone is written in the
document — the host's locale or default timezone never changes what a
schedule means. The spec version is written in the document, and an
implementation that does not know that version must reject the document
rather than guess.

What a document cannot contain — this year's public holidays as
computed by a library, closures kept in a database — it declares
instead: **resolvers** name what the host must supply. The list is
complete by rule: every name that is used and not defined must be
declared, so what a document needs is readable from the document alone.
Dynamic data flows in, while the document keeps the *intent* — "the
days this holiday source says are holidays", not the days that happened
to be true when it was written.

## One spelling, closed sets

Yrnk is strict on purpose:

- every meaning has exactly **one spelling** — no scalar shorthand for
  one-element lists, zero-padded times, a single way to write "requires
  nothing"
- every vocabulary is a **closed set** — unknown keys and unknown words
  are errors, never silently ignored

One spelling means round-tripping a valid document is the identity,
which gives implementations a canonical form to be checked against,
mechanically. Closed sets mean a mistake cannot be quietly read as
something else — and they widen compatibly: additions raise the minor
version, and every existing document keeps its meaning.

## Two planes that never cross

A schedule with days and times is a product of two orthogonal planes:
days are selected with no knowledge of times, and times are laid on the
selected days without moving them. Nothing in the language can make a
day selection depend on a time, or a time depend on how its day was
chosen. Rules that would cross the planes ("two hours after the same
time on the previous business day") are deliberately not expressible —
the orthogonality is a design decision, and the specification keeps an
explicit list of such deliberate omissions.

## A specification above the implementations

Yarunoka puts the specification at the top: the language is defined
here, language implementations conform to it, and their agreement is
checked by a shared conformance suite rather than by comparing
implementations to each other. Two programs in different languages
reading the same document reach the same set of occurrences because
both answer to this specification — and that is what lets a schedule
written once be stored, shipped, and judged anywhere.
