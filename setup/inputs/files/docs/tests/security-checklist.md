# Security checklist

The minimum set of security-weakness checks a change runs, or a reviewer checks
for by hand until the automation is wired. It is the checklist behind the
security track described in
[`test-levels.md`](test-levels.md#security-tests-sit-alongside-the-ladder) — a
parallel track to the test ladder, not a rung on it.

> **How to adapt this file.** Fill `‹security scanner›` and
> `‹security test command›` with your stack's real tool and command everywhere
> they appear here — [`test-levels.md`](test-levels.md) is the one place they are
> defined, and this file inherits them. Add a project-specific check to
> [the table](#the-minimum-checks) or [Add your own](#add-your-own) for anything
> this minimum list does not cover. Delete this note once your checks are in.

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
`‹…›` steps are filled for your stack.

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

## For `tally`

- **Dependencies.** The product uses the Go standard library only, so the
  dependency scan has one subject: the Go toolchain version the module names. A
  new module requirement is itself a finding, since it needs an [ADR](../adr/).
- **Untrusted input.** `tally` reads any file a person names, of any size and any
  content. A file must never make it read without bound into memory (the 64 KiB
  buffers), and an error message prints the name as given and the reason, never
  the content of the file.
- **Generated test files.** The 1 GiB test file is written to a temporary
  directory and removed; it is never committed.

## Add your own

This is a minimum, not a ceiling. Add a row for any other weakness class your
project needs checked — for example a licence scan, a container-image scan, or
an infrastructure-as-code scan — as its own `‹…›` check, tool, and pass
condition.
