# 共通 UI コンポーネント実装

- 担当: frontend-developer
- 依存: 8-3, 8-5
- ブランチ: `step8/8-7-common-ui-components`
- 出力先: expense-saas/frontend/src/components/
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| 共通コンポーネント設計 | deliverables/docs/55_ui_component/common-components.md | 全体（Props 定義・配置パス・使用箇所マトリクス） |
| UI ガイドライン | deliverables/docs/50_detail_design/ui-guidelines.md | MUI テーマ設定・レスポンシブ対応 |
| 画面一覧 | deliverables/docs/40_basic_design/screens.md | §4.1 レイアウト構成 |

## 責務

- レイアウトコンポーネント（AppLayout, AuthLayout, AppHeader, AppSidebar）
- UI ガイドラインのカスタムコンポーネント（AppDataGrid, AppTextField, AppSelect, AppDatePicker）
- 業務共通コンポーネント（ConfirmDialog, AppToast, AppPagination, StatusChip, EmptyState, PageSkeleton, AppBreadcrumbs, PageTitle, FormAlert, SubmitButton, SelfLabel, FilterResetButton, AuthNavLinks）
- レポート関連共通コンポーネント（ReportForm, ReportPeriodField, ReportFormActions）
- 含めない: 画面固有コンポーネント（Step 10）、機能実装（Step 10）

## 完了条件

- common-components.md に定義された全コンポーネントの実装が存在する
- 各コンポーネントの Props 型が common-components.md の TypeScript interface と一致する
- レイアウトコンポーネントがレスポンシブ対応している（AppSidebar: md 以上で permanent, md 未満で temporary）
- AppToast の AuthLayout 配下除外ルールが守られている
- フロントエンドのビルドが通る
