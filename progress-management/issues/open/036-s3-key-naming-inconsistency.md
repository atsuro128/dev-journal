# S3 キー命名規則の表記揺れ

## 発見日
2026-03-22

## カテゴリ
detail-design

## 影響度
中

## 発見経緯
user-report

## 関連ステップ
Step 5（詳細設計）

## 問題

上流成果物間で S3 オブジェクトキーの命名規則が不統一:

| ドキュメント | パス形式 |
|---|---|
| `domain_model.md` | `{tenant_id}/{report_id}/{attachment_id}` |
| `security-policy.md` (§3, §5) | `{tenant_id}/{report_id}/{file_id}` |

`attachment_id` と `file_id` が混在しており、用語集（glossary.md）との整合も取れていない。

## 影響

Step 5 の詳細設計（files.md, openapi.yaml, db_schema.md）で参照する際に混乱を招く。実装時にも命名の不一致が伝播するリスクがある。

## 提案

`{tenant_id}/{report_id}/{attachment_id}` に統一する。理由:
- `attachment_id` は UUID でユニーク、`item_id` 階層は不要
- ファイル名は DB に保持し S3 キーには含めない（パストラバーサル防止）
- `file_id` は用語揺れであり `attachment_id` に統一

修正対象:
- `.claude/rules/security-policy.md` §3 テナント分離（S3 パス）
- `.claude/rules/security-policy.md` §5 ファイルアップロード（該当箇所があれば）

---

## 解決内容

## 解決日
