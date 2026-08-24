# Manual validation status

CI covers YAML formatting, ansible-lint and syntax checks only. Nothing
below is proven by a green CI run. Items under "Open" have not been
confirmed on real hardware or in a VM; items under "Confirmed" have.

Which tools follow latest stable and which are pinned is documented in
README.md under "Tool update policy".

## Open

- k3s has not actually been installed and re-run to confirm idempotency.
  The verification machine never started the systemd service, so only
  `--syntax-check` and check-mode behaviour were exercised.
- The Fedora Workstation and LXC entrypoints
  (`playbooks/fedora_workstation.yml`, `playbooks/fedora_system.yml`,
  `playbooks/fedora_lxc.yml`, and their `fedora44_*` compatibility
  wrappers) have not been run twice in full on real Fedora hardware or LXC
  guests with recaps compared to confirm second-run idempotency. Only the
  `snapper` tag of `playbooks/fedora_system.yml` has been exercised this
  way; every other role in those entrypoints remains unverified.
- The Ubuntu/Debian and macOS paths of `playbooks/zsh.yml` have not been
  exercised on real hosts: package availability, login shell, direnv,
  headless override, legacy `.zshrc` migration.
- AMD/Vulkan graphics validation has not been confirmed on real AMD or
  mixed-GPU hardware: per-device amdgpu/RADV checks, non-AMD layouts,
  mixed-GPU layouts.
- Hazkey has not been rebuilt against a real external static Swift
  toolchain and confirmed to link statically and work end to end as an
  installed input method.
- Whether `snapper-cleanup` actually prunes snapshots once the configured
  retention limits are exceeded. `snapper-timeline.timer` firing and
  creating a real timeline snapshot is now confirmed below; cleanup has not
  been observed pruning anything yet, because retention (`hourly` limit of
  6) has not been exceeded since the timer started running.
- Actual success of baseline snapshot creation via
  `fedora_snapper_create_baseline_snapshot=true`.
- Recovery from a partially applied `fedora_snapper` run (for example a
  `/.snapshots` subvolume left without a configuration file). The two
  guard assertions that detect these states have passed on a healthy host,
  but neither has been triggered against a genuinely inconsistent one.
- Whether the role fails closed when a target mountpoint is a symlink has
  not been verified. The non-empty case is confirmed below; the symlink
  branch of the same assertion has never been triggered.
- Whether the role can stop a *running* Docker or containerd unit and bring
  it back afterwards has not been verified. Both real runs happened to
  start from already-stopped units, so only the "was not running, so do not
  start it" branch was exercised.
- Whether services stopped on a failed subvolume setup are actually
  restarted by the `rescue` block added around the mount/permission/SELinux
  steps in `tasks/subvolume.yml` has not been verified.
- Behavior on a host with SELinux disabled, where `restorecon` is now
  skipped, has not been verified.
- Two parts of the `fonts` role remain unverified:
  - The fallback path where a GitHub release lookup fails and the role keeps
    whatever is already installed, warning instead of failing. Successful
    lookups are confirmed below; the unavailable-network branch is not.
  - Replacing an already-installed archive font when its resolved version
    changes. Only the first install and the unchanged re-run have been
    exercised.
  - The rejecting side of the `playbooks/fonts.yml` OS assert. It has been
    satisfied on Fedora and on Ubuntu 24.04, but never triggered against an
    unsupported distribution or an older Ubuntu.

## Confirmed

- The `fonts` role (renamed from its earlier Fedora-only name) was run on
  real Fedora 44 hardware with `fonts_install_archives=true`. The 23 packaged
  fonts installed, and all three archive fonts landed flattened under
  `/usr/local/share/fonts/`: UDEV Gothic 16 files, HackGen 8, GenEi Gothic
  21, each with a `.fedora-fonts-version` marker (now named `.fonts-version`
  after the rename) recording `v2.2.0`, `v2.10.0` and `1.1a` respectively.
- The GitHub release lookups resolve real tags: the markers hold the tags
  the API returned, not values written into the role.
- A second run reports no changes, so the marker comparison skips both the
  download and the extraction.
- `fc-cache` makes the result usable: `fc-list :lang=ja` reports 145 fonts
  and the new families (BIZ UD, IPAex, GenEi Gothic, HackGen, UDEV Gothic,
  Motoya, Hanazono) are selectable by name.
- The Debian-family path works end to end, exercised in a clean
  `ubuntu:24.04` container under Podman. All ten `fonts_packages_debian`
  entries install (metapackages expand to fourteen dpkg entries), the three
  archive fonts land with the same file counts and markers as on Fedora, and
  the resolved GitHub tags again come from the API rather than the role. A
  second run reports `changed=0`. The container also ran Ansible 2.16.3,
  older than the workstation's 2.20.7.
- The `restorecon` handler is skipped on Debian-family hosts, and the
  `playbooks/fonts.yml` OS assert passes on both Fedora and Ubuntu 24.04.
- That container run first exposed a real defect: on a host without
  `fontconfig`, the `fc-cache` handler failed with `No such file or
  directory: fc-cache`. Fedora desktops always carry it, so no amount of
  testing there would have surfaced this. `fontconfig` is now installed
  alongside `unzip` as an archive-path prerequisite rather than being listed
  with the font packages, so a host that installs no font packages at all
  still gets it. Re-verified from a fresh container where `fc-cache` did not
  initially exist.
- GenEi Gothic's archive stores its readme filenames in Shift-JIS, which
  breaks `unarchive`: the module lists members through Python's `ZipFile`,
  decodes them as CP437, and then fails reading the SELinux context of a
  path that does not exist under that name. Passing `-O CP932` through
  `extra_opts` makes it worse, because only the extraction side changes and
  the two names then disagree. Restricting the task with
  `include: ["*.ttf", "*.otf"]` avoids the files entirely and is confirmed
  working.
- Helm is idempotent across a real v3.21.4 to v4.2.4 upgrade, with the
  second run reporting `changed=0`.
- Hazkey reaches `changed=0` on a second run with the same pin, and
  rebuilds only when the pin moves.
- Tools that follow latest stable resolve stable release tags rather than
  development HEAD, and do not report spurious `changed` on a second run.
- `snapper set-config KEY=VAL` writes the value back to the configuration
  file as `KEY="VALUE"`, and reapplying the same value a second time does
  not change the file (identical sha256 before and after).
- The `fedora_snapper` retention diff logic was exercised against a
  configuration file that snapper itself had written, on a temporary
  snapper configuration created on a loopback-mounted Btrfs filesystem:
  the first apply changed 6 keys, and a second apply against the same
  desired values changed 0 keys.
- A real `snapper -c root create-config /` against an actual `/` (a
  `subvol=root` on a LUKS-backed Btrfs filesystem) succeeded. `/.snapshots`
  was created as a nested subvolume with the SELinux label
  `snapperd_data_t`, and `/etc/fstab` was left unchanged.
- `snapper-timeline.timer` and `snapper-cleanup.timer` were enabled and
  started by the role, and both report `enabled` and `active` afterwards.
  Both survived a real reboot and came back `enabled` and `active`.
  `snapper-timeline.service` ran at its scheduled 23:00:02 and logged
  `Running timeline for 'root'` to the journal, and `snapper -c root list`
  afterwards showed snapshot #1 actually present, with type `single`,
  cleanup algorithm `timeline`, and description `timeline`.
- systemd-helper picks up the root configuration even though the role
  leaves `/etc/sysconfig/snapper`'s `SNAPPER_CONFIGS=""` unchanged:
  `snapper-cleanup.service` logged `Running cleanup for 'root'` and exited
  with status 0.
- Two consecutive real runs of `fedora_snapper` on Fedora 44 hardware
  (`--tags snapper -e fedora_snapper_create_root_config=true`) reported
  `changed=3` then `changed=0`, both with `failed=0`. The first run applied
  exactly the 6 retention keys predicted by the loopback test
  (`TIMELINE_LIMIT_HOURLY`, `_DAILY`, `_WEEKLY`, `_MONTHLY`, `_YEARLY` and
  `NUMBER_LIMIT`) and skipped the other 4.
- A `--check` run of `fedora_snapper` on a host with no root configuration
  completes with `changed=0` and no undefined-variable failures. Note that
  it skips the retention and timer tasks, because they are gated on a
  configuration that check mode cannot create.
- `fedora_btrfs_layout` was run on real Fedora 44 hardware with only
  `fedora_btrfs_layout_libvirt_images=true`. `/var/lib/libvirt/images` was
  created as a top-level subvolume (ID 260, top level 5) and mounted with
  the same UUID as `/` and `FSROOT` `/libvirt-images`.
- The generated `/etc/fstab` line for that mount is
  `subvol=libvirt-images,compress=zstd:1,x-systemd.device-timeout=0`, with
  no kernel-added runtime-only mount options mixed into it.
- `findmnt --verify` reported 0 parse errors and 0 errors against the
  resulting `/etc/fstab`.
- The SELinux label on `/var/lib/libvirt/images` came out as `virt_image_t`
  after the run, confirming `restorecon` relabeled it correctly.
- The generated `/etc/fstab` entry survives a reboot. After restarting the
  host, `/var/lib/libvirt/images` is mounted again from `/etc/fstab`: the
  generated `var-lib-libvirt-images.mount` unit reports `active` with
  `SourcePath=/etc/fstab`, `FSROOT` `/libvirt-images` and `subvolid=260`.
- The declared mode `0711` and the `virt_image_t` label both persist across
  that reboot.
- A second run of `fedora_btrfs_layout` with the same options reports
  `changed=0`, including after the permission-convergence and the
  handler-based daemon-reload were added. The convergence task reports no
  change against an already-correct mountpoint, and the `flush_handlers`
  step is a no-op when nothing notified it.
- The permission-convergence task also corrects drift rather than only
  holding a correct value: with `/var/lib/libvirt/images` manually set to
  `0755`, a run reports `changed=1` and leaves it back at its declared
  `0711`.
- Five more subvolumes were applied on the same Fedora 44 host:
  `/var/lib/docker`, `/var/lib/containerd`, `/var/lib/containers/storage`
  and `~/.local/share/containers/storage`, alongside the existing
  `/var/lib/libvirt/images`. All five mount from `/etc/fstab` after a
  reboot, including the rootless Podman one nested under the separate
  `/home` subvolume, which systemd orders after `/home` on its own.
- Every mountpoint converged to its declared mode and stayed there across
  that reboot: `/var/lib/docker` `0710`, `/var/lib/containerd` `0700`,
  `/var/lib/containers/storage` `0755`, `/var/lib/libvirt/images` `0711`,
  and `~/.local/share/containers/storage` `0700` owned by the target user.
  The last one is the only entry that declares `owner`/`group`, so it is
  also the only exercise of that path.
- SELinux labels came back correct for all of them after `restorecon`:
  `container_var_lib_t` for the three under `/var/lib`, `data_home_t` for
  the rootless Podman store.
- Both container runtimes work on the new layout: `docker run --rm
  hello-world` pulls from Docker Hub and runs, and `podman run --rm
  hello-world` does the same rootless, both after a reboot.
- The role fails closed on a non-empty mountpoint during a real run, not
  only under `--check`: two separate runs stopped at the assertion with
  `changed=0`, left the target untouched, and still unmounted the
  `subvolid=5` helper through the `always` block.
- The role does not start services it did not stop: with `podman.socket`
  inactive and disabled beforehand, applying `podman-root` left it that
  way.
- The conditional `target_user` resolution behaves in both directions. It
  is skipped when `fedora_btrfs_layout_podman_rootless` is false, and when
  true it resolves the login account and places the mountpoint under that
  account's home without the username appearing anywhere in the role.
