# 035: usecases.md の一覧サマリが Accounting の Member 相当フローをまだ除外している

## 指摘概要
Issue 024 の再修正で `UC-M08` と `UC-M09` 本文のアクター定義は `Accounting` を含む形に更新されたが、`usecases.md` 末尾のユースケース一覧サマリは依然として両ユースケースを `Member` のみとして記載している。Step 1 の同一成果物内で actor 定義が二重化されているため、参照箇所によって解釈が分かれる状態が残っている。

## 根拠
- [dev-journal/deliverables/docs/10_requirements/usecases.md](/root-project/dev-journal/deliverables/docs/10_requirements/usecases.md#L217) `UC-M08` の本文アクターは `Member（Approver/Admin/Accounting も自分のレポートで実行可能）`
- [dev-journal/deliverables/docs/10_requirements/usecases.md](/root-project/dev-journal/deliverables/docs/10_requirements/usecases.md#L234) `UC-M09` の本文アクターは `Member（Approver/Admin/Accounting も自分のレポートで実行可能）`
- [dev-journal/deliverables/docs/10_requirements/usecases.md](/root-project/dev-journal/deliverables/docs/10_requirements/usecases.md#L586) ユースケース一覧サマリでは `UC-M08` のアクターが `Member` のみ
- [dev-journal/deliverables/docs/10_requirements/usecases.md](/root-project/dev-journal/deliverables/docs/10_requirements/usecases.md#L587) ユースケース一覧サマリでは `UC-M09` のアクターが `Member` のみ

## 判定
中 / Step 1 文書内の内部不整合

## 修正方針案
`usecases.md` の一覧サマリでも、`UC-M08` と `UC-M09` が `Approver/Admin/Accounting` の自己申請フローを含むことが読み取れる表記にそろえる。少なくとも本文との齟齬が出ない actor 表現に統一する。

## 対応内容
usecases.md のサマリ表で UC-M01〜UC-M09, UC-M03a のアクター列を本文と整合させた。
