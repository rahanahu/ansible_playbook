# Manual validation status

CI covers YAML formatting, ansible-lint and syntax checks only. Nothing
below is proven by a green CI run; none of it has been confirmed on real
hardware or in a VM.

Which tools follow latest stable and which are pinned is documented in
README.md under "Tool update policy".

## Open

- k3s has not actually been installed and re-run to confirm idempotency.
  The verification machine never started the systemd service, so only
  `--syntax-check` and check-mode behaviour were exercised.
- The Fedora Workstation and LXC entrypoints
  (`playbooks/fedora_workstation.yml`, `playbooks/fedora_system.yml`,
  `playbooks/fedora_lxc.yml`, and their `fedora44_*` compatibility
  wrappers) have not been run twice on real Fedora hardware or LXC guests
  with recaps compared to confirm second-run idempotency.
- The Ubuntu/Debian and macOS paths of `playbooks/zsh.yml` have not been
  exercised on real hosts: package availability, login shell, direnv,
  headless override, legacy `.zshrc` migration.
- AMD/Vulkan graphics validation has not been confirmed on real AMD or
  mixed-GPU hardware: per-device amdgpu/RADV checks, non-AMD layouts,
  mixed-GPU layouts.
- Hazkey has not been rebuilt against a real external static Swift
  toolchain and confirmed to link statically and work end to end as an
  installed input method.

## Confirmed

- Helm is idempotent across a real v3.21.4 to v4.2.4 upgrade, with the
  second run reporting `changed=0`.
- Hazkey reaches `changed=0` on a second run with the same pin, and
  rebuilds only when the pin moves.
- Tools that follow latest stable resolve stable release tags rather than
  development HEAD, and do not report spurious `changed` on a second run.
