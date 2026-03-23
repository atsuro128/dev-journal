# テストケース: テナント管理（TNT-）

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

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|---------|---------------|-----------------|---------|
| TNT-001 | 統合 | handler | `TestGetTenant_Admin_OK` | 認証済み Admin（`aaaaaaaa-1111-1111-1111-000000000001`）で `GET /api/tenant` を実行 | 200 OK。レスポンス `data.id` = テナントA の UUID、`data.name` = `"Test Company A"`、`data.created_at` が RFC3339 形式の文字列であること |

### 異常系: 未認証

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|---------|---------------|-----------------|---------|
| TNT-002 | 統合 | handler | `TestGetTenant_Unauthenticated` | Authorization ヘッダーなしで `GET /api/tenant` を実行 | 401 UNAUTHORIZED |

### 異常系: RBAC（Admin 以外のロールは全て 403）

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|---------|---------------|-----------------|---------|
| TNT-003 | 統合 | handler | `TestGetTenant_Approver_Forbidden` | 認証済み Approver（`aaaaaaaa-2222-2222-2222-000000000002`）で `GET /api/tenant` を実行 | 403 FORBIDDEN。エラーコード `FORBIDDEN` |
| TNT-004 | 統合 | handler | `TestGetTenant_Member_Forbidden` | 認証済み Member（`aaaaaaaa-3333-3333-3333-000000000003`）で `GET /api/tenant` を実行 | 403 FORBIDDEN。エラーコード `FORBIDDEN` |
| TNT-005 | 統合 | handler | `TestGetTenant_Accounting_Forbidden` | 認証済み Accounting（`aaaaaaaa-4444-4444-4444-000000000004`）で `GET /api/tenant` を実行 | 403 FORBIDDEN。エラーコード `FORBIDDEN` |

---

## GET /api/tenant/members（listTenantMembers）

### 正常系

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|---------|---------------|-----------------|---------|
| TNT-006 | 統合 | handler | `TestListTenantMembers_Admin_OK` | 認証済み Admin（`aaaaaaaa-1111-1111-1111-000000000001`）で `GET /api/tenant/members` を実行。テナントAに Admin, Approver, Member, Accounting の 4 ユーザーが登録済み | 200 OK。`data` が配列で、4 件のメンバーが返ること。各要素に `id`（UUID）と `name`（文字列）が含まれること（`UserSummary` スキーマ準拠） |
| TNT-007 | 統合 | handler | `TestListTenantMembers_Accounting_OK` | 認証済み Accounting（`aaaaaaaa-4444-4444-4444-000000000004`）で `GET /api/tenant/members` を実行 | 200 OK。`data` が配列で、テナントA の全メンバーが返ること |
| TNT-008 | 統合 | handler | `TestListTenantMembers_ReturnsOwnTenantOnly` | 認証済み Admin（テナントA）で `GET /api/tenant/members` を実行。テナントBにも別ユーザーが登録済み | 200 OK。レスポンスに含まれるユーザーIDが全てテナントA のメンバーのみであること。テナントBのユーザー（`bbbbbbbb-3333-3333-3333-000000000003`）が含まれないこと |

### 異常系: 未認証

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|---------|---------------|-----------------|---------|
| TNT-009 | 統合 | handler | `TestListTenantMembers_Unauthenticated` | Authorization ヘッダーなしで `GET /api/tenant/members` を実行 | 401 UNAUTHORIZED |

### 異常系: RBAC（Approver および Member は 403）

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|---------|---------------|-----------------|---------|
| TNT-010 | 統合 | handler | `TestListTenantMembers_Approver_Forbidden` | 認証済み Approver（`aaaaaaaa-2222-2222-2222-000000000002`）で `GET /api/tenant/members` を実行 | 403 FORBIDDEN。エラーコード `FORBIDDEN` |
| TNT-011 | 統合 | handler | `TestListTenantMembers_Member_Forbidden` | 認証済み Member（`aaaaaaaa-3333-3333-3333-000000000003`）で `GET /api/tenant/members` を実行 | 403 FORBIDDEN。エラーコード `FORBIDDEN` |

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
    // 前提: テナントA に 4 ユーザー登録済み
    // 実行: GET /api/tenant/members（Admin JWT）
    // 検証: 200 OK、data が 4 件、各要素に id と name が存在
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
