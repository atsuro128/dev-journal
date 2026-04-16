# ディレクトリ構成（実装レベル）

> このファイルは基盤構築（Step 8）完了時点のスナップショットである。コードが正本であり、構造変更時の更新は推奨だが必須ではない。

## 1. リポジトリルート

```
expense-saas/
├── cmd/                         # CLI エントリーポイント
├── internal/                    # アプリケーション内部パッケージ
├── db/                          # DB 関連定義
├── scripts/                     # 運用スクリプト
├── frontend/                    # フロントエンド（React + TypeScript）
├── .github/                     # GitHub 設定（CI/CD ワークフロー、PR テンプレート）
├── Dockerfile                   # 本番ビルド用 Dockerfile
├── docker-compose.yml           # ローカル開発環境定義
├── go.mod                       # Go モジュール定義
└── go.sum                       # Go 依存ロックファイル
```

---

## 2. バックエンド

### 2.1 cmd/

```
cmd/
├── server/
│   └── main.go                  # API サーバーエントリーポイント。ルーター構築・ミドルウェア登録・サーバー起動
└── seed/
    └── main.go                  # シードデータ投入 CLI。開発・テスト用の初期データを DB に挿入
```

### 2.2 internal/

```
internal/
├── config/                      # 環境変数・設定管理
│   └── config.go                #   環境変数からアプリケーション設定を読み込む構造体・関数
├── domain/                      # ドメイン層（エンティティ・値オブジェクト直置き）
│   ├── entity.go                #   User, Tenant, TenantMembership 等のエンティティ定義
│   ├── report.go                #   ExpenseReport 集約ルート。状態遷移メソッド・不変条件検証
│   ├── item.go                  #   ExpenseItem エンティティ。金額バリデーション
│   ├── auth.go                  #   認証関連（RefreshToken, PasswordResetToken 等）
│   ├── actor.go                 #   Actor（認可主体）。ロール判定・所有権チェック
│   ├── enum.go                  #   ReportStatus, Role, Category 等の列挙型
│   ├── errors.go                #   ドメインエラー定義
│   └── repository.go            #   リポジトリインターフェース定義
├── service/                     # アプリケーションサービス層
│   ├── auth_service.go          #   認証ユースケース（ログイン・サインアップ・トークンリフレッシュ等）
│   ├── report_service.go        #   レポート CRUD ユースケース
│   ├── item_service.go          #   明細 CRUD ユースケース
│   ├── attachment_service.go    #   添付ファイルユースケース（アップロード・ダウンロード・削除）
│   ├── workflow_service.go      #   ワークフローユースケース（承認・却下・支払完了）
│   ├── dashboard_service.go     #   ダッシュボード集計ユースケース
│   ├── category_service.go      #   カテゴリ取得ユースケース
│   ├── tenant_service.go        #   テナント情報取得ユースケース
│   ├── authorizer.go            #   認可ロジック（所有権・ロール判定の共通実装）
│   ├── dto.go                   #   レイヤー間転送オブジェクト（DTO）定義
│   ├── interfaces.go            #   サービスインターフェース定義
│   ├── params.go                #   サービスメソッドパラメータ型
│   └── not_implemented.go       #   未実装サービスメソッドのスタブ
├── handler/                     # HTTP ハンドラ層
│   ├── auth.go                  #   認証エンドポイント（login, signup, refresh, logout, me, password-reset）
│   ├── report.go                #   レポートエンドポイント（CRUD + submit）
│   ├── item.go                  #   明細エンドポイント（CRUD）
│   ├── attachment.go            #   添付ファイルエンドポイント（upload, download, delete, list）
│   ├── workflow.go              #   ワークフローエンドポイント（approve, reject, pay, pending, payable）
│   ├── dashboard.go             #   ダッシュボードエンドポイント
│   ├── category.go              #   カテゴリエンドポイント
│   ├── tenant.go                #   テナントエンドポイント
│   ├── health.go                #   ヘルスチェックエンドポイント（/health）
│   └── helpers.go               #   ハンドラ共通ヘルパー（リクエストパース・レスポンス生成）
├── middleware/                  # HTTP ミドルウェア
│   ├── cors.go                  #   CORS 制御
│   ├── security_headers.go      #   セキュリティヘッダー（HSTS, X-Content-Type-Options 等）
│   ├── request_id.go            #   リクエスト ID 生成・コンテキスト設定
│   ├── logger.go                #   構造化ログ出力
│   ├── ratelimit.go             #   レート制限
│   ├── auth.go                  #   JWT 検証
│   ├── tenant.go                #   テナントコンテキスト設定 + RLS（SET LOCAL app.current_tenant）
│   ├── rbac.go                  #   ロール検証（RBAC）
│   ├── context.go               #   コンテキストヘルパー（user_id, tenant_id, role の取得）
│   └── response.go              #   レスポンスヘルパー
├── repository/postgres/         # リポジトリ実装（PostgreSQL）
│   ├── sqlcgen/                 #   sqlc 自動生成コード（DO NOT EDIT）
│   │   ├── db.go                #     DBTX インターフェース・Queries 構造体
│   │   ├── models.go            #     sqlc モデル定義
│   │   ├── expense_reports.sql.go
│   │   ├── expense_items.sql.go
│   │   ├── attachments.sql.go
│   │   ├── users.sql.go
│   │   ├── tenants.sql.go
│   │   ├── tenant_memberships.sql.go
│   │   ├── categories.sql.go
│   │   ├── refresh_tokens.sql.go
│   │   └── password_reset_tokens.sql.go
│   ├── db.go                    #   DB 接続ヘルパー（pgxpool ラッパー）
│   ├── convert.go               #   sqlc モデル <-> ドメインモデル変換
│   ├── report_repo.go           #   ExpenseReport リポジトリ実装
│   ├── item_repo.go             #   ExpenseItem リポジトリ実装
│   ├── attachment_repo.go       #   Attachment リポジトリ実装
│   ├── user_repo.go             #   User リポジトリ実装
│   ├── tenant_repo.go           #   Tenant リポジトリ実装
│   ├── membership_repo.go       #   TenantMembership リポジトリ実装
│   ├── category_repo.go         #   Category リポジトリ実装
│   ├── refresh_token_repo.go    #   RefreshToken リポジトリ実装
│   └── password_reset_repo.go   #   PasswordResetToken リポジトリ実装
├── pkg/                         # 内部共有パッケージ
│   ├── jwt/                     #   JWT 発行・検証
│   │   └── jwt.go
│   └── s3/                      #   S3 操作（署名付き URL 発行等）
│       ├── client.go            #     AWS S3 クライアントラッパー
│       └── inmemory.go          #     テスト用インメモリ S3 実装
├── seed/                        # シードデータ
│   ├── seed.go                  #   開発用テストデータ挿入ロジック
│   └── testdata/                #   テスト用添付ファイル（receipt_sample.jpg 等）
└── testutil/                    # テストユーティリティ
    ├── db.go                    #   テスト用 DB 接続・クリーンアップ
    ├── migrate.go               #   テスト用マイグレーション実行
    ├── fixture.go               #   テスト用フィクスチャ生成
    ├── jwt.go                   #   テスト用 JWT トークン生成
    ├── http.go                  #   テスト用 HTTP リクエストヘルパー
    └── assert.go                #   カスタムアサーション関数
```

### 2.3 db/

```
db/
├── migrations/                  # golang-migrate マイグレーションファイル
│   ├── 000001_create_extensions.{up,down}.sql
│   ├── 000002_create_roles.{up,down}.sql
│   ├── 000003_create_tenants.{up,down}.sql
│   ├── 000004_create_users.{up,down}.sql
│   ├── 000005_create_tenant_memberships.{up,down}.sql
│   ├── 000006_create_categories.{up,down}.sql
│   ├── 000007_create_expense_reports.{up,down}.sql
│   ├── 000008_create_expense_items.{up,down}.sql
│   ├── 000009_create_attachments.{up,down}.sql
│   ├── 000010_create_refresh_tokens.{up,down}.sql
│   └── 000011_create_password_reset_tokens.{up,down}.sql
├── queries/                     # sqlc クエリ定義（.sql）
│   ├── expense_reports.sql
│   ├── expense_items.sql
│   ├── attachments.sql
│   ├── users.sql
│   ├── tenants.sql
│   ├── tenant_memberships.sql
│   ├── categories.sql
│   ├── refresh_tokens.sql
│   └── password_reset_tokens.sql
└── sqlc.yaml                    # sqlc 設定ファイル
```

### 2.4 scripts/

```
scripts/
├── generate-keys.sh             # JWT 署名用 RSA 鍵ペア生成
├── init-db.sh                   # 開発用 DB 初期化
└── init-db-test.sh              # テスト用 DB 初期化
```

---

## 3. フロントエンド

### 3.1 ルート

```
frontend/
├── src/                         # ソースコード（後述）
├── index.html                   # SPA エントリー HTML
├── vite.config.ts               # Vite ビルド設定
├── vitest.config.ts             # Vitest テスト設定
├── eslint.config.js             # ESLint 設定
├── .prettierrc.json             # Prettier 設定
├── theme.ts                     # MUI テーマ設定
├── Dockerfile.dev               # 開発用 Dockerfile
├── .dockerignore
├── tsconfig.json                # TypeScript コンパイラ設定
└── package.json                 # 依存定義・npm スクリプト
```

### 3.2 src/

```
src/
├── main.tsx                     # エントリーポイント。React アプリ起動・Provider 設定
├── App.tsx                      # ルーティング定義（React Router）
├── vite-env.d.ts                # Vite 環境型定義
├── api/                         # API クライアント
│   ├── client.ts                #   fetch ラッパー（JWT 自動付与・401 リトライ）
│   ├── reports.ts               #   レポート関連 API 関数
│   ├── types.ts                 #   API レスポンス型定義
│   └── adminTypes.ts            #   管理者向け API 型定義
├── components/                  # 共有 UI コンポーネント
│   ├── ui/                      #   汎用 UI（SubmitButton, ConfirmDialog, StatusChip, PageTitle 等）
│   ├── layout/                  #   レイアウト（AppHeader, AppSidebar, AppLayout, AuthLayout）
│   ├── report/                  #   経費レポート関連（ReportForm, ReportFormActions, ReportPeriodField）
│   ├── dashboard/               #   ダッシュボード関連（CountCard, MonthlySummaryTable 等）
│   └── auth/                    #   認証関連（PrivateRoute）
├── contexts/                    # React Context
│   └── SnackbarContext.ts       #   通知（Snackbar）用コンテキスト
├── hooks/                       # カスタムフック
│   ├── useAuth.ts               #   認証状態管理
│   ├── useLogin.ts              #   ログインミューテーション
│   ├── useLogout.ts             #   ログアウトミューテーション
│   ├── useSignup.ts             #   サインアップミューテーション
│   ├── useCurrentUser.ts        #   現在のユーザー情報取得
│   ├── useReports.ts            #   レポート一覧取得（自分）
│   ├── useAllReports.ts         #   全レポート一覧取得（Admin/Accounting）
│   ├── useDashboard.ts          #   ダッシュボードデータ取得
│   ├── useTenant.ts             #   テナント情報取得
│   ├── useTenantMembers.ts      #   テナントメンバー一覧取得
│   ├── useCategories.ts         #   カテゴリ一覧取得
│   ├── useItems.ts              #   明細操作
│   ├── useAttachments.ts        #   添付ファイル一覧取得
│   ├── useAttachmentDownload.ts #   添付ファイルダウンロード
│   ├── useUploadAttachment.ts   #   添付ファイルアップロード
│   ├── useDeleteAttachment.ts   #   添付ファイル削除
│   ├── useApproveReport.ts      #   レポート承認ミューテーション
│   ├── useRejectReport.ts       #   レポート却下ミューテーション
│   ├── useMarkAsPaid.ts         #   支払完了ミューテーション
│   ├── useSnackbar.ts           #   Snackbar 操作フック
│   ├── useRequestPasswordReset.ts  # パスワードリセット要求
│   └── useExecutePasswordReset.ts  # パスワードリセット実行
├── lib/                         # ユーティリティ関数
│   ├── constants.ts             #   定数定義
│   └── format.ts                #   日付・金額フォーマット
├── pages/                       # ページコンポーネント（ルーティング単位）
│   ├── login/                   #   ログイン（LoginPage, LoginForm, loginSchema）
│   ├── signup/                  #   サインアップ（SignupPage, SignupForm, signupSchema）
│   ├── password-reset/          #   パスワードリセット（RequestPage, ResetPage, Complete 等）
│   ├── auth/                    #   認証共通（AuthNavLinks）
│   ├── dashboard/               #   ダッシュボード（DashboardPage）
│   ├── reports/                 #   経費レポート（List/Create/Edit/Detail ページ、明細・添付コンポーネント）
│   ├── admin/                   #   管理者（TenantPage, AllReportsPage）
│   ├── workflow/                #   ワークフロー（ApprovalListPage, PaymentListPage）
│   └── error/                   #   エラー（NotFoundPage）
├── stores/                      # Zustand ストア
│   └── auth.ts                  #   認証状態（トークン保持・ユーザー情報キャッシュ）
└── test/                        # テストセットアップ
    ├── setup.ts                 #   Vitest グローバルセットアップ
    ├── renderWithProviders.tsx   #   テスト用 Provider ラッパー
    └── App.test.tsx             #   アプリケーションレベルテスト
```

テストを持つディレクトリ（`pages/*`, `components/*`, `hooks`, `api`, `stores`, `lib`）には `__tests__/` サブディレクトリを配置し、ユニットテストを格納する（コロケーションパターン）。

---

## 4. CI/CD

```
.github/
├── workflows/
│   ├── ci.yml                   # PR パイプライン（lint -> test -> build）
│   └── deploy.yml               # master マージパイプライン（lint -> test -> build -> deploy）
└── PULL_REQUEST_TEMPLATE.md     # PR テンプレート
```
