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

### BE テストID一覧

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
| **BE 合計** | **RPT-001〜RPT-090** | **90** |

### FE テストID一覧

| 範囲 | テストID範囲 | 件数 |
|------|------------|------|
| レポート一覧画面 コンポーネント | RPT-FE-001〜RPT-FE-020 | 20 |
| レポート一覧画面 Hook（useMyReports） | RPT-FE-021〜RPT-FE-023 | 3 |
| レポート作成画面 コンポーネント | RPT-FE-024〜RPT-FE-045 | 22 |
| レポート作成画面 Hook（useCreateReport） | RPT-FE-046〜RPT-FE-049 | 4 |
| レポート編集画面 コンポーネント | RPT-FE-050〜RPT-FE-058 | 9 |
| レポート編集画面 Hook（useReport, useUpdateReport） | RPT-FE-059〜RPT-FE-063 | 5 |
| レポート詳細画面 レポート閲覧・操作コンポーネント | RPT-FE-064〜RPT-FE-096 | 33 |
| レポート詳細画面 Hook（useSubmitReport, useDeleteReport） | RPT-FE-097〜RPT-FE-102 | 6 |
| **FE 合計** | **RPT-FE-001〜RPT-FE-102, RPT-FE-090-A** | **103** |

### 全体合計

| 区分 | 件数 |
|------|------|
| BE テストケース | 90 |
| FE テストケース | 103 |
| **総合計** | **193** |

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

---

### FE テストケース

FE テストケースは `55_ui_component/screens/*.md` のコンポーネント単位・Props 単位で導出する。

**設計識別子の規約**: 「対応設計ID」欄には `55_ui_component/screens/{screen-name}.md §{ComponentName}` の形式で記載する。Hook のテストの場合は `55_ui_component/state-management.md §{useHookName}` の形式で記載する。

#### FE-1. レポート一覧画面（SCR-RPT-001）

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| RPT-FE-001 | 単体 | ReportListPage | useMyReports | 正常系 | RPT-F02 | 55_ui_component/screens/report-list.md §ReportListPage | `test_ReportListPage_renders_report_list` | useMyReports が 3 件のレポートデータを返す | ReportListHeader, ReportListFilter, ReportListTable, AppPagination が描画される |
| RPT-FE-002 | 単体 | ReportListPage | useMyReports | 正常系 | RPT-F02 | 55_ui_component/screens/report-list.md §ReportListPage | `test_ReportListPage_restores_filters_from_url` | URL クエリパラメータ `?status=draft&from=2026-03-01&to=2026-03-31` が設定されている | ReportListFilter にフィルタ値が反映される。useMyReports にフィルタパラメータが渡される |
| RPT-FE-003 | 単体 | ReportListPage | useMyReports | 正常系 | RPT-F02 | 55_ui_component/screens/report-list.md §ReportListPage | `test_ReportListPage_resets_page_on_filter_change` | ページ 2 を表示中にステータスフィルタを変更 | URL クエリパラメータの page が 1 にリセットされる |
| RPT-FE-004 | 単体 | ReportListPage | onRowClick: (reportId: string) => void | 正常系 | RPT-F03 | 55_ui_component/screens/report-list.md §ReportListPage | `test_ReportListPage_navigates_to_detail_on_row_click` | テーブル行をクリック（reportId = "test-id-001"） | `/reports/test-id-001` に遷移する |
| RPT-FE-005 | 単体 | ReportListPage | onCreateReport: () => void | 正常系 | RPT-F01 | 55_ui_component/screens/report-list.md §ReportListPage | `test_ReportListPage_navigates_to_create` | レポート作成ボタンをクリック | `/reports/new` に遷移する |
| RPT-FE-006 | 単体 | ReportListPage | useMyReports | 正常系 | RPT-F02 | 55_ui_component/screens/report-list.md §ReportListPage | `test_ReportListPage_shows_skeleton_while_loading` | useMyReports の isLoading が true | PageSkeleton（variant: 'table'）が表示される |
| RPT-FE-007 | 単体 | ReportListPage | useMyReports | 異常系 | RPT-F02 | 55_ui_component/screens/report-list.md §ReportListPage | `test_ReportListPage_shows_toast_on_api_error` | useMyReports がエラーを返す | AppToast（severity: 'error'）が表示される |
| RPT-FE-008 | 単体 | ReportListHeader | onCreateReport: () => void | 正常系 | RPT-F02 | 55_ui_component/screens/report-list.md §ReportListHeader | `test_ReportListHeader_renders_title_and_button` | onCreateReport コールバックを渡す | 「マイレポート」タイトルと「+ レポート作成」ボタンが描画される |
| RPT-FE-009 | 単体 | ReportListHeader | onCreateReport: () => void | 正常系 | RPT-F01 | 55_ui_component/screens/report-list.md §ReportListHeader | `test_ReportListHeader_calls_onCreateReport` | 「+ レポート作成」ボタンをクリック | onCreateReport コールバックが呼び出される |
| RPT-FE-010 | 単体 | CreateReportButton | onClick: () => void | 正常系 | RPT-F01 | 55_ui_component/screens/report-list.md §CreateReportButton | `test_CreateReportButton_renders_and_calls_onClick` | onClick コールバックを渡してボタンをクリック | ボタンが描画され onClick コールバックが呼び出される |
| RPT-FE-011 | 単体 | ReportListFilter | values: ReportListFilterValues | 正常系 | RPT-F02 | 55_ui_component/screens/report-list.md §ReportListFilter | `test_ReportListFilter_renders_filter_controls` | values = { status: '', from: '', to: '' } | ステータスフィルタ（AppSelect）と日付ピッカー（AppDatePicker x 2）が描画される |
| RPT-FE-012 | 単体 | ReportListFilter | onFilterChange: (values: ReportListFilterValues) => void | 正常系 | RPT-F02 | 55_ui_component/screens/report-list.md §ReportListFilter | `test_ReportListFilter_calls_onFilterChange_on_status` | ステータスフィルタで「下書き」を選択 | onFilterChange が { status: 'draft', from: '', to: '' } で呼び出される |
| RPT-FE-013 | 単体 | ReportListFilter | onFilterChange: (values: ReportListFilterValues) => void | 正常系 | RPT-F02 | 55_ui_component/screens/report-list.md §ReportListFilter | `test_ReportListFilter_calls_onFilterChange_on_date` | 開始日に「2026-03-01」を入力 | onFilterChange が from: '2026-03-01' を含む値で呼び出される（デバウンス後） |
| RPT-FE-014 | 単体 | ReportListFilter | values: ReportListFilterValues | 正常系 | RPT-F02 | 55_ui_component/screens/report-list.md §ReportListFilter | `test_ReportListFilter_shows_status_options` | ステータスフィルタのドロップダウンを開く | 「全て」「下書き」「提出済み」「承認済み」「却下」「支払済み」の選択肢が表示される |

**備考**: RPT-FE-011〜014 は `ReportListFilter` 単体テストとして設計されているが、実装では `ReportListPage.test.tsx` のページ統合テスト内でフィルタ操作を検証している（RPT-FE-001〜003 に包含）。独立した `ReportListFilter.test.tsx` は存在しない。
| RPT-FE-015 | 単体 | ReportListTable | reports: ReportListItem[] | 正常系 | RPT-F02 | 55_ui_component/screens/report-list.md §ReportListTable | `test_ReportListTable_renders_columns` | reports に 2 件のデータを渡す | タイトル・対象期間・合計金額・ステータス・作成日カラムが描画される |
| RPT-FE-016 | 単体 | ReportListTable | reports: ReportListItem[] | 正常系 | RPT-F02 | 55_ui_component/screens/report-list.md §ReportListTable | `test_ReportListTable_formats_amount_with_commas` | reports に totalAmount: 1234567 のデータを渡す | 金額が「1,234,567」と 3 桁カンマ区切りで表示される |
| RPT-FE-017 | 単体 | ReportListTable | reports: ReportListItem[] | 正常系 | RPT-F02 | 55_ui_component/screens/report-list.md §ReportListTable | `test_ReportListTable_renders_status_chip` | reports に status: 'submitted' のデータを渡す | StatusChip が「提出済み」として描画される |
| RPT-FE-018 | 単体 | ReportListTable | onRowClick: (reportId: string) => void | 正常系 | RPT-F03 | 55_ui_component/screens/report-list.md §ReportListTable | `test_ReportListTable_calls_onRowClick` | 行をクリック | onRowClick がクリックした行の reportId で呼び出される |
| RPT-FE-019 | 単体 | ReportListTable | reports: ReportListItem[], onCreateReport: () => void | 正常系 | RPT-F02 | 55_ui_component/screens/report-list.md §ReportListTable | `test_ReportListTable_shows_empty_state` | reports = []（0 件） | EmptyState が「経費レポートはまだありません。レポートを作成して経費精算を始めましょう。」メッセージとレポート作成ボタンで表示される |
| RPT-FE-020 | 単体 | ReportListTable | loading: boolean | 正常系 | RPT-F02 | 55_ui_component/screens/report-list.md §ReportListTable | `test_ReportListTable_passes_loading_to_grid` | loading = true | AppDataGrid に loading = true が渡される |
| RPT-FE-021 | 単体 | useMyReports | useMyReports | 正常系 | RPT-F02 | 55_ui_component/state-management.md §useMyReports | `test_useMyReports_fetches_data` | params = { page: 1, per_page: 20 } | GET /api/reports?page=1&per_page=20 を呼び出し、data にレポート一覧、pagination に件数情報が格納される |
| RPT-FE-022 | 単体 | useMyReports | useMyReports | 正常系 | RPT-F02 | 55_ui_component/state-management.md §useMyReports | `test_useMyReports_sends_filter_params` | params = { status: 'draft', from: '2026-03-01', to: '2026-03-31' } | GET /api/reports?status=draft&from=2026-03-01&to=2026-03-31 が送信される |
| RPT-FE-023 | 単体 | useMyReports | useMyReports | 異常系 | RPT-F02 | 55_ui_component/state-management.md §useMyReports | `test_useMyReports_handles_error` | API が 500 エラーを返す | isError = true、error にエラー情報が格納される |

---

#### FE-2. レポート作成画面（SCR-RPT-002）

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| RPT-FE-024 | 単体 | ReportCreatePage | useCreateReport | 正常系 | RPT-F01 | 55_ui_component/screens/report-create.md §ReportCreatePage | `test_ReportCreatePage_submit_success_navigates_to_detail` | フォームに有効な値を入力して送信。useCreateReport が成功（id: "new-id-001"）を返す | `/reports/new-id-001` に遷移する |
| RPT-FE-025 | 単体 | ReportCreatePage | useCreateReport | 異常系 | RPT-F01 | 55_ui_component/screens/report-create.md §ReportCreatePage | `test_ReportCreatePage_submit_error_shows_alert` | useCreateReport がサーバーエラーを返す | ReportForm に apiError が渡され FormAlert にエラーメッセージが表示される |
| RPT-FE-026 | 単体 | ReportCreatePage | useReport | 正常系 | RPT-F01, RPT-015 | 55_ui_component/screens/report-create.md §ReportCreatePage | `test_ReportCreatePage_prefills_from_ref` | URL パラメータ `?ref=rejected-report-id`。useReport が元レポートデータ（title, periodStart, periodEnd）を返す | ReportForm の defaultValues に元レポートのデータがプリフィルされる |
| RPT-FE-027 | 単体 | ReportCreatePage | onCancel: () => void | 正常系 | RPT-F01 | 55_ui_component/screens/report-create.md §ReportCreatePage | `test_ReportCreatePage_cancel_navigates_to_list` | キャンセルボタンをクリック | `/reports` に遷移する |
| RPT-FE-028 | 単体 | ReportForm | onSubmit: (data: ReportFormValues) => void | 正常系 | RPT-F01 | 55_ui_component/screens/report-create.md §ReportForm | `test_ReportForm_submits_valid_data` | title = "出張費 3月"、periodStart = "2026-03-01"、periodEnd = "2026-03-31" を入力して送信 | onSubmit が { title: "出張費 3月", periodStart: "2026-03-01", periodEnd: "2026-03-31" } で呼び出される |
| RPT-FE-029 | 単体 | ReportForm | onSubmit: (data: ReportFormValues) => void | 異常系 | RPT-F01, RPT-001 | 55_ui_component/screens/report-create.md §ReportForm | `test_ReportForm_validates_empty_title` | title = "" で送信 | バリデーションエラー表示。onSubmit は呼び出されない |
| RPT-FE-030 | 単体 | ReportForm | onSubmit: (data: ReportFormValues) => void | 異常系 | RPT-F01, RPT-002 | 55_ui_component/screens/report-create.md §ReportForm | `test_ReportForm_validates_title_max_length` | title = 101 文字の文字列で送信 | バリデーションエラー表示。onSubmit は呼び出されない |
| RPT-FE-031 | 単体 | ReportForm | onSubmit: (data: ReportFormValues) => void | 異常系 | RPT-F01, RPT-003 | 55_ui_component/screens/report-create.md §ReportForm | `test_ReportForm_validates_period_range` | periodStart = "2026-03-31"、periodEnd = "2026-03-01"（終了が開始より前）で送信 | バリデーションエラー表示。onSubmit は呼び出されない |
| RPT-FE-032 | 単体 | ReportForm | onSubmit: (data: ReportFormValues) => void | 異常系 | RPT-F01 | 55_ui_component/screens/report-create.md §ReportForm | `test_ReportForm_validates_empty_period_start` | periodStart = "" で送信 | バリデーションエラー表示。onSubmit は呼び出されない |
| RPT-FE-033 | 単体 | ReportForm | onSubmit: (data: ReportFormValues) => void | 異常系 | RPT-F01 | 55_ui_component/screens/report-create.md §ReportForm | `test_ReportForm_validates_empty_period_end` | periodEnd = "" で送信 | バリデーションエラー表示。onSubmit は呼び出されない |
| RPT-FE-034 | 単体 | ReportForm | isPending: boolean | 正常系 | RPT-F01 | 55_ui_component/screens/report-create.md §ReportForm | `test_ReportForm_disables_fields_when_pending` | isPending = true | 全入力フィールドと送信ボタンが disabled になる |
| RPT-FE-035 | 単体 | ReportForm | apiError: string \| null | 正常系 | RPT-F01 | 55_ui_component/screens/report-create.md §ReportForm | `test_ReportForm_shows_api_error` | apiError = "サーバーエラーが発生しました" | FormAlert にエラーメッセージが表示される |
| RPT-FE-036 | 単体 | ReportForm | defaultValues: ReportFormValues | 正常系 | RPT-F01, RPT-015 | 55_ui_component/screens/report-create.md §ReportForm | `test_ReportForm_prefills_default_values` | defaultValues = { title: "元レポート", periodStart: "2026-03-01", periodEnd: "2026-03-31" } | フォームフィールドに defaultValues の値がプリフィルされる |
| RPT-FE-037 | 単体 | FormAlert | message: string \| null | 正常系 | RPT-F01 | 55_ui_component/screens/report-create.md §FormAlert | `test_FormAlert_renders_message` | message = "エラーが発生しました" | Alert コンポーネントにメッセージが表示される |
| RPT-FE-038 | 単体 | FormAlert | message: string \| null | 正常系 | RPT-F01 | 55_ui_component/screens/report-create.md §FormAlert | `test_FormAlert_hidden_when_null` | message = null | コンポーネントが描画されない（非表示） |
| RPT-FE-039 | 単体 | ReportPeriodField | control: Control, periodStartError?: string, periodEndError?: string | 正常系 | RPT-F01 | 55_ui_component/screens/report-create.md §ReportPeriodField | `test_ReportPeriodField_renders_date_pickers` | React Hook Form の control を渡す | 開始日・終了日の AppDatePicker と「~」区切りテキストが描画される |
| RPT-FE-040 | 単体 | ReportPeriodField | periodStartError?: string | 異常系 | RPT-F01 | 55_ui_component/screens/report-create.md §ReportPeriodField | `test_ReportPeriodField_shows_start_error` | periodStartError = "開始日は必須です" | 開始日の AppDatePicker にエラーメッセージが表示される |
| RPT-FE-041 | 単体 | ReportPeriodField | periodEndError?: string | 異常系 | RPT-F01 | 55_ui_component/screens/report-create.md §ReportPeriodField | `test_ReportPeriodField_shows_end_error` | periodEndError = "終了日は開始日以降にしてください" | 終了日の AppDatePicker にエラーメッセージが表示される |
| RPT-FE-042 | 単体 | ReportPeriodField | disabled?: boolean | 正常系 | RPT-F01 | 55_ui_component/screens/report-create.md §ReportPeriodField | `test_ReportPeriodField_disables_pickers` | disabled = true | 両方の AppDatePicker が disabled になる |
| RPT-FE-043 | 単体 | ReportFormActions | submitLabel: string, loading: boolean, onCancel: () => void | 正常系 | RPT-F01 | 55_ui_component/screens/report-create.md §ReportFormActions | `test_ReportFormActions_renders_buttons` | submitLabel = "作成する"、loading = false | 「作成する」送信ボタンと「キャンセル」ボタンが描画される |
| RPT-FE-044 | 単体 | ReportFormActions | loading: boolean | 正常系 | RPT-F01 | 55_ui_component/screens/report-create.md §ReportFormActions | `test_ReportFormActions_shows_loading` | loading = true | 送信ボタンが disabled + スピナー表示になる |
| RPT-FE-045 | 単体 | ReportFormActions | onCancel: () => void | 正常系 | RPT-F01 | 55_ui_component/screens/report-create.md §ReportFormActions | `test_ReportFormActions_calls_onCancel` | キャンセルボタンをクリック | onCancel コールバックが呼び出される |
| RPT-FE-046 | 単体 | useCreateReport | useCreateReport | 正常系 | RPT-F01 | 55_ui_component/state-management.md §useCreateReport | `test_useCreateReport_calls_api` | mutate({ title: "テスト", periodStart: "2026-03-01", periodEnd: "2026-03-31" }) | POST /api/reports が呼び出され、data に作成されたレポートが格納される |
| RPT-FE-047 | 単体 | useCreateReport | useCreateReport | 正常系 | RPT-F01, RPT-015 | 55_ui_component/state-management.md §useCreateReport | `test_useCreateReport_sends_reference_id` | mutate に reference_report_id を含むリクエスト | POST /api/reports のリクエストボディに reference_report_id が含まれる |
| RPT-FE-048 | 単体 | useCreateReport | useCreateReport | 正常系 | RPT-F01 | 55_ui_component/state-management.md §useCreateReport | `test_useCreateReport_invalidates_cache` | ミューテーション成功後 | レポート一覧のクエリキャッシュが無効化される |
| RPT-FE-049 | 単体 | useCreateReport | useCreateReport | 異常系 | RPT-F01 | 55_ui_component/state-management.md §useCreateReport | `test_useCreateReport_handles_validation_error` | API が 422 VALIDATION_ERROR を返す | isError = true、error にバリデーションエラー情報が格納される |

---

#### FE-3. レポート編集画面（SCR-RPT-003）

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| RPT-FE-050 | 単体 | ReportEditPage | useReport, useUpdateReport | 正常系 | RPT-F04 | 55_ui_component/screens/report-edit.md §ReportEditPage | `test_ReportEditPage_prefills_existing_data` | useReport が既存レポート（title: "既存タイトル", periodStart: "2026-03-01", periodEnd: "2026-03-31", status: "draft"）を返す | ReportForm に defaultValues として既存データがプリフィルされる |
| RPT-FE-051 | 単体 | ReportEditPage | useUpdateReport | 正常系 | RPT-F04 | 55_ui_component/screens/report-edit.md §ReportEditPage | `test_ReportEditPage_save_success_navigates_to_detail` | フォームに有効な値を入力して送信。useUpdateReport が成功を返す | `/reports/:id` に遷移する |
| RPT-FE-052 | 単体 | ReportEditPage | useReport | 認可 | RPT-F04, RPT-012 | 55_ui_component/screens/report-edit.md §ReportEditPage | `test_ReportEditPage_redirects_when_not_found` | useReport が 404 エラーを返す | `/reports` にリダイレクトされトーストが表示される |
| RPT-FE-053 | 単体 | ReportEditPage | useReport, useCurrentUser | 認可 | RPT-F04, RPT-012 | 55_ui_component/screens/report-edit.md §ReportEditPage | `test_ReportEditPage_shows_403_when_not_owner` | useReport がレポートを返す（submitter.id != currentUser.id） | 403 トーストが表示される |
| RPT-FE-054 | 単体 | ReportEditPage | useReport | 認可 | RPT-F04, RPT-011 | 55_ui_component/screens/report-edit.md §ReportEditPage | `test_ReportEditPage_redirects_when_not_draft` | useReport が status: "submitted" のレポートを返す | `/reports/:id` にリダイレクトされトーストが表示される |
| RPT-FE-055 | 単体 | ReportEditPage | useUpdateReport | 異常系 | RPT-F04 | 55_ui_component/screens/report-edit.md §ReportEditPage | `test_ReportEditPage_handles_409_conflict` | useUpdateReport が 409 CONFLICT を返す | FormAlert に「他のユーザーがこのレポートを更新しました。ページを再読み込みしてください。」が表示される |
| RPT-FE-056 | 単体 | ReportEditPage | onCancel: () => void | 正常系 | RPT-F04 | 55_ui_component/screens/report-edit.md §ReportEditPage | `test_ReportEditPage_cancel_navigates_to_detail` | キャンセルボタンをクリック | `/reports/:id` に遷移する |
| RPT-FE-057 | 単体 | ReportEditPage | useReport | 正常系 | RPT-F04 | 55_ui_component/screens/report-edit.md §ReportEditPage | `test_ReportEditPage_shows_skeleton_while_loading` | useReport の isLoading が true | PageSkeleton（variant: 'form'）が表示される |
| RPT-FE-058 | 単体 | ReportForm | submitLabel: string | 正常系 | RPT-F04 | 55_ui_component/screens/report-edit.md §ReportForm | `test_ReportForm_renders_save_label` | submitLabel = "保存する" | 送信ボタンのラベルが「保存する」と表示される |
| RPT-FE-059 | 単体 | useReport | useReport | 正常系 | RPT-F03 | 55_ui_component/state-management.md §useReport | `test_useReport_fetches_data` | reportId = "test-report-id" | GET /api/reports/test-report-id を呼び出し、data にレポート詳細が格納される |
| RPT-FE-060 | 単体 | useReport | useReport | 異常系 | RPT-F03 | 55_ui_component/state-management.md §useReport | `test_useReport_handles_404` | API が 404 RESOURCE_NOT_FOUND を返す | isError = true、error に 404 エラー情報が格納される |
| RPT-FE-061 | 単体 | useUpdateReport | useUpdateReport | 正常系 | RPT-F04 | 55_ui_component/state-management.md §useUpdateReport | `test_useUpdateReport_sends_updated_at` | mutate({ id: "test-id", title: "更新", periodStart: "2026-03-01", periodEnd: "2026-03-31", updated_at: "2026-03-01T00:00:00Z" }) | PUT /api/reports/test-id のリクエストボディに updated_at が含まれる |
| RPT-FE-062 | 単体 | useUpdateReport | useUpdateReport | 正常系 | RPT-F04 | 55_ui_component/state-management.md §useUpdateReport | `test_useUpdateReport_invalidates_cache` | ミューテーション成功後 | レポート詳細・一覧のクエリキャッシュが無効化される |
| RPT-FE-063 | 単体 | useUpdateReport | useUpdateReport | 異常系 | RPT-F04 | 55_ui_component/state-management.md §useUpdateReport | `test_useUpdateReport_handles_409_conflict` | API が 409 CONFLICT を返す | isError = true、error に 409 コンフリクト情報が格納される |

---

#### FE-4. レポート詳細画面（SCR-RPT-004）-- レポート閲覧・操作コンポーネント

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| RPT-FE-064 | 単体 | ReportDetailPage | useReport | 正常系 | RPT-F03 | 55_ui_component/screens/report-detail.md §ReportDetailPage | `test_ReportDetailPage_renders_report_detail` | useReport が draft 状態のレポート詳細データを返す | ReportInfoCard, ReportActionBar が描画される |
| RPT-FE-065 | 単体 | ReportDetailPage | useReport | 正常系 | RPT-F03 | 55_ui_component/screens/report-detail.md §ReportDetailPage | `test_ReportDetailPage_shows_skeleton_while_loading` | useReport の isLoading が true | PageSkeleton が表示される |
| RPT-FE-066 | 単体 | ReportDetailPage | useReport | 異常系 | RPT-F03 | 55_ui_component/screens/report-detail.md §ReportDetailPage | `test_ReportDetailPage_shows_not_found` | useReport が 404 を返す | 「指定されたデータが見つかりません。」メッセージとレポート一覧へのリンクが表示される |
| RPT-FE-067 | 単体 | ReportDetailPage | useSubmitReport | 正常系 | RPT-F06 | 55_ui_component/screens/report-detail.md §ReportDetailPage | `test_ReportDetailPage_submit_dialog_confirm` | 提出ボタン押下後、確認ダイアログで「はい」を選択 | useSubmitReport が実行され、成功後にレポートデータが更新される |
| RPT-FE-068 | 単体 | ReportDetailPage | useDeleteReport | 正常系 | RPT-F05 | 55_ui_component/screens/report-detail.md §ReportDetailPage | `test_ReportDetailPage_delete_dialog_confirm` | 削除ボタン押下後、確認ダイアログで「はい」を選択 | useDeleteReport が実行され、成功後に `/reports` に遷移する |
| RPT-FE-069 | 単体 | ReportDetailPage | ConfirmDialog | 正常系 | RPT-F06 | 55_ui_component/screens/report-detail.md §ReportDetailPage | `test_ReportDetailPage_dialog_cancel` | 提出確認ダイアログで「キャンセル」を選択 | ダイアログが閉じ、ミューテーションは実行されない |
| RPT-FE-070 | 単体 | ReportInfoCard | report: ExpenseReportDetail | 正常系 | RPT-F03 | 55_ui_component/screens/report-detail.md §ReportInfoCard | `test_ReportInfoCard_renders_basic_and_workflow` | report にレポート詳細データを渡す | ReportBasicInfo と ReportWorkflowInfo が描画される |
| RPT-FE-071 | 単体 | ReportBasicInfo | title: string, status: ReportStatus | 正常系 | RPT-F03 | 55_ui_component/screens/report-detail.md §ReportBasicInfo | `test_ReportBasicInfo_renders_all_fields` | title = "出張費", status = "draft", periodStart = "2026-03-01", periodEnd = "2026-03-31", totalAmount = 50000, submitterName = "テスト太郎", createdAt = "2026-03-01T00:00:00Z" | タイトル、StatusChip（下書き）、対象期間、合計金額（50,000）、作成者名、作成日が描画される |
| RPT-FE-072 | 単体 | ReportBasicInfo | totalAmount: number | 正常系 | RPT-F03 | 55_ui_component/screens/report-detail.md §ReportBasicInfo | `test_ReportBasicInfo_formats_amount` | totalAmount = 1234567 | 金額が「1,234,567」と 3 桁カンマ区切りで表示される |
| RPT-FE-073 | 単体 | ReportWorkflowInfo | status: ReportStatus | 正常系 | RPT-F03 | 55_ui_component/screens/report-detail.md §ReportWorkflowInfo | `test_ReportWorkflowInfo_draft_no_workflow_fields` | status = "draft"、他の全ワークフロー Props は null | ワークフロー関連項目（提出日・承認情報・却下情報・支払情報）が非表示 |
| RPT-FE-074 | 単体 | ReportWorkflowInfo | status: ReportStatus, submittedAt: string \| null | 正常系 | RPT-F03 | 55_ui_component/screens/report-detail.md §ReportWorkflowInfo | `test_ReportWorkflowInfo_submitted_shows_submitted_at` | status = "submitted"、submittedAt = "2026-03-15T10:00:00Z" | 提出日が表示される |
| RPT-FE-075 | 単体 | ReportWorkflowInfo | status: ReportStatus, approverName: string \| null, approvedAt: string \| null, approvalComment: string \| null | 正常系 | RPT-F03 | 55_ui_component/screens/report-detail.md §ReportWorkflowInfo | `test_ReportWorkflowInfo_approved_shows_approval_info` | status = "approved"、approverName = "承認者太郎"、approvedAt = "2026-03-16T10:00:00Z"、approvalComment = "問題ありません" | 承認者名、承認日、承認コメントが表示される |
| RPT-FE-076 | 単体 | ReportWorkflowInfo | status: ReportStatus, rejectorName: string \| null, rejectedAt: string \| null, rejectionReason: string \| null | 正常系 | RPT-F03 | 55_ui_component/screens/report-detail.md §ReportWorkflowInfo | `test_ReportWorkflowInfo_rejected_shows_rejection_info` | status = "rejected"、rejectorName = "承認者次郎"、rejectedAt = "2026-03-16T10:00:00Z"、rejectionReason = "領収書が不足しています" | 却下者名、却下日、却下理由が表示される。却下理由は赤色背景で目立って表示される |
| RPT-FE-077 | 単体 | ReportWorkflowInfo | status: ReportStatus, paidByName: string \| null, paidAt: string \| null | 正常系 | RPT-F03 | 55_ui_component/screens/report-detail.md §ReportWorkflowInfo | `test_ReportWorkflowInfo_paid_shows_payment_info` | status = "paid"、paidByName = "経理花子"、paidAt = "2026-03-20T10:00:00Z" | 支払処理者名と支払完了日が表示される |
| RPT-FE-078 | 単体 | ReportWorkflowInfo | approvalComment: string \| null | 正常系 | RPT-F03 | 55_ui_component/screens/report-detail.md §ReportWorkflowInfo | `test_ReportWorkflowInfo_no_comment_hides_field` | status = "approved"、approvalComment = null | 承認コメント欄が非表示 |
| RPT-FE-079 | 単体 | ReportReferenceLink | referenceReportId: string \| null, referenceReportTitle: string \| null | 正常系 | RPT-F03, RPT-016 | 55_ui_component/screens/report-detail.md §ReportReferenceLink | `test_ReportReferenceLink_renders_link` | referenceReportId = "ref-report-id"、referenceReportTitle = "元レポート" | `/reports/ref-report-id` へのリンクが「元レポート」テキストで描画される |
| RPT-FE-080 | 単体 | ReportReferenceLink | referenceReportId: string \| null | 正常系 | RPT-F03 | 55_ui_component/screens/report-detail.md §ReportReferenceLink | `test_ReportReferenceLink_hidden_when_null` | referenceReportId = null | コンポーネントが描画されない（非表示） |
| RPT-FE-081 | 単体 | ReportActionBar | status: ReportStatus, isOwner: boolean, currentUserRole: string | 正常系 | RPT-F03 | 55_ui_component/screens/report-detail.md §ReportActionBar | `test_ReportActionBar_draft_owner_shows_owner_actions` | status = "draft"、isOwner = true、currentUserRole = "member" | OwnerActions が描画される（編集・提出・削除ボタン）。WorkflowActions は描画されない |
| RPT-FE-082 | 単体 | ReportActionBar | status: ReportStatus, isOwner: boolean, currentUserRole: string | 正常系 | RPT-F03 | 55_ui_component/screens/report-detail.md §ReportActionBar | `test_ReportActionBar_submitted_non_owner_approver` | status = "submitted"、isOwner = false、currentUserRole = "approver" | WorkflowActions が描画される（承認・却下ボタン）。OwnerActions は描画されない |
| RPT-FE-083 | 単体 | ReportActionBar | status: ReportStatus, isOwner: boolean, currentUserRole: string | 認可 | RPT-F03 | 55_ui_component/screens/report-detail.md §ReportActionBar | `test_ReportActionBar_submitted_owner_no_workflow_actions` | status = "submitted"、isOwner = true、currentUserRole = "approver" | WorkflowActions が描画されない（自己承認禁止: RBC-016） |
| RPT-FE-084 | 単体 | ReportActionBar | status: ReportStatus, isOwner: boolean, currentUserRole: string | 正常系 | RPT-F03 | 55_ui_component/screens/report-detail.md §ReportActionBar | `test_ReportActionBar_rejected_owner_shows_resubmit` | status = "rejected"、isOwner = true、currentUserRole = "member" | OwnerActions が描画され再申請ボタンが含まれる |
| RPT-FE-085 | 単体 | ReportActionBar | status: ReportStatus, isOwner: boolean, currentUserRole: string | 正常系 | RPT-F03 | 55_ui_component/screens/report-detail.md §ReportActionBar | `test_ReportActionBar_approved_non_owner_accounting` | status = "approved"、isOwner = false、currentUserRole = "accounting" | WorkflowActions が描画される（支払完了ボタン） |
| RPT-FE-086 | 単体 | ReportActionBar | status: ReportStatus | 正常系 | RPT-F03 | 55_ui_component/screens/report-detail.md §ReportActionBar | `test_ReportActionBar_paid_no_actions` | status = "paid" | OwnerActions も WorkflowActions も描画されない（終端状態） |
| RPT-FE-087 | 単体 | ReportActionBar | pendingAction: string \| null | 正常系 | RPT-F06 | 55_ui_component/screens/report-detail.md §ReportActionBar | `test_ReportActionBar_pending_disables_buttons` | pendingAction = "submit" | 提出ボタンが disabled + スピナー表示。他のボタンも disabled |
| RPT-FE-088 | 単体 | OwnerActions | status: ReportStatus | 正常系 | RPT-F03 | 55_ui_component/screens/report-detail.md §OwnerActions | `test_OwnerActions_draft_shows_edit_submit_delete` | status = "draft"、itemCount = 1 | 編集・提出・削除ボタンが表示される |
| RPT-FE-089 | 単体 | OwnerActions | status: ReportStatus, itemCount: number | 正常系 | RPT-F06, RPT-014 | 55_ui_component/screens/report-detail.md §OwnerActions | `test_OwnerActions_submit_disabled_no_items` | status = "draft"、itemCount = 0 | 提出ボタンが disabled + ツールチップが表示される |
| RPT-FE-090 | 単体 | OwnerActions | status: ReportStatus | 正常系 | RPT-F03 | 55_ui_component/screens/report-detail.md §OwnerActions | `test_OwnerActions_rejected_shows_resubmit` | status = "rejected" | 再申請ボタンが表示される。編集・提出・削除ボタンは非表示 |

**備考**: ReportDetailPage 側の明細行クリックプリフィルテスト（issue-103 対応）は、OwnerActions 側の RPT-FE-090 と ID が重複するため RPT-FE-090-A に振り直した。OwnerActions 側の RPT-FE-090（本行）はそのまま維持する。
| RPT-FE-091 | 単体 | OwnerActions | status: ReportStatus | 正常系 | RPT-F03 | 55_ui_component/screens/report-detail.md §OwnerActions | `test_OwnerActions_submitted_no_buttons` | status = "submitted" | 編集・提出・削除・再申請ボタンが全て非表示 |
| RPT-FE-092 | 単体 | OwnerActions | onEdit: () => void | 正常系 | RPT-F04 | 55_ui_component/screens/report-detail.md §OwnerActions | `test_OwnerActions_edit_calls_callback` | status = "draft"、編集ボタンをクリック | onEdit コールバックが呼び出される |
| RPT-FE-093 | 単体 | OwnerActions | onSubmitReport: () => void | 正常系 | RPT-F06 | 55_ui_component/screens/report-detail.md §OwnerActions | `test_OwnerActions_submit_calls_callback` | status = "draft"、itemCount = 1、提出ボタンをクリック | onSubmitReport コールバックが呼び出される |
| RPT-FE-094 | 単体 | OwnerActions | onDelete: () => void | 正常系 | RPT-F05 | 55_ui_component/screens/report-detail.md §OwnerActions | `test_OwnerActions_delete_calls_callback` | status = "draft"、削除ボタンをクリック | onDelete コールバックが呼び出される |
| RPT-FE-095 | 単体 | OwnerActions | onResubmit: () => void | 正常系 | RPT-F01, RPT-015 | 55_ui_component/screens/report-detail.md §OwnerActions | `test_OwnerActions_resubmit_calls_callback` | status = "rejected"、再申請ボタンをクリック | onResubmit コールバックが呼び出される |
| RPT-FE-096 | 単体 | OwnerActions | pendingAction: 'submit' \| 'delete' \| null | 正常系 | RPT-F06 | 55_ui_component/screens/report-detail.md §OwnerActions | `test_OwnerActions_pending_submit_disables` | pendingAction = "submit" | 提出ボタンが disabled + スピナー。編集・削除ボタンも disabled |
| RPT-FE-097 | 単体 | useSubmitReport | useSubmitReport | 正常系 | RPT-F06 | 55_ui_component/state-management.md §useSubmitReport | `test_useSubmitReport_calls_api` | mutate({ id: "test-id", updated_at: "2026-03-01T00:00:00Z" }) | POST /api/reports/test-id/submit が updated_at 付きで呼び出される |
| RPT-FE-098 | 単体 | useSubmitReport | useSubmitReport | 正常系 | RPT-F06 | 55_ui_component/state-management.md §useSubmitReport | `test_useSubmitReport_invalidates_cache` | ミューテーション成功後 | レポート詳細・一覧のクエリキャッシュが無効化される |
| RPT-FE-099 | 単体 | useSubmitReport | useSubmitReport | 異常系 | RPT-F06 | 55_ui_component/state-management.md §useSubmitReport | `test_useSubmitReport_handles_error` | API が 422 EMPTY_REPORT_SUBMISSION を返す | isError = true、error にエラー情報が格納される |
| RPT-FE-100 | 単体 | useDeleteReport | useDeleteReport | 正常系 | RPT-F05 | 55_ui_component/state-management.md §useDeleteReport | `test_useDeleteReport_calls_api` | mutate("test-id") | DELETE /api/reports/test-id が呼び出される |
| RPT-FE-101 | 単体 | useDeleteReport | useDeleteReport | 正常系 | RPT-F05 | 55_ui_component/state-management.md §useDeleteReport | `test_useDeleteReport_invalidates_cache` | ミューテーション成功後 | レポート一覧のクエリキャッシュが無効化される |
| RPT-FE-102 | 単体 | useDeleteReport | useDeleteReport | 異常系 | RPT-F05 | 55_ui_component/state-management.md §useDeleteReport | `test_useDeleteReport_handles_error` | API が 422 REPORT_NOT_DELETABLE を返す | isError = true、error にエラー情報が格納される |

---

#### FE-5. 追記テスト（明細行クリックプリフィル）

以下のテスト ID は旧 RPT-FE-090（ReportDetailPage 側）の重複解消のために追記する。

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| RPT-FE-090-A | 単体 | ReportDetailPage | useReport | 正常系 | RPT-F03 | 55_ui_component/screens/report-detail.md §ReportDetailPage | `test_ReportDetailPage_item_row_click_prefills_panel` | useReport が draft 状態のレポート（明細2件）を返す。明細行をクリック | ItemSlidePanel が開き、対象明細の値（金額・摘要等）がフォームにプリフィルされる（handleItemClick が formKey をインクリメントして ItemSlidePanel を再マウントし、既存値をプリフィルする） |

**備考**: RPT-FE-090-A は旧 RPT-FE-090（ReportDetailPage 側の明細行クリックプリフィル）を振り直した ID。OwnerActions 側の RPT-FE-090（test_OwnerActions_rejected_shows_resubmit）はそのまま維持する。

---

#### FE テストID一覧（サマリー）

| 範囲 | テストID範囲 | 件数 |
|------|------------|------|
| レポート一覧画面 コンポーネント | RPT-FE-001 -- RPT-FE-020 | 20 |
| レポート一覧画面 Hook（useMyReports） | RPT-FE-021 -- RPT-FE-023 | 3 |
| レポート作成画面 コンポーネント | RPT-FE-024 -- RPT-FE-045 | 22 |
| レポート作成画面 Hook（useCreateReport） | RPT-FE-046 -- RPT-FE-049 | 4 |
| レポート編集画面 コンポーネント | RPT-FE-050 -- RPT-FE-058 | 9 |
| レポート編集画面 Hook（useReport, useUpdateReport） | RPT-FE-059 -- RPT-FE-063 | 5 |
| レポート詳細画面 レポート閲覧・操作コンポーネント | RPT-FE-064 -- RPT-FE-096 | 33 |
| レポート詳細画面 Hook（useSubmitReport, useDeleteReport） | RPT-FE-097 -- RPT-FE-102 | 6 |
| 追記テスト（明細行クリックプリフィル） | RPT-FE-090-A | 1 |
| **FE 合計** | **RPT-FE-001 -- RPT-FE-102, RPT-FE-090-A** | **103** |

---

#### FE テストファイル構成

```
expense-saas/frontend/src/
  pages/reports/
    __tests__/
      ReportListPage.test.tsx        # RPT-FE-001〜RPT-FE-007
      ReportListHeader.test.tsx      # RPT-FE-008〜RPT-FE-009
      CreateReportButton.test.tsx    # RPT-FE-010
      ReportListFilter.test.tsx      # RPT-FE-011〜RPT-FE-014
      ReportListTable.test.tsx       # RPT-FE-015〜RPT-FE-020
      ReportCreatePage.test.tsx      # RPT-FE-024〜RPT-FE-027
      ReportEditPage.test.tsx        # RPT-FE-050〜RPT-FE-058
      ReportDetailPage.test.tsx      # RPT-FE-064〜RPT-FE-069
      ReportInfoCard.test.tsx        # RPT-FE-070
      ReportBasicInfo.test.tsx       # RPT-FE-071〜RPT-FE-072
      ReportWorkflowInfo.test.tsx    # RPT-FE-073〜RPT-FE-078
      ReportReferenceLink.test.tsx   # RPT-FE-079〜RPT-FE-080
      ReportActionBar.test.tsx       # RPT-FE-081〜RPT-FE-087
      OwnerActions.test.tsx          # RPT-FE-088〜RPT-FE-096
  components/report/
    __tests__/
      ReportForm.test.tsx            # RPT-FE-028〜RPT-FE-036, RPT-FE-058
      ReportPeriodField.test.tsx     # RPT-FE-039〜RPT-FE-042
      ReportFormActions.test.tsx     # RPT-FE-043〜RPT-FE-045
  components/ui/
    __tests__/
      FormAlert.test.tsx             # RPT-FE-037〜RPT-FE-038
  hooks/
    __tests__/
      useMyReports.test.ts           # RPT-FE-021〜RPT-FE-023
      useCreateReport.test.ts        # RPT-FE-046〜RPT-FE-049
      useReport.test.ts              # RPT-FE-059〜RPT-FE-060
      useUpdateReport.test.ts        # RPT-FE-061〜RPT-FE-063
      useSubmitReport.test.ts        # RPT-FE-097〜RPT-FE-099
      useDeleteReport.test.ts        # RPT-FE-100〜RPT-FE-102
```

#### FE テスト実行コマンド

```bash
# レポート一覧画面のコンポーネントテスト
npx vitest run src/pages/reports/__tests__/ReportList*.test.tsx src/pages/reports/__tests__/CreateReportButton.test.tsx

# レポート作成画面のコンポーネントテスト
npx vitest run src/pages/reports/__tests__/ReportCreatePage.test.tsx src/components/report/__tests__/ReportForm.test.tsx

# レポート編集画面のコンポーネントテスト
npx vitest run src/pages/reports/__tests__/ReportEditPage.test.tsx

# レポート詳細画面のコンポーネントテスト（レポート閲覧・操作のみ）
npx vitest run src/pages/reports/__tests__/ReportDetailPage.test.tsx src/pages/reports/__tests__/ReportInfoCard.test.tsx src/pages/reports/__tests__/ReportBasicInfo.test.tsx src/pages/reports/__tests__/ReportWorkflowInfo.test.tsx src/pages/reports/__tests__/ReportReferenceLink.test.tsx src/pages/reports/__tests__/ReportActionBar.test.tsx src/pages/reports/__tests__/OwnerActions.test.tsx

# Hook テスト
npx vitest run src/hooks/__tests__/useMyReports.test.ts src/hooks/__tests__/useCreateReport.test.ts src/hooks/__tests__/useReport.test.ts src/hooks/__tests__/useUpdateReport.test.ts src/hooks/__tests__/useSubmitReport.test.ts src/hooks/__tests__/useDeleteReport.test.ts

# レポート FE 全テスト一括実行
npx vitest run --reporter=verbose src/pages/reports/__tests__/ src/components/report/__tests__/ src/components/ui/__tests__/FormAlert.test.tsx src/hooks/__tests__/use*Report*.test.ts
```

#### FE テスト実装方針

コンポーネントテストは Vitest + React Testing Library で実装する。API 呼び出しは MSW（Mock Service Worker）でモックする。

```typescript
// 例: ReportListPage のテスト（RPT-FE-001）
import { render, screen } from '@testing-library/react';
import { describe, it, expect, vi } from 'vitest';
import { ReportListPage } from '../ReportListPage';

// useMyReports のモック
vi.mock('@/hooks/useMyReports', () => ({
  useMyReports: () => ({
    data: {
      data: [
        { id: '1', title: 'レポート1', status: 'draft', totalAmount: 1000, periodStart: '2026-03-01', periodEnd: '2026-03-31', createdAt: '2026-03-01' },
        { id: '2', title: 'レポート2', status: 'submitted', totalAmount: 2000, periodStart: '2026-03-01', periodEnd: '2026-03-31', createdAt: '2026-03-02' },
        { id: '3', title: 'レポート3', status: 'approved', totalAmount: 3000, periodStart: '2026-03-01', periodEnd: '2026-03-31', createdAt: '2026-03-03' },
      ],
      pagination: { current_page: 1, total_pages: 1, total_count: 3 },
    },
    isLoading: false,
    isError: false,
  }),
}));

describe('ReportListPage', () => {
  it('renders report list with header, filter, table, and pagination', () => {
    render(<ReportListPage />, { wrapper: TestProviders });
    expect(screen.getByText('マイレポート')).toBeInTheDocument();
    expect(screen.getByText('レポート1')).toBeInTheDocument();
    expect(screen.getByText('レポート2')).toBeInTheDocument();
    expect(screen.getByText('レポート3')).toBeInTheDocument();
  });
});
```

```typescript
// 例: useMyReports の Hook テスト（RPT-FE-021）
import { renderHook, waitFor } from '@testing-library/react';
import { describe, it, expect } from 'vitest';
import { useMyReports } from '../useMyReports';
import { server } from '@/mocks/server';
import { http, HttpResponse } from 'msw';

describe('useMyReports', () => {
  it('fetches report list data', async () => {
    server.use(
      http.get('/api/reports', () =>
        HttpResponse.json({
          data: [{ id: '1', title: 'テスト', status: 'draft' }],
          pagination: { current_page: 1, total_pages: 1, total_count: 1 },
        }),
      ),
    );

    const { result } = renderHook(
      () => useMyReports({ page: 1, per_page: 20 }),
      { wrapper: TestProviders },
    );

    await waitFor(() => expect(result.current.isSuccess).toBe(true));
    expect(result.current.data?.data).toHaveLength(1);
  });
});
```
