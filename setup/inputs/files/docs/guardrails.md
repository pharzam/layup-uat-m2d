# Guardrails — the known pitfalls, and the numbers you must not move

This document is the target of gate step 2, "Honor the guardrails", in
[`engineering-discipline.md`](engineering-discipline.md). It holds the pitfalls a
task author on `tally` must take into account **before** writing code, so the team
does not re-derive a known trap every time.

It merges two kinds of guardrail that many projects keep in separate files —
**decision gates** (pre-registered pass/fail rules) and **validation** (how you
check you are not fooling yourself). Both are read before work starts.

## In plain terms

`tally` looks too small to get wrong, and that is the trap: a counter that agrees
with `wc` on the easy file and disagrees with the problem statement on the edges —
a file without a final line feed, a carriage return, a no-break space, a character
of two bytes cut by a buffer — passes every casual check and is wrong. The defense
is a test for each edge, written before the code, and quality targets whose
numbers and methods are frozen here before anything is measured.

## 1. Pre-registered decisions — or the goalposts move

A decision rule chosen **after** seeing the result is a fitted parameter, not a
rule. Write the pass/fail numbers first, somewhere they cannot be quietly edited.

- **What must be pre-registered:** the three quality targets of the problem
  statement, with the method of measurement that the idea owner fixed in the
  answers of [`F-0001`](facts/F-0001-setup-answers.md):

  | Target | Pass | Method, frozen |
  |--------|------|----------------|
  | Time for a file of 10 MiB | At most 2 seconds | A generated file of 10 MiB of ASCII text in lines of about 80 characters, already in the cache of the operating system, on a computer with 4 cores and a solid-state disk; the elapsed time of the whole process, from its start to its exit (answer `Q-008`). |
  | Memory for a file of 1 GiB | At most 64 MiB of heap | The peak of `HeapAlloc` from `runtime.ReadMemStats`, sampled every 10 milliseconds during the count — never the heap at the end (answer `Q-009`). |
  | Coverage of the package `count` | At least 90 percent of the statements | The output of `go test -cover` for the package `count` (answer `Q-011` names the package). |

- **Where the numbers freeze:** here, and in the requirements of the PRD that
  cite [`F-0001`](facts/F-0001-setup-answers.md) and the
  [problem statement](facts/problem-statement-brief.md). A change of a number or
  a method is a new fact from the idea owner, never an edit to make a run pass.
- **The bands, not a single line:** the time measure is noisy, so it has three
  bands. **Pass**: at most 2 seconds. **Investigate**: above 2 seconds and at most
  2.5 seconds — one rerun on the same machine with the file in the cache, with its
  scope fixed in advance; a second result above 2 seconds counts as Fail.
  **Fail**: above 2.5 seconds. The heap and the coverage targets are not noisy and
  have a single line.

## 2. Known pitfalls — the traps specific to this domain

The failure modes that hurt a counter of text. For each: the trap, why it is
silent, and the check that catches it.

- ❌ **An empty input, or an input that ends with a line feed, gets one line too
  many.** The rule "the last line counts when the file does not end with a line
  feed" invites a counter that adds one line at the end of every input. It is
  silent because the common file, which ends with a line feed, may still look
  right if the code counts line feeds and adds one only in some branch. **The
  check:** unit cases for `""` (0 lines), `"a"` (1), `"a\n"` (1) and `"a\nb"` (2).
- ❌ **Counting bytes where the rule says characters.** In Go, `len` of a string
  and the size of a read are bytes. It is silent because every ASCII test file
  has as many bytes as characters. **The check:** a test with code points of two,
  three and four bytes, for example `é`, `€` and an emoji.
- ❌ **A code point cut by the boundary of two buffers.** `tally` reads in fixed
  buffers of 64 KiB (answer `Q-010`), and a code point of two to four bytes can
  start in one buffer and end in the next. A decoder per buffer then sees two
  invalid fragments: it counts too many characters, or reports a valid file as
  not valid UTF-8. It is silent because a small test file fits in one buffer.
  **The check:** a test that places a multi-byte code point across the boundary,
  at each possible cut, with a small buffer size injected for the test and with
  the real size of 64 KiB.
- ❌ **ASCII white space where the rule says `White_Space`.** A word splitter on
  space, tab and line feed only joins two words that a no-break space (U+00A0) or
  an ideographic space (U+3000) separates. It is silent on English test text.
  **The check:** a test with each of those separators.
- ❌ **Treating a carriage return as a line end, or dropping it.** A reader that
  splits on `\r\n` or trims `\r` gives the wrong character count for a file
  written on Windows (answer `Q-007`). **The check:** `"a\r\n"` is one line and
  three characters.
- ❌ **Reading a whole file into memory.** `os.ReadFile` or `io.ReadAll` passes
  every test on a small file and breaks the memory target on a file of 1 GiB. It
  is silent until the large test runs. **The check:** the memory test of a file
  of 1 GiB, which samples the peak heap; and a review that rejects any read of a
  whole file.
- ❌ **A heap measured at the end.** A garbage collection after the count makes the
  end value small, so a test that reads the heap once at the end passes a program
  that peaked far above 64 MiB. **The check:** sample every 10 milliseconds during
  the count and assert on the largest sample (answer `Q-009`).
- ❌ **An error that stops the other files, or a failed file in the total.** An
  early `return` on the first unreadable file, or a sum taken before the error is
  known, breaks rule 5 or the answer `Q-004`. It is silent with one file. **The
  check:** an integration test with a good file, a missing file, a directory and
  a file that is not valid UTF-8, asserting the rows, the total, the messages on
  standard error and the exit code 1.
- ❌ **Treating `-` as standard input.** Many Unix tools do, and `tally` does not
  (answer `Q-003`). **The check:** a test with a file named `-`.
- ❌ **Printing the counts in the order of the flags.** `tally -w -l` must print
  lines before words (answer `Q-002`). **The check:** a test for each order of two
  and of three flags.

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

These traps are not domain-specific: they hurt every project's test suite, so the
kit shipped them filled. The traps of `tally` itself are above.

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
  the test timeout that [`tests/test-levels.md`](tests/test-levels.md) defines —
  for `tally`, the `-timeout` of `go test` — see [`tests/scaling-checklist.md`](tests/scaling-checklist.md).

### Reference-sweep pitfalls (kit-wide — keep these)

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
  structure only the deleted record holds — wrong on the adopter's tree, and sharing
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
  kit once said `three`, `four` and `five` at once, and named an `agent-entry` linter that
  had been cut. **The check:** when you add or remove a discipline check, grep the whole
  tree for the check-set enumeration — the old name and each spelled count — and reconcile
  every living mirror in the same change; the immutable ADR and archived decision copies
  stay as history.

## 3. Validation — how you check you are not fooling yourself

A result is **untrusted** until it passes the checks below, and the pass is a
recorded event, not a memory. Order the checks cheap-first, so a failure stops the
expensive ones.

| # | Check | Pass condition | Cost |
|---|-------|----------------|------|
| 1 | `gofmt -l .` and `go vet ./...` | No file listed, no finding | seconds |
| 2 | `go test -short ./...`, with `-cover` for the package `count` | Every test passes; coverage of the package `count` at least 90 percent | seconds |
| 3 | The tests of a file of 10 MiB and of 1 GiB, and the end-to-end tests of the built binary | The time and heap targets of §1 met; every end-to-end test passes | minutes |

Notes on how to read a failure: check 1 catches style and the plain mistakes the
Go toolchain can find; check 2 catches every edge of §2 that a unit or integration
test covers, and is cheap enough for the commit hook; check 3 catches the memory
and time traps and the wiring of the command, and runs in CI. A failure of
check 3 on a shared CI machine that is not the reference machine of §1 is first
rerun on a reference machine before it counts as Fail; a pass on a faster machine
is not evidence that the target holds on the reference.

**The automated gate is this validation layer, mechanized.** The cheap, always-on
checks — the [discipline linters](engineering-discipline.md#testing) (ADR, PRD
and link) and their
[fixture self-tests](engineering-discipline.md#testing), the
[test levels](engineering-discipline.md#testing), lint, a security
scan, and the [commit-format](engineering-discipline.md#commit-messages)
check — run in the [`pre-commit` hook](engineering-discipline.md#git-hooks) for
fast local feedback and in [CI](engineering-discipline.md#continuous-integration-optional)
as the authority. Treat those checks as pre-registered pass/fail rules under
section 1: they predate any single result and are not edited to make a change
pass. Checks 1 and 2 of the table above go into both layers; check 3 goes into CI.

## 4. Mechanics

- **Frozen rules do not get edited.** Changing a guardrail after it is set means a
  new version with a written reason, the old one preserved. Legitimate reasons
  exist (a bug in the measure); silent edits do not.
- **This document holds the structure; the PRD of `tally` holds the frozen
  values.** The numbers of §1 come from the problem statement and
  [`F-0001`](facts/F-0001-setup-answers.md), and the requirements that cite them
  are the record of record. Measured values go to the evidence store that
  [`engineering-discipline.md`](engineering-discipline.md#honesty-and-evidence)
  names, never into this file as the primary copy.

## Sources

- The [problem statement](facts/problem-statement-brief.md) of the idea owner:
  the rules of behaviour and the three quality targets.
- The answers of the idea owner in [`F-0001`](facts/F-0001-setup-answers.md):
  the method of each measurement, the buffers of 64 KiB and the package `count`.
- The Unicode Standard, Annex 44 (the Unicode Character Database), for the
  property `White_Space`; and the Go documentation of the packages `unicode`,
  `unicode/utf8`, `runtime` and `testing`.
