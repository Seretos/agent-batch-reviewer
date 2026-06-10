<!-- AGENTS.md authoring rule (keep this comment in the template; delete it in a real plugin):
     Document ONLY what an agent cannot derive by reading the code and the file tree.
     - DO capture: cross-file / cross-repo contracts, non-obvious conventions, gotchas and
       their "why", external requirements (secrets, services), and deliberate design choices.
     - DON'T restate: the directory layout, what a workflow YAML does step-by-step, or how a
       build script works line-by-line — an agent reads those directly. If a sentence only
       narrates a file the reader already has in front of them, cut it.
     A lean AGENTS.md the agent trusts beats an exhaustive one it has to re-verify. -->

# agent-batch-reviewer

Skill + subagent plugin — no binary, no MCP server. Ships `skills/batch-reviewer/SKILL.md` (the orchestrator Claude Code loads when the skill's `description` matches intent) and `agents/pr-reviewer.md` (the per-PR reviewer subagent the skill fans out, one per PR).

## Contracts an agent won't infer from the tree

- **The skill is batch-only by design, and that scoping is load-bearing.** `batch-reviewer` must engage *only* for parallel review of 2+ PRs; single-PR / working-tree-diff / security / ticket requests are deliberately deferred to `/review`, `/code-review`, `/security-review`, and the autonomous-developer skills. The negative scoping in the skill `description` is what keeps it from stealing those single-target intents — don't loosen it. At 0–1 target PRs the skill stops rather than spinning up a fleet.
- **Reviewers run in per-agent isolated worktrees, and that isolation is required, not an optimisation.** Each `pr-reviewer` checks out a *different* PR's head branch as its HEAD; the skill spawns them with `isolation="worktree"` so their checkouts don't collide on the repo working tree and so Codex's `--scope branch` reads each PR's own diff. Spawning them without isolation corrupts every review.
- **Codex is used by the subagents but is a soft dependency.** Each reviewer locates the newest cached `codex-companion.mjs` by *numeric* version, runs `review --wait --scope branch --base origin/<base>` (never `--write`), and folds blocking findings into its verdict — but degrades silently when Codex is absent or not ready. Codex is therefore documented as recommended, **not** declared in `dependencies` (it's a separate marketplace plugin, `@openai/codex`); only `agent-project-issues` is a hard dependency.
- **Each reviewer posts its own PR review; the orchestrator posts nothing.** A `pr-reviewer` submits exactly one `submit_pr_review` (approve / request_changes / comment) to the PR it owns and returns a compact verdict block; the skill only aggregates a report. Independent PRs mean no write-collision, so the write is pushed to the leaf rather than centralised (the inverse of process-ticket, where the orchestrator owns the single PR write).
- **`agents/` is a release artifact.** `release.yml`'s stage step copies `agents/` into the staging tree alongside `skills/`; without it the released plugin ships a skill that fans out an undefined subagent type. Keep the agents-copy line whenever editing the staging step.
- **Release is orphan-branch + marketplace dispatch.** `release.yml` (manual: Actions → release → `version=X.Y.Z`) stamps the version, then force-pushes an orphan `release` branch holding only install-ready files and POSTs a dispatch (`category: skill`) to `Seretos/agent-marketplace`. `main` and `release` share no history. Clients install at the tag `agent-batch-reviewer--vX.Y.Z`.
- **Required secret:** `MARKETPLACE_DISPATCH_TOKEN` — fine-grained PAT, `Contents: RW` + `Pull requests: RW` on `Seretos/agent-marketplace` only.
- **`assets/icon.png` is a release artifact, not just a repo file.** The dispatch payload sends a `raw.githubusercontent.com/${repo}/${TAG}/assets/icon.png` URL to the marketplace, so the file must live on the orphan `release` branch at the tagged commit — `release.yml`'s stage step copies `assets/` into the staging tree for exactly that reason. Ship `assets/icon.png` from day one or the marketplace listing has no image.
- **`description.md` is a release artifact, not just a repo file.** The dispatch payload sends a `raw.githubusercontent.com/${repo}/${TAG}/description.md` URL in the `description_url` field, so the file must live on the orphan `release` branch at the tagged commit — `release.yml` copies it into the staging tree alongside `assets/`. Fill in its Key Features before cutting v0.0.1.
- **Depending on an MCP plugin:** declare it under `dependencies` in `.claude-plugin/plugin.json` (`{ "name": "agent-<mcp>", "version": ">=0.0.1 <1.0.0" }`); Claude Code installs/loads it automatically with this skill.
