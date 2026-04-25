# 106: Step5.5 のレポートフォーム参照が V5-S/V5-E に追随していない

## 指摘概要
issue #141 で対象期間バリデーションを `V5-S` / `V5-E` に分割した一方、Step 5.5 の `55_ui_component/screens/report-create.md` と `55_ui_component/screens/report-edit.md` では引き続き `V1-V5` と記載されている。  
このままだと、UI コンポーネント設計だけ読む後続担当者には「対象期間の相関バリデーションは 1 件なのか、2 経路なのか」が判別できず、採用方針の「両フィールド blur で両経路を再評価する」という設計意図を取りこぼす。

## 根拠
- `deliverables/docs/50_detail_design/screens/report-create.md:94-97`
  - `V5-S` / `V5-E` を別行で定義し、両フィールド `onBlur` で `trigger(['periodStart', 'periodEnd'])` を呼ぶことを明記
- `deliverables/docs/50_detail_design/screens/report-edit.md:78-89`
  - 編集画面側も同じく `V5-S` / `V5-E` を正本化
- `deliverables/docs/55_ui_component/screens/report-create.md:168,184`
  - `V1-V5: クライアントサイド Zod バリデーション`
  - `フォーム入力・バリデーション（V1-V5）`
- `deliverables/docs/55_ui_component/screens/report-edit.md:177`
  - `V1-V5: 作成画面と同一ルール`

## 判定
重大度: 中  
分類: Step 5.5 の上流整合性 / 設計書間参照整合

## 修正方針案
- `55_ui_component/screens/report-create.md` の §8 / §9 を `V1-V4, V5-S, V5-E` へ更新する
- `55_ui_component/screens/report-edit.md` の §8 も同様に更新する
- 可能なら `ReportForm` / `ReportPeriodField` の説明にも「両フィールド blur で `periodStart` / `periodEnd` を再評価する」旨を 1 行追記し、Step 5.5 単体でも issue #141 の設計意図を追跡できるようにする
