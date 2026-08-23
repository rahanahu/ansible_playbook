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
