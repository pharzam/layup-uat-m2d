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
- `questions.tsv`, `intake/`: the questions of Intake and their answers.
- `setup/record.tsv`: each value of the setup, with its source, and one done
  row per step.
- `setup/verify.tsv`: the table of `layup setup verify` before the records
  commit of the setup.
- `setup/tree.tsv`: the commits of the setup that `layup setup verify` read,
  which the Operator then pushes.
- `setup/inputs/files/`: the texts of the setup that a role session wrote.
- `setup/rulesets.txt`: the Operator's apply of the two rulesets, with their
  bypass lists.
- `setup/forge-check.tsv`: the rules read back and the probes of the last start.
- `rule-paths.tsv`: the rule-path register: the paths whose change is a change
  of the rules.
