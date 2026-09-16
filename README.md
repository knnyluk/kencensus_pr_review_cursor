# Kencensus PR Review (Cursor-native)

A consensus-based PR review skill for Cursor. Two independent reviewer
subagents — Fable 5.1 (extra-high reasoning) and GPT 5.6 Sol — review the same
diff in parallel, then cross-examine each other's findings before
anything is reported. The result is a ranked list of PR comments worth
leaving, with the weak findings filtered out by debate rather than by
hope.

This is the Cursor port of the Claude Code
[kencensus-pr-review](https://github.com/knnyluk/kencensus-pr-review)
skill. It has no external dependencies: no Codex CLI, no companion
runtime, no separate login — both reviewers are Cursor Task subagents.

## How to use

In a Cursor chat, with the target repo available and the PR branch
checked out, ask in plain words:

- "run a kencensus review of this branch against main"
- "consensus review this PR"

The skill reviews local git state (branch vs. base ref). For a GitHub
PR by number, it checks the branch out first via `gh pr checkout <n>`.

## What you get

1. **PR comments to leave** — ranked most → least important, each with
   repo-relative `file:line-range`, severity, consensus status, and the
   comment text to post. Comments that did **not** reach consensus stay
   in the list, marked `CONTESTED`, with each reviewer's position.
2. **Debunked during review** — findings one reviewer raised that were
   refuted with evidence during cross-examination, and who raised them.

Nothing is posted anywhere automatically; the output is the list.

## How it works

1. **Parallel independent reviews** — the chat agent acts purely as
   orchestrator and launches two reviewer subagents in one batch:
   Fable on `claude-fable-5-1-thinking-xhigh` and Sol on
   `gpt-5.6-sol-medium` (the highest-effort Sol slug subagents can use;
   the skill upgrades automatically if a higher one appears).
   Subagents can't see the conversation or each other, so independence
   is structural.
2. **Reconciliation** — findings are matched by file, line overlap, and
   root cause. Matches become consensus. Findings raised by only one
   reviewer are sent to the *other* reviewer for cross-examination by
   resuming its Stage-1 subagent (CONFIRM/REFUTE protocol — see
   `debate-prompt.md`), so the reviewer keeps its full diff context.
   The orchestrator independently verifies every REFUTE before
   accepting it; persistent disagreement gets one rebuttal round, then
   ships as contested.
3. **Ranking and report** — consensus and contested comments ranked by
   real-world impact; refuted findings listed as debunked with the
   evidence that killed them.

## Dependencies

- Cursor with Task-tool subagents and the two model slugs above — no
  plugins, CLIs, or extra auth.
- git (the diff under review).
- GitHub CLI (`gh`) — optional, only to check out a PR by number.

## Files

- `SKILL.md` — the agent-facing pipeline instructions (the skill itself)
- `review-prompt.md` — the Stage-1 reviewer prompt template (identical
  for both reviewers except the finding ID prefix)
- `debate-prompt.md` — the cross-examination prompt template
- `README.md` — this file

## Caveats

- Reviewers may execute the code read-only during review and debate —
  that's what makes their verdicts trustworthy, and it means you should
  only run this on code you'd be willing to execute.
- Subagent model slugs resolve from a fixed allow-list; changing the
  chat's default model or reasoning effort does not affect which slugs
  subagents can use.
