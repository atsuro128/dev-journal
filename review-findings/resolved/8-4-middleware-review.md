# 再レビュー結果: 8-4 共通ミドルウェア + ヘルスチェック

- 初回レビュー日: 2026-03-30
- 再レビュー日: 2026-03-30
- レビュアー: reviewer
- 対象: Step 8 タスク 8-4
- 判定: **PASS**

## 判定理由

前回指摘した blocker 2 件、warning 4 件、info 1 件のうち、要対応とされた 7 件すべてが適切に修正されている。修正により新たな問題は発生していない。`go build ./...` および `go vet ./...` がクリーンに通過。`go mod tidy` の差分なし。

---

## 修正確認結果

| # | 前回指摘 | 前回重大度 | 修正内容 | 確認結果 |
|---|---------|----------|---------|---------|
| 1 | TenantContext SQL インジェクション + UUID 未検証 | **blocker** | (a) `jwt.go` L80-85: `Verify()` 内で `claims.UserID` と `claims.TenantID` の UUID バリデーションを `uuid.Parse()` で追加。不正形式はエラーを返す。(b) `tenant.go` L53: `fmt.Sprintf` による文字列補間を `SELECT set_config('app.current_tenant', $1, true)` のパラメータ化クエリに変更。`set_config` の第三引数 `true` は `SET LOCAL` と同等のトランザクションスコープ動作を保証。 | **解消** |
| 3 | ルートグループの枠不足 | **blocker** | 指揮役判定: PASS with NOTE。8-6 のチケットが auth ハンドラのルート定義を担当する。`main.go` L97 にコメント「Handlers will be registered in step 8-6.」が追加され、責務の所在が明確化。 | **解消（指揮役判定に従う）** |
| 4 | X-RateLimit-* ヘッダー未実装 | **warning** | `ratelimit.go` L89-99: `setRateLimitHeaders()` 関数を追加。`X-RateLimit-Limit`（設定上限）、`X-RateLimit-Remaining`（`math.Floor(l.Tokens())` で残トークン数を算出）、`X-RateLimit-Reset`（ウィンドウ終了の UNIX タイムスタンプ）の 3 ヘッダーを全レスポンスに付与。`RateLimitByIP` (L118) と `RateLimitByUser` (L152) の両方で、Allow 判定の前にヘッダーを設定している。security.md SS4.3 の仕様と合致。 | **解消** |
| 5 | Logger msg が "request" | **warning** | `logger.go` L68,70,72: `slog.Info/Warn/Error` の第一引数を `"request"` から `"request completed"` に変更。monitoring.md SS2.2 の仕様と合致。 | **解消** |
| 6 | Cache-Control: no-store 未実装 | **warning** | `security_headers.go` L34-36: `/api/` プレフィックスのパスに対して `Cache-Control: no-store` ヘッダーを設定。`strings.HasPrefix(r.URL.Path, "/api/")` で判定。security.md SS6.3 の仕様と合致。 | **解消** |
| 8 | CORS ワイルドカードフォールバック | **warning** | `cors.go` L14-16: 空値時のフォールバックを `"*"` から `"http://localhost:5173"` に変更。security.md SS5.2 のワイルドカード禁止ルール（SEC-013）に準拠。開発環境のデフォルト値として妥当。 | **解消** |
| 9 | go.mod の direct/indirect 不整合 | **info** | `go mod tidy` 実行済み。`go-chi/cors`, `golang-jwt/jwt/v5`, `google/uuid`, `golang.org/x/time` が direct 依存として正しく分類。`go mod tidy -diff` で差分なし。 | **解消** |
| 12 | UserID UUID 未検証 | **warning** | #1(a) と合わせて `jwt.go` L80-82 で `claims.UserID` の UUID バリデーションを実装。 | **解消** |

---

## 前回指摘のうち対応不要だったもの（info）

以下は前回 info として記録したもので、修正対象外。再レビューでの確認は不要。

| # | 内容 | 前回重大度 | 状態 |
|---|------|----------|------|
| 2 | RateLimit 配置位置が architecture.md SS3.2 と異なる | info | 変更なし。security.md SS4.4 準拠の妥当な実装。 |
| 7 | ErrorBody の details フィールドが architecture.md SS3.5 に未定義 | info | 変更なし。合理的な拡張。 |
| 10 | ヘルスチェックが共通レスポンス形式の適用外 | info | 変更なし。 |
| 11 | COMMIT 失敗時のリカバリ戦略 | info | 変更なし。MVP として許容。 |

---

## 修正により新たに発生した問題

なし。

---

## ビルド・静的解析結果

| チェック | 結果 |
|---------|------|
| `go build ./...` | クリーン（エラーなし） |
| `go vet ./...` | クリーン（警告なし） |
| `go mod tidy -diff` | 差分なし |

---

## 完了条件の充足状況

| 完了条件 | 状態 | 備考 |
|---------|------|------|
| ミドルウェアが architecture.md SS3.2 の順序で適用される | **合格** | RateLimit の配置は security.md SS4.4 に準拠した合理的な実装。 |
| ヘルスチェックが DB ステータスを返す | **合格** | `pool.Ping()` で DB 接続を確認し、結果を返している。 |
| 認証不要ルート（login, signup, health 等）がスキップされる | **合格** | `/health` は登録済み。auth 関連ルートは 8-6 チケットの責務（指揮役判定）。 |
| SQL インジェクション対策 | **合格** | パラメータ化クエリ + UUID バリデーションの二重防御。 |
| セキュリティヘッダーが security.md 準拠 | **合格** | Cache-Control: no-store が /api/ パスに追加。 |
| レート制限ヘッダーが security.md SS4.3 準拠 | **合格** | X-RateLimit-* 3 ヘッダーが全レスポンスに付与。 |
| CORS がワイルドカード禁止ルール準拠 | **合格** | フォールバックが localhost:5173 に変更。 |
