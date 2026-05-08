# 117: S3 バージョニング方針が上流の backup_restore と不整合なまま差分管理されていない

## 指摘概要
計画書の Terraform 方針では領収書バケットの versioning を有効化している一方、上流の `backup_restore.md` は MVP の S3 バージョニングを無効と定義している。  
この差分が `env_config.md prod との差分` や 11-F 引き継ぎ項目に含まれていないため、復旧期待値・コスト前提・誤削除時の対処が文書間でずれたまま下流へ渡る。

## 根拠
- 計画書は `s3.tf` の最低定義として `aws_s3_bucket_versioning` を `Enabled`、さらに noncurrent 30 日 expire の lifecycle を採るとしている。  
  `dev-journal/progress-management/step11-e-deployment-plan.md:193`
- しかし計画書の差分整理は DB/JWT 保管、CORS、S3 バケット名、TLS に限られており、S3 versioning の差分がない。  
  `dev-journal/progress-management/step11-e-deployment-plan.md:262-269`
- 上流の `backup_restore.md` は MVP で S3 バージョニング無効、誤削除時は復旧不可、将来対応として versioning 有効化を検討すると定義している。  
  `dev-journal/deliverables/docs/70_operations/backup_restore.md:33-40, 76-91, 125-134`
- MVP スコープも「データのバックアップ・リストア手順は RDS の自動バックアップに委ねる」としており、S3 復旧戦略の変更は差分として明示すべき位置づけにある。  
  `dev-journal/deliverables/docs/02_scope.md:102`

## 判定
重大度: 中  
分類: warning

## 修正方針案
- portfolio 実装で S3 versioning を有効化するなら、少なくとも以下を明示すること。
- `backup_restore.md` の MVP 前提との差分であること。
- 11-F 引き継ぎでの復旧期待値が「誤削除復旧不可」から変わること。
- 追加コストと lifecycle 方針。
- 逆に上流どおり MVP を維持するなら、計画書側の `aws_s3_bucket_versioning` / lifecycle 方針を削除して整合させること。
