# agent-batch-reviewer

A Claude Code **skill** plugin that reviews **multiple open pull requests of a
project in parallel**. The `batch-reviewer` skill fans out one Codex-backed
`pr-reviewer` subagent per PR — each isolated in its own git worktree — and every
reviewer posts its own verdict back to the PR it owns, then the orchestrator
rolls the results up into one batch report.

It engages **only** for multi-PR batches; single-PR, working-tree-diff, and
security reviews are deferred to `/review`, `/code-review`, and
`/security-review`.

This plugin ships **skill + subagent content** — no binaries, no MCP server.

## Install

```
/plugin marketplace add Seretos/agent-marketplace
/plugin install agent-batch-reviewer@agent-marketplace
```

## Dependencies

- **agent-project-issues** (declared in `.claude-plugin/plugin.json`) — reads PRs
  and posts reviews; Claude Code installs/loads it automatically with this skill.
- **Codex** plugin (`@openai/codex`) — optional but recommended; each reviewer's
  Codex second opinion is skipped gracefully when it isn't installed.

## What the skill teaches

See `skills/batch-reviewer/SKILL.md` for the orchestrator and
`agents/pr-reviewer.md` for the per-PR reviewer subagent.
