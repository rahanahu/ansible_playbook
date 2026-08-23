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

- [ ] `fzf_version` の固定defaultを廃止し、latest stable tagを安全に解決する仕組みに変更する。
- [ ] fzfは現在versionとlatest stableを比較し、必要なときだけ更新する。
- [ ] `sheldon_version` の固定defaultを廃止し、latest stable releaseを安全に解決する仕組みに変更する。
- [ ] Sheldonは現在versionとlatest stableを比較し、必要なときだけ更新する。
- [ ] uvが意図的にlatest stable追従であることをREADMEまたはroleコメントへ明記する。
- [ ] Node.jsは最新LTS追従を維持し、その方針を明記する。
- [ ] Hazkeyの現在のVM検証済みcommit SHAを確認し、`hazkey_version` をそのSHAへpinする。
- [ ] Hazkey更新手順を「新commitをVMで検証 → pin更新」に統一する。
- [ ] k3sのversion変数を追加し、install時に明示versionを使用する。
- [ ] Helmのversion変数を追加し、release artifactまたはversion指定可能な安全なinstall方式へ変更する。
- [ ] latest追従とpinの方針をREADMEへ短くまとめる。

## Acceptance

- [ ] latest追従対象はdevelopment HEADではなくstable releaseのみを使う。
- [ ] latest追従対象は2回目のAnsible実行で不要な `changed` を出さない。
- [ ] Hazkeyは同じpinで2回目 `changed=0` になる。
- [ ] Hazkeyのpinを変更したときだけbuild/installが再実行される。
- [ ] k3s/Helmは同じversion指定で再実行してもidempotentである。
