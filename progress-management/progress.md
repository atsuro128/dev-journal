# プロジェクト進捗管理

## マイルストーン

| # | マイルストーン | 状態 | 完了日 | ガイド |
|---|--------------|------|--------|--------|
| 0 | 事前準備（プロジェクトの土台づくり） | 完了 | 2026-03-04 | `ai-dev-framework/guide/work-breakdown/step0-preparation/` |
| 1 | 要件定義（業務理解 → ユースケース化） | 完了 | 2026-03-09 | `ai-dev-framework/guide/work-breakdown/step1-requirements/` |
| 2 | ドメイン設計（データとルールの核） | 完了 | 2026-03-14 | `ai-dev-framework/guide/work-breakdown/step2-domain/` |
| 3 | アーキテクチャ設計（技術選定・構成決定） | 完了 | 2026-03-16 | `ai-dev-framework/guide/work-breakdown/step3-architecture/` |
| 4 | 基本設計（画面一覧・画面遷移） | 完了 | 2026-03-22 | `ai-dev-framework/guide/work-breakdown/step4-basic-design/` |
| 5 | 詳細設計（API・DB・認可・セキュリティ） | 完了 | 2026-03-23 | `ai-dev-framework/guide/work-breakdown/step5-detail-design/` |
| 5.5 | UI コンポーネント設計 | 完了 | 2026-04-06 | `ai-dev-framework/guide/work-breakdown/step5.5-ui-component/` |
| 6 | テスト設計 | 完了 | 2026-03-23 | `ai-dev-framework/guide/work-breakdown/step6-testing/` |
| 7 | 運用設計 | 完了 | 2026-03-26 | `ai-dev-framework/guide/work-breakdown/step7-operations/` |
| 8 | 基盤構築 | 完了 | 2026-03-31 | `ai-dev-framework/guide/work-breakdown/step8-foundation/` |
| 9 | テストコード実装 | 完了 | 2026-04-08 | `ai-dev-framework/guide/work-breakdown/step9-test-implementation/` |
| 10 | 機能実装 | 進行中 | - | `ai-dev-framework/guide/work-breakdown/step10-feature-implementation/` |
| 11 | システムテスト・UAT | 未着手 | - | `ai-dev-framework/guide/work-breakdown/step11-system-test/` |

## タスク状態定義

| 状態 | 意味 | 遷移条件 |
|------|------|----------|
| 未着手 | 未着手 | — |
| 作業中 | サブエージェントが作業中 | 依存先が全て `完了` |
| レビュー待ち | 成果物完成、レビュー未実施 | サブエージェント作業完了 |
| 修正中 | レビュー指摘を対応中 | レビューで FIX 判定 |
| 完了 | レビュー LGTM 済み | 品質ゲート PASS |

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
| 8-11 | 整理 | platform-builder | なし | 完了 | `tickets/step8/8-11-cleanup.md` |

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
| 10-G | 添付ファイル | backend-developer, frontend-developer | 10-E | 未着手 | `tickets/step10/10-G-attachments.md` |
| 10-X | 横断レビュー | reviewer (codex) | 10-A〜10-G | 未着手 | `tickets/step10/10-X-cross-review.md` |

## 課題・ブロッカー
`issues/open/` を参照。
