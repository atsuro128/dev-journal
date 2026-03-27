# コード生成・スケルトン

- 担当: backend-developer
- 依存: 8-4
- 出力先: expense-saas/internal/ (handler, service, domain, repository), expense-saas/frontend/src/api/types.ts
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| API 契約 | deliverables/docs/50_detail_design/openapi.yaml | 全エンドポイント定義 |
| ドメインモデル | deliverables/docs/20_domain/domain_model.md | エンティティ、値オブジェクト、列挙型 |
| DB スキーマ | deliverables/docs/50_detail_design/db_schema.md | 全テーブル |

## 責務

- 全エンドポイントのハンドラメソッド（空実装: 501 Not Implemented を返す）
- サービス interface と空実装
- ドメインモデル定義（エンティティ、値オブジェクト、列挙型）
- sqlc クエリ定義と生成コード（全テーブルの基本 CRUD、テナント分離条件必須）
- API 定義の全スキーマからフロントエンド型を生成
- ルーティング定義（chi router に全エンドポイント登録）
- 含めない: ビジネスロジック実装（Step 10）

## 完了条件

- 全エンドポイント定義に対応するハンドラが存在する
- バックエンド・フロントエンドのビルドが通る
- 全テーブルの基本クエリにテナント分離条件が含まれる
