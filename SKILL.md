---
name: kencensus-pr-review
description: Consensus PR review by two independent reviewer subagents — Fable 5.1 (extra-high reasoning) and GPT 5.6 Sol each review the diff, cross-examine each other's findings, and output a ranked list of PR comments (consensus and contested) plus a list of debunked comments. Use when asked to run kencensus, kencensus review, consensus review, or to review a PR/branch with two models.
---

# Kencensus PR Review (Cursor-native)

Two independent reviewer subagents review the same diff, then reconcile
through cross-examination. The deliverable is a ranked list of PR comments
to leave (most important first, with file + line for each), contested
comments included and marked, followed by the comments that were debunked
during reconciliation.

You are the **orchestrator**, not a reviewer. Both reviews come from
subagents so the models are exactly the ones requested:

| Reviewer | Task tool model slug | Finding ID prefix |
|----------|----------------------|-------------------|
| Fable    | `claude-fable-5-1-thinking-xhigh` | F1, F2, … |
| Sol      | `gpt-5.6-sol-medium` (see note) | S1, S2, … |

**Sol slug note:** the intent is the highest reasoning effort available for
GPT 5.6 Sol. If the Task tool's allowed-model list contains a
higher-effort Sol slug (e.g. `-high` or `-xhigh`), use that instead of
`-medium`. If either reviewer's slug is unavailable, stop and tell the
user — never silently substitute a different model family.

## Prerequisites

- A git repo with the PR branch checked out. If given a GitHub PR number,
  check the branch out first (`gh pr checkout <n>`), then proceed — the
  pipeline only needs local git state.
- `<base>` is the branch the PR merges into (usually `main` or `master`).
  Sanity-check before launching: `git diff --shortstat <base>...HEAD` must
  show changes. If empty, the base ref or checkout is wrong — fix that
  before spending reviewer tokens.

## Pipeline

### Stage 1 — two independent reviews in parallel

Launch BOTH reviewer subagents in a single message (two `generalPurpose`
Task calls in one batch) so they run concurrently. Subagents cannot see
this conversation or each other, which makes independence structural —
but only if each prompt is self-contained and neither prompt mentions the
other reviewer or leaks findings.

Build each prompt from the template in
[review-prompt.md](review-prompt.md), filling in the repo path, base
branch, head branch, and that reviewer's finding ID prefix. The prompts
must be identical except for the ID prefix.

Each reviewer returns findings as: ID, severity
(critical/high/medium/low), `file:line-range` (repo-relative), the claim,
and the evidence. Reviewers verify against the full changed files, not
just diff hunks, and may run code read-only when safe and cheap. They
must never edit files.

Keep each subagent's ID — Stage 2 resumes them.

### Stage 2 — reconcile

Match the two finding sets by file + overlapping lines + same root cause.

- **Both raised it** → consensus. Keep the clearer wording and the higher
  of the two severities.
- **Raised by only one reviewer** → cross-examine via the OTHER reviewer.
  Resume that reviewer's Stage-1 subagent (Task `resume` with its agent
  ID) with the debate prompt from [debate-prompt.md](debate-prompt.md),
  carrying ALL findings disputed against that reviewer in ONE resume call
  (letter them A, B, C, …). Restate each finding fully (file, lines,
  claim, severity) — do not assume the reviewer remembers anything.

  The reviewer answers `A: CONFIRM|REFUTE` per finding with evidence.
  - CONFIRM → consensus (mark it "consensus after debate").
  - REFUTE → verify the refutation evidence yourself against the actual
    code (run it if safe). If the refutation holds → debunked. If you
    still believe the defect is real → contested — one rebuttal round
    max (a second resume presenting your counter-evidence), then record
    both positions and move on.

### Stage 3 — output

Produce exactly this structure (all paths repo-relative):

```markdown
## PR comments to leave (most important first)

1. **`cart.js:3`** [high · consensus] — <comment text to post, imperative,
   states the defect and the fix>
2. **`auth.js:7-8`** [high · consensus] — …
3. **`cart.js:10-12`** [low · consensus after debate] — …
4. **`foo.js:42`** [medium · CONTESTED — Fable: real, Sol: refutes] —
   <comment text, plus one line per side's position>

## Debunked during review

- **`cart.js:17-18`** — "sort() mutates the caller's array" — refuted:
  `.filter()` on line 16 creates a fresh array; runtime check confirmed
  the caller's array is unchanged. (Raised by: Fable)
```

Rank by real-world impact, not by who raised it. Contested comments stay
in the ranked list, marked, with both positions in one line each. Every
debunked item names who raised it (Fable or Sol) and the evidence that
killed it. If a section is empty, say so explicitly ("No contested
comments." / "Nothing was debunked.").

Do NOT post the comments anywhere — the deliverable is the list.

## Gotchas

- Do not review the diff yourself and do not editorialize findings beyond
  reconciliation — your Stage-2 verification of REFUTE evidence is the
  only place your own code reading changes an outcome.
- Both Stage-1 Task calls must be in the same message; sequential launches
  waste wall-clock time and risk contaminating the second prompt with the
  first's results.
- Resume the existing Stage-1 subagents for debates; a fresh subagent
  loses the reviewer's context and wastes a full re-read of the diff.
  Restate disputed findings fully anyway — cheap insurance.
- Reviewers may run the code read-only during review and debate; that
  makes CONFIRM/REFUTE verdicts trustworthy, and it means this skill
  should only run on code the user would be willing to execute.
- If a reviewer returns prose instead of the findings format, extract the
  findings yourself rather than re-running the review.
