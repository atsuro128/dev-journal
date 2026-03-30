# 082: workflow 系エンドポイントの updated_at が service interface に入っていない

## 指摘概要
OpenAPI では、`submit` / `approve` / `reject` / `pay` の各操作で楽観的ロック用 `updated_at` が必須です。しかし `ReportService.SubmitReport`、`WorkflowService.ApproveReport`、`RejectReport`、`MarkReportAsPaid` は `updated_at` を受け取れません。現状の interface では Step 10 実装者が 409 Conflict 契約を守れず、service interface 変更が必須になります。

## 根拠
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:906-917`
  - `POST /api/reports/{id}/submit` の request body で `updated_at` が必須。
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:1445-1460`
  - `POST /api/workflow/{id}/approve` の request body で `updated_at` が必須。
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:1524-1529`
  - `POST /api/workflow/{id}/reject` は `RejectRequest` を参照し、`updated_at` 必須。
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:1641-1652`
  - `POST /api/workflow/{id}/pay` の request body で `updated_at` が必須。
- `expense-saas/internal/service/interfaces.go:43-44,73-80`
  - `SubmitReport` は `reportID` のみ、`ApproveReport` は `comment` のみ、`RejectReport` は `reason` のみ、`MarkReportAsPaid` は `reportID` のみを受け取る。
- `expense-saas/db/queries/expense_reports.sql:48-66`
  - `UpdateReportStatus` は WHERE 句で `updated_at = $14` を要求しており、下流では必ず version 値が必要。

## 判定
高 / FIX

## 修正方針案
- `SubmitReportParams` / `ApproveReportParams` / `RejectReportParams` / `PayReportParams` のような request params struct を service 層に追加し、`updated_at` を含める。
- handler もそれに合わせて request body 契約をそのまま service へ渡す形にする。
- 409 Conflict の責務境界を Step 8 の時点で固定する。
