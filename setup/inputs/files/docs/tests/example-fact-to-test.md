# Worked example — from a fact to a test

One concrete walk down the traceability line of `tally`, so the abstract chain in
[`traceability-template.md`](traceability-template.md) has a filled example beside
it. It follows a single requirement from the idea owner's words all the way to the
test that proves it.

    fact (F-0001#8) → requirement (REQ-001) → guardrail → ADR → task (T-ab12) → test

This example needs no architecturally-significant decision, so the **ADR** link is
empty (`—`, as the [row at the end](#5-the-traceability-row-the-line-written-down)
shows); every other link is filled.

> **What is real here and what is not.** The fact is real: it is the eighth fact
> of [`F-0001`](../facts/F-0001-setup-answers.md). The requirement ID `REQ-001`
> and the task ID `T-ab12` are illustrative, because the PRD of `tally` and its
> first task do not exist yet; replace them with the real IDs when they do. The
> test's command stays the `‹unit test
> command›` placeholder until [`test-levels.md`](test-levels.md) fills it.

## In plain terms

> Nothing in this project gets tested by accident. Every test traces back through
> a written requirement to something the idea owner actually said. This page shows
> that trace once, in full, so the pattern is concrete: a sentence of the idea
> owner becomes a requirement, the requirement inherits a known pitfall, and a
> test proves the pitfall is handled.

## 1. The fact (Layer 1 — the idea owner's words)

A [facts record](../facts/) stores the words unchanged, numbered so a requirement
can cite one line. In [`F-0001`](../facts/F-0001-setup-answers.md), fact 8 is the
answer to the question `Q-007`:

    8. `Q-007` A carriage return before a line feed is part of the line and is
       counted as a character, like any other code point. An empty file has
       zero lines, zero words and zero characters.

This is `F-0001#8` — the eighth fact in facts document `F-0001`. It is immutable
evidence: it is not rewritten, only cited.

## 2. The requirement (Layer 2 — derived, and it cites the fact)

A [PRD](../prd/) requirement is written *from* that fact and cites it. In the
PRD's `§6` functional-requirements table:

    | REQ | Statement | MoSCoW | Phase | Facts |
    | --- | --------- | ------ | ----- | ----- |
    | REQ-001 | For an empty file, tally gives zero lines, zero words and zero characters. | Must | 1 | F-0001#8 |

`REQ-001` is a stable ID, assigned once. Because it cites `F-0001#8`, a reader can
always trace the requirement back to the exact words —
[`prd-lint.sh`](../prd/prd-lint.sh) checks that the cited fact resolves.

## 3. The guardrail (the known pitfall it inherits)

The rule "the last line counts when the file does not end with a line feed" makes
a counter easy to get wrong at the edge: a counter that adds one line at the end
of every input gives an empty file one line. That trap is a known pitfall in
[`guardrails.md`](../guardrails.md) §2:

    - ❌ An empty input, or an input that ends with a line feed, gets one line too many.

The requirement now carries that guardrail: the test must prove the empty file
gives *zero* lines, not merely that a normal file gives the right count.

## 4. The test (strict TDD — red, then green)

The requirement is proved at the [unit level](test-levels.md) — the package
`count`, in isolation, on a reader in memory — following the
[`template-unit.md`](template-unit.md) pattern and the strict
[red-then-green](../engineering-discipline.md#requirements-traceability) order:

1. **Red.** Write the test first: count an empty reader, assert zero lines, zero
   words and zero characters. Run `‹unit test command›`; watch it **fail**
   against a counter that still adds a last line to every input — failing for the
   *right* reason, an assertion on the line count, not a compile error.
2. **Green.** Count the last line only when at least one character follows the
   last line feed, run `‹unit test command›` again, watch it **pass**.
3. **Freeze.** Once a fresh context confirms it
   ([R9](../issue-workflow.md#r9--test-freeze-after-confirmation)), the test is
   frozen: not weakened later to make new code pass.

In Go the test is a `_test.go` file beside the package `count`, not a file under
`‹test directory›`, and it is a unit test because it reads no file, so it runs
alone with the other tests of that package.

## 5. The traceability row (the line, written down)

Finally the test earns a row in the
[traceability table](traceability-template.md), closing the loop from fact to
test:

    | Test ID | Level | Covers | Fact | Guardrail | ADR | Task | Status |
    |---------|-------|--------|------|-----------|-----|------|--------|
    | count/TestEmptyInput | unit | REQ-001 | F-0001#8 | guardrails.md §2 | — | T-ab12 | frozen |

Read left to right, the row is the whole story: the idea owner said it
(`F-0001#8`), it became a requirement (`REQ-001`), it inherited a pitfall
(`guardrails.md §2`), a task delivered it (`T-ab12`), and a frozen unit test
proves it. A requirement with no such row is a requirement with no proof — which
the [DoD checklist](dod-checklist.md) treats as not done.
