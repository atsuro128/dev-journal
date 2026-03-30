# 開発環境構築 + DB

- 担当: platform-builder
- 依存: なし
- ブランチ: main 直接（済）
- 出力先: expense-saas/ (docker-compose.yml, db/migrations/, scripts/)
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| DB スキーマ | deliverables/docs/50_detail_design/db_schema.md | 全テーブル DDL |
| アーキテクチャ設計 | deliverables/docs/30_arch/architecture.md | §インフラ構成 |

## 責務

- コンテナ構成（PostgreSQL + API・フロントエンドのサービス定義枠）の docker-compose.yml
- db_schema.md に基づく全テーブルの DDL マイグレーション（golang-migrate 形式）
- テナント分離設定（RLS ポリシー、app_user ロール、owner ロール）
- DB ロール・権限作成
- 初期データ投入（カテゴリマスタ等）
- 判断ポイント: 開発環境の複数インスタンス対応（ポート競合回避、DB分離、環境変数管理）
- 含めない: バックエンド/フロントエンドのアプリケーションコード

## 完了条件

- docker compose up で db サービスが起動する（api・frontend は 8-2/8-3 完了後に起動可能になる）
- マイグレーションが正常終了する
- RLS ポリシーが適用されている
- 初期データが投入されている
