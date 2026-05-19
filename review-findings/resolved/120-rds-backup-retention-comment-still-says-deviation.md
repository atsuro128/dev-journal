# 120: RDS backup_retention_period の実装コメントが解決後の NFR と矛盾している

## 指摘概要
NFR-AVAIL-003 は portfolio 仕様として「RDS 自動バックアップ 1 日間保持」に緩和され、設計成果物側も 1 日保持へ更新されている。一方で、実装側の Terraform コメントがまだ「NFR-AVAIL-003（7 日間保持）からの逸脱」「issue #181 で post-MVP に追跡」と説明しており、解決後の正本と矛盾している。

このままだと、後続担当が `backup_retention_period = 1` を未解決の暫定逸脱と誤読し、issue #181 の「残置事項なし」「将来 7 日に戻す予定はない」という解決内容と整合しない。

## 根拠
- `dev-journal/deliverables/docs/10_requirements/requirements.md:444`
  - NFR-AVAIL-003 は `RDS 自動バックアップ（1日間保持）` と定義されている。
- `dev-journal/issues/open/181-rds-backup-retention-free-tier-deviation.md:68-91`
  - 案 3 を採用し、`backup_retention_period = 1` と設計成果物を整合済み、残置事項なしと記載している。
- `expense-saas/infra/terraform/rds.tf:55-57`
  - `backup_retention_period = 1` 自体は正しいが、直前コメントが `NFR-AVAIL-003（7 日間保持）からの逸脱は issue #181 で post-MVP に追跡。` のまま残っている。

## 判定
重大度: 中

分類: トレーサビリティ不整合 / issue 解決内容の不足

14 箇所の設計成果物修正自体は NFR-AVAIL-003 緩和方針と整合しているが、実装側コメントが旧方針を正本のように残しているため、requirements → architecture → ADR → traceability → backup_restore → Terraform の説明が最後で途切れる。

## 修正方針案
`expense-saas/infra/terraform/rds.tf:55-56` のコメントを、解決後の方針に合わせて更新する。

例:

```hcl
# AWS Free Tier アカウントの制約により backup_retention_period の上限が 1 日。
# NFR-AVAIL-003 は portfolio 仕様として 1 日保持に緩和済み（issue #181）。
```

あわせて issue #181 の解決内容に「Terraform コメントも解決後方針へ更新済み」と追記できると、レビュー後に issue を pending-review / resolved へ移す際の根拠が明確になる。
