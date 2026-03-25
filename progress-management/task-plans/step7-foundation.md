# Step 7 基盤構築 — 作業計画

## 判断ポイント（確定済み）

| # | 論点 | 決定 |
|---|------|------|
| 1 | ディレクトリ構成 | Go 標準の `cmd/server/` + `internal/` + `frontend/` を採用。work-breakdown の成果物表を修正済み |
| 2 | CI ステージ構成 | `lint` → `test` → `build` の3ステージ。lint: Go(golangci-lint) + TS(ESLint + Prettier check)。test: `go test ./...` + `npm run test`。build: `go build ./...` + `npm run build` |
| 3 | CI トリガー条件 | PR 作成・更新時: lint + 単体テスト + 統合テスト + build。main マージ後: 上記に加え E2E + レスポンスタイム スモークテスト（test_strategy.md 準拠。Step 7 時点ではテストコード未実装のため E2E/スモークのジョブは枠のみ用意、Step 9 以降で実装） |
| 4 | ブランチ保護ルール | main への直接 push 禁止、CI 全ステージ通過を必須 |
| 5 | デプロイ | MVP では自動デプロイしない。手動デプロイのみ |
| 6 | hooks: Edit/Write 後の自動 format | Go: `gofmt`/`goimports`、TS: `prettier --write`。失敗時は警告のみ（ブロックしない） |
| 7 | hooks: 危険変更前 lint | `go vet ./...` を commit 前に実行。失敗時はブロック。詳細な lint / 型チェックは GitHub Actions に委ねる |
| 8 | Dev Container ポート競合回避 | 環境変数（`API_PORT`, `WEB_PORT`, `DB_PORT`）でポート番号を可変に。デフォルト 8080/3000/5432 |
| 9 | Dev Container DB 分離 | インスタンスごとに独立した PostgreSQL コンテナ。DB 名は環境変数で指定（デフォルト: `expense_saas_dev`） |
| 10 | OpenAPI コード生成ツール | Go: `oapi-codegen`（型定義と interface 生成）。TS: `openapi-typescript`（API 型生成）。sqlc でクエリ生成 |
| 11 | SPA 配信方式（ローカル） | Vite dev server (port 3000) + Go API (port 8080) 別プロセス。Vite が `/api/*` と `/health` を Go にプロキシ。本番は `go:embed` |

## 共通前提・着手条件

**共通契約**（Phase 2-4 で確定、Phase 4 完了時に固定）:

| 契約 | 確定場所 | 確定フェーズ |
|------|----------|-------------|
| API エラーレスポンス形式 | `internal/handler/error.go` | Phase 2 |
| 認証コンテキスト受け渡し方式 | `internal/middleware/auth.go`, `internal/middleware/tenant.go` | Phase 2 |
| RBAC チェックの入口と責務分担 | `internal/middleware/rbac.go` + authz.md | Phase 2 |
| Repository 層での tenant_id 強制ルール | `internal/domain/repository.go` + sqlc クエリ | Phase 4 |
| OpenAPI 生成物の配置先と手書きコードの責務境界 | `db/sqlc.yaml`, `frontend/package.json` の generate スクリプト | Phase 4 |
| 共通テストヘルパー・フィクスチャの配置先 | `internal/testutil/` | Phase 4 |
| フロントエンド API クライアントの責務 | `frontend/src/api/client.ts` | Phase 3 |

**担当境界**:
- backend-developer: `cmd/`, `internal/`, `db/`, `go.mod`, `go.sum`, `Dockerfile`
- frontend-developer: `frontend/` 配下全て
- platform-builder: `docker-compose.yml`, `.github/workflows/`, hooks 定義, `.env.example`
- 担当外ファイルの直接編集は禁止。他担当の領域に変更が必要な場合は指揮役を経由

**禁止事項**:
- MVP スコープ外の機能実装
- `expense-saas/` 以外へのソースコード作成・編集
- 共通基盤の独断変更（Phase 2-3 以降は指揮役承認が必要）
- テストの実装（Step 7 ではスケルトンとヘルパーのみ。テスト本体は Step 8）

**参照先**:
- 上流成果物: `dev-journal/deliverables/docs/` 配下
- 用語集: `dev-journal/deliverables/docs/01_glossary.md`
- 並列ブランチ運用: `dev-journal/guide/parallel-branch-operation.md`

## Phase 構成

### 各 Phase の進め方

**実行前:**
1. 並列タスクがある場合、共通ルール・判断ポイントの適用箇所・タスクごとの要点を整理する
2. 整理した内容をサブエージェントのプロンプトに含めて委譲する
3. 並列タスクがある場合、担当境界・マージ順・トレーサビリティ更新対象を明記する

**実行後:**
1. **内部レビュー**: Phase 成果物の整合性チェック
   - 下流が困る問題があれば修正 → 再レビュー（PASS まで繰り返す）
2. **ユーザーにコミットを提案**
3. **codex レビュー**（/codex-review）: コミット後に実行
   - 指摘があれば対応 → 再レビュー（LGTM まで繰り返す）
4. 次の Phase に進む

---

### Phase 1（逐次）: プロジェクト初期化

| タスク | 成果物 | 入力（主要） | エージェント | 状態 |
|--------|--------|-------------|------------|------|
| 7-B-1: Go バックエンド初期化 | `expense-saas/go.mod`, `expense-saas/cmd/server/main.go`, `expense-saas/internal/` 構造 | architecture.md SS3.1 | platform-builder | 未着手 |
| 7-B-2: React フロントエンド初期化 | `expense-saas/frontend/` 配下（Vite + TypeScript + MUI） | architecture.md SS4.1 | platform-builder | 未着手 |
| 7-B-3: Docker Compose 構成 | `expense-saas/docker-compose.yml`, `expense-saas/Dockerfile` | architecture.md SS2 | platform-builder | 未着手 |
| 7-B-4: DB マイグレーション | `expense-saas/db/migrations/` 配下 | db_schema.md 全セクション | platform-builder | 未着手 |
| Phase 1-R1: 内部レビュー | — | — | impl-unit-reviewer | 未着手 |

成果物配置先: `expense-saas/`

タスクごとの補足:

| タスク | 担当境界 | 依存タスク | ブランチ | 完了条件 | レビュー時の明記項目 |
|--------|----------|-----------|----------|----------|----------------------|
| 7-B-1 | `cmd/`, `internal/`, `go.mod`, `go.sum` | — | `step7/foundation` | `go build ./...` が通る。main.go が HTTP サーバーを起動する最小コード | Go バージョン、主要依存ライブラリ |
| 7-B-2 | `frontend/` 配下全て | — | `step7/foundation` | `npm install && npm run build` が通る。Vite dev server が起動する | Node.js バージョン、主要依存パッケージ |
| 7-B-3 | `docker-compose.yml`, `Dockerfile`, `.env.example` | 7-B-1, 7-B-2 | `step7/foundation` | `docker compose up` で PostgreSQL + Go API + Vite dev server が起動する | ポート設定、環境変数の可変化 |
| 7-B-4 | `db/migrations/` | 7-B-3 | `step7/foundation` | マイグレーション正常終了、RLS ポリシー適用済み、シードデータ投入済み、expense_owner / expense_app ロール作成済み | RLS ポリシー一覧、ロール権限 |

Phase 1 完了条件:
- `docker compose up` で PostgreSQL + Go API + React dev server が起動する
- マイグレーションが正常終了し、全テーブル + RLS + シードデータが投入されている
- `go build ./...` と `npm run build` が通る

並列実行可能性: 7-B-1 と 7-B-2 は並列可能（Go / React は独立）

---

### Phase 2（逐次）: 共通基盤

| タスク | 成果物 | 入力（主要） | エージェント | 状態 |
|--------|--------|-------------|------------|------|
| 7-B-5: 設定管理 | `internal/config/config.go` | architecture.md, security.md | backend-developer | 未着手 |
| 7-B-6: 共通エラーハンドリング | `internal/handler/error.go`, `internal/domain/error.go` | architecture.md SS3.5, openapi.yaml エラー形式, security.md SS8.4 | backend-developer | 未着手 |
| 7-B-7: ミドルウェアチェーン | `internal/middleware/*.go` | architecture.md SS3.2, authz.md SS3-5, security.md SS2-4, monitoring.md SS2 | backend-developer | 未着手 |
| 7-B-8: ヘルスチェック + ルーティング | `internal/handler/health.go`, `cmd/server/routes.go` | openapi.yaml `/health`, authz.md SS5.1 | backend-developer | 未着手 |
| Phase 2-R1: 内部レビュー | — | — | impl-unit-reviewer | 未着手 |

成果物配置先: `expense-saas/internal/`

タスクごとの補足:

| タスク | 担当境界 | 依存タスク | ブランチ | 完了条件 | レビュー時の明記項目 |
|--------|----------|-----------|----------|----------|----------------------|
| 7-B-5 | `internal/config/` | 7-B-1 | `step7/foundation` | 環境変数から DB 接続情報、JWT 鍵パス、CORS 許可オリジン、ログレベル等を読み込む | 環境変数一覧 |
| 7-B-6 | `internal/handler/error.go`, `internal/domain/error.go` | 7-B-1 | `step7/foundation` | ドメインエラー → HTTP エラーレスポンス変換が openapi.yaml のエラー形式に準拠 | エラーコード一覧、マッピング表 |
| 7-B-7 | `internal/middleware/` | 7-B-5, 7-B-6 | `step7/foundation` | ミドルウェアチェーンが architecture.md SS3.2 の順序（CORS → SecurityHeaders → RequestID → Logger → RateLimit → Auth → TenantContext → RBAC）で適用される。個別完了条件: CORS が許可オリジンを反映、SecurityHeaders がセキュリティヘッダーを付与、RequestID が X-Request-ID を生成しログに伝播、Logger が構造化ログ（JSON）に request_id/tenant_id/user_id を含む、RateLimit が閾値超過時に 429 + rate limit ヘッダーを返す、Auth が JWT 検証し不正トークンで 401 を返す、TenantContext が SET LOCAL app.current_tenant を実行、RBAC が authz.md SS5.1 のロール定義に基づき不許可で 403 を返す | ミドルウェア適用順、各要素の設定値と適用対象 |
| 7-B-8 | `internal/handler/health.go`, ルーティング定義 | 7-B-7 | `step7/foundation` | `GET /health` が DB 接続を含むヘルスチェックを返す。認証不要/認証必須のルートグループが authz.md SS5.1 のとおり | ルートグループ一覧 |

Phase 2 完了条件:
- `GET /health` が HTTP 200 で DB 接続ステータスを返す
- 認証不要エンドポイント（/health, /api/auth/*）に認証ミドルウェアが適用されない
- 不正な JWT で認証必須エンドポイントにアクセスすると 401 が返る
- 不許可ロールでアクセスすると 403 が返る
- 構造化ログ（JSON）に request_id, tenant_id, user_id が含まれる
- レスポンスにセキュリティヘッダー（X-Content-Type-Options, X-Frame-Options 等）が付与される
- レスポンスに X-Request-ID が付与される
- CORS が設定した許可オリジンからのリクエストを許可する
- レート制限の閾値超過時に 429 + rate limit ヘッダーが返る
- 全ミドルウェアが architecture.md SS3.2 の順序で適用されている

---

### Phase 3（逐次）: FE-BE 連携基盤

| タスク | 成果物 | 入力（主要） | エージェント | 状態 |
|--------|--------|-------------|------------|------|
| 7-B-9: Vite プロキシ設定 | `frontend/vite.config.ts` | architecture.md SS4.0 | frontend-developer | 未着手 |
| 7-B-10: API クライアント基盤 | `frontend/src/api/client.ts`, `frontend/src/stores/auth.ts` | architecture.md SS4.2, security.md SS2.1, openapi.yaml エラー形式 | frontend-developer | 未着手 |
| 7-B-11: FE-BE 疎通確認 | `frontend/src/App.tsx` | — | frontend-developer | 未着手 |
| Phase 3-R1: 内部レビュー | — | — | impl-unit-reviewer | 未着手 |

成果物配置先: `expense-saas/frontend/`

タスクごとの補足:

| タスク | 担当境界 | 依存タスク | ブランチ | 完了条件 | レビュー時の明記項目 |
|--------|----------|-----------|----------|----------|----------------------|
| 7-B-9 | `frontend/vite.config.ts` | 7-B-2 | `step7/foundation` | Vite dev server が `/api/*` と `/health` を Go API にプロキシ | プロキシ設定内容 |
| 7-B-10 | `frontend/src/api/`, `frontend/src/stores/` | 7-B-9 | `step7/foundation` | fetch ラッパーが Authorization ヘッダー自動付与。401 時リフレッシュ → リトライ | トークン管理フロー |
| 7-B-11 | `frontend/src/App.tsx` | 7-B-9, 7-B-10 | `step7/foundation` | ブラウザで localhost:3000 → `/health` 呼び出し → 結果表示 | — |

Phase 3 完了条件:
- フロントエンド（localhost:3000）から `GET /health` でバックエンドと疎通できる
- API クライアントの共通エラーハンドリング、トークン管理が実装されている

---

### Phase 4（並列あり）: OpenAPI コード生成 + スケルトン

| タスク | 成果物 | 入力（主要） | エージェント | 状態 |
|--------|--------|-------------|------------|------|
| 7-B-12: Go interface + スタブ生成 | `internal/handler/*.go`, `internal/service/*.go`, `internal/domain/model/*.go`, `internal/domain/repository.go` | openapi.yaml, db_schema.md, architecture.md SS3.1 | backend-developer | 未着手 |
| 7-B-13: sqlc クエリ定義 + 生成 | `db/queries/*.sql`, `db/sqlc.yaml`, `internal/repository/postgres/sqlc/` | db_schema.md, openapi.yaml | backend-developer | 未着手 |
| 7-B-14: TypeScript 型定義生成 | `frontend/src/api/types.ts` | openapi.yaml | frontend-developer | 未着手 |
| 7-B-15: テストヘルパー + フィクスチャ基盤 | `internal/testutil/` | test_strategy.md SS5, db_schema.md | backend-developer | 未着手 |
| Phase 4-R1: 内部レビュー | — | — | impl-unit-reviewer | 未着手 |

成果物配置先: `expense-saas/internal/`, `expense-saas/db/`, `expense-saas/frontend/src/api/`

タスクごとの補足:

| タスク | 担当境界 | 依存タスク | ブランチ | 完了条件 | レビュー時の明記項目 |
|--------|----------|-----------|----------|----------|----------------------|
| 7-B-12 | `internal/handler/`, `internal/service/`, `internal/domain/` | 7-B-8 | `step7/go-skeleton` | 全 operationId に対応する Handler メソッド（空実装）、Service interface、ドメインモデル。`go build` 通過 | operationId 対応表 |
| 7-B-13 | `db/queries/`, `db/sqlc.yaml`, `internal/repository/postgres/` | 7-B-4, 7-B-12 | `step7/go-skeleton` | sqlc 生成コードがコンパイル。全テーブルの基本 CRUD。WHERE tenant_id = $1 が全クエリに含まれる | クエリ一覧 |
| 7-B-14 | `frontend/src/api/types.ts` | 7-B-2 | `step7/ts-types` | openapi.yaml の全 schema が TypeScript 型として生成。`npm run build` 通過 | 生成コマンド |
| 7-B-15 | `internal/testutil/` | 7-B-4, 7-B-12 | `step7/go-skeleton` | テスト用 DB セットアップ/クリーンアップ、fixture factory、HTTP テストヘルパー。`go test ./...` 通過 | ヘルパー一覧 |

Phase 4 完了条件:
- スケルトン（interface + スタブ）が全ハンドラ/サービス/リポジトリに存在
- `go build ./...` と `npm run build` が通る
- テストヘルパーが配置され Step 8 が即座に開始可能

並列実行可能性: 7-B-12 と 7-B-14 は並列可能。7-B-13 は 7-B-12 に依存。7-B-15 は 7-B-12 に依存

---

### Phase 5（並列）: CI/CD + hooks + 整理

| タスク | 成果物 | 入力（主要） | エージェント | 状態 |
|--------|--------|-------------|------------|------|
| 7-B-16: GitHub Actions CI | `expense-saas/.github/workflows/ci.yml` | 判断ポイント #2-4 | platform-builder | 未着手 |
| 7-B-17: hooks 設定 | `.claude/settings.json` または hooks 定義 | 判断ポイント #6-7 | platform-builder | 未着手 |
| 7-B-18: 不要ファイル整理 + .env.example | `packages/` 削除, `.env.example`, `.gitignore` 更新 | issue 008 | platform-builder | 未着手 |
| Phase 5-R1: 内部レビュー | — | — | impl-unit-reviewer | 未着手 |

成果物配置先: `expense-saas/.github/workflows/`, `.claude/`

タスクごとの補足:

| タスク | 担当境界 | 依存タスク | ブランチ | 完了条件 | レビュー時の明記項目 |
|--------|----------|-----------|----------|----------|----------------------|
| 7-B-16 | `.github/workflows/` | Phase 1-4 完了 | `step7/ci` | PR 作成時に lint → test → build が自動実行。main マージ後のワークフロー定義が存在する（E2E/スモークのジョブは枠のみ、テスト未実装のため skip 可） | ステージ構成、トリガー条件 |
| 7-B-17 | hooks 定義 | Phase 1-4 完了 | `step7/hooks` | Go: gofmt/goimports が Edit/Write 後に自動実行（警告のみ）。go vet が commit 前に実行（ブロック）。TS: prettier が Edit/Write 後に自動実行（警告のみ） | hooks 一覧、失敗時挙動 |
| 7-B-18 | `packages/` 削除, `.gitignore`, `.env.example` | Phase 1-4 完了 | `step7/cleanup` | packages/ 削除済み。.gitignore に .env, node_modules/ 等。.env.example に必要な環境変数列挙 | 削除対象、環境変数一覧 |

Phase 5 完了条件:
- CI パイプラインが動作する
- hooks により format と go vet が自動実行される
- 不要ディレクトリが整理されている

並列実行可能性: 7-B-16, 7-B-17, 7-B-18 は並列可能

---

### マージ・統合ルール

- **ブランチ戦略**: Phase 1-3（逐次）は `step7/foundation` 共通ブランチ。Phase 4 の並列タスク: 7-B-12/13/15 は `step7/go-skeleton`、7-B-14 は `step7/ts-types`。Phase 5 の並列タスク: 7-B-16 は `step7/ci`、7-B-17 は `step7/hooks`、7-B-18 は `step7/cleanup`
- **コミット粒度**: Phase 単位（Phase 完了 + レビュー通過でコミット提案）
- **main 統合**: Step 7 全 Phase 完了後に PR を作成し main にマージ
- **競合発生時**: 指揮役が一次対応
- **再確認条件**: Phase 4 で `step7/go-skeleton`（7-B-12）マージ後、7-B-13/15 は `go build ./...` を再確認。Phase 5 の各ブランチマージ後、全体で `docker compose up` + `go build ./...` + `npm run build` を再確認

### 共有ファイル運用表

| 対象 | 初回作成担当 | 他担当の追記可否 | 競合時一次対応責任者 | 直列化要否 |
|------|------------|----------------|-------------------|-----------|
| `internal/middleware/` | backend-developer | 不可 | 指揮役 | 直列 |
| `internal/handler/error.go` | backend-developer | 不可 | 指揮役 | 直列 |
| `internal/domain/` | backend-developer | 不可 | 指揮役 | 直列 |
| `frontend/src/api/client.ts` | frontend-developer | 不可 | 指揮役 | 直列 |
| `docker-compose.yml` | platform-builder | 指揮役承認で可 | 指揮役 | 直列 |
| `.github/workflows/` | platform-builder | 不可 | 指揮役 | 直列 |
| `internal/testutil/` | backend-developer | 不可 | 指揮役 | 直列 |

## トレーサビリティ更新対象

- **openapi.yaml ↔ Go Handler/Service interface**: 全 operationId が Go の Handler メソッドに対応。記録場所: コード内コメントに operationId を記載
- **db_schema.md ↔ マイグレーション SQL**: テーブル定義・RLS ポリシーが一致。記録場所: マイグレーションファイル内コメント
- **authz.md ↔ RBAC ミドルウェア**: ルートごとの許可ロール定義が一致。記録場所: ルーティング定義内コメント
- **test_strategy.md ↔ テストヘルパー**: テスト戦略の方針がテストヘルパーに反映。記録場所: testutil 内コメント

## 依存グラフ

```
Phase 1（プロジェクト初期化）
  7-B-1 (Go 初期化) ──┐
  7-B-2 (React 初期化)─┤→ 7-B-3 (Docker Compose) → 7-B-4 (DB マイグレーション)
                       │
                       ↓
Phase 2（共通基盤）[7-B-1 に依存]
  7-B-5 (設定管理) → 7-B-6 (エラーハンドリング) → 7-B-7 (ミドルウェア) → 7-B-8 (ヘルスチェック)
                       │
                       ↓
Phase 3（FE-BE 連携）[7-B-2, 7-B-8 に依存]
  7-B-9 (Vite プロキシ) → 7-B-10 (API クライアント) → 7-B-11 (疎通確認) ← 7-B-8 (ヘルスチェック)
                       │
                       ↓
Phase 4（スケルトン）[7-B-8, 7-B-4 に依存]
  7-B-12 (Go interface + スタブ) ──→ 7-B-13 (sqlc)
  7-B-14 (TS 型定義)                  7-B-15 (テストヘルパー)
                       │
                       ↓
Phase 5（CI/CD + hooks）[Phase 1-4 完了に依存]
  7-B-16 (GitHub Actions) ─┐
  7-B-17 (hooks)           ├→ 最終検証
  7-B-18 (整理)           ─┘
```

## 品質基準

- **下流作業可能性**: Step 8（テストコード実装）が曖昧さなく着手できるか（唯一の判定基準）
- **上流整合性**: architecture.md, openapi.yaml, db_schema.md, authz.md と矛盾がないか
- **共通前提遵守**: 共通契約・担当境界・禁止事項が守られているか
- **トレーサビリティ**: operationId, テーブル定義, 認可ルールの対応関係が追跡可能か
- **MVP スコープ**: `deliverables/docs/02_scope.md` の範囲内か
- **用語集準拠**: `dev-journal/deliverables/docs/01_glossary.md` の用語を使用しているか
- **完了条件**: work-breakdown に定義された完了条件を全て満たしているか
