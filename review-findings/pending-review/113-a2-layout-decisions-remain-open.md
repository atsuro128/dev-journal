# 113: A2 案の見た目調整が未確定のまま実装判断に委ねられている

## 指摘概要
issue #147 の再々オープンでは A2 案「独自実装維持 + 見た目を MUI 標準に寄せる」が確定方針になっているが、`PageSizeSelector` の `variant` と `AppPaginationFooter` の中央寄せ手法が設計上まだ未確定で、実装フェーズでの判断に委ねられている。これでは「どの見た目を正とするか」が文書だけでは閉じず、同じ設計書を読んでも実装者ごとに別解を取りうる。

## 根拠
- issue 本文では A2 案を「確定方針」とし、完了条件にも「4 画面のフッター見栄えが MUI 標準に近い」を置いている: [dev-journal/issues/open/147-per-page-ui-selector-and-paginated-list-pagination-footer.md](/root-project/dev-journal/issues/open/147-per-page-ui-selector-and-paginated-list-pagination-footer.md:501), [dev-journal/issues/open/147-per-page-ui-selector-and-paginated-list-pagination-footer.md](/root-project/dev-journal/issues/open/147-per-page-ui-selector-and-paginated-list-pagination-footer.md:544)
- しかし `PageSizeSelector` は `outlined` を初期方針としつつ、収まらなければ `standard` への切り替えを「検討」「実装フェーズで確定」としている: [dev-journal/deliverables/docs/55_ui_component/common-components.md](/root-project/dev-journal/deliverables/docs/55_ui_component/common-components.md:336)
- `AppPaginationFooter` も `flex` で中央寄せを保証すると書きながら、期待通り作用しない場合は CSS Grid への切り替えを「検討」「実装時に確定」としている: [dev-journal/deliverables/docs/55_ui_component/common-components.md](/root-project/dev-journal/deliverables/docs/55_ui_component/common-components.md:416)
- 4 画面詳細設計と state-management は `common-components.md` を参照するだけで、この未確定事項を解消していない: [dev-journal/deliverables/docs/50_detail_design/screens/report-list.md](/root-project/dev-journal/deliverables/docs/50_detail_design/screens/report-list.md:108), [dev-journal/deliverables/docs/55_ui_component/state-management.md](/root-project/dev-journal/deliverables/docs/55_ui_component/state-management.md:240)

## 判定
中 / FIX

## 修正方針案
- `PageSizeSelector` の見た目を `outlined` と `standard` のどちらで確定するかを設計書で明示し、`margin`/`sx` 含めて実装者に判断を残さない
- `AppPaginationFooter` の中央寄せ手法も `flex` か `grid` のどちらかに確定し、代替案の記述は削除する
- その確定仕様に合わせて 4 画面詳細設計とテストケースの期待結果を同期させる
