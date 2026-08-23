# Project instructions

This repository uses a coordinated Claude Code agent workflow.

## Main workflow

For non-trivial work:

```text
orchestrator
  -> architect
  -> implementer
  -> reviewer
  -> implementer (only for real fixes)
  -> verifier
```

The parent/orchestrator owns scope, architecture decisions, final review, and the final user-facing report.

## Critical subagent-delivery rule

Read-only/plan agents must return their work through their final response.

Do not ask `architect` or `reviewer` to persist their report to repository files.

After invoking `architect`:

1. Wait for the architect's final response.
2. Do not continue until a usable architecture result has been received.
3. The architect's final response is the deliverable.
4. Restate/summarize the architecture result in the parent conversation before implementation begins.
5. If the architect returns no usable final report, stop and report the failure.
6. Do not silently retry the same architect task more than once.
7. Never switch to file-writing as a workaround for a plan/read-only agent.

The same principle applies to reviewer/verifier reports: receive them in the parent session and inspect the actual diff/output yourself.

## Git operations

Never commit, push, merge, rebase, reset, or force-update refs unless the user explicitly requests that Git operation.

## Repository-specific contracts

- `target_user` and `target_home` are repository-wide public contracts.
- Do not rename repository-wide public variables merely to satisfy role-local lint prefix rules.
- Preserve Workstation vs LXC responsibility boundaries.
- Preserve latest-vs-pinned tool update policy.
- Do not mutate persistent state merely to make check mode pass.
- Do not blindly overwrite existing Fcitx configuration.
- Root-executed installer scripts must not be staged in target-user-writable locations.

## Lint policy

Allowed narrow global exception:

```yaml
skip_list:
  - var-naming[no-role-prefix]
```

Do not globally disable `no-changed-when`.
Existing reviewed pre-CI debt may be grandfathered via line-specific `.ansible-lint-ignore`; new findings should normally be fixed.

## Tracking policy

Track project-level Claude configuration in Git:

```text
CLAUDE.md
.claude/settings.json
.claude/agents/orchestrator.md
.claude/agents/architect.md
.claude/agents/implementer.md
.claude/agents/reviewer.md
.claude/agents/verifier.md
```

Do not track user/machine-local state:

```text
.claude/settings.local.json
.claude/agent-memory-local/
temporary logs/caches
```
