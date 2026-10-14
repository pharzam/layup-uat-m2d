# Guardrails — the known pitfalls, and the numbers you must not move

This document is the target of gate step 2, "Honor the guardrails", in
[`engineering-discipline.md`](engineering-discipline.md). It holds the pitfalls a
task author must take into account **before** writing code, so the team does not
re-derive a known trap every time.

It holds the guardrails of `tally`, the Go command that counts the lines, words and
characters of text files. It merges two kinds of guardrail that many projects keep
in separate files — **decision gates** (pre-registered pass/fail rules) and
**validation** (how we check we are not fooling ourselves). Both exist here and both
are read before work starts. Keep each entry short — a guardrail nobody reads is not
a guardrail.

The facts behind these rules are the problem statement,
[`facts/problem-statement-brief.md`](facts/problem-statement-brief.md), and the idea
owner's answers in [`facts/F-0001-setup-answers.md`](facts/F-0001-setup-answers.md),
cited as `F-0001#n`.

## In plain terms

The worst mistake `tally` can make is to print a count that looks right and is
wrong — a line, a word or a character miscounted at the edge of a buffer, in a
multi-byte character, or in a file that should have been refused — while the exit
code says all went well. The defense is to write the counting rules and the
quality targets down as frozen invariants before the code, and to test each one
first, at its edges.

## 1. Pre-registered decisions — or the goalposts move

A decision rule chosen **after** seeing the result is a fitted parameter, not a
rule. Write the pass/fail numbers first, somewhere they cannot be quietly edited.

- **What must be pre-registered:** the three quality targets of the problem
  statement — at most 2 seconds for a file of 10 MiB on the reference machine
  (F-0001#9), at most 64 MiB of peak heap for a file of 1 GiB (F-0001#10), and at
  least 90 percent statement coverage of the package `count` (F-0001#12) — and the
  counting rules of the invariants below.
- **Where the numbers freeze:** in this document and in the facts it cites, which
  are committed before the code that is judged against them; a test that measures
  a target reads its bound from the frozen value, never from a run.
- **The bands, not a single line:** prefer **Pass / Investigate / Fail** to a
  single pass line on a noisy measure. Define "Investigate" with a rule written
  before you look — for example, one re-examination whose scope is fixed in
  advance; landing there twice counts as Fail.

## 2. Known pitfalls — the traps specific to this domain

The failure modes that a command which counts text is known to meet. For each: the
trap, why it is silent, and the check that catches it.

- ❌ **A character split across two buffers.** `tally` reads in fixed buffers of
  64 KiB (F-0001#11), so a multi-byte UTF-8 sequence can start in one buffer and end
  in the next. Decoding each buffer alone counts the pieces as invalid bytes or as
  extra characters. It is silent because every test with a small file fits in one
  buffer. The check: a test that puts a sequence of two, three and four bytes across
  each position of a buffer boundary, and an invalid sequence there too.
- ❌ **Bytes counted as characters.** `len` of a Go string counts bytes; a character
  is one code point (problem statement, rule 3). It is silent on ASCII text, where
  the two agree. The check: a test whose text holds multi-byte characters.
- ❌ **Only ASCII white space.** A word ends at any character of the Unicode property
  `White_Space` (problem statement, rule 2), not only at a space, a tab or a line
  feed. It is silent on ASCII text. The check: a test with a no-break space and an
  ideographic space between words.
- ❌ **The last line lost or counted twice.** A file that does not end with a line
  feed still has a last line, and an empty file has zero lines (F-0001#8). It is
  silent because most test files end with a line feed. The check: tests for an
  empty file, a file with and without a final line feed, and a file of one line feed.
- ❌ **A peak hidden by the garbage collector.** The heap read at the end of a run can
  be small after a collection, while the peak during the run was large. The limit is
  on the peak, sampled every 10 milliseconds (F-0001#10). The check: the memory test
  samples `HeapAlloc` during the count and takes the largest value.
- ❌ **A failed file in the total.** A file that cannot be read, or that is not valid
  UTF-8, adds no row and nothing to the total (F-0001#5, F-0001#6). Counting what was
  read before the error is silent, because the total still looks plausible. The
  check: a test with one good and one bad file whose total equals the good file.

### Writing a lesson back (baseline-wide — keep this)

A trap caught once should not be re-derived by the next task, so a lesson does not
stay on the issue that learned it. When a task ends, its author asks whether the task
taught a trap the next reader could hit; if it did, the lesson is written **here** as a
new `❌` pitfall — the trap, why it is silent, and the check that catches it — in the
same pull request. Gate step 7 asks the question, so the rule is applied rather than
merely written (see [Keeping documentation current](engineering-discipline.md#keeping-documentation-current)).

This is the one **cross-task** reach the discipline adds on purpose.
[R6](issue-workflow.md#r6--agent-to-agent-communication-through-the-issue) and
[R7](issue-workflow.md#r7--decision-transparency-on-every-action) already keep the
coordination and the reasoning on the issue, and
[Honesty and evidence](engineering-discipline.md#honesty-and-evidence) already reports a
failure as a failure — but each is scoped to *one* issue thread. A lesson on issue #N is
discoverable only by someone who reads #N; §2 is where it reaches issue #N+1.

**The filter — or §2 grows until nobody reads it.** Write back only a trap that would
**catch the next reader**: a silent failure mode, a check that looked green for the
wrong reason, a footgun in the discipline or the domain. Do **not** write back a one-off with
no general lesson, a restatement of a rule that already lives elsewhere, or the
blow-by-blow of the task — those belong to the issue thread and the commit history.
Volume is the failure mode here, not absence: a pitfall list nobody finishes reading
guards nothing.

### Gate pitfalls (baseline-wide — keep these)

The gate is only as real as the thing that runs it. These traps let it report
success without having done its job.

- ❌ **An absolute `core.hooksPath`.** Worktrees **share** `.git/config`, so an
  absolute path binds every worktree to one checkout's hooks. A check added on a
  branch then does not run on that branch's own commits: the hook reports success
  having run something other than what the branch says it runs. It is silent
  because the hook still runs, still passes, and still uses the *right* files —
  only the *set of checks* comes from elsewhere. **The check:** install with a
  **relative** path, `git config core.hooksPath .githooks`, which git resolves per
  working tree. The `pre-commit` hook's own block 0 then refuses to run when the
  resolved hooks directory lies outside the tree being committed to, and
  [`.githooks/tests/provenance-check.sh`](../.githooks/tests/provenance-check.sh)
  proves it against real worktrees, reporting how many cases it ran rather than a
  count written down here to go stale.
  **A relative path escapes just as surely:** `../elsewhere/.githooks` is as
  foreign as any absolute one, so the check resolves the value instead of trusting
  that relative means local.
  **No path at all is the same trap:** with `core.hooksPath` unset git falls back
  to `.git/hooks`, which a linked worktree reaches through the shared *common* git
  directory. So block 0 judges the resolved directory whatever set it, and names
  the source it actually found — telling an operator to fix a setting they never
  set is its own dishonest report.
  **Bound on the damage:** CI invokes each check script directly and never through
  `core.hooksPath`, so this costs a local round trip, not a landed bug — a
  developer-experience gap, not an open gate.
- ❌ **A check that cannot fail.** A grep whose pattern also matches its own error
  message, a fixture harness that compares only exit codes, a coverage floor that
  counts zero as success. The check: for every assertion, make it fail on purpose
  once and read the reason — a green nobody attacked is not evidence.
- ❌ **A check that runs but does not block.** CI is green, the pull request
  merges, and nothing connects the two: no check is required on the default
  branch, so the green was a run result, not a merge control, and a red would
  have merged the same way. It is silent because the run result looks identical
  either way. The check:
  [make the checks required](ci/README.md#make-the-checks-required) on the
  default branch, and take the branch API read as the evidence — not the green run.

- ❌ **A check the change supplies is not a control.** CI checks out the pull
  request's own head and then runs the check from that checkout, so **the script
  that judges the change comes from the change**. Measured: a branch that replaces
  `docs/links/link-lint.sh` with `exit 0` passes that job — and replacing
  `docs/tests/run-discipline-tests.sh` as well turns **every** required job green
  over a dead link in the tree. Gutting a linter alone does not, because the
  fixture harness asserts exit codes and 19 `bad-*` cases stop failing; the runner
  is the single point.
  **The trap inside the remedy:** each script roots its scan at its own directory
  (`dirname $0`), so running the default branch's copy *where it sits* lints the
  wrong tree and reports OK. That was measured too, while building the fix.
  **The check:** every job in [`ci.yml`](../.github/workflows/ci.yml) restores the
  check scripts from the default branch **in place** before running them, so the
  branch's copy is never the judge.
  **Bound on the damage, and it is not nil:** a `pull_request` event runs the
  workflow as the branch has it, so a branch that edits `ci.yml` removes the
  restore step — closed only by review of `.github/**`, which wants a `CODEOWNERS`
  entry and a second human the forge knows about. Restoring also stops a bypass,
  not a merge: once a weakened check lands it *is* the default branch's copy. And
  a change that *improves* a check is judged by the older copy, so it lands in two
  steps. These checks are a control against forgetting, not against an operator
  who edits the check.

### Testing pitfalls (baseline-wide — keep these)

These traps are not domain-specific: they hurt every test suite, so the baseline
wrote them down already. Keep them, and add the traps of `tally` to section 2 above.

- ❌ **Testing after the code.** A test written to fit code that already "works"
  tends to encode the code's bugs as expected behaviour. The check: write the test
  first and watch it fail for the right reason
  ([strict TDD](engineering-discipline.md#requirements-traceability)).
- ❌ **Tests that depend on external state.** A test that reads a shared database, a
  live network, the wall clock, or another test's leftovers passes or fails for
  reasons unrelated to the code. The check: isolate and control every dependency,
  with a fresh fixture per run — see
  [`tests/scaling-checklist.md`](tests/scaling-checklist.md).
- ❌ **Tests that pass for the wrong reason.** A test that asserts nothing, asserts
  the wrong thing, or never actually exercises the path reports a safety that is not
  there — worse than no test. The check: confirm the test fails when the behaviour
  is broken; the red step is the proof.
- ❌ **Stale tests after a requirement changes.** When a requirement changes but its
  test does not, the suite now guards the old behaviour and blocks the new. The
  check: the [old-tests conflict rule](engineering-discipline.md#testing) — fix the
  code, update the requirement with a written reason, or retire the test; never
  weaken a passing old test.
- ❌ **Tests that slow down as the project grows.** A suite that creeps past the
  hook's patience gets skipped, and a skipped gate is no gate. The check: keep the
  cheap levels fast and cheap-first, push slow ones to CI, and bound each with
  a timeout, set with `go test -timeout` — see [`tests/scaling-checklist.md`](tests/scaling-checklist.md).

### Reference-sweep pitfalls (baseline-wide — keep these)

A change that edits references or a rule's wording across the tree has three silent
failure modes worth keeping.

- ❌ **A blanket find-and-replace over a renamed record's citations.** When a record
  moves or a directory is renumbered, the same bare token can name *different*
  records in two places — a bare `ADR-0005` is the living `docs/adr/` record to one
  reader and the archived `docs/decisions/` one to another, because the two sequences
  once shared numbers. A global replace of the token silently rewrites the citations
  you must **not** touch alongside the ones you must; and the reverse — a citation the
  sweep's pattern never matched (a compound like `ADR-0003/0005`, a token in a code
  span or a `.sh`/`.yml` comment, one split across a line break) — is silently *left*
  pointing at the wrong record. It is silent because **no linter catches it**:
  `link-lint` checks only that a *link* resolves, and a bare textual mention resolves
  to nothing, so a citation that now sends a reader to the wrong record still passes
  every check. **The check:** classify each occurrence by its **link target**, not its
  token — a link into `../adr/` is the living record and stays, a link into
  `../decisions/` is the archive and is rewritten — and read every *bare* mention by
  hand, in every token shape, since it carries no path to classify it. A
  pre-registered grep that must finish returning only the intended survivors (the
  mapping table and deliberate historical prose) is the closest thing to a gate; run
  it against the whole tree, not only the files you expected to touch.
- ❌ **A repoint that orphans a bare back-reference.** Repointing a citation can
  strand a *different* reference that named the target only through it. A comment
  reading `section 6 says …` leaned on a nearby `D-0003 section 6` for its antecedent;
  repoint every `D-0003 section 6` and the bare `section 6` is left pointing at a
  structure only the deleted record holds — wrong on the tree of the repository that took the change, and sharing
  **no token** with the thing you renamed. It is silent because a grep keyed on the
  obvious token (`D-000N`) cannot match a bare `section 6`, so the pre-registered
  check goes green over the survivor. **The check:** grep for the *shapes* a reference
  takes, not only the token — a bare `section N`, a `§`, a pronoun (`that section`,
  `the record`) whose antecedent you removed — and read the neighbourhood of every
  citation you changed, not the citation alone.
- ❌ **Editing a rule whose decision record is archived.** A rule lives in two places
  — its operative statement in a living doc, and the immutable decision record that
  first set it under `docs/decisions/`. Change the living one and the archived one
  still asserts the old, and **no check compares them** (`adr-lint` never reads
  `docs/decisions/`; `link-lint` checks resolution, not agreement). You cannot rewrite
  the immutable body to match; discharge the divergence with a `Status`-line
  **amendment pointer** on the archived record. **The check:** grep the whole tree —
  archive and forge templates included — for the old wording, and reconcile each living
  mirror or point each immutable one; a dated log entry recording history stays.
- ❌ **A hand-mirrored count or check-set that no linter guards.** The set of discipline
  linters — and how many there are — is spelled out by hand across many living
  docs, among them [`engineering-discipline.md`](engineering-discipline.md), this file,
  [`tests/test-levels.md`](tests/test-levels.md), the two `tests/README.md` files,
  [`ci/README.md`](ci/README.md) and
  [`.githooks/README.md`](../.githooks/README.md). Add or remove a check and every one
  can go stale, and a **removed** check leaves its name behind as a linter that no longer
  exists — a `link-lint` run stays green, because it resolves a *link*, not a claim. It is
  silent because the sentence still reads well and the count still looks deliberate: the
  baseline once said `three`, `four` and `five` at once, and named an `agent-entry` linter that
  had been cut. **The check:** when you add or remove a discipline check, grep the whole
  tree for the check-set enumeration — the old name and each spelled count — and reconcile
  every living mirror in the same change; the immutable ADR and archived decision copies
  stay as history.

## 2a. Invariants of tally

Each invariant is a rule that every version of `tally` keeps. Its `Check:` line names
the file that enforces it, or says that no check enforces it yet.

- **Inv-1** — A line is a run of characters that ends with a line feed; the last
  line counts when the file does not end with a line feed; an empty file has zero
  lines, zero words and zero characters (F-0001#8).
  Check: no check yet
- **Inv-2** — A word is a run of characters that are not white space, where white
  space is a character of the Unicode property `White_Space`.
  Check: no check yet
- **Inv-3** — A character is one Unicode code point; a carriage return before a line
  feed is a character like any other (F-0001#8).
  Check: no check yet
- **Inv-4** — The selected counts print in the fixed order lines, words, characters,
  whatever the order of the flags; with no flag all three print (F-0001#2, F-0001#3).
  Check: no check yet
- **Inv-5** — The row for standard input has no name, and the name `-` is a file
  named `-` (F-0001#4).
  Check: no check yet
- **Inv-6** — The total row sums only the files counted without an error; a file that
  cannot be read, a directory, or a file that is not valid UTF-8 adds no row, gives a
  message on standard error with its name, and makes the exit code 1 (F-0001#5,
  F-0001#6, F-0001#7).
  Check: no check yet
- **Inv-7** — `tally` reads each file in fixed buffers of 64 KiB, never a whole file,
  and decodes UTF-8 across the boundary of two buffers (F-0001#11).
  Check: no check yet
- **Inv-8** — The package `count` keeps at least 90 percent statement coverage
  (F-0001#12).
  Check: no check yet

## 3. Validation — how you check you are not fooling yourself

A result is **untrusted** until it passes the checks below, and the pass is a
recorded event, not a memory. Order the checks cheap-first, so a failure stops the
expensive ones.

| # | Check | Pass condition | Cost |
|---|-------|----------------|------|
| 1 | `go vet ./...` and `go test ./...` | no finding, and every test passes | minutes |
| 2 | `go test -cover` on the package `count` | at least 90 percent of the statements (F-0001#12) | minutes |
| 3 | the time test on a generated file of 10 MiB | at most 2 seconds of elapsed time for the whole process, on the reference machine (F-0001#9) | minutes |
| 4 | the memory test on a generated file of 1 GiB | at most 64 MiB of peak heap (F-0001#10) | tens of minutes |

Notes on how to read a failure: checks 1 and 2 catch a wrong count and an untested
path, and are cheap enough to run in CI on every change. Checks 3 and 4 catch a
regression of speed or memory; the time test means something only on the reference
machine, and the memory test generates a large file, so both run once per change to
the reading or counting code, and their result is recorded on the issue.

**The automated gate is this validation layer, mechanized.** The cheap, always-on
checks — the [discipline linters](engineering-discipline.md#testing) the baseline
ships (ADR, PRD and link) and their
[fixture self-tests](engineering-discipline.md#testing), the
[test levels](engineering-discipline.md#testing), lint, a security
scan, and the [commit-format](engineering-discipline.md#commit-messages)
check — run in the [`pre-commit` hook](engineering-discipline.md#git-hooks) for
fast local feedback and in [CI](engineering-discipline.md#continuous-integration)
as the authority. Treat those checks as pre-registered pass/fail rules under
section 1: they predate any single result and are not edited to make a change
pass. Wire the "cheap enough to wire into CI" checks from the table above into
both layers.

## 4. Mechanics

- **Frozen rules do not get edited.** Changing a guardrail after it is set means a
  new version with a written reason, the old one preserved. Legitimate reasons
  exist (a bug in the measure); silent edits do not.
- **The facts hold the frozen values; this document holds the structure.** The
  numbers of `tally` come from the problem statement and from F-0001; a value here
  mirrors its fact and cites it, and is never the primary copy.

## Sources

- [`facts/problem-statement-brief.md`](facts/problem-statement-brief.md) — the
  behaviour and the quality targets of `tally`.
- [`facts/F-0001-setup-answers.md`](facts/F-0001-setup-answers.md) — the answers
  that fix the flags (F-0001#2, F-0001#3), standard input (F-0001#4), the total row
  (F-0001#5), invalid and unreadable files (F-0001#6, F-0001#7), line ends
  (F-0001#8), the reference machine (F-0001#9), the heap peak (F-0001#10), the
  buffers (F-0001#11) and the package `count` (F-0001#12).
