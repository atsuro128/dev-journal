# 明細

- 担当: backend-developer, frontend-developer
- 依存: 10-B
- ブランチ: `step10/10-E-items`
- 出力先: expense-saas/ 内のバックエンド・フロントエンドディレクトリ
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| API 契約 | deliverables/docs/50_detail_design/openapi.yaml | 明細エンドポイント（/reports/{id}/items/*） |
| DB スキーマ | deliverables/docs/50_detail_design/db_schema.md | expense_items |
| 画面仕様（レポート詳細） | deliverables/docs/50_detail_design/screens/report-detail.md | 明細セクション |
| UI コンポーネント設計（レポート系） | deliverables/docs/55_ui_component/screens/report-detail.md | 明細コンポーネントツリー・Props 型 |
| テストケース（明細） | deliverables/docs/60_test/test_cases/items.md | 全テストケース |
| BE テストコード | expense-saas/internal/domain/expense_item_test.go, internal/handler/item_handler_test.go | 明細テスト |
| FE テストコード（部品） | expense-saas/frontend/src/pages/reports/__tests__/ItemForm.test.tsx, ItemListHeader.test.tsx, ItemListSection.test.tsx, ItemSlidePanel.test.tsx, ItemTable.test.tsx | コンポーネントテスト |
| FE テストコード（Hooks） | expense-saas/frontend/src/hooks/__tests__/useItems.test.tsx | 明細 Hook |

## 責務

- 明細 CRUD API の機能実装（作成・更新・削除）
- 明細追加/編集 UI の機能実装（スライドパネル、フォーム、一覧テーブル）
- 明細の金額計算・合計金額の自動更新
- 含めない: レポート CRUD（10-B）、添付ファイル（10-G）

## 完了条件

- test_cases/items.md の全テストケースが通過している
- Step 9 で実装済みの明細関連テストコード（BE/FE）が全て PASS する
