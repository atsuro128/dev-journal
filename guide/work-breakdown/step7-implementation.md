# Step 7: 実装・運用 — 作業分解

## 概要

動くものを出し、継続的に改善できる形にする。基盤構築 → 機能実装（認証 → 経費CRUD → 添付 → 承認）→ 横断検証の順に進める。

## 前提

### 上流成果物（入力）

| 成果物 | パス |
|--------|------|
| 画面一覧 | `deliverables/docs/40_basic_design/screens.md` |
| 画面遷移図 | `deliverables/docs/40_basic_design/ui_flow.md` |
| 画面仕様（機能別） | `deliverables/docs/50_detail_design/screens/*.md` |
| DB スキーマ | `deliverables/docs/50_detail_design/db_schema.md` |
| OpenAPI | `deliverables/docs/50_detail_design/openapi.yaml` |
| 認可設計 | `deliverables/docs/50_detail_design/authz.md` |
| 添付ファイル設計 | `deliverables/docs/50_detail_design/files.md` |
| セキュリティ設計 | `deliverables/docs/50_detail_design/security.md` |
| 監視・ログ設計 | `deliverables/docs/50_detail_design/monitoring.md` |
| テスト戦略 | `deliverables/docs/60_test/test_strategy.md` |
| テストケース | `deliverables/docs/60_test/test_cases.md` |

### 作業計画

Step 着手時にまず作業計画を立案し、以下に保存する:

- テンプレート: `guide/templates/task-plan.md`
- 配置先: `progress-management/task-plans/step7-implementation.md`
- 計画が確定してから成果物作成に入ること

### 完了条件（Step 全体）

- デプロイ済みで第三者が触れる
- ヘルスチェックエンドポイント（`/health`）が存在する
- CI で自動テスト（単体・統合・E2E）が通る
- 申請 → 承認 → 支払の一連フローが API + UI で通る

---

## レビュー観点

成果物内の「品質チェック」は作成者のセルフチェック。以下は reviewer が検証する観点。

> Step 7 は未着手のため、以下は現時点で定義できる観点。着手時に実装スコープを踏まえて精緻化すること。具体的なコードレビュー基準（コーディング規約、エラーハンドリング方針等）は着手時に定義する。

### 全タスク共通

#### 1. 設計準拠性
- 実装が上流設計（OpenAPI, db_schema, authz, security, files, screens/*.md）の仕様どおりであるか
- テナント分離の二重保証（アプリ層 WHERE + RLS）が全クエリで実装されているか
- レイヤー構成（Handler → Service → Domain → Repository）が `architecture.md` に従っているか

#### 2. セキュリティ
- SQL インジェクション対策（プレースホルダの使用）
- JWT 検証がミドルウェアで全保護エンドポイントに適用されているか
- パスワードが Argon2id でハッシュされているか
- セキュリティヘッダーが `security.md` どおりに設定されているか
- RLS のコンテキスト設定（`SET LOCAL app.current_tenant`）が同一コネクション・同一トランザクションで実行されているか

#### 3. テスト品質
- `test_cases.md` の全テストケースがコードとして実装されているか
- テスト結果が CI で自動検証されているか

### 基盤構築固有

#### 4. 基盤
- `docker compose up` で全サービスが起動するか
- マイグレーションが正常終了し RLS が設定されているか
- `go build ./...` と `npm run build` が通るか
- CI パイプラインが動作するか
- ヘルスチェックエンドポイント（`/health`）が存在するか

### 機能実装固有

#### 5. 機能
- OpenAPI 定義どおりの API が実装されているか
- 画面仕様（`screens/*.md`）どおりの UI が実装されているか
- 不正な状態遷移で 422 を返すか
- 権限外ロールで 403 を返すか
- テナント境界越えで 404 を返すか

### 横断レビュー固有

#### 6. 横断品質
- 機能間で共通基盤（ミドルウェア・エラーハンドリング・API クライアント）が正しく使用されているか
- コード重複が適切に排除されているか
- 全機能横断で申請→承認→支払完了のフローが通るか
- テスト網羅性（test_cases.md との対応）が確認されているか

---

## 成果物ファイルの扱い

| 成果物 | 出力先 | 作成タスク | 追記タスク |
|--------|--------|-----------|-----------|
| バックエンドコード | `expense-saas/apps/api/` | 7-B（基盤） | 7-C-2, D-2, E-2, F-2（機能別） |
| フロントエンドコード | `expense-saas/apps/web/` | 7-B（基盤） | 7-C-1, D-1, E-1, F-1（機能別） |
| テストコード | `expense-saas/` 内各所 | 7-C-3（認証） | D-3, E-3, F-3（機能別） |
| DB マイグレーション | `expense-saas/database/migrations/` | 7-B | — |
| Docker Compose | `expense-saas/docker/` | 7-B | — |
| CI 設定 | `expense-saas/.github/workflows/` | 7-B | — |

---

## タスク一覧

| ID | タスク | 種別 | 依存 | 状態 |
|----|--------|------|------|------|
| 7-B | 基盤構築 | 基盤 | Step 6 完了 | 未着手 |
| 7-C-1 | 認証フロントエンド | 機能 | 7-B | 未着手 |
| 7-C-2 | 認証バックエンド | 機能 | 7-B | 未着手 |
| 7-C-3 | 認証テスト実装 | 機能 | 7-C-1, 7-C-2 | 未着手 |
| 7-D-1 | 経費フロントエンド | 機能 | 7-C-1, 7-C-2 | 未着手 |
| 7-D-2 | 経費バックエンド | 機能 | 7-C-1, 7-C-2 | 未着手 |
| 7-D-3 | 経費テスト実装 | 機能 | 7-D-1, 7-D-2 | 未着手 |
| 7-E-1 | 添付フロントエンド | 機能 | 7-D-1, 7-D-2 | 未着手 |
| 7-E-2 | 添付バックエンド | 機能 | 7-D-1, 7-D-2 | 未着手 |
| 7-E-3 | 添付テスト実装 | 機能 | 7-E-1, 7-E-2 | 未着手 |
| 7-F-1 | 承認フロントエンド | 機能 | 7-D-1, 7-D-2 | 未着手 |
| 7-F-2 | 承認バックエンド | 機能 | 7-D-1, 7-D-2 | 未着手 |
| 7-F-3 | 承認テスト実装 | 機能 | 7-F-1, 7-F-2 | 未着手 |
| 7-R | 横断レビュー | 統合 | 7-C-3, 7-D-3, 7-E-3, 7-F-3 | 未着手 |

### 依存グラフ

```
Step 6 完了
  └→ 7-B (基盤) ─┬→ 7-C-1 (認証FE) ─┐
                  └→ 7-C-2 (認証BE) ──┤
                                      └→ 7-C-3 (認証テスト) ──┐
                  ┌→ 7-D-1 (経費FE) ─┐                       │
  7-C-1,C-2 完了 ─┤                  └→ 7-D-3 (経費テスト)    │
                  └→ 7-D-2 (経費BE) ─┘                       │
                  ┌→ 7-E-1 (添付FE) ─┐                       │
  7-D-1,D-2 完了 ─┤                  └→ 7-E-3 (添付テスト) ──┤
                  └→ 7-E-2 (添付BE) ─┘                       │
                  ┌→ 7-F-1 (承認FE) ─┐                       │
  7-D-1,D-2 完了 ─┤                  └→ 7-F-3 (承認テスト) ──┤
                  └→ 7-F-2 (承認BE) ─┘                       │
                                                              ↓
                                                      7-R (横断レビュー)
```

---

## タスク詳細

### 7-B: 基盤構築

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

- **入力**: screens/auth-signup.md, auth-login.md, auth-password-reset-request.md, auth-password-reset.md, openapi.yaml §認証
- **出力**: `apps/web/src/features/auth/`
- **作業内容**:
  - ログイン画面、サインアップ画面、パスワードリセット画面
  - JWT 管理（アクセストークン・リフレッシュトークン）
  - 認証状態に基づくルーティングガード
- **完了条件**:
  - 認証画面が screens/auth-signup.md, auth-login.md, auth-password-reset-request.md, auth-password-reset.md の仕様通りに実装されている
  - トークンリフレッシュが自動実行される

### 7-C-2: 認証バックエンド

- **入力**: openapi.yaml §認証, db_schema.md, authz.md, security.md
- **出力**: `apps/api/internal/` §認証
- **作業内容**:
  - POST /auth/signup, login, refresh, logout, GET /auth/me
  - パスワードハッシュ（Argon2id）、JWT 発行（RS256）
  - リフレッシュトークンの DB 保存（token_hash）
- **完了条件**:
  - OpenAPI 定義通りの API が実装されている
  - signup → login → refresh → logout の一連フローが通る

### 7-C-3: 認証テスト実装

- **入力**: test_cases.md §認証, 7-C-1/7-C-2 の実装コード
- **出力**: テストコード
- **作業内容**:
  - 単体テスト: Argon2id ハッシュ/検証、JWT 生成/検証
  - 統合テスト: 各認証エンドポイント、無効トークンで 401
- **完了条件**:
  - test_cases.md の認証関連テストケースが全て実装されている

### 7-D-1: 経費フロントエンド

- **入力**: screens/expenses.md, openapi.yaml §経費
- **出力**: `apps/web/src/features/expenses/`
- **作業内容**:
  - レポート一覧、作成/編集、明細追加/編集、詳細画面
  - ダッシュボード画面
  - ロール別の表示差異、ページネーション・フィルタリング・ソート
- **完了条件**:
  - CRUD 全操作の画面が screens/expenses.md の仕様通りに実装されている

### 7-D-2: 経費バックエンド

- **入力**: openapi.yaml §経費, db_schema.md, authz.md, state_machine.md
- **出力**: `apps/api/internal/` §経費
- **作業内容**:
  - GET/POST/PUT/DELETE /reports, GET/POST/PUT/DELETE /reports/:id/items
  - GET /dashboard, GET /tenant
  - 状態遷移ロジック（state_machine.md 準拠）
  - テナント分離（全クエリに tenant_id 条件）
- **完了条件**:
  - OpenAPI 定義通りの API が実装されている
  - 不正な状態遷移で 422 を返す

### 7-D-3: 経費テスト実装

- **入力**: test_cases.md §経費, 7-D-1/7-D-2 の実装コード
- **出力**: テストコード
- **作業内容**:
  - 状態遷移テスト（全パターン）、テナント分離テスト、RBAC テスト
- **完了条件**:
  - test_cases.md の経費関連テストケースが全て実装されている

### 7-E-1: 添付フロントエンド

- **入力**: screens/attachments.md, openapi.yaml §添付, files.md
- **出力**: `apps/web/src/features/expenses/` §添付
- **作業内容**:
  - 明細内の添付エリア（アップロード・プレビュー・削除）
  - 署名付き URL を使ったアップロード/ダウンロード
- **完了条件**:
  - 添付ファイルの画面操作が screens/attachments.md の仕様通りに実装されている

### 7-E-2: 添付バックエンド

- **入力**: openapi.yaml §添付, files.md, db_schema.md, authz.md
- **出力**: `apps/api/internal/` §添付
- **作業内容**:
  - 署名付き URL 発行 API（アップロード用・ダウンロード用）
  - S3 クライアント設定、MIME バリデーション、サイズ制限
  - 発行前認可チェック
- **完了条件**:
  - files.md の設計通りに実装されている

### 7-E-3: 添付テスト実装

- **入力**: test_cases.md §添付, 7-E-1/7-E-2 の実装コード
- **出力**: テストコード
- **作業内容**:
  - MIME/サイズ制限テスト、認可テスト
- **完了条件**:
  - test_cases.md の添付関連テストケースが全て実装されている

### 7-F-1: 承認フロントエンド

- **入力**: screens/approvals.md, openapi.yaml §承認
- **出力**: `apps/web/src/features/approvals/`
- **作業内容**:
  - 承認キュー画面、承認/却下アクション
  - 却下理由の入力、差戻し後の再編集フロー
- **完了条件**:
  - 承認フロー画面が screens/approvals.md の仕様通りに実装されている

### 7-F-2: 承認バックエンド

- **入力**: openapi.yaml §承認, db_schema.md, authz.md, state_machine.md, workflow.md
- **出力**: `apps/api/internal/` §承認
- **作業内容**:
  - POST /reports/:id/submit, approve, reject, mark-paid
  - 状態遷移の強制
- **完了条件**:
  - 承認フローの全アクションが実装されている
  - 権限外ロールで 403、不正遷移で 422 を返す

### 7-F-3: 承認テスト実装

- **入力**: test_cases.md §承認, 7-F-1/7-F-2 の実装コード
- **出力**: テストコード
- **作業内容**:
  - 承認フロー全パターンテスト、RBAC テスト
- **完了条件**:
  - test_cases.md の承認関連テストケースが全て実装されている

### 7-R: 横断レビュー

- **入力**: 全機能の実装コード・テストコード、設計成果物全体
- **出力**: レビューレポート
- **作業内容**:
  - テスト品質・網羅性チェック（test_cases.md との対応）
  - 全機能横断レビュー（セキュリティ・設計整合性・共通基盤の正しい使用）
- **完了条件**:
  - テスト網羅性が確認されている
  - 機能間の整合性が確認されている
