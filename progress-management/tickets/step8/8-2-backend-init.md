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
- go.mod 初期化、必要な依存追加（chi, pgx）
  - golang-jwt → 8-4（共通ミドルウェア）で import 時に追加
  - validator → 8-6（スケルトン）で import 時に追加
  - golang-migrate → CLI 利用のため go.mod に不要（Makefile から直接実行）
- api 用 Dockerfile の作成（docker-compose.yml の api サービスが起動可能になる）
- 含めない: ミドルウェア実装（8-4）、ハンドラ実装（8-6）

## 完了条件

- go build が通る
- ディレクトリ構成が architecture.md §3.1 と一致する
- docker compose up で api サービスが起動する
