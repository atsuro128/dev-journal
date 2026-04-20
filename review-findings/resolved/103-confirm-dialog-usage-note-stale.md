# 103: ConfirmDialog のマトリクス補足が ITM-007 用途を含んでいない

## 指摘概要

`common-components.md` の `ConfirmDialog` 責務欄には、明細保存時の期間外警告（ITM-007）で使用することが追記されている。一方で、同ファイルのマトリクス補足では `ConfirmDialog` の用途を「7種類の確認操作」と説明しており、今回追加された ITM-007 の期間外警告が含まれていない。

このままだと、同一ファイル内で `ConfirmDialog` の責務・使用範囲が食い違い、Step 10 の実装者が共通コンポーネントの対象用途を読み違える可能性がある。

## 根拠

- `dev-journal/deliverables/docs/55_ui_component/common-components.md:203`
  - 責務欄では「明細保存時の期間外警告（ITM-007）」を含めている
- `dev-journal/deliverables/docs/55_ui_component/common-components.md:579`
  - マトリクス補足では「提出・削除・承認・却下・支払完了・明細削除・添付削除の7種類」と記載しており、ITM-007 が含まれていない

Step 5.5 レビュー観点:

- `common-components.md` の共通コンポーネントが `screens/*.md` で正しく参照されているか
- 同一コンポーネントの Props 定義・責務が成果物間で矛盾していないか

## 判定

重大度: 低

分類: UI コンポーネント設計 / 内部整合性

## 修正方針案

マトリクス補足の `ConfirmDialog` 説明に「明細保存時の期間外警告（ITM-007）」を追加し、件数表現を避けるか、実際の用途数に合わせる。例: 「提出・削除・承認・却下・支払完了・明細削除・添付削除・明細保存時の期間外警告（ITM-007）などの確認操作を担う」。
