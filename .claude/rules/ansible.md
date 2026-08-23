---
paths:
  - "playbooks/**/*.yml"
  - "playbooks/**/*.yaml"
  - "requirements.yml"
  - ".ansible-lint"
  - ".ansible-lint-ignore"
---

# Ansible repository rules

- `target_user` and `target_home` are repository-wide public contracts.
- Do not rename repository-wide public variables merely to satisfy role-local lint prefix rules.
- Preserve Workstation vs LXC responsibility boundaries.
- Preserve latest-vs-pinned tool update policy.
- Preserve idempotency, second-run behavior, and correct changed-state reporting.
- Treat check-mode success and real runtime validation as different claims.
- Do not mutate persistent state merely to make check mode pass.
- Review become, ownership, target user, and target home semantics.
- Do not blindly overwrite existing Fcitx configuration.
- Root-executed installer scripts must not be staged in target-user-writable locations.
- Prefer built-in Ansible modules over shell/command when they model the operation cleanly.
- Do not expose secrets or sensitive inventory data in logs, diffs, reports, or commits.

## Lint policy

The only allowed narrow global variable-name exception is:

```yaml
skip_list:
  - var-naming[no-role-prefix]
```

Do not globally disable or demote `no-changed-when`. Existing reviewed pre-CI
debt may be grandfathered via line-specific `.ansible-lint-ignore`; new findings
should normally be fixed.
