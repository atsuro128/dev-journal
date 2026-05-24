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
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->
