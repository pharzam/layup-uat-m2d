# Onboarding for engineers

> **Who this is for.** Software engineers joining `tally` who do **not** necessarily
> know how text is counted. You know how to build software. This document teaches
> you the terms of the project and the few facts that shape every decision, then
> hands you off to the deeper documents.
>
> **Read this first.** Every other document is written for someone already fluent
> in these terms. This one gets you to that point. Budget half an hour.
>
> **Keep it current.** This is the first document a new engineer reads, so a stale
> number here is worse than a stale number anywhere else (see the
> [plain-language-summaries](engineering-discipline.md#plain-language-summaries)
> rule). When a headline number changes, change it here in the same pull request.

## 1. Problem statement

A person who works in a terminal wants to know how long a text file is. `tally` is a
small command, written in Go with the standard library only, that reads the files
the person names and prints the number of lines, words and characters of each, and
a total:

```
tally [-l] [-w] [-c] [file...]
```

With no file name, it reads standard input. The flags `-l`, `-w` and `-c` select the
lines, the words and the characters; with no flag, all three print. The full text
of the request is the [problem statement](facts/problem-statement-brief.md), and the
answers of the idea owner to its open questions are
[F-0001](facts/F-0001-setup-answers.md). Nothing is built yet: the questions are
answered, and the work starts from these two records.

Out of scope: other counts (bytes, paragraphs, sentences), archives, directories
and files that are not text, and any graphical interface, network service or
plug-in system.

## 2. Crash course: the domain

The terms below build on each other, in that order. The same vocabulary, as a
reference, is in [glossary.md](glossary.md).

### 2.1 Character

A **character** is one Unicode code point of a text in **UTF-8**. UTF-8 writes a
code point in one to four bytes, so a character and a byte are not the same thing:

```
text:        h   é        😀
bytes:       68  C3 A9    F0 9F 98 80
characters:  1   1        1            → 3 characters, 7 bytes
```

A file that is not valid UTF-8 is not text for `tally`: it gets no row, a message on
standard error with its name, and the exit code 1 (F-0001#6, F-0001#7). A carriage
return before a line feed is a character like any other (F-0001#8).

### 2.2 Line

A **line** is a run of characters that ends with a line feed. The last line of a
file counts even when the file does not end with a line feed. An empty file has
zero lines, zero words and zero characters (F-0001#8).

```
"a\nb\n"  → 2 lines
"a\nb"    → 2 lines (the last line has no line feed, and still counts)
""        → 0 lines
```

### 2.3 Word

A **word** is a run of characters that are not white space, where white space is
any character of the Unicode property `White_Space` — not only the space, the tab
and the line feed, but also, for example, the no-break space.

### 2.4 Row and total

The output has one **row** for each file: the selected counts, then the file name,
each separated from the next by one space. The counts always print in the fixed
order lines, words, characters, whatever the order of the flags (F-0001#2,
F-0001#3). The row for standard input has no name, and a file named `-` is just a
file (F-0001#4). With two files or more, the last row is the **total**: the sum of
the files counted without an error, followed by the word `total` (F-0001#5).

```
$ tally -w -l a.txt b.txt
3 12 a.txt
1 4 b.txt
4 16 total
```

### 2.5 Errors and exit code

A file that cannot be read — a directory included — gives a message on standard
error with its name and the reason, adds no row and nothing to the total, and the
other files are still counted (F-0001#7). Any such error makes the exit code 1;
with no error, it is 0.

## 3. What the system actually does

The command has two parts, and the boundary between them is fixed by the idea owner
(F-0001#12):

```
package main   flags → for each file: open → count → print row → total → exit code
                                        │
package count                           └─ reads a reader in buffers of 64 KiB,
                                           decodes UTF-8, counts lines, words, characters
```

The package `main` parses the flags, opens each file (or standard input), prints the
rows and the total, and sets the exit code. The package `count` holds the functions
that count the lines, the words and the characters of a reader. It reads in fixed
buffers of 64 KiB and never reads a whole file into memory; a UTF-8 sequence, and
the detection of invalid UTF-8, are handled across the boundary of two buffers
(F-0001#11).

## 4. Why it is hard

The counting rules are simple; the numbers make them hard.

- **Memory.** A file of 1 GiB must be counted with at most 64 MiB of peak heap,
  sampled every 10 milliseconds during the run (F-0001#10). A file sixteen times
  larger than the memory budget cannot be read whole, so the reading streams, and a
  character can fall across two buffers.
- **Time.** A file of 10 MiB must be counted in at most 2 seconds of elapsed time for
  the whole process, on a computer with 4 cores and a solid-state disk, with the
  file already in the cache (F-0001#9).
- **Coverage.** The package `count` must keep at least 90 percent statement
  coverage, as `go test -cover` reports it.

The obvious thing that does not work is to read the file into a string and range
over it: it breaks the memory target, and decoding each buffer on its own breaks the
count at buffer edges.

## 5. How this project works

### The one cultural thing to understand

We test the count before we trust it. A count that looks right is not evidence; a
test that fails first, for the right reason, at the edges — an empty file, a missing
final line feed, a character across a buffer boundary, an invalid byte — is.

Consequences you will meet immediately, and which are not negotiable:

- Every rule of counting is a frozen invariant in
  [guardrails.md](guardrails.md#2a-invariants-of-tally), and a change does not move
  one without a written reason.
- Every substantive task passes the quality gate of
  [engineering-discipline.md](engineering-discipline.md): an issue, a worktree, a
  test first, reviews, and the documentation kept current.

### Repo layout

| Path | What |
|------|------|
| [`facts/problem-statement-brief.md`](facts/problem-statement-brief.md) | The problem statement of the idea owner, kept as evidence. |
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

Nothing of `tally` is built yet. The problem statement is recorded, and the idea
owner has answered its open questions (F-0001). The next step is the requirements
derived from those facts, then the package `count` and its tests, then the command.
Keep this section in step with the backlog and the numbers above.
