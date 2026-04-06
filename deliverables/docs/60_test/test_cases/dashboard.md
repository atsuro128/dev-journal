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

---

## FE テストケース

FE テストケースは `55_ui_component/screens/dashboard.md` のコンポーネント単位・Props 単位で導出する。

### 対応対象

- 対応画面: SCR-DASH-001（ダッシュボード）
- 対応コンポーネント設計: `55_ui_component/screens/dashboard.md`
- 参照設計書:
  - `55_ui_component/screens/dashboard.md`
  - `55_ui_component/common-components.md`
  - `55_ui_component/state-management.md`（useDashboard, useCurrentUser）

### FE フィクスチャ参照

| 参照名 | 値 | 用途 |
|---|---|---|
| `mockMemberUser` | `{ role: 'member', name: 'Test Member' }` | useCurrentUser のモック（Member ロール） |
| `mockApproverUser` | `{ role: 'approver', name: 'Test Approver' }` | useCurrentUser のモック（Approver ロール） |
| `mockAccountingUser` | `{ role: 'accounting', name: 'Test Accounting' }` | useCurrentUser のモック（Accounting ロール） |
| `mockAdminUser` | `{ role: 'admin', name: 'Test Admin' }` | useCurrentUser のモック（Admin ロール） |
| `mockDashboardMember` | `{ my_draft_count: 2, my_submitted_count: 1, my_rejected_count: 1, recent_reports: [...] }` | useDashboard のモック（Member レスポンス） |
| `mockDashboardApprover` | `{ ...mockDashboardMember, pending_approval_count: 3, monthly_summary: [...] }` | useDashboard のモック（Approver レスポンス） |
| `mockDashboardAccounting` | `{ ...mockDashboardMember, pending_payment_count: 2, monthly_summary: [...] }` | useDashboard のモック（Accounting レスポンス） |
| `mockDashboardAdmin` | `{ tenant_draft_count: 5, tenant_submitted_count: 3, tenant_approved_count: 2, tenant_rejected_count: 1, tenant_paid_count: 4, tenant_member_count: 10, monthly_summary: [...] }` | useDashboard のモック（Admin レスポンス） |
| `mockRecentReports` | 5件の `RecentReport` 配列 | RecentReportList の描画検証用 |
| `mockMonthlySummary` | `[{ yearMonth: '2026-04', totalAmount: 150000 }, { yearMonth: '2026-03', totalAmount: 120000 }, { yearMonth: '2026-02', totalAmount: 80000 }]` | MonthlySummaryTable の描画検証用 |

テストID プレフィックス: `DSH-FE-`

---

### DashboardPage -- ロール別表示分岐テスト

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| DSH-FE-001 | 単体 | DashboardPage | useCurrentUser (role: member) | 正常系 | DASH-F01, DASH-001 | 55_ui_component/screens/dashboard.md §DashboardPage | `test_DashboardPage_Member_ShowsMyReportCountCards` | useCurrentUser が `{ role: 'member' }` を返す。useDashboard が mockDashboardMember を返す | MyReportCountCards（下書き・提出中・却下）が表示される。TenantStatusCards / MonthlySummaryTable が表示されない |
| DSH-FE-002 | 単体 | DashboardPage | useCurrentUser (role: member) | 正常系 | DASH-F01, DASH-001 | 55_ui_component/screens/dashboard.md §DashboardPage | `test_DashboardPage_Member_ShowsRecentReportList` | useCurrentUser が `{ role: 'member' }` を返す。useDashboard が mockDashboardMember を返す | RecentReportList が表示される。承認待ちカード / 支払待ちカードが表示されない |
| DSH-FE-003 | 単体 | DashboardPage | useCurrentUser (role: approver) | 正常系 | DASH-F01, DASH-002 | 55_ui_component/screens/dashboard.md §DashboardPage | `test_DashboardPage_Approver_ShowsPendingApproval` | useCurrentUser が `{ role: 'approver' }` を返す。useDashboard が mockDashboardApprover を返す | MyReportCountCards + 承認待ちカード + MonthlySummaryTable + RecentReportList が表示される。支払待ちカード / TenantStatusCards が表示されない |
| DSH-FE-004 | 単体 | DashboardPage | useCurrentUser (role: accounting) | 正常系 | DASH-F01, DASH-003 | 55_ui_component/screens/dashboard.md §DashboardPage | `test_DashboardPage_Accounting_ShowsPendingPayment` | useCurrentUser が `{ role: 'accounting' }` を返す。useDashboard が mockDashboardAccounting を返す | MyReportCountCards + 支払待ちカード + MonthlySummaryTable + RecentReportList が表示される。承認待ちカード / TenantStatusCards が表示されない |
| DSH-FE-005 | 単体 | DashboardPage | useCurrentUser (role: admin) | 正常系 | DASH-F01, DASH-004 | 55_ui_component/screens/dashboard.md §DashboardPage | `test_DashboardPage_Admin_ShowsTenantStatusCards` | useCurrentUser が `{ role: 'admin' }` を返す。useDashboard が mockDashboardAdmin を返す | TenantStatusCards + メンバー数カード + MonthlySummaryTable が表示される。MyReportCountCards / RecentReportList / 承認待ちカード / 支払待ちカードが表示されない |

---

### DashboardPage -- ローディング・エラー表示テスト

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| DSH-FE-006 | 単体 | DashboardPage | useDashboard (isLoading) | 正常系 | DASH-F01 | 55_ui_component/screens/dashboard.md §DashboardPage | `test_DashboardPage_Loading_ShowsPageSkeleton` | useDashboard が `{ isLoading: true, data: undefined }` を返す | PageSkeleton（variant="card"）が表示される。カウントカード・テーブル等のコンテンツが表示されない |
| DSH-FE-007 | 単体 | DashboardPage | useDashboard (error) | 異常系 | DASH-F01 | 55_ui_component/screens/dashboard.md §DashboardPage | `test_DashboardPage_Error_ShowsAppToast` | useDashboard が `{ error: { status: 500, code: 'INTERNAL_ERROR', message: 'サーバーエラーが発生しました' } }` を返す | AppToast が severity="error" で表示される。エラーメッセージが画面上に表示される |

---

### CountCard -- Props 描画テスト

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| DSH-FE-008 | 単体 | CountCard | label: string, count: number | 正常系 | DASH-F01 | 55_ui_component/screens/dashboard.md §CountCard | `test_CountCard_RendersLabelAndCount` | `<CountCard label="下書き" count={5} />` | 「下書き」ラベルと「5」件数が表示される。単位「件」がデフォルトで表示される |
| DSH-FE-009 | 単体 | CountCard | href: string | 正常系 | DASH-F01 | 55_ui_component/screens/dashboard.md §CountCard | `test_CountCard_WithHref_RendersAsLink` | `<CountCard label="下書き" count={3} href="/reports?status=draft" />` | カードがリンクとしてレンダリングされ、クリックで `/reports?status=draft` に遷移する |
| DSH-FE-010 | 単体 | CountCard | href なし | 正常系 | DASH-F01 | 55_ui_component/screens/dashboard.md §CountCard | `test_CountCard_WithoutHref_NotClickable` | `<CountCard label="メンバー数" count={10} />` | カードがリンクとしてレンダリングされない。クリックしても遷移しない |
| DSH-FE-011 | 単体 | CountCard | count: 0 | 正常系 | DASH-F01 | 55_ui_component/screens/dashboard.md §CountCard | `test_CountCard_ZeroCount_StillDisplayed` | `<CountCard label="却下" count={0} />` | 件数 0 でもカードが表示され、「0」が表示される |
| DSH-FE-012 | 単体 | CountCard | showBadge: boolean | 正常系 | DASH-F01 | 55_ui_component/screens/dashboard.md §CountCard | `test_CountCard_ShowBadge_WhenCountPositive` | `<CountCard label="承認待ち" count={3} showBadge={true} />` | 要対応バッジ（赤丸）が表示される |
| DSH-FE-013 | 単体 | CountCard | showBadge: boolean, count: 0 | 正常系 | DASH-F01 | 55_ui_component/screens/dashboard.md §CountCard | `test_CountCard_ShowBadge_NotShownWhenZero` | `<CountCard label="承認待ち" count={0} showBadge={true} />` | showBadge が true でも count が 0 の場合、バッジが表示されない（count >= 1 のときのみ表示） |
| DSH-FE-014 | 単体 | CountCard | accentColor: string | 正常系 | DASH-F01 | 55_ui_component/screens/dashboard.md §CountCard | `test_CountCard_AccentColor_Applied` | `<CountCard label="承認済み" count={2} accentColor="success" />` | アクセントカラー success が視覚的に適用される |
| DSH-FE-015 | 単体 | CountCard | unit: string | 正常系 | DASH-F01, DASH-004 | 55_ui_component/screens/dashboard.md §CountCard | `test_CountCard_CustomUnit_DisplaysPerson` | `<CountCard label="メンバー数" count={10} unit="人" />` | 単位が「人」で表示される（デフォルトの「件」ではない） |

---

### MyReportCountCards -- 描画テスト

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| DSH-FE-016 | 単体 | MyReportCountCards | draftCount, submittedCount, rejectedCount | 正常系 | DASH-F01, DASH-001 | 55_ui_component/screens/dashboard.md §MyReportCountCards | `test_MyReportCountCards_RendersThreeCards` | `<MyReportCountCards draftCount={2} submittedCount={1} rejectedCount={1} />` | 「下書き: 2」「提出中: 1」「却下: 1」の 3 枚の CountCard が表示される |
| DSH-FE-017 | 単体 | MyReportCountCards | draftCount: 0, submittedCount: 0, rejectedCount: 0 | 正常系 | DASH-F01, DASH-001 | 55_ui_component/screens/dashboard.md §MyReportCountCards | `test_MyReportCountCards_AllZero_StillDisplayed` | `<MyReportCountCards draftCount={0} submittedCount={0} rejectedCount={0} />` | 全件数 0 でも 3 枚のカードが表示される |
| DSH-FE-018 | 単体 | MyReportCountCards | href（各カード） | 正常系 | DASH-F01 | 55_ui_component/screens/dashboard.md §MyReportCountCards | `test_MyReportCountCards_CardLinks_NavigateToReportList` | `<MyReportCountCards draftCount={2} submittedCount={1} rejectedCount={1} />` | 下書きカードのクリックで `/reports?status=draft` に遷移する。提出中カードは `/reports?status=submitted`、却下カードは `/reports?status=rejected` に遷移する |

---

### TenantStatusCards -- 描画テスト

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| DSH-FE-019 | 単体 | TenantStatusCards | draftCount, submittedCount, approvedCount, rejectedCount, paidCount | 正常系 | DASH-F01, DASH-004 | 55_ui_component/screens/dashboard.md §TenantStatusCards | `test_TenantStatusCards_RendersFiveCards` | `<TenantStatusCards draftCount={5} submittedCount={3} approvedCount={2} rejectedCount={1} paidCount={4} />` | 「下書き: 5」「提出済み: 3」「承認済み: 2」「却下: 1」「支払済み: 4」の 5 枚の CountCard が表示される |
| DSH-FE-020 | 単体 | TenantStatusCards | accentColor（各カード） | 正常系 | DASH-F01, DASH-004 | 55_ui_component/screens/dashboard.md §TenantStatusCards | `test_TenantStatusCards_AccentColors_MatchStatus` | `<TenantStatusCards draftCount={1} submittedCount={1} approvedCount={1} rejectedCount={1} paidCount={1} />` | 各カードにステータスに対応するアクセントカラーが適用される（default / info / success / error / secondary） |
| DSH-FE-021 | 単体 | TenantStatusCards | href（各カード） | 正常系 | DASH-F01, DASH-004 | 55_ui_component/screens/dashboard.md §TenantStatusCards | `test_TenantStatusCards_CardLinks_NavigateToAdminReportList` | `<TenantStatusCards draftCount={1} submittedCount={1} approvedCount={1} rejectedCount={1} paidCount={1} />` | 各カードクリックで SCR-ADM-001（管理者レポート一覧）にステータスフィルタ付きで遷移する |

---

### MonthlySummaryTable -- 描画テスト

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| DSH-FE-022 | 単体 | MonthlySummaryTable | items: MonthlySummaryItem[] (3件) | 正常系 | DASH-F01, DASH-005 | 55_ui_component/screens/dashboard.md §MonthlySummaryTable | `test_MonthlySummaryTable_RendersThreeMonths` | `<MonthlySummaryTable items={mockMonthlySummary} />`（3件の月別データ） | 3 行のテーブルが表示される。年月カラムと金額カラムが存在する |
| DSH-FE-023 | 単体 | MonthlySummaryTable | items[].totalAmount | 正常系 | DASH-F01, DASH-005 | 55_ui_component/screens/dashboard.md §MonthlySummaryTable | `test_MonthlySummaryTable_AmountFormat_YenWithComma` | `<MonthlySummaryTable items={[{ yearMonth: '2026-04', totalAmount: 150000 }]} />` | 金額が「¥150,000」形式で表示される（¥ プレフィックス + 3桁カンマ区切り） |
| DSH-FE-024 | 単体 | MonthlySummaryTable | items[].yearMonth | 正常系 | DASH-F01, DASH-005 | 55_ui_component/screens/dashboard.md §MonthlySummaryTable | `test_MonthlySummaryTable_YearMonthFormat` | `<MonthlySummaryTable items={[{ yearMonth: '2026-04', totalAmount: 100000 }]} />` | 年月が「2026年4月」形式で表示される |
| DSH-FE-025 | 単体 | MonthlySummaryTable | items の順序 | 正常系 | DASH-F01, DASH-005 | 55_ui_component/screens/dashboard.md §MonthlySummaryTable | `test_MonthlySummaryTable_DescendingOrder` | `<MonthlySummaryTable items={mockMonthlySummary} />`（2026-04, 2026-03, 2026-02 の順） | 最新月（2026年4月）が最上行に表示される（降順） |
| DSH-FE-026 | 単体 | MonthlySummaryTable | items: [] (空配列) | 正常系 | DASH-F01, DASH-005 | 55_ui_component/screens/dashboard.md §MonthlySummaryTable | `test_MonthlySummaryTable_EmptyItems_ShowsZero` | `<MonthlySummaryTable items={[]} />` | データが存在しない旨が表示される、またはテーブルが空行で表示される |

---

### RecentReportList -- 描画テスト

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| DSH-FE-027 | 単体 | RecentReportList | reports: RecentReport[] (5件) | 正常系 | DASH-F01, DASH-001 | 55_ui_component/screens/dashboard.md §RecentReportList | `test_RecentReportList_RendersFiveReports` | `<RecentReportList reports={mockRecentReports} />`（5件） | 5 件のレポート行が表示される |
| DSH-FE-028 | 単体 | RecentReportList | reports: [] (空配列) | 正常系 | DASH-F01, DASH-001 | 55_ui_component/screens/dashboard.md §RecentReportList | `test_RecentReportList_Empty_ShowsEmptyState` | `<RecentReportList reports={[]} />` | EmptyState コンポーネントが表示される。メッセージ「経費レポートはまだありません。レポートを作成して経費精算を始めましょう。」が表示される |
| DSH-FE-029 | 単体 | RecentReportList | ViewAllLink | 正常系 | DASH-F01 | 55_ui_component/screens/dashboard.md §RecentReportList | `test_RecentReportList_ViewAllLink_NavigatesToReports` | `<RecentReportList reports={mockRecentReports} />` | 「すべてのレポートを見る」リンクが表示され、クリックで `/reports` に遷移する |

---

### RecentReportRow -- 描画テスト

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| DSH-FE-030 | 単体 | RecentReportRow | title, id | 正常系 | DASH-F01 | 55_ui_component/screens/dashboard.md §RecentReportRow | `test_RecentReportRow_TitleLink_NavigatesToDetail` | `<RecentReportRow id="uuid-001" title="4月交通費" periodStart="2026-04-01" periodEnd="2026-04-30" totalAmount={15000} status="draft" />` | タイトル「4月交通費」がリンクとして表示され、クリックで `/reports/uuid-001`（SCR-RPT-004）に遷移する |
| DSH-FE-031 | 単体 | RecentReportRow | periodStart, periodEnd | 正常系 | DASH-F01 | 55_ui_component/screens/dashboard.md §RecentReportRow | `test_RecentReportRow_PeriodFormat` | `<RecentReportRow id="uuid-001" title="テスト" periodStart="2026-04-01" periodEnd="2026-04-30" totalAmount={10000} status="draft" />` | 対象期間が表示される（例: 「2026/04/01 - 2026/04/30」） |
| DSH-FE-032 | 単体 | RecentReportRow | totalAmount | 正常系 | DASH-F01 | 55_ui_component/screens/dashboard.md §RecentReportRow | `test_RecentReportRow_AmountFormat_YenWithComma` | `<RecentReportRow id="uuid-001" title="テスト" periodStart="2026-04-01" periodEnd="2026-04-30" totalAmount={150000} status="draft" />` | 合計金額が「¥150,000」形式で表示される |
| DSH-FE-033 | 単体 | RecentReportRow | status | 正常系 | DASH-F01 | 55_ui_component/screens/dashboard.md §RecentReportRow | `test_RecentReportRow_StatusChip_Rendered` | `<RecentReportRow id="uuid-001" title="テスト" periodStart="2026-04-01" periodEnd="2026-04-30" totalAmount={10000} status="submitted" />` | StatusChip コンポーネントが status="submitted" で描画され、「提出済み」と表示される |

---

### useDashboard Hook -- テスト

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| DSH-FE-034 | 単体 | - | useDashboard | 正常系 | DASH-F01 | 55_ui_component/state-management.md §useDashboard | `test_useDashboard_FetchesData` | useDashboard を呼び出す。MSW で `GET /api/dashboard` が 200 を返すようモック設定 | `data` に DashboardResponse が格納される。`isLoading` が false になる |
| DSH-FE-035 | 単体 | - | useDashboard | 異常系 | DASH-F01 | 55_ui_component/state-management.md §useDashboard | `test_useDashboard_Error_SetsErrorState` | useDashboard を呼び出す。MSW で `GET /api/dashboard` が 500 を返すようモック設定 | `error` が設定される。`data` が undefined である |
| DSH-FE-036 | 単体 | - | useDashboard | 正常系 | DASH-F01 | 55_ui_component/state-management.md §useDashboard | `test_useDashboard_Cache_StaleTime60s` | useDashboard を連続で 2 回呼び出す（60秒以内）。MSW で `GET /api/dashboard` のリクエスト回数を記録 | API 呼び出しが 1 回のみ発生する（staleTime: 60秒のキャッシュが有効） |

---

### useCurrentUser Hook -- テスト

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| DSH-FE-037 | 単体 | - | useCurrentUser | 正常系 | DASH-F01 | 55_ui_component/state-management.md §useCurrentUser | `test_useCurrentUser_FetchesUserInfo` | useCurrentUser を呼び出す。MSW で `GET /api/auth/me` が `{ name: 'Test', role: 'member' }` を返すようモック設定 | `data` に AuthUser が格納される。`role` が `'member'` である |
| DSH-FE-038 | 単体 | - | useCurrentUser | 異常系 | DASH-F01 | 55_ui_component/state-management.md §useCurrentUser | `test_useCurrentUser_Error_UnauthenticatedRedirect` | useCurrentUser を呼び出す。MSW で `GET /api/auth/me` が 401 を返すようモック設定 | エラーが設定される。ログイン画面へのリダイレクトが発生する（api/client.ts のリフレッシュフロー経由） |

---

### FE テストケース -- 実装ガイド

#### テストファイルの配置

```
expense-saas/
  src/
    pages/dashboard/__tests__/
      DashboardPage.test.tsx        # DSH-FE-001〜DSH-FE-007
    components/dashboard/__tests__/
      CountCard.test.tsx             # DSH-FE-008〜DSH-FE-015
      MyReportCountCards.test.tsx     # DSH-FE-016〜DSH-FE-018
      TenantStatusCards.test.tsx      # DSH-FE-019〜DSH-FE-021
      MonthlySummaryTable.test.tsx    # DSH-FE-022〜DSH-FE-026
      RecentReportList.test.tsx       # DSH-FE-027〜DSH-FE-029
      RecentReportRow.test.tsx        # DSH-FE-030〜DSH-FE-033
    hooks/__tests__/
      useDashboard.test.ts           # DSH-FE-034〜DSH-FE-036
      useCurrentUser.test.ts         # DSH-FE-037〜DSH-FE-038
```

#### テストツール

- **テストランナー**: Vitest
- **レンダリング**: @testing-library/react（`render`, `screen`, `within`）
- **イベントシミュレーション**: @testing-library/user-event
- **API モック**: MSW (Mock Service Worker)
- **ルーティングモック**: `MemoryRouter`（react-router-dom）でラップ
- **Hook テスト**: `renderHook`（@testing-library/react）+ QueryClientProvider

#### DashboardPage テストのセットアップ

DashboardPage はルートコンポーネントのため、`useDashboard` と `useCurrentUser` をモック化してレンダリングする。

```typescript
// DashboardPage.test.tsx（擬似コード）
vi.mock('@/hooks/useDashboard');
vi.mock('@/hooks/useCurrentUser');

const renderDashboard = (role: string, dashboardData: DashboardResponse) => {
  (useCurrentUser as Mock).mockReturnValue({
    data: { data: { role, name: 'Test User' } },
    isLoading: false,
  });
  (useDashboard as Mock).mockReturnValue({
    data: { data: dashboardData },
    isLoading: false,
    error: null,
  });
  return render(
    <MemoryRouter>
      <QueryClientProvider client={new QueryClient()}>
        <DashboardPage />
      </QueryClientProvider>
    </MemoryRouter>
  );
};
```

#### CountCard の遷移先テスト

```typescript
// CountCard.test.tsx（擬似コード）
test('href が指定されている場合、リンクとしてレンダリングされる', () => {
  render(
    <MemoryRouter>
      <CountCard label="下書き" count={3} href="/reports?status=draft" />
    </MemoryRouter>
  );
  const link = screen.getByRole('link', { name: /下書き/ });
  expect(link).toHaveAttribute('href', '/reports?status=draft');
});
```

#### Hook テストのセットアップ

```typescript
// useDashboard.test.ts（擬似コード）
const wrapper = ({ children }: { children: React.ReactNode }) => (
  <QueryClientProvider client={new QueryClient({
    defaultOptions: { queries: { retry: false } },
  })}>
    {children}
  </QueryClientProvider>
);

test('API からダッシュボードデータを取得する', async () => {
  server.use(
    http.get('/api/dashboard', () => {
      return HttpResponse.json({ data: mockDashboardMember });
    })
  );
  const { result } = renderHook(() => useDashboard(), { wrapper });
  await waitFor(() => expect(result.current.isSuccess).toBe(true));
  expect(result.current.data?.data).toEqual(mockDashboardMember);
});
```

---

### FE テストケース -- 優先度

| 優先度 | テストID |
|--------|---------|
| 高（必須） | DSH-FE-001, DSH-FE-003, DSH-FE-004, DSH-FE-005, DSH-FE-006, DSH-FE-007, DSH-FE-008, DSH-FE-016, DSH-FE-027, DSH-FE-028, DSH-FE-034 |
| 中 | DSH-FE-002, DSH-FE-009, DSH-FE-011, DSH-FE-012, DSH-FE-017, DSH-FE-018, DSH-FE-019, DSH-FE-022, DSH-FE-023, DSH-FE-029, DSH-FE-030, DSH-FE-035, DSH-FE-037 |
| 低（ベストエフォート） | DSH-FE-010, DSH-FE-013, DSH-FE-014, DSH-FE-015, DSH-FE-020, DSH-FE-021, DSH-FE-024, DSH-FE-025, DSH-FE-026, DSH-FE-031, DSH-FE-032, DSH-FE-033, DSH-FE-036, DSH-FE-038 |

**優先度「高」の選定理由**:

- DSH-FE-001 / DSH-FE-003 / DSH-FE-004 / DSH-FE-005: ロール別表示分岐はダッシュボード画面の核心機能。4 ロール全パターンの表示制御を保証する
- DSH-FE-006 / DSH-FE-007: ローディング状態とエラー表示はユーザー体験の基盤。スケルトン表示とエラートーストの動作を保証する
- DSH-FE-008: CountCard はダッシュボード画面で最も多用される基本コンポーネント。label / count の基本描画を保証する
- DSH-FE-016: MyReportCountCards は Member / Approver / Accounting の 3 ロールで表示される主要セクション。3 枚のカード描画を保証する
- DSH-FE-027 / DSH-FE-028: RecentReportList のデータ有無による描画分岐（5件表示 / EmptyState）を保証する
- DSH-FE-034: useDashboard Hook は画面全体のデータ取得基盤。API 呼び出しとデータ格納を保証する
