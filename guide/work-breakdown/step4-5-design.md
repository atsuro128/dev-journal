# Step 4+5: 基本設計 + 詳細設計 — 作業分解

## 概要

基本設計（画面・UI フロー）と詳細設計（API・DB・認可・セキュリティ）を**機能単位の縦割り**で進める。
従来の Step 4 → Step 5 の横割り（全画面 → 全 API）ではなく、1つの機能について画面から API まで一貫して設計する。

## 方針

### なぜ統合するか

- 画面設計と API 設計は密結合（画面の入出力 = API の入出力）
- 機能単位にすることで、複数セッションの並列実行が可能
- 手戻りが機能内に閉じる

### 成果物ファイルの扱い

出力先は project_steps.md の定義通り。複数タスクが同じファイルに書き込む場合がある。

| 成果物 | パス | 書き込むタスク |
|--------|------|---------------|
| 画面一覧 | `deliverables/docs/40_basic_design/screens.md` | 各機能タスク |
| 画面遷移 | `deliverables/docs/40_basic_design/ui_flow.md` | 4+5-G（統合） |
| DB スキーマ | `deliverables/docs/50_detail_design/db_schema.md` | 4+5-A |
| OpenAPI | `deliverables/docs/50_detail_design/openapi.yaml` | 各機能タスク |
| 認可設計 | `deliverables/docs/50_detail_design/authz.md` | 4+5-F |
| 添付ファイル設計 | `deliverables/docs/50_detail_design/files.md` | 4+5-E |
| セキュリティ設計 | `deliverables/docs/50_detail_design/security.md` | 4+5-B |

## 前提

### 上流成果物（入力）

| 成果物 | パス |
|--------|------|
| ドメインモデル | `deliverables/docs/20_domain/domain_model.md` |
| 状態遷移 | `deliverables/docs/20_domain/state_machine.md` |
| 要件定義 | `deliverables/docs/10_requirements/requirements.md` |
| ユースケース | `deliverables/docs/10_requirements/usecases.md` |
| RBAC | `deliverables/docs/10_requirements/rbac.md` |
| ワークフロー | `deliverables/docs/10_requirements/workflow.md` |
| アーキテクチャ | `deliverables/docs/30_arch/architecture.md`（Step 3 成果物） |
| ADR | `deliverables/docs/30_arch/adr/`（Step 3 成果物） |

### 完了条件（Step 4 + Step 5 統合）

- 主要画面（申請、承認キュー、管理画面）の仕様が揃っている
- 権限で操作可否が変わる点が明記されている
- 画面遷移図がある
- API の入出力が決まっている（OpenAPI で機械可読）
- 主要テーブル / 関係・インデックス方針が決まっている
- 認可チェックの責務と実装場所が決まっている
- セキュリティ要件の実装方針が明記されている

---

## タスク一覧

| ID | タスク | 種別 | 依存 | 並列可 | 状態 |
|----|--------|------|------|--------|------|
| 4+5-A | DB スキーマ設計（全体） | 基盤 | Step 3 完了 | Yes | 未着手 |
| 4+5-B | セキュリティ設計（横断） | 基盤 | Step 3 完了 | Yes | 未着手 |
| 4+5-C | 認証・ユーザー管理（画面 + API） | 機能 | 4+5-A | Yes | 未着手 |
| 4+5-D | 経費レポート CRUD（画面 + API） | 機能 | 4+5-A | Yes | 未着手 |
| 4+5-E | 添付ファイル（画面 + API + S3） | 機能 | 4+5-A, 4+5-D | No | 未着手 |
| 4+5-F | 承認フロー（画面 + API） | 機能 | 4+5-A, 4+5-D | No | 未着手 |
| 4+5-G | 認可設計 + 画面遷移統合 | 統合 | 4+5-C〜F | No | 未着手 |

### 依存グラフ

```
Step 3 完了
  ├→ 4+5-A (DB スキーマ) ─→ 4+5-C (認証) ─────────────┐
  │                       ├→ 4+5-D (経費CRUD) → 4+5-E (添付) ─→ 4+5-G (認可 + 統合)
  │                       │                  → 4+5-F (承認) ─┘
  └→ 4+5-B (セキュリティ) ──────────────────────────────────┘
```

**並列実行パターン**:
- Wave 1: 4+5-A, 4+5-B（同時実行）
- Wave 2: 4+5-C, 4+5-D（同時実行）
- Wave 3: 4+5-E, 4+5-F（同時実行）
- Wave 4: 4+5-G（統合・整合性チェック）

---

## タスク詳細

### 4+5-A: DB スキーマ設計（全体）

全機能の基盤となるテーブル定義を設計する。

- **入力**: domain_model.md, state_machine.md, ADR 0002 (multi-tenant), ADR 0003 (RLS)
- **出力**: `50_detail_design/db_schema.md`
- **作業内容**:
  - domain_model.md のエンティティを PostgreSQL テーブルに変換
  - tenant_id カラムの配置、RLS ポリシー定義
  - インデックス方針（tenant_id 複合インデックス等）
  - マイグレーション方針（ツール選定: golang-migrate 等）
  - audit_logs テーブル（INSERT ONLY 制約）
- **完了条件**:
  - 全主要テーブルの DDL（概要レベル）が定義されている
  - RLS ポリシーの適用対象テーブルが明示されている
  - インデックス方針が記載されている

### 4+5-B: セキュリティ設計（横断）

全 API に適用される横断的セキュリティ要件を設計する。

- **入力**: requirements.md（非機能要件セクション）, architecture.md
- **出力**: `50_detail_design/security.md`
- **作業内容**:
  - レート制限（認証済み: 100 req/min/user、未認証: 20 req/min/IP）
  - CORS 方針（許可オリジン、メソッド、ヘッダー）
  - セキュリティヘッダー（HSTS, X-Content-Type-Options 等）
  - JWT 検証フロー（RS256、トークンローテーション）
  - govulncheck / npm audit の CI 組み込み方針
- **完了条件**:
  - レート制限の具体的な数値と実装方式が決まっている
  - セキュリティヘッダー一覧が定義されている

### 4+5-C: 認証・ユーザー管理（画面 + API）

- **入力**: requirements.md §2.1, rbac.md, db_schema.md (4+5-A)
- **出力**: screens.md（認証関連セクション）, openapi.yaml（認証エンドポイント）
- **作業内容**:
  - 画面: ログイン、サインアップ、パスワードリセット
  - API: POST /auth/signup, POST /auth/login, POST /auth/refresh, POST /auth/logout, GET /auth/me, POST /auth/password-reset/*
  - 画面ごとの項目・バリデーション・権限
- **完了条件**:
  - 認証画面の仕様（入力項目・エラー表示・遷移先）が定義されている
  - 認証 API の OpenAPI 定義（リクエスト/レスポンス/エラー）が書かれている

### 4+5-D: 経費レポート CRUD（画面 + API）

- **入力**: requirements.md §2.3, domain_model.md, db_schema.md (4+5-A)
- **出力**: screens.md（経費関連セクション）, openapi.yaml（経費エンドポイント）
- **作業内容**:
  - 画面: レポート一覧、レポート作成/編集、明細追加/編集、レポート詳細
  - API: GET/POST/PUT/DELETE /reports, GET/POST/PUT/DELETE /reports/{id}/items
  - ロール別の表示差異（Member: 自分のみ、Admin: 全件、Accounting: 承認済み以降）
  - ページネーション・フィルタリング・ソートの仕様
- **完了条件**:
  - CRUD 全操作の画面仕様が定義されている
  - ロール別の表示・操作制限が明記されている
  - API のリクエスト/レスポンス/エラー形式が OpenAPI で定義されている

### 4+5-E: 添付ファイル（画面 + API + S3 設計）

- **入力**: requirements.md §2.5, domain_model.md（Attachment エンティティ）, db_schema.md (4+5-A)
- **出力**: screens.md（添付関連セクション）, openapi.yaml（添付エンドポイント）, `50_detail_design/files.md`
- **依存**: 4+5-D（経費レポートの画面・API に添付を統合するため）
- **作業内容**:
  - 画面: 明細内の添付エリア（アップロード・プレビュー・削除）
  - API: 署名付き URL 発行（アップロード用・ダウンロード用）
  - S3 バケット構成、キー命名規則（tenant_id/report_id/item_id/filename）
  - MIME バリデーション（許可形式: image/*, application/pdf）
  - 発行前認可チェック（リクエスト者がそのレポートにアクセス可能か）
- **完了条件**:
  - 署名付き URL の発行フローが図示されている
  - MIME / サイズ制限が明記されている
  - 認可チェックの責務（API 層で実施）が明記されている

### 4+5-F: 承認フロー（画面 + API）

- **入力**: requirements.md §2.4, state_machine.md, workflow.md, db_schema.md (4+5-A)
- **出力**: screens.md（承認関連セクション）, openapi.yaml（承認エンドポイント）
- **依存**: 4+5-D（レポートの状態遷移は経費レポート API と連携するため）
- **作業内容**:
  - 画面: 承認キュー（Approver 用）、レポート詳細の承認/却下アクション
  - API: POST /reports/{id}/submit, POST /reports/{id}/approve, POST /reports/{id}/reject, POST /reports/{id}/mark-paid
  - 状態遷移の API レベルでの強制（state_machine.md の遷移表に準拠）
  - 却下理由の必須入力、差戻し後の再編集→再提出フロー
- **完了条件**:
  - 承認フローの全アクションが API 定義されている
  - 状態遷移の API レスポンス（成功/不正遷移エラー）が定義されている
  - 承認キュー画面の仕様（フィルタ・ソート・ロール別表示）が定義されている

### 4+5-G: 認可設計 + 画面遷移統合

全機能タスクの成果物を統合し、横断的な認可設計と画面遷移図をまとめる。

- **入力**: 4+5-C〜F の全成果物, rbac.md, security.md (4+5-B)
- **出力**: `50_detail_design/authz.md`, `40_basic_design/ui_flow.md`
- **作業内容**:
  - 認可設計:
    - RBAC ミドルウェアの設計（エンドポイント × ロール の許可マトリクス）
    - テナント分離の認可チェック（アプリ層 + RLS の二重保証）
    - リソースオーナーシップチェック（自分のレポートのみ編集可能等）
  - 画面遷移統合:
    - 各機能タスクで定義した画面を統合した全体遷移図（Mermaid）
    - ロール別の遷移パス
- **完了条件**:
  - 全エンドポイントの認可ルール（ロール × 操作）が一覧化されている
  - 画面遷移図が全画面を網羅している
  - 認可チェックの実装場所（ミドルウェア / ハンドラー / リポジトリ）が明確
