# Git operation rules

- Repository modification and repository publication are separate actions.
- Do not commit, push, merge, rebase, reset, or force-update refs merely because implementation is complete.
- Perform a state-changing Git operation only when the user explicitly requested that operation.
- Never force-push a protected/default branch.
- Inspect status, branch, and intended diff before an explicitly authorized commit or publication operation.
- Prefer repository permissions, CI, and GitHub branch protection/rulesets for hard enforcement; prompt instructions are not a security boundary.
