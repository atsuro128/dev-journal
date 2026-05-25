# 監視設計のロググループ命名残置をリネーム（monitoring.md + 実装側 docker awslogs-group 設定の同期更新）

## 発見日
2026-05-24

## カテゴリ
infrastructure

## 影響度
低

## 発見経緯
review

## 関連ステップ
Step 11-E / issue #186（reviewer レビュー I-01 で検出）

## ブロッカー
なし

## 問題

issue #186 の reviewer レビューで I-01 として検出された残課題:

`dev-journal/deliverables/docs/50_detail_design/monitoring.md` のロググループ名 `/ecs/expense-saas/api` が ECS 命名の名残として残置されている。issue #186 のスコープは「設計成果物の ECS 記述を EC2 実装に整合」だが、ロググループ名は設計書上のリテラル変更だけでは不十分で、**実装側（Terraform / Docker awslogs ドライバ設定の `--log-opt awslogs-group=...`）の同期変更**が必要なため designer 単独では確定できず、別 issue として切り出された。

### 該当箇所

| 種別 | ファイル | 該当 | 現状 |
|---|---|---|---|
| 設計 | `dev-journal/deliverables/docs/50_detail_design/monitoring.md` | L624 周辺 | `/ecs/expense-saas/api` |
| 実装 | `expense-saas/infra/terraform/`（要調査）or `user_data.sh.tpl` | docker run の `--log-opt awslogs-group=...` | 要確認（recommended `/ecs/expense-saas/api`） |
| 実装 | CloudWatch Logs の実 LogGroup | AWS 側で作成済みの LogGroup 名 | 要確認（既存 LogGroup を rename or 新規作成 + 旧削除） |

## 影響

- **命名規約の一貫性欠如**: 設計成果物全体は EC2 構成に整合済みだが、ロググループ名だけ ECS 名残のため、運用時に「実装は EC2 なのに ECS と名前が付いている」混乱を招く
- **既存 LogGroup のデータ移行**: AWS 上に既存 LogGroup（`/ecs/expense-saas/api`）に蓄積されたログが存在する場合、リネーム不可（AWS CloudWatch Logs の LogGroup は rename 不可）のため、**新規 LogGroup 作成 + 旧 LogGroup の保持 or 削除**を判断する必要あり
- **影響度は低**: 機能・性能・セキュリティへの影響なし、純粋に命名の問題

## 提案

### 命名候補

| 案 | 名前 | メリット | デメリット |
|---|---|---|---|
| A | `/ec2/expense-saas/api` | コンピュート基盤を明示、現在の命名規約と対称 | 将来別基盤に変わったらまた変える必要あり |
| B | `/expense-saas/api` | 中立、シンプル、AWS 公式の慣例（プロジェクト名/コンポーネント名） | コンピュート基盤情報がロググループ名から失われる |

**推奨**: B `/expense-saas/api`（中立・将来変更不要・AWS Well-Architected の命名指針に近い）

### 変更スコープ

1. **設計**: `monitoring.md` L624 周辺の `/ecs/expense-saas/api` → 新命名にリネーム
2. **実装**: `expense-saas/infra/terraform/` 配下の Terraform 定義（`aws_cloudwatch_log_group` リソース）+ `user_data.sh.tpl` の `docker run --log-opt awslogs-group=...` 値を同期更新
3. **AWS 上**: 新 LogGroup を Terraform で作成 → docker awslogs ドライバの転送先を新 LogGroup に切替 → 旧 LogGroup `/ecs/expense-saas/api` のログ retention 確認後に削除（or 保持期限まで残す）

### 着手タイミング

- **post-MVP** 推奨。Step 11-F UAT 完了後 or Step 11-E 実デプロイ時の併せ作業に組み込む
- ブロッカー issue ではないため、MVP クリティカルパス（11-E → 11-F）の進行は阻害しない
- issue #187（SSM Session Manager + Parameter Store）の Terraform 修正と同セッションで実施するのも効率的

### 受け入れ基準

- [ ] `monitoring.md` 内に `/ecs/expense-saas/api` の残存なし（grep 0 件）
- [ ] 新ロググループ名が設計と実装で一致（grep で確認）
- [ ] AWS 上の新 LogGroup にログが転送されていることを確認
- [ ] 旧 LogGroup のデータ移行 or 削除方針が記録されている

---

## 解決内容

PR #153（expense-saas、squash commit `420f0ea`）でマージ完了。当初想定の「リネームのみ」スコープから、設計（UD-2=B awslogs ドライバ採用）vs 実装（awslogs ゼロ）の重大な乖離が判明し、**awslogs 実装そのもの**を含む完全対応スコープに拡大して対応した。

### 確定したユーザー判断（計画書 v2、§2）

- **P-0=c**: 旧 LogGroup `/ecs/expense-saas/api` は Step 11-E 実 apply 直前に AWS console で存在確認 → 手動削除 or 放置判断（そもそも実装側に awslogs ドライバが一切なかったため AWS 上に旧 LogGroup が存在しない可能性が高い）
- **P-1=b**: ロググループ命名 `/expense-saas/${var.environment}/api`（env 込み）。SSM Parameter Store 命名 `/expense-saas/{env}/...` と階層整合させるため、issue §提案案 B（`/expense-saas/api`、env なし）から逸脱
- **P-2=a**: awslogs-stream は `${INSTANCE_ID}/api`（IMDSv2 で EC2 起動時取得 + bash 展開）。systemd `%i` は instance template identifier のため使用不可
- **P-3=a**: `logs:CreateLogGroup` 不要。Terraform で LogGroup を先行作成し、awslogs ドライバには `awslogs-create-group=false` で書き込み専用
- **P-4=b**: AWS 実環境疎通検証（受け入れ基準 #3）は Step 11-E 実 apply 時に切り出し（本 PR は静的検証 = fmt / grep / 構造レビューまで）
- **P-5=a**: nginx 等の追加 LogGroup は本 issue では扱わない
- **P-6=a**: env_config.md への新規変数追加なし（既存 `LOG_LEVEL` で十分）

### 実装変更（PR #153、計画書 §3 / §4.1 T1-T4 + T10）

| ファイル | 変更内容 |
|---|---|
| `infra/terraform/cloudwatch.tf`（新規） | `aws_cloudwatch_log_group "app"` 定義（name = `/${var.project_name}/${var.environment}/api`、retention 30 日（ADR-0005）、tags） |
| `infra/terraform/iam.tf` | `aws_iam_role_policy "ec2_cloudwatch_logs"` 追加（`logs:CreateLogStream` + `logs:PutLogEvents` を LogGroup ARN:* 限定） |
| `infra/terraform/ec2.tf` | `templatefile()` 引数に `log_group_name = aws_cloudwatch_log_group.app.name` 追加、`depends_on` に `ec2_cloudwatch_logs` + `app` LogGroup 追加 |
| `infra/terraform/user_data.sh.tpl` | IMDSv2 で INSTANCE_ID を user_data 早期（機密区間より前）に取得、systemd unit heredoc を `<<'SYSTEMD_EOF'` → `<<SYSTEMD_EOF`（方式 X）に変更、`ExecStart` に `--log-driver=awslogs` + 5 オプション（region / group / stream / create-group=false）追加 |
| `infra/terraform/README.md` | §10 ヘルスチェック後に「§ ログ確認」セクション追加（`aws logs tail /expense-saas/portfolio/api --follow` 例示） |

### dev-journal 側コミット（T5-T9 + 計画書）

- `1d1352b` monitoring.md / env_config.md / runbook.md / release.md / architecture.md の awslogs 実装同期（T5-T9）
- `2e6c327` 計画書 v2 を `progress-management/working/188-architect-plan.md` にアーカイブ

### レビューサイクル

- **architect 計画書**: v1 → reviewer CONDITIONAL PASS（Blocker 2 + Warning 4 + Info 5）→ v2（B-01 分岐元コミット訂正 / B-02 `%i` 排除 + heredoc 方式 X 確定 + Info 反映）→ reviewer PASS
- **内部レビュー**: 1 ラウンド PASS（Blocker 0 / Warning 0 / Info 1、PR #152 確立パターン整合確認済み）
- **codex レビュー**: 1 ラウンド APPROVE 相当（Blocker 0、Docker awslogs ドライバの IAM 要件と整合）

### 本 issue スコープ外（Step 11-E 実 apply で実施）

- **受け入れ基準 #3**: AWS 上の新 LogGroup `/expense-saas/portfolio/api` にログが転送されていることを確認（`aws logs tail` で実検証）
- **受け入れ基準 #4**: 旧 LogGroup `/ecs/expense-saas/api` の存在確認 → 削除 or 放置判断（`aws logs describe-log-groups --log-group-name-prefix /ecs/`）

これらは PR #153 マージ時点では未検証。Step 11-E 実 apply セッションで `terraform apply` 実施時に併せて消化する。

### 派生 issue

- **本 issue から新規起票なし**（codex / 内部レビュー指摘ゼロ、計画書 §6 リスクは全て対応済み）

## 解決日
2026-05-25
