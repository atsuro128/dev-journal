# 完了 Step チケット一覧アーカイブ

progress.md から退避した完了 Step のチケット一覧を保持する。

## Step 8: 基盤構築 — チケット一覧

| ID | タスク | 担当 | 依存 | 状態 | チケット |
|----|--------|------|------|------|---------|
| 8-1 | 開発環境構築 + DB | platform-builder | なし | 完了 | `tickets/step8/8-1-dev-environment.md` |
| 8-2 | バックエンド初期化 | backend-developer | 8-1 | 完了 | `tickets/step8/8-2-backend-init.md` |
| 8-3 | フロントエンド初期化 | frontend-developer | なし | 完了 | `tickets/step8/8-3-frontend-init.md` |
| 8-4 | 共通ミドルウェア + ヘルスチェック | backend-developer | 8-2 | 完了 | `tickets/step8/8-4-middleware.md` |
| 8-5 | FE-BE 連携 | frontend-developer | 8-3, 8-4 | 完了 | `tickets/step8/8-5-fe-be-integration.md` |
| 8-6 | コード生成・スケルトン | backend-developer | 8-4 | 完了 | `tickets/step8/8-6-skeleton.md` |
| 8-7 | 共通 UI コンポーネント実装 | frontend-developer | 8-3, 8-5 | 完了 | `tickets/step8/8-7-common-ui-components.md` |
| 8-8 | テスト基盤 | backend-developer | 8-6 | 完了 | `tickets/step8/8-8-test-infra.md` |
| 8-9 | CI/CD パイプライン | platform-builder | 8-2, 8-3 | 完了 | `tickets/step8/8-9-cicd.md` |
| 8-10 | 開発者ツール | platform-builder | 8-2, 8-3 | 完了 | `tickets/step8/8-10-dev-tools.md` |
| 8-11 | ローカル開発環境統合 | platform-builder | 8-1, 8-2, 8-3, 8-8 | 完了 | `tickets/step8/8-11-local-dev-integration.md` |
| 8-12 | 整理 | platform-builder | なし | 完了（旧 8-11） | `tickets/step8/8-12-cleanup.md` |

## Step 6: テスト設計 — 追加チケット（issue 074 対応）

| ID | タスク | 担当 | 依存 | 状態 | チケット |
|----|--------|------|------|------|---------|
| 6-D | 手動チェックリスト定義 | designer | 6-C | 完了 | `tickets/step6/6-D-manual-checklists.md` |

## Step 5.5: UI コンポーネント設計 — チケット一覧

| ID | タスク | 担当 | 依存 | 状態 | チケット |
|----|--------|------|------|------|---------|
| 5.5-A | 共通コンポーネント設計 | designer | なし | 完了 | `tickets/step5.5/5.5-A-common-components.md` |
| 5.5-B | 状態管理方針策定 | designer | なし | 完了 | `tickets/step5.5/5.5-B-state-management.md` |
| 5.5-C-1 | 画面別コンポーネント設計（認証系） | designer | 5.5-A, 5.5-B | 完了 | `tickets/step5.5/5.5-C-1-auth-components.md` |
| 5.5-C-2 | 画面別コンポーネント設計（ダッシュボード） | designer | 5.5-A, 5.5-B | 完了 | `tickets/step5.5/5.5-C-2-dashboard-components.md` |
| 5.5-C-3 | 画面別コンポーネント設計（レポート系） | designer | 5.5-A, 5.5-B | 完了 | `tickets/step5.5/5.5-C-3-report-components.md` |
| 5.5-C-4 | 画面別コンポーネント設計（ワークフロー系） | designer | 5.5-A, 5.5-B | 完了 | `tickets/step5.5/5.5-C-4-workflow-components.md` |
| 5.5-C-5 | 画面別コンポーネント設計（管理系） | designer | 5.5-A, 5.5-B | 完了 | `tickets/step5.5/5.5-C-5-admin-components.md` |
| 5.5-D | 横断レビュー | reviewer | 5.5-C-1〜5 | 完了 | `tickets/step5.5/5.5-D-cross-review.md` |

## Step 9: テストコード実装 — チケット一覧

| ID | タスク | 担当 | 依存 | 状態 | チケット |
|----|--------|------|------|------|---------|
| 9-A | 認証テスト（共通フィクスチャ含む） | test-implementer | Step 8 完了 | 完了 | `tickets/step9/9-A-auth-test.md` |
| 9-B | レポートテスト | test-implementer | 9-A | 完了 | `tickets/step9/9-B-report-test.md` |
| 9-C | ダッシュボードテスト | test-implementer | 9-A | 完了 | `tickets/step9/9-C-dashboard-test.md` |
| 9-D | テナント管理テスト | test-implementer | 9-A | 完了 | `tickets/step9/9-D-tenant-test.md` |
| 9-E | 明細テスト | test-implementer | 9-B | 完了 | `tickets/step9/9-E-items-test.md` |
| 9-F | ワークフローテスト | test-implementer | 9-B | 完了 | `tickets/step9/9-F-workflow-test.md` |
| 9-G | 添付ファイルテスト | test-implementer | 9-E | 完了 | `tickets/step9/9-G-attachments-test.md` |
| 9-X | 横断レビュー | reviewer (codex) | 9-A〜9-G | 完了 | `tickets/step9/9-X-cross-review.md` |

## Step 10: 機能実装 — チケット一覧

| ID | タスク | 担当 | 依存 | 状態 | チケット |
|----|--------|------|------|------|---------|
| 10-A | 認証 | backend-developer, frontend-developer | Step 9 完了 | 完了 | `tickets/step10/10-A-auth.md` |
| 10-B | レポート | backend-developer, frontend-developer | 10-A | 完了 | `tickets/step10/10-B-report.md` |
| 10-C | ダッシュボード | backend-developer, frontend-developer | 10-A | 完了 | `tickets/step10/10-C-dashboard.md` |
| 10-D | テナント管理 | backend-developer, frontend-developer | 10-A | 完了 | `tickets/step10/10-D-tenant.md` |
| 10-E | 明細 | backend-developer, frontend-developer | 10-B | 完了 | `tickets/step10/10-E-items.md` |
| 10-F | ワークフロー | backend-developer, frontend-developer | 10-B | 完了 | `tickets/step10/10-F-workflow.md` |
| 10-G | 添付ファイル | backend-developer, frontend-developer | 10-E | 完了 | `tickets/step10/10-G-attachments.md` |
| 10-H | CI 安定化 + リファクタリング | backend-developer, frontend-developer | 10-A〜10-G | 完了 | `tickets/step10/10-H-ci-stabilization.md` |
| 10-X | 横断レビュー | reviewer (codex) | 10-H | 完了 | `tickets/step10/10-X-cross-review.md` |
