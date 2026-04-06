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

## 再レビュー結果（2026-04-06）

対応不十分（差し戻し）。

### 確認内容
- `ai-dev-framework/templates/docs-v2/55_ui_component/screens/component-design-template.md` には今回 `表示条件・認可` セクション追加と状態カテゴリ例の修正が入っている
- 一方で `ai-dev-framework/templates/docs-v2/60_test/test_cases/test-cases-template.md` のテストケース表は依然として `対象コンポーネント` や `対象 Props / Hook` の専用欄を持たない
- `55_ui_component` 側にも、テストケースから一意に参照するための設計識別子規約は追加されていない
- `dev-journal/issues/open/ops-056-step5.5-downstream-wb-update.md` には対応計画が記載されている

### 差し戻し理由
- issue 化は妥当だが、元指摘で問題にしていたテンプレート上の欠落自体は未解消のまま残っている
- Step 5.5 work-breakdown の完了条件と品質ゲートは、Step 6/9 がコンポーネント単位・Props 単位で追跡できることを要求しており、現状のテンプレートだけでは判定基準がまだ執筆者依存になる
- したがって、この指摘単体をクローズできる状態には達していない

### 追加で確認すべきファイルや観点
- `ai-dev-framework/templates/docs-v2/60_test/test_cases/test-cases-template.md` に FE テスト追跡欄を追加するか
- もしくは `ai-dev-framework/templates/docs-v2/55_ui_component/` と Step 6 側テンプレートで、`対応設計ID` の記法を相互に明文化すること
- `ai-dev-framework/guide/work-breakdown/step6-testing.md` など下流 work-breakdown 側で、Step 5.5 成果物を入力として受ける契約が整備されること

### このまま進めた場合の下流影響
- Step 6 の FE テストケース補強で、どのコンポーネント / Props を根拠にしたケースかをテンプレート上で統一記述できない
- Step 9 実装者がコンポーネント単位・Props 単位のテスト対象を機械的に拾えず、完了条件の判定がぶれる

## 再レビュー結果（2026-04-06・再確認）

対応妥当（クローズ）。

### 確認内容
- `ai-dev-framework/templates/docs-v2/60_test/test_cases/test-cases-template.md` の FE テストケース表に `対象コンポーネント`、`対象 Props / Hook`、`対応設計ID` の運用を受ける列が追加されている
- 同テンプレートに「対応設計ID」の記法として `55_ui_component/screens/{screen-name}.md §{ComponentName}` および `55_ui_component/state-management.md §{useHookName}` が明文化されている
- `ai-dev-framework/templates/docs-v2/55_ui_component/screens/component-design-template.md` に `## 9. テスト追跡用 設計識別子` が追加され、テストケース側と同じ参照形式が定義されている

### クローズ判断
- 元指摘で問題としていた「FE テストケースから Step 5.5 成果物をコンポーネント単位・Props / Hook 単位で追跡するためのテンプレート契約欠落」は解消された
- Step 5.5 work-breakdown が要求する Step 6/9 への追跡可能性を、テンプレート上で一貫した書式として判定できる状態に戻っている
