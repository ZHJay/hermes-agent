# Hermes Issue Autopilot Design

## Goal

Provide a standalone, local automation that monitors only
`NousResearch/hermes-agent`, claims eligible open bug issues, develops fixes in
isolated worktrees with Codex CLI, and creates a draft pull request only after
the targeted test suite passes.

The automation is installed outside the Hermes core tree as a user plugin under
`~/.hermes/plugins/`. It must not add a permanent model tool to Hermes.

## Configuration and lifecycle

The plugin owns `~/.hermes/issue-autopilot/`:

- `config.yaml` records the fixed upstream repository, a two-minute poll
  interval, a maximum of three concurrent workers, the Codex command, test
  timeout, and fork/push configuration.
- `state.db` records each issue's state, claim comment id, branch, worktree,
  test result, cooldown, and draft PR URL.
- `logs/` stores scanner and worker logs.

A macOS LaunchAgent invokes the plugin every two minutes. It uses the existing
`gh` and Codex CLI authenticated sessions; secrets are neither copied into the
plugin configuration nor written to logs.

## Selection and conflict handling

Each scan considers only open issues in `NousResearch/hermes-agent` carrying
the `type/bug` or `bug` label. `needs-repro`, `needs-decision`, `duplicate`, and
`invalid` labels do not exclude an issue, but are preserved in the claim and PR
description so unverified premises are not presented as confirmed facts.

Before posting a claim, and again immediately before creating a PR, the plugin
checks issue assignees, all issue comments, the issue timeline, linked commits,
and open or merged PRs mentioning the issue. It skips an issue when another
contributor has claimed it, a related PR exists, the issue closed, or current
`main` no longer reproduces the problem.

The claim comment names the intended scope and test plan. If Codex cannot
reproduce the report, makes no useful change, or fails the targeted tests, the
plugin does not open a PR; it posts a release update with the reason and places
the issue on cooldown. Restarting the service must resume from `state.db`
without duplicate comments, worktrees, or PRs.

## Worker flow

At most three workers run simultaneously. Each receives an isolated Git
worktree and a branch named for its issue. The worker invokes Codex CLI with a
bounded task contract:

1. Read repository instructions and verify the report on current `main`.
2. Add a failing regression test before implementing the repair.
3. Make the smallest compatible change and run the focused test command.
4. Return structured evidence: reproduction result, files changed, test command
   and result, and any blocker.

The orchestrator rejects a worker result unless the targeted tests pass and the
worktree contains a non-empty, issue-scoped diff. It then rechecks GitHub,
commits the changes, pushes the worker branch to the configured fork, and opens
a draft PR against `NousResearch/hermes-agent:main`. The PR description includes
the issue link, reproduction, implementation summary, and exact test result.

## Interfaces

The standalone CLI supplies `start`, `stop`, `status`, `scan-once`, and
`logs` commands. `status` shows active workers, claimed issues, test outcomes,
cooldowns, and draft PRs. `scan-once` supports deterministic local testing
without installing the LaunchAgent.

## Verification

Tests use a fake GitHub client and temporary Git repositories to cover:

- bug-label selection and exclusion of features;
- claim races, linked-PR detection, and a second pre-PR conflict check;
- persistent state and restart recovery;
- the three-worker limit and worktree isolation;
- failed-test, no-diff, and unverified-reproduction paths that never create a
  PR; and
- a passing worker path that creates exactly one draft PR.

## Explicit non-goals

- Managing repositories other than `NousResearch/hermes-agent`.
- Merging PRs, editing labels, or closing issues.
- Creating a PR when the required targeted test command fails.
- Adding Hermes core tools or modifying the Hermes core to host this automation.
