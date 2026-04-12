# 環境設定・シークレット管理

## 1. この文書の役割

| 項目 | 内容 |
|------|------|
| 目的 | dev / stg / prod 環境の差分とシークレット管理方針を定義する |
| 正本情報 | 環境差分一覧、環境変数一覧、シークレット管理方針、ローテーション方針 |
| 扱わない内容 | JWT 生成・検証の実装方式（security.md に委譲）、インフラリソースの詳細設計（architecture.md に委譲） |
| 主な参照元 | `../30_arch/architecture.md`, `../50_detail_design/security.md` |
| 主な参照先 | `./release.md` |

### 参照ドキュメント

| ドキュメント | 参照内容 |
|------------|---------|
| `30_arch/architecture.md` | システム構成、ミドルウェアチェーン、SPA 配信方式 |
| `30_arch/adr/0004-infra.md` | 環境構成（ローカル / ステージング / 本番） |
| `50_detail_design/security.md` | JWT 鍵管理、CORS 設定、レート制限 |
| `50_detail_design/monitoring.md` | ログレベル設定 |
| `70_operations/release.md` | リリース前確認での環境変数チェック |

---

## ポートフォリオ対応

本文書は実務想定の構成（ECS Fargate）で記載している。ポートフォリオでの実装は [ADR-0004](../30_arch/adr/0004-infra.md) §ポートフォリオ対応を参照し、コンピュートを EC2 t3.micro に読み替えること。環境変数・シークレット管理方針はコンピュート方式に依存しないため、本文書の内容はそのまま適用可能。ただし §5.3 の ECS タスク定義による注入方式は EC2 では適用外となり、代替手段（systemd 環境変数、EC2 ユーザーデータ等）を Step 11 で設計する。

## 2. 環境一覧

| 環境名 | 識別子 | 用途 | インフラ構成 |
|--------|--------|------|------------|
| ローカル開発 | dev | 開発者のローカルマシンでの開発・テスト | Docker Compose（Go + PostgreSQL + MinIO） |
| ステージング | stg | リリース前の検証。本番同等構成の小スペック | ECS Fargate + RDS（本番同等構成） |
| 本番 | prod | 運用環境 | ECS Fargate + RDS + S3 |

---

## 3. 環境ごとの差分一覧

### 3.1 インフラ構成差分

| 項目 | dev | stg | prod |
|------|-----|-----|------|
| コンピュート | Docker Compose | ECS Fargate（0.25 vCPU / 0.5 GB x 1 タスク） | ECS Fargate（0.25 vCPU / 0.5 GB x 2 タスク） |
| DB | PostgreSQL（Docker） | RDS db.t3.micro Single-AZ | RDS db.t3.micro Single-AZ（MVP） |
| オブジェクトストレージ | MinIO（Docker） | S3 | S3 |
| ロードバランサ | なし | ALB | ALB |
| TLS | なし（HTTP） | ACM 証明書 | ACM 証明書 |
| ドメイン | localhost | staging.expense-saas.example.com | expense-saas.example.com |

### 3.2 アプリケーション設定差分

| 項目 | dev | stg | prod |
|------|-----|-----|------|
| ログレベル | DEBUG | INFO | INFO |
| CORS 許可オリジン | `http://localhost:5173` | `https://staging.expense-saas.example.com` | `https://expense-saas.example.com` |
| レート制限（認証済み） | 無制限（開発効率優先） | 100 req/min/user | 100 req/min/user |
| レート制限（未認証） | 無制限 | 20 req/min/IP | 20 req/min/IP |
| DB 自動バックアップ | 無効 | 有効（7 日間） | 有効（7 日間） |
| Container Insights | 無効 | 有効 | 有効 |
| CloudWatch Alarms | なし | 設定あり | 設定あり |
| ALB ヘルスチェック | なし（手動確認） | 有効（30 秒間隔） | 有効（30 秒間隔） |

### 3.3 DB 設定差分

| 項目 | dev | stg | prod |
|------|-----|-----|------|
| ホスト | localhost | `<stg-rds-endpoint>.rds.amazonaws.com` | `<prod-rds-endpoint>.rds.amazonaws.com` |
| ポート | 5432 | 5432 | 5432 |
| データベース名 | expense_saas | expense_saas | expense_saas |
| オーナーロール | expense_owner | expense_owner | expense_owner |
| アプリロール | expense_app | expense_app | expense_app |
| SSL | 無効 | 必須（`sslmode=require`） | 必須（`sslmode=require`） |
| RLS | 有効 | 有効 | 有効 |
| max_connections | PostgreSQL デフォルト（100） | db.t3.micro デフォルト（約 87） | db.t3.micro デフォルト（約 87） |

---

## 4. 環境変数一覧

### 4.1 サーバー設定

| 変数名 | 説明 | 型 | dev | stg | prod |
|--------|------|-----|-----|-----|------|
| `PORT` | HTTP サーバーのリッスンポート | int | `8080` | `8080` | `8080` |
| `ENV` | 実行環境識別子 | string | `development` | `staging` | `production` |
| `LOG_LEVEL` | ログ出力レベル | string | `DEBUG` | `INFO` | `INFO` |

### 4.2 データベース接続

| 変数名 | 説明 | 型 | dev | stg | prod |
|--------|------|-----|-----|-----|------|
| `DATABASE_URL` | オーナーロール接続 URL | string (secret) | `postgres://expense_owner:localpass@localhost:5432/expense_saas?sslmode=disable` | Secrets Manager | Secrets Manager |
| `DATABASE_APP_URL` | アプリロール接続 URL | string (secret) | `postgres://expense_app:localpass@localhost:5432/expense_saas?sslmode=disable` | Secrets Manager | Secrets Manager |
| `DB_MAX_OPEN_CONNS` | コネクションプール最大接続数 | int | `10` | `10` | `20` |
| `DB_MAX_IDLE_CONNS` | コネクションプール最大アイドル数 | int | `5` | `5` | `10` |
| `DB_CONN_MAX_LIFETIME` | 接続の最大生存時間 | duration | `30m` | `30m` | `30m` |

### 4.3 JWT 認証

| 変数名 | 説明 | 型 | dev | stg | prod |
|--------|------|-----|-----|-----|------|
| `JWT_PRIVATE_KEY` | RS256 署名用秘密鍵（PEM） | string (secret) | ローカル PEM ファイル | Secrets Manager | Secrets Manager |
| `JWT_PUBLIC_KEY` | RS256 検証用公開鍵（PEM） | string (secret) | ローカル PEM ファイル | Secrets Manager | Secrets Manager |
| `JWT_ACCESS_TOKEN_EXPIRY` | アクセストークン有効期間 | duration | `15m` | `15m` | `15m` |
| `JWT_REFRESH_TOKEN_EXPIRY` | リフレッシュトークン有効期間 | duration | `7d` | `7d` | `7d` |
| `JWT_ISSUER` | JWT 発行者（iss クレーム） | string | `expense-saas` | `expense-saas` | `expense-saas` |

### 4.4 S3 / オブジェクトストレージ

| 変数名 | 説明 | 型 | dev | stg | prod |
|--------|------|-----|-----|-----|------|
| `S3_BUCKET` | 領収書格納バケット名 | string | `expense-saas-receipts-dev` | `expense-saas-receipts-stg` | `expense-saas-receipts-prod` |
| `AWS_REGION` | AWS リージョン（S3 アクセスで使用） | string | `ap-northeast-1` | `ap-northeast-1` | `ap-northeast-1` |
| `S3_ENDPOINT` | S3 互換エンドポイント（MinIO 用） | string | `http://localhost:9000` | 未設定（AWS デフォルト） | 未設定（AWS デフォルト） |
| `S3_PRESIGNED_URL_EXPIRY` | 署名付き URL 有効期間 | duration | `15m` | `15m` | `15m` |
| `AWS_ACCESS_KEY_ID` | AWS アクセスキー（MinIO 用） | string (secret) | MinIO デフォルト | ECS タスクロール | ECS タスクロール |
| `AWS_SECRET_ACCESS_KEY` | AWS シークレットキー（MinIO 用） | string (secret) | MinIO デフォルト | ECS タスクロール | ECS タスクロール |

**stg / prod での AWS 認証**: ECS Fargate ではタスクロール（IAM ロール）を使用して S3 にアクセスする。`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` の環境変数は設定しない。

### 4.5 CORS

| 変数名 | 説明 | 型 | dev | stg | prod |
|--------|------|-----|-----|-----|------|
| `CORS_ALLOWED_ORIGINS` | CORS 許可オリジン（カンマ区切り） | string | `http://localhost:5173` | `https://staging.expense-saas.example.com` | `https://expense-saas.example.com` |

### 4.6 レート制限

| 変数名 | 説明 | 型 | dev | stg | prod |
|--------|------|-----|-----|-----|------|
| `RATE_LIMIT_AUTHENTICATED` | 認証済みリクエスト制限 | int (req/min) | `0`（無制限） | `100` | `100` |
| `RATE_LIMIT_UNAUTHENTICATED` | 未認証リクエスト制限 | int (req/min) | `0`（無制限） | `20` | `20` |
| `RATE_LIMIT_LOGIN` | ログイン試行制限 | int (req/min) | `0`（無制限） | `5` | `5` |
| `RATE_LIMIT_UPLOAD` | ファイルアップロード制限 | int (req/min) | `0`（無制限） | `10` | `10` |

### 4.7 パスワードハッシュ

| 変数名 | 説明 | 型 | dev | stg | prod |
|--------|------|-----|-----|-----|------|
| `ARGON2_MEMORY` | Argon2id メモリコスト（KiB） | int | `65536` | `65536` | `65536` |
| `ARGON2_ITERATIONS` | Argon2id 反復回数 | int | `3` | `3` | `3` |
| `ARGON2_PARALLELISM` | Argon2id 並列度 | int | `4` | `4` | `4` |

---

## 5. シークレット管理方針

### 5.1 シークレットの分類

| 分類 | 管理方式 | 対象 |
|------|---------|------|
| DB 接続情報 | AWS Secrets Manager（stg/prod）、環境変数（dev） | `DATABASE_URL`, `DATABASE_APP_URL` |
| JWT 鍵ペア | AWS Secrets Manager（stg/prod）、ローカルファイル（dev） | `JWT_PRIVATE_KEY`, `JWT_PUBLIC_KEY` |
| AWS 認証情報 | ECS タスクロール（stg/prod）、環境変数（dev、MinIO 用） | S3 アクセス |

### 5.2 AWS Secrets Manager の構成

| シークレット名 | 格納値 | 環境 |
|--------------|--------|------|
| `expense-saas/stg/database` | DB 接続 URL（オーナー / アプリ） | stg |
| `expense-saas/prod/database` | DB 接続 URL（オーナー / アプリ） | prod |
| `expense-saas/stg/jwt-keys` | JWT 秘密鍵 / 公開鍵（PEM） | stg |
| `expense-saas/prod/jwt-keys` | JWT 秘密鍵 / 公開鍵（PEM） | prod |

**シークレットの構造例（database）**:

```json
{
  "DATABASE_URL": "postgres://expense_owner:<password>@<endpoint>:5432/expense_saas?sslmode=require",
  "DATABASE_APP_URL": "postgres://expense_app:<password>@<endpoint>:5432/expense_saas?sslmode=require"
}
```

**シークレットの構造例（jwt-keys）**:

```json
{
  "JWT_PRIVATE_KEY": "-----BEGIN RSA PRIVATE KEY-----\n...\n-----END RSA PRIVATE KEY-----",
  "JWT_PUBLIC_KEY": "-----BEGIN PUBLIC KEY-----\n...\n-----END PUBLIC KEY-----"
}
```

### 5.3 ECS タスク定義でのシークレット参照

ECS タスク定義で Secrets Manager のシークレットを環境変数として注入する。

```json
{
  "containerDefinitions": [
    {
      "name": "api",
      "secrets": [
        {
          "name": "DATABASE_URL",
          "valueFrom": "arn:aws:secretsmanager:ap-northeast-1:123456789012:secret:expense-saas/prod/database:DATABASE_URL::"
        },
        {
          "name": "DATABASE_APP_URL",
          "valueFrom": "arn:aws:secretsmanager:ap-northeast-1:123456789012:secret:expense-saas/prod/database:DATABASE_APP_URL::"
        },
        {
          "name": "JWT_PRIVATE_KEY",
          "valueFrom": "arn:aws:secretsmanager:ap-northeast-1:123456789012:secret:expense-saas/prod/jwt-keys:JWT_PRIVATE_KEY::"
        },
        {
          "name": "JWT_PUBLIC_KEY",
          "valueFrom": "arn:aws:secretsmanager:ap-northeast-1:123456789012:secret:expense-saas/prod/jwt-keys:JWT_PUBLIC_KEY::"
        }
      ],
      "environment": [
        { "name": "PORT", "value": "8080" },
        { "name": "ENV", "value": "production" },
        { "name": "LOG_LEVEL", "value": "INFO" },
        { "name": "S3_BUCKET", "value": "expense-saas-receipts-prod" },
        { "name": "AWS_REGION", "value": "ap-northeast-1" },
        { "name": "CORS_ALLOWED_ORIGINS", "value": "https://expense-saas.example.com" },
        { "name": "RATE_LIMIT_AUTHENTICATED", "value": "100" },
        { "name": "RATE_LIMIT_UNAUTHENTICATED", "value": "20" },
        { "name": "RATE_LIMIT_LOGIN", "value": "5" },
        { "name": "RATE_LIMIT_UPLOAD", "value": "10" },
        { "name": "DB_MAX_OPEN_CONNS", "value": "20" },
        { "name": "DB_MAX_IDLE_CONNS", "value": "10" },
        { "name": "DB_CONN_MAX_LIFETIME", "value": "30m" },
        { "name": "JWT_ACCESS_TOKEN_EXPIRY", "value": "15m" },
        { "name": "JWT_REFRESH_TOKEN_EXPIRY", "value": "7d" },
        { "name": "JWT_ISSUER", "value": "expense-saas" },
        { "name": "S3_PRESIGNED_URL_EXPIRY", "value": "15m" },
        { "name": "ARGON2_MEMORY", "value": "65536" },
        { "name": "ARGON2_ITERATIONS", "value": "3" },
        { "name": "ARGON2_PARALLELISM", "value": "4" }
      ]
    }
  ]
}
```

### 5.4 アクセス制御

| 対象 | アクセスできるロール | 方法 |
|------|---------------------|------|
| Secrets Manager シークレット | ECS タスクロール | IAM ポリシーで `secretsmanager:GetSecretValue` を許可 |
| Secrets Manager シークレット | 運用担当者 | IAM ユーザー/ロールで AWS コンソール経由 |
| Secrets Manager シークレット | CI/CD パイプライン | GitHub Actions OIDC 連携の IAM ロール |
| RDS | ECS タスク（expense_app ロール） | DB 接続 URL 経由 |
| RDS | マイグレーションジョブ（expense_owner ロール） | DB 接続 URL 経由 |
| S3 バケット | ECS タスクロール | IAM ポリシーで `s3:GetObject`, `s3:PutObject` を許可 |

---

## 6. ローテーション方針

### 6.1 JWT 鍵ペア

| 項目 | 方針 |
|------|------|
| ローテーション周期 | 90 日（四半期ごと）（security.md SS2.1） |
| ローテーション方式 | 二鍵並行方式（security.md SS2.1） |
| MVP での運用 | 単一鍵ペアで運用。鍵ローテーション（二鍵並行方式・手順の自動化含む）は Phase 3 で対応する |

### 6.2 DB パスワード

| 項目 | 方針 |
|------|------|
| ローテーション周期 | 90 日（四半期ごと） |
| 対象 | expense_owner ロール、expense_app ロール |
| MVP での運用 | 手動ローテーション |

**ローテーション手順**:

```
[手順]

1. RDS で DB パスワードを変更する
   - expense_owner ロール:
     $ aws rds modify-db-instance \
         --db-instance-identifier expense-saas-prod \
         --master-user-password '<new_password>'

   - expense_app ロール:
     DB に接続して ALTER ROLE を実行する
     ALTER ROLE expense_app WITH PASSWORD '<new_password>';

2. Secrets Manager のシークレットを更新する
   $ aws secretsmanager update-secret \
       --secret-id expense-saas/prod/database \
       --secret-string '{
         "DATABASE_URL": "postgres://expense_owner:<new_password>@<endpoint>:5432/expense_saas?sslmode=require",
         "DATABASE_APP_URL": "postgres://expense_app:<new_password>@<endpoint>:5432/expense_saas?sslmode=require"
       }'

3. ECS サービスを再デプロイする
   $ aws ecs update-service \
       --cluster expense-saas-prod \
       --service expense-saas-api \
       --force-new-deployment

4. ヘルスチェックで DB 接続が正常であることを確認する
   $ curl -s https://<domain>/health
   期待結果: {"status":"ok","checks":{"database":"ok"}}
```

### 6.3 ローテーションスケジュール

| 対象 | 次回ローテーション | 間隔 |
|------|------------------|------|
| JWT 鍵ペア | 初回デプロイ後 90 日 | 90 日 |
| DB パスワード（expense_owner） | 初回デプロイ後 90 日 | 90 日 |
| DB パスワード（expense_app） | 初回デプロイ後 90 日 | 90 日 |

---

## 7. ローカル開発環境（dev）のセットアップ

### 7.1 Docker Compose 環境変数

ローカル開発では `.env` ファイルで環境変数を管理する。

```bash
# .env（Git 管理外。.gitignore に追加済み）
PORT=8080
ENV=development
LOG_LEVEL=DEBUG

# DB
DATABASE_URL=postgres://expense_owner:localpass@localhost:5432/expense_saas?sslmode=disable
DATABASE_APP_URL=postgres://expense_app:localpass@localhost:5432/expense_saas?sslmode=disable
DB_MAX_OPEN_CONNS=10
DB_MAX_IDLE_CONNS=5
DB_CONN_MAX_LIFETIME=30m

# JWT（開発用鍵ペア。本番とは異なる鍵を使用すること）
JWT_PRIVATE_KEY=<dev用秘密鍵PEM>
JWT_PUBLIC_KEY=<dev用公開鍵PEM>
JWT_ACCESS_TOKEN_EXPIRY=15m
JWT_REFRESH_TOKEN_EXPIRY=7d
JWT_ISSUER=expense-saas

# S3（MinIO）
S3_BUCKET=expense-saas-receipts-dev
AWS_REGION=ap-northeast-1
S3_ENDPOINT=http://localhost:9000
S3_PRESIGNED_URL_EXPIRY=15m
AWS_ACCESS_KEY_ID=minioadmin
AWS_SECRET_ACCESS_KEY=minioadmin

# CORS
CORS_ALLOWED_ORIGINS=http://localhost:5173

# レート制限（開発時は無制限）
RATE_LIMIT_AUTHENTICATED=0
RATE_LIMIT_UNAUTHENTICATED=0
RATE_LIMIT_LOGIN=0
RATE_LIMIT_UPLOAD=0

# Argon2id
ARGON2_MEMORY=65536
ARGON2_ITERATIONS=3
ARGON2_PARALLELISM=4
```

### 7.2 開発用鍵ペアの生成

```bash
# 開発用 JWT 鍵ペアの生成
$ openssl genrsa -out dev-private.pem 2048
$ openssl rsa -in dev-private.pem -pubout -out dev-public.pem
```

開発用鍵ペアはリポジトリに含めない。各開発者がローカルで生成する。

---

## 8. 品質チェック

- [x] dev / stg / prod の 3 環境の差分が一覧として定義されている
- [x] 全環境変数が変数名・説明・環境ごとの値（または値の種類）で定義されている
- [x] シークレット（DB 接続 URL、JWT 鍵ペア）が Secrets Manager で管理される方針が明記されている
- [x] ECS タスク定義でのシークレット参照方法が具体的に記載されている
- [x] JWT 鍵ペアのローテーションは Phase 3 で対応する方針が明記されている（security.md SS2.1 と整合）
- [x] DB パスワードのローテーション手順がコマンドレベルで記載されている
- [x] ローカル開発環境のセットアップ手順が記載されている
- [x] security.md の JWT 実装方式を再定義していない（管理方針のみ定義）
- [x] architecture.md のインフラ構成を再定義していない（環境差分のみ定義）
- [x] release.md のリリース前確認で参照可能な環境変数一覧が整備されている
- [x] stg/prod での AWS 認証が ECS タスクロール（IAM ロール）経由であることが明記されている
