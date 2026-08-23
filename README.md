# ansible_playbook

This repository contains Ansible playbooks and roles to provision developer workstations.

## Supported targets

- Ubuntu / Debian
- Fedora
- macOS (Darwin) — Docker and k3s are skipped

## Quick start

Install required Ansible collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

Run the developer workstation playbook as the normal login user. Tasks that need root privileges use `become: true` internally.

```bash
ansible-playbook playbooks/dev_pc.yml -i localhost, -c local
```

Run the k3s workstation playbook on Linux:

```bash
ansible-playbook playbooks/k3s_pc.yml -i localhost, -c local
```

## Fedora 44 KDE workstation

The Fedora workstation playbook provisions the repeatable baseline from
`fedora44_workstation_runbook.md`: Radeon/Mesa diagnostics, the existing
developer roles, RPM Fusion, Steam/MangoHud, and Fcitx5/Mozc.

It does not change UEFI settings, upgrade or reboot the operating system, select
KDE virtual-keyboard settings, run games, or apply GPU tuning.

Run it from a KDE Wayland session as the normal login user:

```bash
ansible-playbook playbooks/fedora44_workstation.yml -i localhost, -c local -K
```

Snapper is opt-in because creating its root configuration can change the Btrfs
subvolume layout. Installing the packages only:

```bash
ansible-playbook playbooks/fedora44_workstation.yml \
  -i localhost, -c local -K \
  -e fedora44_enable_snapper=true
```

To also create the root configuration and one baseline snapshot, explicitly set
both switches:

```bash
ansible-playbook playbooks/fedora44_workstation.yml \
  -i localhost, -c local -K \
  -e fedora44_enable_snapper=true \
  -e fedora44_snapper_create_root_config=true \
  -e fedora44_snapper_create_baseline_snapshot=true
```

GameMode, gamescope, and protontricks are also opt-in with
`fedora44_install_gaming_extras=true`. ProtonPlus uses
`fedora44_install_protonplus=true`, and the optional CLI package set uses
`fedora44_install_cli_extras=true`.

### Hazkey

Hazkey is deliberately separate from the workstation baseline. After Fcitx5 and
Mozc have been validated, build the CPU version with:

```bash
ansible-playbook playbooks/hazkey.yml -i localhost, -c local -K
```

The first run clones `yosagi/hazkey:dev`; later runs do not advance the moving
branch unless `hazkey_update_repository=true` is supplied. The checked-out SHA
is recorded in `~/src/hazkey-yosagi-dev.commit`.

After manually validating the CPU build, the Vulkan build is enabled explicitly:

```bash
ansible-playbook playbooks/hazkey.yml \
  -i localhost, -c local -K \
  -e hazkey_enable_vulkan=true
```

The Hazkey role does not fetch or cherry-pick upstream PR #25, resolve conflicts,
or automate KDE/Fcitx5 UI and input validation.

## Notes

- macOS: Homebrew is installed automatically if missing. Docker Desktop is intentionally not installed by this repository.
- k3s is Linux-only; no implicit kind/minikube replacement is installed on macOS.
- Fedora and Debian-family systems use Docker Engine from Docker's official repositories.
- `fzf` is installed from a pinned Git tag into `~/.local/share/fzf`; its binary is linked into `~/.local/bin`.
- `fzf_enable_completion` defaults to `true`. Disable it with `-e fzf_enable_completion=false`.
- Ansible-managed zsh settings live in `~/.config/zsh/ansible.zsh`. The role adds a small loader block at the start of `~/.zshrc`, so settings below that block remain user-editable and can override managed defaults.
- On Linux, `sheldon` is installed at the version pinned by `sheldon_version`. On macOS, Homebrew manages `sheldon`.
- Run playbooks as the normal user, not with `sudo ansible-playbook`.

## Contributing

Keep OS-specific behavior inside roles where practical. Verify syntax and idempotency by running the same playbook twice; the second run should not report unnecessary changes.
