# 018: `User` の設計と `TNT-001` が矛盾している

## 指摘概要
`User` エンティティは `tenant_id` を持たない設計になっている一方で、不変条件 `TNT-001` は「全テーブルに tenant_id が存在する」と定義されている。Step 2 の中でテナント分離ルールの例外条件が未整理のまま残っている。

## 根拠
- `domain_model.md` では `User` に `tenant_id` がなく、明示的に「User テーブルには tenant_id を持たない」としている
  - `dev-journal/deliverables/docs/20_domain/domain_model.md:39-46`
  - `dev-journal/deliverables/docs/20_domain/domain_model.md:134`
- 同じ文書の不変条件では `TNT-001` を「全テーブルに tenant_id が存在する」と定義している
  - `dev-journal/deliverables/docs/20_domain/domain_model.md:349`
- 品質チェックでも「全エンティティに tenant_id」と自己判定しており、自己整合していない
  - `dev-journal/deliverables/docs/20_domain/domain_model.md:488`
- Step 1 の要件定義も同じ `TNT-001` 文言を前提にしている
  - `dev-journal/deliverables/docs/10_requirements/requirements.md:122`

## 判定
テナント分離の最重要不変条件の矛盾（中）。

## 修正方針案
`TNT-001` を「テナント境界を持つ業務テーブルには tenant_id 必須」に修正するか、`User` を例外として明記する。合わせて品質チェックと判断ログも同じ前提に揃える。
