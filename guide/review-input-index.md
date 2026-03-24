# レビュー入力インデックス

## 目的

reviewer が毎回広範囲を探索しなくても、対象 Step のレビューに必要な入力へ最短で到達できるようにする。

この文書は**参照インデックス**であり、完了条件・レビュー観点・依存関係の正本ではない。
それらの判定は、必ず対象 Step の work-breakdown を正本として行うこと。

- 正本: `guide/work-breakdown/step*.md`
- 本文書の役割: 主要成果物と上流参照先の入口整理
- 禁止事項: 完了条件・レビュー観点・判断基準をここに重複記載しない

## 使い方

1. 対象 Step の work-breakdown を読む
2. 本文書で、主要成果物と必読の上流資料を特定する
3. 必要に応じて関連 issue / review findings を確認する
4. 最終判定は work-breakdown の完了条件・レビュー観点に従う

## Step 別インデックス

| Step | 主成果物 | 必読の上流資料 | 任意参照 | 関連確認先 |
|------|----------|----------------|----------|------------|
| 0 事前準備 | `deliverables/docs/00_goals.md`, `01_glossary.md`, `02_scope.md` | なし | `references/glossary.md`, `references/links.md` | `progress-management/issues/`, `review-findings/` |
| 1 要件定義 | `deliverables/docs/10_requirements/` | `deliverables/docs/00_goals.md`, `01_glossary.md`, `02_scope.md` | `references/glossary.md` | `progress-management/issues/`, `review-findings/` |
| 2 ドメイン設計 | `deliverables/docs/20_domain/` | `deliverables/docs/10_requirements/requirements.md`, `usecases.md`, `workflow.md`, `rbac.md` | `references/glossary.md` | `progress-management/issues/`, `review-findings/` |
| 3 アーキテクチャ設計 | `deliverables/docs/30_arch/` | `deliverables/docs/10_requirements/`, `deliverables/docs/20_domain/` | `deliverables/docs/30_arch/adr/` | `progress-management/issues/`, `review-findings/` |
| 4 基本設計 | `deliverables/docs/40_basic_design/` | `deliverables/docs/10_requirements/usecases.md`, `deliverables/docs/20_domain/state_machine.md`, `deliverables/docs/30_arch/architecture.md` | `deliverables/docs/30_arch/diagrams.md` | `progress-management/issues/`, `review-findings/` |
| 5 詳細設計 | `deliverables/docs/50_detail_design/` | `deliverables/docs/10_requirements/rbac.md`, `deliverables/docs/20_domain/`, `deliverables/docs/30_arch/`, `deliverables/docs/40_basic_design/` | `deliverables/docs/50_detail_design/screens/`, `ui-guidelines.md` | `progress-management/issues/`, `review-findings/` |
| 6 テスト設計 | `deliverables/docs/60_test/` | `deliverables/docs/10_requirements/`, `deliverables/docs/20_domain/`, `deliverables/docs/50_detail_design/` | `.claude/rules/testing.md` | `progress-management/issues/`, `review-findings/` |
| 7 基盤構築 | `expense-saas/` の基盤コード・設定 | `deliverables/docs/30_arch/architecture.md`, `deliverables/docs/50_detail_design/openapi.yaml`, `db_schema.md`, `authz.md`, `security.md`, `monitoring.md`, `deliverables/docs/60_test/test_strategy.md` | `progress-management/task-plans/step7-foundation.md`, `guide/parallel-branch-operation.md` | `progress-management/issues/open/`, `review-findings/` |
| 8 テストコード実装 | `expense-saas/` のテストコード | `deliverables/docs/60_test/test_strategy.md`, `deliverables/docs/60_test/test_cases/`, Step 7 成果物 | `progress-management/task-plans/`, `guide/parallel-branch-operation.md` | `progress-management/issues/open/`, `review-findings/` |
| 9 機能実装 | `expense-saas/` の機能コード | `deliverables/docs/50_detail_design/`, `deliverables/docs/60_test/test_cases/`, Step 7 成果物, Step 8 成果物 | `progress-management/task-plans/`, `guide/parallel-branch-operation.md` | `progress-management/issues/open/`, `review-findings/` |
| 10 システムテスト・UAT | システムテスト成果物、UAT 記録、運用系成果物 | Step 7, Step 8, Step 9 成果物一式 | `progress-management/task-plans/`, `progress-management/session-log.md` | `progress-management/issues/open/`, `review-findings/` |

## 補足ルール

- Step 名・成果物構成・分割は将来変わりうる。変更時は本文書を最小差分で更新する
- 本文書は「入口」だけを持つ。レビュー観点の詳細化は各 Step の work-breakdown 側で行う
- reviewer は本文書だけで判定してはならない
