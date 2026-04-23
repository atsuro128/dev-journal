# DevContainer: Claude Code を Dockerfile 時点で native installer に切り替える検討

## 発見日
2026-04-23

## カテゴリ
infrastructure

## 影響度
低

## 発見経緯
proactive

## 関連ステップ
DevContainer 運用（全ステップ共通）

## ブロッカー
なし

## 問題

現在の `.devcontainer/Dockerfile` (line 117) では Claude Code を npm 経由でインストールしている:

```dockerfile
RUN npm install -g @anthropic-ai/claude-code@${CLAUDE_CODE_VERSION} @openai/codex@${CODEX_VERSION}
```

一方、Anthropic は native installer を推奨経路としており、初回の `claude install` / `claude update` 実行時に `/home/node/.local/share/claude/versions/` 配下へ自動移行する。結果、次の乖離が発生している。

- リビルド直後: npm 版（`/usr/local/share/npm-global/...`、Dockerfile が固定した `CLAUDE_CODE_VERSION` で動作）
- 初回更新後: native 版（`~/.local/share/claude/versions/`、native updater が取得したバージョン）

`devcontainer-design.md` §4 の記述（`CLI: @anthropic-ai/claude-code, @openai/codex`）は npm 前提で書かれており、実ランタイム状態（native）と食い違う。

## 影響

- 設計書と実ランタイム状態が暗黙に食い違い、`node:20-bookworm` 以外の要素でも再現性の説明が崩れる
- Dockerfile で固定した `CLAUDE_CODE_VERSION` は初回 `claude update` で上書きされるため、ピン止めの意図が実質効いていない
- 新規参加者がインストール経路の二重状態に混乱する可能性
- 今回発生した `downloads.claude.ai` の allowlist 漏れのような「native 経路特有の要件」が、npm 前提の設計認識だと見落とされやすい

## 提案

Dockerfile を native installer 直接導入に切り替えることを検討する。具体的には line 117 を以下のような形に置換する案:

```dockerfile
RUN curl -fsSL https://claude.ai/install.sh | bash -s -- <version-pin-option>
```

ただし採用判断には以下の事前検証が必要:

1. **リダイレクト先確認**: `https://claude.ai/install.sh` が実際にどこへリダイレクトされ、スクリプトがどこからバイナリを取得するかを実地で確認。`claude.ai` を allowlist に追加する必要があるか、`downloads.claude.ai` のみで完結するかを特定する。
2. **バージョン固定方式**: native installer が環境変数・引数でのバージョン指定をサポートするか確認。できない場合、固定版 URL を直接 `wget`/`curl` で取得する代替案も検討。
3. **監査性**: `curl | bash` パターンは npm パッケージ署名と比べてスクリプト内容の変更追跡が難しい。取得スクリプトのハッシュ固定などの補強策を検討。
4. **codex との分離**: codex は npm 経由のままで妥当か、同時に native 化可能かを整理。`registry.npmjs.org` は codex / フロントエンドで結局必須のため、claude 分だけ外しても allowlist 縮小効果は無い。

## 受け入れ条件

- 採用可否を判断材料と併せて決定し、設計書に記録する
- 採用する場合は Dockerfile / proxy-allowlist.txt / proxy-allowlist-rationale.md / devcontainer-design.md § 4 の更新を 1 回のリビルドにまとめる
- 採用しない場合は「npm 経由 → 初回 update で native 化」という現状を許容する理由（監査性・バージョン固定の意図など）を rationale として devcontainer-design.md もしくは本 issue に残す

## 関連

- 本 issue の発端: allowlist に `downloads.claude.ai` が無く `claude update` が HTTP 403 で失敗した件（root-project `60af7ff` / dev-journal `83c6168` で対処済）
- 参照: Anthropic 公式 `code.claude.com/docs/en/network-config`、`code.claude.com/docs/en/setup`

---

## 解決内容

Dockerfile を Anthropic 公式 native installer 経由の導入に切り替えた。

### 変更内容

1. `.devcontainer/Dockerfile` line 117
   - 旧: `RUN npm install -g @anthropic-ai/claude-code@${CLAUDE_CODE_VERSION} @openai/codex@${CODEX_VERSION}`
   - 新:
     ```dockerfile
     RUN npm install -g @openai/codex@${CODEX_VERSION}
     RUN curl -fsSL https://claude.ai/install.sh | bash -s ${CLAUDE_CODE_VERSION}
     ENV PATH=/home/node/.local/bin:$PATH
     ```
   - `CLAUDE_CODE_VERSION` は install.sh の位置引数で解釈される。これにより build 時点で `/home/node/.local/share/claude/versions/${CLAUDE_CODE_VERSION}` にバイナリが配置される。

2. `dev-journal/references/devcontainer-docs/devcontainer-design.md` §4
   - CLI 項目を Claude Code / Codex で導入経路別に記述
   - 「導入物の原則」に、npm 経由導入の暗黙差し替えを避けるため Dockerfile 時点で native installer に寄せる旨を追記

### 検証項目の結論

- **リダイレクト先**: runtime では `downloads.claude.ai` のみ必要で allowlist 追加済（`60af7ff`）。`claude.ai` は build 時のみ使用し、docker build は in-container squid を経由しないため allowlist 追加不要と判断した。
- **バージョン固定**: `install.sh` の位置引数で可能（例: `bash -s 2.1.96`）。`CLAUDE_CODE_VERSION` ARG で引き続き管理する。
- **監査性**: 既存 Dockerfile で git-delta / Go / github-mcp-server を同様の `wget` ベースで導入済みのため、監査ポリシーとして新しい例外は作らない。
- **codex との分離**: codex は npm 経由を継続。`registry.npmjs.org` はフロントエンドと codex のため必須のまま変更なし。

### 受け入れ条件の充足

- [x] 採用可否を判断材料と併せて決定し、設計書に記録（design doc §4 更新）
- [x] 採用したため Dockerfile / devcontainer-design.md を 1 回のリビルドにまとめる（proxy-allowlist.txt / rationale は変更不要と判断）

### 残作業

ユーザー側でのコンテナリビルドと、`claude --version` が `CLAUDE_CODE_VERSION` と一致すること、および `which claude` が `/home/node/.local/bin/claude` を指すことの確認。

## 解決日
2026-04-23
