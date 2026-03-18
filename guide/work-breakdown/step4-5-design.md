# Step 4+5: 基本設計 + 詳細設計 — 作業分解

## 概要

基本設計（画面・UI フロー）と詳細設計（API・DB・認可・セキュリティ・監視）を**機能単位の縦割り**で進める。
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

| 成果物 | パス | 作成タスク | 追記タスク |
|--------|------|-----------|-----------|
| 画面一覧 | `40_basic_design/screens.md` | 4+5-I | — |
| 画面遷移図 | `40_basic_design/ui_flow.md` | 4+5-I（初版） | 4+5-G（最終統合） |
| 認証 画面仕様 | `40_basic_design/screens/auth.md` | 4+5-C-1 | — |
| 経費 画面仕様 | `40_basic_design/screens/expenses.md` | 4+5-D-1 | — |
| 添付 画面仕様 | `40_basic_design/screens/attachments.md` | 4+5-E-1 | — |
| 承認 画面仕様 | `40_basic_design/screens/approvals.md` | 4+5-F-1 | — |
| DB スキーマ | `50_detail_design/db_schema.md` | 4+5-A | — |
| OpenAPI | `50_detail_design/openapi.yaml` | 4+5-C-2（共通定義 + 認証） | D-2, E-2, F-2（tags 単位で追記） |
| 認可設計 | `50_detail_design/authz.md` | 4+5-G | — |
| 添付ファイル設計 | `50_detail_design/files.md` | 4+5-E-2 | — |
| セキュリティ設計 | `50_detail_design/security.md` | 4+5-B | — |
| 監視・ログ設計 | `50_detail_design/monitoring.md` | 4+5-H | — |

**画面一覧と画面仕様の分離**: 画面一覧（screens.md）は全画面の俯瞰（画面ID・画面名・目的・対応ロール）を定義する。各画面の詳細仕様（入力項目・バリデーション・エラー表示等）は機能別ファイル（screens/*.md）に分離する。抽象度が異なるため同一ファイルに混在させない。

**openapi.yaml の追記方式**: C-2 で共通定義（info, servers, securitySchemes, Error/Pagination スキーマ）と認証エンドポイントを初回作成する。以降のタスクは tags セクション単位で追記する。

---

## タスク一覧

| ID | タスク | 種別 |
|----|--------|------|
| 4+5-A | DB スキーマ設計（全体） | 基盤 |
| 4+5-B | セキュリティ設計（横断） | 基盤 |
| 4+5-H | 監視・ログ詳細設計（横断） | 基盤 |
| 4+5-I | 画面一覧・遷移の全体設計 | 基盤 |
| 4+5-C-1 | 認証・ユーザー管理（画面詳細化） | 機能 |
| 4+5-C-2 | 認証・ユーザー管理（API 定義） | 機能 |
| 4+5-D-1 | 経費レポート CRUD + ダッシュボード（画面詳細化） | 機能 |
| 4+5-D-2 | 経費レポート CRUD + ダッシュボード（API 定義） | 機能 |
| 4+5-E-1 | 添付ファイル（画面詳細化） | 機能 |
| 4+5-E-2 | 添付ファイル（API + S3 設計） | 機能 |
| 4+5-F-1 | 承認フロー（画面詳細化） | 機能 |
| 4+5-F-2 | 承認フロー（API 定義） | 機能 |
| 4+5-G | 認可設計 + 最終統合 | 統合 |

---

## タスク詳細

### 4+5-A: DB スキーマ設計（全体）

- **入力**: domain_model.md, state_machine.md, ADR 0002 (multi-tenant), ADR 0003 (RLS), requirements.md §4.4
- **出力**: `50_detail_design/db_schema.md`
- **作業内容**:
  - domain_model.md のエンティティを PostgreSQL テーブルに変換
  - tenant_id カラムの配置、RLS ポリシー定義
  - インデックス方針（tenant_id 複合インデックス等）
  - マイグレーション方針（golang-migrate）
  - audit_logs テーブル（INSERT ONLY 制約、Phase 3 事前設計）
  - 論理削除（deleted_at）、タイムスタンプ（created_at, updated_at）の統一方針
- **完了条件**:
  - 全主要テーブルの DDL（概要レベル）が定義されている
  - RLS ポリシーの適用対象テーブルと具体的なポリシー定義が記載されている
  - インデックス方針が記載されている
  - domain_model.md の全エンティティ・全属性がテーブル定義に対応している

### 4+5-B: セキュリティ設計（横断）

- **入力**: requirements.md §4.2（SEC-001〜014）, architecture.md §3.2, §3.3, §6
- **出力**: `50_detail_design/security.md`
- **作業内容**:
  - レート制限（認証済み: 100 req/min/user、未認証: 20 req/min/IP）
  - CORS 方針（許可オリジン、メソッド、ヘッダー、credentials）
  - セキュリティヘッダー（HSTS, X-Content-Type-Options, X-Frame-Options 等）
  - JWT 検証フロー（RS256、鍵管理、トークンローテーション）
  - パスワードリセットのトークン生成・検証・有効期限フロー
  - govulncheck / npm audit の CI 組み込み方針
- **完了条件**:
  - レート制限の具体的な数値と実装方式が決まっている
  - セキュリティヘッダー一覧が定義されている
  - JWT 検証フローが定義されている

### 4+5-H: 監視・ログ詳細設計（横断）

- **入力**: adr/0005-monitoring-logging.md, architecture.md §7, requirements.md §4.1/§4.3, security.md（4+5-B）
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

- **入力**: usecases.md, requirements.md, rbac.md, workflow.md, architecture.md §4.3/§5.1
- **出力**: `40_basic_design/screens.md`（画面一覧）, `40_basic_design/ui_flow.md`（画面遷移図 — 初版）
- **作業内容**:
  - ユースケース・ワークフローから必要な全画面を洗い出す
  - 画面一覧の定義（画面ID・画面名・目的・主要表示項目・対応ロール）
  - 共通 UI パターンの定義（ヘッダー、ナビゲーション、エラー表示、ローディング、確認ダイアログ）
  - 画面遷移図の作成（Mermaid — ロール別の遷移パス）
  - 画面IDの命名規則を定義（後続タスクで参照可能にする）
- **完了条件**:
  - 全画面の一覧（画面ID・画面名・目的・対応ロール）が定義されている
  - 画面遷移図が全画面を網羅している
  - 共通 UI パターンが定義されている
  - 後続の機能タスク（C〜F）が画面一覧を参照して詳細化できる状態になっている
- **注意**: ここでは全体の俯瞰を作る。各画面の入力項目やバリデーションの詳細は機能タスクで定義

### 4+5-C-1: 認証・ユーザー管理（画面詳細化）

- **入力**: requirements.md §2.1, rbac.md §3.1, usecases.md UC-SYS01〜03/UC-SYS05, screens.md（4+5-I）
- **出力**: `40_basic_design/screens/auth.md`
- **作業内容**:
  - ログイン画面の仕様（入力項目・バリデーション・エラー表示・遷移先）
  - サインアップ画面の仕様
  - パスワードリセット画面（要求・実行の 2 画面）の仕様
  - SEC-011（認証失敗時のユーザー存在漏洩防止）をエラー表示に反映
- **完了条件**:
  - 認証画面の仕様（入力項目・エラー表示・遷移先）が定義されている
  - screens.md の画面IDと対応している

### 4+5-C-2: 認証・ユーザー管理（API 定義）

- **入力**: requirements.md §2.1, rbac.md §3.1, architecture.md §5.1/§5.2/§5.3, db_schema.md（4+5-A）, screens/auth.md（4+5-C-1）
- **出力**: `50_detail_design/openapi.yaml`（初回作成: 共通定義 + 認証エンドポイント）
- **作業内容**:
  - openapi.yaml の共通部分（info, servers, securitySchemes, Error/Pagination スキーマ）を定義
  - 認証エンドポイント: POST /auth/signup, login, refresh, logout, GET /auth/me, POST /auth/password-reset/*
  - rbac.md §3.1 および architecture.md §5.1 との突き合わせ、差分記録
- **完了条件**:
  - openapi.yaml の共通定義が整っている
  - 認証 API のリクエスト/レスポンス/エラーが定義されている
  - 上流 API 一覧との差分が記録されている
- **注意**: openapi.yaml の**初回作成タスク**。後続タスクが追記する前提で拡張しやすく設計する

### 4+5-D-1: 経費レポート CRUD + ダッシュボード（画面詳細化）

- **入力**: requirements.md §2.4/§2.7, domain_model.md §3.4/§3.5, rbac.md §3.2, usecases.md UC-M01〜M09/UC-SYS04/UC-AD01〜02/UC-AC03, screens.md（4+5-I）
- **出力**: `40_basic_design/screens/expenses.md`
- **作業内容**:
  - レポート一覧画面（フィルタ・ソート・ページネーション・ロール別表示差異）
  - レポート作成/編集画面（入力項目・バリデーション）
  - レポート詳細画面（明細一覧・状態表示・アクションボタンの出しわけ）
  - 明細追加/編集（入力項目・カテゴリ選択・バリデーション）
  - ダッシュボード画面（ロール別表示内容: DASH-001〜004）
  - 再申請フロー（UC-M09）の画面操作
- **完了条件**:
  - CRUD 全操作の画面仕様が定義されている
  - ロール別の表示・操作制限が明記されている
  - ダッシュボード画面の仕様（ロール別）が定義されている

### 4+5-D-2: 経費レポート CRUD + ダッシュボード（API 定義）

- **入力**: requirements.md §2.4/§2.7, domain_model.md, rbac.md §3.2, architecture.md §5.1/§5.3, db_schema.md（4+5-A）, screens/expenses.md（4+5-D-1）, openapi.yaml（4+5-C-2）
- **出力**: `50_detail_design/openapi.yaml`（追記: 経費エンドポイント）
- **作業内容**:
  - 経費レポート CRUD: GET/POST/PUT/DELETE /reports, GET /reports/all
  - 経費明細 CRUD: POST/PUT/DELETE /reports/:id/items
  - ダッシュボード: GET /dashboard
  - テナント情報取得: GET /tenant
  - ページネーション・フィルタリング・ソートのクエリパラメータ
  - rbac.md §3.2 および architecture.md §5.1 との突き合わせ、差分記録
  - レポート作成 API に reference_report_id（再申請元）をオプションパラメータとして含める
- **完了条件**:
  - 経費関連 API のリクエスト/レスポンス/エラーが定義されている
  - 上流 API 一覧との差分が記録されている

### 4+5-E-1: 添付ファイル（画面詳細化）

- **入力**: requirements.md §2.6, domain_model.md §3.6/§4.4, usecases.md UC-M03/UC-M03a, screens/expenses.md（4+5-D-1）
- **出力**: `40_basic_design/screens/attachments.md`
- **作業内容**:
  - 添付ファイルのアップロード UI（ファイル選択・ドラッグ&ドロップ・プログレス表示）
  - ファイル形式制限（JPEG, PNG, PDF）とサイズ制限（5MB）のバリデーション表示
  - プレビュー・ダウンロード操作
  - 削除操作（確認ダイアログ含む）
  - draft 以外の状態ではアップロード・削除ボタンの非表示/無効化
- **完了条件**:
  - 添付ファイルの画面操作仕様が定義されている

### 4+5-E-2: 添付ファイル（API + S3 設計）

- **入力**: requirements.md §2.6（ATT-002〜020）, domain_model.md §3.6, rbac.md §3.4, architecture.md §5.1, db_schema.md（4+5-A）, screens/attachments.md（4+5-E-1）, openapi.yaml（4+5-C-2）
- **出力**: `50_detail_design/openapi.yaml`（追記: 添付エンドポイント）, `50_detail_design/files.md`
- **作業内容**:
  - 添付ファイルエンドポイント（POST, GET, GET/:attId, DELETE）
  - 署名付き URL 発行フロー（アップロード用・ダウンロード用）の図示
  - S3 バケット構成、キー命名規則（`{tenant_id}/{report_id}/{attachment_id}`）
  - MIME バリデーション（JPEG, PNG, PDF）とサイズ制限（5MB）の実装方式
  - 発行前認可チェック（リクエスト者がそのレポートにアクセス可能か）
  - rbac.md §3.4 および architecture.md §5.1 との突き合わせ
- **完了条件**:
  - 署名付き URL の発行フローが図示されている
  - MIME / サイズ制限が明記されている
  - 認可チェックの責務（API 層で実施）が明記されている

### 4+5-F-1: 承認フロー（画面詳細化）

- **入力**: requirements.md §2.5, state_machine.md, workflow.md, rbac.md §3.3, usecases.md UC-A01〜03/UC-AC01〜02, screens/expenses.md（4+5-D-1）
- **出力**: `40_basic_design/screens/approvals.md`
- **作業内容**:
  - 承認待ち一覧画面（Approver 向け: フィルタ・ソート・申請者名・合計金額表示）
  - 承認/却下操作（確認ダイアログ・却下理由必須入力 WFL-012）
  - 支払待ち一覧画面（Accounting 向け）
  - 支払完了操作（確認ダイアログ）
  - 自己承認禁止の UI 表現
- **完了条件**:
  - 承認キュー画面の仕様（フィルタ・ソート・ロール別表示）が定義されている
  - 承認/却下後の画面遷移が定義されている

### 4+5-F-2: 承認フロー（API 定義）

- **入力**: requirements.md §2.5, state_machine.md §3/§4, workflow.md §4, rbac.md §3.3, architecture.md §5.1, db_schema.md（4+5-A）, screens/approvals.md（4+5-F-1）, openapi.yaml（4+5-C-2, D-2）
- **出力**: `50_detail_design/openapi.yaml`（追記: 承認エンドポイント）
- **作業内容**:
  - 承認フローエンドポイント: submit, approve, reject, mark-paid, pending, payable
  - 状態遷移の API レベルでの強制（state_machine.md の遷移表に準拠）
  - 不正遷移時のエラーレスポンス（422 INVALID_STATE_TRANSITION）
  - 楽観的ロック（409 Conflict）のエラーレスポンス
  - rbac.md §3.3 および architecture.md §5.1 との突き合わせ
- **完了条件**:
  - 承認フローの全アクションが API 定義されている
  - 状態遷移の API レスポンス（成功/不正遷移エラー）が定義されている

### 4+5-G: 認可設計 + 最終統合

- **入力**: 4+5-C-2〜F-2 の全成果物, 4+5-I の成果物, screens/*.md, rbac.md, security.md（4+5-B）, architecture.md §5.1
- **出力**: `50_detail_design/authz.md`, `40_basic_design/ui_flow.md`（最終版）
- **作業内容**:
  - 認可設計:
    - RBAC ミドルウェアの設計（全エンドポイント × 全ロール の認可マトリクス）
    - テナント分離の認可チェック（アプリ層 + RLS の二重保証）
    - リソースオーナーシップチェック（自分のレポートのみ編集可能等）
    - 自己承認禁止の実装方針
    - 認可チェックの実装場所（ミドルウェア / ハンドラ / ドメイン層 / リポジトリ）をエンドポイントごとに定義
  - 最終統合:
    - 画面遷移図の最終更新（機能タスクで追加・変更された画面を反映）
    - 画面ID → API パス → DB テーブル → 認可ルール の一貫性検証
    - 全 API エンドポイントの最終一覧と上流（rbac.md, architecture.md §5.1）との差分集約
- **完了条件**:
  - 全エンドポイントの認可ルール（ロール × 操作）が一覧化されている
  - 認可チェックの実装場所がエンドポイントごとに明確
  - 画面遷移図が全画面を網羅し、機能タスクの結果と整合している
  - 上流 API 一覧との差分（追加・削除・変更）が記録されている
