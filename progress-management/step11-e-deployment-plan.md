# Step 11-E デプロイ・スモークテスト 作業計画

作成日: 2026-05-08
作成者: planner（Claude）
対象チケット: `dev-journal/progress-management/tickets/step11/11-E-deploy.md`
依存: Step 11-D（CONDITIONAL PASS, 2026-05-07）/ PR #144（issue #175）

---

## 0. このドキュメントの位置付け

- 11-E チケットの詳細化版。本書を runbook として参照しながら作業を進められる粒度で記述する。
- 実コード・Terraform ファイルの作成は本フェーズでは行わず、計画策定のみ。
- ADR-0004 §ポートフォリオ対応（EC2 t3.micro / RDS db.t3.micro / S3 / ALB、無料枠内）に厳密に従う。Fargate 案は採用しない。
- サブエージェントは外部通信コマンド（AWS CLI / Terraform apply / docker push 等）を実行できない。これらは「ユーザー実行」とマークする（feedback_subagent_permissions.md）。
- 監視・アラート等の本番運用設定は 11-E スコープ外（11 完了後の post-MVP 対応）。

---

## 1. 全体作業フロー

```
[Phase 0] 事前条件確認
   |  (a) PR #144 (issue #175 deploy.yml 整合) merged
   |  (b) 11-D の CONDITIONAL 4 条件のうち、ローカル可能分を先消化
   |  (c) AWS アカウント・支払い設定・IAM ユーザー(Terraform 用) 準備完了 [USER]
   v
[Phase 1] Terraform IaC 作成 [SUBAGENT: platform-builder]
   |  - infra/terraform/ ディレクトリ構成・main.tf/variables.tf/...
   |  - 静的検証のみ（terraform fmt / terraform validate がローカルで通る状態）
   v
[Phase 2] state バックエンド初期化 [USER]
   |  - S3 (state) バケット + DynamoDB (lock) を AWS CLI で先行作成
   |  - terraform init -backend-config=...
   v
[Phase 3] terraform plan -> apply [USER]
   |  - VPC / SG / EC2 / RDS / S3 / ALB / IAM をプロビジョニング
   |  - apply 後に EC2 public DNS / RDS endpoint / ALB DNS を outputs から取得
   v
[Phase 4] DB マイグレーション投入 [USER 主導 / 指揮役支援]
   |  - 一時的に EC2 から (or ローカルから踏み台経由で) migrate up
   v
[Phase 5] アプリ Docker イメージ配布 [USER]
   |  - ECR を使うか EC2 上 build かは §3.4 の判断による
   |  - 初期方針: GHCR 利用 or EC2 上で git clone + docker build
   v
[Phase 6] EC2 上でアプリ起動 [USER]
   |  - systemd unit or docker run --restart=always
   |  - JWT 鍵ペアを EC2 上に配置（Secrets Manager は無料枠外なので使わない）
   v
[Phase 7] ヘルスチェック・スモークテスト [指揮役 + USER]
   |  - curl https://<alb-dns>/health
   |  - 11-D CONDITIONAL の 4 条件残り（時刻ドリフト確認）
   |  - 申請 -> 承認 -> 支払 の手動フロー
   v
[Phase 8] deploy.yml の if: false 解除（任意）[SUBAGENT: platform-builder]
   |  - PR #144 マージ後、E2E ジョブを実環境ではなく docker-compose.e2e.yml で動かす
   |  - スコープ判断: MVP では deploy.yml は参考実装のままでも 11-E 完了条件は満たす
   v
[Phase 9] 11-F への引き継ぎ
   |  - 公開 URL / UAT アカウント / デプロイ日時 / 既知非ブロッカー
```

依存関係の要点:

- Phase 1 と Phase 0(c) は並列可（Terraform 雛形作成は AWS アカウント不要）。
- Phase 2 以降は AWS 認証が必要。サブエージェントは関与不可。
- Phase 7 のスモークは Phase 6 完了が前提。
- Phase 8 は 11-E 完了条件には含めない（deploy.yml は §8.2 で「現時点では手動デプロイ」と明記済み）。

---

## 2. AWS リソース構成と無料枠制約

### 2.1 ポートフォリオ実装の構成図（テキスト）

```
[Internet]
   |
   v
[ALB] (HTTPS:443, HTTP:80 -> redirect)
   - subnet: public-a, public-c (2AZ 必須)
   - target group: HTTP:8080 -> EC2
   - ACM 証明書: ALB に貼る（必要なら Route53 + ACM、簡略化のため
                 ALB の DNS をそのまま使う案 §3.5 で判断）
   |
   v
[EC2 t3.micro]  Amazon Linux 2023 (Docker 入り)
   - subnet: public-a (Single-AZ で受容)
   - SG: ALB SG からの 8080 のみ許可。SSH (22) は my-IP のみ
   - ボリューム: gp3 8GB (無料枠は 30GB まで)
   - IAM Role:
       * S3: GetObject / PutObject / DeleteObject (受領書バケットのみ)
       * (CloudWatch Logs は MVP スコープ外)
   - Docker:
       expense-saas API コンテナ（Go バイナリ + SPA embed）
       host network or bridge で 8080 公開
   |
   v (private)
[RDS PostgreSQL 16, db.t3.micro]
   - subnet group: db-subnet-private-a, db-subnet-private-c (2AZ 必須)
   - SG: EC2 SG からの 5432 のみ許可
   - Single-AZ（ADR-0004: MVP リスク受容）
   - Storage: gp3 20GB（無料枠は 20GB まで）
   - Backup: 7 日間（NFR-AVAIL-003）
   - Public access: No
   |
   ====================== 別系統 ======================
[S3]  expense-saas-receipts-prod (or -portfolio)
   - region: ap-northeast-1
   - パブリックアクセス全面ブロック
   - SSE-S3 (デフォルト暗号化)
   - 署名付き URL でのみアクセス（ATT-010-012）

[S3]  expense-saas-tfstate-portfolio (Terraform state 用)
[DynamoDB] expense-saas-tflock (Terraform lock 用、PAY_PER_REQUEST)
```

### 2.2 無料枠条件と超過リスク

| リソース | 無料枠条件 | 超過リスク | 緩和策 |
|---|---|---|---|
| EC2 t3.micro | 12ヶ月、750h/月 | 13ヶ月目以降は約 $8/月 | 期限到来前に停止判断。`terraform destroy` を runbook 化 |
| RDS db.t3.micro Single-AZ | 12ヶ月、750h/月、Storage 20GB | Multi-AZ 化や 20GB 超で課金 | Multi-AZ にしない、`allocated_storage = 20` 固定 |
| S3 | 12ヶ月、5GB、20k GET、2k PUT | 領収書蓄積で 5GB 超 | UAT 後に `aws s3 rm --recursive` で掃除 |
| ALB | 12ヶ月、750h/月、15GB データ | 既存に 1 つでもあると同時に同一勘定 | 他用途 ALB を作らない |
| データ転送 | アウトバウンド 100GB/月 | UAT 規模では問題なし | - |
| EBS | 30GB gp3 | EC2 1 台 8GB なら問題なし | - |
| Elastic IP | EC2 にアタッチされていれば無料 | 未アタッチで $3.6/月 | 使わない（ALB 経由でアクセスする） |
| NAT Gateway | 無料枠なし。$32/月+データ課金 | **絶対に作らない** | EC2 を public subnet に配置し、private subnet は RDS のみで NAT 不要にする |
| CloudWatch | 詳細監視は無料枠外 | - | MVP では基本メトリクスのみ |
| Secrets Manager | 30 日無料枠後に $0.4/secret/月 | - | 使わず、JWT 鍵は EC2 上のファイル + SSM Parameter Store SecureString（無料）か Terraform variable で投入 |
| ECR | 無料枠なし（500MB まで $0.10/月程度） | - | §3.4 で「ECR 使わない」案と比較判断 |
| Route53 Hosted Zone | $0.50/月（無料枠なし） | - | カスタムドメイン不要なら ALB DNS をそのまま使う |
| ACM 証明書 | 無料 | - | ただし ACM は Route53 か外部 DNS 検証が必要。§3.5 で判断 |

**最大想定月額**（無料枠超過後 / 13ヶ月目以降）: §7 で算定。

### 2.3 Terraform state 管理方針

採用案: **S3 + DynamoDB バックエンド**（推奨）

- 理由:
  - 無料枠内（S3 < 1GB、DynamoDB PAY_PER_REQUEST で実質無料）
  - apply 失敗時の lock 残骸を扱える
  - 「ローカル state」案は紛失リスクが高く、ポートフォリオでも避ける
- 制約: state バケットと lock テーブルは Terraform で作れない（chicken-and-egg）。
  → AWS CLI で先行作成する手順を §5.3 に明記。

代替案: ローカル state（不採用、参考まで）

- メリット: セットアップ不要
- デメリット: PC 紛失で復旧不能、複数端末で作業不可

---

## 3. Terraform ファイル構成

### 3.1 ディレクトリ構成（提案）

```
expense-saas/
└── infra/
    └── terraform/
        ├── README.md                 # 本書 §5 の抜粋（apply 手順）
        ├── backend.tf                # S3 + DynamoDB backend 設定
        ├── providers.tf              # aws provider, version 制約
        ├── variables.tf              # 入力変数
        ├── outputs.tf                # ALB DNS、RDS endpoint 等
        ├── locals.tf                 # 命名プレフィックス等
        ├── network.tf                # VPC, subnets, route tables, IGW
        ├── security_groups.tf        # ALB SG, EC2 SG, RDS SG
        ├── ec2.tf                    # EC2 instance, key_pair (data), user_data
        ├── rds.tf                    # DB subnet group, RDS instance, parameter group
        ├── s3.tf                     # 領収書バケット、ライフサイクル、暗号化
        ├── alb.tf                    # ALB, target group, listener, ACM (option)
        ├── iam.tf                    # EC2 instance profile (S3 access)
        ├── terraform.tfvars.example  # tfvars テンプレート（gitignore せず）
        └── .gitignore                # *.tfstate, .terraform/, terraform.tfvars
```

### 3.2 各ファイルの責務と最低限定義すべきリソース

| ファイル | 主なリソース / 内容 |
|---|---|
| `backend.tf` | `terraform { backend "s3" { ... } }`（bucket, key, region, dynamodb_table, encrypt=true） |
| `providers.tf` | `aws` プロバイダ v5 系、`required_version >= 1.6` |
| `variables.tf` | `aws_region`, `project_name`, `environment` (= "portfolio"), `key_pair_name`, `db_password`（sensitive）, `allowed_ssh_cidr`（自宅 IP）, `jwt_private_key_pem`（sensitive）, `jwt_public_key_pem`（sensitive）, `cors_allowed_origins` |
| `network.tf` | `aws_vpc`（10.0.0.0/16）, `aws_subnet` ×4（public-a/c, private-a/c）, `aws_internet_gateway`, `aws_route_table` × public のみ。NAT Gateway は作らない |
| `security_groups.tf` | `alb_sg`（80/443 from 0.0.0.0/0）, `ec2_sg`（8080 from alb_sg, 22 from allowed_ssh_cidr）, `rds_sg`（5432 from ec2_sg） |
| `ec2.tf` | `aws_instance`（t3.micro, AL2023 AMI data source, instance_profile, user_data でDocker・コンテナ起動） |
| `rds.tf` | `aws_db_subnet_group`（private-a/c）, `aws_db_instance`（engine=postgres 16, db.t3.micro, allocated_storage=20, backup_retention_period=7, multi_az=false, publicly_accessible=false, storage_encrypted=true） |
| `s3.tf` | `aws_s3_bucket`（領収書）、`aws_s3_bucket_public_access_block`（all true）、`aws_s3_bucket_server_side_encryption_configuration`（AES256）、`aws_s3_bucket_versioning`（Enabled、誤削除防止）、`aws_s3_bucket_lifecycle_configuration`（noncurrent 30d expire でコスト抑制） |
| `alb.tf` | `aws_lb`（application, public-a/c）, `aws_lb_target_group`（HTTP:8080, health_check.path=/health）, `aws_lb_target_group_attachment`（EC2）, `aws_lb_listener`（443 + ACM 連携 or 80 のみ §3.5 で判断） |
| `iam.tf` | `aws_iam_role`（EC2 用）, `aws_iam_instance_profile`, `aws_iam_role_policy`（受領書バケットへの GetObject/PutObject/DeleteObject、prefix で tenant 制限はしない MVP は IAM ではなくアプリ層で担保） |
| `outputs.tf` | `alb_dns_name`, `ec2_public_ip`, `rds_endpoint`（sensitive）, `s3_bucket_name` |

### 3.3 変数化すべき値

| 変数 | 例 | 備考 |
|---|---|---|
| `aws_region` | `ap-northeast-1` | 固定でよい |
| `project_name` | `expense-saas` | リソース名プレフィックス |
| `environment` | `portfolio` | env_config.md の prod は使わず portfolio とする |
| `key_pair_name` | `expense-saas-portfolio` | 事前に AWS コンソールで作成 |
| `allowed_ssh_cidr` | `xxx.xxx.xxx.xxx/32` | ユーザーの自宅 IP |
| `db_password` | `(sensitive)` | `terraform.tfvars` 直書き or `TF_VAR_db_password` 環境変数 |
| `jwt_private_key_pem` | `(sensitive)` | `openssl genrsa` で生成済みのものを TF_VAR で渡す |
| `jwt_public_key_pem` | `(sensitive)` | 同上 |
| `cors_allowed_origins` | `https://<alb-dns>` | apply 後に再設定する 2 段階デプロイ or プレースホルダ |
| `image_tag` | `latest` or git SHA | user_data で `docker pull ghcr.io/.../expense-saas:${image_tag}` |

---

## 4. シークレット・環境変数一覧

env_config.md §4 を全文確認のうえ、本番（= portfolio 環境）で必要なものを列挙。

### 4.1 AWS 側で管理するもの

#### IAM ユーザー（Terraform 実行用、ユーザーローカル）

- 用途: `terraform apply` 実行時の AWS API 呼び出し
- 推奨ポリシー: 簡略化のため `AdministratorAccess` を一時的に付与（ポートフォリオ用途）。実務想定では VPC/EC2/RDS/S3/ALB/IAM の必要権限のみに絞る
- アクセスキー: `~/.aws/credentials` に `[expense-saas-portfolio]` プロファイルとして登録

#### IAM Role（EC2 instance profile）

- 用途: EC2 上のアプリから S3 にアクセス
- ポリシー: 領収書バケットへの `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject`, `s3:ListBucket`

#### キーペア

- AWS コンソールで `expense-saas-portfolio.pem` を作成 → ローカル `~/.ssh/` に保存（chmod 400）

### 4.2 アプリ側環境変数（EC2 上のコンテナに渡す）

env_config.md §4 prod 列を写したもの。Secrets Manager は無料枠外のため、JWT 鍵と DB URL は **EC2 上の `/etc/expense-saas/` 配下のファイルに root:600 で配置**して `docker run -v` でマウントする。

| 変数 | 値 | 取得元 |
|---|---|---|
| `PORT` | `8080` | 固定 |
| `LOG_LEVEL` | `info` | 固定 |
| `DATABASE_URL` | `postgres://expense_owner:<pw>@<rds-endpoint>:5432/expense_saas?sslmode=require` | Terraform output + tfvar の db_password |
| `APP_DATABASE_URL` | `postgres://expense_app:<pw>@<rds-endpoint>:5432/expense_saas?sslmode=require` | 同上（migrate 時に CREATE ROLE expense_app する） |
| `JWT_PRIVATE_KEY_PATH` | `/app/keys/private.pem` | コンテナ内パス（host の `/etc/expense-saas/keys/` を bind） |
| `JWT_PUBLIC_KEY_PATH` | `/app/keys/public.pem` | 同上 |
| `S3_BUCKET` | `expense-saas-receipts-portfolio` | Terraform output |
| `AWS_REGION` | `ap-northeast-1` | 固定 |
| `S3_ENDPOINT` | （未設定）| AWS デフォルトを使う |
| `S3_PUBLIC_ENDPOINT` | （未設定）| 同上 |
| `AWS_ACCESS_KEY_ID` | （未設定）| EC2 instance profile を使う |
| `AWS_SECRET_ACCESS_KEY` | （未設定）| 同上 |
| `CORS_ALLOWED_ORIGINS` | `http://<alb-dns>`（HTTPS 化したら `https://...`） | Terraform output |
| `LOGIN_RATE_LIMIT_PER_MINUTE` | （未設定。デフォルト 5） | env_config.md §4.6 の本番方針 |
| `UNAUTH_RATE_LIMIT_PER_MINUTE` | （未設定。デフォルト 20） | 同上 |

### 4.3 GitHub Secrets に登録すべき項目（Phase 8 で deploy.yml を有効化する場合のみ）

11-E 完了条件には含めない。参考のみ。

| Secret | 用途 |
|---|---|
| `AWS_ACCESS_KEY_ID` | GHA から ECR push する場合 |
| `AWS_SECRET_ACCESS_KEY` | 同上 |
| もしくは `AWS_DEPLOY_ROLE_ARN` | OIDC 連携を使う場合（推奨） |

§3.4 で「ECR 使わず EC2 上で `git clone + docker build`」案を採るなら、これらは不要。

---

## 5. デプロイ手順（runbook ドラフト）

### 5.1 事前準備

1. AWS アカウント作成と支払い情報登録（USER）
2. Budget アラートを $5/月で設定（無料枠超過の早期警告）（USER）
3. IAM ユーザー `terraform-deployer` 作成、AdministratorAccess（ポートフォリオ簡略化）、アクセスキー発行（USER）
4. ローカル `~/.aws/credentials` に追加:
   ```
   [expense-saas-portfolio]
   aws_access_key_id = ...
   aws_secret_access_key = ...
   region = ap-northeast-1
   ```
5. キーペア `expense-saas-portfolio` を AWS コンソール（EC2）で作成、`.pem` をローカル保存、`chmod 400`
6. Terraform をローカルにインストール（>= 1.6）
7. JWT 鍵ペア生成:
   ```bash
   mkdir -p /tmp/expense-saas-keys
   openssl genrsa -out /tmp/expense-saas-keys/private.pem 2048
   openssl rsa -in /tmp/expense-saas-keys/private.pem -pubout -out /tmp/expense-saas-keys/public.pem
   ```

### 5.2 11-D CONDITIONAL の事前消化（指揮役）

11-E 進入条件 4 つのうち、ローカルで完了できるものを先に消化する:

| # | 条件 | 実施タイミング | 担当 |
|---|---|---|---|
| 1 | test DB 起動 + 統合テスト再実行 | **事前消化（Phase 0）** | 指揮役（`/test` スキル） |
| 4 | frontend build をクリーン環境で実行 | **事前消化（Phase 0）** | 指揮役 |
| 2 | 起動済み API/frontend で `npm run e2e` 10件 PASS | **事前消化（Phase 0、ローカル E2E で代替）** | 指揮役（root で `npm run e2e`） |
| 3 | デプロイ環境の時刻同期 smoke | **Phase 7（実環境必須）** | USER + 指揮役 |

条件 1, 2, 4 は本物のローカル環境で先消化することで、AWS デプロイ後の手戻りを減らす。

### 5.3 Terraform state バックエンド初期化（USER 一回限り）

```bash
# AWS CLI で先行作成
export AWS_PROFILE=expense-saas-portfolio
aws s3api create-bucket \
  --bucket expense-saas-tfstate-<8桁ランダム> \
  --region ap-northeast-1 \
  --create-bucket-configuration LocationConstraint=ap-northeast-1
aws s3api put-bucket-versioning \
  --bucket expense-saas-tfstate-<8桁ランダム> \
  --versioning-configuration Status=Enabled
aws s3api put-bucket-encryption \
  --bucket expense-saas-tfstate-<8桁ランダム> \
  --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
aws dynamodb create-table \
  --table-name expense-saas-tflock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region ap-northeast-1
```

`backend.tf` の `bucket` を上で作成したバケット名に書き換える（または `-backend-config=bucket=...` で渡す）。

### 5.4 Terraform init / plan / apply（USER）

```bash
cd expense-saas/infra/terraform

# 初期化
terraform init

# tfvars 作成（terraform.tfvars.example をコピー）
cp terraform.tfvars.example terraform.tfvars
# エディタで編集: db_password, allowed_ssh_cidr, jwt_*_pem を埋める
# あるいは環境変数で渡す:
export TF_VAR_db_password='...'
export TF_VAR_jwt_private_key_pem="$(cat /tmp/expense-saas-keys/private.pem)"
export TF_VAR_jwt_public_key_pem="$(cat /tmp/expense-saas-keys/public.pem)"
export TF_VAR_allowed_ssh_cidr="$(curl -s ifconfig.me)/32"

# 計画確認
terraform plan -out=tfplan

# 確認後 apply（10-15 分かかる、特に RDS 作成）
terraform apply tfplan

# outputs 確認
terraform output
# alb_dns_name = "expense-saas-portfolio-alb-xxxxxxxx.ap-northeast-1.elb.amazonaws.com"
# ec2_public_ip = "xx.xx.xx.xx"
# rds_endpoint = (sensitive)
# s3_bucket_name = "expense-saas-receipts-portfolio-xxxxxxxx"
```

### 5.5 Docker イメージ配布の方針判断（要 USER 判断）

**案A: GHCR（GitHub Container Registry）経由**（推奨）

- メリット: 無料、private repo の image も pull できる、GHA から push 簡単
- デメリット: PAT (GITHUB_TOKEN) を EC2 に渡す必要

**案B: EC2 上で git clone + docker build**

- メリット: レジストリ不要、一番シンプル
- デメリット: ビルド時間が EC2 リソースを食う（t3.micro メモリ 1GB は厳しい、swap 必要）

**案C: ECR**

- メリット: AWS ネイティブ、IAM Role で認証
- デメリット: 無料枠なし（500MB で $0.05/月、データ転送で多少課金）

**推奨**: **案A（GHCR）**。GHA で master push 時に GHCR へ push、EC2 の user_data で `docker pull` する。GITHUB_TOKEN は read-only PAT を SSM Parameter Store SecureString（無料）で管理。

ただし最初の 1 回目は **案B（EC2 上で build）** を使って素早く検証し、運用化フェーズで案A に移行するのが現実的。

### 5.6 EC2 上でのアプリ起動（USER）

apply 完了後の user_data 自動実行か、手動 ssh で:

```bash
# ssh
ssh -i ~/.ssh/expense-saas-portfolio.pem ec2-user@<ec2_public_ip>

# Docker 動作確認
sudo systemctl status docker

# 鍵ペア配置（scp で先に置く想定）
sudo mkdir -p /etc/expense-saas/keys
sudo cp /tmp/private.pem /etc/expense-saas/keys/private.pem
sudo cp /tmp/public.pem /etc/expense-saas/keys/public.pem
sudo chmod 600 /etc/expense-saas/keys/*.pem

# 環境変数ファイル
sudo tee /etc/expense-saas/app.env <<EOF
PORT=8080
LOG_LEVEL=info
DATABASE_URL=postgres://expense_owner:<pw>@<rds-endpoint>:5432/expense_saas?sslmode=require
APP_DATABASE_URL=postgres://expense_app:<pw>@<rds-endpoint>:5432/expense_saas?sslmode=require
JWT_PRIVATE_KEY_PATH=/app/keys/private.pem
JWT_PUBLIC_KEY_PATH=/app/keys/public.pem
S3_BUCKET=<bucket>
AWS_REGION=ap-northeast-1
CORS_ALLOWED_ORIGINS=http://<alb-dns>
EOF
sudo chmod 600 /etc/expense-saas/app.env

# 案B (EC2 build) の場合
sudo dnf install -y git
git clone https://github.com/<user>/expense-saas.git
cd expense-saas
sudo docker build -t expense-saas:portfolio .

# 案A (GHCR) の場合
echo "$GHCR_PAT" | sudo docker login ghcr.io -u <user> --password-stdin
sudo docker pull ghcr.io/<user>/expense-saas:latest

# systemd unit 配置
sudo tee /etc/systemd/system/expense-saas.service <<'EOF'
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
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now expense-saas
sudo systemctl status expense-saas
```

### 5.7 DB マイグレーション投入（USER + 指揮役）

EC2 から実行（RDS は private なので EC2 経由が最も簡単）:

```bash
# EC2 上
ssh -i ~/.ssh/expense-saas-portfolio.pem ec2-user@<ec2_public_ip>

# golang-migrate を取得
GOMIG_VER=v4.17.0
curl -L https://github.com/golang-migrate/migrate/releases/download/${GOMIG_VER}/migrate.linux-amd64.tar.gz \
  | tar xz
sudo mv migrate /usr/local/bin/

# マイグレーション SQL を ssh 経由で送る or git clone 済みのものを使う
cd ~/expense-saas

# expense_owner で全マイグレーション実行
export DATABASE_URL="postgres://expense_owner:<pw>@<rds-endpoint>:5432/expense_saas?sslmode=require"
migrate -path db/migrations -database "$DATABASE_URL" up

# expense_app ロール作成（init-db.sh と同じ作業を手動で）
# scripts/init-db.sh の prod 版を確認して、 CREATE ROLE expense_app LOGIN PASSWORD '...';
# GRANT 等を psql で流す
psql "$DATABASE_URL" <<'SQL'
CREATE ROLE expense_app LOGIN PASSWORD '<app_pw>';
GRANT CONNECT ON DATABASE expense_saas TO expense_app;
GRANT USAGE ON SCHEMA public TO expense_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO expense_app;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO expense_app;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO expense_app;
SQL
```

> **要確認**: `expense-saas/scripts/init-db.sh` の中身を確認してから本手順を確定する（指揮役タスク §9）。RLS ポリシー作成も含まれている可能性あり。

### 5.8 ロールバック手順

| 失敗箇所 | 手順 |
|---|---|
| `terraform apply` 失敗 | エラー内容確認後 `terraform destroy` で全削除、または部分的に `terraform state rm` してリトライ |
| EC2 起動成功・アプリ起動失敗 | `docker logs expense-saas` でログ確認。`systemctl restart expense-saas` |
| マイグレーション失敗 | `migrate -database "$DATABASE_URL" down 1`、ダメなら RDS スナップショットからリストア |
| 全体的な切り戻し | `terraform destroy` → state は残るので再度 apply で再現可能 |

---

## 6. 検証手順

### 6.1 ヘルスチェック疎通（指揮役）

```bash
# ALB 経由
curl -i http://<alb-dns>/health
# 期待: HTTP/1.1 200 OK, body {"status":"ok",...}

# EC2 直接（ALB をバイパスして切り分け）
curl -i http://<ec2_public_ip>:8080/health
# ※ EC2 SG が ALB SG のみ許可なら外からは 8080 直叩きは届かない（正しい）

# ssh で EC2 内から
ssh ec2-user@<ec2_public_ip> 'curl -s http://localhost:8080/health'
```

### 6.2 11-D CONDITIONAL 4 条件の最終確認タイミング

| # | 条件 | タイミング | 実施方法 |
|---|---|---|---|
| 1 | test DB + Go 統合テスト | Phase 0（事前消化） | 指揮役 `/test` スキル。ローカル `docker compose --profile test run --rm migrate-test && go test -tags integration ./...` |
| 2 | `npm run e2e` 10件 PASS | Phase 0（ローカルで事前消化） | 指揮役 root で `docker compose up -d && npm run e2e`、10 件 PASS をログに記録 |
| 3 | デプロイ環境の時刻ドリフト smoke | **Phase 7（実環境必須）** | EC2: `timedatectl` で chrony 動作確認、`date -u` を JWT iat / exp 境界で確認。RDS: `SHOW timezone; SELECT NOW();` |
| 4 | frontend build をクリーン環境で実行 | Phase 0（事前消化） | 指揮役 `cd frontend && rm -rf dist && npm ci && npm run build` |

### 6.3 スモークテスト（申請 → 承認 → 支払）

11-D で確定した E2E `payment_flow.spec.ts`（CRS-055）と同等の操作を、デプロイ済み環境で **手動で** 実行する。テストデータは `seed` コマンドで投入する。

#### 6.3.1 シードデータ投入

```bash
# EC2 上で seed を実行
ssh ec2-user@<ec2_public_ip>
cd ~/expense-saas
sudo docker run --rm \
  --env-file /etc/expense-saas/app.env \
  --entrypoint /app/seed \
  expense-saas:portfolio
```

#### 6.3.2 手動スモーク手順

| # | 操作 | 期待結果 |
|---|---|---|
| 1 | ブラウザで `http://<alb-dns>/` を開く | ログイン画面が表示される |
| 2 | Member（`member1@tenant1.example.com` / seed 既定パスワード）でログイン | ダッシュボードに遷移する |
| 3 | レポート作成 → 明細追加（金額・カテゴリ・日付） → 添付（PNG/JPG 1ファイル） → 保存 → 提出 | レポートが submitted 状態になる |
| 4 | ログアウト → Approver でログイン → 承認待ち一覧 → 該当レポートを承認 | レポートが approved 状態になる |
| 5 | ログアウト → Accounting でログイン → 支払待ち一覧 → 該当レポートを支払完了 | レポートが paid 状態になる |
| 6 | 添付ダウンロード（署名付き URL）を Member 側で確認 | 添付ファイルがダウンロードできる |

#### 6.3.3 確認観点（11-E 完了条件）

- [ ] 第三者がアクセスできる（公開 URL がブラウザから到達する）
- [ ] `/health` が 200 を返す
- [ ] 申請 → 承認 → 支払のフローが画面操作で完走する

---

## 7. コスト見積り

### 7.1 12ヶ月間（無料枠内）

| リソース | 想定月額 |
|---|---|
| EC2 t3.micro | $0（無料枠 750h/月） |
| EBS gp3 8GB | $0（無料枠 30GB） |
| RDS db.t3.micro Single-AZ + 20GB | $0（無料枠 750h + 20GB） |
| ALB | $0（無料枠 750h + 15GB） |
| S3 5GB 以下 | $0（無料枠） |
| データ転送 100GB 以下 | $0（無料枠） |
| DynamoDB（state lock） | $0（PAY_PER_REQUEST、ほぼ呼ばれない） |
| **合計** | **$0/月** |

### 7.2 12ヶ月超過後（最大想定）

| リソース | 月額（USD） |
|---|---|
| EC2 t3.micro 24h × 30d | ~$8 |
| EBS gp3 8GB | ~$1 |
| RDS db.t3.micro Single-AZ | ~$15 |
| RDS Storage 20GB gp3 | ~$2.5 |
| RDS Backup 7d（DB容量分は無料、超過分のみ） | ~$0 |
| ALB | ~$20 |
| S3 5-10GB | ~$0.5 |
| データ転送 | ~$1-5 |
| **合計** | **~$48/月** |

UAT 期間（1-2ヶ月）終了後は `terraform destroy` で全停止し、$0/月に戻すのを runbook 化する。

### 7.3 ALB を使わない選択肢（参考）

- ALB を使わず EC2 に Elastic IP + Route53 + Caddy/Nginx で TLS 終端する案も技術的には可能（ALB 約 $20/月の節約）
- ただし MVP 期間は無料枠内、ポートフォリオ品質では ALB 構成（ADR 整合）を選ぶ
- §11 ユーザー判断事項として記載

---

## 8. リスクと緩和策

| # | リスク | 影響度 | 緩和策 |
|---|---|---|---|
| 1 | 無料枠誤超過（特に NAT Gateway / Elastic IP / Multi-AZ） | 高 | NAT Gateway を絶対に作らない、EIP を未アタッチで残さない、Budget アラート $5/月、`terraform plan` を必ず確認 |
| 2 | Terraform state 破損 | 中 | S3 versioning 有効、DynamoDB lock、`terraform.tfstate` をコミットしない（.gitignore） |
| 3 | RDS db.t3.micro メモリ不足（1GB） | 中 | UAT 程度の同時接続数なら問題なし。pgxpool の MaxConns を絞る |
| 4 | EC2 t3.micro メモリ不足（1GB）で docker build OOM | 中 | swap 1-2GB を user_data で確保、または GHCR で配布（案A） |
| 5 | デプロイ失敗時のロールバック | 中 | RDS スナップショットを apply 前後で取得、`terraform destroy` 手順を §5.8 に明記 |
| 6 | IAM 権限過多（Terraform 用 AdministratorAccess） | 低（ポートフォリオなら受容） | アクセスキーを使い終わったら無効化。本番運用想定では権限を絞る |
| 7 | 公開 URL のセキュリティ（パスワード総当たり等） | 中 | レート制限はデフォルト値（本番）で運用、`UNAUTH_RATE_LIMIT_PER_MINUTE` を未設定にする |
| 8 | JWT 鍵の漏洩 | 高 | EC2 上 `/etc/expense-saas/keys/` を root:600、ssh 鍵をローカル `~/.ssh/`、tfvars をコミットしない |
| 9 | RDS public access | 高 | `publicly_accessible = false` を明示、SG で EC2 SG のみ許可 |
| 10 | S3 バケット public 化 | 高 | `aws_s3_bucket_public_access_block` で全ブロック、署名付き URL のみ許可 |
| 11 | HTTPS 化忘れ | 中 | ACM 証明書 + Route53 が無いとカスタムドメイン HTTPS は困難。MVP では HTTP のみで運用するか §3.5 で判断 |
| 12 | 実環境時刻ドリフト | 低 | EC2 は AL2023 デフォルトで chrony が動く。RDS も AWS 管理。Phase 7 で `timedatectl` 確認 |
| 13 | RLS ポリシー / `expense_app` ロールの再現 | 高 | scripts/init-db.sh を確認し、prod では migrate up に加えて手動で role / RLS を作成する手順を §5.7 に明記 |

---

## 9. 作業順序（タスクリスト）と担当割り

### 9.1 Phase 0: 事前確認（並列可）

| # | タスク | 担当 | 並列 |
|---|---|---|---|
| 0-1 | PR #144（issue #175 deploy.yml 整合）レビュー & merge | 指揮役 | 並列1 |
| 0-2 | 11-D CONDITIONAL #1（Go 統合テスト再実行） | 指揮役（`/test`） | 並列1 |
| 0-3 | 11-D CONDITIONAL #2（`npm run e2e` 10件 PASS、ローカル） | 指揮役 | 並列1 |
| 0-4 | 11-D CONDITIONAL #4（frontend クリーンビルド） | 指揮役 | 並列1 |
| 0-5 | AWS アカウント / 支払い情報 / Budget アラート $5 設定 | USER | 並列2 |
| 0-6 | IAM ユーザー `terraform-deployer` 作成 + アクセスキー | USER | 並列2 |
| 0-7 | キーペア `expense-saas-portfolio` 作成 | USER | 並列2 |
| 0-8 | ローカル Terraform >= 1.6 インストール | USER | 並列2 |
| 0-9 | `scripts/init-db.sh` 確認、prod 版マイグレーション手順確定（§5.7） | 指揮役（read-only） | 並列1 |

### 9.2 Phase 1: Terraform IaC 作成（サブエージェント委譲可）

| # | タスク | 担当 | 出力 |
|---|---|---|---|
| 1-1 | `expense-saas/infra/terraform/` 配下のファイル骨子作成 | platform-builder | §3.1 のディレクトリ構成 |
| 1-2 | network.tf / security_groups.tf | platform-builder | VPC + 4 subnet + 3 SG |
| 1-3 | ec2.tf / iam.tf | platform-builder | EC2 + instance profile + S3 IAM |
| 1-4 | rds.tf | platform-builder | DB subnet group + RDS instance |
| 1-5 | s3.tf / alb.tf | platform-builder | 領収書バケット + ALB + target group |
| 1-6 | variables.tf / outputs.tf / terraform.tfvars.example | platform-builder | 入出力定義 |
| 1-7 | `terraform fmt -recursive && terraform validate` の検証 | platform-builder | エラー 0 |
| 1-8 | README.md（apply 手順、本書 §5 を抜粋） | platform-builder | runbook |

サブエージェントは `terraform plan` 等の AWS 認証コマンドは実行しない（feedback_subagent_permissions.md）。ローカル静的検証（fmt / validate）のみ。

### 9.3 Phase 2-7: AWS apply 〜 検証（USER 主導）

| # | タスク | 担当 | 備考 |
|---|---|---|---|
| 2-1 | state バケット + DynamoDB lock 作成（§5.3） | USER | 一回限り |
| 2-2 | `terraform init` | USER | |
| 3-1 | `terraform plan -out=tfplan` | USER | 指揮役が出力をレビュー |
| 3-2 | `terraform apply tfplan` | USER | 10-15 分 |
| 3-3 | outputs 取得（ALB DNS、RDS endpoint、S3 bucket） | USER | 指揮役に共有 |
| 4-1 | EC2 ssh 疎通確認 | USER | |
| 4-2 | RDS 疎通確認（EC2 から psql） | USER | |
| 4-3 | migrate up 実行（§5.7） | USER + 指揮役支援 | |
| 4-4 | expense_app ロール / RLS 投入 | USER + 指揮役支援 | |
| 5-1 | Docker イメージ配布（§3.4 で選択した方式） | USER | |
| 6-1 | systemd unit 配置 + 起動 | USER | |
| 6-2 | `docker logs` でアプリ起動成功確認 | USER + 指揮役 | |
| 7-1 | ALB 経由 `/health` 200 OK | 指揮役 | curl |
| 7-2 | 11-D CONDITIONAL #3 時刻ドリフト smoke | USER + 指揮役 | `timedatectl`、`SELECT NOW()` |
| 7-3 | seed 投入 | USER | |
| 7-4 | 手動スモーク（申請→承認→支払） | USER（操作者） + 指揮役（観察・記録） | §6.3 |
| 7-5 | 11-E チケット §「実施ログ」記録 | 指揮役 | |

### 9.4 Phase 9: 11-F 引き継ぎ準備（指揮役）

| # | タスク | 担当 |
|---|---|---|
| 9-1 | 公開 URL / UAT アカウント / デプロイ日時 / コミット SHA を 11-E チケットに記録 | 指揮役 |
| 9-2 | 既知非ブロッカー（11-D N-01〜N-05、post-MVP issue 一覧）を引き継ぎリストに整理 | 指揮役 |
| 9-3 | 11-F でユーザーが確認すべき重点箇所を列挙（11-D §7 ベース） | 指揮役 |

### 9.5 並列化可能な箇所

- Phase 0 の指揮役タスク（0-1, 0-2, 0-3, 0-4, 0-9）と USER の AWS 準備（0-5〜0-8）は完全並列
- Phase 1 内では 1-2 と 1-4（network と RDS は依存少ない）を別エージェントで並列も可能だが、ファイル数が少ないので 1 エージェントで直列の方がコスト効率は良い

---

## 10. 完了判定（11-E チケット完了条件との対応）

| 11-E 完了条件 | 本書での確認手段 |
|---|---|
| デプロイ済み環境で第三者がアクセスできる | §6.1 ALB DNS 経由で `/health` 200、§6.3 ログイン画面到達 |
| ヘルスチェックエンドポイントが応答する | §6.1 `curl http://<alb-dns>/health` で `{"status":"ok"}` |
| 主要フロー1本（申請→承認→支払）が通る | §6.3.2 手動スモーク 6 ステップ全て期待結果通り |

加えて、11-D CONDITIONAL の 4 条件全てを Phase 0 + Phase 7 で消化済みであること。

---

## 11. ユーザー判断が必要な事項

以下は本計画で「方針提示のみ・最終決定は USER」とした事項。Phase 1 着手前にユーザーに確認する。

| # | 判断事項 | 選択肢 | 推奨 |
|---|---|---|---|
| Q1 | Docker イメージ配布方式 | 案A: GHCR / 案B: EC2 上 build / 案C: ECR | 初回は案B、運用化で案A |
| Q2 | TLS（HTTPS）化 | (a) ALB DNS のまま HTTP のみ / (b) Route53 ドメイン取得 + ACM + ALB HTTPS | (a)（無料枠優先、ポートフォリオなら可） |
| Q3 | Multi-AZ | Single-AZ（無料枠） / Multi-AZ（有料） | Single-AZ（ADR-0004 で受容） |
| Q4 | Terraform 用 IAM 権限 | AdministratorAccess（簡易） / 必要権限のみ（厳密） | AdministratorAccess（ポートフォリオ簡略化） |
| Q5 | UAT 後の停止運用 | 維持 / `terraform destroy` で停止 | 13ヶ月目以降は destroy |
| Q6 | deploy.yml の if: false 解除 | 11-E で実施 / 11-E 完了後に別チケット | 別チケット（11-E スコープ外） |
| Q7 | カスタムドメイン | 取得しない / 取得する（年 $12 程度 + Route53 $0.5/月） | 取得しない |

---

## 12. 工数見積り

| カテゴリ | 内訳 | 想定時間 |
|---|---|---|
| サブエージェント時間（platform-builder） | Phase 1: Terraform 雛形作成・静的検証 | 1.5-2.5h |
| 指揮役時間（Phase 0 事前消化） | Go 統合テスト / e2e ローカル / FE クリーンビルド / scripts 確認 / PR #144 レビュー | 1.5-2.5h |
| 指揮役時間（Phase 3-7 支援） | plan レビュー、ログ確認、スモーク観察、チケット記録 | 1.5-2h |
| ユーザー手動時間（Phase 0 AWS 準備） | アカウント / IAM / キーペア / Terraform install | 0.5-1h |
| ユーザー手動時間（Phase 2-3 apply） | state 作成 / init / plan / apply | 0.5-1h（apply 待ちは 10-15 分） |
| ユーザー手動時間（Phase 4-6 デプロイ） | ssh / migrate / role 作成 / Docker / systemd / 起動確認 | 1.5-2.5h |
| ユーザー手動時間（Phase 7 スモーク） | seed / 手動 6 ステップ / 不具合切り分け | 0.5-1.5h |
| バッファ（不具合切り分け・リトライ） | apply 失敗 / マイグレ失敗 / Docker run 失敗 等 | 1-3h |
| **合計** | | **約 8-16h** |

### 12.1 クリティカルパス

```
[CP] Phase 1 Terraform 雛形 (1.5-2.5h)
   -> Phase 2 state init (0.2h)
   -> Phase 3 apply (10-15 分の待ち + 監視)
   -> Phase 4 migrate (0.3-0.5h)
   -> Phase 5-6 Docker 配布 + 起動 (0.5-1.5h、案B build なら +0.5h)
   -> Phase 7 スモーク (0.5-1.5h)
```

ボトルネック:

1. **Phase 3 apply の待ち時間（特に RDS 作成 10 分前後）** — 並行作業不可、ただし長時間ブロックではない
2. **Phase 5 案B（EC2 build）の場合の docker build** — t3.micro メモリ 1GB で OOM リスク。swap 設定が必要
3. **Phase 7 不具合切り分け** — RDS 接続失敗 / RLS 設定漏れ / SG 設定漏れ等が起きると一気に+2-3h

---

## 13. このあと指揮役が確認すべき事項（チェックリスト）

- [ ] §11 Q1〜Q7 の判断をユーザーに確認する
- [ ] PR #144（issue #175）のステータス確認、Phase 0 と並行で merge する
- [ ] `expense-saas/scripts/init-db.sh` を読み、prod 用マイグレーション + role 作成手順を §5.7 にフィードバック
- [ ] 11-E チケットを Phase 別の小チケット（11-E-1 Terraform 雛形 / 11-E-2 AWS apply / 11-E-3 デプロイ + 検証）に分割するかを判断
- [ ] Phase 0 の 11-D CONDITIONAL 事前消化を即時着手

---

## 14. 参考リンク

- 11-E チケット: `dev-journal/progress-management/tickets/step11/11-E-deploy.md`
- 11-D 再レビューレポート: `dev-journal/progress-management/step11-d-review-report.md`
- ADR-0004（インフラ選定 + ポートフォリオ対応）: `dev-journal/deliverables/docs/30_arch/adr/0004-infra.md`
- リリース手順: `dev-journal/deliverables/docs/70_operations/release.md`
- 環境設定: `dev-journal/deliverables/docs/70_operations/env_config.md`
- アーキテクチャ: `dev-journal/deliverables/docs/30_arch/architecture.md`
- deploy.yml: `expense-saas/.github/workflows/deploy.yml`
- issue #175: `dev-journal/issues/open/175-deploy-yml-e2e-job-misalignment.md`
