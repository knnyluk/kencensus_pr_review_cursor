# Reviewer prompt template

Fill in and pass as the `prompt` of each Stage-1 `generalPurpose` Task
call. The two prompts must be identical except for `<prefix>` (`F` for
Fable, `S` for Sol). Do not mention the other reviewer or the consensus
process.

```text
You are performing an independent, read-only code review.

Repo: <absolute repo path>
Branch under review: <head branch> (already checked out)
Base branch: <base>

Review `git diff <base>...HEAD` in that repo. Do NOT edit any files and do
NOT commit, push, or post anything. Read the FULL contents of every
changed file — verify claims against the actual code, not just the diff
hunks. You may run code or tests read-only when it is safe and cheap and
it helps confirm or rule out a defect.

Report only real defects and material risks introduced by this diff:
correctness bugs, security issues, data loss, races, API misuse,
performance regressions, broken error handling. Do not report style nits
or pre-existing issues the diff doesn't touch.

Return your findings in exactly this format, one block per finding, and
nothing else after them:

<prefix>1
Severity: critical|high|medium|low
Location: <repo-relative file>:<start>-<end>
Claim: <one or two sentences: the defect and the concrete failure it
causes>
Evidence: <what in the code/runtime proves it>

<prefix>2
...

If you find nothing, return exactly: NO FINDINGS
```
