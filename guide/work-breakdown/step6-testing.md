# Step 6: テスト設計 — 作業分解

## 概要

重要領域（テナント分離・RBAC・状態遷移・認可）のテスト戦略を策定し、テストケースを定義する。

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
| セキュリティ設計 | `deliverables/docs/50_detail_design/security.md` |
| 状態遷移 | `deliverables/docs/20_domain/state_machine.md` |
| RBAC | `deliverables/docs/10_requirements/rbac.md` |

### 作業計画

Step 着手時にまず作業計画を立案し、以下に保存する:

- テンプレート: `guide/templates/task-plan.md`
- 配置先: `progress-management/task-plans/step6-testing.md`
- 計画が確定してから成果物作成に入ること

### 完了条件（Step 全体）

- テスト戦略（レベル別の方針・ツール・カバレッジ基準）が定義されている
- 重要領域のテストケースが網羅されている
- CI で自動テスト（単体・統合）が走る設計になっている
- E2E テストの Playwright 設定方針が整っている

---

## レビュー観点

成果物内の「品質チェック」は作成者のセルフチェック。以下は reviewer が検証する観点。

> Step 6 は未着手のため、以下は現時点で定義できる観点。着手時にテスト戦略の具体内容を踏まえて精緻化すること。

### 1. 上流整合性
- テスト対象が Step 5 の全成果物（OpenAPI, db_schema, authz, security, files, screens）を網羅しているか
- テストケースの期待値が `state_machine.md` の遷移ルール・`rbac.md` の権限マトリクスと一致しているか
- 非機能要件（レスポンスタイム、レート制限）に対応するテスト方針が含まれているか

### 2. 重要領域の網羅性
- テナント分離テスト: Tenant A → Tenant B のデータアクセス不可が全リソースで検証されているか
- RBAC テスト: 各ロールが許可されていない操作で 403 を返すことが全エンドポイントで検証されているか
- 状態遷移テスト: 不正な遷移が 422 で拒否されることが全パターンで検証されているか
- 自己承認禁止・自己処理禁止のテストケースが含まれているか
- テナント境界越え=404 のテストケースが含まれているか

### 3. テストレベルの適切性
- 各テストケースにテストレベル（単体/統合/E2E）が割り当てられているか
- ドメインロジック（状態遷移・バリデーション）は単体テストに分類されているか
- API エンドポイント・認証認可・DB 操作は統合テストに分類されているか
- 業務フロー（申請→承認→支払完了）は E2E テストに分類されているか

### 4. テスト設計の実現可能性
- テストデータ戦略（テナント分離を考慮したシード設計）が定義されているか
- CI 組み込み方針（PR 時に単体+統合、マージ時に E2E）が Step 3 の CI パイプライン設計と整合しているか
- Playwright の設定方針が定義されているか

### 5. 用語準拠
- テストケース名・期待値の記述が `glossary.md` に準拠しているか

### 6. 下流作業可能性
- Step 7（実装）の作業者が、各テストケースを迷わずコードに落とし込める粒度で書かれているか
- テストケースごとに「入力」「操作」「期待結果」が明確に定義されているか
- 各ファイル（`test_cases/*.md`）が対応するハンドラのテスト実装に必要な情報を単独で提供しているか

---

## 成果物ファイルの扱い

| 成果物 | パス | 作成タスク |
|--------|------|-----------|
| テスト戦略 | `deliverables/docs/60_test/test_strategy.md` | 6-A |
| 認証テストケース | `deliverables/docs/60_test/test_cases/auth.md` | 6-B-1 |
| レポートテストケース | `deliverables/docs/60_test/test_cases/reports.md` | 6-B-2 |
| 明細テストケース | `deliverables/docs/60_test/test_cases/items.md` | 6-B-3 |
| 添付テストケース | `deliverables/docs/60_test/test_cases/attachments.md` | 6-B-4 |
| ワークフローテストケース | `deliverables/docs/60_test/test_cases/workflow.md` | 6-B-5 |
| ダッシュボード・カテゴリテストケース | `deliverables/docs/60_test/test_cases/dashboard.md` | 6-B-6 |
| テナント管理テストケース | `deliverables/docs/60_test/test_cases/tenant.md` | 6-B-7 |
| 横断テストケース | `deliverables/docs/60_test/test_cases/cross-cutting.md` | 6-B-8 |

---

## タスク一覧

| ID | タスク | 種別 | 依存 | 状態 |
|----|--------|------|------|------|
| 6-A | テスト戦略策定 | 基盤 | Step 5 完了 | 未着手 |
| 6-B-1 | 認証テストケース定義 | 機能 | 6-A | 未着手 |
| 6-B-2 | レポートテストケース定義 | 機能 | 6-A | 未着手 |
| 6-B-3 | 明細テストケース定義 | 機能 | 6-A | 未着手 |
| 6-B-4 | 添付テストケース定義 | 機能 | 6-A | 未着手 |
| 6-B-5 | ワークフローテストケース定義 | 機能 | 6-A | 未着手 |
| 6-B-6 | ダッシュボード・カテゴリテストケース定義 | 機能 | 6-A | 未着手 |
| 6-B-7 | テナント管理テストケース定義 | 機能 | 6-A | 未着手 |
| 6-B-8 | 横断テストケース定義 | 横断 | 6-B-1〜7 | 未着手 |

### 依存グラフ

```
Step 5 完了
  └→ 6-A (テスト戦略)
       ├→ 6-B-1 (認証TC)          ─┐
       ├→ 6-B-2 (レポートTC)      ─┤
       ├→ 6-B-3 (明細TC)          ─┤
       ├→ 6-B-4 (添付TC)          ─┤→ 6-B-8 (横断TC)
       ├→ 6-B-5 (ワークフローTC)  ─┤
       ├→ 6-B-6 (ダッシュボードTC) ─┤
       └→ 6-B-7 (テナント管理TC)  ─┘
```

---

## タスク詳細

### 6-A: テスト戦略策定

- **入力**: 上流成果物全体（特に authz.md, db_schema.md, openapi.yaml, state_machine.md）
- **出力**: `deliverables/docs/60_test/test_strategy.md`
- **作業内容**:
  - テストレベルと対象ツールの定義
    - 単体テスト: Go 標準テスト（ドメインロジック・状態遷移・バリデーション）
    - 統合テスト: Go + テスト用 DB（API エンドポイント・認証認可・DB 操作）
    - E2E テスト: Playwright（申請〜承認〜支払いの一連フロー）
  - カバレッジ基準（重要領域は高カバレッジ、UI は E2E で主要パスのみ）
  - テストデータ定義（テスト用テナント・ユーザー・ロール構成のフィクスチャ定義）
  - CI 組み込み方針（PR 時に単体+統合、マージ時に E2E）
  - テストID命名規則（リソース略称プレフィックス: AUTH-, RPT-, ITM-, ATT-, WFL-, DSH-, TNT-）
  - 機能別ファイルと cross-cutting.md の責務境界ルール:
    - テナント分離テスト → cross-cutting.md に集約（機能別ファイルには書かない）
    - RBACテスト: 各エンドポイントの権限別レスポンス → 機能別ファイル、ロールマトリクス全体の整合性 → cross-cutting.md
    - 状態遷移テスト → 機能別ファイル（reports.md）で完結
    - E2Eシナリオ → cross-cutting.md に集約
  - 各テストケースファイルに含める標準列の定義:
    - テストID（プレフィックス + 連番）
    - テストレベル（単体/統合/E2E）
    - レイヤー（domain/repository/handler）
    - テスト関数名候補
    - 入力（前提条件含む）
    - 期待結果
- **完了条件**:
  - テストレベル別の方針・ツール・カバレッジ基準が定義されている
  - CI でのテスト実行方針が決まっている

### 6-B-1: 認証テストケース定義

- **入力**: test_strategy.md（6-A）、openapi.yaml §/api/auth/*, security.md、authz.md §認証
- **出力**: `deliverables/docs/60_test/test_cases/auth.md`
- **対応ハンドラ**: auth_handler_test.go
- **作業内容**:
  - signup/login/refresh/logout/me/password-reset の各エンドポイントテスト（エンドポイント数: 7）
  - ドメイン層テスト（Argon2id ハッシュ検証、JWT 生成・検証）
  - 標準列（テストID: AUTH-XXX、テストレベル、レイヤー、テスト関数名候補、入力、期待結果）に従って記述
- **完了条件**:
  - 7 エンドポイントの正常系・異常系テストケースが網羅されている
  - ドメイン層テストが単体テストとして分類されている

### 6-B-2: レポートテストケース定義

- **入力**: test_strategy.md（6-A）、openapi.yaml §/api/reports/*, state_machine.md、authz.md §レポート
- **出力**: `deliverables/docs/60_test/test_cases/reports.md`
- **対応ハンドラ**: report_handler_test.go、domain/report_test.go
- **作業内容**:
  - レポートCRUD + submit のエンドポイントテスト（GET/POST /reports、GET/PUT/DELETE /reports/{id}、POST /reports/{id}/submit、/reports/all も含む）（エンドポイント数: 4）
  - ドメイン層状態遷移テスト（許可遷移 T1〜T5、禁止遷移 X1〜X10）
  - 標準列（テストID: RPT-XXX）に従って記述
- **完了条件**:
  - 全エンドポイントの正常系・異常系テストケースが網羅されている
  - 許可遷移 T1〜T5 と禁止遷移 X1〜X10 が単体テストとして網羅されている

### 6-B-3: 明細テストケース定義

- **入力**: test_strategy.md（6-A）、openapi.yaml §/api/reports/{id}/items/*, authz.md §明細
- **出力**: `deliverables/docs/60_test/test_cases/items.md`
- **対応ハンドラ**: item_handler_test.go
- **作業内容**:
  - 明細CRUD のエンドポイントテスト（GET/POST /items、GET/PUT/DELETE /items/{itemId}）（エンドポイント数: 3）
  - レポートが submitted 以降の状態では PUT/DELETE items は拒否される前提条件テスト
  - 標準列（テストID: ITM-XXX）に従って記述
- **完了条件**:
  - 3 エンドポイントの正常系・異常系テストケースが網羅されている
  - レポート状態による操作制限テストが含まれている

### 6-B-4: 添付テストケース定義

- **入力**: test_strategy.md（6-A）、openapi.yaml §attachments、files.md、authz.md §添付
- **出力**: `deliverables/docs/60_test/test_cases/attachments.md`
- **対応ハンドラ**: attachment_handler_test.go
- **作業内容**:
  - 署名付きURL発行・削除のエンドポイントテスト（エンドポイント数: 2）
  - MIME タイプ制限・ファイルサイズ制限のバリデーションテスト
  - 標準列（テストID: ATT-XXX）に従って記述
- **完了条件**:
  - 2 エンドポイントの正常系・異常系テストケースが網羅されている
  - MIME/サイズ制限テストが含まれている

### 6-B-5: ワークフローテストケース定義

- **入力**: test_strategy.md（6-A）、openapi.yaml §/api/workflow/*, state_machine.md、authz.md §承認
- **出力**: `deliverables/docs/60_test/test_cases/workflow.md`
- **対応ハンドラ**: workflow_handler_test.go
- **作業内容**:
  - pending 一覧・approve/reject/pay アクション・payable 一覧のエンドポイントテスト（エンドポイント数: 5）
  - 自己承認禁止テスト
  - 標準列（テストID: WFL-XXX）に従って記述
- **完了条件**:
  - 5 エンドポイントの正常系・異常系テストケースが網羅されている
  - 自己承認禁止テストが含まれている

### 6-B-6: ダッシュボード・カテゴリテストケース定義

- **入力**: test_strategy.md（6-A）、openapi.yaml §/api/dashboard、§/api/categories
- **出力**: `deliverables/docs/60_test/test_cases/dashboard.md`
- **対応ハンドラ**: dashboard_handler_test.go、category_handler_test.go
- **作業内容**:
  - ダッシュボード集計のロール別表示差異テスト、カテゴリ一覧取得テスト（エンドポイント数: 2）
  - 標準列（テストID: DSH-XXX）に従って記述
- **完了条件**:
  - 2 エンドポイントの正常系・異常系テストケースが網羅されている
  - ロール別の集計結果差異が検証されている

### 6-B-7: テナント管理テストケース定義

- **入力**: test_strategy.md（6-A）、openapi.yaml §/api/tenant/*, authz.md §テナント管理
- **出力**: `deliverables/docs/60_test/test_cases/tenant.md`
- **対応ハンドラ**: tenant_handler_test.go
- **作業内容**:
  - テナント情報取得・メンバー一覧のエンドポイントテスト（エンドポイント数: 2）
  - Admin 専用認可テスト（Admin 以外のロールで 403 を返すことを検証）
  - 標準列（テストID: TNT-XXX）に従って記述
- **完了条件**:
  - 2 エンドポイントの正常系・異常系テストケースが網羅されている
  - Admin 専用認可テストが含まれている

### 6-B-8: 横断テストケース定義

- **入力**: 6-B-1〜7 の全テストケースファイル、authz.md、rbac.md
- **出力**: `deliverables/docs/60_test/test_cases/cross-cutting.md`
- **作業内容**:
  - §テナント分離マトリクス: 全リソース × テナント境界越え = 404 の検証一覧
  - §RBACマトリクス: 全エンドポイント × 全ロールの許可/拒否マトリクス
  - §E2Eシナリオ: 申請→承認→支払完了の正常フロー、却下→再申請フロー等
- **完了条件**:
  - テナント分離マトリクスが全リソースを網羅している
  - RBACマトリクスが全エンドポイント × 全ロールをカバーしている
  - 主要業務フローの E2E シナリオが定義されている
