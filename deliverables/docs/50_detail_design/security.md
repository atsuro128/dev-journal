# セキュリティ設計

## この文書の役割

| 項目 | 内容 |
|------|------|
| 目的 | 認証とセキュリティ基盤の実装仕様を定義する |
| 正本情報 | JWT、パスワードハッシュ、レート制限、CORS、セキュリティヘッダー、エラーコード |
| 扱わない内容 | 認可責務の詳細（authz.md）、業務ルールの正本（policies.md） |
| 主な参照元 | `10_requirements/requirements.md`, `10_requirements/policies.md`, `30_arch/architecture.md` |
| 主な参照先 | `50_detail_design/authz.md`, `60_test/test_cases/auth.md`, `60_test/test_cases/cross-cutting.md` |

## 1. 概要

本書は経費精算SaaS のセキュリティに関する詳細設計を定義する。`architecture.md` SS6（セキュリティアーキテクチャ）の多層防御方針を具体的な実装仕様に落とし込む。

### 参照ドキュメント

| ドキュメント | 参照内容 |
|------------|---------|
| `10_requirements/requirements.md` SS4.2 | セキュリティ要件（SEC-*, TNT-*） |
| `30_arch/architecture.md` SS6 | セキュリティアーキテクチャ、ミドルウェアチェーン |
| `30_arch/adr/0001-tech-stack.md` | JWT ライブラリ（golang-jwt/jwt）、パスワードハッシュライブラリ（alexedwards/argon2id） |
| `30_arch/adr/0003-rls-tenant-isolation.md` | RLS テナント分離方式の判断根拠 |
| `.claude/rules/security-policy.md` | セキュリティポリシー全体 |
| `10_requirements/policies.md` | RBAC（SS3）+ テナント分離（SS6）+ 認可チェックフロー（SS3.9） |
| `deliverables/docs/01_glossary.md` | 用語集 |

---

## 2. 認証

### 2.1 JWT 実装詳細

#### クレーム構造

アクセストークンとリフレッシュトークンの JWT ペイロード構造を定義する。

**アクセストークン**:

```json
{
  "iss": "expense-saas",
  "sub": "<user_id (UUID)>",
  "iat": 1711100000,
  "exp": 1711100900,
  "jti": "<token_id (UUID)>",
  "tenant_id": "<tenant_id (UUID)>",
  "role": "member",
  "token_type": "access"
}
```

| クレーム | 型 | 説明 |
|---------|-----|------|
| iss | String | トークン発行者。固定値 `expense-saas` |
| sub | UUID | ユーザー識別子（user_id） |
| iat | Integer | 発行日時（Unix タイムスタンプ） |
| exp | Integer | 有効期限（iat + 15分）（SEC-003） |
| jti | UUID | トークン一意識別子（トークン無効化に使用） |
| tenant_id | UUID | テナント識別子（TenantContext ミドルウェアで使用） |
| role | String | テナント内ロール（`admin`, `approver`, `member`, `accounting`） |
| token_type | String | トークン種別。固定値 `access` |

**リフレッシュトークン**:

```json
{
  "iss": "expense-saas",
  "sub": "<user_id (UUID)>",
  "iat": 1711100000,
  "exp": 1711704800,
  "jti": "<token_id (UUID)>",
  "token_type": "refresh"
}
```

| クレーム | 型 | 説明 |
|---------|-----|------|
| iss | String | トークン発行者。固定値 `expense-saas` |
| sub | UUID | ユーザー識別子 |
| iat | Integer | 発行日時 |
| exp | Integer | 有効期限（iat + 7日）（SEC-003） |
| jti | UUID | トークン一意識別子 |
| token_type | String | トークン種別。固定値 `refresh` |

**クレーム設計の原則**:

- リフレッシュトークンには `tenant_id` と `role` を含めない。リフレッシュ時に DB から最新の tenant_memberships を再取得し、ロール変更を即座に反映する
- パスワードハッシュ、メールアドレス等の機密情報は一切含めない（security-policy.md SS1）
- `jti` をすべてのトークンに付与し、トークン単位の無効化を可能にする

#### RS256 鍵管理

| 項目 | 仕様 |
|------|------|
| 署名アルゴリズム | RS256（RSA + SHA-256）（SEC-004） |
| 鍵長 | RSA 2048bit 以上 |
| 秘密鍵の保管 | ファイルパスを環境変数（`JWT_PRIVATE_KEY_PATH`）で指定。stg/prod では Secrets Manager から取得した PEM を一時ファイルに配置（env_config.md §5.3 参照） |
| 公開鍵の保管 | ファイルパスを環境変数（`JWT_PUBLIC_KEY_PATH`）で指定。検証のみに使用 |
| 鍵フォーマット | PEM エンコード |
| 鍵ペア生成 | `openssl genrsa -out private.pem 2048` / `openssl rsa -in private.pem -pubout -out public.pem` |

**鍵ローテーション**（Phase 3 で実装）:

MVP では単一鍵ペアで運用する。鍵ローテーション（二鍵並行方式）は Phase 3 で実装する。

| 項目 | 方針 |
|------|------|
| ローテーション周期 | 90日（四半期ごと）— Phase 3 |
| ローテーション方式 | 二鍵並行方式 — Phase 3 |
| 手順 | 1. 新しい鍵ペアを生成 2. 新鍵で署名開始 3. 旧公開鍵での検証を猶予期間（7日）継続 4. 猶予期間後に旧鍵を廃棄 — Phase 3 |
| 鍵識別 | JWT ヘッダーの `kid`（Key ID）で署名鍵を識別 |

JWT ヘッダーの構造:

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "<key_id>"
}
```

#### JWT 検証フロー

Auth ミドルウェアで実行する JWT 検証の手順を定義する。

```
Authorization ヘッダーから Bearer トークンを取得
  |
  v
[1] トークン形式の検証（3パートの Base64URL 文字列）
  | NG -> 401 {"error": {"code": "INVALID_TOKEN", "message": "Invalid token format"}}
  v
[2] ヘッダーの alg が RS256 であることを確認（alg 混乱攻撃の防止）
  | NG -> 401
  v
[3] kid に対応する公開鍵を取得
  | NG -> 401
  v
[4] RS256 署名検証
  | NG -> 401
  v
[5] exp クレームの検証（有効期限切れチェック）
  | NG -> 401 {"error": {"code": "TOKEN_EXPIRED", "message": "Token has expired"}}
  v
[6] iss クレームの検証（"expense-saas" であること）
  | NG -> 401
  v
[7] token_type クレームの検証（"access" であること）
  | NG -> 401
  v
[8] sub, tenant_id, role をコンテキストに設定
  |
  v
検証成功 -> 次のミドルウェアへ
```

**認証不要エンドポイント**（Auth ミドルウェアをスキップ）:

| エンドポイント | 理由 |
|-------------|------|
| `POST /api/auth/signup` | 未認証でのアカウント作成 |
| `POST /api/auth/login` | 未認証でのログイン |
| `POST /api/auth/refresh` | リフレッシュトークンで認証 |
| `POST /api/auth/logout` | リフレッシュトークンで認証（期限切れアクセストークンでもログアウト可能） |
| `POST /api/auth/password-reset` | 未認証でのパスワードリセット要求 |
| `PUT /api/auth/password-reset/:token` | リセットトークンで認証 |
| `GET /health` | ヘルスチェック |

#### トークンリフレッシュフロー

```
クライアント                  サーバー                         DB
  |                           |                              |
  |-- POST /api/auth/refresh ->|                              |
  |   {refresh_token}         |                              |
  |                           |                              |
  |                           | [1] JWT 検証（token_type=refresh） |
  |                           |                              |
  |                           | [2] jti で無効化チェック       |
  |                           |-- SELECT refresh_token ------>|
  |                           |<-- is_revoked ------------------|
  |                           | NG -> 401                    |
  |                           |                              |
  |                           | [3] 最新の membership 取得    |
  |                           |-- SELECT tenant_memberships ->|
  |                           |<-- role, tenant_id -----------|
  |                           |                              |
  |                           | [4] 旧リフレッシュトークンを無効化 |
  |                           |-- UPDATE set is_revoked=true ->|
  |                           |                              |
  |                           | [5] 新しい access_token 生成  |
  |                           |    （最新の role, tenant_id）  |
  |                           | [6] 新しい refresh_token 生成 |
  |                           |    （リフレッシュトークンローテーション）|
  |                           |                              |
  |<-- {access_token, --------|                              |
  |     refresh_token}        |                              |
```

**リフレッシュトークンローテーション**: リフレッシュ実行時に新しいリフレッシュトークンを発行し、旧トークンを無効化する。盗まれたリフレッシュトークンの再利用を検出可能にするため、無効化済みトークンでのリフレッシュ試行が検出された場合は、同一ユーザーの全リフレッシュトークンを無効化する。

#### リフレッシュトークンの DB 管理

リフレッシュトークンの無効化状態を DB で管理する。

| カラム | 型 | 説明 |
|--------|-----|------|
| jti | UUID | PK。JWT の jti クレーム |
| user_id | UUID | FK -> users。トークン所有者 |
| is_revoked | Boolean | 無効化フラグ。ログアウト時・ローテーション時に true |
| expires_at | Timestamp | 有効期限。期限切れレコードの定期削除に使用 |
| created_at | Timestamp | 発行日時 |

**ログアウト処理**: `POST /api/auth/logout` でリクエストボディの `refresh_token` の `jti` に対応するレコードの `is_revoked` を true に更新する。

---

### 2.2 パスワードハッシュ（Argon2id）

ライブラリ: `alexedwards/argon2id`（ADR-0001）

#### パラメータ

OWASP 推奨パラメータに準拠する。

| パラメータ | 値 | 説明 |
|-----------|-----|------|
| Memory | 64 MB (65536 KiB) | メモリコスト。ASIC/GPU 攻撃への耐性 |
| Iterations | 3 | 時間コスト（反復回数） |
| Parallelism | 4 | 並列度 |
| Salt Length | 16 bytes | ソルト長。暗号学的乱数で生成 |
| Key Length | 32 bytes | 出力ハッシュ長 |

**ハッシュ出力形式**: `$argon2id$v=19$m=65536,t=3,p=4$<salt_base64>$<hash_base64>`

**運用方針**:

- パラメータはアプリケーション設定として定義し、将来的な変更に対応可能にする
- パスワード検証時にハッシュのパラメータが現行設定と異なる場合は、検証後に再ハッシュする（パラメータマイグレーション）
- パスワードの平文をログに出力しない（security-policy.md SS1）

### 2.3 パスワードリセット

| 項目 | 仕様 |
|------|------|
| トークン生成 | 暗号学的乱数で 32 bytes 生成し、hex エンコード（64文字） |
| トークン保存 | SHA-256 ハッシュ化して DB に保存（漏洩時の悪用防止） |
| 有効期限 | 発行から 1 時間（SEC-006） |
| 使用回数 | 1回限り。使用後に `used_at` に使用日時を記録し再利用を防止 |
| レスポンス | ユーザーの存在に関わらず同一の成功メッセージを返却（SEC-011） |
| リセット実行時 | 対象ユーザーの全リフレッシュトークンを無効化（セッション強制終了） |

---

## 3. テナント分離

### 3.1 多層防御の構成

テナント分離は ADR-0003 に基づき、アプリケーション層と DB 層の二重保証で実現する。

```
[1] ミドルウェア層 — JWT claims から tenant_id を取得、コンテキストに設定
         |
[2] TenantContext ミドルウェア — BEGIN + SET LOCAL app.current_tenant
         |
[3] リポジトリ層 — 全クエリに WHERE tenant_id = $1 を含める（TNT-002, TNT-003）
         |
[4] DB 層（RLS）— tenant_id = current_setting('app.current_tenant')::uuid
```

### 3.2 テナント境界越えのエラーレスポンス

他テナントのリソースへのアクセスは **404 Not Found** を返却する（403 ではない）。リソースの存在自体を漏洩させない（TNT-006）。

```json
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "Resource not found"
  }
}
```

### 3.3 DB ロール構成

ADR-0003 で定義された 2 ロール構成を採用する。

| DB ロール | 用途 | RLS |
|-----------|------|-----|
| `expense_owner` | テーブルオーナー。マイグレーション + 認証処理（ログイン/サインアップ） | バイパス（ENABLE のみ、FORCE なし） |
| `expense_app` | 業務用。通常リクエストの全クエリ | 適用 |

- `expense_app` ロールは `SET LOCAL app.current_tenant` が設定されたトランザクション内でのみクエリを実行する
- `expense_owner` ロールは認証サービス内でのみ使用し、業務用リポジトリ層には公開しない

### 3.4 S3 テナント分離

S3 オブジェクトキーにテナント ID を含めることで、テナント分離を物理的に保証する。

- キー命名規則: `{tenant_id}/{report_id}/{attachment_id}`
- 署名付き URL 発行前に、リクエストユーザーのテナントとリソースのテナントが一致することを検証する

---

## 4. レート制限

### 4.1 制限値

| 対象 | 制限値 | キー | 根拠 |
|------|--------|------|------|
| 認証済みリクエスト | 100 req/min | user_id | SEC-012 |
| 未認証リクエスト | 20 req/min | IP アドレス | SEC-012 |
| ログイン試行 | 5 req/min | IP アドレス | ブルートフォース対策 |
| ファイルアップロード | 10 req/min | user_id | リソース保護 |

### 4.2 実装方式

| 項目 | 仕様 |
|------|------|
| アルゴリズム | トークンバケット方式 |
| ストレージ | インメモリ（MVP）。Go の `sync.Map` または `golang.org/x/time/rate` |
| キー生成 | 認証済み: `user:{user_id}`、未認証: `ip:{remote_addr}`、ログイン: `login:{remote_addr}`、アップロード: `upload:{user_id}` |
| ウィンドウ | 1分間のスライディングウィンドウ |
| IP 取得 | `X-Forwarded-For` ヘッダーの最左 IP（ALB 経由のため）。ヘッダー不在時は `RemoteAddr` |

### 4.3 制限超過時のレスポンス

```
HTTP/1.1 429 Too Many Requests
Retry-After: 30
Content-Type: application/json

{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Please try again later."
  }
}
```

| ヘッダー | 説明 |
|---------|------|
| `Retry-After` | 再試行可能になるまでの秒数 |
| `X-RateLimit-Limit` | ウィンドウあたりの制限値 |
| `X-RateLimit-Remaining` | ウィンドウ内の残りリクエスト数 |
| `X-RateLimit-Reset` | ウィンドウリセットの Unix タイムスタンプ |

### 4.4 レート制限のミドルウェア配置

ミドルウェアチェーン内の配置位置（architecture.md SS3.2 参照）:

```
CORS -> SecurityHeaders -> RequestID -> Logger -> RateLimit -> Auth -> TenantContext -> RBAC
```

- RateLimit は Auth の前に配置する。未認証リクエストにもレート制限を適用するため
- 認証済みリクエストの場合は Auth ミドルウェア通過後に user_id ベースのレート制限を追加適用する
- ログインエンドポイントには専用のレート制限（5 req/min/IP）を適用する
- ファイルアップロードエンドポイントには専用のレート制限（10 req/min/user）を適用する

---

## 5. CORS ポリシー

### 5.1 設定値

| 項目 | 値 | 根拠 |
|------|-----|------|
| 許可オリジン | 環境変数 `CORS_ALLOWED_ORIGINS` で指定 | SEC-013 |
| 許可メソッド | `GET`, `POST`, `PUT`, `DELETE`, `OPTIONS` | API で使用するメソッドのみ |
| 許可ヘッダー | `Authorization`, `Content-Type`, `X-Request-ID` | 認証 + コンテンツ + トレーサビリティ |
| 公開ヘッダー | `X-Request-ID`, `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `Retry-After` | クライアントから参照が必要なヘッダー |
| 認証情報 | `credentials: true`（`Access-Control-Allow-Credentials: true`） | Cookie は使用しないが、Authorization ヘッダーの送信に必要 |
| プリフライトキャッシュ | `Access-Control-Max-Age: 3600`（1時間） | OPTIONS リクエストの削減 |

### 5.2 環境別設定

| 環境 | CORS_ALLOWED_ORIGINS |
|------|---------------------|
| 開発（ローカル） | `http://localhost:5173`（Vite dev server） |
| ステージング | `https://staging.expense-saas.example.com` |
| 本番 | `https://expense-saas.example.com` |

**禁止事項**:

- ワイルドカード（`*`）の使用は禁止（SEC-013）
- `null` オリジンの許可は禁止
- 本番環境では HTTP オリジンの許可は禁止（HTTPS のみ）

### 5.3 MVP の SPA 同梱方式との関係

MVP では Go コンテナに Vite build 成果物を同梱して配信する（architecture.md SS4.0）。SPA と API が同一オリジンで配信されるため、本番環境では同一オリジンのリクエストには CORS ヘッダーは不要である。ただし、以下の理由で CORS ミドルウェアを実装する:

- 開発環境では Vite dev server（`localhost:5173`）と API サーバー（`localhost:8080`）が別オリジン
- 将来的な S3 + CloudFront 配信への移行に備える

---

## 6. セキュリティヘッダー

### 6.1 ヘッダー一覧

SecurityHeaders ミドルウェアで全レスポンスに付与するヘッダーを定義する。

| ヘッダー | 値 | 説明 | 根拠 |
|---------|-----|------|------|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | HSTS。1年間 HTTPS を強制 | security-policy.md SS6 |
| `X-Content-Type-Options` | `nosniff` | MIME タイプスニッフィング防止 | security-policy.md SS6 |
| `X-Frame-Options` | `DENY` | クリックジャッキング防止。iframe 埋め込み完全禁止 | security-policy.md SS6 |
| `X-XSS-Protection` | `0` | ブラウザの XSS フィルタを無効化。CSP に委ねる | security-policy.md SS6 |
| `Content-Security-Policy` | `default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https://*.amazonaws.com; connect-src 'self'; font-src 'self'; object-src 'none'; frame-ancestors 'none'; base-uri 'self'; form-action 'self'` | コンテンツセキュリティポリシー | security-policy.md SS6 |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | リファラー情報の制御 | プライバシー保護 |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` | ブラウザ機能の制限 | 不要な機能を無効化 |

### 6.2 CSP ディレクティブの詳細

| ディレクティブ | 値 | 理由 |
|-------------|-----|------|
| `default-src` | `'self'` | デフォルトで同一オリジンのみ許可 |
| `script-src` | `'self'` | 同一オリジンのスクリプトのみ。インラインスクリプト禁止 |
| `style-src` | `'self' 'unsafe-inline'` | MUI の Emotion によるインラインスタイルに対応 |
| `img-src` | `'self' data: https://*.amazonaws.com` | 同一オリジン + data URI + S3 署名付き URL からの画像読み込み。プレビュー機能では `window.open` で新タブを開くため、`img-src` による制約はプレビュータブには適用されない（新タブは独立したブラウジングコンテキスト） |
| `connect-src` | `'self'` | API 通信は同一オリジン。S3 ダウンロード/プレビューは `window.open` で新タブを開く方式（`about:blank` -> `location.href` 差し替え）であり、`connect-src` の制約対象外。CSP は新タブの `window.open` による遷移を制限しない |
| `object-src` | `'none'` | プラグインコンテンツを完全禁止 |
| `frame-ancestors` | `'none'` | X-Frame-Options: DENY と同等 |
| `base-uri` | `'self'` | base タグの href 制限 |
| `form-action` | `'self'` | フォーム送信先の制限 |

### 6.3 API レスポンスのヘッダー

API エンドポイント（`/api/*`）のレスポンスには以下を追加する:

| ヘッダー | 値 | 説明 |
|---------|-----|------|
| `Content-Type` | `application/json; charset=utf-8` | JSON レスポンスの明示 |
| `Cache-Control` | `no-store` | 認証情報を含むレスポンスのキャッシュ禁止 |
| `X-Request-ID` | `<request_id (UUID)>` | リクエスト追跡用 |

---

## 7. 入力バリデーション

### 7.1 バリデーション戦略

信頼境界はバックエンド（Go サーバー）に設定する。フロントエンドのバリデーションは UX 向上のためのものであり、セキュリティ保証はバックエンドで行う。

```
[フロントエンド]                   [バックエンド（信頼境界）]
  リアルタイムバリデーション           ハンドラ層: 構造体タグによるバリデーション
  （UX 向上、即時フィードバック）       ドメイン層: ビジネスルールの検証
                                    リポジトリ層: DB 制約による最終保証
```

### 7.2 バリデーション実装

ハンドラ層で `go-playground/validator` を使用した構造体タグベースのバリデーションを実施する。

#### 共通バリデーションルール

| フィールド種別 | バリデーション | 根拠 |
|-------------|-------------|------|
| メールアドレス | `required,email,max=254` | RFC 5321 |
| パスワード | `required,min=8,max=128` | SEC-010 |
| ユーザー名 | `required,min=1,max=100` | requirements.md SS2.1 |
| 会社名 | `required,min=1,max=200` | requirements.md SS2.1 |
| レポートタイトル | `required,min=1,max=200` | RPT-001 |
| 摘要 | `required,min=1,max=500` | ITM-004 |
| 却下理由 | `required,min=1,max=1000` | WFL-012 |
| 承認コメント | `omitempty,max=1000` | 任意 |
| 金額 | `required,gt=0` | ITM-002。正の整数のみ |
| 日付 | `required` + カスタムバリデーション（YYYY-MM-DD 形式） | ITM-001 |
| カテゴリ（category_id） | `required,uuid` + アプリケーション層で categories テーブル存在チェック | ITM-005 |
| UUID | `required,uuid` | ID フィールド共通 |
| ページネーション limit | `omitempty,min=1,max=100` | デフォルト 20 |

### 7.3 SQL インジェクション対策

| 対策 | 実装 |
|------|------|
| パラメータバインディング | sqlc が生成するクエリは全てプレースホルダ（`$1`, `$2`, ...）を使用 |
| 文字列結合禁止 | クエリ文字列の動的構築を禁止。sqlc の型安全なコード生成で保証 |
| ORM 不使用 | ORM の暗黙的なクエリ生成による意図しない SQL 実行のリスクを排除 |

### 7.4 XSS 対策

| 対策 | 実装 |
|------|------|
| React のデフォルトエスケープ | JSX のテキスト出力は自動エスケープ |
| `dangerouslySetInnerHTML` 禁止 | コードレビューで排除。ESLint ルールで検出 |
| CSP | `script-src 'self'` でインラインスクリプト実行を防止 |
| API レスポンス | `Content-Type: application/json` を明示。HTML として解釈されることを防止 |

### 7.5 パストラバーサル対策

| 対策 | 実装 |
|------|------|
| S3 キーの生成 | UUID ベースのキー（`{tenant_id}/{report_id}/{attachment_id}`）。クライアント提供のファイル名を S3 キーに使用しない |
| ファイル名の保存 | DB の `file_name` カラムに元ファイル名を保存するが、S3 アクセスには使用しない |

---

## 8. エラーレスポンスのサニタイズ

### 8.1 基本方針

エラーレスポンスにはシステムの内部実装を含めない。クライアントに返すエラー情報は、ユーザーが次のアクションを判断するのに必要十分な情報に限定する。

### 8.2 エラーレスポンス形式

全エラーレスポンスは以下の統一形式に従う（architecture.md SS5.2）。

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "人間可読なメッセージ",
    "details": [
      {"field": "title", "message": "Title is required"}
    ]
  }
}
```

| フィールド | 型 | 説明 |
|-----------|-----|------|
| code | String | 機械可読なエラーコード。クライアントのエラーハンドリングに使用 |
| message | String | 人間可読なエラーメッセージ。ユーザー向け表示に使用可能 |
| details | Array\<ValidationError\> | 400/422 のバリデーションエラー時のみ付与。フィールド単位のエラー情報を配列で返却する。それ以外のステータスコードでは省略 |

> この形式は openapi.yaml の `components/schemas/Error` と一致する。エラーコード一覧の正本は本書 SS8.4 を参照。

### 8.3 レスポンスに含めてはいけない情報

| 禁止項目 | 理由 | 代替 |
|---------|------|------|
| スタックトレース | 内部実装の漏洩 | request_id でサーバーログと紐付け |
| SQL 文・DB エラー詳細 | スキーマ・クエリの漏洩 | 汎用メッセージ（"Internal server error"） |
| ファイルパス・ディレクトリ構造 | システム構成の漏洩 | なし |
| 内部エラーメッセージ | 実装詳細の漏洩 | 汎用メッセージ |
| ユーザーの存在情報 | 列挙攻撃 | 「メールアドレスまたはパスワードが正しくありません」で統一（SEC-011） |

### 8.4 HTTP ステータスコード別のレスポンス

> **注記**: `POST /api/auth/login` は SEC-011（ユーザー存在の秘匿）に基づき、認証関連の失敗（メール形式不正・ユーザー不存在・パスワード不一致・TenantMembership 不在）を `401 INVALID_CREDENTIALS` に統一する。これらに対して `400 VALIDATION_ERROR` は返さない。リクエストボディ不正（JSON パースエラー・必須項目欠落）は通常どおり `400 BAD_REQUEST` を返す。フロントエンド側でメール形式・パスワード長のバリデーションを実施し、ユーザーに適切なフィードバックを提供する。

| ステータス | コード | メッセージ | 使用場面 |
|---------|--------|---------|---------|
| 400 | `BAD_REQUEST` | "Invalid request body" | リクエストボディのパースエラー |
| 400 | `VALIDATION_ERROR` | フィールド別のバリデーションエラーメッセージ | 入力バリデーション失敗 |
| 401 | `UNAUTHORIZED` | "Authentication required" | 認証なし（トークン未送信） |
| 401 | `INVALID_CREDENTIALS` | "Invalid email or password" | ログイン認証失敗（メール/パスワード不一致、ユーザー不存在を含む統一コード。SEC-011） |
| 401 | `INVALID_TOKEN` | "Invalid token" | JWT 検証失敗（形式不正・署名不正・alg/kid/iss/token_type 不一致） |
| 401 | `TOKEN_EXPIRED` | "Token has expired" | トークン有効期限切れ（アクセストークン・リフレッシュトークン共通） |
| 403 | `FORBIDDEN` | "Insufficient permissions" | ロール不足・所有権不足 |
| 403 | `SELF_APPROVAL_NOT_ALLOWED` | "Self-approval is not allowed" | 自己承認 |
| 403 | `SELF_PAYMENT_NOT_ALLOWED` | "Self-payment processing is not allowed" | 自己支払処理 |
| 404 | `RESOURCE_NOT_FOUND` | "Resource not found" | リソース未発見・テナント境界越え |
| 409 | `CONFLICT` | "Resource has been modified by another user" | 楽観的ロック競合 |
| 413 | `FILE_TOO_LARGE` | "File size exceeds the 5MB limit" | ファイルサイズ超過 |
| 422 | `INVALID_STATE_TRANSITION` | "This state transition is not allowed" | 不正な状態遷移 |
| 422 | `REPORT_NOT_EDITABLE` | "Report can only be edited in draft status" | draft 以外での編集 |
| 422 | `REPORT_NOT_DELETABLE` | "Report can only be deleted in draft status" | draft 以外での削除 |
| 422 | `EMPTY_REPORT_SUBMISSION` | "Report must have at least one item" | 明細なし提出 |
| 422 | `NO_APPROVER_IN_TENANT` | "No approver found in the tenant" | Approver 不在での提出 |
| 422 | `INVALID_PERIOD` | "Start date must be before or equal to end date" | 期間不正 |
| 422 | `INVALID_AMOUNT` | "Amount must be a positive integer" | 金額不正 |
| 422 | `INVALID_FILE_TYPE` | "Allowed file types: JPEG, PNG, PDF" | ファイル形式不正 |
| 422 | `MISSING_REJECTION_REASON` | "Rejection reason is required" | 却下理由なし |
| 429 | `RATE_LIMIT_EXCEEDED` | "Too many requests. Please try again later." | レート制限超過 |
| 500 | `INTERNAL_ERROR` | "Internal server error" | サーバー内部エラー（詳細は隠蔽） |

**バリデーションエラーの詳細形式**:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Input validation failed",
    "details": [
      {"field": "title", "message": "Title is required"},
      {"field": "amount", "message": "Amount must be a positive integer"}
    ]
  }
}
```

`details` フィールドはバリデーションエラー（400/422）でのみ使用し、フィールド単位のエラー情報を配列で返却する。

### 8.5 500 エラーのハンドリング

```
[ハンドラ / サービス / リポジトリで予期しないエラー発生]
    |
    v
[1] エラー詳細をサーバーログに出力（request_id を含む）
    level: ERROR
    fields: request_id, error_type, error_message, stack_trace
    |
    v
[2] クライアントには汎用エラーを返却
    {
      "error": {
        "code": "INTERNAL_ERROR",
        "message": "Internal server error"
      }
    }
    |
    v
[3] レスポンスヘッダーに X-Request-ID を含める
    -> クライアントがサポート問い合わせ時に使用可能
```

---

## 9. 認可チェック

認可の詳細設計は `authz.md` に定義する。本セクションでは security.md との関連を示す参照のみを記載する。

- **層別責務**: `authz.md` SS3（ミドルウェア層 / サービス層 / リポジトリ層 / DB 層の責務分担）
- **エンドポイント別認可ルール**: `authz.md` SS6（全エンドポイントの許可ロール、所有者条件、状態条件、ビジネスルール、エラーコード）
- **所有権チェック**: `authz.md` SS7（Authorizer パターン）
- **ビジネスルールに基づく認可**: `authz.md` SS8（自己承認禁止、自己支払処理禁止）
- **レポート閲覧権限**: `authz.md` SS10（ロール別閲覧範囲、Visibility Scope）

本書 SS2（JWT 検証）で認証が成功した後、上記の認可チェックが実行される。認可チェックの実行順序は `authz.md` SS4 を参照。

---

## 10. 依存関係のセキュリティ

### 10.1 脆弱性スキャン

| ツール | 対象 | 実行タイミング |
|--------|------|-------------|
| `govulncheck` | Go 依存パッケージ | PR 作成時 + 週次定期実行 |
| `npm audit` | Node.js 依存パッケージ | PR 作成時 + 週次定期実行 |

### 10.2 対応方針

| 深刻度 | 対応期限 |
|--------|---------|
| Critical | 即座に対応（24時間以内） |
| High | 即座に対応（48時間以内） |
| Medium | 次スプリントで対応 |
| Low | バックログに記録、優先度に応じて対応 |

---

## 11. HTTPS / TLS

| 項目 | 仕様 |
|------|------|
| TLS 終端 | ALB で終端。ECS コンテナとの通信は VPC 内部で HTTP |
| TLS バージョン | TLS 1.2 以上（ALB のセキュリティポリシーで制御） |
| 証明書管理 | AWS Certificate Manager (ACM) |
| HTTP リダイレクト | ALB リスナールールで HTTP (80) -> HTTPS (443) リダイレクト |
| HSTS | `Strict-Transport-Security: max-age=31536000; includeSubDomains` でブラウザに HTTPS を強制 |

---

## 12. API エンドポイントセキュリティチェックリスト

新規エンドポイント追加時に以下をすべて確認する（security-policy.md SS10 に基づく）。

- [ ] 認証ミドルウェアが適用されているか（認証不要エンドポイントを除く）
- [ ] RBAC ミドルウェアで必要なロールが検証されているか
- [ ] 所有権チェックが実装されているか（該当する場合）
- [ ] リポジトリ層で tenant_id フィルタが適用されているか
- [ ] 入力バリデーションが go-playground/validator で実装されているか
- [ ] エラーレスポンスが内部情報を含んでいないか
- [ ] レート制限の対象になっているか
- [ ] 構造化ログで request_id, tenant_id, user_id が出力されているか

---

## 13. 上流成果物との差分

| 項目 | 上流の記載 | 本書での対応 |
|------|-----------|------------|
| `architecture.md` SS6.2 レート制限 | 認証済み 100 req/min, 未認証 20 req/min の2種 | ログイン 5 req/min/IP、ファイルアップロード 10 req/min/user を追加（security-policy.md SS4 に準拠） |

---

## 14. 品質チェック

- [x] JWT 実装詳細（RS256 鍵管理、クレーム構造、ローテーション）が定義されているか
- [x] レート制限の具体値（4種類）と実装方式が定義されているか
- [x] CORS ポリシーが環境別に定義されているか
- [x] セキュリティヘッダー一覧と値が定義されているか
- [x] 入力バリデーション戦略（バックエンドを信頼境界）が定義されているか
- [x] パスワードハッシュパラメータ（Argon2id）が定義されているか
- [x] エラーレスポンスのサニタイズ方針が定義されているか
- [x] テナント境界越えは 404 を返却する方針が明記されているか
- [x] 上流成果物との差分が記録されているか
- [x] 用語が glossary.md と一致しているか
- [x] MVP スコープ内に収まっているか
