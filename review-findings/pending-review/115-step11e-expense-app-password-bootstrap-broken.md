# 115: Step11-E の expense_app 初期化手順では APP_DATABASE_URL が成立しない

## 指摘概要
`step11-e-deployment-plan.md` の DB 初期化手順は、`migrate up` 実行後に `expense_app` を `IF NOT EXISTS` で作成する前提になっているが、実際には `000002_create_roles.up.sql` が `expense_app` を `localdev` パスワードで先に作成する。  
そのため後続の `CREATE ROLE expense_app WITH LOGIN PASSWORD '${APP_DB_PW}'` は実行されず、`APP_DATABASE_URL` に設定した本番用パスワードと実 DB ロールのパスワードが一致しない。結果としてアプリ起動後のアプリロール接続が失敗し、11-E 完了条件の疎通・スモークに進めない。

## 根拠
- 計画書は `APP_DATABASE_URL=postgres://expense_app:<pw>@...` を前提にしつつ、「migrate 時に CREATE ROLE expense_app する」と記載している。  
  `dev-journal/progress-management/step11-e-deployment-plan.md:247`
- 同計画書の手順では `migrate up` の後に、`IF NOT EXISTS` 付きで `expense_app` を作成している。  
  `dev-journal/progress-management/step11-e-deployment-plan.md:530-550`
- しかし実マイグレーション `000002_create_roles.up.sql` は `expense_app` を `PASSWORD 'localdev'` で作成する。  
  `expense-saas/db/migrations/000002_create_roles.up.sql:12-27`
- localdev 用 `init-db.sh` も同様に `localdev` パスワードで `expense_app` を作るだけであり、計画書の「prod 等価版を後から手動投入する」前提ではパスワード更新にならない。  
  `expense-saas/scripts/init-db.sh:8-26`
- `env_config.md` でも prod は `APP_DATABASE_URL` を正しいアプリロール接続情報として注入する前提。  
  `dev-journal/deliverables/docs/70_operations/env_config.md:89-96`

## 判定
重大度: 高  
分類: blocker

## 修正方針案
- `expense_app` の初期化方式を 1 つに固定すること。
- 例1: RDS 作成時の管理ユーザーで `ALTER ROLE expense_app WITH PASSWORD '...'` を必ず実行する手順に改める。
- 例2: 本番用ロール作成を migration ではなく Terraform / bootstrap SQL に寄せ、`000002_create_roles.up.sql` の localdev 依存を本番手順から切り離す。
- いずれの場合も、`DATABASE_URL` / `APP_DATABASE_URL` に設定する資格情報と、実際に DB 上へ作成・更新されるロール資格情報の対応を runbook 上で一意に示すこと。
