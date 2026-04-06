# 一覧画面のページネーション方式をカーソルベースからオフセットベースに変更

## 発見日
2026-04-06

## カテゴリ
ui-design

## 影響度
高

## 発見経緯
user-report

## 関連ステップ
Step 4（基本設計）、Step 5（詳細設計）、Step 5.5（UI コンポーネント設計）、Step 8（基盤構築）

## ブロッカー
なし

## 問題

一覧画面（レポート一覧、テナント全レポート一覧、承認待ち一覧、支払待ち一覧）のページネーション方式が「カーソルベース + さらに読み込むボタン」で設計されている。この方式では:

1. 「さらに読み込む」を繰り返すとブラウザに全件が蓄積され、画面が重くなる
2. 特定のページに直接飛べない
3. 経費精算のように過去データを検索する業務アプリにはページ番号式の方が適している

## 影響

以下の成果物に変更が必要:

### 上流（正本）
- `40_basic_design/screens.md` — 共通 UI パターンのページネーション定義
- `50_detail_design/screens/report-list.md` — レポート一覧のページネーション仕様
- `50_detail_design/screens/admin-all-reports.md` — テナント全レポート一覧のページネーション仕様
- `50_detail_design/screens/workflow-pending.md` — 承認待ち一覧のページネーション仕様
- `50_detail_design/screens/workflow-payable.md` — 支払待ち一覧のページネーション仕様
- `50_detail_design/openapi.yaml` — クエリパラメータを cursor から page + per_page に変更

### 下流（追従）
- `55_ui_component/state-management.md` — 一覧 Hook は useQuery のまま（useInfiniteQuery 不要）、パラメータ型に page を追加
- `55_ui_component/common-components.md` — ページネーション用の共通コンポーネント追加を検討
- `expense-saas/backend/` — リポジトリ層のカーソル処理をオフセット処理に変更

## 提案

1. screens.md の共通 UI パターンを「オフセットベース + ページ番号」に変更
2. 各画面詳細仕様のページネーションセクションを更新
3. openapi.yaml のクエリパラメータを `cursor` から `page` + `per_page` に変更
4. state-management.md のパラメータ型を更新（page: number を追加、cursor を削除）
5. バックエンドのリポジトリ層を修正

---

## 解決内容

設計文書 8 ファイルを修正し、カーソルベース + 「さらに読み込む」からオフセットベース + ページ番号コントロールに変更した。

- screens.md §4.9: 共通 UI パターンのページネーション定義を変更
- 個別画面仕様 4 ファイル (report-list, admin-all-reports, workflow-pending, workflow-payable): ページネーションセクションを更新
- openapi.yaml: クエリパラメータを cursor から page + per_page に変更
- state-management.md: パラメータ型を更新 (page: number 追加、cursor 削除)
- common-components.md: AppPagination コンポーネントを追加

加えて、deliverables/ 全体の残存参照 (architecture.md, requirements.md, glossary.md, traceability.md, ADR-0003, workflow.md テストケース, project_summary.md) も修正済み。

### 再対応（Step 8 スケルトンコード修正）

codex レビューの差し戻し指摘に基づき、Step 8 のスケルトンコード 6 ファイルをオフセットベースに修正した。

- `db/queries/expense_reports.sql`: 4 クエリから cursor 条件を削除し OFFSET パラメータに変更。COUNT クエリ 4 本を新規追加
- `internal/repository/postgres/sqlcgen/expense_reports.sql.go`: sqlc 再生成
- `internal/domain/dto.go`: Pagination 構造体を NextCursor/HasMore → CurrentPage/PerPage/TotalCount/TotalPages に変更
- `internal/domain/repository.go`: ReportListParams/WorkflowListParams の Cursor/Limit → Page/PerPage。ReportRepository の List/ListPending/ListPayable 戻り値に int（総件数）を追加
- `internal/repository/postgres/report_repo.go`: cursor 関連コード削除、OFFSET 計算・COUNT クエリ呼び出し実装
- `frontend/src/api/types.ts`: Pagination 型を current_page/per_page/total_count/total_pages に変更

BE `go build ./...` / FE `vite build` ともに通過。

## 解決日
2026-04-06

## レビュー結果

### 2026-04-06 Codex Issue 解決レビュー

- 判定: 差し戻し
- 確認1: `40_basic_design/screens.md`、`50_detail_design/screens/report-list.md`、`50_detail_design/screens/admin-all-reports.md`、`50_detail_design/screens/workflow-pending.md`、`50_detail_design/screens/workflow-payable.md`、`50_detail_design/openapi.yaml`、`55_ui_component/state-management.md`、`55_ui_component/common-components.md` では、一覧画面のページネーション契約が `page` / `per_page` とページ番号 UI に更新されている。
- 指摘1: Step 8 の FE 型出力 `expense-saas/frontend/src/api/types.ts` が依然として `pagination.next_cursor` / `has_more` を保持しており、設計文書の `current_page` / `per_page` / `total_count` / `total_pages` と不一致のまま。これでは Step 10 の FE 実装者がどちらを正本として使うべきか迷う。
- 指摘2: Step 8 の BE スケルトンも未更新で、`expense-saas/internal/domain/dto.go`、`expense-saas/internal/domain/repository.go`、`expense-saas/db/queries/expense_reports.sql`、`expense-saas/internal/repository/postgres/report_repo.go` が `cursor` / `limit` ベースの契約と SQL を維持している。Issue 本文で影響範囲に含めた下流成果物の修正が完了していない。
- まだ解消されていない問題: 設計文書と Step 8 生成物・スケルトンの間でページネーション契約が二重化しており、下流が追加解釈なしで実装を始められない。
- 追加で修正すべき成果物: `expense-saas/frontend/src/api/types.ts`、`expense-saas/internal/domain/dto.go`、`expense-saas/internal/domain/repository.go`、`expense-saas/db/queries/expense_reports.sql`、`expense-saas/internal/repository/postgres/report_repo.go`、必要ならそれに追従する sqlc 生成物。
- 上流修正が必要か: 不要。上流設計文書の方向性はオフセットベースで揃っているため、下流の Step 8 / 実装スケルトンを追従修正すればよい。
- そのまま進めた場合に困る工程: Step 10 の FE / BE 機能実装、Step 9 のテスト実装。一覧 API とページネーション UI の契約が文書とコードで食い違い、型・クエリ・テスト期待値を一意に確定できない。
- 再対応方針: Step 8 の生成型・DTO・repository params・SQL・sqlc 生成物をオフセットベースへ揃え、設計文書とコードスケルトンの契約を一致させたうえで再レビューに回すこと。

### 2026-04-06 Codex Issue 解決再レビュー

- 判定: 解決
- 確認1: `expense-saas/db/queries/expense_reports.sql` で一覧 4 クエリの `cursor` 条件が除去され、`LIMIT ... OFFSET ...` と対応する `COUNT` クエリ追加に更新されている。
- 確認2: `expense-saas/internal/domain/dto.go` と `expense-saas/frontend/src/api/types.ts` の Pagination 契約が `current_page` / `per_page` / `total_count` / `total_pages` に統一されている。
- 確認3: `expense-saas/internal/domain/repository.go` の `ReportListParams` / `WorkflowListParams` が `Page` / `PerPage` に更新され、`ReportRepository` の一覧系メソッド戻り値に総件数 `int` が追加されている。
- 確認4: `expense-saas/internal/repository/postgres/report_repo.go` と `expense-saas/internal/repository/postgres/sqlcgen/expense_reports.sql.go` が offset 計算・COUNT 呼び出し・sqlc 生成物まで含めて整合している。
- 波及確認: `expense-saas` 配下の `cursor` / `next_cursor` / `has_more` 残存参照は今回確認範囲では検出されなかった。
- 検証: `cd expense-saas && go test ./internal/...` は通過。`cd expense-saas/frontend && npm run build` も通過。
- 結論: 前回差し戻しの Step 8 スケルトンコード未追従は解消されており、下流実装を阻害する不整合は今回確認した範囲では解消済み。
