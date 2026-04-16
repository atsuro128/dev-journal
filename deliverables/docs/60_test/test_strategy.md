# テスト戦略

## この文書の役割

| 項目 | 内容 |
|------|------|
| 目的 | テストレベル、対象、カバレッジ基準、フィクスチャ方針を定義する |
| 正本情報 | テストレベル、ツール、フィクスチャ、CI 実行方針 |
| 扱わない内容 | 個別テストケースの詳細 |
| 主な参照元 | `50_detail_design/*`, `20_domain/*`, `10_requirements/*` |
| 主な参照先 | `60_test/test_cases/*.md`, `60_test/traceability.md` |

---

## 1. 概要

本書は経費精算SaaS MVP のテスト戦略を定義する。テスト設計者・実装者が共通の方針を持ち、下流のテストケース定義（6-B-1〜8）と実装（Step 9）を迷いなく進めるための正本として機能する。

### 参照ドキュメント

| ドキュメント | 参照内容 |
|------------|---------|
| `50_detail_design/openapi.yaml` | 全エンドポイント定義（31 operationId） |
| `50_detail_design/authz.md` | 認可ルール、認可マトリクス |
| `50_detail_design/db_schema.md` | DBスキーマ、RLSポリシー |
| `50_detail_design/security.md` | JWT設計、テナント分離実装 |
| `50_detail_design/files.md` | 添付ファイル設計、署名付きURL認可 |
| `20_domain/state_machine.md` | 状態遷移（T1〜T5、X1〜X10） |
| `20_domain/domain_model.md` | エンティティ定義、不変条件 |
| `10_requirements/policies.md` | RBACルール（RBC-001〜016）（SS3）、業務ルール（SS5）、状態遷移ルール（SS4） |
| `.claude/rules/testing.md` | テスト方針ルール |

---

## 2. テストレベルと対象ツール

### 2.1 テストピラミッド

```
        ┌──────────────┐
        │   E2E テスト  │  ← Playwright（主要フローのみ）
        │  （少数）     │
        ├──────────────┤
        │  統合テスト   │  ← Go testing + テスト用DB
        │  （中程度）   │
        ├──────────────┤
        │  単体テスト   │  ← Go testing（ドメイン層）
        │  （多数）     │
        └──────────────┘
```

### 2.2 各テストレベルの定義

#### 単体テスト（Unit Test）

| 項目 | 定義 |
|------|------|
| 対象 | ドメイン層のロジック（状態遷移メソッド、バリデーション、不変条件） |
| ツール | Go 標準 `testing` パッケージ |
| ファイル配置 | `*_test.go`（本体と同ディレクトリ） |
| DB使用 | なし（純粋な関数テスト） |
| 実行速度 | 高速（ミリ秒単位） |
| テスト対象の例 | `ExpenseReport.Submit()`、`ExpenseReport.Approve()`、金額バリデーション |

#### 統合テスト（Integration Test）

| 項目 | 定義 |
|------|------|
| 対象 | APIエンドポイント（ハンドラ → サービス → リポジトリ → DB）、認証認可、DB操作 |
| ツール | Go 標準 `testing` パッケージ + `net/http/httptest` |
| ファイル配置 | `*_test.go`（本体と同ディレクトリ、またはテスト専用 `tests/integration/` ディレクトリ） |
| DB使用 | **あり**（テスト用PostgreSQL、Docker Compose で起動） |
| 実行速度 | 中速（秒単位） |
| テスト対象の例 | `POST /api/reports` の全層貫通テスト、RBAC ミドルウェアの許可・拒否、RLS によるテナント分離 |

#### E2E テスト（End-to-End Test）

| 項目 | 定義 |
|------|------|
| 対象 | 主要ユーザーフロー（申請→承認→支払完了、却下→再申請） |
| ツール | Playwright |
| ファイル配置 | `expense-saas/e2e/` 以下 |
| DB使用 | **あり**（E2E用テスト環境のDB） |
| 実行速度 | 低速（分単位） |
| テスト対象の例 | Member がログインしてレポートを作成・提出、Approver がログインして承認 |

---

## 3. カバレッジ目標

| 対象 | 目標 | 計測ツール | 根拠 |
|------|------|----------|------|
| ドメイン層（Go） | **80% 以上** | `go test -cover` | `.claude/rules/testing.md` |
| リポジトリ層（Go） | ベストエフォート | `go test -cover` | 実DB依存のため強制化しない |
| ハンドラ層（Go） | ベストエフォート | `go test -cover` | 統合テストで間接的に検証 |
| フロントエンド（TS） | ベストエフォート | Vitest カバレッジ | MVP では優先度低 |

**補足**: 過度なモックは避ける（`.claude/rules/testing.md`）。カバレッジのために無意味なテストを書くことより、重要なビジネスロジックを網羅することを優先する。

### 2.3 非機能テスト方針

#### レート制限テスト

| 項目 | 内容 |
|------|------|
| テストレベル | 統合テスト |
| 実施タイミング | PR 時（単体テスト + 統合テストに含む） |
| テスト方針 | 制限値を超えるリクエストを連続送信し、`HTTP 429 Too Many Requests` と `Retry-After` ヘッダーが返ることを確認する |
| テストケースの記載先 | `cross-cutting.md`（複数エンドポイントにまたがるため） |

**対象エンドポイントと制限値**（`50_detail_design/security.md` § 4.1 より）:

| 対象 | 制限値 | キー |
|------|--------|------|
| 認証済みリクエスト（全般） | 100 req/min | user_id |
| 未認証リクエスト | 20 req/min | IP アドレス |
| ログイン試行（`POST /api/auth/login`） | 5 req/min | IP アドレス |
| ファイルアップロード（`POST .../attachments`） | 10 req/min | user_id |

#### レスポンスタイムテスト

| 項目 | 内容 |
|------|------|
| テストレベル | 統合テスト（軽量スモーク） |
| 実施タイミング | ローカルで実行（E2E テストと同タイミング） |
| テスト方針 | 主要エンドポイントに対して p95 レスポンスタイムが閾値以内であることを確認する |
| テストケースの記載先 | `cross-cutting.md` |

**期待値**（`10_requirements/requirements.md` § 4.1 より）:

| 指標 | 目標値 |
|------|--------|
| API レスポンスタイム（p95） | 500ms 以下（一覧取得を含む） |
| ファイルアップロード（5MB） | 5 秒以下 |

**備考**: MVP では本格的な負荷テストは行わない。ローカルでの軽量スモークテストで閾値超過を検知するレベルとする。

**MVP スコープ外の非機能テスト**:

| テスト | 対応要件 | MVP で行わない理由 | 想定されるリスク |
|--------|---------|-------------------|----------------|
| 同時接続100ユーザーの負荷テスト | NFR-PERF-003 | MVP のインフラ構成（EC2 t3.micro 1台 + RDS db.t3.micro）では接続数上限が約87であり、100ユーザー同時接続にはコネクションプーリング（PgBouncer 等）またはインスタンスサイズの引き上げが必要。MVP ではコスト優先 | コネクションプール枯渇、RLS の set_config が高負荷時に競合する可能性 |
| 稼働率 99.5% の計測 | NFR-AVAIL-001 | 稼働率は本番環境での長期運用でのみ計測可能。テスト環境では検証不可 | Single-AZ 構成のため、AZ 障害時にダウンタイムが発生（ADR-0004 でリスク受容済み） |
| レートリミッターの高負荷テスト | security.md §4.1 | 単体での動作確認は統合テストで実施するが、高負荷時のメモリ消費・性能影響は未検証 | インメモリ実装の場合、大量の IP/ユーザーキーでメモリ圧迫の可能性 |

---

## 4. テストデータ定義（フィクスチャ）

全テストケースファイルから共通参照される標準フィクスチャを定義する。テナント分離を意識し、テスト間でテナントIDが混在しないよう設計する。

### 4.1 テナント構成

| 役割 | テナント名 | テナントID |
|------|----------|-----------|
| **テナントA**（メインテスト用） | Test Company A | `aaaaaaaa-0001-0001-0001-000000000001` |
| **テナントB**（クロステナント検証用） | Test Company B | `bbbbbbbb-0002-0002-0002-000000000002` |

### 4.2 ユーザー構成

#### テナントAのユーザー（4ロール）

| ロール | ユーザー名 | メールアドレス | ユーザーID | テスト用パスワード |
|--------|----------|--------------|-----------|-----------------|
| Admin | Test Admin | `test-admin@example.com` | `aaaaaaaa-1111-1111-1111-000000000001` | `TestPass1!` |
| Approver | Test Approver | `test-approver@example.com` | `aaaaaaaa-2222-2222-2222-000000000002` | `TestPass1!` |
| Member | Test Member | `test-member@example.com` | `aaaaaaaa-3333-3333-3333-000000000003` | `TestPass1!` |
| Accounting | Test Accounting | `test-accounting@example.com` | `aaaaaaaa-4444-4444-4444-000000000004` | `TestPass1!` |

#### テナントBのユーザー（クロステナント検証用）

| ロール | ユーザー名 | メールアドレス | ユーザーID | テスト用パスワード |
|--------|----------|--------------|-----------|-----------------|
| Member | Test Member B | `test-member-b@example.com` | `bbbbbbbb-3333-3333-3333-000000000003` | `TestPass1!` |

**注意**: テストデータには本物のパスワードを含めない。`TestPass1!` はテスト専用の仮パスワードである。

### 4.3 経費レポートのフィクスチャ（テナントA所属、作成者: Test Member）

合計 8 件。issue-087 対応で月別ダッシュボード集計を観察可能にするため、paid を直近 3 ヶ月に分散。

| フィクスチャ名 | 状態 | レポートID | タイトル | 補足 |
|-------------|------|----------|---------|------|
| `report_draft` | `draft` | `cccccccc-0001-0001-0001-000000000001` | テスト下書きレポート | 明細1件あり |
| `report_draft_empty` | `draft` | `cccccccc-0001-0001-0001-000000000002` | テスト下書き（明細なし） | 提出不可テスト用 |
| `report_submitted` | `submitted` | `cccccccc-0002-0002-0002-000000000002` | テスト提出済みレポート | 承認・却下テスト用 |
| `report_approved` | `approved` | `cccccccc-0003-0003-0003-000000000003` | テスト承認済みレポート | 支払完了テスト用 |
| `report_rejected` | `rejected` | `cccccccc-0004-0004-0004-000000000004` | テスト却下レポート | 再申請テスト用 |
| `report_paid`（前月） | `paid` | `cccccccc-0005-0005-0005-000000000005` | テスト支払済みレポート（前月） | 終端状態テスト用・period は直近3ヶ月の前月 |
| `report_paid_prev2`（前々月） | `paid` | `dddddddd-0002-0002-0002-000000000001` | テスト支払済みレポート（前々月） | issue-087: 月別集計分散用 |
| `report_paid_cur`（当月） | `paid` | `dddddddd-0003-0003-0003-000000000001` | テスト支払済みレポート（当月） | issue-087: 月別集計分散用 |

`paid` レポート 3 件の `period_start`/`period_end` は `seed.Run()` 呼び出し時の `time.Now()` を基準に動的生成される（当月・前月・前々月の月初〜月末）。固定日付ではないため、時間経過で集計範囲が変わらない。

**report_draft の明細**:

| 明細ID | 金額 | カテゴリ | 支出日 |
|--------|------|---------|--------|
| `dddddddd-0001-0001-0001-000000000001` | 1000 | `transportation` | 2026-03-01 |

### 4.4 カテゴリのフィクスチャ

カテゴリは `categories` テーブルのグローバルマスタ（`tenant_id = NULL`）を使用する。

| カテゴリコード | 日本語名 |
|-------------|--------|
| `transportation` | 交通費 |
| `accommodation` | 宿泊費 |
| `food` | 飲食費 |
| `supplies` | 消耗品費 |
| `communication` | 通信費 |
| `other` | その他 |

### 4.5 テナントBのレポートフィクスチャ（クロステナント検証用）

| フィクスチャ名 | 状態 | レポートID | テナント | 用途 |
|-------------|------|----------|--------|------|
| `report_tenant_b_draft` | `draft` | `eeeeeeee-0001-0001-0001-000000000001` | テナントB | テナント分離テスト用（読み取り操作） |
| `report_tenant_b_submitted` | `submitted` | `eeeeeeee-0002-0002-0002-000000000002` | テナントB | テナント分離テスト用（承認操作） |
| `report_tenant_b_approved` | `approved` | `eeeeeeee-0003-0003-0003-000000000003` | テナントB | テナント分離テスト用（支払操作） |

**用途**: テナントAのユーザーがテナントBのレポートにアクセスしようとしたとき、404 が返ることを検証する。

---

## 5. CI 組み込み方針

### 5.1 テスト実行方針（概要）

| タイミング | 実行内容 | 目的 |
|----------|---------|------|
| **PR 作成・更新時** | 単体テスト + 統合テスト（レート制限テストを含む）をローカルで実行 | 変更の安全性を素早く確認 |
| **main ブランチへのマージ前** | E2E テスト + レスポンスタイム スモークテストをローカルで実行 | 主要フローの回帰保証 + 非機能閾値の監視 |

**補足**: ローカルテスト実行結果は PR body のローカルテスト結果セクションに記載し、手動確認で担保する。

### 5.2 テスト実行コマンド（ローカル開発時）

```bash
# 単体テスト（ドメイン層のみ）
go test ./internal/domain/... -v -cover

# 統合テスト（テスト用DBが必要）
go test ./... -v -tags=integration

# フロントエンドユニットテスト
npm test  # または npx vitest

# E2Eテスト
npx playwright test
```

---

## 6. テストID命名規則

### 6.1 プレフィックス定義

| プレフィックス | 対象リソース | 対応エンドポイント群 |
|-------------|------------|------------------|
| `AUTH-` | 認証 | `/api/auth/*` |
| `RPT-` | 経費レポート | `/api/reports/*` |
| `ITM-` | 経費明細 | `/api/reports/{id}/items/*` |
| `ATT-` | 添付ファイル | `/api/reports/{id}/items/{itemId}/attachments/*` |
| `WFL-` | ワークフロー | `/api/workflow/*` |
| `DSH-` | ダッシュボード・カテゴリ | `/api/dashboard`, `/api/categories` |
| `TNT-` | テナント管理 | `/api/tenant/*` |
| `CRS-` | 横断（テナント分離・RBAC・E2E） | 複数エンドポイントにまたがるテスト |

### 6.2 連番形式

```
{PREFIX}-{NNN}
```

例: `AUTH-001`, `RPT-012`, `CRS-001`

- 各プレフィックス内で `001` から連番
- ファイル内で連番を通し番号とする（同ファイル内で重複させない）

---

## 7. テストケースファイルの構成と責務境界

### 7.1 ファイル構成

| ファイル名 | プレフィックス | 担当範囲 |
|----------|-------------|---------|
| `auth.md` | `AUTH-` | 認証エンドポイント（signup, login, refresh, logout, me, password-reset） |
| `reports.md` | `RPT-` | 経費レポート CRUD、状態遷移テスト |
| `items.md` | `ITM-` | 経費明細 CRUD |
| `attachments.md` | `ATT-` | 添付ファイル一覧取得・アップロード・ダウンロード・削除、署名付きURL認可（エンドポイント4件: uploadAttachment, listAttachments, getAttachmentDownload, deleteAttachment） |
| `workflow.md` | `WFL-` | 承認・却下・支払完了 |
| `dashboard.md` | `DSH-` | ダッシュボード・カテゴリ |
| `tenant.md` | `TNT-` | テナント情報取得・メンバー一覧 |
| `cross-cutting.md` | `CRS-` | テナント分離、RBACマトリクス、E2Eシナリオ |

### 7.2 責務境界ルール

テスト設計者・実装者が迷わないよう、以下の基準で記載先を決定する。

| テスト観点 | 記載先 | 理由 |
|-----------|--------|------|
| テナント分離テスト（全リソース × テナント越境 = 404） | `cross-cutting.md` に集約 | 機能別に分散すると網羅性が見えない |
| RBACテスト: 各エンドポイントの権限別レスポンス（200/403） | 各機能別ファイル | 各エンドポイント実装時に書くテスト |
| RBACマトリクス: 全エンドポイント × 全ロールの整合性確認 | `cross-cutting.md` | 横断レビューで使う |
| 状態遷移テスト（許可遷移 T1〜T5、禁止遷移 X1〜X10） | `reports.md` / `workflow.md` | ドメインロジックとして完結（振り分け詳細は下記注記を参照） |
| E2E シナリオ（申請→承認→支払完了 等） | `cross-cutting.md` | 複数機能をまたぐ |
| ドメイン不変条件テスト（バリデーション等） | 各機能別ファイル | 機能の一部として実装 |
| 添付ファイル署名付きURL認可チェック | `attachments.md` | 添付機能の一部として完結 |

#### 状態遷移テストの振り分け詳細

状態遷移テストは「ドメイン層単体テスト」と「ハンドラ層統合テスト」で記載先が異なる。

**ドメイン層単体テスト（`domain/report_test.go`）**:
T1〜T5 および X1〜X10 の全遷移を `reports.md` に記載する。ドメイン層のメソッド（`Submit()`、`Approve()` 等）を直接呼び出し、状態変化とエラーを検証する。

**ハンドラ層統合テストのファイル振り分け**:

| ファイル | 担当する遷移 | 理由 |
|---------|-----------|------|
| `reports.md` | T1（draft→submitted）、T5（draft→deleted）、X1〜X4（draft/submitted 起点の禁止遷移） | レポート所有者が実行する操作（submit, delete） |
| `workflow.md` | T2（submitted→approved）、T3（submitted→rejected）、T4（approved→paid）、X5〜X10（submitted/approved/rejected/paid 起点の禁止遷移） | Approver/Accounting が実行する操作 |

---

## 8. テストケース列定義

全テストケースファイルで統一する列定義。

| 列名 | 説明 | 記入例 |
|------|------|--------|
| テストID | プレフィックス+連番 | `AUTH-001` |
| テストレベル | 単体 / 統合 / E2E | `統合` |
| レイヤー | domain / repository / handler（Go のどのテストファイルに書くかの指標） | `handler` |
| 保証種別 | そのテストが保証する種別（正常系 / 異常系 / 境界値 / セキュリティ / 性能 / 認可 / 状態遷移 / テナント分離） | `正常系` |
| 対応要件ID | テストケースが保証する要件の ID（`requirements.md` または `policies.md` の ID） | `AUTH-F01, SEC-002` |
| 対応設計ID | テストケースが対象とする設計成果物の参照 | `openapi.yaml#signup, db_schema.md#users` |
| テスト関数名候補 | Go または Playwright の関数名候補 | `TestSignup_DuplicateEmail` |
| 入力（前提条件含む） | テストの入力値と前提状態 | `メール: 既存ユーザーと同一` |
| 期待結果 | HTTPステータス、エラーコード、DB状態変化等 | `409 EMAIL_ALREADY_EXISTS` |

---

## 9. テスト環境

### 9.1 テスト用DB

| 項目 | 設定 |
|------|------|
| DBMS | PostgreSQL（本番と同一バージョン） |
| 起動方法 | Docker Compose（`docker compose up db-test`） |
| 接続先 | `postgresql://testuser:testpass@localhost:5433/expense_test` |
| マイグレーション | テスト実行前に自動適用（`go test` のセットアップ関数内） |
| RLS有効 | **あり**（本番と同一設定） |

### 9.2 RLS の検証方法

RLS（Row Level Security）はテナント分離の DB 層保護である。テストでは以下の二つの観点で検証する。

#### アプリケーション経由の正常動作確認

統合テストでは、テスト用DBに対して実際の API ハンドラを経由してリクエストを実行する。このとき、`TenantContext` ミドルウェアが `SET LOCAL app.current_tenant = '{tenant_id}'` を実行し、RLS が自動的に適用される。

```sql
-- TenantContext ミドルウェアが実行するSQL（テスト時も同様）
SET LOCAL app.current_tenant = 'aaaaaaaa-0001-0001-0001-000000000001';
```

#### RLS 単体の動作確認（リポジトリ層テスト）

リポジトリ層のテストでは、テスト DB に直接接続し `SET LOCAL` の有無でアクセス制御を確認する。

```go
// テスト例（擬似コード）
// テナントコンテキストなし → RLS が全行をブロック
db.Exec("SET LOCAL app.current_tenant = ''")
rows := repo.ListReports(ctx)  // → 0件

// テナントコンテキストあり → 該当テナントの行のみ返る
db.Exec("SET LOCAL app.current_tenant = 'aaaaaaaa-...'")
rows := repo.ListReports(ctx)  // → テナントAの行のみ
```

### 9.3 テストデータのクリーンアップ戦略

| 戦略 | 採用場面 | 方法 |
|------|---------|------|
| **トランザクションロールバック** | 単一テスト関数内での完結するテスト | テスト終了時に `tx.Rollback()` |
| **テスト前 TRUNCATE** | フィクスチャを使い回す統合テスト | `TestMain` の `setup()` で全テーブルを TRUNCATE し、フィクスチャを再投入 |

**推奨方針**: テスト間の独立性を確保するため、**テスト前 TRUNCATE + フィクスチャ再投入**を統合テストの基本戦略とする。ただし、テスト内でのみ作成・削除するリソースはトランザクションロールバックで対応してもよい。

### 9.4 ローカル開発環境のS3互換サービス

添付ファイルテストには MinIO を使用する。

| 項目 | 設定 |
|------|------|
| サービス | MinIO（Docker Compose） |
| エンドポイント | `http://localhost:9000` |
| バケット名 | `expense-saas-attachments-local` |
| 認証情報 | 環境変数（`MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD`）で設定 |

---

## 10. Playwright E2E テスト方針

### 10.1 基本設定

| 項目 | 設定 |
|------|------|
| ブラウザ | Chromium のみ（MVP） |
| テストランナー | Playwright Test |
| ファイル配置 | `expense-saas/e2e/` |
| テスト環境 | E2E 専用テスト環境（フロントエンド + バックエンド + DB + MinIO を Docker Compose で起動） |

### 10.2 対象フロー

MVP で E2E テストの対象とする主要フローは以下の2フローに限定する。

#### フロー1: 申請→承認→支払完了

```
1. Test Member がログイン
2. 経費レポートを新規作成（タイトル・期間を入力）
3. 経費明細を追加（金額・カテゴリ・摘要を入力）
4. レポートを提出
5. Test Approver がログイン
6. 承認待ちレポート一覧でレポートを確認
7. レポートを承認
8. Test Accounting がログイン
9. 支払待ちレポート一覧でレポートを確認
10. 支払完了を記録
```

#### フロー2: 却下→再申請

```
1. Test Member がログイン
2. 経費レポートを新規作成・明細追加・提出
3. Test Approver がログイン
4. レポートを却下（却下理由を入力）
5. Test Member がログイン
6. 却下されたレポートを確認
7. 却下されたレポートから再申請（新規 draft レポートが作成される）
8. 再申請レポートの内容を確認（元レポートの明細がコピーされている）
```

### 10.3 テストアカウント

フィクスチャ定義（4.2節）のユーザーを使用する。

| フロー | 使用ユーザー |
|--------|-----------|
| フロー1（申請者） | `test-member@example.com` |
| フロー1（承認者） | `test-approver@example.com` |
| フロー1（経理） | `test-accounting@example.com` |
| フロー2（申請者） | `test-member@example.com` |
| フロー2（承認者） | `test-approver@example.com` |

---

## 11. 必須テスト領域

以下の領域は下流のテストケース定義（6-B-1〜8）で必ずテストケースを定義すること。

### 11.1 テナント分離（`cross-cutting.md` に集約）

全リソースに対してテナント越境アクセスが 404 を返すことを検証する。

| 検証対象リソース | テスト内容 |
|--------------|---------|
| 経費レポート（GET, PUT, DELETE, submit） | テナントBのレポートIDでテナントAのユーザーがアクセス → 404 |
| 経費明細（POST, PUT, DELETE） | テナントBのレポート配下の明細 → 404 |
| 添付ファイル（POST, GET, DELETE） | テナントBの添付 → 404 |
| ワークフロー（approve, reject, pay） | テナントBのレポートへの操作 → 404 |
| テナントメンバー（GET） | テナントBのメンバー一覧をテナントAのユーザーが取得しようとしても、テナントAのメンバーのみ返る（RLS による分離） |

### 11.2 RBAC（各機能別ファイル）

各エンドポイントに対して、許可ロール・禁止ロールの両方を検証する。
`cross-cutting.md` に全エンドポイント × 全ロールのマトリクスを定義し、整合性を確認する。

重点検証対象:

| エンドポイント | 禁止ロール | 期待レスポンス |
|-------------|----------|-------------|
| `GET /api/workflow/pending` | Member, Accounting, Admin | 403 FORBIDDEN |
| `POST /api/workflow/{id}/approve` | Member, Accounting, Admin | 403 FORBIDDEN |
| `POST /api/workflow/{id}/reject` | Member, Accounting, Admin | 403 FORBIDDEN |
| `GET /api/workflow/payable` | Member, Approver, Admin | 403 FORBIDDEN |
| `POST /api/workflow/{id}/pay` | Member, Approver, Admin | 403 FORBIDDEN |
| `GET /api/reports/all` | Member, Approver | 403 FORBIDDEN |
| `GET /api/tenant` | Approver, Member, Accounting | 403 FORBIDDEN |
| `GET /api/tenant/members`（listTenantMembers） | Member, Approver | 403 FORBIDDEN（許可: Admin, Accounting — authz.md §6.7） |

### 11.3 状態遷移（`reports.md` / `workflow.md`）

#### 許可遷移（T1〜T5）

| 遷移ID | 遷移 | 対応エンドポイント |
|--------|------|-----------------|
| T1 | draft → submitted | `POST /api/reports/{id}/submit` |
| T2 | submitted → approved | `POST /api/workflow/{id}/approve` |
| T3 | submitted → rejected | `POST /api/workflow/{id}/reject` |
| T4 | approved → paid | `POST /api/workflow/{id}/pay` |
| T5 | draft → 削除 | `DELETE /api/reports/{id}` |

**T1 の追加テスト観点（事前条件違反）**:

| 条件違反 | エラーコード | ルールID |
|---------|-----------|---------|
| 明細が0件で提出 | `422 EMPTY_REPORT_SUBMISSION` | RPT-014 |
| 同一テナントに Approver が1人も存在しない | `422 NO_APPROVER_IN_TENANT` | WFL-014 |

#### 禁止遷移（X1〜X10）

全禁止遷移が `422 INVALID_STATE_TRANSITION`（または対応するエラーコード）を返すことを検証する。

| 遷移ID | 遷移 | エラーコード |
|--------|------|-----------|
| X1 | draft → approved（スキップ） | `INVALID_STATE_TRANSITION` |
| X2 | draft → rejected | `INVALID_STATE_TRANSITION` |
| X3 | draft → paid | `INVALID_STATE_TRANSITION` |
| X4 | submitted → draft（取消不可） | `INVALID_STATE_TRANSITION` |
| X5 | submitted → paid（スキップ） | `INVALID_STATE_TRANSITION` |
| X6 | approved → draft | `INVALID_STATE_TRANSITION` |
| X7 | approved → submitted | `INVALID_STATE_TRANSITION` |
| X8 | approved → rejected | `INVALID_STATE_TRANSITION` |
| X9 | rejected → 任意 | `INVALID_STATE_TRANSITION` |
| X10 | paid → 任意 | `INVALID_STATE_TRANSITION` |

### 11.4 ドメイン不変条件（各機能別ファイル）

`domain_model.md` で定義された不変条件を全て検証する。

| 不変条件 | ルールID | テスト観点 |
|---------|---------|----------|
| レポートタイトルは1〜200文字 | RPT-001 | 空文字・201文字で422 |
| period_end ≧ period_start | RPT-003 | 逆順日付で422 |
| 明細金額 > 0 | ITM-002 | 0円・負数で422 |
| 明細摘要は1〜500文字 | ITM-004 | 空文字・501文字で422 |
| 添付ファイルサイズ ≦ 5MB | ATT-003 | 5MB超で413 |
| 添付MIMEタイプは許可リスト内 | ATT-002 | 非許可タイプで422 |
| 提出時の明細 ≧ 1件 | RPT-014 | 明細なし提出で422 |
| 却下理由は1〜1000文字 | WFL-012 | 空文字で422 |
| 再申請時、元レポート（rejected）の状態は変化しない | RPT-015 | 再申請後も元レポートのstatusがrejectedのままであることを確認 |
| 再申請レポートの reference_report_id に元レポートIDが設定される | RPT-016 | 再申請レポートのreference_report_idが元レポートIDと一致することを確認 |
| 再申請時、元レポートの添付ファイルはコピーされない | （state_machine.md §6.2） | 再申請レポートの明細にはattachmentsが存在しないことを確認 |

### 11.5 添付ファイルURL認可（`attachments.md`）

署名付きURL発行前の認可チェックを検証する（`files.md` 4.5節）。

| テスト内容 | 期待結果 |
|----------|---------|
| 認証なしでダウンロードURL取得 | 401 UNAUTHORIZED |
| 他テナントの添付にダウンロードURL取得を試みる | 404 RESOURCE_NOT_FOUND |
| 他者の（自分のものでない）添付にMember がダウンロードURL取得を試みる | 403 FORBIDDEN |
| 認可チェック通過後にダウンロードURLが返る | 200 + signed URL |

### 11.6 ビジネスルールに基づく認可（各機能別ファイル）

| ルールID | 内容 | テスト観点 |
|---------|------|----------|
| RBC-016 | 自己承認禁止 | Approver が自分のレポートを承認 → 403 SELF_APPROVAL_NOT_ALLOWED |
| RBC-016 | 自己却下禁止 | Approver が自分のレポートを却下 → 403 SELF_APPROVAL_NOT_ALLOWED |
| RBC-012 | 自己支払処理禁止 | Accounting が自分のレポートの支払完了を記録 → 403 SELF_PAYMENT_NOT_ALLOWED |
| RBC-014 | Admin は他者のレポートを編集不可 | Admin が他者の draft レポートを編集 → 403 FORBIDDEN |

---

## 12. テスト実装ガイド（概要）

本節は Step 9 の実装者向けの補足ガイドである。詳細な実装手順は Step 9 で定義する。

### 12.1 Go テストの書き方

```go
// テスト関数の命名規則
// TestXxx_{正常ケース説明} または TestXxx_{異常ケース説明}
func TestSubmitReport_Success(t *testing.T) { ... }
func TestSubmitReport_EmptyItems(t *testing.T) { ... }
func TestSubmitReport_InvalidState(t *testing.T) { ... }

// テーブル駆動テスト（複数ケースをまとめる場合）
func TestCreateReport_Validation(t *testing.T) {
    tests := []struct {
        name    string
        input   CreateReportInput
        wantErr string
    }{
        {"空タイトル", CreateReportInput{Title: ""}, "VALIDATION_ERROR"},
        {"201文字タイトル", CreateReportInput{Title: strings.Repeat("a", 201)}, "VALIDATION_ERROR"},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) { ... })
    }
}
```

### 12.2 統合テストの構造

```go
// 統合テストのビルドタグ
//go:build integration

func TestCreateReport_Handler(t *testing.T) {
    // 1. テスト用DBのセットアップ（フィクスチャ投入）
    db := testutil.SetupTestDB(t)

    // 2. テスト対象のハンドラ組み立て
    handler := NewTestRouter(db)

    // 3. テスト用JWTトークン生成
    token := testutil.GenerateTestToken(t, testutil.UserMember)

    // 4. リクエスト送信
    req := httptest.NewRequest("POST", "/api/reports", body)
    req.Header.Set("Authorization", "Bearer "+token)
    rec := httptest.NewRecorder()
    handler.ServeHTTP(rec, req)

    // 5. レスポンス検証
    assert.Equal(t, http.StatusCreated, rec.Code)
}
```

### 12.3 フロントエンドテスト（Vitest）

```typescript
// Vitest によるコンポーネントテスト
import { render, screen } from '@testing-library/react'
import { describe, it, expect } from 'vitest'

describe('ReportForm', () => {
  it('タイトルが空のとき送信ボタンを無効にする', () => {
    render(<ReportForm />)
    expect(screen.getByRole('button', { name: '保存' })).toBeDisabled()
  })
})
```

### 12.4 Playwright E2E テストの書き方

```typescript
// Playwright テスト例
import { test, expect } from '@playwright/test'
import { loginAs } from './helpers/auth'
import { fixtures } from './fixtures'

test('申請→承認→支払完了フロー', async ({ page, browser }) => {
  // Member でログイン
  await loginAs(page, fixtures.users.member)

  // レポート作成
  await page.goto('/reports/new')
  await page.fill('[name=title]', 'E2Eテストレポート')
  // ...
})
```

---

## 13. MVPスコープ外の除外項目

以下はMVPスコープ外のため、テストケースに含めない。

| 除外項目 | 理由 |
|---------|------|
| メンバー招待・ロール変更 | Phase 3 機能 |
| CSVエクスポート | Phase 3 機能 |
| 監査ログ | Phase 3 機能 |
| 通知（メール送信） | MVP スコープ外 |
| 多言語対応 | MVP スコープ外 |
| 複数テナント所属 | MVP では1ユーザー=1テナント（RBC-002） |

---

## 改訂履歴

| 版 | 日付 | 変更内容 |
|----|------|---------|
| 1.0 | 2026-03-23 | 初版作成 |
