# tally

*A small Go command that counts the lines, words and characters of text files.*

> A person who works in a terminal wants to know how long a text file is. `tally`
> reads the files that the person names and prints the number of lines, words and
> characters of each, and a total.

**Status.** The requirements are being collected and written down. The facts from
the idea owner are recorded in [`docs/facts/`](docs/facts/); no product code and
no product test exist yet. The first tasks write the Product Requirements Document
(PRD), then the counting package and the command, test first.

## What tally does

```
tally [-l] [-w] [-c] [file...]
```

| Input | Behaviour |
|-------|-----------|
| No flag | Prints the three counts: lines, words, characters. |
| `-l`, `-w`, `-c` | Prints only the selected counts. Flags combine, and the counts always print in the order lines, words, characters. |
| No file name | Reads standard input. Its row has no name. |
| `-` as a name | A file named `-`, not standard input. |
| Two files or more | A last row `total`, the sum of the files counted without an error. |
| A file that cannot be read, or a directory | A message on standard error with the name and the reason; no row; exit code 1. The other files are still counted. |
| A file that is not valid UTF-8 | A message on standard error with the name; no row; exit code 1. |
| No error | Exit code 0. |

Each row is the selected counts, then the name, each separated from the next by
one space. The definitions are exact:

- A **line** is a run of characters that ends with a line feed. The last line
  counts when the file does not end with a line feed. An empty file has zero
  lines.
- A **word** is a run of characters that are not white space, where white space
  is a character of the Unicode property `White_Space`.
- A **character** is one Unicode code point of UTF-8 text. A carriage return is a
  character like any other. Note that `-c` counts characters, not bytes, which is
  not what the same flag means in `wc`.

Other counts (bytes, paragraphs, sentences), archives, a graphical interface, a
network service and plug-ins are out of scope.

## Quality targets

| Target | Bar |
|--------|-----|
| Speed | A file of 10 MiB of ASCII text is counted in at most 2 seconds, start to exit, on a computer with 4 cores and a solid-state disk, with the file in the cache of the operating system. |
| Memory | Counting a file of 1 GiB keeps the peak heap at or under 64 MiB, sampled every 10 milliseconds during the run. |
| Test coverage | At least 90 percent of the statements of the counting package, by `go test -cover`. |

The targets, how each is measured, and the traps that make them easy to get
wrong are in [`docs/guardrails.md`](docs/guardrails.md). The facts behind them
are [`F-0001`](docs/facts/F-0001-setup-answers.md) and the
[problem statement](docs/facts/problem-statement-brief.md).

## Technology

Go, with the standard library only, built with the Go toolchain. The counting
lives in a package `count`, separate from the package `main` of the command, and
reads each file in fixed buffers of 64 KiB, so it never holds a whole file in
memory.

## Start here

1. **[`docs/onboarding-for-engineers.md`](docs/onboarding-for-engineers.md)** — read
   this first. It states the
   [Problem statement](docs/onboarding-for-engineers.md#1-problem-statement) and
   teaches the vocabulary: line, word, character, code point, `White_Space`.
2. **[`docs/engineering-discipline.md`](docs/engineering-discipline.md)** — how we
   work: the quality gate every substantive task passes, plus branches, tests,
   reviews, and ADRs. Read before your first commit.
3. **[`AGENTS.md`](AGENTS.md)** — if you are a coding agent, start here instead. It
   summarises [`docs/engineering-discipline.md`](docs/engineering-discipline.md)
   and [`docs/issue-workflow.md`](docs/issue-workflow.md), and indexes the rest.
   The long documents stay authoritative.

## What's inside

| Piece | What it holds |
|-------|---------------|
| [`AGENTS.md`](AGENTS.md) | The agent entry point: the product in one screen, the quality gate, the checks, a pointer to the R1–R13 rules, and which document is authoritative for each. |
| [`CLAUDE.md`](CLAUDE.md) | One line, `@AGENTS.md`, so Claude Code loads the same guide. No second copy to drift. |
| [`docs/agents/`](docs/agents/) | What the entry points are and what they may not become ([ADR-0004](docs/adr/0004-ship-agent-entry-points.md)). |
| [`docs/onboarding-for-engineers.md`](docs/onboarding-for-engineers.md) | The first door: the problem statement and a crash course in counting text. |
| [`docs/engineering-discipline.md`](docs/engineering-discipline.md) | The quality gate, the solution-selection standard, and every working practice. |
| [`docs/issue-workflow.md`](docs/issue-workflow.md) | The issue-first workflow (R1–R13): the ticket policy the gate assumes. |
| [`docs/glossary.md`](docs/glossary.md) | The shared vocabulary: the process terms and the terms of counting text. |
| [`docs/guardrails.md`](docs/guardrails.md) | The known pitfalls of counting text, the pre-registered quality targets, and validation. |
| [`docs/adr/`](docs/adr/) | Architecture Decision Records, plus [`adr-lint.sh`](docs/adr/adr-lint.sh), the discipline test that keeps them honest. [ADR-0009](docs/adr/0009-pin-the-baseline.md) pins the kit this repository started from. |
| [`docs/facts/`](docs/facts/) | The idea owner's facts, kept word for word as immutable evidence. |
| [`docs/prd/`](docs/prd/) | Product Requirements Documents derived from the facts, plus [`prd-lint.sh`](docs/prd/prd-lint.sh). |
| [`docs/tests/`](docs/tests/) | The testing conventions — the test levels, a pattern per level, the security, scaling and Definition-of-Done checklists, and test-to-requirement traceability — plus [`run-discipline-tests.sh`](docs/tests/run-discipline-tests.sh), which tests the discipline linters against fixtures. |
| [`tests/`](tests/) | Where the product tests of the built command go. Empty until the first one lands. |
| [`docs/tasks/`](docs/tasks/) | The task index — [`backlog.md`](docs/tasks/backlog.md) and [`completed.md`](docs/tasks/completed.md). |
| [`docs/setup/`](docs/setup/) | The pin of the baseline and the checksums of the facts, written when this repository was set up. |
| [`.githooks/`](.githooks/) | Git hooks that enforce the cheap gate locally — a commit-message check and a pre-commit runner. Install with `sh .githooks/install.sh`. |
| [`.gitattributes`](.gitattributes) | Keeps the scripts and hooks at line-feed endings, without which none of them runs on a Windows checkout, and pins the fixtures whose Windows endings *are* the assertion. |
| [`docs/ci/`](docs/ci/) | CI templates (GitHub Actions and GitLab CI) that run the same gate on every pull request. The discipline jobs already run from [`.github/workflows/`](.github/workflows/). |
| [`docs/templates/`](docs/templates/) | Inert GitHub and GitLab issue and pull-request templates that embody the issue-first workflow. |

## Working on tally

1. Install the git hooks once per clone: `sh .githooks/install.sh`.
2. Take a task from [`docs/tasks/backlog.md`](docs/tasks/backlog.md), with an open
   issue, and follow the
   [quality gate](docs/engineering-discipline.md#working-a-task-under-the-quality-gate).
3. Write the failing test first. The product tests run through the Go toolchain;
   the exact commands for each test level are recorded in
   [`docs/tests/test-levels.md`](docs/tests/test-levels.md) by the task that
   selects them.
4. Keep the product on the Go standard library. A new dependency is a decision
   that needs an [ADR](docs/adr/).

## Where the process comes from

The discipline in this repository is the Armature engineering-discipline kit,
copied once at the commit pinned in [`docs/setup/armature.pin`](docs/setup/armature.pin)
and adapted to `tally`. The copy keeps no link to its source; a later version of
the kit comes in only by a change that names the new commit
([ADR-0009](docs/adr/0009-pin-the-baseline.md)).
