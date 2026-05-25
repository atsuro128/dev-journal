# issue #187 実装計画書（architect）

| 項目 | 値 |
|------|----|
| issue | #187 `expense-saas/infra/terraform/`: SSM Session Manager + SSM Parameter Store 対応 |
| 上流 | issue #186 UD-1=A（SSM 接続）/ UD-5=B（SSM Parameter Store SecureString） |
| 作業ブランチ | `step11/issue-187-terraform-ssm`（origin/master 4812a01 から分岐） |
| 担当 | architect（計画策定のみ）/ 実装は platform-builder を想定 |
| 出力先 PR | `expense-saas` リポジトリ |
| 作成日 | 2026-05-25 |

---

## 0. 重大リスクの先出し（ユーザー判断 P-0 として扱う）

### R-0 機密情報の git 履歴混入リスク

`expense-saas/infra/terraform/terraform.tfvars` に以下が **平文** で含まれていることを確認した（`.gitignore` 除外済みだが、本 PR で SSM 化する以上、本ファイルの扱い方針も同時確定が必要）。

- `db_password`
- `expense_owner_db_password`
- `expense_app_db_password`
- `jwt_private_key_pem`（RSA 秘密鍵 PEM 全文、複数行）
- `jwt_public_key_pem`（RSA 公開鍵 PEM 全文）

**SSM Parameter Store 移行後の tfvars 扱い**: 上記 5 変数のうち **JWT PEM 2 件のみ** を SSM Parameter Store の正本に移行するため tfvars から削除する。**DB password 3 件は RDS の `master_password` / `expense_owner` / `expense_app` ロール作成のため Terraform variable として必須** であり、tfvars には残置する（正本は引き続き tfvars だが `.gitignore` 対象）。これにより JWT PEM 平文の git 履歴混入リスクを潰しつつ、RDS 作成の依存も維持する。

→ **P-0**: JWT PEM 2 件を tfvars から削除する移行手順をどう設計するか（後述 §3 P-0）。

---

## 1. 現状把握サマリ

### 1.1 ファイル別の現状記述

| ファイル | 現状記述（行番号） | issue #186 設計確定との差分 |
|---|---|---|
| `ec2.tf` | L26 `key_name = var.key_pair_name` / L41-49 user_data に PEM/CORS/proxy 等を template 変数として直接埋め込み | `key_name` 削除、user_data から PEM 引数削除、IAM ロールに SSM ポリシー追加（iam.tf 経由） |
| `iam.tf` | L1-22 `ec2_role`（S3 アクセス用 ARN 信頼）/ L36-61 `ec2_s3_policy`（S3 GetObject/PutObject/DeleteObject/ListBucket）。SSM / KMS は **一切なし** | `AmazonSSMManagedInstanceCore` をアタッチ、`ssm:GetParameters` カスタムポリシー追加、`kms:Decrypt`（`alias/aws/ssm`）追加 |
| `security_groups.tf` | L80-114 `ec2` SG。L95-101 で **22 番ポートを `var.allowed_ssh_cidr` から許可** | L95-101 の SSH ingress ルール全削除（443 outbound は L103-109 で既に許可済み、変更不要） |
| `user_data.sh.tpl` | L29-60: `/etc/expense-saas/keys/private.pem` `public.pem` `app.env` を template 変数から直接 here-doc 書き出し。L51-52 で DATABASE_URL を `CHANGEME_AFTER_MIGRATE` プレースホルダで書く | env_config.md §5.3.3 のスクリプト（`aws ssm get-parameters --with-decryption` で 4 パラメータ取得 → `/etc/expense-saas/app.env` と `/etc/expense-saas/keys/*.pem` 生成）に置換 |
| `variables.tf` | L21-25 `key_pair_name`（default `expense-saas-portfolio`）/ L27-30 `allowed_ssh_cidr`（required）/ L56-66 `jwt_private_key_pem` `jwt_public_key_pem`（sensitive） | `key_pair_name` / `allowed_ssh_cidr` を削除（または非推奨化）、`jwt_*_pem` を削除（SSM 経由になるため）、`ssm_parameter_path_prefix` 等を追加 |
| `terraform.tfvars.example` | L21-25 で `key_pair_name` / `allowed_ssh_cidr` 設定例 / L9-10 で PEM 投入手順 | SSH 関連を削除、SSM への投入手順（`aws ssm put-parameter --type SecureString`）に置換 |
| `terraform.tfvars` (gitignored) | **R-0 参照**: 5 sensitive 変数の平文値あり | tfvars 削除 → SSM 移行手順をリリースノートに記述 |
| `outputs.tf` | L11-14 `ec2_public_ip`（SSH アクセス用と書かれている） | description 文言を更新（「SSM Session Manager 接続時の `--target` には `aws_instance.app.id` を使う」と注記、または `ec2_instance_id` output を追加） |
| `locals.tf` / `network.tf` / `alb.tf` / `cloudfront.tf` / `rds.tf` / `s3.tf` / `backend.tf` / `providers.tf` | SSM / SSH / シークレット注入には**非関与** | 変更なし（干渉は §6 リスクで確認） |

### 1.2 issue #186 設計確定（env_config.md §5）との差分一覧

| 設計確定（env_config.md / runbook.md / release.md） | 現状 Terraform | 差分の規模 |
|---|---|---|
| **SSM パラメータ命名**: `/expense-saas/{env}/database/url` `/database/app_url` `/jwt/private_key` `/jwt/public_key`（env_config.md §5.3.2） | パラメータ未作成。user_data で template 変数から直書き | 大（user_data 全面書き換え + SSM 命名規約定義） |
| **IAM**: `ssm:GetParameters` を `arn:aws:ssm:ap-northeast-1:<account>:parameter/expense-saas/prod/*` に限定 + `kms:Decrypt` を `alias/aws/ssm` に限定（env_config.md §5.3.5） | `ec2_s3_policy` のみ。SSM/KMS なし | 中（iam.tf に 2 ポリシー追加） |
| **接続**: `aws ssm start-session --target i-xxxxx`（runbook.md §3 / release.md §4.1） | SSH 22 番開放 + key_pair | 中（SG + ec2.tf + variables.tf） |
| **systemd EnvironmentFile**: `/etc/expense-saas/app.env`（権限 `0640`、`root:appgroup`）（env_config.md §5.3.3） | 同じパスに書き出しているが、権限 `0600`、owner デフォルト | 軽微（権限と owner 修正） |
| **AMI**: SSM Agent プリインストール済み AMI（Amazon Linux 2023 はデフォルトで含む） | Amazon Linux 2023（既に AL2023 を使用、追加 install 不要） | 確認のみ |

---

## 2. 実装スコープ詳細

### 2.1 `iam.tf`（追加のみ）

**追加リソース 3 件**:

```hcl
# A. AWS マネージドポリシーのアタッチ（SSM Session Manager + SSM Agent 基本機能）
resource "aws_iam_role_policy_attachment" "ec2_ssm_core" {
  role       = aws_iam_role.ec2_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

# B. SSM Parameter Store 読み取り用カスタムポリシー
resource "aws_iam_role_policy" "ec2_ssm_parameters" {
  name = "${local.prefix}-ec2-ssm-parameters-policy"
  role = aws_iam_role.ec2_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        # B-01=案a（jq による単一 GetParameters 呼び出し）採用のため、Action は GetParameters のみ
        Action = ["ssm:GetParameters"]
        Resource = [
          # `arn:aws:ssm:<region>:<account>:parameter/expense-saas/*`
          # 注: env_config.md §5.3.5 は `/expense-saas/<env>/*` (prod 固定) と記載されているが、
          #     本 Terraform 構成では env を増やすときに IAM 修正不要にするため `/expense-saas/*`
          #     (全 env 共通) で発行する。env_config.md §5.3.5 との差分は後続 ADR で明示する
          "arn:aws:ssm:${var.aws_region}:*:parameter${var.ssm_parameter_path_prefix}/*"
        ]
      }
    ]
  })
}

# C. KMS 復号権限（SecureString 復号、AWS マネージドキー alias/aws/ssm）
#    P-2=A 採用時のみ。P-2=B（カスタマー管理 KMS）採用時はキー ARN を変数化
resource "aws_iam_role_policy" "ec2_kms_decrypt" {
  name = "${local.prefix}-ec2-kms-decrypt-policy"
  role = aws_iam_role.ec2_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["kms:Decrypt"]
        Resource = "arn:aws:kms:${var.aws_region}:*:alias/aws/ssm"
      }
    ]
  })
}
```

**完了条件**: `terraform validate` で構文 OK、`terraform plan` で 3 リソース新規作成のみが diff として現れる。

### 2.2 `ec2.tf`（属性削除 + templatefile 引数削減）

| 削除 | 変更 |
|---|---|
| L26 `key_name = var.key_pair_name` | 行削除 |
| L41-49 `templatefile()` 引数 | `jwt_private_key_pem` / `jwt_public_key_pem` を削除、`environment` だけ残す（user_data から SSM を引く際の env 名として必要） |

変更後の templatefile 引数（**P-6=B 採用前提**、5 項目 → 6 項目に再編。機密 2 項目を削除し非機密 2 項目を temple 引数のまま維持）:

```hcl
user_data = templatefile("${path.module}/user_data.sh.tpl", {
  environment               = var.environment
  ssm_parameter_path_prefix = var.ssm_parameter_path_prefix
  aws_region                = var.aws_region
  image_tag                 = var.image_tag
  # P-6=B 採用前提: 非機密項目（CORS, proxy count）は SSM 化せず Terraform variable のまま埋め込む
  cors_allowed_origins      = var.cors_allowed_origins
  trusted_proxy_count       = var.trusted_proxy_count
})
```

→ **P-6**: 非機密項目（`cors_allowed_origins`, `trusted_proxy_count`）も SSM 化するか、Terraform variable のままにするか。本計画は **B（variable のまま）を推奨** し §2.4 スニペットも B 前提で記述。P-6=A に変更する場合は §2.4 の `${cors_allowed_origins}` `${trusted_proxy_count}` を `aws ssm get-parameters` 取得値に差し替える。

### 2.3 `security_groups.tf`（22 番ルール削除）

L94-101 の SSH ingress ブロック全削除:

```diff
-  # SSH: 自宅 IP のみ許可
-  ingress {
-    description = "SSH from allowed CIDR"
-    from_port   = 22
-    to_port     = 22
-    protocol    = "tcp"
-    cidr_blocks = [var.allowed_ssh_cidr]
-  }
```

**egress は変更不要**（L103-109 で `0.0.0.0/0` 全許可済み。SSM エンドポイント疎通用の 443 outbound はこれでカバー）。

### 2.4 `user_data.sh.tpl`（全面書き換え）

env_config.md §5.3.3 のスクリプトを正本として転記。主要骨子:

```bash
#!/bin/bash
# EC2 user_data: SSM Parameter Store からシークレット取得 + Docker + systemd 設定
set -euo pipefail

# B-03 対策 案c（防御 2 段）:
#   (1) /var/log/user-data.log を最初に 0600 root:root で touch して permission を絞る
#       （cloud-init のデフォルト 0644 を上書き）
#   (2) 機密フェッチ・書き出し区間のみ /dev/null にリダイレクトし、平文出力を残さない
install -m 0600 -o root -g root /dev/null /var/log/user-data.log
exec > /var/log/user-data.log 2>&1

ENV_NAME="${environment}"
REGION="${aws_region}"
PARAM_PREFIX="${ssm_parameter_path_prefix}"
SECRET_DIR="/etc/expense-saas"
KEY_DIR="$${SECRET_DIR}/keys"

# 1. swap 設定（現状維持。t3.micro OOM 回避）
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab

# 2. Docker / jq インストール（jq は B-01=案a の JSON parse に必須）
#    SSM Agent は Amazon Linux 2023 にプリインストール済み
dnf install -y docker git jq
systemctl enable --now docker
systemctl enable --now amazon-ssm-agent  # 確認のため明示有効化（idempotent）

# 3. ディレクトリ準備
mkdir -p "$${KEY_DIR}"
chmod 750 "$${SECRET_DIR}" "$${KEY_DIR}"
getent group appgroup >/dev/null || groupadd -r appgroup

# 4. SSM Parameter Store から SecureString を一括取得（KMS 復号付き、B-01=案a: JSON + jq）
#    fail-fast: aws ssm 取得失敗時は set -e で user_data 全体が失敗する
#    B-03 対策: 機密値が PARAMS / KEY 変数に乗る区間は /dev/null にリダイレクトし
#               万一 set -x が混入しても /var/log/user-data.log に平文が残らないようにする
exec 3>&1 4>&2
exec > /dev/null 2>&1

PARAMS=$(aws ssm get-parameters \
  --names \
    "$${PARAM_PREFIX}/$${ENV_NAME}/database/url" \
    "$${PARAM_PREFIX}/$${ENV_NAME}/database/app_url" \
    "$${PARAM_PREFIX}/$${ENV_NAME}/jwt/private_key" \
    "$${PARAM_PREFIX}/$${ENV_NAME}/jwt/public_key" \
  --with-decryption \
  --region "$${REGION}" \
  --output json)

# jq で Name -> Value を引く（PEM の改行を含む多行 SecureString も無傷で取り出せる）
get_param() {
  printf '%s' "$${PARAMS}" | jq -r --arg n "$1" '.Parameters[] | select(.Name == $n) | .Value'
}

DATABASE_URL=$(get_param "$${PARAM_PREFIX}/$${ENV_NAME}/database/url")
APP_DATABASE_URL=$(get_param "$${PARAM_PREFIX}/$${ENV_NAME}/database/app_url")
JWT_PRIVATE_KEY=$(get_param "$${PARAM_PREFIX}/$${ENV_NAME}/jwt/private_key")
JWT_PUBLIC_KEY=$(get_param "$${PARAM_PREFIX}/$${ENV_NAME}/jwt/public_key")

# 取得失敗時の fail-fast（empty チェックは値そのものを出力しないので stderr に出して OK）
missing=()
for v in DATABASE_URL APP_DATABASE_URL JWT_PRIVATE_KEY JWT_PUBLIC_KEY; do
  if [ -z "$${!v}" ]; then
    missing+=("$$v")
  fi
done

# 5. JWT 鍵ペアをファイルに書き出し（PEM の多行は printf '%s\n' で完全保持）
printf '%s\n' "$${JWT_PRIVATE_KEY}" > "$${KEY_DIR}/private.pem"
printf '%s\n' "$${JWT_PUBLIC_KEY}"  > "$${KEY_DIR}/public.pem"
chmod 600 "$${KEY_DIR}"/*.pem
chown root:appgroup "$${KEY_DIR}"/*.pem

# 6. /etc/expense-saas/app.env を生成（systemd EnvironmentFile）
cat > "$${SECRET_DIR}/app.env" <<EOF
PORT=8080
LOG_LEVEL=info
DATABASE_URL=$${DATABASE_URL}
APP_DATABASE_URL=$${APP_DATABASE_URL}
JWT_PRIVATE_KEY_PATH=/app/keys/private.pem
JWT_PUBLIC_KEY_PATH=/app/keys/public.pem
S3_BUCKET=expense-saas-receipts-$${ENV_NAME}
AWS_REGION=$${REGION}
CORS_ALLOWED_ORIGINS=${cors_allowed_origins}
TRUSTED_PROXY_COUNT=${trusted_proxy_count}
EOF
chmod 640 "$${SECRET_DIR}/app.env"
chown root:appgroup "$${SECRET_DIR}/app.env"

# 7. 機密値を環境からクリア
unset PARAMS DATABASE_URL APP_DATABASE_URL JWT_PRIVATE_KEY JWT_PUBLIC_KEY

# 8. ログ出力を /var/log/user-data.log に復帰（機密区間終了）
exec 1>&3 2>&4
exec 3>&- 4>&-

# fail-fast のエラー出力は復帰後の log に書き出す
if [ "$${#missing[@]}" -gt 0 ]; then
  echo "ERROR: SSM parameter(s) empty: $${missing[*]}" >&2
  exit 1
fi

# 9. systemd unit 配置（現状の expense-saas.service を維持）
cat > /etc/systemd/system/expense-saas.service <<'SYSTEMD_EOF'
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
SYSTEMD_EOF

systemctl daemon-reload
%{ if image_tag != "" ~}
docker pull "${image_tag}"
docker tag "${image_tag}" expense-saas:portfolio
systemctl enable --now expense-saas
%{ endif ~}

echo "user_data completed: $(date)"
```

**注**: Terraform `templatefile` の `${...}` と bash の `${...}` は競合するため、bash の変数参照は `$${...}` でエスケープする（上記サンプルは反映済み）。

### 2.5 `variables.tf`（追加 + 削除）

**削除**:

| 変数 | 削除理由 |
|---|---|
| `key_pair_name` | SSH 廃止により不要（P-5 で「フラグ切り替え」採用時のみ deprecated 維持） |
| `allowed_ssh_cidr` | 同上 |
| `jwt_private_key_pem` | SSM Parameter Store 経由に移行（正本は SSM） |
| `jwt_public_key_pem` | 同上 |

**追加**:

```hcl
variable "ssm_parameter_path_prefix" {
  description = "SSM Parameter Store のパス接頭辞（env_config.md §5.3.2）。例: /expense-saas"
  type        = string
  default     = "/expense-saas"
}
```

**P-1=A（Terraform で SSM パラメータ管理）採用時はさらに以下を追加**:

```hcl
variable "ssm_db_url_value" {
  description = "SSM Parameter Store に格納する DATABASE_URL の値（sensitive、tfvars または TF_VAR で渡す）"
  type        = string
  sensitive   = true
}
# 以下、ssm_app_db_url_value / ssm_jwt_private_key_value / ssm_jwt_public_key_value も同様
```

**P-1=B（手動作成 + data source 参照）採用時は値変数は追加せず、`data "aws_ssm_parameter"` を参照（ただし user_data 内 aws CLI で取得するため Terraform data source も不要なケースが多い）**。

### 2.6 SSM Parameter リソース定義（P-1 の結果次第）

**新規ファイル候補**: `ssm.tf`（P-1=A 採用時のみ）

```hcl
# P-1=A 採用時の Terraform 管理パターン
resource "aws_ssm_parameter" "database_url" {
  name        = "${var.ssm_parameter_path_prefix}/${var.environment}/database/url"
  type        = "SecureString"
  value       = var.ssm_db_url_value
  description = "DB owner role connection URL"
  key_id      = "alias/aws/ssm"
  tags        = local.common_tags
}

# database_app_url / jwt/private_key / jwt/public_key も同様に 4 リソース定義
```

**P-1=B 採用時**: このファイルを作成せず、手動投入手順を `README.md` に記述する。

**P-1=C（ハイブリッド）採用時**: `aws_ssm_parameter` リソースで `lifecycle { ignore_changes = [value] }` を付けて「枠だけ作る」設計。

---

## 3. ユーザー判断項目（事前確定）

| ID | 論点 | 推奨 | 影響範囲 |
|----|------|------|---------|
| **P-0** | tfvars 中の JWT PEM 2 件の扱い（DB password 3 件は Terraform 必須のため残置） | A（JWT PEM 削除 + 手動 SSM 投入） | terraform.tfvars 移行手順 |
| **P-1** | SSM パラメータの作成方法 | **B**（手動 put-parameter + Terraform data source 不要） | ssm.tf 有無、variables.tf 増減 |
| **P-2** | KMS キー選定 | **A**（`alias/aws/ssm`） | iam.tf の kms ポリシー Resource |
| **P-3** | 既存 EC2 への適用方法 | **A**（terraform apply で再作成、`user_data_replace_on_change = true` を本 PR で導入） | release 手順、ec2.tf |
| **P-4** | AWS 実環境疎通検証の扱い | **B**（Step 11-E 実 apply 時に切り出し） | 受け入れ基準の分類 |
| **P-5** | SSH 関連変数の後方互換 | **A**（完全削除） | variables.tf の削除規模 |
| **P-6** | 非機密項目（CORS, proxy count）の SSM 化 | **B**（Terraform variable のまま） | user_data 引数の構成 |
| **P-8** | env_config.md §5.3.3 の awk 欠陥（B-01 と同一）を本 PR で同時修正するか別 issue 化するか | **A**（本 PR で env_config.md §5.3.3 / §5.3.5 を同時修正） | env_config.md §5.3.3 / §5.3.5、コミット計画 |

### P-0: tfvars 中の JWT PEM 2 件の扱い（DB password 3 件は残置確定）

| 案 | メリット | デメリット |
|----|----|----|
| ★ A. **JWT PEM 2 件のみ tfvars から削除、SSM に手動 put-parameter** | JWT PEM 平文の git 履歴混入リスク解消、正本が SSM に一本化、RDS 作成依存も維持 | 移行時の手作業（4 パラメータの `aws ssm put-parameter`） |
| B. tfvars 保持 + Terraform で SSM に流し込み | tfvars 一箇所管理 | 「正本が tfvars と SSM の二重」になりローテーションが破綻、平文 tfvars 残存 |
| C. DB password も含めて全削除（PEM もローテして SSM に手動投入、DB 接続も SSM 経由） | 履歴クリーン | PEM 再生成 + RDS の master_password を Terraform 外で管理する仕組みが追加で必要（MVP には過剰） |

**推奨**: A。本 PR の趣旨（脱平文）と整合。**tfvars に残るのは `db_password` `expense_owner_db_password` `expense_app_db_password`（RDS の master_password / アプリ用ロール作成のため Terraform variable として必須）の 3 件のみ**。`jwt_private_key_pem` / `jwt_public_key_pem` の 2 件を tfvars から削除し、SSM Parameter Store に正本を移す。

**注**: `db_password` 等は RDS の `master_password` および PostgreSQL ロール作成（`expense_owner` / `expense_app`）に必要なため Terraform variable のまま残す。SSM 化対象は **DB 接続文字列（DATABASE_URL / APP_DATABASE_URL）と JWT PEM 2 件** の計 4 パラメータ。

### P-1: SSM パラメータの作成方法

| 案 | メリット | デメリット |
|----|----|----|
| A. Terraform で `aws_ssm_parameter` 管理 | IaC 一貫性、`terraform apply` だけで完結 | SecureString の `value` を tfvars に書く必要があり、平文残存リスクが復活（または `TF_VAR_*` 環境変数で渡す手間） |
| ★ **B. 手動作成 + data source 不要** | シークレット完全分離、tfvars がクリーン、ローテーションが Terraform 非依存 | 手順依存（README に明記必要）、初回投入時の手作業 |
| C. ハイブリッド（`lifecycle.ignore_changes = [value]`） | 枠は Terraform、値は手動 | 概念が複雑、ハマりやすい |

**推奨**: B。R-0 のリスク（平文残存）を最小化する観点で最も安全。`aws ssm put-parameter --type SecureString` の手順を tfvars.example または README に記載すれば再現性も担保できる。

### P-2: KMS キー選定

| 案 | メリット | デメリット |
|----|----|----|
| ★ **A. AWS マネージドキー `alias/aws/ssm`** | 無料、追加設定なし、env_config.md §5.3.5 と整合 | キーポリシーの細かい制御不可（MVP では不要） |
| B. カスタマー管理 KMS（CMK） | キーポリシー細制御、Key rotation 90 日有効化 | $1/月、ローテーション設計が追加で必要 |

**推奨**: A。env_config.md §5.3.5 が既に `alias/aws/ssm` 前提で書かれている。Portfolio スコープで CMK は過剰。

### P-3: 既存 EC2 への適用方法

| 案 | メリット | デメリット |
|----|----|----|
| ★ **A. terraform apply で EC2 再作成** | クリーンな状態（user_data 完全反映）、再現性担保 | ダウンタイム数分、SSM パラメータ事前投入が必要、`ec2_public_ip` 変化 |
| B. インスタンスプロファイル付け替え + 手動 systemctl restart | ダウンタイム最小 | user_data の冪等性未検証、状態が不明確、`apply` 後に手動操作が混入 |

**推奨**: A。本 PR は Portfolio スコープのため、ダウンタイム数分は許容範囲。**P-3=A 採用の必須付随変更として、本 PR の T2（ec2.tf）で `user_data_replace_on_change = true` を 1 行追加する**（R-3 対策。現状 `false` のままだと user_data 変更時に EC2 が再作成されず、SSM 移行が反映されない）。**ただし、SSM パラメータの事前投入が前提条件**（投入前に apply すると user_data の `aws ssm get-parameters` が失敗 → EC2 起動失敗）。

**手順順序の確定**（P-1=B + P-3=A 採用時）:

```
1. SSM パラメータ 4 件を手動投入（aws ssm put-parameter --type SecureString）
2. terraform plan で diff 確認
3. terraform apply（EC2 再作成 + IAM ポリシー追加 + SG 22 番削除）
4. SSM Session Manager で接続 → docker run + systemctl enable
```

### P-4: AWS 実環境疎通系の受け入れ基準の扱い

issue #187 受け入れ基準 7 項目のうち以下は AWS 実 apply 必須:

- `aws ssm start-session --target i-xxxxx` で EC2 接続
- SSH 不可確認
- SSM Parameter Store から取得 → systemd EnvironmentFile 確認
- CloudTrail SSM 接続ログ確認
- `aws ec2 describe-instance-attribute --attribute userData` 確認
- アプリ起動・ヘルスチェック疎通

| 案 | メリット | デメリット |
|----|----|----|
| A. 本 PR スコープに含める（apply 必須） | 1 PR で完結 | Step 11-E 実 apply セッションが必要、PR が大きくなる |
| ★ **B. Step 11-E 実 apply 時に切り出し** | PR レビューは静的検証（plan / validate）のみで完結、適用は別セッション | 受け入れ基準を「コード変更（PR）」と「実 apply 検証」の 2 つに分割管理が必要 |

**推奨**: B。本 PR は **静的検証（`terraform validate` + `terraform plan` の正常終了 + コードレビュー）まで** をスコープとし、AWS 実 apply とその検証は Step 11-E 実 apply セッションでチェックリスト消化する。受け入れ基準のマッピングは §5 参照。

### P-5: SSH 関連変数の後方互換

| 案 | メリット | デメリット |
|----|----|----|
| ★ **A. 完全削除（`key_pair_name`, `allowed_ssh_cidr`）** | コードシンプル、SSH 戻し不可で誤操作防止 | SSH に戻す場合は revert PR が必要 |
| B. フラグで切り替え可能化（`enable_ssh = false` デフォルト） | 緊急時 SSH 再有効化が可能 | コード複雑化、フラグ管理コスト |

**推奨**: A。issue #186 で SSM への移行が確定済みであり、後戻り想定なし。緊急時の AWS Console からの直接操作（EC2 Connect 等）で代替可能。

### P-6: 非機密項目（CORS, proxy count）の SSM 化

| 案 | メリット | デメリット |
|----|----|----|
| A. SSM 化（全環境変数の正本を SSM に集約） | 設定一元化 | 非機密値で SecureString を使う必要なし、Terraform variable のままで十分 |
| ★ **B. Terraform variable のまま** | 現状維持、env_config.md §3.2 と整合 | CORS 変更時に terraform apply 必要 |

**推奨**: B。SSM 化の本来目的（機密管理）から外れる項目を混ぜると設計が散漫になる。env_config.md §3.2 アプリケーション設定差分でも CORS は「アプリケーション設定」扱い。

### P-8: env_config.md §5.3.3 / §5.3.5 の波及修正範囲

**背景**: B-01 で指摘された awk による PEM 多行値復元失敗は、Terraform 側 `user_data.sh.tpl` だけでなく **設計書本体 `dev-journal/deliverables/docs/06_env_config.md` §5.3.3 の参照スクリプト雛形にも同じ欠陥が存在する**。また B-02 と関連して §5.3.5 の IAM Action 記載（`ssm:GetParameters` 単一）と本 PR の Action 採用方針（`ssm:GetParameters` 一本に統一）が一致したため、§5.3.5 自体に直接の修正は不要だが、§5.3.5 の Resource 表記が `/expense-saas/<env>/*`（prod 固定）になっているのに対し本 Terraform 構成は `/expense-saas/*`（全 env 共通）を採用するため、差分注記が必要。

| 案 | メリット | デメリット |
|----|----|----|
| ★ A. **本 PR で env_config.md §5.3.3（awk → jq）と §5.3.5（Resource 表記差分の注記）を同時修正** | 設計書と実装が同時に正しくなり、後続の参照（runbook.md など）が安全。レビュー 1 回で完結 | PR スコープが Terraform 単独 → Terraform + ドキュメント に拡大。`dev-journal` 側にも diff が乗る |
| B. 別 issue として切り出し、本 PR は Terraform のみ修正 | 本 PR のスコープが純粋（Terraform だけ）。レビュー粒度が小さい | 設計書に欠陥が残った状態がしばらく続き、別人が env_config.md を参照して同じ awk 実装を再生産するリスク |

**推奨**: A。理由:
- B-01 / B-02 は **設計書（env_config.md）と実装（Terraform）の両方に同じ根本欠陥が存在**しており、片方だけ直すと設計書が正本として腐る
- 修正規模は env_config.md §5.3.3 のスクリプト雛形（awk → jq）と §5.3.5 の Resource 注記（`/expense-saas/<env>/*` と `/expense-saas/*` の差分注記）のみで小さい
- 本 PR のコミットを 4 件目（`docs(env-config): fix awk PEM multiline issue + clarify SSM resource scope (issue #187, B-01/B-02 波及)`）として追加すれば分離は維持可能

→ B 採用時は P-8 別 issue を本計画書承認時に同時起票し、本 PR にリンクすること。

---

## 4. タスク分解と依存関係

### 4.1 ファイル単位タスク（T1-T8）

| ID | 対象ファイル | 内容 | 想定差分 | 担当 |
|----|-------------|------|---------|------|
| T1 | `iam.tf` | §2.1 の 3 リソース追加 | +30 行 | platform-builder |
| T2 | `ec2.tf` | `key_name` 削除、templatefile 引数調整（5 → 6 項目、機密 2 件削除 / CORS 等 2 件追加）、**`user_data_replace_on_change = true` を 1 行追加（R-3 / P-3=A 対策）** | -1 行 +/-数行 +1 行 | platform-builder |
| T3 | `security_groups.tf` | L94-101 SSH ingress 削除 | -8 行 | platform-builder |
| T4 | `user_data.sh.tpl` | §2.4 全面書き換え | -60 行 +90 行 | platform-builder |
| T5 | `variables.tf` | `key_pair_name`/`allowed_ssh_cidr`/`jwt_*_pem` 削除、`ssm_parameter_path_prefix` 追加 | -25 行 +6 行 | platform-builder |
| T6 | `terraform.tfvars.example` | SSH 関連削除、SSM 投入手順コメント追加 | -10 行 +20 行 | platform-builder |
| T7 | `outputs.tf` | `ec2_public_ip` description 更新、必要なら `ec2_instance_id` output 追加 | +/-5 行 | platform-builder |
| T8 | `README.md`（infra/terraform/） | SSM パラメータ事前投入手順、apply 手順を SSM 前提で更新 | +30 行 | platform-builder |

**P-1=A 採用時のみ追加**:

| T9 | `ssm.tf`（新規） | §2.6 の 4 リソース定義 | +50 行 | platform-builder |

**P-8=A 採用時のみ追加**（推奨）:

| T10 | `dev-journal/deliverables/docs/06_env_config.md` | §5.3.3 のスクリプト雛形を awk → jq に書き換え（B-01 波及）、§5.3.5 の Resource 表記（`/expense-saas/<env>/*` prod 固定）と本 Terraform 構成（`/expense-saas/*` 全 env 共通）の差分注記を追加（B-02 波及） | +20 行 -10 行 | platform-builder |

### 4.2 依存関係（並列 / 直列）

全タスクが単一ファイル編集で完結し、コードレベルの相互依存は弱い。ただし `terraform validate` / `plan` 検証はすべて完了後に一括で行う必要があるため、**1 ブランチ・1 エージェントで直列実行を推奨**:

```
T5 (variables.tf) → T1 (iam.tf) → T2 (ec2.tf) → T3 (sg.tf) → T4 (user_data) → T6 (tfvars.example) → T7 (outputs.tf) → T8 (README) → validate/plan
```

理由:
- ファイル単位で並列化しても、Terraform は全体整合性を `validate` で確認するため、最終的に統合作業が必要
- 1 PR = 1 機能（SSM 移行）なので分割の価値が薄い
- 並列化のオーバーヘッド（worktree 複数管理、コンフリクト解消）に見合わない

### 4.3 エージェント割当

**platform-builder を推奨**:

- 全タスクが Terraform / bash テンプレート / IaC 設定で、`.claude/agents/platform-builder.md` の責務（インフラ / CI / プラットフォーム基盤）に合致
- backend-developer は Go アプリケーション層が主領域で、Terraform は守備範囲外

---

## 5. 受け入れ基準のマッピング

issue #187 §受け入れ基準の 7 項目を、本 PR スコープ vs Step 11-E 実 apply スコープに分類:

| # | 受け入れ基準 | 本 PR スコープ | Step 11-E 実 apply スコープ |
|---|--------------|---------------|---------------------------|
| 1 | `aws ssm start-session --target i-xxxxx` で EC2 接続できる | — | ✓ 実 apply 後に検証 |
| 2 | SSH（22 番）での接続が不可（SG ルール削除確認） | ✓ `terraform plan` で SG diff 確認 | ✓ 実 apply 後に `nmap -p 22` 等で検証 |
| 3 | SSM Parameter Store から DB パスワードが起動時に取得され systemd EnvironmentFile に書き出されている | ✓ user_data の bash スクリプト構造をコードレビュー | ✓ 実 apply 後に `sudo cat /etc/expense-saas/app.env` で検証 |
| 4 | tfvars / user_data 出力内に平文シークレットが残っていない | ✓ tfvars と user_data.sh.tpl を grep + **B-03 対策（`/var/log/user-data.log` を 0600 root:root 化 + 機密フェッチ区間 `/dev/null` リダイレクト）を §2.4 に反映済みでコードレビュー** | ✓ 実 apply 後に `aws ec2 describe-instance-attribute --attribute userData` で検証 + **`sudo ls -l /var/log/user-data.log`（0600 root:root）** + **`sudo grep -E 'BEGIN (RSA )?PRIVATE KEY' /var/log/user-data.log` が 0 件** |
| 5 | CloudTrail に SSM 接続ログが記録されている | — | ✓ 実 apply 後に検証 |
| 6 | `aws ec2 describe-instance-attribute --attribute userData` で平文シークレットなし | ✓ user_data.sh.tpl 静的レビュー | ✓ 実 apply 後に上記コマンドで検証 |
| 7 | アプリ起動・ヘルスチェック疎通確認 OK | — | ✓ 実 apply 後に `curl /health` で検証 |

**本 PR の必須完了条件**:

1. `terraform fmt -recursive` がパス
2. `terraform validate`（init 済み環境で）がパス
3. `terraform plan` の diff が「3 IAM リソース追加 / 1 SG ルール削除 / 1 EC2 user_data 変更（taint） / 4 variables 削除 / 1 variable 追加」のみ
4. `expense-saas/infra/terraform/terraform.tfvars.example` および `user_data.sh.tpl` を全文 grep して平文 PEM / パスワード文字列が 0 件
5. レビュアー（codex）レビューを通過

---

## 6. リスクと留意事項

### 6.1 重大リスク

| ID | リスク | 緩和策 |
|----|-------|--------|
| R-0 | tfvars に sensitive 値（DB password 3 種 + JWT PEM 2 種）平文存在 | P-0=A で JWT PEM のみ tfvars 削除、DB password は Terraform 必須のため保持。`.gitignore` 含有を再確認。**git log を grep して tfvars が誤って commit されていないか確認** |
| R-1 | SSM パラメータ未投入のまま apply すると user_data 失敗 → EC2 起動失敗 | P-3 手順順序（SSM 投入 → apply）を README に明記。`terraform apply` 前のチェックリスト化 |
| R-2 | `templatefile` の bash `${...}` エスケープ漏れ → terraform validate エラー | `$${...}` 二重エスケープを徹底。`terraform validate` で構文確認 |
| R-3 | 既存 EC2 が `user_data_replace_on_change = false` のため user_data 変更で**再作成されない** | **本 PR の T2（ec2.tf）で `user_data_replace_on_change = true` を 1 行追加し恒久対策とする**（P-3=A 推奨採用に伴う必須付随変更、W-04 対応）。初回 apply 時の `terraform taint aws_instance.app` は不要になる |
| R-4 | RDS のパブリック IP に依存していたヘルスチェック手順が SSH 廃止で実行不可 | runbook.md は既に SSM 経由手順に書き直し済み（#186 完了）。本 PR は実態を追従するだけ |

### 6.2 既存リソースとの干渉確認

| ファイル | 干渉懸念 | 確認結果 |
|---|---|---|
| `cloudfront.tf` | SSM 関連リソースなし | ✓ 干渉なし |
| `alb.tf` | SSM 関連リソースなし | ✓ 干渉なし |
| `rds.tf` | `db_password` variable 依存 | ✓ 干渉なし（db_password は SSM 化対象外） |
| `s3.tf` | `s3_bucket_suffix` のみ依存 | ✓ 干渉なし |
| `backend.tf` / `providers.tf` | 構成的に独立 | ✓ 干渉なし |
| `outputs.tf` | `ec2_public_ip` 残置 | description 更新（SSH 廃止注記）と `ec2_instance_id` 追加を推奨 |

### 6.3 SSM Agent 関連の前提確認

| 項目 | 確認結果 |
|---|---|
| Amazon Linux 2023 における SSM Agent | **プリインストール済み**（AL2 / AL2023 とも `amazon-ssm-agent` パッケージが含まれる） |
| systemd unit 自動起動 | AL2023 デフォルトで enable 済み。user_data で `systemctl enable --now amazon-ssm-agent` を明示 idempotent で実行することを推奨（環境差吸収） |
| インスタンスプロファイル経由の認証 | `AmazonSSMManagedInstanceCore` ポリシーが付与されていれば SSM Agent が自動で AWS API 認証 |
| VPC エンドポイント（SSM / SSMMessages / EC2Messages） | 現状の VPC は **public subnet** で 443 outbound 全許可のため、インターネット経由で SSM エンドポイントに到達可能。VPC エンドポイント不要 |

### 6.4 監査・ログ留意

- CloudTrail で `StartSession` / `GetParameters` API がログされる（既定有効）。**CloudTrail を有効化していない場合は別 issue として起票が必要**（現状 Terraform に CloudTrail リソースなし → ユーザー判断 P-7 候補）
- → **P-7 候補**: CloudTrail 有効化を本 PR スコープに含めるか、別 issue で扱うか
  - **指揮役推奨**: 別 issue（本 PR スコープが膨らみすぎるため）

### 6.5 命名規約上の留意

- env_config.md §5.3.2 で確定済みのパス: `/expense-saas/{env}/database/url` `/database/app_url` `/jwt/private_key` `/jwt/public_key`
- 現状 `var.environment = "portfolio"` のため、パスは `/expense-saas/portfolio/...` となる。env_config.md は `stg` `prod` を例示しているが、ADR-0004 で「env_config.md の prod は使わず portfolio とする」と定めているため、`portfolio` で正しい
- IAM ポリシーの Resource 限定は `/expense-saas/${var.environment}/*` ではなく `/expense-saas/*` に広げるか議論余地あり → 推奨は **環境名展開しない**（env を増やすときに IAM 修正不要にするため `/expense-saas/*` 推奨）
- **env_config.md §5.3.5 との表記差分**: env_config.md §5.3.5 は `arn:aws:ssm:...:parameter/expense-saas/<env>/*`（prod 固定で記述）となっているが、本 Terraform 構成では `/expense-saas/*`（全 env 共通）を発行する。この差分は P-8=A 採用時に env_config.md §5.3.5 に注記として追記する。P-8=B 採用時は本計画書の本セクションと §2.1 コード内コメントで差分明示にとどめ、別 issue で env_config.md 修正を切り出す

---

## 7. コミット計画

単一 PR で以下 3〜4 コミットに分割（レビューしやすさ優先）:

| # | コミットメッセージ | 含まれるタスク |
|---|-------------------|-----------------|
| 1 | `feat(infra): add SSM Session Manager + Parameter Store IAM policies (issue #187)` | T1 + T5（variables.tf 追加分のみ） |
| 2 | `refactor(infra): switch user_data to fetch secrets from SSM Parameter Store (issue #187)` | T2（`user_data_replace_on_change = true` 追加含む） + T4 + T5（jwt_*_pem 削除） |
| 3 | `chore(infra): remove SSH key/SG/variables now superseded by SSM (issue #187)` | T3 + T5（key_pair_name/allowed_ssh_cidr 削除） + T6 + T7 + T8 |
| 4 | `docs(env-config): fix awk PEM multiline issue + clarify SSM resource scope (issue #187, B-01/B-02 波及)` | **P-8=A 採用時のみ** T10 |

**PR タイトル**: `infra(terraform): migrate EC2 access + secrets to SSM (issue #187)`

**注**: コミット 4 は `expense-saas` リポジトリではなく `dev-journal` リポジトリへのコミット。PR 構成について:
- P-8=A 採用の場合: `expense-saas` PR（コミット 1-3）+ `dev-journal` への直 push または別 PR（コミット 4）の 2 リポジトリ同時更新
- P-8=B 採用の場合: `expense-saas` PR（コミット 1-3）のみ、env_config.md 修正は別 issue として後続対応

---

## 8. 次アクション（指揮役向け）

1. 本計画書をユーザーに提示し **P-0 〜 P-6 および P-8** を確定する
2. 確定後、計画書を `dev-journal/progress-management/working/` から `task-plans/` 等の正式な保管場所に移すか、issue #187 コメントに転記
3. platform-builder にブランチ `step11/issue-187-terraform-ssm` を割り当てて実装委譲
4. 実装完了後、reviewer / codex でレビュー
5. PR マージ後、Step 11-E 実 apply セッションで受け入れ基準 1/5/7（実 apply 系）を消化

---

## 付録 A: 検証コマンド集（platform-builder 用）

```bash
# 1. terraform fmt
terraform -chdir=expense-saas/infra/terraform fmt -recursive -check

# 2. terraform init（既存 .terraform.lock.hcl を使用）
terraform -chdir=expense-saas/infra/terraform init -backend=false

# 3. terraform validate
terraform -chdir=expense-saas/infra/terraform validate

# 4. terraform plan（要 AWS credentials。本 PR ではスキップ可。Step 11-E で実施）
terraform -chdir=expense-saas/infra/terraform plan

# 5. 平文シークレット残存チェック
#    W-05 対応: `terraform.tfvars`（gitignored だがローカルには存在）も対象に含める。
#    P-0=A 採用後は tfvars 内に JWT PEM が 0 件であることを必ず確認する
#    （DB password 3 件は Terraform 必須のため残置が正常）
grep -rE "BEGIN (RSA )?PRIVATE KEY|BEGIN PUBLIC KEY" \
  expense-saas/infra/terraform/*.tf* \
  expense-saas/infra/terraform/*.tpl \
  expense-saas/infra/terraform/*.example \
  expense-saas/infra/terraform/terraform.tfvars 2>/dev/null
# 期待: 0 件（tfvars に JWT PEM が残っていれば P-0=A が未完）

grep -E "jwt_.*_pem\s*=\s*\".*[A-Za-z0-9]" \
  expense-saas/infra/terraform/terraform.tfvars.example \
  expense-saas/infra/terraform/terraform.tfvars 2>/dev/null
# 期待: 0 件（コメントは可、値は不可。tfvars / tfvars.example 双方）

# 注: db_password 系は tfvars 内に残置が仕様（RDS 必須）。grep 対象から除外
```

## 付録 B: SSM パラメータ手動投入コマンド（運用者向け、P-1=B 採用時）

```bash
# DATABASE_URL（owner）
aws ssm put-parameter \
  --name "/expense-saas/portfolio/database/url" \
  --type SecureString \
  --key-id alias/aws/ssm \
  --value "postgres://expense_owner:<password>@<rds-endpoint>:5432/expense_saas?sslmode=require" \
  --region ap-northeast-1

# APP_DATABASE_URL（app）
aws ssm put-parameter \
  --name "/expense-saas/portfolio/database/app_url" \
  --type SecureString \
  --key-id alias/aws/ssm \
  --value "postgres://expense_app:<password>@<rds-endpoint>:5432/expense_saas?sslmode=require" \
  --region ap-northeast-1

# JWT 秘密鍵
aws ssm put-parameter \
  --name "/expense-saas/portfolio/jwt/private_key" \
  --type SecureString \
  --key-id alias/aws/ssm \
  --value "$(cat /path/to/private.pem)" \
  --region ap-northeast-1

# JWT 公開鍵
aws ssm put-parameter \
  --name "/expense-saas/portfolio/jwt/public_key" \
  --type SecureString \
  --key-id alias/aws/ssm \
  --value "$(cat /path/to/public.pem)" \
  --region ap-northeast-1
```

---

## 改版履歴

### v2 (2026-05-25) — reviewer CONDITIONAL PASS 指摘の反映

**Blocker 修正（3 件）**:

- **B-01 反映**: §2.4 `user_data.sh.tpl` の SSM 値取得を `aws ssm get-parameters --output text` + awk から **`--output json` + `jq -r`（案 a）** に切り替え。PEM 多行値を無傷で取り出せる実装に変更。採用根拠は「AL2023 に jq プリインストール（dnf で明示 install も §2.4 step2 に追加）、API 呼び出し 1 回で済む、IAM が `ssm:GetParameters` 一本で env_config.md §5.3.5 と整合」。これに伴い §2.4 step2 に `dnf install -y ... jq` を追記
- **B-02 反映**: §2.1 IAM ポリシーの Action を `["ssm:GetParameters", "ssm:GetParameter"]` から `["ssm:GetParameters"]` のみに修正（B-01=案 a の単一 GetParameters 呼び出しに整合）。Resource コメントを env_config.md §5.3.5（prod 固定）との差分を明示する内容に書き換え（W-03 対応も同時）
- **B-03 反映**: §2.4 に **案 c（防御 2 段）** を反映。(1) `install -m 0600 -o root -g root /dev/null /var/log/user-data.log` で log ファイル permission を絞り、(2) 機密フェッチ・書き出し区間を `exec > /dev/null 2>&1` で /dev/null にリダイレクトし区間終了後 `exec 1>&3 2>&4` で復帰。§5 受け入れ基準 #4 のマッピングにも本対応の検証手順を追記

**Warning 修正（5 件）**:

- **W-01**: §0 R-0 / §3 P-0 の文言を「JWT PEM 2 件のみ削除、DB password 3 件は Terraform 必須のため残置」で統一。要約表の P-0 行も同じ表現に書き換え
- **W-02**: §2.2 templatefile 引数表に「**P-6=B 採用前提**」を明記。`cors_allowed_origins` `trusted_proxy_count` を引数として明示表記し、P-6=A への切替手順を注記
- **W-03**: §2.1 IAM ポリシーのコメントを `/expense-saas/*`（全 env 共通）と env_config.md §5.3.5（prod 固定）の差分明示に書き換え。§6.5 末尾にも同じ差分注記を追記
- **W-04**: §3 P-3 解説と §6.1 R-3 緩和策に「本 PR の T2（ec2.tf）で `user_data_replace_on_change = true` を 1 行追加」を明記。§4.1 T2 タスク差分行数も `+1 行` を追加
- **W-05**: §付録 A の grep 対象に `expense-saas/infra/terraform/terraform.tfvars` を追加（gitignored だがローカルチェック必須）。db_password 系は意図的に対象外とする旨も注記

**新規追加項目**:

- **P-8（ユーザー判断項目を追加）**: env_config.md §5.3.3 の awk 欠陥（B-01 と同根）を本 PR で同時修正するか別 issue 化するか。**推奨は A（本 PR で env_config.md §5.3.3 / §5.3.5 を同時修正）**。理由: 設計書と実装に同根欠陥が同時に残ると正本が腐るため。修正規模は小さく、コミット 4 件目（`docs(env-config): ...`）として分離可能
- **§7 コミット計画**: P-8=A 採用時の 4 件目コミット（`docs(env-config): fix awk PEM multiline issue + clarify SSM resource scope (issue #187, B-01/B-02 波及)`）と 2 リポジトリ同時更新の注記を追加
- **§4.1 タスク表**: P-8=A 採用時のみ追加する **T10**（env_config.md §5.3.3 / §5.3.5 修正）を追加
