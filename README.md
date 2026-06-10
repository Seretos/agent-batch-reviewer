# agent-batch-reviewer

A Claude Code **skill** plugin. Skill that orchestrates parallel subagents to review a batch of open PRs or tickets across one or more projects, fanning out the review work and collecting structured findings.

This plugin ships **only the skill content** — no binaries, no MCP server.

## Install

```
/plugin marketplace add Seretos/agent-marketplace
/plugin install agent-batch-reviewer@agent-marketplace
```

If the skill teaches Claude how to use a specific MCP, declare that MCP as a dependency in `.claude-plugin/plugin.json` (`dependencies` array). Claude Code will install/load it automatically.

## What the skill teaches

See `skills/batch-reviewer/SKILL.md` for the full content.
