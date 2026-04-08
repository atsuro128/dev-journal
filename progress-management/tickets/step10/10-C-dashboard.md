# ダッシュボード

- 担当: backend-developer, frontend-developer
- 依存: 10-A
- ブランチ: `step10/10-C-dashboard`
- 出力先: expense-saas/ 内のバックエンド・フロントエンドディレクトリ
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| API 契約 | deliverables/docs/50_detail_design/openapi.yaml | ダッシュボードエンドポイント（/dashboard） |
| 画面仕様（ダッシュボード） | deliverables/docs/50_detail_design/screens/dashboard.md | ダッシュボード画面 |
| UI コンポーネント設計（ダッシュボード） | deliverables/docs/55_ui_component/screens/dashboard.md | コンポーネントツリー・Props 型 |
| テストケース（ダッシュボード） | deliverables/docs/60_test/test_cases/dashboard.md | 全テストケース |
| BE テストコード | expense-saas/internal/handler/dashboard_handler_test.go | ハンドラテスト |
| FE テストコード（ページ） | expense-saas/frontend/src/pages/dashboard/__tests__/DashboardPage.test.tsx | ページテスト |
| FE テストコード（部品） | expense-saas/frontend/src/components/dashboard/__tests__/CountCard.test.tsx, MonthlySummaryTable.test.tsx, MyReportCountCards.test.tsx, RecentReportList.test.tsx, RecentReportRow.test.tsx, TenantStatusCards.test.tsx | コンポーネントテスト |
| FE テストコード（Hooks） | expense-saas/frontend/src/hooks/__tests__/useDashboard.test.tsx | ダッシュボード Hook |

## 責務

- ダッシュボード集計 API の機能実装（ロール別集計データの返却）
- ダッシュボード画面の機能実装（カウントカード、月次サマリ、最近のレポート一覧）
- 含めない: レポート CRUD（10-B）、テナント管理（10-D）

## 完了条件

- test_cases/dashboard.md の全テストケースが通過している
- Step 9 で実装済みのダッシュボード関連テストコード（BE/FE）が全て PASS する
