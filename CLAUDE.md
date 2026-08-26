# Project instructions

Use the smallest Claude Code workflow appropriate to the task.

## Agent workflow

The main Claude session owns requirement interpretation, ordinary design decisions,
ordinary implementation, delegation, synthesis, and the final user-facing report.
Subagents are optional specialists, not mandatory workflow stages.

### Trivial and normal changes

For trivial and ordinary scoped changes:

1. Inspect only the relevant repository context.
2. Implement directly in the main session.
3. Run proportional fast validation.
4. Inspect the resulting diff.
5. Finish.

Do not invoke a subagent merely because a change is non-trivial.

### Repository exploration

Use `repo-explorer` only when broad repository discovery is necessary to identify
relevant files, callers, ownership boundaries, or variable flow. Do not use it when
the user already supplied exact paths or the main session can cheaply identify the
relevant scope.

### Implementation delegation

Use `implementer` only when implementation is large enough that delegation clearly
reduces main-session context, and the work can be expressed as a bounded brief.
Being expressible as a bounded brief is not on its own a reason to delegate.
Pass exact paths, requirements, known facts, and constraints. Do not ask the
implementer to rediscover information already known by the main session.

### Independent review

Use `reviewer` only for changes with meaningful semantic risk. Typical reasons
include:

- privilege, ownership, or become behavior;
- destructive operations or migrations;
- systemd, boot, mount, or service lifecycle behavior;
- package source or update-policy changes;
- security-sensitive behavior;
- public contract changes;
- cross-role changes;
- complex fresh-host, upgrade, or second-run semantics;
- runtime behavior that cannot be confidently inferred from the implementation.

Do not invoke `reviewer` solely because a change is non-trivial. For ordinary local
changes, main-session diff inspection is sufficient.

If review finds a genuine Critical or Important defect, make the smallest focused
fix. Do not restart broad repository exploration or the entire implementation
workflow. Re-run reviewer only when the fix materially changes the behavior under
review; Minor fixes normally need only main-session inspection.

Do not ask multiple agents to repeat the same repository-wide search. Pass exact
paths, constraints, and already-measured facts forward instead. Do not assign
multiple writers to the same files concurrently.

A subagent's final response is its deliverable. Do not ask read-only agents to
persist reports to repository files.

## Validation

Validation must be proportional to the change. Prefer the cheapest check that
provides meaningful evidence, and do not repeat deterministic checks across the
main session, implementer, and reviewer without a concrete reason. Rely on CI for
repository-wide deterministic checks already covered by CI.

## Project rules

Repository-specific policy is split under `.claude/rules/`:

- `ansible.md` -- Ansible contracts, idempotency, check mode, ownership, and lint policy.
- `git.md` -- Git authority and publication rules.
- `validation.md` -- What local checks and CI do and do not prove.

## Tracking policy

Track project-level Claude configuration in Git. Keep user/machine-local state,
including `.claude/settings.local.json`, `.claude/agent-memory-local/`, temporary
logs, and caches untracked.
