# 引き継ぎメモ

## セッション: 2026-04-01 08:40

### ゴール
- Step 8 の成果物をユーザーが確認・理解する

### 作業ログ
- **Step 8 成果物確認**
  - `.github/PULL_REQUEST_TEMPLATE.md` — 中身空、成果物外。削除せず保留
  - `.github/workflows/ci.yml` / `deploy.yml` — CI/デプロイパイプライン確認。deploy.yml は ci.yml とジョブがコピペで二重
  - PR #2, #4 が CI 失敗のままマージされていた問題を発見 → 現在の master では解消済み
  - `.claude/` — 不要、削除
  - `.claude/worktrees/` — 不要、削除
- **PR フロー改善: CI 監視ステップ追加**
  - `workflow.md` の PR フローに手順 3（CI 監視）を追加。指揮役が `gh run watch` をバックグラウンドで実行し、失敗時は実装担当に修正指示（最大2回）
- **issue 起票 2件**
  - `039-environment-config-missing-in-architecture` — architecture.md に環境構成（開発/ステージング/本番）の定義が欠落。テンプレート・work-breakdown の修正も必要
  - `ops-048-work-breakdown-not-referenced-at-session-start` — セッション開始時に work-breakdown を参照する手順がない。完了条件未検証のまま Step 完了扱いになるリスク
- **Step 11 work-breakdown 更新**
  - 11-A（デプロイ環境構築）を新規追加、既存タスクを B/C/D にずらし
  - 上流成果物に運用設計・deploy.yml を追加
  - ただしデプロイ先（ステージング/本番）は issue 039 の環境構成定義待ち
- **docker compose up 起動確認**
  - `.dockerignore` が欠落していてビルド失敗 → 追加して解消
  - ホスト側 PowerShell から `docker compose up --build` で全サービス（db, db-test, api, frontend）起動確認
  - ヘルスチェック（`/health`）OK、フロントエンド表示 OK
- **VSCode タスク整備**
  - `.vscode/tasks.json` + `.vscode/launch.json` を root-project に作成
  - 「Docker: 起動 + ブラウザ」— build + ヘルスチェック待機 + ブラウザ自動オープン
  - 「Docker: 停止」— Ctrl+C 時に `docker compose down` が自動実行される `finally` 付きスクリプト
  - 「Setup: 初回セットアップ」— .env コピー + JWT 鍵生成（Docker 経由）
- **成果物の学習（ユーザー向け説明）**
  - JWT 認証（秘密鍵/公開鍵）の仕組み、RSA とハッシュの違い
  - Docker / docker-compose の役割、マルチステージビルド、キャッシュの仕組み
  - DB のロール分離（expense_owner / expense_app）と RLS の関係

### 未完了
- Step 8 成果物確認の残り: `Dockerfile`, `frontend/Dockerfile.dev`, バックエンド（`cmd/`, `internal/`, `server/`, `db/`）, フロントエンド（`frontend/`）, `Makefile`

### ブロッカー
- なし

### 次にやること
1. Step 8 成果物確認の続き（Dockerfile, バックエンド, フロントエンド, Makefile）
2. 確認完了後、issue 039（環境構成定義）/ ops-048（work-breakdown 参照手順）の対応判断
3. Step 9（テストコード実装）着手

### 学び・気づき
- **Step 8 の完了条件が実際に検証されていなかった**: `docker compose up` で起動確認が完了条件にあったが、Dev Container 内に Docker がなく未検証のままマージ。`.dockerignore` 欠落でビルド自体が失敗する状態だった
- **docker compose up の確認はホスト側で行う**: Dev Container に Docker を入れるのはセキュリティリスク（AI エージェントがホスト Docker を操作可能になる）。ホスト側ターミナルで実行する運用が正しい
- **ユーザーが技術を理解する時間を大切にする**: JWT、RSA、Docker など初見の概念が多く、丁寧な説明が必要だった

### 意思決定ログ
- **CI 監視の責務は指揮役**: サブエージェントに CI 待機させるのではなく、指揮役が `gh run watch` をバックグラウンドで実行。サブエージェントは実装に集中
- **デプロイ環境構築は Step 11-A**: Step 11 の最初のタスクとして追加。ただしデプロイ先はステージングが基本（本番で直接テストしない）。詳細は issue 039 で確定
- **Dev Container に Docker-in-Docker は入れない**: Codex の助言に従い、セキュリティ優先でホスト側実行の運用
- **VSCode タスクは root-project 直下に配置**: expense-saas 内ではなく root-project/.vscode/ に置き、`cwd` で expense-saas を指定
- **PULL_REQUEST_TEMPLATE.md / generate-keys.sh は削除しない**: ユーザー判断で保留

---

## セッション: 2026-03-31 19:44（前回）

### ゴール
- session-start スキルの二重管理・推論コスト問題を解決する

### 作業ログ
- **問題分析**: workflow.md の内容が session-start スキル内にハードコードされており二重管理。かつメモリ読み込み等の冗長ステップで推論コストが高い
- **コミュニティ調査**: Claude Code の rules/サブエージェント/コンテキスト管理のベストプラクティスをWeb検索
  - GitHub Issue #8395, #4908, #11759（Lead専用ルール、スコープ付きコンテキスト、遅延ロード）はいずれも closed/not planned
  - Skills を「遅延ロードルール」として使うパターンが最も推奨されている
  - Web情報では「カスタムエージェントはフレッシュコンテキスト」とされていたが、実機テストで否定（rules/、CLAUDE.md、MEMORY.md 全て読み込まれる）
- **実機テスト**: architect エージェントを起動し、rules/ と CLAUDE.md がコンテキストに含まれることを確認 → サブエージェントへのノイズ問題は実在
- **解決策の実施**:
  - `rules/lead-prohibitions.md` 新規作成（絶対禁止3項目を自動ロードで常時ガード）
  - `session-start SKILL.md` 軽量化（113行→48行）: フローコピー削除、メモリ読み込み削除、冗長な例の短縮
  - `feedback_always_read_workflow.md` 削除（rules/ でガードするため不要）
- **project-rules.md の検討**: Lead専用ルールが2つ混在しているが、8項目程度のノイズは許容範囲として現状維持
- **コンテキスト査定（/audit-context）**: 自動読み込み対象を全て評価
  - 不整合2件修正: CLAUDE.md の古い記述、implementation-workflow.md の廃止済み「main 直接」セクション
  - 重複メモリ削除: `feedback_planning_delegation.md`（workflow.md にルール化済み）
  - メモリ短縮: `feedback_no_emoji_in_pr.md`（12行→3行）
  - 低頻度スキル6個の description を1行に圧縮（daily-report, analyze, self-review, adr, check-structure, audit-context）

### 未完了
- なし

### ブロッカー
- なし

### 次にやること
1. Step 9（テストコード実装）の着手 — Step 8 全成果物が揃っている
2. ops-036（LSP連携）/ ops-047（work-breakdown 分割）— 低優先

### 学び・気づき
- **Web情報より実機テストを信頼する**: 「カスタムエージェントはフレッシュコンテキスト」という情報は誤りだった。ユーザーが過去のテスト結果を覚えていて修正できた
- **rules/ に置くべきは「違反が不可逆なもの」**: 全ルールを rules/ に入れるのではなく、マージ・コミット等の不可逆操作に関する禁止事項のみを常時ガード。手順の詳細は guide/ + スキルで十分

### 意思決定ログ
- **禁止事項と手順詳細の分離**: 不可逆な違反（codexレビュー前マージ等）は rules/ で自動ロード、フロー手順の詳細は guide/workflow.md に残して session-start スキルで Read する方式。サブエージェントへのノイズを最小限に抑えつつ致命的違反をガード
- **メモリ読み込みステップの削除**: MEMORY.md は自動ロードされることを実機テストで確認。スキル内で重複して Read する必要なし
- **project-rules.md は現状維持**: Lead専用ルール2項目の混在は許容。分割するとファイル管理コストが増え、8項目程度のノイズは実害がない
