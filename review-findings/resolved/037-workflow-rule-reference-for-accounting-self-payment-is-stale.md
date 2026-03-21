# 037: workflow.md の Accounting 自己処理禁止ルール参照が旧IDのまま

## 指摘概要
Issue 024 の修正で Accounting の自己処理禁止ルール自体は `requirements.md` と `rbac.md` に反映されたが、`workflow.md` の `approved → paid` 遷移だけが事前条件の参照先を存在しない `RBC-017` のまま残している。今回の差分は自己処理禁止ルールの導入・整理を目的としているため、状態遷移定義でも同じルールIDに収束していないと Step 1 内のトレーサビリティが再び崩れる。

## 根拠
- [dev-journal/deliverables/docs/10_requirements/workflow.md](/root-project/dev-journal/deliverables/docs/10_requirements/workflow.md#L77) `T4` の事前条件が「自己処理禁止、RBC-017」と記載している
- [dev-journal/deliverables/docs/10_requirements/rbac.md](/root-project/dev-journal/deliverables/docs/10_requirements/rbac.md#L147) 自己処理禁止を含む支払完了ルールは `RBC-012`
- [dev-journal/deliverables/docs/10_requirements/rbac.md](/root-project/dev-journal/deliverables/docs/10_requirements/rbac.md#L155) 特殊ルールでも「Accounting の自己処理禁止」として定義済み
- [dev-journal/deliverables/docs/10_requirements/requirements.md](/root-project/dev-journal/deliverables/docs/10_requirements/requirements.md#L244) 承認フロー要件では同ルールを `WFL-013` として定義済み

## 判定
中 / ルールIDトレーサビリティ不整合

## 修正方針案
`workflow.md` の `T4` 事前条件にある `RBC-017` を、実際に存在する参照先へ更新する。RBAC 制約を参照したいなら `RBC-012`、ワークフロー要件を参照したいなら `WFL-013` に統一し、Step 1 文書間で同じ禁止ルールを同じIDで追跡できるようにする。

## 対応内容
自己処理禁止を RBC-012 に統一。WFL-013 は元の遷移ルール定義（approved → paid の実行主体は Accounting）に復元。workflow.md, requirements.md, domain_model.md, state_machine.md の全参照を RBC-012 に変更済み。
