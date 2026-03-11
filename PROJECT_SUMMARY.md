# マルチテナント型 経費精算SaaS（ポートフォリオ用）

> 本ドキュメントは、経費精算SaaSプロジェクトのスコープ・設計方針を定義し、転職活動で「実務レベル」を証明するための成果物チェックリストを兼ねる。

## 目的

外資・SaaS企業などの高年収エンジニア転職に通用する、**実務レベルのポートフォリオ**を構築する。

### 転職先に評価されるポイント（本プロジェクトで証明するもの）

| 評価ポイント | 本プロジェクトでの対応 |
|-------------|------------------------|
| マルチテナント設計の理解 | Tenant 単位のデータ分離、全クエリでの `tenant_id` 必須化 |
| RBAC（権限設計） | Admin / Approver / Member / Accounting の4ロールと API 制御 |
| 実務的な業務フロー実装 | 経費申請〜承認〜支払の状態遷移、却下フロー |
| クラウド運用経験 | AWS デプロイ、CI/CD、運用・監視方針の明文化 |
| 英語ドキュメント対応 | 英語 README、API ドキュメント（英語） |
| テスト戦略 | テナント分離テスト、RBAC テスト、状態遷移テストの網羅 |
| 技術選定の合理性 | 各技術の採用理由を明文化し、設計判断を説明できる |

---

## サービス概要

企業向け経費精算SaaS。
複数企業（テナント）が1つのアプリを利用しつつ、**データは企業ごとに完全分離**される。

---

## テナントモデル

- **1会社 = 1テナント = 1 Tenant**（本ドキュメントでは `Tenant` を統一用語とする）
- 全データは `tenant_id` で分離
- 他テナントのデータは取得不可（ミドルウェア・リポジトリ層で保証）
- DB 上のテーブル名は `tenant` を用い、カラムは `tenant_id` で統一

---

## 技術スタック・選定理由

| レイヤー | 技術 | 選定理由 |
|----------|------|----------|
| Backend | Go | シンプルな言語仕様による高い生産性。SaaS 企業での採用率の高さ、充実した標準ライブラリ、コンパイル速度の速さが利点 |
| Frontend | React (TypeScript) | コンポーネント設計の柔軟性、エコシステムの成熟度、TypeScript による型安全なフロント開発 |
| DB | PostgreSQL | JSONB 型による柔軟なデータ格納、Row Level Security (RLS) によるテナント分離の二重保証が可能、実務での採用実績の多さ |
| DB Access | Step 3 で選定（sqlc / pgx 等） | 型安全なクエリ生成またはシンプルなDBドライバ |
| Infra | AWS (ECS Fargate / RDS / S3 / CloudFront) | コンテナベースのデプロイで運用負荷を軽減、マネージドサービス活用 |
| CI/CD | GitHub Actions | GitHub との統合、ワークフロー定義の容易さ |
| 認証 | JWT (RS256) | ステートレスな認証、マイクロサービス拡張時にも対応可能 |
| ログ | tracing + JSON 形式出力 | 構造化ログにより CloudWatch Logs での検索・アラート設定が容易 |

---

## コア機能（MVP）

### 1. 認証・ユーザー管理

- サインアップ / ログイン（メールアドレス + パスワード）
- JWT 認証（アクセストークン: 15分、リフレッシュトークン: 7日）
- リフレッシュトークンは DB に保存し、明示的な無効化（ログアウト）に対応
- パスワードハッシュ: Argon2id
- ユーザー招待（同一 Tenant 内、ロール付与）

#### 招待フロー詳細

1. Admin がメールアドレスとロールを指定して招待を作成
2. 招待トークン（UUID v4）を生成し、`invitations` テーブルに保存（有効期限: 72時間）
3. 招待メールを送信（SendGrid or AWS SES）
4. 招待リンクからアクセス → アカウント作成 → 自動的に該当 Tenant・ロールに紐づけ
5. 使用済み or 期限切れトークンは無効化

### 2. RBAC（権限）

| ロール | 主な権限 |
|--------|----------|
| Admin | 会社設定、メンバー管理、全データ閲覧・操作、招待の発行 |
| Approver | 経費承認・却下、自担当分の閲覧 |
| Member | 経費申請・編集（自分のみ）、状態確認 |
| Accounting | 経費一覧・エクスポート、支払済み更新 |

- 全 API はロールに応じたアクセス制御（権限不足時は `403 Forbidden`）
- リソース所有者チェック: Member は自分の経費のみ操作可能
- ミドルウェアで `tenant_id` + `role` を検証し、ハンドラに到達する前にガード

### 3. 経費申請

- 経費レポート作成（タイトル・期間を指定）
- 明細追加（日付・金額・摘要・カテゴリ）
- 領収書アップロード（S3、パス: `tenants/{tenant_id}/users/{user_id}/receipts/{file_id}`）
  - ファイルタイプ制限: JPEG, PNG, PDF のみ
  - サイズ制限: 5MB/ファイル
- 下書き保存 → 提出（状態: `draft` → `submitted`）

#### 経費カテゴリ（マスタ）

| カテゴリ | 説明 |
|---------|------|
| transportation | 交通費 |
| accommodation | 宿泊費 |
| food | 飲食費 |
| supplies | 消耗品費 |
| communication | 通信費 |
| other | その他 |

※ 将来的にはテナントごとのカスタムカテゴリに拡張可能な設計とする。

### 4. 承認フロー

状態遷移：

```
draft → submitted → approved → paid
                  → rejected
```

- `draft` → `submitted`: Member が提出（Approver に通知）
- `submitted` → `approved`: Approver が承認（Accounting に通知）
- `submitted` → `rejected`: Approver が却下（申請者に通知、却下理由必須）
- `approved` → `paid`: Accounting が支払処理完了を記録
- **不正な遷移（例: `draft` → `approved`）はバックエンドで厳密に拒否**
- 却下後の再申請: 新規レポートとして作成（元レポートへの参照を保持）

### 5. 通知

| イベント | 通知先 | 方法 |
|---------|--------|------|
| 経費提出 | Approver | アプリ内通知 |
| 承認 | 申請者, Accounting | アプリ内通知 |
| 却下 | 申請者 | アプリ内通知 |
| 支払完了 | 申請者 | アプリ内通知 |
| 招待 | 被招待者 | メール |

※ MVP ではアプリ内通知のみ。メール通知は Phase 2 で追加。

### 6. 監査ログ

誰が・いつ・何を変更したかを記録する。

- **記録対象**: 経費レポートの作成・更新・提出・承認・却下・支払済み更新、ユーザー招待、ロール変更
- **保存項目**: `tenant_id`, `user_id`, `action`, `resource_type`, `resource_id`, `old_value` (JSONB), `new_value` (JSONB), `ip_address`, `user_agent`, `created_at`
- **保存方針**: 監査ログは更新・削除不可（INSERT ONLY）
- **保持期間**: 1年（将来的にはテナント設定で変更可能）

### 7. データ出力

- 経費一覧の CSV エクスポート（自 Tenant 内のみ、権限は Accounting / Admin）
- フィルタ条件: 期間、ステータス、カテゴリ、申請者
- 出力項目: レポートID、申請者名、日付、金額、カテゴリ、ステータス、承認者、承認日

---

## API 設計方針

### 共通規約

- RESTful 設計、リソース名は複数形（`/expense-reports`, `/users`）
- レスポンス形式: JSON
- 日時: ISO 8601（UTC）
- ページネーション: カーソルベース（`?cursor=xxx&limit=20`）
- API バージョニング: URL パス方式（`/api/v1/...`）

### エラーレスポンス形式

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Amount must be greater than 0",
    "details": [
      { "field": "amount", "message": "must be greater than 0" }
    ]
  }
}
```

### 主要エンドポイント

#### 認証

| Method | Path | 説明 | 権限 |
|--------|------|------|------|
| POST | `/api/v1/auth/signup` | サインアップ | Public |
| POST | `/api/v1/auth/login` | ログイン | Public |
| POST | `/api/v1/auth/refresh` | トークンリフレッシュ | Public |
| POST | `/api/v1/auth/logout` | ログアウト | Authenticated |

#### テナント・ユーザー

| Method | Path | 説明 | 権限 |
|--------|------|------|------|
| GET | `/api/v1/tenant` | 自テナント情報取得 | Authenticated |
| PATCH | `/api/v1/tenant` | テナント情報更新 | Admin |
| GET | `/api/v1/tenant/members` | メンバー一覧 | Admin |
| POST | `/api/v1/tenant/invitations` | 招待作成 | Admin |
| PATCH | `/api/v1/tenant/members/:userId/role` | ロール変更 | Admin |
| DELETE | `/api/v1/tenant/members/:userId` | メンバー削除 | Admin |

#### 経費レポート

| Method | Path | 説明 | 権限 |
|--------|------|------|------|
| POST | `/api/v1/expense-reports` | レポート作成 | Member+ |
| GET | `/api/v1/expense-reports` | レポート一覧 | 権限に応じたフィルタ |
| GET | `/api/v1/expense-reports/:id` | レポート詳細 | 所有者 or Approver+ |
| PATCH | `/api/v1/expense-reports/:id` | レポート更新 | 所有者（draft のみ） |
| POST | `/api/v1/expense-reports/:id/submit` | 提出 | 所有者 |
| POST | `/api/v1/expense-reports/:id/approve` | 承認 | Approver |
| POST | `/api/v1/expense-reports/:id/reject` | 却下 | Approver |
| POST | `/api/v1/expense-reports/:id/mark-paid` | 支払済み | Accounting |

#### 経費明細

| Method | Path | 説明 | 権限 |
|--------|------|------|------|
| POST | `/api/v1/expense-reports/:reportId/items` | 明細追加 | 所有者（draft のみ） |
| PATCH | `/api/v1/expense-reports/:reportId/items/:id` | 明細更新 | 所有者（draft のみ） |
| DELETE | `/api/v1/expense-reports/:reportId/items/:id` | 明細削除 | 所有者（draft のみ） |

#### 添付ファイル

| Method | Path | 説明 | 権限 |
|--------|------|------|------|
| POST | `/api/v1/expense-reports/:reportId/items/:itemId/attachments` | アップロード | 所有者（draft のみ） |
| GET | `/api/v1/attachments/:id/url` | 署名付きURL取得 | 所有者 or Approver+ |
| DELETE | `/api/v1/attachments/:id` | 削除 | 所有者（draft のみ） |

#### エクスポート

| Method | Path | 説明 | 権限 |
|--------|------|------|------|
| GET | `/api/v1/exports/expenses.csv` | CSV エクスポート | Accounting / Admin |

#### 通知

| Method | Path | 説明 | 権限 |
|--------|------|------|------|
| GET | `/api/v1/notifications` | 通知一覧 | Authenticated |
| PATCH | `/api/v1/notifications/:id/read` | 既読にする | 所有者 |

---

## DB 設計（主要テーブル）

### tenants

| カラム | 型 | 説明 |
|--------|------|------|
| id | UUID (PK) | テナントID |
| name | VARCHAR(255) | 会社名 |
| created_at | TIMESTAMPTZ | 作成日時 |
| updated_at | TIMESTAMPTZ | 更新日時 |

### users

| カラム | 型 | 説明 |
|--------|------|------|
| id | UUID (PK) | ユーザーID |
| email | VARCHAR(255) UNIQUE | メールアドレス |
| password_hash | VARCHAR(255) | パスワードハッシュ (Argon2id) |
| display_name | VARCHAR(100) | 表示名 |
| created_at | TIMESTAMPTZ | 作成日時 |
| updated_at | TIMESTAMPTZ | 更新日時 |

### tenant_memberships

| カラム | 型 | 説明 |
|--------|------|------|
| id | UUID (PK) | |
| tenant_id | UUID (FK → tenants) | テナントID |
| user_id | UUID (FK → users) | ユーザーID |
| role | VARCHAR(20) | admin / approver / member / accounting |
| created_at | TIMESTAMPTZ | |
| UNIQUE | (tenant_id, user_id) | |

### expense_reports

| カラム | 型 | 説明 |
|--------|------|------|
| id | UUID (PK) | |
| tenant_id | UUID (FK → tenants) | テナントID |
| user_id | UUID (FK → users) | 申請者 |
| title | VARCHAR(255) | レポートタイトル |
| status | VARCHAR(20) | draft / submitted / approved / rejected / paid |
| period_start | DATE | 対象期間開始 |
| period_end | DATE | 対象期間終了 |
| total_amount | INTEGER | 合計金額（明細から算出・円単位・整数） |
| rejection_reason | TEXT | 却下理由（rejected 時のみ） |
| submitted_at | TIMESTAMPTZ | 提出日時 |
| approved_at | TIMESTAMPTZ | 承認日時 |
| paid_at | TIMESTAMPTZ | 支払日時 |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | |

### expense_items

| カラム | 型 | 説明 |
|--------|------|------|
| id | UUID (PK) | |
| tenant_id | UUID (FK → tenants) | テナントID |
| report_id | UUID (FK → expense_reports) | レポートID |
| date | DATE | 発生日 |
| amount | INTEGER | 金額（円単位・整数） |
| category | VARCHAR(50) | カテゴリ |
| description | TEXT | 摘要 |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | |

### attachments

| カラム | 型 | 説明 |
|--------|------|------|
| id | UUID (PK) | |
| tenant_id | UUID (FK → tenants) | テナントID |
| item_id | UUID (FK → expense_items) | 明細ID |
| file_name | VARCHAR(255) | 元ファイル名 |
| file_type | VARCHAR(50) | MIME タイプ |
| file_size | INTEGER | バイト数 |
| storage_key | VARCHAR(500) | S3 キー |
| created_at | TIMESTAMPTZ | |

### approvals

| カラム | 型 | 説明 |
|--------|------|------|
| id | UUID (PK) | |
| tenant_id | UUID (FK → tenants) | テナントID |
| report_id | UUID (FK → expense_reports) | レポートID |
| approver_id | UUID (FK → users) | 承認者 |
| action | VARCHAR(20) | approved / rejected |
| comment | TEXT | コメント・却下理由 |
| created_at | TIMESTAMPTZ | |

### invitations

| カラム | 型 | 説明 |
|--------|------|------|
| id | UUID (PK) | |
| tenant_id | UUID (FK → tenants) | テナントID |
| email | VARCHAR(255) | 招待先メール |
| role | VARCHAR(20) | 付与するロール |
| token | UUID UNIQUE | 招待トークン |
| invited_by | UUID (FK → users) | 招待者 |
| expires_at | TIMESTAMPTZ | 有効期限 |
| used_at | TIMESTAMPTZ | 使用日時（NULL = 未使用） |
| created_at | TIMESTAMPTZ | |

### notifications

| カラム | 型 | 説明 |
|--------|------|------|
| id | UUID (PK) | |
| tenant_id | UUID (FK → tenants) | テナントID |
| user_id | UUID (FK → users) | 通知先 |
| type | VARCHAR(50) | イベント種別 |
| title | VARCHAR(255) | 通知タイトル |
| body | TEXT | 通知本文 |
| resource_type | VARCHAR(50) | 関連リソース種別 |
| resource_id | UUID | 関連リソースID |
| is_read | BOOLEAN DEFAULT false | 既読フラグ |
| created_at | TIMESTAMPTZ | |

### refresh_tokens

| カラム | 型 | 説明 |
|--------|------|------|
| id | UUID (PK) | |
| user_id | UUID (FK → users) | ユーザーID |
| token_hash | VARCHAR(255) | トークンハッシュ |
| expires_at | TIMESTAMPTZ | 有効期限 |
| revoked_at | TIMESTAMPTZ | 無効化日時 |
| created_at | TIMESTAMPTZ | |

### audit_logs

| カラム | 型 | 説明 |
|--------|------|------|
| id | UUID (PK) | |
| tenant_id | UUID (FK → tenants) | テナントID |
| user_id | UUID (FK → users) | 操作者 |
| action | VARCHAR(50) | 操作種別 |
| resource_type | VARCHAR(50) | リソース種別 |
| resource_id | UUID | リソースID |
| old_value | JSONB | 変更前の値 |
| new_value | JSONB | 変更後の値 |
| ip_address | INET | IPアドレス |
| user_agent | TEXT | User-Agent |
| created_at | TIMESTAMPTZ | |

**インデックス方針**:
- 全テーブルの `tenant_id` にインデックス
- `expense_reports`: `(tenant_id, status)`, `(tenant_id, user_id)` に複合インデックス
- `audit_logs`: `(tenant_id, created_at)` に複合インデックス
- `users.email` にユニークインデックス

**テナント分離の二重保証**:
1. アプリケーション層: 全クエリに `WHERE tenant_id = $1` を付与（リポジトリ層で強制）
2. DB 層: PostgreSQL Row Level Security (RLS) を設定し、万が一アプリ層を迂回してもデータ漏洩を防止

---

## テスト戦略

### テストレベル

| レベル | ツール | 対象 | カバレッジ目標 |
|--------|--------|------|---------------|
| 単体テスト | Go 標準テスト | ドメインロジック（状態遷移、金額計算、バリデーション） | 90%+ |
| 統合テスト | Go + テスト用DB | API エンドポイント、認証・認可、DB 操作 | 主要パス網羅 |
| E2E テスト | Playwright | ユーザーシナリオ（申請〜承認〜支払の一連フロー） | 主要フロー網羅 |

### 重点テスト項目

1. **テナント分離テスト**: Tenant A のユーザーが Tenant B のデータを取得・更新・削除できないことを全リソースで検証
2. **RBAC テスト**: 各ロールが許可されていない操作で `403` を返すことを全エンドポイントで検証
3. **状態遷移テスト**: 不正な遷移（例: `draft` → `approved`）が拒否されることを検証
4. **認証テスト**: 期限切れトークン、無効トークン、リフレッシュフロー
5. **境界値テスト**: 金額の上限・下限、ファイルサイズ制限

---

## 非機能要件

| 項目 | 目標 |
|------|------|
| レスポンスタイム | API 95%ile < 500ms（一覧取得を含む） |
| 可用性 | 99.5%（月間ダウンタイム約3.6時間） |
| 同時テナント数 | 100テナント（MVP） |
| データ保持 | 監査ログ: 1年、その他: 無期限 |
| API レート制限 | 認証済み: 100 req/min/user、未認証: 20 req/min/IP |
| ファイルストレージ | テナントあたり 10GB（MVP） |

---

## フロントエンド設計方針

| 項目 | 選択 | 理由 |
|------|------|------|
| 言語 | TypeScript | 型安全性 |
| フレームワーク | React 19 + Vite | 高速なビルド、HMR |
| ルーティング | React Router v7 | 標準的、Nested Routes 対応 |
| 状態管理 | TanStack Query (サーバー状態) + Zustand (UI状態) | サーバーキャッシュとUI状態の分離 |
| UIライブラリ | shadcn/ui + Tailwind CSS | カスタマイズ性、バンドルサイズの最適化 |
| フォーム | React Hook Form + Zod | バリデーションスキーマの共有 |
| テスト | Vitest + Testing Library | React コンポーネントテスト |

### 主要画面

| 画面 | パス | 説明 |
|------|------|------|
| ログイン | `/login` | メール + パスワード |
| ダッシュボード | `/` | 経費サマリー、直近の申請状況 |
| 経費レポート一覧 | `/expense-reports` | フィルタ・ソート・ページネーション |
| 経費レポート詳細 | `/expense-reports/:id` | 明細一覧、領収書プレビュー、承認操作 |
| 経費レポート作成/編集 | `/expense-reports/new`, `/expense-reports/:id/edit` | フォーム、ファイルアップロード |
| メンバー管理 | `/settings/members` | 一覧、招待、ロール変更（Admin のみ） |
| テナント設定 | `/settings/tenant` | 会社情報編集（Admin のみ） |
| 通知 | `/notifications` | 通知一覧、既読管理 |

---

## インフラ構成

```
                         ┌─────────────┐
                         │ CloudFront  │
                         │  (CDN)      │
                         └──────┬──────┘
                                │
                    ┌───────────┴───────────┐
                    │                       │
             ┌──────┴──────┐        ┌──────┴──────┐
             │  S3 Bucket  │        │     ALB     │
             │ (Frontend)  │        │             │
             └─────────────┘        └──────┬──────┘
                                           │
                                    ┌──────┴──────┐
                                    │ ECS Fargate │
                                    │  (Backend)  │
                                    └──────┬──────┘
                                           │
                              ┌────────────┼────────────┐
                              │            │            │
                       ┌──────┴──┐  ┌──────┴──┐  ┌─────┴────┐
                       │   RDS   │  │   S3    │  │CloudWatch│
                       │ (Postgres)│ │(Receipts)│ │ (Logs)   │
                       └─────────┘  └─────────┘  └──────────┘
```

### 環境

| 環境 | 用途 | デプロイ |
|------|------|---------|
| dev | ローカル開発 | Docker Compose |
| staging | 結合テスト・レビュー | `main` ブランチ push で自動デプロイ |
| production | 本番 | Git tag で手動トリガー |

---

## 開発フェーズ

### Phase 1: 基盤構築
- プロジェクト初期設定（Go + React + Docker Compose）
- DB マイグレーション（全テーブル作成）
- 認証 API（signup / login / refresh / logout）
- テナント作成・基本 CRUD
- CI 基盤（lint, test, build）

### Phase 2: コア機能
- RBAC ミドルウェア実装
- 経費レポート CRUD + 状態遷移
- 経費明細 CRUD
- 領収書アップロード（S3）
- 承認フロー API
- フロントエンド: ログイン、ダッシュボード、経費レポート画面

### Phase 3: 運用機能
- 監査ログ
- 通知機能（アプリ内）
- CSV エクスポート
- 招待フロー
- メンバー管理画面
- テナント分離テスト・RBAC テスト追加

### Phase 4: デプロイ・仕上げ
- AWS インフラ構築（Terraform or CDK）
- staging / production デプロイ
- ヘルスチェックエンドポイント（`/health`）
- CloudWatch アラート設定
- 英語 README 作成
- アーキテクチャ図作成

---

## セキュリティ・品質

- 認証: JWT (RS256) の検証、リフレッシュトークンローテーション
- 認可: 全 API で Tenant 所属とロールをチェック（ミドルウェア層）
- 入力検証: リクエストボディのバリデーション（型・範囲・長さ）、パラメタライズドクエリで SQL インジェクション防止
- ファイルアップロード: MIME タイプ・拡張子の二重チェック、サイズ制限 (5MB)、ストレージパスに `tenant_id` を含める
- CORS: フロントエンドドメインのみ許可
- レート制限: API Gateway or ミドルウェアで実施
- セキュリティヘッダー: HSTS, X-Content-Type-Options, X-Frame-Options
- 依存関係監査: `govulncheck` / `npm audit` を CI に組み込み

---

## 成果物チェックリスト

転職用ポートフォリオとして揃えたい成果物。

- [ ] **GitHub 公開リポジトリ**（履歴が分かるコミットメッセージ、PR ベースの開発）
- [ ] **デプロイ済み URL**（動作確認可能）
- [ ] **英語 README**（概要・セットアップ・アーキテクチャ・技術選定理由を簡潔に）
- [ ] **アーキテクチャ図**（システム構成・データフローが分かる）
- [ ] **API ドキュメント**（OpenAPI / Swagger）
- [ ] **テスト結果**（CI バッジ、カバレッジレポート）
- [ ] **運用・監視の記載**: デプロイ方法、ヘルスチェック、ログ・アラート方針を README または docs に記載
- [ ] **テナント分離・権限のテスト**: README または CI で言及し、評価者に「分離を意識している」ことを示す
- [ ] **技術選定ドキュメント**: ADR (Architecture Decision Records) 形式で主要な選定理由を記録

---

## 用語統一メモ

- **Tenant** = 1 企業 = 1 テナント。DB では `tenants` テーブル、カラムは `tenant_id` で統一する。
