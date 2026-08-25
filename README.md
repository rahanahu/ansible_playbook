# ansible_playbook

This repository contains Ansible playbooks and roles to provision developer
workstations and Fedora LXC guests.

## Which playbook should I run?

| Goal | Playbook |
|---|---|
| Install only the managed zsh environment | `playbooks/zsh.yml` |
| Install only Japanese fonts | `playbooks/fonts.yml` |
| Build a complete Fedora desktop | `playbooks/fedora_workstation.yml` |
| Configure a Fedora LXC guest | `playbooks/fedora_lxc.yml` |
| Repair workstation system components only | `playbooks/fedora_system.yml` |
| Configure one user's home environment | `playbooks/user.yml` |
| Build the complete developer environment | `playbooks/dev_pc.yml` |
| Build Hazkey after validating Fcitx5/Mozc | `playbooks/hazkey.yml` |
| Set up the local AI environment (opt-in) | `playbooks/ai_local.yml` |

For an ordinary Fedora desktop, use this one command as the normal login user:

```bash
ansible-playbook playbooks/fedora_workstation.yml -i localhost, -c local -K
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

## Managed zsh environment

`playbooks/zsh.yml` installs only the managed zsh environment. It does not
install Docker, uv, nvm, or Fedora desktop components.

On Ubuntu, Debian, or Fedora, run it as the normal user. System packages use
privilege escalation internally:

```bash
ansible-playbook playbooks/zsh.yml -i localhost, -c local -K
```

This installs zsh, git, zoxide, direnv, optional desktop clipboard integration,
fzf and Sheldon, and the managed zsh configuration. The normal Linux user's
login shell becomes zsh. For a headless machine, omit the desktop package:

```bash
ansible-playbook playbooks/zsh.yml -i localhost, -c local -K \
  -e zsh_install_desktop_packages=false
```

Linux root is never selected implicitly. To configure root from a direct root
login, specify it explicitly:

```bash
ansible-playbook playbooks/zsh.yml -i localhost, -c local \
  -e target_user=root
```

Root receives the managed zsh environment, fzf, zoxide, and Sheldon, but not
direnv or desktop integration. Its login shell remains unchanged. A different
existing Linux account can also be selected; its home is read from passwd:

```bash
ansible-playbook playbooks/zsh.yml -i localhost, -c local -K \
  -e target_user=yusuke
```

On macOS, run the playbook directly as the normal user being configured:

```bash
ansible-playbook playbooks/zsh.yml -i localhost, -c local
```

macOS uses the system `/bin/zsh`; the playbook does not install Homebrew zsh or
change the login shell. Homebrew provides the supporting tools. Running through
sudo/root or targeting another macOS account is intentionally rejected before
Homebrew or user configuration changes.

For the initial complete setup, run without tags or use `--tags zsh`. The
`system` and `user` tags are available for later repair runs; `--tags user` does
not install missing system packages.

## Fedora KDE workstation

Fedora 44 is the currently validated release and the minimum supported release.
Newer Fedora releases continue with an explicit unvalidated-release warning;
package or configuration failures are not ignored. Use
`-e fedora_strict_release_check=true` to require a release listed as validated.

A release can also be marked known-blocked, which is different from being
merely unvalidated: it means the generic Fedora baseline has actually been
found unsafe or non-functional on that release, and the run fails immediately
regardless of strict mode. Use `-e '{"fedora_blocked_releases": [45]}'`
(default empty) to reject a specific known-blocked release; the JSON form is
required because a plain `key=[45]` argument is parsed as a string, not a
list. The variable can equally be set in inventory or `group_vars`, which is
often the more natural place for a policy list.

### Migrating from the fedora44_* names

The canonical Fedora entrypoints are `playbooks/fedora_workstation.yml`,
`playbooks/fedora_system.yml`, and `playbooks/fedora_lxc.yml`.
`playbooks/fedora44_workstation.yml`, `fedora44_system.yml`, and
`fedora44_lxc.yml` remain temporarily as thin compatibility wrappers that
simply import the canonical playbook, so existing commands and any `-e`
flags keep working unchanged.

The published `fedora44_*` feature switches used below are also still
accepted. When both an old and a new name are supplied, the new `fedora_*`
name wins. The release-policy variables `fedora44_required_release` and
`fedora44_allow_other_release` are likewise still accepted.

| Old variable | New variable |
|---|---|
| `fedora44_install_cli_tools` | `fedora_install_cli_tools` |
| `fedora44_install_heavy_debug_tools` | `fedora_install_heavy_debug_tools` |
| `fedora44_install_nas_tools` | `fedora_install_nas_tools` |
| `fedora44_install_network_gui_tools` | `fedora_install_network_gui_tools` |
| `fedora44_install_gaming_extras` | `fedora_install_gaming_extras` |
| `fedora44_install_protonplus` | `fedora_install_protonplus` |
| `fedora44_graphics_require_amd_session_renderer` | `fedora_graphics_require_amd_session_renderer` |
| `fedora44_configure_root_zsh` | `fedora_configure_root_zsh` |
| `fedora44_lxc_install_firewalld` | `fedora_lxc_install_firewalld` |
| `fedora44_snapper_create_root_config` | `fedora_snapper_create_root_config` |
| `fedora44_snapper_create_baseline_snapshot` | `fedora_snapper_create_baseline_snapshot` |

New configurations should use the `fedora_*` entrypoints and `fedora_*`
variables directly; the compatibility layer above is temporary and intended
for one migration cycle. The underlying roles were also renamed from
`fedora44_*` to `fedora_*`, which matters only if something outside this
repository referenced them by role name directly.

The Fedora workstation entrypoint combines machine-wide `system` tasks,
per-account `user` tasks, root zsh, and conditional graphical `session` checks. It installs the
common Mesa baseline, detected AMD support, RPM Fusion, desktop essentials, CLI diagnostics,
development tools, Steam/MangoHud, and Fcitx5/Mozc.

Running as the normal login user is recommended. The target defaults to that
account and its home directory is resolved from the account database:

```bash
ansible-playbook playbooks/fedora_workstation.yml -i localhost, -c local -K
```

When Ansible runs as root through sudo, a valid non-root `SUDO_USER` is used
automatically. From a direct root login, the desktop account is ambiguous and
must be explicit. Its home is resolved from `getent`; `/home/<user>` is never
assumed:

```bash
ansible-playbook playbooks/fedora_workstation.yml \
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

Use `-e fedora_configure_root_zsh=false` to skip root's zsh environment.

Scope and feature tags can limit recovery runs:

```bash
ansible-playbook playbooks/fedora_workstation.yml \
  -i localhost, -c local -K --tags system

ansible-playbook playbooks/fedora_workstation.yml \
  -i localhost, -c local -K --tags user
```

For workstation system-only recovery, especially from a direct root login, no
target user is needed. This remains a workstation baseline and intentionally
rejects LXC containers:

```bash
ansible-playbook playbooks/fedora_system.yml -i localhost, -c local
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

### Fedora LXC

The LXC entrypoint is a separate headless baseline. It validates the Fedora
release, an LXC guest, and a usable systemd environment, then installs CLI
tools, headless zsh dependencies, and the target user's `server` profile. A
direct root login needs only:

```bash
ansible-playbook playbooks/fedora_lxc.yml -i localhost, -c local
```

That command targets `/root`, installs zsh/fzf/zoxide/Sheldon/uv/nvm, and keeps
root's login shell as bash. To configure an existing ordinary account while
running Ansible as root, specify it explicitly; its home is read from passwd:

```bash
ansible-playbook playbooks/fedora_lxc.yml \
  -i localhost, -c local \
  -e target_user=yusuke
```

The LXC baseline does not manage Docker, graphics/GPU devices, KDE, Wayland,
desktop multimedia, gaming, Fcitx, Hazkey, or Snapper. It does not require
SELinux Enforcing or active firewalld. Guest firewalld is opt-in with
`-e fedora_lxc_install_firewalld=true`; Proxmox host networking and firewall
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
default. Set `fedora_graphics_require_amd_session_renderer=true` only when the
session is intentionally expected to use the AMD adapter.

Desktop essentials include full FFmpeg, RPM Fusion GStreamer/VA-API components,
archive/filesystem support, KDE integration, Japanese/emoji fonts, and hardware
diagnostics. CLI/diagnostic tools are enabled by default, including packet/DNS,
route, throughput, TCP/UDP, syscall, debugger, sensor, SMART, and NVMe tools.
Disable them with `fedora_install_cli_tools=false` or disable only the heavier
debuggers with `fedora_install_heavy_debug_tools=false`.

Snapper packages are installed by the system baseline, along with
`btrfs-assistant` unless `fedora_install_snapper_gui=false`. Creating its root
configuration and a baseline snapshot remains explicit because it can affect the
Btrfs subvolume layout:

```bash
ansible-playbook playbooks/fedora_workstation.yml \
  -i localhost, -c local -K \
  -e fedora_snapper_create_root_config=true \
  -e fedora_snapper_create_baseline_snapshot=true
```

Retention (`fedora_snapper_manage_retention`) and timer management
(`fedora_snapper_manage_timers`) default to following
`fedora_snapper_create_root_config`. When enabled, the role sets
`TIMELINE_LIMIT_HOURLY`/`_DAILY`/`_WEEKLY`/`_MONTHLY`/`_QUARTERLY`/`_YEARLY`
via `fedora_snapper_timeline_limit_hourly`, `_daily`, `_weekly`, `_monthly`,
`_quarterly`, and `_yearly` (defaults `6`/`7`/`2`/`0`/`0`/`0`), and
`NUMBER_LIMIT`/`NUMBER_LIMIT_IMPORTANT` via `fedora_snapper_number_limit` and
`fedora_snapper_number_limit_important` (both default to `10`). It also
enables the units listed in `fedora_snapper_timers`, which defaults to
`snapper-timeline.timer` and `snapper-cleanup.timer`; `snapper-boot.timer` is
not enabled by default. Snapper only manages the `/` subvolume; `/home` is
not covered.

Hosts that already ran with `fedora_snapper_create_root_config=true` will see
their retention settings move from Fedora's package defaults (`HOURLY=10`,
`DAILY=10`, `WEEKLY=0`, `MONTHLY=10`, `QUARTERLY=0`, `YEARLY=10`,
`NUMBER_LIMIT=50`) to the role's defaults above the first time this role
runs. `QUARTERLY` is `0` in both cases, so only `HOURLY`/`DAILY`/`WEEKLY`/
`MONTHLY`/`YEARLY`/`NUMBER_LIMIT` actually change. Set
`fedora_snapper_manage_retention=false` to leave existing retention settings
untouched.

GameMode, gamescope, and protontricks are also opt-in with
`fedora_install_gaming_extras=true`. ProtonPlus uses
`fedora_install_protonplus=true`; its Flatpak data belongs to the workstation
user rather than root. Optional NAS clients use `fedora_install_nas_tools=true`
and Wireshark/nmap use `fedora_install_network_gui_tools=true`.

The playbook does not change UEFI settings, upgrade or reboot the operating
system, select KDE virtual-keyboard settings, run games, or apply GPU tuning.

### Japanese fonts

The `fonts` role broadens Japanese font coverage beyond the
`langpacks-fonts-ja` and Noto emoji fonts already installed by
`fedora_desktop`. It follows a package-manager-first policy: anything
packaged by the target OS is installed from its own package manager, and only
fonts with no OS package are fetched from upstream. It runs as part of
`playbooks/fedora_system.yml` / `playbooks/fedora_workstation.yml`, or on its
own via the dedicated `playbooks/fonts.yml` entrypoint, which additionally
supports Ubuntu 24.04+ and Debian 12+:

```bash
ansible-playbook playbooks/fonts.yml -i localhost, -c local -K
```

The package side (`fonts_install_packages`, default `true`) installs from
`fonts_packages_redhat` on Fedora and `fonts_packages_debian` on Ubuntu/
Debian. The two lists differ, because not every font packaged for Fedora also
has a Debian-family package:

- Fedora installs IPA, IPAex, M+, VL Gothic, Sazanami, Motoya, Hanamin, UD
  fonts, Source Han Code JP, JetBrains Mono, and Hack from Fedora's own
  repositories.
- Ubuntu 24.04+/Debian 12+ install BIZ UD, IPA, IPAex, M+, VL Gothic, Motoya,
  Hanazono, and JetBrains Mono from the distribution's repositories. Sazanami,
  Hack, and Source Han Code JP are intentionally left out because none of them
  has a Debian-family package. Ubuntu's M+ package resolves to upstream
  version 063, newer than the 056 that Fedora's `mplus-*-fonts` packages.

The archive side (`fonts_install_archives`, default `false`) is opt-in
because it pulls roughly 150 MB from upstream on first run, and is identical
on every supported OS:

```bash
ansible-playbook playbooks/fonts.yml \
  -i localhost, -c local -K \
  -e fonts_install_archives=true
```

It installs three fonts not available from any of the package managers above,
each individually switchable:

- UDEV Gothic, the Nerd Font build (`fonts_install_udev_gothic`, default
  `true`)
- HackGen (`fonts_install_hackgen`, default `true`) — the standard build is
  used rather than the Nerd Font build, because HackGen's Nerd Font release
  only bundles the four Console variants and omits the plain HackGen faces
- GenEi Gothic (`fonts_install_genei_gothic`, default `true`)

Nerd Font icon glyphs only need to live in one font actually selected in a
terminal; UDEV Gothic NF already carries them, so HackGen and GenEi Gothic do
not need their own Nerd Font builds for icons to show up.

Archive fonts install under `fonts_install_dir` (default
`/usr/local/share/fonts`), one subdirectory per font, each with a
`.fonts-version` marker file so unchanged versions are skipped on later runs.
This applies the same way regardless of OS family. When a release cannot be
resolved from the GitHub API, the role keeps whatever is already installed and
warns instead of failing. The role also restores the SELinux context after
installing archive fonts, but only on RedHat-family systems; Debian-family
systems skip that step entirely.

The marker filename changed from `.fedora-fonts-version` to `.fonts-version`
when the Fedora-only role this section documents was renamed to `fonts` and
gained Debian-family support. Hosts that already installed archive fonts
under the old marker name will not find the new one, so each archive font is
reinstalled once on the first run after upgrading.

### Fedora Btrfs subvolume layout

`playbooks/fedora_btrfs_layout.yml` splits selected paths under `/var` out of
the root Btrfs subvolume into their own top-level subvolumes, so Snapper
rollbacks of `/` do not also roll back that data. It is a separate,
dedicated playbook and is not part of `playbooks/fedora_system.yml` or the
LXC baseline; run it explicitly when you want it.

The role requires `/` to be a Btrfs filesystem and `/home` to already be
mounted from its own independent subvolume; the validation step fails
closed before making any change if either prerequisite is not met.

This role declares and maintains a subvolume layout — it is not a migration
tool. If the target mountpoint already contains data, the role fails without
deleting, moving, or backing up anything. Inspect and migrate or remove the
existing data by hand, once, then rerun the playbook; there is no built-in
rsync/move step or special case that is allowed to delete data automatically.

Everything is opt-in and defaults to `false`:

```bash
ansible-playbook playbooks/fedora_btrfs_layout.yml \
  -i localhost, -c local -K \
  -e fedora_btrfs_layout_var_log=true \
  -e fedora_btrfs_layout_var_cache=true \
  -e fedora_btrfs_layout_flatpak=true \
  -e fedora_btrfs_layout_docker=true \
  -e fedora_btrfs_layout_containerd=true \
  -e fedora_btrfs_layout_libvirt_images=true \
  -e fedora_btrfs_layout_podman_root=true \
  -e fedora_btrfs_layout_podman_rootless=true
```

`fedora_btrfs_layout_var_log` covers `/var/log`, `fedora_btrfs_layout_var_cache`
covers `/var/cache`, `fedora_btrfs_layout_flatpak` covers `/var/lib/flatpak`,
`fedora_btrfs_layout_docker` covers `/var/lib/docker` (stopping and restarting
`docker.socket`/`docker.service` around the remount),
`fedora_btrfs_layout_containerd` covers `/var/lib/containerd` (stopping and
restarting `containerd.service`), `fedora_btrfs_layout_libvirt_images`
covers `/var/lib/libvirt/images`, `fedora_btrfs_layout_podman_root` covers
rootful Podman storage at `/var/lib/containers/storage` (stopping and
restarting `podman.socket` around the remount, if present and active), and
`fedora_btrfs_layout_podman_rootless` covers rootless Podman storage at
`<target_home>/.local/share/containers/storage`.

Enabling `fedora_btrfs_layout_podman_rootless` resolves `target_user` (and
therefore `target_home`) via the `target_user` role before the layout runs,
using the same resolution order as the rest of the repository. Pass
`-e target_user=<login-user>` to make the intended account explicit rather
than relying on autodetection. Because the resulting mountpoint lives under
`/home`, and `/home` is already its own independent Btrfs subvolume outside
`/`, rootless Podman storage was never covered by a Snapper snapshot of `/`
in the first place; this entry mainly matters if you snapshot `/home`
itself, or want to exclude container image/layer churn from a `/home`
backup.

`services` for `/var/log` is intentionally empty, so remounting it does not
stop and restart `systemd-journald`. That means an already-running
`systemd-journald` keeps writing to the old, now-hidden inode after the
remount, and `/var/log/journal` under the new subvolume stays empty until
the service is restarted. For that reason, only opt `/var/log` into this
role on a fresh system before journald has accumulated logs on the old
mountpoint; splitting it out on an already-running host requires a human to
stop journald and move things over by hand, which this role deliberately
does not automate.

Each subvolume's `mode` (`docker` `0710`, `containerd` `0700`,
`libvirt-images` `0711`, `podman-root` `0755`, `podman-rootless` `0700`, and
`var-log`/`var-cache`/`flatpak` `0755`) is a declared value, and the role
converges the mountpoint to that value on every run, after the subvolume is
mounted. If the mountpoint previously had a different permission mode set on
it by hand, that mode is overwritten by the declared value. Ownership
(`owner`/`group`) is only managed when an item explicitly declares it; by
default the role leaves ownership unchanged. `podman-rootless` is the only
built-in entry that declares `owner`/`group` (both `target_user`), since its
mountpoint must be writable by that user rather than root.

If a mountpoint's parent directory does not already exist, creating the
mountpoint also creates that parent, and the parent is created with default
(root) ownership regardless of any `owner`/`group` declared for the
mountpoint itself. For `podman-rootless` this is not expected to matter in
practice, because its parent (`~/.local/share/containers`) is normally
created by Podman itself before this role ever runs, but on a host where it
does not yet exist the role can leave a root-owned parent directory above a
user-owned mountpoint.

Snapper only manages the `/` subvolume; it does not gain a configuration for
any subvolume this role creates, so those paths are not covered by Snapper
snapshots or rollbacks either — which is the point of splitting them out.

`/boot` is a separate ext4 partition and is never part of the root Btrfs
subvolume, so it is never covered by a Snapper root snapshot, and rolling
back `/` is not atomic with the installed kernel. After a rollback, boot the
kernel version that exists both in `/boot` and under `/usr/lib/modules`; if
no matching kernel remains in `/boot`, recovering may require booting rescue
media and reinstalling a kernel package by hand. This is a known limitation
of Fedora's Btrfs/Snapper setup that `fedora_btrfs_layout` does not solve.

### Local AI (Ollama)

Local AI is opt-in. Nothing in `playbooks/fedora_workstation.yml`,
`playbooks/fedora_system.yml`, `playbooks/dev_pc.yml`, or
`playbooks/fedora_lxc.yml` installs it. Run it explicitly:

```bash
ansible-playbook playbooks/ai_local.yml -i localhost, -c local -K
```

Models live under `/srv/ai/models`. The playbook enables the
`fedora_btrfs_layout` `ai-models` subvolume, so `/srv/ai/models` is an
independent Btrfs subvolume and is not included in root Snapper snapshots.
Ollama is explicitly dependent on and ordered after that mount: the
systemd drop-in declares `RequiresMountsFor` against it. The service must
not fall back to running against a plain root-subvolume directory, so the
drop-in also declares `ConditionPathIsMountPoint`; if `/srv/ai/models` is
ever not a mount point, systemd skips starting the unit rather than
starting it against the wrong path. That behaviour is what
`systemd.unit(5)` specifies for a failing condition; it has not been
observed on this host, because the test would mean unmounting a store
that holds real models.

`fedora_strict_release_check=true` is set for the overall Fedora
workstation/Btrfs provisioning baseline, which is validated on Fedora 44;
`fedora_preflight`'s strict-release assert still rejects an unvalidated
Fedora release during `pre_tasks`. Ollama itself no longer depends on that
guard, or on any Fedora package topology: it is installed from the
upstream release binary (see below), so there is no
`ollama_allow_unvalidated_release`-style escape hatch to combine with
`-e fedora_strict_release_check=false` any more.

Ollama backend: upstream release binary, not a Fedora package. The role
installs `ollama-linux-amd64.tar.zst` and, unless
`-e ollama_install_rocm=false`, `ollama-linux-amd64-rocm.tar.zst` from the
project's own GitHub releases, extracted under `/usr/local` as
`/usr/local/bin/ollama` and `/usr/local/lib/ollama`. It follows the latest
upstream release by default; set `ollama_version` (without a leading `v`)
to pin one. Each archive's checksum is verified at install time against
that same release's `sha256sum.txt`, fetched over the network — this
detects transfer corruption, not upstream tampering, since both the
archive and its checksum manifest come from the same GitHub release.

The Fedora `ollama` RPM, if present, is explicitly removed first (`dnf
remove ollama`), so it never conflicts with the files this role puts under
`/usr/local`. Removing it does **not** use `dnf autoremove`-style dependency
cleanup: Ansible's dnf/dnf5 modules default `clean_requirements_on_remove`
to their `autoremove` parameter, which this role does not set, so it stays
`false`. That means the Fedora-native ROCm userspace previously pulled in
as dependencies of the `ollama` RPM (`rocm-hip`, `rocblas`, `hipblas`, and
the rest) is left installed as orphaned packages after migrating a host
from an older run of this role. The role does not run `dnf autoremove`
itself, because that would also sweep up unrelated orphaned packages
system-wide. If you want to reclaim that space, review the output of `dnf
autoremove` yourself before applying it.

Because the release archives are extracted under `/usr/local`, which lives
on the root Btrfs subvolume, each version update (a few GB) is captured by
root Snapper snapshots. That is accepted deliberately, the same way it was
before this role tracked upstream directly: only the models under
`/srv/ai/models`, on their own dedicated subvolume, are excluded from root
snapshots, and that is the tradeoff that actually matters for snapshot
size.

Ansible does not download models. `ollama pull` is left to the user.

### Hazkey

Hazkey is deliberately separate from the workstation baseline. After Fcitx5 and
Mozc have been validated, build the CPU version with:

```bash
ansible-playbook playbooks/hazkey.yml -i localhost, -c local -K
```

The role checks out a pinned commit that was verified end to end on a Fedora
VM, so every machine builds from the same source. Every run converges the
checkout to that pin: an existing clone sitting on some other commit is
corrected rather than silently left as is. Passing
`hazkey_update_repository=true` ignores the pin and tracks the moving `dev`
branch instead, which is how a candidate commit is obtained for testing. To
move the pin forward, fetch `dev` with that flag, verify the result on a VM
(build, install, and confirm real input), then set `hazkey_version` to the
newly verified SHA. The checked-out SHA is still recorded in
`~/src/hazkey-yosagi-dev.commit`.

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

Fedora's `swift-lang` currently ships no static Swift standard library, so the
role builds `hazkey-server` linked dynamically against `swift-lang-runtime`.
This is the default and it works. It does mean the binary depends on that
runtime staying installed and version-compatible, and Swift on Linux offers no
ABI stability guarantee across runtime updates. The role prints a message when
it falls back to dynamic linking.

An official swift.org toolchain does ship a static standard library. It can be
downloaded and extracted separately, for example into `~/swift-toolchain`
(about a 1 GB download, about 3.4 GB extracted):

```bash
mkdir -p ~/swift-toolchain
curl -L -o /tmp/swift-toolchain.tar.gz \
  https://download.swift.org/swift-6.3.3-release/fedora41/swift-6.3.3-RELEASE/swift-6.3.3-RELEASE-fedora41.tar.gz
tar -xzf /tmp/swift-toolchain.tar.gz \
  --strip-components=1 -C ~/swift-toolchain
```

Point the role at it with `hazkey_swift_toolchain_path`:

```bash
ansible-playbook playbooks/hazkey.yml \
  -i localhost, -c local -K \
  -e hazkey_swift_toolchain_path=/home/<user>/swift-toolchain
```

For regular use this belongs in inventory or `group_vars` rather than being
retyped on every run. The role's `defaults/main.yml` is the wrong place for it
because the path is machine-specific.

`hazkey_swift_toolchain_path` is the toolchain root, not the `usr/bin`
directory inside it; the role appends `usr/bin` itself. The expected layout
is:

```text
<root>/usr/bin/swiftc
<root>/usr/lib/swift/
<root>/usr/lib/swift_static/
```

A wrong path fails fast, before any build work happens: the role checks that
`<path>/usr/bin/swiftc` exists and is executable and stops with an actionable
message naming the path it checked if it does not. Leaving
`hazkey_swift_toolchain_path` empty selects the system Swift instead. Confirm
a valid toolchain actually links statically with:

```bash
ansible-playbook playbooks/hazkey.yml -i localhost, -c local --check -v \
  -e hazkey_swift_toolchain_path=/home/<user>/swift-toolchain \
  | grep -o 'STATIC_STDLIB=[A-Z]*'
```

`STATIC_STDLIB=ON` means the toolchain was picked up and `hazkey-server`
will be linked statically. `STATIC_STDLIB=OFF` on a valid explicit toolchain
means that toolchain has no static standard library, so linking falls back
to dynamic. Changing the toolchain path changes the build signature,
so the next run rebuilds once; that is expected.

`hazkey_static_swift_stdlib` overrides the auto-detection and is rarely
needed. Forcing it to `true` without a static standard library available
makes the build fail.

The Hazkey role does not fetch or cherry-pick upstream PR #25, resolve conflicts,
or automate KDE/Fcitx5 UI and input validation.

## Tool update policy

Tools split into two groups. Development machines are kept current on
tooling that is safe to move, while anything where a version change can
break a cluster, a build, or an input method is pinned and moved
deliberately. Release lookups need network access; when one is
unavailable, the role keeps whatever is already installed and warns
instead of failing.

Follows the latest stable release:

- `fzf` (`fzf_version` pins it)
- Ollama (`ollama_version` pins it) — installed from the upstream release archive, not a remote install script; each archive's checksum is fetched from that same release's `sha256sum.txt` at install time
- Sheldon (`sheldon_version` pins it) — the installer script itself stays pinned by `sheldon_installer_commit` and verified by `sheldon_installer_checksum`; only the release tag it fetches moves, so integrity verification is unaffected
- `uv`, via the upstream `astral.sh/uv/install.sh` installer
- Node.js, via nvm at the latest LTS
- Docker Engine, from Docker's own repository
- Fedora packages, from the target release's repositories
- UDEV Gothic (`fonts_udev_gothic_version` pins it)
- HackGen (`fonts_hackgen_version` pins it)

Pinned:

- Hazkey (`hazkey_version`, `hazkey_update_repository`) — see the Hazkey section above
- Helm (`helm_version`, `helm_archive_checksums`) — installed from the release archive, not a remote install script
- k3s (`k3s_version`) — the installer is fetched from the matching release tag rather than the moving `get.k3s.io` endpoint
- GenEi Gothic (`fonts_genei_gothic_url`, `fonts_genei_gothic_checksum`) — okoneya.jp publishes no release feed for this font, and the archive has not changed since around 2016

## Notes

- macOS: the system `/bin/zsh` is used. Homebrew is installed automatically if missing for supporting tools; Docker Desktop is intentionally not installed by this repository.
- k3s is Linux-only; no implicit kind/minikube replacement is installed on macOS.
- k3s writes the kubeconfig to `{{ target_home }}/.kube/config`, owned by `target_user`, not the invoking user.
- Helm installs the Linux `amd64`/`arm64` release archive, resolved from the host architecture; other architectures fail explicitly.
- Fedora and Debian-family systems use Docker Engine from Docker's official repositories.
- `fzf` is checked out from its latest stable release tag into `~/.local/share/fzf`, with the binary linked into `~/.local/bin`; `fzf_version` pins it.
- `fzf_enable_completion` defaults to `true`. Disable it with `-e fzf_enable_completion=false`.
- Ansible-managed zsh settings live in `~/.config/zsh/ansible.zsh`. The role adds a small loader block at the start of `~/.zshrc`, so settings below that block remain user-editable and can override managed defaults.
- On Linux, `sheldon` follows the latest stable release; set `sheldon_version` to pin it. On macOS, Homebrew manages `sheldon`.
- Normal-user execution is preferred. Direct-root workstation runs require an explicit non-root `target_user`; system-only runs do not.

## Contributing

Keep OS-specific behavior inside roles where practical. Verify syntax and idempotency by running the same playbook twice; the second run should not report unnecessary changes.
