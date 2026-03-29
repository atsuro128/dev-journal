# バックエンド初期化

- 担当: backend-developer
- 依存: 8-1
- ブランチ: main 直接
- 出力先: expense-saas/ (cmd/, internal/, go.mod)
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| アーキテクチャ設計 | deliverables/docs/30_arch/architecture.md | §ディレクトリ構成 |

## 責務

- エントリポイント（cmd/server/main.go）
- レイヤー構造のディレクトリ作成（internal/handler, service, domain, repository）
- 環境変数・設定管理（internal/config/）— DB接続情報、認証鍵パス、CORS許可オリジン、ログレベル
- go.mod 初期化、必要な依存追加（chi, pgx, golang-migrate, golang-jwt, validator）
- 含めない: ミドルウェア実装（8-4）、ハンドラ実装（8-6）

## 完了条件

- go build が通る
- ディレクトリ構成が architecture.md §3.1 と一致する
