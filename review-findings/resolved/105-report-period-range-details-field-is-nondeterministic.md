# 105: RPT-010 の details[].field が period_start / period_end のどちらでも可になっている

## 指摘概要
`reports.md` の `RPT-010` は 422 VALIDATION_ERROR に `details` 配列が付くことは追記されたが、`details[].field` を `period_end` または `period_start` のどちらでも可としている。

既存の `items.md` / `auth.md` の details 記述は、入力項目ごとに期待する field を固定している。`RPT-010` だけ実装側の選択に委ねると、後続の実装・テストで `period_start` と `period_end` のどちらを UI field-level error に紐づけるべきかが確定せず、issue 121 の「フィールド単位のエラー情報」を test_cases 側で十分に固定できない。

## 根拠
- `dev-journal/deliverables/docs/60_test/test_cases/reports.md:87`
  - `RPT-010` の期待結果が「details 配列が付与される」「details[].field は実装側で period_end もしくは period_start のいずれか」となっている。
- `dev-journal/deliverables/docs/60_test/test_cases/items.md:80`-`81`
  - `ITM-011` / `ITM-012` は `details[].field = "amount"` を固定している。
- `dev-journal/deliverables/docs/60_test/test_cases/items.md:84`
  - `ITM-015` は `details[].field = "expense_date"` を固定している。
- `dev-journal/deliverables/docs/60_test/test_cases/auth.md:97`-`98`
  - `AUTH-026` / `AUTH-027` は details に含まれる field を `company_name` / `email` として固定している。
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:2088`-`2100`
  - `ExpenseReportCreateRequest` の入力フィールドは `period_start` / `period_end`。
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:2098`-`2100`
  - `period_end` は「開始日以降」と説明されており、`period_start="2026-03-31"`, `period_end="2026-03-01"` のケースでは `period_end` 側の制約違反として固定しやすい。
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:1862`-`1870`
  - `ValidationError` は `field` を required としている。

## 判定
- 重大度: 中
- 分類: details 粒度不統一 / テスト期待の曖昧さ

## 修正方針案
`RPT-010` の期待結果を既存 details 記述と同じ粒度にそろえ、field を固定する。

推奨は `period_end` に固定すること。

```md
422 VALIDATION_ERROR。`details[].field = "period_end"` を含む（`period_end` は `period_start` 以降である必要がある）
```

別方針として `period_start` に寄せる場合でも、test_cases 上ではどちらか一方に固定し、後続の実装・テストが追加解釈なしで検証できる状態にする。
