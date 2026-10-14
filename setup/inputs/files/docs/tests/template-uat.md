# User acceptance test (UAT) template

A generic pattern for writing a user-acceptance-test (UAT) scenario — the human
layer that sits on top of the [end-to-end](template-e2e.md) path, described in
[`test-levels.md`](test-levels.md#3-end-to-end-e2e-tests). A UAT scenario checks
that the delivered behaviour is what a real person actually asked for; a human
reads the plain-language steps and signs off, rather than a command asserting a
result.

> **Using this file in tally.** Copy the [skeleton](#fill-in-skeleton) for each
> user-facing scenario that needs a human sign-off, usually the same scenario an
> [E2E test](template-e2e.md) already automates. Replace every `‹…›`
> placeholder. The person who signs off a `tally` scenario is the idea owner, who
> gave the answers recorded in [`F-0001`](../facts/F-0001-setup-answers.md), or a
> person that the idea owner names on the issue.

## In plain terms

> A user-acceptance scenario is a short story — given this starting point, when a
> person does this, then this should happen — that a real person reads, tries, and
> either accepts or rejects. It proves the system does what was asked, not just
> what the code was told to do.

## The pattern

- **Write it in plain Given/When/Then.** Given the starting state, when the
  user does something, then a visible outcome follows. No code, no
  internal jargon — a stakeholder who has never read the system's code should
  be able to follow every step.
- **It rides the same path an E2E test automates.** A UAT scenario is not a new
  path to invent; it is the same user-facing journey, described in the words a
  person uses instead of the steps a machine runs. See
  [`template-e2e.md`](template-e2e.md).
- **A person signs it off.** The scenario is judged by a human reading the
  outcome, not by an automated assertion. Name who is qualified to sign off —
  usually whoever asked for the behaviour, or a stand-in with the authority to
  accept on their behalf.
- **The acceptance is recorded, not just observed.** Trying the scenario and
  liking the result is not enough — write down who accepted it and when, so the
  sign-off is a fact in the repo, not a memory.

## Fill-in skeleton

```text
Scenario: ‹title›

Given ‹starting state›
When ‹the user does …›
Then ‹the visible outcome›

Accepted by ‹person› on ‹date›
Covers ‹REQ-…›
```

## Example in tally

```text
Scenario: count a file that I pipe into tally

Given a text of three lines, each ending with a line feed
When I pipe it into tally with no file name and no flag
Then tally prints one row with the counts of lines, words and characters,
     with no name after the counts, and ends with the exit code 0
```

The row for standard input has no name, and `-` is a file named `-`, not
standard input (answer `Q-003`). The acceptance line and the `Covers` line are
written when the scenario is run and the requirement has its ID.

## Checklist

- [ ] Every user-facing scenario that needs sign-off has a UAT written for it.
- [ ] The steps are plain words a stakeholder understands, with no code or
      internal jargon.
- [ ] Acceptance is recorded — a named person and a date — not just observed.
- [ ] Names the requirement it accepts via a
      [traceability](traceability-template.md) row.

## See also

A UAT scenario is the project's human acceptance step, not another rung of the
automated ladder in [`test-levels.md`](test-levels.md) — nothing here runs in
the hook or in CI. It confirms, by a person's own judgement, that the path an
[E2E test](template-e2e.md) automates does what was actually asked for.
