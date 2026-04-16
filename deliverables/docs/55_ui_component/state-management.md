# 状態管理方針

## この文書の役割

| 項目 | 内容 |
|---|---|
| 目的 | フロントエンドの状態管理方針を定義し、全画面で一貫したパターンを確立する |
| 正本情報 | 状態カテゴリの使い分け方針、カスタム Hook 一覧、グローバル store 構成 |
| 扱わない内容 | 実装コード（Step 10）、バックエンドの状態管理（state_machine.md） |
| 主な参照元 | `30_arch/architecture.md`, `50_detail_design/openapi.yaml`, 既存 FE スケルトン |
| 主な参照先 | `55_ui_component/screens/*.md`, `60_test/test_cases/*.md` |

---

## 1. 状態カテゴリ

| カテゴリ | 定義 | 管理手段 | 例 |
|---------|------|---------|---|
| サーバー状態 | API から取得するデータ | TanStack Query v5（useQuery / useMutation） | 経費レポート一覧、ダッシュボード集計、カテゴリ一覧 |
| クライアント状態 | ブラウザ側で永続化する状態 | モジュール変数 + export 関数（既存 `stores/auth.ts` パターン） | アクセストークン、リフレッシュトークン |
| UI 状態 | 画面の一時的な表示制御 | React useState / useReducer（コンポーネントローカル） | モーダル開閉、ローディング、スライドパネル表示 |
| フォーム状態 | フォーム入力値・バリデーション | React Hook Form + Zod | レポート作成フォーム、明細入力フォーム |

### 使い分け判断基準

1. **API から取得するデータか?** -- Yes -> サーバー状態（TanStack Query）
2. **セッション横断で保持すべきか?** -- Yes -> クライアント状態（auth store）
3. **フォームの入力値・バリデーションか?** -- Yes -> フォーム状態（React Hook Form）
4. **上記いずれでもない** -> UI 状態（useState / useReducer）

---

## 2. グローバル store 構成

### AuthStore

- 責務: JWT トークンペア（アクセストークン・リフレッシュトークン）の保持と操作
- 永続化: sessionStorage（タブを閉じると破棄）。アクセストークン・リフレッシュトークンを sessionStorage に保存し、F5 リロード後もログイン状態を維持する。XSS リスクは React の自動エスケープと将来の CSP 設定で軽減し、本番運用では httpOnly Cookie への移行を検討する（issue 084）
- sessionStorage キー名: `auth.access_token`, `auth.refresh_token`

```typescript
// stores/auth.ts
// モジュール変数方式: Zustand を導入せず、export 関数で操作する。
// アクセストークン・リフレッシュトークンを sessionStorage に永続化し、
// F5 リロード後もログイン状態を維持する。
// プライベートモード等で sessionStorage が利用不可の場合はメモリ専用にフォールバックする。

interface AuthStoreActions {
  getAccessToken(): string | null;
  getRefreshToken(): string | null;
  setTokens(access: string, refresh: string): void;
  clearTokens(): void;
}
```

> **設計判断**: Zustand は package.json に含まれていない。認証トークンのみが唯一のグローバルクライアント状態であり、既存スケルトンのモジュール変数方式で十分なため、追加ライブラリは導入しない。ユーザー情報（name, email, role, tenant）はサーバー状態として TanStack Query の `['auth', 'me']` クエリで管理する。

---

## 3. カスタム Hook 一覧

### 認証系

| Hook 名 | 対応 API | 引数 | 戻り値 | 使用画面 |
|---------|---------|------|--------|---------|
| `useAuth` | - | なし | `{ isAuthenticated: boolean }` | 全画面（認証ガード） |
| `useCurrentUser` | `GET /api/auth/me` | なし | `UseQueryResult<AuthUser>` | 全認証済み画面（ヘッダー、RBAC 表示制御） |
| `useSignup` | `POST /api/auth/signup` | なし | `UseMutationResult<ApiResponse<AuthTokens>, ApiClientError, SignupInput>` | SCR-AUTH-001 |
| `useLogin` | `POST /api/auth/login` | なし | `UseMutationResult<ApiResponse<AuthTokens>, ApiClientError, LoginInput>` | SCR-AUTH-002 |
| `useLogout` | `POST /api/auth/logout` | なし | `UseMutationResult<void, ApiClientError, void>` | 全認証済み画面（ヘッダー） |
| `useRequestPasswordReset` | `POST /api/auth/password-reset` | なし | `UseMutationResult<ApiResponse<{ message: string }>, ApiClientError, { email: string }>` | SCR-AUTH-003 |
| `useExecutePasswordReset` | `PUT /api/auth/password-reset/:token` | なし | `UseMutationResult<ApiResponse<{ message: string }>, ApiClientError, { token: string; new_password: string }>` | SCR-AUTH-004 |

### 入出力型定義（認証系）

```typescript
// useAuth（既存スケルトン維持）
interface UseAuthReturn {
  isAuthenticated: boolean;
}

// useCurrentUser
// 入力: なし
// 出力: UseQueryResult<AuthUser>
// AuthUser は api/types.ts で定義済み

// useSignup
interface SignupInput {
  company_name: string;
  user_name: string;
  email: string;
  password: string;
}
// 出力: ApiResponse<AuthTokens>

// useLogin
interface LoginInput {
  email: string;
  password: string;
}
// 出力: ApiResponse<AuthTokens>

// useLogout
// 入力: なし
// 出力: void

// useRequestPasswordReset
// 入力: { email: string }
// 出力: ApiResponse<{ message: string }>

// useExecutePasswordReset
// 入力: { token: string; new_password: string }
// 出力: ApiResponse<{ message: string }>
```

### データフェッチ系

| Hook 名 | 対応 API | 引数 | 戻り値 | 使用画面 |
|---------|---------|------|--------|---------|
| `useDashboard` | `GET /api/dashboard` | なし | `UseQueryResult<ApiResponse<DashboardResponse>>` | SCR-DASH-001 |
| `useCategories` | `GET /api/categories` | なし | `UseQueryResult<ApiResponse<Category[]>>` | SCR-RPT-004（明細追加・編集） |
| `useMyReports` | `GET /api/reports` | `ReportListParams` | `UseQueryResult<ApiListResponse<ExpenseReportSummary>>` | SCR-RPT-001 |
| `useReport` | `GET /api/reports/:id` | `reportId: string` | `UseQueryResult<ApiResponse<ExpenseReportDetail>>` | SCR-RPT-003, SCR-RPT-004 |
| `useAllReports` | `GET /api/reports/all` | `AllReportListParams` | `UseQueryResult<ApiListResponse<ExpenseReportSummary & { submitter: UserSummary }>>` | SCR-ADM-001 |
| `usePendingReports` | `GET /api/workflow/pending` | `PendingReportListParams` | `UseQueryResult<ApiListResponse<PendingReport>>` | SCR-WFL-001 |
| `usePayableReports` | `GET /api/workflow/payable` | `PayableReportListParams` | `UseQueryResult<ApiListResponse<PayableReport>>` | SCR-WFL-002 |
| `useAttachments` | `GET /api/reports/:id/items/:itemId/attachments` | `{ reportId: string; itemId: string }` | `UseQueryResult<ApiResponse<Attachment[]>>` | SCR-RPT-004 |
| `useAttachmentDownloadUrl` | `GET /api/reports/:id/items/:itemId/attachments/:attId/download` | `{ reportId: string; itemId: string; attId: string }` | `UseQueryResult<ApiResponse<AttachmentAccess>>` | SCR-RPT-004 |
| `useAttachmentPreviewUrl` | `GET /api/reports/:id/items/:itemId/attachments/:attId/preview` | `{ reportId: string; itemId: string; attId: string }` | `UseQueryResult<ApiResponse<AttachmentAccess>>` | SCR-RPT-004 |
| `useTenant` | `GET /api/tenant` | なし | `UseQueryResult<ApiResponse<TenantInfo>>` | SCR-ADM-002 |
| `useTenantMembers` | `GET /api/tenant/members` | なし | `UseQueryResult<ApiResponse<UserSummary[]>>` | SCR-ADM-001 |

### 入出力型定義（データフェッチ系）

```typescript
// useDashboard
// 入力: なし
// 出力: ApiResponse<DashboardResponse>
// DashboardResponse は api/types.ts で定義済み

// useCategories
// 入力: なし
// 出力: ApiResponse<Category[]>

// useMyReports
interface ReportListParams {
  page?: number;
  per_page?: number;
  status?: ReportStatus;
  from?: string;   // YYYY-MM-DD
  to?: string;     // YYYY-MM-DD
}
// 出力: ApiListResponse<ExpenseReportSummary>

// useReport
// 入力: reportId: string
// 出力: ApiResponse<ExpenseReportDetail>

// useAllReports
interface AllReportListParams {
  page?: number;
  per_page?: number;
  status?: ReportStatus;
  from?: string;         // YYYY-MM-DD
  to?: string;           // YYYY-MM-DD
  submitter_id?: string; // UUID
}
// 出力: ApiListResponse<ExpenseReportSummary & { submitter: UserSummary }>

// usePendingReports
interface PendingReportListParams {
  page?: number;
  per_page?: number;
  applicant_name?: string;
}
// 出力: ApiListResponse<PendingReport>

// usePayableReports
interface PayableReportListParams {
  page?: number;
  per_page?: number;
  applicant_name?: string;
}
// 出力: ApiListResponse<PayableReport>

// useAttachments
// 入力: { reportId: string; itemId: string }
// 出力: ApiResponse<Attachment[]>

// useAttachmentDownloadUrl
// 入力: { reportId: string; itemId: string; attId: string }
// 出力: ApiResponse<AttachmentAccess>
// enabled: false + 明示的 refetch() 方式（初期ロード不要、クリック時に取得）

// useAttachmentPreviewUrl
// 入力: { reportId: string; itemId: string; attId: string }
// 出力: ApiResponse<AttachmentAccess>
// enabled: false + 明示的 refetch() 方式（初期ロード不要、クリック時に取得）

// useTenant
// 入力: なし
// 出力: ApiResponse<TenantInfo>

// useTenantMembers
// 入力: なし
// 出力: ApiResponse<UserSummary[]>
```

### ミューテーション系

| Hook 名 | 対応 API | 引数 | 戻り値 | 使用画面 |
|---------|---------|------|--------|---------|
| `useCreateReport` | `POST /api/reports` | なし | `UseMutationResult<ApiResponse<ExpenseReportDetail>, ApiClientError, ExpenseReportCreateRequest>` | SCR-RPT-002 |
| `useUpdateReport` | `PUT /api/reports/:id` | なし | `UseMutationResult<ApiResponse<ExpenseReportDetail>, ApiClientError, { id: string } & ExpenseReportUpdateRequest>` | SCR-RPT-003 |
| `useDeleteReport` | `DELETE /api/reports/:id` | なし | `UseMutationResult<void, ApiClientError, string>` | SCR-RPT-004 |
| `useSubmitReport` | `POST /api/reports/:id/submit` | なし | `UseMutationResult<ApiResponse<ExpenseReportDetail>, ApiClientError, { id: string; updated_at: string }>` | SCR-RPT-004 |
| `useCreateItem` | `POST /api/reports/:id/items` | なし | `UseMutationResult<ApiResponse<ExpenseItem>, ApiClientError, { reportId: string } & ExpenseItemCreateRequest>` | SCR-RPT-004 |
| `useUpdateItem` | `PUT /api/reports/:id/items/:itemId` | なし | `UseMutationResult<ApiResponse<ExpenseItem>, ApiClientError, { reportId: string; itemId: string } & ExpenseItemUpdateRequest>` | SCR-RPT-004 |
| `useDeleteItem` | `DELETE /api/reports/:id/items/:itemId` | なし | `UseMutationResult<void, ApiClientError, { reportId: string; itemId: string }>` | SCR-RPT-004 |
| `useUploadAttachment` | `POST /api/reports/:id/items/:itemId/attachments` | なし | `UseMutationResult<ApiResponse<Attachment>, ApiClientError, { reportId: string; itemId: string; file: File }>` | SCR-RPT-004 |
| `useDeleteAttachment` | `DELETE /api/reports/:id/items/:itemId/attachments/:attId` | なし | `UseMutationResult<void, ApiClientError, { reportId: string; itemId: string; attId: string }>` | SCR-RPT-004 |
| `useApproveReport` | `POST /api/workflow/:id/approve` | なし | `UseMutationResult<ApiResponse<ExpenseReportDetail>, ApiClientError, { id: string } & ApproveRequest>` | SCR-RPT-004 |
| `useRejectReport` | `POST /api/workflow/:id/reject` | なし | `UseMutationResult<ApiResponse<ExpenseReportDetail>, ApiClientError, { id: string } & RejectRequest>` | SCR-RPT-004 |
| `useMarkAsPaid` | `POST /api/workflow/:id/pay` | なし | `UseMutationResult<ApiResponse<ExpenseReportDetail>, ApiClientError, { id: string; updated_at: string }>` | SCR-RPT-004 |

### 入出力型定義（ミューテーション系）

```typescript
// useCreateReport
// 入力: ExpenseReportCreateRequest（api/types.ts で定義済み）
// 出力: ApiResponse<ExpenseReportDetail>

// useUpdateReport
// 入力: { id: string } & ExpenseReportUpdateRequest
// 出力: ApiResponse<ExpenseReportDetail>

// useDeleteReport
// 入力: string（レポート ID）
// 出力: void（204 No Content）

// useSubmitReport
// 入力: { id: string; updated_at: string }
// 出力: ApiResponse<ExpenseReportDetail>

// useCreateItem
// 入力: { reportId: string } & ExpenseItemCreateRequest
// 出力: ApiResponse<ExpenseItem>

// useUpdateItem
// 入力: { reportId: string; itemId: string } & ExpenseItemUpdateRequest
// 出力: ApiResponse<ExpenseItem>

// useDeleteItem
// 入力: { reportId: string; itemId: string }
// 出力: void（204 No Content）

// useUploadAttachment
// 入力: { reportId: string; itemId: string; file: File }
// リクエスト形式: multipart/form-data（api/client.ts の FormData 対応済み）
// 出力: ApiResponse<Attachment>

// useDeleteAttachment
// 入力: { reportId: string; itemId: string; attId: string }
// 出力: void（204 No Content）

// useApproveReport
// 入力: { id: string } & ApproveRequest
// ApproveRequest = { comment?: string; updated_at: string }
// 出力: ApiResponse<ExpenseReportDetail>

// useRejectReport
// 入力: { id: string } & RejectRequest
// RejectRequest = { reason: string; updated_at: string }（api/types.ts で定義済み）
// 出力: ApiResponse<ExpenseReportDetail>

// useMarkAsPaid
// 入力: { id: string; updated_at: string }
// 出力: ApiResponse<ExpenseReportDetail>
```

### クエリキー設計

| クエリキー | 対応 Hook | staleTime | 備考 |
|-----------|----------|-----------|------|
| `['auth', 'me']` | `useCurrentUser` | 5分 | ユーザー情報の変更頻度が低い |
| `['dashboard']` | `useDashboard` | 60秒 | ダッシュボードは集計データ |
| `['categories']` | `useCategories` | Infinity | カテゴリはマスタデータ（固定6種類） |
| `['reports', 'mine', params]` | `useMyReports` | 30秒 | フィルタ条件を含む |
| `['reports', 'detail', id]` | `useReport` | 30秒 | レポート ID ごと |
| `['reports', 'all', params]` | `useAllReports` | 30秒 | Admin/Accounting 用 |
| `['workflow', 'pending', params]` | `usePendingReports` | 30秒 | Approver 用 |
| `['workflow', 'payable', params]` | `usePayableReports` | 30秒 | Accounting 用 |
| `['reports', id, 'items', itemId, 'attachments']` | `useAttachments` | 30秒 | 明細ごとの添付一覧 |
| `['reports', id, 'items', itemId, 'attachments', attId, 'download']` | `useAttachmentDownloadUrl` | 0 | 署名付き URL は毎回取得（有効期限15分）。enabled: false |
| `['reports', id, 'items', itemId, 'attachments', attId, 'preview']` | `useAttachmentPreviewUrl` | 0 | 署名付き URL は毎回取得（有効期限15分）。enabled: false |
| `['tenant']` | `useTenant` | 5分 | テナント情報の変更頻度が低い |
| `['tenant', 'members']` | `useTenantMembers` | 60秒 | メンバー一覧 |

### ミューテーション後のキャッシュ無効化

| ミューテーション Hook | invalidate するクエリキー |
|---------------------|------------------------|
| `useCreateReport` | `['reports', 'mine']`, `['dashboard']` |
| `useUpdateReport` | `['reports', 'detail', id]`, `['reports', 'mine']` |
| `useDeleteReport` | `['reports', 'mine']`, `['dashboard']` |
| `useSubmitReport` | `['reports', 'detail', id]`, `['reports', 'mine']`, `['dashboard']`, `['workflow', 'pending']` |
| `useCreateItem` | `['reports', 'detail', reportId]` |
| `useUpdateItem` | `['reports', 'detail', reportId]` |
| `useDeleteItem` | `['reports', 'detail', reportId]` |
| `useUploadAttachment` | `['reports', 'detail', reportId]`, `['reports', reportId, 'items', itemId, 'attachments']` |
| `useDeleteAttachment` | `['reports', 'detail', reportId]`, `['reports', reportId, 'items', itemId, 'attachments']` |
| `useApproveReport` | `['reports', 'detail', id]`, `['workflow', 'pending']`, `['workflow', 'payable']`, `['dashboard']` |
| `useRejectReport` | `['reports', 'detail', id]`, `['workflow', 'pending']`, `['dashboard']` |
| `useMarkAsPaid` | `['reports', 'detail', id]`, `['workflow', 'payable']`, `['dashboard']`, `['reports', 'all']` |
| `useSignup` | `['auth', 'me']` |
| `useLogin` | `['auth', 'me']` |
| `useLogout` | 全クエリキャッシュをクリア（`queryClient.clear()`） |

---

## 4. 非同期処理パターン

### データフェッチ

- **キャッシュ戦略**: TanStack Query のデフォルト設定をベースとし、クエリキーごとに staleTime を設定（SS3 クエリキー設計参照）。gcTime（旧 cacheTime）はデフォルト 5 分を維持する
- **再検証タイミング**: ウィンドウフォーカス時の refetch は無効（既存 main.tsx の QueryClient 設定: `refetchOnWindowFocus: false`）。ミューテーション成功時に関連クエリを invalidate して再取得する
- **リトライ**: 1回（既存 main.tsx の QueryClient 設定: `retry: 1`）。401 エラーは api/client.ts のリフレッシュフローで処理されるため、TanStack Query のリトライ対象外
- **エラーハンドリング**: 各 Hook の `error` プロパティから `ApiClientError` を取得し、画面ごとにエラー表示する。共通エラー（401, 429, 500）はグローバルハンドラで処理する

### ミューテーション

- **楽観的更新**: 不採用。理由: 経費レポートのワークフロー操作（提出・承認・却下・支払完了）は業務上の正確性が最優先であり、楽観的更新のロールバック処理が複雑化するリスクを避ける。ミューテーション成功後に invalidateQueries で再取得する方針を採用する
- **エラー時のロールバック**: 楽観的更新を不採用のためロールバック不要。ミューテーション失敗時は `onError` コールバックでエラーメッセージを表示する
- **楽観的ロック**: レポート更新・提出・承認・却下・支払完了には `updated_at` フィールドを送信し、409 Conflict 時はエラーメッセージ表示 + データ再取得を行う
- **ローディング表示**: ミューテーション実行中は `isPending` フラグで操作ボタンを disabled にし、二重送信を防止する

---

## 5. フォーム状態管理

- **採用ライブラリ**: React Hook Form + Zod
  - **React Hook Form を選定した理由**: 非制御コンポーネントベースで再レンダリングを最小化できる。MUI コンポーネントとの統合が `Controller` で容易。バリデーションライブラリとの連携が柔軟
  - **Zod を選定した理由**: TypeScript ファーストのスキーマバリデーションライブラリ。API リクエスト型（api/types.ts で定義済み）と一致するバリデーションスキーマを定義でき、型安全性を保てる
- **バリデーション方針**:
  - クライアント側: 入力中のリアルタイムバリデーション（必須チェック、形式チェック、文字数制限）を React Hook Form + Zod で実施。送信前にフォーム全体のバリデーションを実行する
  - サーバー側: ビジネスルールに依存するバリデーション（メールアドレス重複、状態遷移の整合性、テナント内 Approver 存在チェック等）はサーバーレスポンスのエラーで処理する。422 レスポンスの `details` 配列をフィールドレベルのエラーとしてフォームに反映する
- **バリデーションスキーマ定義方針**:
  - `src/lib/validations/` ディレクトリにフォームごとの Zod スキーマを配置する
  - スキーマ名は `{操作名}Schema` とする（例: `reportCreateSchema`, `loginSchema`）
  - API リクエスト型（api/types.ts）と Zod スキーマの入力型の一致を `z.infer<typeof schema>` で保証する

### フォーム一覧と対応バリデーションスキーマ

| 画面 | フォーム | スキーマ名 | 主要フィールド |
|------|---------|-----------|--------------|
| SCR-AUTH-001 | サインアップ | `signupSchema` | company_name, user_name, email, password |
| SCR-AUTH-002 | ログイン | `loginSchema` | email, password |
| SCR-AUTH-003 | パスワードリセット要求 | `passwordResetRequestSchema` | email |
| SCR-AUTH-004 | パスワードリセット実行 | `passwordResetSchema` | new_password |
| SCR-RPT-002 | レポート作成 | `reportCreateSchema` | title, period_start, period_end |
| SCR-RPT-003 | レポート編集 | `reportUpdateSchema` | title, period_start, period_end |
| SCR-RPT-004 | 明細追加 | `itemCreateSchema` | expense_date, amount, category_id, description |
| SCR-RPT-004 | 明細編集 | `itemUpdateSchema` | expense_date, amount, category_id, description |
| SCR-RPT-004 | 却下理由 | `rejectSchema` | reason |
| SCR-RPT-004 | 承認コメント | `approveSchema` | comment (任意) |

---

## 6. エラーハンドリングの状態管理パターン

### エラーの分類と処理方針

| エラー種別 | HTTP ステータス | エラーコード | 処理方式 | 表示方法 |
|-----------|----------------|-------------|---------|---------|
| 認証エラー | 401 | UNAUTHORIZED, INVALID_TOKEN, TOKEN_EXPIRED | api/client.ts でリフレッシュ試行 -> 失敗時ログイン画面にリダイレクト | 自動遷移（Snackbar 不要） |
| 認証失敗 | 401 | INVALID_CREDENTIALS | ミューテーション `onError` | フォーム上部にアラート表示 |
| 権限エラー | 403 | FORBIDDEN, SELF_APPROVAL_NOT_ALLOWED, SELF_PAYMENT_NOT_ALLOWED | ミューテーション `onError` | Snackbar（エラー） |
| リソース未発見 | 404 | RESOURCE_NOT_FOUND | クエリ `error` / ミューテーション `onError` | 404 画面 or Snackbar |
| 競合エラー | 409 | CONFLICT | ミューテーション `onError` + データ再取得 | Snackbar（エラー）+ 画面更新 |
| バリデーションエラー | 422 | VALIDATION_ERROR | ミューテーション `onError` | フォームフィールドにエラー反映 |
| 業務ルール違反 | 422 | INVALID_STATE_TRANSITION, REPORT_NOT_EDITABLE 等 | ミューテーション `onError` | Snackbar（エラー）+ 画面更新 |
| ファイルサイズ超過 | 413 | FILE_TOO_LARGE | ミューテーション `onError` | Snackbar（エラー） |
| レート制限 | 429 | RATE_LIMIT_EXCEEDED | ミューテーション `onError` | Snackbar（警告）※1 |
| サーバーエラー | 500 | INTERNAL_ERROR | グローバルハンドラ | Snackbar（エラー）※1 |

※1 **AuthLayout 配下の例外**: 認証画面（SCR-AUTH-001〜004）では Snackbar を使用せず、フォーム上部の Alert で表示する（各認証画面仕様 §S1〜S3 準拠、`common-components.md` AppToast の AuthLayout 除外規定と整合）

### グローバルエラーハンドラ

TanStack Query の `QueryClient` に `onError` グローバルコールバックを設定し、共通エラー処理を集約する。

```typescript
// グローバルエラーハンドラの方針
// 1. 401: api/client.ts が処理（リフレッシュ -> リダイレクト）
// 2. 429: Snackbar で「しばらく待ってから再試行してください」を表示
// 3. 500: Snackbar で「サーバーエラーが発生しました」を表示
// 4. その他: 各 Hook / 画面の onError で個別処理
```

### Snackbar 状態管理

エラー・成功メッセージの Snackbar 表示は、React Context を使用してアプリ全体で共有する。

```typescript
interface SnackbarState {
  open: boolean;
  message: string;
  severity: 'success' | 'error' | 'warning' | 'info';
}

interface SnackbarContextValue {
  showSnackbar(message: string, severity: SnackbarState['severity']): void;
}
```

### 6.5 エラーメッセージ管理方針

API エラーコードからユーザー向け日本語メッセージへの変換方式を定義する。

> **注記**: 本セクションの TypeScript コードブロックは設計方針の例示であり、実装コードではない。

#### 6.5.1 メッセージ保持場所

| 項目 | 方針 |
|------|------|
| 集約先 | `src/lib/errorMessages.ts` に API エラーコード → ユーザー向けメッセージの定数マッピングを定義する |
| i18n | MVP は日本語固定のため i18n ライブラリを導入しない（スコープ外） |
| `constants.ts` との分離 | `src/lib/constants.ts` の `ERROR_CODES` はコード識別子（API 通信・条件分岐用）。`errorMessages.ts` の `ERROR_MESSAGES` は画面表示用の日本語文字列。責務が異なるため別ファイルに分離する |

#### 6.5.2 API エラーコード → 表示メッセージ変換マッピング

```typescript
/**
 * API エラーコードに対応するユーザー向け日本語メッセージ。
 * security.md §8.4 の全エラーコードをカバーする。
 */
export const ERROR_MESSAGES: Record<string, string> = {
  // 400 不正リクエスト
  BAD_REQUEST:
    'リクエストの形式が正しくありません。',
  VALIDATION_ERROR:
    '入力内容に誤りがあります。各項目を確認してください。',

  // 401 認証エラー
  UNAUTHORIZED:
    '認証が必要です。再度ログインしてください。',
  INVALID_CREDENTIALS:
    'メールアドレスまたはパスワードが正しくありません',
  INVALID_TOKEN:
    '認証情報が無効です。再度ログインしてください。',
  TOKEN_EXPIRED:
    '認証の有効期限が切れました。再度ログインしてください。',

  // 403 権限エラー
  FORBIDDEN:
    'この操作を行う権限がありません。',
  SELF_APPROVAL_NOT_ALLOWED:
    '自分のレポートは承認できません',
  SELF_PAYMENT_NOT_ALLOWED:
    '自分が作成したレポートの支払完了は記録できません',

  // 404 リソース未検出
  RESOURCE_NOT_FOUND:
    '指定されたデータが見つかりません。',

  // 409 競合
  CONFLICT:
    '他のユーザーがこのレポートを更新しました。ページを再読み込みしてください。',

  // 413 ファイルサイズ超過
  FILE_TOO_LARGE:
    'ファイルサイズは5MB以下にしてください',

  // 422 業務ルール違反
  INVALID_STATE_TRANSITION:
    'この状態遷移は許可されていません。',
  REPORT_NOT_EDITABLE:
    '提出済みのレポートは編集できません',
  REPORT_NOT_DELETABLE:
    '提出済みのレポートは削除できません',
  EMPTY_REPORT_SUBMISSION:
    '明細を1件以上追加してから提出してください',
  NO_APPROVER_IN_TENANT:
    '承認者が登録されていないため提出できません',
  INVALID_PERIOD:
    '開始日は終了日以前を指定してください',
  INVALID_AMOUNT:
    '金額は正の整数で入力してください',
  INVALID_FILE_TYPE:
    'JPEG, PNG, PDF のみアップロード可能です',
  MISSING_REJECTION_REASON:
    '却下理由を入力してください',

  // 429 レート制限超過
  RATE_LIMIT_EXCEEDED:
    'しばらく待ってから再試行してください',

  // 500 サーバー内部エラー
  INTERNAL_ERROR:
    'サーバーとの通信に失敗しました。しばらくしてから再度お試しください。',
} as const;

/**
 * ERROR_MESSAGES に該当するコードがない場合に使用するフォールバックメッセージ。
 */
export const DEFAULT_ERROR_MESSAGE =
  '予期しないエラーが発生しました。しばらくしてから再度お試しください。';
```

**メッセージ出典一覧**:

| エラーコード | メッセージ | 出典 |
|------------|-----------|------|
| INVALID_CREDENTIALS | メールアドレスまたはパスワードが正しくありません | auth-login.md §5 S1 |
| RATE_LIMIT_EXCEEDED | しばらく待ってから再試行してください | 汎用メッセージ（画面固有の上書きは §6.5.6 参照） |
| INTERNAL_ERROR | サーバーとの通信に失敗しました。しばらくしてから再度お試しください。 | auth-login.md §5 S3、auth-password-reset-request.md §5 S1、report-detail.md §11、workflow-pending.md §11、workflow-payable.md §11 |
| FORBIDDEN | この操作を行う権限がありません。 | report-detail.md §11（403） |
| SELF_APPROVAL_NOT_ALLOWED | 自分のレポートは承認できません | report-detail.md §11 |
| SELF_PAYMENT_NOT_ALLOWED | 自分が作成したレポートの支払完了は記録できません | report-detail.md §11 |
| RESOURCE_NOT_FOUND | 指定されたデータが見つかりません。 | report-detail.md §11（404）、report-edit.md §6 |
| CONFLICT | 他のユーザーがこのレポートを更新しました。ページを再読み込みしてください。 | report-detail.md §11（409）、report-edit.md §7 |
| FILE_TOO_LARGE | ファイルサイズは5MB以下にしてください | report-detail.md §11 |
| INVALID_FILE_TYPE | JPEG, PNG, PDF のみアップロード可能です | report-detail.md §11 |
| EMPTY_REPORT_SUBMISSION | 明細を1件以上追加してから提出してください | report-detail.md §11 |
| NO_APPROVER_IN_TENANT | 承認者が登録されていないため提出できません | report-detail.md §11 |
| MISSING_REJECTION_REASON | 却下理由を入力してください | report-detail.md §11 |
| REPORT_NOT_EDITABLE | 提出済みのレポートは編集できません | report-edit.md §6 |
| REPORT_NOT_DELETABLE | 提出済みのレポートは削除できません | security.md §8.4 の使用場面「draft 以外での削除」から一般的な日本語メッセージを作成 |
| INVALID_STATE_TRANSITION | この状態遷移は許可されていません。 | security.md §8.4 の使用場面から一般的な日本語メッセージを作成（report-detail.md §11 では文脈に応じた個別メッセージを使用） |
| INVALID_PERIOD | 開始日は終了日以前を指定してください | report-edit.md §4 V5 |
| INVALID_AMOUNT | 金額は正の整数で入力してください | security.md §8.4 の使用場面から一般的な日本語メッセージを作成 |
| BAD_REQUEST | リクエストの形式が正しくありません。 | 一般的な日本語メッセージ |
| VALIDATION_ERROR | 入力内容に誤りがあります。各項目を確認してください。 | 一般的な日本語メッセージ（フォールバック用。フィールド別メッセージは §6.5.3 参照） |
| UNAUTHORIZED | 認証が必要です。再度ログインしてください。 | 一般的な日本語メッセージ（通常は自動リダイレクトのため表示されない） |
| INVALID_TOKEN | 認証情報が無効です。再度ログインしてください。 | 一般的な日本語メッセージ（通常は自動リダイレクトのため表示されない） |
| TOKEN_EXPIRED | 認証の有効期限が切れました。再度ログインしてください。 | 一般的な日本語メッセージ（通常は自動リダイレクトのため表示されない） |

#### 6.5.3 変換ユーティリティ関数

```typescript
import { ERROR_MESSAGES, DEFAULT_ERROR_MESSAGE } from '@/lib/errorMessages';

/**
 * 任意のエラーオブジェクトからユーザー向けメッセージを取得する。
 *
 * - ApiClientError の場合: error.code で ERROR_MESSAGES を引く
 * - VALIDATION_ERROR の場合: フィールド別メッセージは各フォームの
 *   onError で処理するため、ここではフォールバックメッセージを返す
 * - 該当コードがない場合: DEFAULT_ERROR_MESSAGE を返す
 */
export function getErrorMessage(error: unknown): string {
  if (error instanceof ApiClientError) {
    return ERROR_MESSAGES[error.code] ?? DEFAULT_ERROR_MESSAGE;
  }
  return DEFAULT_ERROR_MESSAGE;
}
```

**VALIDATION_ERROR の扱い**: `VALIDATION_ERROR` は `details` 配列にフィールド単位のエラー情報を持つ（security.md §8.2）。各フォームの `onError` ハンドラ内で `details` をフォームフィールドにマッピングして表示するため、`getErrorMessage` はフォールバックメッセージ（「入力内容に誤りがあります。各項目を確認してください。」）を返す。

#### 6.5.4 SEC-011 セキュリティメッセージ上書きルール

SEC-011（ユーザー存在の秘匿）に関連して、以下のルールを適用する。

| ルール | 内容 | 根拠 |
|--------|------|------|
| INVALID_CREDENTIALS の統一メッセージ | メール未登録・パスワード不一致・TenantMembership 不在を区別せず、「メールアドレスまたはパスワードが正しくありません」を表示する | auth-login.md §5 S1、security.md §8.4 |
| パスワードリセット要求のエラー分岐不要 | `POST /api/auth/password-reset` は未登録メールでも常に成功レスポンスを返す。FE 側でエラー分岐は不要（常に送信完了状態に遷移する） | auth-password-reset-request.md §5、SEC-011 |
| API の英語メッセージを表示しない | API レスポンスの `message` フィールド（英語）をそのままユーザーに表示してはならない。必ず `ERROR_MESSAGES` の日本語メッセージに変換する | SEC-011、UX 方針 |

#### 6.5.5 画面固有エラーメッセージの扱い

以下のメッセージは `ERROR_MESSAGES` マッピングに含めず、各画面コンポーネント内で定数オブジェクトとして管理する（文字列リテラル直書きを回避する）。

| 種類 | 例 | 管理場所 |
|------|-----|---------|
| 空状態メッセージ | 「レポートはまだありません」「承認待ちのレポートはありません」 | 各画面コンポーネント内の定数オブジェクト |
| 確認ダイアログ文言 | 「このレポートを提出しますか?」「このレポートを削除しますか?」 | 各画面コンポーネント内の定数オブジェクト |
| 成功メッセージ | 「レポートを作成しました」「承認しました」 | 各 Hook の `onSuccess` 内の定数オブジェクト |
| クライアントサイドバリデーションメッセージ | 「タイトルを入力してください」等 | Zod スキーマ定義（`src/lib/validations/*.ts`） |

#### 6.5.6 認証画面固有の上書きルール

`ERROR_MESSAGES` の汎用メッセージと異なるメッセージが必要な画面では、各 Hook の `onError` で個別に上書きする。

| 画面 | エラーコード | 汎用メッセージ | 画面固有メッセージ | 上書き方法 |
|------|------------|--------------|------------------|-----------|
| SCR-AUTH-002 ログイン | RATE_LIMIT_EXCEEDED | 「しばらく待ってから再試行してください」 | 「ログイン試行回数の上限に達しました。しばらくしてからお試しください。」 | `useLogin` Hook の `onError` で `error.code === 'RATE_LIMIT_EXCEEDED'` を判定し、画面固有メッセージを設定する |

```typescript
// useLogin Hook の onError での画面固有メッセージ上書き例
const LOGIN_ERROR_OVERRIDES: Record<string, string> = {
  RATE_LIMIT_EXCEEDED:
    'ログイン試行回数の上限に達しました。しばらくしてからお試しください。',
};

// onError ハンドラ内
const message =
  (error instanceof ApiClientError && LOGIN_ERROR_OVERRIDES[error.code])
    || getErrorMessage(error);
```

この方式により、汎用メッセージをデフォルトとしつつ、画面固有の要件がある場合のみ局所的に上書きできる。

---

## 7. 既存スケルトンとの整合

| 既存ファイル | 現在の実装 | 変更要否 | 備考 |
|------------|----------|---------|------|
| `stores/auth.ts` | モジュール変数方式でアクセストークン・リフレッシュトークンを保持。`getAccessToken`, `getRefreshToken`, `setTokens`, `clearTokens` を export | 変更なし | 本方針のクライアント状態管理方式と一致。Zustand への置き換えは不要 |
| `hooks/useAuth.ts` | `getAccessToken()` の null チェックで `isAuthenticated` を返却 | 変更なし | 認証ガードとして利用可能。ユーザー情報の取得は別途 `useCurrentUser` Hook で対応する |
| `hooks/useReports.ts` | 空ファイル（`export {};`） | 変更あり | 本方針に従いレポート関連 Hook（`useMyReports`, `useReport`, `useCreateReport`, `useUpdateReport`, `useDeleteReport`, `useSubmitReport`）を実装する |
| `api/client.ts` | fetch ラッパー。JWT 自動付与、401 時のリフレッシュフロー、FormData 対応、エラーハンドリング実装済み | 変更なし | 全カスタム Hook の API 呼び出し基盤として使用する。TanStack Query の queryFn / mutationFn 内で `api.get`, `api.post`, `api.put`, `api.delete` を呼び出す |
| `api/types.ts` | 全 API レスポンス型、リクエスト型を定義済み（AuthTokens, AuthUser, ExpenseReportDetail, Category 等） | 変更あり（追加のみ） | カスタム Hook の引数型（`ReportListParams`, `AllReportListParams` 等）を追加する。既存の型定義は変更しない |
| `api/auth.ts` | `login`, `logout`, `healthCheck` 関数を実装済み | 変更あり | TanStack Query の useMutation と統合する形でリファクタリングする。既存の `login` 関数のロジック（トークン保存 + me 取得）は `useLogin` Hook の `onSuccess` に移動する |
| `api/reports.ts` | 空ファイル（`export {};`） | 変更あり | レポート・明細・添付の API 関数を実装する |
| `main.tsx` | `QueryClient` を `retry: 1`, `refetchOnWindowFocus: false` で設定済み。`QueryClientProvider` で App をラップ済み | 変更なし | 本方針の非同期処理パターンと一致 |
| `lib/constants.ts` | エラーコード定数を定義済み | 変更なし | エラーハンドリングで参照する |

### 追加が必要なファイル

| ファイルパス | 責務 |
|------------|------|
| `src/hooks/useDashboard.ts` | ダッシュボード Hook |
| `src/hooks/useCategories.ts` | カテゴリ Hook |
| `src/hooks/useItems.ts` | 明細 Hook（useCreateItem, useUpdateItem, useDeleteItem） |
| `src/hooks/useAttachments.ts` | 添付 Hook（useUploadAttachment, useDeleteAttachment, useAttachmentDownloadUrl, useAttachmentPreviewUrl） |
| `src/hooks/useWorkflow.ts` | ワークフロー Hook（usePendingReports, usePayableReports, useApproveReport, useRejectReport, useMarkAsPaid） |
| `src/hooks/useTenant.ts` | テナント Hook（useTenant, useTenantMembers） |
| `src/api/dashboard.ts` | ダッシュボード API 関数 |
| `src/api/categories.ts` | カテゴリ API 関数 |
| `src/api/items.ts` | 明細 API 関数 |
| `src/api/attachments.ts` | 添付 API 関数 |
| `src/api/workflow.ts` | ワークフロー API 関数 |
| `src/api/tenant.ts` | テナント API 関数 |
| `src/lib/validations/*.ts` | Zod バリデーションスキーマ |
| `src/lib/queryKeys.ts` | クエリキー定数の一元管理 |
| `src/contexts/SnackbarContext.tsx` | Snackbar 状態管理 Context |
| `src/lib/errorMessages.ts` | API エラーコード → ユーザー向けメッセージ変換マッピング + ヘルパー関数 |

---

## 8. API エンドポイント - Hook 対応表（網羅性チェック）

openapi.yaml の全エンドポイントがカスタム Hook でカバーされていることを確認する。

| # | メソッド | エンドポイント | operationId | 対応 Hook | カバー状況 |
|---|---------|-------------|-------------|----------|-----------|
| 1 | GET | /health | getHealth | - | 対象外（インフラ用、FE から直接呼ばない） |
| 2 | POST | /api/auth/signup | signup | `useSignup` | OK |
| 3 | POST | /api/auth/login | login | `useLogin` | OK |
| 4 | POST | /api/auth/refresh | refreshToken | - | api/client.ts 内で自動処理（Hook 不要） |
| 5 | POST | /api/auth/logout | logout | `useLogout` | OK |
| 6 | GET | /api/auth/me | getMe | `useCurrentUser` | OK |
| 7 | POST | /api/auth/password-reset | requestPasswordReset | `useRequestPasswordReset` | OK |
| 8 | PUT | /api/auth/password-reset/:token | executePasswordReset | `useExecutePasswordReset` | OK |
| 9 | GET | /api/dashboard | getDashboard | `useDashboard` | OK |
| 10 | GET | /api/categories | listCategories | `useCategories` | OK |
| 11 | GET | /api/reports | listMyReports | `useMyReports` | OK |
| 12 | POST | /api/reports | createReport | `useCreateReport` | OK |
| 13 | GET | /api/reports/all | listAllReports | `useAllReports` | OK |
| 14 | GET | /api/reports/:id | getReport | `useReport` | OK |
| 15 | PUT | /api/reports/:id | updateReport | `useUpdateReport` | OK |
| 16 | DELETE | /api/reports/:id | deleteReport | `useDeleteReport` | OK |
| 17 | POST | /api/reports/:id/submit | submitReport | `useSubmitReport` | OK |
| 18 | POST | /api/reports/:id/items | createItem | `useCreateItem` | OK |
| 19 | PUT | /api/reports/:id/items/:itemId | updateItem | `useUpdateItem` | OK |
| 20 | DELETE | /api/reports/:id/items/:itemId | deleteItem | `useDeleteItem` | OK |
| 21 | POST | /api/reports/:id/items/:itemId/attachments | uploadAttachment | `useUploadAttachment` | OK |
| 22 | GET | /api/reports/:id/items/:itemId/attachments | listAttachments | `useAttachments` | OK |
| 23 | GET | /api/reports/:id/items/:itemId/attachments/:attId/download | getAttachmentDownload | `useAttachmentDownloadUrl` | OK |
| 23b | GET | /api/reports/:id/items/:itemId/attachments/:attId/preview | getAttachmentPreview | `useAttachmentPreviewUrl` | OK |
| 24 | DELETE | /api/reports/:id/items/:itemId/attachments/:attId | deleteAttachment | `useDeleteAttachment` | OK |
| 25 | GET | /api/workflow/pending | listPendingReports | `usePendingReports` | OK |
| 26 | POST | /api/workflow/:id/approve | approveReport | `useApproveReport` | OK |
| 27 | POST | /api/workflow/:id/reject | rejectReport | `useRejectReport` | OK |
| 28 | GET | /api/workflow/payable | listPayableReports | `usePayableReports` | OK |
| 29 | POST | /api/workflow/:id/pay | markReportAsPaid | `useMarkAsPaid` | OK |
| 30 | GET | /api/tenant | getTenant | `useTenant` | OK |
| 31 | GET | /api/tenant/members | listTenantMembers | `useTenantMembers` | OK |

**結果**: /health（インフラ専用）と /api/auth/refresh（api/client.ts 自動処理）を除く全30エンドポイント（/preview 追加）を Hook でカバー。
