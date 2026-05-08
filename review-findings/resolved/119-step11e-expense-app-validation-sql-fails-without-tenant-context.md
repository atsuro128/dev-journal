# 119: Step11-E の `expense_app` 検証 SQL は `app.current_tenant` 未設定のままでは成立しない

## 指摘概要
§5.7.3 Step 6 は `expense_app` 接続で `SELECT count(*) FROM tenants;` / `expense_reports;` を実行し、DML 権限の実地検証を行う手順になっている。  
しかしこれらのテーブルは RLS policy が `current_setting('app.current_tenant')::uuid` を前提としており、アプリ本体でもテスト基盤でも事前に `set_config('app.current_tenant', ...)` を入れてからアクセスしている。  
runbook の Step 6 にはその設定手順がないため、`expense_app` の direct `psql` 実行は権限の有無に関係なく `app.current_tenant` 未設定で失敗し、受け入れゲートが false negative になる。

## 根拠
- Step 6 は `expense_app` で `SELECT count(*) FROM tenants;` / `users;` / `expense_reports;` をそのまま流し、「seed 前なので 0 行が返ること」を期待している。  
  `dev-journal/progress-management/step11-e-deployment-plan.md:786-818`
- しかし `tenants` と `expense_reports` の RLS policy は `current_setting('app.current_tenant')::uuid` をそのまま参照しており、missing_ok 指定がない。  
  `expense-saas/db/migrations/000003_create_tenants.up.sql:12-21`  
  `expense-saas/db/migrations/000007_create_expense_reports.up.sql:43-60`
- 実装側も `expense_app` 接続では毎回 `SELECT set_config('app.current_tenant', $1, true)` を先に実行してから業務クエリを流している。  
  `expense-saas/internal/middleware/tenant.go:46-55`
- テスト基盤も同じ前提で tenant context を先に設定している。  
  `expense-saas/internal/testutil/db.go:73-90`
- さらに Step 6 自身が「seed 前」を前提にしているため、既知 tenant ID を使った検証フローも現状の runbook には存在しない。  
  `dev-journal/progress-management/step11-e-deployment-plan.md:791-797`

## 判定
重大度: 高  
分類: blocker

## 修正方針案
- Step 6 の `expense_app` 実地検証は、RLS 対象テーブルへの素の `SELECT count(*)` ではなく、少なくとも以下のどちらかに変更すること。
- 例1: `information_schema.role_table_grants` / `has_table_privilege()` を使って、対象テーブルごとの `SELECT/INSERT/UPDATE/DELETE` をメタデータで検証する。
- 例2: `SELECT set_config('app.current_tenant', '<known-tenant-uuid>', true);` を先に実行できる前提条件を手順に追加し、その tenant context 下で代表テーブルを検証する。
- いずれの場合も、「seed 前なので 0 行が返ること」を受け入れ条件に使う現行記述は削除し、RLS 前提を崩さない検証手順に置き換えること。
