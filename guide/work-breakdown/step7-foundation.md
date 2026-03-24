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
| 並列開発運用ルール | `dev-journal/ai-operations/subagent-workflow.md` |

### 作業計画

Step 着手時にまず作業計画を立案し、以下に保存する:

- テンプレート: `guide/templates/task-plan.md`
- 配置先: `progress-management/task-plans/step7-foundation.md`
- 計画が確定してから成果物作成に入ること

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

#### 1. 基盤
- `docker compose up` で全サービスが起動するか
- マイグレーションが正常終了し RLS が設定されているか
- `go build ./...` と `npm run build` が通るか
- CI パイプラインが動作するか
- hooks による format / 軽量 lint が動作するか
- ヘルスチェックエンドポイント（`/health`）が存在するか
- Step 8 / Step 9 の担当が参照する共通契約（エラー形式、認証/テナントコンテキスト、RBAC 入口、OpenAPI 生成物配置、テストヘルパー配置）が固定されているか

---

## 成果物ファイルの扱い

| 成果物 | 出力先 | 作成タスク |
|--------|--------|-----------|
| バックエンドコード（基盤） | `expense-saas/apps/api/` | 7-B |
| フロントエンドコード（基盤） | `expense-saas/apps/web/` | 7-B |
| DB マイグレーション | `expense-saas/database/migrations/` | 7-B |
| Docker Compose | `expense-saas/docker/` | 7-B |
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

- **入力**: architecture.md, openapi.yaml, db_schema.md, authz.md, security.md, monitoring.md, subagent-workflow.md
- **出力**: `expense-saas/` 配下のディレクトリ構造・共通基盤
- **ゴール**: 7-B 完了後、各機能タスク（8-A〜8-G）が依存関係に従って並列開発できる状態にする
- **作業内容**:
  - Go バックエンド初期化（go.mod, cmd/api/main.go, internal/ 構造）
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
