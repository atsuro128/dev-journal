# ワークフロー

- 担当: backend-developer, frontend-developer
- 依存: 10-B
- ブランチ: `step10/10-F-workflow`
- 出力先: expense-saas/ 内のバックエンド・フロントエンドディレクトリ
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| API 契約 | deliverables/docs/50_detail_design/openapi.yaml | ワークフローエンドポイント（/workflow/*） |
| 認可設計 | deliverables/docs/50_detail_design/authz.md | Approver・Accounting ロールの権限 |
| 状態遷移設計 | deliverables/docs/20_domain/state_machine.md | 提出済み → 承認済み → 支払済み、却下 |
| 画面仕様（承認キュー） | deliverables/docs/50_detail_design/screens/workflow-pending.md | 承認待ちキュー画面 |
| 画面仕様（支払キュー） | deliverables/docs/50_detail_design/screens/workflow-payable.md | 支払待ちキュー画面 |
| UI コンポーネント設計（ワークフロー系） | deliverables/docs/55_ui_component/screens/workflow-*.md | コンポーネントツリー・Props 型 |
| テストケース（ワークフロー） | deliverables/docs/60_test/test_cases/workflow.md | 全テストケース |
| BE テストコード | expense-saas/internal/handler/workflow_handler_test.go | ハンドラテスト |
| FE テストコード（ページ） | expense-saas/frontend/src/pages/__tests__/ApprovalListPage.test.tsx, PaymentListPage.test.tsx | ページテスト |
| FE テストコード（部品） | expense-saas/frontend/src/pages/reports/__tests__/WorkflowActions.test.tsx, ReportWorkflowInfo.test.tsx | コンポーネントテスト |
| FE テストコード（Hooks） | expense-saas/frontend/src/hooks/__tests__/usePendingReports.test.tsx, useApproveReport.test.tsx, useRejectReport.test.tsx, usePayableReports.test.tsx, useMarkAsPaid.test.tsx | ワークフロー関連 Hooks |

## 責務

- 申請・承認・却下・支払 API の機能実装
- 承認キュー取得 API、支払キュー取得 API の機能実装
- 状態遷移ロジックの実装（提出済み → 承認済み / 却下、承認済み → 支払済み）
- RBAC 認可チェック（Approver のみ承認・却下可、Accounting のみ支払可）
- 承認キュー画面、支払キュー画面の機能実装
- 承認/却下アクション、却下理由入力の実装
- 含めない: レポート CRUD（10-B）、明細（10-E）、添付ファイル（10-G）

## 完了条件

- test_cases/workflow.md の全テストケースが通過している
- 権限外ロールで 403 を返す
- 不正な状態遷移で 422 を返す
- Step 9 で実装済みのワークフロー関連テストコード（BE/FE）が全て PASS する
