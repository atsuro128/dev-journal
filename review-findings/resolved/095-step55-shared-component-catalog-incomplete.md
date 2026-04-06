# 095: Step5.5 共通コンポーネント正本に共有 UI コンポーネントが未収録

## 指摘概要
`common-components.md` は「複数画面で共有するコンポーネントの一覧・Props 定義・使用箇所」を定義する正本とされているが、実際には複数画面で共有される `PageTitle`、`FormAlert`、`SubmitButton`、`SelfLabel`、`FilterResetButton`、`AuthNavLinks`、`ReportForm` などが未収録で、使用箇所マトリクスにも現れない。これにより共通 Props の単一正本が成立しておらず、Step 6/9 のテスト対象特定と Step 10 の実装着手時に参照先が分散する。

## 根拠
- `dev-journal/deliverables/docs/55_ui_component/common-components.md:7-8`
  - 「複数画面で共有するコンポーネントの一覧・Props 定義・使用箇所を定義する」
  - 「正本情報 | 共通コンポーネント名・Props 型定義・配置パス・使用箇所マトリクス」
- `dev-journal/deliverables/docs/55_ui_component/common-components.md:372-388`
  - 使用箇所マトリクスに `PageTitle` / `FormAlert` / `SubmitButton` / `SelfLabel` / `FilterResetButton` / `AuthNavLinks` / `ReportForm` が存在しない
- `dev-journal/deliverables/docs/55_ui_component/screens/admin-all-reports.md:62-77`
  - `components/ui/PageTitle.tsx` を「他画面と共通利用可能」として定義
- `dev-journal/deliverables/docs/55_ui_component/screens/workflow-pending.md:163-215`
  - `components/ui/SelfLabel.tsx` と `components/ui/FilterResetButton.tsx` を共通として定義
- `dev-journal/deliverables/docs/55_ui_component/screens/auth-signup.md:78-150`
  - `components/ui/FormAlert.tsx`、`components/ui/SubmitButton.tsx`、`pages/auth/AuthNavLinks.tsx` を定義
- `dev-journal/deliverables/docs/55_ui_component/screens/report-create.md:60-168`
  - `components/report/ReportForm.tsx` を SCR-RPT-002 / SCR-RPT-003 共有として定義
- `ai-dev-framework/guide/work-breakdown/step5.5-ui-component.md:46-54,92`
  - 共通コンポーネント Props の正本は `55_ui_component/common-components.md`

## 判定
- 重大度: 中
- 分類: FIX

## 修正方針案
- `common-components.md` に、実際に複数画面で共有されるコンポーネントを追加する
- 追加対象ごとに、配置パス・責務・Props interface・使用箇所画面 ID を明記する
- 使用箇所マトリクスを更新し、全 13 画面横断で共有コンポーネントの参照漏れがない状態にする

---

## 再レビュー結果（2026-04-06）

- `common-components.md` 本文には `PageTitle`、`FormAlert`、`SubmitButton`、`SelfLabel`、`FilterResetButton`、`AuthNavLinks`、`ReportForm`、`ReportPeriodField`、`ReportFormActions` の 9 コンポーネント追加を確認
- ただし、`common-components.md` の「4. 使用箇所マトリクス」には `ReportPeriodField` と `ReportFormActions` の行が存在しない
- `screens/report-create.md` と `screens/report-edit.md` では両コンポーネントを `common-components.md` 参照の共通コンポーネントとして使用しており、正本の使用箇所マトリクスが未完成のまま

### 未解消の論点

- 使用箇所マトリクスが「追加済み 9 コンポーネント」を網羅できていない

### 追加で確認すべきファイルや観点

- `dev-journal/deliverables/docs/55_ui_component/common-components.md`
  - `ReportPeriodField`
  - `ReportFormActions`
  - 「4. 使用箇所マトリクス」

### このまま進めた場合の下流影響

- Step 6/9 のテスト対象特定、および Step 10 の実装時に、共有コンポーネントの使用画面をマトリクスだけで追跡できない
