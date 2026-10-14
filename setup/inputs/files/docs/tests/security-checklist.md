# Security checklist

The minimum set of security-weakness checks a change runs, or a reviewer checks
for by hand until the automation is wired. It is the checklist behind the
security track described in
[`test-levels.md`](test-levels.md#security-tests-sit-alongside-the-ladder) — a
parallel track to the test ladder, not a rung on it.

> **Security in tally.** `‹security scanner›` and
> `‹security test command›` are defined in one place,
> [`test-levels.md`](test-levels.md), and this file inherits them. They stay
> placeholders until a task wires them. `tally` uses the Go standard library
> only, so the dependency scan has no third-party module to read; it still runs,
> because a new `require` line in `go.mod` would break the stack rule of the
> problem statement and must be caught. The static analysis starts with
> `go vet`, which the Go toolchain already holds.

## In plain terms

> A committed credential, a known-vulnerable dependency, or an insecure code
> pattern can slip into a change without anyone noticing. This checklist runs a
> small set of automated scans — a fast subset on every commit, the full set
> before a merge — so a machine catches the weakness before it reaches
> production, rather than a person catching it by luck.

## The minimum checks

| Check | What it catches | Where it runs | Pass condition |
|-------|------------------|----------------|-----------------|
| Secret scan | committed credentials, keys, or tokens | hook (staged changes) + CI (full history) | no secret found |
| Dependency scan | known-vulnerable third-party dependencies | CI (and hook if fast enough) | no known-vulnerable dependency at or above the agreed severity |
| Static analysis | insecure code patterns, found without running the code | CI (and a hook subset) | no finding at or above the agreed severity |

Run them cheap-first, in the order above: a failing secret scan stops the
slower checks from running at all.

## How it is wired

Like the [ADR and PRD linters](../engineering-discipline.md#testing), the
security checks are wired into two layers, cheap-first:

- The fast subset — a secret scan on staged changes, and a static-analysis
  subset where it is fast enough — runs via `‹security test command›` in the
  [`pre-commit` hook](../../.githooks/pre-commit), before a commit is recorded.
- The full set — all three checks, run in full — runs via `‹security test command›`
  in [CI](../ci/), as the authority.

`‹security scanner›` names the tool both layers drive. Both are inert until the
`‹…›` steps are filled with the commands of `tally`.

## A pre-registered bar

The severity that fails a build must be written down **before** a scan produces
a result — see
[Pre-registered decisions](../guardrails.md#1-pre-registered-decisions--or-the-goalposts-move)
in the guardrails.

- [ ] Agree the failing severity for each check in
      [the table above](#the-minimum-checks) — the lowest severity a finding
      can carry and still pass.
- [ ] Freeze that severity somewhere it cannot be quietly edited (mirror it in
      [`guardrails.md`](../guardrails.md) once it is real).
- [ ] Treat a scan finding at or above the frozen severity as a failed change,
      not a judgement call made after the fact.

## Add your own

`tally` reads files that a person names, so one more check belongs to it: a test
that a file name the command cannot open — a directory, a missing file, a file
without read permission — gives a message on standard error and the exit code 1,
and never stops the count of the other files. That check is a product test at
the [integration level](test-levels.md#2-integration-tests), listed here so a
security reviewer knows it exists.

This is a minimum, not a ceiling. Add a row for any other weakness class your
project needs checked — for example a licence scan, a container-image scan, or
an infrastructure-as-code scan — as its own `‹…›` check, tool, and pass
condition.
