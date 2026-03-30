# 083: workflow 一覧の applicant_name フィルタが repository / sqlc まで届いていない

## 指摘概要
workflow 一覧 API は `applicant_name` の部分一致フィルタを OpenAPI で公開しています。`domain.WorkflowListParams` にも `ApplicantName` がある一方、repository 実装と sqlc クエリは cursor と limit しか扱っていません。このままだと Step 10 で query 追加と sqlc 再生成が必要になり、スケルトンの受け渡しが未完成です。

## 根拠
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:1384-1391`
  - `GET /api/workflow/pending` に `applicant_name` query parameter が定義されている。
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:1581-1588`
  - `GET /api/workflow/payable` にも `applicant_name` query parameter が定義されている。
- `expense-saas/internal/domain/repository.go:28-35`
  - `WorkflowListParams` には `ApplicantName *string` が定義されている。
- `expense-saas/internal/repository/postgres/report_repo.go:287-306,318-337`
  - `ListPending` / `ListPayable` は `ApplicantName` を参照せず、sqlc へも渡していない。
- `expense-saas/db/queries/expense_reports.sql:76-90`
  - `ListPendingReports` / `ListPayableReports` の SQL に applicant 名での JOIN / WHERE が存在しない。

## 判定
中 / FIX

## 修正方針案
- workflow 一覧クエリに `users` との JOIN と `name ILIKE ...` 条件を追加し、sqlc を再生成する。
- service interface でも `applicant_name` を受け取る一覧 params を定義し、handler -> service -> repository の流れを固定する。
- カーソル pagination と併用した場合の並び順・検索条件をコメントで明示する。
