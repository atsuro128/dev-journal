# テストケース: 横断テスト（CRS-）

## この文書の役割

| 項目 | 内容 |
|------|------|
| 目的 | テナント分離・RBAC マトリクス・E2E シナリオ・非機能テストなど、複数エンドポイントをまたがる横断観点のテストケースを集約する |
| 正本情報 | テナント分離（CRS-001〜）、RBAC マトリクス（CRS-021〜）、E2E シナリオ（CRS-055〜）、非機能（CRS-076〜）のテストケース一覧 |
| 扱わない内容 | 個別エンドポイント固有のビジネスロジックテスト（各機能別 test_cases/*.md に記載） |
| 主な参照元 | `10_requirements/policies.md#RBC-*`, `10_requirements/policies.md#TNT-*`, `50_detail_design/authz.md`, `50_detail_design/security.md` |
| 主な参照先 | `60_test/traceability.md`, テスト実装（Step 9） |

---

本ファイルは複数のエンドポイントをまたがる横断観点のテストケースを集約する。

| 観点 | テストID範囲 |
|------|-------------|
| §1 テナント分離マトリクス | CRS-001〜CRS-020（実定義: CRS-001〜CRS-016） |
| §2 RBAC マトリクス | CRS-021〜CRS-054 |
| §3 E2E シナリオ | CRS-055〜CRS-075 |
| §4 非機能テスト（レート制限・レスポンスタイム） | CRS-076〜CRS-088 |

### 責務境界（`test_strategy.md` §7.2 より）

- テナント分離テスト（全リソース × テナント越境 = 404）: このファイルに集約
- RBACマトリクス（全エンドポイント × 全ロールの整合性確認）: このファイルに集約
- E2E シナリオ（複数機能をまたぐ主要フロー）: このファイルに集約
- 非機能テスト（レート制限・レスポンスタイム）: このファイルに集約

---

## フィクスチャ参照

標準フィクスチャは `test_strategy.md` §4 を参照。本ファイルで使用するフィクスチャを以下にまとめる。

### テナント / ユーザー

| 変数名 | テナント | ロール | ユーザーID |
|--------|---------|--------|-----------|
| `tenantA` | Test Company A | - | `aaaaaaaa-0001-0001-0001-000000000001` |
| `tenantB` | Test Company B | - | `bbbbbbbb-0002-0002-0002-000000000002` |
| `userAdmin` | テナントA | Admin | `aaaaaaaa-1111-1111-1111-000000000001` |
| `userApprover` | テナントA | Approver | `aaaaaaaa-2222-2222-2222-000000000002` |
| `userMember` | テナントA | Member | `aaaaaaaa-3333-3333-3333-000000000003` |
| `userAccounting` | テナントA | Accounting | `aaaaaaaa-4444-4444-4444-000000000004` |
| `userMemberB` | テナントB | Member | `bbbbbbbb-3333-3333-3333-000000000003` |

### テナントAのレポート（作成者: `userMember`）

| フィクスチャ名 | 状態 | レポートID |
|-------------|------|----------|
| `report_draft` | `draft` | `cccccccc-0001-0001-0001-000000000001` |
| `report_submitted` | `submitted` | `cccccccc-0002-0002-0002-000000000002` |
| `report_approved` | `approved` | `cccccccc-0003-0003-0003-000000000003` |

### テナントBのレポート（作成者: `userMemberB`、クロステナント検証用）

| フィクスチャ名 | 状態 | レポートID | テナント |
|-------------|------|----------|--------|
| `report_tenant_b_draft` | `draft` | `eeeeeeee-0001-0001-0001-000000000001` | テナントB |
| `report_tenant_b_submitted` | `submitted` | `eeeeeeee-0002-0002-0002-000000000002` | テナントB |
| `report_tenant_b_approved` | `approved` | `eeeeeeee-0003-0003-0003-000000000003` | テナントB |

**注意**: テナントBのレポートIDはテナントAのレポートIDと重複しない。`eeeeeeee-` プレフィックスで識別する。

### テナントBの明細 / 添付（クロステナント検証用）

| フィクスチャ名 | 種別 | ID |
|-------------|------|-----|
| `item_tenant_b` | 経費明細 | `ffffffff-0001-0001-0001-000000000001` |
| `attachment_tenant_b` | 添付ファイル | `gggggggg-0001-0001-0001-000000000001` |

---

## §1 テナント分離マトリクス

### 方針

テナントAのユーザー（`userMember`）がテナントBのリソースに対してリクエストを送信したとき、アプリケーション層のテナントIDフィルタ（リポジトリ層の `WHERE tenant_id = ?`）と DB 層の RLS により、リソースが存在しないかのように見せる（404 を返す）。403 を返すとリソースの存在が漏洩するため、テナント境界を越えたアクセスでは必ず 404 を返す（`policies.md` SS9 参照）。

### テナント分離マトリクス（全リソース一覧）

| リソース | テスト内容 | 対応テストID |
|---------|----------|-------------|
| 経費レポート GET | テナントBのレポートIDで詳細取得 → 404 | CRS-001 |
| 経費レポート PUT | テナントBのレポートIDで更新 → 404 | CRS-002 |
| 経費レポート DELETE | テナントBのレポートIDで削除 → 404 | CRS-003 |
| 経費レポート POST (submit) | テナントBのレポートIDで提出 → 404 | CRS-004 |
| 経費明細 POST | テナントBのレポート配下への明細追加 → 404 | CRS-005 |
| 経費明細 PUT | テナントBのレポート配下の明細更新 → 404 | CRS-006 |
| 経費明細 DELETE | テナントBのレポート配下の明細削除 → 404 | CRS-007 |
| 添付ファイル POST (upload) | テナントBのレポート配下へのアップロード → 404 | CRS-008 |
| 添付ファイル GET (list) | テナントBのレポート配下の添付一覧取得 → 404 | CRS-009 |
| 添付ファイル GET (download) | テナントBの添付の署名付きURL取得（ダウンロード） → 404 | CRS-010 |
| 添付ファイル GET (preview) | テナントBの添付の署名付きURL取得（プレビュー） → 404 | CRS-010b |
| 添付ファイル DELETE | テナントBの添付削除 → 404 | CRS-011 |
| ワークフロー POST (approve) | テナントBのレポート承認 → 404 | CRS-012 |
| ワークフロー POST (reject) | テナントBのレポート却下 → 404 | CRS-013 |
| ワークフロー POST (pay) | テナントBのレポート支払完了 → 404 | CRS-014 |
| テナントメンバー GET | テナントBのメンバーがテナントAのレスポンスに混入しないこと（RLS分離） | CRS-015 |
| ダッシュボード GET | テナントBのデータがテナントAの集計値に混入しないこと | CRS-016 |

**補足: ワークフロー pending / payable 一覧（GET）について**

`GET /api/workflow/pending`（listPendingReports）と `GET /api/workflow/payable`（listPayableReports）は JWT の `tenant_id` スコープで自動フィルタされる。テナントAのユーザーがテナントBのリソースIDを URL パスに指定する操作ではないため、「テナントBのリソースが一覧に混入しないこと」の確認は CRS-016（ダッシュボード）と同様の観点で網羅される。一覧エンドポイントの個別テナント分離テストは各機能別ファイル（`workflow.md` WFL-001〜003）の範囲でカバーされているため、本ファイルでは省略する。

### DSH-018 との関係

`dashboard.md` に DSH-018（`TestGetDashboard_TenantIsolation_CountsExcludeOtherTenant`）が存在する。

**判断**: CRS-016 が正本とし、DSH-018 はダッシュボード固有の集計値正確性テスト（テナントBのデータが `my_draft_count` の数値に混入しないことを具体的な数値で検証するもの）として残す。

| ID | 役割 | 観点 |
|----|------|------|
| CRS-016（本ファイル） | 横断テナント分離の正本 | 「テナントBのデータが集計に混入しないこと」の概念確認 |
| DSH-018（dashboard.md） | ダッシュボード固有の集計正確性テスト | `my_draft_count` の数値レベルで具体的に検証 |

実装時は DSH-018 で具体的な数値検証も併せて行うこと。

---

### テストケース一覧（§1）

#### 経費レポート — テナント分離

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| CRS-001 | 統合 | handler | テナント分離 | TNT-F02, TNT-005 | authz.md#5, db_schema.md#RLS | `TestTenantIsolation_GetReport_OtherTenant_404` | `userMember`（テナントA）のトークンで `GET /api/reports/{report_tenant_b_draft.id}` を実行。テナントBの `report_tenant_b_draft` がDB上に存在する | 404 RESOURCE_NOT_FOUND。テナントBのレポートが存在しないかのように扱われる |
| CRS-002 | 統合 | handler | テナント分離 | TNT-F02, TNT-005 | authz.md#5, db_schema.md#RLS | `TestTenantIsolation_UpdateReport_OtherTenant_404` | `userMember`（テナントA）のトークンで `PUT /api/reports/{report_tenant_b_draft.id}` を実行 | 404 RESOURCE_NOT_FOUND |
| CRS-003 | 統合 | handler | テナント分離 | TNT-F02, TNT-005 | authz.md#5, db_schema.md#RLS | `TestTenantIsolation_DeleteReport_OtherTenant_404` | `userMember`（テナントA）のトークンで `DELETE /api/reports/{report_tenant_b_draft.id}` を実行 | 404 RESOURCE_NOT_FOUND |
| CRS-004 | 統合 | handler | テナント分離 | TNT-F02, TNT-005 | authz.md#5, db_schema.md#RLS | `TestTenantIsolation_SubmitReport_OtherTenant_404` | `userMember`（テナントA）のトークンで `POST /api/reports/{report_tenant_b_draft.id}/submit` を実行 | 404 RESOURCE_NOT_FOUND |

#### 経費明細 — テナント分離

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| CRS-005 | 統合 | handler | テナント分離 | TNT-F02, TNT-005 | authz.md#5, db_schema.md#RLS | `TestTenantIsolation_CreateItem_OtherTenant_404` | `userMember`（テナントA）のトークンで `POST /api/reports/{report_tenant_b_draft.id}/items` を実行 | 404 RESOURCE_NOT_FOUND |
| CRS-006 | 統合 | handler | テナント分離 | TNT-F02, TNT-005 | authz.md#5, db_schema.md#RLS | `TestTenantIsolation_UpdateItem_OtherTenant_404` | `userMember`（テナントA）のトークンで `PUT /api/reports/{report_tenant_b_draft.id}/items/{item_tenant_b.id}` を実行 | 404 RESOURCE_NOT_FOUND |
| CRS-007 | 統合 | handler | テナント分離 | TNT-F02, TNT-005 | authz.md#5, db_schema.md#RLS | `TestTenantIsolation_DeleteItem_OtherTenant_404` | `userMember`（テナントA）のトークンで `DELETE /api/reports/{report_tenant_b_draft.id}/items/{item_tenant_b.id}` を実行 | 404 RESOURCE_NOT_FOUND |

#### 添付ファイル — テナント分離

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| CRS-008 | 統合 | handler | テナント分離 | TNT-F02, TNT-005 | authz.md#5, db_schema.md#RLS | `TestTenantIsolation_UploadAttachment_OtherTenant_404` | `userMember`（テナントA）のトークンで `POST /api/reports/{report_tenant_b_draft.id}/items/{item_tenant_b.id}/attachments` を実行 | 404 RESOURCE_NOT_FOUND |
| CRS-009 | 統合 | handler | テナント分離 | TNT-F02, TNT-005 | authz.md#5, db_schema.md#RLS | `TestTenantIsolation_ListAttachments_OtherTenant_404` | `userMember`（テナントA）のトークンで `GET /api/reports/{report_tenant_b_draft.id}/items/{item_tenant_b.id}/attachments` を実行 | 404 RESOURCE_NOT_FOUND |
| CRS-010 | 統合 | handler | テナント分離 | TNT-F02, TNT-005 | authz.md#5, db_schema.md#RLS | `TestTenantIsolation_GetAttachmentDownload_OtherTenant_404` | `userMember`（テナントA）のトークンで `GET /api/reports/{report_tenant_b_draft.id}/items/{item_tenant_b.id}/attachments/{attachment_tenant_b.id}/download` を実行 | 404 RESOURCE_NOT_FOUND |
| CRS-010b | 統合 | handler | テナント分離 | TNT-F02, TNT-005 | authz.md#5, db_schema.md#RLS | `TestTenantIsolation_GetAttachmentPreview_OtherTenant_404` | `userMember`（テナントA）のトークンで `GET /api/reports/{report_tenant_b_draft.id}/items/{item_tenant_b.id}/attachments/{attachment_tenant_b.id}/preview` を実行 | 404 RESOURCE_NOT_FOUND |
| CRS-011 | 統合 | handler | テナント分離 | TNT-F02, TNT-005 | authz.md#5, db_schema.md#RLS | `TestTenantIsolation_DeleteAttachment_OtherTenant_404` | `userMember`（テナントA）のトークンで `DELETE /api/reports/{report_tenant_b_draft.id}/items/{item_tenant_b.id}/attachments/{attachment_tenant_b.id}` を実行 | 404 RESOURCE_NOT_FOUND |

#### ワークフロー — テナント分離

テナントAの Approver がテナントBの `submitted` レポートを操作しようとする、テナントAの Accounting がテナントBの `approved` レポートを操作しようとするケースを検証する。

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| CRS-012 | 統合 | handler | テナント分離 | TNT-F02, TNT-005 | authz.md#5, db_schema.md#RLS | `TestTenantIsolation_ApproveReport_OtherTenant_404` | `userApprover`（テナントA）のトークンで `POST /api/workflow/{report_tenant_b_submitted.id}/approve` を実行 | 404 RESOURCE_NOT_FOUND |
| CRS-013 | 統合 | handler | テナント分離 | TNT-F02, TNT-005 | authz.md#5, db_schema.md#RLS | `TestTenantIsolation_RejectReport_OtherTenant_404` | `userApprover`（テナントA）のトークンで `POST /api/workflow/{report_tenant_b_submitted.id}/reject` を実行（却下理由あり） | 404 RESOURCE_NOT_FOUND |
| CRS-014 | 統合 | handler | テナント分離 | TNT-F02, TNT-005 | authz.md#5, db_schema.md#RLS | `TestTenantIsolation_PayReport_OtherTenant_404` | `userAccounting`（テナントA）のトークンで `POST /api/workflow/{report_tenant_b_approved.id}/pay` を実行 | 404 RESOURCE_NOT_FOUND |

#### テナントメンバー — テナント分離

`GET /api/tenant/members` は JWT の `tenant_id` スコープでフィルタされるため、テナントBのメンバーはレスポンスに含まれない（テナントBを直接指定するURL操作ではなく、RLS による分離）。

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| CRS-015 | 統合 | handler | テナント分離 | TNT-F02, TNT-005 | authz.md#5, db_schema.md#RLS | `TestTenantIsolation_ListTenantMembers_ExcludesOtherTenant` | `userAdmin`（テナントA）のトークンで `GET /api/tenant/members` を実行。テナントBにも `userMemberB` が登録済み | 200 OK。レスポンスの `data` にテナントAのメンバーのみ含まれ、`userMemberB`（`bbbbbbbb-3333-3333-3333-000000000003`）が含まれないこと |

**備考**: CRS-015 がテナント分離テストの正本。TNT-008（tenant.md）は同一テナント内のメンバー一覧のデータ正確性テストとして残す（テナント越境アクセステストではない）。実装時は CRS-015 でテナント越境を検証し、TNT-008 でレスポンスのフィルタリング正確性を検証する。両方実装する。

#### ダッシュボード — テナント分離

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| CRS-016 | 統合 | handler | テナント分離 | TNT-F02, TNT-005 | authz.md#5, db_schema.md#RLS | `TestTenantIsolation_Dashboard_ExcludesOtherTenantData` | `userMember`（テナントA）のトークンで `GET /api/dashboard` を実行。テナントBに `report_tenant_b_draft` が1件存在。テナントAには `userMember` の draft が1件のみ | 200 OK。集計値にテナントBのデータが含まれないこと（`data.my_draft_count == 1`）。DSH-018 と対応（CRS-016 が横断観点の正本） |

---

## §2 RBAC マトリクス

### 方針

各機能別ファイル（`auth.md`、`reports.md`、`items.md`、`attachments.md`、`workflow.md`、`dashboard.md`、`tenant.md`）には各エンドポイントの個別 RBAC テストが記載されている。本セクションでは全エンドポイント × 全ロールの「整合性確認マトリクス」を一覧化し、設計漏れ・矛盾の横断レビューを可能にする。

**凡例**:
- `○` = 許可（200 系）
- `×` = 禁止（403 FORBIDDEN）
- `※` = 所有者のみ許可（所有者: 200、非所有者: 403）
- `△` = レポート閲覧権限に準ずる（authz.md §10: 所有 + submitted + 自分が approved_by/rejected_by）
- `-` = 対象外（未認証エンドポイント等）

### RBACマトリクス（全30エンドポイント、health 除く）

| エンドポイント | operationId | Admin | Approver | Member | Accounting | 備考 |
|--------------|------------|-------|----------|--------|------------|------|
| POST /api/auth/signup | signup | - | - | - | - | 未認証エンドポイント（全員許可） |
| POST /api/auth/login | login | - | - | - | - | 未認証エンドポイント（全員許可） |
| POST /api/auth/refresh | refreshToken | - | - | - | - | 未認証エンドポイント（全員許可） |
| POST /api/auth/logout | logout | ○ | ○ | ○ | ○ | 認証済み全ロール許可（refresh_token で認証） |
| GET /api/auth/me | getMe | ○ | ○ | ○ | ○ | 認証済み全ロール許可 |
| POST /api/auth/password-reset | requestPasswordReset | - | - | - | - | 未認証エンドポイント（全員許可） |
| PUT /api/auth/password-reset/{token} | executePasswordReset | - | - | - | - | 未認証エンドポイント（全員許可） |
| GET /api/dashboard | getDashboard | ○ | ○ | ○ | ○ | 全ロール許可（ロールごとに返却フィールドが異なる） |
| GET /api/categories | listCategories | ○ | ○ | ○ | ○ | 全ロール許可（グローバルマスタ） |
| GET /api/reports | listMyReports | ○ | ○ | ○ | ○ | 全ロール許可（自分のレポートのみ返却） |
| POST /api/reports | createReport | ○ | ○ | ○ | ○ | 全ロール許可（RBC-010） |
| GET /api/reports/all | listAllReports | ○ | × | × | ○ | Admin: 管理目的、Accounting: 支払管理目的。Member・Approver は 403 |
| GET /api/reports/{id} | getReport | ○ | ※ | ※ | ○ | Admin: 全件閲覧。Approver: 自分 + submitted + 承認済み/却下済みのもの。Member: 自分のみ。Accounting: 全件閲覧 |
| PUT /api/reports/{id} | updateReport | ※ | ※ | ※ | ※ | 所有者のみ（draft のみ） |
| DELETE /api/reports/{id} | deleteReport | ※ | ※ | ※ | ※ | 所有者のみ（draft のみ） |
| POST /api/reports/{id}/submit | submitReport | ※ | ※ | ※ | ※ | 所有者のみ（draft のみ） |
| POST /api/reports/{id}/items | createItem | ※ | ※ | ※ | ※ | 所有者のみ（draft のみ） |
| PUT /api/reports/{id}/items/{itemId} | updateItem | ※ | ※ | ※ | ※ | 所有者のみ（draft のみ） |
| DELETE /api/reports/{id}/items/{itemId} | deleteItem | ※ | ※ | ※ | ※ | 所有者のみ（draft のみ） |
| POST /api/reports/{id}/items/{itemId}/attachments | uploadAttachment | ※ | ※ | ※ | ※ | 所有者のみ（draft のみ）。レート制限 10 req/min/user |
| GET /api/reports/{id}/items/{itemId}/attachments | listAttachments | ○ | △ | ※ | ○ | authz.md §10: 所有 + submitted + 自分が approved_by/rejected_by |
| GET /api/reports/{id}/items/{itemId}/attachments/{attId}/download | getAttachmentDownload | ○ | △ | ※ | ○ | authz.md §10: 所有 + submitted + 自分が approved_by/rejected_by。URL 発行前に認可チェック |
| GET /api/reports/{id}/items/{itemId}/attachments/{attId}/preview | getAttachmentPreview | ○ | △ | ※ | ○ | getAttachmentDownload と同一の認可ルール。Content-Disposition: inline |
| DELETE /api/reports/{id}/items/{itemId}/attachments/{attId} | deleteAttachment | ※ | ※ | ※ | ※ | 所有者のみ（draft のみ） |
| GET /api/workflow/pending | listPendingReports | × | ○ | × | × | Approver 専用。他ロールは 403 |
| POST /api/workflow/{id}/approve | approveReport | × | ○ | × | × | Approver 専用。自己承認禁止（RBC-016）。他ロールは 403 |
| POST /api/workflow/{id}/reject | rejectReport | × | ○ | × | × | Approver 専用。自己却下禁止（RBC-016）。他ロールは 403 |
| GET /api/workflow/payable | listPayableReports | × | × | × | ○ | Accounting 専用。他ロールは 403 |
| POST /api/workflow/{id}/pay | markReportAsPaid | × | × | × | ○ | Accounting 専用。自己支払禁止（RBC-012）。他ロールは 403 |
| GET /api/tenant | getTenant | ○ | × | × | × | Admin 専用。他ロールは 403 |
| GET /api/tenant/members | listTenantMembers | ○ | × | × | ○ | Admin・Accounting 許可。Approver・Member は 403 |

### 整合性チェック観点

以下の点について機能別ファイルとの整合性を確認すること:

1. `listAllReports`（GET /api/reports/all）: Member・Approver が 403 であることを `reports.md` で検証済みか
2. `listPendingReports` / `approveReport` / `rejectReport`: Member・Admin・Accounting が 403 であることを `workflow.md` で検証済みか（WFL-006〜WFL-008, WFL-017〜WFL-019, WFL-027〜WFL-029 参照）
3. `listPayableReports` / `markReportAsPaid`: Member・Approver・Admin が 403 であることを `workflow.md` で検証済みか
4. `getTenant`: Approver・Member・Accounting が 403 であることを `tenant.md` で検証済みか（TNT-003〜TNT-005 参照）
5. `listTenantMembers`: Approver・Member が 403 であることを `tenant.md` で検証済みか（TNT-010〜TNT-011 参照）

### 整合性確認テストケース（横断マトリクス検証）

下記テストケースは RBAC マトリクスの「重点検証ポイント」（`test_strategy.md` §11.2）のうち、機能別ファイルでカバーされていない組み合わせを補完する。機能別ファイルに既存テストがある場合は重複させず、テストIDのクロス参照のみ行う。

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| CRS-021 | 統合 | handler | 認可 | RPT-F07, RBC-013 | openapi.yaml#listAllReports, authz.md#4.4 | `TestRBAC_ListAllReports_Member_Forbidden` | `userMember` のトークンで `GET /api/reports/all` | 403 FORBIDDEN。`reports.md` での検証を補完（マトリクス確認用） |
| CRS-022 | 統合 | handler | 認可 | RPT-F07, RBC-013 | openapi.yaml#listAllReports, authz.md#4.4 | `TestRBAC_ListAllReports_Approver_Forbidden` | `userApprover` のトークンで `GET /api/reports/all` | 403 FORBIDDEN |
| CRS-023 | 統合 | handler | 認可 | RPT-F07, RBC-013 | openapi.yaml#listAllReports, authz.md#4.4 | `TestRBAC_ListAllReports_Admin_OK` | `userAdmin` のトークンで `GET /api/reports/all` | 200 OK |
| CRS-024 | 統合 | handler | 認可 | RPT-F07, RBC-013 | openapi.yaml#listAllReports, authz.md#4.4 | `TestRBAC_ListAllReports_Accounting_OK` | `userAccounting` のトークンで `GET /api/reports/all` | 200 OK |
| CRS-025 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#listPendingReports, authz.md#3 | `TestRBAC_ListPendingReports_Member_Forbidden` | `userMember` のトークンで `GET /api/workflow/pending` | 403 FORBIDDEN（WFL-006 とクロス参照） |
| CRS-026 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#listPendingReports, authz.md#3 | `TestRBAC_ListPendingReports_Admin_Forbidden` | `userAdmin` のトークンで `GET /api/workflow/pending` | 403 FORBIDDEN（WFL-007 とクロス参照） |
| CRS-027 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#listPendingReports, authz.md#3 | `TestRBAC_ListPendingReports_Accounting_Forbidden` | `userAccounting` のトークンで `GET /api/workflow/pending` | 403 FORBIDDEN（WFL-008 とクロス参照） |
| CRS-028 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#approveReport, authz.md#3 | `TestRBAC_ApproveReport_Member_Forbidden` | `userMember` のトークンで `POST /api/workflow/{report_submitted.id}/approve` | 403 FORBIDDEN（WFL-017 とクロス参照） |
| CRS-029 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#approveReport, authz.md#3 | `TestRBAC_ApproveReport_Admin_Forbidden` | `userAdmin` のトークンで `POST /api/workflow/{report_submitted.id}/approve` | 403 FORBIDDEN（WFL-018 とクロス参照） |
| CRS-030 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#approveReport, authz.md#3 | `TestRBAC_ApproveReport_Accounting_Forbidden` | `userAccounting` のトークンで `POST /api/workflow/{report_submitted.id}/approve` | 403 FORBIDDEN（WFL-019 とクロス参照） |
| CRS-031 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#rejectReport, authz.md#3 | `TestRBAC_RejectReport_Member_Forbidden` | `userMember` のトークンで `POST /api/workflow/{report_submitted.id}/reject` | 403 FORBIDDEN（WFL-027 とクロス参照） |
| CRS-032 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#rejectReport, authz.md#3 | `TestRBAC_RejectReport_Admin_Forbidden` | `userAdmin` のトークンで `POST /api/workflow/{report_submitted.id}/reject` | 403 FORBIDDEN（WFL-028 とクロス参照） |
| CRS-033 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#rejectReport, authz.md#3 | `TestRBAC_RejectReport_Accounting_Forbidden` | `userAccounting` のトークンで `POST /api/workflow/{report_submitted.id}/reject` | 403 FORBIDDEN（WFL-029 とクロス参照） |
| CRS-034 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#listPayableReports, authz.md#3 | `TestRBAC_ListPayableReports_Member_Forbidden` | `userMember` のトークンで `GET /api/workflow/payable` | 403 FORBIDDEN |
| CRS-035 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#listPayableReports, authz.md#3 | `TestRBAC_ListPayableReports_Approver_Forbidden` | `userApprover` のトークンで `GET /api/workflow/payable` | 403 FORBIDDEN |
| CRS-036 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#listPayableReports, authz.md#3 | `TestRBAC_ListPayableReports_Admin_Forbidden` | `userAdmin` のトークンで `GET /api/workflow/payable` | 403 FORBIDDEN |
| CRS-037 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#markReportAsPaid, authz.md#3 | `TestRBAC_MarkReportAsPaid_Member_Forbidden` | `userMember` のトークンで `POST /api/workflow/{report_approved.id}/pay` | 403 FORBIDDEN |
| CRS-038 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#markReportAsPaid, authz.md#3 | `TestRBAC_MarkReportAsPaid_Approver_Forbidden` | `userApprover` のトークンで `POST /api/workflow/{report_approved.id}/pay` | 403 FORBIDDEN |
| CRS-039 | 統合 | handler | 認可 | RBAC-F01, RBC-001 | openapi.yaml#markReportAsPaid, authz.md#3 | `TestRBAC_MarkReportAsPaid_Admin_Forbidden` | `userAdmin` のトークンで `POST /api/workflow/{report_approved.id}/pay` | 403 FORBIDDEN |
| CRS-040 | 統合 | handler | 認可 | ADM-F01, RBC-001 | openapi.yaml#getTenant, authz.md#3 | `TestRBAC_GetTenant_Approver_Forbidden` | `userApprover` のトークンで `GET /api/tenant` | 403 FORBIDDEN（TNT-003 とクロス参照） |
| CRS-041 | 統合 | handler | 認可 | ADM-F01, RBC-001 | openapi.yaml#getTenant, authz.md#3 | `TestRBAC_GetTenant_Member_Forbidden` | `userMember` のトークンで `GET /api/tenant` | 403 FORBIDDEN（TNT-004 とクロス参照） |
| CRS-042 | 統合 | handler | 認可 | ADM-F01, RBC-001 | openapi.yaml#getTenant, authz.md#3 | `TestRBAC_GetTenant_Accounting_Forbidden` | `userAccounting` のトークンで `GET /api/tenant` | 403 FORBIDDEN（TNT-005 とクロス参照） |
| CRS-043 | 統合 | handler | 認可 | ADM-F01, RBC-001 | openapi.yaml#listTenantMembers, authz.md#3 | `TestRBAC_ListTenantMembers_Approver_Forbidden` | `userApprover` のトークンで `GET /api/tenant/members` | 403 FORBIDDEN（TNT-010 とクロス参照） |
| CRS-044 | 統合 | handler | 認可 | ADM-F01, RBC-001 | openapi.yaml#listTenantMembers, authz.md#3 | `TestRBAC_ListTenantMembers_Member_Forbidden` | `userMember` のトークンで `GET /api/tenant/members` | 403 FORBIDDEN（TNT-011 とクロス参照） |
| CRS-045 | 統合 | handler | 認可 | ADM-F01 | openapi.yaml#getTenant, authz.md#3 | `TestRBAC_GetTenant_Admin_OK` | `userAdmin` のトークンで `GET /api/tenant` | 200 OK（TNT-001 とクロス参照） |
| CRS-046 | 統合 | handler | 認可 | ADM-F01 | openapi.yaml#listTenantMembers, authz.md#3 | `TestRBAC_ListTenantMembers_Admin_OK` | `userAdmin` のトークンで `GET /api/tenant/members` | 200 OK（TNT-006 とクロス参照） |
| CRS-047 | 統合 | handler | 認可 | ADM-F01 | openapi.yaml#listTenantMembers, authz.md#3 | `TestRBAC_ListTenantMembers_Accounting_OK` | `userAccounting` のトークンで `GET /api/tenant/members` | 200 OK（TNT-007 とクロス参照） |

**内部統制ルールの整合性確認（RBC-016 / RBC-012）**:

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| CRS-048 | 統合 | handler | 認可 | RBC-016 | openapi.yaml#approveReport, authz.md#4.2 | `TestRBAC_SelfApproval_Forbidden` | `userApprover` が自分で作成した submitted レポートに対して `POST /api/workflow/{id}/approve` を実行（RBC-016 自己承認禁止） | 403 SELF_APPROVAL_NOT_ALLOWED。自己承認禁止（`workflow.md` WFL-020 とクロス参照） |
| CRS-049 | 統合 | handler | 認可 | RBC-016 | openapi.yaml#rejectReport, authz.md#4.2 | `TestRBAC_SelfRejection_Forbidden` | `userApprover` が自分で作成した submitted レポートに対して `POST /api/workflow/{id}/reject` を実行（RBC-016 自己却下禁止） | 403 SELF_APPROVAL_NOT_ALLOWED。自己却下禁止（`workflow.md` WFL-030 とクロス参照） |
| CRS-050 | 統合 | handler | 認可 | RBC-012 | openapi.yaml#markReportAsPaid, authz.md#4.3 | `TestRBAC_SelfPayment_Forbidden` | `userAccounting` が自分で作成した approved レポートに対して `POST /api/workflow/{id}/pay` を実行（RBC-012 自己支払禁止） | 403 SELF_PAYMENT_NOT_ALLOWED。自己支払処理禁止（`workflow.md` WFL-040 とクロス参照） |

**Admin の二面性確認（RBC-014）**:

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| CRS-051 | 統合 | handler | 認可 | RBC-014 | openapi.yaml#updateReport, authz.md#4.4 | `TestRBAC_Admin_CanEditOwnReport` | `userAdmin` が自分で作成した draft レポートを `PUT /api/reports/{id}` で更新 | 200 OK。Admin は自分のレポートを申請者として操作可能（RBC-014） |
| CRS-052 | 統合 | handler | 認可 | RBC-014 | openapi.yaml#updateReport, authz.md#4.4 | `TestRBAC_Admin_CannotEditOtherReport` | `userAdmin` が `userMember` の draft レポートを `PUT /api/reports/{id}` で更新しようとする | 403 FORBIDDEN。Admin は他者のレポートを編集不可（RBC-014） |
| CRS-053 | 統合 | handler | 認可 | RBC-013 | openapi.yaml#getReport, authz.md#4.4 | `TestRBAC_Admin_CanViewAllReports` | `userAdmin` のトークンで `GET /api/reports/{userMember_report.id}` を実行 | 200 OK。Admin は全レポートを閲覧可能（RBC-013） |
| CRS-054 | 統合 | handler | 認可 | RBC-013 | openapi.yaml#approveReport, authz.md#3 | `TestRBAC_Admin_CannotApprove` | `userAdmin` のトークンで `POST /api/workflow/{report_submitted.id}/approve` を実行 | 403 FORBIDDEN。Admin は承認不可（RBAC マトリクス §3.3）。CRS-029 とクロス参照 |

---

## §3 E2E シナリオ

### 方針

Playwright を使用した主要業務フローのテストシナリオ。フロントエンド（ブラウザ操作）から API、DB の状態変化まで一貫して検証する（`test_strategy.md` §10）。

### テスト環境

- ブラウザ: Chromium（MVP では Chromium のみ）
- テスト環境: E2E 専用テスト環境（`expense-saas/e2e/` 配下）
- テストアカウント: `test_strategy.md` §4.2 の標準フィクスチャを使用

---

### フロー1: 申請→承認→支払完了（正常フロー）

#### テストケース

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| CRS-055 | E2E | - | 正常系 | WFL-F01, WFL-F02, WFL-F03 | openapi.yaml, state_machine.md | `test_flow1_submit_approve_pay` | フロー1 全体（CRS-056〜CRS-068 のステップを一つの E2E テストとして実行） | 全ステップが正常完了し、最終的にレポートの状態が `paid` になること |

#### ステップ詳細（CRS-055 の内部ステップ）

| ステップ | 操作 | 期待結果 |
|---------|------|---------|
| 1 | `test-member@example.com` でログイン | ダッシュボード画面に遷移。HTTP 200。JWT に `role: "member"` が含まれること |
| 2 | 「レポート新規作成」ボタンをクリックしてタイトル・対象期間を入力し送信 | `POST /api/reports` → 201。レポートが `draft` 状態で作成される。URLが `/reports/{id}` に遷移 |
| 3 | 経費明細フォームから金額・カテゴリ（交通費）・支出日・摘要を入力して追加 | `POST /api/reports/{id}/items` → 201。明細が追加され、合計金額が更新される |
| 4 | 「提出する」ボタンをクリック | `POST /api/reports/{id}/submit` → 200。レポートの状態が `submitted` に変更。画面にステータス「提出済み」が表示される |
| 5 | ログアウトし、`test-approver@example.com` でログイン | ダッシュボードに `pending_approval_count >= 1` が表示される |
| 6 | 承認待ち一覧ページ（`/workflow/pending`）に遷移し、対象レポートを確認 | `GET /api/workflow/pending` → 200。一覧にステップ2で作成したレポートが表示される |
| 7 | 対象レポートの「承認」ボタンをクリック（コメントは任意で入力） | `POST /api/workflow/{id}/approve` → 200。レポートの状態が `approved` に変更。画面にステータス「承認済み」が表示される |
| 8 | ログアウトし、`test-accounting@example.com` でログイン | ダッシュボードに `pending_payment_count >= 1` が表示される |
| 9 | 支払待ち一覧ページ（`/workflow/payable`）に遷移し、対象レポートを確認 | `GET /api/workflow/payable` → 200。一覧にステップ2で作成したレポートが表示される |
| 10 | 対象レポートの「支払完了」ボタンをクリック | `POST /api/workflow/{id}/pay` → 200。レポートの状態が `paid` に変更。画面にステータス「支払済み」が表示される |

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| CRS-056 | E2E | - | 正常系 | AUTH-F02 | openapi.yaml#login | `test_flow1_step1_login_as_member` | `test-member@example.com` / `TestPass1!` でログイン | ダッシュボード画面表示。JWT 取得成功 |
| CRS-057 | E2E | - | 正常系 | RPT-F01 | openapi.yaml#createReport | `test_flow1_step2_create_report` | タイトル: 「2026年3月経費」、対象期間: 2026-03-01〜2026-03-31 | `POST /api/reports` → 201。`status: "draft"` |
| CRS-058 | E2E | - | 正常系 | ITM-F01 | openapi.yaml#createItem | `test_flow1_step3_add_item` | 金額: 1000、カテゴリ: 交通費、支出日: 2026-03-10、摘要: 「電車代」 | `POST /api/reports/{id}/items` → 201。レポートの `total_amount == 1000` |
| CRS-059 | E2E | - | 正常系 | RPT-F06, WFL-010 | openapi.yaml#submitReport, state_machine.md#T1 | `test_flow1_step4_submit_report` | 「提出する」ボタンクリック | `POST /api/reports/{id}/submit` → 200。`status: "submitted"` |
| CRS-060 | E2E | - | 正常系 | AUTH-F02, DASH-002 | openapi.yaml#login, openapi.yaml#getDashboard | `test_flow1_step5_login_as_approver` | ログアウト後、`test-approver@example.com` でログイン | ダッシュボードに `pending_approval_count >= 1` |
| CRS-061 | E2E | - | 正常系 | WFL-F04 | openapi.yaml#listPendingReports | `test_flow1_step6_view_pending_list` | `/workflow/pending` に遷移 | `GET /api/workflow/pending` → 200。提出済みレポートが一覧に表示される |
| CRS-062 | E2E | - | 正常系 | WFL-F01, WFL-011 | openapi.yaml#approveReport, state_machine.md#T2 | `test_flow1_step7_approve_report` | 「承認」ボタンクリック | `POST /api/workflow/{id}/approve` → 200。`status: "approved"` |
| CRS-063 | E2E | - | 正常系 | AUTH-F02, DASH-003 | openapi.yaml#login, openapi.yaml#getDashboard | `test_flow1_step8_login_as_accounting` | ログアウト後、`test-accounting@example.com` でログイン | ダッシュボードに `pending_payment_count >= 1` |
| CRS-064 | E2E | - | 正常系 | WFL-F05 | openapi.yaml#listPayableReports | `test_flow1_step9_view_payable_list` | `/workflow/payable` に遷移 | `GET /api/workflow/payable` → 200。承認済みレポートが一覧に表示される |
| CRS-065 | E2E | - | 正常系 | WFL-F03, WFL-013 | openapi.yaml#markReportAsPaid, state_machine.md#T4 | `test_flow1_step10_pay_report` | 「支払完了」ボタンクリック | `POST /api/workflow/{id}/pay` → 200。`status: "paid"` |

---

### フロー2: 却下→再申請フロー

#### テストケース

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| CRS-066 | E2E | - | 正常系 | WFL-F02, RPT-015 | openapi.yaml, state_machine.md | `test_flow2_reject_and_resubmit` | フロー2 全体（CRS-067〜CRS-074 のステップを一つの E2E テストとして実行） | 全ステップが正常完了し、再申請レポートが submitted 状態になること |

#### ステップ詳細（CRS-066 の内部ステップ）

| ステップ | 操作 | 期待結果 |
|---------|------|---------|
| 1 | `test-member@example.com` でログイン | ダッシュボード画面表示 |
| 2 | レポートを新規作成し、明細を追加して提出 | `POST /api/reports` → 201。`POST /api/reports/{id}/items` → 201。`POST /api/reports/{id}/submit` → 200。`status: "submitted"` |
| 3 | ログアウトし、`test-approver@example.com` でログイン | 承認待ち一覧に遷移 |
| 4 | 対象レポートの「却下」ボタンをクリックし、却下理由「金額の証憑が不足しています」を入力して送信 | `POST /api/workflow/{id}/reject` → 200。`status: "rejected"` |
| 5 | ログアウトし、`test-member@example.com` でログイン | ダッシュボードに却下レポートが表示される |
| 6 | 却下レポートの詳細画面を開き、却下理由が表示されていることを確認 | `GET /api/reports/{id}` → 200。`status: "rejected"`、`rejection_reason: "金額の証憑が不足しています"` が表示される |
| 7 | 「再申請する」ボタンをクリック（新規 draft レポートが作成される） | `POST /api/reports` に `reference_report_id` が設定される → 201。新規 draft レポートが作成される。元レポートの明細がコピーされていることを確認 |
| 8 | 再申請レポートの内容を確認し、提出 | `POST /api/reports/{new_id}/submit` → 200。`status: "submitted"` |

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| CRS-067 | E2E | - | 正常系 | RPT-F01, RPT-F06 | openapi.yaml#createReport, openapi.yaml#submitReport | `test_flow2_step1_login_and_submit` | Member でログイン → レポート作成 → 明細追加 → 提出 | `status: "submitted"` |
| CRS-068 | E2E | - | 正常系 | WFL-F02, WFL-012 | openapi.yaml#rejectReport, state_machine.md#T3 | `test_flow2_step2_login_as_approver_and_reject` | Approver でログイン → 却下理由入力 → 却下 | `POST /api/workflow/{id}/reject` → 200。`status: "rejected"` |
| CRS-069 | E2E | - | 正常系 | RPT-F03 | openapi.yaml#getReport | `test_flow2_step3_login_as_member_and_view_rejection` | Member でログイン → 却下レポート詳細確認 | 却下理由が画面に表示される |
| CRS-070 | E2E | - | 正常系 | RPT-015, RPT-016 | openapi.yaml#createReport, db_schema.md#expense_reports.reference_report_id | `test_flow2_step4_resubmit` | 「再申請する」ボタンクリック | 新規 draft レポートが `reference_report_id` 付きで作成される。元レポートの明細がコピーされている |
| CRS-071 | E2E | - | 正常系 | RPT-015 | openapi.yaml#createReport | `test_flow2_step5_confirm_copied_items` | 再申請レポートの明細一覧を確認 | 元レポートの明細（金額・カテゴリ・摘要）がコピーされていること。添付ファイルはコピーされていないこと（openapi.yaml の createReport 説明参照） |
| CRS-072 | E2E | - | 正常系 | RPT-F06, WFL-010 | openapi.yaml#submitReport, state_machine.md#T1 | `test_flow2_step6_submit_resubmit` | 再申請レポートを提出 | `POST /api/reports/{new_id}/submit` → 200。`status: "submitted"` |

---

### フロー3（オプション）: Admin のテナント管理フロー

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| CRS-073 | E2E | - | 正常系 | DASH-F01, DASH-004 | openapi.yaml#getDashboard | `test_flow3_admin_dashboard` | `test-admin@example.com` でログイン | ダッシュボードに Admin 専用フィールド（`tenant_draft_count`、`tenant_submitted_count`、`tenant_member_count` 等）が表示される |
| CRS-074 | E2E | - | 正常系 | RPT-F07 | openapi.yaml#listAllReports | `test_flow3_admin_list_all_reports` | Admin でログイン → `/reports/all` に遷移 | `GET /api/reports/all` → 200。テナント内の全ユーザーのレポートが一覧表示される |
| CRS-075 | E2E | - | 正常系 | ADM-F01 | openapi.yaml#getTenant | `test_flow3_admin_view_tenant_info` | Admin でログイン → テナント設定画面に遷移 | `GET /api/tenant` → 200。テナント名・作成日等が表示される |

---

## §4 非機能テスト（レート制限・レスポンスタイム）

### レート制限テスト

`security.md` §4.1 に定義された4種類のレート制限を検証する（`test_strategy.md` §2.3 参照）。

**対象制限値**:

| 対象 | 制限値 | キー |
|------|--------|------|
| 認証済みリクエスト（全般） | 100 req/min | user_id |
| 未認証リクエスト | 20 req/min | IP アドレス |
| ログイン試行（`POST /api/auth/login`） | 5 req/min | IP アドレス |
| ファイルアップロード（`POST .../attachments`） | 10 req/min | user_id |

**テスト方針**: 制限値を1回超えるリクエストを連続送信し、`429 Too Many Requests` と `Retry-After` ヘッダーが返ることを確認する。レート制限ウィンドウ（1分）をまたがないよう、テスト後にウィンドウリセットを待機するか、テスト環境でのウィンドウ設定を短くする。

#### テストケース

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| CRS-076 | 統合 | handler | 性能 | NFR-SEC-005 | security.md#4.1 | `TestRateLimit_AuthenticatedRequests_ExceedsLimit_429` | 認証済みユーザー制限（`RateLimitByUser`、本番値 100 req/min/user_id）の検証。`userMember` のトークンで `GET /api/dashboard` を `authLimit+1` 回連続送信（テストでは authLimit=3、他制限は 100 に設定して干渉排除） | `authLimit` 回目までは 200 OK。`authLimit+1` 回目（4 回目）で `429 Too Many Requests` が返る。レスポンスヘッダーに `Retry-After` が含まれる |
| CRS-077 | 統合 | handler | 性能 | NFR-SEC-005 | security.md#4.1 | `TestRateLimit_Unauthenticated_Login_ExceedsLimit_429` | グローバル未認証 IP 制限（global `RateLimitByIP(unauthLimit)`、本番値 20 req/min/IP）の検証。経路分離のため `/health` を使用し（ログイン専用制限の二重適用を回避）、`unauthLimit+1` 回連続送信（テストでは unauthLimit=5、loginLimit=100 に設定して干渉排除） | `unauthLimit` 回目までは 200 OK。`unauthLimit+1` 回目（6 回目）で `429 Too Many Requests` が返る。`Retry-After` ヘッダーが含まれる |
| CRS-078 | 統合 | handler | 性能 | NFR-SEC-005 | security.md#4.1 | `TestRateLimit_LoginAttempts_ExceedsLimit_429` | ログイン専用 IP 制限（`/api/auth/login` への追加 `RateLimitByIP(loginLimit)`、本番値 5 req/min/IP）の検証。同一 IP から `POST /api/auth/login`（存在しないメール）を `loginLimit+1` 回連続送信（テストでは loginLimit=3、unauthLimit=100 に設定して干渉排除） | `loginLimit` 回目までは 401 INVALID_CREDENTIALS。`loginLimit+1` 回目（4 回目）で `429 Too Many Requests` が返る。`Retry-After` ヘッダーが含まれる |
| CRS-079 | 統合 | handler | 性能 | NFR-SEC-005 | security.md#4.1 | `TestRateLimit_FileUpload_ExceedsLimit_429` | ファイルアップロード専用ユーザー制限（`POST .../attachments` への追加 `RateLimitByUser(uploadLimit)`、本番値 10 req/min/user_id）の検証。`userMember` のトークンで `POST .../attachments` を `uploadLimit+1` 回連続送信（テストでは uploadLimit=3、authLimit/unauthLimit/loginLimit=100 に設定して干渉排除） | `uploadLimit` 回目までは 201 Created（または 422 バリデーションエラー）。`uploadLimit+1` 回目（4 回目）で `429 Too Many Requests` が返る。`Retry-After` ヘッダーが含まれる |
| CRS-080 | 統合 | handler | 性能 | NFR-SEC-005 | security.md#4.1, security.md#8.2 | `TestRateLimit_ResponseBody_ContainsRetryAfter` | 認証済みユーザー制限超過後のレスポンスボディ確認。`userMember` のトークンで `GET /api/dashboard` を `authLimit+1` 回送信（テストでは authLimit=2、他制限は 100 に設定して干渉排除） | `authLimit` 回目までは 200 OK、`authLimit+1` 回目で 429。レスポンスボディは `{"error": {"code": "RATE_LIMIT_EXCEEDED", "message": "Too many requests. Please try again later."}}` 形式であること（`security.md` §8.2 準拠）。`Retry-After` ヘッダーが含まれる |
| CRS-081 | 統合 | handler | 性能 | NFR-SEC-005 | security.md#4.1, security.md#4.3 | `TestRateLimit_RateLimitHeaders_Present` | `userMember` のトークンで `GET /api/dashboard` を制限内で 1 回実行 | 200 OK が返り、レスポンスヘッダーに `X-RateLimit-Limit`、`X-RateLimit-Remaining`、`X-RateLimit-Reset` が含まれること（`security.md` §4.3 準拠） |
| CRS-082 | 統合 | handler | 性能 | NFR-SEC-005 | security.md#4.1 | `TestRateLimit_DifferentUsers_IndependentLimits` | 認証済みユーザー制限（`RateLimitByUser`）の user_id 別独立性検証。`userMember`（テナントA）が `authLimit+1` 回送信して制限超過させ、同じウィンドウ内で `userAccounting`（テナントA）は `authLimit-1` 回しか送信しない（テストでは authLimit=5、unauthLimit=100 に設定して IP 制限干渉排除） | `userMember` は `authLimit+1` 回目で 429 Too Many Requests が返る。`userAccounting` は `authLimit-1` 回しか送信していないため 200 OK が返り、`userMember` のカウンターに影響されないこと（user_id ベースのレート制限の独立性） |

---

### レスポンスタイムテスト

`test_strategy.md` §2.3 に定義された非機能目標値を検証する軽量スモークテスト。

**期待値** (`requirements.md` §4.1 より):

| 指標 | 目標値 |
|------|--------|
| API レスポンスタイム（p95） | 500ms 以下（一覧取得を含む） |
| ファイルアップロード（5MB） | 5 秒以下 |

**テスト方針**: 各エンドポイントに対して複数回リクエストを送信し、p95 レスポンスタイムを計測する。ローカルで実行する（E2E テストと同タイミング）。本格的な負荷テストは行わない（軽量スモークテスト）。

| テストID | テストレベル | レイヤー | 保証種別 | 対応要件ID | 対応設計ID | テスト関数名候補 | 入力（前提条件含む） | 期待結果 |
|---------|------------|---------|---------|-----------|-----------|----------------|-------------------|---------|
| CRS-083 | 統合 | handler | 性能 | NFR-PERF-001 | architecture.md#8.1 | `TestResponseTime_GetDashboard_P95_Under500ms` | `userMember` のトークンで `GET /api/dashboard` を 20 回実行し、レスポンスタイムを計測 | p95 レスポンスタイムが 500ms 以下であること |
| CRS-084 | 統合 | handler | 性能 | NFR-PERF-001 | architecture.md#8.1 | `TestResponseTime_ListMyReports_P95_Under500ms` | `userMember` のトークンで `GET /api/reports` を 20 回実行（テナントAに 50 件のレポートが存在する状態） | p95 レスポンスタイムが 500ms 以下であること |
| CRS-085 | 統合 | handler | 性能 | NFR-PERF-001 | architecture.md#8.1 | `TestResponseTime_ListAllReports_P95_Under500ms` | `userAdmin` のトークンで `GET /api/reports/all` を 20 回実行（テナントAに 50 件のレポートが存在する状態） | p95 レスポンスタイムが 500ms 以下であること |
| CRS-086 | 統合 | handler | 性能 | NFR-PERF-001 | architecture.md#8.1 | `TestResponseTime_GetReport_P95_Under500ms` | `userMember` のトークンで `GET /api/reports/{report_draft.id}` を 20 回実行 | p95 レスポンスタイムが 500ms 以下であること |
| CRS-087 | 統合 | handler | 性能 | NFR-PERF-001 | architecture.md#8.1 | `TestResponseTime_ListPendingReports_P95_Under500ms` | `userApprover` のトークンで `GET /api/workflow/pending` を 20 回実行 | p95 レスポンスタイムが 500ms 以下であること |
| CRS-088 | 統合 | handler | 性能 | NFR-PERF-002 | architecture.md#8.1, files.md | `TestResponseTime_FileUpload_5MB_Under5s` | `userMember` のトークンで `POST .../attachments` に 5MB の PDF ファイルをアップロード（テスト用ダミーファイル、MinIO 使用） | アップロード完了まで 5 秒以下であること |

---

## 実装ガイド

### テストファイルの配置

```
expense-saas/
  internal/
    handler/
      cross_cutting_test.go    # CRS-001〜CRS-054（テナント分離・RBACマトリクス、統合テスト）
      rate_limit_test.go       # CRS-076〜CRS-088（非機能テスト、統合テスト）
  e2e/
    flow1_test.ts              # CRS-055〜CRS-065（E2E フロー1）
    flow2_test.ts              # CRS-066〜CRS-072（E2E フロー2）
    flow3_admin_test.ts        # CRS-073〜CRS-075（E2E フロー3、オプション）
```

### テナント分離テスト実装の注意点

テナント分離テスト（CRS-001〜CRS-016）では、テナントBのフィクスチャを正しく投入する必要がある。`test_strategy.md` §4.5 のテナントBフィクスチャを参照し、テナントAのフィクスチャIDとテナントBのフィクスチャIDが混在しないよう設計する。

```go
//go:build integration

// cross_cutting_test.go（擬似コード）
func TestTenantIsolation_GetReport_OtherTenant_404(t *testing.T) {
    // 前提: テナントAのフィクスチャ + テナントBのフィクスチャを投入
    setupTenantAFixtures(t, db)
    setupTenantBFixtures(t, db)

    // テナントAの userMember トークンでテナントBのレポートにアクセス
    token := generateJWT(t, tenantA_ID, userMember_ID, "member")
    tenantBReportID := "eeeeeeee-0001-0001-0001-000000000001"

    w := httptest.NewRecorder()
    req := newAuthenticatedRequest(t, "GET", "/api/reports/"+tenantBReportID, nil, token)
    router.ServeHTTP(w, req)

    require.Equal(t, http.StatusNotFound, w.Code)
    // エラーコードの確認
    var resp struct {
        Error struct {
            Code string `json:"code"`
        } `json:"error"`
    }
    require.NoError(t, json.Unmarshal(w.Body.Bytes(), &resp))
    assert.Equal(t, "RESOURCE_NOT_FOUND", resp.Error.Code)
}
```

### RBACマトリクステスト実装の注意点

CRS-021〜CRS-054 のうち、機能別ファイルに既存テスト（WFL-006〜WFL-008 等）があるものはクロス参照のみ記載し、テスト関数を重複実装しない。ローカル実行時に同一テストが二重に実行されないよう注意する。

**実装方針（優先度）**:

1. 機能別ファイルに未存在の組み合わせ（CRS-021〜CRS-024, CRS-034〜CRS-036, CRS-040〜CRS-050 等）は `cross_cutting_test.go` に実装する
2. 機能別ファイルにすでにテストがある組み合わせ（WFL-006〜WFL-008, TNT-003〜TNT-005 等）は機能別ファイルのテストIDをコメントで参照するのみとし、重複実装しない

### レート制限テスト実装の注意点

レート制限テスト（CRS-076〜CRS-082）はローカル実行環境のレート制限ウィンドウ設定に依存する。テスト環境では以下のいずれかの方法でテストを安定化させること:

1. **環境変数でウィンドウを短縮**: テスト環境では `RATE_LIMIT_WINDOW=10s` 等の環境変数でウィンドウを短縮し、テスト後にすぐリセットされるようにする
2. **テスト間の待機**: レート制限ウィンドウがリセットされるまで待機する（テスト実行が遅くなるが安全）
3. **テスト用エンドポイント**: テスト環境限定でレート制限カウンターをリセットするエンドポイントを実装する

### E2E テスト実装の注意点

```typescript
// expense-saas/e2e/flow1_test.ts（擬似コード）
import { test, expect } from '@playwright/test';

test('CRS-055: 申請→承認→支払完了 正常フロー', async ({ page }) => {
  // Step 1: Member でログイン
  await page.goto('/login');
  await page.fill('[name="email"]', 'test-member@example.com');
  await page.fill('[name="password"]', 'TestPass1!');
  await page.click('[type="submit"]');
  await expect(page).toHaveURL('/dashboard');

  // Step 2: レポート作成
  await page.click('[data-testid="create-report-button"]');
  await page.fill('[name="title"]', '2026年3月経費');
  // ... （フォーム入力）
  await page.click('[type="submit"]');
  // レポートIDを URL から取得
  const reportUrl = page.url();
  const reportId = reportUrl.split('/').pop();

  // Step 3: 明細追加
  // ...

  // Step 4: 提出
  await page.click('[data-testid="submit-report-button"]');
  await expect(page.locator('[data-testid="report-status"]')).toHaveText('提出済み');

  // ... 以降のステップ
});
```

### E2E テスト前のDB状態の確認

E2E テストでは API モック不使用（実サーバーに接続）。テスト実行前に E2E 用の DB フィクスチャを投入する。

```typescript
// e2e/setup.ts
import { test as setup } from '@playwright/test';

setup('E2E フィクスチャ投入', async ({ request }) => {
  // テスト用 DB をリセットして標準フィクスチャを投入する
  // 実装方法は Step 8（基盤構築）フェーズで決定する
});
```

---

## 優先度

| 優先度 | テストID |
|--------|---------|
| 高（必須） | CRS-001〜CRS-016（テナント分離全件）、CRS-048〜CRS-050（内部統制ルール）、CRS-055（E2E フロー1）、CRS-066（E2E フロー2）、CRS-076〜CRS-079（レート制限）|
| 中 | CRS-021〜CRS-047（RBACマトリクス整合性確認）、CRS-051〜CRS-054（Admin 二面性）、CRS-080〜CRS-082（レート制限詳細）、CRS-083〜CRS-088（レスポンスタイム）|
| 低（ベストエフォート） | CRS-073〜CRS-075（E2E フロー3 Admin フロー）|

**優先度「高」の選定理由**:

- CRS-001〜CRS-016: テナント分離はマルチテナント SaaS の根幹セキュリティ要件。全リソースで漏れなく検証すること
- CRS-048〜CRS-050: 自己承認禁止（RBC-016）・自己支払禁止（RBC-012）は内部統制上の必須ルール
- CRS-055 / CRS-066: 主要業務フローの回帰保証（申請→承認→支払完了、却下→再申請）
- CRS-076〜CRS-079: レート制限はブルートフォース・DoS 対策の基本保護

---

## テストID連番まとめ

| 範囲 | 内容 | 件数 |
|------|------|------|
| CRS-001〜CRS-016 | §1 テナント分離 | 16 件 |
| CRS-017〜CRS-020 | （予備） | 4 件 |
| CRS-021〜CRS-054 | §2 RBACマトリクス整合性確認 | 34 件 |
| CRS-055〜CRS-075 | §3 E2E シナリオ（フロー1〜3） | 21 件 |
| CRS-076〜CRS-088 | §4 非機能テスト（レート制限・レスポンスタイム） | 13 件 |
| **合計** | | **84 件**（予備除く） |
