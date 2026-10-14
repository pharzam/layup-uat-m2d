# Forge templates (optional, inert)

Issue and pull/merge-request templates for `tally` that embody the [issue-first
workflow](../issue-workflow.md) — single-goal scope (R11), the duplicate check
(R2), solution selection (R3), and the action/why/tradeoff comment (R7).

**These are inert here on purpose.** An issue or PR template changes the live forge
interface the moment it lands, so the Armature kit that `tally` started from does
not activate one — it stays forge-free. The `tally` repository is hosted on
GitHub, where its issues already carry the questions and answers of the idea
owner, so the GitHub set is the one to copy; activating it is a change of its
own, with its issue, and fills each `‹…›` marker at the same time.

## Activate

**GitHub:**

```bash
mkdir -p .github/ISSUE_TEMPLATE
cp docs/templates/github/ISSUE_TEMPLATE/task.md   .github/ISSUE_TEMPLATE/
cp docs/templates/github/PULL_REQUEST_TEMPLATE.md .github/
```

**GitLab:**

```bash
mkdir -p .gitlab/issue_templates .gitlab/merge_request_templates
cp docs/templates/gitlab/issue_templates/Task.md            .gitlab/issue_templates/
cp docs/templates/gitlab/merge_request_templates/Default.md .gitlab/merge_request_templates/
```

The GitHub issue template carries a small metadata header (`name`, `about`,
`labels`) the forge reads; the GitLab files are plain Markdown. The GitLab set
stays here unused unless the project moves to GitLab.

> Not to be confused with the per-record templates — [`adr/template.md`](../adr/template.md),
> [`facts/template.md`](../facts/template.md), [`prd/template.md`](../prd/template.md) —
> which are copied *inside* their own directories, not into a forge path.
