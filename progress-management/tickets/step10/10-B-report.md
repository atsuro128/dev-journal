# レポート

- 担当: backend-developer, frontend-developer
- 依存: 10-A
- ブランチ: `step10/10-B-report`
- 出力先: expense-saas/ 内のバックエンド・フロントエンドディレクトリ
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| API 契約 | deliverables/docs/50_detail_design/openapi.yaml | レポートエンドポイント（/reports/*） |
| DB スキーマ | deliverables/docs/50_detail_design/db_schema.md | expense_reports, categories |
| 認可設計 | deliverables/docs/50_detail_design/authz.md | RBAC・テナント分離 |
| 状態遷移設計 | deliverables/docs/20_domain/state_machine.md | レポート状態遷移 |
| 画面仕様（レポート一覧） | deliverables/docs/50_detail_design/screens/report-list.md | レポート一覧画面 |
| 画面仕様（レポート作成） | deliverables/docs/50_detail_design/screens/report-create.md | レポート作成画面 |
| 画面仕様（レポート詳細） | deliverables/docs/50_detail_design/screens/report-detail.md | レポート詳細画面 |
| 画面仕様（レポート編集） | deliverables/docs/50_detail_design/screens/report-edit.md | レポート編集画面 |
| UI コンポーネント設計（レポート系） | deliverables/docs/55_ui_component/screens/report-*.md | コンポーネントツリー・Props 型 |
| テストケース（レポート） | deliverables/docs/60_test/test_cases/reports.md | 全テストケース |
| BE テストコード | expense-saas/internal/domain/report_test.go, internal/handler/report_handler_test.go, internal/handler/category_handler_test.go | レポート・カテゴリテスト |
| FE テストコード（ページ） | expense-saas/frontend/src/pages/__tests__/ReportListPage.test.tsx, ReportCreatePage.test.tsx, ReportDetailPage.test.tsx, ReportEditPage.test.tsx | ページテスト |
| FE テストコード（部品） | expense-saas/frontend/src/pages/reports/__tests__/*.test.tsx, components/report/__tests__/*.test.tsx | コンポーネントテスト |
| FE テストコード（Hooks） | expense-saas/frontend/src/hooks/__tests__/useMyReports.test.tsx, useCreateReport.test.tsx, useReport.test.tsx, useUpdateReport.test.tsx, useDeleteReport.test.tsx, useSubmitReport.test.tsx, useCategories.test.tsx | レポート関連 Hooks |

## 責務

- レポート CRUD API の機能実装（作成・取得・一覧・更新・削除・提出）
- 状態遷移ロジックの実装（下書き → 提出済みの遷移、不正遷移のバリデーション）
- レポート一覧・作成・編集・詳細画面の機能実装
- カテゴリ取得 API の機能実装
- 含めない: 明細 CRUD（10-E）、ワークフロー（10-F）、添付ファイル（10-G）

## 完了条件

- test_cases/reports.md の全テストケースが通過している
- 不正な状態遷移で 422 を返す
- Step 9 で実装済みのレポート関連テストコード（BE/FE）が全て PASS する
