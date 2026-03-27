# 開発環境構築 + DB

- 担当: platform-builder
- 依存: なし
- 出力先: expense-saas/ (docker-compose.yml, Dockerfile, db/migrations/)
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| DB スキーマ | deliverables/docs/50_detail_design/db_schema.md | 全テーブル DDL |
| アーキテクチャ設計 | deliverables/docs/30_arch/architecture.md | §2 システム全体構成 |

## 責務

- コンテナ構成（PostgreSQL、Go API サーバー、フロントエンド開発サーバー）の docker-compose.yml
- db_schema.md に基づく全テーブルの DDL マイグレーション（golang-migrate 形式）
- テナント分離設定（RLS ポリシー、app_user ロール、owner ロール）
- DB ロール・権限作成
- 初期データ投入（カテゴリマスタ等）
- 判断ポイント: 開発環境の複数インスタンス対応（ポート競合回避、DB分離、環境変数管理）
- 含めない: バックエンド/フロントエンドのアプリケーションコード

## 完了条件

- docker compose up で全サービスが起動する
- マイグレーションが正常終了する
- RLS ポリシーが適用されている
- 初期データが投入されている
