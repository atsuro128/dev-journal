# レビュー: 8-1 開発環境構築 + DB

## 対象コミット
a14a3e6 (初回) → 再レビュー対象: B-001, W-002 の修正コミット

## 判定
PASS

## blocker

### B-001: .gitignore に keys/ ディレクトリが未登録（セキュリティ） — 解消済み

- **対象ファイル**: `expense-saas/.gitignore`
- **問題**: `scripts/generate-keys.sh` が生成する `keys/` ディレクトリ（JWT 秘密鍵 `private.pem` と公開鍵 `public.pem`）が `.gitignore` に含まれていない。秘密鍵がリポジトリに混入するリスクがある。
- **再レビュー**: `.gitignore` 33-34行目に `# JWT keys` コメント付きで `keys/` が追加されている。解消を確認。

## warning

### W-001: DB ロール作成が2箇所に分散

- **対象ファイル**: `db/migrations/000002_create_roles.up.sql`, `scripts/init-db.sh`
- **問題**: `expense_app` ロールの作成と権限付与が両方に存在。`IF NOT EXISTS` で冪等だが、パスワード（`localdev`）が2箇所にハードコードされており変更時に整合性が崩れるリスク。

### W-002: down マイグレーションで expense_owner を DROP するリスク — 解消済み

- **対象ファイル**: `db/migrations/000002_create_roles.down.sql`
- **問題**: `DROP ROLE IF EXISTS expense_owner;` が含まれているが、expense_owner はインフラ管理のロール。マイグレーション全戻し時に DB 接続不能になる。
- **再レビュー**: 該当行が削除され、コメント `-- ロール削除（expense_owner はインフラ管理のため対象外）` が追加されている。解消を確認。

## note

### N-001: フロントエンドサービスがコメントアウト状態

8-3（フロントエンド初期化）の責務のため、意図的な未実装と判断。

### N-002: Makefile に generate-keys ターゲットがない

8-2 以降のタスクで追加可能。

### N-003: テーブル定義・RLS・インデックス・シードデータの整合性は問題なし

全9テーブルのカラム・型・制約・FK・RLS ポリシー・インデックス・シードデータが db_schema.md と完全一致。
