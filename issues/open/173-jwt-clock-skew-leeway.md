# JWT 検証にクロックずれ許容（leeway）を追加する

## 発見日
2026-05-07

## カテゴリ
backend / security / e2e-test-blocker

## 影響度
中（11-C E2E テストの 1 件失敗の真因。本番では稀だが「クライアントとサーバー間の時刻ずれ秒単位」で同種の 401 が発生しうるため、本番品質にも影響）

## 発見経緯

11-C E2E テスト CRS-066（フロー2 - 却下→再申請）が 9/10 PASS のうち 1 件 FAIL。
trace.zip 解析の結果、**Docker コンテナの時刻が ~18 秒巻き戻り、JWT iat が「未来発行」と判定されて 401**。

trace タイムライン（同一 access token を使用、JWT iat=1778120227）:

| クライアント送信時刻 | サーバ Date ヘッダ | エンドポイント | status |
|---|---|---|---|
| 17:08.231 | 02:17:08 GMT | GET /api/auth/me | 200 |
| 17:09.759 | 02:17:09 GMT | GET /api/reports/:id | 200 |
| **17:15.283** | **02:16:57 GMT (~18 秒巻戻り)** | POST /api/reports/:id/submit | **401** |
| 17:15.305 | 02:16:57 GMT | POST /api/auth/refresh | 401 |

`expense-saas/internal/domain/auth.go:183` の `gojwt.ParseWithClaims` に `gojwt.WithIssuedAt()` が指定されているため、`iat > now` で `ErrTokenExpired` 相当エラー → `ErrInvalidToken` 返却 → 401。

refresh も同様に refresh token の iat が未来扱いで 401 → frontend `client.ts:34` で `window.location.href = '/login'`。

## 関連ステップ

Step 11-C（E2E テスト）の CRS-066 失敗の真因。本 issue 解消後に PR #138 を 10/10 PASS でクローズ可能。

## ブロッカー

- PR #138（11-C E2E テスト）の最終 PASS

## 問題

### 現状の実装

`internal/domain/auth.go:166-186` の `parseToken`:

```go
token, err := gojwt.ParseWithClaims(
    tokenString,
    claims,
    func(t *gojwt.Token) (any, error) { /* signing key 検証 */ },
    gojwt.WithIssuedAt(),         // iat 検証を有効化
    gojwt.WithIssuer("expense-saas"),
    gojwt.WithExpirationRequired(),
)
```

`WithIssuedAt()` は iat クレーム検証を有効化するが、**leeway（許容誤差）が未指定**のため、サーバー時刻 < iat になった瞬間 invalid token 扱いになる。同様に exp 検証も leeway なし。

### なぜ問題か

- **E2E テスト**: Docker Desktop on WSL2 のクロックドリフトで 401 が発生し、フレークの原因となる
- **本番**: モバイル端末のクロックずれ、サーバ間の時刻同期遅延（NTP 同期前後の数秒）で同種の 401 が稀に発生しうる
- **業界標準**: RFC 7519 §4.1.4 は leeway の使用を明示的に許可。Auth0、AWS Cognito の SDK はデフォルトで 60 秒の leeway を設定

## 提案

### JWT 検証に leeway 60 秒を追加

`internal/domain/auth.go:183` の `parseToken` に `gojwt.WithLeeway(60 * time.Second)` を追加:

```go
token, err := gojwt.ParseWithClaims(
    tokenString,
    claims,
    func(t *gojwt.Token) (any, error) { /* ... */ },
    gojwt.WithIssuedAt(),
    gojwt.WithIssuer("expense-saas"),
    gojwt.WithExpirationRequired(),
    gojwt.WithLeeway(60 * time.Second),  // ← 追加
)
```

`WithLeeway` は exp / nbf / iat 全てに適用される（golang-jwt/jwt/v5 の挙動）。

### leeway 値の根拠（60 秒）

- **RFC 7519 §4.1.4**: 実装が leeway を提供することを明示的に許可
- **実環境のクロックずれ**: NTP 同期されていれば <1s。NTP 未同期 / Docker Desktop on WSL2 / モバイル端末では数秒〜数十秒のドリフトが現実的（今回 18 秒を観測）
- **セキュリティ影響**:
  - exp leeway 60s: 期限切れトークンを 60 秒だけ余分に使える。access token TTL=15 分の運用でリスク影響度は微小
  - iat leeway 60s: 未来発行トークンを許容。トークン発行は自サーバーのみのため、攻撃面は実質ゼロ
  - refresh token については DB の `is_revoked` チェックで二重防御済み（leeway 経由の reuse 攻撃は revoke 検知で防止される）
- **業界相場**: Auth0 / AWS Cognito ともに 60 秒推奨

### 設計書反映

- `security.md` §2.1 の「JWT 検証フロー」に「`WithLeeway(60s)` を全クレームに適用」を追記
- 根拠（クロックずれ吸収・RFC 準拠）を脚注で明記

### テスト

`internal/domain/auth_test.go` に追加:

| テストID | テストケース | 入力 | 期待 |
|---|---|---|---|
| AUTH-081 | アクセストークン iat 30 秒未来 | iat = now + 30s | PASS |
| AUTH-082 | アクセストークン iat 61 秒未来 | iat = now + 61s | `ErrInvalidToken` |
| AUTH-083 | アクセストークン exp 30 秒過去 | exp = now - 30s | PASS |
| AUTH-084 | アクセストークン exp 61 秒過去 | exp = now - 61s | `ErrTokenExpired` |
| AUTH-085 | リフレッシュトークン iat 30 秒未来 | iat = now + 30s | PASS |
| AUTH-086 | リフレッシュトークン exp 30 秒過去 | exp = now - 30s | PASS |
| AUTH-087 | リフレッシュトークン iat 61 秒未来 | iat = now + 61s | `ErrInvalidToken` |
| AUTH-088 | リフレッシュトークン exp 61 秒過去 | exp = now - 61s | `ErrTokenExpired` |

ID 採番: `test_cases/auth.md` の既存最大 AUTH-080 → 新規 AUTH-081〜088 を §1.2「JWT 生成・検証」表末尾に追加。

## 影響範囲（事前評価）

- 既存の認証関連テスト: 全て PASS のまま（leeway は緩める方向のみで、既存テストの「正常系は PASS、異常系は FAIL」を破らない）
- 本番動作: 不変（クロックが同期されている環境では leeway は使われない）
- セキュリティ: トークン窃取後の使用可能時間が最大 60 秒延長されるが、access token TTL=15 分・refresh token DB 検証ありの設計でリスク影響度は微小

## 対応優先度

11-C のブロッカー（10/10 PASS の最後の 1 件）かつ本番品質改善のため、**MVP 内で対応必須**。

## 対応方針

別 PR（小規模、変更行数 ~5）で対応:

1. designer: `security.md` §2.1 に leeway 仕様追記
2. backend-developer: `internal/domain/auth.go` + `auth_test.go` 修正、worktree 内で実装
3. test_cases/auth.md に新規 ID 追加（実装後に対応）
4. PR フローで /test → reviewer → codex → マージ
5. PR #138（11-C）に master 取り込み → E2E 再実行で 10/10 PASS 確認

## 関連 issue / PR

- 関連 PR: #138（11-C E2E テスト、本 issue のブロック対象）
- 設計参照: `security.md` §2.1（JWT 検証フロー）、`internal/domain/auth.go:166-186`（parseToken）
- 真因解析: 本 session の trace.zip（test-results/flow2_test.ts-CRS-066-.../trace.zip）
