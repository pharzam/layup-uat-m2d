# tally

*A small Go command that counts the lines, words and characters of text files.*

This is the repository `pharzam/layup-uat-m2d`. It holds `tally`: a person who
works in a terminal names one or more text files, and `tally` prints the number of
lines, words and characters of each, and a total. With no file name it reads
standard input.

```
tally [-l] [-w] [-c] [file...]
```

- `-l` selects the lines, `-w` the words and `-c` the characters. Flags may be
  combined; the selected counts always print in the fixed order lines, words,
  characters, and with no flag all three print.
- Each row holds the selected counts and then the file name, separated by one
  space. The row for standard input has no name. With two files or more, the last
  row is the `total` of the files that were counted without an error.
- A file that cannot be read, a directory, or a file that is not valid UTF-8 gives
  a message on standard error, adds no row and nothing to the total, and makes the
  exit code 1. The other files are still counted. With no error the exit code is 0.

The product is written in Go with the standard library only, and built with the Go
toolchain. The full statement of the problem is
[`docs/facts/problem-statement-brief.md`](docs/facts/problem-statement-brief.md),
and the answers that close its gaps are
[`docs/facts/F-0001-setup-answers.md`](docs/facts/F-0001-setup-answers.md).

## Start here

1. **[`docs/onboarding-for-engineers.md`](docs/onboarding-for-engineers.md)** — read this first
   (about 30 minutes). It states the [problem](docs/onboarding-for-engineers.md#1-problem-statement)
   and teaches the vocabulary of `tally`: line, word, character, UTF-8, the total row.
2. **[`docs/engineering-discipline.md`](docs/engineering-discipline.md)** — how we work: the
   quality gate every substantive task passes, plus branches, tests, reviews and
   ADRs. Read it before your first commit.
3. **[`AGENTS.md`](AGENTS.md)** — if you are a coding agent, start here instead. It
   summarises [`docs/engineering-discipline.md`](docs/engineering-discipline.md)
   and [`docs/issue-workflow.md`](docs/issue-workflow.md), and indexes the rest.
   The long documents stay authoritative.

## What's inside

| Piece | What it holds |
|-------|---------------|
| [`AGENTS.md`](AGENTS.md) | The agent entry point: the quality gate, the checks, a pointer to the R1–R13 rules, and which document is authoritative for each. |
| [`CLAUDE.md`](CLAUDE.md) | One line, `@AGENTS.md`, so Claude Code loads the same guide. No second copy to drift. |
| [`docs/agents/`](docs/agents/) | What the entry points are and what they may not become ([ADR-0004](docs/adr/0004-ship-agent-entry-points.md)). |
| [`docs/onboarding-for-engineers.md`](docs/onboarding-for-engineers.md) | The first door: the problem statement and a crash course in counting text. |
| [`docs/engineering-discipline.md`](docs/engineering-discipline.md) | The quality gate, the solution-selection standard, and every working practice. |
| [`docs/issue-workflow.md`](docs/issue-workflow.md) | The issue-first workflow (R1–R13): the ticket policy the gate assumes. |
| [`docs/glossary.md`](docs/glossary.md) | The shared vocabulary the other documents assume. |
| [`docs/guardrails.md`](docs/guardrails.md) | Known pitfalls of `tally`, pre-registered pass and fail rules, and validation. |
| [`docs/adr/`](docs/adr/) | Architecture Decision Records — the *why* behind structural choices — plus [`adr-lint.sh`](docs/adr/adr-lint.sh), the discipline test that keeps them honest. |
| [`docs/facts/`](docs/facts/) | The idea owner's problem statement and answers, kept as immutable evidence, and the citation convention that derives requirements from them. |
| [`docs/prd/`](docs/prd/) | Product Requirements Documents derived from the facts, plus [`prd-lint.sh`](docs/prd/prd-lint.sh), the discipline test that keeps them honest. |
| [`docs/tests/`](docs/tests/) | The testing conventions — the test levels, a pattern per level, the security, scaling and Definition-of-Done checklists, and test-to-requirement traceability — plus [`run-discipline-tests.sh`](docs/tests/run-discipline-tests.sh), which tests the document linters against fixtures. |
| [`tests/`](tests/) | The repository-root place for the product tests of `tally` that sit outside a Go package. |
| [`docs/tasks/`](docs/tasks/) | The task index — [`backlog.md`](docs/tasks/backlog.md) and [`completed.md`](docs/tasks/completed.md). |
| [`.githooks/`](.githooks/) | Git hooks that enforce the cheap gate locally — a commit-message check and a pre-commit runner. Install with `sh .githooks/install.sh`. |
| [`.gitattributes`](.gitattributes) | Keeps the scripts and hooks at line-feed endings, without which none of them runs on a Windows checkout, and pins the fixtures whose Windows endings *are* the assertion. |
| [`docs/ci/`](docs/ci/) | CI definitions (GitHub Actions and GitLab CI) that run the same gate on every pull request. |
| [`docs/templates/`](docs/templates/) | GitHub and GitLab issue and pull-request forms that embody the issue-first workflow. |

## Where the discipline comes from

The way of working in this repository comes from Armature, an engineering-discipline
baseline. It was copied once, at a fixed commit, and is not a dependency: the pin
that records the source, the commit and the method of the copy is
[`docs/setup/armature.pin`](docs/setup/armature.pin). Later improvements to
Armature are taken by hand, if and when the team wants them.

The repository was set up with LAYUP. The records of that setup — the questions,
the answers and the steps — are kept on the branch `layup-records`, apart from the
work on `main`.

## Checks

The checks read only text. Install the hooks once per clone with
`sh .githooks/install.sh`; the list of checks, and which run locally and which in
CI, is in [`AGENTS.md`](AGENTS.md#checks-you-can-run). The product itself is built
and tested with the Go toolchain.
