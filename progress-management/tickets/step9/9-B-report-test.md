# レポートテスト

- 担当: test-implementer
- 依存: 9-A
- ブランチ: `step9/9-B-report-test`
- 出力先: expense-saas/ 内のバックエンド・フロントエンドテストディレクトリ
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| テストケース（レポート） | deliverables/docs/60_test/test_cases/reports.md | 全テストケース |
| API 契約 | deliverables/docs/50_detail_design/openapi.yaml | レポートエンドポイント |
| 状態遷移設計 | deliverables/docs/20_domain/state_machine.md | レポート状態遷移 |
| 認可設計 | deliverables/docs/50_detail_design/authz.md | RBAC・テナント分離 |
| UI コンポーネント設計（レポート系） | deliverables/docs/55_ui_component/screens/report-*.md | コンポーネント単位・Props |

## 責務

- reports.md の全テストケースをバックエンド・フロントエンドのテストコードとして実装する
- レポート CRUD テスト、状態遷移全パターンテスト、テナント分離テスト、RBAC テスト
- FE コンポーネント単体テスト: report-*.md に記載のコンポーネント・Props 単位で導出
- 9-A で作成された共通フィクスチャ・ヘルパーを利用する（重複作成しない）
- 含めない: 明細・添付ファイル・ワークフロー関連テスト、機能実装

## 完了条件

- reports.md の全テストケースに対応するテストコードが存在する
- テストケース ID とテストコードの対応が追跡可能である
- 全テストがコンパイルを通過し、実行可能な状態（全テスト失敗）
