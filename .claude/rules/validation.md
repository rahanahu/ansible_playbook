# Validation rules

Use the lightest validation that provides meaningful evidence for the change.

Typical static/local ladder:

1. `git diff --check`
2. YAML/repository formatting checks where applicable
3. `ansible-lint`
4. relevant playbook/load/syntax checks
5. safe targeted `--check` when it adds evidence
6. repository-specific tests

Deterministic checks should be repeated by CI when available.

Do not claim more than was executed:
- lint/syntax success proves static/load properties, not runtime behavior;
- Ansible check mode is predictive and is not a substitute for a real run;
- fresh-install, upgrade, service, GPU, Fcitx, k3s, and other environment-dependent behavior requires real runtime validation where practical;
- skipped or unsupported checks must be reported as such, not silently replaced by weaker evidence.
