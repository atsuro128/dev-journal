# 8-2 バックエンド初期化 レビュー結果

- レビュー日: 2026-03-30
- 対象コミット: 未コミット（ワーキングツリー上の変更）
- 判定: **FIX**

## 判定理由

go build / go vet は通り、ディレクトリ構成も architecture.md §3.1 と一致している。Dockerfile のマルチステージビルドと docker-compose.yml の変更も適切。ただし、チケットの責務欄に明記された依存パッケージが 3 つ不足しており、下流タスク（8-4 共通ミドルウェア）が依存追加から始める必要がある。また、コード上の軽微な問題がある。

## レビュー観点と結果

### 1. バックエンドのビルドが通るか

`go build ./cmd/server` および `go vet ./...` がエラーなしで完了。**OK**。

### 2. ディレクトリ構成が architecture.md §3.1 と一致するか

architecture.md §3.1 で定義されている internal/ 配下のディレクトリ:

| architecture.md §3.1 | 実装 | 状態 |
|---|---|---|
| cmd/server/main.go | 存在 | OK |
| internal/config/ | config.go が存在 | OK |
| internal/middleware/ | .gitkeep | OK |
| internal/handler/ | .gitkeep | OK |
| internal/service/ | .gitkeep | OK |
| internal/domain/ | .gitkeep | OK |
| internal/domain/model/ | .gitkeep | OK |
| internal/repository/postgres/ | .gitkeep | OK |
| internal/pkg/jwt/ | .gitkeep | OK |
| internal/pkg/s3/ | .gitkeep | OK |
| db/migrations/ | 8-1 で作成済み | OK（8-2 スコープ外） |
| db/queries/ | 未作成 | OK（8-6 スコープ） |
| db/sqlc.yaml | 未作成 | OK（8-6 スコープ） |

**OK** — 8-2 の責務範囲は全て満たしている。

### 3. docker-compose.yml の変更が最小限か

差分は api サービスの `volumes: [".:/app"]` の削除のみ。Dockerfile でバイナリをビルド・コピーする方式への移行に伴う正当な変更。**OK**。

### 4. Dockerfile がマルチステージビルドで適切に構成されているか

- ビルドステージ: `golang:1.24-alpine` で依存ダウンロード後にバイナリをビルド
- ランタイムステージ: `alpine:3.21` で ca-certificates のみ追加、バイナリをコピー
- `CGO_ENABLED=0` で静的リンク

**OK**。

### 5. 環境変数の管理が .env.example / docker-compose.yml と整合しているか

| 環境変数 | config.go | docker-compose.yml | .env.example | 整合性 |
|---|---|---|---|---|
| DATABASE_URL | 必須 | あり | あり | OK |
| APP_DATABASE_URL | 必須 | あり | あり | OK |
| JWT_PRIVATE_KEY_PATH | 任意 | あり | あり | OK |
| JWT_PUBLIC_KEY_PATH | 任意 | あり | あり | OK |
| CORS_ALLOWED_ORIGINS | 任意 | あり | あり | OK |
| LOG_LEVEL | 任意（default: info） | あり（default: debug） | あり | OK |
| PORT | 任意（default: 8080） | あり（default: 8080） | あり | OK |

**OK**。

### 6. cmd/server/main.go の実装が 8-2 のスコープ内に収まっているか

main.go の実装内容:
1. 設定読み込み — OK（8-2 責務）
2. 構造化ログ設定 — OK（初期化の一部として妥当）
3. DB 接続プール作成 + Ping — OK（起動確認に必要）
4. Chi ルーター作成 — OK（最小限のルーター初期化）
5. ヘルスチェックエンドポイント（静的 OK レスポンス） — OK（docker compose up での起動確認用。DB ステータス付きヘルスチェックは 8-4 の責務）
6. HTTP サーバー起動 + グレースフルシャットダウン — OK

ミドルウェア登録やハンドラ実装は含まれていない。**OK**。

## 指摘事項

| # | チェック種別 | 対象ファイル | 問題内容 | 重大度 | 推奨対応 |
|---|---|---|---|---|---|
| 1 | チケット責務との整合 | go.mod | チケットの責務欄に「go.mod 初期化、必要な依存追加（chi, pgx, golang-migrate, golang-jwt, validator）」と明記されているが、golang-migrate (`github.com/golang-migrate/migrate/v4`)、golang-jwt (`github.com/golang-jwt/jwt/v5`)、validator (`github.com/go-playground/validator/v10`) が go.mod に含まれていない。8-4（共通ミドルウェア）が JWT 検証で golang-jwt を、8-6（スケルトン）が入力バリデーションで validator を必要とするため、下流タスクで依存追加が発生する。 | warning | 3 パッケージを go.mod に追加する。実際に import するコードがなければ `tools.go` パターン（`//go:build tools` + blank import）で保持するか、下流タスクに委譲する旨をチケットに明記する。 |
| 2 | コード品質 | cmd/server/main.go:49,97 | `pool.Close()` が defer（L49）と明示呼び出し（L97）で二重に実行される。pgxpool は二重 Close でパニックしないが、冗長であり意図が不明確。 | info | L97 の明示的な `pool.Close()` を削除し、defer に統一する。 |
| 3 | ビルド出力の gitignore | .gitignore | `go build ./cmd/server` をプロジェクトルートで実行すると `server` バイナリが生成されるが、.gitignore の `bin/` パターンではカバーされない。誤コミットのリスクがある。 | info | .gitignore に `server` または `/server` を追加する。あるいは Makefile で `-o bin/server` を指定するよう統一する。 |

## まとめ

| 観点 | 結果 |
|---|---|
| go build が通る | PASS |
| go vet が通る | PASS |
| ディレクトリ構成が architecture.md §3.1 と一致 | PASS |
| docker-compose.yml の変更が最小限 | PASS |
| Dockerfile が適切に構成 | PASS |
| 環境変数が .env.example / docker-compose.yml と整合 | PASS |
| スコープ内に収まっている（8-4/8-6 の実装を含まない） | PASS |
| チケット責務の依存パッケージが全て追加されている | FAIL（3 パッケージ不足） |

## 判定

**FIX** — blocker レベルの問題はない。指摘 #1（go.mod の依存不足）は warning であり、下流タスク（8-4）の着手時に追加すれば技術的には問題ないが、チケットの責務として明記されている以上、8-2 の完了前に対応するか、意図的に下流に委譲する場合はその旨を記録すべき。指摘 #2, #3 は info レベルであり、対応は任意。
