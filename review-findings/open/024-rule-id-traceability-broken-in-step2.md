---
step: 2
severity: medium
status: open
---

# 024: Step 2 で上流と異なる意味のルールIDを再利用している

## 指摘概要

`20_domain/` の文書が、Step 1 で定義済みのルールIDを別の意味で再利用している。これにより、上流のビジネスルールと Step 2 の不変条件のトレーサビリティが壊れている。

## 根拠

- 上流の `04_business-rules.md` では `WFL-013` は `approved → paid` の遷移ルールとして定義されている
  - `dev-journal/deliverables/docs/10_requirements/preliminary/04_business-rules.md:127`
- しかし Step 2 では `WFL-013` を「同一テナントに Approver が1人以上存在すること」の提出前提条件として使っている
  - `dev-journal/deliverables/docs/20_domain/domain_model.md:335`
  - `dev-journal/deliverables/docs/20_domain/state_machine.md:67`
- 上流の `04_business-rules.md` では `RBC-014` は「Admin は自分が作成したレポートのみ申請者として操作可能。他者のレポートは閲覧のみ」のルールである
  - `dev-journal/deliverables/docs/10_requirements/preliminary/04_business-rules.md:189`
- しかし Step 2 では `RBC-014` を「Approver の自己承認・自己却下は禁止」の意味で使っている
  - `dev-journal/deliverables/docs/20_domain/domain_model.md:373`
  - `dev-journal/deliverables/docs/20_domain/state_machine.md:93`
  - `dev-journal/deliverables/docs/20_domain/state_machine.md:125`
- 自己承認禁止は Step 1 で要件としては存在するが、ID付きルールとしては定義されていない
  - `dev-journal/deliverables/docs/10_requirements/rbac.md:149`
  - `dev-journal/deliverables/docs/10_requirements/requirements.md:255`

## 影響

- Step 2 から Step 5/6 にルールIDでトレースした際に、別ルールとして誤実装・誤テスト化する
- 上流で定義済みの `RBC-014`（Admin の二面性）が Step 2 の不変条件上で行方不明になる
- レビュー観点で求められている「ビジネスルールIDの対応」が成立しない

## 修正方針案

- 既存の `WFL-013` / `RBC-014` は上流と同じ意味に戻す
- 「Approver が0人なら提出不可」「自己承認・自己却下禁止」を ID 管理したい場合は、Step 1 側で新しいルールIDを採番してから Step 2 に反映する
- そうしない場合でも、Step 2 では新規概念に既存IDを流用しない
