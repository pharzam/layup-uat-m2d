# tests/

The home for the **product tests** of `tally` that exercise the built command as
a whole — the end-to-end tests, and the speed and memory tests of the quality
targets. It is empty until the first one lands: no Go module and no product code
exist yet. This directory is the default `‹test directory›`; Go's own convention
puts the unit and integration tests of a package beside its source, as
`*_test.go` files, so the tests of package `count` live with package `count`.

## In plain terms

> Put the tests that run the finished `tally` command here: give it files, read
> its rows, its messages and its exit code. Tests of the counting functions on
> their own sit next to the counting code, as Go expects. The rules for what to
> write and how live one folder over, in `docs/tests/`.

## What goes here

- **End-to-end tests** of the built command: the flags and their fixed order,
  standard input, a file named `-`, a directory, an unreadable file, a file that
  is not valid UTF-8, the `total` row and the exit code.
- **The speed test**: a generated file of 10 MiB, counted by the built command,
  timed from the start of the process to its exit.
- **The memory test**: a generated file of 1 GiB, counted while the heap is
  sampled every 10 milliseconds.

The generated files are written to a temporary directory and removed in every
outcome; none is committed. The targets and how each is measured are in
[`docs/guardrails.md`](../docs/guardrails.md).

## Where the conventions live

This directory holds the tests; the **conventions** for writing them live in
[`docs/tests/`](../docs/tests/):

- [`docs/tests/test-levels.md`](../docs/tests/test-levels.md) — the test levels and
  the command placeholders.
- [`docs/tests/template-unit.md`](../docs/tests/template-unit.md),
  [`template-integration.md`](../docs/tests/template-integration.md),
  [`template-e2e.md`](../docs/tests/template-e2e.md) — a pattern to copy per level.
- [`docs/tests/traceability-template.md`](../docs/tests/traceability-template.md) —
  the row that ties each test back to the requirement it proves.

The [discipline tests](../docs/tests/test-levels.md) — the ADR, PRD,
link, PR-link and review-record linters — are not product tests and do
**not** live here; they stay beside the conventions they enforce, under
[`docs/`](../docs/).

> **When the first test lands.** Add it here, and fill the `‹test
> directory›` placeholder in [`docs/tests/test-levels.md`](../docs/tests/test-levels.md)
> and the [hook](../.githooks/pre-commit)/[CI](../docs/ci/) steps to point at it.
> The `.gitkeep` file only exists to keep this empty directory in git — delete it
> in the same change. Replace this note once the tests are in.
