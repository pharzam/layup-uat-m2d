# Onboarding for engineers

> **Who this is for.** Software engineers joining `tally` who do **not**
> necessarily know its domain — counting text, Unicode and UTF-8. You know how to
> build software. This document teaches you the project's terminology and the few
> facts that shape every decision, then hands you off to the deeper docs.
>
> **Read this first.** Every other document is written for someone already fluent
> in the terminology. This one gets you to that point. Budget half an hour.
>
> **Keep the numbers in step.** This is the first document a new engineer reads,
> so a stale number here is worse than a stale number anywhere else (see the
> [plain-language-summaries](engineering-discipline.md#plain-language-summaries)
> rule). The numbers below come from the
> [problem statement](facts/problem-statement-brief.md) and the answers of the
> idea owner in [`F-0001`](facts/F-0001-setup-answers.md); when either changes,
> this file changes in the same pull request.

## 1. Problem statement

A person who works in a terminal wants to know how long a text file is. `tally`
is a small command, written in Go with the standard library only, that reads the
files the person names and prints the number of lines, words and characters of
each, and a total:

```text
$ tally notes.txt todo.txt
12 85 512 notes.txt
3 9 40 todo.txt
15 94 552 total
```

The flags `-l`, `-w` and `-c` select the lines, the words or the characters, and
they can be combined. With no file name, `tally` reads standard input. Out of
scope: other counts such as bytes or sentences, archives and directories, and any
graphical interface, network service or plug-in system.

The question the project answers is not "can we count" — the classic `wc` already
does — but "can we count *exactly to these rules*, in UTF-8, fast and in small
memory, with proof". No code exists yet: the facts are collected, and the first
task writes the requirements and the first tests.

## 2. Crash course: the domain

The vocabulary you will meet in every other document, in dependency order. The
[glossary](glossary.md) holds the same terms as a reference.

### 2.1 Bytes, code points and UTF-8

A file is a sequence of bytes. Unicode gives every character a number, a **code
point**. **UTF-8** writes each code point in one to four bytes:

```text
  a   →  61              one byte
  é   →  C3 A9           two bytes
  €   →  E2 82 AC        three bytes
```

For `tally`, a **character is one code point**, not one byte. `"héllo"` is five
characters in six bytes. A file whose bytes are not valid UTF-8 is not text:
`tally` reports it on standard error, prints no row for it, adds nothing of it to
the total, and ends with the exit code 1. In Go, a code point is a `rune`, and
the package `unicode/utf8` decodes and validates it.

### 2.2 Lines, building on characters

A **line** is a run of characters that ends with a line feed (LF, U+000A). The
last line counts even when the file does not end with a line feed:

```text
  "one\ntwo\n"   → 2 lines
  "one\ntwo"     → 2 lines   (the last line has no LF, and still counts)
  ""             → 0 lines
```

A carriage return (CR) is not a line end. In a file written on Windows, each line
ends with CR then LF; the CR is part of the line and counts as one character.

### 2.3 Words, building on characters

A **word** is a run of characters that are not white space, where white space is
any character of the Unicode property `White_Space` — not only the space and the
tab, but also the no-break space U+00A0 and the ideographic space U+3000.

```text
  "a b"        → 2 words
  "a b"   → 2 words   (a no-break space separates them)
  "  "         → 0 words
```

### 2.4 Rows, the total and the exit code

Each counted file gives one **row**: the selected counts in the fixed order lines,
words, characters — whatever the order of the flags — then the file name, each
separated by one space. The row for standard input has no name, and `-` is a file
named `-`, not standard input. With two files or more, the last row is the `total`
row: the sums of the files counted without an error.

A file that cannot be read — missing, without permission, or a directory — gives
a message on standard error with its name and the reason, and no row; the other
files are still counted. The **exit code** is 0 when no error occurs and 1
otherwise.

## 3. What the system actually does

One loop per file, and one streaming counter:

```text
  arguments ──► for each name (or standard input)
                   │
                   ├─ open ── fails ──► message on stderr, remember exit 1
                   │
                   └─ read in buffers of 64 KiB ──► package count
                                                     │  decode UTF-8 across buffers
                                                     │  count LF, words, code points
                                                     ▼
                         invalid UTF-8 ──► message on stderr, remember exit 1
                         valid        ──► print the row, add to the total
                ──► if two names or more: print the total row
                ──► exit 0, or 1 if any file failed
```

The Go module has two packages. The package `count` holds the functions that
count the lines, words and characters of a reader; the idea owner defined that
boundary, and the coverage target applies to it. The package `main` reads the
flags, opens the files, prints the rows and sets the exit code.

## 4. Why it is hard

Three numbers make the obvious design wrong:

- **64 MiB of heap for a file of 1 GiB.** Reading a whole file into memory —
  the first thing most code does — needs sixteen times the limit. So `tally`
  streams each file in fixed buffers of 64 KiB, and a code point of two to four
  bytes can be cut where one buffer ends and the next begins; both the counting
  and the detection of invalid UTF-8 must carry that cut across. The limit is on
  the *peak* heap, sampled every 10 milliseconds, not the heap at the end.
- **2 seconds for a file of 10 MiB** of ASCII text, for the whole process on a
  computer with 4 cores, with the file already in the cache. Generous for a
  plain loop, but a decoder that allocates for every rune can lose it.
- **90 percent statement coverage** of the package `count`, from
  `go test -cover`. Every branch of the edge cases needs a test.

And one soft fact: every edge — no final line feed, a CR, a no-break space, a cut
code point — passes a casual test on an ASCII file. The
[guardrails](guardrails.md) list them as known pitfalls.

## 5. How this project works

### The one cultural thing to understand

**A count is right only when a test says so.** The rules of `tally` are exact and
the edges are silent, so we write the failing test for an edge before the code
that handles it, and we freeze the numbers of the quality targets before we
measure them. A number we chose after seeing the result is not evidence.

Consequences you will meet immediately, and which are not negotiable:

- Test first: the failing test, watched failing for the right reason, then the
  code — gate step 3 in [`engineering-discipline.md`](engineering-discipline.md#working-a-task-under-the-quality-gate).
- The quality targets and their methods are pre-registered in
  [`guardrails.md`](guardrails.md#1-pre-registered-decisions--or-the-goalposts-move)
  and are never edited to make a run pass.
- Go standard library only: no third-party module enters `go.mod`.
- No change without an open issue, and every change lands through a pull request
  that links it — [`issue-workflow.md`](issue-workflow.md).

### Repo layout

| Path | What |
|------|------|
| `count/` (from the first task) | The package `count` and its unit tests. |
| The module root (from the first task) | `go.mod`, the package `main` of the command and its integration tests. |
| [`tests/`](../tests/) | The end-to-end tests that build and run the `tally` binary. |
| [`AGENTS.md`](../AGENTS.md) | The agent entry point — the gate in brief and a pointer to the R1–R13 rules, in one short file. [`CLAUDE.md`](../CLAUDE.md) imports it for Claude Code. |
| [`engineering-discipline.md`](engineering-discipline.md) | **How we work**: the quality gate, solution selection, branches, worktrees, commits, tests, reviews, and ADRs. Read before your first commit. |
| [`glossary.md`](glossary.md) | The shared vocabulary. Skim it; come back constantly. |
| [`facts/`](facts/) | Facts collected from the idea owner, stored as-is as immutable evidence. Derived requirements cite them by `F-NNNN` ID. |
| [`prd/`](prd/) | Product Requirements Documents, derived from the facts; each `REQ`/`NFR` cites an `F-NNNN` fact. |
| [`issue-workflow.md`](issue-workflow.md) | The issue-first rules (R1–R13): the ticket policy the gate assumes. |
| [`tasks/backlog.md`](tasks/backlog.md) | What to work on next. |

### What to read next, in order

0. [`AGENTS.md`](../AGENTS.md) — the one-page summary, if you are a coding agent
   (or a human who wants the shape before the detail).
1. [`engineering-discipline.md`](engineering-discipline.md) — how we work.
2. [`issue-workflow.md`](issue-workflow.md) — the issue-first rules the gate assumes.
3. [`glossary.md`](glossary.md) — skim, then reference.
4. [`guardrails.md`](guardrails.md) — the pitfalls and the frozen numbers.
5. [`tasks/backlog.md`](tasks/backlog.md) — what needs doing.

### Where the project stands

Built: nothing of the product yet. The repository holds the engineering
discipline taken from the Armature kit, the problem statement, and the answers of
the idea owner to the open questions. Next: the PRD that turns those facts into
requirements, then the package `count`, test first, then the command. The idea
owner set the appetite of the delivery at 48 hours of wall-clock time, and a money
band of 50.00 to 100.00 USD. Open question: whether a `total` row follows when
two files are named and only one of them is counted; the PRD settles it with the
idea owner before a test asserts it.
