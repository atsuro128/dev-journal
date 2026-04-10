# 10-X 横断レビュー — 観点別確認マップ

これは特定の PR のレビューではない。expense-saas/ リポジトリの `step10/10-X-cross-review` ブランチにある Step 10 全機能の実装を、観点ごとに分割してレビューするためのガイドである。

---

## 観点 1: テナント分離

### 確認すべきこと
- 全 SQL クエリに `tenant_id` 条件が付与されているか（テナント横断アクセスの防止）
- PostgreSQL RLS（Row Level Security）の設定と `set_config('app.current_tenant', ...)` の呼び出しが正しいか
- ミドルウェアでテナントコンテキストが同一コネクション・同一トランザクション内で設定されているか

### 確認ファイル
| ファイル | 確認内容 |
|---------|---------|
| `db/queries/expense_reports.sql` | 全クエリに `WHERE tenant_id = $N` があるか |
| `db/queries/expense_items.sql` | 同上 |
| `db/queries/attachments.sql` | 同上 |
| `db/queries/tenant_memberships.sql` | 同上 |
| `db/queries/tenants.sql` | 同上 |
| `db/queries/categories.sql` | カテゴリはテナント共通リソースか？ tenant_id 不要なら理由を確認 |
| `db/queries/users.sql` | ユーザーはテナント横断か？ 設計意図を確認 |
| `db/migrations/` | RLS ポリシー定義（`CREATE POLICY ... USING (tenant_id = current_setting('app.current_tenant')::uuid)`）|
| `internal/middleware/tenant.go` | `set_config('app.current_tenant', ...)` の呼び出し箇所（52行目付近）|
| `internal/repository/postgres/report_repo.go` | リポジトリ層でテナントコンテキスト付きコネクションを使っているか |
| `internal/repository/postgres/item_repo.go` | 同上 |
| `internal/repository/postgres/attachment_repo.go` | 同上 |

### 上流設計
| 資料 | パス |
|------|------|
| アーキテクチャ設計 | `dev-journal/deliverables/docs/30_arch/architecture.md` の「テナント分離」セクション |
| DB スキーマ | `dev-journal/deliverables/docs/50_detail_design/db_schema.md` の RLS 定義 |

---

## 観点 2: 認可パターン

### 確認すべきこと
- ルーティングの `RequireRole(...)` 設定が authz.md のロール別アクセス制御と一致しているか
- サービス層の `Authorizer` メソッド（CanModifyReport, CanViewReport, CanApproveOrReject, CanMarkAsPaid）のロジックが authz.md に準拠しているか
- 権限外アクセスで 403 が返るか

### 確認ファイル
| ファイル | 確認内容 |
|---------|---------|
| `cmd/server/main.go` 147-212行 | ルーティング定義。各 Group の `RequireRole(...)` が正しいロールを指定しているか |
| `internal/middleware/rbac.go` | `RequireRole` の実装。コンテキストからロールを取得し、許可リストと照合しているか |
| `internal/service/authorizer.go` | `CanModifyReport`（32行）: owner のみ + draft 限定か。`CanViewReport`（44行）: owner or admin/accounting か。`CanApproveOrReject`（76行）: approver のみか。`CanMarkAsPaid`（85行）: accounting のみか |
| `internal/handler/report.go` | ハンドラが `authorizer` を呼んでいるか |
| `internal/handler/workflow.go` | 同上 |

### 上流設計
| 資料 | パス |
|------|------|
| 認可設計 | `dev-journal/deliverables/docs/50_detail_design/authz.md` |

### authz.md との照合ポイント
| エンドポイント | 期待されるロール | ルーティング定義箇所 |
|--------------|----------------|-------------------|
| `GET /api/reports` (自分のレポート一覧) | member, approver, admin, accounting | main.go:171 |
| `POST /api/reports` | member, approver, admin, accounting | main.go:172 |
| `GET/PUT/DELETE /api/reports/{id}` | owner（サービス層で検証） | main.go:173-175 |
| `POST /api/reports/{id}/submit` | owner（サービス層で検証） | main.go:176 |
| `GET /api/workflow/pending` | approver | main.go:193 |
| `POST /api/workflow/{id}/approve` | approver | main.go:194 |
| `POST /api/workflow/{id}/reject` | approver | main.go:195 |
| `GET /api/workflow/payable` | accounting | main.go:200 |
| `POST /api/workflow/{id}/pay` | accounting | main.go:201 |
| `GET /api/reports/all` | admin, accounting | main.go:206 |
| `GET /api/tenant/members` | admin, accounting | main.go:207 |
| `GET /api/tenant` | admin | main.go:212 |

---

## 観点 3: 状態遷移

### 確認すべきこと
- ドメイン層の状態遷移メソッド（Submit, Approve, Reject, MarkAsPaid）が state_machine.md の遷移ルールと一致しているか
- 不正な遷移で 422 が返るか
- 遷移時の事前条件（明細0件で申請不可、approver ロール不在時の申請不可等）が実装されているか

### 確認ファイル
| ファイル | 確認内容 |
|---------|---------|
| `internal/domain/report.go` | `Submit`（17行）: draft→submitted、明細0件チェック、approver 存在チェック。`Approve`（49行）: submitted→approved。`Reject`（73行）: submitted→rejected。`MarkAsPaid`（99行）: approved→paid |
| `internal/domain/enum.go` | ステータス定義: draft, submitted, approved, rejected, paid |
| `internal/service/workflow_service.go` | サービス層が `report.Approve()` / `report.Reject()` / `report.MarkAsPaid()` を呼んでいるか。楽観ロックの比較後にドメインメソッドを呼ぶ順序 |
| `internal/service/report_service.go` | `SubmitReport` がドメイン層の `report.Submit()` を呼んでいるか |
| `internal/handler/helpers.go` | `respondDomainError`（45行）: ドメインエラーを HTTP ステータスにマッピング（ErrInvalidTransition → 422 等） |

### 上流設計
| 資料 | パス |
|------|------|
| 状態遷移設計 | `dev-journal/deliverables/docs/20_domain/state_machine.md` |

### state_machine.md との照合ポイント
| 遷移 | 許可 | 禁止 |
|------|------|------|
| draft → submitted | Submit() | — |
| submitted → approved | Approve() | — |
| submitted → rejected | Reject() | — |
| approved → paid | MarkAsPaid() | — |
| rejected → draft | 再編集（UpdateReport で draft に戻るか？）| — |
| draft → approved/rejected/paid | — | 禁止されているか確認 |
| submitted → draft/paid | — | 禁止されているか確認 |
| approved → draft/submitted/rejected | — | 禁止されているか確認 |
| paid → 任意 | — | 禁止されているか確認 |

---

## 観点 4: 共通基盤統一（バックエンド）

### 確認すべきこと
- 全ハンドラが同じエラーレスポンス形式（`respondDomainError` / `middleware.RespondError`）を使っているか
- 全ハンドラが `actorFromRequest` でコンテキストから Actor を取得しているか
- レスポンス形式が `{ "data": ... }` または `{ "error": { "code": "...", "message": "..." } }` で統一されているか

### 確認ファイル
| ファイル | 確認内容 |
|---------|---------|
| `internal/handler/helpers.go` | `actorFromRequest`（15行）、`respondDomainError`（45行）の実装 |
| `internal/middleware/response.go` | `RespondJSON`、`RespondError` の実装。レスポンス形式の定義 |
| `internal/handler/auth.go` | エラー処理パターンが他ハンドラと統一されているか |
| `internal/handler/report.go` | 同上 |
| `internal/handler/item.go` | 同上 |
| `internal/handler/attachment.go` | 同上 |
| `internal/handler/workflow.go` | 同上 |
| `internal/handler/dashboard.go` | 同上 |
| `internal/handler/tenant.go` | 同上 |
| `internal/handler/category.go` | 同上 |
| `internal/domain/errors.go` | ドメインエラー定義（ErrNotFound, ErrForbidden, ErrConflict 等）が網羅的か |

### 確認方法
各ハンドラファイルで以下のパターンが統一されているか grep で横断確認:
```bash
grep -n "respondDomainError\|RespondError\|RespondJSON" internal/handler/*.go
```

---

## 観点 5: 共通基盤統一（フロントエンド）

### 確認すべきこと
- 全画面が共通 `apiClient`（`src/api/client.ts`）を使っているか（fetch 直接呼び出しがないか）
- カスタム Hook の queryKey 体系が state-management.md に従っているか
- 共通 UI コンポーネント（FormAlert, PageTitle, SubmitButton, StatusChip 等）が適切に利用されているか
- エラーハンドリング（403/409/422）の表示が統一されているか

### 確認ファイル
| ファイル | 確認内容 |
|---------|---------|
| `frontend/src/api/client.ts` | 共通 API クライアント。トークン管理、リフレッシュ、エラーハンドリング |
| `frontend/src/api/reports.ts` | API 関数が apiClient を使っているか |
| `frontend/src/api/auth.ts` | 同上 |
| `frontend/src/api/types.ts` | 共通型定義 |
| `frontend/src/hooks/useReports.ts` | queryKey が `['reports', 'mine']` 等の体系に従っているか |
| `frontend/src/hooks/useAllReports.ts` | queryKey 体系 |
| `frontend/src/hooks/useDashboard.ts` | queryKey 体系 |
| `frontend/src/hooks/useItems.ts` | queryKey 体系 |
| `frontend/src/hooks/useAttachments.ts` | queryKey 体系 |
| `frontend/src/hooks/useApproveReport.ts` | mutation 後のキャッシュ無効化が正しいか |
| `frontend/src/hooks/useRejectReport.ts` | 同上 |
| `frontend/src/hooks/useMarkAsPaid.ts` | 同上 |
| `frontend/src/stores/auth.ts` | トークン管理（zustand） |

### 上流設計
| 資料 | パス |
|------|------|
| 状態管理設計 | `dev-journal/deliverables/docs/55_ui_component/state-management.md` |

### 確認方法
```bash
# fetch 直接呼び出しがないか（テストファイル以外）
grep -rn "fetch(" frontend/src --include="*.ts" --include="*.tsx" | grep -v __tests__ | grep -v node_modules
# queryKey の体系確認
grep -rn "queryKey" frontend/src/hooks/*.ts
```

---

## 観点 6: セキュリティ

### 確認すべきこと
- SQL クエリが全てプレースホルダ（`$1`, `$2`, ...）を使っているか（文字列結合がないか）
- パスワードハッシュが bcrypt を使っているか（security.md 準拠）
- 認証ミドルウェアが全保護エンドポイントに適用されているか
- セキュリティヘッダーが設定されているか
- JWT 署名アルゴリズムが security.md 準拠か

### 確認ファイル
| ファイル | 確認内容 |
|---------|---------|
| `db/queries/*.sql` | 全クエリがプレースホルダを使っているか。`fmt.Sprintf` や文字列結合で SQL を組み立てている箇所がないか |
| `internal/service/auth_service.go` | `bcrypt.GenerateFromPassword` / `bcrypt.CompareHashAndPassword` を使っているか |
| `internal/pkg/jwt/jwt.go` | 署名アルゴリズム（RS256 等）、鍵の読み込み方式 |
| `internal/middleware/auth.go` | JWT 検証、コンテキストへのユーザー情報設定 |
| `internal/middleware/security_headers.go` | X-Content-Type-Options, X-Frame-Options, Strict-Transport-Security 等 |
| `internal/middleware/cors.go` | CORS 設定（AllowedOrigins が設定から読まれているか） |
| `internal/middleware/ratelimit.go` | レートリミット設定 |
| `cmd/server/main.go` 140-144行 | ミドルウェアチェーン。セキュリティヘッダー・CORS・レートリミットが全ルートに適用されているか |
| `cmd/server/main.go` 159-163行 | 認証ミドルウェア（Auth + Tenant）が保護グループに適用されているか |

### 上流設計
| 資料 | パス |
|------|------|
| セキュリティ設計 | `dev-journal/deliverables/docs/50_detail_design/security.md` |

### 確認方法
```bash
# SQL 文字列結合がないか
grep -rn "fmt.Sprintf.*SELECT\|fmt.Sprintf.*INSERT\|fmt.Sprintf.*UPDATE\|fmt.Sprintf.*DELETE" internal/ --include="*.go"
# bcrypt 使用確認
grep -rn "bcrypt" internal/ --include="*.go"
```
