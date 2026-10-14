# AGENTS.md

Agent context for **tally**, a small Go command that counts the lines, words and
characters of text files. Read this before you change anything in this
repository.

This file is a startup index and summary of how we work here. Where it disagrees
with a document it points to or summarises, **the document wins** — and the
disagreement is a defect to fix in the same change
([R10](docs/issue-workflow.md#r10--sync-with-governance)).

## What this repository is

The product `tally`: the command `tally [-l] [-w] [-c] [file...]`, written in Go
with the standard library only and built with the Go toolchain. It reads the
named files, or standard input when there is no name, and prints one row per file
— the selected counts in the fixed order lines, words, characters, then the name —
and a `total` row with two files or more. The rules of what counts as a line, a
word and a character, the errors and the exit codes are in the
[problem statement](docs/facts/problem-statement-brief.md) and the answers of the
idea owner in [`F-0001`](docs/facts/F-0001-setup-answers.md); the
[onboarding](docs/onboarding-for-engineers.md) teaches them.

No product code exists yet. The planned layout is one Go module with a package
`count`, which counts the lines, words and characters of a reader, and the
package `main` of the command. Unit and integration tests sit beside the Go code
in `_test.go` files; the root [`tests/`](tests/) directory holds the end-to-end
tests of the built binary and is empty until the first one lands.

The way of working came from the Armature engineering-discipline kit, pinned in
[ADR-0009](docs/adr/0009-pin-the-baseline.md). Two things catch agents out: the
markers between single angle quotes are deliberate — values no task has set yet —
and the quality targets are frozen numbers, not goals to tune.

## How these instructions rank

A platform or operator instruction of higher priority stays higher priority than
this file. Within its scope, this file governs work in this repository. The
documents under [`docs/`](docs/) are authoritative for their own subject, and this
file only summarises them. A nested instruction file, added lower in the tree, may
add a local constraint and may **never** weaken the quality gate. A conflict
between any two of them stops work until a decision note or an
[ADR](docs/adr/) resolves it.

## Start here

Read [`docs/engineering-discipline.md`](docs/engineering-discipline.md) for how we
work, then [`docs/issue-workflow.md`](docs/issue-workflow.md) for the ticket rules
that gate assumes, then [`docs/guardrails.md`](docs/guardrails.md) for the
pitfalls of counting text.
[`docs/onboarding-for-engineers.md`](docs/onboarding-for-engineers.md) is the
human first door. Which document owns which class of rule is the table under
Sources of truth, written once.

## The quality gate

Every substantive task passes **eight** ordered steps, in this order. Before step
1, an issue is open, and the work is sliced into an ordered, Definition-of-Done
covering, test-first plan that is reviewed once and recorded on the issue. That
plan review is architecture and scope, never implementation approval.

1. **Isolate.** Work in a per-task git worktree branched off `origin/main`, never in the operator's own checkout.
2. **Honor the guardrails.** Before you write code, read the acceptance criteria, [`docs/guardrails.md`](docs/guardrails.md), and the [ADRs](docs/adr/) the ticket references.
3. **Test first.** Write the failing test, watch it fail for the right reason, then write the code.
4. **Make long tasks visible.** Anything that can run over ten seconds shows which step runs and that it lives.
5. **Review until findings decay.** Freeze the head, then run independent blind rounds on it, a different lens each round. A fix re-freezes; at most two fix-and-review cycles follow the first freeze, and the last round ends `nothing material in scope` or `not mergeable, findings recorded` — see [Reviewing until findings decay](docs/engineering-discipline.md#reviewing-until-findings-decay). A defect the change *revealed*, off the path its Definition of Done names, opens an issue instead of entering the branch. A reviewer is a person or a fresh agent session — the requirement is [independence](docs/engineering-discipline.md#who-may-review), not reviewer type — and summarised text gets a clause-by-clause semantic pass.
6. **Be honest, keep evidence.** Report a failure as a failure, and review the producing code before a costly action.
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

## What tally must not get wrong

The [guardrails](docs/guardrails.md) are authoritative; in short:

- A character is a code point, not a byte; a word ends at any `White_Space`
  character; the last line counts without a final line feed, and an empty file has
  zero lines; a carriage return is a character, not a line end.
- Read each file in fixed buffers of 64 KiB, never a whole file, and carry a UTF-8
  sequence and the detection of invalid UTF-8 across the boundary of two buffers.
- A file that cannot be read, or is not valid UTF-8, gets a message on standard
  error, no row and no share of the total; the others are still counted; the exit
  code is then 1. The name `-` is a file, not standard input.
- The quality targets — at most 2 seconds for a file of 10 MiB, at most 64 MiB of
  peak heap for a file of 1 GiB, at least 90 percent statement coverage of the
  package `count` — and their methods are pre-registered; never move them.
- Go standard library only: never add a third-party module.

## Checks you can run

These read only text, so they need no toolchain. Install the hooks once per clone
by running `sh .githooks/install.sh` (it pins `core.hooksPath` to the relative
`.githooks`); the first four then run before every
commit, and in CI; `nested-checkout-check.sh` needs `git`, so it runs in CI only;
and `git diff --check` you run yourself. Keep
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

Once Go code exists, the product checks are the commands of the Go toolchain —
`gofmt -l .` (must print nothing), `go vet ./...`, `go build ./...` and
`go test ./...`, with `go test -cover` for the package `count`. The task that adds
the first Go code wires them into the hook and CI and sets the per-level commands
in [`docs/tests/test-levels.md`](docs/tests/test-levels.md); until then, do not
claim a Go check ran, and never invent a command beyond the Go toolchain.

## Branches, worktrees, commits, and pull requests

Work in a per-task git worktree under the worktree directory, in a subdirectory
named for the task, branched off `origin/main`, never in the operator's own
checkout. Commit at each logical step, with a subject that follows Conventional
Commits — `<type>: <ID> <description>` when it carries a task. Rebase onto the
latest `origin/main` and land with a plain merge; **never squash** — but a branch
already under a frozen-head verdict merges `origin/main` in instead, so the
reviewed SHA survives. The pull-request body links its issue with `Closes #N`, or
`Refs #N` when it does not close it.

## The task index

[`docs/tasks/backlog.md`](docs/tasks/backlog.md) holds one line per open task, and
[`docs/tasks/completed.md`](docs/tasks/completed.md) the dated log of finished
ones. Any detail belongs in that task's own file, never in either index. The same
pull request that lands the work records the line in the completed log, moving it
from the backlog where the task had one.

## Values not set yet

A marker between single angle quotes is a value no task has set yet — the test
commands, the evidence store, the task-ID scheme, the worktree directory. The
stack is fixed, but each value is set by the task that first needs it, with its
reason on the issue. Never replace a marker with a guess, never invent a command,
path or number, and never delete a marker to make a check pass. Search for the
opening angle quote to find every one of them.

## Safety limits

Never commit secrets, and never expose sensitive data. Never rewrite published
history — no force-push, no amend of a landed commit. Never run a destructive,
costly or irreversible operation without explicit authorization, and review the
code that will do the work before it runs. The tests of a file of 1 GiB write
that file to a temporary directory: never commit it. The full rule is
[Safety limits](docs/engineering-discipline.md#safety-limits).

## Decisions and questions

Record the plan, the selected option, the rejected alternatives, the tradeoffs and
the evidence on the issue — that is where a fresh context picks the work up. Ask
another operator — the idea owner included — on the issue thread, with a severity
and an expected response time, never directly. An architecturally significant
decision becomes an [ADR](docs/adr/).

## Sources of truth

| Document | Authoritative for |
|----------|-------------------|
| [`docs/facts/`](docs/facts/) and [`docs/prd/`](docs/prd/) | What `tally` must do: the facts of the idea owner as evidence, and the requirements derived from them. |
| [`docs/engineering-discipline.md`](docs/engineering-discipline.md) | The quality gate, solution selection, testing, reviews, commits and the safety limits. |
| [`docs/issue-workflow.md`](docs/issue-workflow.md) | The numbered rules themselves, and the honest table of what a mechanism backs today. |
| [`docs/guardrails.md`](docs/guardrails.md) | Known pitfalls, the pre-registered quality targets, and how a result is validated. |
| [`docs/glossary.md`](docs/glossary.md) | The shared vocabulary, and the rule that every abbreviation earns an entry. |
| [`docs/adr/`](docs/adr/) | Architecture decisions that constitute the project, with the context and the consequences of each one. |
| [`docs/tests/`](docs/tests/) | The test levels, a pattern for each, and the Definition-of-Done coverage checklist. |
| [`.githooks/`](.githooks/) and [`docs/ci/`](docs/ci/) | What the gate enforces locally, and what CI enforces as the authority. |
