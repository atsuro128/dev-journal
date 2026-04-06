# 096: Step5.5 画面別設計書が共通コンポーネント Props を重複定義している

## 指摘概要
Step 5.5 の正本テーブルでは共通コンポーネント Props の正本を `common-components.md` に固定し、`screens/*.md` は参照のみ許可されている。しかし実際の画面別設計書では、共有コンポーネントの Props interface を各画面で再定義している。現時点では `PageTitleProps`、`FormAlertProps`、`SelfLabelProps` などは偶然一致しているが、今後の修正で画面間ドリフトが起きても検知しにくく、横断レビューの負荷と下流実装の判断コストを増やす。

## 根拠
- `ai-dev-framework/guide/work-breakdown/step5.5-ui-component.md:70-72`
  - `screens/*.md` は「全画面共通のコンポーネント詳細（common-components.md が正本）」を含めない
- `ai-dev-framework/guide/work-breakdown/step5.5-ui-component.md:90-94`
  - 「共通コンポーネント Props | 55_ui_component/common-components.md | 55_ui_component/screens/*.md」
- `dev-journal/deliverables/docs/55_ui_component/screens/admin-all-reports.md:62-77`
  - `PageTitleProps` を画面別設計書側で定義
- `dev-journal/deliverables/docs/55_ui_component/screens/admin-tenant.md:57-70`
  - 同じ `PageTitleProps` を別画面で再定義
- `dev-journal/deliverables/docs/55_ui_component/screens/workflow-pending.md:163-215`
  - `SelfLabelProps` と `PageTitleProps` を再定義
- `dev-journal/deliverables/docs/55_ui_component/screens/auth-signup.md:78-118`
  - `FormAlertProps` と `SubmitButtonProps` を再定義
- `dev-journal/deliverables/docs/55_ui_component/screens/report-create.md:101-119`
  - `FormAlertProps` を別画面で再定義

## 判定
- 重大度: 中
- 分類: FIX

## 修正方針案
- 共有コンポーネントの Props 定義は `common-components.md` に集約し、画面別設計書は参照記述のみに切り替える
- 画面別設計書には「どの Props をどう使うか」「どの固定値を渡すか」に限定して記述する
- 共有コンポーネントの定義場所と使用箇所の書き分けルールをテンプレート化し、再発を防ぐ
