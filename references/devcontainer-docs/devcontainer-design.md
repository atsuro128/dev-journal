# DevContainer セキュア構成テンプレート

日付: 2026-03-25
対象: `C:\Users\atsur\Desktop\root-project\.devcontainer`
目的: 現在採用している DevContainer の構成、制約、運用上の注意点を記録し、ローカル開発で採用すべきセキュリティ基準をテンプレート化する。

## 1. 設計要約

この DevContainer は、通常の開発環境に加えて、外向き通信を制御するための明示プロキシと default-deny firewall を組み合わせた構成を採用している。
加えて、コンテナ権限を最小化し、取得ツールの更新導線を制御することで、開発利便性を維持しながら攻撃面を狭める。

要点は以下のとおり。

- 開発ユーザーは `node`
- workspace は `/root-project` に bind mount
- Claude、Codex、GitHub CLI の設定は named volume に保持
- 全外向き通信は `127.0.0.1:3127` の Squid explicit proxy を前提にする
- proxy を通らない direct outbound は `iptables` で遮断する
- Squid 側では allowlist に載ったホストだけを許可する
- 実行時 capability は `NET_ADMIN` のみに絞る
- `seccomp=unconfined` や `apparmor=unconfined` は使わない
- `no-new-privileges` を有効化して特権昇格を抑止する
- ベースイメージや主要ツールは固定を優先し、CLI も build 時に固定版を導入する
- コンテナ起動時に named volume の所有権を補正して、CLI 設定読込エラーを回避する

## 2. 対象ファイル

現在設計の中核は以下のファイルで構成される。

- `.devcontainer/devcontainer.json`
- `.devcontainer/Dockerfile`
- `.devcontainer/init-devcontainer.sh`
- `.devcontainer/devcontainer-entrypoint.sh`
- `.devcontainer/init-firewall.sh`
- `.devcontainer/announce-egress-status.sh`
- `.devcontainer/setup-shell-aliases.sh`
- `.devcontainer/render-squid-config.sh`
- `.devcontainer/verify-egress.sh`
- `.devcontainer/proxy-allowlist.txt`

## 3. 全体構成

```mermaid
flowchart LR
    Host[Windows / Docker Host] --> Container[DevContainer]
    Container --> Workspace[/root-project]
    Container --> ClaudeVol[/home/node/.claude]
    Container --> CodexVol[/home/node/.codex]
    Container --> GhVol[/home/node/.config/gh]
    Container --> HistoryVol[/commandhistory]
    Container --> GitConfig[/home/node/.gitconfig.host]

    Tools[CLI / VS Code / Claude / Codex] --> ProxyEnv[HTTP_PROXY / HTTPS_PROXY / ALL_PROXY]
    ProxyEnv --> Squid[Squid 127.0.0.1:3127]
    Squid --> Allowlist[proxy-allowlist.txt]
    Allowlist -->|allow| Internet[Internet]
    Allowlist -->|deny| Rejected[Rejected]

    Tools -. direct outbound .-> Firewall[iptables OUTPUT DROP]
    Firewall --> Rejected
```

## 4. イメージ設計

ベースイメージは `node:20-bookworm`。

実務上の推奨は digest pin まで行うことだが、本テンプレートでは少なくとも OS 系統が変わらないタグに固定する。
Claude Code / Codex のような開発補助 CLI も、コンテナ再作成のたびにネットワーク取得が走らないよう、イメージ build 時に固定版を導入する。
更新は `Dockerfile` / `devcontainer.json` の build arg を明示的に更新して行う。

主な導入物は以下。

- 基本ツール: `git`, `gh`, `fzf`, `zsh`, `vim`, `nano`
- ネットワーク制御: `iptables`, `iproute2`, `dnsutils`, `squid-openssl`, `openssl`
- 言語・周辺: Go, Python, PostgreSQL client
- CLI: `@anthropic-ai/claude-code`, `@openai/codex`

導入物は以下の原則で扱う。

- OS、言語、lint など再現性に直結するものは可能な限りバージョンを固定する
- 日次開発の補助 CLI も再現性と build cache を優先してバージョン固定する
- `curl | sh` や `wget | sh` で導入するものは最小化し、必要時はバージョンを固定する
- 将来的には checksum 検証または digest pin に移行する

また、イメージ build 時点で以下のディレクトリを作成する。

- `/root-project`
- `/home/node/.claude`
- `/home/node/.codex`
- `/home/node/.config/gh`
- `/etc/squid/ssl`
- `/var/log/squid`
- `/var/spool/squid`

所有権は用途ごとに分離する。

- `node:node`: `/root-project`, `/home/node/.claude`, `/home/node/.codex`, `/home/node/.config/gh`
- `proxy:proxy`: Squid の管理ディレクトリ

## 5. devcontainer.json 設計

### 5.1 実行ユーザー

- `remoteUser`: `node`

日常開発は非 root で行う。

### 5.2 capabilities

`runArgs` では以下を付与する。

- `NET_ADMIN`

加えて以下を有効化する。

- `--security-opt=no-new-privileges:true`

理由は `iptables` を用いた firewall 制御のため。
`NET_RAW` は packet crafting 等の追加攻撃面を増やすため、必要性が明確でない限り付与しない。
このため Squid の ICMP pinger は設定で無効化し、追加 capability に依存しない構成にする。
`seccomp=unconfined` や `apparmor=unconfined` は、コンテナ隔離を意図的に弱めるため不採用とする。

### 5.3 mount

主な mount は以下。

- workspace: `/root-project`
- Claude 設定 volume: `/home/node/.claude`
- Codex 設定 volume: `/home/node/.codex`
- GitHub CLI 設定 volume: `/home/node/.config/gh`
- bash history volume: `/commandhistory`
- ホスト `.gitconfig`: `/home/node/.gitconfig.host` readonly

### 5.4 環境変数

コンテナには proxy 関連を明示設定する。

- `HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY`
- `http_proxy`, `https_proxy`, `all_proxy`
- `NO_PROXY`, `no_proxy`

値は `http://127.0.0.1:3127` を中心に統一する。

補助設定:

- `GOPATH=/home/node/go`
- `HOST_GATEWAY_TCP_PORTS=3000,8080`
- upstream proxy 用の `UPSTREAM_PROXY_*`

`5432` のような DB ポートは、必要なときだけ明示的に追加する。開発テンプレートでは既定で開けない。

### 5.5 postCreateCommand

`postCreateCommand` では以下を行う。

- `/home/node/.gitconfig.host` を `/home/node/.gitconfig` にコピー
- `git config --global core.autocrlf input`
- `git config --global core.fileMode false`
- `safe.directory` に `/root-project` と主要 subrepo を追加
- `setup-shell-aliases.sh` を実行して `cc` / `ccr` / `cx` を設定

### 5.6 起動時 bootstrap

コンテナ起動直後に root の entrypoint から `/usr/local/bin/init-devcontainer.sh` を実行する。
このため、Dev Containers 側では `overrideCommand=false` を設定し、Dockerfile の `ENTRYPOINT` を上書きさせない。
Dockerfile 側の既定コマンドは `sleep infinity` とし、bootstrap 完了後は `node` 権限に落として待機させる。

`no-new-privileges` を有効化したまま `remoteUser=node` で `sudo` 昇格することはできないため、root が保持している権限の範囲で起動直後に必要処理を完了させ、その後に `node` へ権限を落として通常運用に入る。

起動時処理では以下を順に実行する。

1. `/home/node/.claude` の存在と所有権を補正
2. `/home/node/.codex` の存在と所有権を補正
3. `/home/node/.config/gh` の存在と所有権を補正
4. `/commandhistory` の存在と所有権を補正
5. `init-firewall.sh` を実行して proxy / firewall を初期化

VS Code の `postStartCommand` には root 昇格処理を置かない。
`overrideCommand` が既定値のまま Docker の command/entrypoint を Dev Containers に上書きされると、`HTTP_PROXY=http://127.0.0.1:3127` だけが有効になり、Squid が未起動のまま Codex / Claude Code が `Connection refused` を返すため、この設定は必須とする。

### 5.7 postAttachCommand

`postAttachCommand` は `setup-shell-aliases.sh` と `/usr/local/bin/announce-egress-status.sh` を順に実行する。

attach 直後に以下を短く表示する。

- `cc` / `ccr` / `cx` alias の再適用
- egress 検証の最新結果が `PASS` か `FAIL` か
- 実行時刻
- 詳細ログの保存先

これにより、リビルド直後や再接続直後でも、手動でログを探しに行かずに疎通確認結果を把握できる。

## 6. volume 所有権回復

### 6.1 背景

`/home/node/.claude`、`/home/node/.codex`、`/home/node/.config/gh` は named volume で保持するため、過去のコンテナや root 実行の影響で所有者が `root` になる可能性がある。

この状態で `remoteUser=node` のまま CLI が設定ディレクトリへアクセスすると、`Permission denied (os error 13)` が発生する。

### 6.2 対応

`init-devcontainer.sh` で以下を実施する。

- `install -d -o node -g node /home/node/.claude /home/node/.codex /home/node/.config/gh /commandhistory`
- `chown -R node:node /home/node/.claude /home/node/.codex /home/node/.config/gh /commandhistory`

これにより、既存 volume が残っていても起動時に自己回復する。

### 6.3 期待効果

- Codex 認証情報の読込失敗を防ぐ
- Claude 設定 volume の所有権崩れも同時に回復する
- GitHub CLI 認証情報 volume の所有権崩れも同時に回復する
- history volume への追記失敗も防ぐ

## 7. ネットワーク制御設計

### 7.1 基本方針

この構成は `explicit proxy + default-deny firewall` を採用する。

不採用としたもの:

- transparent interception
- `ssl_bump`
- HTTPS を中間者方式で解読する構成

理由:

- 実装が複雑になる
- 誤検知や運用事故のリスクが高い
- 開発用途としては FQDN allowlist の explicit proxy で十分

### 7.2 Squid

Squid は `127.0.0.1:3127` で待ち受ける explicit proxy として動作する。

制御内容:

- `pinger_enable off` で Squid の ICMP pinger を無効化
- `proxy-allowlist.txt` を元に URL regex を生成
- allowlist に載ったホストのみ `http_access allow`
- allowlist 外は deny
- upstream proxy 設定時は `cache_peer` を利用

### 7.3 firewall

`init-firewall.sh` は `iptables` で以下を構成する。

- `INPUT`, `FORWARD`, `OUTPUT` の既定ポリシーを `DROP`
- loopback を許可
- `ESTABLISHED,RELATED` を許可
- DNS を許可
- host gateway からの指定ポート inbound を許可
- Squid プロセスだけに必要な outbound を許可
- 最後に `OUTPUT REJECT` を設定

結果として、一般プロセスの direct outbound は fail-closed になる。

この fail-closed は、開発環境でも実務上有効な基準である。
allowlist に未登録の宛先へ誤って通信した場合は、黙って通すのではなく失敗させる。

## 8. host gateway 設計

### 8.1 inbound（ホスト → コンテナ）

host gateway からの inbound は `HOST_GATEWAY_TCP_PORTS` で制御する。

既定値:

- `3000`
- `8080`

想定用途:

- ローカルアプリ接続
- 開発用 API / UI 公開

### 8.2 運用方針

- inbound は必要最小限のポートだけを開ける
- 使わないポートは外す

### 8.3 BE テスト実行

devcontainer 内から Docker を操作する方式（DinD / DooD）は不採用とした（SECURITY.md 参照）。BE integration テストはホスト側で `docker compose --profile test run --rm test-be` により実行し、結果を共有ディレクトリ（`dev-journal/logs/test-results/`）に出力する。VS Code タスクでワンクリック実行可能。

## 9. allowlist 設計

allowlist は `.devcontainer/proxy-allowlist.txt` で定義する。

現在の主な対象:

- OpenAI: `chatgpt.com`, `api.openai.com`, `auth.openai.com`
- Anthropic: `api.anthropic.com`, `platform.claude.com`
- Node / Go / Python / VS Code 関連
補足:

- Codex の認証だけではなく、実際のチャット送信では `chatgpt.com` への通信が必要
- そのため `api.openai.com` と `auth.openai.com` だけでは不足する

## 10. upstream proxy 対応

必要に応じてローカル Squid の upstream に企業 proxy を設定できる。

利用環境変数:

- `UPSTREAM_PROXY_HOST`
- `UPSTREAM_PROXY_PORT`
- `UPSTREAM_PROXY_USERNAME`
- `UPSTREAM_PROXY_PASSWORD`
- `UPSTREAM_PROXY_PASSWORD_FILE`

動作:

- upstream 未設定時は Squid が直接 `tcp/80`, `tcp/443` に出る
- upstream 設定時は Squid が upstream proxy にのみ接続する
- 認証が必要な場合は `cache_peer ... login=user:password` を利用する

セキュリティ上は `UPSTREAM_PROXY_PASSWORD` の直接利用より、`UPSTREAM_PROXY_PASSWORD_FILE` を優先する。
平文の環境変数はプロセス一覧、デバッグ出力、設定共有時に露出しやすいためである。

## 11. 検証設計

`verify-egress.sh` では起動時に以下を確認する。

- allowlist 外の代表例として `https://example.com` が拒否されること
- OpenAI 必須エンドポイント（`chatgpt.com`, `auth.openai.com`, `api.openai.com`）が proxy 経由で到達可能であること
- allowlist に登録された全ホストが proxy 経由で到達可能であること
- proxy を使わない direct outbound が fail-closed で遮断されること

`init-firewall.sh` はこの検証結果を以下に保存する。

- ステータス: `/tmp/devcontainer-egress-check.status`
- 詳細ログ: `/tmp/devcontainer-egress-check.log`

attach 時には `announce-egress-status.sh` が status を読み取り、結果概要とログパスを表示する。

詳細を手動確認したい場合は以下を実行する。

```bash
cat /tmp/devcontainer-egress-check.status
cat /tmp/devcontainer-egress-check.log
```

1. `https://example.com` が proxy 経由で拒否される
2. `https://chatgpt.com` が proxy 経由で到達できる
3. `https://auth.openai.com` が proxy 経由で到達できる
4. `https://api.openai.com` が proxy 経由で到達できる
5. `https://api.openai.com` への direct outbound が失敗する
6. allowlist にある全ホストが proxy 経由で到達できる

これにより、allowlist 抜けと fail-closed 崩れを早期に検知する。

## 12. 運用ルール

### 12.0 このテンプレートで守る基準

- 日常操作は `remoteUser=node` の非 root 実行とする
- capability は必要最小限にする
- `unconfined` 系の設定は避ける
- 特権昇格防止として `no-new-privileges` を有効にする
- 外向き通信は allowlist 方式で fail-closed にする
- ビルド時取得物はバージョン固定する
- シークレットは環境変数よりファイルや secret manager を優先する
- 不要な host 公開ポートは既定で閉じる

### 12.1 allowlist 追加時

- 必要なホストだけを追加する
- 追加理由を明文化する
- 追加後は `verify-egress.sh` の観点に反映する

### 12.2 host gateway 変更時

- 必要最小限のポートだけを開ける
- 使わないポートは外す
- DB や管理系ポートは既定で開けず、必要時のみ一時的に追加する

### 12.3 volume 運用

対象 volume:

- `codex-config-*`
- `claude-code-config-*`

注意点:

- 認証情報は volume に残る
- 不要になった volume は削除する
- 所有権崩れは `init-devcontainer.sh` で回復する前提

## 13. 既知の注意点

- allowlist にないホストへの通信はすべて失敗する
- DNS や upstream proxy 側障害の影響を受ける
- named volume の認証情報は永続化される
- 起動時 bootstrap が失敗するとコンテナ起動自体が失敗する
- `overrideCommand=false` を外すと Dockerfile の entrypoint が実行されず、proxy 環境変数だけ残って `127.0.0.1:3127` への接続が `Connection refused` になる
- `NET_ADMIN` は依然として強い capability であるため、不要になれば削除を検討する
- `node:20-bookworm` のタグ固定だけでは十分ではなく、より厳密には digest pin が望ましい

## 14. 完了条件

現在設計が成立していると判断する条件は以下。

1. DevContainer が build できる
2. `overrideCommand=false` の状態で Dockerfile の entrypoint が実行される
3. 起動時 bootstrap が成功する
4. `init-devcontainer.sh` が volume 所有権を補正できる
5. Squid が起動する
6. `example.com` が拒否される
7. `chatgpt.com`, `auth.openai.com`, `api.openai.com` が許可される
8. direct outbound が fail-closed になる
9. Claude / Codex / npm / Go / Python / VS Code の必要通信が継続する
10. コンテナは bootstrap 後も待機し続け、attach 可能である
11. コンテナは非 root で運用される
12. `no-new-privileges` が有効である
13. `NET_RAW`, `seccomp=unconfined`, `apparmor=unconfined` を使わない
14. ツール導入に `latest` を使わない

## 15. 結論

現在の DevContainer は、開発利便性と通信制御を両立するために、明示 proxy と firewall を組み合わせた実装になっている。
さらに、最小権限とバージョン固定を加えることで、単なる「通信制限付きコンテナ」ではなく、実務で説明可能なセキュア開発環境に寄せる。

特に重要なのは以下の 2 点。

- 通信制御の実体は allowlist 付き Squid と `iptables`
- CLI の安定動作のため、起動時に named volume 所有権を補正する
- bootstrap を成立させるため、Dev Containers に Dockerfile の entrypoint を上書きさせない
- 最小権限の原則により、不要な capability と `unconfined` を避ける
- 取得物を固定し、`latest` 依存を避ける

この設計により、Codex / Claude を含む日常開発を維持しつつ、許可されていない外向き通信を抑止し、コンテナ権限と供給網リスクも抑える。
