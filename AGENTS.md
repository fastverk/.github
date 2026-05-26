# Agent dispatch in fastverk

Coding-agent work in fastverk runs via a label-driven GitHub
Actions workflow. Botnoc's `AgentService.Dispatch` adds an
`agent/start` label to an issue; the per-repo
`agent-dispatch.yml` workflow picks it up, transitions labels
through the lifecycle, and runs the chosen coding-agent action.

Formal spec: `fastverk/botnoc/lean/Botnoc/AgentLifecycle.lean`.
The label state machine is the wire-level realization of the
`Phase` ADT defined there.

## Lifecycle

```
                                      botnoc.AgentService.Cancel
                                       ↓ (sets agent/cancelled)
botnoc.AgentService.Dispatch    workflow starts    workflow ends
   ↓ (sets                          ↓ (sets         ↓ (clears
   agent/start)                     agent/active)   labels)
   ┌─────────────┐  start label   ┌───────────────┐  cleanup  ┌──────────┐
   │ Dispatched  │ ─────────────► │   Working     │ ────────► │ Terminal │
   │ (agent/start│                │ (agent/active │           │ (no      │
   │  present)   │                │  present)     │           │  labels) │
   └─────────────┘                └───────────────┘           └──────────┘
                                                                  ▲
                                                                  │
                                                                  │ workflow
                                                                  │ opens PR
```

## Required setup per fastverk module

1. Copy `agent-workflow-template/agent-dispatch.yml` from this
   repo into `.github/workflows/agent-dispatch.yml` in the
   target module.
2. Configure the repo's `ANTHROPIC_API_KEY` secret if using the
   Claude Code backend.
3. Ensure the module's default branch CI is green (so generated
   PRs land cleanly).

## Backend matrix

| Backend label | Action | Status |
|---|---|---|
| `agent/claude` | `anthropics/claude-code-action@v1` | available |
| `agent/copilot` | Copilot Workspaces dispatch | not yet wired (workflow exits 1 with a TODO) |
| `agent/github-coding-agent` | GitHub's official coding agent | reserved; integration depends on its release |

Botnoc picks the backend per-dispatch; defaults to `agent/claude`
when `AgentBackend::Unspecified` arrives.

## How botnoc sees this from the other side

`AgentService.ListActive` runs a GitHub Search for
`is:issue is:open org:fastverk label:"agent/active"`
+
`is:issue is:open org:fastverk label:"agent/start"`,
merges + dedupes, and reports the result as `AgentRun` records.
The "id" is the issue ref; the synthetic shape is documented in
`fastverk/botnoc/services/agent/src/backend.rs::synthetic_run`.

`AgentService.Cancel` removes `agent/start` + `agent/active`,
adds `agent/cancelled`, and posts a cancellation comment with
the supplied reason.

## Why label-driven (and not workflow_dispatch directly)?

- **Single source of truth.** Labels live on the issue itself,
  visible to everyone, queryable via search, surviveable across
  workflow re-runs.
- **No per-repo workflow ID to track in botnoc.** Botnoc only
  needs to know "did the label transition?", not "what workflow
  run id is associated."
- **GitHub-native cancellation.** The cancel path is just
  label removal; no orphan workflow-run state.

## ROADMAP follow-ups

- [ ] Install `agent-dispatch.yml` into every fastverk module.
  Currently only the *template* exists in this repo; modules
  need to copy it.
- [ ] Reusable-workflow variant: turn this into a
  `workflow_call`-only file in `fastverk/.github/.github/workflows/`
  so modules can `uses:` it instead of duplicating.
- [ ] AgentService.Watch streaming — replace polling with a
  GitHub webhook-driven pubsub once we host a webhook receiver.
- [ ] Cost accounting — capture Anthropic API token spend per
  dispatch and surface in `PlanningService.TechDebtReport`.
