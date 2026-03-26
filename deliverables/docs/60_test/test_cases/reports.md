# テストケース一覧: 経費レポート（RPT-）

## この文書の役割

| 項目 | 内容 |
|------|------|
| 目的 | 経費レポート CRUD・提出エンドポイント（/api/reports/*）のテストケースを定義する |
| 正本情報 | RPT-F01〜F07 に対応するテストケース一覧 |
| 扱わない内容 | 明細・添付・ワークフローの詳細テスト、テナント横断テスト（→ cross-cutting.md） |
| 主な参照元 | `50_detail_design/openapi.yaml#/api/reports/*`, `20_domain/state_machine.md`, `10_requirements/requirements.md#RPT-F*` |
| 主な参照先 | `60_test/traceability.md`, テスト実装（Step 9） |

---

## 概要

本ファイルは経費レポート（`/api/reports/*`）に関するテストケースを定義する。

対応実装ファイル:
- `report_handler_test.go` — ハンドラ層統合テスト
- `domain/report_test.go` — ドメイン層単体テスト（状態遷移）

### 責務境界

本ファイルに含めるテスト:
- `GET /api/reports` — 自分のレポート一覧（RBAC・フィルタ）
- `POST /api/reports` — レポート作成（通常・再申請）
- `GET /api/reports/all` — 全レポート一覧（RBAC: Admin + Accounting のみ）
- `GET /api/reports/{id}` — レポート詳細（RBAC・閲覧範囲）
- `PUT /api/reports/{id}` — レポート更新（RBAC・所有権・状態）
- `DELETE /api/reports/{id}` — レポート削除（RBAC・所有権・状態）
- `POST /api/reports/{id}/submit` — レポート提出（RBAC・所有権・状態遷移 T1・X1〜X4）
- ドメイン層: 状態遷移 T1〜T5 全許可遷移 + X1〜X10 全禁止遷移

本ファイルに含めないテスト:
- テナント分離テスト（クロステナントアクセス）→ `cross-cutting.md`
- T2（approve）・T3（reject）・T4（pay）・X5〜X10 → `workflow.md`

---

## フィクスチャ参照（test_strategy.md §4）

### テナントAユーザー

| ロール | ユーザーID |
|--------|-----------|
| Admin | `aaaaaaaa-1111-1111-1111-000000000001` |
| Approver | `aaaaaaaa-2222-2222-2222-000000000002` |
| Member | `aaaaaaaa-3333-3333-3333-000000000003` |
| Accounting | `aaaaaaaa-4444-4444-4444-000000000004` |

### 経費レポートフィクスチャ（テナントA・作成者: Test Member）

| フィクスチャ名 | 状態 | レポートID |
|-------------|------|----------|
| `report_draft` | `draft` | `cccccccc-0001-0001-0001-000000000001` |
| `report_draft_empty` | `draft` | `cccccccc-0001-0001-0001-000000000002` |
| `report_submitted` | `submitted` | `cccccccc-0002-0002-0002-000000000002` |
| `report_approved` | `approved` | `cccccccc-0003-0003-0003-000000000003` |
| `report_rejected` | `rejected` | `cccccccc-0004-0004-0004-000000000004` |
| `report_paid` | `paid` | `cccccccc-0005-0005-0005-000000000005` |

---

## テストケース一覧

### 1. GET /api/reports — 自分のレポート一覧（listMyReports）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|---------|---------|-----------|-----------|--------------|-----------------|---------|
| RPT-001 | 統合 | handler | 正常系 | RPT-F02 | openapi.yaml#listMyReports | `TestListMyReports_Success` | Actor: Test Member（テナントA）。自分のレポートが3件存在 | 200 OK。自分の3件が返る。他ユーザーのレポートは含まれない |
| RPT-002 | 統合 | handler | 正常系 | RPT-F02 | openapi.yaml#listMyReports | `TestListMyReports_StatusFilter` | Actor: Test Member。`status=draft` クエリパラメータ | 200 OK。draft 状態のレポートのみ返る |
| RPT-003 | 統合 | handler | 正常系 | RPT-F02 | openapi.yaml#listMyReports | `TestListMyReports_EmptyResult` | Actor: Test Member。自分のレポートが0件 | 200 OK。`data: []`、`pagination` 含む |
| RPT-004 | 統合 | handler | 異常系 | RPT-F02 | openapi.yaml#listMyReports | `TestListMyReports_Unauthorized` | 認証トークンなし | 401 UNAUTHORIZED |
| RPT-005 | 統合 | handler | 認可 | RPT-F02, RBC-010 | openapi.yaml#listMyReports, authz.md#4.1 | `TestListMyReports_RBAC_Approver` | Actor: Test Approver（テナントA）。自分のレポートが1件 | 200 OK。自分の1件が返る（Approver は申請系操作可能: RBC-010） |
| RPT-006 | 統合 | handler | 認可 | RPT-F02, RBC-010 | openapi.yaml#listMyReports, authz.md#4.1 | `TestListMyReports_RBAC_Accounting` | Actor: Test Accounting（テナントA）。自分のレポートが1件 | 200 OK。自分の1件が返る |
| RPT-007 | 統合 | handler | 認可 | RPT-F02, RBC-010 | openapi.yaml#listMyReports, authz.md#4.1 | `TestListMyReports_RBAC_Admin` | Actor: Test Admin（テナントA）。自分のレポートが1件 | 200 OK。自分の1件が返る |

---

### 2. POST /api/reports — レポート作成（createReport）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|---------|---------|-----------|-----------|--------------|-----------------|---------|
| RPT-008 | 統合 | handler | 正常系 | RPT-F01 | openapi.yaml#createReport, db_schema.md#expense_reports | `TestCreateReport_Success` | Actor: Test Member。リクエスト: `title="出張費 3月"`, `period_from="2026-03-01"`, `period_to="2026-03-31"` | 201 Created。`status=draft`、`user_id=Test Member ID`、`tenant_id=テナントA ID` がセットされた ExpenseReportDetail を返す |
| RPT-009 | 統合 | handler | 異常系 | RPT-F01, RPT-001 | openapi.yaml#createReport | `TestCreateReport_ValidationError_EmptyTitle` | Actor: Test Member。`title=""` | 422 VALIDATION_ERROR（title は必須） |
| RPT-010 | 統合 | handler | 異常系 | RPT-F01, RPT-003 | openapi.yaml#createReport, db_schema.md#expense_reports#CHECK | `TestCreateReport_ValidationError_PeriodRange` | Actor: Test Member。`period_from="2026-03-31"`, `period_to="2026-03-01"`（終了が開始より前） | 422 VALIDATION_ERROR |
| RPT-011 | 統合 | handler | 異常系 | RPT-F01 | openapi.yaml#createReport | `TestCreateReport_Unauthorized` | 認証トークンなし | 401 UNAUTHORIZED |
| RPT-012 | 統合 | handler | 認可 | RPT-F01, RBC-010 | openapi.yaml#createReport, authz.md#4.1 | `TestCreateReport_RBAC_Approver` | Actor: Test Approver。有効なリクエストボディ | 201 Created（Approver は申請系操作可能） |
| RPT-013 | 統合 | handler | 認可 | RPT-F01, RBC-014 | openapi.yaml#createReport, authz.md#4.4 | `TestCreateReport_RBAC_Admin` | Actor: Test Admin。有効なリクエストボディ | 201 Created（Admin は自分のレポート作成可能: RBC-014） |

---

### 3. POST /api/reports — 再申請（createReport with reference_report_id）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|---------|---------|-----------|-----------|--------------|-----------------|---------|
| RPT-014 | 統合 | handler | 正常系 | RPT-F01, RPT-015, RPT-016 | openapi.yaml#createReport, db_schema.md#expense_reports.reference_report_id | `TestCreateReport_Resubmit_Success` | Actor: Test Member。`reference_report_id=report_rejected のID`。`report_rejected` は Test Member 所有のテナントA レポート | 201 Created。新規レポートが `status=draft`、`reference_report_id` がセットされ、元レポートの明細がコピーされた状態で返る |
| RPT-015 | 単体 | domain | 正常系 | RPT-015 | db_schema.md#expense_reports | `TestCreateReport_Resubmit_OriginalStatusUnchanged` | ドメイン層: 再申請処理後に元レポートの状態を確認 | 元の `report_rejected` の状態が `rejected` のまま変わらない（RPT-015） |
| RPT-016 | 単体 | domain | 正常系 | RPT-016 | db_schema.md#expense_reports.reference_report_id | `TestCreateReport_Resubmit_ReferenceIdSet` | ドメイン層: 再申請で作成した新規レポートを検査 | 新規レポートに `reference_report_id == 元レポートID` がセットされている（RPT-016） |
| RPT-017 | 単体 | domain | 正常系 | RPT-F01 | openapi.yaml#createReport | `TestCreateReport_Resubmit_AttachmentsNotCopied` | ドメイン層: 元レポートの明細に添付ファイルが紐付いている状態で再申請 | 新規レポートの明細には添付ファイルがコピーされていない（添付 0 件） |
| RPT-018 | 統合 | handler | 正常系 | RPT-F01, RPT-015 | openapi.yaml#createReport | `TestCreateReport_Resubmit_ItemsCopied` | Actor: Test Member。`reference_report_id` 指定。元レポートに明細2件あり | 201 Created。新規レポートに明細が2件コピーされている |
| RPT-019 | 統合 | handler | 異常系 | RPT-F01, RPT-015 | openapi.yaml#createReport | `TestCreateReport_Resubmit_NonRejectedSourceFails` | Actor: Test Member。`reference_report_id` に `report_draft`（draft 状態）を指定 | 422 VALIDATION_ERROR（再申請元は rejected 状態のみ許可） |

---

### 4. GET /api/reports/all — 全レポート一覧（listAllReports）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|---------|---------|-----------|-----------|--------------|-----------------|---------|
| RPT-020 | 統合 | handler | 正常系 | RPT-F07, RBC-013 | openapi.yaml#listAllReports, authz.md#4.4 | `TestListAllReports_Admin_Success` | Actor: Test Admin（テナントA）。テナントA に複数ユーザーのレポートが存在 | 200 OK。テナントA 全レポートが返る（`submitter` フィールド含む）。自分以外のレポートも含む |
| RPT-021 | 統合 | handler | 正常系 | RPT-F07 | openapi.yaml#listAllReports | `TestListAllReports_Accounting_Success` | Actor: Test Accounting（テナントA） | 200 OK。テナントA 全レポートが返る |
| RPT-022 | 統合 | handler | 認可 | RPT-F07, RBC-013 | openapi.yaml#listAllReports, authz.md#4.4 | `TestListAllReports_Member_Forbidden` | Actor: Test Member（テナントA） | 403 FORBIDDEN（Member は全レポート閲覧不可） |
| RPT-023 | 統合 | handler | 認可 | RPT-F07, RBC-013 | openapi.yaml#listAllReports, authz.md#4.4 | `TestListAllReports_Approver_Forbidden` | Actor: Test Approver（テナントA） | 403 FORBIDDEN（Approver は全レポート閲覧不可） |
| RPT-024 | 統合 | handler | 異常系 | RPT-F07 | openapi.yaml#listAllReports | `TestListAllReports_Unauthorized` | 認証トークンなし | 401 UNAUTHORIZED |
| RPT-025 | 統合 | handler | 正常系 | RPT-F07 | openapi.yaml#listAllReports | `TestListAllReports_StatusFilter` | Actor: Test Admin。`status=submitted` クエリパラメータ | 200 OK。submitted 状態のレポートのみ返る |
| RPT-026 | 統合 | handler | 正常系 | RPT-F07 | openapi.yaml#listAllReports | `TestListAllReports_SubmitterFilter` | Actor: Test Admin。`submitter_id=Test Member ID` クエリパラメータ | 200 OK。Test Member が作成したレポートのみ返る |

---

### 5. GET /api/reports/{id} — レポート詳細（getReport）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|---------|---------|-----------|-----------|--------------|-----------------|---------|
| RPT-027 | 統合 | handler | 正常系 | RPT-F03 | openapi.yaml#getReport | `TestGetReport_Owner_Success` | Actor: Test Member（所有者）。`report_draft` | 200 OK。レポート詳細（items + attachments ネスト）が返る |
| RPT-028 | 統合 | handler | 認可 | RPT-F03, RBC-010 | openapi.yaml#getReport, authz.md#4.1 | `TestGetReport_Member_OtherUserDraft_Forbidden` | Actor: Test Member。Test Approver が所有する `draft` レポートにアクセス | 403 FORBIDDEN（Member は他者の draft は閲覧不可: RBC-010） |
| RPT-029 | 統合 | handler | 認可 | RPT-F03, RBC-011 | openapi.yaml#getReport, authz.md#4.2 | `TestGetReport_Approver_OtherUserSubmitted_Success` | Actor: Test Approver。Test Member 所有の `report_submitted` にアクセス | 200 OK（Approver は submitted 状態の他者レポートを閲覧可能: RBC-011） |
| RPT-030 | 統合 | handler | 認可 | RPT-F03, RBC-011 | openapi.yaml#getReport, authz.md#4.2 | `TestGetReport_Approver_OtherUserDraft_Forbidden` | Actor: Test Approver。Test Member 所有の `report_draft` にアクセス（承認/却下した履歴なし） | 403 FORBIDDEN（Approver は他者の draft は閲覧不可） |
| RPT-031 | 統合 | handler | 認可 | RPT-F03 | openapi.yaml#getReport, authz.md#4.3 | `TestGetReport_Accounting_OtherUserDraft_Success` | Actor: Test Accounting。Test Member 所有の `report_draft` にアクセス | 200 OK（Accounting はテナント内全レポート閲覧可能） |
| RPT-032 | 統合 | handler | 認可 | RPT-F03, RBC-013 | openapi.yaml#getReport, authz.md#4.4 | `TestGetReport_Admin_OtherUserDraft_Success` | Actor: Test Admin。Test Member 所有の `report_draft` にアクセス | 200 OK（Admin はテナント内全レポート閲覧可能: RBC-013） |
| RPT-033 | 統合 | handler | 異常系 | RPT-F03 | openapi.yaml#getReport | `TestGetReport_NotFound` | Actor: Test Member。存在しないレポートID | 404 RESOURCE_NOT_FOUND |
| RPT-034 | 統合 | handler | 異常系 | RPT-F03 | openapi.yaml#getReport | `TestGetReport_Unauthorized` | 認証トークンなし | 401 UNAUTHORIZED |

---

### 6. PUT /api/reports/{id} — レポート更新（updateReport）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|---------|---------|-----------|-----------|--------------|-----------------|---------|
| RPT-035 | 統合 | handler | 正常系 | RPT-F04, RPT-011 | openapi.yaml#updateReport | `TestUpdateReport_Success` | Actor: Test Member（所有者）。`report_draft`。リクエスト: `title="更新タイトル"`, `updated_at=現在値` | 200 OK。更新後のレポートが返る |
| RPT-036 | 統合 | handler | 認可 | RPT-F04, RBC-010, RPT-012 | openapi.yaml#updateReport, authz.md#4.1 | `TestUpdateReport_NotOwner_Forbidden` | Actor: Test Approver（非所有者）。`report_draft`（Test Member 所有） | 403 FORBIDDEN（所有者以外は更新不可: RBC-010） |
| RPT-037 | 統合 | handler | 状態遷移 | RPT-F04, RPT-011 | openapi.yaml#updateReport | `TestUpdateReport_NotDraft_Unprocessable` | Actor: Test Member（所有者）。`report_submitted`（submitted 状態） | 422 REPORT_NOT_EDITABLE（draft 以外は編集不可） |
| RPT-038 | 統合 | handler | 認可 | RPT-F04, RBC-014 | openapi.yaml#updateReport, authz.md#4.4 | `TestUpdateReport_Admin_NotOwner_Forbidden` | Actor: Test Admin（非所有者）。`report_draft`（Test Member 所有） | 403 FORBIDDEN（Admin であっても他者の更新不可: RBC-014） |
| RPT-039 | 統合 | handler | 異常系 | RPT-F04 | openapi.yaml#updateReport | `TestUpdateReport_Conflict_OptimisticLock` | Actor: Test Member（所有者）。`report_draft`。`updated_at` が古い値（楽観的ロック競合） | 409 CONFLICT |
| RPT-040 | 統合 | handler | 異常系 | RPT-F04 | openapi.yaml#updateReport | `TestUpdateReport_NotFound` | Actor: Test Member。存在しないレポートID | 404 RESOURCE_NOT_FOUND |
| RPT-041 | 統合 | handler | 異常系 | RPT-F04 | openapi.yaml#updateReport | `TestUpdateReport_Unauthorized` | 認証トークンなし | 401 UNAUTHORIZED |

---

### 7. DELETE /api/reports/{id} — レポート削除（deleteReport）

#### 7.1 ハンドラ層統合テスト

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|---------|---------|-----------|-----------|--------------|-----------------|---------|
| RPT-042 | 統合 | handler | 正常系 | RPT-F05, RPT-013, NFR-DATA-001 | openapi.yaml#deleteReport | `TestDeleteReport_Success` | Actor: Test Member（所有者）。`report_draft` | 204 No Content。`report_draft` が論理削除される（`deleted_at` がセット）。関連明細・添付も論理削除 |
| RPT-043 | 統合 | handler | 認可 | RPT-F05, RBC-010 | openapi.yaml#deleteReport, authz.md#4.1 | `TestDeleteReport_NotOwner_Forbidden` | Actor: Test Approver（非所有者）。`report_draft`（Test Member 所有） | 403 FORBIDDEN |
| RPT-044 | 統合 | handler | 認可 | RPT-F05, RBC-014 | openapi.yaml#deleteReport, authz.md#4.4 | `TestDeleteReport_Admin_NotOwner_Forbidden` | Actor: Test Admin（非所有者）。`report_draft`（Test Member 所有） | 403 FORBIDDEN（Admin であっても他者の削除不可: RBC-014） |
| RPT-045 | 統合 | handler | 状態遷移 | RPT-F05, RPT-013, WFL-015 | openapi.yaml#deleteReport, state_machine.md#T5 | `TestDeleteReport_Submitted_Unprocessable` | Actor: Test Member（所有者）。`report_submitted`（T5 禁止遷移） | 422 REPORT_NOT_DELETABLE（state_machine.md §T5 事前条件 No.1: status == draft の違反） |
| RPT-046 | 統合 | handler | 異常系 | RPT-F05 | openapi.yaml#deleteReport | `TestDeleteReport_NotFound` | Actor: Test Member。存在しないレポートID | 404 RESOURCE_NOT_FOUND |
| RPT-047 | 統合 | handler | 異常系 | RPT-F05 | openapi.yaml#deleteReport | `TestDeleteReport_Unauthorized` | 認証トークンなし | 401 UNAUTHORIZED |

#### 7.2 ドメイン層単体テスト（T5: draft → 削除）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|---------|---------|-----------|-----------|--------------|-----------------|---------|
| RPT-048 | 単体 | domain | 状態遷移 | RPT-F05, WFL-015 | state_machine.md#T5 | `TestReport_Delete_DraftSuccess` | ドメイン: `report.status = draft` の状態で Delete() を呼ぶ | エラーなし。`deleted_at` がセットされる（T5） |
| RPT-049 | 単体 | domain | 状態遷移 | RPT-013, WFL-015 | state_machine.md#T5 | `TestReport_Delete_SubmittedFails` | ドメイン: `report.status = submitted` の状態で Delete() を呼ぶ | `ReportNotDeletable` エラー（state_machine.md §T5 事前条件 No.1: status == draft の違反） |
| RPT-050 | 単体 | domain | 状態遷移 | RPT-013, WFL-015 | state_machine.md#T5 | `TestReport_Delete_ApprovedFails` | ドメイン: `report.status = approved` の状態で Delete() を呼ぶ | `ReportNotDeletable` エラー（state_machine.md §T5 事前条件 No.1: status == draft の違反） |
| RPT-051 | 単体 | domain | 状態遷移 | RPT-013, WFL-015 | state_machine.md#T5 | `TestReport_Delete_RejectedFails` | ドメイン: `report.status = rejected` の状態で Delete() を呼ぶ | `ReportNotDeletable` エラー（state_machine.md §T5 事前条件 No.1: status == draft の違反） |
| RPT-052 | 単体 | domain | 状態遷移 | RPT-013, WFL-015 | state_machine.md#T5 | `TestReport_Delete_PaidFails` | ドメイン: `report.status = paid` の状態で Delete() を呼ぶ | `ReportNotDeletable` エラー（state_machine.md §T5 事前条件 No.1: status == draft の違反） |

---

### 8. POST /api/reports/{id}/submit — レポート提出（submitReport）

#### 8.1 ハンドラ層統合テスト（T1: draft → submitted）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|---------|---------|-----------|-----------|--------------|-----------------|---------|
| RPT-053 | 統合 | handler | 正常系 | RPT-F06, WFL-010 | openapi.yaml#submitReport, state_machine.md#T1 | `TestSubmitReport_Success` | Actor: Test Member（所有者）。`report_draft`（明細1件あり）。テナントA に Approver 1人存在。`updated_at=現在値` | 200 OK。`status=submitted`、`submitted_at` および `submitted_by` がセットされたレポートが返る（T1） |
| RPT-054 | 統合 | handler | 認可 | RPT-F06, RBC-010 | openapi.yaml#submitReport, authz.md#4.1 | `TestSubmitReport_NotOwner_Forbidden` | Actor: Test Approver（非所有者）。`report_draft`（Test Member 所有） | 403 FORBIDDEN |
| RPT-055 | 統合 | handler | 状態遷移 | RPT-F06, WFL-002 | openapi.yaml#submitReport, state_machine.md#X4 | `TestSubmitReport_NotDraft_Unprocessable` | Actor: Test Member（所有者）。`report_submitted`（すでに提出済み） | 422 INVALID_STATE_TRANSITION（X4: submitted → submitted は禁止遷移） |
| RPT-056 | 統合 | handler | 異常系 | RPT-F06, RPT-014 | openapi.yaml#submitReport | `TestSubmitReport_EmptyReport_Unprocessable` | Actor: Test Member（所有者）。`report_draft_empty`（明細0件）。テナントA に Approver 存在 | 422 EMPTY_REPORT_SUBMISSION（明細なし: RPT-014） |
| RPT-057 | 統合 | handler | 異常系 | RPT-F06, WFL-014 | openapi.yaml#submitReport | `TestSubmitReport_NoApproverInTenant_Unprocessable` | Actor: Test Member（所有者）。`report_draft`（明細1件）。テナント内に Approver が0人（WFL-014） | 422 NO_APPROVER_IN_TENANT |
| RPT-058 | 統合 | handler | 異常系 | RPT-F06 | openapi.yaml#submitReport | `TestSubmitReport_Conflict_OptimisticLock` | Actor: Test Member（所有者）。`report_draft`。`updated_at` が古い値 | 409 CONFLICT |
| RPT-059 | 統合 | handler | 異常系 | RPT-F06 | openapi.yaml#submitReport | `TestSubmitReport_NotFound` | Actor: Test Member。存在しないレポートID | 404 RESOURCE_NOT_FOUND |
| RPT-060 | 統合 | handler | 異常系 | RPT-F06 | openapi.yaml#submitReport | `TestSubmitReport_Unauthorized` | 認証トークンなし | 401 UNAUTHORIZED |
| RPT-061 | 統合 | handler | 状態遷移 | WFL-002 | state_machine.md#X1 | `TestSubmitReport_X1_DirectApprove_Unprocessable` | Actor: Test Member（所有者）。`report_draft` に対して直接 approve 操作（submit をスキップして状態を approved に変更しようとする場合のハンドラ側検証） | 422 INVALID_STATE_TRANSITION（X1: draft → approved は禁止） |
| RPT-062 | 統合 | handler | 状態遷移 | WFL-002 | state_machine.md#X2 | `TestSubmitReport_X2_DirectReject_Unprocessable` | Actor: Test Member（所有者）。`report_draft` に対して直接 reject 操作（状態を rejected に変更しようとする場合） | 422 INVALID_STATE_TRANSITION（X2: draft → rejected は禁止） |
| RPT-063 | 統合 | handler | 状態遷移 | WFL-002 | state_machine.md#X3 | `TestSubmitReport_X3_DirectPay_Unprocessable` | Actor: Test Member（所有者）。`report_draft` に対して直接 pay 操作（状態を paid に変更しようとする場合） | 422 INVALID_STATE_TRANSITION（X3: draft → paid は禁止） |
| RPT-064 | 統合 | handler | 状態遷移 | WFL-002 | state_machine.md#X4 | `TestSubmitReport_X4_RevertToDraft_Unprocessable` | Actor: Test Member（所有者）。`report_submitted` に対して draft に戻す操作を試みる | 422 INVALID_STATE_TRANSITION（X4: submitted → draft は禁止） |

#### 8.2 ドメイン層単体テスト（T1: draft → submitted）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|---------|---------|-----------|-----------|--------------|-----------------|---------|
| RPT-065 | 単体 | domain | 状態遷移 | RPT-F06, WFL-010 | state_machine.md#T1 | `TestReport_Submit_DraftSuccess` | ドメイン: `report.status = draft`、items.count = 1、has_approver = true | エラーなし。`status=submitted`、`submitted_at` と `submitted_by` がセット。`total_amount` が再計算される（T1 事後処理） |
| RPT-066 | 単体 | domain | 異常系 | RPT-F06, RPT-014 | state_machine.md#T1 | `TestReport_Submit_EmptyItemsFails` | ドメイン: `report.status = draft`、items.count = 0 | `EmptyReportSubmission` エラー（RPT-014） |
| RPT-067 | 単体 | domain | 状態遷移 | WFL-001, WFL-002 | state_machine.md#T1 | `TestReport_Submit_AlreadySubmittedFails` | ドメイン: `report.status = submitted` | `InvalidStateTransition` エラー（state_machine.md §T1 事前条件 No.1: status == draft の違反） |
| RPT-068 | 単体 | domain | 状態遷移 | WFL-001, WFL-002 | state_machine.md#T1 | `TestReport_Submit_ApprovedFails` | ドメイン: `report.status = approved` | `InvalidStateTransition` エラー（state_machine.md §T1 事前条件 No.1: status == draft の違反） |
| RPT-069 | 単体 | domain | 状態遷移 | WFL-001, WFL-002 | state_machine.md#T1 | `TestReport_Submit_RejectedFails` | ドメイン: `report.status = rejected` | `InvalidStateTransition` エラー（state_machine.md §T1 事前条件 No.1: status == draft の違反） |
| RPT-070 | 単体 | domain | 状態遷移 | WFL-001, WFL-002 | state_machine.md#T1 | `TestReport_Submit_PaidFails` | ドメイン: `report.status = paid` | `InvalidStateTransition` エラー（state_machine.md §T1 事前条件 No.1: status == draft の違反） |
| RPT-071 | 単体 | domain | 異常系 | WFL-014 | state_machine.md#T1 | `TestReport_Submit_NoApproverInTenantFails` | ドメイン: `report.status = draft`、items.count = 1、has_approver = false | `NoApproverInTenant` エラー（WFL-014） |
| RPT-072 | 単体 | domain | 正常系 | RPT-006, RPT-F06 | state_machine.md#T1 | `TestReport_Submit_TotalAmountRecalculated` | ドメイン: `report.status = draft`、明細2件（1000円 + 2000円） | 提出後 `total_amount = 3000`（T1 事後処理 RPT-006） |

---

### 9. ドメイン層単体テスト — 禁止遷移（X1〜X10）

X1〜X4 はハンドラ層統合テストでも検証するが、ドメイン層での拒否ロジックは単体テストで直接確認する。

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|---------|---------|-----------|-----------|--------------|-----------------|---------|
| RPT-073 | 単体 | domain | 状態遷移 | WFL-002 | state_machine.md#X1 | `TestReport_Transition_X1_DraftToApproved` | ドメイン: `report.status = draft` で Approve() を呼ぶ | `InvalidStateTransition` エラー（X1: draft → approved は禁止） |
| RPT-074 | 単体 | domain | 状態遷移 | WFL-002 | state_machine.md#X2 | `TestReport_Transition_X2_DraftToRejected` | ドメイン: `report.status = draft` で Reject() を呼ぶ | `InvalidStateTransition` エラー（X2: draft → rejected は禁止） |
| RPT-075 | 単体 | domain | 状態遷移 | WFL-002 | state_machine.md#X3 | `TestReport_Transition_X3_DraftToPaid` | ドメイン: `report.status = draft` で MarkAsPaid() を呼ぶ | `InvalidStateTransition` エラー（X3: draft → paid は禁止） |
| RPT-076 | 単体 | domain | 状態遷移 | WFL-002 | state_machine.md#X4 | `TestReport_Transition_X4_SubmittedToDraft` | ドメイン: `report.status = submitted` で状態を draft に戻す遷移を呼ぶ | `InvalidStateTransition` エラー（X4: submitted → draft は禁止） |
| RPT-077 | 単体 | domain | 状態遷移 | WFL-002 | state_machine.md#X5 | `TestReport_Transition_X5_SubmittedToPaid` | ドメイン: `report.status = submitted` で MarkAsPaid() を呼ぶ | `InvalidStateTransition` エラー（X5: submitted → paid は禁止） |
| RPT-078 | 単体 | domain | 状態遷移 | WFL-002 | state_machine.md#X6 | `TestReport_Transition_X6_ApprovedToDraft` | ドメイン: `report.status = approved` で状態を draft に戻す遷移を呼ぶ | `InvalidStateTransition` エラー（X6: approved → draft は禁止） |
| RPT-079 | 単体 | domain | 状態遷移 | WFL-002 | state_machine.md#X7 | `TestReport_Transition_X7_ApprovedToSubmitted` | ドメイン: `report.status = approved` で Submit() を呼ぶ | `InvalidStateTransition` エラー（X7: approved → submitted は禁止） |
| RPT-080 | 単体 | domain | 状態遷移 | WFL-002 | state_machine.md#X8 | `TestReport_Transition_X8_ApprovedToRejected` | ドメイン: `report.status = approved` で Reject() を呼ぶ | `InvalidStateTransition` エラー（X8: approved → rejected は禁止） |
| RPT-081 | 単体 | domain | 状態遷移 | WFL-004 | state_machine.md#X9 | `TestReport_Transition_X9_RejectedToAny` | ドメイン: `report.status = rejected` で Submit()・Approve()・Reject()・MarkAsPaid() のいずれかを呼ぶ（4パターン） | 全て `InvalidStateTransition` エラー（X9: rejected は終端状態） |
| RPT-082 | 単体 | domain | 状態遷移 | WFL-004 | state_machine.md#X10 | `TestReport_Transition_X10_PaidToAny` | ドメイン: `report.status = paid` で Submit()・Approve()・Reject()・MarkAsPaid() のいずれかを呼ぶ（4パターン） | 全て `InvalidStateTransition` エラー（X10: paid は終端状態） |

---

### 10. ドメイン層単体テスト — 許可遷移 T2〜T4（ドメインロジック確認）

T2（approve）・T3（reject）・T4（pay）の統合テストは `workflow.md` 担当だが、ドメイン層のメソッド自体は `domain/report_test.go` で検証する。

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|-----------|---------|---------|-----------|-----------|--------------|-----------------|---------|
| RPT-083 | 単体 | domain | 状態遷移 | WFL-F01, WFL-011 | state_machine.md#T2 | `TestReport_Approve_SubmittedSuccess` | ドメイン: `report.status = submitted`、`actor.id != report.user_id` | エラーなし。`status=approved`、`approved_at` と `approved_by` がセット（T2 事後処理） |
| RPT-084 | 単体 | domain | 認可 | RBC-016 | state_machine.md#T2, authz.md#4.2 | `TestReport_Approve_SelfApprovalFails` | ドメイン: `report.status = submitted`、`actor.id == report.user_id` | `SelfApprovalNotAllowed` エラー（RBC-016） |
| RPT-085 | 単体 | domain | 正常系 | WFL-F01 | state_machine.md#T2 | `TestReport_Approve_WithComment` | ドメイン: `report.status = submitted`、`approval_comment="承認コメント"` | エラーなし。`approval_comment` がセットされる（T2 入力項目） |
| RPT-086 | 単体 | domain | 状態遷移 | WFL-F02, WFL-012 | state_machine.md#T3 | `TestReport_Reject_SubmittedSuccess` | ドメイン: `report.status = submitted`、`actor.id != report.user_id`、`rejection_reason="理由"` | エラーなし。`status=rejected`、`rejected_at`・`rejected_by`・`rejection_reason` がセット（T3 事後処理） |
| RPT-087 | 単体 | domain | 異常系 | WFL-012 | state_machine.md#T3 | `TestReport_Reject_EmptyReasonFails` | ドメイン: `report.status = submitted`、`rejection_reason=""` | `MissingRejectionReason` エラー（WFL-012） |
| RPT-088 | 単体 | domain | 認可 | RBC-016 | state_machine.md#T3, authz.md#4.2 | `TestReport_Reject_SelfRejectionFails` | ドメイン: `report.status = submitted`、`actor.id == report.user_id`、`rejection_reason="理由"` | `SelfApprovalNotAllowed` エラー（RBC-016） |
| RPT-089 | 単体 | domain | 状態遷移 | WFL-F03, WFL-013 | state_machine.md#T4 | `TestReport_MarkAsPaid_ApprovedSuccess` | ドメイン: `report.status = approved`、`actor.id != report.user_id` | エラーなし。`status=paid`、`paid_at` と `paid_by` がセット（T4 事後処理） |
| RPT-090 | 単体 | domain | 認可 | RBC-012 | state_machine.md#T4, authz.md#4.3 | `TestReport_MarkAsPaid_SelfPaymentFails` | ドメイン: `report.status = approved`、`actor.id == report.user_id` | `SelfPaymentNotAllowed` エラー（RBC-012） |

---

## テストID一覧（サマリー）

| 範囲 | テストID範囲 | 件数 |
|------|------------|------|
| listMyReports（RBAC・基本動作） | RPT-001〜RPT-007 | 7 |
| createReport（通常作成） | RPT-008〜RPT-013 | 6 |
| createReport（再申請） | RPT-014〜RPT-019 | 6 |
| listAllReports（RBAC・フィルタ） | RPT-020〜RPT-026 | 7 |
| getReport（閲覧範囲・RBAC） | RPT-027〜RPT-034 | 8 |
| updateReport（所有権・状態・楽観的ロック） | RPT-035〜RPT-041 | 7 |
| deleteReport ハンドラ統合 | RPT-042〜RPT-047 | 6 |
| deleteReport ドメイン単体（T5） | RPT-048〜RPT-052 | 5 |
| submitReport ハンドラ統合（T1・X1〜X4） | RPT-053〜RPT-064 | 12 |
| submitReport ドメイン単体（T1） | RPT-065〜RPT-072 | 8 |
| 禁止遷移 X1〜X10 ドメイン単体 | RPT-073〜RPT-082 | 10 |
| 許可遷移 T2〜T4 ドメイン単体 | RPT-083〜RPT-090 | 8 |
| **合計** | **RPT-001〜RPT-090** | **90** |

---

## 実装ガイド

### テストファイル構成

```
expense-saas/
  internal/
    domain/
      report_test.go          # RPT-048〜RPT-052, RPT-065〜RPT-090（単体テスト）
    handler/
      report_handler_test.go  # RPT-001〜RPT-047, RPT-053〜RPT-064（統合テスト）
```

### 単体テスト実装方針（domain/report_test.go）

DBを使わず、ドメインオブジェクトを直接構築してメソッドを呼ぶ。

```go
// 例: T1 成功パス（RPT-065）
func TestReport_Submit_DraftSuccess(t *testing.T) {
    report := &ExpenseReport{
        Status: "draft",
        Items:  []ExpenseItem{{Amount: 1000}},
    }
    actorID := uuid.MustParse("aaaaaaaa-3333-3333-3333-000000000003")
    hasApprover := true

    err := report.Submit(actorID, hasApprover)

    assert.NoError(t, err)
    assert.Equal(t, "submitted", report.Status)
    assert.NotNil(t, report.SubmittedAt)
    assert.Equal(t, actorID, report.SubmittedBy)
}
```

### 統合テスト実装方針（report_handler_test.go）

`net/http/httptest` を使い、実際のルーターを通してリクエストを送る。テスト用PostgreSQLが必要（`-tags=integration`）。

```go
// 例: T1 ハンドラ成功パス（RPT-053）
func TestSubmitReport_Success(t *testing.T) {
    // テスト前: フィクスチャを投入（report_draft + Test Member トークン）
    token := testhelper.LoginAs(t, "member") // Test Member のJWTを取得
    body := `{"updated_at": "2026-03-01T00:00:00Z"}`

    resp := testhelper.POST(t, token,
        "/api/reports/cccccccc-0001-0001-0001-000000000001/submit",
        body)

    assert.Equal(t, 200, resp.StatusCode)
    var result map[string]interface{}
    json.NewDecoder(resp.Body).Decode(&result)
    assert.Equal(t, "submitted", result["data"].(map[string]interface{})["status"])
}
```

### フィクスチャのセットアップ

統合テストでは `TestMain` でフィクスチャを投入する。

```go
func TestMain(m *testing.M) {
    testDB := testhelper.SetupDB()       // Docker の test DB に接続
    testhelper.Truncate(testDB)          // 全テーブルを TRUNCATE
    testhelper.InsertFixtures(testDB)    // test_strategy.md §4 のフィクスチャを投入
    os.Exit(m.Run())
}
```

### テスト実行コマンド

```bash
# ドメイン層単体テスト（RPT-048〜RPT-090）
go test ./internal/domain/... -v -run TestReport -cover

# ハンドラ層統合テスト（RPT-001〜RPT-064、DBが必要）
go test ./internal/handler/... -v -tags=integration -run TestListMyReports
go test ./internal/handler/... -v -tags=integration -run TestCreateReport
go test ./internal/handler/... -v -tags=integration -run TestListAllReports
go test ./internal/handler/... -v -tags=integration -run TestGetReport
go test ./internal/handler/... -v -tags=integration -run TestUpdateReport
go test ./internal/handler/... -v -tags=integration -run TestDeleteReport
go test ./internal/handler/... -v -tags=integration -run TestSubmitReport

# 全テスト一括実行
go test ./... -tags=integration
```
