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
- Whether the role fails closed as intended when a target mountpoint is
  non-empty, including the case where the mountpoint is a symlink, has not
  been verified.
- Whether stopping and restarting the Docker and containerd units around
  the remount works correctly has not been verified.
- Whether services stopped on a failed subvolume setup are actually
  restarted by the `rescue` block added around the mount/permission/SELinux
  steps in `tasks/subvolume.yml` has not been verified.
- Behavior on a host with SELinux disabled, where `restorecon` is now
  skipped, has not been verified.

## Confirmed

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
