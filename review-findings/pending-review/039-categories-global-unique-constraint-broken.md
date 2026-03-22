# 039: グローバルカテゴリの一意性を DB で保証できていない

## 指摘概要
`categories` テーブルは MVP の 6 固定カテゴリをグローバルデータとして保持する設計ですが、現行の一意制約 `UNIQUE (tenant_id, code)` では `tenant_id IS NULL` の行同士が重複可能です。PostgreSQL では `NULL` 同士は一意制約で同一とみなされないため、同じ `code` のグローバルカテゴリを複数登録できてしまいます。これにより、カテゴリ一覧取得や `category_id` ベースの入力値との対応が不安定になり、MVP の固定 6 カテゴリ前提が崩れます。

## 根拠
- [db_schema.md](/root-project/dev-journal/deliverables/docs/50_detail_design/db_schema.md#L231) では `tenant_id` を nullable にした `categories` テーブルを定義している
- [db_schema.md](/root-project/dev-journal/deliverables/docs/50_detail_design/db_schema.md#L241) では一意制約を `UNIQUE (tenant_id, code)` としている
- [db_schema.md](/root-project/dev-journal/deliverables/docs/50_detail_design/db_schema.md#L248) では `tenant_id IS NULL` を「MVP の 6 固定カテゴリ」と定義している
- [report.md](/root-project/dev-journal/deliverables/docs/40_basic_design/screens/report.md#L480) では画面のカテゴリドロップダウンがこの固定カテゴリ前提で動作する

## 判定
高 / DB制約不備

## 修正方針案
グローバルカテゴリとテナント固有カテゴリで一意性を分けて保証する。例えば以下のいずれかに統一する。
- `tenant_id IS NULL` 向けの部分ユニークインデックス `UNIQUE (code) WHERE tenant_id IS NULL`
- `tenant_id IS NOT NULL` 向けの部分ユニークインデックス `UNIQUE (tenant_id, code) WHERE tenant_id IS NOT NULL`
- もしくは `NULLS NOT DISTINCT` を使う方針を採るが、採用する PostgreSQL バージョン前提を明記する
