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
| WFL-025 | 統合 | handler | 異常系 | WFL-F01 | openapi.yaml#approveReport | `TestApproveReport_MissingUpdatedAt` | Approver で認証。リクエストボディに `updated_at` フィールドなし | 400 または 422（バリデーションエラー） |

---

### rejectReport — POST /api/workflow/{id}/reject

#### 正常系

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|-------------------|---------|
| WFL-026 | 統合 | handler | 正常系 | WFL-F02, WFL-012 | openapi.yaml#rejectReport, state_machine.md#T3 | `TestRejectReport_Success` | Approver で認証。対象: `report_submitted`。リクエストボディ: `{"rejection_reason": "領収書が不明瞭です", "updated_at": "<updated_at>"}` | 200 OK。レスポンスの `data.status` が `"rejected"`。`data.rejected_by` が Approver のユーザーID。`data.rejection_reason` が `"領収書が不明瞭です"` |

#### 却下理由バリデーション（state_machine.md §T3 事前条件 No.3: rejection_reason 非空）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|-------------------|---------|
| WFL-027 | 統合 | handler | 異常系 | WFL-012 | openapi.yaml#rejectReport, state_machine.md#T3 | `TestRejectReport_MissingRejectionReason` | Approver で認証。リクエストボディに `rejection_reason` フィールドなし（`{"updated_at": "<updated_at>"}` のみ） | 422 MISSING_REJECTION_REASON |
| WFL-028 | 統合 | handler | 異常系 | WFL-012 | openapi.yaml#rejectReport, state_machine.md#T3 | `TestRejectReport_EmptyRejectionReason` | Approver で認証。リクエストボディ: `{"rejection_reason": "", "updated_at": "<updated_at>"}` | 422 MISSING_REJECTION_REASON |
| WFL-029 | 統合 | handler | 境界値 | WFL-012 | openapi.yaml#rejectReport | `TestRejectReport_RejectionReasonMaxLength` | Approver で認証。`rejection_reason` が1000文字の文字列 | 200 OK（境界値: 上限ちょうど） |
| WFL-030 | 統合 | handler | 境界値 | WFL-012 | openapi.yaml#rejectReport | `TestRejectReport_RejectionReasonTooLong` | Approver で認証。`rejection_reason` が1001文字の文字列 | 400 または 422（バリデーションエラー） |

#### 状態遷移の禁止パス（ハンドラ層統合テスト: X8 相当）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|-------------------|---------|
| WFL-031 | 統合 | handler | 状態遷移 | WFL-002 | openapi.yaml#rejectReport, state_machine.md#X8 | `TestRejectReport_AlreadyApproved` | Approver で認証。対象: `report_approved`（X8: approved→rejected は禁止）。リクエストボディ: `{"rejection_reason": "理由", "updated_at": "<updated_at>"}` | 422 INVALID_STATE_TRANSITION |
| WFL-032 | 統合 | handler | 状態遷移 | WFL-002 | openapi.yaml#rejectReport | `TestRejectReport_DraftState` | Approver で認証。対象: draft 状態のレポート | 422 INVALID_STATE_TRANSITION |
| WFL-033 | 統合 | handler | 状態遷移 | WFL-004 | openapi.yaml#rejectReport, state_machine.md#X9 | `TestRejectReport_RejectedState` | Approver で認証。対象: rejected 状態のレポート（X9: 終端状態からの遷移不可） | 422 INVALID_STATE_TRANSITION |
| WFL-034 | 統合 | handler | 状態遷移 | WFL-004 | openapi.yaml#rejectReport, state_machine.md#X10 | `TestRejectReport_PaidState` | Approver で認証。対象: paid 状態のレポート（X10: 終端状態からの遷移不可） | 422 INVALID_STATE_TRANSITION |

#### 自己却下禁止（RBC-016）

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|---------------|-------------------|---------|
| WFL-035 | 統合 | handler | 認可 | RBC-016 | openapi.yaml#rejectReport, authz.md#4.2 | `TestRejectReport_SelfRejection` | Test Approver 本人が作成した submitted レポートを、同じ Test Approver で却下しようとする。リクエストボディ: `{"rejection_reason": "理由", "updated_at": "<updated_at>"}` | 403 SELF_APPROVAL_NOT_ALLOWED |

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
| WFL-063 | 統合 | handler | 異常系 | WFL-F03 | openapi.yaml#markReportAsPaid | `TestMarkReportAsPaid_MissingUpdatedAt` | Accounting で認証。リクエストボディに `updated_at` フィールドなし | 400 または 422（バリデーションエラー） |

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
| **合計** | WFL-001〜WFL-062 | **62 件** |

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
