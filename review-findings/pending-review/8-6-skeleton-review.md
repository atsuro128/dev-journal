# レビュー結果: 8-6 コード生成・スケルトン

- レビュー日: 2026-03-30
- レビュー対象: Step 8 タスク 8-6 スケルトン・コード生成の全成果物
- 判定: **FIX** (blocker: 2件, warning: 5件)

---

## 1. エンドポイント・ハンドラ対応 (openapi.yaml vs main.go)

openapi.yaml に定義された全24エンドポイント（health 含む）に対し、main.go のルート登録とハンドラメソッドの対応を検証した。

| エンドポイント | メソッド | ハンドラ | ルート登録 | 判定 |
|--------------|---------|---------|-----------|------|
| `/health` | GET | `handler.NewHealthHandler` | main.go:131 | OK |
| `/api/auth/signup` | POST | `authHandler.Signup` | main.go:133 | OK |
| `/api/auth/login` | POST | `authHandler.Login` | main.go:134 | OK |
| `/api/auth/refresh` | POST | `authHandler.RefreshToken` | main.go:135 | OK |
| `/api/auth/logout` | POST | `authHandler.Logout` | main.go:136 | OK |
| `/api/auth/password-reset` | POST | `authHandler.RequestPasswordReset` | main.go:137 | OK |
| `/api/auth/password-reset/{token}` | PUT | `authHandler.ExecutePasswordReset` | main.go:138 | OK |
| `/api/auth/me` | GET | `authHandler.GetMe` | main.go:149 | OK |
| `/api/dashboard` | GET | `dashboardHandler.GetDashboard` | main.go:150 | OK |
| `/api/categories` | GET | `categoryHandler.ListCategories` | main.go:151 | OK |
| `/api/reports` | GET | `reportHandler.ListMyReports` | main.go:154 | OK |
| `/api/reports` | POST | `reportHandler.CreateReport` | main.go:155 | OK |
| `/api/reports/all` | GET | `reportHandler.ListAllReports` | main.go:188 | OK |
| `/api/reports/{id}` | GET | `reportHandler.GetReport` | main.go:156 | OK |
| `/api/reports/{id}` | PUT | `reportHandler.UpdateReport` | main.go:157 | OK |
| `/api/reports/{id}` | DELETE | `reportHandler.DeleteReport` | main.go:158 | OK |
| `/api/reports/{id}/submit` | POST | `reportHandler.SubmitReport` | main.go:159 | OK |
| `/api/reports/{id}/items` | POST | `itemHandler.CreateItem` | main.go:162 | OK |
| `/api/reports/{id}/items/{itemId}` | PUT | `itemHandler.UpdateItem` | main.go:163 | OK |
| `/api/reports/{id}/items/{itemId}` | DELETE | `itemHandler.DeleteItem` | main.go:164 | OK |
| `/api/reports/{id}/items/{itemId}/attachments` | POST | `attachmentHandler.UploadAttachment` | main.go:167 | OK |
| `/api/reports/{id}/items/{itemId}/attachments` | GET | `attachmentHandler.ListAttachments` | main.go:168 | OK |
| `/api/reports/{id}/items/{itemId}/attachments/{attId}` | GET | `attachmentHandler.GetAttachmentDownload` | main.go:169 | OK |
| `/api/reports/{id}/items/{itemId}/attachments/{attId}` | DELETE | `attachmentHandler.DeleteAttachment` | main.go:170 | OK |
| `/api/workflow/pending` | GET | `workflowHandler.ListPendingReports` | main.go:175 | OK |
| `/api/workflow/{id}/approve` | POST | `workflowHandler.ApproveReport` | main.go:176 | OK |
| `/api/workflow/{id}/reject` | POST | `workflowHandler.RejectReport` | main.go:177 | OK |
| `/api/workflow/payable` | GET | `workflowHandler.ListPayableReports` | main.go:181 | OK |
| `/api/workflow/{id}/pay` | POST | `workflowHandler.MarkReportAsPaid` | main.go:183 | OK |
| `/api/tenant` | GET | `tenantHandler.GetTenant` | main.go:194 | OK |
| `/api/tenant/members` | GET | `tenantHandler.ListTenantMembers` | main.go:189 | OK |

**結果: 全エンドポイントに対応するハンドラメソッドとルート登録が存在する。**

---

## 2. ルート登録と authz.md 5.1 の整合性

authz.md 5.1 の擬似コードで定義されたルートグループ・ロール要件と main.go の実装を照合した。

| ルートグループ | authz.md 5.1 定義 | main.go 実装 | 判定 |
|-------------|-----------------|------------|------|
| 認証不要 | signup, login, refresh, logout, password-reset, password-reset/{token}, health | main.go:130-139 | OK |
| 全ロール許可 | me, dashboard, categories, reports CRUD, items CRUD, attachments CRUD | main.go:148-171 (RequireRole: member, approver, admin, accounting) | OK |
| Approver のみ | workflow/pending, workflow/{id}/approve, workflow/{id}/reject | main.go:174-178 (RequireRole: approver) | OK |
| Accounting のみ | workflow/payable, workflow/{id}/pay | main.go:181-184 (RequireRole: accounting) | OK |
| Admin, Accounting | reports/all, tenant/members | main.go:187-190 (RequireRole: admin, accounting) | OK |
| Admin のみ | tenant | main.go:193-195 (RequireRole: admin) | OK |

**結果: ルート登録は authz.md 5.1 と完全に整合している。**

---

## 3. サービス interface と空実装

| サービス | interface 定義 | 空実装(ErrNotImplemented) | DI ワイヤリング | 判定 |
|---------|-------------|----------------------|-------------|------|
| AuthService | interfaces.go:12-27 | auth_service.go | main.go:100 | OK |
| ReportService | interfaces.go:30-45 | report_service.go | main.go:101 | OK |
| ItemService | interfaces.go:48-55 | item_service.go | main.go:102 | OK |
| AttachmentService | interfaces.go:58-67 | attachment_service.go | main.go:103 | OK |
| WorkflowService | interfaces.go:70-81 | workflow_service.go | main.go:104 | OK |
| DashboardService | interfaces.go:84-87 | dashboard_service.go | main.go:105 | OK |
| CategoryService | interfaces.go:90-93 | category_service.go | main.go:106 | OK |
| TenantService | interfaces.go:96-101 | tenant_service.go | main.go:107 | OK |
| Authorizer | authorizer.go:9-19 | authorizer.go:24-45 | main.go:97 | OK |

**結果: 全サービスに interface・空実装・DI ワイヤリングが存在する。**

---

## 4. ドメインモデル検証 (domain_model.md vs entity.go / enum.go / errors.go)

### 4.1 エンティティ

| ドメインモデル | 実装 (entity.go) | 判定 |
|-------------|-----------------|------|
| Tenant (4属性) | entity.go:11-16 (4属性) | OK |
| User (6属性) | entity.go:20-27 (6属性) | OK |
| TenantMembership (5属性) | entity.go:32-38 (5属性) | OK |
| ExpenseReport (20属性) | entity.go:55-78 (20属性) | OK |
| ExpenseItem (10属性) | entity.go:82-93 (10属性) | OK - category は CategoryID (uuid.UUID) に変換 (db_schema.md 3.5 値オブジェクト対応に準拠) |
| Attachment (10属性) | entity.go:99-110 (10属性) | OK - s3_path は S3Key に改名 (db_schema.md 11 参照) |
| RefreshToken (6属性) | entity.go:113-120 (6属性) | OK (DB固有テーブル) |
| PasswordResetToken (6属性) | entity.go:123-130 (6属性) | OK (DB固有テーブル) |
| Category (8属性) | entity.go:42-51 (8属性) | OK (DB固有テーブル、マスタテーブル化) |

### 4.2 値オブジェクト (enum.go)

| ドメインモデル | 実装 | 判定 |
|-------------|------|------|
| ReportStatus (5値) | enum.go:4-12 (draft, submitted, approved, rejected, paid) | OK |
| Role (4値) | enum.go:24-31 (admin, approver, member, accounting) | OK |
| MimeType (3値) | enum.go:43-49 (image/jpeg, image/png, application/pdf) | OK |

### 4.3 ドメインエラー (errors.go)

domain_model.md 8. で定義された16個のドメインエラーのうち、errors.go で定義済みを確認した。

| ドメインエラー | errors.go | 判定 |
|-------------|----------|------|
| InvalidStateTransition | ErrInvalidStateTransition | OK |
| SelfApprovalNotAllowed | ErrSelfApprovalNotAllowed | OK |
| SelfPaymentNotAllowed | ErrSelfPaymentNotAllowed | OK |
| EmptyReportSubmission | ErrEmptyReportSubmission | OK |
| InvalidPeriod | ErrInvalidPeriod | OK |
| InvalidAmount | ErrInvalidAmount | OK |
| ReportNotEditable | ErrReportNotEditable | OK |
| NoApproverInTenant | ErrNoApproverInTenant | OK |
| ReportNotDeletable | ErrReportNotDeletable | OK |
| ResourceNotFound | ErrResourceNotFound | OK |
| Forbidden | ErrForbidden | OK |
| InvalidFileType | ErrInvalidFileType | OK |
| FileTooLarge | ErrFileTooLarge | OK |
| MissingRejectionReason | ErrMissingRejectionReason | OK |
| ConflictError | ErrConflict | OK |
| (追加) EmailAlreadyExists | ErrEmailAlreadyExists | OK |
| (追加) InvalidCredentials | ErrInvalidCredentials | OK |
| (追加) TokenExpired | ErrTokenExpired | OK |
| (追加) TokenRevoked | ErrTokenRevoked | OK |

### 4.4 Actor (actor.go)

authz.md 5.3 の Actor 定義 (UserID, TenantID, Role) に一致。

**結果: ドメインモデルは設計ドキュメントと整合している。**

---

## 5. sqlc クエリのテナント分離条件検証

チケット完了条件「全テーブルの基本クエリにテナント分離条件が含まれる」を検証。

### 5.1 テナント境界を持つテーブル

| テーブル | クエリ | tenant_id フィルタ | 判定 |
|---------|-------|------------------|------|
| expense_reports | CreateReport | INSERT に tenant_id 含む | OK |
| expense_reports | GetReportByID | `WHERE tenant_id = $1` | OK |
| expense_reports | ListReportsByUser | `WHERE tenant_id = $1` | OK |
| expense_reports | ListAllReports | `WHERE tenant_id = $1` | OK |
| expense_reports | UpdateReport | `WHERE tenant_id = $1` | OK |
| expense_reports | UpdateReportStatus | `WHERE tenant_id = $1` | OK |
| expense_reports | SoftDeleteReport | `WHERE tenant_id = $1` | OK |
| expense_reports | ListPendingReports | `WHERE tenant_id = $1` | OK |
| expense_reports | ListPayableReports | `WHERE tenant_id = $1` | OK |
| expense_reports | CountReportsByStatus | `WHERE tenant_id = $1` | OK |
| expense_reports | CountMyReportsByStatus | `WHERE tenant_id = $1` | OK |
| expense_reports | MonthlySummaryAll | `WHERE tenant_id = $1` | OK |
| expense_reports | MonthlySummaryByUser | `WHERE tenant_id = $1` | OK |
| expense_reports | ListRecentReports | `WHERE tenant_id = $1` | OK |
| expense_reports | UpdateReportTotalAmount | `WHERE tenant_id = $1` | OK |
| expense_items | CreateExpenseItem | INSERT に tenant_id 含む | OK |
| expense_items | GetExpenseItemByID | `WHERE tenant_id = $1` | OK |
| expense_items | ListExpenseItemsByReportID | `WHERE tenant_id = $1` | OK |
| expense_items | UpdateExpenseItem | `WHERE tenant_id = $1` | OK |
| expense_items | SoftDeleteExpenseItem | `WHERE tenant_id = $1` | OK |
| expense_items | SoftDeleteExpenseItemsByReportID | `WHERE tenant_id = $1` | OK |
| expense_items | CountExpenseItemsByReportID | `WHERE tenant_id = $1` | OK |
| attachments | CreateAttachment | INSERT に tenant_id 含む | OK |
| attachments | GetAttachmentByID | `WHERE tenant_id = $1` | OK |
| attachments | ListAttachmentsByItemID | `WHERE tenant_id = $1` | OK |
| attachments | SoftDeleteAttachment | `WHERE tenant_id = $1` | OK |
| attachments | SoftDeleteAttachmentsByItemID | `WHERE tenant_id = $1` | OK |
| attachments | SoftDeleteAttachmentsByReportID | `WHERE tenant_id = $1` | OK |
| categories | ListActiveCategories | `WHERE tenant_id IS NULL OR tenant_id = $1` | OK (グローバル + テナント固有) |
| categories | GetCategoryByID | tenant_id フィルタなし | 後述 (warning-1) |
| tenant_memberships | ListMembershipsByTenantID | `WHERE tenant_id = $1` | OK |

### 5.2 テナント境界を持たないテーブル (設計上正当)

| テーブル | 理由 |
|---------|------|
| tenants | テナント自身 (PK = tenant_id) |
| users | tenant_id を持たない設計 (domain_model.md 3.2) |
| refresh_tokens | user_id ベース (認証基盤) |
| password_reset_tokens | user_id ベース (認証基盤) |

**結果: テナント分離条件は概ね良好。warning-1 で1件の指摘あり。**

---

## 6. フロントエンド型検証 (openapi.yaml schemas vs types.ts)

openapi.yaml の components/schemas に定義された全スキーマと types.ts の TypeScript 型を照合した。

| openapi.yaml スキーマ | types.ts 型 | 判定 |
|---------------------|------------|------|
| Error | ApiError | OK |
| ValidationError | ValidationError | OK |
| AuthTokens | AuthTokens | OK |
| UserProfile | AuthUser | OK |
| TenantInfo | TenantInfo | OK |
| ExpenseReportSummary | ExpenseReportSummary | OK |
| ExpenseReportDetail | ExpenseReportDetail | OK |
| ExpenseReportCreateRequest | ExpenseReportCreateRequest | OK |
| ExpenseReportUpdateRequest | ExpenseReportUpdateRequest | OK |
| ExpenseItem | ExpenseItem + ExpenseItemWithAttachments | OK |
| ExpenseItemCreateRequest | ExpenseItemCreateRequest | OK |
| ExpenseItemUpdateRequest | ExpenseItemUpdateRequest | OK |
| Attachment | Attachment | OK |
| AttachmentDownload | AttachmentDownload | OK |
| PendingReport | PendingReport | OK |
| PayableReport | PayableReport | OK |
| RejectRequest | RejectRequest | OK |
| (approve request - inline) | ApproveRequest | OK |
| DashboardResponse | DashboardResponse | OK |
| MonthlySummary | MonthlySummary | OK |
| RecentReport | RecentReport | OK |
| Pagination | Pagination | OK |
| UserSummary | UserSummary | OK |
| Category | Category | OK |
| HealthCheckResponse | HealthCheckResponse | OK |
| ReportStatus | ReportStatus (type alias) | OK |
| (追加) | ApiResponse, ApiListResponse | OK (汎用ラッパー) |
| (追加) | Role, MimeType | OK |

**結果: API 定義の全スキーマがフロントエンド型として定義されている。**

---

## 7. ビルド検証

| 検証項目 | 結果 |
|---------|------|
| `go build ./...` | PASS (エラーなし) |
| `go vet ./...` | PASS (警告なし) |
| `npx tsc --noEmit` (frontend) | PASS (エラーなし) |

---

## 8. 指摘事項

### blocker

| # | チェック種別 | 対象ファイル | 問題内容 | 推奨対応 |
|---|-----------|------------|---------|---------|
| B-1 | sqlc クエリ不足 | `db/queries/expense_reports.sql` (ListReportsByUser, ListAllReports) | openapi.yaml の `GET /api/reports` (line 565-581) は `status`, `from`, `to` のクエリパラメータを定義し、`GET /api/reports/all` (line 668-693) はそれに加えて `submitter_id` フィルタも定義している。しかし sqlc クエリ `ListReportsByUser` と `ListAllReports` にはこれらのフィルタ条件が含まれていない。Step 10 の実装時にクエリを書き直すか、スケルトン段階でフィルタパラメータ付きクエリを定義するか、方針を明確にする必要がある。現状の sqlc 生成コードを前提に Step 10 を進めると、実装者がクエリの追加・sqlc 再生成を行う必要があり、スケルトンの「下流工程が曖昧さなく作業できる」条件を満たさない。 | ListReportsByUser と ListAllReports のクエリに `status`, `from` (period_start), `to` (period_end) のオプショナルフィルタを追加する。ListAllReports には `submitter_id` (user_id) フィルタも追加する。sqlc を再生成し、リポジトリ層の `domain.ReportListParams` の既存フィールド (`Status`, `From`, `To`, `SubmitterID`) をクエリに反映させる。 |
| B-2 | 楽観的ロック未反映 | `db/queries/expense_reports.sql` (UpdateReport:29-38, UpdateReportStatus:40-57), `internal/service/params.go` | openapi.yaml は `PUT /api/reports/{id}` (line 2050), `POST /api/reports/{id}/submit` (line 912-917), `POST /api/workflow/{id}/approve` (line 1457-1460), `POST /api/workflow/{id}/reject` (line 2255-2265), `POST /api/workflow/{id}/pay` (line 1649-1652) で `updated_at` (楽観的ロック用) を必須としている。db_schema.md 2.4 でも楽観的ロック方式が明記されている。しかし (1) sqlc の `UpdateReport` クエリの WHERE 句に `AND updated_at = $N` がない、(2) `UpdateReportStatus` クエリにも同様に WHERE 句の楽観的ロック条件がない、(3) `UpdateReportParams` / `UpdateItemParams` に `UpdatedAt` フィールドがない。Step 10 実装者がクエリとパラメータを修正する必要があり、スケルトンとして不完全。 | (1) `UpdateReport` の WHERE 句に `AND updated_at = $N` を追加する。(2) `UpdateReportStatus` の WHERE 句にも同様の条件を追加する。(3) `expense_items.sql` の `UpdateExpenseItem` にも同様の条件を追加する。(4) `service/params.go` の `UpdateReportParams` と `UpdateItemParams` に `UpdatedAt time.Time` フィールドを追加する。(5) sqlc を再生成する。 |

### warning

| # | チェック種別 | 対象ファイル | 問題内容 | 推奨対応 |
|---|-----------|------------|---------|---------|
| W-1 | テナント分離漏れの可能性 | `db/queries/categories.sql` (GetCategoryByID:8-10) | `GetCategoryByID` クエリに `tenant_id` フィルタがない。categories テーブルはグローバルカテゴリ (`tenant_id IS NULL`) とテナント固有カテゴリ (`tenant_id IS NOT NULL`) が混在する設計 (db_schema.md 4.4)。Phase 3 でテナント固有カテゴリを導入した場合、他テナントのカスタムカテゴリが参照可能になる。MVP では全カテゴリがグローバル (`tenant_id IS NULL`) のため実害はないが、Phase 3 の拡張性を考慮すると `AND (tenant_id IS NULL OR tenant_id = $2)` のフィルタを追加しておくことが望ましい。 | MVP スコープでは許容可能だが、Phase 3 拡張時の安全性のため `GetCategoryByID` に `(tenant_id IS NULL OR tenant_id = $2)` フィルタの追加を検討する。 |
| W-2 | リポジトリ interface の不足 | `internal/domain/repository.go` (ReportRepository) | `ReportRepository` に `UpdateReportStatus` に対応するメソッドがない。`Update` メソッドは title/period のみを更新する sqlc の `UpdateReport` に対応しているが、ステータス遷移用の `UpdateReportStatus` クエリに対応するメソッドが存在しない。Step 10 実装者はワークフロー処理 (submit/approve/reject/pay) で `UpdateReportStatus` を使う必要があるが、リポジトリ interface にメソッドがないため追加が必要。 | `ReportRepository` interface に `UpdateStatus(ctx context.Context, report *ExpenseReport) error` メソッドを追加する。あるいは既存の `Update` メソッドをステータス関連フィールドも含む汎用メソッドに拡張する。 |
| W-3 | SoftDelete の連動処理不足 | `internal/repository/postgres/report_repo.go` (SoftDelete:126-135) | `reportRepo.SoftDelete` がレポートのみ論理削除しており、関連する expense_items と attachments の連動論理削除を行っていない。sqlc には `SoftDeleteExpenseItemsByReportID` と `SoftDeleteAttachmentsByReportID` クエリが存在するが、`reportRepo.SoftDelete` 内でこれらが呼ばれていない。openapi.yaml DELETE /api/reports/{id} (line 833-834) は「関連する明細・添付も連動して論理削除される」と明記。 | `reportRepo.SoftDelete` 内で `SoftDeleteAttachmentsByReportID` と `SoftDeleteExpenseItemsByReportID` を呼び出してから `SoftDeleteReport` を実行する。または、この連動処理をサービス層で実装する方針であればその旨をコメントで明記する。 |
| W-4 | DTO の `password_hash` 露出リスク | `internal/domain/entity.go` (User:20-27) | `User` 構造体に `json:"password_hash"` タグが付与されている。スケルトン段階では User をレスポンスに直接シリアライズしないため即座の問題はないが、Step 10 実装時に誤って User 構造体をそのまま JSON レスポンスに含めた場合、password_hash が外部に漏洩する。 | `User` 構造体の `PasswordHash` フィールドから `json` タグを削除するか `json:"-"` に変更する。API レスポンスには `UserProfile` / `UserSummary` DTO を使用する設計であることは理解しているが、防御的措置として推奨。 |
| W-5 | Cursor のパース未定義 | `internal/domain/repository.go`, `internal/service/interfaces.go` | サービス interface の一覧系メソッド (`ListMyReports`, `ListAllReports`, `ListPendingReports`, `ListPayableReports`) は `cursor *string` を受け取るが、リポジトリ層の `ReportListParams.Cursor` は `*time.Time` 型。cursor 文字列から `time.Time` への変換責務がどの層にあるかが未定義。Step 10 実装者がサービス層で変換を実装する必要があるが、カーソル値のエンコード/デコード仕様 (ISO 8601? Base64?) が未記載。 | カーソル値の形式 (推奨: ISO 8601 UTC) をコメントまたは DTO に明記する。あるいはサービス層に `parseCursor(s string) (time.Time, error)` のようなヘルパーのスケルトンを用意する。 |

### info

| # | チェック種別 | 対象ファイル | 問題内容 | 推奨対応 |
|---|-----------|------------|---------|---------|
| I-1 | 命名の一貫性 | `internal/domain/entity.go` (Attachment.S3Key:108) | domain_model.md では `s3_path`、db_schema.md / DDL では `s3_key`。entity.go は `S3Key` を使用しており db_schema.md と整合しているが、domain_model.md との差異がある。db_schema.md セクション 11 で改名理由が説明されているため問題はないが、下流の実装者が domain_model.md を参照した場合に混乱する可能性がある。 | domain_model.md に対する修正は本チケットのスコープ外。下流工程で domain_model.md を参照する際は db_schema.md セクション 11 の改名注記を確認するよう注意喚起する。 |
| I-2 | テスト基盤への影響 | `internal/repository/postgres/item_repo.go` (Create:46-52, Update:107-112, SoftDelete:134-139) | item_repo の Create/Update/SoftDelete 内で `UpdateReportTotalAmount` を呼んでいる。この副作用はリポジトリ層内でのトランザクション整合性として妥当だが、テスト時にアイテム操作のテストが暗黙的にレポートの total_amount 更新も検証する必要がある点をテスト計画時に留意すべき。 | テスト計画 (Step 9) でアイテム CRUD テストに total_amount の再計算検証を含めることを推奨。 |

---

## 9. サマリ

### 良い点

- 全24エンドポイントに対するハンドラ・サービス・リポジトリの三層構造が正しく構築されている
- ルート登録が authz.md 5.1 のルートグループ・ロール定義と完全に一致している
- ドメインモデル (エンティティ・列挙型・エラー) が domain_model.md を忠実に反映している
- テナント分離条件が全業務テーブルの sqlc クエリに正しく含まれている
- フロントエンド型が openapi.yaml の全スキーマをカバーしている
- Go / TypeScript のビルドが共にクリーンに通る
- Authorizer パターンが authz.md 5.3 に準拠して設計されている
- DI ワイヤリングが正しく、全依存関係が解決されている

### 修正が必要な理由

blocker 2件はいずれも「openapi.yaml / db_schema.md で明確に定義された仕様がスケルトンの sqlc クエリ・パラメータに反映されていない」問題であり、Step 10 実装者がクエリの修正・sqlc 再生成・パラメータ型の変更を行う必要が生じる。これはスケルトンの役割（下流工程が曖昧さなく作業できる土台を提供する）に反するため、本チケット内で修正すべきである。

---

## 判定

**FIX** -- blocker 2件 (B-1: 一覧クエリのフィルタ不足, B-2: 楽観的ロック未反映) を修正すること。warning 5件は Step 10 開始前に対応を推奨するが、必須ではない。
