# The records of this repository

This branch, `layup-records`, holds the records that LAYUP keeps for this
repository. It is an orphan branch: it shares no commit with the default
branch.

Only `layup run` writes this branch from Start on. Its first commit holds the
records of the Start.

Each file is a table of tab-separated values with a header row, or Markdown,
so a person reads it with no tool. A plain `git clone` carries the branch as
`origin/layup-records`.

- `start/start.tsv`: each value of the Start, with its source.
- `start/problem-statement.md`, `start/vision.md`: the two briefs, byte for byte.
- `approvers.tsv`: each account whose comment can decide, by its numeric ID and
  role.
- `lease.tsv`: the run that holds this target.
