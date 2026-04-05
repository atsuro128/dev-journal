# プロジェクト進捗管理

## マイルストーン

| # | マイルストーン | 状態 | 完了日 | ガイド |
|---|--------------|------|--------|--------|
| 0 | 事前準備（プロジェクトの土台づくり） | 完了 | 2026-03-04 | `ai-dev-framework/guide/work-breakdown/step0-preparation.md` |
| 1 | 要件定義（業務理解 → ユースケース化） | 完了 | 2026-03-09 | `ai-dev-framework/guide/work-breakdown/step1-requirements.md` |
| 2 | ドメイン設計（データとルールの核） | 完了 | 2026-03-14 | `ai-dev-framework/guide/work-breakdown/step2-domain.md` |
| 3 | アーキテクチャ設計（技術選定・構成決定） | 完了 | 2026-03-16 | `ai-dev-framework/guide/work-breakdown/step3-architecture.md` |
| 4 | 基本設計（画面一覧・画面遷移） | 完了 | 2026-03-22 | `ai-dev-framework/guide/work-breakdown/step4-basic-design.md` |
| 5 | 詳細設計（API・DB・認可・セキュリティ） | 完了 | 2026-03-23 | `ai-dev-framework/guide/work-breakdown/step5-detail-design.md` |
| 5.5 | UI コンポーネント設計 | 未着手 | - | `ai-dev-framework/guide/work-breakdown/step5.5-ui-component.md` |
| 6 | テスト設計 | 完了 | 2026-03-23 | `ai-dev-framework/guide/work-breakdown/step6-testing.md` |
| 7 | 運用設計 | 完了 | 2026-03-26 | `ai-dev-framework/guide/work-breakdown/step7-operations.md` |
| 8 | 基盤構築 | 完了 | 2026-03-31 | `ai-dev-framework/guide/work-breakdown/step8-foundation.md` |
| 9 | テストコード実装 | 未着手 | - | `ai-dev-framework/guide/work-breakdown/step9-test-implementation.md` |
| 10 | 機能実装 | 未着手 | - | `ai-dev-framework/guide/work-breakdown/step10-feature-implementation.md` |
| 11 | システムテスト・UAT | 未着手 | - | `ai-dev-framework/guide/work-breakdown/step11-system-test.md` |

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
| 8-7 | テスト基盤 | backend-developer | 8-6 | 完了 | `tickets/step8/8-7-test-infra.md` |
| 8-8 | CI/CD パイプライン | platform-builder | 8-2, 8-3 | 完了 | `tickets/step8/8-8-cicd.md` |
| 8-9 | 開発者ツール | platform-builder | 8-2, 8-3 | 完了 | `tickets/step8/8-9-dev-tools.md` |
| 8-10 | 整理 | platform-builder | なし | 完了 | `tickets/step8/8-10-cleanup.md` |

## 課題・ブロッカー
`issues/open/` を参照。
