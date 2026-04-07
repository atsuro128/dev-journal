# ダッシュボードテスト

- 担当: test-implementer
- 依存: 9-A
- ブランチ: `step9/9-C-dashboard-test`
- 出力先: expense-saas/ 内のバックエンド・フロントエンドテストディレクトリ
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| テストケース（ダッシュボード） | deliverables/docs/60_test/test_cases/dashboard.md | 全テストケース |
| API 契約 | deliverables/docs/50_detail_design/openapi.yaml | ダッシュボードエンドポイント |
| UI コンポーネント設計（ダッシュボード） | deliverables/docs/55_ui_component/screens/dashboard.md | コンポーネント単位・Props |

## 責務

- dashboard.md の全テストケースをバックエンド・フロントエンドのテストコードとして実装する
- 集計値テスト、ロール別表示テスト、テナント分離テスト
- FE コンポーネント単体テスト: dashboard.md に記載のコンポーネント・Props 単位で導出
- 9-A で作成された共通フィクスチャ・ヘルパーを利用する（重複作成しない）
- 含めない: レポート・明細等の個別機能テスト、機能実装

## 完了条件

- dashboard.md の全テストケースに対応するテストコードが存在する
- テストケース ID とテストコードの対応が追跡可能である
- 全テストがコンパイルを通過し、実行可能な状態（全テスト失敗）
