# 081: レポート一覧 API のフィルタが service interface に露出していない

## 指摘概要
OpenAPI では `/api/reports` に `status` / `from` / `to`、`/api/reports/all` に加えて `submitter_id` が定義されています。一方で `ReportService` の一覧メソッドは `cursor` と `limit` しか受け取れず、handler から service へこれらのフィルタを渡す契約が存在しません。repository 層には `ReportListParams` と sqlc クエリが既に用意されているため、service interface だけがボトルネックになっています。

## 根拠
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:562-581`
  - `GET /api/reports` に `status` / `from` / `to` の query parameter が定義されている。
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:668-693`
  - `GET /api/reports/all` に `status` / `from` / `to` / `submitter_id` の query parameter が定義されている。
- `expense-saas/internal/service/interfaces.go:35-38`
  - `ListMyReports(ctx, actor, cursor, limit)` / `ListAllReports(ctx, actor, cursor, limit)` で、上記フィルタを受け取れない。
- `expense-saas/internal/domain/repository.go:10-26`
  - repository 側には `ReportListParams` として `Status` / `From` / `To` / `SubmitterID` が定義済み。
- `expense-saas/db/queries/expense_reports.sql:12-33`
  - sqlc クエリにも対応する optional filter 条件が含まれている。

## 判定
高 / FIX

## 修正方針案
- `ReportService` に一覧用の params struct を追加し、`status` / `from` / `to` / `submitter_id` / `cursor` / `limit` を service 境界で固定する。
- handler はその params struct を組み立てて service に渡し、service は `domain.ReportListParams` へ変換するだけで済む形にする。
- Step 10 が interface 変更から始めなくて済むよう、このチケット内でシグネチャを固める。
