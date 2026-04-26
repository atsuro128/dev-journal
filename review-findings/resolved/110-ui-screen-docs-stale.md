severity: blocker

[55_ui_component/screens/report-list.md](/root-project/dev-journal/deliverables/docs/55_ui_component/screens/report-list.md:44) ほか 4 画面の UI 画面設計が `AppPagination` 直置き・旧状態管理のままです。`AppPaginationFooter`/`PageSizeSelector`、`per_page`、AllReportsPage の URL 駆動化へ更新しないと設計成果物間で矛盾が残ります。
