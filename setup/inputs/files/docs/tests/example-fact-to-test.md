# Worked example — from a fact to a test

One concrete walk down the baseline's traceability line, so the abstract chain in
[`traceability-template.md`](traceability-template.md) has a filled example beside
it. It follows a single requirement from the customer's words all the way to the
test that proves it, using only the baseline's conventions.

    fact (F-0001#5) → requirement (REQ-001) → guardrail → ADR → task (T-ab12) → test

This example needs no architecturally-significant decision, so the **ADR** link is
empty (`—`, as the [row at the end](#5-the-traceability-row-the-line-written-down)
shows); every other link is filled.

> **About this file.** This is an **illustrative** example — it walks one real
> line of tally, the total row of F-0001#5, from fact to test. The requirement ID
> and the task ID are samples. It names no test runner beyond the Go toolchain: the
> test's command stays the `‹unit test
> command›` marker throughout. Remove this
> note when the example is kept as the project's own.

## In plain terms

> Nothing in this project gets tested by accident. Every test traces back through
> a written requirement to something the customer actually said. This page shows
> that trace once, in full, so the pattern is concrete: a customer sentence
> becomes a requirement, the requirement inherits a known pitfall, and a test
> proves the pitfall is handled.

## 1. The fact (Layer 1 — the customer's words)

A [facts record](../facts/) stores the customer's words unchanged, numbered so a
requirement can cite one line. In [`facts/F-0001-...`](../facts/):

    5. `Q-004` The total row is the sum of the counts of the files that were counted
       without an error, followed by the word `total`, in the layout of the other
       rows. A file that cannot be read, or is not valid UTF-8, adds nothing to the
       total.

This is `F-0001#5` — the first fact in facts document `F-0001`. It is immutable
evidence: it is not rewritten, only cited.

## 2. The requirement (Layer 2 — derived, and it cites the fact)

A [PRD](../prd/) requirement is written *from* that fact and cites it. In the
PRD's `§6` functional-requirements table:

    | REQ | Statement | MoSCoW | Phase | Facts |
    | --- | --------- | ------ | ----- | ----- |
    | REQ-001 | The total row sums only the files counted without an error; a file that cannot be read or is not valid UTF-8 adds nothing to it. | Must | 1 | F-0001#5 |

`REQ-001` is a stable ID, assigned once. Because it cites `F-0001#5`, a reader can
always trace the requirement back to the customer's exact words —
[`prd-lint.sh`](../prd/prd-lint.sh) checks that the cited fact resolves.

## 3. The guardrail (the known pitfall it inherits)

A failed input leaking into an aggregate is a classic trap, so it belongs in
[`guardrails.md`](../guardrails.md) as a known pitfall:

    - ❌ An input that failed is still counted in the aggregate instead of left out.

The requirement now carries that guardrail: the test must prove the failed file
adds *nothing* to the total, not merely that good files are summed.

## 4. The test (strict TDD — red, then green)

The requirement is proved at the [unit level](test-levels.md) — one component, in
isolation — following the [`template-unit.md`](template-unit.md) pattern and the
strict [red-then-green](../engineering-discipline.md#requirements-traceability)
order:

1. **Red.** Write the test first: give the total one file counted without error and one
   that is not valid UTF-8, and assert the total equals the counts of the first
   file alone. Run `‹unit test command›`; watch it **fail** against code that still
   adds the invalid file to the total — failing for the *right* reason.
2. **Green.** Leave the failed file out of the total, run `‹unit test command›` again, watch it
   **pass**.
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
    | total-skips-failed-file | unit | REQ-001 | F-0001#5 | guardrails.md §2 | — | T-ab12 | frozen |

Read left to right, the row is the whole story: the customer said it (`F-0001#5`),
it became a requirement (`REQ-001`), it inherited a pitfall (`guardrails.md §2`), a
task delivered it (`T-ab12`), and a frozen unit test proves it. A requirement with
no such row is a requirement with no proof — which the
[DoD checklist](dod-checklist.md) treats as not done.
