# Tool update policy TODO

## 方針

このリポジトリは開発PCを継続的に良い状態へ保つことを重視する。
そのため、すべてを固定するのではなく、ツールの性質ごとに更新方針を分ける。

### Latest stable を追従するもの

- `uv`
  - 最新stableを利用する。
  - 現在の `https://astral.sh/uv/install.sh` を使う方針は維持してよい。
  - pin漏れではなく、意図的なlatest追従であることをREADMEまたはroleコメントに明記する。
- `fzf`
  - 固定versionではなく、最新stable releaseを追従する。
  - `main` / `master` HEADは使わず、GitHubの最新stable tagを解決する。
  - インストール済みversionと最新stableが異なるときだけ更新する。
- `Sheldon`
  - 固定versionではなく、最新stable releaseを追従する。
  - development HEADは使わない。
  - インストール済みversionと最新stableが異なるときだけ更新する。
  - installerを利用する場合は取得元を明確にし、可能ならchecksum検証を維持する。
- `Node.js`
  - nvm経由で最新LTSを利用する。
- Docker Engine
  - Docker公式repositoryの最新stableを利用する。
- Fedora package群
  - 対象Fedora releaseのrepositoryで提供される最新packageを利用する。

### 明示的にpinするもの

- `Hazkey`
  - moving branch (`dev`) ではなく、VMでbuild・install・実入力・2回目 `changed=0` まで確認したknown-good commit SHAへpinする。
  - 更新時は新しいcommitをVMで検証してからpinを進める。
  - source SHAは既存のbuild signatureへ含まれているため、pin更新時だけrebuildされる設計を維持する。
- `k3s`
  - Kubernetes ecosystemとの互換性を考慮してversionをpinする。
- `Helm`
  - versionをpinする。
  - `main` branchのinstall scriptをそのまま実行する方式はやめる。

## 実装TODO

- [x] `fzf_version` の固定defaultを廃止し、latest stable tagを安全に解決する仕組みに変更する。
- [x] fzfは現在versionとlatest stableを比較し、必要なときだけ更新する。
- [x] `sheldon_version` の固定defaultを廃止し、latest stable releaseを安全に解決する仕組みに変更する。
- [x] Sheldonは現在versionとlatest stableを比較し、必要なときだけ更新する。
- [x] uvが意図的にlatest stable追従であることをREADMEまたはroleコメントへ明記する。
- [x] Node.jsは最新LTS追従を維持し、その方針を明記する。
- [x] Hazkeyの現在のVM検証済みcommit SHAを確認し、`hazkey_version` をそのSHAへpinする。
- [x] Hazkey更新手順を「新commitをVMで検証 → pin更新」に統一する。
- [x] k3sのversion変数を追加し、install時に明示versionを使用する。
- [x] Helmのversion変数を追加し、release artifactまたはversion指定可能な安全なinstall方式へ変更する。
- [x] latest追従とpinの方針をREADMEへ短くまとめる。

## Acceptance

k3sはsystemd serviceを起動するため、検証機ではあえてinstallしていない。
`--syntax-check` とcheck-modeの挙動のみ確認済みで、実installでの
idempotency確認は未実施。

- [x] latest追従対象はdevelopment HEADではなくstable releaseのみを使う。
- [x] latest追従対象は2回目のAnsible実行で不要な `changed` を出さない。
- [x] Hazkeyは同じpinで2回目 `changed=0` になる。
- [x] Hazkeyのpinを変更したときだけbuild/installが再実行される。
- [x] Helmは同じversion指定で再実行してもidempotentである（v3.21.4 → v4.2.4へ実際にupgradeし、2回目 `changed=0` を確認）。
- [ ] k3sは同じversion指定で再実行してもidempotentである（manual validation required: systemd serviceを起動するため検証機では未install）。

## Manual validation never performed on real hardware

The Fedora LXC/graphics refactor (`docs/history/fedora44_lxc_refactor.md`),
the generic Fedora baseline refactor (`docs/history/generic_fedora_refactor.md`),
and the generic zsh entrypoint refactor
(`docs/history/generic_zsh_playbook_refactor.md`) all shipped, but the
following acceptance steps they call for were never actually run on real
hardware and remain genuinely open:

- The Fedora Workstation and LXC entrypoints (`playbooks/fedora_workstation.yml`,
  `playbooks/fedora_system.yml`, `playbooks/fedora_lxc.yml`, and their
  `fedora44_*` compatibility wrappers) have not been run twice on real Fedora
  hardware/LXC guests with recaps compared to confirm second-run idempotency.
- k3s has not actually been installed and re-run to confirm idempotency (see
  the open item directly above; the verification machine never started the
  systemd service).
- The Ubuntu/Debian and macOS paths of `playbooks/zsh.yml` have not been
  exercised on real hosts (package availability, login shell, direnv,
  headless override, legacy `.zshrc` migration).
- AMD/Vulkan graphics validation (per-device amdgpu/RADV checks, non-AMD and
  mixed-GPU layouts) has not been confirmed on real AMD or mixed-GPU
  hardware.
- Hazkey has not been rebuilt against a real external static Swift toolchain
  and confirmed to link statically and work end-to-end as an installed input
  method.
