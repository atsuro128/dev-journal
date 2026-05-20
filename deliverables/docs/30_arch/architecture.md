# アーキテクチャ設計

## この文書の役割

| 項目 | 内容 |
|------|------|
| 目的 | システム全体の構造と共通方式を定義する |
| 正本情報 | レイヤ責務、認証/認可の通し方、URL 方針、テナント分離方式、非機能実現方針 |
| 扱わない内容 | 判断理由の詳細（ADR に委譲）、代替案比較表・採否理由（ADR に委譲）、図表現全般（Mermaid・ASCII・ブロック図。diagrams.md に委譲）、機能別詳細設計 |
| 主な参照元 | `../10_requirements/requirements.md`, `../10_requirements/policies.md`, `../20_domain/*`, `./adr/*.md` |
| 主な参照先 | `./diagrams.md`, `../50_detail_design/*`, `../70_operations/*` |

## 1. 概要

本書は ADR-0001〜0005 の決定を統合し、経費精算SaaS のシステム全体構造を定義する。

### 参照ドキュメント

| ドキュメント | 役割 |
|------------|------|
| `adr/0001-tech-stack.md` | 技術スタック・主要ライブラリの選定理由 |
| `adr/0002-multi-tenant.md` | マルチテナント方式（Shared DB + tenant_id） |
| `adr/0003-rls-tenant-isolation.md` | RLS テナント分離方式の判断根拠 |
| `adr/0004-infra.md` | インフラ構成（ECS Fargate / RDS / S3） |
| `adr/0005-monitoring-logging.md` | 監視・ログ戦略 |
| `20_domain/domain_model.md` | ドメインモデル・集約・不変条件 |
| `10_requirements/requirements.md` | 機能要件・非機能要件 |

---

## 2. システム全体構成

> 関連 ADR: [ADR-0001](adr/0001-tech-stack.md), [ADR-0004](adr/0004-infra.md), [ADR-0007](adr/0007-cloudfront-https.md)

### レイヤー構成

> 図については [diagrams.md](diagrams.md) 1 システム構成図 および 2 リクエスト処理フロー を参照

クライアント（React + TypeScript + Vite）からのリクエストは CloudFront（HTTPS 終端）→ ALB を経由し、ECS Fargate 上の Go API サーバーが処理する（経路: `Browser → CloudFront → ALB → Task`）。CloudFront は ALB の前段で TLS を終端し、エンドユーザー〜サーバー間を HTTPS 化する（issue #185 / ADR-0007）。`/api/*` は非キャッシュ、それ以外（SPA 静的ファイル）はエッジキャッシュする 2 ビヘイビア構成とし、ALB へのオリジン直アクセスはカスタムヘッダ検証 + CloudFront マネージドプレフィックスリストで遮断する（B-1-b）。サーバー内部は以下のレイヤーで構成される。

| レイヤー | 責務 |
|---------|------|
| ミドルウェアチェーン | CORS → SecurityHeaders → RequestID → Logger → RateLimit → Auth（JWT検証） → TenantContext（RLS設定） → RBAC（ロール検証） |
| ハンドラ層（HTTP Handler） | リクエスト/レスポンスの変換、入力バリデーション |
| サービス層（Application Service） | ユースケースの実行、トランザクション管理、集約間の調整、所有権・ビジネスルール判定（詳細は `authz.md` SS3 で定義） |
| ドメイン層（Domain） | エンティティ・値オブジェクト、状態遷移の制御（WFL-001）、不変条件の検証、ドメインエラーの生成 |
| リポジトリ層（Repository） | sqlc 生成コードの利用、tenant_id フィルタの強制（TNT-002, TNT-003）、論理削除の適用（DAT-002） |

データストアとして RDS（PostgreSQL + RLS）と S3（領収書ストレージ）を使用し、CloudWatch にログ・メトリクスを出力する。

S3 オブジェクトキーは `{tenant_id}/{report_id}/{attachment_id}` 形式とし、テナント単位のプレフィックスで分離する。詳細は `50_detail_design/files.md` §2.2 を参照。

---

## 3. バックエンド詳細

> 関連 ADR: [ADR-0001](adr/0001-tech-stack.md), [ADR-0002](adr/0002-multi-tenant.md), [ADR-0003](adr/0003-rls-tenant-isolation.md)

### 3.1 ディレクトリ構成

```
expense-saas/
├── cmd/
│   ├── server/              # エントリーポイント
│   └── seed/                # シードデータ投入 CLI
├── internal/
│   ├── config/              # 環境変数・設定管理
│   ├── domain/              # エンティティ・値オブジェクト・リポジトリIF・ドメインエラー
│   ├── service/             # ユースケース・DTO・認可ロジック・ビジネスロジック
│   ├── handler/             # HTTP ハンドラ・リクエスト/レスポンス変換
│   ├── repository/postgres/ # データアクセス実装（PostgreSQL / sqlc）
│   ├── middleware/          # HTTP ミドルウェア（認証・認可・テナント分離・ロギング）
│   ├── pkg/                 # 内部共有パッケージ（JWT・S3 等）
│   ├── seed/                # 開発用シードデータ
│   └── testutil/            # テストユーティリティ
├── db/
│   ├── migrations/          # golang-migrate マイグレーションファイル
│   └── queries/             # sqlc クエリ定義
├── scripts/                 # 運用スクリプト（鍵生成、DB 初期化等）
├── frontend/                # フロントエンド（§4 参照）
├── .github/workflows/       # CI/CD パイプライン定義
├── Dockerfile
├── docker-compose.yml
├── go.mod
└── go.sum
```

> ファイルレベルの構成は [`../80_foundation/directory_structure.md`](../80_foundation/directory_structure.md) を参照。

### 3.2 ミドルウェアチェーン

> 関連 ADR: [ADR-0001](adr/0001-tech-stack.md), [ADR-0003](adr/0003-rls-tenant-isolation.md)

リクエスト処理の順序。各ミドルウェアは Chi のミドルウェアパターンで実装する。

> 図については [diagrams.md](diagrams.md) 2 リクエスト処理フロー を参照

| 順序 | ミドルウェア | 責務 | 関連要件 |
|------|------------|------|---------|
| [1] | CORS | 許可オリジンを環境変数で指定 | SEC-013 |
| [2] | SecurityHeaders | HSTS, X-Content-Type-Options, X-Frame-Options | SEC-014 |
| [3] | RequestID | UUID を生成し context + レスポンスヘッダー（X-Request-ID）に設定 | - |
| [4] | Logger | 構造化ログ出力（request_id, method, path, status, duration_ms） | - |
| [5] | RateLimit | 認証済み 100 req/min/user, 未認証 20 req/min/IP | SEC-012 |
| [6] | Auth（JWT検証） | RS256 で署名検証、有効期限確認。認証不要エンドポイントはスキップ | SEC-003, SEC-004 |
| [7] | TenantContext | pool.Acquire でコネクション固定。BEGIN + SET LOCAL app.current_tenant。リクエスト完了時に COMMIT/ROLLBACK | TNT-002 |
| [8] | RBAC | ルートごとに必要ロールを検証。権限不足は 403 | RBC-001 |

### 3.3 認証フロー

> 関連 ADR: [ADR-0001](adr/0001-tech-stack.md)

#### ログイン → JWT 発行

> 図については [diagrams.md](diagrams.md) 3 認証フロー を参照

1. クライアントが `POST /api/auth/login` に email, password を送信
2. サーバーはオーナーロール接続（RLS バイパス）で users テーブルを参照
3. Argon2id でパスワードを検証
4. tenant_memberships から role, tenant_id を取得
5. JWT を生成: access_token（15分、claims: user_id, tenant_id, role）、refresh_token（7日）
6. トークンペアをクライアントに返却

#### 認証付きリクエスト

> 図については [diagrams.md](diagrams.md) 3 認証フロー を参照

1. クライアントが `Authorization: Bearer {access_token}` ヘッダー付きでリクエスト
2. Auth ミドルウェアが JWT RS256 検証
3. TenantContext ミドルウェアが Acquire conn + BEGIN + SET LOCAL app.current_tenant
4. RBAC ミドルウェアがロール検証
5. ハンドラがビジネスロジックを実行（同一 conn、RLS 適用）
6. COMMIT（SET LOCAL 自動リセット）後、レスポンスを返却

### 3.4 テナント分離の実行フロー

二重保証の動作を示す。

TenantContext ミドルウェアが JWT claims から tenant_id を取得し、`SET LOCAL app.current_tenant` でコネクションにテナントを紐付ける。リポジトリ層は sqlc 生成クエリで `WHERE tenant_id = $1` を明示的に付与し、RLS ポリシーがセーフティネットとして WHERE 句の漏れを防ぐ。他テナントのリソースへのアクセスは 404 Not Found を返し、リソースの存在自体を漏洩しない（TNT-006）。

フロー図は [diagrams.md](diagrams.md) §4 テナント分離フロー を参照。

> **RLS 方式のトレードオフ**: コネクションプール上限、RDS Proxy との相性、リードレプリカ非対応、集計クエリの性能など既知の限界とスケール時の対応方針は [ADR-0003](adr/0003-rls-tenant-isolation.md) §スケール時の既知の限界 を参照。

### 3.5 エラーハンドリング

ドメインエラーは domain_model.md §8 で定義済み。ハンドラ層で HTTP レスポンスに変換する。

```go
// レスポンス形式（統一）
{
  "error": {
    "code": "INVALID_STATE_TRANSITION",
    "message": "この状態からの遷移は許可されていません"
  }
}
```

| ドメインエラー | HTTP ステータス | エラーコード |
|---------------|----------------|-------------|
| InvalidStateTransition | 422 | INVALID_STATE_TRANSITION |
| SelfApprovalNotAllowed | 403 | SELF_APPROVAL_NOT_ALLOWED |
| EmptyReportSubmission | 422 | EMPTY_REPORT_SUBMISSION |
| InvalidPeriod | 422 | INVALID_PERIOD |
| ReportNotEditable | 422 | REPORT_NOT_EDITABLE |
| NoApproverInTenant | 422 | NO_APPROVER_IN_TENANT |
| ResourceNotFound | 404 | RESOURCE_NOT_FOUND |
| Forbidden | 403 | FORBIDDEN |
| InvalidFileType | 422 | INVALID_FILE_TYPE |
| InvalidAmount | 422 | INVALID_AMOUNT |
| ReportNotDeletable | 422 | REPORT_NOT_DELETABLE |
| MissingRejectionReason | 422 | MISSING_REJECTION_REASON |
| FileTooLarge | 413 | FILE_TOO_LARGE |
| ConflictError | 409 | CONFLICT |

---

## 4. フロントエンド詳細

> 関連 ADR: [ADR-0001](adr/0001-tech-stack.md), [ADR-0004](adr/0004-infra.md)

### 4.0 SPA 配信方式

MVP では **Go コンテナに Vite build 成果物を同梱**して配信する。SPA 配信方式の選定理由は [ADR-0004](adr/0004-infra.md) を参照。

Vite でビルドした静的ファイル（`dist/`）を `go:embed` で Go バイナリに埋め込む。リクエストルーティングは `/api/*` を Go API ハンドラ、`/health` をヘルスチェック、それ以外を埋め込み静的ファイル（SPA fallback: `index.html`）に振り分ける。

配信フロー図は [diagrams.md](diagrams.md) §6 デプロイ・配信 を参照。

### 4.1 ディレクトリ構成

```
expense-saas/frontend/
├── src/
│   ├── api/               # API クライアント・型定義
│   ├── components/        # 共有 UI コンポーネント
│   │   ├── ui/            #   MUI カスタムコンポーネント
│   │   ├── layout/        #   レイアウト（Header, Sidebar, AppLayout 等）
│   │   ├── report/        #   経費レポート関連コンポーネント
│   │   ├── dashboard/     #   ダッシュボード関連コンポーネント
│   │   └── auth/          #   認証関連（PrivateRoute 等）
│   ├── contexts/          # React Context（通知等）
│   ├── hooks/             # カスタムフック（認証・レポート・ワークフロー・添付等）
│   ├── lib/               # ユーティリティ関数
│   ├── pages/             # ページコンポーネント（ルーティング単位）
│   │   ├── login/         #   ログイン
│   │   ├── signup/        #   サインアップ
│   │   ├── password-reset/#   パスワードリセット
│   │   ├── auth/          #   認証共通（ナビリンク等）
│   │   ├── dashboard/     #   ダッシュボード
│   │   ├── reports/       #   経費レポート（一覧・詳細・作成・編集・明細・添付）
│   │   ├── admin/         #   管理者（テナント情報・全レポート一覧）
│   │   ├── workflow/      #   ワークフロー（承認待ち・支払待ち）
│   │   └── error/         #   エラー（404 等）
│   ├── stores/            # Zustand ストア（認証状態等）
│   └── test/              # テストセットアップ
├── index.html
├── vite.config.ts
├── theme.ts               # MUI テーマ設定
└── package.json
```

テストを持つディレクトリ（`pages/*`, `components/*`, `hooks`, `api`, `stores`, `lib`）には `__tests__/` サブディレクトリを配置し、ユニットテストを格納する（コロケーションパターン）。

> ファイルレベルの構成は [`../80_foundation/directory_structure.md`](../80_foundation/directory_structure.md) を参照。

### 4.2 認証状態管理

```
[ログイン成功]
  ↓
access_token → メモリ（変数）に保持
refresh_token → メモリ（変数）に保持
  ↓
[API リクエスト]
  fetch ラッパーが Authorization: Bearer {access_token} を自動付与
  ↓
[401 レスポンス受信]
  refresh_token で /api/auth/refresh を呼び出し
  → 成功: 新しい access_token で元のリクエストをリトライ
  → 失敗: ログイン画面にリダイレクト
```

### 4.3 ページとロールの対応

| ページ | 対応ロール | 主な機能 |
|--------|-----------|---------|
| ログイン | 全ロール（未認証） | メール・パスワード入力 |
| ダッシュボード | 全ロール | ロール別の件数サマリー |
| レポート一覧（自分） | Member, Approver, Admin, Accounting | 自分のレポート管理 |
| レポート作成・編集 | Member, Approver, Admin, Accounting | 経費レポート・明細の入力 |
| レポート詳細 | 全ロール（権限に準ずる） | 明細・添付・状態の閲覧 |
| 承認待ち一覧 | Approver | submitted レポートの承認・却下 |
| 支払待ち一覧 | Accounting | approved レポートの支払完了記録 |
| テナント全レポート一覧 | Admin, Accounting | テナント内の全レポート閲覧 |

### 4.4 サーバー状態管理（TanStack Query）

TanStack Query でサーバーデータのキャッシュ・再取得を管理する。

| クエリキー | エンドポイント | staleTime |
|-----------|--------------|-----------|
| ['reports', filters] | GET /api/reports | 30秒 |
| ['reports', id] | GET /api/reports/:id | 30秒 |
| ['dashboard'] | GET /api/dashboard | 60秒 |
| ['approval-pending'] | GET /api/workflow/pending | 30秒 |
| ['me'] | GET /api/auth/me | 5分 |

ミューテーション（状態変更）成功時に関連するクエリキーを invalidate して再取得をトリガーする。

---

## 5. API 設計方針

> 関連 ADR: [ADR-0001](adr/0001-tech-stack.md)

### 5.1 URL 設計

```
/api/auth/signup              POST    サインアップ
/api/auth/login               POST    ログイン
/api/auth/refresh             POST    トークンリフレッシュ
/api/auth/logout              POST    ログアウト
/api/auth/me                  GET     認証情報取得
/api/auth/password-reset      POST    パスワードリセット要求
/api/auth/password-reset/:token PUT   パスワードリセット実行

/api/reports                  GET     レポート一覧（自分）
/api/reports                  POST    レポート作成
/api/reports/:id              GET     レポート詳細
/api/reports/:id              PUT     レポート編集
/api/reports/:id              DELETE  レポート削除
/api/reports/:id/submit       POST    レポート提出
/api/reports/all              GET     テナント全レポート一覧（Admin, Accounting）

/api/reports/:id/items        POST    明細追加
/api/reports/:id/items/:itemId PUT    明細編集
/api/reports/:id/items/:itemId DELETE 明細削除

/api/reports/:id/items/:itemId/attachments      POST    添付アップロード（API プロキシ方式、multipart/form-data）
/api/reports/:id/items/:itemId/attachments      GET     添付一覧
/api/reports/:id/items/:itemId/attachments/:attId GET   添付ダウンロード（署名付きURL）
/api/reports/:id/items/:itemId/attachments/:attId DELETE 添付削除

/api/workflow/pending         GET     承認待ち一覧（Approver）
/api/workflow/:id/approve     POST    承認
/api/workflow/:id/reject      POST    却下
/api/workflow/payable         GET     支払待ち一覧（Accounting）
/api/workflow/:id/pay         POST    支払完了

/api/dashboard                GET     ダッシュボード

/api/tenant                   GET     テナント情報取得
/api/tenant/members           GET     テナント内メンバー一覧取得

/api/categories               GET     カテゴリ一覧取得

/health                       GET     ヘルスチェック（※ /api 外）
```

### 5.2 共通レスポンス形式

```json
// 成功（単一リソース）
{
  "data": { ... }
}

// 成功（一覧、オフセットベースページネーション）
{
  "data": [ ... ],
  "pagination": {
    "current_page": 1,
    "per_page": 20,
    "total_count": 150,
    "total_pages": 8
  }
}

// エラー
{
  "error": {
    "code": "ERROR_CODE",
    "message": "人間可読なメッセージ"
  }
}
```

### 5.3 ページネーション

オフセットベース（requirements.md §4.1: デフォルト20件/ページ）。

```
GET /api/reports?page=1&per_page=20
```

- `page`: ページ番号（デフォルト1）
- `per_page`: 1ページあたりの取得件数（デフォルト20、最大100）

---

## 6. セキュリティアーキテクチャ

> 関連 ADR: [ADR-0001](adr/0001-tech-stack.md), [ADR-0002](adr/0002-multi-tenant.md), [ADR-0003](adr/0003-rls-tenant-isolation.md), [ADR-0007](adr/0007-cloudfront-https.md)

### 6.1 多層防御

```
[1] ネットワーク層
    - CloudFront で TLS 終端（HTTPS 化、ADR-0007）
    - ALB はカスタムヘッダ検証 + CloudFront マネージドプレフィックスリストで CloudFront 経由を強制（B-1-b）
    - セキュリティグループで ECS / RDS のアクセス制御

[2] トランスポート層
    - HSTS（SEC-014）
    - CORS 制御（SEC-013）

[3] 認証層
    - JWT RS256（SEC-004, ADR-0006）
    - アクセストークン 15分 + リフレッシュトークン 7日（SEC-003）
    - Argon2id パスワードハッシュ（SEC-002）

[4] 認可層
    - RBAC ミドルウェア（RBC-001）
    - 所有権チェック（RBC-003）

[5] データアクセス層
    - リポジトリ層の tenant_id 強制（TNT-002, TNT-003）
    - RLS（TNT-004）

[6] 出力制御
    - テナント境界越え → 404（TNT-006）
    - 認証失敗 → ユーザー存在を推測させない応答（SEC-011）
```

### 6.2 レート制限

| 対象 | 制限 | キー |
|------|------|------|
| 認証済みリクエスト | 100 req/min | user_id |
| 未認証リクエスト | 20 req/min | IP アドレス |

---

## 7. 監視・ログ方針

> 関連 ADR: [ADR-0005](adr/0005-monitoring-logging.md)

| 領域 | 方針 |
|------|------|
| ログ | log/slog → JSON → stdout → CloudWatch Logs |
| メトリクス | CloudWatch（ECS/RDS 自動収集 + メトリクスフィルタ + Logs Insights） |
| ヘルスチェック | GET /health（DB 接続確認含む） |
| アラート | CloudWatch Alarms → SNS → メール |

全リクエストに request_id, tenant_id, duration_ms を含む構造化ログを出力する。

---

## 8. CI/CD パイプライン構成

> 関連 ADR: [ADR-0001](adr/0001-tech-stack.md)

### 8.1 PR パイプライン（プルリクエスト作成・更新時）

| ステージ | バックエンド | フロントエンド |
|---------|-------------|--------------|
| lint | go vet, staticcheck | eslint, tsc --noEmit |
| test | go test -tags integration（ホスト側 db-test + host gateway 経由） | vitest run |
| build | go build ./cmd/server | npm run build |

各ステージは前ステージ成功時のみ実行（fail-fast）。ローカルで `/test` スキルまたは手動で実行する。

### 8.2 master マージパイプライン（master ブランチへのプッシュ時）

PR パイプラインと同じ lint → test → build に加え、以下を実行する。

| ステージ | 内容 | MVP での状態 |
|---------|------|-------------|
| Docker イメージビルド | ECR へプッシュ（タグ: git SHA） | 枠のみ |
| デプロイ | ECS サービス更新 | 枠のみ（現時点では手動デプロイ） |

### 8.3 ブランチ保護ルール

| ルール | 設定 |
|--------|------|
| master への直接プッシュ | 禁止（運用規約で担保） |
| PR マージ条件 | ローカルテスト実行結果を PR body に記載し、手動確認で担保する |
| ステータスチェック最新化 | ローカル実行のため N/A |

### 8.4 テスト実行環境

ホスト側で `docker compose up db-test` により PostgreSQL 16 を起動し、devcontainer から host gateway 経由で接続する。テスト前にマイグレーションを実行する。

---

## 9. 非機能要求マッピング

`requirements.md` §4 の非機能要求が本アーキテクチャのどこに反映されたかを示す。

### 9.1 パフォーマンス（§4.1）

| 要件概要 | 関連ルールID | 反映先 ADR | 反映先セクション |
|----------|-------------|-----------|----------------|
| API レスポンスタイム p95 500ms 以下 | NFR-PERF-001 | ADR-0001（Go 選定理由）, ADR-0005（メトリクス閾値） | §2 レイヤー構成, §7 監視・ログ方針 |
| ファイルアップロード 5秒以下（5MB） | NFR-PERF-002, ATT-003 | ADR-0004（S3 選定） | §5.1 URL 設計（添付エンドポイント） |
| 同時接続ユーザー数 100 | NFR-PERF-003 | ADR-0002（Shared DB 選定理由）, ADR-0004（Fargate スペック） | §2 システム全体構成 |
| オフセットベースページネーション（デフォルト20件） | NFR-PERF-004 | - | §5.3 ページネーション |

### 9.2 セキュリティ（§4.2）

| 要件概要 | 関連ルールID | 反映先 ADR | 反映先セクション |
|----------|-------------|-----------|----------------|
| JWT RS256 認証 | NFR-SEC-001, SEC-003, SEC-004 | ADR-0001（golang-jwt 選定）, ADR-0006（署名アルゴリズム選定） | §3.3 認証フロー, §6.1 多層防御 [3] |
| パスワード Argon2id ハッシュ | NFR-SEC-002, SEC-002, SEC-010 | ADR-0001（argon2id ライブラリ選定） | §6.1 多層防御 [3] |
| テナント分離（アプリ層 + RLS 二重保証） | NFR-SEC-003, TNT-001〜005 | ADR-0002, ADR-0003 | §3.4 テナント分離の実行フロー, §6.1 多層防御 [5] |
| RBAC ミドルウェア | NFR-SEC-004, RBC-001 | - | §3.2 ミドルウェアチェーン [8], §6.1 多層防御 [4] |
| レート制限 | NFR-SEC-005, SEC-012 | - | §3.2 ミドルウェアチェーン [5], §6.2 レート制限 |
| CORS 許可オリジン明示指定 | NFR-SEC-006, SEC-013 | - | §3.2 ミドルウェアチェーン [1], §6.1 多層防御 [2] |
| セキュリティヘッダー（HSTS 等） | NFR-SEC-007, SEC-014 | - | §3.2 ミドルウェアチェーン [2], §6.1 多層防御 [2] |
| ファイルアップロード MIME 検証・5MB制限 | NFR-SEC-008, ATT-013, ATT-003 | ADR-0004（S3 選定） | §5.1 URL 設計（添付エンドポイント） |

### 9.3 可用性（§4.3）

| 要件概要 | 関連ルールID | 反映先 ADR | 反映先セクション |
|----------|-------------|-----------|----------------|
| 稼働率 99.5% | NFR-AVAIL-001 | ADR-0004（Single-AZ リスク受容） | §2 システム全体構成 |
| ヘルスチェック `/health` | NFR-AVAIL-002 | ADR-0005（ヘルスチェック定義） | §7 監視・ログ方針 |
| RDS 自動バックアップ 1日間保持（AWS Free Tier 制約により portfolio 仕様、issue #181） | NFR-AVAIL-003 | ADR-0004（RDS 選定） | §2 システム全体構成 |

### 9.4 データ（§4.4）

| 要件概要 | 関連ルールID | 反映先 ADR | 反映先セクション |
|----------|-------------|-----------|----------------|
| 論理削除（deleted_at） | NFR-DATA-001, DAT-002 | - | §2 レイヤー構成（リポジトリ層の責務） |
| 全レコードに created_at, updated_at | NFR-DATA-002, DAT-004 | - | §2 レイヤー構成（リポジトリ層の責務） |
| 提出以降の物理削除不可 | NFR-DATA-003, DAT-001 | - | §3.5 エラーハンドリング（ReportNotDeletable） |
| MVP監査証跡（改ざんしにくいデータ構造） | NFR-DATA-004 | - | §2 レイヤー構成（ドメイン層の不変条件検証）, §3.5 エラーハンドリング |
| 日本円（JPY）整数値 | NFR-DATA-005, ITM-002 | - | §5.2 共通レスポンス形式 |
| 文字コード UTF-8 | NFR-DATA-006 | ADR-0004（RDS PostgreSQL: デフォルト UTF-8） | §2 システム全体構成 |
| UTC 保存、表示時 JST 変換 | NFR-DATA-007 | - | §4.2 認証状態管理（フロントエンド責務） |

### 9.5 ユーザビリティ（§4.5）

| 要件概要 | 関連ルールID | 反映先 ADR | 反映先セクション |
|----------|-------------|-----------|----------------|
| レスポンシブデザイン | NFR-UX-001 | ADR-0001（MUI 選定） | §4.1 ディレクトリ構成 |
| UI 日本語固定 | NFR-UX-002 | - | §4.3 ページとロールの対応 |
| 操作結果の即時フィードバック | NFR-UX-003 | - | §4.4 サーバー状態管理（TanStack Query） |
| クライアントサイドバリデーション | NFR-UX-004 | - | §4.1 ディレクトリ構成 |
| ローディング表示 | NFR-UX-005 | - | §4.4 サーバー状態管理（TanStack Query） |

---

## 10. 品質チェック

- [x] 「なぜその技術か」を ADR で短く言えるか → ADR-0001〜0005 で記録済み
- [x] 図（構成図・データフロー）が1枚以上あるか → diagrams.md に Mermaid 図を集約。本書の各セクションから参照リンクで連携
- [x] テナント分離の二重保証の方針が決まっているか → アプリ層 WHERE + RLS（ADR-0003）
- [x] 監視・ログの方針が決まっているか → ADR-0005、§7 で方針記載
- [x] バックエンドのレイヤー構成が明確か → Handler → Service → Domain → Repository
- [x] フロントエンドの構成が明確か → Pages / Components / API Client / TanStack Query
- [x] 認証フローが明確か → JWT 発行 → 検証 → RLS 設定の流れを定義
- [x] CI/CD パイプライン構成が定義されているか → §8 で PR パイプライン・master マージパイプライン・ブランチ保護ルール・テスト実行環境を定義- [x] MVP スコープ内に収まっているか → 02_scope.md と整合
- [x] 用語が glossary.md と一致しているか → 確認済み
