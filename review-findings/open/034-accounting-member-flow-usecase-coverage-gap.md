# 034: Accounting の Member 相当フローが usecases.md で最後まで網羅されていない

## 指摘概要
Issue 024 の修正で Accounting に Member 相当の申請権限を付与している一方、`usecases.md` では自分のレポート状況確認と再申請が依然として Member 専用のままで、Accounting の申請フローが途中で切れている。Step 4 の画面設計では Accounting に `マイレポート` と `再申請` が開放されているため、Step 1 のユースケース定義が追随できていない。

## 根拠
- [dev-journal/deliverables/docs/10_requirements/usecases.md](/root-project/dev-journal/deliverables/docs/10_requirements/usecases.md#L217) `UC-M08` のアクターが `Member` のみ
- [dev-journal/deliverables/docs/10_requirements/usecases.md](/root-project/dev-journal/deliverables/docs/10_requirements/usecases.md#L234) `UC-M09` のアクターが `Member` のみ
- [dev-journal/deliverables/docs/10_requirements/usecases.md](/root-project/dev-journal/deliverables/docs/10_requirements/usecases.md#L518) `UC-SYS04` では Accounting に「直近のレポート一覧（自分）」を表示すると定義している
- [dev-journal/deliverables/docs/40_basic_design/screens.md](/root-project/dev-journal/deliverables/docs/40_basic_design/screens.md#L78) `SCR-RPT-001` は `Member, Approver, Admin, Accounting` に対応
- [dev-journal/deliverables/docs/40_basic_design/screens.md](/root-project/dev-journal/deliverables/docs/40_basic_design/screens.md#L94) `再申請` ボタンは `所有者かつステータスが rejected` で表示され、Accounting を除外していない
- [dev-journal/deliverables/docs/40_basic_design/screens.md](/root-project/dev-journal/deliverables/docs/40_basic_design/screens.md#L172) サイドナビで Accounting に `マイレポート` を表示している

## 判定
中 / ユースケース網羅漏れ

## 修正方針案
`UC-M08` と `UC-M09` のアクター記述を、少なくとも `Approver/Admin/Accounting も自分のレポートで同様に実行可能` に更新する。あわせてユースケース一覧サマリのアクター表記も、Member 相当の自己申請フローがロール横断で使えることが読み取れるよう補足する。
