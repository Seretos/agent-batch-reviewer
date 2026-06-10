# agent-batch-reviewer

Reviews **several open pull requests of a project in parallel**. The
`batch-reviewer` skill fans out one reviewer subagent per PR, each in its own
isolated git worktree, takes a Codex second opinion on every PR, and has each
reviewer post its own verdict back to the PR it owns — then rolls the results up
into one batch report.

## Key features

- **Parallel fan-out, one reviewer per PR.** Spawns a `pr-reviewer` subagent for
  every target PR concurrently, each isolated in its own worktree so their
  branch checkouts never collide.
- **Codex-backed reviews.** Each reviewer runs an additional Codex correctness
  pass (`--scope branch`) on top of its own analysis and folds the blocking
  findings into its verdict — degrading silently when Codex isn't installed.
- **Posts a verdict per PR.** Every reviewer submits a single PR review
  (approve / request changes / comment) with concrete, file-and-line findings;
  the orchestrator aggregates a one-look batch summary.
- **Batch-only by design.** Engages strictly for reviewing 2+ PRs at once;
  single-PR, working-tree-diff, and security reviews are deferred to `/review`,
  `/code-review`, and `/security-review`.

## Requirements

- **agent-project-issues** plugin (declared dependency) — for reading PRs and
  posting reviews.
- **Codex** plugin (`@openai/codex`) — optional but recommended; the per-PR
  Codex second opinion is skipped gracefully when it isn't present.
