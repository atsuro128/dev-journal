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

> **注**: 本文書は EC2 t3.micro 上で稼働する単一インスタンス構成（[ADR-0004](../30_arch/adr/0004-infra.md) §ポートフォリオ対応）を前提に記述する。シークレット注入は SSM Parameter Store（SecureString）+ systemd EnvironmentFile 方式を採用する（issue #186 UD-5=B 確定。Terraform 実装は issue #187 で対応）。

## 2. 環境一覧

| 環境名 | 識別子 | 用途 | インフラ構成 |
|--------|--------|------|------------|
| ローカル開発 | dev | 開発者のローカルマシンでの開発・テスト | Docker Compose（Go + PostgreSQL + MinIO） |
| ステージング | stg | リリース前の検証。本番同等構成の小スペック | EC2 t3.micro + RDS（本番同等構成、Single-AZ） |
| 本番 | prod | 運用環境 | EC2 t3.micro + RDS + S3 + CloudFront（[ADR-0007](../30_arch/adr/0007-cloudfront-https.md)） |

---

## 3. 環境ごとの差分一覧

### 3.1 インフラ構成差分

| 項目 | dev | stg | prod |
|------|-----|-----|------|
| コンピュート | Docker Compose | EC2 t3.micro（2 vCPU / 1 GB × 1 インスタンス） | EC2 t3.micro（2 vCPU / 1 GB × 1 インスタンス） |
| DB | PostgreSQL（Docker） | RDS db.t3.micro Single-AZ | RDS db.t3.micro Single-AZ（MVP） |
| オブジェクトストレージ | MinIO（Docker） | S3 | S3 |
| ロードバランサ | なし | ALB | ALB |
| CDN | なし | CloudFront（ALB 前段、TLS 終端） | CloudFront（ALB 前段、TLS 終端、ADR-0007） |
| TLS | なし（HTTP） | ACM 証明書 | CloudFront デフォルト証明書（`*.cloudfront.net`）。viewer TLS 終端は CloudFront。最小 TLS バージョン TLSv1（security.md §11「TLS 1.2 以上」と乖離、B-5 受容逸脱、ADR-0007、issue #185 C 案） |
| ドメイン | localhost | staging.expense-saas.example.com | `<dist>.cloudfront.net`（CloudFront ドメイン。独自ドメイン不採用、ADR-0007） |

※ portfolio MVP の prod は ADR-0007（issue #185）により CloudFront を ALB 前段に置き、`*.cloudfront.net` デフォルト証明書で HTTPS 化している。CloudFront〜ALB 間 HTTP（B-3）と viewer TLS 最小 TLSv1（B-5）は ADR-0007 に「受容する逸脱」として記録済み。stg 列の ACM 証明書は将来構築時の推奨値（独自ドメイン採用時）。

### 3.2 アプリケーション設定差分

| 項目 | dev | stg | prod |
|------|-----|-----|------|
| ログレベル | `debug` | `info` | `info` |
| CORS 許可オリジン | `http://localhost:5173` | `https://staging.expense-saas.example.com` | CloudFront ドメイン `https://<dist>.cloudfront.net`（ADR-0007） |
| レート制限（認証済み） | 100 req/min/user | 100 req/min/user | 100 req/min/user |
| レート制限（未認証） | 20 req/min/IP | 20 req/min/IP | 20 req/min/IP |
| DB 自動バックアップ | 無効 | 有効（7 日間） | 有効（7 日間） |
| CloudWatch Agent | 無効 | 有効（メトリクスのみモード） | 有効（メトリクスのみモード） |
| CloudWatch Alarms | なし | 設定あり | 設定あり |
| ALB ヘルスチェック | なし（手動確認） | 有効（30 秒間隔） | 有効（30 秒間隔） |

※ stg/prod 列は将来構築時の推奨値。portfolio MVP は dev 相当の単一環境で運用しており、NFR-AVAIL-003 は AWS Free Tier 制約により 1 日間保持に緩和済み（issue #181 参照）。

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
| `LOG_LEVEL` | ログ出力レベル | string | `debug` | `info` | `info` |
| `TRUSTED_PROXY_COUNT` | 信頼するリバースプロキシ段数。実クライアント IP = `X-Forwarded-For[len - TRUSTED_PROXY_COUNT]`。レート制限のキーとなるクライアント IP の取得に使う（issue #185 B-2-c / ADR-0007） | int | `0` | `2` | `2` |

`TRUSTED_PROXY_COUNT` は prod で `2`（CloudFront 追記 1 段 + ALB 追記 1 段）を設定する。dev はプロキシ段がないため `0`（`X-Forwarded-For` を無視し `RemoteAddr` を採用）。CloudFront を ALB 前段に追加したことでプロキシ段が増えており、未設定だとレート制限のクライアント IP 判定が誤動作する（ADR-0007 影響・結果を参照）。

### 4.2 データベース接続

| 変数名 | 説明 | 型 | dev | stg | prod |
|--------|------|-----|-----|-----|------|
| `DATABASE_URL` | オーナーロール接続 URL | string (secret) | `postgres://expense_owner:localpass@localhost:5432/expense_saas?sslmode=disable` | SSM Parameter Store | SSM Parameter Store |
| `APP_DATABASE_URL` | アプリロール接続 URL | string (secret) | `postgres://expense_app:localpass@localhost:5432/expense_saas?sslmode=disable` | SSM Parameter Store | SSM Parameter Store |

コネクションプール設定（最大接続数・アイドル数・接続生存時間）は環境変数として外部化していない。アプリプールは pgxpool のデフォルト値、オーナープールは `MaxConns = 5` をハードコードで使用。

### 4.3 JWT 認証

| 変数名 | 説明 | 型 | dev | stg | prod |
|--------|------|-----|-----|-----|------|
| `JWT_PRIVATE_KEY_PATH` | RS256 署名用秘密鍵のファイルパス | string | `/app/keys/private.pem` | （注1） | （注1） |
| `JWT_PUBLIC_KEY_PATH` | RS256 検証用公開鍵のファイルパス | string | `/app/keys/public.pem` | （注1） | （注1） |

（注1）stg/prod では SSM Parameter Store（SecureString）から user_data で取得した PEM 値を `/etc/expense-saas/keys/{private,public}.pem` に配置する。詳細は §5.3 を参照。

トークン有効期限（アクセス: 15分 / リフレッシュ: 7日）は SEC-003、発行者（`expense-saas`）は security.md §2.1 に基づき、実装内でハードコードされており環境変数として外部化していない。

### 4.4 S3 / オブジェクトストレージ

| 変数名 | 説明 | 型 | dev | stg | prod |
|--------|------|-----|-----|-----|------|
| `S3_BUCKET` | 領収書格納バケット名 | string | `expense-saas-receipts-dev` | `expense-saas-receipts-stg` | `expense-saas-receipts-prod` |
| `AWS_REGION` | AWS リージョン（S3 アクセスで使用） | string | `ap-northeast-1` | `ap-northeast-1` | `ap-northeast-1` |
| `S3_ENDPOINT` | S3 互換エンドポイント（MinIO 用）。内部通信・アップロード用 | string | `http://minio:9000` | 未設定（AWS デフォルト） | 未設定（AWS デフォルト） |
| `S3_PUBLIC_ENDPOINT` | 署名付きダウンロード URL 生成に使うエンドポイント。未設定時は `S3_ENDPOINT` にフォールバック。ローカル開発ではホストブラウザから到達可能な URL を設定する | string | `http://localhost:9000` | 未設定（AWS デフォルト） | 未設定（AWS デフォルト） |
| `AWS_ACCESS_KEY_ID` | AWS アクセスキー（MinIO 用） | string (secret) | MinIO デフォルト | EC2 インスタンスプロファイル | EC2 インスタンスプロファイル |
| `AWS_SECRET_ACCESS_KEY` | AWS シークレットキー（MinIO 用） | string (secret) | MinIO デフォルト | EC2 インスタンスプロファイル | EC2 インスタンスプロファイル |

**stg / prod での AWS 認証**: EC2 インスタンスプロファイル（IAM ロール）を使用して S3 にアクセスする。`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` の環境変数は設定しない。

### 4.5 CORS

| 変数名 | 説明 | 型 | dev | stg | prod |
|--------|------|-----|-----|-----|------|
| `CORS_ALLOWED_ORIGINS` | CORS 許可オリジン（カンマ区切り） | string | `http://localhost:5173` | `https://staging.expense-saas.example.com` | CloudFront ドメイン `https://<dist>.cloudfront.net` |

prod の `CORS_ALLOWED_ORIGINS` は ADR-0007（issue #185 C 案）により CloudFront ドメインを設定する。CloudFront のドメイン名は Terraform apply 後に `cloudfront_domain_name` output で判明するため、apply 後に実値を設定する 2 段階デプロイとなる。

### 4.6 レート制限

未認証 IP ベースのレート制限値（ログインエンドポイント、未認証リクエスト全般）は環境変数で上書き可能。本番では env 未設定でデフォルト値が使われる（security.md §4.5 参照）。

| 変数名 | 説明 | 型 | dev | stg | prod |
|--------|------|-----|-----|-----|------|
| `LOGIN_RATE_LIMIT_PER_MINUTE` | `/api/auth/login` の IP ベースレート制限値（req/min）。未設定時のデフォルト値は 5 | int | `100`（E2E テスト緩和用） | 未設定（デフォルト 5） | 未設定（デフォルト 5） |
| `UNAUTH_RATE_LIMIT_PER_MINUTE` | 未認証リクエスト全般の IP ベースレート制限値（req/min）。未設定時のデフォルト値は 20 | int | `200`（E2E テスト緩和用） | 未設定（デフォルト 20） | 未設定（デフォルト 20） |

**設定環境の方針**: 開発・テスト環境のみ上書き推奨。本番（stg/prod）は未設定でデフォルト値（5 / 20）を使うこと。env 上書きはブルートフォース対策の有効性を損なうため、本番値の変更は行わない。

その他のレート制限値（認証済み: 100 req/min/user、アップロード: 10 req/min/user）は実装内でハードコードされており、環境変数として外部化していない。

### 4.7 パスワードハッシュ

Argon2id パラメータは SEC-002（OWASP 推奨値）に基づき実装内でハードコードされており、環境変数として外部化していない（Memory: 65536 KiB、Iterations: 3、Parallelism: 4）。

---

## 5. シークレット管理方針

### 5.1 シークレットの分類

| 分類 | 管理方式 | 対象 |
|------|---------|------|
| DB 接続情報 | AWS SSM Parameter Store（SecureString、stg/prod）、環境変数（dev） | `DATABASE_URL`, `APP_DATABASE_URL` |
| JWT 鍵ペア | AWS SSM Parameter Store（SecureString、stg/prod）、ローカルファイル（dev） | RS256 秘密鍵・公開鍵（PEM）。アプリは `JWT_PRIVATE_KEY_PATH` / `JWT_PUBLIC_KEY_PATH` でファイルパスを参照 |
| AWS 認証情報 | EC2 インスタンスプロファイル（stg/prod）、環境変数（dev、MinIO 用） | S3 アクセス |

### 5.2 SSM Parameter Store の構成

SecureString として KMS（AWS マネージドキー `alias/aws/ssm`）で暗号化する。パラメータは項目ごとに 1 つ作成する（複数値の JSON 化はしない。SSM の標準的な利用パターンに従う）。詳細は §5.3.2 を参照。

| パラメータ名（例: prod） | 型 | 格納値 |
|---|---|---|
| `/expense-saas/prod/database/url` | SecureString | `postgres://expense_owner:<password>@<endpoint>:5432/expense_saas?sslmode=require` |
| `/expense-saas/prod/database/app_url` | SecureString | `postgres://expense_app:<password>@<endpoint>:5432/expense_saas?sslmode=require` |
| `/expense-saas/prod/jwt/private_key` | SecureString | RS256 秘密鍵（PEM 全文） |
| `/expense-saas/prod/jwt/public_key` | SecureString | RS256 公開鍵（PEM 全文） |

stg は `prod` を `stg` に読み替え。Terraform 実装（パラメータ作成・IAM 付与・user_data 改修）は **issue #187** で対応する。

### 5.3 SSM Parameter Store + systemd EnvironmentFile によるシークレット注入

EC2 単一インスタンス構成（[ADR-0004](../30_arch/adr/0004-infra.md) §ポートフォリオ対応）では、AWS Systems Manager Parameter Store（SecureString）にシークレットを格納し、EC2 起動時の user_data で復号取得 → systemd EnvironmentFile に書き出し → Docker コンテナへ `--env-file` で注入する方式を採用する。

> **Terraform 実装**: IAM ポリシー（`ssm:GetParameters` + `kms:Decrypt`）、`user_data.sh.tpl` の書き換え、SSM パラメータ定義は **issue #187** で対応する。本文書は設計方針のみを記述する。

#### 5.3.1 採用理由

| 観点 | SSM Parameter Store + EnvironmentFile |
|------|---------------------------------------|
| 機密保管 | KMS による暗号化保存（AWS マネージドキー `alias/aws/ssm` を使用、無料枠内） |
| アクセス制御 | EC2 インスタンスプロファイルに `ssm:GetParameters` + `kms:Decrypt` を付与（パラメータ ARN / KMS キー単位で限定） |
| ローテーション | SSM パラメータ値の更新 + `sudo systemctl restart expense-saas` のみ（Terraform apply 不要、§6.2 参照） |
| コスト | Standard tier は無料（10,000 パラメータまで、API コール 40 TPS 無料） |
| 監査 | CloudTrail に `GetParameters` / `Decrypt` のログが残る |

#### 5.3.2 SSM パラメータ命名規則

| パラメータ名 | 型 | 値 | 環境 |
|------------|-----|-----|------|
| `/expense-saas/stg/database/url` | SecureString | DB 接続 URL（オーナーロール） | stg |
| `/expense-saas/stg/database/app_url` | SecureString | DB 接続 URL（アプリロール） | stg |
| `/expense-saas/stg/jwt/private_key` | SecureString | JWT 秘密鍵（PEM） | stg |
| `/expense-saas/stg/jwt/public_key` | SecureString | JWT 公開鍵（PEM） | stg |
| `/expense-saas/prod/database/url` | SecureString | DB 接続 URL（オーナーロール） | prod |
| `/expense-saas/prod/database/app_url` | SecureString | DB 接続 URL（アプリロール） | prod |
| `/expense-saas/prod/jwt/private_key` | SecureString | JWT 秘密鍵（PEM） | prod |
| `/expense-saas/prod/jwt/public_key` | SecureString | JWT 公開鍵（PEM） | prod |

KMS キー: AWS マネージドキー `alias/aws/ssm` を使用（カスタマー管理キーは MVP では作成しない）。

#### 5.3.3 user_data スニペット（取得 → EnvironmentFile 生成）

EC2 起動時に user_data 内で SSM から取得し、`/etc/expense-saas/app.env`（permissions `0640`、owner `root:appgroup`）と JWT 鍵ファイル（`/etc/expense-saas/keys/*.pem`、permissions `0600`）を生成する。

```bash
#!/bin/bash
set -euo pipefail

ENV_NAME="prod"  # stg / prod を切り替え
REGION="ap-northeast-1"
SECRET_DIR="/etc/expense-saas"
KEY_DIR="${SECRET_DIR}/keys"

# ディレクトリ準備
mkdir -p "${KEY_DIR}"
chmod 750 "${SECRET_DIR}" "${KEY_DIR}"
getent group appgroup >/dev/null || groupadd -r appgroup

# SSM Parameter Store から SecureString を一括取得（KMS 復号付き）
# --output json + jq -r 方式: PEM の改行を含む多行 SecureString も無傷で取り出せる
# （awk + --output text 方式は PEM 多行値で改行が欠落するため使用不可。issue #187 B-01 修正）
# jq は Amazon Linux 2023 にプリインストールされていないため dnf install -y jq が必要
PARAMS=$(aws ssm get-parameters \
  --names \
    "/expense-saas/${ENV_NAME}/database/url" \
    "/expense-saas/${ENV_NAME}/database/app_url" \
    "/expense-saas/${ENV_NAME}/jwt/private_key" \
    "/expense-saas/${ENV_NAME}/jwt/public_key" \
  --with-decryption \
  --region "${REGION}" \
  --output json)

# jq で Name -> Value を引くヘルパ
get_param() {
  printf '%s' "${PARAMS}" | jq -r --arg n "$1" '.Parameters[] | select(.Name == $n) | .Value'
}

DATABASE_URL=$(get_param "/expense-saas/${ENV_NAME}/database/url")
APP_DATABASE_URL=$(get_param "/expense-saas/${ENV_NAME}/database/app_url")
JWT_PRIVATE_KEY=$(get_param "/expense-saas/${ENV_NAME}/jwt/private_key")
JWT_PUBLIC_KEY=$(get_param "/expense-saas/${ENV_NAME}/jwt/public_key")

# JWT 鍵ペアをファイルに書き出し
printf '%s\n' "${JWT_PRIVATE_KEY}" > "${KEY_DIR}/private.pem"
printf '%s\n' "${JWT_PUBLIC_KEY}"  > "${KEY_DIR}/public.pem"
chmod 600 "${KEY_DIR}"/*.pem
chown root:appgroup "${KEY_DIR}"/*.pem

# アプリ環境変数ファイルを生成（systemd EnvironmentFile が読み込む）
cat > "${SECRET_DIR}/app.env" <<EOF
PORT=8080
LOG_LEVEL=info
DATABASE_URL=${DATABASE_URL}
APP_DATABASE_URL=${APP_DATABASE_URL}
JWT_PRIVATE_KEY_PATH=/app/keys/private.pem
JWT_PUBLIC_KEY_PATH=/app/keys/public.pem
S3_BUCKET=expense-saas-receipts-${ENV_NAME}
AWS_REGION=${REGION}
CORS_ALLOWED_ORIGINS=https://<dist>.cloudfront.net
TRUSTED_PROXY_COUNT=2
EOF
chmod 640 "${SECRET_DIR}/app.env"
chown root:appgroup "${SECRET_DIR}/app.env"

# 機密値を環境からクリア
unset PARAMS DATABASE_URL APP_DATABASE_URL JWT_PRIVATE_KEY JWT_PUBLIC_KEY
```

#### 5.3.4 systemd unit スニペット

`/etc/systemd/system/expense-saas.service` を配置し、`docker run --env-file /etc/expense-saas/app.env` でコンテナに環境変数を注入する。本スニペットは `expense-saas/infra/terraform/user_data.sh.tpl` L67-80 と一致させる（実装側の正本）。ローカル image tag は `expense-saas:portfolio` 固定（ECR pull 後は `docker tag <remote-uri> expense-saas:portfolio` で付け替える運用、release.md §4.2 参照）。

```ini
[Unit]
Description=Expense SaaS API
After=docker.service
Requires=docker.service

[Service]
Restart=always
ExecStartPre=-/usr/bin/docker rm -f expense-saas
ExecStart=/usr/bin/docker run --name expense-saas \
  --env-file /etc/expense-saas/app.env \
  -v /etc/expense-saas/keys:/app/keys:ro \
  -p 8080:8080 \
  expense-saas:portfolio
ExecStop=/usr/bin/docker stop expense-saas

[Install]
WantedBy=multi-user.target
```

#### 5.3.5 IAM 権限（EC2 インスタンスプロファイル）

EC2 インスタンスプロファイルに以下のポリシーを付与する（パラメータ ARN と KMS キーを限定）。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ssm:GetParameters"],
      "Resource": [
        "arn:aws:ssm:ap-northeast-1:<account>:parameter/expense-saas/prod/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": ["kms:Decrypt"],
      "Resource": "arn:aws:kms:ap-northeast-1:<account>:alias/aws/ssm"
    }
  ]
}
```

> **Terraform 実装との差分（issue #187 B-02 / §6.5 注記）**: 上記 IAM Resource は `parameter/expense-saas/prod/*`（prod 環境固定）で記述している。一方、issue #187 の Terraform 実装（`iam.tf`）では `parameter/expense-saas/*`（全 env 共通）を採用している。理由: env 名（portfolio / stg / prod）を追加するたびに IAM ポリシーを修正しなくて済むよう、パスプレフィックスを env 名の上位で絞っている。本文書（設計方針）は prod を例示しているが、Terraform 実装では `/expense-saas/*` として env 横断のアクセスを許可していることに注意する。

> **B-05 修正（PR #152 codex レビュー）**: `kms:Decrypt` の明示ポリシーは Terraform 実装から削除済み。alias ARN (`alias/aws/ssm`) は KMS IAM ポリシーの key 操作 Resource として無効であり、alias 操作にしか使用できない。AWS managed key `alias/aws/ssm` は SSM Parameter Store SecureString の `ssm:GetParameters` 呼び出し時に default grant が自動付与されるため、明示的な `kms:Decrypt` ポリシーは不要。本文書の JSON 例は設計上の経緯として残すが、Terraform 実装（`iam.tf`）では `aws_iam_role_policy.ec2_kms_decrypt` を削除している。

### 5.4 アクセス制御

| 対象 | アクセスできるロール | 方法 |
|------|---------------------|------|
| SSM Parameter Store パラメータ | EC2 インスタンスプロファイル | IAM ポリシーで `ssm:GetParameters` + `kms:Decrypt` を許可（パラメータ ARN / KMS キー単位で限定） |
| SSM Parameter Store パラメータ | 運用担当者 | IAM ユーザー/ロールで AWS コンソールまたは `aws ssm put-parameter` 経由 |
| SSM Parameter Store パラメータ | CI/CD パイプライン | GitHub Actions OIDC 連携の IAM ロール |
| RDS | EC2 上のアプリ（expense_app ロール） | DB 接続 URL 経由 |
| RDS | マイグレーションジョブ（expense_owner ロール） | DB 接続 URL 経由 |
| S3 バケット | EC2 インスタンスプロファイル | IAM ポリシーで `s3:GetObject`, `s3:PutObject` を許可 |

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

EC2 + SSM Parameter Store 方式では Terraform apply 不要。SSM パラメータ更新 + EC2 上でサービス再起動のみで完結する（ダウンタイムは `systemctl restart` 中の数秒〜十数秒）。

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

2. SSM Parameter Store のパラメータ値を更新する（terraform apply 不要）
   $ aws ssm put-parameter \
       --name /expense-saas/prod/database/url \
       --type SecureString \
       --value "postgres://expense_owner:<new_password>@<endpoint>:5432/expense_saas?sslmode=require" \
       --overwrite

   $ aws ssm put-parameter \
       --name /expense-saas/prod/database/app_url \
       --type SecureString \
       --value "postgres://expense_app:<new_password>@<endpoint>:5432/expense_saas?sslmode=require" \
       --overwrite

3. EC2 に SSM Session Manager で接続し、systemd unit に再読込させる
   $ aws ssm start-session --target i-xxxxxxxxxxxxxxxxx

   # EC2 上で /etc/expense-saas/app.env を SSM から再生成するスクリプトを実行
   # （user_data と同じロジック。/usr/local/bin/refresh-env.sh として配置を想定）
   $ sudo /usr/local/bin/refresh-env.sh

   # アプリを再起動して新しい DATABASE_URL を読み込ませる
   $ sudo systemctl restart expense-saas

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
LOG_LEVEL=debug

# DB
DATABASE_URL=postgres://expense_owner:localpass@localhost:5432/expense_saas?sslmode=disable
APP_DATABASE_URL=postgres://expense_app:localpass@localhost:5432/expense_saas?sslmode=disable

# JWT（開発用鍵ペア。本番とは異なる鍵を使用すること）
JWT_PRIVATE_KEY_PATH=/app/keys/private.pem
JWT_PUBLIC_KEY_PATH=/app/keys/public.pem

# S3（MinIO）
S3_BUCKET=expense-saas-receipts-dev
AWS_REGION=ap-northeast-1
S3_ENDPOINT=http://localhost:9000
# 署名付きダウンロード URL 生成用エンドポイント。未設定時は S3_ENDPOINT にフォールバック
S3_PUBLIC_ENDPOINT=http://localhost:9000
AWS_ACCESS_KEY_ID=minioadmin
AWS_SECRET_ACCESS_KEY=minioadmin

# CORS
CORS_ALLOWED_ORIGINS=http://localhost:5173
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
- [x] シークレット（DB 接続 URL、JWT 鍵ペア）が SSM Parameter Store（SecureString）で管理される方針が明記されている
- [x] SSM Parameter Store + systemd EnvironmentFile によるシークレット注入方法が具体的に記載されている
- [x] JWT 鍵ペアのローテーションは Phase 3 で対応する方針が明記されている（security.md SS2.1 と整合）
- [x] DB パスワードのローテーション手順がコマンドレベルで記載されている
- [x] ローカル開発環境のセットアップ手順が記載されている
- [x] security.md の JWT 実装方式を再定義していない（管理方針のみ定義）
- [x] architecture.md のインフラ構成を再定義していない（環境差分のみ定義）
- [x] release.md のリリース前確認で参照可能な環境変数一覧が整備されている
- [x] stg/prod での AWS 認証が EC2 インスタンスプロファイル（IAM ロール）経由であることが明記されている
