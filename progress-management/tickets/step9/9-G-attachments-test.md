# 添付ファイルテスト

- 担当: test-implementer
- 依存: 9-E
- ブランチ: `step9/9-G-attachments-test`
- 出力先: expense-saas/ 内のバックエンド・フロントエンドテストディレクトリ
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| テストケース（添付ファイル） | deliverables/docs/60_test/test_cases/attachments.md | 全テストケース |
| API 契約 | deliverables/docs/50_detail_design/openapi.yaml | 添付ファイルエンドポイント |
| 添付ファイル設計 | deliverables/docs/50_detail_design/files.md | MIME/サイズ制限、署名付き URL |
| 認可設計 | deliverables/docs/50_detail_design/authz.md | RBAC・テナント分離 |
| UI コンポーネント設計（レポート詳細） | deliverables/docs/55_ui_component/screens/report-detail.md | 添付関連コンポーネント |

## 責務

- attachments.md の全テストケースをバックエンド・フロントエンドのテストコードとして実装する
- MIME/サイズ制限テスト、署名付き URL テスト、認可テスト、テナント分離テスト
- FE コンポーネント単体テスト: report-detail.md の添付関連コンポーネント・Props 単位で導出
- 9-A の共通フィクスチャ・ヘルパーを利用する（重複作成しない）
- 含めない: 明細本体・レポート関連テスト、機能実装

## 完了条件

- attachments.md の全テストケースに対応するテストコードが存在する
- テストケース ID とテストコードの対応が追跡可能である
- 全テストがコンパイルを通過し、実行可能な状態（全テスト失敗）
