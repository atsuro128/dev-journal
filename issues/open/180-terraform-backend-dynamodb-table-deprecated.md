# backend.tf の dynamodb_table が Terraform 1.10+ で deprecated（use_lockfile に移行）

## 発見日
2026-05-18

## カテゴリ
infrastructure

## 影響度
低

## 発見経緯
proactive

## 関連ステップ
Step 11-E（デプロイ）Phase 2: terraform init

## ブロッカー
なし

## 問題

Step 11-E Phase 2 の `terraform init` 実行時、Terraform 1.15.3 から以下の deprecation 警告が出る:

```
Warning: Deprecated Parameter

  on backend.tf line 12, in terraform:
  12:     dynamodb_table = "expense-saas-tflock"

The parameter "dynamodb_table" is deprecated. Use parameter "use_lockfile" instead.
```

Terraform 1.10（2024-11 リリース）から S3 backend が native lock 機能（S3 のみで排他制御）を持ち、外部 DynamoDB テーブルを使う旧方式は非推奨化された。

## 影響

- 現状は警告のみで apply / plan は正常動作する
- 将来の Terraform バージョン（おそらく 2.x）で `dynamodb_table` パラメーターがサポート終了する可能性
- DynamoDB テーブル `expense-saas-tflock` の維持コスト（PAY_PER_REQUEST なので実質無料だが、リソースとして残る）

## 提案

post-MVP で以下の対応を検討:

1. **backend.tf 修正**: `dynamodb_table = "expense-saas-tflock"` を削除し、`use_lockfile = true` を追加
2. **既存 state ファイル migrate**: `terraform init -migrate-state`（state 自体は S3 にあるため、lock 機構のみ切り替え）
3. **DynamoDB テーブル削除**: `aws dynamodb delete-table --table-name expense-saas-tflock --region ap-northeast-1`
4. **11-E チケット §5.3 の手順を更新**: state バケット作成のみで完結、DynamoDB lock テーブル作成手順を削除

### MVP 期間中は現状維持で問題なし

- 動作上の支障なし（警告のみ）
- ポートフォリオ運用期間（13 ヶ月の無料枠内）はサポート終了の心配なし
- 着手タイミング: Step 11-F 完了後 or `terraform destroy` 直前に整理

---

## 解決内容

## 解決日
