# ADR-003: Dev Container 導入

## ステータス
採用（2026-03-11）

## コンテキスト

Windows 11 ネイティブでは Claude Code のサンドボックスが正常動作しないため WSL2 に移行済み。
さらにセキュリティ強化（ネットワーク隔離）のため、公式推奨の Dev Container に移行する。

公式の devcontainer は Node.js のみ対応のため、Rust + PostgreSQL クライアント + Python（hooks用）を追加カスタマイズする。

## 決定

公式 Claude Code devcontainer をベースに、プロジェクト固有のカスタマイズを加えて `.devcontainer/` に配置する。

## 作成ファイル（3ファイル）

すべて `.devcontainer/` に配置。

### 1. `devcontainer.json`

公式ベース + 以下の変更:

| 項目 | 公式 | 変更 | 理由 |
|------|------|------|------|
| name | `Claude Code Sandbox` | `Expense SaaS Dev` | プロジェクト識別 |
| TZ | `America/Los_Angeles` | `Asia/Tokyo` | 日本在住 |
| build.args | node 系のみ | `RUST_VERSION`, `SQLX_CLI_VERSION` 追加 | Rust ツールチェーン管理 |
| extensions | eslint, prettier, gitlens | `rust-analyzer`, `even-better-toml` 追加 | Rust 開発必須 |
| forwardPorts | なし | `[5432, 3000, 8080]` | DB, フロントエンド, バックエンド |
| containerEnv | 3変数 | `CARGO_HOME`, `RUSTUP_HOME` 追加 | Rust パス設定 |
| postCreateCommand | なし | git config（user, autocrlf, safe.directory） | 初回のみ実行 |

**マウント設計:**
- `/workspace` ← ホストの `root-project/` をバインドマウント（`.claude/` 含む）
- `/home/node/.claude` ← 名前付きボリューム（Claude Code CLI 認証情報用、プロジェクト設定とは別）
- `/commandhistory` ← 名前付きボリューム（bash 履歴永続化）

### 2. `Dockerfile`

公式ベース（`node:20`）+ 以下を追加:

| 追加パッケージ | 理由 |
|---------------|------|
| `python3`, `python-is-python3` | hooks（edit-scope-check.py, stop-check.py）が Python 使用 |
| `postgresql-client`, `libpq-dev` | psql コマンド、sqlx-cli ビルド依存 |
| `pkg-config`, `libssl-dev`, `build-essential` | Rust クレート（actix-web, sqlx 等）のビルド依存 |
| Rust ツールチェーン（rustup 経由） | バックエンド開発。clippy + rustfmt コンポーネント込み |
| `sqlx-cli`（cargo install） | DB マイグレーション管理 |

**レイヤー順序:** Rust（変更少）→ sqlx-cli → zsh → Claude Code CLI（変更多）の順で、キャッシュ効率を最大化。

### 3. `init-firewall.sh`

公式ベース + 以下のドメインを追加:

| 追加ドメイン | 用途 |
|-------------|------|
| `crates.io` | Cargo レジストリ API |
| `static.crates.io` | クレートダウンロード |
| `index.crates.io` | Sparse レジストリインデックス |
| `static.rust-lang.org` | rustup コンポーネント更新 |

ファイアウォール検証セクションに crates.io への疎通確認を追加。

## `.claude/` 設定の動作方式

プロジェクト設定（hooks, rules, skills, settings.json）はワークスペースバインドマウント経由で `/workspace/.claude/` として自動参照される。変更不要。

- hooks コマンド: `python "$CLAUDE_PROJECT_DIR/.claude/hooks/..."` → `/workspace/.claude/hooks/...` に展開
- memory, settings.local.json: `.gitignore` 対象だがバインドマウントでホスト側に永続化
- CLI 認証情報: `/home/node/.claude/` の名前付きボリュームに保存（再認証不要）

## 導入手順（ユーザー操作）

1. VS Code に **Dev Containers 拡張**（`ms-vscode-remote.remote-containers`）をインストール
2. 3ファイル作成後、VS Code で `root-project/` を開く
3. `Ctrl+Shift+P` → `Dev Containers: Reopen in Container`
4. 初回ビルド（6-10分、2回目以降はキャッシュで1分以内）
5. コンテナ内ターミナルで `claude login` → ブラウザ認証
6. `gh auth login` → `gh auth setup-git`

## 動作確認チェックリスト

- [ ] `rustc --version` / `cargo --version` / `node --version` / `python --version`
- [ ] `claude --version` → `claude login` で認証
- [ ] ファイアウォール: `curl https://example.com` がブロックされる
- [ ] ファイアウォール: `curl https://crates.io/api/v1/crates?per_page=1` が通る
- [ ] hooks: `/workspace/.claude/hooks/edit-scope-check.py` が Python で実行可能
- [ ] git: 各サブリポジトリで `git status` が正常動作
- [ ] Rust: `cd /workspace/expense-saas/apps/api && cargo check`

## リスクと対策

| リスク | 対策 |
|--------|------|
| `static.crates.io` の CDN IP が DNS 解決と異なる | まず DNS 方式で試行。失敗時は CloudFront IP 範囲を追加 |
| Windows→Linux バインドマウントのパーミッション問題 | `git config core.fileMode false` を postCreateCommand に追加 |
| 初回ビルドに時間がかかる | 事前告知。Docker レイヤーキャッシュで2回目以降は高速 |

## 代替案

| 選択肢 | 不採用理由 |
|--------|-----------|
| WSL2 サンドボックスのみ | ネットワーク隔離なし。ポートフォリオとしてのアピールが弱い |
| GitHub Codespaces | コスト発生。ローカル開発の柔軟性が失われる |
| Docker Compose のみ（Dev Container なし） | VS Code 統合がない。IDE 設定の共有ができない |
