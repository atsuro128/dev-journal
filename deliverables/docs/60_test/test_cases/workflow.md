# ワークフローテストケース一覧

## この文書の役割

| 項目 | 内容 |
|------|------|
| 目的 | 承認・却下・支払完了エンドポイント（/api/workflow/*）のテストケースを定義する |
| 正本情報 | WFL-F01〜F05 に対応するテストケース一覧 |
| 扱わない内容 | テナント横断テスト（→ cross-cutting.md）、レポート CRUD テスト（→ reports.md） |
| 主な参照元 | `50_detail_design/openapi.yaml#/api/workflow/*`, `20_domain/state_machine.md`, `10_requirements/requirements.md#WFL-F*` |
| 主な参照先 | `60_test/traceability.md`, テスト実装（Step 9） |

---

## 概要

本ファイルは `/api/workflow/*` エンドポイント群のテストケースを定義する。
対応ハンドラ: `workflow_handler_test.go`

### 対象エンドポイント（5 operationId）

| operationId | メソッド | パス | 許可ロール |
|-------------|---------|------|-----------|
| listPendingReports | GET | `/api/workflow/pending` | Approver |
| approveReport | POST | `/api/workflow/{id}/approve` | Approver |
| rejectReport | POST | `/api/workflow/{id}/reject` | Approver |
| listPayableReports | GET | `/api/workflow/payable` | Accounting |
| markReportAsPaid | POST | `/api/workflow/{id}/pay` | Accounting |

### 責務境界

- テナント分離テスト（クロステナントアクセスが 404 になること）は `cross-cutting.md` に集約する（このファイルには記載しない）
- RBACテスト（各エンドポイントのロール別許可・拒否）はこのファイルに記載する
- 状態遷移テスト（ハンドラ層統合テスト）: T2（approve）、T3（reject）、T4（pay）、X5〜X10 はこのファイルに記載する
- T1（submit）、T5（delete）、X1〜X4 は `reports.md` の担当（このファイルには記載しない）
- ドメイン層単体テストの全遷移（T1〜T5、X1〜X10）は `reports.md` に記載済み（このファイルには記載しない）

### 参照フィクスチャ（`test_strategy.md` §4）

| フィクスチャ名 | 状態 | レポートID | 作成者 |
|-------------|------|----------|--------|
| `report_submitted` | `submitted` | `cccccccc-0002-0002-0002-000000000002` | Test Member（`aaaaaaaa-3333-3333-3333-000000000003`） |
| `report_approved` | `approved` | `cccccccc-0003-0003-0003-000000000003` | Test Member（`aaaaaaaa-3333-3333-3333-000000000003`） |

| ロール | ユーザーID |
|--------|-----------|
| Test Approver | `aaaaaaaa-2222-2222-2222-000000000002` |
| Test Member | `aaaaaaaa-3333-3333-3333-000000000003` |
| Test Accounting | `aaaaaaaa-4444-4444-4444-000000000004` |
| Test Admin | `aaaaaaaa-1111-1111-1111-000000000001` |

---

## テストケース一覧

### listPendingReports — GET /api/workflow/pending

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|-------------------|---------|
| WFL-001 | 統合 | handler | 正常系 | WFL-F04, RBC-015 | openapi.yaml#listPendingReports | `TestListPendingReports_Success` | 認証済み Approver でリクエスト。`report_submitted` がDB上に存在する | 200 OK。`data` にテナントA の submitted レポートが含まれる。`pagination` が返る |
| WFL-002 | 統合 | handler | 正常系 | WFL-F04 | openapi.yaml#listPendingReports | `TestListPendingReports_IncludesOwnReport` | Test Approver 本人が作成した submitted レポートがDB上に存在する状態でリクエスト | 200 OK。自分のレポートも一覧に含まれ、`is_own_report: true` が設定されている |
| WFL-003 | 統合 | handler | 正常系 | WFL-F04 | openapi.yaml#listPendingReports | `TestListPendingReports_ExcludesNonSubmitted` | テナントA に draft / approved / rejected / paid 状態のレポートのみ存在する | 200 OK。`data` が空配列 `[]` |
| WFL-004 | 統合 | handler | 正常系 | WFL-F04 | openapi.yaml#listPendingReports | `TestListPendingReports_FilterByApplicantName` | クエリパラメータ `applicant_name=Test` を指定してリクエスト | 200 OK。申請者名が部分一致するレポートのみ返る |
| WFL-005 | 統合 | handler | 正常系 | WFL-F04, NFR-PERF-004 | openapi.yaml#listPendingReports | `TestListPendingReports_Pagination` | `per_page=1` かつ submitted レポートが2件以上存在する状態でリクエスト | 200 OK。`data` の件数が1件。`pagination.total_pages >= 2` |
| WFL-006 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#listPendingReports, authz.md#3 | `TestListPendingReports_Forbidden_Member` | Member ロールのユーザーで認証してリクエスト | 403 FORBIDDEN |
| WFL-007 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#listPendingReports, authz.md#3 | `TestListPendingReports_Forbidden_Admin` | Admin ロールのユーザーで認証してリクエスト | 403 FORBIDDEN |
| WFL-008 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#listPendingReports, authz.md#3 | `TestListPendingReports_Forbidden_Accounting` | Accounting ロールのユーザーで認証してリクエスト | 403 FORBIDDEN |
| WFL-009 | 統合 | handler | 異常系 | WFL-F04 | openapi.yaml#listPendingReports | `TestListPendingReports_Unauthorized` | 認証ヘッダーなしでリクエスト | 401 UNAUTHORIZED |

---

### approveReport — POST /api/workflow/{id}/approve

#### 正常系

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|-------------------|---------|
| WFL-010 | 統合 | handler | 正常系 | WFL-F01, WFL-011 | openapi.yaml#approveReport, state_machine.md#T2 | `TestApproveReport_Success` | Approver で認証。対象: `report_submitted`（作成者: Test Member）。リクエストボディ: `{"updated_at": "<report_submitted の updated_at>"}` | 200 OK。レスポンスの `data.status` が `"approved"` になっている。`data.approved_by` が Approver のユーザーID。DBのレコードも `approved` に更新されている |
| WFL-011 | 統合 | handler | 正常系 | WFL-F01 | openapi.yaml#approveReport | `TestApproveReport_SuccessWithComment` | Approver で認証。リクエストボディ: `{"comment": "問題ありません", "updated_at": "<updated_at>"}` | 200 OK。レスポンスの `data.approval_comment` が `"問題ありません"` |
| WFL-012 | 統合 | handler | 正常系 | WFL-F01 | openapi.yaml#approveReport | `TestApproveReport_SuccessWithoutComment` | Approver で認証。リクエストボディに `comment` フィールドを含めない | 200 OK。承認コメントは空または null |

#### 状態遷移の禁止パス（ハンドラ層統合テスト: X5〜X10）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|-------------------|---------|
| WFL-013 | 統合 | handler | 状態遷移 | WFL-002 | openapi.yaml#approveReport, state_machine.md#T2 | `TestApproveReport_AlreadyApproved` | Approver で認証。対象: `report_approved`（approved 状態）（state_machine.md §T2 事前条件 No.1: status == submitted の違反）。リクエストボディ: `{"updated_at": "<updated_at>"}` | 422 INVALID_STATE_TRANSITION |
| WFL-014 | 統合 | handler | 状態遷移 | WFL-002 | openapi.yaml#approveReport | `TestApproveReport_DraftState` | Approver で認証。対象: draft 状態のレポート | 422 INVALID_STATE_TRANSITION |

**備考**: report_handler_test.go の RPT-057（TestSubmitReport_NoApprover）は対応要件 ID として WFL-014 を参照しているが、テスト ID としては RPT-057 のまま維持する。テスト ID の重複は発生しない。本行の WFL-014（TestApproveReport_DraftState）もそのまま維持する。
| WFL-015 | 統合 | handler | 状態遷移 | WFL-004 | openapi.yaml#approveReport, state_machine.md#X9 | `TestApproveReport_RejectedState` | Approver で認証。対象: rejected 状態のレポート（X9: rejected→approved は禁止） | 422 INVALID_STATE_TRANSITION |
| WFL-016 | 統合 | handler | 状態遷移 | WFL-004 | openapi.yaml#approveReport, state_machine.md#X10 | `TestApproveReport_PaidState` | Approver で認証。対象: paid 状態のレポート（X10: paid→approved は禁止） | 422 INVALID_STATE_TRANSITION |

#### 自己承認禁止（RBC-016）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|-------------------|---------|
| WFL-018 | 統合 | handler | 認可 | RBC-016 | openapi.yaml#approveReport, authz.md#4.2 | `TestApproveReport_SelfApproval` | Test Approver 本人が作成した submitted レポートを、同じ Test Approver で承認しようとする | 403 SELF_APPROVAL_NOT_ALLOWED |

#### RBACテスト（ロール別拒否）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|-------------------|---------|
| WFL-019 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#approveReport, authz.md#3 | `TestApproveReport_Forbidden_Member` | Member で認証。対象: `report_submitted` | 403 FORBIDDEN |
| WFL-020 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#approveReport, authz.md#3 | `TestApproveReport_Forbidden_Admin` | Admin で認証。対象: `report_submitted` | 403 FORBIDDEN |
| WFL-021 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#approveReport, authz.md#3 | `TestApproveReport_Forbidden_Accounting` | Accounting で認証。対象: `report_submitted` | 403 FORBIDDEN |
| WFL-022 | 統合 | handler | 異常系 | WFL-F01 | openapi.yaml#approveReport | `TestApproveReport_Unauthorized` | 認証ヘッダーなし | 401 UNAUTHORIZED |

#### その他の異常系

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|-------------------|---------|
| WFL-023 | 統合 | handler | 異常系 | WFL-F01 | openapi.yaml#approveReport | `TestApproveReport_NotFound` | Approver で認証。存在しないレポートID（`00000000-0000-0000-0000-000000000099`）を指定 | 404 RESOURCE_NOT_FOUND |
| WFL-024 | 統合 | handler | 異常系 | WFL-F01 | openapi.yaml#approveReport | `TestApproveReport_Conflict_OptimisticLock` | Approver で認証。対象: `report_submitted`。リクエストボディの `updated_at` に古い（不一致の）タイムスタンプを指定 | 409 CONFLICT |
| WFL-025 | 統合 | handler | 異常系 | WFL-F01 | openapi.yaml#approveReport | `TestApproveReport_MissingUpdatedAt` | Approver で認証。リクエストボディに `updated_at` フィールドなし | 422 VALIDATION_ERROR。`details[].field = "updated_at"` を含む |

---

### rejectReport — POST /api/workflow/{id}/reject

#### 正常系

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|-------------------|---------|
| WFL-026 | 統合 | handler | 正常系 | WFL-F02, WFL-012 | openapi.yaml#rejectReport, state_machine.md#T3 | `TestRejectReport_Success` | Approver で認証。対象: `report_submitted`。リクエストボディ: `{"reason": "領収書が不明瞭です", "updated_at": "<updated_at>"}` | 200 OK。レスポンスの `data.status` が `"rejected"`。`data.rejected_by` が Approver のユーザーID。`data.rejection_reason` が `"領収書が不明瞭です"` |

#### 却下理由バリデーション（state_machine.md §T3 事前条件 No.3: rejection_reason 非空）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|-------------------|---------|
| WFL-027 | 統合 | handler | 異常系 | WFL-012 | openapi.yaml#rejectReport, state_machine.md#T3 | `TestRejectReport_MissingRejectionReason` | Approver で認証。リクエストボディに `reason` フィールドなし（`{"updated_at": "<updated_at>"}` のみ） | 422 MISSING_REJECTION_REASON |
| WFL-028 | 統合 | handler | 異常系 | WFL-012 | openapi.yaml#rejectReport, state_machine.md#T3 | `TestRejectReport_EmptyRejectionReason` | Approver で認証。リクエストボディ: `{"reason": "", "updated_at": "<updated_at>"}` | 422 MISSING_REJECTION_REASON |
| WFL-029 | 統合 | handler | 境界値 | WFL-012 | openapi.yaml#rejectReport | `TestRejectReport_RejectionReasonMaxLength` | Approver で認証。`reason` が1000文字の文字列 | 200 OK（境界値: 上限ちょうど） |
| WFL-030 | 統合 | handler | 境界値 | WFL-012 | openapi.yaml#rejectReport | `TestRejectReport_RejectionReasonTooLong` | Approver で認証。`reason` が1001文字の文字列 | 422 VALIDATION_ERROR。`details[].field = "reason"` を含む |

#### 状態遷移の禁止パス（ハンドラ層統合テスト: X8 相当）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|-------------------|---------|
| WFL-031 | 統合 | handler | 状態遷移 | WFL-002 | openapi.yaml#rejectReport, state_machine.md#X8 | `TestRejectReport_AlreadyApproved` | Approver で認証。対象: `report_approved`（X8: approved→rejected は禁止）。リクエストボディ: `{"reason": "理由", "updated_at": "<updated_at>"}` | 422 INVALID_STATE_TRANSITION |
| WFL-032 | 統合 | handler | 状態遷移 | WFL-002 | openapi.yaml#rejectReport | `TestRejectReport_DraftState` | Approver で認証。対象: draft 状態のレポート | 422 INVALID_STATE_TRANSITION |
| WFL-033 | 統合 | handler | 状態遷移 | WFL-004 | openapi.yaml#rejectReport, state_machine.md#X9 | `TestRejectReport_RejectedState` | Approver で認証。対象: rejected 状態のレポート（X9: 終端状態からの遷移不可） | 422 INVALID_STATE_TRANSITION |
| WFL-034 | 統合 | handler | 状態遷移 | WFL-004 | openapi.yaml#rejectReport, state_machine.md#X10 | `TestRejectReport_PaidState` | Approver で認証。対象: paid 状態のレポート（X10: 終端状態からの遷移不可） | 422 INVALID_STATE_TRANSITION |

#### 自己却下禁止（RBC-016）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|-------------------|---------|
| WFL-035 | 統合 | handler | 認可 | RBC-016 | openapi.yaml#rejectReport, authz.md#4.2 | `TestRejectReport_SelfRejection` | Test Approver 本人が作成した submitted レポートを、同じ Test Approver で却下しようとする。リクエストボディ: `{"reason": "理由", "updated_at": "<updated_at>"}` | 403 SELF_APPROVAL_NOT_ALLOWED |

#### RBACテスト（ロール別拒否）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|-------------------|---------|
| WFL-036 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#rejectReport, authz.md#3 | `TestRejectReport_Forbidden_Member` | Member で認証。対象: `report_submitted` | 403 FORBIDDEN |
| WFL-037 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#rejectReport, authz.md#3 | `TestRejectReport_Forbidden_Admin` | Admin で認証。対象: `report_submitted` | 403 FORBIDDEN |
| WFL-038 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#rejectReport, authz.md#3 | `TestRejectReport_Forbidden_Accounting` | Accounting で認証。対象: `report_submitted` | 403 FORBIDDEN |
| WFL-039 | 統合 | handler | 異常系 | WFL-F02 | openapi.yaml#rejectReport | `TestRejectReport_Unauthorized` | 認証ヘッダーなし | 401 UNAUTHORIZED |

#### その他の異常系

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|-------------------|---------|
| WFL-040 | 統合 | handler | 異常系 | WFL-F02 | openapi.yaml#rejectReport | `TestRejectReport_NotFound` | Approver で認証。存在しないレポートID を指定 | 404 RESOURCE_NOT_FOUND |
| WFL-041 | 統合 | handler | 異常系 | WFL-F02 | openapi.yaml#rejectReport | `TestRejectReport_Conflict_OptimisticLock` | Approver で認証。リクエストボディの `updated_at` に古いタイムスタンプを指定 | 409 CONFLICT |

---

### listPayableReports — GET /api/workflow/payable

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|-------------------|---------|
| WFL-042 | 統合 | handler | 正常系 | WFL-F05 | openapi.yaml#listPayableReports | `TestListPayableReports_Success` | Accounting で認証。`report_approved` がDB上に存在する | 200 OK。`data` にテナントA の approved レポートが含まれる。`pagination` が返る |
| WFL-043 | 統合 | handler | 正常系 | WFL-F05 | openapi.yaml#listPayableReports | `TestListPayableReports_IncludesOwnReport` | Test Accounting 本人が作成した approved レポートがDB上に存在する状態でリクエスト | 200 OK。自分のレポートも一覧に含まれ、`is_own_report: true` が設定されている |
| WFL-044 | 統合 | handler | 正常系 | WFL-F05 | openapi.yaml#listPayableReports | `TestListPayableReports_ExcludesNonApproved` | テナントA に draft / submitted / rejected / paid 状態のレポートのみ存在する | 200 OK。`data` が空配列 `[]` |
| WFL-045 | 統合 | handler | 正常系 | WFL-F05 | openapi.yaml#listPayableReports | `TestListPayableReports_FilterByApplicantName` | クエリパラメータ `applicant_name=Test` を指定してリクエスト | 200 OK。申請者名が部分一致するレポートのみ返る |
| WFL-046 | 統合 | handler | 正常系 | WFL-F05, NFR-PERF-004 | openapi.yaml#listPayableReports | `TestListPayableReports_Pagination` | `per_page=1` かつ approved レポートが2件以上存在する状態でリクエスト | 200 OK。`data` の件数が1件。`pagination.total_pages >= 2` |
| WFL-047 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#listPayableReports, authz.md#3 | `TestListPayableReports_Forbidden_Member` | Member で認証 | 403 FORBIDDEN |
| WFL-048 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#listPayableReports, authz.md#3 | `TestListPayableReports_Forbidden_Approver` | Approver で認証 | 403 FORBIDDEN |
| WFL-049 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#listPayableReports, authz.md#3 | `TestListPayableReports_Forbidden_Admin` | Admin で認証 | 403 FORBIDDEN |
| WFL-050 | 統合 | handler | 異常系 | WFL-F05 | openapi.yaml#listPayableReports | `TestListPayableReports_Unauthorized` | 認証ヘッダーなし | 401 UNAUTHORIZED |

---

### markReportAsPaid — POST /api/workflow/{id}/pay

#### 正常系

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|-------------------|---------|
| WFL-051 | 統合 | handler | 正常系 | WFL-F03, WFL-013 | openapi.yaml#markReportAsPaid, state_machine.md#T4 | `TestMarkReportAsPaid_Success` | Accounting で認証。対象: `report_approved`（作成者: Test Member）。リクエストボディ: `{"updated_at": "<report_approved の updated_at>"}` | 200 OK。レスポンスの `data.status` が `"paid"`。`data.paid_by` が Accounting のユーザーID。DBのレコードも `paid` に更新されている |

#### 状態遷移の禁止パス（ハンドラ層統合テスト: X5〜X7）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|-------------------|---------|
| WFL-052 | 統合 | handler | 状態遷移 | WFL-002 | openapi.yaml#markReportAsPaid, state_machine.md#X5 | `TestMarkReportAsPaid_X5_SubmittedToPaid` | Accounting で認証。対象: `report_submitted`（submitted 状態）（X5: submitted→paid は禁止）。リクエストボディ: `{"updated_at": "<updated_at>"}` | 422 INVALID_STATE_TRANSITION |
| WFL-053 | 統合 | handler | 状態遷移 | WFL-002 | openapi.yaml#markReportAsPaid | `TestMarkReportAsPaid_X6_DraftToPaid` | Accounting で認証。対象: draft 状態のレポート（X3 相当: draft→paid は禁止） | 422 INVALID_STATE_TRANSITION |
| WFL-054 | 統合 | handler | 状態遷移 | WFL-004 | openapi.yaml#markReportAsPaid, state_machine.md#X10 | `TestMarkReportAsPaid_AlreadyPaid` | Accounting で認証。対象: paid 状態のレポート（X10: paid→paid は禁止） | 422 INVALID_STATE_TRANSITION |
| WFL-055 | 統合 | handler | 状態遷移 | WFL-004 | openapi.yaml#markReportAsPaid, state_machine.md#X9 | `TestMarkReportAsPaid_RejectedState` | Accounting で認証。対象: rejected 状態のレポート（X9: 終端状態からの遷移不可） | 422 INVALID_STATE_TRANSITION |

#### 自己支払処理禁止（RBC-012）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|-------------------|---------|
| WFL-056 | 統合 | handler | 認可 | RBC-012 | openapi.yaml#markReportAsPaid, authz.md#4.3 | `TestMarkReportAsPaid_SelfPayment` | Test Accounting 本人が作成した approved レポートを、同じ Test Accounting で支払完了しようとする。リクエストボディ: `{"updated_at": "<updated_at>"}` | 403 SELF_PAYMENT_NOT_ALLOWED |

#### RBACテスト（ロール別拒否）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|-------------------|---------|
| WFL-057 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#markReportAsPaid, authz.md#3 | `TestMarkReportAsPaid_Forbidden_Member` | Member で認証。対象: `report_approved` | 403 FORBIDDEN |
| WFL-058 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#markReportAsPaid, authz.md#3 | `TestMarkReportAsPaid_Forbidden_Approver` | Approver で認証。対象: `report_approved` | 403 FORBIDDEN |
| WFL-059 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#markReportAsPaid, authz.md#3 | `TestMarkReportAsPaid_Forbidden_Admin` | Admin で認証。対象: `report_approved` | 403 FORBIDDEN |
| WFL-060 | 統合 | handler | 異常系 | WFL-F03 | openapi.yaml#markReportAsPaid | `TestMarkReportAsPaid_Unauthorized` | 認証ヘッダーなし | 401 UNAUTHORIZED |

#### その他の異常系

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|-------------------|---------|
| WFL-061 | 統合 | handler | 異常系 | WFL-F03 | openapi.yaml#markReportAsPaid | `TestMarkReportAsPaid_NotFound` | Accounting で認証。存在しないレポートID を指定 | 404 RESOURCE_NOT_FOUND |
| WFL-062 | 統合 | handler | 異常系 | WFL-F03 | openapi.yaml#markReportAsPaid | `TestMarkReportAsPaid_Conflict_OptimisticLock` | Accounting で認証。リクエストボディの `updated_at` に古いタイムスタンプを指定 | 409 CONFLICT |
| WFL-063 | 統合 | handler | 異常系 | WFL-F03 | openapi.yaml#markReportAsPaid | `TestMarkReportAsPaid_MissingUpdatedAt` | Accounting で認証。リクエストボディに `updated_at` フィールドなし | 422 VALIDATION_ERROR。`details[].field = "updated_at"` を含む |

---

## テストID サマリー

| 範囲 | テストID | 件数 |
|------|---------|------|
| listPendingReports | WFL-001〜WFL-009 | 9 件 |
| approveReport（正常系） | WFL-010〜WFL-012 | 3 件 |
| approveReport（禁止状態遷移） | WFL-013〜WFL-016 | 4 件 |
| approveReport（自己承認禁止） | WFL-018 | 1 件 |
| approveReport（RBAC） | WFL-019〜WFL-022 | 4 件 |
| approveReport（異常系） | WFL-023〜WFL-025 | 3 件 |
| rejectReport（正常系） | WFL-026 | 1 件 |
| rejectReport（却下理由バリデーション） | WFL-027〜WFL-030 | 4 件 |
| rejectReport（禁止状態遷移） | WFL-031〜WFL-034 | 4 件 |
| rejectReport（自己却下禁止） | WFL-035 | 1 件 |
| rejectReport（RBAC） | WFL-036〜WFL-039 | 4 件 |
| rejectReport（異常系） | WFL-040〜WFL-041 | 2 件 |
| listPayableReports | WFL-042〜WFL-050 | 9 件 |
| markReportAsPaid（正常系） | WFL-051 | 1 件 |
| markReportAsPaid（禁止状態遷移） | WFL-052〜WFL-055 | 4 件 |
| markReportAsPaid（自己支払処理禁止） | WFL-056 | 1 件 |
| markReportAsPaid（RBAC） | WFL-057〜WFL-060 | 4 件 |
| markReportAsPaid（異常系） | WFL-061〜WFL-063 | 3 件 |
| **合計** | WFL-001〜WFL-063（WFL-017 欠番） | **62 件** |

---

## 実装ガイド

### テストファイル配置

```
expense-saas/
  internal/
    handler/
      workflow_handler_test.go   ← このファイルに統合テストを実装する
```

### テスト関数の基本構造

```go
// 統合テストのビルドタグ（テスト用DBが必要なため）
//go:build integration

package handler_test

import (
    "net/http"
    "net/http/httptest"
    "testing"
    // ...
)

func TestApproveReport_Success(t *testing.T) {
    // フィクスチャのセットアップ（TestMain で TRUNCATE + 再投入済みを前提）
    // Approver の JWT を発行
    // POST /api/workflow/{report_submitted_id}/approve を実行
    // レスポンスの status コード、body を検証
    // DB のレコードも検証（status = "approved", approved_by = approver_id）
}
```

### 自己承認・自己支払処理テストのデータ準備

WFL-018（自己承認禁止）と WFL-056（自己支払処理禁止）は、テスト実行時にその場でレポートを作成・提出・承認するか、専用フィクスチャをテストケース内でセットアップする。標準フィクスチャ `report_submitted` の作成者は Test Member であるため、Test Approver には自己承認禁止は適用されない。自己承認テスト用には以下のデータを用意する。

- Test Approver（`aaaaaaaa-2222-2222-2222-000000000002`）が作成した submitted レポート（テストケース内で動的作成を推奨）
- Test Accounting（`aaaaaaaa-4444-4444-4444-000000000004`）が作成した approved レポート（同上）

### クリーンアップ

`test_strategy.md` §9.3 の方針に従い、統合テストは **テスト前 TRUNCATE + フィクスチャ再投入** を基本戦略とする。テストケース内で動的に作成するデータはトランザクションロールバックで対応してもよい。

### 楽観的ロックテスト（WFL-024, WFL-041, WFL-062）

楽観的ロックの競合テストは以下の手順で実装する。

1. フィクスチャからレポートの `updated_at` を取得する
2. 意図的に異なるタイムスタンプ（例: `"2000-01-01T00:00:00Z"`）をリクエストボディに設定する
3. 409 CONFLICT が返ることを確認する

---

## FE テストケース

FE テストケースは `55_ui_component/screens/*.md` のコンポーネント単位・Props 単位で導出する。

### 対象画面

| 画面 ID | 画面名 | コンポーネント設計書 |
|---------|--------|-------------------|
| SCR-WFL-001 | 承認待ち一覧 | `55_ui_component/screens/workflow-pending.md` |
| SCR-WFL-002 | 支払待ち一覧 | `55_ui_component/screens/workflow-payable.md` |
| SCR-RPT-004 | レポート詳細（WorkflowActions のみ） | `55_ui_component/screens/report-detail.md` |

### 参照設計書

- `55_ui_component/screens/workflow-pending.md`
- `55_ui_component/screens/workflow-payable.md`
- `55_ui_component/screens/report-detail.md`（WorkflowActions コンポーネントのみ）
- `55_ui_component/common-components.md`（SelfLabel, FilterResetButton）
- `55_ui_component/state-management.md`（usePendingReports, usePayableReports, useApproveReport, useRejectReport, useMarkAsPaid）

---

### SCR-WFL-001 承認待ち一覧

#### PendingApprovalsPage

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| WFL-FE-001 | 単体 | PendingApprovalsPage | usePendingReports | 正常系 | WFL-F04 | `55_ui_component/screens/workflow-pending.md §PendingApprovalsPage` | `renders_pending_approvals_page_with_data` | usePendingReports が 3 件のレポートデータを返す | PendingApprovalsContent にレポート一覧が表示される。件数表示「3 件の承認待ちレポート」が表示される |
| WFL-FE-002 | 単体 | PendingApprovalsPage | usePendingReports | 正常系 | WFL-F04 | `55_ui_component/screens/workflow-pending.md §PendingApprovalsPage` | `manages_filter_state` | 申請者名フィルタに「田中」と入力後 300ms 待機 | usePendingReports が `{ applicant_name: "田中", page: 1 }` で呼び出される |
| WFL-FE-003 | 単体 | PendingApprovalsPage | usePendingReports | 正常系 | WFL-F04 | `55_ui_component/screens/workflow-pending.md §PendingApprovalsPage` | `manages_pagination_state` | ページネーションで 2 ページ目をクリック | usePendingReports が `{ page: 2 }` で呼び出される |
| WFL-FE-004 | 単体 | PendingApprovalsPage | usePendingReports | 正常系 | WFL-F04 | `55_ui_component/screens/workflow-pending.md §PendingApprovalsPage` | `resets_page_on_filter_change` | 2 ページ目を表示中にフィルタを変更 | ページが 1 にリセットされ、usePendingReports が `{ page: 1 }` で呼び出される |
| WFL-FE-005 | 単体 | PendingApprovalsPage | usePendingReports | 認可 | RBAC-F01, RBC-001 | `55_ui_component/screens/workflow-pending.md §PendingApprovalsPage` | `redirects_non_approver_on_403` | usePendingReports が 403 エラーを返す（Member ロールでのアクセスを想定） | ダッシュボード（SCR-DASH-001）にリダイレクトされる |

#### PendingApprovalsContent

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| WFL-FE-006 | 単体 | PendingApprovalsContent | isLoading: boolean | 正常系 | WFL-F04 | `55_ui_component/screens/workflow-pending.md §PendingApprovalsContent` | `shows_skeleton_when_loading` | `isLoading: true` | PageSkeleton（variant: "table"）が表示される。テーブルは非表示 |
| WFL-FE-007 | 単体 | PendingApprovalsContent | reports: PendingReport[] | 正常系 | WFL-F04 | `55_ui_component/screens/workflow-pending.md §PendingApprovalsContent` | `shows_empty_state_no_filter` | `reports: []`, `isLoading: false`, `filters: {}` | EmptyState に「承認待ちのレポートはありません。」が表示される |
| WFL-FE-008 | 単体 | PendingApprovalsContent | reports: PendingReport[] | 正常系 | WFL-F04 | `55_ui_component/screens/workflow-pending.md §PendingApprovalsContent` | `shows_empty_state_with_filter` | `reports: []`, `isLoading: false`, `filters: { applicant_name: "存在しない名前" }` | 「条件に一致するレポートはありません。」が表示される。フィルタリセットボタンが表示される |
| WFL-FE-009 | 単体 | PendingApprovalsContent | reports: PendingReport[] | 正常系 | WFL-F04 | `55_ui_component/screens/workflow-pending.md §PendingApprovalsContent` | `shows_table_with_data` | `reports` に 2 件のレポートデータ、`isLoading: false` | テーブルが表示され、2 行のレポートが描画される |
| WFL-FE-010 | 単体 | PendingApprovalsContent | error: ApiClientError | 異常系 | WFL-F04 | `55_ui_component/screens/workflow-pending.md §PendingApprovalsContent` | `shows_toast_on_server_error` | `error` に 500 エラーオブジェクト | AppToast でサーバーエラーメッセージが表示される |

#### PendingFilterBar

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| WFL-FE-011 | 単体 | PendingFilterBar | filters: PendingReportListParams | 正常系 | WFL-F04 | `55_ui_component/screens/workflow-pending.md §PendingFilterBar` | `debounces_applicant_name_input` | 申請者名入力フィールドに「テスト」と入力 | 300ms 経過後に onFilterChange が `{ applicant_name: "テスト" }` で呼び出される。300ms 以内には呼び出されない。※ ページ統合テスト WFL-FE-002 に包含して実装 |
| WFL-FE-012 | 単体 | PendingFilterBar | onFilterChange: function | 正常系 | WFL-F04 | `55_ui_component/screens/workflow-pending.md §PendingFilterBar` | `resets_filter_on_button_click` | フィルタ適用中（`filters: { applicant_name: "田中" }`）にリセットボタンをクリック | onFilterChange が `{ applicant_name: undefined }` で呼び出される。入力フィールドがクリアされる。※ ページ統合テスト WFL-FE-002/004 に包含して実装 |

#### PendingReportCount

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| WFL-FE-013 | 単体 | PendingReportCount | totalCount: number | 正常系 | WFL-F04 | `55_ui_component/screens/workflow-pending.md §PendingReportCount` | `displays_report_count` | `totalCount: 5`, `isFiltered: false` | 「5 件の承認待ちレポート」が表示される |
| WFL-FE-014 | 単体 | PendingReportCount | totalCount: number, isFiltered: boolean | 正常系 | WFL-F04 | `55_ui_component/screens/workflow-pending.md §PendingReportCount` | `hidden_when_zero_no_filter` | `totalCount: 0`, `isFiltered: false` | 件数コンポーネントが非表示になる。※ ページ統合テスト WFL-FE-007 に包含して実装 |
| WFL-FE-015 | 単体 | PendingReportCount | totalCount: number, isFiltered: boolean | 正常系 | WFL-F04 | `55_ui_component/screens/workflow-pending.md §PendingReportCount` | `shows_no_match_message_when_filtered` | `totalCount: 0`, `isFiltered: true` | 「条件に一致するレポートはありません。」が表示される。※ ページ統合テスト WFL-FE-008 に包含して実装 |

#### PendingReportTable

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| WFL-FE-016 | 単体 | PendingReportTable | reports: PendingReport[] | 正常系 | WFL-F04 | `55_ui_component/screens/workflow-pending.md §PendingReportTable` | `renders_table_columns` | `reports` に 1 件のレポート（applicant_name: "田中太郎", title: "4月交通費", total_amount: 15000, submitted_at: "2026-03-15"） | テーブルに申請者名「田中太郎」、タイトル「4月交通費」、合計金額「15,000」（3桁カンマ区切り）、提出日「2026/03/15」（YYYY/MM/DD）、遷移アイコンの 5 カラムが描画される |
| WFL-FE-017 | 単体 | PendingReportTable | reports: PendingReport[] | 正常系 | WFL-F04 | `55_ui_component/screens/workflow-pending.md §PendingReportTable` | `shows_self_label_for_own_report` | `reports` に `is_own_report: true` のレポート | 申請者名の横に「自分」ラベル（SelfLabel）が表示される |
| WFL-FE-018 | 単体 | PendingReportTable | reports: PendingReport[] | 正常系 | WFL-F04 | `55_ui_component/screens/workflow-pending.md §PendingReportTable` | `hides_self_label_for_other_report` | `reports` に `is_own_report: false` のレポート | 「自分」ラベルが表示されない |
| WFL-FE-019 | 単体 | PendingReportTable | onRowClick: function | 正常系 | WFL-F04 | `55_ui_component/screens/workflow-pending.md §PendingReportTable` | `navigates_to_detail_on_row_click` | レポート行をクリック（reportId: "test-report-id"） | onRowClick が "test-report-id" で呼び出される（SCR-RPT-004 への遷移） |
| WFL-FE-020 | 単体 | PendingReportTable | isLoading: boolean | 正常系 | WFL-F04 | `55_ui_component/screens/workflow-pending.md §PendingReportTable` | `shows_skeleton_when_loading` | `isLoading: true` | PageSkeleton（variant: "table"）が表示される。※ ページ統合テスト WFL-FE-006 に包含して実装 |
| WFL-FE-021 | 単体 | PendingReportTable | reports: PendingReport[] | 正常系 | WFL-F04 | `55_ui_component/screens/workflow-pending.md §PendingReportTable` | `shows_empty_state_when_no_reports` | `reports: []`, `isLoading: false`, `emptyMessage: "承認待ちのレポートはありません。"` | EmptyState にメッセージが表示される。※ ページ統合テスト WFL-FE-007 に包含して実装 |

#### SelfLabel（SCR-WFL-001 コンテキスト）

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| WFL-FE-022 | 単体 | SelfLabel | isOwnReport: boolean | 正常系 | WFL-F04 | `55_ui_component/screens/workflow-pending.md §SelfLabel` | `renders_self_chip_when_own` | `isOwnReport: true` | 「自分」テキストの Chip が描画される |
| WFL-FE-023 | 単体 | SelfLabel | isOwnReport: boolean | 正常系 | WFL-F04 | `55_ui_component/screens/workflow-pending.md §SelfLabel` | `renders_nothing_when_not_own` | `isOwnReport: false` | 何も描画されない（null を返す） |

#### FilterResetButton（SCR-WFL-001 コンテキスト）

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| WFL-FE-024 | 単体 | FilterResetButton | isFiltered: boolean | 正常系 | WFL-F04 | `55_ui_component/screens/workflow-pending.md §FilterResetButton` | `enabled_when_filter_applied` | `isFiltered: true` | ボタンが有効状態（disabled でない）で描画される |
| WFL-FE-025 | 単体 | FilterResetButton | isFiltered: boolean | 正常系 | WFL-F04 | `55_ui_component/screens/workflow-pending.md §FilterResetButton` | `disabled_when_no_filter` | `isFiltered: false` | ボタンが disabled で描画される |
| WFL-FE-026 | 単体 | FilterResetButton | onReset: function | 正常系 | WFL-F04 | `55_ui_component/screens/workflow-pending.md §FilterResetButton` | `calls_on_reset_on_click` | `isFiltered: true` でボタンをクリック | onReset コールバックが呼び出される |

#### usePendingReports

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| WFL-FE-027 | 単体 | - | usePendingReports | 正常系 | WFL-F04 | `55_ui_component/state-management.md §usePendingReports` | `fetches_pending_reports_with_params` | `{ page: 1, per_page: 20, applicant_name: "田中" }` | GET /api/workflow/pending?page=1&per_page=20&applicant_name=田中 が呼び出される。レスポンスデータが data として返る |
| WFL-FE-028 | 単体 | - | usePendingReports | 正常系 | WFL-F04 | `55_ui_component/state-management.md §usePendingReports` | `uses_correct_query_key` | `{ page: 1 }` | クエリキーが `['workflow', 'pending', { page: 1 }]` である |
| WFL-FE-029 | 単体 | - | usePendingReports | 正常系 | WFL-F04 | `55_ui_component/state-management.md §usePendingReports` | `respects_stale_time` | 初回フェッチ後 30 秒以内に再レンダリング | 再フェッチが発生しない（staleTime: 30秒） |

---

### SCR-WFL-002 支払待ち一覧

#### PayableReportsPage

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| WFL-FE-030 | 単体 | PayableReportsPage | usePayableReports | 正常系 | WFL-F05 | `55_ui_component/screens/workflow-payable.md §PayableReportsPage` | `renders_payable_reports_page_with_data` | usePayableReports が 3 件のレポートデータを返す | PayableReportsContent にレポート一覧が表示される。件数表示「3 件の支払待ちレポート」が表示される |
| WFL-FE-031 | 単体 | PayableReportsPage | usePayableReports | 正常系 | WFL-F05 | `55_ui_component/screens/workflow-payable.md §PayableReportsPage` | `manages_filter_state` | 申請者名フィルタに「田中」と入力後 300ms 待機 | usePayableReports が `{ applicant_name: "田中", page: 1 }` で呼び出される |
| WFL-FE-032 | 単体 | PayableReportsPage | usePayableReports | 正常系 | WFL-F05 | `55_ui_component/screens/workflow-payable.md §PayableReportsPage` | `manages_pagination_state` | ページネーションで 2 ページ目をクリック | usePayableReports が `{ page: 2 }` で呼び出される |
| WFL-FE-033 | 単体 | PayableReportsPage | usePayableReports | 正常系 | WFL-F05 | `55_ui_component/screens/workflow-payable.md §PayableReportsPage` | `resets_page_on_filter_change` | 2 ページ目を表示中にフィルタを変更 | ページが 1 にリセットされ、usePayableReports が `{ page: 1 }` で呼び出される |
| WFL-FE-034 | 単体 | PayableReportsPage | usePayableReports | 認可 | RBAC-F01, RBC-001 | `55_ui_component/screens/workflow-payable.md §PayableReportsPage` | `redirects_non_accounting_on_403` | usePayableReports が 403 エラーを返す（Member ロールでのアクセスを想定） | ダッシュボード（SCR-DASH-001）にリダイレクトされる |

#### PayableReportsContent

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| WFL-FE-035 | 単体 | PayableReportsContent | isLoading: boolean | 正常系 | WFL-F05 | `55_ui_component/screens/workflow-payable.md §PayableReportsContent` | `shows_skeleton_when_loading` | `isLoading: true` | PageSkeleton（variant: "table"）が表示される。テーブルは非表示 |
| WFL-FE-036 | 単体 | PayableReportsContent | reports: PayableReport[] | 正常系 | WFL-F05 | `55_ui_component/screens/workflow-payable.md §PayableReportsContent` | `shows_empty_state_no_filter` | `reports: []`, `isLoading: false`, `filters: {}` | EmptyState に「支払待ちのレポートはありません。」が表示される |
| WFL-FE-037 | 単体 | PayableReportsContent | reports: PayableReport[] | 正常系 | WFL-F05 | `55_ui_component/screens/workflow-payable.md §PayableReportsContent` | `shows_empty_state_with_filter` | `reports: []`, `isLoading: false`, `filters: { applicant_name: "存在しない名前" }` | 「条件に一致するレポートはありません。」が表示される。フィルタリセットボタンが表示される |
| WFL-FE-038 | 単体 | PayableReportsContent | reports: PayableReport[] | 正常系 | WFL-F05 | `55_ui_component/screens/workflow-payable.md §PayableReportsContent` | `shows_table_with_data` | `reports` に 2 件のレポートデータ、`isLoading: false` | テーブルが表示され、2 行のレポートが描画される |
| WFL-FE-039 | 単体 | PayableReportsContent | error: ApiClientError | 異常系 | WFL-F05 | `55_ui_component/screens/workflow-payable.md §PayableReportsContent` | `shows_toast_on_server_error` | `error` に 500 エラーオブジェクト | AppToast でサーバーエラーメッセージが表示される |

#### PayableFilterBar

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| WFL-FE-040 | 単体 | PayableFilterBar | filters: PayableReportListParams | 正常系 | WFL-F05 | `55_ui_component/screens/workflow-payable.md §PayableFilterBar` | `debounces_applicant_name_input` | 申請者名入力フィールドに「テスト」と入力 | 300ms 経過後に onFilterChange が `{ applicant_name: "テスト" }` で呼び出される。300ms 以内には呼び出されない。※ ページ統合テスト WFL-FE-031 に包含して実装 |
| WFL-FE-041 | 単体 | PayableFilterBar | onFilterChange: function | 正常系 | WFL-F05 | `55_ui_component/screens/workflow-payable.md §PayableFilterBar` | `resets_filter_on_button_click` | フィルタ適用中（`filters: { applicant_name: "田中" }`）にリセットボタンをクリック | onFilterChange が `{ applicant_name: undefined }` で呼び出される。入力フィールドがクリアされる。※ ページ統合テスト WFL-FE-031/033 に包含して実装 |

#### PayableReportCount

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| WFL-FE-042 | 単体 | PayableReportCount | totalCount: number | 正常系 | WFL-F05 | `55_ui_component/screens/workflow-payable.md §PayableReportCount` | `displays_report_count` | `totalCount: 5`, `isFiltered: false` | 「5 件の支払待ちレポート」が表示される |
| WFL-FE-043 | 単体 | PayableReportCount | totalCount: number, isFiltered: boolean | 正常系 | WFL-F05 | `55_ui_component/screens/workflow-payable.md §PayableReportCount` | `hidden_when_zero_no_filter` | `totalCount: 0`, `isFiltered: false` | 件数コンポーネントが非表示になる。※ ページ統合テスト WFL-FE-036 に包含して実装 |
| WFL-FE-044 | 単体 | PayableReportCount | totalCount: number, isFiltered: boolean | 正常系 | WFL-F05 | `55_ui_component/screens/workflow-payable.md §PayableReportCount` | `shows_no_match_message_when_filtered` | `totalCount: 0`, `isFiltered: true` | 「条件に一致するレポートはありません。」が表示される。※ ページ統合テスト WFL-FE-037 に包含して実装 |

#### PayableReportTable

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| WFL-FE-045 | 単体 | PayableReportTable | reports: PayableReport[] | 正常系 | WFL-F05 | `55_ui_component/screens/workflow-payable.md §PayableReportTable` | `renders_table_columns` | `reports` に 1 件のレポート（applicant_name: "田中太郎", title: "4月交通費", total_amount: 15000, approved_at: "2026-03-20"） | テーブルに申請者名「田中太郎」、タイトル「4月交通費」、合計金額「15,000」（3桁カンマ区切り）、承認日「2026/03/20」（YYYY/MM/DD）、遷移アイコンの 5 カラムが描画される |
| WFL-FE-046 | 単体 | PayableReportTable | reports: PayableReport[] | 正常系 | WFL-F05 | `55_ui_component/screens/workflow-payable.md §PayableReportTable` | `shows_self_label_for_own_report` | `reports` に `is_own_report: true` のレポート | 申請者名の横に「自分」ラベル（SelfLabel）が表示される |
| WFL-FE-047 | 単体 | PayableReportTable | reports: PayableReport[] | 正常系 | WFL-F05 | `55_ui_component/screens/workflow-payable.md §PayableReportTable` | `hides_self_label_for_other_report` | `reports` に `is_own_report: false` のレポート | 「自分」ラベルが表示されない |
| WFL-FE-048 | 単体 | PayableReportTable | onRowClick: function | 正常系 | WFL-F05 | `55_ui_component/screens/workflow-payable.md §PayableReportTable` | `navigates_to_detail_on_row_click` | レポート行をクリック（reportId: "test-report-id"） | onRowClick が "test-report-id" で呼び出される（SCR-RPT-004 への遷移） |
| WFL-FE-049 | 単体 | PayableReportTable | isLoading: boolean | 正常系 | WFL-F05 | `55_ui_component/screens/workflow-payable.md §PayableReportTable` | `shows_skeleton_when_loading` | `isLoading: true` | PageSkeleton（variant: "table"）が表示される。※ ページ統合テスト WFL-FE-035 に包含して実装 |
| WFL-FE-050 | 単体 | PayableReportTable | reports: PayableReport[] | 正常系 | WFL-F05 | `55_ui_component/screens/workflow-payable.md §PayableReportTable` | `shows_empty_state_when_no_reports` | `reports: []`, `isLoading: false`, `emptyMessage: "支払待ちのレポートはありません。"` | EmptyState にメッセージが表示される。※ ページ統合テスト WFL-FE-036 に包含して実装 |
| WFL-FE-051 | 単体 | PayableReportTable | reports: PayableReport[] | 正常系 | WFL-F05 | `55_ui_component/screens/workflow-payable.md §PayableReportTable` | `renders_approved_date_column` | `reports` に承認日データを含むレポート | 日付カラムが「承認日」であること（SCR-WFL-001 の「提出日」との差異） |

#### usePayableReports

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| WFL-FE-052 | 単体 | - | usePayableReports | 正常系 | WFL-F05 | `55_ui_component/state-management.md §usePayableReports` | `fetches_payable_reports_with_params` | `{ page: 1, per_page: 20, applicant_name: "田中" }` | GET /api/workflow/payable?page=1&per_page=20&applicant_name=田中 が呼び出される。レスポンスデータが data として返る |
| WFL-FE-053 | 単体 | - | usePayableReports | 正常系 | WFL-F05 | `55_ui_component/state-management.md §usePayableReports` | `uses_correct_query_key` | `{ page: 1 }` | クエリキーが `['workflow', 'payable', { page: 1 }]` である |
| WFL-FE-054 | 単体 | - | usePayableReports | 正常系 | WFL-F05 | `55_ui_component/state-management.md §usePayableReports` | `respects_stale_time` | 初回フェッチ後 30 秒以内に再レンダリング | 再フェッチが発生しない（staleTime: 30秒） |

#### SelfLabel（支払待ち一覧）

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| WFL-FE-079 | 単体 | SelfLabel | isOwnReport: boolean | 正常系 | RBC-012 | `55_ui_component/screens/workflow-payable.md §SelfLabel` | `payable_shows_self_label_true` | `isOwnReport: true` | 「自分」ラベルが Chip で表示される |
| WFL-FE-080 | 単体 | SelfLabel | isOwnReport: boolean | 正常系 | RBC-012 | `55_ui_component/screens/workflow-payable.md §SelfLabel` | `payable_hides_self_label_false` | `isOwnReport: false` | 「自分」ラベルが表示されない |

#### FilterResetButton（支払待ち一覧）

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| WFL-FE-081 | 単体 | FilterResetButton | isFiltered: boolean | 正常系 | WFL-F05 | `55_ui_component/screens/workflow-payable.md §FilterResetButton` | `payable_enabled_when_filtered` | `isFiltered: true` | リセットボタンが有効化される |
| WFL-FE-082 | 単体 | FilterResetButton | onReset: function | 正常系 | WFL-F05 | `55_ui_component/screens/workflow-payable.md §FilterResetButton` | `payable_calls_on_reset` | リセットボタンをクリック | onReset が呼び出される |

---

### SCR-RPT-004 レポート詳細（WorkflowActions）

#### WorkflowActions

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| WFL-FE-055 | 単体 | WorkflowActions | status: ReportStatus, currentUserRole: string | 正常系 | WFL-F01, WFL-F02 | `55_ui_component/screens/report-detail.md §WorkflowActions` | `shows_approve_reject_for_approver_submitted` | `status: "submitted"`, `currentUserRole: "approver"` | 「承認」ボタンと「却下」ボタンが表示される。「支払完了」ボタンは表示されない |
| WFL-FE-056 | 単体 | WorkflowActions | status: ReportStatus, currentUserRole: string | 正常系 | WFL-F03 | `55_ui_component/screens/report-detail.md §WorkflowActions` | `shows_pay_for_accounting_approved` | `status: "approved"`, `currentUserRole: "accounting"` | 「支払完了」ボタンが表示される。「承認」「却下」ボタンは表示されない |
| WFL-FE-057 | 単体 | WorkflowActions | status: ReportStatus, currentUserRole: string | 認可 | RBC-016 | `55_ui_component/screens/report-detail.md §WorkflowActions` | `hides_approve_reject_for_non_approver` | `status: "submitted"`, `currentUserRole: "member"` | 「承認」「却下」「支払完了」ボタンのいずれも表示されない |
| WFL-FE-058 | 単体 | WorkflowActions | status: ReportStatus, currentUserRole: string | 認可 | RBC-012 | `55_ui_component/screens/report-detail.md §WorkflowActions` | `hides_pay_for_non_accounting` | `status: "approved"`, `currentUserRole: "approver"` | 「支払完了」ボタンが表示されない |
| WFL-FE-059 | 単体 | WorkflowActions | status: ReportStatus, currentUserRole: string | 正常系 | WFL-F01 | `55_ui_component/screens/report-detail.md §WorkflowActions` | `hides_all_for_approver_non_submitted` | `status: "draft"`, `currentUserRole: "approver"` | 「承認」「却下」「支払完了」ボタンのいずれも表示されない |
| WFL-FE-060 | 単体 | WorkflowActions | status: ReportStatus, currentUserRole: string | 正常系 | WFL-F01 | `55_ui_component/screens/report-detail.md §WorkflowActions` | `hides_all_for_approver_approved` | `status: "approved"`, `currentUserRole: "approver"` | 「承認」「却下」「支払完了」ボタンのいずれも表示されない |
| WFL-FE-061 | 単体 | WorkflowActions | status: ReportStatus, currentUserRole: string | 正常系 | WFL-F03 | `55_ui_component/screens/report-detail.md §WorkflowActions` | `hides_all_for_accounting_non_approved` | `status: "submitted"`, `currentUserRole: "accounting"` | 「承認」「却下」「支払完了」ボタンのいずれも表示されない |
| WFL-FE-062 | 単体 | WorkflowActions | status: ReportStatus, currentUserRole: string | 正常系 | WFL-F03 | `55_ui_component/screens/report-detail.md §WorkflowActions` | `hides_all_for_accounting_paid` | `status: "paid"`, `currentUserRole: "accounting"` | 「承認」「却下」「支払完了」ボタンのいずれも表示されない |
| WFL-FE-063 | 単体 | WorkflowActions | onApprove: function | 正常系 | WFL-F01 | `55_ui_component/screens/report-detail.md §WorkflowActions` | `calls_on_approve_on_click` | `status: "submitted"`, `currentUserRole: "approver"`。「承認」ボタンをクリック | onApprove コールバックが呼び出される |
| WFL-FE-064 | 単体 | WorkflowActions | onReject: function | 正常系 | WFL-F02 | `55_ui_component/screens/report-detail.md §WorkflowActions` | `calls_on_reject_on_click` | `status: "submitted"`, `currentUserRole: "approver"`。「却下」ボタンをクリック | onReject コールバックが呼び出される |
| WFL-FE-065 | 単体 | WorkflowActions | onMarkAsPaid: function | 正常系 | WFL-F03 | `55_ui_component/screens/report-detail.md §WorkflowActions` | `calls_on_mark_as_paid_on_click` | `status: "approved"`, `currentUserRole: "accounting"`。「支払完了」ボタンをクリック | onMarkAsPaid コールバックが呼び出される |
| WFL-FE-066 | 単体 | WorkflowActions | pendingAction: string | 正常系 | WFL-F01 | `55_ui_component/screens/report-detail.md §WorkflowActions` | `disables_buttons_during_approve` | `status: "submitted"`, `currentUserRole: "approver"`, `pendingAction: "approve"` | 「承認」ボタンが disabled かつスピナー表示。「却下」ボタンも disabled |
| WFL-FE-067 | 単体 | WorkflowActions | pendingAction: string | 正常系 | WFL-F02 | `55_ui_component/screens/report-detail.md §WorkflowActions` | `disables_buttons_during_reject` | `status: "submitted"`, `currentUserRole: "approver"`, `pendingAction: "reject"` | 「却下」ボタンが disabled かつスピナー表示。「承認」ボタンも disabled |
| WFL-FE-068 | 単体 | WorkflowActions | pendingAction: string | 正常系 | WFL-F03 | `55_ui_component/screens/report-detail.md §WorkflowActions` | `disables_button_during_pay` | `status: "approved"`, `currentUserRole: "accounting"`, `pendingAction: "pay"` | 「支払完了」ボタンが disabled かつスピナー表示 |

#### useApproveReport

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| WFL-FE-069 | 単体 | - | useApproveReport | 正常系 | WFL-F01 | `55_ui_component/state-management.md §useApproveReport` | `calls_approve_api` | `{ id: "report-1", updated_at: "2026-04-01T00:00:00Z" }` | POST /api/workflow/report-1/approve が `{ updated_at: "2026-04-01T00:00:00Z" }` で呼び出される |
| WFL-FE-070 | 単体 | - | useApproveReport | 正常系 | WFL-F01 | `55_ui_component/state-management.md §useApproveReport` | `calls_approve_api_with_comment` | `{ id: "report-1", comment: "問題ありません", updated_at: "2026-04-01T00:00:00Z" }` | POST /api/workflow/report-1/approve が `{ comment: "問題ありません", updated_at: "..." }` で呼び出される |
| WFL-FE-071 | 単体 | - | useApproveReport | 正常系 | WFL-F01 | `55_ui_component/state-management.md §useApproveReport` | `invalidates_caches_on_success` | 承認 API が成功レスポンスを返す | `['reports', 'detail', id]`, `['workflow', 'pending']`, `['workflow', 'payable']`, `['dashboard']` のキャッシュが無効化される |
| WFL-FE-072 | 単体 | - | useApproveReport | 異常系 | WFL-F01 | `55_ui_component/state-management.md §useApproveReport` | `returns_error_on_409_conflict` | 承認 API が 409 CONFLICT を返す | error に ApiClientError が設定される。キャッシュは無効化されない |

#### useRejectReport

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| WFL-FE-073 | 単体 | - | useRejectReport | 正常系 | WFL-F02 | `55_ui_component/state-management.md §useRejectReport` | `calls_reject_api_with_reason` | `{ id: "report-1", reason: "領収書が不明瞭です", updated_at: "2026-04-01T00:00:00Z" }` | POST /api/workflow/report-1/reject が `{ reason: "領収書が不明瞭です", updated_at: "..." }` で呼び出される |
| WFL-FE-074 | 単体 | - | useRejectReport | 正常系 | WFL-F02 | `55_ui_component/state-management.md §useRejectReport` | `invalidates_caches_on_success` | 却下 API が成功レスポンスを返す | `['reports', 'detail', id]`, `['workflow', 'pending']`, `['dashboard']` のキャッシュが無効化される |
| WFL-FE-075 | 単体 | - | useRejectReport | 異常系 | WFL-F02 | `55_ui_component/state-management.md §useRejectReport` | `returns_error_on_409_conflict` | 却下 API が 409 CONFLICT を返す | error に ApiClientError が設定される。キャッシュは無効化されない |

#### useMarkAsPaid

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| WFL-FE-076 | 単体 | - | useMarkAsPaid | 正常系 | WFL-F03 | `55_ui_component/state-management.md §useMarkAsPaid` | `calls_pay_api` | `{ id: "report-1", updated_at: "2026-04-01T00:00:00Z" }` | POST /api/workflow/report-1/pay が `{ updated_at: "2026-04-01T00:00:00Z" }` で呼び出される |
| WFL-FE-077 | 単体 | - | useMarkAsPaid | 正常系 | WFL-F03 | `55_ui_component/state-management.md §useMarkAsPaid` | `invalidates_caches_on_success` | 支払完了 API が成功レスポンスを返す | `['reports', 'detail', id]`, `['workflow', 'payable']`, `['dashboard']`, `['reports', 'all']` のキャッシュが無効化される |
| WFL-FE-078 | 単体 | - | useMarkAsPaid | 異常系 | WFL-F03 | `55_ui_component/state-management.md §useMarkAsPaid` | `returns_error_on_409_conflict` | 支払完了 API が 409 CONFLICT を返す | error に ApiClientError が設定される。キャッシュは無効化されない |

---

### FE テストID サマリー

| 範囲 | テストID | 件数 |
|------|---------|------|
| PendingApprovalsPage | WFL-FE-001〜WFL-FE-005 | 5 件 |
| PendingApprovalsContent | WFL-FE-006〜WFL-FE-010 | 5 件 |
| PendingFilterBar | WFL-FE-011〜WFL-FE-012 | 2 件 |
| PendingReportCount | WFL-FE-013〜WFL-FE-015 | 3 件 |
| PendingReportTable | WFL-FE-016〜WFL-FE-021 | 6 件 |
| SelfLabel | WFL-FE-022〜WFL-FE-023 | 2 件 |
| FilterResetButton | WFL-FE-024〜WFL-FE-026 | 3 件 |
| usePendingReports | WFL-FE-027〜WFL-FE-029 | 3 件 |
| PayableReportsPage | WFL-FE-030〜WFL-FE-034 | 5 件 |
| PayableReportsContent | WFL-FE-035〜WFL-FE-039 | 5 件 |
| PayableFilterBar | WFL-FE-040〜WFL-FE-041 | 2 件 |
| PayableReportCount | WFL-FE-042〜WFL-FE-044 | 3 件 |
| PayableReportTable | WFL-FE-045〜WFL-FE-051 | 7 件 |
| usePayableReports | WFL-FE-052〜WFL-FE-054 | 3 件 |
| SelfLabel（支払待ち） | WFL-FE-079〜WFL-FE-080 | 2 件 |
| FilterResetButton（支払待ち） | WFL-FE-081〜WFL-FE-082 | 2 件 |
| WorkflowActions | WFL-FE-055〜WFL-FE-068 | 14 件 |
| useApproveReport | WFL-FE-069〜WFL-FE-072 | 4 件 |
| useRejectReport | WFL-FE-073〜WFL-FE-075 | 3 件 |
| useMarkAsPaid | WFL-FE-076〜WFL-FE-078 | 3 件 |
| ApprovalListPage per_page UI 結合（issue #147） | WFL-FE-083〜WFL-FE-086 | 4 件 |
| PaymentListPage per_page UI 結合（issue #147） | WFL-FE-087〜WFL-FE-090 | 4 件 |
| **FE 合計** | WFL-FE-001〜WFL-FE-090 | **90 件** |
| **APR-FE 合計** | APR-FE-001〜APR-FE-004 | **4 件** |
| **PAY-FE 合計** | PAY-FE-001〜PAY-FE-005 | **5 件** |
| **FE 全体合計** | | **99 件** |

---

### 追記テスト（issue-106 同期ロールチェック + 403 トースト）

#### ApprovalListPage -- 同期ロールチェック + 403 トースト

テスト ID プレフィックス: `APR-FE-`（ApprovalListPage 専用。承認待ち一覧が PendingApprovalsPage からリネームされた画面に対応）

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| APR-FE-001 | 単体 | ApprovalListPage | usePendingReports | 認可 | RBAC-F01, RBC-001 | `55_ui_component/screens/workflow-pending.md §PendingApprovalsPage` | `redirects_on_403_with_toast` | usePendingReports が 403 エラーを返す | ダッシュボード（`/dashboard`）にリダイレクトされ、navigate の state.toast にトーストメッセージ「この画面にアクセスする権限がありません。」が含まれる（issue-088 対応） |
| APR-FE-002 | 単体 | ApprovalListPage | useCurrentUser | 認可 | RBAC-F01, RBC-001 | `55_ui_component/screens/workflow-pending.md §PendingApprovalsPage` | `sync_role_check_member_redirects` | useCurrentUser が Member ロールを返す | 同期ロールチェックにより即座にダッシュボード（`/dashboard`）にリダイレクトされる（issue-106 対応） |
| APR-FE-003 | 単体 | ApprovalListPage | useCurrentUser | 認可 | RBAC-F01, RBC-001 | `55_ui_component/screens/workflow-pending.md §PendingApprovalsPage` | `sync_role_check_approver_renders` | useCurrentUser が Approver ロールを返す | 同期ロールチェックを通過し、通常レンダリングされる（issue-106 対応） |
| APR-FE-004 | 単体 | ApprovalListPage | useCurrentUser | 認可 | RBAC-F01, RBC-001 | `55_ui_component/screens/workflow-pending.md §PendingApprovalsPage` | `sync_role_check_admin_redirects` | useCurrentUser が Admin ロールを返す | 同期ロールチェックにより即座にダッシュボード（`/dashboard`）にリダイレクトされる（authz.md 正本: Approver のみ許可。issue-106 対応） |

#### PaymentListPage -- 同期ロールチェック + 403 トースト

テスト ID プレフィックス: `PAY-FE-`（PaymentListPage 専用。支払待ち一覧が PayableReportsPage からリネームされた画面に対応）

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| PAY-FE-001 | 単体 | PaymentListPage | usePayableReports | 認可 | RBAC-F01, RBC-001 | `55_ui_component/screens/workflow-payable.md §PayableReportsPage` | `redirects_on_403_with_toast` | usePayableReports が 403 エラーを返す | ダッシュボード（`/dashboard`）にリダイレクトされ、navigate の state.toast にトーストメッセージ「この画面にアクセスする権限がありません。」が含まれる（issue-088 対応） |
| PAY-FE-002 | 単体 | PaymentListPage | useCurrentUser | 認可 | RBAC-F01, RBC-001 | `55_ui_component/screens/workflow-payable.md §PayableReportsPage` | `sync_role_check_member_redirects` | useCurrentUser が Member ロールを返す | 同期ロールチェックにより即座にダッシュボード（`/dashboard`）にリダイレクトされる（issue-106 対応） |
| PAY-FE-003 | 単体 | PaymentListPage | useCurrentUser | 認可 | RBAC-F01, RBC-001 | `55_ui_component/screens/workflow-payable.md §PayableReportsPage` | `sync_role_check_accounting_renders` | useCurrentUser が Accounting ロールを返す | 同期ロールチェックを通過し、通常レンダリングされる（issue-106 対応） |
| PAY-FE-004 | 単体 | PaymentListPage | useCurrentUser | 認可 | RBAC-F01, RBC-001 | `55_ui_component/screens/workflow-payable.md §PayableReportsPage` | `sync_role_check_admin_redirects` | useCurrentUser が Admin ロールを返す | 同期ロールチェックにより即座にダッシュボード（`/dashboard`）にリダイレクトされる（authz.md 正本: Accounting のみ許可。issue-106 対応） |
| PAY-FE-005 | 単体 | PaymentListPage | useCurrentUser | 認可 | RBAC-F01, RBC-001 | `55_ui_component/screens/workflow-payable.md §PayableReportsPage` | `sync_role_check_approver_redirects` | useCurrentUser が Approver ロールを返す | 同期ロールチェックにより即座にダッシュボード（`/dashboard`）にリダイレクトされる（issue-106 対応） |

---

### 追記テスト（issue #147: per_page UI セレクタ + URL 駆動 Page 結合）

issue #147 対応として、`ApprovalListPage`（SCR-WFL-001）および `PaymentListPage`（SCR-WFL-002）における URL ⇔ UI ⇔ API 連動経路を Page 結合テストとして採番する。`PageSizeSelector` / `AppPaginationFooter` の単体テストは `reports.md` §FE-6 の `PSS-001〜005` / `APF-001〜007` に集約済み。BE 側は WFL-005 / WFL-046（既存 `_Pagination` テスト）が既にカバーしているため、BE への追加は行わない。

採番方針: 既存 WFL-FE 系最大 ID は `WFL-FE-082`（FE テスト ID サマリー参照）。次番 `WFL-FE-083` から採用する。`ApprovalListPage` 用に `WFL-FE-083〜086`、`PaymentListPage` 用に `WFL-FE-087〜090` を割り当てる。

#### ApprovalListPage（SCR-WFL-001）

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| WFL-FE-083 | 単体 | ApprovalListPage | usePendingReports（MSW）、useSearchParams | 正常系 | WFL-F04 | `50_detail_design/screens/workflow-pending.md §7`, `55_ui_component/state-management.md §3.1` | `test_ApprovalListPage_url_per_page_reflects_to_selector_and_api` | `/approvals?per_page=10` で開く。MSW で `usePendingReports` が 10 件 + `pagination.total_pages > 1` を返すよう設定。useCurrentUser が Approver ロールを返すようモック | テーブルに 10 件のみ描画される。フッターの `PageSizeSelector` が「10」を表示する。`usePendingReports` への引数に `per_page: 10` が渡り、API URL に `?per_page=10` が含まれる |
| WFL-FE-084 | 単体 | ApprovalListPage | usePendingReports（MSW）、useSearchParams | 正常系 | WFL-F04 | `50_detail_design/screens/workflow-pending.md §7`, `55_ui_component/state-management.md §3.1` | `test_ApprovalListPage_selector_change_updates_url_and_resets_page` | `/approvals?page=3&per_page=10` で開いた状態で `PageSizeSelector` から「50」を選択 | URL が `/approvals?page=1&per_page=50` に更新される（`page=1` リセット）。`setSearchParams` は 1 回のコールに集約される（race 回避）。`PageSizeSelector` の現在値が「50」に更新される |
| WFL-FE-085 | 単体 | ApprovalListPage | usePendingReports、useSearchParams | 正常系 | WFL-F04 | `50_detail_design/screens/workflow-pending.md §7`, `55_ui_component/state-management.md §3.1` | `test_ApprovalListPage_url_non_standard_per_page_appends_to_options` | `/approvals?per_page=1` で開く（URL に標準外値）。useCurrentUser が Approver ロールを返すようモック | `PageSizeSelector` の選択肢が `[1, 10, 20, 50, 100]` に動的拡張され、現在値「1」が選択される（パターン X 動作） |
| WFL-FE-086 | 単体 | ApprovalListPage | usePendingReports、useSearchParams | 正常系 | WFL-F04 | `50_detail_design/screens/workflow-pending.md §7`, `55_ui_component/state-management.md §3.1` | `test_ApprovalListPage_url_invalid_per_page_falls_back_to_20` | `/approvals?per_page=abc`（NaN） / `?per_page=-5`（負数） の 2 サブケース。useCurrentUser が Approver ロールを返すようモック | 両サブケースとも `usePendingReports` への引数 `per_page` が `20`（FE フォールバック）になり、`PageSizeSelector` も「20」を表示する（issue #147 Q4） |

#### PaymentListPage（SCR-WFL-002）

| テストID | テストレベル | 対象コンポーネント | 対象 Props / Hook | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---|---|---|---|---|---|---|---|---|---|
| WFL-FE-087 | 単体 | PaymentListPage | usePayableReports（MSW）、useSearchParams | 正常系 | WFL-F05 | `50_detail_design/screens/workflow-payable.md §7`, `55_ui_component/state-management.md §3.1` | `test_PaymentListPage_url_per_page_reflects_to_selector_and_api` | `/payments?per_page=10` で開く。MSW で `usePayableReports` が 10 件 + `pagination.total_pages > 1` を返すよう設定。useCurrentUser が Accounting ロールを返すようモック | テーブルに 10 件のみ描画される。フッターの `PageSizeSelector` が「10」を表示する。`usePayableReports` への引数に `per_page: 10` が渡り、API URL に `?per_page=10` が含まれる |
| WFL-FE-088 | 単体 | PaymentListPage | usePayableReports（MSW）、useSearchParams | 正常系 | WFL-F05 | `50_detail_design/screens/workflow-payable.md §7`, `55_ui_component/state-management.md §3.1` | `test_PaymentListPage_selector_change_updates_url_and_resets_page` | `/payments?page=3&per_page=10` で開いた状態で `PageSizeSelector` から「50」を選択 | URL が `/payments?page=1&per_page=50` に更新される（`page=1` リセット）。`setSearchParams` は 1 回のコールに集約される（race 回避）。`PageSizeSelector` の現在値が「50」に更新される |
| WFL-FE-089 | 単体 | PaymentListPage | usePayableReports、useSearchParams | 正常系 | WFL-F05 | `50_detail_design/screens/workflow-payable.md §7`, `55_ui_component/state-management.md §3.1` | `test_PaymentListPage_url_non_standard_per_page_appends_to_options` | `/payments?per_page=1` で開く（URL に標準外値）。useCurrentUser が Accounting ロールを返すようモック | `PageSizeSelector` の選択肢が `[1, 10, 20, 50, 100]` に動的拡張され、現在値「1」が選択される（パターン X 動作） |
| WFL-FE-090 | 単体 | PaymentListPage | usePayableReports、useSearchParams | 正常系 | WFL-F05 | `50_detail_design/screens/workflow-payable.md §7`, `55_ui_component/state-management.md §3.1` | `test_PaymentListPage_url_invalid_per_page_falls_back_to_20` | `/payments?per_page=abc`（NaN） / `?per_page=-5`（負数） の 2 サブケース。useCurrentUser が Accounting ロールを返すようモック | 両サブケースとも `usePayableReports` への引数 `per_page` が `20`（FE フォールバック）になり、`PageSizeSelector` も「20」を表示する（issue #147 Q4） |

**備考**:
- 採番根拠: `WFL-FE-082` の次番から開始。`APR-FE-` / `PAY-FE-` 系（同期ロールチェック追記）は別系統のため通番に影響しない。
- 実装ファイル候補: `expense-saas/frontend/src/pages/workflow/__tests__/ApprovalListPage.test.tsx` / `PaymentListPage.test.tsx`（**既存ファイル拡張**。issue #147 末尾「実ファイルパスの確定」に従い、画面ファイル名 `ApprovalListPage.tsx` / `PaymentListPage.tsx` に対応する確定パス）。
- BE per_page 動作テストは既存 `WFL-005`（`TestListPendingReports_Pagination`）/ `WFL-046`（`TestListPayableReports_Pagination`）が既にカバー済み。本 issue では BE への追加は行わない。
- `ApprovalListPage` / `PaymentListPage` 用のフッター常時表示（Q3）の検証は `APF-002 / APF-003` に集約しているため本節では再検証しない。

---

### FE テスト実装ガイド

#### テストファイル配置

```
expense-saas/frontend/
  src/
    pages/approvals/__tests__/
      PendingApprovalsPage.test.tsx
      PendingApprovalsContent.test.tsx
      PendingFilterBar.test.tsx
      PendingReportCount.test.tsx
      PendingReportTable.test.tsx
    pages/payments/__tests__/
      PayableReportsPage.test.tsx
      PayableReportsContent.test.tsx
      PayableFilterBar.test.tsx
      PayableReportCount.test.tsx
      PayableReportTable.test.tsx
    pages/reports/__tests__/
      WorkflowActions.test.tsx
    components/ui/__tests__/
      SelfLabel.test.tsx
      FilterResetButton.test.tsx
    hooks/__tests__/
      useWorkflow.test.ts
```

#### テスト環境・ツール

- テストランナー: Vitest
- コンポーネントテスト: React Testing Library（`@testing-library/react`）
- Hook テスト: `renderHook`（`@testing-library/react`）
- API モック: MSW（Mock Service Worker）
- ルーティングモック: `MemoryRouter`（`react-router-dom`）
- TanStack Query ラッパー: テスト用 `QueryClientProvider` を用意（`retry: false`, `cacheTime: 0`）

#### コンポーネントテストの基本構造

```typescript
import { render, screen } from '@testing-library/react';
import { describe, it, expect, vi } from 'vitest';
import { PendingReportTable } from '../PendingReportTable';

describe('PendingReportTable', () => {
  it('renders_table_columns', () => {
    const reports = [
      {
        id: 'test-report-1',
        applicant_name: '田中太郎',
        title: '4月交通費',
        total_amount: 15000,
        submitted_at: '2026-03-15T00:00:00Z',
        is_own_report: false,
      },
    ];

    render(
      <PendingReportTable
        reports={reports}
        isLoading={false}
        emptyMessage="承認待ちのレポートはありません。"
        onRowClick={vi.fn()}
      />
    );

    expect(screen.getByText('田中太郎')).toBeInTheDocument();
    expect(screen.getByText('4月交通費')).toBeInTheDocument();
    expect(screen.getByText('15,000')).toBeInTheDocument();
    expect(screen.getByText('2026/03/15')).toBeInTheDocument();
  });
});
```

#### Hook テストの基本構造

```typescript
import { renderHook, waitFor } from '@testing-library/react';
import { describe, it, expect } from 'vitest';
import { usePendingReports } from '../useWorkflow';
import { createQueryWrapper } from '../../test-utils/queryWrapper';
import { server } from '../../mocks/server';
import { http, HttpResponse } from 'msw';

describe('usePendingReports', () => {
  it('fetches_pending_reports_with_params', async () => {
    server.use(
      http.get('/api/workflow/pending', ({ request }) => {
        const url = new URL(request.url);
        // パラメータの検証
        return HttpResponse.json({
          data: [/* ... */],
          pagination: { page: 1, per_page: 20, total_count: 1, total_pages: 1 },
        });
      })
    );

    const { result } = renderHook(
      () => usePendingReports({ page: 1, per_page: 20, applicant_name: '田中' }),
      { wrapper: createQueryWrapper() }
    );

    await waitFor(() => expect(result.current.isSuccess).toBe(true));
    expect(result.current.data?.data).toHaveLength(1);
  });
});
```

#### 補足

- SelfLabel / FilterResetButton は共通コンポーネントだが、SCR-WFL-001 / SCR-WFL-002 での使用コンテキストに即したテストケースをここに記載している。共通コンポーネント自体の汎用テストは `common-components.md` のテストケースファイルに集約する
- WorkflowActions は SCR-RPT-004 のコンポーネントだが、ワークフロー機能のテストとしてこのファイルに記載する。レポート詳細画面全体のテストは `reports.md` に記載する
- 自己承認禁止（RBC-016）・自己支払処理禁止（RBC-012）の UI 制御は、ReportActionBar が `isOwner` フラグで WorkflowActions の表示/非表示を制御するため、WorkflowActions 自体には `isOwner` Props が存在しない。この制御のテストは ReportActionBar のテストケース（`reports.md`）に記載する
