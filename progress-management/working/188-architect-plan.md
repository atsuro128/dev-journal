# issue #188 実装計画書: 監視ロググループ命名残置リネーム + Docker awslogs ドライバ実装

- 作成日: 2026-05-25（v2 改版: 2026-05-25）
- 作業ブランチ: `step11/issue-188-cloudwatch-logs`
- 分岐元: `expense-saas` の `origin/master` HEAD = `445240b infra(terraform): migrate EC2 access + secrets to SSM (#187)`（issue #187 / PR #152 マージスカッシュ）
- 作成者: architect（計画策定のみ。実装は別エージェントに委譲）

---

## 0. 役割と本書のスコープ

本書は **計画策定のみ**。ユーザー判断項目（P-0〜P-6）を確定したのち、designer / platform-builder に実装委譲する想定。コードは書かない。

---

## 1. 現状把握サマリ

### 1.1 設計 vs 実装の乖離マッピング

| 項目 | 設計（monitoring.md / ADR-0005） | 実装（PR #152・issue #187 マージ後） | 乖離 |
|---|---|---|---|
| ログ転送ドライバ | `docker run --log-driver=awslogs` を `systemd unit` に指定（L47, L367, UD-2=B） | `user_data.sh.tpl` L138-142 の `ExecStart` に `--log-driver` / `--log-opt` **一切なし**（デフォルト `json-file`） | **致命的（CloudWatch 転送ゼロ）** |
| ロググループ命名 | `/ecs/expense-saas/api`（monitoring.md L624、ECS 名残） | `aws_cloudwatch_log_group` リソース **未定義**（grep 0 件） | 設計も実装も未確定 |
| ロググループ retention | 30 日（ADR-0005, monitoring.md §9.1） | Terraform 未定義 | 未実装 |
| IAM 権限 | （明示記述なし）— awslogs 動作には `logs:CreateLogStream` / `logs:PutLogEvents` 必須 | `iam.tf` に `logs:*` 権限 **一切なし**（SSM Core + SSM GetParameters + S3 のみ） | **致命的（IAM 権限不足）** |

### 1.2 経緯の確認

issue #186 Phase 1-4 は「設計成果物の ECS → EC2 言い換え」に集中しており、「設計が指す awslogs ドライバの **実装**」は別 issue 切り出し対象として認識されないまま落ちていた。issue #188 は当初 reviewer の I-01 として「monitoring.md L624 リテラル変更だけ」のスコープで起票されていたが、調査の結果以下が判明:

- monitoring.md は §1（L17, L35, L47, L367, L527）で **awslogs ドライバを採用する** と明記
- にも関わらず実装は **何もしていない**（json-file ドライバ = ホスト内 `/var/lib/docker/containers/.../<id>-json.log` に滞留するだけ）
- runbook.md §3 / §4 の「CloudWatch Logs Insights で〜」手順は **そもそも転送先がない** ため動作しない

つまり issue #188 は実質「設計と実装が両方未完だった awslogs 実装を完成させる」issue であり、純粋なリネーム issue ではない。

### 1.3 user_data 実装の現状（重要部位）

```bash
# user_data.sh.tpl L128-147 抜粋
ExecStart=/usr/bin/docker run --name expense-saas \
  --env-file /etc/expense-saas/app.env \
  -v /etc/expense-saas/keys:/app/keys:ro \
  -p 8080:8080 \
  expense-saas:portfolio
```

→ `--log-driver` / `--log-opt` 不在 = json-file ドライバが暗黙適用される。EC2 内のディスクに留まり CloudWatch に届かない。

---

## 2. ユーザー判断項目（事前確定）

| ID | 論点 | 推奨案 | 影響範囲 |
|---|---|---|---|
| P-0 | 既存 AWS 上の旧 LogGroup `/ecs/expense-saas/api` の扱い | **c**（無視。AWS 実 apply 時に手動判断） | Step 11-E 実 apply |
| P-1 | ロググループのパス階層 | **b**（`/expense-saas/${var.environment}/api`） | Terraform / monitoring.md / runbook.md |
| P-2 | awslogs-stream 命名 | **a**（`${INSTANCE_ID}/api`） | user_data |
| P-3 | `logs:CreateLogGroup` 権限の要否 | **a**（不要。Terraform 先行作成） | iam.tf |
| P-4 | AWS 実環境疎通検証の扱い | **b**（Step 11-E 切り出し） | スコープ境界 |
| P-5 | nginx 等の追加 LogGroup | **a**（本 issue では扱わない） | スコープ境界 |
| P-6 | env_config.md への新規変数追加 | **a**（既存 `LOG_LEVEL` で十分。追加なし） | env_config.md |

### P-0: 既存 AWS 上の旧 LogGroup の扱い

| 案 | 内容 | メリット | デメリット |
|---|---|---|---|
| a | AWS console で手動削除（Terraform 管理外） | 簡単。Terraform state 汚染しない | 手動オペが残る |
| b | Terraform で旧名 `legacy_ecs` リソースを `prevent_destroy=false` で import → 別 PR で destroy | 監査痕跡が残る | 2 段階 PR 必要。state 操作の事故リスク |
| ★ c | 旧 LogGroup は無視（実 apply 時に **AWS console で存在確認** → 手動削除 or 放置判断） | 本 PR スコープを純粋な「forward 実装」に保てる | 旧データの扱いを Step 11-E に先送り |

**推奨理由（c）**: そもそも旧 LogGroup が AWS に **存在するかも未確認**（issue #187 までの実装で `aws_cloudwatch_log_group` は一度も定義されていない＝Terraform 経由で旧 LogGroup を作ったことがない）。`docker awslogs` ドライバが `auto-create` した形跡もない（`--log-driver=awslogs` 自体が未設定）。よって **旧 LogGroup は AWS 上に存在しない可能性が高い**。本 PR では確認スコープには入れず、Step 11-E 実 apply 直前に `aws logs describe-log-groups --log-group-name-prefix /ecs/` で 1 行確認すれば足りる。

### P-1: ロググループのパス階層

| 案 | 名前 | メリット | デメリット |
|---|---|---|---|
| a | `/expense-saas/api` | シンプル | env 増設時に衝突。stg/prod を作る場合に変更必要 |
| ★ b | `/expense-saas/${var.environment}/api` | env_config.md §2 で **stg/prod** が既に定義済み（将来構築時の推奨値）。SSM Parameter Store 命名 `/expense-saas/{env}/database/*` と階層が完全に揃う | パス階層が 1 段深くなる |

**推奨理由（b）**: env_config.md §5.3.2 が既に `/expense-saas/{env}/...` を採用しており、整合性が高い。issue #188 §提案の推奨案 B（`/expense-saas/api`）はあくまで「ECS 名残を外す」観点であり env 階層は議論されていなかった。SSM パス命名規約と揃える方が運用上一貫する。**ユーザー判断必須**（issue 起票案からの逸脱のため）。

### P-2: awslogs-stream 命名

| 案 | 値 | メリット | デメリット |
|---|---|---|---|
| ★ a | `${INSTANCE_ID}/api` | AWS 公式推奨。インスタンス再作成（user_data 変更で `user_data_replace_on_change = true`）時に stream が分離され、追跡しやすい | stream 名がランダム |
| b | `api` 固定 | シンプル | インスタンス入れ替え時に新旧ログが同じ stream に混在 |
| c | `${hostname}/api` | hostname も追跡可能 | hostname は instance-id ベースなので a と実質同等 |

**推奨理由（a）**: awslogs ドライバには予約変数 `{instance_id}` がない代わりに `awslogs-stream` を user_data 内で `IMDSv2` から取得して埋め込むか、`awslogs-create-group=false` + `tag` で代用する方法がある。最も標準的なのは **user_data で IMDSv2 から `instance-id` を取得 → templatefile 引数として渡す** か、**`tag` オプションを使用しない素直な `awslogs-stream` 直書き**。実装詳細は §3.4 で確定する。

### P-3: `logs:CreateLogGroup` 権限の要否

| 案 | 内容 | メリット | デメリット |
|---|---|---|---|
| ★ a | 不要（Terraform で LogGroup を先行作成、awslogs ドライバは既存 LogGroup に書くだけ） | 最小権限。LogGroup の retention/タグを Terraform で確実に管理 | Terraform apply 順序の依存制御必要（`depends_on`） |
| b | 必要（awslogs ドライバの auto-create 保険） | EC2 起動時に LogGroup 不在でも自走可能 | 余計な権限。retention 設定が auto-create では「期限なし」になり ADR-0005 違反 |

**推奨理由（a）**: ADR-0005 で retention 30 日が確定しているため、auto-create は **retention 設定漏れリスク**を持ち込む。Terraform 先行作成 + `depends_on = [aws_cloudwatch_log_group.app]` で堅く制御する。

### P-4: AWS 実環境疎通検証の扱い

| 案 | 内容 |
|---|---|
| a | 本 PR スコープに含める（terraform apply 実行 + CloudWatch Logs にログ到達確認） |
| ★ b | Step 11-E 実 apply 時に切り出し（PR #152 / issue #187 と同じ方針） |

**推奨理由（b）**: 本 PR は Terraform validate / fmt / plan までで完結させ、実 apply は Step 11-E まで温存（コスト・運用負荷の観点）。issue #187 も同方針で merge 済み。

### P-5: nginx などの追加 LogGroup

| 案 | 内容 |
|---|---|
| ★ a | 本 issue では扱わない（API ログのみ） |
| b | nginx access log も同 PR で追加 |

**推奨理由（a）**: スコープ拡大回避。そもそも現実装に nginx は無い（Docker コンテナ直接 8080 開放、ALB が前段）。

### P-6: env_config.md への新規変数追加

| 案 | 内容 |
|---|---|
| ★ a | 追加なし（既存 `LOG_LEVEL` で十分） |
| b | `LOG_GROUP_NAME` 等を新規定義 |

**推奨理由（a）**: アプリは stdout に書くだけで、転送先（LogGroup 名）はインフラ側の関心事。アプリは LogGroup 名を知る必要がない。

---

## 3. 実装スコープ詳細

### 3.1 Terraform: `cloudwatch.tf` 新規作成

新規ファイル `expense-saas/infra/terraform/cloudwatch.tf` を作成（既存ファイルへの追加ではなく **集約 1 ファイル** とする。理由: 関連 IAM・user_data 変更との対応関係が明示的になり、将来 Alarm / Metric Filter 追加時の置き場も明確）。

```hcl
# aws_cloudwatch_log_group: アプリログの転送先（Docker awslogs ドライバから書き込み）
# - retention 30 日（ADR-0005、monitoring.md §9.1）
# - 命名: /expense-saas/${var.environment}/api（P-1=b、env_config.md §5.3.2 と階層整合）
# - name には local.prefix を流用しない（I-01: local.prefix は "-" 区切りのフラット命名規約で
#   階層 "/" を含まない設計。本 LogGroup は SSM パス命名 /expense-saas/{env}/... と階層整合させたい
#   ため、明示的に "/" 区切りの独自命名とする）
resource "aws_cloudwatch_log_group" "app" {
  name              = "/${var.project_name}/${var.environment}/api"
  retention_in_days = 30

  tags = merge(local.common_tags, {
    Name = "${local.prefix}-app-logs"
  })
}
```

### 3.2 Terraform: `iam.tf` 拡張

`aws_iam_role_policy.ec2_cloudwatch_logs`（新規）を追加:

```hcl
# CloudWatch Logs 書き込み権限（Docker awslogs ドライバ用）
# - logs:CreateLogStream / PutLogEvents のみ（P-3=a: LogGroup は Terraform 先行作成）
# - Resource は対象 LogGroup ARN に限定（最小権限）
resource "aws_iam_role_policy" "ec2_cloudwatch_logs" {
  name = "${local.prefix}-ec2-cloudwatch-logs-policy"
  role = aws_iam_role.ec2_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogStream",
          "logs:PutLogEvents",
        ]
        Resource = [
          "${aws_cloudwatch_log_group.app.arn}:*",
        ]
      }
    ]
  })
}
```

- `Resource` は `<arn>:*` 形式で stream をワイルドカード許可
- `logs:CreateLogGroup` は **意図的に除外**（P-3=a）

### 3.3 Terraform: `ec2.tf` の `templatefile()` 引数追加

本 issue で追加する引数は **`log_group_name` のみ**。`aws_region` は **既存 ec2.tf L43 で既に `aws_region = var.aws_region` を渡している**ため新規追加不要、user_data 側でも既存の `${aws_region}` 参照をそのまま流用する（W-02）。

```hcl
user_data = templatefile("${path.module}/user_data.sh.tpl", {
  # 既存引数（aws_region 含む 7 個）...
  log_group_name = aws_cloudwatch_log_group.app.name  # 新規追加（これ 1 個のみ）
})
```

加えて `depends_on` に LogGroup と IAM policy attachment を追加:

```hcl
depends_on = [
  aws_iam_role_policy_attachment.ec2_ssm_core,
  aws_iam_role_policy.ec2_ssm_parameters,
  aws_iam_role_policy.ec2_cloudwatch_logs,   # 追加
  aws_cloudwatch_log_group.app,              # 追加
]
```

### 3.4 user_data: awslogs ドライバ設定

#### 3.4.1 heredoc 切替方式の確定（B-02 対応）

systemd の specifier `%i` は **instance template の identifier** であり EC2 の instance-id ではない。よって `%i` は使えない。本計画書では実装の一意化のため、以下の方針を **計画書側で確定** し platform-builder の判断余地を排除する。

**事前検査結果（user_data.sh.tpl 現状 L129-147 の systemd unit 中身）**:

```
[Unit] / [Service] / [Install] ブロック全体を Read で確認
→ `$` を含む箇所は ゼロ（変数参照が一切ない、純粋な静的テキスト）
```

→ heredoc を `<<'SYSTEMD_EOF'`（クォート付き = 展開なし）から `<<SYSTEMD_EOF`（クォートなし = 展開あり）に変更しても、**既存の意図しない箇所で展開が走るリスクはない**（展開対象が皆無のため副作用ゼロ）。

**採用方式: 方式 X（heredoc クォート除去 + bash 変数展開）**

- 現状 L129 の `cat > /etc/systemd/system/expense-saas.service <<'SYSTEMD_EOF'` を `cat > /etc/systemd/system/expense-saas.service <<SYSTEMD_EOF` に変更
- systemd unit 内では `${INSTANCE_ID}` と書き、bash heredoc 展開で実値に置換される
- templatefile レイヤから渡される `${log_group_name}` / `${aws_region}` は terraform templatefile で先に展開済みのため、heredoc に到達時は既に静的文字列

**不採用方式: 2 段階生成方式（sed 置換）**

- `<<'SYSTEMD_EOF'` を維持しつつ `__INSTANCE_ID__` プレースホルダを書き、生成後 `sed -i "s|__INSTANCE_ID__|$${INSTANCE_ID}|g" /etc/systemd/system/expense-saas.service` で置換
- 事前検査で副作用ゼロが確認できたため、方式 X で必要十分。2 段階生成は採用しない（複雑性回避）

#### 3.4.2 user_data 実装

`user_data.sh.tpl` 冒頭近く（**L8 直後 = 機密区間より前**、I-02 対応）に IMDSv2 取得を挿入:

```bash
# IMDSv2 で instance-id を取得（awslogs-stream 命名用、P-2=a）
# 配置: L8 直後（機密区間より前。INSTANCE_ID は非機密のため unset 対象外で扱える、R-6 対応）
TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
INSTANCE_ID=$(curl -sH "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)
```

`app.env` 生成への追記は **既存 L100-111 の heredoc 内に組み込む**（I-02 対応、独立 echo を追加しない）。ただし方式 X 採用により systemd unit が直接 `${INSTANCE_ID}` を参照できるため、`app.env` への書き出しは **必須ではない**。アプリケーションが知る必要がないため、`app.env` 追記は行わない方針とする。

#### 3.4.3 systemd unit の `ExecStart`

```ini
# heredoc を <<SYSTEMD_EOF（クォートなし）に変更したため、${INSTANCE_ID} は bash 展開される
# ${log_group_name} / ${aws_region} は templatefile が先に展開済み（heredoc 到達時は静的文字列）
ExecStart=/usr/bin/docker run --name expense-saas \
  --log-driver=awslogs \
  --log-opt awslogs-region=${aws_region} \
  --log-opt awslogs-group=${log_group_name} \
  --log-opt awslogs-stream=${INSTANCE_ID}/api \
  --log-opt awslogs-create-group=false \
  --env-file /etc/expense-saas/app.env \
  -v /etc/expense-saas/keys:/app/keys:ro \
  -p 8080:8080 \
  expense-saas:portfolio
```

templatefile レイヤ（`user_data.sh.tpl` のソース）では `awslogs-stream=$${INSTANCE_ID}/api` と書く（templatefile の `$$` エスケープで `${INSTANCE_ID}` がそのまま残る → bash heredoc 段で展開される）。

`%i` 表記は **一切使わない**。

`json` フォーマット指定（`awslogs-multiline-pattern` 等）は不要。slog の JSON 出力は 1 行 1 オブジェクトのため Logs Insights が自動 parse する。

### 3.5 設計書更新

#### `monitoring.md` 更新

| 行 | 現状 | 変更後 |
|---|---|---|
| L35 | `EC2 t3.micro（ADR-0004 §ポートフォリオ対応）` | **不変**（W-03: 既に正しい記述。誤って書き換えない） |
| L47 | （変更なし）| 「systemd unit の `docker run --log-driver=awslogs --log-opt awslogs-group=<確定値> --log-opt awslogs-stream=<instance-id>/api` 指定」と具体化 |
| L367 | `/ecs/expense-saas/api` 想定の Logs Insights クエリ言及箇所 | 新ロググループ名 `/expense-saas/${environment}/api` に置換 |
| §9.1（L622-624 付近） | `/ecs/expense-saas/api` 単一行のみ | 新名 `/expense-saas/${environment}/api` に置換 + LogGroup 命名規約・stream 命名規約を 1 段落追加（env 階層採用の理由 = SSM パス階層と整合） |

**改修対象範囲（W-03 明示）**: L47 / L367 / §9.1 の 3 箇所。L35 は不変。
**T5 完了条件（W-03、§5 自主基準 #11 に追加）**: `grep -n "/ecs/expense-saas/api" /root-project/dev-journal/deliverables/docs/50_detail_design/monitoring.md` が **0 件**。

#### `env_config.md` 更新

- §3.2 アプリケーション設定差分テーブルに `CloudWatch Logs LogGroup` 行を追加（dev: なし / stg: `/expense-saas/stg/api` / prod: `/expense-saas/prod/api`）

#### `runbook.md` 更新

- §3 / §4 の Logs Insights クエリは LogGroup 名を明記していないため、**変更不要**（Insights では LogGroup 選択は UI 操作で行う前提）
- ただし冒頭か §1 参照表のどこかに「ロググループ: `/expense-saas/portfolio/api`（monitoring.md §9.1 参照）」と 1 行追記推奨

#### `release.md` 更新

- §5.1 即時確認に「CloudWatch Logs `/expense-saas/portfolio/api` に新インスタンスからのログが到達している」を 1 項目追加

#### `architecture.md` 更新

- 対象パス: `/root-project/dev-journal/deliverables/docs/30_arch/architecture.md`（W-01: パス揺れ防止のため絶対パス明示。`30_architecture/` は存在しない）
- §7 監視・ログ方針サマリーに「ロググループ命名: `/{project_name}/{env}/api`」を 1 行追記

#### `ADR-0005` 扱い

- **更新しない**（決定時点記録、issue #186 §1.3 でも対象外指定。本書は「ECS 自動転送」と書いてあるが歴史記録として保存）

### 3.6 `terraform.tfvars.example` / `README.md` の更新

- `terraform.tfvars.example`: 変更不要（`LOG_GROUP_NAME` は variable 化しないため、P-6=a）
- `README.md`: §10 ヘルスチェック確認の後に「§ ログ確認」セクションを追加し `aws logs tail /expense-saas/portfolio/api --follow` を 1 行例示

---

## 4. タスク分解と依存関係

### 4.1 ファイル別タスク

| # | エージェント | 対象ファイル | 内容 | 依存 |
|---|---|---|---|---|
| T1 | platform-builder | `expense-saas/infra/terraform/cloudwatch.tf`（新規） | `aws_cloudwatch_log_group "app"` 定義 | P-1, P-3 確定後 |
| T2 | platform-builder | `expense-saas/infra/terraform/iam.tf` | `ec2_cloudwatch_logs` policy 追加 | T1（ARN 参照のため） |
| T3 | platform-builder | `expense-saas/infra/terraform/ec2.tf` | `templatefile()` 引数追加 + `depends_on` 拡張 | T1, T2 |
| T4 | platform-builder | `expense-saas/infra/terraform/user_data.sh.tpl` | IMDSv2 で instance-id 取得 + systemd unit に `--log-driver=awslogs` 追加 | T3（引数授受の整合） |
| T5 | designer | `dev-journal/deliverables/docs/50_detail_design/monitoring.md` | L624 リネーム + §9.1 加筆 + §2.1/§設計方針表の具体化 | P-1, P-2 確定後 |
| T6 | designer | `dev-journal/deliverables/docs/70_operations/env_config.md` | §3.2 表に CloudWatch Logs LogGroup 行追加 | P-1 確定後 |
| T7 | designer | `dev-journal/deliverables/docs/70_operations/runbook.md` | §1 に LogGroup 参照を 1 行追記 | P-1 確定後 |
| T8 | designer | `dev-journal/deliverables/docs/70_operations/release.md` | §5.1 にログ到達確認項目追加 | P-1 確定後 |
| T9 | designer | `dev-journal/deliverables/docs/30_arch/architecture.md` | §7 にロググループ命名規約 1 行追記 | P-1 確定後 |
| T10 | platform-builder | `expense-saas/infra/terraform/README.md` | §10 後にログ確認例追加 | T1-T4 完了後 |
| T11 | platform-builder | 検証 | `terraform fmt -recursive`, `terraform validate`, `terraform plan` までを実行（**apply はしない、P-4=b**） | T1-T10 完了後 |
| T12 | architect / reviewer | issue #188 受け入れ基準セルフチェック + PR 作成 | grep 整合確認 | 全 T 完了後 |

### 4.2 依存関係グラフ

```
[Phase 0] ユーザー判断（P-0〜P-6）
   ↓
[Phase 1] Terraform 基盤（直列、ファイル相互参照のため）
   T1 (cloudwatch.tf) → T2 (iam.tf) → T3 (ec2.tf) → T4 (user_data.sh.tpl)
   ↓
[Phase 2] 設計書更新（並列可、ファイル独立）
   T5 (monitoring.md) ∥ T6 (env_config.md) ∥ T7 (runbook.md) ∥ T8 (release.md) ∥ T9 (architecture.md)
   ↓
[Phase 3] 補助
   T10 (README.md)
   ↓
[Phase 4] 検証
   T11 (terraform validate / plan)
   ↓
[Phase 5] セルフチェック + PR
   T12
```

**並列化判断**: Phase 1 は同一ディレクトリ内 4 ファイルが相互参照するため **単一ブランチで直列**（platform-builder 1 セッション完結）。Phase 2 はファイル独立だが差分が小さいため designer 1 セッションでまとめて実施可（無理に worktree を切らない）。

---

## 5. 受け入れ基準のマッピング

issue #188 §受け入れ基準 4 項目を本 PR / Step 11-E に分類:

| # | 受け入れ基準 | 本 PR で達成 | Step 11-E 実 apply で達成 |
|---|---|:-:|:-:|
| 1 | `monitoring.md` 内に `/ecs/expense-saas/api` の残存なし（grep 0 件） | ○ | - |
| 2 | 新ロググループ名が設計と実装で一致（grep で確認） | ○（grep で `/expense-saas/{env}/api` / `/${project_name}/${environment}/api` の整合確認） | - |
| 3 | AWS 上の新 LogGroup にログが転送されていることを確認 | - | ○（実 apply 後に `aws logs tail` で確認） |
| 4 | 旧 LogGroup のデータ移行 or 削除方針が記録されている | ○（本計画書 §P-0 で「c: 実 apply 直前に AWS 状況確認 → 手動判断」と明示） | ○（Step 11-E でその通り実施） |

**追加自主基準（reviewer 観点）**:

| # | 自主基準 | 確認方法 |
|---|---|---|
| 5 | `iam.tf` に `logs:CreateLogStream` + `logs:PutLogEvents` が LogGroup ARN 限定で定義されている | grep + `terraform plan` のリソース差分 |
| 6 | `aws_instance.app` の `depends_on` に LogGroup と新規 IAM policy が追加されている | grep |
| 7 | `terraform fmt -recursive` / `terraform validate` / `terraform plan` が success | T11 で実行 |
| 8 | `user_data.sh.tpl` の systemd unit 内 `ExecStart` に `--log-driver=awslogs` が含まれ、`awslogs-group` / `awslogs-region` / `awslogs-stream` の 3 オプションが全て揃っている | grep |
| 9 | `awslogs-create-group=false` 明示（P-3=a の意図表現） | grep |
| 10 | env_config.md §3.2 表に CloudWatch Logs LogGroup 行が追加され、SSM パス命名と階層整合している | 目視 |
| 11 | monitoring.md grep `/ecs/expense-saas/api` 残存ゼロ（T5 完了条件、W-03 対応） | `grep -n "/ecs/expense-saas/api" /root-project/dev-journal/deliverables/docs/50_detail_design/monitoring.md` が 0 件 |

---

## 6. リスクと留意事項

### 6.1 技術リスク

| ID | リスク | 対応 |
|---|---|---|
| R-1 | Amazon Linux 2023 の Docker に awslogs プラグインがデフォルト含まれているか | Docker Engine 標準で awslogs / json-file / syslog 等は同梱。AL2023 公式 docker パッケージで動作実績あり（要 platform-builder 確認） |
| R-2 | IAM 伝播遅延（policy attachment → EC2 起動 → docker daemon が awslogs に書き込む順序） | `depends_on = [aws_iam_role_policy.ec2_cloudwatch_logs]` で attachment 完了を待たせる（PR #152 / #187 と同パターン） |
| R-3 | LogGroup 不在状態で `awslogs-create-group=false` 指定 → コンテナ起動失敗 | `depends_on = [aws_cloudwatch_log_group.app]` で LogGroup を先行作成。Terraform 並列化を抑止 |
| R-4 | systemd unit の `%i` は instance template 識別子であり instance-id にはならない | §3.4.1 で **方式 X 確定**（heredoc クォート除去 + bash 変数展開）。`%i` は計画書全体で排除済み。platform-builder に判断余地なし |
| R-5 | 既存実装の `<<'SYSTEMD_EOF'`（シングルクォート付き = 変数展開なし）と新方式（変数展開ありにしたい）の混在 | §3.4.1 事前検査で systemd unit 中身に `$` を含む箇所ゼロを確認済み。`<<'SYSTEMD_EOF'` → `<<SYSTEMD_EOF` に変更しても副作用なし（B-02 で方式 X 確定） |
| R-6 | 既存 user_data の `unset PARAMS DATABASE_URL ...` でクリアされた後に `INSTANCE_ID` を使う場合の順序 | INSTANCE_ID は IMDSv2 から再取得可能で機密でもないため、`unset` 区間の外で扱えば問題なし。実装時に注意 |

### 6.2 スコープリスク

| ID | リスク | 対応 |
|---|---|---|
| R-7 | 旧 LogGroup `/ecs/expense-saas/api` が AWS 上に存在し、本 PR では削除されないまま放置 | P-0=c で「Step 11-E 実 apply 直前に AWS console / CLI で確認 → 手動削除 or 放置判断」と明示。本 PR スコープ外と割り切る |
| R-8 | nginx access log や CloudWatch Agent の収集ログを「同じ」LogGroup に押し込む案が後から発生する | P-5=a で本 issue スコープ外と確定。将来追加時は別 issue（命名を `/expense-saas/{env}/nginx-access` 等で分離するか議論） |
| R-9 | env=portfolio が現状唯一だが env_config.md は stg/prod 表記もある。LogGroup を env 込みで作ると monitoring.md の例示も env 込みで書く必要 | 本計画書 §3.5 で monitoring.md §9.1 加筆時に「現状は portfolio のみ。stg/prod は将来構築時 SSM パス命名と同様に増設」と注記する |

### 6.3 ドキュメント整合リスク

| ID | リスク | 対応 |
|---|---|---|
| R-10 | ADR-0005 が「stdout → CloudWatch Logs（ECS が自動転送）」と古いまま | issue #186 §1.3 で ADR は対象外と確定済み。本 issue でも触らない（決定時点記録）|

（**R-11 削除（W-04 対応）**: `grep -n "ADR-0004\|ECS Fargate\|Fargate" /root-project/dev-journal/deliverables/docs/50_detail_design/monitoring.md` 実行結果 = `L17: EC2 t3.micro（ADR-0004 §ポートフォリオ対応）` の 1 件のみで、Fargate 表記の残存はゼロ。issue #186 で対応済みのため本 issue スコープから除外。）

---

## 7. コミット計画

| # | 種別 | スコープ | コミットメッセージ案 |
|---|---|---|---|
| 1 | feat | Terraform Phase 1 (T1-T3) | `feat(infra): add CloudWatch Log Group + IAM policy for awslogs driver (issue #188)` |
| 2 | feat | user_data Phase 1 (T4) | `feat(infra): enable Docker awslogs driver in systemd unit (issue #188)` |
| 3 | docs | 設計書 Phase 2 (T5-T9) | `docs(monitoring): rename log group /ecs/expense-saas/api -> /expense-saas/{env}/api (issue #188)` |
| 4 | docs | README (T10) | `docs(terraform): add log-tail example in README (issue #188)` |

**コミット分割理由**:
- Terraform リソース追加と user_data 修正は意味的に同 PR だが、コミットを分けることで「インフラリソース定義」と「アプリ転送設定」のレイヤを明示
- 設計書更新は独立コミットで、grep ベースのレビュー（受け入れ基準 #1, #2）が追いやすい

**PR タイトル案**: `issue #188: monitoring log group rename + Docker awslogs driver implementation`

**コミット予定数（計画上の見積り）**: **4 コミット**

**コミット順序の厳守（I-05）**: 上記 #1 → #2 → #3 → #4 の順でコミットすること。レビュー時もこの順で見ると、「LogGroup/IAM 先行 → user_data で awslogs 有効化 → 設計書整合 → README 補助」のレイヤ依存が一目で追える。順序を入れ替えると awslogs 有効化が LogGroup 未存在状態を一瞬作るように見え、レビュー負荷が増える。

---

## 8. 計画策定で確認した上流資料・実装ファイル一覧

### 設計書（読み込み済み）

- `/root-project/dev-journal/issues/open/188-monitoring-log-group-naming-ecs-residue.md`
- `/root-project/dev-journal/issues/resolved/186-architecture-docs-compute-layer-ec2-mismatch.md`
- `/root-project/dev-journal/issues/resolved/187-terraform-ssm-session-manager-and-parameter-store.md`
- `/root-project/dev-journal/deliverables/docs/50_detail_design/monitoring.md`
- `/root-project/dev-journal/deliverables/docs/30_arch/adr/0005-monitoring-logging.md`
- `/root-project/dev-journal/deliverables/docs/70_operations/env_config.md`
- `/root-project/dev-journal/deliverables/docs/70_operations/runbook.md`

### 実装（読み込み済み）

- `/root-project/expense-saas/infra/terraform/iam.tf`（91 行、SSM Core + SSM GetParameters + S3 のみ。`logs:*` 不在を確認）
- `/root-project/expense-saas/infra/terraform/ec2.tf`（66 行、`templatefile()` の引数 7 個、`depends_on` 2 件を確認）
- `/root-project/expense-saas/infra/terraform/user_data.sh.tpl`（157 行、L128-147 の systemd unit に `--log-driver` 不在を確認）
- `/root-project/expense-saas/infra/terraform/variables.tf`（106 行、`aws_region` / `environment` / `project_name` 等の既存変数を確認）
- `/root-project/expense-saas/infra/terraform/locals.tf`（12 行、`common_tags` の構造を確認）
- `/root-project/expense-saas/infra/terraform/outputs.tf`
- `/root-project/expense-saas/infra/terraform/terraform.tfvars.example`
- `/root-project/expense-saas/infra/terraform/README.md`

### grep 確認結果

- `grep -rn "/ecs/" /root-project/dev-journal/deliverables/`: **1 件のみ**（monitoring.md L624、想定通り）
- `grep -rn "awslogs\|CloudWatch Logs\|log_group" /root-project/expense-saas/infra/`: **0 件**（実装ゼロを再確認）

---

## 9. 完了条件（本計画書のセルフチェック）

- [x] §1 で設計と実装の乖離をマッピングした
- [x] §2 でユーザー判断 7 項目（P-0〜P-6）を選択肢付きで提示し、推奨案を明示した
- [x] §3 で Terraform / user_data / 設計書の具体的変更箇所を列挙した
- [x] §4 でタスク分解と依存関係グラフを作成した
- [x] §5 で受け入れ基準を本 PR vs Step 11-E に分類した
- [x] §6 でリスク 11 件を列挙し対応方針を記載した
- [x] §7 でコミット計画を 4 コミットで提示した
- [x] §8 で参照資料を網羅した

---

## 10. ユーザーへのエスカレーション事項（着手前に確定が必要）

1. **分岐元コミットの確認**: `expense-saas` リポジトリの `origin/master` HEAD = `445240b infra(terraform): migrate EC2 access + secrets to SSM (#187)`（issue #187 / PR #152 マージスカッシュ）。これを分岐元とする（B-01 修正: v1 で誤って `619a702` を分岐元と記載していたが、`619a702` は root-project メタリポジトリの HEAD であり expense-saas には存在しない。`git cat-file -t 619a702` = `fatal: Not a valid object name` で確認済み）。
2. **P-0〜P-6 の最終確定**: 7 項目すべて推奨案で進める場合は「全推奨案で OK」の 1 行返答で可。
3. **P-1 の重要判断**: issue #188 §提案では推奨 B = `/expense-saas/api`（env なし）だが、本計画では env_config.md §5.3.2 SSM 命名と階層整合を理由に P-1=b（`/expense-saas/{env}/api`）を推奨。**issue 起票案からの逸脱**のため明示確認推奨。

---

## 改版履歴

### v2 (2026-05-25) — reviewer CONDITIONAL PASS 指摘反映

Blocker 2 件 + Warning 4 件 + Info 3 件を反映。

**Blocker（2 件、全て反映）**:
- **B-01**: 分岐元コミット記述を訂正（L5 / §10.1）。誤: `619a702`（root-project メタリポジトリ HEAD、expense-saas に存在せず `git cat-file -t` で fatal）→ 正: `expense-saas` の `origin/master` HEAD = `445240b infra(terraform): migrate EC2 access + secrets to SSM (#187)`。v1 で書いていた「ユーザー指示の `445240b` は古い」旨の異議を削除。
- **B-02**: §3.4 を全面再構成。`%i` を完全排除し、heredoc 切替方式を **計画書側で「方式 X（クォートなし）」に確定**。事前検査として user_data.sh.tpl L129-147 の systemd unit 中身を Read で確認し、`$` を含む箇所がゼロ（変数参照皆無）であることを明記。副作用がないため 2 段階生成方式（sed 置換）は不採用。platform-builder の判断余地を排除。

**Warning（4 件、全て反映）**:
- **W-01**: architecture.md パス揺れ防止。§3.5 architecture.md セクションに対象パス絶対値 `/root-project/dev-journal/deliverables/docs/30_arch/architecture.md` を明示（`30_architecture/` は存在しない）。
- **W-02**: §3.3 templatefile 引数の明確化。「追加するのは `log_group_name` のみ、`aws_region` は既存 ec2.tf L43 で渡し済みのため流用」と明記。
- **W-03**: §3.5 monitoring.md 改修範囲を行単位で明示（L35 は不変、L47 / L367 / §9.1 を改修）。§5 自主基準 #11 に「monitoring.md grep `/ecs/expense-saas/api` 残存ゼロ」を T5 完了条件として追加。
- **W-04**: R-11 を削除。事前検査 `grep -n "ADR-0004\|ECS Fargate\|Fargate" monitoring.md` 結果 = L17 の `EC2 t3.micro（ADR-0004 §ポートフォリオ対応）` 1 件のみで Fargate 残置ゼロ → issue #186 で対応済み、本 issue スコープ外と確定。両論併記を解消。

**Info（3 件反映、I-03/I-04 は該当箇所なし）**:
- **I-01**: §3.1 で `local.prefix` を流用しない理由を 1 行注記（`-` 区切りのフラット命名規約のため、`/` 階層を要する LogGroup 名には不適）。
- **I-02**: §3.4.2 で IMDSv2 取得を `L8 直後（機密区間より前）` に配置すると明示。`app.env` への追記は方式 X 採用により不要と判断し独立 echo を削除。
- **I-05**: §7 末尾に「コミット順序を厳守し、レビューも #1 → #2 → #3 → #4 順で見る」と注記。

**v1 (2026-05-25)**: 初版。issue #188 への計画書として作成。
