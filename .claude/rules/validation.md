# Validation rules

Use proportional validation, not a mandatory validation ladder.

The checks below are available options, ordered roughly from cheap to expensive.
Do not run all of them by default. Choose only checks that provide meaningful
evidence for the actual changed behavior.

Typical static/local checks:

1. `git diff --check`
2. YAML/repository formatting checks where applicable
3. `ansible-lint`
4. relevant playbook/load/syntax checks
5. safe targeted `--check` when it adds evidence
6. repository-specific tests

`git diff --check` is the normal minimum for code/configuration changes.

Repository-wide `ansible-lint`, syntax checks, and check-mode runs should normally
be left to CI unless:
- the changed area is directly covered by the check;
- CI feedback would be unnecessarily slow;
- the change is risky enough to justify local execution;
- the user explicitly asks for local validation.

Do not execute the same deterministic check twice in different agents without a
specific reason.

Deterministic checks should be repeated by CI when available.

Do not claim more than was executed:
- lint/syntax success proves static/load properties, not runtime behavior;
- Ansible check mode is predictive and is not a substitute for a real run;
- fresh-install, upgrade, service, GPU, Fcitx, k3s, and other environment-dependent behavior requires real runtime validation where practical;
- skipped or unsupported checks must be reported as such, not silently replaced by weaker evidence.
