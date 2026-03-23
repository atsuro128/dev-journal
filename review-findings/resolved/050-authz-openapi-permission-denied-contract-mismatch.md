# 050: `authz.md` と `openapi.yaml` で同一テナント内の閲覧拒否エラーコードが不一致

## 指摘概要

`authz.md` は、同一テナント内でロール検証は通過したがリソース閲覧権限がないケースを `403 PERMISSION_DENIED` と定義している。一方、`openapi.yaml` の `GET /api/reports/{id}` は同じケースを `403 FORBIDDEN` と記述し、レスポンス定義も汎用 `Forbidden` のみを参照している。認可設計の正本と API 契約でエラーコードが揺れており、実装者・テスターがどちらを正とすべきか判断できない。

## 根拠

- `dev-journal/deliverables/docs/50_detail_design/authz.md:299`
- `dev-journal/deliverables/docs/50_detail_design/authz.md:414`
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:625`
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:643`
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:2186`

## 影響

- `GET /api/reports/{id}` の同一テナント内権限不足時に、`FORBIDDEN` を返す実装と `PERMISSION_DENIED` を返す実装の両方が成立してしまう
- SDK 生成、API テスト、画面のエラーハンドリングで `403` の内訳を確定できない
- Step 5 Phase 3 の完了条件である「認可チェックの責務と実装場所が決まっている」は満たしていても、Phase 2/3 間の契約整合性が崩れる

## 修正案

以下のいずれかに統一する。

1. `authz.md` を正本として、同一テナント内の所有権不足・閲覧権限不足は `403 PERMISSION_DENIED` に統一し、`openapi.yaml` の説明文とレスポンス例を更新する
2. 逆に `FORBIDDEN` を採用するなら、`authz.md` SS6/SS7/SS11 と `security.md` のエラーコード定義を合わせて修正する

現状の設計文脈では、ロール不足 = `FORBIDDEN`、リソース単位の権限不足 = `PERMISSION_DENIED` に分けているため、前者で統一するのが自然。
