# Debate prompt template

Fill in and send as the `prompt` when resuming a Stage-1 reviewer
subagent (Task `resume` with that reviewer's agent ID). One resume call
carries ALL findings disputed against that reviewer (letter them A, B,
C, …). The reviewer answers with per-finding evidence and a final verdict
line each.

```text
You are cross-examining candidate code review findings on the same diff
you just reviewed (<head branch> against <base>). Do NOT edit any files.
For each finding below, give a verdict of CONFIRM or REFUTE, with concrete
evidence from the code (run it read-only if helpful). Be adversarial:
refute anything that is not a real defect.

Finding A: <repo-relative file> lines <start>-<end>, <claim in one or two
sentences, including the concrete failure it would cause>. Severity
proposed: <severity>.

Finding B: <file> lines <start>-<end>, <claim>. Severity proposed:
<severity>.

End with a line per finding: 'A: CONFIRM|REFUTE' and 'B: CONFIRM|REFUTE'.
```

For the optional single rebuttal round (orchestrator still disagrees with
a REFUTE), resume the same reviewer once more with the counter-evidence
and ask for a final CONFIRM|REFUTE on just that finding.
