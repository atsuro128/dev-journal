# テナント管理テスト

- 担当: test-implementer
- 依存: 9-A
- ブランチ: `step9/9-D-tenant-test`
- 出力先: expense-saas/ 内のバックエンド・フロントエンドテストディレクトリ
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| テストケース（テナント管理） | deliverables/docs/60_test/test_cases/tenant.md | 全テストケース |
| API 契約 | deliverables/docs/50_detail_design/openapi.yaml | テナントエンドポイント |
| 認可設計 | deliverables/docs/50_detail_design/authz.md | RBAC・テナント分離 |
| UI コンポーネント設計（管理系） | deliverables/docs/55_ui_component/screens/admin-*.md | コンポーネント単位・Props |

## 責務

- tenant.md の全テストケースをバックエンド・フロントエンドのテストコードとして実装する
- メンバー一覧テスト、管理者専用アクセステスト、テナント分離テスト
- FE コンポーネント単体テスト: admin-*.md に記載のコンポーネント・Props 単位で導出
- 9-A で作成された共通フィクスチャ・ヘルパーを利用する（重複作成しない）
- 含めない: テナント管理以外の機能テスト、機能実装

## 完了条件

- tenant.md の全テストケースに対応するテストコードが存在する
- テストケース ID とテストコードの対応が追跡可能である
- 全テストがコンパイルを通過し、実行可能な状態（全テスト失敗）
