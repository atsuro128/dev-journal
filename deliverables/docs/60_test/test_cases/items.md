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

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|------------------|---------|
| ITM-001 | 統合 | handler | 正常系 | ITM-F01 | openapi.yaml#createItem, db_schema.md#expense_items | `TestCreateItem_Success` | 前提: `report_draft`（所有者: Member）でリクエスト実行者 = Member。リクエスト: `{"expense_date":"2026-03-10","amount":2000,"category_id":"<transportation UUID>","description":"タクシー代"}` | 201 Created。レスポンスボディに `data.id`（UUID）、`data.amount=2000`、`data.category`、`data.description`、`data.expense_date` が含まれる |
| ITM-002 | 統合 | handler | 正常系 | ITM-F01, RPT-006 | openapi.yaml#createItem, db_schema.md#expense_reports.total_amount | `TestCreateItem_TotalAmountRecalculated` | 前提: `report_draft`（既存明細 amount=1000）に amount=500 の明細を追加 | 201 Created。レポートの total_amount が 1500 に更新されている（DB で確認） |
| ITM-003 | 統合 | handler | 認可 | ITM-F01, RBC-010 | openapi.yaml#createItem, authz.md#4.1 | `TestCreateItem_ByApprover` | 前提: Approver が自分の draft レポートに明細追加 | 201 Created（暗黙的ロール包含により Approver も作成可能） |
| ITM-004 | 統合 | handler | 認可 | ITM-F01, RBC-010 | openapi.yaml#createItem, authz.md#4.1 | `TestCreateItem_ByAccounting` | 前提: Accounting が自分の draft レポートに明細追加 | 201 Created（暗黙的ロール包含により Accounting も作成可能） |
| ITM-005 | 統合 | handler | 認可 | ITM-F01, RBC-010 | openapi.yaml#createItem, authz.md#4.1 | `TestCreateItem_ByAdmin` | 前提: Admin が自分の draft レポートに明細追加 | 201 Created（暗黙的ロール包含により Admin も作成可能） |
| ITM-006 | 単体 | domain | 境界値 | ITM-002 | db_schema.md#expense_items.amount | `TestExpenseItem_AmountMinimum` | amount=1（最小値）で明細を作成 | エラーなし。amount=1 の明細が生成される |

### 1.2 バリデーションエラー（422）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|------------------|---------|
| ITM-011 | 統合 | handler | 境界値 | ITM-002 | openapi.yaml#createItem, db_schema.md#expense_items.amount | `TestCreateItem_AmountZero` | `amount=0`、他フィールドは有効値 | 422 VALIDATION_ERROR。`details[].field = "amount"` を含む |
| ITM-012 | 統合 | handler | 境界値 | ITM-002 | openapi.yaml#createItem, db_schema.md#expense_items.amount | `TestCreateItem_AmountNegative` | `amount=-1`、他フィールドは有効値 | 422 VALIDATION_ERROR。`details[].field = "amount"` を含む |
| ITM-013 | 単体 | domain | 境界値 | ITM-002 | db_schema.md#expense_items.amount#CHECK | `TestExpenseItem_AmountZero` | ドメイン層で amount=0 の明細を直接生成 | `InvalidAmount` エラーが返る（ITM-002 不変条件: amount > 0） |
| ITM-014 | 単体 | domain | 境界値 | ITM-002 | db_schema.md#expense_items.amount#CHECK | `TestExpenseItem_AmountNegative` | ドメイン層で amount=-100 の明細を直接生成 | `InvalidAmount` エラーが返る |
| ITM-015 | 統合 | handler | 異常系 | ITM-001 | openapi.yaml#createItem | `TestCreateItem_MissingExpenseDate` | `expense_date` を省略 | 422 VALIDATION_ERROR。`details[].field = "expense_date"` を含む |
| ITM-016 | 統合 | handler | 異常系 | ITM-001 | openapi.yaml#createItem | `TestCreateItem_InvalidExpenseDateFormat` | `expense_date="not-a-date"` | 422 VALIDATION_ERROR |
| ITM-017 | 統合 | handler | 異常系 | ITM-003 | openapi.yaml#createItem | `TestCreateItem_MissingCategoryId` | `category_id` を省略 | 422 VALIDATION_ERROR。`details[].field = "category_id"` を含む |
| ITM-018 | 統合 | handler | 異常系 | ITM-003 | openapi.yaml#createItem | `TestCreateItem_InvalidCategoryId` | `category_id="invalid-uuid"` | 422 VALIDATION_ERROR |
| ITM-019 | 統合 | handler | 異常系 | ITM-003, ITM-005 | openapi.yaml#createItem, db_schema.md#categories | `TestCreateItem_NonExistentCategoryId` | `category_id` に存在しない UUID を指定 | 422 VALIDATION_ERROR または 404。実装方針に応じて確認 |
| ITM-020 | 統合 | handler | 異常系 | ITM-004 | openapi.yaml#createItem | `TestCreateItem_MissingDescription` | `description` を省略 | 422 VALIDATION_ERROR。`details[].field = "description"` を含む |
| ITM-021 | 統合 | handler | 異常系 | ITM-004 | openapi.yaml#createItem | `TestCreateItem_EmptyDescription` | `description=""` | 422 VALIDATION_ERROR（minLength=1 違反） |
| ITM-022 | 統合 | handler | 境界値 | ITM-004 | openapi.yaml#createItem | `TestCreateItem_DescriptionTooLong` | `description` を 501 文字の文字列で指定 | 422 VALIDATION_ERROR（maxLength=500 違反） |
| ITM-023 | 統合 | handler | 境界値 | ITM-004 | openapi.yaml#createItem | `TestCreateItem_DescriptionMaxLength` | `description` を 500 文字の文字列で指定 | 201 Created（境界値: 500 文字は許容） |

### 1.3 認証エラー（401）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|------------------|---------|
| ITM-031 | 統合 | handler | 異常系 | ITM-F01 | openapi.yaml#createItem | `TestCreateItem_Unauthorized` | JWT トークンなしでリクエスト | 401 UNAUTHORIZED |
| ITM-032 | 統合 | handler | 異常系 | ITM-F01, SEC-003 | openapi.yaml#createItem, security.md#2.1 | `TestCreateItem_ExpiredToken` | 期限切れ JWT でリクエスト | 401 TOKEN_EXPIRED |

### 1.4 認可エラー（403）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|------------------|---------|
| ITM-041 | 統合 | handler | 認可 | ITM-F01, RBC-010 | openapi.yaml#createItem, authz.md#4.1 | `TestCreateItem_ForbiddenByNonOwner` | 前提: Member Bが Member A の `report_draft` に対してリクエスト（同一テナント・別ユーザー） | 403 FORBIDDEN（所有権不足） |
| ITM-042 | 統合 | handler | 認可 | ITM-F01, RBC-014 | openapi.yaml#createItem, authz.md#4.4 | `TestCreateItem_ForbiddenByAdminNonOwner` | 前提: Admin が他者（Member）の `report_draft` に対してリクエスト（RBC-014） | 403 FORBIDDEN（Admin も他者のレポートは操作不可） |

### 1.5 リソース不在（404）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|------------------|---------|
| ITM-051 | 統合 | handler | 異常系 | ITM-F01 | openapi.yaml#createItem | `TestCreateItem_ReportNotFound` | 存在しないレポートID（`00000000-0000-0000-0000-000000000000`）で POST | 404 RESOURCE_NOT_FOUND |

---

## 2. 前提条件: レポート状態による明細追加の制限

レポートが `submitted` 以降の状態では明細追加は拒否される（ITM-010 不変条件: draft 制約）。

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|------------------|---------|
| ITM-061 | 統合 | handler | 状態遷移 | ITM-010, ITM-F01 | openapi.yaml#createItem | `TestCreateItem_ReportSubmitted_Rejected` | 前提: `report_submitted` に対して所有者 Member が POST | 422 REPORT_NOT_EDITABLE |
| ITM-062 | 統合 | handler | 状態遷移 | ITM-010, ITM-F01 | openapi.yaml#createItem | `TestCreateItem_ReportApproved_Rejected` | 前提: `report_approved` に対して所有者 Member が POST | 422 REPORT_NOT_EDITABLE |
| ITM-063 | 統合 | handler | 状態遷移 | ITM-010, ITM-F01 | openapi.yaml#createItem | `TestCreateItem_ReportRejected_Rejected` | 前提: `report_rejected` に対して所有者 Member が POST | 422 REPORT_NOT_EDITABLE |
| ITM-064 | 統合 | handler | 状態遷移 | ITM-010, ITM-F01 | openapi.yaml#createItem | `TestCreateItem_ReportPaid_Rejected` | 前提: `report_paid` に対して所有者 Member が POST | 422 REPORT_NOT_EDITABLE |
| ITM-065 | 単体 | domain | 状態遷移 | ITM-010 | state_machine.md | `TestExpenseReport_AddItem_NotDraft` | ドメイン層で submitted 状態のレポートに明細を追加 | `ReportNotEditable` エラーが返る |

---

## 3. PUT /api/reports/{id}/items/{itemId}（明細更新）

### 3.1 正常系

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|------------------|---------|
| ITM-101 | 統合 | handler | 正常系 | ITM-F02 | openapi.yaml#updateItem, db_schema.md#expense_items | `TestUpdateItem_Success` | 前提: `report_draft` の明細 `dddddddd-0001-0001-0001-000000000001`（amount=1000）。リクエスト: `{"expense_date":"2026-03-11","amount":1500,"category_id":"<transportation UUID>","description":"タクシー代（修正）","updated_at":"<現在の updated_at>"}` | 200 OK。レスポンスボディに更新後の明細データ（amount=1500 等）が含まれる |
| ITM-102 | 統合 | handler | 正常系 | ITM-F02, RPT-006 | openapi.yaml#updateItem, db_schema.md#expense_reports.total_amount | `TestUpdateItem_TotalAmountRecalculated` | 前提: `report_draft`（明細 amount=1000）を amount=2000 に更新 | 200 OK。レポートの total_amount が 2000 に更新されている（DB で確認） |
| ITM-103 | 統合 | handler | 認可 | ITM-F02, RBC-010 | openapi.yaml#updateItem, authz.md#4.1 | `TestUpdateItem_ByApprover` | 前提: Approver が自分の draft レポートの明細を更新 | 200 OK |
| ITM-104 | 統合 | handler | 認可 | ITM-F02, RBC-010 | openapi.yaml#updateItem, authz.md#4.1 | `TestUpdateItem_ByAccounting` | 前提: Accounting が自分の draft レポートの明細を更新 | 200 OK |
| ITM-105 | 統合 | handler | 認可 | ITM-F02, RBC-010 | openapi.yaml#updateItem, authz.md#4.1 | `TestUpdateItem_ByAdmin` | 前提: Admin が自分の draft レポートの明細を更新 | 200 OK |

### 3.2 バリデーションエラー（422）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|------------------|---------|
| ITM-111 | 統合 | handler | 境界値 | ITM-002, ITM-F02 | openapi.yaml#updateItem | `TestUpdateItem_AmountZero` | `amount=0`、`updated_at` は有効値 | 422 VALIDATION_ERROR |
| ITM-112 | 統合 | handler | 境界値 | ITM-002, ITM-F02 | openapi.yaml#updateItem | `TestUpdateItem_AmountNegative` | `amount=-1`、`updated_at` は有効値 | 422 VALIDATION_ERROR |
| ITM-113 | 統合 | handler | 異常系 | ITM-F02 | openapi.yaml#updateItem | `TestUpdateItem_MissingUpdatedAt` | `updated_at` を省略（楽観的ロックフィールド必須） | 422 VALIDATION_ERROR。`details[].field = "updated_at"` を含む |
| ITM-114 | 統合 | handler | 異常系 | ITM-004, ITM-F02 | openapi.yaml#updateItem | `TestUpdateItem_MissingDescription` | `description` を省略 | 422 VALIDATION_ERROR |
| ITM-115 | 統合 | handler | 異常系 | ITM-004, ITM-F02 | openapi.yaml#updateItem | `TestUpdateItem_EmptyDescription` | `description=""` | 422 VALIDATION_ERROR（minLength=1 違反） |
| ITM-116 | 統合 | handler | 境界値 | ITM-004, ITM-F02 | openapi.yaml#updateItem | `TestUpdateItem_DescriptionTooLong` | `description` を 501 文字で指定 | 422 VALIDATION_ERROR（maxLength=500 違反） |
| ITM-117 | 統合 | handler | 異常系 | ITM-001, ITM-F02 | openapi.yaml#updateItem | `TestUpdateItem_InvalidExpenseDateFormat` | `expense_date="2026/03/10"`（スラッシュ区切り） | 422 VALIDATION_ERROR |

### 3.3 楽観的ロック競合（409）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|------------------|---------|
| ITM-121 | 統合 | handler | 異常系 | ITM-F02 | openapi.yaml#updateItem | `TestUpdateItem_OptimisticLockConflict` | 前提: `report_draft` の明細を取得後、別リクエストで同明細を更新済み。古い `updated_at` を使って PUT | 409 CONFLICT |

### 3.4 認証エラー（401）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|------------------|---------|
| ITM-131 | 統合 | handler | 異常系 | ITM-F02 | openapi.yaml#updateItem | `TestUpdateItem_Unauthorized` | JWT トークンなしでリクエスト | 401 UNAUTHORIZED |

### 3.5 認可エラー（403）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|------------------|---------|
| ITM-141 | 統合 | handler | 認可 | ITM-F02, RBC-010 | openapi.yaml#updateItem, authz.md#4.1 | `TestUpdateItem_ForbiddenByNonOwner` | 前提: 別ユーザー（Member B）が Member A の `report_draft` の明細を更新（同一テナント） | 403 FORBIDDEN |
| ITM-142 | 統合 | handler | 認可 | ITM-F02, RBC-014 | openapi.yaml#updateItem, authz.md#4.4 | `TestUpdateItem_ForbiddenByAdminNonOwner` | 前提: Admin が他者（Member）の `report_draft` の明細を更新（RBC-014） | 403 FORBIDDEN |

### 3.6 リソース不在（404）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|------------------|---------|
| ITM-151 | 統合 | handler | 異常系 | ITM-F02 | openapi.yaml#updateItem | `TestUpdateItem_ReportNotFound` | 存在しないレポートID で PUT | 404 RESOURCE_NOT_FOUND |
| ITM-152 | 統合 | handler | 異常系 | ITM-F02 | openapi.yaml#updateItem | `TestUpdateItem_ItemNotFound` | 正しいレポートID・存在しない明細ID で PUT | 404 RESOURCE_NOT_FOUND |
| ITM-153 | 統合 | handler | 異常系 | ITM-F02 | openapi.yaml#updateItem | `TestUpdateItem_ItemBelongsToDifferentReport` | 正しいレポートID・別レポートに属する明細ID で PUT | 404 RESOURCE_NOT_FOUND（明細の親チェック違反） |

---

## 4. 前提条件: レポート状態による明細更新の制限

レポートが `submitted` 以降の状態では明細更新は拒否される（ITM-010 不変条件: draft 制約）。

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|------------------|---------|
| ITM-161 | 統合 | handler | 状態遷移 | ITM-010, ITM-F02 | openapi.yaml#updateItem | `TestUpdateItem_ReportSubmitted_Rejected` | 前提: `report_submitted` の明細に対して所有者 Member が PUT | 422 REPORT_NOT_EDITABLE |
| ITM-162 | 統合 | handler | 状態遷移 | ITM-010, ITM-F02 | openapi.yaml#updateItem | `TestUpdateItem_ReportApproved_Rejected` | 前提: `report_approved` の明細に対して所有者 Member が PUT | 422 REPORT_NOT_EDITABLE |
| ITM-163 | 統合 | handler | 状態遷移 | ITM-010, ITM-F02 | openapi.yaml#updateItem | `TestUpdateItem_ReportRejected_Rejected` | 前提: `report_rejected` の明細に対して所有者 Member が PUT | 422 REPORT_NOT_EDITABLE |
| ITM-164 | 統合 | handler | 状態遷移 | ITM-010, ITM-F02 | openapi.yaml#updateItem | `TestUpdateItem_ReportPaid_Rejected` | 前提: `report_paid` の明細に対して所有者 Member が PUT | 422 REPORT_NOT_EDITABLE |
| ITM-165 | 単体 | domain | 状態遷移 | ITM-010 | state_machine.md | `TestExpenseReport_UpdateItem_NotDraft` | ドメイン層で submitted 状態のレポートの明細を更新 | `ReportNotEditable` エラーが返る |

---

## 5. DELETE /api/reports/{id}/items/{itemId}（明細削除）

### 5.1 正常系

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|------------------|---------|
| ITM-201 | 統合 | handler | 正常系 | ITM-F03 | openapi.yaml#deleteItem | `TestDeleteItem_Success` | 前提: `report_draft` の明細 `dddddddd-0001-0001-0001-000000000001` を所有者 Member が DELETE | 204 No Content |
| ITM-202 | 統合 | handler | 正常系 | ITM-F03, RPT-006 | openapi.yaml#deleteItem, db_schema.md#expense_reports.total_amount | `TestDeleteItem_TotalAmountRecalculated` | 前提: `report_draft`（明細 amount=1000）を削除 | 204 No Content。レポートの total_amount が 0 に更新されている（DB で確認） |
| ITM-203 | 統合 | handler | 正常系 | ITM-F03, NFR-DATA-001 | openapi.yaml#deleteItem, db_schema.md#expense_items.deleted_at | `TestDeleteItem_SoftDelete` | 前提: 明細を DELETE 後、DB を直接確認 | 削除された明細の `deleted_at` が NULL でない（論理削除: DAT-002 不変条件） |
| ITM-204 | 統合 | handler | 正常系 | ITM-F03 | openapi.yaml#deleteItem, db_schema.md#attachments.deleted_at | `TestDeleteItem_AttachmentsCascadeSoftDeleted` | 前提: 添付ファイルを持つ明細を DELETE | 204 No Content。紐づく添付ファイルも `deleted_at` が設定される（連動論理削除） |
| ITM-205 | 統合 | handler | 認可 | ITM-F03, RBC-010 | openapi.yaml#deleteItem, authz.md#4.1 | `TestDeleteItem_ByApprover` | 前提: Approver が自分の draft レポートの明細を DELETE | 204 No Content |
| ITM-206 | 統合 | handler | 認可 | ITM-F03, RBC-010 | openapi.yaml#deleteItem, authz.md#4.1 | `TestDeleteItem_ByAccounting` | 前提: Accounting が自分の draft レポートの明細を DELETE | 204 No Content |
| ITM-207 | 統合 | handler | 認可 | ITM-F03, RBC-010 | openapi.yaml#deleteItem, authz.md#4.1 | `TestDeleteItem_ByAdmin` | 前提: Admin が自分の draft レポートの明細を DELETE | 204 No Content |

### 5.2 認証エラー（401）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|------------------|---------|
| ITM-211 | 統合 | handler | 異常系 | ITM-F03 | openapi.yaml#deleteItem | `TestDeleteItem_Unauthorized` | JWT トークンなしでリクエスト | 401 UNAUTHORIZED |

### 5.3 認可エラー（403）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|------------------|---------|
| ITM-221 | 統合 | handler | 認可 | ITM-F03, RBC-010 | openapi.yaml#deleteItem, authz.md#4.1 | `TestDeleteItem_ForbiddenByNonOwner` | 前提: 別ユーザー（Member B）が Member A の `report_draft` の明細を DELETE（同一テナント） | 403 FORBIDDEN |
| ITM-222 | 統合 | handler | 認可 | ITM-F03, RBC-014 | openapi.yaml#deleteItem, authz.md#4.4 | `TestDeleteItem_ForbiddenByAdminNonOwner` | 前提: Admin が他者（Member）の `report_draft` の明細を DELETE（RBC-014） | 403 FORBIDDEN |

### 5.4 リソース不在（404）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|------------------|---------|
| ITM-231 | 統合 | handler | 異常系 | ITM-F03 | openapi.yaml#deleteItem | `TestDeleteItem_ReportNotFound` | 存在しないレポートID で DELETE | 404 RESOURCE_NOT_FOUND |
| ITM-232 | 統合 | handler | 異常系 | ITM-F03 | openapi.yaml#deleteItem | `TestDeleteItem_ItemNotFound` | 正しいレポートID・存在しない明細ID で DELETE | 404 RESOURCE_NOT_FOUND |
| ITM-233 | 統合 | handler | 異常系 | ITM-F03 | openapi.yaml#deleteItem | `TestDeleteItem_ItemBelongsToDifferentReport` | 正しいレポートID・別レポートに属する明細ID で DELETE | 404 RESOURCE_NOT_FOUND（明細の親チェック違反） |
| ITM-234 | 統合 | handler | 異常系 | ITM-F03, NFR-DATA-001 | openapi.yaml#deleteItem | `TestDeleteItem_AlreadyDeleted` | 論理削除済み明細に再度 DELETE | 404 RESOURCE_NOT_FOUND |

---

## 6. 前提条件: レポート状態による明細削除の制限

レポートが `submitted` 以降の状態では明細削除は拒否される（ITM-010 不変条件: draft 制約）。

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|------------------|---------|
| ITM-241 | 統合 | handler | 状態遷移 | ITM-010, ITM-F03 | openapi.yaml#deleteItem | `TestDeleteItem_ReportSubmitted_Rejected` | 前提: `report_submitted` の明細に対して所有者 Member が DELETE | 422 REPORT_NOT_EDITABLE |
| ITM-242 | 統合 | handler | 状態遷移 | ITM-010, ITM-F03 | openapi.yaml#deleteItem | `TestDeleteItem_ReportApproved_Rejected` | 前提: `report_approved` の明細に対して所有者 Member が DELETE | 422 REPORT_NOT_EDITABLE |
| ITM-243 | 統合 | handler | 状態遷移 | ITM-010, ITM-F03 | openapi.yaml#deleteItem | `TestDeleteItem_ReportRejected_Rejected` | 前提: `report_rejected` の明細に対して所有者 Member が DELETE | 422 REPORT_NOT_EDITABLE |
| ITM-244 | 統合 | handler | 状態遷移 | ITM-010, ITM-F03 | openapi.yaml#deleteItem | `TestDeleteItem_ReportPaid_Rejected` | 前提: `report_paid` の明細に対して所有者 Member が DELETE | 422 REPORT_NOT_EDITABLE |
| ITM-245 | 単体 | domain | 状態遷移 | ITM-010 | state_machine.md | `TestExpenseReport_DeleteItem_NotDraft` | ドメイン層で submitted 状態のレポートの明細を削除 | `ReportNotEditable` エラーが返る |

---

## 7. RBACテスト: エンドポイント × ロール

各エンドポイントに対するロール別アクセス許可・拒否を網羅する。全エンドポイントの許可ロールは `Member, Approver, Admin, Accounting`（全ロール）だが、所有者条件（RBC-010）により非所有者は 403 となる。

本セクションでは「RBACミドルウェア層での拒否」が発生するケース（認証必須エンドポイントへの未認証アクセス）のみを定義する。ロールによるアクセス拒否はすべてのロールに POST /PUT /DELETE が許可されているため、非所有者の 403 は §1.4 / §3.5 / §5.3 で扱う。

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|------------------|---------|
| ITM-301 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | authz.md#3, openapi.yaml#createItem | `TestCreateItem_RBACAllRolesAllowed_Member` | Member が自分の draft レポートに POST | 201 Created（RBAC ミドルウェアは通過。所有権チェックも通過） |
| ITM-302 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | authz.md#3, openapi.yaml#createItem | `TestCreateItem_RBACAllRolesAllowed_Approver` | Approver が自分の draft レポートに POST | 201 Created |
| ITM-303 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | authz.md#3, openapi.yaml#createItem | `TestCreateItem_RBACAllRolesAllowed_Accounting` | Accounting が自分の draft レポートに POST | 201 Created |
| ITM-304 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | authz.md#3, openapi.yaml#createItem | `TestCreateItem_RBACAllRolesAllowed_Admin` | Admin が自分の draft レポートに POST | 201 Created |
| ITM-305 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | authz.md#3, openapi.yaml#updateItem | `TestUpdateItem_RBACAllRolesAllowed_Member` | Member が自分の draft レポートの明細に PUT | 200 OK |
| ITM-306 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | authz.md#3, openapi.yaml#updateItem | `TestUpdateItem_RBACAllRolesAllowed_Approver` | Approver が自分の draft レポートの明細に PUT | 200 OK |
| ITM-307 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | authz.md#3, openapi.yaml#updateItem | `TestUpdateItem_RBACAllRolesAllowed_Accounting` | Accounting が自分の draft レポートの明細に PUT | 200 OK |
| ITM-308 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | authz.md#3, openapi.yaml#updateItem | `TestUpdateItem_RBACAllRolesAllowed_Admin` | Admin が自分の draft レポートの明細に PUT | 200 OK |
| ITM-309 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | authz.md#3, openapi.yaml#deleteItem | `TestDeleteItem_RBACAllRolesAllowed_Member` | Member が自分の draft レポートの明細に DELETE | 204 No Content |
| ITM-310 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | authz.md#3, openapi.yaml#deleteItem | `TestDeleteItem_RBACAllRolesAllowed_Approver` | Approver が自分の draft レポートの明細に DELETE | 204 No Content |
| ITM-311 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | authz.md#3, openapi.yaml#deleteItem | `TestDeleteItem_RBACAllRolesAllowed_Accounting` | Accounting が自分の draft レポートの明細に DELETE | 204 No Content |
| ITM-312 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | authz.md#3, openapi.yaml#deleteItem | `TestDeleteItem_RBACAllRolesAllowed_Admin` | Admin が自分の draft レポートの明細に DELETE | 204 No Content |

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

---

## 10. FE テストケース

本セクションは `55_ui_component/screens/report-detail.md` の明細関連コンポーネント（ItemListSection, ItemListHeader, ItemTable, ItemSlidePanel, ItemForm）と、関連する状態管理 Hook（useCreateItem, useUpdateItem, useDeleteItem, useCategories）のテストケースを定義する。

**テストIDプレフィックス**: `ITM-FE-`（001 から連番）

**対象テストファイル**: `src/pages/reports/__tests__/Item*.test.tsx`, `src/hooks/__tests__/useItems.test.tsx`, `src/hooks/__tests__/useCategories.test.tsx`

**責務境界**:
- 添付ファイル関連コンポーネント（AttachmentArea, AttachmentList, AttachmentUploader）は `attachments.md` に記載する。本ファイルには書かない。
- 共通コンポーネント（AppDataGrid, EmptyState, FormAlert, AppTextField, AppSelect, AppDatePicker）の単体テストは `common-components.md` に記載する。本ファイルでは Props 経由の統合的な動作のみ検証する。
- レポート詳細ページ全体のテスト（ReportDetailPage）は `reports.md` に記載する。

### 参照設計書

- `55_ui_component/screens/report-detail.md` -- ItemListSection, ItemListHeader, ItemTable, ItemSlidePanel, ItemForm
- `55_ui_component/state-management.md` -- useCreateItem, useUpdateItem, useDeleteItem, useCategories
- `55_ui_component/common-components.md` -- FormAlert
- `50_detail_design/screens/report-detail.md` -- バリデーションルール V1-V7

### FE フィクスチャ参照

| 参照名 | 値 | 用途 |
|---|---|---|
| `mockItems` | `ExpenseItem[]`（2件以上） | ItemTable / ItemListSection の描画テスト |
| `mockEmptyItems` | `[]` | 空状態テスト |
| `mockCategories` | `[{ value: '<UUID>', label: '交通費' }, ...]`（6件） | ItemForm のカテゴリ選択肢 |
| `mockItem` | `ExpenseItem`（amount: 1000, category: 'transportation', description: 'タクシー代'） | 編集・閲覧モードの初期値 |

### 10.1 ItemListSection

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ITM-FE-001 | 単体 | ItemListSection | items: ExpenseItem[] | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemListSection | `renders_item_list_section_with_items` | items=mockItems（2件）, isOwner=true, status='draft' | セクションが描画され、ItemListHeader と ItemTable が表示される |
| ITM-FE-002 | 単体 | ItemListSection | items: ExpenseItem[] | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemListSection | `renders_empty_state_when_no_items_owner_draft` | items=[], isOwner=true, status='draft' | EmptyState が「明細はまだ追加されていません。「明細追加」から経費を登録してください。」メッセージで表示される |
| ITM-FE-003 | 単体 | ItemListSection | items: ExpenseItem[] | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemListSection | `renders_empty_state_when_no_items_non_owner` | items=[], isOwner=false, status='draft' | EmptyState が「明細はまだ追加されていません。」メッセージで表示される（所有者向けのガイド文言なし） |
| ITM-FE-004 | 単体 | ItemListSection | items: ExpenseItem[] | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemListSection | `renders_empty_state_when_no_items_non_draft` | items=[], isOwner=true, status='submitted' | EmptyState が「明細はまだ追加されていません。」メッセージで表示される（draft 以外では操作ガイドなし） |

### 10.2 ItemListHeader

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ITM-FE-005 | 単体 | ItemListHeader | itemCount: number | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemListHeader | `renders_item_count_in_header` | itemCount=3, canAddItem=true | 「明細一覧（3件）」のような見出しテキストが表示される |
| ITM-FE-006 | 単体 | ItemListHeader | itemCount: number | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemListHeader | `renders_zero_item_count` | itemCount=0, canAddItem=true | 「明細一覧（0件）」のような見出しテキストが表示される |
| ITM-FE-007 | 単体 | ItemListHeader | canAddItem: boolean | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemListHeader | `shows_add_button_when_canAddItem_true` | canAddItem=true | 「明細追加」ボタンが表示される |
| ITM-FE-008 | 単体 | ItemListHeader | canAddItem: boolean | 正常系 | ITM-010 | 55_ui_component/screens/report-detail.md §ItemListHeader | `hides_add_button_when_canAddItem_false` | canAddItem=false | 「明細追加」ボタンが表示されない |
| ITM-FE-009 | 単体 | ItemListHeader | onAddItem: () => void | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemListHeader | `calls_onAddItem_when_add_button_clicked` | canAddItem=true, onAddItem=jest.fn() | 「明細追加」ボタンをクリックすると onAddItem が呼ばれる |

### 10.3 ItemTable

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ITM-FE-010 | 単体 | ItemTable | items: ExpenseItem[] | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemTable | `renders_all_columns` | items=mockItems, canEditItems=true | 日付・金額・カテゴリ・摘要・添付数・操作の各カラムが表示される |
| ITM-FE-011 | 単体 | ItemTable | items: ExpenseItem[] | 正常系 | ITM-002 | 55_ui_component/screens/report-detail.md §ItemTable | `formats_amount_as_currency` | items=[{ amount: 12345, ... }], canEditItems=false | 金額カラムに通貨フォーマット（例: "12,345" や "¥12,345"）で表示される |
| ITM-FE-012 | 単体 | ItemTable | canEditItems: boolean | 正常系 | ITM-010 | 55_ui_component/screens/report-detail.md §ItemTable | `shows_action_column_when_canEditItems_true` | canEditItems=true | 操作列（編集・削除ボタン）が表示される |
| ITM-FE-013 | 単体 | ItemTable | canEditItems: boolean | 正常系 | ITM-010 | 55_ui_component/screens/report-detail.md §ItemTable | `hides_action_column_when_canEditItems_false` | canEditItems=false | 操作列が表示されない |
| ITM-FE-014 | 単体 | ItemTable | onItemClick: (itemId: string) => void | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemTable | `calls_onItemClick_when_row_clicked` | onItemClick=jest.fn(), items=mockItems | 明細行をクリックすると onItemClick が該当 itemId で呼ばれる |
| ITM-FE-015 | 単体 | ItemTable | onEditItem: (itemId: string) => void | 正常系 | ITM-F02 | 55_ui_component/screens/report-detail.md §ItemTable | `calls_onEditItem_when_edit_button_clicked` | canEditItems=true, onEditItem=jest.fn() | 編集ボタンをクリックすると onEditItem が該当 itemId で呼ばれる |
| ITM-FE-016 | 単体 | ItemTable | onDeleteItem: (itemId: string) => void | 正常系 | ITM-F03 | 55_ui_component/screens/report-detail.md §ItemTable | `calls_onDeleteItem_when_delete_button_clicked` | canEditItems=true, onDeleteItem=jest.fn() | 削除ボタンをクリックすると onDeleteItem が該当 itemId で呼ばれる |
| ITM-FE-017 | 単体 | ItemTable | onEditItem / onDeleteItem | 正常系 | ITM-F02, ITM-F03 | 55_ui_component/screens/report-detail.md §ItemTable | `edit_button_does_not_trigger_row_click` | canEditItems=true, onItemClick=jest.fn(), onEditItem=jest.fn() | 編集ボタンクリック時に onItemClick は呼ばれない（イベント伝播の停止） |

### 10.4 ItemSlidePanel

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ITM-FE-018 | 単体 | ItemSlidePanel | open: boolean | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemSlidePanel | `renders_panel_when_open_true` | open=true, mode='add' | スライドパネルが表示される |
| ITM-FE-019 | 単体 | ItemSlidePanel | open: boolean | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemSlidePanel | `hides_panel_when_open_false` | open=false | スライドパネルが表示されない |
| ITM-FE-020 | 単体 | ItemSlidePanel | mode: PanelMode | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemSlidePanel | `renders_add_mode_title` | open=true, mode='add', item=null | パネルタイトルが追加モードの表記（例: 「明細追加」）で表示される |
| ITM-FE-021 | 単体 | ItemSlidePanel | mode: PanelMode | 正常系 | ITM-F02 | 55_ui_component/screens/report-detail.md §ItemSlidePanel | `renders_edit_mode_title` | open=true, mode='edit', item=mockItem | パネルタイトルが編集モードの表記（例: 「明細編集」）で表示される |
| ITM-FE-022 | 単体 | ItemSlidePanel | mode: PanelMode | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemSlidePanel | `renders_view_mode_title` | open=true, mode='view', item=mockItem | パネルタイトルが閲覧モードの表記（例: 「明細詳細」）で表示される |
| ITM-FE-023 | 単体 | ItemSlidePanel | mode: PanelMode | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemSlidePanel | `view_mode_renders_readonly_form` | open=true, mode='view', item=mockItem | ItemForm が mode='view' で描画され、フォームフィールドが readonly になる |
| ITM-FE-024 | 単体 | ItemSlidePanel | item: ExpenseItem / null | 正常系 | ITM-F02 | 55_ui_component/screens/report-detail.md §ItemSlidePanel | `passes_default_values_in_edit_mode` | open=true, mode='edit', item=mockItem | ItemForm に mockItem の値が defaultValues として渡される |
| ITM-FE-025 | 単体 | ItemSlidePanel | onClose: () => void | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemSlidePanel | `calls_onClose_when_close_button_clicked` | open=true, onClose=jest.fn() | パネルの閉じるボタンをクリックすると onClose が呼ばれる |

### 10.5 ItemForm -- バリデーション

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ITM-FE-026 | 単体 | ItemForm | mode: PanelMode | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemForm | `renders_all_form_fields_in_add_mode` | mode='add', categories=mockCategories | 日付・金額・カテゴリ・摘要の各フィールドが入力可能な状態で表示される |
| ITM-FE-027 | 単体 | ItemForm | mode: PanelMode | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemForm | `renders_all_fields_readonly_in_view_mode` | mode='view', categories=mockCategories, defaultValues=mockItem | 全フィールドが readonly で表示される |
| ITM-FE-028 | 単体 | ItemForm | - | 異常系（V1） | ITM-001 | 55_ui_component/screens/report-detail.md §ItemForm | `shows_validation_error_when_date_empty` | mode='add'。日付を未入力で保存ボタン押下 | 「日付を入力してください」のバリデーションエラーが表示される |
| ITM-FE-029 | 単体 | ItemForm | - | 異常系（V2） | ITM-002 | 55_ui_component/screens/report-detail.md §ItemForm | `shows_validation_error_when_amount_empty` | mode='add'。金額を未入力で保存ボタン押下 | 「金額を入力してください」のバリデーションエラーが表示される |
| ITM-FE-030 | 単体 | ItemForm | - | 異常系（V3） | ITM-002 | 55_ui_component/screens/report-detail.md §ItemForm | `shows_validation_error_when_amount_negative` | mode='add'。金額に -100 を入力 | 「正の金額を入力してください」のバリデーションエラーが表示される |
| ITM-FE-031 | 単体 | ItemForm | - | 異常系（V3） | ITM-002 | 55_ui_component/screens/report-detail.md §ItemForm | `shows_validation_error_when_amount_zero` | mode='add'。金額に 0 を入力 | 「正の金額を入力してください」のバリデーションエラーが表示される |
| ITM-FE-032 | 単体 | ItemForm | - | 異常系（V4） | ITM-002 | 55_ui_component/screens/report-detail.md §ItemForm | `shows_validation_error_when_amount_decimal` | mode='add'。金額に 100.5 を入力 | 「円単位の整数で入力してください」のバリデーションエラーが表示される |
| ITM-FE-033 | 単体 | ItemForm | - | 異常系（V5） | ITM-003 | 55_ui_component/screens/report-detail.md §ItemForm | `shows_validation_error_when_category_not_selected` | mode='add'。カテゴリを未選択で保存ボタン押下 | 「カテゴリを選択してください」のバリデーションエラーが表示される |
| ITM-FE-034 | 単体 | ItemForm | - | 異常系（V6） | ITM-004 | 55_ui_component/screens/report-detail.md §ItemForm | `shows_validation_error_when_description_empty` | mode='add'。摘要を未入力で保存ボタン押下 | 「摘要を入力してください」のバリデーションエラーが表示される |
| ITM-FE-035 | 単体 | ItemForm | - | 異常系（V7） | ITM-004 | 55_ui_component/screens/report-detail.md §ItemForm | `shows_validation_error_when_description_exceeds_500` | mode='add'。摘要に 501 文字の文字列を入力 | 「摘要は500文字以内で入力してください」のバリデーションエラーが表示される |
| ITM-FE-036 | 単体 | ItemForm | - | 境界値（V7） | ITM-004 | 55_ui_component/screens/report-detail.md §ItemForm | `allows_description_with_500_characters` | mode='add'。摘要に 500 文字の文字列を入力して保存ボタン押下 | バリデーションエラーが表示されない（500 文字は許容） |

### 10.6 ItemForm -- 送信・エラー表示

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ITM-FE-037 | 単体 | ItemForm | onSubmit: (data: ItemFormValues) => void | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemForm | `calls_onSubmit_with_valid_data` | mode='add'。全フィールドに有効値を入力して保存ボタン押下 | onSubmit が ItemFormValues 型のデータで呼ばれる（expenseDate, amount, categoryId, description を含む） |
| ITM-FE-038 | 単体 | ItemForm | onSubmit: (data: ItemFormValues) => void | 正常系 | ITM-F02 | 55_ui_component/screens/report-detail.md §ItemForm | `calls_onSubmit_with_edited_data` | mode='edit', defaultValues=mockItem。金額を 2000 に変更して保存ボタン押下 | onSubmit が更新後の ItemFormValues で呼ばれる（amount=2000） |
| ITM-FE-039 | 単体 | ItemForm | onSaveAndContinue: (data: ItemFormValues) => void | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemForm | `shows_save_and_continue_button_in_add_mode` | mode='add' | 「保存して続けて追加」ボタンが表示される |
| ITM-FE-040 | 単体 | ItemForm | onSaveAndContinue: (data: ItemFormValues) => void | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemForm | `calls_onSaveAndContinue_with_valid_data` | mode='add', onSaveAndContinue=jest.fn()。全フィールド有効値入力後「保存して続けて追加」押下 | onSaveAndContinue が ItemFormValues で呼ばれる |
| ITM-FE-041 | 単体 | ItemForm | onSaveAndContinue: (data: ItemFormValues) => void | 正常系 | ITM-F02 | 55_ui_component/screens/report-detail.md §ItemForm | `hides_save_and_continue_button_in_edit_mode` | mode='edit', defaultValues=mockItem | 「保存して続けて追加」ボタンが表示されない |
| ITM-FE-042 | 単体 | ItemForm | onCancel: () => void | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemForm | `calls_onCancel_when_cancel_button_clicked` | mode='add', onCancel=jest.fn() | キャンセルボタン押下で onCancel が呼ばれる |
| ITM-FE-043 | 単体 | ItemForm | apiError: string / null | 異常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemForm | `shows_api_error_in_form_alert` | mode='add', apiError='サーバーエラーが発生しました' | フォーム上部に FormAlert が severity='error' で表示され、エラーメッセージが表示される |
| ITM-FE-044 | 単体 | ItemForm | apiError: string / null | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemForm | `hides_form_alert_when_apiError_null` | mode='add', apiError=null | FormAlert が表示されない |
| ITM-FE-045 | 単体 | ItemForm | isPending: boolean | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemForm | `disables_submit_button_when_isPending_true` | mode='add', isPending=true | 保存ボタンが disabled になる |
| ITM-FE-046 | 単体 | ItemForm | isPending: boolean | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemForm | `does_not_call_onSubmit_when_isPending_true` | mode='add', isPending=true。保存ボタン押下を試みる | onSubmit が呼ばれない |
| ITM-FE-047 | 単体 | ItemForm | categories: Array<{ value: string; label: string }> | 正常系 | ITM-005 | 55_ui_component/screens/report-detail.md §ItemForm | `renders_category_options_from_props` | mode='add', categories=mockCategories（6件） | カテゴリドロップダウンに 6 件の選択肢が表示される |

### 10.6-B ItemForm -- 期間外警告（ITM-007、ConfirmDialog）

ITM-007（明細日付がレポート対象期間外の場合の警告）に対応する ConfirmDialog の表示・挙動を検証する。表示方式は `55_ui_component/state-management.md §6.5.6` / 画面仕様は `50_detail_design/screens/report-detail.md §6「保存時の期間外警告（ITM-007）」` を参照。

**ID 採番の補足**: `ITM-007` はポリシー ID（`policies.md`）として既に使用されており、テスト ID との混同を避けるため、FE テストレンジ（`ITM-FE-***`）の既存最大 ID（`ITM-FE-098-8`）に続く番号を採用する。

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ITM-FE-099 | 単体 | ItemForm | reportPeriodStart / reportPeriodEnd | 警告（ITM-007） | ITM-007 | 50_detail_design/screens/report-detail.md §6「保存時の期間外警告」、55_ui_component/state-management.md §6.5.6 | `shows_confirm_dialog_when_expense_date_before_period_start_on_save` | mode='add', reportPeriodStart='2026-04-01', reportPeriodEnd='2026-04-30', categories=mockCategories。expenseDate='2026-03-15'（開始日より前）、他フィールドは有効値を入力して「保存する」ボタン押下 | ConfirmDialog が表示される（本文に「明細日付がレポートの対象期間外です。入力を確認してください。」）。onSubmit は呼ばれない |
| ITM-FE-100 | 単体 | ItemForm | reportPeriodStart / reportPeriodEnd | 警告（ITM-007） | ITM-007 | 50_detail_design/screens/report-detail.md §6「保存時の期間外警告」 | `shows_confirm_dialog_when_expense_date_after_period_end_on_save` | mode='edit', reportPeriodStart='2026-04-01', reportPeriodEnd='2026-04-30', defaultValues=mockItem。expenseDate='2026-05-05'（終了日より後）に変更して「保存する」ボタン押下 | ConfirmDialog が表示される。onSubmit は呼ばれない |
| ITM-FE-101 | 単体 | ItemForm | onSubmit: (data) => void | 警告（ITM-007） | ITM-007 | 50_detail_design/screens/report-detail.md §6「保存時の期間外警告」 | `confirm_dialog_confirm_button_triggers_save` | mode='add', reportPeriodStart='2026-04-01', reportPeriodEnd='2026-04-30', onSubmit=vi.fn()。expenseDate='2026-03-15' で「保存する」→ ConfirmDialog 表示後に「保存する」ボタン（confirm）を押下 | onSubmit が期間外の入力値を含む ItemFormValues で呼ばれる。ConfirmDialog が閉じる |
| ITM-FE-102 | 単体 | ItemForm | onSubmit: (data) => void | 警告（ITM-007） | ITM-007 | 50_detail_design/screens/report-detail.md §6「保存時の期間外警告」 | `confirm_dialog_cancel_button_preserves_form_and_does_not_save` | mode='add', reportPeriodStart='2026-04-01', reportPeriodEnd='2026-04-30', onSubmit=vi.fn()。expenseDate='2026-03-15', amount=1000, categoryId=<UUID>, description='タクシー代' を入力して「保存する」→ ConfirmDialog 表示後に「キャンセル」ボタンを押下 | onSubmit は呼ばれない。ConfirmDialog が閉じる。フォームの入力値（expenseDate / amount / categoryId / description）が維持されている |
| ITM-FE-103 | 単体 | ItemForm | reportPeriodStart / reportPeriodEnd | 正常系（期間内） | ITM-007 | 50_detail_design/screens/report-detail.md §6「保存時の期間外警告」 | `does_not_show_confirm_dialog_when_expense_date_within_period` | mode='add', reportPeriodStart='2026-04-01', reportPeriodEnd='2026-04-30', onSubmit=vi.fn()。expenseDate='2026-04-15'（期間内）で全フィールド有効値入力後「保存する」押下 | ConfirmDialog は表示されない。onSubmit が即座に呼ばれる |
| ITM-FE-104 | 単体 | ItemForm | reportPeriodStart / reportPeriodEnd | 境界値（ITM-007） | ITM-007 | 50_detail_design/screens/report-detail.md §6「保存時の期間外警告」 | `does_not_show_confirm_dialog_on_period_boundary_dates` | mode='add', reportPeriodStart='2026-04-01', reportPeriodEnd='2026-04-30'。expenseDate='2026-04-01'（開始日）および '2026-04-30'（終了日）で保存押下 | いずれのケースでも ConfirmDialog は表示されず onSubmit が呼ばれる（境界値は期間内扱い） |
| ITM-FE-105 | 単体 | ItemForm | onSaveAndContinue: (data) => void | 警告（ITM-007） | ITM-007 | 50_detail_design/screens/report-detail.md §6「保存時の期間外警告」 | `save_and_continue_triggers_confirm_dialog_for_out_of_period_date` | mode='add', reportPeriodStart='2026-04-01', reportPeriodEnd='2026-04-30', onSaveAndContinue=vi.fn()。expenseDate='2026-03-15' で全フィールド有効値入力後「保存して続けて追加」押下 | ConfirmDialog が表示される。onSaveAndContinue は呼ばれない。確認ボタン押下後に onSaveAndContinue が呼ばれる |
| ITM-FE-106 | 単体 | ItemForm | mode: PanelMode | 警告（ITM-007） | ITM-007 | 50_detail_design/screens/report-detail.md §6「保存時の期間外警告」 | `view_mode_does_not_show_confirm_dialog` | mode='view', reportPeriodStart='2026-04-01', reportPeriodEnd='2026-04-30', defaultValues={expenseDate: '2026-03-15', ...}（期間外） | ConfirmDialog は表示されない（閲覧モードは保存操作が存在しないため対象外） |

### 10.7 Hook テスト -- useCreateItem

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ITM-FE-048 | 単体 | - | useCreateItem | 正常系 | ITM-F01 | 55_ui_component/state-management.md §useCreateItem | `useCreateItem_success_returns_created_item` | MSW で POST /api/reports/:id/items を 201 でモック。mutate({ reportId, expenseDate, amount, categoryId, description }) を呼び出し | data に作成された明細データが含まれる。isPending が false に戻る |
| ITM-FE-049 | 単体 | - | useCreateItem | 正常系 | ITM-F01 | 55_ui_component/state-management.md §useCreateItem | `useCreateItem_success_invalidates_report_detail_cache` | MSW で 201 をモック。mutate 成功後 | queryClient.invalidateQueries が ['reports', 'detail', reportId] で呼ばれる |
| ITM-FE-050 | 単体 | - | useCreateItem | 異常系 | ITM-F01 | 55_ui_component/state-management.md §useCreateItem | `useCreateItem_failure_returns_api_error` | MSW で POST を 422 VALIDATION_ERROR でモック | error に ApiClientError が設定される。isPending が false に戻る |

### 10.8 Hook テスト -- useUpdateItem

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ITM-FE-051 | 単体 | - | useUpdateItem | 正常系 | ITM-F02 | 55_ui_component/state-management.md §useUpdateItem | `useUpdateItem_success_returns_updated_item` | MSW で PUT /api/reports/:id/items/:itemId を 200 でモック。mutate({ reportId, itemId, ...updateData }) を呼び出し | data に更新後の明細データが含まれる |
| ITM-FE-052 | 単体 | - | useUpdateItem | 正常系 | ITM-F02 | 55_ui_component/state-management.md §useUpdateItem | `useUpdateItem_success_invalidates_report_detail_cache` | MSW で 200 をモック。mutate 成功後 | queryClient.invalidateQueries が ['reports', 'detail', reportId] で呼ばれる |
| ITM-FE-053 | 単体 | - | useUpdateItem | 異常系 | ITM-F02 | 55_ui_component/state-management.md §useUpdateItem | `useUpdateItem_failure_conflict_returns_error` | MSW で PUT を 409 CONFLICT でモック | error に ApiClientError（409 Conflict）が設定される |

### 10.9 Hook テスト -- useDeleteItem

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ITM-FE-054 | 単体 | - | useDeleteItem | 正常系 | ITM-F03 | 55_ui_component/state-management.md §useDeleteItem | `useDeleteItem_success_returns_void` | MSW で DELETE /api/reports/:id/items/:itemId を 204 でモック。mutate({ reportId, itemId }) を呼び出し | 正常終了。isPending が false に戻る |
| ITM-FE-055 | 単体 | - | useDeleteItem | 正常系 | ITM-F03 | 55_ui_component/state-management.md §useDeleteItem | `useDeleteItem_success_invalidates_report_detail_cache` | MSW で 204 をモック。mutate 成功後 | queryClient.invalidateQueries が ['reports', 'detail', reportId] で呼ばれる |
| ITM-FE-056 | 単体 | - | useDeleteItem | 異常系 | ITM-F03 | 55_ui_component/state-management.md §useDeleteItem | `useDeleteItem_failure_returns_api_error` | MSW で DELETE を 404 RESOURCE_NOT_FOUND でモック | error に ApiClientError が設定される |

### 10.10 Hook テスト -- useCategories

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ITM-FE-057 | 単体 | - | useCategories | 正常系 | ITM-005 | 55_ui_component/state-management.md §useCategories | `useCategories_success_returns_category_list` | MSW で GET /api/categories を 200 でモック（6件） | data に 6 件のカテゴリ配列が含まれる |
| ITM-FE-058 | 単体 | - | useCategories | 正常系 | ITM-005 | 55_ui_component/state-management.md §useCategories | `useCategories_uses_infinite_stale_time` | MSW で 200 をモック。2 回目のレンダリング | 2 回目の API 呼び出しが発生しない（staleTime: Infinity によりキャッシュが有効） |
| ITM-FE-059 | 単体 | - | useCategories | 異常系 | ITM-005 | 55_ui_component/state-management.md §useCategories | `useCategories_failure_returns_error` | MSW で GET /api/categories を 500 でモック | error が設定され、data は undefined |

---

## 11. FE ドメイン不変条件サマリー

本ファイルの FE テストケースがカバーする不変条件・バリデーションルールの対応表。

| 不変条件 / ルールID | 内容 | カバーするテストID |
|---|---|---|
| V1 | 日付は空でないこと | ITM-FE-028 |
| V2 | 金額は空でないこと | ITM-FE-029 |
| V3 | 金額は正の整数であること | ITM-FE-030, ITM-FE-031 |
| V4 | 金額は整数であること（小数不可） | ITM-FE-032 |
| V5 | カテゴリは選択されていること | ITM-FE-033 |
| V6 | 摘要は空でないこと | ITM-FE-034 |
| V7 | 摘要は 500 文字以内 | ITM-FE-035, ITM-FE-036 |
| ITM-010 | draft 以外での明細操作 UI 非表示 | ITM-FE-004, ITM-FE-008, ITM-FE-013 |
| ITM-005 | カテゴリは固定 6 種類 | ITM-FE-047, ITM-FE-057 |
| ITM-007 | 明細日付が対象期間外の場合、警告（ConfirmDialog）を表示するが保存は許可する | ITM-FE-099, ITM-FE-100, ITM-FE-101, ITM-FE-102, ITM-FE-103, ITM-FE-104, ITM-FE-105, ITM-FE-106 |

---

## 12. FE 実装ガイド

### 12.1 テストファイル配置

```
expense-saas/
  frontend/
    src/
      pages/
        reports/
          __tests__/
            ItemListSection.test.tsx   -- ITM-FE-001 ~ 004
            ItemListHeader.test.tsx    -- ITM-FE-005 ~ 009
            ItemTable.test.tsx         -- ITM-FE-010 ~ 017
            ItemSlidePanel.test.tsx    -- ITM-FE-018 ~ 025
            ItemForm.test.tsx          -- ITM-FE-026 ~ 047, ITM-FE-099 ~ 106
      hooks/
        __tests__/
          useItems.test.tsx            -- ITM-FE-048 ~ 056
          useCategories.test.tsx       -- ITM-FE-057 ~ 059
```

### 12.2 テスト実行コマンド

```bash
# 明細コンポーネントテスト
npx vitest run src/pages/reports/__tests__/Item*.test.tsx

# 明細 Hook テスト
npx vitest run src/hooks/__tests__/useItems.test.tsx src/hooks/__tests__/useCategories.test.tsx

# 全 FE 明細テスト
npx vitest run --reporter=verbose src/pages/reports/__tests__/Item* src/hooks/__tests__/useItems* src/hooks/__tests__/useCategories*
```

### 12.3 追記テスト（UX 修正・view/edit モード）

以下のテスト ID は実装済みテストコードとの整合のために追記する。

#### ItemSlidePanel -- view/edit モード追加テスト

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ITM-FE-091-A | 単体 | ItemSlidePanel | mode: PanelMode | 認可 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemSlidePanel | `view_mode_hides_attachment_uploader` | open=true, mode='view', isOwner=true, reportStatus='draft', item=mockItem | mode=view のとき AttachmentUploader が表示されない（canModify=false） |
| ITM-FE-091-B | 単体 | ItemSlidePanel | mode: PanelMode | 正常系 | ITM-F02 | 55_ui_component/screens/report-detail.md §ItemSlidePanel | `edit_mode_shows_attachment_area` | open=true, mode='edit', isOwner=true, reportStatus='draft', item=mockItem | mode=edit のとき canModify=true で AttachmentArea が描画される（対照ケース） |
| ITM-FE-092-A | 単体 | ItemSlidePanel | onClose: () => void | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemSlidePanel | `esc_key_calls_onClose` | open=true, onClose=jest.fn() | ESC キー押下で onClose が呼ばれる（MUI Drawer の標準挙動） |
| ITM-FE-092-B | 単体 | ItemSlidePanel | onClose: () => void | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemSlidePanel | `backdrop_click_calls_onClose` | open=true, onClose=jest.fn() | Backdrop クリックで onClose が呼ばれる（MUI Drawer の標準挙動） |

#### ItemSlidePanel / ItemForm / ReportDetailPage -- UX 修正テスト

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| ITM-FE-098-1 | 単体 | ItemSlidePanel | open: boolean | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemSlidePanel | `drawer_width_responsive` | open=true | Drawer Paper に width スタイル sx が設定されている（xs: '100%', sm: 480px） |
| ITM-FE-098-1b | 単体 | ItemSlidePanel | open: boolean | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemSlidePanel | `drawer_paper_props_sm_width` | open=true | ItemSlidePanel が Drawer に渡す PaperProps.sx の sm 幅が 480 である |
| ITM-FE-098-2 | 単体 | ItemSlidePanel | open: boolean | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemSlidePanel | `close_button_in_header` | open=true, onClose=jest.fn() | ヘッダー右上に aria-label="閉じる" の IconButton が存在し、クリックで onClose が呼ばれる |
| ITM-FE-098-3 | 単体 | ItemSlidePanel | open: boolean | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemSlidePanel | `open_false_to_true_shows_drawer` | open=false から open=true に変更 | Drawer が表示される |
| ITM-FE-098-3b | 単体 | ReportDetailPage | useReport | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ReportDetailPage | `item_click_flushSync_panel_reopen` | 明細行をクリック後、編集中に別の明細行をクリック | flushSync pattern でパネルが一度 'closed' を経由してから再度開く |
| ITM-FE-098-4 | 単体 | ItemSlidePanel | mode: PanelMode | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ItemSlidePanel | `view_mode_readonly_fields` | mode='view', item=mockItem | ItemForm 内の全フィールドが readOnly 状態になる（disabled ではない） |
| ITM-FE-098-5 | 単体 | ReportDetailPage | useReport | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ReportDetailPage | `row_click_then_add_maintains_add_mode` | 行クリック直後に明細追加ボタンを押す | 追加モードが維持される（flushSync 置換により setTimeout の race が解消） |
| ITM-FE-098-6 | 単体 | ReportDetailPage | useReport | 正常系 | ITM-F01 | 55_ui_component/screens/report-detail.md §ReportDetailPage | `edit_click_then_row_click_shows_view` | 編集クリック直後に別の行をクリック | 最新の行の閲覧モードが維持される（flushSync 置換により setTimeout の race が解消） |
| ITM-FE-098-7 | 単体 | ItemForm | mode: PanelMode | 正常系 | ITM-005 | 55_ui_component/screens/report-detail.md §ItemForm | `view_mode_category_select_not_openable` | mode='view', categories=mockCategories, defaultValues=mockItem | mode=view のときカテゴリ Select をクリックしても listbox が開かない（MUI Select readOnly 制御の回帰テスト） |
| ITM-FE-098-8 | 単体 | ItemForm | mode: PanelMode | 正常系 | ITM-005 | 55_ui_component/screens/report-detail.md §ItemForm | `add_mode_category_select_openable` | mode='add', categories=mockCategories | mode=add のときカテゴリ Select をクリックすると listbox が開く（対照ケース） |

---

### 12.4 テスト記述ガイド（旧 12.3）

**コンポーネントテスト**:
- `@testing-library/react` の `render` / `screen` / `userEvent` を使用する
- Props のモックは各テストで明示的に指定する。コールバック Props は `vi.fn()` でモックする
- 共通コンポーネント（AppDataGrid, FormAlert 等）は実物を使用し、モックしない

**Hook テスト**:
- `@testing-library/react` の `renderHook` を使用する
- API モックは MSW（Mock Service Worker）で定義する
- TanStack Query の `QueryClient` はテストごとに新規作成し、`QueryClientProvider` でラップする
- キャッシュ無効化の検証は `queryClient.getQueryState` または `queryClient.isFetching` で確認する

### 12.4 フィクスチャの注意事項

- テストデータに機密情報を含めない
- テナント ID は単一テナント（テナント A）のみ使用し、テナント間の混在を避ける
- カテゴリデータは固定 6 種類のマスタデータとしてモックする
