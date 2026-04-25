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
| 6 | テスト設計 | 完了 | 2026-04-12 | `ai-dev-framework/guide/work-breakdown/step6-testing/` |
| 7 | 運用設計 | 完了 | 2026-03-26 | `ai-dev-framework/guide/work-breakdown/step7-operations/` |
| 8 | 基盤構築 | 完了 | 2026-03-31（初回完了）/ 2026-04-12（8-11 追加・再完了） | `ai-dev-framework/guide/work-breakdown/step8-foundation/` |
| 9 | テストコード実装 | 完了 | 2026-04-08 | `ai-dev-framework/guide/work-breakdown/step9-test-implementation/` |
| 10 | 機能実装 | 完了 | 2026-04-11 | `ai-dev-framework/guide/work-breakdown/step10-feature-implementation/` |
| 11 | システムテスト・UAT | 進行中 | - | `ai-dev-framework/guide/work-breakdown/step11-system-test/` |

## タスク状態定義

| 状態 | 意味 | 遷移条件 |
|------|------|----------|
| 未着手 | 未着手 | — |
| 作業中 | サブエージェントが作業中 | 依存先が全て `完了` |
| レビュー待ち | 成果物完成、レビュー未実施 | サブエージェント作業完了 |
| 修正中 | レビュー指摘を対応中 | レビューで FIX 判定 |
| 完了 | レビュー LGTM 済み | 品質ゲート PASS |

## 完了 Step のチケット一覧

Step 5.5 / 6（追加）/ 8 / 9 / 10 のチケット一覧はアーカイブ済み（`archives/progress/steps.md` 参照）。

## Step 11: システムテスト・UAT — チケット一覧

| ID | タスク | 担当 | 依存 | 状態 | チケット |
|----|--------|------|------|------|---------|
| 11-A | ローカル動作確認 | ユーザー + 指揮役 | Step 10 全完了、Step 8-11 完了 | 未着手 | `tickets/step11/11-A-local-verification.md` |
| 11-B | 横断テスト（Go） | test-implementer | 11-A | 未着手 | `tickets/step11/11-B-cross-cutting-test.md` |
| 11-C | E2E テスト（Playwright） | test-implementer | 11-A | 未着手 | `tickets/step11/11-C-e2e-test.md` |
| 11-D | 横断レビュー | reviewer (codex) | 11-B, 11-C | 未着手 | `tickets/step11/11-D-cross-review.md` |
| 11-E | デプロイ・スモークテスト | platform-builder | 11-D | 未着手 | `tickets/step11/11-E-deploy.md` |
| 11-F | UAT | ユーザー | 11-E | 未着手 | `tickets/step11/11-F-uat.md` |

## 課題・ブロッカー

### 解決済み（2026-04-12 〜 2026-04-16）

アーカイブ済み（`archives/progress/issues.md` 参照）。

### 残存 issue（Step 11-A 関連）
| ID | タイトル | 起票日 | 状態 |
|----|---------|-------|------|
| 133 | ログ・エラー出力の言語ポリシー整理（FE/BE 共通、日本語メッセージの棚卸しと削減） | 2026-04-21 | 別セッションで対応予定 |
| 140 | フォーム必須マーカー（`*`）と HTML5 `required` 属性の不統一を全フォームで解消 | 2026-04-22 | 未着手 |
| 145 | ReportWorkflowInfo セクション見出し追加（post-MVP） | 2026-04-24 | 起票のみ（post-MVP） |
| 146 | 大規模クロステナント環境での性能テスト計画（post-MVP） | 2026-04-25 | 起票のみ（post-MVP） |
| 147 | per_page UI セレクタ実装 + 4 画面適用 + URL/UI 整合 + テスト追加 | 2026-04-25 | 未着手 |

### 残存 issue（運用・基盤系）
| ID | タイトル |
|----|---------|
| ops-055 | work-breakdown テンプレート不整合 |
| 060 | devcontainer egress allowlist の厳格化と根拠の欠如 |
| 061 | devcontainer マウントとシークレット露出の最小化 |
| ops-062 | ワークフロースキルの粒度 |
| 064 | MCP ジョブログが proxy allowlist でブロック |
| ops-080 | Post-MVP スコープ管理方法 |
| 081 | Post-MVP テストカバレッジ項目 |
| 084 | Post-MVP HttpOnly Cookie 移行 |
| 104 | UI 層の表示・振る舞いカバレッジ監査（ロール別表示制御 + レスポンシブ） |
| 122 | Post-MVP セッション期限切れ時のリダイレクト UX 改善（サイレント → インラインアラート + return_to） |
