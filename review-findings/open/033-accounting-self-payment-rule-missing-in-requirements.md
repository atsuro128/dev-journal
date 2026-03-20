# 033: Accounting の自己処理禁止が requirements.md に未反映

## 指摘概要
Issue 024 の提案には「Accounting は自分が作成したレポートの支払完了を記録できない」という自己処理禁止ルールの追加が含まれているが、`requirements.md` の承認フロー要件にはこの制約が反映されていない。`rbac.md`・`usecases.md`・`workflow.md` では禁止ルールが明記されており、Step 1 の要件定義書だけが取り残されている。

## 根拠
- [dev-journal/deliverables/docs/10_requirements/requirements.md](/root-project/dev-journal/deliverables/docs/10_requirements/requirements.md#L218) `WFL-F03` は「Accounting が approved レポートに支払完了を記録」とだけ記載している
- [dev-journal/deliverables/docs/10_requirements/requirements.md](/root-project/dev-journal/deliverables/docs/10_requirements/requirements.md#L238) 承認フローのビジネスルールに自己処理禁止が存在しない
- [dev-journal/deliverables/docs/10_requirements/requirements.md](/root-project/dev-journal/deliverables/docs/10_requirements/requirements.md#L254) MVP スコープ判断には Approver の自己承認禁止のみがあり、Accounting の自己処理禁止が未記載
- [dev-journal/deliverables/docs/10_requirements/rbac.md](/root-project/dev-journal/deliverables/docs/10_requirements/rbac.md#L96) `支払完了` の補足として「自分が作成したレポートの支払完了は記録不可」と定義済み
- [dev-journal/deliverables/docs/10_requirements/usecases.md](/root-project/dev-journal/deliverables/docs/10_requirements/usecases.md#L355) `UC-AC02` の例外系で自分のレポートの支払処理を禁止している

## 判定
中 / 要件定義の内部不整合

## 修正方針案
`requirements.md` の承認フロー節に、Accounting の自己処理禁止をビジネスルールまたは MVP スコープ判断として追加する。`WFL-F03` の説明にも「自分以外が作成した approved レポートのみ」を明示すると後続設計への引き継ぎが安定する。
