# 認証

- 担当: backend-developer, frontend-developer
- 依存: Step 9 完了（9-A）
- ブランチ: `step10/10-A-auth`
- 出力先: expense-saas/ 内のバックエンド・フロントエンドディレクトリ
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| API 契約 | deliverables/docs/50_detail_design/openapi.yaml | 認証エンドポイント（/auth/*） |
| DB スキーマ | deliverables/docs/50_detail_design/db_schema.md | users, tenants, tenant_memberships, refresh_tokens, password_reset_tokens |
| 認可設計 | deliverables/docs/50_detail_design/authz.md | RBAC・テナント分離 |
| セキュリティ設計 | deliverables/docs/50_detail_design/security.md | パスワードポリシー、トークン仕様、セキュリティヘッダー |
| 画面仕様（ログイン） | deliverables/docs/50_detail_design/screens/auth-login.md | ログイン画面 |
| 画面仕様（サインアップ） | deliverables/docs/50_detail_design/screens/auth-signup.md | サインアップ画面 |
| 画面仕様（パスワードリセット依頼） | deliverables/docs/50_detail_design/screens/auth-password-reset-request.md | パスワードリセット依頼画面 |
| 画面仕様（パスワードリセット） | deliverables/docs/50_detail_design/screens/auth-password-reset.md | パスワードリセット実行画面 |
| UI コンポーネント設計（認証系） | deliverables/docs/55_ui_component/screens/auth-*.md | コンポーネントツリー・Props 型 |
| テストケース（認証） | deliverables/docs/60_test/test_cases/auth.md | 全テストケース |
| BE テストコード | expense-saas/internal/domain/auth_test.go, internal/handler/auth_test.go, internal/service/auth_service_test.go, internal/service/auth_service_helper_test.go | 認証テスト |
| FE テストコード | expense-saas/frontend/src/pages/login/__tests__/*.test.tsx, pages/signup/__tests__/*.test.tsx, pages/password-reset/__tests__/*.test.tsx, pages/auth/__tests__/*.test.tsx, hooks/__tests__/useLogin.test.tsx, useSignup.test.tsx, useCurrentUser.test.tsx, useRequestPasswordReset.test.tsx, useExecutePasswordReset.test.tsx | 認証テスト |

## 責務

- 認証 API の機能実装（ユーザー登録・ログイン・トークンリフレッシュ・ログアウト・プロフィール取得・パスワードリセット）
- パスワードハッシュ（bcrypt）、JWT トークン発行・検証のロジック実装
- ログイン・サインアップ・パスワードリセット画面の機能実装
- トークン管理（アクセストークン・リフレッシュトークンの保持・自動リフレッシュ）
- ルーティングガード（未認証時のリダイレクト）
- 含めない: 認証以外の機能実装（レポート、ダッシュボード等）

## 完了条件

- test_cases/auth.md の全テストケースが通過している
- 登録 → ログイン → リフレッシュ → ログアウトの一連フローが API + UI で通る
- Step 9 で実装済みの認証関連テストコード（BE/FE）が全て PASS する
