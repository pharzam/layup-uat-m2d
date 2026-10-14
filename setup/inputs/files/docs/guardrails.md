# Guardrails — the known pitfalls, and the numbers you must not move

This document is the target of gate step 2, "Honor the guardrails", in
[`engineering-discipline.md`](engineering-discipline.md). It holds the pitfalls a
task author on `tally` must take into account **before** writing code, so the
team does not re-derive a known trap every time.

It merges two kinds of guardrail — **decision gates** (the pre-registered
pass/fail rules of the quality targets) and **validation** (how you check you are
not fooling yourself). Both are read before work starts.

## In plain terms

> The easy mistake is a `tally` that agrees with `wc` on a small English file and
> is wrong everywhere else: on a character split between two reads, on a file that
> is not valid UTF-8, on a space that is not the ASCII space, on a file of 1 GiB.
> The defense is that every count rule, and every speed and memory limit, is
> written down here before the code exists, and each one has a test that was seen
> to fail first.

## 1. Pre-registered decisions — or the goalposts move

A decision rule chosen **after** seeing the result is a fitted parameter, not a
rule. The numbers below are frozen before any code exists.

- **What is pre-registered:** the three quality targets of the problem statement,
  as the idea owner refined them in [`F-0001`](facts/F-0001-setup-answers.md):

  | Target | Pass | Fail | Measurement | Facts |
  |--------|------|------|-------------|-------|
  | Speed | at most 2.0 seconds | over 2.0 seconds | A test generates a file of 10 MiB (10,485,760 bytes) of ASCII text with lines of about 80 characters, runs the built command on it with the file already in the cache of the operating system, and measures the elapsed time of the whole process, from its start to its exit. Reference: a computer with 4 cores and a solid-state disk. | problem statement §4, `F-0001#9` |
  | Memory | peak heap at most 64 MiB (67,108,864 bytes) | peak heap over 64 MiB | A test generates a file of 1 GiB, counts it, and samples `runtime.ReadMemStats` (`HeapAlloc`) every 10 milliseconds during the count; the largest sample is the result. | problem statement §4, `F-0001#10` |
  | Coverage | at least 90.0 percent of the statements of package `count` | under 90.0 percent | The output of `go test -cover` for package `count`. | problem statement §4, `F-0001#12` |

- **Where the numbers freeze:** here, and in the `NFR` rows of the PRD that
  derives from the facts. The facts records are immutable, so a number in this
  table that disagrees with its fact is a defect in this table.
- **The bands:** the speed target is a noisy measure on shared CI hardware. A
  result over 2.0 seconds on a machine that does not match the reference (4
  cores, solid-state disk, warm cache) is **Investigate**, not Pass and not Fail:
  run it once more on a reference machine, with the file in the cache, and record
  both results. A result over 2.0 seconds on a reference machine is Fail. Landing
  in Investigate twice for the same change counts as Fail. Memory and coverage
  are deterministic enough for a single line: no Investigate band.
- **The budget:** the delivery's money band is B = 50.00 USD and U = 100.00 USD,
  and its wall-clock appetite is 48.0 hours (`F-0001#13`–`#15`). Below B the
  delivery goes on without asking the idea owner; the band is not moved after a
  spend is seen.

## 2. Known pitfalls — the traps specific to this domain

Each pitfall names the trap, why it is silent, and the check that catches it.

- ❌ **A UTF-8 sequence split across two buffers.** `count` reads 64 KiB at a time
  (`F-0001#11`), and a two-, three- or four-byte sequence can start at the end of
  one buffer and finish in the next. Decoding each buffer on its own counts the
  halves as invalid bytes, or as two characters. It is silent because a test file
  smaller than 64 KiB, or one of ASCII only, never puts a sequence on a boundary.
  **The check:** make the buffer size a parameter of `count` that a test can set
  small (for example 1, 2 and 3 bytes), and assert the same counts for every
  size, on text with characters of every length.
- ❌ **Invalid UTF-8 at a boundary, or at the end.** The mirror of the last trap:
  an incomplete sequence at the end of a buffer is not yet invalid, but an
  incomplete sequence at the end of the *file* is. A reader that only checks
  inside each buffer accepts a file that ends in a lone `C3`. **The check:** a
  test file that ends with each incomplete prefix of a multi-byte sequence, at
  every buffer size, must be reported invalid.
- ❌ **Taking a real `U+FFFD` for an error.** Go's decoder returns
  `utf8.RuneError` (`U+FFFD`) both for an invalid byte and for the valid encoding
  `EF BF BD`. A check on the returned rune alone reports a valid file as not
  UTF-8. It is silent because `U+FFFD` is rare in test data. **The check:**
  decide on the rune *and* its size (an invalid byte gives size 1), and test a
  file that holds a real `U+FFFD`.
- ❌ **A row printed before the file is known to be valid.** A file is valid
  UTF-8 only when its last byte has been read. A command that prints counts as it
  goes, or adds a file to the total before the end, prints a row and a share of
  the total for a file that must have neither (`F-0001#5`, `F-0001#6`). **The
  check:** an end-to-end test with an invalid byte in the last buffer of a file
  larger than 64 KiB asserts no row, an unchanged total and exit code 1.
- ❌ **The wrong set of white space.** A hand-written list (`' '`, `'\t'`, `'\n'`)
  misses `U+00A0`, `U+0085`, `U+2028`, `U+3000` and the rest of `White_Space`; a
  generous list adds `U+200B` or `U+FEFF`, which are not in it. `wc` follows the
  locale, so comparing with `wc` does not catch it. **The check:** test against
  the property itself, `unicode.White_Space` in the standard library, with a
  table that holds every member and the near misses `U+200B`, `U+200C`, `U+200D`,
  `U+2060` and `U+FEFF`. `unicode.IsSpace` agrees with that property today; a test
  that pins it keeps that true.
- ❌ **A word that spans two buffers.** The state "inside a word" must survive the
  end of a buffer, or one word read in two parts counts as two. **The check:** the
  same small-buffer test as above, on text with long words.
- ❌ **The last line and the empty file.** Counting line feeds gives one line too
  few when the file does not end with a line feed (rule 1), and adding one for
  "the last line" gives one too many for an empty file and for a file that does
  end with one (`F-0001#8`). **The check:** a table test over `""`, `"\n"`, `"a"`,
  `"a\n"`, `"a\nb"` and `"a\r\nb"`, with the carriage return counted as a
  character.
- ❌ **Copying `wc`.** `wc -c` counts bytes, `wc` pads its columns, treats `-` as
  standard input and prints the counts in its own order for any combination of
  flags. `tally` counts code points, separates fields by exactly one space, treats
  `-` as a file named `-` and prints lines, words, characters in that order
  whatever the order of the flags (`F-0001#2`–`#4`). It is silent because `wc` is
  where a developer checks a result. **The check:** expected outputs come from
  the facts and are written in the test, never captured from another tool.
- ❌ **A directory that opens without an error.** On Linux, opening a directory
  succeeds and the error comes only on the first read. A command that checks only
  the open treats a directory as an empty file and prints a row of zeros. A
  directory is an unreadable file (`F-0001#7`). **The check:** an end-to-end test
  that passes a directory asserts a message with its name and the reason, no row
  and exit code 1.
- ❌ **Measuring the heap at the end.** The problem statement says "reads the heap
  size at the end"; `F-0001#10` replaces that with the peak during the run,
  because a garbage collection just before the end can make a design that read
  the whole file look small. **The check:** the memory test samples during the
  count, as §1 states, and its own test shows it reports a peak that an
  allocation of 100 MiB inside the count would push over the limit.
- ❌ **A timing test that measures only the counting.** Calling `count` from a
  test and timing it leaves out the start of the process, the opening of the file
  and the writing of the output, which `F-0001#9` includes. **The check:** the
  speed test runs the built command as a process and times it from start to exit.
- ❌ **A requirement that cites the problem statement and fails `prd-lint`.** The
  problem statement is kept in [`facts/`](facts/) as
  `problem-statement-brief.md`, with no `F-NNNN` ID, and
  [`prd-lint.sh`](prd/prd-lint.sh) accepts only a cited fact that resolves to an
  `F-NNNN` file. A requirement that cites the brief by name has no resolvable
  fact. **The check:** cite `F-0001` where it states the rule, and give the brief
  an ID through a new facts record, as
  [Correcting a facts document](facts/README.md#correcting-a-facts-document)
  allows, before a requirement rests on it alone.
- ❌ **A generated file of 1 GiB left behind.** The memory test writes 1 GiB. A
  test that writes it into the tree, or does not remove it on failure, fills a
  disk or a commit. **The check:** generate it in a temporary directory that the
  test removes in every outcome, and keep the test out of the commit hook.

### Open questions — not yet decided

These are recorded so that no task decides them silently. Each goes to the idea
owner on an issue, and the answer becomes a facts record.

- **The total row when one of two files fails.** Rule 4 says "with two files or
  more, the last row is `total`", and `F-0001#5` says the total sums only the
  files counted without an error. When two files are named and one fails, one
  reading prints a total of the one counted file; another prints none, because
  only one row was printed. That is two honest readings of one statement
  ([R13](issue-workflow.md#r13--one-reading-not-two)).
- **The message format on standard error.** Rules 5 and 6 fix the content (the
  name, and for rule 5 the reason) but not the form, so a test asserts the content
  and not the exact text until a fact fixes it.

### Writing a lesson back (kit-wide — keep this)

A trap caught once should not be re-derived by the next task, so a lesson does not
stay on the issue that learned it. When a task ends, its author asks whether the task
taught a trap the next reader could hit; if it did, the lesson is written **here** as a
new `❌` pitfall — the trap, why it is silent, and the check that catches it — in the
same pull request. Gate step 7 asks the question, so the rule is applied rather than
merely written (see [Keeping documentation current](engineering-discipline.md#keeping-documentation-current)).

This is the one **cross-task** reach the kit adds on purpose.
[R6](issue-workflow.md#r6--agent-to-agent-communication-through-the-issue) and
[R7](issue-workflow.md#r7--decision-transparency-on-every-action) already keep the
coordination and the reasoning on the issue, and
[Honesty and evidence](engineering-discipline.md#honesty-and-evidence) already reports a
failure as a failure — but each is scoped to *one* issue thread. A lesson on issue #N is
discoverable only by someone who reads #N; §2 is where it reaches issue #N+1.

**The filter — or §2 grows until nobody reads it.** Write back only a trap that would
**catch the next reader**: a silent failure mode, a check that looked green for the
wrong reason, a footgun in the kit or the domain. Do **not** write back a one-off with
no general lesson, a restatement of a rule that already lives elsewhere, or the
blow-by-blow of the task — those belong to the issue thread and the commit history.
Volume is the failure mode here, not absence: a pitfall list nobody finishes reading
guards nothing.

### Gate pitfalls (kit-wide — keep these)

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

### Testing pitfalls (kit-wide — keep these)

These traps are not specific to counting text: they hurt every project's test
suite. Keep them, and add your own above.

- ❌ **Testing after the code.** A test written to fit code that already "works"
  tends to encode the code's bugs as expected behaviour. The check: write the test
  first and watch it fail for the right reason
  ([strict TDD](engineering-discipline.md#requirements-traceability)).
- ❌ **Tests that depend on external state.** A test that reads a shared file, a
  live network, the wall clock, or another test's leftovers passes or fails for
  reasons unrelated to the code. The check: isolate and control every dependency,
  with a fresh fixture per run — see
  [`tests/scaling-checklist.md`](tests/scaling-checklist.md). The speed test is
  the one test that reads the clock, which is why §1 gives it a band.
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
  cheap levels fast and cheap-first, push slow ones — the 10 MiB and 1 GiB tests
  among them — to CI, and bound each with the test timeout defined in
  [`tests/test-levels.md`](tests/test-levels.md) — see
  [`tests/scaling-checklist.md`](tests/scaling-checklist.md).

### Reference-sweep pitfalls (kit-wide — keep these)

A change that edits references or a rule's wording across the tree has silent
failure modes worth keeping.

- ❌ **A blanket find-and-replace over a renamed record's citations.** When a record
  moves or a directory is renumbered, the same bare token can name *different*
  records in two places. A global replace of the token silently rewrites the
  citations you must **not** touch alongside the ones you must; and the reverse — a
  citation the sweep's pattern never matched (a compound like `ADR-0003/0005`, a
  token in a code span or a `.sh`/`.yml` comment, one split across a line break) —
  is silently *left* pointing at the wrong record. It is silent because **no linter
  catches it**: `link-lint` checks only that a *link* resolves, and a bare textual
  mention resolves to nothing, so a citation that now sends a reader to the wrong
  record still passes every check. **The check:** classify each occurrence by its
  **link target**, not its token, and read every *bare* mention by hand, in every
  token shape, since it carries no path to classify it. A pre-registered grep that
  must finish returning only the intended survivors is the closest thing to a gate;
  run it against the whole tree, not only the files you expected to touch. The same
  holds for fact citations: `F-0001#9` and `F-0001#10` differ by one character.
- ❌ **A repoint that orphans a bare back-reference.** Repointing a citation can
  strand a *different* reference that named the target only through it — a bare
  `section 6`, a `§`, a pronoun (`that section`, `the record`) whose antecedent you
  removed. It is silent because a grep keyed on the obvious token cannot match it.
  **The check:** grep for the *shapes* a reference takes, not only the token, and
  read the neighbourhood of every citation you changed, not the citation alone.
- ❌ **Editing a rule whose decision record is immutable.** A rule lives in two
  places — its operative statement in a living doc, and the immutable record that
  first set it: an ADR, or a facts record. Change the living one and the record
  still asserts the old, and **no check compares them**. You cannot rewrite the
  immutable body to match; discharge the divergence with a new record and a
  `Status`-line pointer. **The check:** grep the whole tree for the old wording,
  and reconcile each living mirror or point each immutable one.
- ❌ **A hand-mirrored count or check-set that no linter guards.** The set of discipline
  linters — and how many there are — is spelled out by hand across many living
  docs, among them [`engineering-discipline.md`](engineering-discipline.md), this file,
  [`tests/test-levels.md`](tests/test-levels.md), the two `tests/README.md` files,
  [`ci/README.md`](ci/README.md) and
  [`.githooks/README.md`](../.githooks/README.md). Add or remove a check and every one
  can go stale, and a **removed** check leaves its name behind as a linter that no longer
  exists — a `link-lint` run stays green, because it resolves a *link*, not a claim.
  **The check:** when you add or remove a discipline check, grep the whole tree for the
  check-set enumeration — the old name and each spelled count — and reconcile every
  living mirror in the same change. The quality targets in §1 are mirrored the same way
  in [`README.md`](../README.md), [`onboarding-for-engineers.md`](onboarding-for-engineers.md)
  and [`AGENTS.md`](../AGENTS.md); a change to one is a change to all.

## 3. Validation — how you check you are not fooling yourself

A result is **untrusted** until it passes the checks below, and the pass is a
recorded event, not a memory. The checks run cheap-first, so a failure stops the
expensive ones.

| # | Check | Pass condition | Cost |
|---|-------|----------------|------|
| 1 | The table tests of package `count` — lines, words, characters and validity over the cases of §2 — at every small buffer size | every case gives the counts the facts state, at every buffer size | seconds |
| 2 | The coverage of package `count`, by `go test -cover` | at least 90.0 percent of the statements (§1) | seconds |
| 3 | The end-to-end tests of the built command — flags, standard input, `-`, a directory, an unreadable file, an invalid file, the total, the exit code | every row, message and exit code as the facts state | seconds |
| 4 | The speed test on a generated file of 10 MiB | Pass band of §1 | about a minute, with the build |
| 5 | The memory test on a generated file of 1 GiB | Pass line of §1 | minutes, and 1 GiB of temporary disk |

Notes on how to read a failure: check 1 catches the boundary and Unicode traps of
§2; check 3 catches the row, total and exit-code traps that live in package
`main`. Checks 1 to 3 are cheap enough for every change and for the commit hook;
checks 4 and 5 run in CI, and on every change to package `count` or to the
reading loop. A failure in check 4 or 5 is reported as a failure, with the
measured number, never re-run until it passes.

**The automated gate is this validation layer, mechanized.** The cheap, always-on
checks — the [discipline linters](engineering-discipline.md#testing) the kit
ships (ADR, PRD and link) and their
[fixture self-tests](engineering-discipline.md#testing), the
[test levels](engineering-discipline.md#testing), lint, a security
scan, and the [commit-format](engineering-discipline.md#commit-messages)
check — run in the [`pre-commit` hook](engineering-discipline.md#git-hooks) for
fast local feedback and in [CI](engineering-discipline.md#continuous-integration-optional)
as the authority. Treat those checks as pre-registered pass/fail rules under
section 1: they predate any single result and are not edited to make a change
pass.

## 4. Mechanics

- **Frozen rules do not get edited.** Changing a guardrail after it is set means a
  new version with a written reason, the old one preserved. Legitimate reasons
  exist (a bug in the measure, a new fact from the idea owner); silent edits do not.
- **The facts hold the frozen values; this document holds the structure.** The
  numbers in §1 come from [`F-0001`](facts/F-0001-setup-answers.md) and the
  [problem statement](facts/problem-statement-brief.md). Where a number here and
  its fact disagree, the fact wins and this file is corrected in the same change.

## Sources

- [`facts/problem-statement-brief.md`](facts/problem-statement-brief.md) — the
  behaviour rules and the quality targets, in the idea owner's words.
- [`facts/F-0001-setup-answers.md`](facts/F-0001-setup-answers.md) — the answers
  that refine them: flags, order, standard input, total, invalid files,
  directories, line ends, the reference machine, the peak heap, streaming, the
  counting package, and the budget.
- The Unicode Standard, the property `White_Space` in `PropList.txt`, and RFC 3629
  for UTF-8 — the definitions rules 2 and 3 name.
