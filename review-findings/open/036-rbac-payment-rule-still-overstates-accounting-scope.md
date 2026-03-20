# 036: rbac.md の RBC-012 が Accounting の支払完了権限を過大に記述している

## 指摘概要
Issue 024 の対応で `rbac.md` には「Accounting の自己処理禁止」が追加されたが、所有権ルール表の `RBC-012` は依然として「Accounting は同テナントの approved レポートに支払完了を記録可能」とだけ記載している。権限マトリクス・特殊ルール・`requirements.md`・`workflow.md` はいずれも「自分以外が作成したレポートのみ」に制限しており、`RBC-012` だけが広すぎる定義になっている。

## 根拠
- [dev-journal/deliverables/docs/10_requirements/rbac.md](/root-project/dev-journal/deliverables/docs/10_requirements/rbac.md#L96) 権限マトリクスの `支払完了` は「自分が作成したレポートの支払完了は記録不可」
- [dev-journal/deliverables/docs/10_requirements/rbac.md](/root-project/dev-journal/deliverables/docs/10_requirements/rbac.md#L147) `RBC-012` は「同テナントの approved レポートに支払完了を記録可能」とだけ記載
- [dev-journal/deliverables/docs/10_requirements/rbac.md](/root-project/dev-journal/deliverables/docs/10_requirements/rbac.md#L155) 特殊ルールでは「Accounting は自分が作成したレポートの支払完了を記録できない」と定義
- [dev-journal/deliverables/docs/10_requirements/requirements.md](/root-project/dev-journal/deliverables/docs/10_requirements/requirements.md#L220) `WFL-F03` は「自分以外が作成した approved レポート」と明記

## 判定
中 / RBAC ルール定義の内部不整合

## 修正方針案
`RBC-012` を「Accounting は同テナントの approved レポートのうち、自分以外が作成したものに限り支払完了を記録可能」のように更新し、権限マトリクスと特殊ルールの条件をルール表にも反映する。
