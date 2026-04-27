# 114: APF-008〜010 では A2 案の高さ回帰を防げない

## 指摘概要
再々オープン理由は「境界線消失」と「Select がフッター高さを支配して見栄えが崩れた」ことだったが、追加採番された APF-008〜010 は境界線・件数表示・後方互換しか保証していない。A2 案で新たに正本化した `minHeight: 52` と `px: 2` / `py: 0.5`、およびそれにより高さ支配を防ぐ狙いに対する回帰テストがなく、再発防止として不十分。

## 根拠
- issue 本文の追加対応内容は `minHeight` と `px` / `py` 調整を、再々オープン理由 2 の解消策として明示している: [dev-journal/issues/open/147-per-page-ui-selector-and-paginated-list-pagination-footer.md](/root-project/dev-journal/issues/open/147-per-page-ui-selector-and-paginated-list-pagination-footer.md:516)
- `common-components.md` でも `minHeight: 52`、`px: 2`、`py: 0.5` を A2 案の正本値として記載している: [dev-journal/deliverables/docs/55_ui_component/common-components.md](/root-project/dev-journal/deliverables/docs/55_ui_component/common-components.md:410)
- しかし追加テスト APF-008〜010 は `borderTop`、件数表示算出、`totalCount` 未指定時の後方互換しか検証していない: [dev-journal/deliverables/docs/60_test/test_cases/reports.md](/root-project/dev-journal/deliverables/docs/60_test/test_cases/reports.md:622)
- 完了条件には SMK-081 PASS として「フッター高さが Select に支配されない」が入っており、本来は設計テストにもその回帰防止策が必要: [dev-journal/issues/open/147-per-page-ui-selector-and-paginated-list-pagination-footer.md](/root-project/dev-journal/issues/open/147-per-page-ui-selector-and-paginated-list-pagination-footer.md:544)

## 判定
中 / FIX

## 修正方針案
- APF-008 を拡張するか別テストを追加し、ルート `sx` に `minHeight: 52`, `px: 2`, `py: 0.5` が入ることを検証対象に含める
- `PageSizeSelector` 側にも、A2 案で確定した `size`/`variant`/余白がフッター高さを押し上げない前提を検証するケースを追加する
- 追加テストの説明文を「境界線」だけでなく「A2 案の視覚回帰防止」に更新し、再々オープン理由とのトレーサビリティを明示する
