# Step 9: 機能実装 — 作業分解

## 概要

Step 8 で実装した失敗するテストを全て通す。テストケースファイル単位で BE 実装 → FE 実装を進める。

## 前提

### 上流成果物（入力）

| 成果物 | パス |
|--------|------|
| アーキテクチャ設計 | `deliverables/docs/30_architecture/architecture.md` |
| 画面一覧 | `deliverables/docs/40_basic_design/screens.md` |
| 画面遷移図 | `deliverables/docs/40_basic_design/ui_flow.md` |
| 画面仕様（機能別） | `deliverables/docs/50_detail_design/screens/*.md` |
| DB スキーマ | `deliverables/docs/50_detail_design/db_schema.md` |
| OpenAPI | `deliverables/docs/50_detail_design/openapi.yaml` |
| 認可設計 | `deliverables/docs/50_detail_design/authz.md` |
| 状態遷移設計 | `deliverables/docs/20_domain/state_machine.md` |
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
| Step 8 のテストコード | `expense-saas/apps/` |

### 作業計画

Step 着手時にまず作業計画を立案し、以下に保存する:

- テンプレート: `guide/templates/task-plan.md`
- 配置先: `progress-management/task-plans/step9-feature-implementation.md`
- 計画が確定してから成果物作成に入ること

### 計画レビューゲート

作業計画は**成果物作成前のレビュー対象**とする。少なくとも以下を確認し、通過するまで成果物作成に入ってはならない。

- 判断ポイントに未確定のまま残してよい項目と、着手前に確定が必要な項目が切り分けられているか
- 共通前提・着手条件・担当境界・禁止事項・参照先が明記されているか
- レビュー単位、マージ順、共有ファイル調整、トレーサビリティ更新対象が明記されているか
- この Step の完了条件と、下流 Step の着手条件が task plan に落ちているか
- 作業計画レビューで出た blocking 指摘への対応方針が整理されているか


### 完了条件（Step 全体）

- 9-A〜9-G の全テストケースが通過している
- 申請 → 承認 → 支払の一連フローが API + UI で通る

### 作業ルール（並列実装向け）

- 各機能タスクは `1 テストケースファイル = 1 実装担当` を原則とする
- 進行順は `BE 実装 -> BE レビュー完了 -> FE 実装 -> FE レビュー完了 -> 統合確認` とし、未レビューのまま次段階へ進めない
- 各担当は、対応する Step 8 のテストを通すことをタスク完了条件に含める
- 共通基盤（ミドルウェア、共通エラー処理、OpenAPI 生成物、共通 UI 基盤）の変更は、機能タスク内で独断変更せず Lead 判断で扱う
- レビュー依頼時には、変更した API、変更した画面、追加した共通部品、他タスクへの影響を明記する
- マージ順は依存グラフに従う。依存先タスクの変更を取り込んだ場合、対象テストを再実行する

---

## レビュー観点

成果物内の「品質チェック」は作成者のセルフチェック。以下は reviewer が検証する観点。

> Step 9 は未着手のため、以下は現時点で定義できる観点。着手時に実装スコープを踏まえて精緻化すること。具体的なコードレビュー基準（コーディング規約、エラーハンドリング方針等）は着手時に定義する。

### 全タスク共通

#### 1. 設計準拠性
- 実装が上流設計（OpenAPI, db_schema, authz, security, files, screens/*.md）の仕様どおりであるか
- テナント分離の二重保証（アプリ層 WHERE + RLS）が全クエリで実装されているか
- レイヤー構成（Handler → Service → Domain → Repository）が `architecture.md` に従っているか
- Step 7 で固定した共通契約（エラー形式、認証/テナントコンテキスト、RBAC 入口、OpenAPI 生成物の責務境界、共通 UI/API 基盤）から逸脱していないか

#### 2. セキュリティ
- SQL インジェクション対策（プレースホルダの使用）
- JWT 検証がミドルウェアで全保護エンドポイントに適用されているか
- パスワードが Argon2id でハッシュされているか
- セキュリティヘッダーが `security.md` どおりに設定されているか
- RLS のコンテキスト設定（`SET LOCAL app.current_tenant`）が同一コネクション・同一トランザクションで実行されているか
- 監査ログ・運用ログ・メトリクス等、上流で要求された観測性が欠落していないか

#### 3. テスト品質
- `test_cases/*.md` の全テストケースが通過しているか
- テスト結果が CI で自動検証されているか
- 実装コードと Step 8 のテストコードの対応が追跡可能か
- 修正に伴い追加・更新すべきテストが取りこぼされていないか

### 段階別レビュー

#### 4. BE レビュー
- API 入出力、HTTP ステータス、エラーコード形式が OpenAPI と一致しているか
- ドメインルール・認可・状態遷移・テナント分離がバックエンド単体で成立しているか
- 共通基盤への変更が必要な場合、その影響範囲が明示され独断変更になっていないか

#### 5. FE レビュー
- 画面仕様（`screens/*.md`）どおりの UI が実装されているか
- API クライアント、認証トークン管理、共通エラーハンドリングの利用方式が Step 7 の基盤と一致しているか
- 権限不足・入力エラー・状態遷移エラーなどの表示が仕様どおりユーザーに伝わるか

#### 6. 統合確認
- BE / FE を通した主要フローが成立しているか
- 不正な状態遷移で 422 を返すか
- 権限外ロールで 403 を返すか
- テナント境界越えで 404 を返すか
- API 契約・UI・テスト・ログ出力の間に食い違いがないか
- 共通基盤の利用方式が他機能と一致しているか

---

## 成果物ファイルの扱い

| 成果物 | 出力先 | 作成タスク |
|--------|--------|-----------|
| バックエンドコード（機能） | `expense-saas/apps/api/` | 各機能タスク |
| フロントエンドコード（機能） | `expense-saas/apps/web/` | 各機能タスク |

---

## タスク一覧

各機能タスクはテストケースファイル単位。テストは Step 8 で実装済みなので、BE 実装 → FE 実装の2ステップで進める。

| ID | タスク | テストケース | 依存 | 状態 |
|----|--------|-------------|------|------|
| 9-A | 認証 | auth.md（80件） | Step 8 完了（8-A） | 未着手 |
| 9-B | レポート | reports.md（90件） | 9-A | 未着手 |
| 9-C | ダッシュボード | dashboard.md（25件） | 9-A | 未着手 |
| 9-D | テナント管理 | tenant.md（11件） | 9-A | 未着手 |
| 9-E | 明細 | items.md（75件） | 9-B | 未着手 |
| 9-F | ワークフロー | workflow.md（62件） | 9-B | 未着手 |
| 9-G | 添付ファイル | attachments.md（54件） | 9-E | 未着手 |

### 依存グラフ

```
Step 8 完了
  └→ 9-A (認証) ─┬→ 9-B (レポート) ─┬→ 9-E (明細) → 9-G (添付)
                  ├→ 9-C (ダッシュボード)  └→ 9-F (ワークフロー)
                  └→ 9-D (テナント管理)
                                              ↓
                                        Step 10 (システムテスト・UAT)
```

**最大並列数**: 認証完了後に3並列（レポート・ダッシュボード・テナント管理）、レポート完了後に2並列（明細・ワークフロー）

---

## タスク詳細

### 9-A: 認証

- **テストケース**: test_cases/auth.md（80件）
- **入力**: openapi.yaml §認証, db_schema.md, authz.md, security.md, screens/auth-*.md, Step 8 のテストコード §認証
- **出力**: `apps/api/internal/` §認証, `apps/web/src/features/auth/`
- **実装フロー**:
  1. BE 実装: POST /auth/signup, login, refresh, logout, GET /auth/me、パスワードハッシュ（Argon2id）、JWT 発行（RS256）
  2. FE 実装: ログイン・サインアップ・パスワードリセット画面、JWT 管理、ルーティングガード
- **完了条件**:
  - test_cases/auth.md の全テストケースが通過している
  - signup → login → refresh → logout の一連フローが API + UI で通る

### 9-B: レポート

- **テストケース**: test_cases/reports.md（90件）
- **入力**: openapi.yaml §レポート, db_schema.md, authz.md, state_machine.md, screens/expenses.md §レポート, Step 8 のテストコード §レポート
- **出力**: `apps/api/internal/` §レポート, `apps/web/src/features/expenses/` §レポート
- **実装フロー**:
  1. BE 実装: GET/POST/PUT/DELETE /reports、状態遷移ロジック
  2. FE 実装: レポート一覧・作成/編集・詳細画面
- **完了条件**:
  - test_cases/reports.md の全テストケースが通過している
  - 不正な状態遷移で 422 を返す

### 9-C: ダッシュボード

- **テストケース**: test_cases/dashboard.md（25件）
- **入力**: openapi.yaml §ダッシュボード, screens/expenses.md §ダッシュボード, Step 8 のテストコード §ダッシュボード
- **出力**: `apps/api/internal/` §ダッシュボード, `apps/web/src/features/expenses/` §ダッシュボード
- **実装フロー**:
  1. BE 実装: GET /dashboard
  2. FE 実装: ダッシュボード画面
- **完了条件**:
  - test_cases/dashboard.md の全テストケースが通過している

### 9-D: テナント管理

- **テストケース**: test_cases/tenant.md（11件）
- **入力**: openapi.yaml §テナント, authz.md, screens/expenses.md §テナント管理, Step 8 のテストコード §テナント管理
- **出力**: `apps/api/internal/` §テナント, `apps/web/src/features/admin/`
- **実装フロー**:
  1. BE 実装: GET /tenant, GET /tenant/members
  2. FE 実装: テナント管理画面（Admin 専用）
- **完了条件**:
  - test_cases/tenant.md の全テストケースが通過している

### 9-E: 明細

- **テストケース**: test_cases/items.md（75件）
- **入力**: openapi.yaml §明細, db_schema.md, screens/expenses.md §明細, Step 8 のテストコード §明細
- **出力**: `apps/api/internal/` §明細, `apps/web/src/features/expenses/` §明細
- **実装フロー**:
  1. BE 実装: GET/POST/PUT/DELETE /reports/:id/items
  2. FE 実装: 明細追加/編集画面
- **完了条件**:
  - test_cases/items.md の全テストケースが通過している

### 9-F: ワークフロー

- **テストケース**: test_cases/workflow.md（62件）
- **入力**: openapi.yaml §ワークフロー, authz.md, state_machine.md, screens/approvals.md, Step 8 のテストコード §ワークフロー
- **出力**: `apps/api/internal/` §ワークフロー, `apps/web/src/features/approvals/`
- **実装フロー**:
  1. BE 実装: POST /reports/:id/submit, approve, reject, mark-paid、GET /workflow/pending, /workflow/payable
  2. FE 実装: 承認キュー画面、承認/却下アクション、却下理由入力
- **完了条件**:
  - test_cases/workflow.md の全テストケースが通過している
  - 権限外ロールで 403、不正遷移で 422 を返す

### 9-G: 添付ファイル

- **テストケース**: test_cases/attachments.md（54件）
- **入力**: openapi.yaml §添付, files.md, db_schema.md, authz.md, screens/attachments.md, Step 8 のテストコード §添付ファイル
- **出力**: `apps/api/internal/` §添付, `apps/web/src/features/expenses/` §添付
- **実装フロー**:
  1. BE 実装: 署名付き URL 発行 API、S3 クライアント、バリデーション
  2. FE 実装: 添付エリア（アップロード・プレビュー・削除）
- **完了条件**:
  - test_cases/attachments.md の全テストケースが通過している

### タスク完了時の更新義務

各担当はレビュー依頼前に、少なくとも以下を更新または明記すること。

- 対応したテストケース ID と実装コードの対応
- 変更した API / UI / 共通部品の一覧
- 後続タスクが取り込むべき変更点
