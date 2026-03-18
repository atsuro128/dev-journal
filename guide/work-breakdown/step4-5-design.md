# Step 4+5: 基本設計 + 詳細設計 — 作業分解

## 概要

基本設計（画面・UI フロー）と詳細設計（API・DB・認可・セキュリティ）を**機能単位の縦割り**で進める。
画面設計と API 設計は密結合（画面の入出力 = API の入出力）のため、1 つの機能について画面から API まで一貫して設計する。

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
| アーキテクチャ | `deliverables/docs/30_arch/architecture.md` |
| ADR | `deliverables/docs/30_arch/adr/` |

### 完了条件（Step 全体）

- 主要画面（申請、承認キュー、管理画面）の仕様が揃っている
- 権限で操作可否が変わる点が明記されている
- 画面遷移図がある
- API の入出力が決まっている（OpenAPI で機械可読）
- 主要テーブル / 関係・インデックス方針が決まっている
- 認可チェックの責務と実装場所が決まっている
- セキュリティ要件の実装方針が明記されている

---

## 成果物ファイルの扱い

| 成果物 | パス | 初回作成（タスクID） | 追記（タスクID） |
|--------|------|---------------------|-----------------|
| 画面一覧 | `deliverables/docs/40_basic_design/screens.md` | 4+5-I | C-1, D-1, E-1, F-1（機能別セクション詳細化） |
| 画面遷移 | `deliverables/docs/40_basic_design/ui_flow.md` | 4+5-I（初版） | 4+5-G（最終統合） |
| DB スキーマ | `deliverables/docs/50_detail_design/db_schema.md` | 4+5-A | — |
| OpenAPI | `deliverables/docs/50_detail_design/openapi.yaml` | C-2（認証） | D-2, E-2, F-2（機能別に追記） |
| 認可設計 | `deliverables/docs/50_detail_design/authz.md` | 4+5-G | — |
| 添付ファイル設計 | `deliverables/docs/50_detail_design/files.md` | E-2 | — |
| セキュリティ設計 | `deliverables/docs/50_detail_design/security.md` | 4+5-B | — |
| 監視・ログ設計 | `deliverables/docs/50_detail_design/monitoring.md` | 4+5-H | — |

---

## Wave 構成

### Phase 0: 計画

design-architect が上流成果物を分析し、タスク実行計画を作成する。ユーザー合意を得てから Wave 1 に進む。

| ID | タスク | 担当エージェント | 出力先 | 依存 |
|----|--------|----------------|--------|------|
| 4+5-0 | タスク実行計画策定 | design-architect | `task-plans/step4-5.md` | Step 3 完了 |

---

### Wave 1: 基盤設計（並列）

**実行パターン**: 全タスク並列（出力先が独立）

| ID | タスク | 担当エージェント | 出力先 | 依存 |
|----|--------|----------------|--------|------|
| 4+5-A | DB スキーマ設計（全体） | db-designer | `50_detail_design/db_schema.md` | Phase 0 完了 |
| 4+5-B | セキュリティ設計（横断） | detail-designer | `50_detail_design/security.md` | Phase 0 完了 |
| 4+5-H | 監視・ログ詳細設計（横断） | detail-designer | `50_detail_design/monitoring.md` | Phase 0 完了 |
| 4+5-I | 画面一覧・遷移の全体設計 | basic-designer | `40_basic_design/screens.md`, `ui_flow.md` | Phase 0 完了 |

```
┌─ db-designer ──────→ db_schema.md       [4+5-A]
├─ detail-designer ──→ security.md        [4+5-B]
├─ detail-designer ──→ monitoring.md      [4+5-H]
└─ basic-designer ───→ screens.md + ui_flow.md [4+5-I]
```

#### Wave 1 完了後レビュー

```
design-unit-reviewer × 各成果物（並列）: 基盤成果物の上流整合性スポットチェック
  → 品質ゲート判定 → ユーザーに PR マージ確認
```

---

### Wave 2: 機能設計 1 — 認証・経費CRUD（機能間並列・機能内直列）

**実行パターン**: 機能単位並列、機能内は basic-designer → detail-designer の直列

| ID | タスク | 担当エージェント | 出力先 | 依存 |
|----|--------|----------------|--------|------|
| 4+5-C-1 | 認証・ユーザー管理（画面詳細化） | basic-designer | `screens.md` §認証 | Wave 1 完了 |
| 4+5-C-2 | 認証・ユーザー管理（API 定義） | detail-designer | `openapi.yaml` §認証 | 4+5-C-1 |
| 4+5-D-1 | 経費レポート CRUD（画面詳細化） | basic-designer | `screens.md` §経費 | Wave 1 完了 |
| 4+5-D-2 | 経費レポート CRUD（API 定義） | detail-designer | `openapi.yaml` §経費 | 4+5-D-1 |

```
┌─ [認証] basic-designer → screens.md §認証      [4+5-C-1]
│          ↓（完了後）
│         detail-designer → openapi.yaml §認証    [4+5-C-2]
│
└─ [経費] basic-designer → screens.md §経費       [4+5-D-1]
           ↓（完了後）
          detail-designer → openapi.yaml §経費    [4+5-D-2]
```

直列にする理由: API 設計は画面仕様（入力項目・バリデーション・表示条件）を参照するため、画面設計の完了が前提。

#### Wave 2 完了後レビュー

```
design-unit-reviewer × 2（並列）:
  - 認証機能の設計整合性レビュー（画面 × API × DB × 上流）
  - 経費CRUD機能の設計整合性レビュー（画面 × API × DB × 上流）
  → 品質ゲート判定 → ユーザーに PR マージ確認
```

---

### Wave 3: 機能設計 2 — 添付・承認（機能間並列・機能内直列）

**実行パターン**: 機能単位並列、機能内は basic-designer → detail-designer の直列

| ID | タスク | 担当エージェント | 出力先 | 依存 |
|----|--------|----------------|--------|------|
| 4+5-E-1 | 添付ファイル（画面詳細化） | basic-designer | `screens.md` §添付 | Wave 2 完了 |
| 4+5-E-2 | 添付ファイル（API + S3 設計） | detail-designer | `openapi.yaml` §添付, `files.md` | 4+5-E-1 |
| 4+5-F-1 | 承認フロー（画面詳細化） | basic-designer | `screens.md` §承認 | Wave 2 完了 |
| 4+5-F-2 | 承認フロー（API 定義） | detail-designer | `openapi.yaml` §承認 | 4+5-F-1 |

```
┌─ [添付] basic-designer → screens.md §添付           [4+5-E-1]
│          ↓（完了後）
│         detail-designer → openapi.yaml §添付 + files.md [4+5-E-2]
│
└─ [承認] basic-designer → screens.md §承認            [4+5-F-1]
           ↓（完了後）
          detail-designer → openapi.yaml §承認         [4+5-F-2]
```

直列にする理由: Wave 2 と同様。

#### Wave 3 完了後レビュー

```
design-unit-reviewer × 2（並列）:
  - 添付ファイル機能の設計整合性レビュー（画面 × API × S3 × DB × 上流）
  - 承認フロー機能の設計整合性レビュー（画面 × API × 状態遷移 × DB × 上流）
  → 品質ゲート判定 → ユーザーに PR マージ確認
```

---

### Wave 4: 認可設計 + 最終統合

**実行パターン**: 直列（design-architect → design-cross-reviewer）

Wave 1〜3 で各設計者が作った「部品」を、design-architect が全体視点で組み合わせる工程。

| ID | タスク | 担当エージェント | 出力先 | 依存 |
|----|--------|----------------|--------|------|
| 4+5-G | 認可設計 + 最終統合 | design-architect | `authz.md`, `ui_flow.md`（最終版） | Wave 3 完了 |

#### Wave 4 完了後レビュー

```
design-cross-reviewer: 全機能横断レビュー
  - 機能間整合性、用語一貫性、RBAC 完全性、上流トレーサビリティ
  → 品質ゲート判定 → ユーザーに PR マージ確認
  → /codex-review（Step 成果物完成後の外部レビュー）
```

---

## タスク詳細

### 4+5-0: タスク実行計画策定

- **担当エージェント**: design-architect
- **入力**: 本ファイル（step4-5-design.md）、上流成果物（Step 0〜3）
- **出力**: `dev-journal/progress-management/task-plans/step4-5.md`
- **作業内容**:
  - 上流成果物の状態を確認し、リスク（不整合等）があれば報告
  - 各タスクの入力・出力・受け入れ基準を整理
  - タスク実行計画ファイルを作成
- **完了条件**:
  - タスク実行計画ファイルが作成されている
  - ユーザーが計画に合意している

### 4+5-A: DB スキーマ設計（全体）

- **担当エージェント**: db-designer
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

- **担当エージェント**: detail-designer
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

### 4+5-H: 監視・ログ詳細設計（横断）

- **担当エージェント**: detail-designer
- **入力**: adr/0005-monitoring-logging.md, architecture.md, requirements.md（§4.1, §4.3）
- **出力**: `50_detail_design/monitoring.md`
- **作業内容**:
  - ログフォーマット定義（構造化ログのフィールド: timestamp, level, request_id, tenant_id, user_id, message 等）
  - ログレベル運用基準（ERROR / WARN / INFO / DEBUG の使い分け）
  - メトリクス定義（API レスポンスタイム、エラーレート、DB 接続プール使用率等）
  - アラート閾値（p95 レスポンスタイム > 500ms、エラーレート > 1% 等）
  - ヘルスチェックエンドポイント仕様（`/health` のレスポンス形式、チェック項目）
- **完了条件**:
  - ログのフィールド定義と出力先が決まっている
  - MVP で計測するメトリクスとアラート閾値が一覧化されている
  - ヘルスチェックの仕様が定義されている

### 4+5-I: 画面一覧・遷移の全体設計

- **担当エージェント**: basic-designer
- **入力**: usecases.md, requirements.md, rbac.md, workflow.md, architecture.md §5.1（API 一覧）
- **出力**: `40_basic_design/screens.md`（画面一覧）, `40_basic_design/ui_flow.md`（画面遷移図 — 初版）
- **作業内容**:
  - ユースケース・ワークフローから必要な全画面を洗い出す
  - 画面一覧の定義（画面名・目的・主要表示項目・対応ロール）
  - 共通 UI パターンの定義（ヘッダー、ナビゲーション、エラー表示、ローディング）
  - 画面遷移図の作成（Mermaid — ロール別の遷移パス）
  - 機能横断の画面（ダッシュボード等）の定義
- **完了条件**:
  - 全画面の一覧（画面名・目的・対応ロール）が定義されている
  - 画面遷移図が全画面を網羅している
  - 共通 UI パターンが定義されている
  - 後続の機能タスク（C〜F）が画面一覧を参照して詳細化できる状態になっている

### 4+5-C-1: 認証・ユーザー管理（画面詳細化）

- **担当エージェント**: basic-designer
- **入力**: requirements.md §2.1, rbac.md, screens.md（4+5-I で作成した認証関連画面）
- **出力**: `screens.md` §認証（詳細化）
- **作業内容**:
  - 4+5-I で定義した認証関連画面を詳細化（入力項目・バリデーション・エラー表示）
  - 画面ごとの項目・バリデーション・権限
- **完了条件**:
  - 認証画面の仕様（入力項目・エラー表示・遷移先）が定義されている

### 4+5-C-2: 認証・ユーザー管理（API 定義）

- **担当エージェント**: detail-designer
- **入力**: requirements.md §2.1, rbac.md, db_schema.md（4+5-A）, screens.md §認証（4+5-C-1）
- **出力**: `openapi.yaml` §認証エンドポイント
- **作業内容**:
  - API: POST /auth/signup, POST /auth/login, POST /auth/refresh, POST /auth/logout, GET /auth/me, POST /auth/password-reset/*
  - **上流 API 一覧との突き合わせ**: rbac.md §3.1 および architecture.md §5.1 の認証エンドポイントと照合し、過不足を確認・記録
- **完了条件**:
  - 認証 API の OpenAPI 定義（リクエスト/レスポンス/エラー）が書かれている
  - 上流 API 一覧との差分が記録されている

### 4+5-D-1: 経費レポート CRUD（画面詳細化）

- **担当エージェント**: basic-designer
- **入力**: requirements.md §2.3, domain_model.md, screens.md（4+5-I で作成した経費関連画面）
- **出力**: `screens.md` §経費（詳細化）
- **作業内容**:
  - 4+5-I で定義した経費関連画面を詳細化（レポート一覧、作成/編集、明細追加/編集、詳細）
  - ロール別の表示差異（Member: 自分のみ、Admin: 全件、Accounting: 承認済み以降）
  - ページネーション・フィルタリング・ソートの仕様
- **完了条件**:
  - CRUD 全操作の画面仕様が定義されている
  - ロール別の表示・操作制限が明記されている

### 4+5-D-2: 経費レポート CRUD（API 定義）

- **担当エージェント**: detail-designer
- **入力**: requirements.md §2.3, domain_model.md, db_schema.md（4+5-A）, screens.md §経費（4+5-D-1）
- **出力**: `openapi.yaml` §経費エンドポイント
- **作業内容**:
  - API: GET/POST/PUT/DELETE /reports, GET/POST/PUT/DELETE /reports/{id}/items
  - ページネーション・フィルタリング・ソートの API 仕様
  - **上流 API 一覧との突き合わせ**: rbac.md §3.2 および architecture.md §5.1 の経費エンドポイントと照合し、過不足を確認・記録
- **完了条件**:
  - API のリクエスト/レスポンス/エラー形式が OpenAPI で定義されている
  - 上流 API 一覧との差分が記録されている

### 4+5-E-1: 添付ファイル（画面詳細化）

- **担当エージェント**: basic-designer
- **入力**: requirements.md §2.5, domain_model.md（Attachment エンティティ）, screens.md §経費（4+5-D-1）
- **出力**: `screens.md` §添付（詳細化）
- **作業内容**:
  - 明細内の添付エリア（アップロード・プレビュー・削除）の画面仕様
- **完了条件**:
  - 添付ファイルの画面操作仕様が定義されている

### 4+5-E-2: 添付ファイル（API + S3 設計）

- **担当エージェント**: detail-designer
- **入力**: requirements.md §2.5, db_schema.md（4+5-A）, screens.md §添付（4+5-E-1）
- **出力**: `openapi.yaml` §添付エンドポイント, `50_detail_design/files.md`
- **作業内容**:
  - API: 署名付き URL 発行（アップロード用・ダウンロード用）
  - S3 バケット構成、キー命名規則（tenant_id/report_id/item_id/filename）
  - MIME バリデーション（許可形式: image/*, application/pdf）
  - 発行前認可チェック（リクエスト者がそのレポートにアクセス可能か）
  - **上流 API 一覧との突き合わせ**: rbac.md §3.4 および architecture.md §5.1 の添付エンドポイントと照合
- **完了条件**:
  - 署名付き URL の発行フローが図示されている
  - MIME / サイズ制限が明記されている
  - 認可チェックの責務（API 層で実施）が明記されている

### 4+5-F-1: 承認フロー（画面詳細化）

- **担当エージェント**: basic-designer
- **入力**: requirements.md §2.4, state_machine.md, workflow.md, screens.md §経費（4+5-D-1）
- **出力**: `screens.md` §承認（詳細化）
- **作業内容**:
  - 4+5-I で定義した承認関連画面を詳細化（承認キュー、承認/却下アクション）
  - 却下理由の必須入力、差戻し後の再編集→再提出フロー
- **完了条件**:
  - 承認キュー画面の仕様（フィルタ・ソート・ロール別表示）が定義されている

### 4+5-F-2: 承認フロー（API 定義）

- **担当エージェント**: detail-designer
- **入力**: requirements.md §2.4, state_machine.md, workflow.md, db_schema.md（4+5-A）, screens.md §承認（4+5-F-1）
- **出力**: `openapi.yaml` §承認エンドポイント
- **作業内容**:
  - API: POST /reports/{id}/submit, POST /reports/{id}/approve, POST /reports/{id}/reject, POST /reports/{id}/mark-paid
  - 状態遷移の API レベルでの強制（state_machine.md の遷移表に準拠）
  - **上流 API 一覧との突き合わせ**: rbac.md §3.3 および architecture.md §5.1 の承認エンドポイントと照合
- **完了条件**:
  - 承認フローの全アクションが API 定義されている
  - 状態遷移の API レスポンス（成功/不正遷移エラー）が定義されている

### 4+5-G: 認可設計 + 最終統合

- **担当エージェント**: design-architect
- **入力**: 4+5-C〜F の全成果物, 4+5-I の成果物, rbac.md, security.md（4+5-B）
- **出力**: `50_detail_design/authz.md`, `40_basic_design/ui_flow.md`（最終版）
- **作業内容**:
  - 認可設計:
    - RBAC ミドルウェアの設計（エンドポイント × ロール の許可マトリクス）
    - テナント分離の認可チェック（アプリ層 + RLS の二重保証）
    - リソースオーナーシップチェック（自分のレポートのみ編集可能等）
  - 最終統合:
    - 各機能タスクで詳細化された画面仕様と 4+5-I の画面一覧の整合性確認
    - 画面遷移図の最終更新（機能タスクで追加・変更された画面を反映）
    - 全 API エンドポイントの最終一覧と上流（rbac.md, architecture.md §5.1）との差分記録
- **完了条件**:
  - 全エンドポイントの認可ルール（ロール × 操作）が一覧化されている
  - 画面遷移図が全画面を網羅し、機能タスクの詳細化結果と整合している
  - 認可チェックの実装場所（ミドルウェア / ハンドラー / リポジトリ）が明確
  - 上流 API 一覧との差分（追加・削除・変更）が記録されている
