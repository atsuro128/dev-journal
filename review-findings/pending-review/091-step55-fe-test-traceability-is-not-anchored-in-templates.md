# 091: Step 5.5 の FE テスト追跡要件がテンプレートに落ちていない

## 指摘概要
work-breakdown では、Step 5.5 の成果として FE テストケースに「コンポーネント名と Props 型への参照」を含め、Step 9 がコンポーネント単位・Props 単位で実装できることを品質ゲートにしています。しかし、既存の `test-cases-template.md` にはそれを受ける専用欄がなく、`55_ui_component` 側のテンプレートにもテストから参照するための安定した設計識別子の規約がありません。現状だと追跡方法が執筆者依存になり、完了条件の判定がぶれます。

## 根拠
- `ai-dev-framework/guide/work-breakdown/step5.5-ui-component.md:125`-`126`
  - FE テストケース追加と、Step 9 実装者がコンポーネント単位のテスト対象を特定できることを完了条件にしている。
- `ai-dev-framework/guide/work-breakdown/step5.5-ui-component.md:132`-`136`
  - 品質ゲートで「コンポーネント単位・Props 単位」での FE テスト実装可能性を求めている。
- `ai-dev-framework/guide/work-breakdown/step5.5-ui-component.md:314`-`320`
  - 5.5-E で各テストケースに「対応するコンポーネント名と Props 型への参照」を含めること、テストケースからコンポーネント設計への追跡可能性を要求している。
- `ai-dev-framework/templates/docs-v2/60_test/test_cases/test-cases-template.md:36`-`40`
  - テストケース表に `対象コンポーネント` や `対象 Props / Hook` を明示する欄がない。
- `ai-dev-framework/templates/docs-v2/55_ui_component/screens/component-design-template.md:44`-`58`
  - コンポーネント名と Props 表はあるが、テスト側から安定して参照するための設計 ID 規約がない。

## 判定
- 重大度: 中
- 分類: FIX

## 関連 issue
- `issues/open/ops-056-step5.5-downstream-wb-update.md` の項目 5 で対応予定

## 修正方針案
- どちらかを必須化する。
  - `test-cases-template.md` に `対象コンポーネント`、`対象 Props / Hook`、`設計参照` の列を追加する。
  - または `対応設計ID` 列に記載すべき書式を Step 5.5 / Step 6 のテンプレートで明文化する。
- `55_ui_component` 側では、各コンポーネントや Hook をテストケースから一意に参照できる識別子または記法を定義する。
