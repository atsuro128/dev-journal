# 019: 明細・添付の論理削除方針に対してスキーマ属性が不足している

## 指摘概要
Step 2 ではレポート削除時に `ExpenseItem` と `Attachment` も論理削除すると定義しているが、両エンティティに `deleted_at` が定義されていない。設計どおりに実装できない。

## 根拠
- `state_machine.md` は T5 の事後処理として、関連 `ExpenseItem` / `Attachment` の論理削除を要求している
  - `dev-journal/deliverables/docs/20_domain/state_machine.md:186-188`
- `usecases.md` でもレポート削除時は「明細・添付も連動」としている
  - `dev-journal/deliverables/docs/10_requirements/usecases.md:200-206`
- `domain_model.md` では `ExpenseReport` には `deleted_at` がある一方、`ExpenseItem` と `Attachment` には存在しない
  - `dev-journal/deliverables/docs/20_domain/domain_model.md:156-177`
  - `dev-journal/deliverables/docs/20_domain/domain_model.md:185-193`
  - `dev-journal/deliverables/docs/20_domain/domain_model.md:203-211`
- データ要件は `deleted_at` 方式の論理削除を前提としている
  - `dev-journal/deliverables/docs/10_requirements/requirements.md:436`

## 判定
削除設計とエンティティ定義の不整合（中）。

## 修正方針案
`ExpenseItem` / `Attachment` に `deleted_at` を追加するか、子要素は物理削除にする方針へ変更して関連文書を統一する。どちらを採るかを Step 2 で確定させる必要がある。
