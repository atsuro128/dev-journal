# 引き継ぎメモ

## セッション: 2026-04-02 15:18

### ゴール
- Step 8 成果物のレビュー + セッション管理の改善

### 作業ログ
- **Step 8 成果物レビュー**（詳細は `review-log.md` を参照）
  - `main.go` のステップ 1〜7 を確認（ステップ 8〜12 は詳細未レビュー）
- **issue 4件起票**
  - 050: JWT 署名アルゴリズムの耐量子計算機方式への移行
  - 051: アーキテクチャ設計書のスケーラビリティトレードオフ未記載
  - 052: JWT 鍵ファイル未設定でもサーバーが起動する問題
  - 053: 非機能要件のテストカバレッジ不足
- **セッション管理の分離**
  - `/human-review-start`, `/human-review-log` スキルを新規作成
  - 開発セッション（session-log）とレビューセッション（review-log）を分離
  - CLAUDE.md のセッション開始手順を更新

### 未完了
- なし（レビューの未完了は review-log.md で管理）

### ブロッカー
- なし

### 次にやること
1. Step 9（テストコード実装）着手
2. issue 039（環境構成定義）/ ops-048（work-breakdown 参照手順）の対応判断

### 学び・気づき
- **Go 未経験の壁**: ポインタ、レシーバ、暗黙的インターフェースなど Java にない概念が多い。ただし構文自体は少ないので、慣れれば読める
- **レビューの価値**: AI が書いたコードでも人間のレビューで設計上の問題（JWT アルゴリズム、スケーラビリティ）が発見できた
- **「A Tour of Go」の活用**: 体系的な Go 学習の補完に公式チュートリアルを推奨

### 意思決定ログ
- **ML-DSA 採用方針**: RS256 は 2030 年以降 NIST 非推奨。EdDSA も量子耐性なし。ML-DSA（FIPS 204）を検討。署名サイズ（約 2.4KB〜3.3KB）のトレードオフあり。Go では circl ライブラリ + golang-jwt のカスタム SigningMethod で実装
- **スケーラビリティは設計書に明記**: 現状の RLS 方式の限界と段階的移行パスをアーキテクチャ設計書に追記する方針。今からの設計変更はしない（YAGNI）
- **JWT 鍵ファイル必須化**: 開発環境でも鍵ファイルは配置済みなので、省略可能にする理由がない。全環境で必須に変更予定
- **issue の記載はポートフォリオ公開を意識**: 「認識できていなかった」等のネガティブ表現を避け、事実ベースで記載する

---

## セッション: 2026-04-01 13:29（前回）

### ゴール
- Step 8 成果物確認の続き（前回の未完了分）

### 作業ログ
- **Step 8 成果物確認**
  - `Dockerfile`（本番用）— マルチステージビルド、CGO_ENABLED=0、ca-certificates
  - `frontend/Dockerfile.dev`（開発用）— Vite dev server 用、docker compose 専用
  - `cmd/server/main.go` — エントリーポイント確認途中。設定読み込み・ログ・DB接続まで説明
  - `.env` / `.env.example` — 環境変数の仕組みを説明。Docker なしでも使えること、application.properties との共通点
  - `docker-compose.yml` — 4サービス構成（db, db-test, api, frontend）を確認
- **マージ漏れ発見・修正**
  - expense-saas: 8-9（開発者ツール）, 8-10（整理）のコミットが master 未マージだった
  - root-project: 8-8 ブランチに 8-9 以降のコミットが残っていた
  - cherry-pick で master に取り込み。ただし1コミット重複で履歴が汚れた → rebase で整理
- **issue 049 起票**: コード内コメントの日本語化（godoc 向け以外）
- **post-edit-format-lint.sh 削除**: git hook（.githooks/pre-commit）と二重化していたため
- **worktree 並列実装基盤の構築**
  - git worktree は expense-saas に対して使えることを調査で判明（「使えない」は誤りだった）
  - 実装エージェント4種の AGENT.md に `isolation: worktree` を追加
  - WorktreeCreate/Remove フック作成・別セッションで動作確認済み
  - ブランチ名カスタマイズ可能（worktree- prefix なし）
- **/implement スキル新規作成**: チケット読み込み → 依存チェック → エージェント起動 → PR 作成を一括化
- **workflow.md 大幅改訂**
  - PR フローに `/implement` スキルを導入（ステップ数 6→5）
  - 行動原則に「1チケットの1工程 = 1エージェント、並列実行」を追加
  - lead-prohibitions.md を削除し workflow.md に統合（サブエージェントへのノイズ削減）
  - ブランチ運用セクション削除（PR フローに統合）
  - master ブランチチェックルール追加
  - ブランチはマージ後も削除しない方針に変更
- **/commit スキル更新**: master ブランチチェックを追加

### 未完了
- Step 8 成果物確認の残り: `internal/`（handler, service, domain, repository, middleware, config, pkg）, `frontend/`, `Makefile`, `db/`
- `main.go` の続き（DB接続以降: JWT初期化、DI、ルーティング、グレースフルシャットダウン）

### ブロッカー
- なし

### 次にやること
1. Step 8 成果物確認の続き（`main.go` の残り → `internal/` → `frontend/` → `Makefile`）
2. 確認完了後、Step 9（テストコード実装）着手
3. issue 039（環境構成定義）/ ops-048（work-breakdown 参照手順）の対応判断

### 学び・気づき
- **cherry-pick は慎重に**: マージ済みコミットを含めて cherry-pick すると履歴が重複する。差分を正確に確認してから実行すべき
- **「できない」と断言する前に調べる**: worktree が使えないという前提が誤りだった。調査エージェントを活用すべき
- **フックの二重化に注意**: Claude Code フック（post-edit）と git hook（pre-commit）で同じ処理が走っていた
- **PR マージ後はローカルで master に戻す**: ブランチに留まったまま次の作業をするとコミットが迷子になる

### 意思決定ログ
- **worktree 有効化**: git worktree は expense-saas に対して正常に動作する。Claude Code の EnterWorktree が root-project に対して動くだけの問題だった。AGENT.md に `isolation: worktree` を設定し並列実装を可能にした
- **lead-prohibitions.md 削除**: ユーザーの方針で session-start を必ず実行する前提。rules/ に置く必要なし。workflow.md に統合してサブエージェントへのノイズを削減
- **ブランチはマージ後も残す**: 削除するデメリットはなく、何かあった時に履歴を追える
- **コメント日本語化は PR 単位で確認**: 一括リファクタリングではなく、各 PR でユーザーが都度確認する方が理解しやすい
