# 045: 支払完了フロー図の status チェックが誤ったルールIDを参照している

## 指摘概要
`report-detail.md` の「状態遷移操作の権限チェックフロー」で、支払完了フローの `status = approved` 判定に `WFL-013` を付与しているが、上流の状態遷移定義ではこの条件のルールIDは `WFL-002` である。`WFL-013` は実行主体が Accounting であることを示すルールであり、状態条件のトレーサビリティが崩れている。

## 根拠
- `dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md:771-774`
  - `P_Role{ロール = Accounting か?<br/>WFL-013}`
  - `P_ChkStatus{status = approved か?<br/>WFL-013}`
- `dev-journal/deliverables/docs/20_domain/state_machine.md:151-158`
  - 実行者検証の責務は `WFL-013`
  - `status == approved` の事前条件は `WFL-002`

## 判定
重大度: 低
分類: トレーサビリティ不備 / ルールID誤参照

## 修正方針案
`P_ChkStatus` の注記を `WFL-002` に修正し、`P_Role` だけが `WFL-013` を参照する形に揃える。
