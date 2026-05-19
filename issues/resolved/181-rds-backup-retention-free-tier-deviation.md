# RDS backup_retention_period が Free Tier 制約により NFR-AVAIL-003 から逸脱（7 日 → 1 日）

## 発見日
2026-05-19

## カテゴリ
infrastructure

## 影響度
中

## 発見経緯
proactive

## 関連ステップ
Step 11-E（デプロイ）Phase 3: terraform apply

## ブロッカー
なし

## 問題

Step 11-E Phase 3 の `terraform apply` 実行時、RDS インスタンス作成が以下のエラーで失敗:

```
Error: creating RDS DB Instance (expense-saas-portfolio-db):
api error FreeTierRestrictionError: The specified backup retention period
exceeds the maximum available to free tier customers.
To remove all limitations, upgrade your account plan.
```

新規 AWS アカウントの Free Tier 制約により、`backup_retention_period` の上限が 1 日に制限されている（2024 年以降の新仕様）。NFR-AVAIL-003 では 7 日間保持を要求していたが、Free Tier では実現不可能。

暫定対応として `rds.tf` の `backup_retention_period = 7` を `1` に変更し、Phase 3 を続行できる状態にした。

## 影響

- **設計逸脱**: NFR-AVAIL-003（7 日間バックアップ保持）からの逸脱
- **データ復旧能力**: 1 日前までしか PITR（point-in-time recovery）できない
- **ポートフォリオ用途での実害**: 限定的（UAT 期間中のデータ消失リスクは小さい）
- **本番環境への影響**: 本番運用想定なら NFR を満たす必要があるが、本プロジェクトは portfolio スコープのため受容

## 提案

post-MVP で以下のいずれかを検討:

1. **AWS Support に Free Tier 制約解除を申請**
   - 申請理由を具体的に書く（ポートフォリオ用途では通らない可能性高い）
   - 承認されれば追加コストなしで 7 日保持可能

2. **有料アカウントへのアップグレード時に 7 日に戻す**
   - 13 ヶ月の無料枠終了 + `terraform destroy` で停止する前提
   - 復旧運用を想定するなら、別途 AWS アカウントを作り直して有料化

3. **NFR-AVAIL-003 を portfolio 向けに緩和する設計修正**
   - 11-E チケットで「portfolio 環境では 1 日保持を許容」と明示
   - 本番環境向け NFR は別途維持

### 着手タイミング

- Step 11-F UAT 完了後 or `terraform destroy` 時点で判断
- portfolio が無料枠で動いている限りは案 3 が現実的

---

## 解決内容

本プロジェクトは portfolio MVP であり本番運用予定がないため、**案 3（NFR-AVAIL-003 を portfolio 向けに緩和）** を採用し、設計成果物を実装値（`backup_retention_period = 1`、PR #146）と整合させた。

### 実装側修正
- `expense-saas/infra/terraform/rds.tf`: `backup_retention_period = 7 → 1`（PR #146 マージ済み 2026-05-19）
- `expense-saas/infra/terraform/rds.tf:55-56`: コメントを「逸脱 / post-MVP 追跡」→「portfolio 仕様として 1 日保持に緩和済み」に更新（PR #147 マージ済み 2026-05-19、codex review-finding 120 への対応）

### 設計成果物修正（14 箇所、本セッションで対応）

| ファイル | 行 | 修正概要 |
|---------|----|---------|
| `10_requirements/requirements.md` | 444 | NFR-AVAIL-003: 「7日間保持」→「1日間保持」+ Free Tier 制約注記 |
| `30_arch/architecture.md` | 488 | トレーサビリティ表: 「7日間保持」→「1日間保持」+ 注記 |
| `30_arch/adr/0004-infra.md` | 42, 71, 94 | RDS 選定根拠 + 決定表: 「7日間」→「1日間」+ 注記 |
| `60_test/traceability.md` | 234 | NFR-AVAIL-003 行: 「7日間保持」→「1日間保持」+ 注記 |
| `70_operations/backup_restore.md` | 17, 37, 59, 106, 142, 143, 157, 167 | 参照一覧 / 設定表 / 世代管理 / 品質チェックリスト / 将来拡張: 「7日間」→「1日間」 |

### 案 Z（同一文書内整合を優先）方針による意図的維持箇所

以下は「業務 SaaS としての将来想定」として 7 日表記を維持。NFR-AVAIL-003（1 日間保持）とは概念が異なるため矛盾しない:

- `70_operations/backup_restore.md:144`: RDS 手動スナップショット（自動バックアップとは別概念、Free Tier 制約外）
- `70_operations/env_config.md:60`: dev/stg/prod 環境別設定の stg/prod 列（portfolio は単一環境、stg/prod は将来構築時の推奨設定）

### 残置事項
なし。本プロジェクトは portfolio 用途で本番運用予定がないため、`backup_retention_period` を将来 7 日に戻す予定はない（提案案 1・案 2 は採用しない）。

## 解決日
2026-05-19
