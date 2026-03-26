# 066: テストケース表の必須列が全体的に欠落している

## 指摘概要
Step 6 の test case 正本には、各テストケースごとに `保証種別`・`対応要件ID`・`対応設計ID` を記載する必要がある。しかし実際の `test_cases/*.md` は `テストID / テストレベル / レイヤー / テスト関数名候補 / 入力 / 期待結果` だけで構成されており、Step 6 の完了条件と FAIL 条件を満たしていない。

## 根拠
- `ai-dev-framework/guide/work-breakdown/step6-testing.md:78-87`
  - test_cases/*.md の必須項目として `保証種別` `対応要件ID` `対応設計ID` を要求。
- `ai-dev-framework/guide/work-breakdown/step6-testing.md:118-124`
  - 各テストケースに `対応要件ID` `対応設計ID` `保証種別` を記載することをトレーサビリティ要件として明記。
- `ai-dev-framework/guide/work-breakdown/step6-testing.md:136-152`
  - 完了条件および FAIL 条件で同要件を再度必須化。
- `dev-journal/deliverables/docs/60_test/test_strategy.md:314-325`
  - 全テストケースファイルの統一列定義に、`保証種別` `対応要件ID` `対応設計ID` が含まれていない。
- `dev-journal/deliverables/docs/60_test/test_cases/auth.md:48-56`
  - 実表の列は 6 列のみで、必須 3 列が存在しない。
- `dev-journal/deliverables/docs/60_test/test_cases/workflow.md:58-68`
  - 実表でも同様に必須 3 列が欠落。

## 判定
高 / 品質ゲート違反

## 修正方針案
`test_strategy.md` の列定義を work-breakdown に合わせて修正し、すべての `test_cases/*.md` の表に `保証種別` `対応要件ID` `対応設計ID` を追加する。各ケースを要件・設計セクションに直接ひも付け、traceability.md に依存しなくても単体で追跡できる状態にする。
