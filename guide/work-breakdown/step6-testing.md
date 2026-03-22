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

## 成果物ファイルの扱い

| 成果物 | パス | 作成タスク | 追記タスク |
|--------|------|-----------|-----------|
| テスト戦略 | `deliverables/docs/60_test/test_strategy.md` | 6-A | — |
| テストケース | `deliverables/docs/60_test/test_cases.md` | 6-B | — |

---

## タスク一覧

| ID | タスク | 種別 | 依存 | 状態 |
|----|--------|------|------|------|
| 6-A | テスト戦略策定 | 基盤 | Step 4+5 完了 | 未着手 |
| 6-B | テストケース定義 | 基盤 | 6-A | 未着手 |

### 依存グラフ

```
Step 4+5 完了
  └→ 6-A (テスト戦略) → 6-B (テストケース)
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
  - テストデータ戦略（テナント分離を考慮したシード設計）
  - CI 組み込み方針（PR 時に単体+統合、マージ時に E2E）
- **完了条件**:
  - テストレベル別の方針・ツール・カバレッジ基準が定義されている
  - CI でのテスト実行方針が決まっている

### 6-B: テストケース定義

- **入力**: test_strategy.md（6-A）、上流成果物
- **出力**: `deliverables/docs/60_test/test_cases.md`
- **作業内容**:
  - 品質保証の観点で重要なテストケース定義:
    - テナント分離テスト（Tenant A → Tenant B のデータアクセス不可を全リソースで検証）
    - RBAC テスト（各ロールが許可されていない操作で 403 を返すことを検証）
    - ワークフロー状態遷移テスト（不正な遷移が拒否されることを検証）
    - 添付 URL 発行の認可テスト
  - 機能別テストケース（認証・経費CRUD・添付・承認）
  - テストデータ定義（テスト用テナント・ユーザー・レポートの具体値）
- **完了条件**:
  - 重要領域（テナント分離・RBAC・状態遷移・認可）のテストケースが網羅されている
  - 各テストケースにテストレベル（単体/統合/E2E）が割り当てられている
