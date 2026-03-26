# テストケース: ダッシュボード / カテゴリ

## この文書の役割

| 項目 | 内容 |
|------|------|
| 目的 | ダッシュボード取得（getDashboard）・カテゴリ一覧（listCategories）エンドポイントのテストケースを定義する |
| 正本情報 | DASH-F01 および ITM-005（カテゴリ固定6種類）に対応するテストケース一覧 |
| 扱わない内容 | テナント横断テスト（→ cross-cutting.md）、レポート詳細テスト（→ reports.md） |
| 主な参照元 | `50_detail_design/openapi.yaml#/api/dashboard`, `10_requirements/requirements.md#DASH-F*` |
| 主な参照先 | `60_test/traceability.md`, テスト実装（Step 9） |

---

本ファイルは以下の 2 ハンドラに対応するテストケースを定義する。

| ハンドラ | テストファイル | エンドポイント | operationId |
|---------|--------------|--------------|-------------|
| ダッシュボード | `dashboard_handler_test.go` | `GET /api/dashboard` | `getDashboard` |
| カテゴリ一覧 | `category_handler_test.go` | `GET /api/categories` | `listCategories` |

テストID プレフィックス: `DSH-`

---

## フィクスチャ参照

標準フィクスチャは `test_strategy.md` §4 を参照。本ファイルで使用するフィクスチャを以下にまとめる。

### テナント / ユーザー

| 変数名 | テナント | ロール | ユーザーID |
|--------|---------|--------|-----------|
| `tenantA` | Test Company A | - | `aaaaaaaa-0001-0001-0001-000000000001` |
| `userAdmin` | テナントA | Admin | `aaaaaaaa-1111-1111-1111-000000000001` |
| `userApprover` | テナントA | Approver | `aaaaaaaa-2222-2222-2222-000000000002` |
| `userMember` | テナントA | Member | `aaaaaaaa-3333-3333-3333-000000000003` |
| `userAccounting` | テナントA | Accounting | `aaaaaaaa-4444-4444-4444-000000000004` |
| `userMemberB` | テナントB | Member | `bbbbbbbb-3333-3333-3333-000000000003` |

### 経費レポート（テナントA / 作成者: `userMember`）

ダッシュボードの集計値を決定論的に検証するため、テスト前に以下のフィクスチャを投入する。

| フィクスチャ名 | 状態 | 補足 |
|-------------|------|------|
| `report_draft` | `draft` | `userMember` 所有 |
| `report_draft_empty` | `draft` | `userMember` 所有 |
| `report_submitted` | `submitted` | `userMember` 所有（承認者: 未割当） |
| `report_approved` | `approved` | `userMember` 所有 |
| `report_rejected` | `rejected` | `userMember` 所有 |
| `report_paid` | `paid` | `userMember` 所有 |

ダッシュボードの `pending_approval_count` / `pending_payment_count` を検証するために使用するレポートは上記から流用する。

---

## テストケース一覧

### GET /api/dashboard — ダッシュボード取得

#### 認可テスト（未認証 / ロール境界）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| DSH-001 | 統合 | handler | 異常系 | DASH-F01 | openapi.yaml#getDashboard | `TestGetDashboard_Unauthorized_NoToken` | 認証トークンなしで `GET /api/dashboard` | `401 UNAUTHORIZED` |
| DSH-002 | 統合 | handler | 異常系 | DASH-F01, SEC-003 | openapi.yaml#getDashboard, security.md#2.1 | `TestGetDashboard_Unauthorized_ExpiredToken` | 有効期限切れアクセストークンで `GET /api/dashboard` | `401 TOKEN_EXPIRED` |

**備考**: `GET /api/dashboard` の許可ロールは Member / Approver / Admin / Accounting の全 4 ロールである（`authz.md` §6.2）。ロール不足による 403 は発生しないため、RBAC 拒否テストは不要。

---

#### Member ロール — 返却フィールドテスト

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| DSH-003 | 統合 | handler | 正常系 | DASH-F01, DASH-001 | openapi.yaml#getDashboard | `TestGetDashboard_Member_CountsMatchFixtures` | `userMember` のトークンで `GET /api/dashboard`。テナントA に `report_draft` × 2、`report_submitted` × 1、`report_rejected` × 1 が存在する | HTTP 200。`data.my_draft_count == 2`、`data.my_submitted_count == 1`、`data.my_rejected_count == 1` |
| DSH-004 | 統合 | handler | 正常系 | DASH-F01 | openapi.yaml#getDashboard | `TestGetDashboard_Member_RecentReports_MaxFive` | `userMember` のトークンで `GET /api/dashboard`。`userMember` が 6 件のレポートを作成済み | HTTP 200。`data.recent_reports` が最大 5 件である |
| DSH-005 | 統合 | handler | 正常系 | DASH-F01, DASH-001 | openapi.yaml#getDashboard | `TestGetDashboard_Member_NoApproverFields` | `userMember` のトークンで `GET /api/dashboard` | HTTP 200。レスポンス `data` に `pending_approval_count`、`monthly_summary`、`tenant_draft_count` 等の Approver / Admin 専用フィールドが含まれない（またはゼロ値でなく省略される） |

---

#### Approver ロール — 返却フィールドテスト

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| DSH-006 | 統合 | handler | 正常系 | DASH-F01, DASH-002 | openapi.yaml#getDashboard | `TestGetDashboard_Approver_MemberFieldsPresent` | `userApprover` のトークンで `GET /api/dashboard` | HTTP 200。`data` に `my_draft_count`、`my_submitted_count`、`my_rejected_count`、`recent_reports` が含まれる（Approver は Member フィールドも保持） |
| DSH-007 | 統合 | handler | 正常系 | DASH-F01, DASH-002 | openapi.yaml#getDashboard | `TestGetDashboard_Approver_PendingApprovalCount` | `userApprover` のトークンで `GET /api/dashboard`。テナントA に `userMember` 所有の `report_submitted` が 1 件存在し、`userApprover` 自身の submitted レポートが 0 件 | HTTP 200。`data.pending_approval_count == 1`（自分のレポートを除いた件数） |
| DSH-008 | 統合 | handler | 正常系 | DASH-F01, DASH-002 | openapi.yaml#getDashboard | `TestGetDashboard_Approver_PendingApprovalExcludesSelf` | `userApprover` のトークンで `GET /api/dashboard`。`userApprover` 自身が作成した submitted レポートが 1 件、`userMember` 作成の submitted レポートが 1 件 | HTTP 200。`data.pending_approval_count == 1`（自分のレポートはカウントから除外される） |
| DSH-009 | 統合 | handler | 正常系 | DASH-F01, DASH-005 | openapi.yaml#getDashboard | `TestGetDashboard_Approver_MonthlySummaryPresent` | `userApprover` のトークンで `GET /api/dashboard` | HTTP 200。`data.monthly_summary` が配列として存在する（直近 3 ヶ月分、要素数 0〜3） |

---

#### Accounting ロール — 返却フィールドテスト

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| DSH-010 | 統合 | handler | 正常系 | DASH-F01, DASH-003 | openapi.yaml#getDashboard | `TestGetDashboard_Accounting_MemberFieldsPresent` | `userAccounting` のトークンで `GET /api/dashboard` | HTTP 200。`data` に `my_draft_count`、`my_submitted_count`、`my_rejected_count`、`recent_reports` が含まれる |
| DSH-011 | 統合 | handler | 正常系 | DASH-F01, DASH-003 | openapi.yaml#getDashboard | `TestGetDashboard_Accounting_PendingPaymentCount` | `userAccounting` のトークンで `GET /api/dashboard`。テナントA に `report_approved` が 1 件存在 | HTTP 200。`data.pending_payment_count == 1` |
| DSH-012 | 統合 | handler | 正常系 | DASH-F01, DASH-005 | openapi.yaml#getDashboard | `TestGetDashboard_Accounting_MonthlySummaryPresent` | `userAccounting` のトークンで `GET /api/dashboard` | HTTP 200。`data.monthly_summary` が配列として存在する |
| DSH-013 | 統合 | handler | 正常系 | DASH-F01, DASH-003 | openapi.yaml#getDashboard | `TestGetDashboard_Accounting_NoApproverFields` | `userAccounting` のトークンで `GET /api/dashboard` | HTTP 200。レスポンス `data` に `pending_approval_count` が含まれない（Accounting 専用フィールドのみ） |

---

#### Admin ロール — 返却フィールドテスト

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| DSH-014 | 統合 | handler | 正常系 | DASH-F01, DASH-004 | openapi.yaml#getDashboard | `TestGetDashboard_Admin_TenantCounts` | `userAdmin` のトークンで `GET /api/dashboard`。テナントA 全体に `draft` × 2、`submitted` × 1、`approved` × 1、`rejected` × 1、`paid` × 1 のレポートが存在 | HTTP 200。`data.tenant_draft_count == 2`、`data.tenant_submitted_count == 1`、`data.tenant_approved_count == 1`、`data.tenant_rejected_count == 1`、`data.tenant_paid_count == 1` |
| DSH-015 | 統合 | handler | 正常系 | DASH-F01, DASH-004 | openapi.yaml#getDashboard | `TestGetDashboard_Admin_TenantMemberCount` | `userAdmin` のトークンで `GET /api/dashboard`。テナントA に 4 名のメンバーが存在 | HTTP 200。`data.tenant_member_count == 4` |
| DSH-016 | 統合 | handler | 正常系 | DASH-F01, DASH-005 | openapi.yaml#getDashboard | `TestGetDashboard_Admin_MonthlySummaryPresent` | `userAdmin` のトークンで `GET /api/dashboard` | HTTP 200。`data.monthly_summary` が配列として存在する |
| DSH-017 | 統合 | handler | 正常系 | DASH-F01, DASH-004 | openapi.yaml#getDashboard | `TestGetDashboard_Admin_NoMyDraftCount` | `userAdmin` のトークンで `GET /api/dashboard` | HTTP 200。レスポンス `data` に `my_draft_count`、`my_submitted_count`、`my_rejected_count`、`recent_reports`、`pending_approval_count`、`pending_payment_count` が含まれない（Admin は個人カウント / ワークフローカウントを持たない） |

---

#### テナント分離テスト

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| DSH-018 | 統合 | handler | テナント分離 | DASH-F01, TNT-F02 | openapi.yaml#getDashboard, db_schema.md#RLS | `TestGetDashboard_TenantIsolation_CountsExcludeOtherTenant` | `userMember`（テナントA）のトークンで `GET /api/dashboard`。テナントB に `report_tenant_b_draft`（draft）が 1 件存在。テナントA には `userMember` の draft が 1 件のみ | HTTP 200。`data.my_draft_count == 1`（テナントBのレポートはカウントに含まれない） |

---

### GET /api/categories — カテゴリ一覧取得

#### 正常系テスト

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| DSH-019 | 統合 | handler | 正常系 | ITM-005 | openapi.yaml#listCategories, db_schema.md#categories | `TestListCategories_OK_Member` | `userMember` のトークンで `GET /api/categories` | HTTP 200。`data` にカテゴリ配列が返る。各要素に `id`、`code`、`name_ja`、`sort_order` が含まれる |
| DSH-020 | 統合 | handler | 正常系 | ITM-005 | openapi.yaml#listCategories, db_schema.md#categories | `TestListCategories_ContainsStandardCategories` | `userMember` のトークンで `GET /api/categories` | HTTP 200。`data` に `transportation`（交通費）、`accommodation`（宿泊費）、`food`（飲食費）、`supplies`（消耗品費）、`communication`（通信費）、`other`（その他）が含まれる |
| DSH-021 | 統合 | handler | 正常系 | ITM-005 | openapi.yaml#listCategories, db_schema.md#categories | `TestListCategories_SortedBySortOrder` | `userMember` のトークンで `GET /api/categories` | HTTP 200。`data` の要素が `sort_order` 昇順で並んでいる |

#### 認可テスト（全ロール許可）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| DSH-022 | 統合 | handler | 認可 | ITM-005, RBAC-F01 | openapi.yaml#listCategories, authz.md#3 | `TestListCategories_OK_Approver` | `userApprover` のトークンで `GET /api/categories` | HTTP 200 |
| DSH-023 | 統合 | handler | 認可 | ITM-005, RBAC-F01 | openapi.yaml#listCategories, authz.md#3 | `TestListCategories_OK_Accounting` | `userAccounting` のトークンで `GET /api/categories` | HTTP 200 |
| DSH-024 | 統合 | handler | 認可 | ITM-005, RBAC-F01 | openapi.yaml#listCategories, authz.md#3 | `TestListCategories_OK_Admin` | `userAdmin` のトークンで `GET /api/categories` | HTTP 200 |
| DSH-025 | 統合 | handler | 異常系 | ITM-005 | openapi.yaml#listCategories | `TestListCategories_Unauthorized_NoToken` | 認証トークンなしで `GET /api/categories` | `401 UNAUTHORIZED` |

---

## 実装ガイド

### テストファイルの配置

```
expense-saas/
  internal/
    handler/
      dashboard_handler_test.go   # DSH-001〜DSH-018
      category_handler_test.go    # DSH-019〜DSH-025
```

### テストセットアップ（共通）

`test_strategy.md` §9.3 の方針に従い、各テスト関数の先頭で全テーブルを TRUNCATE してフィクスチャを再投入する。

```go
// dashboard_handler_test.go（擬似コード）
func TestMain(m *testing.M) {
    // テスト用DB起動確認
    // マイグレーション適用
    os.Exit(m.Run())
}

func setupDashboardFixtures(t *testing.T, db *sql.DB) {
    t.Helper()
    truncateAllTables(t, db)
    insertTenants(t, db)       // テナントA / テナントB
    insertUsers(t, db)         // 4ロール × テナントA + 1名 × テナントB
    insertReports(t, db)       // フィクスチャのレポート群
}
```

### ダッシュボード集計値の決定論的な検証

ダッシュボードの集計カウントはテスト実行時のDB状態に依存するため、テスト前にフィクスチャを確定してから期待値を固定する。

```go
// テスト例（擬似コード）: DSH-003
func TestGetDashboard_Member_CountsMatchFixtures(t *testing.T) {
    setupDashboardFixtures(t, db) // 状態を確定させる

    w := httptest.NewRecorder()
    req := newAuthenticatedRequest(t, "GET", "/api/dashboard", nil, memberToken)
    router.ServeHTTP(w, req)

    require.Equal(t, http.StatusOK, w.Code)

    var resp struct {
        Data struct {
            MyDraftCount     int `json:"my_draft_count"`
            MySubmittedCount int `json:"my_submitted_count"`
            MyRejectedCount  int `json:"my_rejected_count"`
        } `json:"data"`
    }
    require.NoError(t, json.Unmarshal(w.Body.Bytes(), &resp))
    assert.Equal(t, 2, resp.Data.MyDraftCount)
    assert.Equal(t, 1, resp.Data.MySubmittedCount)
    assert.Equal(t, 1, resp.Data.MyRejectedCount)
}
```

### フィールド有無の検証（DSH-005 / DSH-013 / DSH-017）

ロール別に返却フィールドが異なる。`encoding/json` の `RawMessage` または動的な `map[string]interface{}` でデコードし、キーの有無を検証する。

```go
// フィールド不在の検証例（擬似コード）
var raw map[string]interface{}
require.NoError(t, json.Unmarshal(w.Body.Bytes(), &raw))
data, _ := raw["data"].(map[string]interface{})
_, exists := data["pending_approval_count"]
assert.False(t, exists, "Member には pending_approval_count が含まれないこと")
```

### カテゴリの sort_order 検証（DSH-021）

```go
// sort_order 昇順の検証例（擬似コード）
for i := 1; i < len(categories); i++ {
    assert.LessOrEqual(t, categories[i-1].SortOrder, categories[i].SortOrder)
}
```

---

## 優先度

| 優先度 | テストID |
|--------|---------|
| 高（必須） | DSH-001, DSH-002, DSH-003, DSH-007, DSH-008, DSH-014, DSH-018, DSH-019, DSH-025 |
| 中 | DSH-004, DSH-005, DSH-006, DSH-009, DSH-010, DSH-011, DSH-015, DSH-017, DSH-020, DSH-021 |
| 低（ベストエフォート） | DSH-012, DSH-013, DSH-016, DSH-022, DSH-023, DSH-024 |

**優先度「高」の選定理由**:

- DSH-001 / DSH-002: 認証必須の基本保護
- DSH-003: Member 向け集計の正確性（最も一般的なロールの核心機能）
- DSH-007 / DSH-008: `pending_approval_count` の正確性と自分のレポートの除外ロジック（DASH-001〜005 に相当するビジネスルール）
- DSH-014: Admin の全テナント集計（テナント管理者の核心機能）
- DSH-018: テナント分離（クロステナント集計汚染の防止）
- DSH-019: カテゴリ一覧の基本動作
- DSH-025: カテゴリ一覧の認証保護
