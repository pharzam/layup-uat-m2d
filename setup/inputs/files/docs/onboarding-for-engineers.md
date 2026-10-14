# Onboarding for engineers

> **Who this is for.** Software engineers joining `tally` who have not worked on
> counting text before. You know how to build software and you know Go. This
> document teaches you the project's terminology and the few facts that shape
> every decision, then hands you off to the deeper docs.
>
> **Read this first.** Every other document is written for someone already fluent
> in the terminology. This one gets you to that point. Budget half an hour.

## 1. Problem statement

A person who works in a terminal wants to know how long a text file is. `tally`
is a small command for that person: it reads the files the person names — or
standard input, when the person names none — and prints the number of lines,
words and characters of each file, and a total.

```
tally [-l] [-w] [-c] [file...]
```

It is written in Go, with the standard library only. It does nothing else: no
byte, paragraph or sentence counts, no archives, no graphical interface, no
network service, no plug-ins.

The question the project answers is narrow but exact: *how do you count a text
file correctly, by the Unicode definitions, fast, and in a fixed amount of
memory, whatever the size of the file?* The idea owner has answered every open
question about the behaviour (see [`facts/F-0001`](facts/F-0001-setup-answers.md));
no code exists yet.

## 2. Crash course: counting text

The vocabulary below is in dependency order — each term builds on the last. The
same terms, with their exact definitions, are in [glossary.md](glossary.md).

### 2.1 Code point and UTF-8

Unicode gives each character a number, its **code point**, written `U+` and four
or more hexadecimal digits: `a` is `U+0061`, `é` is `U+00E9`, `€` is `U+20AC`.
**UTF-8** stores one code point in one to four bytes:

```
a   U+0061   61            1 byte
é   U+00E9   C3 A9         2 bytes
€   U+20AC   E2 82 AC      3 bytes
😀  U+1F600  F0 9F 98 80   4 bytes
```

In `tally`, **a character is one code point**. The text `héllo` is 5 characters
and 6 bytes. `-c` counts characters, not bytes — which is the opposite of what
`-c` means in the `wc` command. Keep that collision in mind.

A byte sequence that is not a correct UTF-8 encoding is **invalid UTF-8**: a lone
`C3`, a stray `80`, an overlong form. A file that holds any of it is not text for
`tally`; it is reported and skipped (section 3).

### 2.2 Line

A **line** is a run of characters that ends with a line feed (`U+000A`). The last
line counts even when the file does not end with a line feed:

```
"a\nb\n"   → 2 lines
"a\nb"     → 2 lines   (the last line has no line feed, and still counts)
""         → 0 lines
"\n"       → 1 line    (an empty line is still a line)
"a\r\nb"   → 2 lines, and the carriage return is a character of the first line
```

### 2.3 White space and word

**White space** is a character of the Unicode property `White_Space` — the space
and the tab, but also the line feed, the carriage return, the no-break space
`U+00A0`, the ideographic space `U+3000` and others. The zero-width space
`U+200B` is *not* white space.

A **word** is a run of characters that are not white space:

```
"one  two\tthree\n"   → 3 words
"a b"            → 2 words (U+00A0 is White_Space)
"a​b"            → 1 word  (U+200B is not)
```

### 2.4 Row, total and exit code

The output is one **row** per file: the selected counts, then the name, each
separated from the next by one space — no padding, no columns:

```
$ tally notes.txt todo.txt
12 85 512 notes.txt
3 9 41 todo.txt
15 94 553 total
```

The flags select counts and combine; the counts always print in the fixed order
lines, words, characters. The row for standard input has no name. The **total**
row appears with two files or more and sums only the files that were counted
without an error. The **exit code** is 0 when no error occurred and 1 otherwise.

## 3. What the system actually does

Two parts, with one boundary between them:

```
 command line ──▶ package main ──▶ package count ──▶ counts
   flags, names     opens each        reads 64 KiB      lines, words,
                    file, prints      buffers, decodes  characters,
                    rows, total,      UTF-8, counts     or "invalid"
                    errors, exit
```

- **Package `count`** holds the functions that count the lines, the words and
  the characters of a reader. It reads in fixed buffers of 64 KiB and never holds
  a whole file in memory. A UTF-8 sequence, and the detection of invalid UTF-8,
  can cross the boundary of two buffers, and must still be handled correctly.
- **Package `main`** parses the flags, opens each file in turn, asks `count` for
  the counts, and writes a row — or, when the file cannot be read or is not
  valid UTF-8, a message on standard error with the name. It keeps the total of
  the files counted without an error and sets the exit code at the end.

A file is only known to be valid UTF-8 when its last byte has been read, so a
row is written only after the whole file is counted.

## 4. Why it is hard

Three numbers, written down before any code exists:

- **Speed.** A file of 10 MiB of ASCII text, with lines of about 80 characters,
  is counted in at most **2 seconds**, measured from the start of the process to
  its exit, on a computer with 4 cores and a solid-state disk, with the file
  already in the cache of the operating system.
- **Memory.** Counting a file of 1 GiB keeps the **peak** heap at or under
  **64 MiB**. The peak is measured by sampling the heap in use every 10
  milliseconds during the run. That rules out the obvious approach — read the
  file, convert it to a string, range over it — which needs at least 1 GiB.
- **Coverage.** At least **90 percent** of the statements of package `count` are
  covered by tests, as reported by `go test -cover`.

The consequence: streaming in fixed buffers is not an optimisation, it is the
design. And streaming is exactly where the subtle bugs live — a character split
across two buffers, an invalid byte at a boundary, a word that spans two reads.
The traps are listed in [`guardrails.md`](guardrails.md).

## 5. How this project works

### The one cultural thing to understand

**An exact definition beats a familiar tool.** `tally` looks like `wc`, and it is
tempting to copy what `wc` does. Do not: `wc -c` counts bytes, `wc` pads its
columns, `wc` treats `-` as standard input, and its idea of white space depends
on the locale. Every behaviour here is pinned by a numbered fact, and a test
cites it. When the facts do not settle a case, you ask the idea owner on the
issue — you do not pick the answer that `wc` gives.

Consequences you will meet immediately, and which are not negotiable:

- Every behaviour has a test that cites the fact it comes from, written before
  the code ([Testing](engineering-discipline.md#testing),
  [Requirements traceability](engineering-discipline.md#requirements-traceability)).
- The quality targets are pre-registered and measured as written; a missed target
  is reported as missed
  ([guardrails §1](guardrails.md#1-pre-registered-decisions--or-the-goalposts-move),
  [Honesty and evidence](engineering-discipline.md#honesty-and-evidence)).
- The product uses the Go standard library only; a new dependency needs an
  [ADR](adr/).
- Every change goes through the
  [quality gate](engineering-discipline.md#working-a-task-under-the-quality-gate),
  on its own issue, in its own worktree.

### Repo layout

| Path | What |
|------|------|
| [`tests/`](../tests/) | The product tests of the built command, once they exist. Unit tests of a Go package sit beside its source, as Go expects. |
| [`AGENTS.md`](../AGENTS.md) | The agent entry point — the product in one screen, the gate in brief and a pointer to the R1–R13 rules, in one short file. [`CLAUDE.md`](../CLAUDE.md) imports it for Claude Code. |
| [`engineering-discipline.md`](engineering-discipline.md) | **How we work**: the quality gate, solution selection, branches, worktrees, commits, tests, reviews, and ADRs. Read before your first commit. |
| [`glossary.md`](glossary.md) | The shared vocabulary. Skim it; come back constantly. |
| [`facts/`](facts/) | The idea owner's facts, stored as-is as immutable evidence. Derived requirements cite them by `F-NNNN` ID. |
| [`prd/`](prd/) | Product Requirements Documents, derived from the facts; each `REQ`/`NFR` cites an `F-NNNN` fact. |
| [`guardrails.md`](guardrails.md) | The traps of counting text and the frozen quality targets. |
| [`issue-workflow.md`](issue-workflow.md) | The issue-first rules (R1–R13): the ticket policy the gate assumes. |
| [`tasks/backlog.md`](tasks/backlog.md) | What to work on next. |

### What to read next, in order

0. [`AGENTS.md`](../AGENTS.md) — the one-page summary, if you are a coding agent
   (or a human who wants the shape before the detail).
1. [`engineering-discipline.md`](engineering-discipline.md) — how we work.
2. [`issue-workflow.md`](issue-workflow.md) — the issue-first rules the gate assumes.
3. [`glossary.md`](glossary.md) — skim, then reference.
4. [`guardrails.md`](guardrails.md) — the pitfalls and the frozen numbers.
5. [`facts/F-0001-setup-answers.md`](facts/F-0001-setup-answers.md) and the
   [problem statement](facts/problem-statement-brief.md) — the idea owner's own
   words.
6. [`tasks/backlog.md`](tasks/backlog.md) — what needs doing.

### Where the project stands

- **Done:** the problem statement and the idea owner's answers to twelve open
  questions are recorded as facts; the engineering discipline is in place.
- **Not started:** the PRD, the Go module, package `count`, the command, and
  every product test. The commands for each test level are not selected yet.
- **Open questions:** whether the `total` row appears when two files are given
  and one of them fails, and how the problem statement's single target ("at most
  2 seconds", "the heap size at the end") is reconciled with the refinements in
  `F-0001` in the PRD. Both are listed in [`guardrails.md`](guardrails.md); a
  question goes to the idea owner on the issue.
