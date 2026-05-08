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

## 再レビューコメント（2026-05-08, commit d714373）
- 判定: `FIX` のまま。`ALTER ROLE` への切替自体は正しいが、§5.7.3 が `migrate up` を RDS マスターユーザーで実行する手順に変わったため、`expense_owner` をテーブル owner / RLS bypass 用ロールとして成立させられていない。  
  `dev-journal/progress-management/step11-e-deployment-plan.md:642-645`
- 根拠1: `000002_create_roles.up.sql` が付与しているのは `expense_app` への権限だけで、`expense_owner` への権限付与や ownership 移管はない。  
  `expense-saas/db/migrations/000002_create_roles.up.sql:3-27`
- 根拠2: テスト初期化では `expense_owner` に `GRANT ALL PRIVILEGES` を与えており、プロダクト側も `DATABASE_URL` を owner ロールとして使う前提で実装されている。  
  `expense-saas/scripts/init-db-test.sh:35-39`  
  `expense-saas/cmd/server/main.go:60-82`  
  `expense-saas/cmd/seed/main.go:27-33`
- 根拠3: 上流 runbook でも migration は `expense_owner` で実行する前提になっている。  
  `dev-journal/deliverables/docs/70_operations/release.md:101-119`
- 下流影響: このままでは `DATABASE_URL=expense_owner` での auth/seed/migration 前提が崩れ、少なくとも owner 接続の権限不足または RLS bypass 不成立を招く。`APP_DATABASE_URL` のパスワード不一致だけ解消しても、11-E のスモーク完了条件には到達できない。
- 修正要件: `ALTER ROLE` によるパスワード上書きは維持しつつ、migration 実行主体と owner 権限モデルを再整合させること。少なくとも「`expense_owner` で migrate する」か、「RDS マスターで migrate した後に ownership / owner 権限を `expense_owner` へ明示的に移管する」かを runbook 上で一意に固定する必要がある。
