# ワークフローテスト

- 担当: test-implementer
- 依存: 9-B
- ブランチ: `step9/9-F-workflow-test`
- 出力先: expense-saas/ 内のバックエンド・フロントエンドテストディレクトリ
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| テストケース（ワークフロー） | deliverables/docs/60_test/test_cases/workflow.md | 全テストケース |
| API 契約 | deliverables/docs/50_detail_design/openapi.yaml | ワークフローエンドポイント |
| 認可設計 | deliverables/docs/50_detail_design/authz.md | RBAC・承認権限 |
| 状態遷移設計 | deliverables/docs/20_domain/state_machine.md | 承認フロー状態遷移 |
| UI コンポーネント設計（ワークフロー系） | deliverables/docs/55_ui_component/screens/workflow-*.md | コンポーネント単位・Props |

## 責務

- workflow.md の全テストケースをバックエンド・フロントエンドのテストコードとして実装する
- 承認/却下/差戻し/支払完了テスト、自己承認禁止テスト、RBAC テスト
- FE コンポーネント単体テスト: workflow-*.md に記載のコンポーネント・Props 単位で導出
- 9-A の共通フィクスチャ・ヘルパーを利用する（重複作成しない）
- 含めない: レポート CRUD・明細関連テスト、機能実装

## 完了条件

- workflow.md の全テストケースに対応するテストコードが存在する
- テストケース ID とテストコードの対応が追跡可能である
- 全テストがコンパイルを通過し、実行可能な状態（全テスト失敗）
