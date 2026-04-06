# ADR-003: Dev Container 導入

## ステータス
採用（2026-03-11）

## コンテキスト

Windows 11 ネイティブでは Claude Code のサンドボックスが正常動作しないため WSL2 に移行済み。
さらにセキュリティ強化（ネットワーク隔離）のため、公式推奨の Dev Container に移行する。

公式の devcontainer は Node.js 中心のため、Go + PostgreSQL クライアント + Python（hooks用）と、Claude Code / Codex CLI、送信先制限付きプロキシを追加カスタマイズする。

## 決定

公式 Claude Code devcontainer をベースに、プロジェクト固有のカスタマイズを加えて `.devcontainer/` に配置する。

## 作成ファイル

すべて `.devcontainer/` に配置。

### 1. `devcontainer.json`

公式ベース + 以下の変更:

| 項目 | 公式 | 変更 | 理由 |
|------|------|------|------|
| name | `Claude Code Sandbox` | `Expense SaaS Dev` | プロジェクト識別 |
| TZ | `America/Los_Angeles` | `Asia/Tokyo` | 日本在住 |
| build.args | Claude Code 中心 | `CODEX_VERSION`, `GO_VERSION`, `GOLANGCI_LINT_VERSION` 追加 | 開発ツール固定 |
| extensions | eslint, prettier, gitlens | `anthropic.claude-code`, `golang.go` 追加 | 日常開発必須 |
| forwardPorts | 最小構成 | `[3000, 8080]` | フロントエンド, バックエンド |
| mounts | 認証情報中心 | `.claude`, `.codex`, bash history, ホスト `.gitconfig` をマウント | 認証・履歴・git 設定共有 |
| containerEnv | 最小構成 | Go 環境変数とローカルプロキシ設定を追加 | 開発ツール実行と送信先制限 |
| postCreateCommand | なし | git config（autocrlf, fileMode, safe.directory）と zsh alias 追加 | 初回のみ実行 |

**マウント設計:**
- `/root-project` ← ホストの `root-project/` をバインドマウント（`.claude/` 含む）
- `/home/node/.claude` ← 名前付きボリューム（Claude Code CLI 認証情報用、プロジェクト設定とは別）
- `/home/node/.codex` ← 名前付きボリューム（Codex CLI 認証情報用）
- `/commandhistory` ← 名前付きボリューム（bash 履歴永続化）
- `/home/node/.gitconfig.host` ← ホストの `.gitconfig` を readonly bind mount し、初回にコンテナ側へコピー

### 2. `Dockerfile`

公式ベース（`node:20`）+ 以下を追加:

| 追加パッケージ | 理由 |
|---------------|------|
| `python3`, `python-is-python3` | hooks（edit-scope-check.py, stop-check.py）が Python 使用 |
| `postgresql-client`, `libpq-dev` | psql コマンド、PostgreSQL 接続確認 |
| `pkg-config`, `build-essential` | Go ツールや一部ネイティブ依存のビルド補助 |
| `iptables`, `iproute2`, `dnsutils`, `squid-openssl` | fail-closed なプロキシ/ファイアウォール構成 |
| `wget` | Go, git-delta, zsh セットアップスクリプト取得 |
| Go ツールチェーン | バックエンド開発 |
| `golangci-lint` | Go の静的解析 |
| Claude Code CLI / Codex CLI | AI 開発ツール |

**レイヤー順序:** OS パッケージ → Go ツールチェーン → `golangci-lint` → zsh → Claude Code CLI / Codex CLI の順で、キャッシュ効率を最大化。

### 3. `init-firewall.sh`

送信先を Squid プロキシ経由の allowlist に限定し、コンテナからの直接外向き通信を fail-closed にする。

allowlist には少なくとも以下を含める:

| 許可ドメイン | 用途 |
|-------------|------|
| `api.anthropic.com`, `platform.claude.com` | Claude Code 利用 |
| `chatgpt.com`, `api.openai.com`, `auth.openai.com` | Codex / OpenAI 利用 |
| `github.com`, `api.github.com`, `raw.githubusercontent.com` など | ソース取得、CLI、リリース取得 |
| `registry.npmjs.org` | npm パッケージ取得 |
| `proxy.golang.org`, `sum.golang.org`, `storage.googleapis.com` | Go モジュール取得 |
| `pypi.org`, `files.pythonhosted.org` | Python パッケージ取得 |
| VS Code Marketplace 関連ドメイン | 拡張機能取得 |

ファイアウォール検証では、allowlist 経由の疎通確認に加えて `https://example.com` のブロックと、プロキシを経由しない直接通信のブロックを確認する。

## `.claude/` 設定の動作方式

プロジェクト設定（hooks, rules, skills, settings.json）はワークスペースバインドマウント経由で `/root-project/.claude/` として自動参照される。変更不要。

- hooks コマンド: `python "$CLAUDE_PROJECT_DIR/.claude/hooks/..."` → `/root-project/.claude/hooks/...` に展開
- memory, settings.local.json: `.gitignore` 対象だがバインドマウントでホスト側に永続化
- CLI 認証情報: `/home/node/.claude/` の名前付きボリュームに保存（再認証不要）
- Codex CLI 認証情報: `/home/node/.codex/` の名前付きボリュームに保存（再認証不要）

## 導入手順（ユーザー操作）

1. VS Code に **Dev Containers 拡張**（`ms-vscode-remote.remote-containers`）をインストール
2. `.devcontainer/` 配下の設定を用意した状態で、VS Code で `root-project/` を開く
3. `Ctrl+Shift+P` → `Dev Containers: Reopen in Container`
4. 初回ビルド（6-10分、2回目以降はキャッシュで1分以内）
5. コンテナ内ターミナルで `claude login` → ブラウザ認証
6. 必要に応じて `codex login`
7. `gh auth login` → `gh auth setup-git`

## 動作確認チェックリスト

- [ ] `go version` / `golangci-lint version` / `node --version` / `python --version`
- [ ] `claude --version` と `codex --version` が表示される
- [ ] ファイアウォール: `curl https://example.com` がブロックされる
- [ ] ファイアウォール: `curl https://proxy.golang.org` が通る
- [ ] hooks: `/root-project/.claude/hooks/edit-scope-check.py` が Python で実行可能
- [ ] git: 各サブリポジトリで `git status` が正常動作
- [ ] Go: `cd /root-project/expense-saas/apps/api && go test ./...`

## リスクと対策

| リスク | 対策 |
|--------|------|
| allowlist に不足ドメインがあると CLI や package install が失敗する | `verify-egress.sh` の疎通確認を更新し、必要ドメインを allowlist に追記する |
| Windows→Linux バインドマウントのパーミッション問題 | `git config core.fileMode false` を postCreateCommand に追加 |
| 初回ビルドに時間がかかる | 事前告知。Docker レイヤーキャッシュで2回目以降は高速 |

## 代替案

| 選択肢 | 不採用理由 |
|--------|-----------|
| WSL2 サンドボックスのみ | ネットワーク隔離なし。ポートフォリオとしてのアピールが弱い |
| GitHub Codespaces | コスト発生。ローカル開発の柔軟性が失われる |
| Docker Compose のみ（Dev Container なし） | VS Code 統合がない。IDE 設定の共有ができない |
