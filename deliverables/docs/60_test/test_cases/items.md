# 経費明細テストケース一覧（items.md）

## この文書の役割

| 項目 | 内容 |
|------|------|
| 目的 | 経費明細 CRUD エンドポイント（/api/reports/{id}/items/*）のテストケースを定義する |
| 正本情報 | ITM-F01〜F03 に対応するテストケース一覧 |
| 扱わない内容 | レポート状態遷移テスト（→ reports.md / workflow.md）、テナント横断テスト（→ cross-cutting.md） |
| 主な参照元 | `50_detail_design/openapi.yaml#/api/reports/{id}/items/*`, `10_requirements/requirements.md#ITM-F*` |
| 主な参照先 | `60_test/traceability.md`, テスト実装（Step 9） |

---

**テストIDプレフィックス**: `ITM-`
**対象ハンドラファイル**: `item_handler_test.go`

> **テストID採番規則**: テストIDはエンドポイント（操作）単位でレンジを割り当てる。
> - `ITM-001〜099`: POST（明細追加）
> - `ITM-101〜199`: PUT（明細更新）
> - `ITM-201〜299`: DELETE（明細削除）
> - `ITM-301〜399`: RBACテスト（エンドポイント横断）
>
> 各レンジ内の未使用番号（例: ITM-007〜010、ITM-024〜030 等）は将来の拡張用に予約されており、欠番は意図的なものである。
**対応エンドポイント**:
- `POST /api/reports/{id}/items` — createItem
- `PUT /api/reports/{id}/items/{itemId}` — updateItem
- `DELETE /api/reports/{id}/items/{itemId}` — deleteItem

**責務境界**:
- テナント分離テスト（クロステナントアクセスの 404 検証）は `cross-cutting.md` に記載する。本ファイルには書かない。
- レポート状態遷移テスト（T1〜T5 の正常パス等）は `reports.md` / `workflow.md` に記載する。本ファイルには書かない。ただし「レポート状態による明細操作の制限」テストは本ファイルに記載する。

---

## フィクスチャ参照

`test_strategy.md` §4 で定義された標準フィクスチャを使用する。

| フィクスチャ | 状態 | レポートID | 明細ID |
|------------|------|----------|--------|
| `report_draft` | `draft` | `cccccccc-0001-0001-0001-000000000001` | `dddddddd-0001-0001-0001-000000000001`（金額: 1000, カテゴリ: transportation） |
| `report_submitted` | `submitted` | `cccccccc-0002-0002-0002-000000000002` | — |
| `report_approved` | `approved` | `cccccccc-0003-0003-0003-000000000003` | — |
| `report_rejected` | `rejected` | `cccccccc-0004-0004-0004-000000000004` | — |
| `report_paid` | `paid` | `cccccccc-0005-0005-0005-000000000005` | — |

**テナントAのユーザー**:

| ロール | ユーザーID |
|--------|----------|
| Member | `aaaaaaaa-3333-3333-3333-000000000003` |
| Approver | `aaaaaaaa-2222-2222-2222-000000000002` |
| Admin | `aaaaaaaa-1111-1111-1111-000000000001` |
| Accounting | `aaaaaaaa-4444-4444-4444-000000000004` |

**補足**:
- `report_submitted` / `report_approved` / `report_rejected` / `report_paid` に対して明細操作テストを行う場合、テスト専用の明細をフィクスチャに追加するか、テスト内で事前に作成すること。
- カテゴリIDは `categories` テーブルのグローバルマスタを参照する（例: transportation カテゴリの UUID は実装時にマイグレーションで確定させる）。

---

## 1. POST /api/reports/{id}/items（明細追加）

### 1.1 正常系

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------------|------------------|---------|
| ITM-001 | 統合 | handler | `TestCreateItem_Success` | 前提: `report_draft`（所有者: Member）でリクエスト実行者 = Member。リクエスト: `{"expense_date":"2026-03-10","amount":2000,"category_id":"<transportation UUID>","description":"タクシー代"}` | 201 Created。レスポンスボディに `data.id`（UUID）、`data.amount=2000`、`data.category`、`data.description`、`data.expense_date` が含まれる |
| ITM-002 | 統合 | handler | `TestCreateItem_TotalAmountRecalculated` | 前提: `report_draft`（既存明細 amount=1000）に amount=500 の明細を追加 | 201 Created。レポートの total_amount が 1500 に更新されている（DB で確認） |
| ITM-003 | 統合 | handler | `TestCreateItem_ByApprover` | 前提: Approver が自分の draft レポートに明細追加 | 201 Created（暗黙的ロール包含により Approver も作成可能） |
| ITM-004 | 統合 | handler | `TestCreateItem_ByAccounting` | 前提: Accounting が自分の draft レポートに明細追加 | 201 Created（暗黙的ロール包含により Accounting も作成可能） |
| ITM-005 | 統合 | handler | `TestCreateItem_ByAdmin` | 前提: Admin が自分の draft レポートに明細追加 | 201 Created（暗黙的ロール包含により Admin も作成可能） |
| ITM-006 | 単体 | domain | `TestExpenseItem_AmountMinimum` | amount=1（最小値）で明細を作成 | エラーなし。amount=1 の明細が生成される |

### 1.2 バリデーションエラー（422）

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------------|------------------|---------|
| ITM-011 | 統合 | handler | `TestCreateItem_AmountZero` | `amount=0`、他フィールドは有効値 | 422 VALIDATION_ERROR。`details[].field = "amount"` を含む |
| ITM-012 | 統合 | handler | `TestCreateItem_AmountNegative` | `amount=-1`、他フィールドは有効値 | 422 VALIDATION_ERROR。`details[].field = "amount"` を含む |
| ITM-013 | 単体 | domain | `TestExpenseItem_AmountZero` | ドメイン層で amount=0 の明細を直接生成 | `InvalidAmount` エラーが返る（ITM-002 不変条件: amount > 0） |
| ITM-014 | 単体 | domain | `TestExpenseItem_AmountNegative` | ドメイン層で amount=-100 の明細を直接生成 | `InvalidAmount` エラーが返る |
| ITM-015 | 統合 | handler | `TestCreateItem_MissingExpenseDate` | `expense_date` を省略 | 422 VALIDATION_ERROR。`details[].field = "expense_date"` を含む |
| ITM-016 | 統合 | handler | `TestCreateItem_InvalidExpenseDateFormat` | `expense_date="not-a-date"` | 422 VALIDATION_ERROR |
| ITM-017 | 統合 | handler | `TestCreateItem_MissingCategoryId` | `category_id` を省略 | 422 VALIDATION_ERROR。`details[].field = "category_id"` を含む |
| ITM-018 | 統合 | handler | `TestCreateItem_InvalidCategoryId` | `category_id="invalid-uuid"` | 422 VALIDATION_ERROR |
| ITM-019 | 統合 | handler | `TestCreateItem_NonExistentCategoryId` | `category_id` に存在しない UUID を指定 | 422 VALIDATION_ERROR または 404。実装方針に応じて確認 |
| ITM-020 | 統合 | handler | `TestCreateItem_MissingDescription` | `description` を省略 | 422 VALIDATION_ERROR。`details[].field = "description"` を含む |
| ITM-021 | 統合 | handler | `TestCreateItem_EmptyDescription` | `description=""` | 422 VALIDATION_ERROR（minLength=1 違反） |
| ITM-022 | 統合 | handler | `TestCreateItem_DescriptionTooLong` | `description` を 501 文字の文字列で指定 | 422 VALIDATION_ERROR（maxLength=500 違反） |
| ITM-023 | 統合 | handler | `TestCreateItem_DescriptionMaxLength` | `description` を 500 文字の文字列で指定 | 201 Created（境界値: 500 文字は許容） |

### 1.3 認証エラー（401）

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------------|------------------|---------|
| ITM-031 | 統合 | handler | `TestCreateItem_Unauthorized` | JWT トークンなしでリクエスト | 401 UNAUTHORIZED |
| ITM-032 | 統合 | handler | `TestCreateItem_ExpiredToken` | 期限切れ JWT でリクエスト | 401 TOKEN_EXPIRED |

### 1.4 認可エラー（403）

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------------|------------------|---------|
| ITM-041 | 統合 | handler | `TestCreateItem_ForbiddenByNonOwner` | 前提: Member Bが Member A の `report_draft` に対してリクエスト（同一テナント・別ユーザー） | 403 FORBIDDEN（所有権不足） |
| ITM-042 | 統合 | handler | `TestCreateItem_ForbiddenByAdminNonOwner` | 前提: Admin が他者（Member）の `report_draft` に対してリクエスト（RBC-014） | 403 FORBIDDEN（Admin も他者のレポートは操作不可） |

### 1.5 リソース不在（404）

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------------|------------------|---------|
| ITM-051 | 統合 | handler | `TestCreateItem_ReportNotFound` | 存在しないレポートID（`00000000-0000-0000-0000-000000000000`）で POST | 404 RESOURCE_NOT_FOUND |

---

## 2. 前提条件: レポート状態による明細追加の制限

レポートが `submitted` 以降の状態では明細追加は拒否される（ITM-010 不変条件: draft 制約）。

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------------|------------------|---------|
| ITM-061 | 統合 | handler | `TestCreateItem_ReportSubmitted_Rejected` | 前提: `report_submitted` に対して所有者 Member が POST | 422 REPORT_NOT_EDITABLE |
| ITM-062 | 統合 | handler | `TestCreateItem_ReportApproved_Rejected` | 前提: `report_approved` に対して所有者 Member が POST | 422 REPORT_NOT_EDITABLE |
| ITM-063 | 統合 | handler | `TestCreateItem_ReportRejected_Rejected` | 前提: `report_rejected` に対して所有者 Member が POST | 422 REPORT_NOT_EDITABLE |
| ITM-064 | 統合 | handler | `TestCreateItem_ReportPaid_Rejected` | 前提: `report_paid` に対して所有者 Member が POST | 422 REPORT_NOT_EDITABLE |
| ITM-065 | 単体 | domain | `TestExpenseReport_AddItem_NotDraft` | ドメイン層で submitted 状態のレポートに明細を追加 | `ReportNotEditable` エラーが返る |

---

## 3. PUT /api/reports/{id}/items/{itemId}（明細更新）

### 3.1 正常系

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------------|------------------|---------|
| ITM-101 | 統合 | handler | `TestUpdateItem_Success` | 前提: `report_draft` の明細 `dddddddd-0001-0001-0001-000000000001`（amount=1000）。リクエスト: `{"expense_date":"2026-03-11","amount":1500,"category_id":"<transportation UUID>","description":"タクシー代（修正）","updated_at":"<現在の updated_at>"}` | 200 OK。レスポンスボディに更新後の明細データ（amount=1500 等）が含まれる |
| ITM-102 | 統合 | handler | `TestUpdateItem_TotalAmountRecalculated` | 前提: `report_draft`（明細 amount=1000）を amount=2000 に更新 | 200 OK。レポートの total_amount が 2000 に更新されている（DB で確認） |
| ITM-103 | 統合 | handler | `TestUpdateItem_ByApprover` | 前提: Approver が自分の draft レポートの明細を更新 | 200 OK |
| ITM-104 | 統合 | handler | `TestUpdateItem_ByAccounting` | 前提: Accounting が自分の draft レポートの明細を更新 | 200 OK |
| ITM-105 | 統合 | handler | `TestUpdateItem_ByAdmin` | 前提: Admin が自分の draft レポートの明細を更新 | 200 OK |

### 3.2 バリデーションエラー（422）

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------------|------------------|---------|
| ITM-111 | 統合 | handler | `TestUpdateItem_AmountZero` | `amount=0`、`updated_at` は有効値 | 422 VALIDATION_ERROR |
| ITM-112 | 統合 | handler | `TestUpdateItem_AmountNegative` | `amount=-1`、`updated_at` は有効値 | 422 VALIDATION_ERROR |
| ITM-113 | 統合 | handler | `TestUpdateItem_MissingUpdatedAt` | `updated_at` を省略（楽観的ロックフィールド必須） | 422 VALIDATION_ERROR。`details[].field = "updated_at"` を含む |
| ITM-114 | 統合 | handler | `TestUpdateItem_MissingDescription` | `description` を省略 | 422 VALIDATION_ERROR |
| ITM-115 | 統合 | handler | `TestUpdateItem_EmptyDescription` | `description=""` | 422 VALIDATION_ERROR（minLength=1 違反） |
| ITM-116 | 統合 | handler | `TestUpdateItem_DescriptionTooLong` | `description` を 501 文字で指定 | 422 VALIDATION_ERROR（maxLength=500 違反） |
| ITM-117 | 統合 | handler | `TestUpdateItem_InvalidExpenseDateFormat` | `expense_date="2026/03/10"`（スラッシュ区切り） | 422 VALIDATION_ERROR |

### 3.3 楽観的ロック競合（409）

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------------|------------------|---------|
| ITM-121 | 統合 | handler | `TestUpdateItem_OptimisticLockConflict` | 前提: `report_draft` の明細を取得後、別リクエストで同明細を更新済み。古い `updated_at` を使って PUT | 409 CONFLICT |

### 3.4 認証エラー（401）

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------------|------------------|---------|
| ITM-131 | 統合 | handler | `TestUpdateItem_Unauthorized` | JWT トークンなしでリクエスト | 401 UNAUTHORIZED |

### 3.5 認可エラー（403）

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------------|------------------|---------|
| ITM-141 | 統合 | handler | `TestUpdateItem_ForbiddenByNonOwner` | 前提: 別ユーザー（Member B）が Member A の `report_draft` の明細を更新（同一テナント） | 403 FORBIDDEN |
| ITM-142 | 統合 | handler | `TestUpdateItem_ForbiddenByAdminNonOwner` | 前提: Admin が他者（Member）の `report_draft` の明細を更新（RBC-014） | 403 FORBIDDEN |

### 3.6 リソース不在（404）

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------------|------------------|---------|
| ITM-151 | 統合 | handler | `TestUpdateItem_ReportNotFound` | 存在しないレポートID で PUT | 404 RESOURCE_NOT_FOUND |
| ITM-152 | 統合 | handler | `TestUpdateItem_ItemNotFound` | 正しいレポートID・存在しない明細ID で PUT | 404 RESOURCE_NOT_FOUND |
| ITM-153 | 統合 | handler | `TestUpdateItem_ItemBelongsToDifferentReport` | 正しいレポートID・別レポートに属する明細ID で PUT | 404 RESOURCE_NOT_FOUND（明細の親チェック違反） |

---

## 4. 前提条件: レポート状態による明細更新の制限

レポートが `submitted` 以降の状態では明細更新は拒否される（ITM-010 不変条件: draft 制約）。

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------------|------------------|---------|
| ITM-161 | 統合 | handler | `TestUpdateItem_ReportSubmitted_Rejected` | 前提: `report_submitted` の明細に対して所有者 Member が PUT | 422 REPORT_NOT_EDITABLE |
| ITM-162 | 統合 | handler | `TestUpdateItem_ReportApproved_Rejected` | 前提: `report_approved` の明細に対して所有者 Member が PUT | 422 REPORT_NOT_EDITABLE |
| ITM-163 | 統合 | handler | `TestUpdateItem_ReportRejected_Rejected` | 前提: `report_rejected` の明細に対して所有者 Member が PUT | 422 REPORT_NOT_EDITABLE |
| ITM-164 | 統合 | handler | `TestUpdateItem_ReportPaid_Rejected` | 前提: `report_paid` の明細に対して所有者 Member が PUT | 422 REPORT_NOT_EDITABLE |
| ITM-165 | 単体 | domain | `TestExpenseReport_UpdateItem_NotDraft` | ドメイン層で submitted 状態のレポートの明細を更新 | `ReportNotEditable` エラーが返る |

---

## 5. DELETE /api/reports/{id}/items/{itemId}（明細削除）

### 5.1 正常系

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------------|------------------|---------|
| ITM-201 | 統合 | handler | `TestDeleteItem_Success` | 前提: `report_draft` の明細 `dddddddd-0001-0001-0001-000000000001` を所有者 Member が DELETE | 204 No Content |
| ITM-202 | 統合 | handler | `TestDeleteItem_TotalAmountRecalculated` | 前提: `report_draft`（明細 amount=1000）を削除 | 204 No Content。レポートの total_amount が 0 に更新されている（DB で確認） |
| ITM-203 | 統合 | handler | `TestDeleteItem_SoftDelete` | 前提: 明細を DELETE 後、DB を直接確認 | 削除された明細の `deleted_at` が NULL でない（論理削除: DAT-002 不変条件） |
| ITM-204 | 統合 | handler | `TestDeleteItem_AttachmentsCascadeSoftDeleted` | 前提: 添付ファイルを持つ明細を DELETE | 204 No Content。紐づく添付ファイルも `deleted_at` が設定される（連動論理削除） |
| ITM-205 | 統合 | handler | `TestDeleteItem_ByApprover` | 前提: Approver が自分の draft レポートの明細を DELETE | 204 No Content |
| ITM-206 | 統合 | handler | `TestDeleteItem_ByAccounting` | 前提: Accounting が自分の draft レポートの明細を DELETE | 204 No Content |
| ITM-207 | 統合 | handler | `TestDeleteItem_ByAdmin` | 前提: Admin が自分の draft レポートの明細を DELETE | 204 No Content |

### 5.2 認証エラー（401）

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------------|------------------|---------|
| ITM-211 | 統合 | handler | `TestDeleteItem_Unauthorized` | JWT トークンなしでリクエスト | 401 UNAUTHORIZED |

### 5.3 認可エラー（403）

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------------|------------------|---------|
| ITM-221 | 統合 | handler | `TestDeleteItem_ForbiddenByNonOwner` | 前提: 別ユーザー（Member B）が Member A の `report_draft` の明細を DELETE（同一テナント） | 403 FORBIDDEN |
| ITM-222 | 統合 | handler | `TestDeleteItem_ForbiddenByAdminNonOwner` | 前提: Admin が他者（Member）の `report_draft` の明細を DELETE（RBC-014） | 403 FORBIDDEN |

### 5.4 リソース不在（404）

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------------|------------------|---------|
| ITM-231 | 統合 | handler | `TestDeleteItem_ReportNotFound` | 存在しないレポートID で DELETE | 404 RESOURCE_NOT_FOUND |
| ITM-232 | 統合 | handler | `TestDeleteItem_ItemNotFound` | 正しいレポートID・存在しない明細ID で DELETE | 404 RESOURCE_NOT_FOUND |
| ITM-233 | 統合 | handler | `TestDeleteItem_ItemBelongsToDifferentReport` | 正しいレポートID・別レポートに属する明細ID で DELETE | 404 RESOURCE_NOT_FOUND（明細の親チェック違反） |
| ITM-234 | 統合 | handler | `TestDeleteItem_AlreadyDeleted` | 論理削除済み明細に再度 DELETE | 404 RESOURCE_NOT_FOUND |

---

## 6. 前提条件: レポート状態による明細削除の制限

レポートが `submitted` 以降の状態では明細削除は拒否される（ITM-010 不変条件: draft 制約）。

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------------|------------------|---------|
| ITM-241 | 統合 | handler | `TestDeleteItem_ReportSubmitted_Rejected` | 前提: `report_submitted` の明細に対して所有者 Member が DELETE | 422 REPORT_NOT_EDITABLE |
| ITM-242 | 統合 | handler | `TestDeleteItem_ReportApproved_Rejected` | 前提: `report_approved` の明細に対して所有者 Member が DELETE | 422 REPORT_NOT_EDITABLE |
| ITM-243 | 統合 | handler | `TestDeleteItem_ReportRejected_Rejected` | 前提: `report_rejected` の明細に対して所有者 Member が DELETE | 422 REPORT_NOT_EDITABLE |
| ITM-244 | 統合 | handler | `TestDeleteItem_ReportPaid_Rejected` | 前提: `report_paid` の明細に対して所有者 Member が DELETE | 422 REPORT_NOT_EDITABLE |
| ITM-245 | 単体 | domain | `TestExpenseReport_DeleteItem_NotDraft` | ドメイン層で submitted 状態のレポートの明細を削除 | `ReportNotEditable` エラーが返る |

---

## 7. RBACテスト: エンドポイント × ロール

各エンドポイントに対するロール別アクセス許可・拒否を網羅する。全エンドポイントの許可ロールは `Member, Approver, Admin, Accounting`（全ロール）だが、所有者条件（RBC-010）により非所有者は 403 となる。

本セクションでは「RBACミドルウェア層での拒否」が発生するケース（認証必須エンドポイントへの未認証アクセス）のみを定義する。ロールによるアクセス拒否はすべてのロールに POST /PUT /DELETE が許可されているため、非所有者の 403 は §1.4 / §3.5 / §5.3 で扱う。

| テストID | テストレベル | レイヤー | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------------|------------------|---------|
| ITM-301 | 統合 | handler | `TestCreateItem_RBACAllRolesAllowed_Member` | Member が自分の draft レポートに POST | 201 Created（RBAC ミドルウェアは通過。所有権チェックも通過） |
| ITM-302 | 統合 | handler | `TestCreateItem_RBACAllRolesAllowed_Approver` | Approver が自分の draft レポートに POST | 201 Created |
| ITM-303 | 統合 | handler | `TestCreateItem_RBACAllRolesAllowed_Accounting` | Accounting が自分の draft レポートに POST | 201 Created |
| ITM-304 | 統合 | handler | `TestCreateItem_RBACAllRolesAllowed_Admin` | Admin が自分の draft レポートに POST | 201 Created |
| ITM-305 | 統合 | handler | `TestUpdateItem_RBACAllRolesAllowed_Member` | Member が自分の draft レポートの明細に PUT | 200 OK |
| ITM-306 | 統合 | handler | `TestUpdateItem_RBACAllRolesAllowed_Approver` | Approver が自分の draft レポートの明細に PUT | 200 OK |
| ITM-307 | 統合 | handler | `TestUpdateItem_RBACAllRolesAllowed_Accounting` | Accounting が自分の draft レポートの明細に PUT | 200 OK |
| ITM-308 | 統合 | handler | `TestUpdateItem_RBACAllRolesAllowed_Admin` | Admin が自分の draft レポートの明細に PUT | 200 OK |
| ITM-309 | 統合 | handler | `TestDeleteItem_RBACAllRolesAllowed_Member` | Member が自分の draft レポートの明細に DELETE | 204 No Content |
| ITM-310 | 統合 | handler | `TestDeleteItem_RBACAllRolesAllowed_Approver` | Approver が自分の draft レポートの明細に DELETE | 204 No Content |
| ITM-311 | 統合 | handler | `TestDeleteItem_RBACAllRolesAllowed_Accounting` | Accounting が自分の draft レポートの明細に DELETE | 204 No Content |
| ITM-312 | 統合 | handler | `TestDeleteItem_RBACAllRolesAllowed_Admin` | Admin が自分の draft レポートの明細に DELETE | 204 No Content |

---

## 8. ドメイン不変条件サマリー

本ファイルのテストケースがカバーする不変条件の対応表。

| 不変条件ID | 内容 | カバーするテストID |
|-----------|------|-----------------|
| ITM-002 | amount > 0（正の整数） | ITM-006, ITM-011, ITM-012, ITM-013, ITM-014, ITM-111, ITM-112 |
| ITM-010 | draft 以外の状態での明細操作を拒否 | ITM-061〜065, ITM-161〜165, ITM-241〜245 |
| RPT-006 | total_amount = Σ items.amount（明細追加・更新・削除時の再計算） | ITM-002, ITM-102, ITM-202 |
| DAT-002 | 削除は論理削除（deleted_at） | ITM-203 |
| RBC-010 | 申請系操作は所有者のみ | ITM-041, ITM-042, ITM-141, ITM-142, ITM-221, ITM-222 |
| RBC-014 | Admin も他者のレポートは編集不可 | ITM-042, ITM-142, ITM-222 |

---

## 9. 実装ガイド

### 9.1 テストファイル配置

```
expense-saas/
  internal/
    handler/
      item_handler_test.go    ← 統合テスト（ITM- テストのうち テストレベル=統合）
    domain/
      expense_item_test.go    ← 単体テスト（ITM- テストのうち テストレベル=単体）
```

### 9.2 テスト実行コマンド

```bash
# 単体テスト（ドメイン層のみ）
go test ./internal/domain/... -v -run TestExpenseItem -cover

# 統合テスト（テスト用DBが必要）
go test ./internal/handler/... -v -tags=integration -run TestCreateItem
go test ./internal/handler/... -v -tags=integration -run TestUpdateItem
go test ./internal/handler/... -v -tags=integration -run TestDeleteItem
```

### 9.3 フィクスチャセットアップ例（擬似コード）

```go
// TestMain でフィクスチャを投入する
func TestMain(m *testing.M) {
    db = setupTestDB()
    loadFixtures(db, "testdata/fixtures/tenants.sql")
    loadFixtures(db, "testdata/fixtures/users.sql")
    loadFixtures(db, "testdata/fixtures/reports.sql")   // report_draft, report_submitted 等
    loadFixtures(db, "testdata/fixtures/items.sql")     // dddddddd-0001-... の明細
    os.Exit(m.Run())
}

// テスト前に TRUNCATE + フィクスチャ再投入（テスト間の独立性確保）
func setup(t *testing.T) {
    truncateAllTables(db)
    loadFixtures(db, ...)
}
```

### 9.4 楽観的ロックテストの注意点

ITM-121 の楽観的ロックテストでは、以下の手順で競合状態を再現する:

1. テスト内で明細を取得して `updated_at` を記録する
2. 同じ明細に対して別リクエストで更新を実行（`updated_at` を前進させる）
3. 手順 1 で記録した古い `updated_at` を使って PUT リクエストを送信する
4. 409 CONFLICT が返ることを確認する

### 9.5 テスト用カテゴリIDの取り扱い

カテゴリIDは `categories` テーブルのグローバルマスタから取得する。テスト用フィクスチャの SQL で `transportation` カテゴリの固定 UUID を定義し、テストコード内の定数として参照すること。

```go
const (
    CategoryTransportationID = "ffffffff-0001-0001-0001-000000000001"  // 例（実装時に確定）
    CategoryFoodID           = "ffffffff-0002-0002-0002-000000000002"
)
```
