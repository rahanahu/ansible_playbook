# ansible_playbook

This repository contains Ansible playbooks and roles to provision developer
workstations and Fedora LXC guests.

## Which playbook should I run?

| Goal | Playbook |
|---|---|
| Build a complete Fedora 44 desktop | `playbooks/fedora44_workstation.yml` |
| Configure a Fedora 44 LXC guest | `playbooks/fedora44_lxc.yml` |
| Repair workstation system components only | `playbooks/fedora44_system.yml` |
| Configure one user's home environment | `playbooks/user.yml` |
| Build Hazkey after validating Fcitx5/Mozc | `playbooks/hazkey.yml` |

For an ordinary Fedora desktop, use this one command as the normal login user:

```bash
ansible-playbook playbooks/fedora44_workstation.yml -i localhost, -c local -K
```

## Supported targets

- Ubuntu / Debian
- Fedora Workstation and Fedora LXC
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

The Fedora workstation wrapper combines machine-wide `system` tasks,
per-account `user` tasks, root zsh, and conditional graphical `session` checks. It installs the
common Mesa baseline, detected AMD support, RPM Fusion, desktop essentials, CLI diagnostics,
development tools, Steam/MangoHud, and Fcitx5/Mozc.

Running as the normal login user is recommended. The target defaults to that
account and its home directory is resolved from the account database:

```bash
ansible-playbook playbooks/fedora44_workstation.yml -i localhost, -c local -K
```

When Ansible runs as root through sudo, a valid non-root `SUDO_USER` is used
automatically. From a direct root login, the desktop account is ambiguous and
must be explicit. Its home is resolved from `getent`; `/home/<user>` is never
assumed:

```bash
ansible-playbook playbooks/fedora44_workstation.yml \
  -i localhost, -c local \
  -e target_user=yusuke
```

User configuration, Docker group membership, uv, nvm, user Flatpak data, and
Fcitx environment files always target `target_user`, even during a root
invocation. Session checks run only when Ansible is started by the target user
from an available graphical session, so SSH/root recovery does not block system
tasks.

The zsh role is applied to both the workstation user and root by default. The
workstation user's login shell becomes zsh. Root's login shell is not changed,
and direnv is disabled in root's managed zsh configuration:

```text
sudo -i      -> root login bash
sudo -H zsh  -> root Ansible-managed zsh
```

Use `-e fedora44_configure_root_zsh=false` to skip root's zsh environment.

Scope and feature tags can limit recovery runs:

```bash
ansible-playbook playbooks/fedora44_workstation.yml \
  -i localhost, -c local -K --tags system

ansible-playbook playbooks/fedora44_workstation.yml \
  -i localhost, -c local -K --tags user
```

For workstation system-only recovery, especially from a direct root login, no
target user is needed. This remains a workstation baseline and intentionally
rejects LXC containers:

```bash
ansible-playbook playbooks/fedora44_system.yml -i localhost, -c local
```

To configure only one account, use `user.yml`. It defaults to the invoking
non-root user or a valid `SUDO_USER`; otherwise specify the account:

```bash
ansible-playbook playbooks/user.yml -i localhost, -c local \
  -e target_user=root

ansible-playbook playbooks/user.yml -i localhost, -c local \
  -e target_user=yusuke
```

`user.yml` supports `workstation`, `root-safe`, and `server` profiles. A normal
user defaults to `workstation`; root defaults to `root-safe`. The workstation
profile enables zsh, direnv, uv, nvm, Docker access, and Fedora desktop user
configuration. `root-safe` installs only the managed zsh environment. Direnv,
uv, nvm, Fcitx, ProtonPlus, and login-shell changes are disabled.

The `server` profile enables zsh, uv, and nvm without desktop integration or a
login-shell change. It can be selected explicitly, including for root:

```bash
ansible-playbook playbooks/user.yml -i localhost, -c local \
  -e target_user=root \
  -e user_profile=server
```

Server direnv integration remains off unless
`-e user_server_enable_direnv=true` is supplied.

### Fedora 44 LXC

The LXC entrypoint is a separate headless baseline. It validates Fedora 44, an
LXC guest, and a usable systemd environment, then installs CLI tools, headless
zsh dependencies, and the target user's `server` profile. A direct root login
needs only:

```bash
ansible-playbook playbooks/fedora44_lxc.yml -i localhost, -c local
```

That command targets `/root`, installs zsh/fzf/zoxide/Sheldon/uv/nvm, and keeps
root's login shell as bash. To configure an existing ordinary account while
running Ansible as root, specify it explicitly; its home is read from passwd:

```bash
ansible-playbook playbooks/fedora44_lxc.yml \
  -i localhost, -c local \
  -e target_user=yusuke
```

The LXC baseline does not manage Docker, graphics/GPU devices, KDE, Wayland,
desktop multimedia, gaming, Fcitx, Hazkey, or Snapper. It does not require
SELinux Enforcing or active firewalld. Guest firewalld is opt-in with
`-e fedora44_lxc_install_firewalld=true`; Proxmox host networking and firewall
policy remain outside this repository.

Running the workstation entrypoint in LXC, or the LXC entrypoint on bare metal
or KVM, stops early with the correct replacement command instead of skipping a
large number of incompatible tasks.

### Fedora workstation graphics

The graphics role installs common Mesa, Vulkan, and OpenGL userspace for Fedora
workstations. It detects AMD graphics controllers from PCI vendor ID `1002` and
runs amdgpu/RADV validation only when an AMD GPU is present. AMD absence is not
an error, and no package or existing driver is removed based on GPU vendor.

NVIDIA- and Intel-specific driver management is not implemented. On a mixed-GPU
system, the active graphical session is not required to render on AMD by
default. Set `fedora44_graphics_require_amd_session_renderer=true` only when the
session is intentionally expected to use the AMD adapter.

Desktop essentials include full FFmpeg, RPM Fusion GStreamer/VA-API components,
archive/filesystem support, KDE integration, Japanese/emoji fonts, and hardware
diagnostics. CLI/diagnostic tools are enabled by default, including packet/DNS,
route, throughput, TCP/UDP, syscall, debugger, sensor, SMART, and NVMe tools.
Disable them with `fedora44_install_cli_tools=false` or disable only the heavier
debuggers with `fedora44_install_heavy_debug_tools=false`.

Snapper packages are installed by the system baseline. Creating its root
configuration and a baseline snapshot remains explicit because it can affect the
Btrfs subvolume layout:

```bash
ansible-playbook playbooks/fedora44_workstation.yml \
  -i localhost, -c local -K \
  -e fedora44_snapper_create_root_config=true \
  -e fedora44_snapper_create_baseline_snapshot=true
```

GameMode, gamescope, and protontricks are also opt-in with
`fedora44_install_gaming_extras=true`. ProtonPlus uses
`fedora44_install_protonplus=true`; its Flatpak data belongs to the workstation
user rather than root. Optional NAS clients use `fedora44_install_nas_tools=true`
and Wireshark/nmap use `fedora44_install_network_gui_tools=true`.

The playbook does not change UEFI settings, upgrade or reboot the operating
system, select KDE virtual-keyboard settings, run games, or apply GPU tuning.

### Hazkey

Hazkey is deliberately separate from the workstation baseline. After Fcitx5 and
Mozc have been validated, build the CPU version with:

```bash
ansible-playbook playbooks/hazkey.yml -i localhost, -c local -K
```

The first run clones `yosagi/hazkey:dev`; later runs do not advance the moving
branch unless `hazkey_update_repository=true` is supplied. The checked-out SHA
is recorded in `~/src/hazkey-yosagi-dev.commit`.

When starting the Hazkey playbook as root, specify the target user exactly as for
the main workstation playbook:

```bash
ansible-playbook playbooks/hazkey.yml \
  -i localhost, -c local \
  -e target_user=yusuke
```

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
- Normal-user execution is preferred. Direct-root workstation runs require an explicit non-root `target_user`; system-only runs do not.

## Contributing

Keep OS-specific behavior inside roles where practical. Verify syntax and idempotency by running the same playbook twice; the second run should not report unnecessary changes.
