# AGENTS.md

Agent context for **tally**, a small Go command that counts the lines, words and
characters of text files. Read this before you change anything in this
repository.

This file is a startup index and summary of how we work here. Where it disagrees
with a document it points to or summarises, **the document wins** — and the
disagreement is a defect to fix in the same change
([R10](docs/issue-workflow.md#r10--sync-with-governance)).

## What this repository is

The repository of the product `tally`: the command
`tally [-l] [-w] [-c] [file...]`, written in Go with the standard library only
and built with the Go toolchain. Its engineering discipline — the quality gate,
the guardrails, the decision records, the glossary, the facts-and-requirements
convention, the test section and the task backlog — comes from the Armature kit,
pinned by [ADR-0009](docs/adr/0009-pin-the-baseline.md).

Where the project stands: the facts are collected
([`docs/facts/`](docs/facts/)), and no product code, no Go module and no product
test exists yet. The first tasks write the PRD, then the package `count` and the
command, test first.

Two things catch agents out. The angle-quote markers (U+2039 … U+203A) in the
kit documents are deliberate: each one is a value that a task has not yet
selected. And the root [`tests/`](tests/) directory is empty on purpose until
the first product test lands.

## The product in one screen

- `tally` reads the files it is given, or standard input when it is given none,
  and prints one row per file: the selected counts, then the file name, each
  separated by one space. With two files or more, a last row `total` follows.
- `-l`, `-w` and `-c` select lines, words and characters. Flags combine; the
  counts always print in the order lines, words, characters
  ([`F-0001#2`, `F-0001#3`](docs/facts/F-0001-setup-answers.md)).
- A character is one Unicode code point of UTF-8 text, and white space is the
  Unicode property `White_Space`. Note that `-c` counts characters, not bytes.
- A file that cannot be read, or is not valid UTF-8, gets a message on standard
  error, no row and no share of the total, and the exit code is 1 at the end;
  the other files are still counted.
- The counting lives in a package `count`, separate from the package `main` of
  the command ([`F-0001#12`](docs/facts/F-0001-setup-answers.md)). It reads in
  fixed buffers of 64 KiB and never holds a whole file in memory
  ([`F-0001#11`](docs/facts/F-0001-setup-answers.md)).
- The quality targets — 2 seconds for 10 MiB, a peak heap of 64 MiB for 1 GiB,
  90 percent statement coverage of `count` — and the traps behind them are in
  [`docs/guardrails.md`](docs/guardrails.md). Read it before you write code.

## How these instructions rank

A platform or operator instruction of higher priority stays higher priority than
this file. Within its scope, this file governs work in this repository. The
documents under [`docs/`](docs/) are authoritative for their own subject, and this
file only summarises them. A nested instruction file, added lower in the tree, may
add a local constraint and may **never** weaken the quality gate. A conflict
between any two of them stops work until a decision note or an
[ADR](docs/adr/) resolves it.

## Start here

Read [`docs/onboarding-for-engineers.md`](docs/onboarding-for-engineers.md) for
the product and its vocabulary, then
[`docs/engineering-discipline.md`](docs/engineering-discipline.md) for how we
work, then [`docs/issue-workflow.md`](docs/issue-workflow.md) for the ticket
rules that gate assumes. Which document owns which class of rule is the table
under Sources of truth, written once.

## The quality gate

Every substantive task passes **eight** ordered steps, in this order. Before step
1, an issue is open, and the work is sliced into an ordered, Definition-of-Done
covering, test-first plan that is reviewed once and recorded on the issue. That
plan review is architecture and scope, never implementation approval.

1. **Isolate.** Work in a per-task git worktree branched off `origin/main`, never in the operator's own checkout.
2. **Honor the guardrails.** Before you write code, read the acceptance criteria, [`docs/guardrails.md`](docs/guardrails.md), and the [ADRs](docs/adr/) the ticket references.
3. **Test first.** Write the failing test, watch it fail for the right reason, then write the code.
4. **Make long tasks visible.** Anything that can run over ten seconds shows which step runs and that it lives. The 1 GiB memory test is one such operation.
5. **Review until findings decay.** Freeze the head, then run independent blind rounds on it, a different lens each round. A fix re-freezes; at most two fix-and-review cycles follow the first freeze, and the last round ends `nothing material in scope` or `not mergeable, findings recorded` — see [Reviewing until findings decay](docs/engineering-discipline.md#reviewing-until-findings-decay). A defect the change *revealed*, off the path its Definition of Done names, opens an issue instead of entering the branch. A reviewer is a person or a fresh agent session — the requirement is [independence](docs/engineering-discipline.md#who-may-review), not reviewer type — and summarised text gets a clause-by-clause semantic pass.
6. **Be honest, keep evidence.** Report a failure as a failure — a missed timing or memory target included — and review the producing code before a costly action.
7. **Keep the documentation current.** Every document the change leaves stale is fixed in the same pull request — and any lesson the task taught that the next reader could hit is written back into `guardrails.md` §2 (Known pitfalls).
8. **Close out in the same PR.** Tick the boxes, write the verdict, record the task line in the completed log, and — for a task started under ADR-0007 — write the task's resource record (model, effort, tokens and elapsed time per gate part, recorded not budgeted).

**Model tiers.** Where a step needs a model, a **reasoning tier** does the deciding
and judging — the plan and its review, solution selection, the review rounds, the
verdict — and an **execution tier** does tactical execution and coding once the plan
is fixed. Routing applies only after
[Determinism](docs/engineering-discipline.md#solution-selection) warrants a model at
all, a deterministic check still outranks any tier, and reviewer
[independence](docs/engineering-discipline.md#who-may-review) wins where it meets
routing. The rule and its tier-to-step map live in
[Model tiers](docs/engineering-discipline.md#model-tiers), recorded in
[ADR-0005](docs/adr/0005-route-work-by-model-tier.md).

## The issue rules

The workflow defines **thirteen** numbered rules, R1–R13; cite one by number in a
review or a commit. They are defined once, in
[`docs/issue-workflow.md`](docs/issue-workflow.md) — this file points there rather
than restating them, so there is a single source to keep in step. Which rule a
mechanism backs today, and which is `(written rule)` with none, is that document's
own [enforcement table](docs/issue-workflow.md#what-is-enforced-where), never this
file.

## Checks you can run

The discipline checks read only text, so they need no toolchain. Install the
hooks once per clone by running `sh .githooks/install.sh` (it pins
`core.hooksPath` to the relative `.githooks`); the first four then run before
every commit, and in CI; `nested-checkout-check.sh` needs `git`, so it runs in CI
only; and `git diff --check` you run yourself. Keep
[`.gitattributes`](.gitattributes): it holds these scripts at line-feed endings,
without which none of them runs on a Windows checkout.

```
sh docs/adr/adr-lint.sh
sh docs/prd/prd-lint.sh
sh docs/links/link-lint.sh
sh docs/tests/run-discipline-tests.sh
sh docs/tests/nested-checkout-check.sh
git diff --check
```

[`docs/ci/pr-link-lint.sh`](docs/ci/pr-link-lint.sh) and
[`docs/ci/review-record-lint.sh`](docs/ci/review-record-lint.sh) read forge
artifacts, so they run in CI only and have no local run.

The product checks run through the Go toolchain: the problem statement measures
coverage with `go test -cover`. No Go module exists yet, so no product command
runs today. The exact unit, integration, end-to-end, lint and security commands
are selected by a task under
[solution selection](docs/engineering-discipline.md#solution-selection) and
written into [`docs/tests/test-levels.md`](docs/tests/test-levels.md); until
then, never invent one, and never add a dependency outside the Go standard
library to the product.

## Branches, worktrees, commits, and pull requests

Work in a per-task git worktree under the project's worktree directory, branched
off `origin/main`, never in the operator's own checkout. Commit at each logical
step, with a subject that follows Conventional Commits — `<type>: <ID>
<description>` when it carries a task. Rebase onto the latest `origin/main` and
land with a plain merge; **never squash** — but a branch already under a
frozen-head verdict merges `origin/main` in instead, so the reviewed SHA
survives. The pull-request body links its issue with `Closes #N`, or `Refs #N`
when it does not close it.

## The task index

[`docs/tasks/backlog.md`](docs/tasks/backlog.md) holds one line per open task, and
[`docs/tasks/completed.md`](docs/tasks/completed.md) the dated log of finished
ones. Any detail belongs in that task's own file, never in either index. The same
pull request that lands the work records the line in the completed log, moving it
from the backlog where the task had one.

## Markers and selected values

Every angle-quote marker in the kit documents is a value this project has not yet
selected — the test commands, the evidence store, the task-ID scheme, the
worktree directory. Never replace one with a guess, never invent a command, path
or number, and never delete a marker to make a check pass. A marker is replaced
only by the task that selects its value and records the selection. Search for the
opening angle quote, U+2039, to find every one of them.

## Safety limits

Never commit secrets, and never expose sensitive data. Never rewrite published
history — no force-push, no amend of a landed commit. Never run a destructive,
costly or irreversible operation without explicit authorization, and review the
code that will do the work before it runs. A test that generates a file of 1 GiB
writes it to a temporary directory and removes it. The full rule is
[Safety limits](docs/engineering-discipline.md#safety-limits).

## Decisions and questions

Record the plan, the selected option, the rejected alternatives, the tradeoffs and
the evidence on the issue — that is where a fresh context picks the work up. Ask
another operator on the issue thread, with a severity and an expected response
time, never directly. A question about the product goes to the idea owner, and
the answer becomes a new facts record. An architecturally significant decision
becomes an [ADR](docs/adr/).

## Sources of truth

| Document | Authoritative for |
|----------|-------------------|
| [`docs/engineering-discipline.md`](docs/engineering-discipline.md) | The quality gate, solution selection, testing, reviews, commits and the safety limits. |
| [`docs/issue-workflow.md`](docs/issue-workflow.md) | The numbered rules themselves, and the honest table of what a mechanism backs today. |
| [`docs/guardrails.md`](docs/guardrails.md) | The known pitfalls of counting text, the pre-registered quality targets, and how a result is validated. |
| [`docs/glossary.md`](docs/glossary.md) | The shared vocabulary, and the rule that every abbreviation earns an entry. |
| [`docs/adr/`](docs/adr/) | Architecture decisions that constitute this project, with the context and the consequences of each one. |
| [`docs/tests/`](docs/tests/) | The test levels, a pattern for each, and the Definition-of-Done coverage checklist. |
| [`docs/facts/`](docs/facts/) and [`docs/prd/`](docs/prd/) | The idea owner's facts kept as evidence, and the requirements derived from them. |
| [`.githooks/`](.githooks/) and [`docs/ci/`](docs/ci/) | What the gate enforces locally, and what CI enforces as the authority. |
