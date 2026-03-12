# 実装手順書 — 経費精算SaaS

> 作成日: 2026-03-03
> 現状: project/ 配下はディレクトリ骨格のみ（explanation.md プレースホルダー）
> 目標: Phase 4 完了によるプロダクト完成

---

## 前提確認（作業開始前に一度だけ）

### ローカルツールチェーン確認

```bash
go version              # 1.24+ 推奨
golangci-lint --version
node --version           # 20+ 推奨
npm --version
docker --version
docker compose version
```

### 重要：apps/api の構成について

`project/apps/api/package.json` は骨格作成時の仮置き。
Go バックエンドは `go.mod` で管理するため、以下に差し替える：

- `apps/api/package.json` → 削除（後のステップで対応）
- `apps/api/go.mod` → 新規作成（Step 1 で対応）

---

## Phase 1: 基盤構築

### Step 1: Go バックエンド初期化

**作業場所**: `project/apps/api/`

やること：
1. `apps/api/package.json` を削除
2. `apps/api/go.mod` を作成（`go mod init`）
3. `apps/api/cmd/api/main.go` を作成（HTTP サーバーの最小起動確認）
4. `go build ./...` でコンパイル確認

主要依存パッケージ（go.mod に追加、フレームワーク固有は Step 3 で選定）：
- HTTP フレームワーク — Step 3 で選定（Echo / Chi 等）
- DB アクセス — Step 3 で選定（pgx / sqlc 等）
- `github.com/google/uuid` — UUID 生成
- `github.com/matthewhartstonge/argon2` — パスワードハッシュ (Argon2id)
- `github.com/golang-jwt/jwt/v5` — JWT (RS256)
- `log/slog` — 構造化ログ（Go 標準ライブラリ）
- `github.com/caarlos0/env/v11` — 環境変数/設定管理

確認チェックポイント：
- [ ] `go build ./...` が通る
- [ ] `go vet ./...` で警告ゼロ

---

### Step 2: フロントエンド初期化

**作業場所**: `project/apps/web/`

やること：
1. `apps/web/` を Vite + React + TypeScript で初期化
   ```
   npm create vite@latest . -- --template react-ts
   ```
2. 追加パッケージのインストール：
   - `react-router-dom` v7 — ルーティング
   - `@tanstack/react-query` — サーバー状態管理
   - `zustand` — UI状態管理
   - `react-hook-form` + `zod` — フォーム・バリデーション
   - `tailwindcss` + `shadcn/ui` — UI
   - `axios` or `ky` — HTTPクライアント
3. `tsconfig.json` の strict mode 確認（`"strict": true`）
4. `npm run dev` で起動確認

確認チェックポイント：
- [ ] `npm run dev` でローカル起動できる
- [ ] `npm run build` が通る
- [ ] `npm run lint` でエラーなし

---

### Step 3: Docker Compose 構築

**作業場所**: `project/docker/`

やること：
1. `docker/docker-compose.yml` を作成（以下サービスを定義）
   - `db`: PostgreSQL 16
   - `api`: Go バックエンド（開発時はホットリロード対応で air 使用）
   - `web`: フロントエンド（Vite dev server）
2. `docker/postgres/init.sql` — DB初期化スクリプト（RLS有効化など）
3. プロジェクトルートに `docker-compose.yml` のシンボリックリンクか実体を配置
4. `docker compose up -d db` で PostgreSQL のみ起動確認

確認チェックポイント：
- [ ] `docker compose up -d` でコンテナが全台起動する
- [ ] `docker compose logs api` でバックエンドが起動ログを出力

---

### Step 4: 環境変数・設定管理

**作業場所**: `project/`

やること：
1. `project/.env.example` を作成（コミット可、実際の値は入れない）
   ```
   DATABASE_URL=postgres://user:password@localhost:5432/expensesaas
   JWT_PRIVATE_KEY_PATH=./keys/private.pem
   JWT_PUBLIC_KEY_PATH=./keys/public.pem
   AWS_S3_BUCKET=
   AWS_REGION=ap-northeast-1
   ```
2. `internal/config/config.go` — 設定構造体（環境変数を型安全に読み込み）
3. RS256 用の鍵ペア生成スクリプトを `project/scripts/` に追加
4. `project/.gitignore` に `.env`, `keys/` を追加確認

確認チェックポイント：
- [ ] `.env` はコミットされない
- [ ] 起動時に設定が正しく読み込まれる

---

### Step 5: DB マイグレーション

**作業場所**: `project/database/`

やること：
1. `database/migrations/` 配下に SQL ファイルを順番に作成：
   - `0001_create_tenants.sql`
   - `0002_create_users.sql`
   - `0003_create_tenant_memberships.sql`
   - `0004_create_refresh_tokens.sql`
   - `0005_create_expense_reports.sql`
   - `0006_create_expense_items.sql`
   - `0007_create_attachments.sql`
   - `0008_create_approvals.sql`
   - `0009_create_invitations.sql`
   - `0010_create_notifications.sql`
   - `0011_create_audit_logs.sql`
   - `0012_create_indexes.sql`
   - `0013_setup_rls.sql` — Row Level Security ポリシー設定
2. マイグレーションツール（Step 3 で選定: golang-migrate / goose 等）で全マイグレーション実行
3. 各テーブルに `tenant_id` が存在することを確認

テーブル定義の根拠: TODO：右記は初期構想のため廃止予定：`PROJECT_SUMMARY.md` の「DB 設計（主要テーブル）」セクション

確認チェックポイント：
- [ ] マイグレーションが正常終了
- [ ] `psql` で全テーブルの存在確認
- [ ] RLS が各テーブルに設定されている

---

### Step 6: バックエンド アプリケーション構造

**作業場所**: `project/apps/api/`

やること：
1. ディレクトリ構造を整備：
   ```
   cmd/
   └── api/
       └── main.go
   internal/
   ├── config/
   │   └── config.go
   ├── errors/
   │   └── errors.go       — 共通エラー型 + JSON レスポンス変換
   ├── db/
   │   └── db.go            — DB接続プール初期化
   ├── middleware/
   │   ├── auth.go          — JWT検証
   │   └── tenant.go        — tenant_id 抽出・検証
   ├── domain/
   │   └── expense.go       — 状態遷移ロジック
   ├── repository/
   │   ├── user.go
   │   ├── expense_report.go
   │   └── ...
   └── handler/
       ├── auth.go
       ├── tenant.go
       ├── expense_report.go
       └── ...
   ```
2. `internal/errors/errors.go` で `AppError` 型を定義
   → 全エラーは `{"error": {"code": "...", "message": "...", "details": [...]}}` 形式

確認チェックポイント：
- [ ] `go build ./...` が通る
- [ ] エラー型が統一されている

---

### Step 7: 認証 API 実装

**作業場所**: `project/apps/api/internal/handler/auth.go`

実装するエンドポイント：
- `POST /api/v1/auth/signup` — テナント + Admin ユーザーを同時作成
- `POST /api/v1/auth/login` — JWT アクセストークン(15分) + リフレッシュトークン(7日) を返す
- `POST /api/v1/auth/refresh` — リフレッシュトークンを検証して新しいアクセストークンを発行
- `POST /api/v1/auth/logout` — リフレッシュトークンを無効化

実装の注意点：
- パスワードハッシュは Argon2id を使用
- JWT は RS256（非対称鍵）
- リフレッシュトークンは DB に `token_hash` として保存（平文保存禁止）
- `panic()` 禁止（`error` 型で返す）

確認チェックポイント：
- [ ] signup → login → refresh → logout の一連フローが通る
- [ ] 無効なトークンで 401 を返す
- [ ] 単体テスト: Argon2id のハッシュ/検証
- [ ] 統合テスト: 各エンドポイント

---

### Step 8: CI 基盤

**作業場所**: `project/.github/workflows/`

やること：
1. `ci.yml` を作成：
   - Go: `gofmt -l .`, `go vet ./...`, `golangci-lint run`, `go test ./...`
   - Frontend: `npm run lint`, `npm run build`, `npm test`
   - 依存関係監査: `govulncheck ./...`, `npm audit`
2. PR 時に自動実行されるよう設定

確認チェックポイント：
- [ ] GitHub に push したとき CI が通る
- [ ] PR テンプレート (`PULL_REQUEST_TEMPLATE.md`) が適切に設定されている

---

## Phase 2: コア機能

### Step 9: RBAC ミドルウェア

**作業場所**: `project/apps/api/internal/middleware/`

やること：
1. JWT から `tenant_id`, `user_id`, `role` を抽出するミドルウェア
2. ロール検証関数（`RequireRole(role ...Role)` のように使える形）
3. 全ハンドラで tenant_id を直接受け取る（ハンドラ内で SQL に直接書かない）

確認チェックポイント：
- [ ] Member が Approver 専用エンドポイントを叩くと 403 を返す
- [ ] 別テナントのリソースにアクセスすると 404 or 403 を返す
- [ ] RBAC テスト: 全ロール × 全エンドポイントのマトリクス

---

### Step 10: 経費レポート CRUD + 状態遷移

**作業場所**: `project/apps/api/internal/`

やること：
1. `domain/expense.go` に状態遷移ロジックを実装
   - `CanTransition(from Status, to Status) bool`
   - 不正遷移は `AppError` (InvalidTransition) を返す
2. リポジトリ層 (`repository/expense_report.go`) に全クエリを集約
   - 全クエリに `WHERE tenant_id = $1` を必ず含める
3. ハンドラ実装（`handler/expense_report.go`）

エンドポイント: TODO：右記は初期構想のため廃止予定：`PROJECT_SUMMARY.md` の「経費レポート」セクション参照

確認チェックポイント：
- [ ] `draft → approved` のような不正遷移で 422 を返す
- [ ] 状態遷移テスト（全パターン）
- [ ] テナント分離テスト（Tenant A が Tenant B のレポートを取得できない）

---

### Step 11: 経費明細 CRUD

**作業場所**: `project/apps/api/internal/handler/expense_item.go`

やること：
- 明細の追加・更新・削除（`draft` ステータスのレポートのみ操作可能）
- `total_amount` の自動更新（明細の合計を `expense_reports.total_amount` に反映）

確認チェックポイント：
- [ ] `submitted` 状態のレポートの明細を変更しようとすると 422 を返す

---

### Step 12: 領収書アップロード（S3）

**作業場所**: `project/apps/api/internal/handler/attachment.go`

やること：
1. S3 クライアント設定（`aws-sdk-go-v2` を使用）
2. アップロード処理：
   - MIME タイプ検証（JPEG / PNG / PDF のみ）
   - サイズ制限（5MB）
   - S3 キー: `tenants/{tenant_id}/users/{user_id}/receipts/{file_id}`
3. 署名付き URL 発行（取得時）

確認チェックポイント：
- [ ] 対応外 MIME タイプで 400 を返す
- [ ] 5MB 超のファイルで 400 を返す
- [ ] S3 パスに tenant_id が含まれている

---

### Step 13: 承認フロー API

**作業場所**: `project/apps/api/internal/handler/expense_report.go`

やること：
- `POST /api/v1/expense-reports/:id/submit` — Member のみ
- `POST /api/v1/expense-reports/:id/approve` — Approver のみ
- `POST /api/v1/expense-reports/:id/reject` — Approver のみ（理由必須）
- `POST /api/v1/expense-reports/:id/mark-paid` — Accounting のみ
- 各操作後に `approvals` テーブルへの記録
- 各操作後に `notifications` テーブルへの通知挿入

確認チェックポイント：
- [ ] 権限外ロールが操作すると 403 を返す
- [ ] 却下理由なしの reject で 422 を返す

---

### Step 14: フロントエンド — 認証・基本画面

**作業場所**: `project/apps/web/src/`

やること：
1. ディレクトリ構造を整備：
   ```
   src/
   ├── features/
   │   ├── auth/         — ログイン・サインアップ
   │   ├── dashboard/
   │   └── expenses/
   ├── components/       — 共有コンポーネント
   ├── hooks/
   ├── lib/
   │   ├── api.ts        — APIクライアント
   │   └── auth.ts       — JWT管理 (localStorage or httpOnly cookie 検討)
   └── routes/
   ```
2. ログイン画面 (`/login`)
3. ダッシュボード (`/`) — サマリー表示
4. 経費レポート一覧 (`/expense-reports`)
5. 経費レポート詳細・作成・編集画面

確認チェックポイント：
- [ ] ログイン → ダッシュボード → レポート作成 → 提出の一連フローが UI で完結
- [ ] アクセストークン期限切れ時にリフレッシュが自動実行される

---

## Phase 3: 運用機能

### Step 15: 監査ログ

**作業場所**: `project/apps/api/internal/`

やること：
1. `audit_logs` への書き込み関数（INSERT ONLY、更新・削除禁止）
2. 記録対象アクションに audit_log 挿入処理を追加：
   - 経費レポートの作成・提出・承認・却下・支払済み
   - ユーザー招待・ロール変更

---

### Step 16: 通知機能

**作業場所**: `project/apps/api/internal/handler/notification.go`

やること：
- `GET /api/v1/notifications` — 一覧取得（未読フィルタ対応）
- `PATCH /api/v1/notifications/:id/read` — 既読化
- フロントエンドのヘッダーに未読バッジ表示

---

### Step 17: CSV エクスポート

**作業場所**: `project/apps/api/internal/handler/export.go`

やること：
- `GET /api/v1/exports/expenses.csv` — Accounting / Admin のみ
- フィルタ条件: 期間・ステータス・カテゴリ・申請者
- `encoding/csv`（Go 標準ライブラリ）を使用

---

### Step 18: 招待フロー

**作業場所**: `project/apps/api/internal/handler/invitation.go`

やること：
1. `POST /api/v1/tenant/invitations` — 招待トークン生成（UUID v4、72時間有効）
2. メール送信（AWS SES or SendGrid）
3. 招待リンクからのアカウント作成フロー（`/invite?token=xxx`）
4. 使用済み・期限切れトークンの無効化

---

### Step 19: メンバー管理画面 + テスト追加

**作業場所**: `project/apps/web/src/features/settings/`

やること：
1. メンバー一覧 (`/settings/members`) — Admin のみ
2. ロール変更・メンバー削除
3. テスト追加：
   - テナント分離テスト（全リソース）
   - RBAC テスト（全ロール × 全エンドポイント）

---

## Phase 4: デプロイ・仕上げ

### Step 20: AWS インフラ構築

**作業場所**: `project/infra/`

やること：
1. Terraform or CDK でインフラ定義：
   - ECS Fargate（バックエンド）
   - RDS PostgreSQL
   - S3（領収書ストレージ + フロントエンドホスティング）
   - CloudFront
   - ALB
2. `project/infra/README.md` にデプロイ手順記載

---

### Step 21: GitHub Actions — デプロイパイプライン

**作業場所**: `project/.github/workflows/`

やること：
1. `deploy-staging.yml` — `main` ブランチ push で自動デプロイ
2. `deploy-production.yml` — Git tag で手動トリガー
3. ヘルスチェックエンドポイント `GET /health` をバックエンドに追加

---

### Step 22: ドキュメント整備

やること：
1. `project/docs/architecture.md` — アーキテクチャ図（Mermaid 形式推奨）
2. `project/docs/api.md` — OpenAPI (Swagger) 仕様書（Go の OpenAPI 生成ツールで自動生成も可、Step 3 で選定）
3. `project/docs/tech-stack.md` — 技術選定理由とトレードオフ
4. `project/README.md` — 英語で記述（概要・セットアップ・CI バッジ）
5. `references/decisions/` 配下に ADR を追加（主要な設計判断ごと）

---

## 実装順序の原則

```
Step 1-4（環境）
    ↓
Step 5（DB）     ← ここが全ての土台
    ↓
Step 6（構造）
    ↓
Step 7（認証）   ← 全 API の前提
    ↓
Step 8（CI）     ← 早めに入れると後が楽
    ↓
Step 9（RBAC）   ← コア機能の前提
    ↓
Step 10-13（バックエンド コア機能）
    ↓
Step 14（フロントエンド コア画面）
    ↓
Step 15-19（運用機能）
    ↓
Step 20-22（デプロイ・仕上げ）
```

---

## 各 Step の完了判断基準

| Phase | 完了の目安 |
|-------|-----------|
| Phase 1 | `docker compose up` で全サービス起動、認証 API が curl で叩ける |
| Phase 2 | 申請 → 承認 → 支払の一連フローが API + UI で通る |
| Phase 3 | 監査ログ・通知・CSV が動作、テナント分離 / RBAC テストが CI でグリーン |
| Phase 4 | staging URL が公開状態、CI バッジが README に表示されている |
