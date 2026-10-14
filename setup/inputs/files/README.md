# tally

*A small Go command that counts the lines, words and characters of text files.*

```text
$ tally notes.txt todo.txt
12 85 512 notes.txt
3 9 40 todo.txt
15 94 552 total
```

A person who works in a terminal wants to know how long a text file is. `tally`
reads the files that the person names — or standard input, when there is no name
— and prints the number of lines, words and characters of each, and a total. It
is written in Go, with the standard library only, and built with the Go
toolchain.

## Usage

```text
tally [-l] [-w] [-c] [file...]
```

| Flag | Prints |
|------|--------|
| `-l` | The number of lines. |
| `-w` | The number of words. |
| `-c` | The number of characters. |
| none | All three. |

Flags can be combined. The counts are always printed in the order lines, words,
characters, whatever the order of the flags. Each row is the selected counts,
then the file name, separated by single spaces; the row for standard input has no
name. With two files or more, the last row is `total`, the sums of the files
counted without an error.

## What it counts

- **A line** is a run of characters that ends with a line feed. The last line
  counts when the file does not end with a line feed; an empty file has zero
  lines. A carriage return before a line feed is part of the line.
- **A word** is a run of characters that are not white space, where white space is
  any character of the Unicode property `White_Space`.
- **A character** is one Unicode code point of a text in UTF-8 — not one byte.

## Errors and exit codes

A file that cannot be read — missing, without permission, or a directory — gives
a message on standard error with the file name and the reason. A file that is not
valid UTF-8 gives a message on standard error with the file name. Neither gets a
row or a share of the total, and the other files are still counted. The exit code
is 0 when no error occurs and 1 otherwise. The name `-` is a file named `-`, not
standard input.

## Quality targets

| Metric | Target |
|--------|--------|
| Time for a file of 10 MiB | At most 2 seconds on a computer with 4 cores |
| Memory for a file of 1 GiB | At most 64 MiB of heap |
| Test coverage of the package `count` | At least 90 percent of the statements |

How each one is measured is frozen in
[`docs/guardrails.md`](docs/guardrails.md#1-pre-registered-decisions--or-the-goalposts-move).

## Status

No product code exists yet. The repository holds the problem statement and the
answers of the idea owner under [`docs/facts/`](docs/facts/), and the engineering
discipline the work follows. The next steps are on the
[backlog](docs/tasks/backlog.md). Once the code lands, the command is built with
`go build` and tested with `go test ./...`.

## Start here

1. **[`docs/onboarding-for-engineers.md`](docs/onboarding-for-engineers.md)** — read
   this first (about 30 minutes). It states the
   [problem statement](docs/onboarding-for-engineers.md#1-problem-statement) and
   teaches the vocabulary of counting text.
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
| [`AGENTS.md`](AGENTS.md) | The agent entry point: the quality gate, the checks, a pointer to the R1–R13 rules, and which document is authoritative for each. |
| [`CLAUDE.md`](CLAUDE.md) | One line, `@AGENTS.md`, so Claude Code loads the same guide. |
| [`docs/onboarding-for-engineers.md`](docs/onboarding-for-engineers.md) | The first door: the problem statement and a crash course in counting text. |
| [`docs/engineering-discipline.md`](docs/engineering-discipline.md) | The quality gate, the solution-selection standard, and every working practice. |
| [`docs/issue-workflow.md`](docs/issue-workflow.md) | The issue-first workflow (R1–R13): the ticket policy the gate assumes. |
| [`docs/glossary.md`](docs/glossary.md) | The shared vocabulary — lines, words, code points, UTF-8, the heap. |
| [`docs/guardrails.md`](docs/guardrails.md) | Known pitfalls of counting text, the frozen quality targets, and validation. |
| [`docs/adr/`](docs/adr/) | Architecture Decision Records — the *why* behind structural choices — plus [`adr-lint.sh`](docs/adr/adr-lint.sh), which keeps them honest. |
| [`docs/facts/`](docs/facts/) | The problem statement and the answers of the idea owner, kept as immutable evidence. |
| [`docs/prd/`](docs/prd/) | Product Requirements Documents derived from the facts, plus [`prd-lint.sh`](docs/prd/prd-lint.sh). |
| [`docs/tests/`](docs/tests/) | The testing conventions — the test levels, a pattern per level, the security, scaling, and Definition-of-Done checklists, and test-to-requirement traceability. |
| [`tests/`](tests/) | The end-to-end tests of the built `tally` binary. |
| [`docs/tasks/`](docs/tasks/) | The task index — [`backlog.md`](docs/tasks/backlog.md) and [`completed.md`](docs/tasks/completed.md). |
| [`.githooks/`](.githooks/) | Git hooks that enforce the cheap gate locally. Install with `sh .githooks/install.sh`. |
| [`docs/ci/`](docs/ci/) | The CI templates; the ready-as-is jobs already run under [`.github/workflows/`](.github/workflows/). |
| [`docs/templates/`](docs/templates/) | Inert GitHub and GitLab issue and PR templates. |

## Where the discipline comes from

The way of working — the quality gate, the issue rules, the linters and the hooks —
comes from the Armature engineering-discipline kit. The version it was copied
from is pinned in [ADR-0009](docs/adr/0009-pin-the-baseline.md); `tally` keeps no
link to the kit's history and takes later kit changes only by a change that names
the new version.
