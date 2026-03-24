# Step 7: 基盤構築 — 作業分解

## 概要

各機能タスクが並列開発できる土台を整える。Go バックエンド・React フロントエンドの初期化、DB マイグレーション、共通ミドルウェア、CI/CD パイプラインを構築する。

## 前提

### 上流成果物（入力）

| 成果物 | パス |
|--------|------|
| アーキテクチャ設計 | `deliverables/docs/30_architecture/architecture.md` |
| API 契約 | `deliverables/docs/50_detail_design/openapi.yaml` |
| DB スキーマ | `deliverables/docs/50_detail_design/db_schema.md` |
| 認可設計 | `deliverables/docs/50_detail_design/authz.md` |
| セキュリティ設計 | `deliverables/docs/50_detail_design/security.md` |
| 監視・ログ設計 | `deliverables/docs/50_detail_design/monitoring.md` |

### 作業計画

Step 着手時にまず作業計画を立案し、以下に保存する:

- テンプレート: `guide/templates/task-plan.md`
- 配置先: `progress-management/task-plans/step7-foundation.md`
- 計画が確定してから成果物作成に入ること
- 並列実装のブランチ運用は `guide/parallel-branch-operation.md` を参照し、task plan に反映すること

### 計画レビューゲート

作業計画は**成果物作成前のレビュー対象**とする。少なくとも以下を確認し、通過するまで成果物作成に入ってはならない。

- 判断ポイントに未確定のまま残してよい項目と、着手前に確定が必要な項目が切り分けられているか
- 共通前提・着手条件・担当境界・禁止事項・参照先が明記されているか
- レビュー単位、マージ順、共有ファイル調整、トレーサビリティ更新対象が明記されているか、`guide/parallel-branch-operation.md` と整合しているか
- この Step の完了条件と、下流 Step の着手条件が task plan に落ちているか
- 作業計画レビューで出た blocking 指摘への対応方針が整理されているか


### 完了条件（Step 全体）

- `docker compose up` で全サービスが起動する
- マイグレーションが正常終了し RLS が設定されている
- `go build ./...` と `npm run build` が通る
- フロントエンドから `GET /health` を呼び出してバックエンドと疎通できる
- CI パイプラインが動作する（PR 時に lint / test / build が自動実行される）
- hooks により、Claude Code 実行中の format と軽量 lint が自動実行される
- テストコードがコンパイル可能なスケルトン（interface + スタブ）が存在する
- テストコード実装（Step 8）が即座に開始できる状態になっている

### 並列実装開始条件

Step 8 / Step 9 の担当を並列着手させる前に、以下を Step 7 で固定すること。

- API エラーレスポンス形式（HTTP ステータス、アプリケーションコード、メッセージ、フィールドエラーの構造）
- 認証コンテキストとテナントコンテキストの受け渡し方式（ミドルウェア → Handler → Service）
- RBAC チェックの入口と責務分担
- Repository 層での `tenant_id` 強制ルール
- OpenAPI 生成物の配置先と、手書きコードとの責務境界
- 共通テストヘルパー・フィクスチャの配置先
- フロントエンド API クライアントの責務（認証トークン管理、共通エラーハンドリング、リトライ有無）

上記が未確定の状態で、機能別のテスト実装・機能実装に着手してはならない。

---

## レビュー観点

成果物内の「品質チェック」は作成者のセルフチェック。以下は reviewer が検証する観点。

### 基盤構築固有

#### 1. プロジェクト初期化（Phase 1: 7-B-1〜4）
- `docker compose up` で全サービス（PostgreSQL + Go API + React dev server）が起動するか
- マイグレーションが正常終了し RLS が設定されているか
- `go build ./...` と `npm run build` が通るか
- カテゴリシードデータが投入されているか
- expense_owner / expense_app ロールが作成されているか

#### 2. 共通基盤（Phase 2: 7-B-5〜8）
- 環境変数から DB 接続情報、JWT 鍵パス、CORS 許可オリジン、ログレベル等を読み込めるか
- ミドルウェアチェーンが architecture.md SS3.2 の順序（CORS → SecurityHeaders → RequestID → Logger → RateLimit → Auth → TenantContext → RBAC）で適用されているか
- 各ミドルウェアの検証:
  - CORS: 許可オリジンからのリクエストが通るか
  - SecurityHeaders: セキュリティヘッダー（X-Content-Type-Options, X-Frame-Options 等）が付与されるか
  - RequestID: X-Request-ID が生成され、レスポンスとログに含まれるか
  - Logger: 構造化ログ（JSON）に必須フィールド（request_id, tenant_id, user_id, method, path, status, duration_ms）が出力されるか（monitoring.md SS2 参照）
  - RateLimit: 閾値超過時に 429 + rate limit ヘッダーが返るか
  - Auth: 不正な JWT で 401 が返るか。認証不要ルート（/health, /api/auth/*）がスキップされるか
  - TenantContext: pool.Acquire による接続固定、BEGIN + SET LOCAL app.current_tenant、終了時の COMMIT/ROLLBACK が実装されているか（architecture.md SS3.2 参照）
  - RBAC: 不許可ロールで 403 が返るか（authz.md SS5.1 参照）
- ヘルスチェックエンドポイント（`GET /health`）が DB 接続ステータスを含むレスポンスを返すか
- 認証不要/認証必須のルートグループが authz.md SS5.1 のとおり定義されているか

#### 3. FE-BE 連携（Phase 3: 7-B-9〜11）
- Vite dev server が `/api/*` と `/health` を Go API にプロキシするか
- API クライアントが Authorization ヘッダーを自動付与するか
- 401 受信時にトークンリフレッシュ → リトライが動作するか
- フロントエンド（localhost:3000）から `GET /health` を呼び出してバックエンドと疎通できるか

#### 4. スケルトン・コード生成（Phase 4: 7-B-12〜15）
- 全 operationId に対応する Handler メソッド（空実装）が存在するか（openapi.yaml 参照）
- Service interface と空実装 struct が存在するか
- ドメインモデル（Entity, ValueObject, Enum）が定義されているか
- sqlc で生成されたコードがコンパイルされるか
- 全テーブルの基本 CRUD クエリに `WHERE tenant_id = $1` が含まれるか
- openapi.yaml の全 schema が TypeScript 型として生成されているか
- テスト用 DB セットアップ/クリーンアップ関数が存在するか
- テナント/ユーザー/レポートの fixture factory が存在するか
- HTTP テストヘルパー（認証付きリクエスト発行）が存在するか

#### 5. CI/CD・hooks・整理（Phase 5: 7-B-16〜18）
- CI パイプラインが動作するか（PR 時に lint / test / build が自動実行される）
- main マージ後のワークフロー定義が存在するか（E2E/スモークは枠のみで可）
- hooks による format / lint が動作するか
- `packages/` ディレクトリが削除されているか
- `.gitignore` に `.env`, `node_modules/`, バイナリ等が含まれているか
- `.env.example` に必要な環境変数が列挙されているか

#### 6. 共通契約の固定（全 Phase 横断）
- Step 8 / Step 9 の担当が参照する以下の共通契約が固定されているか:
  - API エラーレスポンス形式（HTTP ステータス、アプリケーションコード、メッセージ、フィールドエラーの構造）
  - 認証コンテキストとテナントコンテキストの受け渡し方式（ミドルウェア → Handler → Service）
  - RBAC チェックの入口と責務分担
  - Repository 層での `tenant_id` 強制ルール
  - OpenAPI 生成物の配置先と、手書きコードとの責務境界
  - 共通テストヘルパー・フィクスチャの配置先
  - フロントエンド API クライアントの責務（認証トークン管理、共通エラーハンドリング、リトライ有無）

---

## 成果物ファイルの扱い

| 成果物 | 出力先 | 作成タスク |
|--------|--------|-----------|
| バックエンドコード（基盤） | `expense-saas/cmd/server/`, `expense-saas/internal/` | 7-B |
| フロントエンドコード（基盤） | `expense-saas/frontend/` | 7-B |
| DB マイグレーション | `expense-saas/db/migrations/` | 7-B |
| Docker Compose | `expense-saas/docker-compose.yml` | 7-B |
| CI 設定 | `expense-saas/.github/workflows/` | 7-B |

---

## タスク一覧

| ID | タスク | テストケース | 依存 | 状態 |
|----|--------|-------------|------|------|
| 7-B | 基盤構築 | — | Step 6 完了 | 未着手 |

### 依存グラフ

```
Step 6 完了
  └→ 7-B (基盤)
       └→ Step 8 (テストコード実装)
```

---

## タスク詳細

### 7-B: 基盤構築

- **入力**: architecture.md, openapi.yaml, db_schema.md, authz.md, security.md, monitoring.md
- **出力**: `expense-saas/` 配下のディレクトリ構造・共通基盤
- **ゴール**: 7-B 完了後、各機能タスク（8-A〜8-G）が依存関係に従って並列開発できる状態にする
- **作業内容**:
  - Go バックエンド初期化（go.mod, cmd/server/main.go, internal/ 構造）
  - React フロントエンド初期化（Vite + TypeScript）
  - DB マイグレーション（db_schema.md に基づく全テーブル作成 + RLS 設定）
  - 共通ミドルウェア（JWT 検証、テナント抽出、RBAC チェック、エラーハンドリング）
  - フロントエンド API クライアント基盤（fetch ラッパー、共通エラーハンドリング、認証トークン管理）
  - Vite dev server → Go API へのプロキシ設定
  - ヘルスチェックエンドポイント（`GET /health`）
  - Docker Compose（PostgreSQL, API, Web）
  - CI/CD パイプライン設計・実装（下記の判断ポイントを計画時に決定する）
  - hooks 設計・実装（Edit/Write 後の format、危険変更前の軽量 lint）
  - 環境変数・設定管理
  - 不要ディレクトリの整理（`packages/` 削除 — issue 008）
  - openapi.yaml からインターフェース/型定義を生成（Handler/Service/Repository の Go interface、TypeScript の API レスポンス/リクエスト型）
  - テストコードがコンパイル可能なスタブ（空の関数・メソッド）を配置
- **CI/CD パイプライン設計（計画時の判断ポイント）**:
  - ステージ構成: lint → test → build の各ステージで何を実行するか
  - トリガー条件: PR 作成時・main マージ時・手動実行のどれで何を走らせるか
  - ブランチ保護ルール: main への直接プッシュ禁止、CI 通過必須とするか
  - マージ前チェック: テスト通過・ビルド成功・lint エラーゼロを必須とするか
  - デプロイ: main マージ時に自動デプロイするか、手動承認を挟むか
  - 参照: 並列開発運用ルール（配置場所は固定しない）
- **hooks 設計（計画時の判断ポイント）**:
  - Edit / Write 後にどの format を自動実行するか
  - 危険変更前にどの軽量 lint を走らせるか
  - hooks 失敗時の挙動（警告のみ / ブロック）
  - GitHub Actions と重複するチェックをどこまで許容するか
- **Dev Container 複数インスタンス対応（計画時の判断ポイント）**:
  - ポート競合回避: 複数インスタンスが同時起動した場合の API / Web / DB ポートの分離方式
  - DB 分離: インスタンスごとに独立した PostgreSQL を使うか、同一 DB で別スキーマとするか
  - 環境変数: インスタンス固有の設定（ポート番号、DB 名等）をどう管理するか
- **完了条件**:
  - `docker compose up` で全サービスが起動する
  - マイグレーションが正常終了し RLS が設定されている
  - `go build ./...` と `npm run build` が通る
  - フロントエンドから `GET /health` を呼び出してバックエンドと疎通できる
  - CI パイプラインが動作する（PR 時に lint / test / build が自動実行される）
  - hooks により、Claude Code 実行中の format と軽量 lint が自動実行される
  - テストコードがコンパイル可能なスケルトン（interface + スタブ）が存在する
  - テストコード実装（Step 8）が即座に開始できる状態になっている
  - 並列実装開始条件で列挙した共通契約が文書またはコード上で固定され、各担当が参照先を一意に判断できる
