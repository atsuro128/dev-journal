# テナント管理

- 担当: backend-developer, frontend-developer
- 依存: 10-A
- ブランチ: `step10/10-D-tenant`
- 出力先: expense-saas/ 内のバックエンド・フロントエンドディレクトリ
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| API 契約 | deliverables/docs/50_detail_design/openapi.yaml | テナントエンドポイント（/tenant/*） |
| 認可設計 | deliverables/docs/50_detail_design/authz.md | Admin ロール専用アクセス |
| 画面仕様（テナント管理） | deliverables/docs/50_detail_design/screens/admin-tenant.md | テナント管理画面 |
| 画面仕様（全レポート一覧） | deliverables/docs/50_detail_design/screens/admin-all-reports.md | 全レポート一覧画面 |
| UI コンポーネント設計（管理系） | deliverables/docs/55_ui_component/screens/admin-*.md | コンポーネントツリー・Props 型 |
| テストケース（テナント管理） | deliverables/docs/60_test/test_cases/tenant.md | 全テストケース |
| BE テストコード | expense-saas/internal/handler/tenant_handler_test.go | ハンドラテスト |
| FE テストコード（テナント管理） | expense-saas/frontend/src/pages/admin/__tests__/TenantPage.test.tsx, TenantInfoCard.test.tsx, TenantInfoField.test.tsx, PhaseNotice.test.tsx | テナント管理コンポーネントテスト |
| FE テストコード（全レポート） | expense-saas/frontend/src/pages/admin/__tests__/AllReportsPage.test.tsx, AllReportsFilterBar.test.tsx, AllReportsTable.test.tsx | 全レポートコンポーネントテスト |
| FE テストコード（Hooks） | expense-saas/frontend/src/hooks/__tests__/useTenant.test.tsx, useTenantMembers.test.tsx, useAllReports.test.tsx | テナント関連 Hooks |

## 責務

- テナント参照 API の機能実装（テナント基本情報の返却）
- メンバー一覧 API の機能実装（テナント内ユーザー一覧の返却）
- 全レポート一覧 API の機能実装（Admin/Accounting 向け全レポート一覧）
- テナント管理画面の機能実装（管理者専用、テナント情報・メンバー一覧表示）
- 全レポート一覧画面の機能実装（フィルタ・ソート付き一覧表示）
- 含めない: テナント作成・更新（Signup で作成済み）、レポート CRUD（10-B）

## 完了条件

- test_cases/tenant.md の全テストケースが通過している
- Step 9 で実装済みのテナント関連テストコード（BE/FE）が全て PASS する
