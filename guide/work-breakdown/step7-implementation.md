# Step 7: 実装・運用（テスト→実装→CI→デプロイ） — 作業分解

## 概要

動くものを出し、継続的に改善できる形にする。

## 実装フェーズ構成

- Phase 1（基盤）：プロジェクト初期設定・Docker Compose（ローカル PostgreSQL）・DBマイグレーション・認証API・CI基盤（lint / test / build）
- Phase 2（コア機能）：RBACミドルウェア・経費レポートCRUD・状態遷移・添付ファイル・承認フロー・フロントエンド主要画面
- Phase 3（付随機能）：監査ログ・通知・CSVエクスポート・招待フロー・メンバー管理
- Phase 4（デプロイ・仕上げ）：AWSインフラ構築（Terraform or CDK）・staging/production デプロイ・ヘルスチェック・CloudWatchアラート

## 成果物（出力）

| 成果物 | 内容 |
|--------|------|
| 実装コード | コミット履歴、PR運用（任意でも強い） |
| IaCコード | Terraform or CDK |
| デプロイURL | 検証用アカウント/デモ手順 |
| README | 日英、運用メモ（ログ/ヘルスチェック/アラート方針） |

## 【root-project 整備】

- `docker-compose.yml`：ローカル開発用の PostgreSQL サービス定義（Dev Container には DB クライアントのみ含まれるため、DB サーバーは Docker Compose で提供する）（Phase 1 着手時に作成）
- `rules/branching.md`：ブランチ戦略（main / develop / feature / hotfix）を確定・記載（Phase 1 着手前に完了）
- `rules/commit-message.md`：Conventional Commits 規約を詳細化（型の一覧・スコープ例・NG例）（Phase 1 着手前に完了）
- `scripts/setup.sh`：ローカル開発環境のセットアップ手順をスクリプト化（Phase 1 完了後）
- `scripts/lint.sh`：`golangci-lint run` / `npm run lint` をまとめた lint スクリプト実装（Phase 1 完了後）
- `scripts/db-reset.sh`：DB リセット・シード投入スクリプト実装（Phase 1 完了後）
- `prompts/implementation.md`：実装フェーズで繰り返し使う指示文テンプレートを整備（Phase 2 着手前）
- `prompts/refactor.md`：リファクタリング指示文テンプレートを整備（Phase 2〜3 中に整備）
- `skills/release/`：リリース前チェックリスト・デプロイ手順・ロールバック手順を整備（Phase 4 着手前）
- `deliverables/docs/30_arch/architecture.md` を公開向けに整形（Phase 4）
- `deliverables/docs/50_detail_design/api.md`：openapi.yaml の概要・利用方法・認証方式を記載（Phase 4）
- `references/tech-stack-notes/`：選定理由サマリの整形（Phase 4）
- `deliverables/docs/70_operations/runbook.md`：デプロイ・障害対応・ヘルスチェック手順を記載（Phase 4）

## 完了条件

- デプロイ済みで第三者が触れる
- READMEだけでセットアップ＆概要が伝わる
- ヘルスチェックエンドポイント（`/health`）が存在する
