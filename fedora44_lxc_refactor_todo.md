# Fedora 44 LXC and graphics acceptance TODO

## Implemented scope

The working implementation now provides:

- a dedicated `playbooks/fedora44_lxc.yml` entrypoint;
- early, non-bypassable Workstation/LXC platform guards;
- shared Fedora release validation;
- `workstation`, `root-safe`, and `server` user profiles;
- a direct-root LXC default using the `server` profile;
- headless zsh dependencies and container-appropriate CLI packages;
- no Docker, desktop, graphics, gaming, Fcitx, Hazkey, or Snapper work in LXC;
- PCI vendor `1002` AMD detection on Workstation;
- AMD-only amdgpu/RADV validation with non-AMD and mixed-GPU-safe behavior;
- updated README entrypoint and profile documentation.

Static checks completed in the Windows development environment:

- changed-file YAML header, tab, and trailing-whitespace checks;
- role, task-file, and playbook import reference checks;
- LXC role-boundary and Workstation guard-order checks;
- AMD PCI matching fixtures for AMD, Intel, NVIDIA, virtual, and mixed layouts;
- Markdown fence and `git diff --check` checks.

Ansible is not installed in that environment. The checks below remain open and
must be run on Fedora and disposable LXC targets before calling the refactor
fully accepted.

## Syntax and tag checks

- [ ] Run `ansible-playbook --syntax-check` for the Workstation, Workstation
      system, LXC, and standalone user entrypoints.
- [ ] Syntax-check `user.yml` with `workstation`, `root-safe`, and `server`.
- [ ] Verify `--tags system`, `user`, `cli`, `zsh`, and `graphics` still run the
      required platform guard, target-user resolution, and profile validation.
- [ ] Run the repository's YAML/Ansible lint tooling when available.

Suggested commands:

```bash
ansible-playbook playbooks/fedora44_workstation.yml \
  -i localhost, -c local -e target_user="$USER" --syntax-check
ansible-playbook playbooks/fedora44_system.yml \
  -i localhost, -c local --syntax-check
ansible-playbook playbooks/fedora44_lxc.yml \
  -i localhost, -c local --syntax-check
ansible-playbook playbooks/user.yml \
  -i localhost, -c local -e target_user=root \
  -e user_profile=server --syntax-check
```

## Fedora 44 LXC acceptance

- [ ] Capture virtualization type/role, PID 1, systemd state, cgroup mount,
      SELinux state, firewalld presence, and target passwd/home data before the
      first run on a disposable Proxmox Fedora 44 LXC.
- [ ] Run the direct-root command and confirm root receives
      zsh/fzf/zoxide/Sheldon/uv/nvm while its login shell remains bash.
- [ ] Confirm the default run does not require SELinux Enforcing or firewalld and
      does not install Docker, graphics, desktop, gaming, Fcitx, Hazkey, or
      Snapper components.
- [ ] Run the LXC playbook twice and investigate every unexpected second-run
      `changed`, especially zsh, fzf, Sheldon, uv, nvm, and CLI packages.
- [ ] Run with an explicit ordinary `target_user`; confirm its passwd-derived
      home changes and `/root` does not.
- [ ] Confirm `user_server_enable_direnv=true` installs and configures direnv,
      while the default server profile does neither.

Standard root invocation:

```bash
ansible-playbook playbooks/fedora44_lxc.yml -i localhost, -c local
```

## Workstation graphics acceptance

- [ ] On an AMD Fedora 44 Workstation, confirm PCI detection, per-device amdgpu
      validation, AMD/RADV Vulkan validation, and second-run idempotency.
- [ ] On a non-AMD Workstation or representative VM, confirm common Mesa/Vulkan
      packages remain installed and AMD-specific checks skip without failure.
- [ ] Fixture- or host-test AMD-only, Intel-only, NVIDIA-only, virtual GPU,
      Intel+AMD, and AMD+NVIDIA layouts through the Ansible tasks themselves.
- [ ] Confirm failed or unavailable `lspci` produces a clear failure rather than
      a false "AMD not detected" result.
- [ ] Confirm a mixed-GPU Vulkan summary finds AMD/RADV without requiring AMD to
      be the default enumerated device.
- [ ] Confirm common session validation accepts a working non-AMD renderer and
      the explicit AMD-session-renderer option rejects a non-AMD/llvmpipe result.
- [ ] Confirm no GPU-vendor package or driver is removed on any layout.

## Negative and regression checks

- [ ] Run the Workstation playbook in LXC and confirm it stops at the platform
      guard before SELinux/firewalld assertions or mutations.
- [ ] Run the LXC playbook on bare metal or KVM and confirm it points to the
      Workstation entrypoints.
- [ ] Confirm direct-root Workstation execution still requires an explicit
      non-root desktop `target_user`.
- [ ] Confirm `user.yml -e target_user=root` defaults to `root-safe`, while an
      explicit `user_profile=server` enables uv/nvm without changing the shell.
- [ ] Confirm invalid profiles, nonexistent users, and mismatched `target_home`
      values fail safely and clearly.
- [ ] Run the Workstation playbook twice and confirm its existing one-command UX
      and idempotency remain intact.

## Out of scope

- Proxmox host configuration and Docker inside LXC.
- GPU/device passthrough into LXC.
- NVIDIA or Intel driver management.
- GPU vendor package pruning or automatic driver cleanup.
