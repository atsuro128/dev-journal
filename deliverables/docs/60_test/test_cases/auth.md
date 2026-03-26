# 認証テストケース

## この文書の役割

| 項目 | 内容 |
|------|------|
| 目的 | 認証・認可エンドポイント（/api/auth/*）のテストケースを定義する |
| 正本情報 | AUTH-F01〜F06 に対応するテストケース一覧 |
| 扱わない内容 | 認証以外のエンドポイントのテスト、テスト実装コード |
| 主な参照元 | `50_detail_design/openapi.yaml#/api/auth/*`, `50_detail_design/security.md`, `10_requirements/requirements.md#AUTH-F*` |
| 主な参照先 | `60_test/traceability.md`, テスト実装（Step 9） |

---

## 対応ハンドラ

- `auth_handler_test.go`（統合テスト）
- `domain/auth_test.go`（単体テスト: Argon2id、JWT 生成・検証）

## 参照設計書

- `50_detail_design/openapi.yaml` §/api/auth/*
- `50_detail_design/security.md` §2.1（JWT）、§2.2（Argon2id）、§2.3（パスワードリセット）
- `50_detail_design/authz.md` §6.1（認証関連の認可ルール）

## フィクスチャ参照

`test_strategy.md` §4 で定義した標準フィクスチャを使用する。

| 参照名 | 値 |
|--------|-----|
| テナントA ID | `aaaaaaaa-0001-0001-0001-000000000001` |
| Admin ユーザー ID | `aaaaaaaa-1111-1111-1111-000000000001` |
| Admin メール | `test-admin@example.com` |
| テスト用パスワード | `TestPass1!` |
| テナントB Member ID | `bbbbbbbb-3333-3333-3333-000000000003` |

---

## テストケース

### 1. ドメイン層単体テスト

対応ファイル: `domain/auth_test.go`

#### 1.1 Argon2id ハッシュ・検証

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|----------------|-------------------|---------|
| AUTH-001 | 単体 | domain | `TestHashPassword_Success` | パスワード: `TestPass1!` | エラーなし。ハッシュ文字列が `$argon2id$` で始まること |
| AUTH-002 | 単体 | domain | `TestHashPassword_EmptyPassword` | パスワード: `""` (空文字) | エラーが返る |
| AUTH-003 | 単体 | domain | `TestVerifyPassword_Correct` | 正しいパスワード + 対応するハッシュ | `true`, エラーなし |
| AUTH-004 | 単体 | domain | `TestVerifyPassword_Wrong` | 誤ったパスワード + ハッシュ | `false`, エラーなし |
| AUTH-005 | 単体 | domain | `TestVerifyPassword_InvalidHash` | パスワード + 不正形式のハッシュ文字列 (`"not-a-hash"`) | エラーが返る |
| AUTH-006 | 単体 | domain | `TestHashPassword_UniquePerCall` | 同一パスワードを 2 回ハッシュ | 生成されたハッシュ文字列が互いに異なること（ソルトが毎回ランダムに生成される） |
| AUTH-007 | 単体 | domain | `TestHashPassword_Argon2idParams` | パスワード: `TestPass1!` | 出力ハッシュのパラメータが `m=65536,t=3,p=4` であること |

#### 1.2 JWT 生成・検証

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|----------------|-------------------|---------|
| AUTH-008 | 単体 | domain | `TestGenerateAccessToken_Success` | ユーザーID: `aaaaaaaa-1111-1111-1111-000000000001`, テナントID: `aaaaaaaa-0001-0001-0001-000000000001`, ロール: `admin` | エラーなし。生成トークンが 3 パート構成の文字列 |
| AUTH-009 | 単体 | domain | `TestGenerateAccessToken_Claims` | 同上 | デコードした claims に `iss=expense-saas`, `sub=<user_id>`, `tenant_id=<tenant_id>`, `role=admin`, `token_type=access`, `jti`（非空 UUID）が含まれること |
| AUTH-010 | 単体 | domain | `TestGenerateAccessToken_Expiry` | 同上 | `exp` が現在時刻 + 15 分であること（±5 秒の誤差を許容） |
| AUTH-011 | 単体 | domain | `TestGenerateRefreshToken_Success` | ユーザーID: `aaaaaaaa-1111-1111-1111-000000000001` | エラーなし。生成トークンが 3 パート構成の文字列 |
| AUTH-012 | 単体 | domain | `TestGenerateRefreshToken_Claims` | 同上 | claims に `token_type=refresh` が含まれ、`tenant_id` と `role` が含まれないこと |
| AUTH-013 | 単体 | domain | `TestGenerateRefreshToken_Expiry` | 同上 | `exp` が現在時刻 + 7 日であること（±5 秒の誤差を許容） |
| AUTH-014 | 単体 | domain | `TestVerifyAccessToken_Valid` | 有効なアクセストークン | エラーなし。claims が正しく返ること |
| AUTH-015 | 単体 | domain | `TestVerifyAccessToken_Expired` | 有効期限切れのアクセストークン（`exp` を過去に設定して生成） | `TOKEN_EXPIRED` エラー |
| AUTH-016 | 単体 | domain | `TestVerifyAccessToken_InvalidSignature` | 別の秘密鍵で署名されたトークン | `INVALID_TOKEN` エラー |
| AUTH-017 | 単体 | domain | `TestVerifyAccessToken_WrongAlgorithm` | HS256 で署名されたトークン（alg 混乱攻撃） | `INVALID_TOKEN` エラー（RS256 以外を拒否） |
| AUTH-018 | 単体 | domain | `TestVerifyAccessToken_WrongIssuer` | `iss=malicious-service` のトークン | `INVALID_TOKEN` エラー |
| AUTH-019 | 単体 | domain | `TestVerifyAccessToken_WrongTokenType` | `token_type=refresh` のリフレッシュトークンをアクセストークン検証に使用 | `INVALID_TOKEN` エラー |
| AUTH-020 | 単体 | domain | `TestVerifyAccessToken_MalformedString` | `"not.a.jwt"` | `INVALID_TOKEN` エラー |
| AUTH-021 | 単体 | domain | `TestVerifyRefreshToken_Valid` | 有効なリフレッシュトークン | エラーなし。`sub`（user_id）と `jti` が正しく返ること |
| AUTH-022 | 単体 | domain | `TestVerifyRefreshToken_Expired` | 有効期限切れのリフレッシュトークン | `TOKEN_EXPIRED` エラー |
| AUTH-023 | 単体 | domain | `TestVerifyRefreshToken_WrongTokenType` | `token_type=access` のアクセストークンをリフレッシュトークン検証に使用 | `INVALID_TOKEN` エラー |

---

### 2. エンドポイント別統合テスト

対応ファイル: `auth_handler_test.go`

全統合テストの共通前提条件:
- テスト実行前に DB を TRUNCATE し、フィクスチャを再投入する（`test_strategy.md` §9.3）
- テスト用 PostgreSQL（`localhost:5433/expense_test`）を使用する

#### POST /api/auth/signup

仕様概要: テナントと Admin ユーザーを同時に新規作成する。認証不要。成功時 201 と AuthTokens を返す。

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|----------------|-------------------|---------|
| AUTH-024 | 統合 | handler | `TestSignup_Success` | `company_name: "New Corp"`, `user_name: "Test Admin"`, `email: "new@example.com"`, `password: "TestPass1!"` | 201。レスポンス `data` に `access_token`, `refresh_token` が存在すること。DB に `tenants`, `users`, `tenant_memberships`（ロール: `admin`）レコードが作成されること |
| AUTH-025 | 統合 | handler | `TestSignup_DuplicateEmail` | 既にフィクスチャで存在する `test-admin@example.com` で登録 | 409。`error.code = EMAIL_ALREADY_EXISTS` |
| AUTH-026 | 統合 | handler | `TestSignup_ValidationError_MissingCompanyName` | `company_name` を省略（他フィールドは有効） | 422。`error.code = VALIDATION_ERROR`。`details` に `company_name` フィールドのエラーが含まれること |
| AUTH-027 | 統合 | handler | `TestSignup_ValidationError_MissingEmail` | `email` を省略 | 422。`error.code = VALIDATION_ERROR`。`details` に `email` フィールドのエラーが含まれること |
| AUTH-028 | 統合 | handler | `TestSignup_ValidationError_InvalidEmail` | `email: "not-an-email"` | 422。`error.code = VALIDATION_ERROR` |
| AUTH-029 | 統合 | handler | `TestSignup_ValidationError_PasswordTooShort` | `password: "short"` (7文字) | 422。`error.code = VALIDATION_ERROR` |
| AUTH-030 | 統合 | handler | `TestSignup_ValidationError_PasswordTooLong` | `password` を 129 文字の文字列に設定 | 422。`error.code = VALIDATION_ERROR` |
| AUTH-031 | 統合 | handler | `TestSignup_ValidationError_CompanyNameTooLong` | `company_name` を 201 文字の文字列に設定 | 422。`error.code = VALIDATION_ERROR` |
| AUTH-032 | 統合 | handler | `TestSignup_InvalidJsonBody` | リクエストボディ: `"not json"` | 400。`error.code = BAD_REQUEST` |
| AUTH-033 | 統合 | handler | `TestSignup_NoAuthRequired` | 有効なリクエスト（Authorization ヘッダーなし） | 201（認証ヘッダーなしでアクセスできること） |

#### POST /api/auth/login

仕様概要: メールとパスワードで認証し JWT を発行する。認証不要。SEC-011 に基づき認証関連の全失敗を 401 に統一する。

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|----------------|-------------------|---------|
| AUTH-034 | 統合 | handler | `TestLogin_Success` | `email: "test-admin@example.com"`, `password: "TestPass1!"` | 200。`data.access_token` と `data.refresh_token` が存在すること。アクセストークンの claims に `role=admin`, `tenant_id=aaaaaaaa-0001-0001-0001-000000000001` が含まれること |
| AUTH-035 | 統合 | handler | `TestLogin_WrongPassword` | `email: "test-admin@example.com"`, `password: "WrongPass!"` | 401。`error.code = INVALID_CREDENTIALS` |
| AUTH-036 | 統合 | handler | `TestLogin_NotExistEmail` | `email: "nobody@example.com"`, `password: "TestPass1!"` | 401。`error.code = INVALID_CREDENTIALS`（ユーザーの存在を漏らさない: SEC-011） |
| AUTH-037 | 統合 | handler | `TestLogin_InvalidEmailFormat` | `email: "not-an-email"`, `password: "TestPass1!"` | 401。`error.code = INVALID_CREDENTIALS`（メール形式不正も認証失敗として統一: SEC-011） |
| AUTH-038 | 統合 | handler | `TestLogin_MissingEmail` | `email` フィールドを省略 | 400。`error.code = BAD_REQUEST` |
| AUTH-039 | 統合 | handler | `TestLogin_MissingPassword` | `password` フィールドを省略 | 400。`error.code = BAD_REQUEST` |
| AUTH-040 | 統合 | handler | `TestLogin_InvalidJsonBody` | リクエストボディ: `"not json"` | 400。`error.code = BAD_REQUEST` |
| AUTH-041 | 統合 | handler | `TestLogin_NoAuthRequired` | 有効なリクエスト（Authorization ヘッダーなし） | 200（認証ヘッダーなしでアクセスできること） |

#### POST /api/auth/refresh

仕様概要: リフレッシュトークンを使用して新しいアクセストークンとリフレッシュトークンを発行する（トークンローテーション）。認証不要（リフレッシュトークン認証）。

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|----------------|-------------------|---------|
| AUTH-042 | 統合 | handler | `TestRefreshToken_Success` | 前提: ログインして取得した有効なリフレッシュトークン。入力: `refresh_token: <valid_token>` | 200。新しい `access_token` と `refresh_token` が返ること。レスポンスの `data` に最新の `user`, `tenant` 情報が含まれること |
| AUTH-043 | 統合 | handler | `TestRefreshToken_Rotation` | 前提: 有効なリフレッシュトークン A。リフレッシュ後に旧トークン A を再度使用 | 1 回目: 200。2 回目: 401（旧リフレッシュトークンが無効化されていること） |
| AUTH-044 | 統合 | handler | `TestRefreshToken_RevokedToken` | 前提: ログアウト済みユーザーのリフレッシュトークン（`is_revoked=true`）。入力: そのリフレッシュトークン | 401。`error.code = UNAUTHORIZED` または `INVALID_TOKEN` |
| AUTH-045 | 統合 | handler | `TestRefreshToken_ExpiredToken` | 有効期限切れのリフレッシュトークン（テスト用に `exp` を過去日時で発行） | 401。`error.code = TOKEN_EXPIRED` |
| AUTH-046 | 統合 | handler | `TestRefreshToken_AccessTokenAsRefresh` | アクセストークン（`token_type=access`）を `refresh_token` として送信 | 401。`error.code = INVALID_TOKEN` |
| AUTH-047 | 統合 | handler | `TestRefreshToken_InvalidFormat` | `refresh_token: "not.a.valid.jwt"` | 401。`error.code = INVALID_TOKEN` |
| AUTH-048 | 統合 | handler | `TestRefreshToken_MissingBody` | `refresh_token` フィールドを省略 | 400。`error.code = BAD_REQUEST` |
| AUTH-049 | 統合 | handler | `TestRefreshToken_ReflectsRoleChange` | 前提: ユーザーのロールを DB で `admin` から `member` に変更後、旧アクセストークンのリフレッシュを実行 | 200。返却された新しいアクセストークンの claims に `role=member` が反映されること（DB の最新 `tenant_memberships` を再取得） |
| AUTH-050 | 統合 | handler | `TestRefreshToken_NoAuthRequired` | 有効なリフレッシュトークン（Authorization ヘッダーなし） | 200（認証ヘッダーなしでアクセスできること） |

#### POST /api/auth/logout

仕様概要: リフレッシュトークンを無効化する。アクセストークンが期限切れでもログアウト可能。認証不要（リフレッシュトークン認証）。

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|----------------|-------------------|---------|
| AUTH-051 | 統合 | handler | `TestLogout_Success` | 前提: ログインして取得した有効なリフレッシュトークン。入力: `refresh_token: <valid_token>` | 200。`data.message` が存在すること。DB の `refresh_tokens` テーブルで当該 `jti` の `is_revoked=true` であること |
| AUTH-052 | 統合 | handler | `TestLogout_WithExpiredAccessToken` | 前提: 有効なリフレッシュトークン（アクセストークンは期限切れ）。Authorization ヘッダーに期限切れアクセストークンを設定、ボディに有効なリフレッシュトークンを設定 | 200（アクセストークンの期限に依存しないこと） |
| AUTH-053 | 統合 | handler | `TestLogout_AlreadyRevokedToken` | 前提: 既に `is_revoked=true` のリフレッシュトークン | 401。`error.code = UNAUTHORIZED` または `INVALID_TOKEN` |
| AUTH-054 | 統合 | handler | `TestLogout_InvalidRefreshToken` | `refresh_token: "invalid.token.value"` | 401。`error.code = INVALID_TOKEN` |
| AUTH-055 | 統合 | handler | `TestLogout_MissingRefreshToken` | `refresh_token` フィールドを省略 | 400。`error.code = BAD_REQUEST` |
| AUTH-056 | 統合 | handler | `TestLogout_NoAuthRequired` | 有効なリフレッシュトークン（Authorization ヘッダーなし） | 200（認証ヘッダーなしでアクセスできること） |

#### GET /api/auth/me

仕様概要: 現在のログインユーザーの情報を取得する。認証必須（全ロール許可）。

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|----------------|-------------------|---------|
| AUTH-057 | 統合 | handler | `TestGetMe_Success_Admin` | 前提: Admin ユーザー（`test-admin@example.com`）のアクセストークン | 200。`data.user_id = aaaaaaaa-1111-1111-1111-000000000001`, `data.email = test-admin@example.com`, `data.role = admin`, `data.tenant_id = aaaaaaaa-0001-0001-0001-000000000001` |
| AUTH-058 | 統合 | handler | `TestGetMe_Success_Member` | 前提: Member ユーザー（`test-member@example.com`）のアクセストークン | 200。`data.role = member` |
| AUTH-059 | 統合 | handler | `TestGetMe_Success_Approver` | 前提: Approver ユーザー（`test-approver@example.com`）のアクセストークン | 200。`data.role = approver` |
| AUTH-060 | 統合 | handler | `TestGetMe_Success_Accounting` | 前提: Accounting ユーザー（`test-accounting@example.com`）のアクセストークン | 200。`data.role = accounting` |
| AUTH-061 | 統合 | handler | `TestGetMe_Unauthorized_NoToken` | Authorization ヘッダーなし | 401。`error.code = UNAUTHORIZED` |
| AUTH-062 | 統合 | handler | `TestGetMe_Unauthorized_ExpiredToken` | 有効期限切れのアクセストークン | 401。`error.code = TOKEN_EXPIRED` |
| AUTH-063 | 統合 | handler | `TestGetMe_Unauthorized_InvalidToken` | `Authorization: Bearer not.a.real.token` | 401。`error.code = INVALID_TOKEN` |
| AUTH-064 | 統合 | handler | `TestGetMe_Unauthorized_RefreshTokenAsAccess` | リフレッシュトークン（`token_type=refresh`）をアクセストークンとして Authorization ヘッダーに設定 | 401。`error.code = INVALID_TOKEN` |

#### POST /api/auth/password-reset

仕様概要: パスワードリセット用メールを送信する。ユーザーの存在に関わらず同一の成功レスポンスを返す（SEC-011）。認証不要。

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|----------------|-------------------|---------|
| AUTH-065 | 統合 | handler | `TestRequestPasswordReset_ExistingEmail` | `email: "test-admin@example.com"` （フィクスチャに存在するメール） | 200。`data.message` が存在すること。DB の `password_reset_tokens` にレコードが作成されること |
| AUTH-066 | 統合 | handler | `TestRequestPasswordReset_NonExistentEmail` | `email: "nobody@example.com"` （存在しないメール） | 200。レスポンス内容が AUTH-065 と同一であること（ユーザーの存在を漏らさない: SEC-011） |
| AUTH-067 | 統合 | handler | `TestRequestPasswordReset_ValidationError_MissingEmail` | `email` フィールドを省略 | 422。`error.code = VALIDATION_ERROR` |
| AUTH-068 | 統合 | handler | `TestRequestPasswordReset_ValidationError_InvalidEmail` | `email: "not-valid"` | 422。`error.code = VALIDATION_ERROR` |
| AUTH-069 | 統合 | handler | `TestRequestPasswordReset_NoAuthRequired` | 有効なリクエスト（Authorization ヘッダーなし） | 200（認証ヘッダーなしでアクセスできること） |

#### PUT /api/auth/password-reset/{token}

仕様概要: リセットトークンを使用して新しいパスワードを設定する。トークンは 1 回使用で無効化される（SEC-006）。リセット実行時に全リフレッシュトークンを無効化する。認証不要。

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|----------------|-------------------|---------|
| AUTH-070 | 統合 | handler | `TestExecutePasswordReset_Success` | 前提: DB に有効なリセットトークンを直接挿入（`test-member@example.com` 宛、有効期限: 現在時刻 + 30 分）。入力: `token: <valid_token>`, `new_password: "NewPass1!"` | 200。`data.message` が存在すること。新しいパスワードでログイン（POST /api/auth/login）できること。使用済みトークンの `used_at` が記録されること。当該ユーザーの全リフレッシュトークンが無効化されること |
| AUTH-071 | 統合 | handler | `TestExecutePasswordReset_InvalidToken` | `token: "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"` （存在しないトークン） | 422。`error.code` が含まれること |
| AUTH-072 | 統合 | handler | `TestExecutePasswordReset_ExpiredToken` | 前提: DB に有効期限切れのリセットトークンを直接挿入（`expires_at` を過去日時に設定）。入力: そのトークン値 | 422。エラーレスポンスが返ること |
| AUTH-073 | 統合 | handler | `TestExecutePasswordReset_AlreadyUsedToken` | 前提: 既に `used_at` が設定されたリセットトークン（AUTH-070 で使用済みのもの）。入力: 同トークン | 422。エラーレスポンスが返ること（1 回限りの使用: SEC-006） |
| AUTH-074 | 統合 | handler | `TestExecutePasswordReset_ValidationError_PasswordTooShort` | 有効なトークン、`new_password: "short"` (7文字) | 422。`error.code = VALIDATION_ERROR` |
| AUTH-075 | 統合 | handler | `TestExecutePasswordReset_ValidationError_MissingPassword` | 有効なトークン、`new_password` フィールドを省略 | 422。`error.code = VALIDATION_ERROR` |
| AUTH-076 | 統合 | handler | `TestExecutePasswordReset_NoAuthRequired` | 有効なトークンと有効なパスワード（Authorization ヘッダーなし） | 200（認証ヘッダーなしでアクセスできること） |

---

### 3. 共通セキュリティ・ミドルウェアの統合テスト

対応ファイル: `auth_handler_test.go`

認証ミドルウェアおよびセキュリティ設計（`security.md` §2.1）のエンドツーエンドな動作を検証する。

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|----------------|-------------------|---------|
| AUTH-077 | 統合 | handler | `TestAuth_AccessTokenFlow` | サインアップ → ログイン → GET /api/auth/me の一連のフロー | 各ステップで正しいレスポンスが返ること |
| AUTH-078 | 統合 | handler | `TestAuth_InvalidAlgorithm` | none アルゴリズム（`"alg": "none"`）のトークンを Authorization ヘッダーに設定して GET /api/auth/me | 401。`error.code = INVALID_TOKEN`（alg 混乱攻撃の防止） |
| AUTH-079 | 統合 | handler | `TestAuth_InvalidIssuer` | `iss: "other-service"` で生成されたトークンを Authorization ヘッダーに設定して GET /api/auth/me | 401。`error.code = INVALID_TOKEN` |
| AUTH-080 | 統合 | handler | `TestAuth_AuthEndpointsPubliclyAccessible` | signup, login, refresh, logout, password-reset の各エンドポイントに Authorization ヘッダーなしでリクエスト | 各エンドポイントが 401 を返さないこと（認証不要エンドポイントであること） |
