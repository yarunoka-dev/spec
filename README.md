# Yarunoka Specification

Yarunoka is a calendar-aware schedule DSL. Schedules — "the third Monday
of every month at 10:00", "payday on the 25th, moved earlier on
non-business days" — are written in JSON, as pure descriptions of sets of
points in time. The notation is called **Yrnk**.

This repository is the language-independent specification of Yrnk.
Language implementations conform to what is defined here.

## Layout

| Path | Content |
|---|---|
| [`schema/`](schema/) | JSON Schemas (draft 2020-12) — the authoritative syntax. Split into `document` / `calendar` / `schedule` / `primitives` so each unit is independently usable |
| [`docs/`](docs/) | The language documentation — the concepts behind the language, the specification, the reference, and the implementer's guides |

## Versioning

The spec version is an `x.y` string: `y` changes are compatible additions,
`x` changes are breaking. A document declares the version it is written
against in its `version` field. See the
[Versioning section](docs/specification.md#versioning) of the
specification.

**Released versions are the git tags of this repository** (`1.0` is the
first); `main` may carry changes not yet in any release.

## License

[MIT](LICENSE)
