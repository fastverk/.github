# botnoc — fastverk's bot-driven NOC

A central console for orchestrating development across 30+
sibling repos. The name is "Network Operations Center" + "bot".

## Problem statement

Today there are 30+ `fastverk/rules_*` repos. Switching between
them is friction:

- Each has its own issues, PRs, CI runs, dependencies.
- A change in `rules_cc_cross` ripples through `rules_microkit`,
  `rules_microkit_tool`, `rules_board`, …
- Agentic development can parallelize the work, but only if there's
  a way to dispatch issues to the right repo, watch agents run,
  triage their PRs, and merge them in coordinated waves.
- "Where are my open PRs right now?" requires N tabs across N
  repos.

## Goal

A single pane of glass — runnable from any host with `gh` + bazel
+ `rels` — that:

1. **Shows org-wide state.** Open issues, open PRs, CI status,
   recent commits, dependency drift, registry-vs-tag drift. One
   query, one table.
2. **Dispatches agent work.** From the same console, file an
   issue across multiple repos, label it `agent/start`, and a
   workflow kicks off Claude Code (or Copilot Workspaces, or
   GitHub's Coding Agent) in a Codespace for each. The agent
   commits to a branch + opens a PR back.
3. **Triages PRs.** Aggregated PR queue across the org, sorted by
   age / by labels / by repo. Per-PR dive into Bazel CI logs,
   diffs, and `ultrareview` results.
4. **Tracks cross-repo deps.** When `rules_cc_cross` cuts 0.2.0,
   show me every consumer that needs a bump + offer to `rels bump`
   them with one command.
5. **Composes with `rels`.** Existing `rels release`, `audit`,
   `matrix`, `bump`, `mcp serve` extend naturally. The new
   surface is `rels noc` + a handful of `rels gh` subcommands.

## Architecture

### Layer 1 — `rels gh` (data plane)

Read-only queries against the GitHub API + local sibling
checkouts. New subcommands:

```
rels gh status [--repo <name>...] [--json]
    For each repo: name, default branch, open issues, open PRs,
    CI status of latest main commit, ahead/behind vs registry,
    last commit age. Tabular by default, JSON for piping.

rels gh inbox [--mine] [--label <label>]
    Aggregated PR queue across the org. Filter by reviewer,
    label, age. Includes Bazel CI status + ultrareview rollup
    if present.

rels gh deps <module>
    Show the consumer graph: which sibling modules depend on
    <module>, and which version each pins. Driven by walking
    sibling MODULE.bazel files for `bazel_dep(name = "<module>")`
    lines.

rels gh issue create --repos <r1,r2,...> --title "..." [--body-file <f>]
    Cross-repo issue creation. Used for "bump rules_cc_cross
    everywhere" style waves.
```

### Layer 2 — `rels noc` (control plane)

The TUI / dashboard. Two surfaces:

- **`rels noc`** with no args: prints a one-screen dashboard
  (build status, inbox count, recent activity). Designed for
  morning standup.
- **`rels noc interactive`**: full-screen TUI (crossterm-based,
  similar to `lazygit` ergonomics). Sections:
  - **Status**: per-repo CI / coverage / staleness badge.
  - **Inbox**: open PRs awaiting your review, sorted oldest-first.
  - **Backlog**: open issues labelled `next-up` or `agent/ready`
    across all repos.
  - **Agents**: in-flight agent dispatches — which repo, what
    they're working on, current commit, ETA.
  - **Releases**: modules with an uncommitted version bump in
    their checkout (next `rels release` candidates).

Each section is keyboard-navigable; pressing enter on an entry
drills into the relevant `gh` view or opens the PR in the
browser.

### Layer 3 — Agent dispatch (action plane)

Issues are the dispatch primitive. Each fastverk repo has a
GitHub Actions workflow at `.github/workflows/agent-dispatch.yml`
(shared via `fastverk/.github`):

```yaml
on:
  issues:
    types: [labeled]
jobs:
  dispatch:
    if: github.event.label.name == 'agent/start'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: anthropics/claude-code-action@v1   # or copilot equiv
        with:
          issue_number: ${{ github.event.issue.number }}
          prompt_file: .github/agent-prompt.md
```

The agent reads the issue body as its prompt, works in a branch
under that repo, opens a PR linked to the issue when done.

`rels noc` supports a "dispatch" key on the Backlog section: pick
an issue, hit `D`, choose the agent backend (claude / copilot /
codex), confirm. The TUI labels the issue, the workflow fires,
the row moves to Agents with live status.

### Layer 4 — Codespaces (workspace plane)

Every repo gets a `.devcontainer/devcontainer.json` so a Codespace
starts with bazel + bazelisk + buildifier + the relevant SDK
pre-installed. For premium modules, the Codespace's secrets
include the registry PAT for `.netrc`.

Why Codespaces matter:

- Cold-start parallelism: dispatching N agents = N Codespaces
  running in parallel. No "my laptop is busy" bottleneck.
- Hermetic re-creation: a Codespace is a fresh image every time
  — same as the agents see, same as CI sees.
- Remote pairing: you can SSH into an active agent's Codespace
  to watch / nudge / take over.

Each devcontainer specifies the right toolchain prebuild so
Bazel's repository cache is warm — important for `rules_qemu`
(400 MB of bottles) and `rules_kicad` (1.3 GB DMG).

### Layer 5 — Projects (planning plane)

Cross-repo planning lives in a single GitHub Projects v2 board
at the org level (`fastverk/projects/1`). Issues from all repos
auto-add via a workflow listening on issue creation. Views:

- **By repo**: groups issues by source repo.
- **By status**: backlog / in progress / review / done.
- **By milestone**: cross-repo coordinated bumps (e.g. "0.2.0
  series").
- **Agents**: filter to `agent/*`-labelled issues.

The TUI's Backlog / Agents sections are thin filtered views over
this single source of truth.

## A typical day with botnoc

```
$ rels noc
fastverk · 31 repos · main green: 28 · red: 1 · stale: 2

INBOX                                                  3 PRs
  rules_microkit#12   bump cc_cross 0.1.0 -> 0.2.0     2d
  rules_kicad#5       add kicad_drc rule               4h
  rules_qemu#8        arm64_sonoma bottles             1h

BACKLOG                                                7 issues (3 agent-ready)
  rules_verilog#3     [agent/ready] add hermetic       —
  rules_chisel#1      [agent/ready] Mill bootstrap     —
  rules_sel4#2        [agent/ready] qemu_arm_virt      —
  ...

AGENTS                                                 2 dispatched
  rules_riscv_core    @claude · ibex_small preset · 12m elapsed
  rules_naga          @copilot · wgsl_stdlib v2     · 4m elapsed

# Hit "D" on rules_verilog#3 to dispatch. Choose claude.
# Issue gets agent/start label, workflow fires, Codespace boots.
# Row moves to AGENTS section.
```

You watch agents run in the background while you review the
inbox PRs. When agent PRs land, they pop into INBOX
automatically. Pick a coordinated milestone, hit `R`, and `rels`
walks you through `bump` + `release` + tag + push for every
affected repo.

## Phased rollout

### Phase 0 — foundations (this turn)

- [x] `fastverk/fastverk` repo with profile README + this doc.
- [ ] `fastverk/.github` repo with shared workflow templates.

### Phase 1 — `rels gh status` + inbox

- `rels gh status` printing the tabular per-repo summary.
- `rels gh inbox` aggregating PRs.
- Both wrap `gh` CLI + parsing.

### Phase 2 — `rels noc` dashboard

- One-shot `rels noc` (no args) prints the dashboard.
- Implemented on top of `rels gh status` + `rels gh inbox`.

### Phase 3 — Projects + workflow templates

- Create `fastverk/projects/1` with the four views.
- Workflow in `.github/workflows/add-to-project.yml` shared via
  `fastverk/.github` so every repo auto-adds issues.

### Phase 4 — Devcontainer rollout

- Per-repo `.devcontainer/devcontainer.json`. Different image
  per module (cc_cross needs cross GCC; kicad needs the bundle;
  qemu the bottles; chisel a JDK).
- Codespaces prebuilds warmed on schedule so cold starts are fast.

### Phase 5 — Agent dispatch

- `.github/workflows/agent-dispatch.yml` template.
- Configure Claude Code action / Copilot Workspaces / GitHub
  Coding Agent (pick one per agent backend).
- `rels noc` adds the `D` keybind.

### Phase 6 — Full TUI

- Replace one-shot `rels noc` with crossterm interactive UI.
- Live update via WebSocket against GitHub events / `gh api graphql`.

### Phase 7 — Cross-repo wave releases

- `rels wave bump rules_cc_cross --to 0.2.0` walks the consumer
  graph, opens coordinated PRs in each consumer, ties them
  together via a Project milestone, and merges in dependency
  order once all CI is green.

## Why not just X?

- **GitHub Projects alone?** Great for tracking, but no way to
  dispatch agents or surface CI status compactly.
- **A custom web dashboard?** Bigger lift, separate deployment
  story, fewer hooks into local dev. CLI + TUI is faster to
  iterate.
- **Claude Code's `/loop` / `/schedule`?** Those are great for
  individual tasks but don't compose into a multi-repo console.
  botnoc would call them, not replace them.
- **dependabot / renovate?** Useful for upstream bumps, but
  doesn't understand the cross-fastverk dep graph or the
  registry-version layer. `rels wave bump` would.

## Open questions

1. **Agent backend portability.** Should `rels noc dispatch`
   abstract over claude / copilot / codex backends, or pick one?
   First answer: claude as default, claude-only in phase 5;
   pluggable backends in phase 6+.
2. **Premium-tier integration.** When premium modules ship, the
   noc dashboard needs to gate visibility on tier (some viewers
   shouldn't see premium-module activity). Filter at API time
   based on user's GitHub token scopes.
3. **State persistence.** Today everything's polled from GitHub
   on demand. If we want history (e.g. "agent dispatch frequency
   over time"), introduce a small SQLite layer next to the
   registry. Out of scope for phase 0-3.
