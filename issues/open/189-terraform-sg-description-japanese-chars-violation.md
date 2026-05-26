# terraform/security_groups.tf: SG ingress description に日本語混入で AWS validation 違反（PR #151 漏れ）

## 発見日
2026-05-26

## カテゴリ
infrastructure

## 影響度
中

## 発見経緯
escalation

## 関連ステップ
Step 11-E（実 apply セッション）/ PR #151（issue #185 T3）

## ブロッカー
なし（緊急 fix は master 直接編集で実施済み、本 issue は履歴化・再発防止追跡用）

## 問題

`expense-saas/infra/terraform/security_groups.tf` の `aws_security_group.alb` の `ingress` ブロック（**2 箇所**）の `description` に日本語が混入していたため、`terraform plan` が AWS 制約違反でエラー停止した。

1. **L46 (`restrict_alb_to_cloudfront=false` 側 / 初回 apply 時の経路)**:
   ```
   Error: "ingress.0.description" doesn't comply with restrictions:
     "HTTP from anywhere (初回 apply 用。カスタムヘッダ検証で保護済み。CloudFront Deployed 確認後に restrict_alb_to_cloudfront=true で絞ること)"
   ```
2. **L58 (`restrict_alb_to_cloudfront=true` 側 / 2 段階目 apply 時の経路)**:
   ```
   Error: "ingress.0.description" doesn't comply with restrictions:
     "HTTP from CloudFront origin-facing prefix list only (B-1-b 2 層目)"
   ```

最初の発覚時 (L46) に L58 もすぐ修正できたはずだが、片方しか直さずに進めて 2 段階目 apply で再び同じ Error に遭遇した（**指揮役側の網羅確認不足**）。教訓: AWS validation 系の Error fix では、同種のフィールドをファイル全体で網羅的に grep する。

AWS Security Group の `description` は ASCII の限定文字（`[0-9A-Za-z_ .:/()#,@\[\]+=&;{}!$*-]`）のみ許可される。CloudFront 構成 PR #151 でこの 2 行だけ日本語のまま残置され、内部レビュー / codex レビューでも検知されなかった。

同ファイル内の他の `description`（`L82/L87/L113/L117/L125` 等）はすべて ASCII。コメント（`#` 始まり）は日本語可。

## 影響

- Step 11-E 実 apply セッション（2026-05-26）で `terraform plan` が apply blocker としてエラー停止
- 緊急 fix（master 直接編集で英語化）で apply は継続できたが、PR フローを経由しない緊急対応となった（過去事例: 5/18 の `rds.tf backup_retention_period` 緊急 fix と同パターン）

## 提案

### 既に実施済み（緊急 fix）

`security_groups.tf` L46 を以下に修正済み（master 直接編集、commit 未確定）:

```hcl
description = "HTTP from anywhere (initial apply; protected by X-Origin-Verify header; switch restrict_alb_to_cloudfront=true after CloudFront Deployed)"
```

### 再発防止策（本 issue で追跡）

1. **lint チェック追加**: `infra/terraform/` 配下の `.tf` ファイルで `description` フィールドに日本語（U+3000-U+9FFF + U+FF00-U+FFEF）を含まないことを CI で検証
   - 候補: pre-commit hook で `rg -nP '\bdescription\s*=\s*"[^"]*[\x{3000}-\x{9fff}\x{ff00}-\x{ffef}]'` を NG 判定
   - または `terraform validate` を CI で必ず通す（ただし `terraform validate` は AWS provider の正規表現バリデーションは実行しないため、別途 lint が必要）
2. **codex / reviewer のチェック観点追加**: AWS リソースの `description` / `name` / `tags` 等のフィールドは ASCII のみ許可される場合があることをレビュー時に確認
3. **テンプレート整備**: `.tf` ファイル内コメント（`#` 始まり）は日本語可、文字列フィールド（`description = "..."` 等）は ASCII のみ、というルールを `ai-dev-framework/rules/` に明文化

### 着手タイミング

post-MVP。本 issue 起票自体は緊急 fix のコミット時に commit message から参照する。
再発防止策（lint 追加）の実装は MVP リリース後に検討。

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->
