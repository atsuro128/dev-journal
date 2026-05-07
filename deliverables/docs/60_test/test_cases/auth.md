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

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| AUTH-001 | 単体 | domain | 正常系 | SEC-002 | security.md#2.2 | `TestHashPassword_Success` | パスワード: `TestPass1!` | エラーなし。ハッシュ文字列が `$argon2id$` で始まること |
| AUTH-002 | 単体 | domain | 異常系 | SEC-002 | security.md#2.2 | `TestHashPassword_EmptyPassword` | パスワード: `""` (空文字) | エラーが返る |
| AUTH-003 | 単体 | domain | 正常系 | SEC-002 | security.md#2.2 | `TestVerifyPassword_Correct` | 正しいパスワード + 対応するハッシュ | `true`, エラーなし |
| AUTH-004 | 単体 | domain | 異常系 | SEC-002 | security.md#2.2 | `TestVerifyPassword_Wrong` | 誤ったパスワード + ハッシュ | `false`, エラーなし |
| AUTH-005 | 単体 | domain | 異常系 | SEC-002 | security.md#2.2 | `TestVerifyPassword_InvalidHash` | パスワード + 不正形式のハッシュ文字列 (`"not-a-hash"`) | エラーが返る |
| AUTH-006 | 単体 | domain | セキュリティ | SEC-002 | security.md#2.2 | `TestHashPassword_UniquePerCall` | 同一パスワードを 2 回ハッシュ | 生成されたハッシュ文字列が互いに異なること（ソルトが毎回ランダムに生成される） |
| AUTH-007 | 単体 | domain | 正常系 | SEC-002 | security.md#2.2 | `TestHashPassword_Argon2idParams` | パスワード: `TestPass1!` | 出力ハッシュのパラメータが `m=65536,t=3,p=4` であること |

#### 1.2 JWT 生成・検証

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| AUTH-008 | 単体 | domain | 正常系 | SEC-003, SEC-004 | security.md#2.1 | `TestGenerateAccessToken_Success` | ユーザーID: `aaaaaaaa-1111-1111-1111-000000000001`, テナントID: `aaaaaaaa-0001-0001-0001-000000000001`, ロール: `admin` | エラーなし。生成トークンが 3 パート構成の文字列 |
| AUTH-009 | 単体 | domain | 正常系 | SEC-003, SEC-004 | security.md#2.1 | `TestGenerateAccessToken_Claims` | 同上 | デコードした claims に `iss=expense-saas`, `sub=<user_id>`, `tenant_id=<tenant_id>`, `role=admin`, `token_type=access`, `jti`（非空 UUID）が含まれること |
| AUTH-010 | 単体 | domain | 正常系 | SEC-003 | security.md#2.1 | `TestGenerateAccessToken_Expiry` | 同上 | `exp` が現在時刻 + 15 分であること（±5 秒の誤差を許容） |
| AUTH-011 | 単体 | domain | 正常系 | SEC-003 | security.md#2.1 | `TestGenerateRefreshToken_Success` | ユーザーID: `aaaaaaaa-1111-1111-1111-000000000001` | エラーなし。生成トークンが 3 パート構成の文字列 |
| AUTH-012 | 単体 | domain | 正常系 | SEC-003 | security.md#2.1 | `TestGenerateRefreshToken_Claims` | 同上 | claims に `token_type=refresh` が含まれ、`tenant_id` と `role` が含まれないこと |
| AUTH-013 | 単体 | domain | 正常系 | SEC-003 | security.md#2.1 | `TestGenerateRefreshToken_Expiry` | 同上 | `exp` が現在時刻 + 7 日であること（±5 秒の誤差を許容） |
| AUTH-014 | 単体 | domain | 正常系 | SEC-003, SEC-004 | security.md#2.1 | `TestVerifyAccessToken_Valid` | 有効なアクセストークン | エラーなし。claims が正しく返ること |
| AUTH-015 | 単体 | domain | 異常系 | SEC-003 | security.md#2.1 | `TestVerifyAccessToken_Expired` | 有効期限切れのアクセストークン（`exp` を過去に設定して生成） | `TOKEN_EXPIRED` エラー |
| AUTH-016 | 単体 | domain | セキュリティ | SEC-004 | security.md#2.1 | `TestVerifyAccessToken_InvalidSignature` | 別の秘密鍵で署名されたトークン | `INVALID_TOKEN` エラー |
| AUTH-017 | 単体 | domain | セキュリティ | SEC-004 | security.md#2.1 | `TestVerifyAccessToken_WrongAlgorithm` | HS256 で署名されたトークン（alg 混乱攻撃） | `INVALID_TOKEN` エラー（RS256 以外を拒否） |
| AUTH-018 | 単体 | domain | セキュリティ | SEC-004 | security.md#2.1 | `TestVerifyAccessToken_WrongIssuer` | `iss=malicious-service` のトークン | `INVALID_TOKEN` エラー |
| AUTH-019 | 単体 | domain | 異常系 | SEC-003 | security.md#2.1 | `TestVerifyAccessToken_WrongTokenType` | `token_type=refresh` のリフレッシュトークンをアクセストークン検証に使用 | `INVALID_TOKEN` エラー |
| AUTH-020 | 単体 | domain | 異常系 | SEC-004 | security.md#2.1 | `TestVerifyAccessToken_MalformedString` | `"not.a.jwt"` | `INVALID_TOKEN` エラー |
| AUTH-021 | 単体 | domain | 正常系 | SEC-003 | security.md#2.1 | `TestVerifyRefreshToken_Valid` | 有効なリフレッシュトークン | エラーなし。`sub`（user_id）と `jti` が正しく返ること |
| AUTH-022 | 単体 | domain | 異常系 | SEC-003 | security.md#2.1 | `TestVerifyRefreshToken_Expired` | 有効期限切れのリフレッシュトークン | `TOKEN_EXPIRED` エラー |
| AUTH-023 | 単体 | domain | 異常系 | SEC-003 | security.md#2.1 | `TestVerifyRefreshToken_WrongTokenType` | `token_type=access` のアクセストークンをリフレッシュトークン検証に使用 | `INVALID_TOKEN` エラー |
| AUTH-081 | 単体 | domain | セキュリティ | SEC-003 | security.md#2.1（クロックスキュー許容） | `TestVerifyAccessToken_IatFuture30s_Allowed` | iat = now + 30 秒のアクセストークン（leeway 60 秒以内） | エラーなし。claims が正しく返ること |
| AUTH-082 | 単体 | domain | セキュリティ | SEC-003 | security.md#2.1（クロックスキュー許容） | `TestVerifyAccessToken_IatFuture61s_Rejected` | iat = now + 61 秒のアクセストークン（leeway 超過） | `INVALID_TOKEN` エラー |
| AUTH-083 | 単体 | domain | セキュリティ | SEC-003 | security.md#2.1（クロックスキュー許容） | `TestVerifyAccessToken_ExpPast30s_Allowed` | exp = now - 30 秒のアクセストークン（leeway 60 秒以内） | エラーなし。claims が正しく返ること |
| AUTH-084 | 単体 | domain | セキュリティ | SEC-003 | security.md#2.1（クロックスキュー許容） | `TestVerifyAccessToken_ExpPast61s_Rejected` | exp = now - 61 秒のアクセストークン（leeway 超過） | `TOKEN_EXPIRED` エラー |
| AUTH-085 | 単体 | domain | セキュリティ | SEC-003 | security.md#2.1（クロックスキュー許容） | `TestVerifyRefreshToken_IatFuture30s_Allowed` | iat = now + 30 秒のリフレッシュトークン（leeway 60 秒以内） | エラーなし。claims が正しく返ること |
| AUTH-086 | 単体 | domain | セキュリティ | SEC-003 | security.md#2.1（クロックスキュー許容） | `TestVerifyRefreshToken_ExpPast30s_Allowed` | exp = now - 30 秒のリフレッシュトークン（leeway 60 秒以内） | エラーなし。claims が正しく返ること |
| AUTH-087 | 単体 | domain | セキュリティ | SEC-003 | security.md#2.1（クロックスキュー許容） | `TestVerifyRefreshToken_IatFuture61s_Rejected` | iat = now + 61 秒のリフレッシュトークン（leeway 超過） | `INVALID_TOKEN` エラー |
| AUTH-088 | 単体 | domain | セキュリティ | SEC-003 | security.md#2.1（クロックスキュー許容） | `TestVerifyRefreshToken_ExpPast61s_Rejected` | exp = now - 61 秒のリフレッシュトークン（leeway 超過） | `TOKEN_EXPIRED` エラー |

---

### 2. エンドポイント別統合テスト

対応ファイル: `auth_handler_test.go`

全統合テストの共通前提条件:
- テスト実行前に DB を TRUNCATE し、フィクスチャを再投入する（`test_strategy.md` §9.3）
- テスト用 PostgreSQL（`localhost:5433/expense_test`）を使用する

#### POST /api/auth/signup

仕様概要: テナントと Admin ユーザーを同時に新規作成する。認証不要。成功時 201 と AuthTokens を返す。

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| AUTH-024 | 統合 | handler | 正常系 | AUTH-F01 | openapi.yaml#signup, db_schema.md#tenants, db_schema.md#users, db_schema.md#tenant_memberships | `TestSignup_Success` | `company_name: "New Corp"`, `user_name: "Test Admin"`, `email: "new@example.com"`, `password: "TestPass1!"` | 201。レスポンス `data` に `access_token`, `refresh_token` が存在すること。DB に `tenants`, `users`, `tenant_memberships`（ロール: `admin`）レコードが作成されること |
| AUTH-025 | 統合 | handler | 異常系 | AUTH-F01 | openapi.yaml#signup | `TestSignup_DuplicateEmail` | 既にフィクスチャで存在する `test-admin@example.com` で登録 | 409。`error.code = EMAIL_ALREADY_EXISTS` |
| AUTH-026 | 統合 | handler | 異常系 | AUTH-F01 | openapi.yaml#signup | `TestSignup_ValidationError_MissingCompanyName` | `company_name` を省略（他フィールドは有効） | 422。`error.code = VALIDATION_ERROR`。`details` に `company_name` フィールドのエラーが含まれること |
| AUTH-027 | 統合 | handler | 異常系 | AUTH-F01 | openapi.yaml#signup | `TestSignup_ValidationError_MissingEmail` | `email` を省略 | 422。`error.code = VALIDATION_ERROR`。`details` に `email` フィールドのエラーが含まれること |
| AUTH-028 | 統合 | handler | 異常系 | AUTH-F01 | openapi.yaml#signup | `TestSignup_ValidationError_InvalidEmail` | `email: "not-an-email"` | 422。`error.code = VALIDATION_ERROR` |
| AUTH-029 | 統合 | handler | 境界値 | AUTH-F01, SEC-010 | openapi.yaml#signup | `TestSignup_ValidationError_PasswordTooShort` | `password: "short"` (7文字) | 422。`error.code = VALIDATION_ERROR` |
| AUTH-030 | 統合 | handler | 境界値 | AUTH-F01 | openapi.yaml#signup | `TestSignup_ValidationError_PasswordTooLong` | `password` を 129 文字の文字列に設定 | 422。`error.code = VALIDATION_ERROR` |
| AUTH-031 | 統合 | handler | 境界値 | AUTH-F01 | openapi.yaml#signup | `TestSignup_ValidationError_CompanyNameTooLong` | `company_name` を 201 文字の文字列に設定 | 422。`error.code = VALIDATION_ERROR` |
| AUTH-032 | 統合 | handler | 異常系 | AUTH-F01 | openapi.yaml#signup | `TestSignup_InvalidJsonBody` | リクエストボディ: `"not json"` | 400。`error.code = BAD_REQUEST` |
| AUTH-033 | 統合 | handler | 正常系 | AUTH-F01 | openapi.yaml#signup | `TestSignup_NoAuthRequired` | 有効なリクエスト（Authorization ヘッダーなし） | 201（認証ヘッダーなしでアクセスできること） |

#### POST /api/auth/login

仕様概要: メールとパスワードで認証し JWT を発行する。認証不要。SEC-011 に基づき認証関連の全失敗を 401 に統一する。

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| AUTH-034 | 統合 | handler | 正常系 | AUTH-F02, SEC-001 | openapi.yaml#login, security.md#2.1 | `TestLogin_Success` | `email: "test-admin@example.com"`, `password: "TestPass1!"` | 200。`data.access_token` と `data.refresh_token` が存在すること。アクセストークンの claims に `role=admin`, `tenant_id=aaaaaaaa-0001-0001-0001-000000000001` が含まれること |
| AUTH-035 | 統合 | handler | 異常系 | AUTH-F02 | openapi.yaml#login | `TestLogin_WrongPassword` | `email: "test-admin@example.com"`, `password: "WrongPass!"` | 401。`error.code = INVALID_CREDENTIALS` |
| AUTH-036 | 統合 | handler | セキュリティ | AUTH-F02, SEC-011 | openapi.yaml#login, security.md#2.4 | `TestLogin_NotExistEmail` | `email: "nobody@example.com"`, `password: "TestPass1!"` | 401。`error.code = INVALID_CREDENTIALS`（ユーザーの存在を漏らさない: SEC-011） |
| AUTH-037 | 統合 | handler | セキュリティ | AUTH-F02, SEC-011 | openapi.yaml#login, security.md#2.4 | `TestLogin_InvalidEmailFormat` | `email: "not-an-email"`, `password: "TestPass1!"` | 401。`error.code = INVALID_CREDENTIALS`（メール形式不正も認証失敗として統一: SEC-011） |
| AUTH-038 | 統合 | handler | 異常系 | AUTH-F02 | openapi.yaml#login | `TestLogin_MissingEmail` | `email` フィールドを省略 | 400。`error.code = BAD_REQUEST` |
| AUTH-039 | 統合 | handler | 異常系 | AUTH-F02 | openapi.yaml#login | `TestLogin_MissingPassword` | `password` フィールドを省略 | 400。`error.code = BAD_REQUEST` |
| AUTH-040 | 統合 | handler | 異常系 | AUTH-F02 | openapi.yaml#login | `TestLogin_InvalidJsonBody` | リクエストボディ: `"not json"` | 400。`error.code = BAD_REQUEST` |
| AUTH-041 | 統合 | handler | 正常系 | AUTH-F02 | openapi.yaml#login | `TestLogin_NoAuthRequired` | 有効なリクエスト（Authorization ヘッダーなし） | 200（認証ヘッダーなしでアクセスできること） |

#### POST /api/auth/refresh

仕様概要: リフレッシュトークンを使用して新しいアクセストークンとリフレッシュトークンを発行する（トークンローテーション）。認証不要（リフレッシュトークン認証）。

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| AUTH-042 | 統合 | handler | 正常系 | AUTH-F03 | openapi.yaml#refreshToken, db_schema.md#refresh_tokens | `TestRefreshToken_Success` | 前提: ログインして取得した有効なリフレッシュトークン。入力: `refresh_token: <valid_token>` | 200。新しい `access_token` と `refresh_token` が返ること。レスポンスの `data` に最新の `user`, `tenant` 情報が含まれること |
| AUTH-043 | 統合 | handler | セキュリティ | AUTH-F03 | openapi.yaml#refreshToken, db_schema.md#refresh_tokens | `TestRefreshToken_Rotation` | 前提: 有効なリフレッシュトークン A。リフレッシュ後に旧トークン A を再度使用 | 1 回目: 200。2 回目: 401（旧リフレッシュトークンが無効化されていること） |
| AUTH-044 | 統合 | handler | 異常系 | AUTH-F03, SEC-005 | openapi.yaml#refreshToken, db_schema.md#refresh_tokens | `TestRefreshToken_RevokedToken` | 前提: ログアウト済みユーザーのリフレッシュトークン（`is_revoked=true`）。入力: そのリフレッシュトークン | 401。`error.code = UNAUTHORIZED` または `INVALID_TOKEN` |
| AUTH-045 | 統合 | handler | 異常系 | AUTH-F03, SEC-003 | openapi.yaml#refreshToken | `TestRefreshToken_ExpiredToken` | 有効期限切れのリフレッシュトークン（テスト用に `exp` を過去日時で発行） | 401。`error.code = TOKEN_EXPIRED` |
| AUTH-046 | 統合 | handler | 異常系 | AUTH-F03 | openapi.yaml#refreshToken | `TestRefreshToken_AccessTokenAsRefresh` | アクセストークン（`token_type=access`）を `refresh_token` として送信 | 401。`error.code = INVALID_TOKEN` |
| AUTH-047 | 統合 | handler | 異常系 | AUTH-F03 | openapi.yaml#refreshToken | `TestRefreshToken_InvalidFormat` | `refresh_token: "not.a.valid.jwt"` | 401。`error.code = INVALID_TOKEN` |
| AUTH-048 | 統合 | handler | 異常系 | AUTH-F03 | openapi.yaml#refreshToken | `TestRefreshToken_MissingBody` | `refresh_token` フィールドを省略 | 400。`error.code = BAD_REQUEST` |
| AUTH-049 | 統合 | handler | 正常系 | AUTH-F03, RBAC-F01 | openapi.yaml#refreshToken, db_schema.md#tenant_memberships | `TestRefreshToken_ReflectsRoleChange` | 前提: ユーザーのロールを DB で `admin` から `member` に変更後、旧アクセストークンのリフレッシュを実行 | 200。返却された新しいアクセストークンの claims に `role=member` が反映されること（DB の最新 `tenant_memberships` を再取得） |
| AUTH-050 | 統合 | handler | 正常系 | AUTH-F03 | openapi.yaml#refreshToken | `TestRefreshToken_NoAuthRequired` | 有効なリフレッシュトークン（Authorization ヘッダーなし） | 200（認証ヘッダーなしでアクセスできること） |

#### POST /api/auth/logout

仕様概要: リフレッシュトークンを無効化する。アクセストークンが期限切れでもログアウト可能。認証不要（リフレッシュトークン認証）。

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| AUTH-051 | 統合 | handler | 正常系 | AUTH-F04, SEC-005 | openapi.yaml#logout, db_schema.md#refresh_tokens | `TestLogout_Success` | 前提: ログインして取得した有効なリフレッシュトークン。入力: `refresh_token: <valid_token>` | 200。`data.message` が存在すること。DB の `refresh_tokens` テーブルで当該 `jti` の `is_revoked=true` であること |
| AUTH-052 | 統合 | handler | 正常系 | AUTH-F04 | openapi.yaml#logout | `TestLogout_WithExpiredAccessToken` | 前提: 有効なリフレッシュトークン（アクセストークンは期限切れ）。Authorization ヘッダーに期限切れアクセストークンを設定、ボディに有効なリフレッシュトークンを設定 | 200（アクセストークンの期限に依存しないこと） |
| AUTH-053 | 統合 | handler | 異常系 | AUTH-F04, SEC-005 | openapi.yaml#logout, db_schema.md#refresh_tokens | `TestLogout_AlreadyRevokedToken` | 前提: 既に `is_revoked=true` のリフレッシュトークン | 401。`error.code = UNAUTHORIZED` または `INVALID_TOKEN` |
| AUTH-054 | 統合 | handler | 異常系 | AUTH-F04 | openapi.yaml#logout | `TestLogout_InvalidRefreshToken` | `refresh_token: "invalid.token.value"` | 401。`error.code = INVALID_TOKEN` |
| AUTH-055 | 統合 | handler | 異常系 | AUTH-F04 | openapi.yaml#logout | `TestLogout_MissingRefreshToken` | `refresh_token` フィールドを省略 | 400。`error.code = BAD_REQUEST` |
| AUTH-056 | 統合 | handler | 正常系 | AUTH-F04 | openapi.yaml#logout | `TestLogout_NoAuthRequired` | 有効なリフレッシュトークン（Authorization ヘッダーなし） | 200（認証ヘッダーなしでアクセスできること） |

#### GET /api/auth/me

仕様概要: 現在のログインユーザーの情報を取得する。認証必須（全ロール許可）。

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| AUTH-057 | 統合 | handler | 正常系 | AUTH-F05 | openapi.yaml#getMe | `TestGetMe_Success_Admin` | 前提: Admin ユーザー（`test-admin@example.com`）のアクセストークン | 200。`data.user_id = aaaaaaaa-1111-1111-1111-000000000001`, `data.email = test-admin@example.com`, `data.role = admin`, `data.tenant_id = aaaaaaaa-0001-0001-0001-000000000001` |
| AUTH-058 | 統合 | handler | 正常系 | AUTH-F05 | openapi.yaml#getMe | `TestGetMe_Success_Member` | 前提: Member ユーザー（`test-member@example.com`）のアクセストークン | 200。`data.role = member` |
| AUTH-059 | 統合 | handler | 正常系 | AUTH-F05 | openapi.yaml#getMe | `TestGetMe_Success_Approver` | 前提: Approver ユーザー（`test-approver@example.com`）のアクセストークン | 200。`data.role = approver` |
| AUTH-060 | 統合 | handler | 正常系 | AUTH-F05 | openapi.yaml#getMe | `TestGetMe_Success_Accounting` | 前提: Accounting ユーザー（`test-accounting@example.com`）のアクセストークン | 200。`data.role = accounting` |
| AUTH-061 | 統合 | handler | 異常系 | AUTH-F05 | openapi.yaml#getMe | `TestGetMe_Unauthorized_NoToken` | Authorization ヘッダーなし | 401。`error.code = UNAUTHORIZED` |
| AUTH-062 | 統合 | handler | 異常系 | AUTH-F05, SEC-003 | openapi.yaml#getMe, security.md#2.1 | `TestGetMe_Unauthorized_ExpiredToken` | 有効期限切れのアクセストークン | 401。`error.code = TOKEN_EXPIRED` |
| AUTH-063 | 統合 | handler | 異常系 | AUTH-F05 | openapi.yaml#getMe | `TestGetMe_Unauthorized_InvalidToken` | `Authorization: Bearer not.a.real.token` | 401。`error.code = INVALID_TOKEN` |
| AUTH-064 | 統合 | handler | 異常系 | AUTH-F05 | openapi.yaml#getMe | `TestGetMe_Unauthorized_RefreshTokenAsAccess` | リフレッシュトークン（`token_type=refresh`）をアクセストークンとして Authorization ヘッダーに設定 | 401。`error.code = INVALID_TOKEN` |

#### POST /api/auth/password-reset

仕様概要: パスワードリセット用メールを送信する。ユーザーの存在に関わらず同一の成功レスポンスを返す（SEC-011）。認証不要。

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| AUTH-065 | 統合 | handler | 正常系 | AUTH-F06 | openapi.yaml#requestPasswordReset, db_schema.md#password_reset_tokens, security.md#2.3 | `TestRequestPasswordReset_ExistingEmail` | `email: "test-admin@example.com"` （フィクスチャに存在するメール） | 200。`data.message` が存在すること。DB の `password_reset_tokens` にレコードが作成されること |
| AUTH-066 | 統合 | handler | セキュリティ | AUTH-F06, SEC-011 | openapi.yaml#requestPasswordReset, security.md#2.4 | `TestRequestPasswordReset_NonExistentEmail` | `email: "nobody@example.com"` （存在しないメール） | 200。レスポンス内容が AUTH-065 と同一であること（ユーザーの存在を漏らさない: SEC-011） |
| AUTH-067 | 統合 | handler | 異常系 | AUTH-F06 | openapi.yaml#requestPasswordReset | `TestRequestPasswordReset_ValidationError_MissingEmail` | `email` フィールドを省略 | 422。`error.code = VALIDATION_ERROR` |
| AUTH-068 | 統合 | handler | 異常系 | AUTH-F06 | openapi.yaml#requestPasswordReset | `TestRequestPasswordReset_ValidationError_InvalidEmail` | `email: "not-valid"` | 422。`error.code = VALIDATION_ERROR` |
| AUTH-069 | 統合 | handler | 正常系 | AUTH-F06 | openapi.yaml#requestPasswordReset | `TestRequestPasswordReset_NoAuthRequired` | 有効なリクエスト（Authorization ヘッダーなし） | 200（認証ヘッダーなしでアクセスできること） |

#### PUT /api/auth/password-reset/{token}

仕様概要: リセットトークンを使用して新しいパスワードを設定する。トークンは 1 回使用で無効化される（SEC-006）。リセット実行時に全リフレッシュトークンを無効化する。認証不要。

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| AUTH-070 | 統合 | handler | 正常系 | AUTH-F06, SEC-006 | openapi.yaml#executePasswordReset, db_schema.md#password_reset_tokens, security.md#2.3 | `TestExecutePasswordReset_Success` | 前提: DB に有効なリセットトークンを直接挿入（`test-member@example.com` 宛、有効期限: 現在時刻 + 30 分）。入力: `token: <valid_token>`, `new_password: "NewPass1!"` | 200。`data.message` が存在すること。新しいパスワードでログイン（POST /api/auth/login）できること。使用済みトークンの `used_at` が記録されること。当該ユーザーの全リフレッシュトークンが無効化されること |
| AUTH-071 | 統合 | handler | 異常系 | AUTH-F06 | openapi.yaml#executePasswordReset | `TestExecutePasswordReset_InvalidToken` | `token: "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"` （存在しないトークン） | 422。`error.code` が含まれること |
| AUTH-072 | 統合 | handler | 異常系 | AUTH-F06 | openapi.yaml#executePasswordReset, db_schema.md#password_reset_tokens | `TestExecutePasswordReset_ExpiredToken` | 前提: DB に有効期限切れのリセットトークンを直接挿入（`expires_at` を過去日時に設定）。入力: そのトークン値 | 422。エラーレスポンスが返ること |
| AUTH-073 | 統合 | handler | セキュリティ | AUTH-F06, SEC-006 | openapi.yaml#executePasswordReset, db_schema.md#password_reset_tokens | `TestExecutePasswordReset_AlreadyUsedToken` | 前提: 既に `used_at` が設定されたリセットトークン（AUTH-070 で使用済みのもの）。入力: 同トークン | 422。エラーレスポンスが返ること（1 回限りの使用: SEC-006） |
| AUTH-074 | 統合 | handler | 境界値 | AUTH-F06, SEC-010 | openapi.yaml#executePasswordReset | `TestExecutePasswordReset_ValidationError_PasswordTooShort` | 有効なトークン、`new_password: "short"` (7文字) | 422。`error.code = VALIDATION_ERROR` |
| AUTH-075 | 統合 | handler | 異常系 | AUTH-F06 | openapi.yaml#executePasswordReset | `TestExecutePasswordReset_ValidationError_MissingPassword` | 有効なトークン、`new_password` フィールドを省略 | 422。`error.code = VALIDATION_ERROR` |
| AUTH-076 | 統合 | handler | 正常系 | AUTH-F06 | openapi.yaml#executePasswordReset | `TestExecutePasswordReset_NoAuthRequired` | 有効なトークンと有効なパスワード（Authorization ヘッダーなし） | 200（認証ヘッダーなしでアクセスできること） |

---

### 3. 共通セキュリティ・ミドルウェアの統合テスト

対応ファイル: `auth_handler_test.go`

認証ミドルウェアおよびセキュリティ設計（`security.md` §2.1）のエンドツーエンドな動作を検証する。

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| AUTH-077 | 統合 | handler | 正常系 | AUTH-F01, AUTH-F02, AUTH-F05 | openapi.yaml#signup, openapi.yaml#login, openapi.yaml#getMe | `TestAuth_AccessTokenFlow` | サインアップ → ログイン → GET /api/auth/me の一連のフロー | 各ステップで正しいレスポンスが返ること |
| AUTH-078 | 統合 | handler | セキュリティ | SEC-004 | security.md#2.1 | `TestAuth_InvalidAlgorithm` | none アルゴリズム（`"alg": "none"`）のトークンを Authorization ヘッダーに設定して GET /api/auth/me | 401。`error.code = INVALID_TOKEN`（alg 混乱攻撃の防止） |
| AUTH-079 | 統合 | handler | セキュリティ | SEC-004 | security.md#2.1 | `TestAuth_InvalidIssuer` | `iss: "other-service"` で生成されたトークンを Authorization ヘッダーに設定して GET /api/auth/me | 401。`error.code = INVALID_TOKEN` |
| AUTH-080 | 統合 | handler | 正常系 | AUTH-F01, AUTH-F02, AUTH-F03, AUTH-F04, AUTH-F06 | openapi.yaml#signup, openapi.yaml#login, openapi.yaml#refreshToken, openapi.yaml#logout, openapi.yaml#requestPasswordReset | `TestAuth_AuthEndpointsPubliclyAccessible` | signup, login, refresh, logout, password-reset の各エンドポイントに Authorization ヘッダーなしでリクエスト | 各エンドポイントが 401 を返さないこと（認証不要エンドポイントであること） |

---

### 4. FE テストケース

FE テストケースは `55_ui_component/screens/*.md` の §9 テスト追跡用設計識別子に列挙された全コンポーネント・Hook を網羅する。

参照設計書:
- `55_ui_component/screens/auth-login.md`
- `55_ui_component/screens/auth-signup.md`
- `55_ui_component/screens/auth-password-reset-request.md`
- `55_ui_component/screens/auth-password-reset.md`
- `55_ui_component/common-components.md`
- `55_ui_component/state-management.md`

共通コンポーネント（FormAlert, SubmitButton, AuthNavLinks）は複数画面で共有されるため、コンポーネント単体テストとして1回ずつ定義する。各画面固有のコンポーネントは画面単位で定義する。

#### 4.1 共通コンポーネント

##### FormAlert

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| AUTH-FE-001 | 単体 | FormAlert | `message: string \| null` | 正常系 | - | `55_ui_component/screens/auth-login.md §FormAlert` | `test_FormAlert_renders_error_message` | `message="メールアドレスまたはパスワードが正しくありません"`, `severity="error"` | Alert コンポーネントが表示され、指定メッセージが描画されること |
| AUTH-FE-002 | 単体 | FormAlert | `message: null` | 正常系 | - | `55_ui_component/screens/auth-login.md §FormAlert` | `test_FormAlert_hidden_when_message_null` | `message=null` | Alert コンポーネントが DOM に存在しないこと |
| AUTH-FE-003 | 単体 | FormAlert | `severity: 'warning'` | 正常系 | - | `55_ui_component/screens/auth-login.md §FormAlert` | `test_FormAlert_renders_with_custom_severity` | `message="しばらく待ってから再試行してください"`, `severity="warning"` | Alert が warning の見た目で描画されること |

##### SubmitButton

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| AUTH-FE-004 | 単体 | SubmitButton | `label: string, loading: false` | 正常系 | - | `55_ui_component/screens/auth-login.md §SubmitButton` | `test_SubmitButton_renders_label` | `label="ログイン"`, `loading=false` | ボタンテキストが「ログイン」で、ボタンが有効（disabled ではない）であること |
| AUTH-FE-005 | 単体 | SubmitButton | `loading: true` | 正常系 | - | `55_ui_component/screens/auth-login.md §SubmitButton` | `test_SubmitButton_disabled_with_spinner_when_loading` | `label="ログイン"`, `loading=true` | ボタンが disabled であり、スピナー（CircularProgress）が表示されること |

##### AuthNavLinks

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| AUTH-FE-006 | 単体 | AuthNavLinks | `links: AuthNavLink[]` | 正常系 | - | `55_ui_component/screens/auth-login.md §AuthNavLinks` | `test_AuthNavLinks_renders_links` | `links=[{ prefix: "アカウントをお持ちでない方は", label: "新規登録", to: "/signup" }, { prefix: "パスワードを忘れた方は", label: "パスワードリセット", to: "/password-reset" }]` | 各リンクの prefix テキスト・リンクテキスト・href が正しく描画されること |
| AUTH-FE-007 | 単体 | AuthNavLinks | `links: AuthNavLink[]`（単一リンク） | 正常系 | - | `55_ui_component/screens/auth-signup.md §AuthNavLinks` | `test_AuthNavLinks_renders_single_link` | `links=[{ prefix: "既にアカウントをお持ちの方は", label: "ログイン", to: "/login" }]` | 1 件のリンクのみが描画されること |
| AUTH-FE-007-A | 単体 | AuthNavLinks | `links: AuthNavLink[]`（prefix 省略） | 正常系 | - | `55_ui_component/screens/auth-password-reset-request.md §AuthNavLinks` | `test_AuthNavLinks_renders_link_without_prefix` | `links=[{ label: "ログイン画面に戻る", to: "/login" }]` | label のみ（prefix テキストなし）でリンクが描画されること |

#### 4.2 ログイン画面（SCR-AUTH-002）

##### LoginPage

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| AUTH-FE-008 | 単体 | LoginPage | useLogin（成功） | 正常系 | AUTH-F02 | `55_ui_component/screens/auth-login.md §LoginPage` | `test_LoginPage_success_saves_tokens_and_redirects` | 前提: useLogin をモック。フォームに有効な email/password を入力して送信。useLogin が `{ access_token, refresh_token }` を返す | AuthStore に setTokens が呼ばれ、ダッシュボード（/dashboard）に遷移すること |
| AUTH-FE-009 | 単体 | LoginPage | useLogin（成功 + リダイレクト元あり） | 正常系 | AUTH-F02 | `55_ui_component/screens/auth-login.md §LoginPage` | `test_LoginPage_success_redirects_to_original_path` | 前提: React Router の location.state に `{ from: "/reports" }` を設定。ログイン成功 | リダイレクト元の `/reports` に遷移すること |
| AUTH-FE-010 | 単体 | LoginPage | useLogin（401 エラー） | 異常系 | AUTH-F02, SEC-011 | `55_ui_component/screens/auth-login.md §LoginPage` | `test_LoginPage_auth_error_displays_unified_message` | 前提: useLogin が 401 INVALID_CREDENTIALS エラーを返す | LoginForm の apiError に SEC-011 準拠の統一メッセージ「メールアドレスまたはパスワードが正しくありません」が伝播されること |
| AUTH-FE-011 | 単体 | LoginPage | useLogin（429 エラー） | 異常系 | AUTH-F02 | `55_ui_component/screens/auth-login.md §LoginPage` | `test_LoginPage_rate_limit_error` | 前提: useLogin が 429 RATE_LIMIT_EXCEEDED エラーを返す | apiError にレート制限エラーメッセージが設定されること |
| AUTH-FE-012 | 単体 | LoginPage | useLogin（500 エラー） | 異常系 | AUTH-F02 | `55_ui_component/screens/auth-login.md §LoginPage` | `test_LoginPage_server_error` | 前提: useLogin が 500 INTERNAL_ERROR エラーを返す | apiError にサーバーエラーメッセージが設定されること |
| AUTH-FE-013 | 単体 | LoginPage | AuthLayout（認証済みリダイレクト） | 認可 | AUTH-F02 | `55_ui_component/screens/auth-login.md §LoginPage` | `test_LoginPage_redirects_when_authenticated` | 前提: useAuth が `{ isAuthenticated: true }` を返す | ダッシュボード（/dashboard）にリダイレクトされること |

##### LoginForm

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| AUTH-FE-014 | 単体 | LoginForm | `onSubmit, apiError: null, isPending: false` | 正常系 | AUTH-F02 | `55_ui_component/screens/auth-login.md §LoginForm` | `test_LoginForm_renders_email_and_password_fields` | `apiError=null`, `isPending=false` | メールアドレスとパスワードの入力フィールド、ログインボタンが描画されること |
| AUTH-FE-015 | 単体 | LoginForm | `onSubmit`（有効な入力） | 正常系 | AUTH-F02 | `55_ui_component/screens/auth-login.md §LoginForm` | `test_LoginForm_calls_onSubmit_with_valid_input` | email: `"user@example.com"`, password: `"TestPass1!"` を入力して送信 | `onSubmit` が `{ email: "user@example.com", password: "TestPass1!" }` で呼ばれること |
| AUTH-FE-016 | 単体 | LoginForm | loginSchema（email 空） | 異常系 | AUTH-F02 | `55_ui_component/screens/auth-login.md §LoginForm` | `test_LoginForm_validation_email_required` | email を空のまま送信 | メールアドレスフィールドに必須エラーが表示されること。`onSubmit` が呼ばれないこと |
| AUTH-FE-017 | 単体 | LoginForm | loginSchema（email 形式不正） | 異常系 | AUTH-F02 | `55_ui_component/screens/auth-login.md §LoginForm` | `test_LoginForm_validation_email_invalid_format` | email: `"not-an-email"` を入力して送信 | メールアドレスフィールドに形式エラーが表示されること。`onSubmit` が呼ばれないこと |
| AUTH-FE-018 | 単体 | LoginForm | loginSchema（password 空） | 異常系 | AUTH-F02 | `55_ui_component/screens/auth-login.md §LoginForm` | `test_LoginForm_validation_password_required` | password を空のまま送信 | パスワードフィールドに必須エラーが表示されること。`onSubmit` が呼ばれないこと |
| AUTH-FE-019 | 単体 | LoginForm | `apiError: string` | 異常系 | AUTH-F02, SEC-011 | `55_ui_component/screens/auth-login.md §LoginForm` | `test_LoginForm_displays_api_error_in_FormAlert` | `apiError="メールアドレスまたはパスワードが正しくありません"` | フォーム上部に FormAlert でエラーメッセージが表示されること |
| AUTH-FE-020 | 単体 | LoginForm | `isPending: true` | 正常系 | AUTH-F02 | `55_ui_component/screens/auth-login.md §LoginForm` | `test_LoginForm_disables_fields_when_pending` | `isPending=true` | メールアドレスフィールド、パスワードフィールド、送信ボタンが全て disabled であること |

##### useLogin Hook

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| AUTH-FE-021 | 単体 | - | useLogin | 正常系 | AUTH-F02 | `55_ui_component/state-management.md §useLogin` | `test_useLogin_calls_api_and_returns_tokens` | 前提: POST /api/auth/login を MSW でモック（200 + `{ access_token, refresh_token }`）。入力: `{ email: "user@example.com", password: "TestPass1!" }` | mutateAsync が AuthTokens を返すこと |
| AUTH-FE-022 | 単体 | - | useLogin | 異常系 | AUTH-F02, SEC-011 | `55_ui_component/state-management.md §useLogin` | `test_useLogin_handles_401_invalid_credentials` | 前提: POST /api/auth/login を MSW でモック（401 + `{ code: "INVALID_CREDENTIALS" }`） | エラーが ApiClientError で、SEC-011 に準拠した統一エラーメッセージに変換できること |
| AUTH-FE-023 | 単体 | - | useLogin | 異常系 | AUTH-F02 | `55_ui_component/state-management.md §useLogin` | `test_useLogin_handles_429_rate_limit` | 前提: POST /api/auth/login を MSW でモック（429 + `{ code: "RATE_LIMIT_EXCEEDED" }`） | エラーが ApiClientError で、レート制限メッセージに変換できること |
| AUTH-FE-024 | 単体 | - | useLogin | 正常系 | AUTH-F02 | `55_ui_component/state-management.md §useLogin` | `test_useLogin_saves_tokens_to_AuthStore` | 前提: POST /api/auth/login を MSW でモック（200 成功）。onSuccess で setTokens を呼ぶことを確認 | AuthStore.setTokens が access_token と refresh_token で呼ばれること |

#### 4.3 サインアップ画面（SCR-AUTH-001）

##### SignupPage

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| AUTH-FE-025 | 単体 | SignupPage | useSignup（成功） | 正常系 | AUTH-F01 | `55_ui_component/screens/auth-signup.md §SignupPage` | `test_SignupPage_success_saves_tokens_and_redirects` | 前提: useSignup をモック。フォームに有効な入力値を設定して送信。useSignup が `{ access_token, refresh_token }` を返す | AuthStore に setTokens が呼ばれ、ダッシュボード（/dashboard）に遷移すること |
| AUTH-FE-026 | 単体 | SignupPage | useSignup（409 エラー） | 異常系 | AUTH-F01 | `55_ui_component/screens/auth-signup.md §SignupPage` | `test_SignupPage_duplicate_email_error` | 前提: useSignup が 409 EMAIL_ALREADY_EXISTS エラーを返す | apiError にメールアドレス重複エラーメッセージが設定されること |
| AUTH-FE-027 | 単体 | SignupPage | useSignup（429 エラー） | 異常系 | AUTH-F01 | `55_ui_component/screens/auth-signup.md §SignupPage` | `test_SignupPage_rate_limit_error` | 前提: useSignup が 429 RATE_LIMIT_EXCEEDED エラーを返す | apiError にレート制限エラーメッセージが設定されること |
| AUTH-FE-028 | 単体 | SignupPage | useSignup（500 エラー） | 異常系 | AUTH-F01 | `55_ui_component/screens/auth-signup.md §SignupPage` | `test_SignupPage_server_error` | 前提: useSignup が 500 INTERNAL_ERROR エラーを返す | apiError にサーバーエラーメッセージが設定されること |
| AUTH-FE-029 | 単体 | SignupPage | AuthLayout（認証済みリダイレクト） | 認可 | AUTH-F01 | `55_ui_component/screens/auth-signup.md §SignupPage` | `test_SignupPage_redirects_when_authenticated` | 前提: useAuth が `{ isAuthenticated: true }` を返す | ダッシュボード（/dashboard）にリダイレクトされること |

##### SignupForm

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| AUTH-FE-030 | 単体 | SignupForm | `onSubmit, apiError: null, isPending: false` | 正常系 | AUTH-F01 | `55_ui_component/screens/auth-signup.md §SignupForm` | `test_SignupForm_renders_all_fields` | `apiError=null`, `isPending=false` | 会社名、ユーザー名、メールアドレス、パスワードの入力フィールドと送信ボタンが描画されること |
| AUTH-FE-031 | 単体 | SignupForm | `onSubmit`（有効な入力） | 正常系 | AUTH-F01 | `55_ui_component/screens/auth-signup.md §SignupForm` | `test_SignupForm_calls_onSubmit_with_valid_input` | company_name: `"Test Corp"`, user_name: `"Test User"`, email: `"new@example.com"`, password: `"TestPass1!"` を入力して送信 | `onSubmit` が正しい SignupInput で呼ばれること |
| AUTH-FE-032 | 単体 | SignupForm | signupSchema（company_name 空） | 異常系 | AUTH-F01 | `55_ui_component/screens/auth-signup.md §SignupForm` | `test_SignupForm_validation_company_name_required` | company_name を空のまま送信 | 会社名フィールドに必須エラーが表示されること。`onSubmit` が呼ばれないこと |
| AUTH-FE-033 | 単体 | SignupForm | signupSchema（company_name 201 文字） | 境界値 | AUTH-F01 | `55_ui_component/screens/auth-signup.md §SignupForm` | `test_SignupForm_validation_company_name_too_long` | company_name に 201 文字の文字列を入力して送信 | 会社名フィールドに文字数超過エラーが表示されること |
| AUTH-FE-034 | 単体 | SignupForm | signupSchema（user_name 空） | 異常系 | AUTH-F01 | `55_ui_component/screens/auth-signup.md §SignupForm` | `test_SignupForm_validation_user_name_required` | user_name を空のまま送信 | ユーザー名フィールドに必須エラーが表示されること |
| AUTH-FE-035 | 単体 | SignupForm | signupSchema（email 形式不正） | 異常系 | AUTH-F01 | `55_ui_component/screens/auth-signup.md §SignupForm` | `test_SignupForm_validation_email_invalid_format` | email: `"not-an-email"` を入力して送信 | メールアドレスフィールドに形式エラーが表示されること |
| AUTH-FE-036 | 単体 | SignupForm | signupSchema（password 7 文字） | 境界値 | AUTH-F01, SEC-010 | `55_ui_component/screens/auth-signup.md §SignupForm` | `test_SignupForm_validation_password_too_short` | password: `"Short1!"` (7 文字) を入力して送信 | パスワードフィールドに最小長エラーが表示されること |
| AUTH-FE-037 | 単体 | SignupForm | signupSchema（password 129 文字） | 境界値 | AUTH-F01 | `55_ui_component/screens/auth-signup.md §SignupForm` | `test_SignupForm_validation_password_too_long` | password に 129 文字の文字列を入力して送信 | パスワードフィールドに文字数超過エラーが表示されること |
| AUTH-FE-038 | 単体 | SignupForm | `apiError: string` | 異常系 | AUTH-F01 | `55_ui_component/screens/auth-signup.md §SignupForm` | `test_SignupForm_displays_api_error_in_FormAlert` | `apiError="このメールアドレスは既に登録されています"` | フォーム上部に FormAlert でエラーメッセージが表示されること |
| AUTH-FE-039 | 単体 | SignupForm | `isPending: true` | 正常系 | AUTH-F01 | `55_ui_component/screens/auth-signup.md §SignupForm` | `test_SignupForm_disables_fields_when_pending` | `isPending=true` | 全入力フィールドと送信ボタンが disabled であること |

##### useSignup Hook

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| AUTH-FE-040 | 単体 | - | useSignup | 正常系 | AUTH-F01 | `55_ui_component/state-management.md §useSignup` | `test_useSignup_calls_api_and_returns_tokens` | 前提: POST /api/auth/signup を MSW でモック（201 + `{ access_token, refresh_token }`）。入力: 有効な SignupInput | mutateAsync が AuthTokens を返すこと |
| AUTH-FE-041 | 単体 | - | useSignup | 異常系 | AUTH-F01 | `55_ui_component/state-management.md §useSignup` | `test_useSignup_handles_409_email_exists` | 前提: POST /api/auth/signup を MSW でモック（409 + `{ code: "EMAIL_ALREADY_EXISTS" }`） | エラーが ApiClientError で、エラーコードが EMAIL_ALREADY_EXISTS であること |
| AUTH-FE-042 | 単体 | - | useSignup | 正常系 | AUTH-F01 | `55_ui_component/state-management.md §useSignup` | `test_useSignup_saves_tokens_to_AuthStore` | 前提: POST /api/auth/signup を MSW でモック（201 成功）。onSuccess で setTokens を呼ぶことを確認 | AuthStore.setTokens が access_token と refresh_token で呼ばれること |

#### 4.4 パスワードリセット要求画面（SCR-AUTH-003）

##### PasswordResetRequestPage

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| AUTH-FE-043 | 単体 | PasswordResetRequestPage | useRequestPasswordReset（成功） | 正常系 | AUTH-F06, SEC-011 | `55_ui_component/screens/auth-password-reset-request.md §PasswordResetRequestPage` | `test_PasswordResetRequestPage_success_shows_complete` | 前提: useRequestPasswordReset をモック（成功）。フォームに有効な email を入力して送信 | isSubmitted が true に切り替わり、PasswordResetRequestComplete が表示され、PasswordResetRequestForm が非表示になること |
| AUTH-FE-044 | 単体 | PasswordResetRequestPage | useRequestPasswordReset（500 エラー） | 異常系 | AUTH-F06 | `55_ui_component/screens/auth-password-reset-request.md §PasswordResetRequestPage` | `test_PasswordResetRequestPage_server_error` | 前提: useRequestPasswordReset が 500 INTERNAL_ERROR エラーを返す | apiError にサーバーエラーメッセージが設定され、フォームが表示されたままであること |
| AUTH-FE-045 | 単体 | PasswordResetRequestPage | useRequestPasswordReset（429 エラー） | 異常系 | AUTH-F06 | `55_ui_component/screens/auth-password-reset-request.md §PasswordResetRequestPage` | `test_PasswordResetRequestPage_rate_limit_error` | 前提: useRequestPasswordReset が 429 RATE_LIMIT_EXCEEDED エラーを返す | apiError にレート制限エラーメッセージが設定されること |
| AUTH-FE-046 | 単体 | PasswordResetRequestPage | AuthLayout（認証済みリダイレクト） | 認可 | AUTH-F06 | `55_ui_component/screens/auth-password-reset-request.md §PasswordResetRequestPage` | `test_PasswordResetRequestPage_redirects_when_authenticated` | 前提: useAuth が `{ isAuthenticated: true }` を返す | ダッシュボード（/dashboard）にリダイレクトされること |
| AUTH-FE-047 | 単体 | PasswordResetRequestPage | 表示状態の切替 | 正常系 | AUTH-F06 | `55_ui_component/screens/auth-password-reset-request.md §PasswordResetRequestPage` | `test_PasswordResetRequestPage_initial_state_shows_form` | 初期表示 | PasswordResetRequestForm が表示され、PasswordResetRequestComplete が非表示であること |

##### PasswordResetRequestForm

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| AUTH-FE-048 | 単体 | PasswordResetRequestForm | `onSubmit, apiError: null, isPending: false` | 正常系 | AUTH-F06 | `55_ui_component/screens/auth-password-reset-request.md §PasswordResetRequestForm` | `test_PasswordResetRequestForm_renders_email_field` | `apiError=null`, `isPending=false` | メールアドレス入力フィールドと送信ボタンが描画されること |
| AUTH-FE-049 | 単体 | PasswordResetRequestForm | `onSubmit`（有効な入力） | 正常系 | AUTH-F06 | `55_ui_component/screens/auth-password-reset-request.md §PasswordResetRequestForm` | `test_PasswordResetRequestForm_calls_onSubmit_with_valid_email` | email: `"user@example.com"` を入力して送信 | `onSubmit` が `{ email: "user@example.com" }` で呼ばれること |
| AUTH-FE-050 | 単体 | PasswordResetRequestForm | passwordResetRequestSchema（email 空） | 異常系 | AUTH-F06 | `55_ui_component/screens/auth-password-reset-request.md §PasswordResetRequestForm` | `test_PasswordResetRequestForm_validation_email_required` | email を空のまま送信 | メールアドレスフィールドに必須エラーが表示されること。`onSubmit` が呼ばれないこと |
| AUTH-FE-051 | 単体 | PasswordResetRequestForm | passwordResetRequestSchema（email 形式不正） | 異常系 | AUTH-F06 | `55_ui_component/screens/auth-password-reset-request.md §PasswordResetRequestForm` | `test_PasswordResetRequestForm_validation_email_invalid` | email: `"not-valid"` を入力して送信 | メールアドレスフィールドに形式エラーが表示されること |
| AUTH-FE-052 | 単体 | PasswordResetRequestForm | `apiError: string` | 異常系 | AUTH-F06 | `55_ui_component/screens/auth-password-reset-request.md §PasswordResetRequestForm` | `test_PasswordResetRequestForm_displays_api_error` | `apiError="サーバーエラーが発生しました"` | フォーム上部に FormAlert でエラーメッセージが表示されること |
| AUTH-FE-053 | 単体 | PasswordResetRequestForm | `isPending: true` | 正常系 | AUTH-F06 | `55_ui_component/screens/auth-password-reset-request.md §PasswordResetRequestForm` | `test_PasswordResetRequestForm_disables_fields_when_pending` | `isPending=true` | メールアドレスフィールドと送信ボタンが disabled であること |

##### PasswordResetRequestComplete

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| AUTH-FE-054 | 単体 | PasswordResetRequestComplete | Props なし | 正常系 | AUTH-F06, SEC-011 | `55_ui_component/screens/auth-password-reset-request.md §PasswordResetRequestComplete` | `test_PasswordResetRequestComplete_renders_message` | コンポーネントを描画 | 送信完了メッセージ（メール送信案内）と迷惑メールフォルダの注意書きが表示されること |

##### useRequestPasswordReset Hook

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| AUTH-FE-055 | 単体 | - | useRequestPasswordReset | 正常系 | AUTH-F06 | `55_ui_component/state-management.md §useRequestPasswordReset` | `test_useRequestPasswordReset_calls_api_and_succeeds` | 前提: POST /api/auth/password-reset を MSW でモック（200 + `{ message: "..." }`）。入力: `{ email: "user@example.com" }` | mutateAsync が成功し、レスポンスに message が含まれること |
| AUTH-FE-056 | 単体 | - | useRequestPasswordReset | 異常系 | AUTH-F06 | `55_ui_component/state-management.md §useRequestPasswordReset` | `test_useRequestPasswordReset_handles_500_error` | 前提: POST /api/auth/password-reset を MSW でモック（500 + `{ code: "INTERNAL_ERROR" }`） | エラーが ApiClientError であること |

#### 4.5 パスワードリセット実行画面（SCR-AUTH-004）

##### PasswordResetPage

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| AUTH-FE-057 | 単体 | PasswordResetPage | useExecutePasswordReset（成功） | 正常系 | AUTH-F06 | `55_ui_component/screens/auth-password-reset.md §PasswordResetPage` | `test_PasswordResetPage_success_shows_complete` | 前提: URL パラメータ `:token` に有効なトークンを設定。useExecutePasswordReset をモック（成功）。フォームに有効なパスワードを入力して送信 | viewState が 'complete' に切り替わり、PasswordResetComplete が表示されること |
| AUTH-FE-058 | 単体 | PasswordResetPage | useExecutePasswordReset（INVALID_TOKEN エラー） | 異常系 | AUTH-F06 | `55_ui_component/screens/auth-password-reset.md §PasswordResetPage` | `test_PasswordResetPage_invalid_token_shows_error_view` | 前提: useExecutePasswordReset が INVALID_TOKEN エラーを返す | viewState が 'token-invalid' に切り替わり、PasswordResetTokenInvalid が表示されること |
| AUTH-FE-059 | 単体 | PasswordResetPage | useExecutePasswordReset（500 エラー） | 異常系 | AUTH-F06 | `55_ui_component/screens/auth-password-reset.md §PasswordResetPage` | `test_PasswordResetPage_server_error` | 前提: useExecutePasswordReset が 500 INTERNAL_ERROR エラーを返す | apiError にサーバーエラーメッセージが設定され、フォーム表示状態が維持されること |
| AUTH-FE-060 | 単体 | PasswordResetPage | useExecutePasswordReset（429 エラー） | 異常系 | AUTH-F06 | `55_ui_component/screens/auth-password-reset.md §PasswordResetPage` | `test_PasswordResetPage_rate_limit_error` | 前提: useExecutePasswordReset が 429 RATE_LIMIT_EXCEEDED エラーを返す | apiError にレート制限エラーメッセージが設定されること |
| AUTH-FE-061 | 単体 | PasswordResetPage | AuthLayout（認証済みリダイレクト） | 認可 | AUTH-F06 | `55_ui_component/screens/auth-password-reset.md §PasswordResetPage` | `test_PasswordResetPage_redirects_when_authenticated` | 前提: useAuth が `{ isAuthenticated: true }` を返す | ダッシュボード（/dashboard）にリダイレクトされること |
| AUTH-FE-062 | 単体 | PasswordResetPage | URL パラメータ取得 | 正常系 | AUTH-F06 | `55_ui_component/screens/auth-password-reset.md §PasswordResetPage` | `test_PasswordResetPage_extracts_token_from_url` | 前提: URL が `/password-reset/abc123token` | URL パラメータからトークン `abc123token` が取得され、useExecutePasswordReset の引数に含まれること |
| AUTH-FE-063 | 単体 | PasswordResetPage | 表示状態の切替 | 正常系 | AUTH-F06 | `55_ui_component/screens/auth-password-reset.md §PasswordResetPage` | `test_PasswordResetPage_initial_state_shows_form` | 初期表示 | PasswordResetForm と AuthNavLinks が表示され、PasswordResetComplete と PasswordResetTokenInvalid が非表示であること |

##### PasswordResetForm

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| AUTH-FE-064 | 単体 | PasswordResetForm | `onSubmit, apiError: null, isPending: false` | 正常系 | AUTH-F06 | `55_ui_component/screens/auth-password-reset.md §PasswordResetForm` | `test_PasswordResetForm_renders_password_fields` | `apiError=null`, `isPending=false` | 新しいパスワードと確認用パスワードの入力フィールド、送信ボタンが描画されること |
| AUTH-FE-065 | 単体 | PasswordResetForm | `onSubmit`（有効な入力） | 正常系 | AUTH-F06 | `55_ui_component/screens/auth-password-reset.md §PasswordResetForm` | `test_PasswordResetForm_calls_onSubmit_with_valid_input` | new_password: `"NewPass1!"`, confirm_password: `"NewPass1!"` を入力して送信 | `onSubmit` が `{ new_password: "NewPass1!" }` で呼ばれること（confirm_password は API に送信しない） |
| AUTH-FE-066 | 単体 | PasswordResetForm | passwordResetSchema（new_password 空） | 異常系 | AUTH-F06 | `55_ui_component/screens/auth-password-reset.md §PasswordResetForm` | `test_PasswordResetForm_validation_password_required` | new_password を空のまま送信 | パスワードフィールドに必須エラーが表示されること |
| AUTH-FE-067 | 単体 | PasswordResetForm | passwordResetSchema（new_password 7 文字） | 境界値 | AUTH-F06, SEC-010 | `55_ui_component/screens/auth-password-reset.md §PasswordResetForm` | `test_PasswordResetForm_validation_password_too_short` | new_password: `"Short1!"` (7 文字), confirm_password: `"Short1!"` を入力して送信 | パスワードフィールドに最小長エラーが表示されること |
| AUTH-FE-068 | 単体 | PasswordResetForm | passwordResetSchema（パスワード不一致） | 異常系 | AUTH-F06 | `55_ui_component/screens/auth-password-reset.md §PasswordResetForm` | `test_PasswordResetForm_validation_password_mismatch` | new_password: `"NewPass1!"`, confirm_password: `"DiffPass1!"` を入力して送信 | 確認用パスワードフィールドにパスワード不一致エラーが表示されること |
| AUTH-FE-069 | 単体 | PasswordResetForm | `apiError: string` | 異常系 | AUTH-F06 | `55_ui_component/screens/auth-password-reset.md §PasswordResetForm` | `test_PasswordResetForm_displays_api_error` | `apiError="サーバーエラーが発生しました"` | フォーム上部に FormAlert でエラーメッセージが表示されること |
| AUTH-FE-070 | 単体 | PasswordResetForm | `isPending: true` | 正常系 | AUTH-F06 | `55_ui_component/screens/auth-password-reset.md §PasswordResetForm` | `test_PasswordResetForm_disables_fields_when_pending` | `isPending=true` | 全入力フィールドと送信ボタンが disabled であること |

##### PasswordResetComplete

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| AUTH-FE-071 | 単体 | PasswordResetComplete | Props なし | 正常系 | AUTH-F06 | `55_ui_component/screens/auth-password-reset.md §PasswordResetComplete` | `test_PasswordResetComplete_renders_message_and_link` | コンポーネントを描画 | パスワード変更完了メッセージが表示され、「ログイン画面へ」ボタン（リンク先: /login）が描画されること |

##### PasswordResetTokenInvalid

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| AUTH-FE-072 | 単体 | PasswordResetTokenInvalid | Props なし | 異常系 | AUTH-F06 | `55_ui_component/screens/auth-password-reset.md §PasswordResetTokenInvalid` | `test_PasswordResetTokenInvalid_renders_message_and_link` | コンポーネントを描画 | トークン無効・期限切れメッセージが表示され、「パスワードリセット画面へ」ボタン（リンク先: /password-reset）が描画されること |

##### useExecutePasswordReset Hook

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| AUTH-FE-073 | 単体 | - | useExecutePasswordReset | 正常系 | AUTH-F06 | `55_ui_component/state-management.md §useExecutePasswordReset` | `test_useExecutePasswordReset_calls_api_and_succeeds` | 前提: PUT /api/auth/password-reset/:token を MSW でモック（200 + `{ message: "..." }`）。入力: `{ token: "valid-token", new_password: "NewPass1!" }` | mutateAsync が成功し、レスポンスに message が含まれること |
| AUTH-FE-074 | 単体 | - | useExecutePasswordReset | 異常系 | AUTH-F06 | `55_ui_component/state-management.md §useExecutePasswordReset` | `test_useExecutePasswordReset_handles_invalid_token` | 前提: PUT /api/auth/password-reset/:token を MSW でモック（422 + `{ code: "INVALID_TOKEN" }`） | エラーが ApiClientError で、エラーコードが INVALID_TOKEN であること |
| AUTH-FE-075 | 単体 | - | useExecutePasswordReset | 異常系 | AUTH-F06 | `55_ui_component/state-management.md §useExecutePasswordReset` | `test_useExecutePasswordReset_handles_expired_token` | 前提: PUT /api/auth/password-reset/:token を MSW でモック（422 + `{ code: "INVALID_TOKEN" }`、期限切れトークン） | エラーが ApiClientError で、エラーコードが INVALID_TOKEN であること |
| AUTH-FE-076 | 単体 | - | useExecutePasswordReset | 異常系 | AUTH-F06 | `55_ui_component/state-management.md §useExecutePasswordReset` | `test_useExecutePasswordReset_handles_500_error` | 前提: PUT /api/auth/password-reset/:token を MSW でモック（500 + `{ code: "INTERNAL_ERROR" }`） | エラーが ApiClientError であること |

---

#### 認証フォーム autoComplete 属性 -- ブラウザ・パスワードマネージャー連携回帰防止（issue #171）

issue #171 の不具合（メールアドレス欄に `autoComplete` 属性が欠落しブラウザのパスワードマネージャーがサジェスト候補を出さない）に対する回帰防止テスト。各認証フォームのメールアドレス欄・パスワード欄に W3C / WHATWG 仕様準拠の `autoComplete` 属性が付与されていることを単体テストで検証する。

観点: フォーム要素にブラウザ・パスワードマネージャー連携用の `autoComplete` 属性が付与されている

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| AUTH-FE-077 | 単体 | LoginForm | メールアドレス欄 | 正常系 | AUTH-F01 | `50_detail_design/screens/login.md` | `sets_autocomplete_username_on_email_field` | LoginForm をデフォルト props でレンダリング。前提: メールアドレス欄は `type="text"`, `inputMode="email"`, `autoComplete="username"` で実装されていること（issue #171 追加対応: `type="email"` のままでは Chromium/Edge の Autofill API 経路に流れパスワードマネージャー候補表示が抑制されるため `type="text"` に変更）。 | メールアドレス欄（`getByLabelText('メールアドレス')`）の `autocomplete` 属性が `"username"` であること。加えて `type` 属性が `"text"` であること、`inputmode` 属性が `"email"` であること |
| AUTH-FE-078 | 単体 | LoginForm | パスワード欄 | 正常系 | AUTH-F01 | `50_detail_design/screens/login.md` | `sets_autocomplete_current_password_on_password_field` | LoginForm をデフォルト props でレンダリング | パスワード欄（`getByLabelText('パスワード')`）の `autocomplete` 属性が `"current-password"` であること |
| AUTH-FE-079 | 単体 | SignupForm | メールアドレス欄 | 正常系 | AUTH-F03 | `50_detail_design/screens/signup.md` | `sets_autocomplete_email_on_email_field` | SignupForm をデフォルト props でレンダリング | メールアドレス欄（`getByLabelText('メールアドレス')`）の `autocomplete` 属性が `"email"` であること |
| AUTH-FE-080 | 単体 | SignupForm | パスワード欄 | 正常系 | AUTH-F03 | `50_detail_design/screens/signup.md` | `sets_autocomplete_new_password_on_password_field` | SignupForm をデフォルト props でレンダリング | パスワード欄（`getByLabelText('パスワード')`）の `autocomplete` 属性が `"new-password"` であること |
| AUTH-FE-081 | 単体 | PasswordResetRequestForm | メールアドレス欄 | 正常系 | AUTH-F05 | `50_detail_design/screens/password-reset-request.md` | `sets_autocomplete_email_on_email_field` | PasswordResetRequestForm をデフォルト props でレンダリング | メールアドレス欄（`getByLabelText('メールアドレス')`）の `autocomplete` 属性が `"email"` であること |
| AUTH-FE-082 | 単体 | PasswordResetForm | 新しいパスワード欄 | 正常系 | AUTH-F06 | `50_detail_design/screens/password-reset.md` | `sets_autocomplete_new_password_on_new_password_field` | PasswordResetForm をデフォルト props でレンダリング | 新しいパスワード欄（`getByLabelText('新しいパスワード')`）の `autocomplete` 属性が `"new-password"` であること |
| AUTH-FE-083 | 単体 | PasswordResetForm | 確認用パスワード欄 | 正常系 | AUTH-F06 | `50_detail_design/screens/password-reset.md` | `sets_autocomplete_new_password_on_confirm_password_field` | PasswordResetForm をデフォルト props でレンダリング | 確認用パスワード欄（`getByLabelText('確認用パスワード')`）の `autocomplete` 属性が `"new-password"` であること |

---

### 4.6 PrivateRoute テスト

テスト ID プレフィックス: `PRT-`（PrivateRoute 専用）

対応テストファイル: `src/components/auth/__tests__/PrivateRoute.test.tsx`

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| PRT-001 | 単体 | PrivateRoute | useAuth (isAuthenticated: true) | 正常系 | AUTH-F02 | `55_ui_component/screens/auth-login.md §PrivateRoute` | `test_PrivateRoute_authenticated_renders_outlet` | useAuth が `{ isAuthenticated: true }` を返すようモック | Outlet（子ルート）が描画される。/login にリダイレクトされない |
| PRT-002 | 単体 | PrivateRoute | useAuth (isAuthenticated: false) | 認可 | AUTH-F02 | `55_ui_component/screens/auth-login.md §PrivateRoute` | `test_PrivateRoute_unauthenticated_redirects_to_login` | useAuth が `{ isAuthenticated: false }` を返すようモック | /login にリダイレクトされる。protected ページは描画されない |
| PRT-003 | 単体 | PrivateRoute | useAuth (isAuthenticated: false) | 正常系 | AUTH-F02 | `55_ui_component/screens/auth-login.md §PrivateRoute` | `test_PrivateRoute_preserves_original_url_in_state` | useAuth が `{ isAuthenticated: false }` を返すようモック。`/protected?tab=expense` にアクセス | /login にリダイレクトされ、location.state.from に元の URL（pathname + search = `/protected?tab=expense`）が保持される |

---

### 5. FE テストケース網羅性チェック

§9 テスト追跡用設計識別子の全コンポーネント・Hook に対するテストケースのカバー状況を確認する。

| 設計識別子 | テストケース ID 範囲 | カバー状況 |
|-----------|---------------------|-----------|
| `auth-login.md §LoginPage` | AUTH-FE-008〜013 | OK |
| `auth-login.md §LoginForm` | AUTH-FE-014〜020 | OK |
| `auth-login.md §FormAlert` | AUTH-FE-001〜003 | OK（共通コンポーネントとして定義） |
| `auth-login.md §SubmitButton` | AUTH-FE-004〜005 | OK（共通コンポーネントとして定義） |
| `auth-login.md §AuthNavLinks` | AUTH-FE-006〜007 | OK（共通コンポーネントとして定義） |
| `state-management.md §useLogin` | AUTH-FE-021〜024 | OK |
| `auth-signup.md §SignupPage` | AUTH-FE-025〜029 | OK |
| `auth-signup.md §SignupForm` | AUTH-FE-030〜039 | OK |
| `auth-signup.md §FormAlert` | AUTH-FE-001〜003 | OK（共通コンポーネントとして定義） |
| `auth-signup.md §SubmitButton` | AUTH-FE-004〜005 | OK（共通コンポーネントとして定義） |
| `auth-signup.md §AuthNavLinks` | AUTH-FE-006〜007 | OK（共通コンポーネントとして定義） |
| `state-management.md §useSignup` | AUTH-FE-040〜042 | OK |
| `auth-password-reset-request.md §PasswordResetRequestPage` | AUTH-FE-043〜047 | OK |
| `auth-password-reset-request.md §PasswordResetRequestForm` | AUTH-FE-048〜053 | OK |
| `auth-password-reset-request.md §PasswordResetRequestComplete` | AUTH-FE-054 | OK |
| `auth-password-reset-request.md §FormAlert` | AUTH-FE-001〜003 | OK（共通コンポーネントとして定義） |
| `auth-password-reset-request.md §SubmitButton` | AUTH-FE-004〜005 | OK（共通コンポーネントとして定義） |
| `auth-password-reset-request.md §AuthNavLinks` | AUTH-FE-006〜007, 007-A | OK（共通コンポーネントとして定義） |
| `state-management.md §useRequestPasswordReset` | AUTH-FE-055〜056 | OK |
| `auth-password-reset.md §PasswordResetPage` | AUTH-FE-057〜063 | OK |
| `auth-password-reset.md §PasswordResetForm` | AUTH-FE-064〜070 | OK |
| `auth-password-reset.md §PasswordResetComplete` | AUTH-FE-071 | OK |
| `auth-password-reset.md §PasswordResetTokenInvalid` | AUTH-FE-072 | OK |
| `auth-password-reset.md §FormAlert` | AUTH-FE-001〜003 | OK（共通コンポーネントとして定義） |
| `auth-password-reset.md §SubmitButton` | AUTH-FE-004〜005 | OK（共通コンポーネントとして定義） |
| `auth-password-reset.md §AuthNavLinks` | AUTH-FE-006〜007, 007-A | OK（共通コンポーネントとして定義） |
| `state-management.md §useExecutePasswordReset` | AUTH-FE-073〜076 | OK |
| `auth-login.md §PrivateRoute` | PRT-001〜003 | OK |
| 認証フォーム autoComplete 属性（issue #171） | AUTH-FE-077〜083 | OK |
