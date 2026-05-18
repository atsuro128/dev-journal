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

## 解決日
