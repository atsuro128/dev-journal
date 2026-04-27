severity: warning

[progress.md](/root-project/dev-journal/progress-management/progress.md:58) の issue #147 再掲載文言が「`AppPaginationFooter` を MUI `<TableFooter>` 内に統合する」のままです。しかし確定方針は [issue #147](/root-project/dev-journal/issues/open/147-per-page-ui-selector-and-paginated-list-pagination-footer.md:386) にある通り、`<TableFooter>` は `DataGrid` 構造上不可で、`AppDataGrid` の `slots.footer` から `MuiDataGrid-footerContainer` に統合する D-1 です。進捗表だけ旧案の表現が残ると、実装担当が誤った統合先を参照するので、`slots.footer` / DataGrid フッターコンテナ統合へ文言を合わせてください。
