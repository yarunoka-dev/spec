---
title: Yrnk Schedule DSL
description: The language-independent documentation of Yrnk, the Yarunoka schedule DSL.
sidebar:
  order: 1
---

Yrnk is a JSON DSL for describing calendar-aware schedules — "the third
Monday of every month at 10:00", "payday on the 25th, moved earlier on
non-business days" — as pure descriptions of sets of points in time.
**Yrnk** is short for Yarunoka: Yarunoka is the project, and Yrnk is the
notation.

This documentation defines the language itself, independently of any
implementation:

- **[Concepts](concepts)** — where Yarunoka comes from, and why the
  language is shaped the way it is
- **[Specification](specification)** — the normative text: the document
  model and the semantics of the language
- **[Reference](reference)** — the language in tables: keys, fields,
  atoms, vocabulary, and literal forms
- **[Guides](guides)** — how to develop a language implementation and
  verify its conformance

The JSON Schemas the repository ships under `schema/` define the
structural syntax; the specification defines the semantics.
Implementations must conform to both.
