# Project instructions

Use the smallest Claude Code workflow appropriate to the task.

## Agent workflow

The main Claude session owns requirement interpretation, ordinary design decisions,
delegation, synthesis, and the final user-facing report.

For normal non-trivial changes:

1. Use `repo-explorer` only when broad repository discovery is necessary.
2. The main session decides the design and gives `implementer` a bounded brief.
3. `implementer` makes the smallest coherent change and runs fast local checks.
4. The main session inspects the resulting diff.
5. Use `reviewer` for independent semantic review.
6. Return genuine defects to `implementer` for focused fixes.
7. Rely on deterministic CI as the final automated static gate.

For trivial changes, skip agents that add no value. Do not invoke agents merely
because they exist.

Do not ask multiple agents to repeat the same repository-wide search. Pass exact
paths, constraints, and already-measured facts forward instead. Do not assign
multiple writers to the same files concurrently.

A subagent's final response is its deliverable. Do not ask read-only agents to
persist reports to repository files.

## Project rules

Repository-specific policy is split under `.claude/rules/`:

- `ansible.md` -- Ansible contracts, idempotency, check mode, ownership, and lint policy.
- `git.md` -- Git authority and publication rules.
- `validation.md` -- What local checks and CI do and do not prove.

## Tracking policy

Track project-level Claude configuration in Git. Keep user/machine-local state,
including `.claude/settings.local.json`, `.claude/agent-memory-local/`, temporary
logs, and caches untracked.
