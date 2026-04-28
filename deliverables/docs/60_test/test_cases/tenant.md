# テストケース: テナント管理（TNT-）

## この文書の役割

| 項目 | 内容 |
|------|------|
| 目的 | テナント情報・メンバー管理エンドポイント（getTenant, listTenantMembers）のテストケースを定義する |
| 正本情報 | ADM-F01 および RPT-F07（メンバー一覧付随機能）に対応するテストケース一覧 |
| 扱わない内容 | テナント横断テスト（→ cross-cutting.md）、Phase 3 のメンバー招待・ロール変更テスト |
| 主な参照元 | `50_detail_design/openapi.yaml#/api/tenant`, `50_detail_design/authz.md#6.7`, `10_requirements/requirements.md#ADM-F01` |
| 主な参照先 | `60_test/traceability.md`, テスト実装（Step 9） |

---

## 概要

| 項目 | 内容 |
|------|------|
| 対象エンドポイント | `GET /api/tenant`（getTenant）、`GET /api/tenant/members`（listTenantMembers） |
| 対応ハンドラ | `tenant_handler_test.go` |
| テストID プレフィックス | `TNT-` |
| 認可ルール参照 | `authz.md` §6.7 テナント管理 |

### 認可ルールサマリー（authz.md §6.7 より）

| エンドポイント | メソッド | 許可ロール | 拒否ロール | 判定層 |
|--------------|---------|-----------|-----------|--------|
| `/api/tenant` | GET | Admin | Approver, Member, Accounting | RBAC MW |
| `/api/tenant/members` | GET | Admin, Accounting | Approver, Member | RBAC MW |

### 責務境界

- **このファイルに書くもの**: 各エンドポイントの RBAC テスト（ロール別 200/403 レスポンス）、正常系の レスポンス構造検証
- **このファイルに書かないもの**: テナント越境アクセステスト（`cross-cutting.md` に記載）

---

## フィクスチャ参照

`test_strategy.md` §4 の標準フィクスチャを使用する。

| 識別子 | ロール | ユーザーID | テナント |
|--------|--------|-----------|---------|
| `admin` | Admin | `aaaaaaaa-1111-1111-1111-000000000001` | テナントA |
| `approver` | Approver | `aaaaaaaa-2222-2222-2222-000000000002` | テナントA |
| `member` | Member | `aaaaaaaa-3333-3333-3333-000000000003` | テナントA |
| `accounting` | Accounting | `aaaaaaaa-4444-4444-4444-000000000004` | テナントA |

テナントA: `aaaaaaaa-0001-0001-0001-000000000001`（Test Company A）

---

## GET /api/tenant（getTenant）

### 正常系

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|---------|---------|-----------|-----------|---------------|-----------------|---------|
| TNT-001 | 統合 | handler | 正常系 | ADM-F01 | openapi.yaml#getTenant, db_schema.md#tenants | `TestGetTenant_Admin_OK` | 認証済み Admin（`aaaaaaaa-1111-1111-1111-000000000001`）で `GET /api/tenant` を実行 | 200 OK。レスポンス `data.id` = テナントA の UUID、`data.name` = `"Test Company A"`、`data.created_at` が RFC3339 形式の文字列であること |

### 異常系: 未認証

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|---------|---------|-----------|-----------|---------------|-----------------|---------|
| TNT-002 | 統合 | handler | 異常系 | ADM-F01 | openapi.yaml#getTenant | `TestGetTenant_Unauthenticated` | Authorization ヘッダーなしで `GET /api/tenant` を実行 | 401 UNAUTHORIZED |

### 異常系: RBAC（Admin 以外のロールは全て 403）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|---------|---------|-----------|-----------|---------------|-----------------|---------|
| TNT-003 | 統合 | handler | 認可 | ADM-F01, RBC-001 | openapi.yaml#getTenant, authz.md#3 | `TestGetTenant_Approver_Forbidden` | 認証済み Approver（`aaaaaaaa-2222-2222-2222-000000000002`）で `GET /api/tenant` を実行 | 403 FORBIDDEN。エラーコード `FORBIDDEN` |
| TNT-004 | 統合 | handler | 認可 | ADM-F01, RBC-001 | openapi.yaml#getTenant, authz.md#3 | `TestGetTenant_Member_Forbidden` | 認証済み Member（`aaaaaaaa-3333-3333-3333-000000000003`）で `GET /api/tenant` を実行 | 403 FORBIDDEN。エラーコード `FORBIDDEN` |
| TNT-005 | 統合 | handler | 認可 | ADM-F01, RBC-001 | openapi.yaml#getTenant, authz.md#3 | `TestGetTenant_Accounting_Forbidden` | 認証済み Accounting（`aaaaaaaa-4444-4444-4444-000000000004`）で `GET /api/tenant` を実行 | 403 FORBIDDEN。エラーコード `FORBIDDEN` |

---

## GET /api/tenant/members（listTenantMembers）

### 正常系

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|---------|---------|-----------|-----------|---------------|-----------------|---------|
| TNT-006 | 統合 | handler | 正常系 | ADM-F01 | openapi.yaml#listTenantMembers | `TestListTenantMembers_Admin_OK` | 認証済み Admin（`aaaaaaaa-1111-1111-1111-000000000001`）で `GET /api/tenant/members` を実行。テナントAに Admin, Approver, Member, Accounting, Member Empty の 5 ユーザーが登録済み | 200 OK。`data` が配列で、5 件のメンバーが返ること。各要素に `id`（UUID）と `name`（文字列）が含まれること（`UserSummary` スキーマ準拠） |
| TNT-007 | 統合 | handler | 正常系 | ADM-F01 | openapi.yaml#listTenantMembers | `TestListTenantMembers_Accounting_OK` | 認証済み Accounting（`aaaaaaaa-4444-4444-4444-000000000004`）で `GET /api/tenant/members` を実行 | 200 OK。`data` が配列で、テナントA の全メンバーが返ること |
| TNT-008 | 統合 | handler | テナント分離 | ADM-F01, TNT-F02 | openapi.yaml#listTenantMembers, db_schema.md#RLS | `TestListTenantMembers_ReturnsOwnTenantOnly` | 認証済み Admin（テナントA）で `GET /api/tenant/members` を実行。テナントBにも別ユーザーが登録済み | 200 OK。レスポンスに含まれるユーザーIDが全てテナントA のメンバーのみであること。テナントBのユーザー（`bbbbbbbb-3333-3333-3333-000000000003`）が含まれないこと |

### 異常系: 未認証

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|---------|---------|-----------|-----------|---------------|-----------------|---------|
| TNT-009 | 統合 | handler | 異常系 | ADM-F01 | openapi.yaml#listTenantMembers | `TestListTenantMembers_Unauthenticated` | Authorization ヘッダーなしで `GET /api/tenant/members` を実行 | 401 UNAUTHORIZED |

### 異常系: RBAC（Approver および Member は 403）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|---------|---------|-----------|-----------|---------------|-----------------|---------|
| TNT-010 | 統合 | handler | 認可 | ADM-F01, RBC-001 | openapi.yaml#listTenantMembers, authz.md#3 | `TestListTenantMembers_Approver_Forbidden` | 認証済み Approver（`aaaaaaaa-2222-2222-2222-000000000002`）で `GET /api/tenant/members` を実行 | 403 FORBIDDEN。エラーコード `FORBIDDEN` |
| TNT-011 | 統合 | handler | 認可 | ADM-F01, RBC-001 | openapi.yaml#listTenantMembers, authz.md#3 | `TestListTenantMembers_Member_Forbidden` | 認証済み Member（`aaaaaaaa-3333-3333-3333-000000000003`）で `GET /api/tenant/members` を実行 | 403 FORBIDDEN。エラーコード `FORBIDDEN` |

---

## GET /api/reports/all（listAllReports）— ページネーション

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|---------|---------|-----------|-----------|---------------|-----------------|---------|
| TNT-012 | 統合 | handler | 正常系 | RPT-F07, NFR-PERF-004 | openapi.yaml#listAllReports | `TestListAllReports_Pagination` | Actor: Test Admin（テナントA）。テナントA に複数ユーザーのレポートが 2 件以上存在する状態で `GET /api/reports/all?per_page=1` を実行 | 200 OK。`data` の件数が 1。`pagination.total_pages >= 2`、`pagination.per_page == 1`（per_page 動作の保証。WFL-005 / RPT-091 と同パターン） |

> 本ファイルでは `TNT-` プレフィックスを使用するため、`/api/reports/all` の per_page 統合テストもここに採番する（reports.md の RPT-091 が `/api/reports`（自分のレポート）対応のため、`/api/reports/all` は本ファイルの担当とする。RPT-020〜026 の listAllReports 採番と整合する位置付け）。

---

## 実装ガイド（tenant_handler_test.go 向け）

### テストファイル配置

```
expense-saas/internal/handler/tenant_handler_test.go
```

### テスト構造の雛形

```go
//go:build integration

package handler_test

import (
    "net/http"
    "net/http/httptest"
    "testing"
)

// TNT-001: Admin で GET /api/tenant → 200
func TestGetTenant_Admin_OK(t *testing.T) {
    // 前提: テナントA の Admin JWT を発行
    // 実行: GET /api/tenant
    // 検証: 200 OK、data.id == テナントA UUID、data.name == "Test Company A"
}

// TNT-003: Approver で GET /api/tenant → 403
func TestGetTenant_Approver_Forbidden(t *testing.T) {
    // 前提: テナントA の Approver JWT を発行
    // 実行: GET /api/tenant
    // 検証: 403 FORBIDDEN
}

// TNT-006: Admin で GET /api/tenant/members → 200
func TestListTenantMembers_Admin_OK(t *testing.T) {
    // 前提: テナントA に 5 ユーザー登録済み（Admin, Approver, Member, Accounting, Member Empty）
    // 実行: GET /api/tenant/members（Admin JWT）
    // 検証: 200 OK、data が 5 件、各要素に id と name が存在
}
```

### フィクスチャ投入

`test_strategy.md` §9.3 の「テスト前 TRUNCATE + フィクスチャ再投入」方針に従い、`TestMain` の `setup()` 内で標準フィクスチャを投入する。

テナント分離検証（TNT-008）では、`TestMain` または サブテストのセットアップ段階でテナントBのユーザー（`bbbbbbbb-3333-3333-3333-000000000003`）を別途投入する。テナントAのフィクスチャIDとテナントBのフィクスチャIDが混在しないよう注意すること（`test_strategy.md` §4.1 参照）。

### レスポンス検証ポイント

#### getTenant（TNT-001）

- `data.id`: テナントA の UUID（`aaaaaaaa-0001-0001-0001-000000000001`）と一致
- `data.name`: `"Test Company A"` と一致
- `data.created_at`: RFC3339 形式の文字列であること
- `TenantInfo` スキーマの required フィールド（`id`, `name`, `created_at`）が全て存在すること

#### listTenantMembers（TNT-006, TNT-007）

- `data`: 配列であること
- 各要素が `UserSummary` スキーマ（`id` UUID, `name` 文字列）を満たすこと
- テナントAに登録された全メンバー件数と一致すること
- テナントBのユーザーが混入していないこと（TNT-008）

### 403 レスポンスの検証ポイント

RBAC ミドルウェアが返す 403 は以下の形式であること（`authz.md` §5.2 準拠）:

```json
{
  "error": {
    "code": "FORBIDDEN",
    "message": "Insufficient permissions"
  }
}
```

---

## FE テストケース

### この文書の FE テスト範囲

| 項目 | 内容 |
|------|------|
| 対象画面 | テナント情報（SCR-ADM-002）、テナント全レポート一覧（SCR-ADM-001） |
| 対応コンポーネント設計 | `55_ui_component/screens/admin-tenant.md`, `55_ui_component/screens/admin-all-reports.md` |
| 対応状態管理 | `55_ui_component/state-management.md`（useTenant, useTenantMembers, useAllReports） |
| テストID プレフィックス | `TNT-FE-` |

### 責務境界

- **このファイルに書くもの**: テナント管理系画面（テナント情報・テナント全レポート一覧）のコンポーネント単体テスト、Hook 単体テスト
- **このファイルに書かないもの**: 共通コンポーネント（AppLayout, AppDataGrid, StatusChip 等）自体のテスト（共通コンポーネントテストケースに記載）、テナント越境アクセスの E2E テスト（cross-cutting.md に記載）

---

### テナント情報画面（SCR-ADM-002）

#### TenantPage

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|------------------|------------------|---------|-----------|-----------|---------------|-----------------|---------|
| TNT-FE-001 | 単体 | TenantPage | useTenant | 正常系 | ADM-F01 | 55_ui_component/screens/admin-tenant.md §TenantPage | `test_TenantPage_renders_tenant_info_on_success` | useTenant が `{ data: { id: "...", name: "Test Company A", created_at: "..." } }` を返すようモック。useCurrentUser が Admin ロールを返すようモック | TenantInfoCard が描画され、会社名「Test Company A」が表示されること。PhaseNotice が表示されること |
| TNT-FE-002 | 単体 | TenantPage | useTenant | 認可 | ADM-F01, RBC-001 | 55_ui_component/screens/admin-tenant.md §TenantPage | `test_TenantPage_redirects_non_admin_to_dashboard` | useCurrentUser が Approver ロールを返すようモック | ダッシュボード（`/`）にリダイレクトされること |
| TNT-FE-003 | 単体 | TenantPage | useTenant | 認可 | ADM-F01, RBC-001 | 55_ui_component/screens/admin-tenant.md §TenantPage | `test_TenantPage_redirects_member_to_dashboard` | useCurrentUser が Member ロールを返すようモック | ダッシュボード（`/`）にリダイレクトされること |
| TNT-FE-004 | 単体 | TenantPage | useTenant | 認可 | ADM-F01, RBC-001 | 55_ui_component/screens/admin-tenant.md §TenantPage | `test_TenantPage_redirects_accounting_to_dashboard` | useCurrentUser が Accounting ロールを返すようモック | ダッシュボード（`/`）にリダイレクトされること |
| TNT-FE-005 | 単体 | TenantPage | useTenant | 異常系 | ADM-F01 | 55_ui_component/screens/admin-tenant.md §TenantPage | `test_TenantPage_redirects_on_403_error` | useTenant が 403 エラーを返すようモック。useCurrentUser が Admin ロールを返すようモック | ダッシュボード（`/`）にリダイレクトされること |
| TNT-FE-006 | 単体 | TenantPage | useTenant | 異常系 | ADM-F01 | 55_ui_component/screens/admin-tenant.md §TenantPage | `test_TenantPage_shows_toast_on_500_error` | useTenant が 500 エラーを返すようモック。useCurrentUser が Admin ロールを返すようモック | AppToast（SnackbarContext）にエラーメッセージが通知されること |
| TNT-FE-007 | 単体 | TenantPage | useTenant | 正常系 | ADM-F01 | 55_ui_component/screens/admin-tenant.md §TenantPage | `test_TenantPage_shows_loading_skeleton` | useTenant が isLoading = true を返すようモック。useCurrentUser が Admin ロールを返すようモック | PageSkeleton（variant="card"）が表示されること |

#### TenantInfoCard

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|------------------|------------------|---------|-----------|-----------|---------------|-----------------|---------|
| TNT-FE-008 | 単体 | TenantInfoCard | tenant: TenantInfo, loading: boolean, error: ApiClientError \| null | 正常系 | ADM-F01 | 55_ui_component/screens/admin-tenant.md §TenantInfoCard | `test_TenantInfoCard_renders_company_name` | `tenant = { id: "...", name: "Test Company A", created_at: "..." }`, `loading = false`, `error = null` | TenantInfoField がラベル「会社名」、値「Test Company A」で描画されること |

**備考**: TenantPage 側のロール不一致リダイレクトトーストテストは、TenantInfoCard 側の TNT-FE-008 と ID が重複するため TNT-FE-044 に振り直した。TenantInfoCard 側の TNT-FE-008（本行）はそのまま維持する。
| TNT-FE-009 | 単体 | TenantInfoCard | loading: boolean | 正常系 | ADM-F01 | 55_ui_component/screens/admin-tenant.md §TenantInfoCard | `test_TenantInfoCard_shows_skeleton_when_loading` | `tenant = undefined`, `loading = true`, `error = null` | PageSkeleton（variant="card"）が描画されること。TenantInfoField が描画されないこと |

**備考**: TenantPage 側の 403 リダイレクトトーストテストは、TenantInfoCard 側の TNT-FE-009 と ID が重複するため TNT-FE-045 に振り直した。TenantInfoCard 側の TNT-FE-009（本行）はそのまま維持する。
| TNT-FE-010 | 単体 | TenantInfoCard | error: ApiClientError \| null | 異常系 | ADM-F01 | 55_ui_component/screens/admin-tenant.md §TenantInfoCard | `test_TenantInfoCard_shows_error_message_on_404` | `tenant = undefined`, `loading = false`, `error = { status: 404, ... }` | 「テナント情報が見つかりません。」というエラーメッセージが表示されること |

#### TenantInfoField

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|------------------|------------------|---------|-----------|-----------|---------------|-----------------|---------|
| TNT-FE-011 | 単体 | TenantInfoField | label: string, value: string | 正常系 | ADM-F01 | 55_ui_component/screens/admin-tenant.md §TenantInfoField | `test_TenantInfoField_renders_label_and_value` | `label = "会社名"`, `value = "Test Company A"` | ラベル「会社名」と値「Test Company A」がそれぞれ描画されること |

#### PhaseNotice

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|------------------|------------------|---------|-----------|-----------|---------------|-----------------|---------|
| TNT-FE-012 | 単体 | PhaseNotice | message: string | 正常系 | ADM-F01 | 55_ui_component/screens/admin-tenant.md §PhaseNotice | `test_PhaseNotice_renders_message` | `message = "テナント情報の編集機能は今後追加予定です。"` | 指定されたメッセージテキストが描画されること |

#### PageTitle（テナント情報）

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|------------------|------------------|---------|-----------|-----------|---------------|-----------------|---------|
| TNT-FE-042 | 単体 | PageTitle | title: string | 正常系 | ADM-F01 | 55_ui_component/screens/admin-tenant.md §PageTitle | `test_PageTitle_renders_tenant_title` | `title = "テナント情報"` | 「テナント情報」がページタイトルとして描画されること |

#### useTenant Hook

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|------------------|------------------|---------|-----------|-----------|---------------|-----------------|---------|
| TNT-FE-013 | 単体 | useTenant | useTenant | 正常系 | ADM-F01 | 55_ui_component/state-management.md §useTenant | `test_useTenant_fetches_tenant_info` | MSW で `GET /api/tenant` が `{ data: { id: "...", name: "Test Company A", created_at: "..." } }` を返すよう設定 | Hook の `data.data.id` がテナント UUID と一致すること。`data.data.name` が `"Test Company A"` であること |
| TNT-FE-014 | 単体 | useTenant | useTenant | 正常系 | ADM-F01 | 55_ui_component/state-management.md §useTenant | `test_useTenant_caches_with_stale_time_5min` | MSW で `GET /api/tenant` を設定。Hook を2回レンダリングする | 2回目のレンダリングでは API が再呼び出しされないこと（staleTime 5分のキャッシュが有効） |
| TNT-FE-015 | 単体 | useTenant | useTenant | 異常系 | ADM-F01 | 55_ui_component/state-management.md §useTenant | `test_useTenant_returns_error_on_api_failure` | MSW で `GET /api/tenant` が 500 を返すよう設定 | Hook の `error` が非 null であること。`isError` が true であること |

---

### テナント全レポート一覧画面（SCR-ADM-001）

#### AllReportsPage

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|------------------|------------------|---------|-----------|-----------|---------------|-----------------|---------|
| TNT-FE-016 | 単体 | AllReportsPage | useAllReports, useTenantMembers | 正常系 | ADM-F01 | 55_ui_component/screens/admin-all-reports.md §AllReportsPage | `test_AllReportsPage_renders_report_list_for_admin` | useAllReports が レポート2件を返すようモック。useTenantMembers がメンバー4件を返すようモック。useCurrentUser が Admin ロールを返すようモック | AllReportsFilterBar と AllReportsTable が描画されること。レポート2件がテーブルに表示されること |
| TNT-FE-017 | 単体 | AllReportsPage | useAllReports, useTenantMembers | 正常系 | ADM-F01 | 55_ui_component/screens/admin-all-reports.md §AllReportsPage | `test_AllReportsPage_renders_report_list_for_accounting` | useAllReports がレポート2件を返すようモック。useTenantMembers がメンバー4件を返すようモック。useCurrentUser が Accounting ロールを返すようモック | AllReportsFilterBar と AllReportsTable が描画されること。Admin と同一の画面構成・表示内容であること |
| TNT-FE-018 | 単体 | AllReportsPage | useCurrentUser | 認可 | ADM-F01, RBC-001 | 55_ui_component/screens/admin-all-reports.md §AllReportsPage | `test_AllReportsPage_redirects_approver_to_dashboard` | useCurrentUser が Approver ロールを返すようモック | ダッシュボード（`/`）にリダイレクトされること |
| TNT-FE-019 | 単体 | AllReportsPage | useCurrentUser | 認可 | ADM-F01, RBC-001 | 55_ui_component/screens/admin-all-reports.md §AllReportsPage | `test_AllReportsPage_redirects_member_to_dashboard` | useCurrentUser が Member ロールを返すようモック | ダッシュボード（`/`）にリダイレクトされること |
| TNT-FE-020 | 単体 | AllReportsPage | useAllReports | 異常系 | ADM-F01 | 55_ui_component/screens/admin-all-reports.md §AllReportsPage | `test_AllReportsPage_redirects_on_403_error` | useAllReports が 403 エラーを返すようモック。useCurrentUser が Admin ロールを返すようモック | ダッシュボード（`/`）にリダイレクトされること |
| TNT-FE-021 | 単体 | AllReportsPage | useAllReports | 異常系 | ADM-F01 | 55_ui_component/screens/admin-all-reports.md §AllReportsPage | `test_AllReportsPage_shows_toast_on_500_error` | useAllReports が 500 エラーを返すようモック。useCurrentUser が Admin ロールを返すようモック | AppToast（SnackbarContext）にエラーメッセージが通知されること |
| TNT-FE-022 | 単体 | AllReportsPage | useAllReports | 正常系 | ADM-F01 | 55_ui_component/screens/admin-all-reports.md §AllReportsPage | `test_AllReportsPage_resets_page_on_filter_change` | useAllReports がレポートを返すようモック。現在のページを 2 に設定した状態でフィルタ条件を変更する | ページ番号が 1 にリセットされること。useAllReports が page=1 で再呼び出しされること |
| TNT-FE-023 | 単体 | AllReportsPage | useAllReports | 正常系 | ADM-F01 | 55_ui_component/screens/admin-all-reports.md §AllReportsPage | `test_AllReportsPage_navigates_to_report_detail_on_row_click` | useAllReports がレポート（id="rpt-001"）を返すようモック。テーブル行をクリックする | レポート詳細画面（`/reports/rpt-001`）に遷移すること |

#### AllReportsFilterBar

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|------------------|------------------|---------|-----------|-----------|---------------|-----------------|---------|
| TNT-FE-024 | 単体 | AllReportsFilterBar | filters: AllReportsFilterValues, onFilterChange, members: UserSummary[], membersLoading: boolean | 正常系 | ADM-F01 | 55_ui_component/screens/admin-all-reports.md §AllReportsFilterBar | `test_AllReportsFilterBar_renders_all_filters` | `filters = { status: "", from: "", to: "", submitterId: "" }`, `members = [{ id: "u1", name: "User1" }]`, `membersLoading = false` | ステータスセレクト、期間（開始日・終了日）の DatePicker、申請者セレクトの 4 つのフィルタ入力が描画されること |

**備考**: AllReportsPage 側のロール不一致リダイレクトトーストテストは、AllReportsFilterBar 側の TNT-FE-024 と ID が重複するため TNT-FE-046 に振り直した。AllReportsFilterBar 側の TNT-FE-024（本行）はそのまま維持する。
| TNT-FE-025 | 単体 | AllReportsFilterBar | onFilterChange | 正常系 | ADM-F01 | 55_ui_component/screens/admin-all-reports.md §AllReportsFilterBar | `test_AllReportsFilterBar_calls_onFilterChange_on_status_change` | ステータスセレクトで「提出済み」を選択する | `onFilterChange` が `{ status: "submitted", from: "", to: "", submitterId: "" }` で呼び出されること |

**備考**: AllReportsPage 側の 403 リダイレクトトーストテストは、AllReportsFilterBar 側の TNT-FE-025 と ID が重複するため TNT-FE-047 に振り直した。AllReportsFilterBar 側の TNT-FE-025（本行）はそのまま維持する。
| TNT-FE-026 | 単体 | AllReportsFilterBar | onFilterChange | 正常系 | ADM-F01 | 55_ui_component/screens/admin-all-reports.md §AllReportsFilterBar | `test_AllReportsFilterBar_calls_onFilterChange_on_submitter_change` | 申請者セレクトでメンバーを選択する | `onFilterChange` が `submitterId` に選択したメンバーの ID が設定された値で呼び出されること |
| TNT-FE-027 | 単体 | AllReportsFilterBar | onFilterChange | 正常系 | ADM-F01 | 55_ui_component/screens/admin-all-reports.md §AllReportsFilterBar | `test_AllReportsFilterBar_calls_onFilterChange_on_date_change` | 開始日 DatePicker で「2025-01-01」を入力する | `onFilterChange` が `from: "2025-01-01"` を含む値で呼び出されること |
| TNT-FE-028 | 単体 | AllReportsFilterBar | filters: AllReportsFilterValues | 異常系 | ADM-F01 | 55_ui_component/screens/admin-all-reports.md §AllReportsFilterBar | `test_AllReportsFilterBar_shows_date_validation_error` | `filters = { status: "", from: "2025-12-31", to: "2025-01-01", submitterId: "" }` | 終了日の DatePicker にバリデーションエラー（開始日が終了日より後）が表示されること |
| TNT-FE-029 | 単体 | AllReportsFilterBar | membersLoading: boolean | 正常系 | ADM-F01 | 55_ui_component/screens/admin-all-reports.md §AllReportsFilterBar | `test_AllReportsFilterBar_disables_submitter_select_while_loading` | `members = []`, `membersLoading = true` | 申請者セレクトがローディング状態（disabled またはローディングインジケータ表示）であること |

#### AllReportsTable

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|------------------|------------------|---------|-----------|-----------|---------------|-----------------|---------|
| TNT-FE-030 | 単体 | AllReportsTable | reports: AllReportRow[], loading: boolean, hasActiveFilters: boolean, onRowClick | 正常系 | ADM-F01 | 55_ui_component/screens/admin-all-reports.md §AllReportsTable | `test_AllReportsTable_renders_report_rows` | `reports = [{ id: "rpt-1", title: "出張費", submitter: { id: "u1", name: "User1" }, totalAmount: 10000, status: "submitted", submittedAt: "2025-01-15T...", createdAt: "2025-01-10T..." }]`, `loading = false`, `hasActiveFilters = false` | テーブルに申請者名「User1」、タイトル「出張費」、合計金額「10,000」、ステータス「提出済み」、提出日が表示されること |
| TNT-FE-031 | 単体 | AllReportsTable | loading: boolean | 正常系 | ADM-F01 | 55_ui_component/screens/admin-all-reports.md §AllReportsTable | `test_AllReportsTable_shows_skeleton_when_loading` | `reports = []`, `loading = true`, `hasActiveFilters = false` | PageSkeleton（variant="table"）が描画されること。テーブル行が表示されないこと |
| TNT-FE-032 | 単体 | AllReportsTable | reports: AllReportRow[], hasActiveFilters: boolean | 正常系 | ADM-F01 | 55_ui_component/screens/admin-all-reports.md §AllReportsTable | `test_AllReportsTable_shows_empty_state_no_filters` | `reports = []`, `loading = false`, `hasActiveFilters = false` | EmptyState が「レポートはまだ作成されていません。」のメッセージで表示されること |
| TNT-FE-033 | 単体 | AllReportsTable | reports: AllReportRow[], hasActiveFilters: boolean | 正常系 | ADM-F01 | 55_ui_component/screens/admin-all-reports.md §AllReportsTable | `test_AllReportsTable_shows_empty_state_with_filters` | `reports = []`, `loading = false`, `hasActiveFilters = true` | EmptyState が「条件に一致するレポートはありません。フィルタを変更してお試しください。」のメッセージで表示されること |
| TNT-FE-034 | 単体 | AllReportsTable | onRowClick | 正常系 | ADM-F01 | 55_ui_component/screens/admin-all-reports.md §AllReportsTable | `test_AllReportsTable_calls_onRowClick_with_report_id` | `reports` に id="rpt-1" のレポートを含む。テーブル行をクリックする | `onRowClick` が `"rpt-1"` で呼び出されること |

#### useAllReports Hook

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|------------------|------------------|---------|-----------|-----------|---------------|-----------------|---------|
| TNT-FE-035 | 単体 | useAllReports | useAllReports | 正常系 | ADM-F01 | 55_ui_component/state-management.md §useAllReports | `test_useAllReports_fetches_reports_with_default_params` | MSW で `GET /api/reports/all` がレポート一覧を返すよう設定。パラメータ指定なしで Hook を呼び出す | Hook の `data.data` がレポート配列であること。API が `page=1` で呼び出されること |
| TNT-FE-036 | 単体 | useAllReports | useAllReports | 正常系 | ADM-F01 | 55_ui_component/state-management.md §useAllReports | `test_useAllReports_sends_filter_params` | MSW で `GET /api/reports/all` を設定。`{ status: "submitted", from: "2025-01-01", to: "2025-01-31", submitter_id: "u1" }` のパラメータで Hook を呼び出す | API が `status=submitted&from=2025-01-01&to=2025-01-31&submitter_id=u1` のクエリパラメータ付きで呼び出されること |
| TNT-FE-037 | 単体 | useAllReports | useAllReports | 正常系 | ADM-F01 | 55_ui_component/state-management.md §useAllReports | `test_useAllReports_sends_pagination_params` | MSW で `GET /api/reports/all` を設定。`{ page: 3, per_page: 20 }` のパラメータで Hook を呼び出す | API が `page=3&per_page=20` のクエリパラメータ付きで呼び出されること |
| TNT-FE-038 | 単体 | useAllReports | useAllReports | 異常系 | ADM-F01 | 55_ui_component/state-management.md §useAllReports | `test_useAllReports_returns_error_on_api_failure` | MSW で `GET /api/reports/all` が 500 を返すよう設定 | Hook の `error` が非 null であること。`isError` が true であること |

#### useTenantMembers Hook

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|------------------|------------------|---------|-----------|-----------|---------------|-----------------|---------|
| TNT-FE-039 | 単体 | useTenantMembers | useTenantMembers | 正常系 | ADM-F01 | 55_ui_component/state-management.md §useTenantMembers | `test_useTenantMembers_fetches_member_list` | MSW で `GET /api/tenant/members` が `{ data: [{ id: "u1", name: "User1" }, { id: "u2", name: "User2" }] }` を返すよう設定 | Hook の `data.data` が UserSummary 配列で2件であること。各要素に `id` と `name` が含まれること |
| TNT-FE-040 | 単体 | useTenantMembers | useTenantMembers | 正常系 | ADM-F01 | 55_ui_component/state-management.md §useTenantMembers | `test_useTenantMembers_caches_with_stale_time_60s` | MSW で `GET /api/tenant/members` を設定。Hook を2回レンダリングする | 2回目のレンダリングでは API が再呼び出しされないこと（staleTime 60秒のキャッシュが有効） |
| TNT-FE-041 | 単体 | useTenantMembers | useTenantMembers | 異常系 | ADM-F01 | 55_ui_component/state-management.md §useTenantMembers | `test_useTenantMembers_returns_error_on_api_failure` | MSW で `GET /api/tenant/members` が 500 を返すよう設定 | Hook の `error` が非 null であること。`isError` が true であること |

#### PageTitle（全レポート一覧）

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|------------------|------------------|---------|-----------|-----------|---------------|-----------------|---------|
| TNT-FE-043 | 単体 | PageTitle | title: string | 正常系 | ADM-F01 | 55_ui_component/screens/admin-all-reports.md §PageTitle | `test_PageTitle_renders_all_reports_title` | `title = "全レポート一覧"` | 「全レポート一覧」がページタイトルとして描画されること |

---

### 追記テスト（リダイレクトトースト）

以下のテスト ID は重複解消のために追記する。

#### TenantPage -- リダイレクトトースト（旧 TNT-FE-008/009 → 振り直し）

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|------------------|------------------|---------|-----------|-----------|---------------|-----------------|---------|
| TNT-FE-044 | 単体 | TenantPage | useCurrentUser | 認可 | ADM-F01, RBC-001 | 55_ui_component/screens/admin-tenant.md §TenantPage | `test_TenantPage_redirects_non_admin_shows_toast` | useCurrentUser が Member ロールを返すようモック | ダッシュボード（`/`）にリダイレクトされ、navigate の state.toast にトーストメッセージ「この画面にアクセスする権限がありません。」が含まれること |
| TNT-FE-045 | 単体 | TenantPage | useTenant | 認可 | ADM-F01 | 55_ui_component/screens/admin-tenant.md §TenantPage | `test_TenantPage_redirects_on_403_shows_toast` | useTenant が 403 エラーを返すようモック。useCurrentUser が Admin ロールを返すようモック | ダッシュボード（`/`）にリダイレクトされ、navigate の state.toast にトーストメッセージが含まれること |

**備考**: TNT-FE-044/045 は旧 TNT-FE-008/009（TenantPage 側の重複 ID）を振り直したもの。TenantInfoCard 側の TNT-FE-008/009 はそのまま維持する。

#### AllReportsPage -- リダイレクトトースト（旧 TNT-FE-024/025 → 振り直し）

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|------------------|------------------|---------|-----------|-----------|---------------|-----------------|---------|
| TNT-FE-046 | 単体 | AllReportsPage | useCurrentUser | 認可 | ADM-F01, RBC-001 | 55_ui_component/screens/admin-all-reports.md §AllReportsPage | `test_AllReportsPage_redirects_non_admin_shows_toast` | useCurrentUser が Member ロールを返すようモック | ダッシュボード（`/`）にリダイレクトされ、navigate の state.toast にトーストメッセージ「この画面にアクセスする権限がありません。」が含まれること |
| TNT-FE-047 | 単体 | AllReportsPage | useAllReports | 認可 | ADM-F01 | 55_ui_component/screens/admin-all-reports.md §AllReportsPage | `test_AllReportsPage_redirects_on_403_shows_toast` | useAllReports が 403 エラーを返すようモック。useCurrentUser が Admin ロールを返すようモック | ダッシュボード（`/`）にリダイレクトされ、navigate の state.toast にトーストメッセージが含まれること |

**備考**: TNT-FE-046/047 は旧 TNT-FE-024/025（AllReportsPage 側の重複 ID）を振り直したもの。AllReportsFilterBar 側の TNT-FE-024/025 はそのまま維持する。

---

### 追記テスト（AllReportsPage URL 駆動化 + per_page UI 結合）

`AllReportsPage`（SCR-ADM-001）における URL 駆動化（`useState` → `useSearchParams`）と per_page UI セレクタ結合経路を Page 結合テストとして採番する。`PageSizeSelector` / `AppPaginationFooter` の単体テストは `reports.md` §FE-6 の `PSS-001〜005` / `APF-001〜007` に集約済み。

採番方針: 既存 TNT-FE 系最大 ID は `TNT-FE-047`。次番 `TNT-FE-048` から採用する。

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|------------------|------------------|---------|-----------|-----------|---------------|-----------------|---------|
| TNT-FE-048 | 単体 | AllReportsPage | useAllReports（MSW）、useSearchParams | 正常系 | RPT-F07 | 50_detail_design/screens/admin-all-reports.md §6, 55_ui_component/state-management.md §3.1 | `test_AllReportsPage_url_per_page_reflects_to_selector_and_api` | `/reports/all?per_page=10` で開く。MSW で `useAllReports` が 10 件 + `pagination.total_pages > 1` を返すよう設定。useCurrentUser が Admin ロールを返すようモック | テーブルに 10 件のみ描画される。フッターの `PageSizeSelector` が「10」を表示する。`useAllReports` への引数に `per_page: 10` が渡り、API URL に `?per_page=10` が含まれる |
| TNT-FE-049 | 単体 | AllReportsPage | useAllReports（MSW）、useSearchParams | 正常系 | RPT-F07 | 50_detail_design/screens/admin-all-reports.md §6, 55_ui_component/state-management.md §3.1 | `test_AllReportsPage_selector_change_updates_url_and_resets_page` | `/reports/all?page=3&per_page=10` で開いた状態で `PageSizeSelector` から「50」を選択 | URL が `/reports/all?page=1&per_page=50` に更新される（`page=1` リセット）。`setSearchParams` は 1 回のコールに集約される（race 回避）。`PageSizeSelector` の現在値が「50」に更新される |
| TNT-FE-050 | 単体 | AllReportsPage | useSearchParams（URL 駆動化） | 正常系 | RPT-F07 | 50_detail_design/screens/admin-all-reports.md §6（URL 駆動化）, 55_ui_component/state-management.md §3.1 | `test_AllReportsPage_page_state_is_url_driven` | `/reports/all?page=2` で開く。useCurrentUser が Admin ロールを返すようモック | `useAllReports` への引数に `page: 2` が渡る（`useState` ではなく `useSearchParams` 経由で URL から読み取られていることを保証）。ページネーションコントロールで「3」をクリックすると URL が `/reports/all?page=3` に更新される（`useState` 由来の独立 state ではない） |
| TNT-FE-051 | 単体 | AllReportsPage | useAllReports、useSearchParams | 正常系 | RPT-F07 | 50_detail_design/screens/admin-all-reports.md §6, 55_ui_component/state-management.md §3.1 | `test_AllReportsPage_url_invalid_per_page_falls_back_to_20` | `/reports/all?per_page=abc`（NaN ケース） / `?per_page=-5`（負数ケース） の 2 サブケース。useCurrentUser が Admin ロールを返すようモック | 両サブケースとも `useAllReports` への引数 `per_page` が `20`（FE フォールバック）になり、`PageSizeSelector` も「20」を表示する |

**備考**:
- 採番根拠: `TNT-FE-047` の次番から開始。
- 実装ファイル候補: `expense-saas/frontend/src/pages/admin/__tests__/AllReportsPage.test.tsx`（**既存ファイル拡張**。実体ファイルは既に存在するため、新規作成ではなく既存ファイルへ TNT-FE-048〜051 のテストケースを追記する形となる）。
- TNT-FE-048〜051 は `AllReportsPage` 単独の経路を検証する。動的選択肢の単体検証は `PSS-002 / PSS-003` に集約済みのため、本節では再検証しない。
- TNT-FE-022（既存: `resets_page_on_filter_change`）は per_page 変更ではなくフィルタ変更時の page リセット検証。追加される TNT-FE-049 は **per_page 変更時の page リセット**を検証する別ケース。

---

#### FE 追記テスト（中間ラッパー paginationFooter prop + 空状態フッター常時表示）

中間ラッパー（`AllReportsTable`）に `paginationFooter?: ReactNode` prop を追加し、`AppDataGrid` の `slots.footer` 経由で DataGrid フッターコンテナに統合する設計を採用した。空状態時もフッターを常時表示する仕様の回帰防止検証を含む。

採番方針: 既存 TNT-FE 系最大 ID は `TNT-FE-051`。次番 `TNT-FE-052` から採用する。

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|------------------|------------------|---------|-----------|-----------|---------------|-----------------|---------|
| TNT-FE-052 | 単体 | AllReportsTable | `paginationFooter?: ReactNode` prop | 正常系 | RPT-F07 | 55_ui_component/common-components.md §AppPaginationFooter 配置・利用前提, 50_detail_design/screens/admin-all-reports.md §6 | `test_AllReportsTable_paginationFooter_prop_renders_in_datagrid_footer_slot` | reports に複数件のデータを渡し、`paginationFooter={<div data-testid="custom-footer">FOOTER</div>}` を渡す | AppDataGrid の `slots.footer` として `paginationFooter` で渡した要素が DataGrid フッターコンテナ内に描画される（`data-testid="custom-footer"` で取得可能） |
| TNT-FE-053 | 単体 | AllReportsTable | `paginationFooter?: ReactNode` prop, 空状態 | 正常系 | RPT-F07 | 55_ui_component/common-components.md §AppPaginationFooter L360 常時表示 | `test_AllReportsTable_paginationFooter_renders_even_when_empty` | reports = [] かつ `paginationFooter={<div data-testid="custom-footer">FOOTER</div>}` を渡す | 空状態でも AppDataGrid が描画され、`slots.noRowsOverlay` 経由で EmptyState が表示されつつ、`slots.footer` の `paginationFooter` も同時に表示される（リグレッション防止） |

**備考**:
- 採番根拠: `TNT-FE-051` の次番から開始。
- 実装ファイル: `expense-saas/frontend/src/pages/admin/__tests__/AllReportsTable.test.tsx`（既存ファイル拡張）。
- 空状態の AppDataGrid 描画は `slots.noRowsOverlay` に EmptyState を渡す形に変更（中間ラッパーの早期 return 撤去）。既存 TNT-FE-032 / TNT-FE-033 も本対応で AppDataGrid 描画 + noRowsOverlay 経由 EmptyState 表示を期待する形に更新。

---

### FE テスト実装ガイド

#### テストファイル配置

```
expense-saas/src/pages/admin/__tests__/TenantPage.test.tsx
expense-saas/src/pages/admin/__tests__/TenantInfoCard.test.tsx
expense-saas/src/pages/admin/__tests__/TenantInfoField.test.tsx
expense-saas/src/pages/admin/__tests__/PhaseNotice.test.tsx
expense-saas/src/pages/admin/__tests__/AllReportsPage.test.tsx
expense-saas/src/pages/admin/__tests__/AllReportsFilterBar.test.tsx
expense-saas/src/pages/admin/__tests__/AllReportsTable.test.tsx
expense-saas/src/hooks/__tests__/useTenant.test.tsx
```

#### テスト技術スタック

| ツール | 用途 |
|--------|------|
| Vitest | テストランナー |
| React Testing Library | コンポーネントレンダリング・DOM クエリ |
| MSW (Mock Service Worker) | API モック（Hook テスト用） |
| renderHook (React Testing Library) | Hook 単体テスト |

#### モックパターン

**Hook モック（コンポーネントテスト用）**:

```typescript
// vi.mock で Hook を差し替え、コンポーネントを単体テストする
vi.mock('@/hooks/useTenant', () => ({
  useTenant: vi.fn().mockReturnValue({
    data: { data: { id: '...', name: 'Test Company A', created_at: '...' } },
    isLoading: false,
    isError: false,
    error: null,
  }),
}));
```

**MSW ハンドラ（Hook テスト用）**:

```typescript
// Hook テストでは MSW で実際の API レスポンスをモックする
const handlers = [
  http.get('/api/tenant', () => {
    return HttpResponse.json({
      data: { id: '...', name: 'Test Company A', created_at: '...' },
    });
  }),
];
```

**React Router モック（リダイレクトテスト用）**:

```typescript
// navigate 関数のモックで画面遷移を検証する
const mockNavigate = vi.fn();
vi.mock('react-router-dom', async () => {
  const actual = await vi.importActual('react-router-dom');
  return { ...actual, useNavigate: () => mockNavigate };
});
```

#### フィクスチャ注意事項

- FE テストではテナント横断のデータ混在は発生しないが、MSW レスポンスに使用するテナント ID・ユーザー ID は BE テストの標準フィクスチャと一致させること（`test_strategy.md` §4 参照）
- テストデータに個人情報を含めない。会社名は「Test Company A」、ユーザー名は「User1」「User2」等の抽象化された値を使用する
