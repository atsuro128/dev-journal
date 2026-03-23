# Step 8: 機能実装（TDD） — 作業分解

## 概要

テストケースファイル単位で TDD（テスト → BE 実装 → FE 実装）により各機能を実装する。認証を起点に依存関係に従って並列開発を進める。

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
| テスト戦略 | `deliverables/docs/60_test/test_strategy.md` |
| テストケース（認証） | `deliverables/docs/60_test/test_cases/auth.md` |
| テストケース（レポート） | `deliverables/docs/60_test/test_cases/reports.md` |
| テストケース（明細） | `deliverables/docs/60_test/test_cases/items.md` |
| テストケース（添付） | `deliverables/docs/60_test/test_cases/attachments.md` |
| テストケース（ワークフロー） | `deliverables/docs/60_test/test_cases/workflow.md` |
| テストケース（ダッシュボード） | `deliverables/docs/60_test/test_cases/dashboard.md` |
| テストケース（テナント管理） | `deliverables/docs/60_test/test_cases/tenant.md` |

### 作業計画

Step 着手時にまず作業計画を立案し、以下に保存する:

- テンプレート: `guide/templates/task-plan.md`
- 配置先: `progress-management/task-plans/step8-feature-implementation.md`
- 計画が確定してから成果物作成に入ること

### 完了条件（Step 全体）

- 8-A〜8-G の全テストケースが実装され CI で通る
- 申請 → 承認 → 支払の一連フローが API + UI で通る

---

## レビュー観点

成果物内の「品質チェック」は作成者のセルフチェック。以下は reviewer が検証する観点。

> Step 8 は未着手のため、以下は現時点で定義できる観点。着手時に実装スコープを踏まえて精緻化すること。具体的なコードレビュー基準（コーディング規約、エラーハンドリング方針等）は着手時に定義する。

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
- `test_cases/*.md` の全テストケースがコードとして実装されているか
- テスト結果が CI で自動検証されているか

### 機能実装固有

#### 4. 機能
- OpenAPI 定義どおりの API が実装されているか
- 画面仕様（`screens/*.md`）どおりの UI が実装されているか
- 不正な状態遷移で 422 を返すか
- 権限外ロールで 403 を返すか
- テナント境界越えで 404 を返すか

---

## 成果物ファイルの扱い

| 成果物 | 出力先 | 作成タスク | 追記タスク |
|--------|--------|-----------|-----------|
| バックエンドコード（機能） | `expense-saas/apps/api/` | 各機能タスク | — |
| フロントエンドコード（機能） | `expense-saas/apps/web/` | 各機能タスク | — |
| テストコード | `expense-saas/` 内各所 | 各機能タスク（TDD: テスト先行） | — |

---

## タスク一覧

各機能タスクはテストケースファイル単位。TDD でテスト → BE 実装 → FE 実装を1タスク内で完結させる。

| ID | タスク | テストケース | 依存 | 状態 |
|----|--------|-------------|------|------|
| 8-A | 認証 | auth.md | Step 7 完了（7-B） | 未着手 |
| 8-B | レポート | reports.md | 8-A | 未着手 |
| 8-C | ダッシュボード | dashboard.md | 8-A | 未着手 |
| 8-D | テナント管理 | tenant.md | 8-A | 未着手 |
| 8-E | 明細 | items.md | 8-B | 未着手 |
| 8-F | ワークフロー | workflow.md | 8-B | 未着手 |
| 8-G | 添付ファイル | attachments.md | 8-E | 未着手 |

### 依存グラフ

```
Step 7 完了（7-B）
  └→ 8-A (認証) ─┬→ 8-B (レポート) ─┬→ 8-E (明細) → 8-G (添付)
                  ├→ 8-C (ダッシュボード)  └→ 8-F (ワークフロー)
                  └→ 8-D (テナント管理)
                                              ↓
                                        Step 9 (システムテスト・UAT)
```

**最大並列数**: 認証完了後に3並列（レポート・ダッシュボード・テナント管理）、レポート完了後に2並列（明細・ワークフロー）

---

## タスク詳細

### 8-A: 認証

- **テストケース**: test_cases/auth.md（80件）
- **入力**: openapi.yaml §認証, db_schema.md, authz.md, security.md, screens/auth-*.md
- **出力**: `apps/api/internal/` §認証, `apps/web/src/features/auth/`, テストコード
- **TDD フロー**:
  1. テスト: Argon2id ハッシュ/検証、JWT 生成/検証、各認証エンドポイント、無効トークンで 401
  2. BE 実装: POST /auth/signup, login, refresh, logout, GET /auth/me、パスワードハッシュ（Argon2id）、JWT 発行（RS256）
  3. FE 実装: ログイン・サインアップ・パスワードリセット画面、JWT 管理、ルーティングガード
- **完了条件**:
  - test_cases/auth.md の全テストケースが実装され通過している
  - signup → login → refresh → logout の一連フローが API + UI で通る

### 8-B: レポート

- **テストケース**: test_cases/reports.md（90件）
- **入力**: openapi.yaml §レポート, db_schema.md, authz.md, state_machine.md, screens/expenses.md §レポート
- **出力**: `apps/api/internal/` §レポート, `apps/web/src/features/expenses/` §レポート, テストコード
- **TDD フロー**:
  1. テスト: レポート CRUD、状態遷移（全パターン）、テナント分離、RBAC
  2. BE 実装: GET/POST/PUT/DELETE /reports、状態遷移ロジック
  3. FE 実装: レポート一覧・作成/編集・詳細画面
- **完了条件**:
  - test_cases/reports.md の全テストケースが実装され通過している
  - 不正な状態遷移で 422 を返す

### 8-C: ダッシュボード

- **テストケース**: test_cases/dashboard.md（25件）
- **入力**: openapi.yaml §ダッシュボード, screens/expenses.md §ダッシュボード
- **出力**: `apps/api/internal/` §ダッシュボード, `apps/web/src/features/expenses/` §ダッシュボード, テストコード
- **TDD フロー**:
  1. テスト: 集計値の正確性、ロール別表示、テナント分離
  2. BE 実装: GET /dashboard
  3. FE 実装: ダッシュボード画面
- **完了条件**:
  - test_cases/dashboard.md の全テストケースが実装され通過している

### 8-D: テナント管理

- **テストケース**: test_cases/tenant.md（11件）
- **入力**: openapi.yaml §テナント, authz.md, screens/expenses.md §テナント管理
- **出力**: `apps/api/internal/` §テナント, `apps/web/src/features/admin/`, テストコード
- **TDD フロー**:
  1. テスト: メンバー一覧、Admin 専用アクセス、テナント分離
  2. BE 実装: GET /tenant, GET /tenant/members
  3. FE 実装: テナント管理画面（Admin 専用）
- **完了条件**:
  - test_cases/tenant.md の全テストケースが実装され通過している

### 8-E: 明細

- **テストケース**: test_cases/items.md（75件）
- **入力**: openapi.yaml §明細, db_schema.md, screens/expenses.md §明細
- **出力**: `apps/api/internal/` §明細, `apps/web/src/features/expenses/` §明細, テストコード
- **TDD フロー**:
  1. テスト: 明細 CRUD、カテゴリ選択、金額バリデーション、レポート連動削除
  2. BE 実装: GET/POST/PUT/DELETE /reports/:id/items
  3. FE 実装: 明細追加/編集画面
- **完了条件**:
  - test_cases/items.md の全テストケースが実装され通過している

### 8-F: ワークフロー

- **テストケース**: test_cases/workflow.md（62件）
- **入力**: openapi.yaml §ワークフロー, authz.md, state_machine.md, screens/approvals.md
- **出力**: `apps/api/internal/` §ワークフロー, `apps/web/src/features/approvals/`, テストコード
- **TDD フロー**:
  1. テスト: 承認/却下/差戻し/支払完了の全パターン、自己承認禁止、RBAC
  2. BE 実装: POST /reports/:id/submit, approve, reject, mark-paid、GET /workflow/pending, /workflow/payable
  3. FE 実装: 承認キュー画面、承認/却下アクション、却下理由入力
- **完了条件**:
  - test_cases/workflow.md の全テストケースが実装され通過している
  - 権限外ロールで 403、不正遷移で 422 を返す

### 8-G: 添付ファイル

- **テストケース**: test_cases/attachments.md（54件）
- **入力**: openapi.yaml §添付, files.md, db_schema.md, authz.md, screens/attachments.md
- **出力**: `apps/api/internal/` §添付, `apps/web/src/features/expenses/` §添付, テストコード
- **TDD フロー**:
  1. テスト: MIME/サイズ制限、署名付き URL、認可、テナント分離
  2. BE 実装: 署名付き URL 発行 API、S3 クライアント、バリデーション
  3. FE 実装: 添付エリア（アップロード・プレビュー・削除）
- **完了条件**:
  - test_cases/attachments.md の全テストケースが実装され通過している
