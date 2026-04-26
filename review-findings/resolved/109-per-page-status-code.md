severity: blocker

Q4 の不正 `per_page` 応答コードが不整合です。[screens.md](/root-project/dev-journal/deliverables/docs/40_basic_design/screens.md:266) と各画面詳細は 422、[state-management.md](/root-project/dev-journal/deliverables/docs/55_ui_component/state-management.md:260) は 400。確定方針に合わせて 1 値へ統一し、OpenAPI/テスト期待値も同時更新してください。
