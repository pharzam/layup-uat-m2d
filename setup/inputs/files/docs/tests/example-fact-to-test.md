# Worked example — from a fact to a test

One concrete walk down the traceability line of `tally`, so the abstract chain in
[`traceability-template.md`](traceability-template.md) has a filled example beside
it. It follows a single requirement from the idea owner's words all the way to the
test that proves it.

    fact (F-0001#8) → requirement (REQ-001) → guardrail → ADR → task (T-ab12) → test

This example needs no architecturally-significant decision, so the **ADR** link is
empty (`—`, as the [row at the end](#5-the-traceability-row-the-line-written-down)
shows); every other link is filled.

> **About this example.** The fact is real: it is in
> [`F-0001`](../facts/F-0001-setup-answers.md). The requirement ID, the task ID
> `T-ab12` and the test name are illustrative, because no PRD and no task exist
> yet; the first PRD gives the real ones, and this page is updated to match. It
> names no exact command: the test's command stays the `‹unit test
> command›` placeholder until a task selects it.

## In plain terms

> Nothing in this project gets tested by accident. Every test traces back through
> a written requirement to something the idea owner actually said. This page shows
> that trace once, in full, so the pattern is concrete: a sentence of the idea
> owner becomes a requirement, the requirement inherits a known pitfall, and a test
> proves the pitfall is handled.

## 1. The fact (Layer 1 — the idea owner's words)

A [facts record](../facts/) stores the idea owner's words unchanged, numbered so a
requirement can cite one line. In [`F-0001`](../facts/F-0001-setup-answers.md),
fact 8 answers the question about carriage returns and empty files, and ends:

    An empty file has zero lines, zero words and zero characters.

This is `F-0001#8` — the eighth fact in facts document `F-0001`. It is immutable
evidence: it is not rewritten, only cited.

## 2. The requirement (Layer 2 — derived, and it cites the fact)

A [PRD](../prd/) requirement is written *from* that fact and cites it. In the
PRD's `§6` functional-requirements table:

    | REQ | Statement | MoSCoW | Phase | Facts |
    | --- | --------- | ------ | ----- | ----- |
    | REQ-001 | An empty input counts zero lines, zero words and zero characters. | Must | 1 | F-0001#8 |

`REQ-001` is a stable ID, assigned once. Because it cites `F-0001#8`, a reader can
always trace the requirement back to the idea owner's exact words —
[`prd-lint.sh`](../prd/prd-lint.sh) checks that the cited fact resolves.

## 3. The guardrail (the known pitfall it inherits)

Counting one line for "the last line" even when there is nothing to count is a
known trap, so it is in [`guardrails.md`](../guardrails.md) §2:

    - ❌ The last line and the empty file.

The requirement now carries that guardrail: the test must prove the empty input
gives **zero** lines, not one, and the same code must still count the last line of
`"a"` as one.

## 4. The test (strict TDD — red, then green)

The requirement is proved at the [unit level](test-levels.md) — package `count`,
in isolation, on an empty in-memory reader — following the
[`template-unit.md`](template-unit.md) pattern and the strict
[red-then-green](../engineering-discipline.md#requirements-traceability) order:

1. **Red.** Write the test first: count an empty reader, assert zero lines, zero
   words, zero characters, and valid UTF-8. Run `‹unit test command›`; watch it
   **fail** against code that adds one line for an unterminated last line without
   checking that any character was read — failing for the *right* reason.
2. **Green.** Count the last line only when at least one character follows the
   last line feed, run `‹unit test command›` again, watch it **pass** — together
   with the tests for `"a"` and `"a\n"`.
3. **Freeze.** Once a fresh context confirms it
   ([R9](../issue-workflow.md#r9--test-freeze-after-confirmation)), the test is
   frozen: not weakened later to make new code pass.

The test lives under `‹test directory›` and is tagged `unit` so it can run alone.

## 5. The traceability row (the line, written down)

Finally the test earns a row in the
[traceability table](traceability-template.md), closing the loop from fact to
test:

    | Test ID | Level | Covers | Fact | Guardrail | ADR | Task | Status |
    |---------|-------|--------|------|-----------|-----|------|--------|
    | empty-input-counts-zero | unit | REQ-001 | F-0001#8 | guardrails.md §2 | — | T-ab12 | frozen |

Read left to right, the row is the whole story: the idea owner said it
(`F-0001#8`), it became a requirement (`REQ-001`), it inherited a pitfall
(`guardrails.md §2`), a task delivered it (`T-ab12`), and a frozen unit test
proves it. A requirement with no such row is a requirement with no proof — which
the [DoD checklist](dod-checklist.md) treats as not done.
