# 118: c 案では `expense_app` に本番テーブルの DML 権限が付与されずアプリ接続が成立しない

## 指摘概要
Step11-E の F-115 再修正版（c 案）は、000001/000002 を RDS マスターで実行したあと 000003 以降を `expense_owner` で実行する。  
しかし `000002_create_roles.up.sql` にある `expense_app` 向け権限付与は、(a) `GRANT ... ON ALL TABLES IN SCHEMA public` 実行時点ではまだ業務テーブルが存在せず実質空振り、(b) `ALTER DEFAULT PRIVILEGES IN SCHEMA public` も **その SQL を実行したロール** が後続で作成するテーブルにしか効かない。  
今回の c 案では 000003 以降のテーブル作成主体が `expense_owner` に切り替わるため、`expense_app` は本番テーブル群への `SELECT/INSERT/UPDATE/DELETE` を自動取得できない。結果として `APP_DATABASE_URL` で起動する通常 API リポジトリ接続が権限不足で失敗する。

## 根拠
- `000002_create_roles.up.sql` が `expense_app` に付与しているのは `USAGE ON SCHEMA public`、その時点の既存テーブルへの DML 権限、そして default privileges のみである。  
  `expense-saas/db/migrations/000002_create_roles.up.sql:21-27`
- c 案では 000001/000002 をマスターで実行し、000003 以降の `CREATE TABLE` は `expense_owner` 接続へ切り替える。  
  `dev-journal/progress-management/step11-e-deployment-plan.md:612-617, 651-699`
- 計画書自身も Step 3 で「000003〜000011 で CREATE TABLE されるテーブルの owner はすべて `expense_owner`」と説明している。  
  `dev-journal/progress-management/step11-e-deployment-plan.md:688-705`
- 一方アプリ本体は通常系 repository を `APP_DATABASE_URL` で張った `expense_app` プールに載せており、こちらのロールに DML 権限が無ければ業務 API は動作しない。  
  `expense-saas/cmd/server/main.go:52-58, 113-122`
- それにもかかわらず計画書は「GRANT 系（`expense_app` への DML 権限・ALTER DEFAULT PRIVILEGES）は 000002 で完了済みのため Step 2 / Step 3 では再投入不要」と断定している。これは c 案の実行主体分割と整合していない。  
  `dev-journal/progress-management/step11-e-deployment-plan.md:722-729`

## 判定
重大度: 高  
分類: blocker

## 修正方針案
- `expense_owner` に切り替える前または直後に、少なくとも以下を runbook へ明示すること。
- `ALTER DEFAULT PRIVILEGES FOR ROLE expense_owner IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO expense_app;`
- Step 3 完了後に既存テーブルへ `GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO expense_app;`
- 可能なら疎通確認も `SELECT 1` だけでなく、`expense_app` で代表テーブルに対する `SELECT` を追加して DML 権限を検証すること。
