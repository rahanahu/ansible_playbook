# Generic zsh playbook implementation TODO

## 評価

実装可能。既存の `target_user` role と `zsh` role には必要な機能がほぼ揃っており、
中心作業は薄い `playbooks/zsh.yml` entrypoint、OS/root preflight、README、受け入れ
テストの追加になる。

既存実装で再利用できるもの:

- passwd / Directory Service による target user と home の安全な解決;
- Linux の generic package 経路による zsh、git、zoxide、direnv、wl-clipboard;
- macOS の既存 Homebrew 経路（zsh本体を除く依存ツール）;
- pinned fzf / Sheldon、managed `.zshrc`、legacy migration;
- root の login shell を変更しない role 内の強制ガード;
- direnv hook を root では描画しない template 内の強制ガード;
- `system` / `user` の既存 task tags。

難易度は低～中。ただし以下は実装前に仕様を固定する。

## 推奨する設計判断

- `zsh.yml` は zsh 専用 entrypoint とし、Docker、uv、nvm、Fedora desktop role は
  呼ばない。
- zoxide 本体は Linux のOS package managerまたはmacOSのHomebrewで導入し、managed
  `.zshrc` の `zoxide init zsh` で初期化する。Sheldonにはzoxideの導入を担わせない。
- direct root は暗黙選択しない。Linux で `-e target_user=root` が明示された場合だけ
  root を許可する。
- Linux通常ユーザーは login shell、direnv package/hook、desktop package を既定で
  有効にする。headless 用に `-e zsh_install_desktop_packages=false` を維持する。
- macOSではOS標準の `/bin/zsh` を利用し、Homebrew版zshを導入しない。login shellも
  変更せず、git、zoxide、direnv、Sheldonなど必要な依存ツールだけをHomebrewで導入する。
- macOSはAnsible実行ユーザー自身だけを正式対象とする。別ユーザー、sudo/root経由、
  `target_user=root` はmutation前に拒否し、通常ユーザーとして再実行するよう案内する。
- graphical session の自動検出で `wl-clipboard` を切り替えない。将来 desktop として
  使う SSH 実行との区別が不安定なため、通常ユーザー ON、root OFF、明示 override を
  予測可能な契約にする。
- macOS の通常実行ユーザー自身を正式対象にする。macOS の別ユーザー/root対応は
  Homebrew ownership/実行ユーザー設計を別途行う場合だけ追加する。
- Ubuntu / Debian / Fedora だけを Linux の正式対象として assert し、別 distribution
  で偶然 generic package task が動く状態を正式サポートとみなさない。
- Ubuntu / Debian の最小対応 release は実機テスト結果に基づいて README に記載する。
  特に `zoxide` package の有無を確認する。
- `zsh` tag は role 全体、`system` と `user` は既存task境界として機能させる。
  dynamic include には `zsh` だけを `apply` し、include 自体を
  `[zsh, system, user]` で選択可能にする。3 tag 全部を `apply` して境界を潰さない。
- 新規・変更箇所では top-level injected facts ではなく `ansible_facts[...]` を使う。
  zsh role 全体の fact 参照移行は、この変更に必要な範囲を超えるなら別TODOとする。

## 実装 TODO

- [x] `playbooks/zsh.yml` を追加し、localhost/local connection と fact gathering を定義。
- [x] `target_user` role を `always` tag で先に実行し、明示rootを解決可能にする。
- [x] Ubuntu、Debian、Fedora、Darwin 以外を mutation 前に停止する preflight を追加。
- [x] Darwinではtarget解決前に `ansible_facts['user_id'] != 'root'` を検証し、さらに
      解決後に `target_user == ansible_facts['user_id']` をmutation前に検証する。
      別ユーザー、sudo/root経由、root指定は理由と再実行例付きで停止。
- [x] profile capability を entrypoint で解決し、通常ユーザー/root差分を role vars に渡す。
- [x] `zsh_install_system_packages: true` を明示し、新規環境で単独完結させる。
- [x] `zsh/tasks/mac.yml` のHomebrew package一覧から `zsh` を除外する。
- [x] macOSでは `zsh_manage_login_shell: false` をentrypointから明示する。
- [x] root では login shell、direnv package/hook、desktop package をすべて無効化。
- [x] 通常ユーザーでは desktop package の既定値を保ち、extra var でOFF可能にする。
- [x] `zsh` / `system` / `user` tag が個別に意味どおり動く include/tag構成を追加。
- [ ] package failure で distribution、target user、対象packageが判別できることを確認。
      generic package module の標準エラーで不足する場合だけ補助メッセージを追加。
- [x] `user.yml`、`dev_pc.yml`、Fedora Workstation/LXC wrapper の責務と呼出しを変更しない。
- [x] README の entrypoint 表に `zsh.yml` と `dev_pc.yml` の用途を追加。
- [x] README に Linux通常ユーザー、macOS自身、Linux root、Linux別ユーザー、
      headless override の例を追加。
- [x] README に root policyと、macOSは通常ユーザー自身のみ対応することを明記。

## 静的検証 TODO

- [ ] `ansible-playbook playbooks/zsh.yml -i localhost, -c local --syntax-check`。
- [ ] `--list-tags` で `always`、`zsh`、`system`、`user` を確認。
- [ ] `--tags system` で target解決とsystem taskだけが実行対象になることを確認。
- [ ] `--tags user` で target解決とuser taskだけが実行対象になることを確認。
- [ ] `--tags zsh` でsystem/user双方が実行対象になることを確認。
- [x] YAML header、tab、trailing whitespace、Markdown fence、`git diff --check` を確認。
- [ ] Ansible/YAML lintを実行。
- [x] `zsh.yml` が docker、uv、nvm、Fedora desktop rolesを参照しないことを確認。

## 実機受け入れ TODO

- [ ] Ubuntu通常ユーザーで初回実行し、zsh/fzf/zoxide/Sheldon/direnvとlogin shellを確認。
- [ ] Debian通常ユーザーで同じ確認を行い、対応releaseとpackage availabilityを記録。
- [ ] Fedora通常ユーザーで初回実行し、Workstation固有roleが動かないことを確認。
- [ ] Linuxで `target_user=root` を明示し、managed zshが使え、passwd上のroot shellが
      bashのまま、direnv hookとwl-clipboardが無いことを確認。
- [ ] direct rootでtarget未指定の場合、rootへ暗黙fallbackせず案内付きで停止することを確認。
- [ ] macOS通常ユーザーでOS標準 `/bin/zsh` と既存Homebrew依存ツール経路を使って
      成功し、Homebrew版zshを追加せずlogin shellも変更しないことを確認。
- [ ] macOSの別ユーザー、sudo/root経由、root指定がmutation前に明確に失敗することを確認。
- [ ] Linuxで別の既存ユーザーを指定し、passwd由来のhomeだけが変更されることを確認。
- [ ] headless Linuxで `-e zsh_install_desktop_packages=false` が機能することを確認。
- [ ] 各正式対象で2回実行し、2回目の不要な `changed` がないことを確認。
- [ ] 既存 `.zshrc` のloaderより後のユーザー編集が維持されることを確認。
- [ ] 既知legacy managed `.zshrc` の移行と、未知checksumの安全停止を確認。
- [ ] `user.yml`、`dev_pc.yml`、Fedora Workstation/LXCの代表コマンドを回帰確認。

## 完了条件

- Ubuntu / Debian / Fedora / macOS の通常ユーザーが `zsh.yml` だけでmanaged zsh環境を
  構築できる。
- Linux rootは明示指定時だけ構築でき、login shellとdirenvの安全条件を破れない。
- system/user/zsh tags、既存customization、idempotency、既存entrypointが維持される。
- 未検証のOS/root組み合わせを、成功したように見せずmutation前に明確に拒否する。

## 対象外

- Docker、uv、nvm、desktop/graphics/gaming/Fcitx/Hazkey/Snapper。
- Fedora release全体の汎用化。
- macOSの別ユーザー/root向けHomebrew bootstrapと所有権管理。
- zsh role全体の大規模なfact記法リファクタ。
