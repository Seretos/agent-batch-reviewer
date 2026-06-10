---
name: batch-reviewer
description: Reviews MULTIPLE open pull requests of a project in parallel — fans out one Codex-backed reviewer subagent per PR and each posts its own review comment. Use ONLY for batch PR review across several PRs at once (e.g. "review all open PRs in acme-api", "review PRs #12, #15 and #18 in parallel", "alle offenen PRs reviewen"). Do NOT use for a single PR or a single diff — that is the job of /review, /code-review, or /security-review; this skill explicitly declines when only one PR is in play.
---

# batch-reviewer — parallel PR-review orchestrator

You dispatch **code review across several open pull requests at once**. You fan
out one `pr-reviewer` subagent per PR — each in its own isolated worktree, each
taking a Codex second opinion, each posting **its own** review comment to the PR
it owns — then you collect their verdicts into one batch report. You review
nothing yourself; you sequence the fan-out and aggregate the results.

## When this skill applies (and when it must not)

This skill exists for **one specific shape of request: reviewing a batch of
several PRs in parallel.** The parallel fan-out is the entire point — it is what
distinguishes this skill from every single-target review tool.

**Engage only when ALL of these hold:**
- The target is **pull requests** (not the working-tree diff, not a ticket).
- There are **two or more** PRs to review **in one go** — "all open PRs", a
  named list of several, or "every PR with label X".
- The user wants them reviewed **concurrently / as a batch**.

**Do NOT engage — defer to the right single-target tool — when:**
- The request is about **one** PR → that's `/review` (or `/code-review` for a
  branch diff). A single PR does not need a fleet; hand it off.
- The request is about the **current working-tree changes / current branch
  diff** → `/code-review` or `/security-review`.
- The request is a **security** review specifically → `/security-review`.
- The request is **ticket** work, not PR review → the autonomous-developer
  `orchestrate-tickets` / `process-ticket` skills.

If you reach Phase A and find **0 or 1** open PR in the target set, **stop and
say so**: there is no batch to parallelise, and a single PR should go through
`/review`. Don't spin up the machinery for one PR.

## Inputs

- A **project id** (e.g. `acme-api`). If missing or unclear, resolve it via
  `search_projects` / `list_projects` and confirm with the user before anything
  else. Thread it into every project-issues call and every subagent prompt —
  never hardcode a project.
- An optional **PR selection**: nothing → all open PRs; an explicit list of
  numbers → just those; a label/filter → the matching open PRs.

## Preconditions

1. **agent-project-issues MCP must be loaded.** This skill drives it for PR
   reads and review posting. If its tools aren't available (fresh sessions don't
   auto-load plugin MCPs — anthropics/claude-code#61866), **STOP** and tell the
   user to `/reload-plugins`, then re-invoke.
2. **Codex is best-effort, not required.** The `pr-reviewer` subagents use the
   Codex plugin for a second opinion and degrade silently if it's missing. You
   do not need to verify Codex yourself; just note in the final report whether
   reviewers reported Codex ran.

## Phase A — enumerate the target PRs

Call `list_prs(project_id, status="open")` (add `labels=[…]` if the user gave a
filter). If the user named specific PR numbers, intersect with that list and
confirm each exists and is open.

- **0 PRs** → report "no open PRs to review" and stop.
- **1 PR** → do **not** fan out. Tell the user a single PR is better served by
  `/review` (offer to hand it off) and stop — this skill is for batches.
- **2+ PRs** → proceed.

Capture for each target PR: number, title, head→base branches, author.

## Phase B — confirm the batch

Spawning N parallel reviewers (each fetching a branch, running Codex) is heavy,
so confirm before launching. Present the list of PRs that will be reviewed (number
· title · head→base) via **AskUserQuestion** and get a go-ahead, letting the user
drop any they don't want reviewed. Keep it light, but always confirm the fan-out.

## Phase C — fan out one reviewer per PR

Spawn the reviewers **concurrently** — issue all the `Agent` calls **in a single
message** so they run in parallel, each isolated in its own worktree:

```
Agent(
  subagent_type="pr-reviewer",
  isolation="worktree",
  description="review PR #<n>",
  prompt="Review pull request #<n> in project '<project_id>'. project_id=<project_id>, pr_id=<n>."
)
```

- **One subagent per PR.** Pass only `project_id` and the single `pr_id`; each
  reviewer fetches the PR's own diff, reviews it, takes its Codex pass, and posts
  its **own** review comment on that PR (see the `pr-reviewer` agent).
- **`isolation="worktree"` is mandatory.** Each reviewer checks out a different
  PR branch as its HEAD; without per-agent worktrees they would collide on the
  repo's working tree and Codex's `--scope branch` review would read the wrong
  diff. The temporary worktrees auto-clean when the agents finish.
- **Don't post comments yourself.** Each reviewer owns the write to its PR. Your
  job is to launch them and collect what they return.
- A reviewer may return `null` (skipped or died after retries) or a fetch-failure
  outcome — keep going; record it in the report rather than aborting the batch.

## Phase D — aggregate report

Once all reviewers return, print **one** batch summary table:

`PR · title · verdict (APPROVE / CHANGES_REQUESTED / COMMENT) · blocking · nits · codex (ran/skipped/unavailable)`

Then below it:
- A short headline per PR (the reviewer's one-line lead finding).
- Any PRs that **failed to review** (fetch failure / null return), called out
  explicitly so none silently vanish.
- A one-line tally: how many approved, how many need changes.

Each reviewer has already posted its verdict to its PR, so your report is the
human's at-a-glance roll-up — you do not post anything to the PRs yourself.

## Hard rules

- **Batch-only.** Engage solely for parallel review of **2+ PRs**. One PR, a
  working-tree diff, a security pass, or ticket work → defer to `/review`,
  `/code-review`, `/security-review`, or the autonomous-developer skills. At 0–1
  target PRs, stop and hand off.
- **Delegate every review.** You never read a PR diff, run Codex, or post a
  review comment. The `pr-reviewer` subagents do all of it; you sequence and
  aggregate. Your tools: `list_prs`/`get_pr` (enumeration only),
  `Agent`, `AskUserQuestion`.
- **Project id is a parameter.** Thread the supplied `project_id` into every call
  and every subagent prompt — never hardcode a project.
- **Fan out in parallel, isolated.** All `Agent` calls in one message;
  `isolation="worktree"` on every reviewer. One PR per subagent.
- **Confirm before launching.** N parallel Codex-backed reviews are expensive and
  outward-facing (they post to PRs) — get a go-ahead in Phase B.
- **One reviewer's failure never aborts the batch.** Record it and report it.
