# Step 7: 実装・運用 — 作業分解

## 概要

動くものを出し、継続的に改善できる形にする。基盤構築 → 機能実装（認証 → 経費CRUD → 添付 → 承認）→ デプロイの順に進める。

## 前提

### 上流成果物（入力）

| 成果物 | パス |
|--------|------|
| 画面一覧 | `deliverables/docs/40_basic_design/screens.md` |
| 画面遷移 | `deliverables/docs/40_basic_design/ui_flow.md` |
| DB スキーマ | `deliverables/docs/50_detail_design/db_schema.md` |
| OpenAPI | `deliverables/docs/50_detail_design/openapi.yaml` |
| 認可設計 | `deliverables/docs/50_detail_design/authz.md` |
| 添付ファイル設計 | `deliverables/docs/50_detail_design/files.md` |
| セキュリティ設計 | `deliverables/docs/50_detail_design/security.md` |
| 監視・ログ設計 | `deliverables/docs/50_detail_design/monitoring.md` |
| テスト戦略 | `deliverables/docs/60_test/test_strategy.md` |
| テストケース | `deliverables/docs/60_test/test_cases.md` |

### 完了条件（Step 全体）

- デプロイ済みで第三者が触れる
- ヘルスチェックエンドポイント（`/health`）が存在する
- CI で自動テスト（単体・統合・E2E）が通る
- 申請 → 承認 → 支払の一連フローが API + UI で通る

---

## 成果物ファイルの扱い

| 成果物 | 出力先 | 初回作成（タスクID） | 追記（タスクID） |
|--------|--------|---------------------|-----------------|
| バックエンドコード | `expense-saas/apps/api/` | 7-B（基盤） | C-2, D-2, E-2, F-2（機能別） |
| フロントエンドコード | `expense-saas/apps/web/` | 7-B（基盤） | C-1, D-1, E-1, F-1（機能別） |
| テストコード | `expense-saas/` 内各所 | C-3（認証） | D-3, E-3, F-3（機能別） |
| DB マイグレーション | `expense-saas/database/migrations/` | 7-B | — |
| Docker Compose | `expense-saas/docker/` | 7-B | — |
| CI 設定 | `expense-saas/.github/workflows/` | 7-B | — |

---

## Wave 構成

### Phase 0: 計画

impl-architect が設計成果物を分析し、タスク実行計画を作成する。ユーザー合意を得てから Wave 1 に進む。

| ID | タスク | 担当エージェント | 出力先 | 依存 |
|----|--------|----------------|--------|------|
| 7-0 | タスク実行計画策定 | impl-architect | `task-plans/step7.md` | Step 6 完了 |

---

### Wave 1: 基盤構築

**実行パターン**: 直列（platform-builder が一括構築）

| ID | タスク | 担当エージェント | 出力先 | 依存 |
|----|--------|----------------|--------|------|
| 7-B | 基盤構築 | platform-builder | `expense-saas/` 各所 | Phase 0 完了 |

**作業範囲**: ディレクトリ構造、共通ミドルウェア（JWT 検証・テナント抽出・エラーハンドリング）、DB 接続・マイグレーション、Docker Compose、CI パイプライン（lint / test / build）

#### Wave 1 完了後レビュー

```
impl-unit-reviewer: 基盤のアーキテクチャ準拠・ビルド成功確認
  → 品質ゲート判定 → ユーザーに PR マージ確認
```

---

### Wave 2: 認証機能（フロント・バック並列 → テスト）

**実行パターン**: バック/フロント並列 → テスト実装の直列

| ID | タスク | 担当エージェント | 出力先 | 依存 |
|----|--------|----------------|--------|------|
| 7-C-1 | 認証フロントエンド | frontend-developer | `apps/web/` §認証 | Wave 1 完了 |
| 7-C-2 | 認証バックエンド | backend-developer | `apps/api/` §認証 | Wave 1 完了 |
| 7-C-3 | 認証テスト実装 | test-implementer | テストコード | 7-C-1, 7-C-2 |

```
┌─ frontend-developer → 認証画面  [7-C-1]
└─ backend-developer  → 認証API   [7-C-2]
    ↓（両方完了後）
  test-implementer → テストコード  [7-C-3]
```

#### Wave 2 完了後レビュー

```
impl-unit-reviewer: 認証機能のフロント/バック横断 + 設計トレーサビリティ
  → 品質ゲート判定 → ユーザーに PR マージ確認
```

---

### Wave 3: 経費レポート CRUD（フロント・バック並列 → テスト）

**実行パターン**: バック/フロント並列 → テスト実装の直列

| ID | タスク | 担当エージェント | 出力先 | 依存 |
|----|--------|----------------|--------|------|
| 7-D-1 | 経費フロントエンド | frontend-developer | `apps/web/` §経費 | Wave 2 完了 |
| 7-D-2 | 経費バックエンド | backend-developer | `apps/api/` §経費 | Wave 2 完了 |
| 7-D-3 | 経費テスト実装 | test-implementer | テストコード | 7-D-1, 7-D-2 |

```
┌─ frontend-developer → 経費画面  [7-D-1]
└─ backend-developer  → 経費API   [7-D-2]
    ↓（両方完了後）
  test-implementer → テストコード  [7-D-3]
```

#### Wave 3 完了後レビュー

```
impl-unit-reviewer: 経費CRUD機能のフロント/バック横断 + 設計トレーサビリティ
  → 品質ゲート判定 → ユーザーに PR マージ確認
```

---

### Wave 4: 添付ファイル + 承認フロー（機能間並列）

**実行パターン**: 機能間並列、各機能内はバック/フロント並列 → テスト

| ID | タスク | 担当エージェント | 出力先 | 依存 |
|----|--------|----------------|--------|------|
| 7-E-1 | 添付フロントエンド | frontend-developer | `apps/web/` §添付 | Wave 3 完了 |
| 7-E-2 | 添付バックエンド | backend-developer | `apps/api/` §添付 | Wave 3 完了 |
| 7-E-3 | 添付テスト実装 | test-implementer | テストコード | 7-E-1, 7-E-2 |
| 7-F-1 | 承認フロントエンド | frontend-developer | `apps/web/` §承認 | Wave 3 完了 |
| 7-F-2 | 承認バックエンド | backend-developer | `apps/api/` §承認 | Wave 3 完了 |
| 7-F-3 | 承認テスト実装 | test-implementer | テストコード | 7-F-1, 7-F-2 |

```
┌─ [添付] frontend-developer + backend-developer（並列）[7-E-1, 7-E-2]
│          ↓（両方完了後）
│         test-implementer  [7-E-3]
│
└─ [承認] frontend-developer + backend-developer（並列）[7-F-1, 7-F-2]
           ↓（両方完了後）
          test-implementer  [7-F-3]
```

#### Wave 4 完了後レビュー

```
impl-unit-reviewer × 2（並列）:
  - 添付ファイル機能のフロント/バック横断 + 設計トレーサビリティ
  - 承認フロー機能のフロント/バック横断 + 設計トレーサビリティ
  → 品質ゲート判定 → ユーザーに PR マージ確認
```

---

### Wave 5: 横断レビュー

**実行パターン**: 並列（test-reviewer + impl-cross-reviewer）

| ID | タスク | 担当エージェント | 出力先 | 依存 |
|----|--------|----------------|--------|------|
| 7-R-1 | テスト品質・網羅性レビュー | test-reviewer | レビューレポート | Wave 4 完了 |
| 7-R-2 | 全機能横断レビュー | impl-cross-reviewer | レビューレポート | Wave 4 完了 |

```
┌─ test-reviewer ────────→ テスト網羅性・品質チェック       [7-R-1]
└─ impl-cross-reviewer ──→ 全機能横断 + セキュリティ + 設計整合性 [7-R-2]
  → 品質ゲート判定 → ユーザーに PR マージ確認
  → /codex-review（Step 成果物完成後の外部レビュー）
```

---

## タスク詳細

### 7-0: タスク実行計画策定

- **担当エージェント**: impl-architect
- **入力**: 本ファイル（step7-implementation.md）、Step 4+5 / Step 6 成果物
- **出力**: `dev-journal/progress-management/task-plans/step7.md`
- **作業内容**:
  - 設計成果物の状態を確認し、実装タスクを具体化
  - 依存関係の整理と実装順序の確定
  - 受け入れ基準の定義
- **完了条件**:
  - タスク実行計画ファイルが作成されている
  - ユーザーが計画に合意している

### 7-B: 基盤構築

- **担当エージェント**: platform-builder
- **入力**: architecture.md, db_schema.md, security.md, monitoring.md
- **出力**: `expense-saas/` 配下のディレクトリ構造・共通基盤
- **作業内容**:
  - Go バックエンド初期化（go.mod, cmd/api/main.go, internal/ 構造）
  - React フロントエンド初期化（Vite + TypeScript）
  - DB マイグレーション（db_schema.md に基づく全テーブル作成 + RLS 設定）
  - 共通ミドルウェア（JWT 検証、テナント抽出、RBAC チェック、エラーハンドリング）
  - Docker Compose（PostgreSQL, API, Web）
  - CI パイプライン（lint / test / build）
  - 環境変数・設定管理
- **完了条件**:
  - `docker compose up` で全サービスが起動する
  - マイグレーションが正常終了し RLS が設定されている
  - `go build ./...` と `npm run build` が通る
  - CI パイプラインが動作する

### 7-C-1: 認証フロントエンド

- **担当エージェント**: frontend-developer
- **入力**: screens.md §認証, openapi.yaml §認証
- **出力**: `apps/web/src/features/auth/`
- **作業内容**:
  - ログイン画面、サインアップ画面、パスワードリセット画面
  - JWT 管理（アクセストークン・リフレッシュトークン）
  - 認証状態に基づくルーティングガード
- **完了条件**:
  - 認証画面が screens.md の仕様通りに実装されている
  - トークンリフレッシュが自動実行される

### 7-C-2: 認証バックエンド

- **担当エージェント**: backend-developer
- **入力**: openapi.yaml §認証, db_schema.md, authz.md, security.md
- **出力**: `apps/api/internal/` §認証
- **作業内容**:
  - POST /auth/signup, POST /auth/login, POST /auth/refresh, POST /auth/logout, GET /auth/me
  - パスワードハッシュ（Argon2id）、JWT 発行（RS256）
  - リフレッシュトークンの DB 保存（token_hash）
- **完了条件**:
  - OpenAPI 定義通りの API が実装されている
  - signup → login → refresh → logout の一連フローが通る

### 7-C-3: 認証テスト実装

- **担当エージェント**: test-implementer
- **入力**: test_cases.md §認証, 7-C-1 / 7-C-2 の実装コード
- **出力**: テストコード
- **作業内容**:
  - 単体テスト: Argon2id ハッシュ/検証、JWT 生成/検証
  - 統合テスト: 各認証エンドポイント、無効トークンで 401
  - test_cases.md の認証関連ケースを網羅
- **完了条件**:
  - test_cases.md の認証関連テストケースが全て実装されている

### 7-D-1: 経費フロントエンド

- **担当エージェント**: frontend-developer
- **入力**: screens.md §経費, openapi.yaml §経費
- **出力**: `apps/web/src/features/expenses/`
- **作業内容**:
  - レポート一覧、作成/編集、明細追加/編集、詳細画面
  - ロール別の表示差異
  - ページネーション・フィルタリング・ソート
- **完了条件**:
  - CRUD 全操作の画面が screens.md の仕様通りに実装されている

### 7-D-2: 経費バックエンド

- **担当エージェント**: backend-developer
- **入力**: openapi.yaml §経費, db_schema.md, authz.md, state_machine.md
- **出力**: `apps/api/internal/` §経費
- **作業内容**:
  - GET/POST/PUT/DELETE /reports, GET/POST/PUT/DELETE /reports/{id}/items
  - 状態遷移ロジック（state_machine.md 準拠）
  - テナント分離（全クエリに tenant_id 条件）
- **完了条件**:
  - OpenAPI 定義通りの API が実装されている
  - 不正な状態遷移で 422 を返す

### 7-D-3: 経費テスト実装

- **担当エージェント**: test-implementer
- **入力**: test_cases.md §経費, 7-D-1 / 7-D-2 の実装コード
- **出力**: テストコード
- **作業内容**:
  - 状態遷移テスト（全パターン）、テナント分離テスト、RBAC テスト
  - test_cases.md の経費関連ケースを網羅
- **完了条件**:
  - test_cases.md の経費関連テストケースが全て実装されている

### 7-E-1: 添付フロントエンド

- **担当エージェント**: frontend-developer
- **入力**: screens.md §添付, openapi.yaml §添付, files.md
- **出力**: `apps/web/src/features/expenses/` §添付
- **作業内容**:
  - 明細内の添付エリア（アップロード・プレビュー・削除）
  - 署名付き URL を使ったアップロード/ダウンロード
- **完了条件**:
  - 添付ファイルの画面操作が screens.md の仕様通りに実装されている

### 7-E-2: 添付バックエンド

- **担当エージェント**: backend-developer
- **入力**: openapi.yaml §添付, files.md, db_schema.md, authz.md
- **出力**: `apps/api/internal/` §添付
- **作業内容**:
  - 署名付き URL 発行 API（アップロード用・ダウンロード用）
  - S3 クライアント設定、MIME バリデーション、サイズ制限
  - 発行前認可チェック
- **完了条件**:
  - files.md の設計通りに実装されている

### 7-E-3: 添付テスト実装

- **担当エージェント**: test-implementer
- **入力**: test_cases.md §添付, 7-E-1 / 7-E-2 の実装コード
- **出力**: テストコード
- **作業内容**:
  - MIME/サイズ制限テスト、認可テスト
  - test_cases.md の添付関連ケースを網羅
- **完了条件**:
  - test_cases.md の添付関連テストケースが全て実装されている

### 7-F-1: 承認フロントエンド

- **担当エージェント**: frontend-developer
- **入力**: screens.md §承認, openapi.yaml §承認
- **出力**: `apps/web/src/features/approvals/`
- **作業内容**:
  - 承認キュー画面、承認/却下アクション
  - 却下理由の入力、差戻し後の再編集フロー
- **完了条件**:
  - 承認フロー画面が screens.md の仕様通りに実装されている

### 7-F-2: 承認バックエンド

- **担当エージェント**: backend-developer
- **入力**: openapi.yaml §承認, db_schema.md, authz.md, state_machine.md, workflow.md
- **出力**: `apps/api/internal/` §承認
- **作業内容**:
  - POST /reports/{id}/submit, approve, reject, mark-paid
  - 状態遷移の強制、approvals テーブルへの記録
- **完了条件**:
  - 承認フローの全アクションが実装されている
  - 権限外ロールで 403、不正遷移で 422 を返す

### 7-F-3: 承認テスト実装

- **担当エージェント**: test-implementer
- **入力**: test_cases.md §承認, 7-F-1 / 7-F-2 の実装コード
- **出力**: テストコード
- **作業内容**:
  - 承認フロー全パターンテスト、RBAC テスト
  - test_cases.md の承認関連ケースを網羅
- **完了条件**:
  - test_cases.md の承認関連テストケースが全て実装されている
