# Generic Fedora baseline refactor TODO

## 評価

実装可能。設計の方向性も妥当。ただしこれは単純なファイルrenameではなく、release
policy、entrypoint互換、public variable互換、role間で共有するfactの同時移行を含む。
難易度は中～高で、Fedora 44の回帰確認を挟みながら段階的に進めるべき変更である。

現行リポジトリの棚卸し結果:

- rename対象は9 role;
- 新設対象は3つのrelease非依存entrypoint;
- tracked実装・READMEには約60種類、約198箇所の `fedora44_*` 参照がある;
- Workstationはsystem、user、session/root-zshの3 playにまたがる;
- `fedora44_has_user_session` など、一つのplay/roleで生成して別roleで利用するfactがある;
- RPM Fusion URLはすでにdetected Fedora major releaseを利用している;
- 現在のrelease判定はFedora 44完全一致が既定で、LXCとWorkstationから共有されている。

## 実装前に固定する判断

- Fedora 44を唯一のvalidated release、44をminimum releaseとして開始する。
- 44より新しいreleaseは既定でwarningを出して続行し、実際のpackage/task failureは
  隠さない。44未満とnon-Fedoraはmutation前に停止する。
- `fedora_strict_release_check=true` のときはvalidated list外を停止する。
- release判定値はintegerへ正規化し、validated listもinteger比較する。
- Workstation/LXCのenvironment guardとsecurity policyは変更しない。
- Fedora release validationをenvironment guardより先に実行し、non-Fedora containerへ
  Fedora LXC entrypointを誤案内しないようにする。
- release policyの機能変更と大量renameを同じ未検証stepにまとめない。まず既存role名の
  ままrelease policyを検証し、その後roleを一つずつrenameする。
- 新しいentrypointをcanonical implementationとし、旧 `fedora44_*` entrypointは
  `import_playbook` だけのcompatibility wrapperにする。
- 旧role directoryのcompatibility shimは作らない。リポジトリ内参照をすべて更新し、
  roleを直接呼ぶ外部利用者についてはmigration noteで扱う。
- public feature switchは1 compatibility cycleだけ旧名を受理する。package list、register、
  loop variable、内部factはaliasせずatomicにrenameする。
- old/new variable判定はrole defaultsまたはentrypoint preflightで一度だけcanonical化し、
  task本体は `fedora_*` だけを参照する。
- new variableとold variableが両方指定された場合はnew variableを優先する。
- zsh、Docker、GPU vendor対応、Workstation/LXC profileの責務は拡張しない。
- top-level injected factの全体移行は別refactorとする。今回新設・変更するrelease logicは
  `ansible_facts[...]` を利用する。

## 注意が必要な互換性

### Release variables

旧release variablesは新policyと意味が一対一ではないため、単純aliasにしない。

- `fedora44_required_release` が明示された場合は、legacy compatibility入力として
  minimumとvalidated singletonへ正規化する。
- `fedora44_allow_other_release=false` が明示された場合はstrict mode相当、`true` は
  non-strict相当として扱う。
- legacy release variableが一つでも明示された場合の組合せをテストし、曖昧な組合せは
  silent fallbackではなく明確にfailさせる。
- 新しい `fedora_minimum_release`、`fedora_validated_releases`、
  `fedora_strict_release_check` が指定されていれば常にそちらを優先する。

### Snapper baseline description

`fedora44_snapper_baseline_description` の既定値は現在
`fedora44-dev-baseline`。これを単純に `fedora-dev-baseline` へ変更すると、既存snapshotを
見つけられず新しいbaseline snapshotを作る可能性がある。今回のrefactorでは次のどちらかを
選び、実機確認する。

- legacy descriptionを既定のまま維持する; または
- new/legacy両descriptionを既存baselineとして認識し、新規環境だけnew descriptionを使う。

推奨は後者。ただしsnapshot作成は従来どおり明示opt-inのままとする。

## Canonical role mapping

| Current | Canonical |
|---|---|
| `fedora44_preflight` | `fedora_preflight` |
| `fedora44_lxc_preflight` | `fedora_lxc_preflight` |
| `fedora44_graphics` | `fedora_graphics` |
| `fedora44_repositories` | `fedora_repositories` |
| `fedora44_desktop` | `fedora_desktop` |
| `fedora44_cli_tools` | `fedora_cli_tools` |
| `fedora44_snapper` | `fedora_snapper` |
| `fedora44_gaming` | `fedora_gaming` |
| `fedora44_input` | `fedora_input` |

## Phase 0: regression baseline

- [ ] 現在のFedora 44 Workstation、system-only、LXCのsyntax check結果を保存。
- [ ] 現在のFedora 44 WorkstationとLXCを各2回実行し、recapを保存。
- [ ] Workstation/LXCのpackage、service、target user、root shell、direnv、GPU結果を記録。
- [ ] `git grep` でrole名、public variable、内部fact、runtime messageを分類した移行表を作る。
- [ ] compatibility対象のpublic variablesを確定する。

## Phase 1: release policyの共通化

- [x] 既存preflight role内でcanonical release variablesを導入する。
- [x] defaultをminimum `44`、validated `[44]`、strict `false` とする。
- [x] non-Fedora、minimum未満、validated、unvalidated、strictの判定を分離する。
- [x] warningは `debug` 等で表示し、`changed` を発生させない。
- [x] warningにdetected release、validated list、strict modeの案内を含める。
- [x] Fedora release checkをWorkstation/LXC双方の最初のplatform mutation前に実行する。
- [x] Workstation preflightからrelease checkの重複実行を除去する。
- [x] LXC preflightをLXC/systemd判定だけに保つ。
- [x] legacy release variablesをcanonical値へ一度だけ正規化する。
- [x] neither/old-only/new-only/both-conflictingのvariable precedenceをコード上で確認する。
- [ ] Fedora 43、44、45相当、strict、non-Fedoraの判定fixtureまたは実機確認を行う。

## Phase 2: roleと内部identifierのrename

- [ ] 9 role directoryを `git mv` し、role単位で参照更新とsyntax checkを行う。
- [ ] `import_role` / `include_role` の旧role名をすべてcanonical名へ変更する。
- [ ] `user.yml` のgaming/input role参照を更新し、OS汎用の責務を維持する。
- [ ] defaults、tasks、register、set_fact、loop variableを `fedora_*` へrenameする。
- [ ] graphicsのdetect → AMD → session間で共有するfactをatomicにrenameする。
- [ ] preflight session factとgraphics session consumerを同じ変更でrenameする。
- [ ] LXC systemd state、Snapper state、desktop package probeなどregister参照を揃える。
- [ ] task名、fail message、entrypoint案内から不要なrelease番号を除去する。
- [ ] RPM Fusion等のrelease URLがdetected major releaseを使い続けることを確認する。
- [ ] tagsにrelease番号を追加せず、既存tag契約を維持する。

## Phase 3: public variable migration

- [ ] README公開済みfeature switchesへcanonical `fedora_*` 名を定義する。
- [ ] role defaultsに旧public variableからcanonical variableへのcompatibility入力を置く。
- [ ] CLI、desktop、graphics、gaming、Snapper、LXC、root-zshのswitchを移行する。
- [ ] package list variablesはcanonical名へ単純移行し、旧aliasを増やさない。
- [ ] 内部fact/registerには旧aliasを作らない。
- [ ] old/new両方指定時にnew valueが勝つテストを各roleで最低1件行う。
- [ ] deprecation対象と削除予定時期をREADMEへ記載する。
- [ ] Snapper legacy baseline descriptionの二重snapshot防止を検証する。

## Phase 4: canonical entrypointとcompatibility wrapper

- [ ] `playbooks/fedora_system.yml` をcanonical Workstation system entrypointとして追加。
- [ ] `playbooks/fedora_workstation.yml` を追加し、canonical system/user/session/root-zshを構成。
- [ ] `playbooks/fedora_lxc.yml` を追加し、共通release policyとLXC境界を維持する。
- [ ] canonical playbooks内ではcanonical role/variable/pathだけを利用する。
- [ ] `fedora44_workstation.yml` をcanonical Workstationの薄いimport wrapperへ変更。
- [ ] `fedora44_system.yml` をcanonical systemの薄いimport wrapperへ変更。
- [ ] `fedora44_lxc.yml` をcanonical LXCの薄いimport wrapperへ変更。
- [ ] old wrapperへdeprecation commentを追加し、実処理を重複させない。
- [ ] old wrapperから渡したextra varsがimport先で同じ結果になることを確認する。
- [ ] zsh.yml、dev_pc.yml、hazkey.ymlの責務と動作を変更しない。

## Phase 5: READMEと検索

- [ ] README入口表と通常例をcanonical entrypointへ変更する。
- [ ] Fedora 44 validated、newer Fedora best effort、minimum、strict policyを明記する。
- [ ] Workstation/LXC/security/root profile/GPU/Snapper policyが不変であることを明記する。
- [ ] canonical public variablesの例へ更新する。
- [ ] 旧entrypointと旧public variableのcompatibility期間をまとめる。
- [ ] `git grep -n 'fedora44_' -- playbooks README.md` で残存箇所を分類する。
- [ ] old role path/import/includeが残っていないことを確認する。
- [ ] instruction、runbook、TODO内の歴史的記述を実装残存として誤検出しない。

許容する `fedora44_*` 残存:

- 3つのcompatibility wrapper名とdeprecation comment;
- compatibility入力として残す旧public variables;
- READMEのmigration/compatibility section;
- Snapper legacy baseline識別子;
- migration TODOや履歴資料。

## Static acceptance

- [ ] canonical/legacyのWorkstation、system、LXCすべてでsyntax check。
- [ ] zsh.yml、user.yml、dev_pc.yml、hazkey.ymlのsyntax check。
- [ ] canonical/legacy entrypointで `--list-tags` が同等であることを確認。
- [ ] `--tags system/user/preflight/graphics/desktop/cli/gaming/input/snapper/session` を確認。
- [ ] role/task/include/import参照がすべて存在することを機械確認。
- [ ] YAML/Ansible lint、Markdown fence、tab、trailing whitespace、`git diff --check`。
- [ ] warning/assert/fail tasksがidempotency上の `changed` を出さないことを確認。

## Fedora 44 regression acceptance

- [ ] canonical Workstationを通常ユーザーから実行し、従来機能を確認。
- [ ] canonical system-onlyをdirect rootで実行し、target user不要を確認。
- [ ] canonical LXCをdirect rootで実行し、server profileとroot bashを確認。
- [ ] legacy wrapper 3種で同じ結果になることを確認。
- [ ] Workstation root-safe、LXC server、通常user workstation profileを維持。
- [ ] SELinux/firewalld要件がWorkstationだけに適用されることを確認。
- [ ] WorkstationをLXCで、LXCをbare metal/KVMで実行し、canonical代替pathを案内して停止。
- [ ] AMDあり/なし/mixed GPUの既存判定を回帰確認。
- [ ] Snapper config/snapshotがopt-inのままで、既存baselineを重複作成しないことを確認。
- [ ] canonicalとlegacyを各2回実行し、2回目の不要なchangedがないことを確認。

推奨コマンド:

```bash
ansible-playbook playbooks/fedora_workstation.yml \
  -i localhost, -c local -K --syntax-check
ansible-playbook playbooks/fedora_system.yml \
  -i localhost, -c local --syntax-check
ansible-playbook playbooks/fedora_lxc.yml \
  -i localhost, -c local --syntax-check
```

## Newer/negative release acceptance

- [ ] テスト可能なFedora 45以降でwarning後にbaselineが続行することを確認。
- [ ] 同じreleaseでstrict modeがmutation前にfailすることを確認。
- [ ] Fedora 43相当がminimum release message付きでfailすることを確認。
- [ ] Ubuntu等でFedora専用playbookであることを明示してfailすることを確認。
- [ ] 未検証releaseで存在しないpackageがあれば通常のtask failureになることを確認。
- [ ] package failureへ `ignore_errors` を追加していないことを確認。

## 推奨コミット分割

1. release policyとcompatibility normalization;
2. role/内部identifier rename;
3. canonical entrypointとlegacy wrapper;
4. public variable migrationとSnapper互換;
5. README、search cleanup、acceptance記録。

各commitでsyntax check可能な状態を保ち、Fedora 44のsystem/LXC境界を壊したまま次へ
進まない。

## 対象外

- Fedora以外のdistribution対応。
- 全Fedora releaseの完全サポート宣言。
- Fedora 45/46を想像した空のrelease別task/vars。
- package failureの無視や自動代替package推測。
- NVIDIA/Intel driver管理やAMD機能拡張。
- Workstation/LXC security policyの統合。
- zsh.ymlやuser profile責務の拡張。
- 旧role directory shimの無期限維持。
- top-level injected facts全体の一括移行。
