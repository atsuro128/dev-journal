# 080: repository が TenantContext の固定接続を使っていない

## 指摘概要
`TenantContext` ミドルウェアは、リクエスト単位で DB 接続を `Acquire` し、`BEGIN` と `set_config('app.current_tenant', ...)` を実行した上で、その接続を context に格納しています。しかし 8-6 で追加された PostgreSQL repository 実装は、context 内の接続を使わず毎回 `sqlcgen.New(r.pool)` でプールを直接参照しています。これでは、8-4 で固定した「TenantContext で pin した接続上で RLS/トランザクションを効かせる」契約が repository 層で崩れます。

## 根拠
- `expense-saas/internal/middleware/tenant.go:10-13,25-27,47-59,64-69`
  - `TenantContext` は接続 acquire、transaction begin、`app.current_tenant` 設定、`SetConn(ctx, conn)`、レスポンス成功時 commit を行う。
- `expense-saas/internal/repository/postgres/report_repo.go:34,50,65,127`
  - `reportRepo` は各メソッドで `sqlcgen.New(r.pool)` を生成し、context の接続を使っていない。
- `expense-saas/internal/repository/postgres/item_repo.go:34,57,73,90,121`
  - `itemRepo` も同様に `sqlcgen.New(r.pool)` を使っている。
- 同種の実装は `attachment_repo.go`、`category_repo.go`、`membership_repo.go`、`password_reset_repo.go`、`refresh_token_repo.go`、`tenant_repo.go`、`user_repo.go` にも存在する。
- `ai-dev-framework/guide/work-breakdown/step8-foundation.md`
  - Step 8 の固定事項として「データアクセス層でのテナント分離強制ルール」が明記されている。

## 判定
高 / FIX

## 修正方針案
- repository / sqlc の実行先を `pool` 固定にせず、context に接続があればそれを優先する共通ヘルパーを導入する。
- もしくは repository 自体を `DBTX` 受け取りに変え、middleware が pin した接続 / transaction を確実に通す。
- あわせて、複数クエリを跨ぐ `SoftDelete` や `UpdateReportTotalAmount` 連動処理が同一 transaction 上で動くことを明示する。
