---
title: Guides
description: How to develop a language implementation of Yrnk, and how to verify its conformance.
sidebar:
  order: 5
---

## What a language implementation is

An implementation of Yrnk reads documents into its language's own
types, answers the three queries of the evaluation model — the judgment
at a point, the judgment over a period, the enumeration — and,
optionally, writes documents back out through a serializer. The engine
it exposes is pure: it executes nothing and stores nothing.

## Build against the spec, not another implementation

- **Syntax** — the JSON Schemas under `schema/` render the structural
  syntax as far as JSON Schema can express it; the specification is the
  authority. An implementation carries a verbatim copy of them, so
  validation needs no network and pins the exact spec version
- **Semantics** — the [specification](../specification/) defines what a
  document means. Validation is two stages: structural validation
  against the schemas, plus the specification's document validation
  rules that the schemas cannot express, applied at parse time
- A document whose declared `version` the implementation does not know
  must be rejected, never guessed at

## Verify with the conformance kit

The spec side authors a conformance suite — language-independent cases
that check "source text + query + resolver bindings → answer" under the
adapter's zone rules. The cases ship embedded in
[**yarunoka-test**](https://github.com/yarunoka-dev/test), a single-binary
runner.

The implementer writes one small **adapter**: an executable the runner
starts once per case, which reads a request (document + query +
bindings) on stdin, hands it to the implementation unvalidated and
unmodified, and answers on stdout. The adapter contract is documented
in the kit's repository
([docs/protocol.md](https://github.com/yarunoka-dev/test/blob/main/docs/protocol.md)).
The adapter's zone rules and the request's resolver bindings form the
evaluation environment for that case.

```console
$ yarunoka-test eval php adapter.php
```

The runner exits non-zero unless every case passes, so this one line is
the whole CI integration. The modes:

- **eval** — the three queries of the evaluation model
- **emit** — the round-trip spelling check against the document as
  authored, for implementations with a serializer
- **all** — both

## Claim conformance

Passing eval and passing emit are independent claims; an
evaluation-only implementation ships with eval alone. State the claim
as the kit prints it — kit version, targeted spec version, mode passed
— and declare in the implementation's documentation which spec version
(`x.y`) it targets.
