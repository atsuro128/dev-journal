# 明細テスト

- 担当: test-implementer
- 依存: 9-B
- ブランチ: `step9/9-E-items-test`
- 出力先: expense-saas/ 内のバックエンド・フロントエンドテストディレクトリ
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| テストケース（明細） | deliverables/docs/60_test/test_cases/items.md | 全テストケース |
| API 契約 | deliverables/docs/50_detail_design/openapi.yaml | 明細エンドポイント |
| DB スキーマ | deliverables/docs/50_detail_design/db_schema.md | items テーブル関連 |
| UI コンポーネント設計（レポート詳細） | deliverables/docs/55_ui_component/screens/report-detail.md | 明細関連コンポーネント |

## 責務

- items.md の全テストケースをバックエンド・フロントエンドのテストコードとして実装する
- 明細 CRUD テスト、カテゴリ選択テスト、金額バリデーションテスト、関連エンティティ連動削除テスト
- FE コンポーネント単体テスト: report-detail.md の明細関連コンポーネント・Props 単位で導出
- 9-A の共通フィクスチャ・ヘルパーを利用する（重複作成しない）
- 含めない: レポート本体・添付ファイル関連テスト、機能実装

## 完了条件

- items.md の全テストケースに対応するテストコードが存在する
- テストケース ID とテストコードの対応が追跡可能である
- 全テストがコンパイルを通過し、実行可能な状態（全テスト失敗）
