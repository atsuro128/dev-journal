# 認証系エンドポイントで expense_owner 専用 DB 接続が未使用

## 発見日
2026-04-11

## カテゴリ
implementation

## 影響度
中

## 発見経緯
review

## 関連ステップ
Step 10（機能実装）

## ブロッカー
なし

## 問題

サーバは `expense_app` ユーザーの単一 DB pool のみを作成しており、認証系エンドポイント（signup / login / refresh / logout）も同じ pool を使用している。

db_schema.md §DB ユーザー設計では、認証系エンドポイントは `expense_owner`（RLS バイパス可能）の専用接続を使う前提で設計されている。理由は以下の通り:

- signup 時: `tenants` テーブルへの INSERT は RLS ポリシーが定義されておらず、`expense_app` ユーザーでは失敗する可能性がある
- login / refresh 時: `tenant_memberships` / `tenants` の RLS 評価時に `app.current_tenant` が未設定で失敗する可能性がある

ただし **CI テスト（signup → login → refresh → logout の一連フロー含む）は全 PASS** しているため、テスト環境での RLS 設定が本番前提と異なる可能性がある。

### 関連ファイル
- `cmd/server/main.go` 53行, 94行, 118行
- `internal/service/auth_service.go` 86行, 171行, 223行
- `db/queries/tenant_memberships.sql` 6行（`GetMembershipByUserID` に tenant_id 条件なし）
- `internal/repository/postgres/refresh_token_repo.go` 62行（`RevokeAllByUserID` が pool 直接利用）

## 影響

- 本番環境で RLS が正しく適用された場合、signup / login / refresh が失敗する可能性がある
- テスト環境では superuser 接続や RLS 無効化により問題が顕在化していない可能性がある

## 提案

1. テスト環境の DB ユーザーと RLS 設定を調査し、本番環境との差異を明確にする
2. 必要に応じて `expense_owner` 用の DB pool を追加し、認証系サービスに注入する
3. `GetMembershipByUserID` に tenant_id 条件を追加するか、owner 接続前提の設計を明文化する
4. `RevokeAllByUserID` を `queries(ctx, pool)` 経由に変更する

対応は Step 11（システムテスト）で本番相当の環境テスト時に確認・修正する。

---

## 解決内容

## 解決日
