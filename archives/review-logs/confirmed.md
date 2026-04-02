# レビュー確認済み一覧

## Step 8: 基盤構築

- [2026-04-01] `.github/PULL_REQUEST_TEMPLATE.md` — 中身空、成果物外。保留
- [2026-04-01] `.github/workflows/ci.yml` — CI パイプライン確認
- [2026-04-01] `.github/workflows/deploy.yml` — デプロイパイプライン確認。ci.yml とジョブ二重の問題あり
- [2026-04-01] `docker-compose.yml` — 4サービス構成（db, db-test, api, frontend）
- [2026-04-01] docker compose up 起動確認 — .dockerignore 追加で解消
- [2026-04-01] `Dockerfile`（本番用）— マルチステージビルド、CGO_ENABLED=0、ca-certificates
- [2026-04-01] `frontend/Dockerfile.dev`（開発用）— Vite dev server 用
- [2026-04-01] `.env` / `.env.example` — 環境変数の仕組み
- [2026-04-02] `cmd/server/main.go` ステップ 1: 設定読み込み（config.LoadConfig、環境変数）
- [2026-04-02] `cmd/server/main.go` ステップ 2: ログ設定（slog、JSON、LOG_LEVEL 切替）
- [2026-04-02] `cmd/server/main.go` ステップ 3: DB コネクションプール作成（pgxpool、プール上限、共有の仕組み）
- [2026-04-02] `cmd/server/main.go` ステップ 4: DB 疎通確認（Ping、失敗時即終了）
- [2026-04-02] `cmd/server/main.go` ステップ 5: JWT Verifier 初期化（RS256、非 fatal → issue 052 で必須化予定）
- [2026-04-02] `cmd/server/main.go` ステップ 6: バックグラウンド Context（レートリミッター掃除用、トークンバケット方式）
- [2026-04-02] `cmd/server/main.go` ステップ 7: DI（Repository → Authorizer → Service → Handler）— 途中
