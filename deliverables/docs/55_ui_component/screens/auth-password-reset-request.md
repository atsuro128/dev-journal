# パスワードリセット要求 -- コンポーネント設計

## この文書の役割

| 項目 | 内容 |
|---|---|
| 目的 | 1 画面のコンポーネントツリー・Props 型定義・データフローを定義する |
| 正本情報 | コンポーネント分割・Props 型・データフロー・状態管理 |
| 扱わない内容 | 入力項目・バリデーション詳細（50_detail_design/screens/auth-password-reset-request.md）、共通コンポーネント詳細（common-components.md） |
| 主な参照元 | `50_detail_design/screens/auth-password-reset-request.md`, `55_ui_component/common-components.md`, `55_ui_component/state-management.md` |
| 主な参照先 | `60_test/test_cases/*.md` |

---

## 1. 基本情報

| 項目 | 内容 |
|---|---|
| 画面 ID | `SCR-AUTH-003` |
| 画面名 | パスワードリセット要求 |
| URL パス | `/password-reset` |
| 画面詳細仕様 | `50_detail_design/screens/auth-password-reset-request.md` |

---

## 2. コンポーネントツリー

```
PasswordResetRequestPage
└── AuthLayout（← common-components.md）
    ├── PasswordResetRequestForm  [表示条件: isSubmitted === false]
    │   ├── FormAlert
    │   ├── AppTextField（← common-components.md）[メールアドレス]
    │   └── SubmitButton
    ├── PasswordResetRequestComplete  [表示条件: isSubmitted === true]
    └── AuthNavLinks
```

---

## 3. コンポーネント定義

### PasswordResetRequestPage

- 配置: `pages/password-reset/PasswordResetRequestPage.tsx`
- 責務: パスワードリセット要求画面のページコンポーネント。AuthLayout でラップし、フォーム表示状態と送信完了状態を切り替える。useRequestPasswordReset Hook を呼び出し、成功時に isSubmitted を true に切り替える。SEC-011 に準拠し、API は常に成功レスポンスを返すため、成功時は必ず送信完了状態に遷移する
- 対応セクション: `50_detail_design/screens/auth-password-reset-request.md` &sect;1, &sect;7, &sect;8

```typescript
// Props なし（ページコンポーネント）
```

### PasswordResetRequestForm

- 配置: `pages/password-reset/PasswordResetRequestForm.tsx`
- 責務: メールアドレス入力フォームの管理。React Hook Form + Zod（passwordResetRequestSchema）でクライアントサイドバリデーションを行い、送信時に onSubmit コールバックを呼び出す。サーバーエラーはフォーム上部の FormAlert に表示する
- 対応セクション: `50_detail_design/screens/auth-password-reset-request.md` &sect;3.1, &sect;4, &sect;5, &sect;6

```typescript
interface PasswordResetRequestFormProps {
  /** フォーム送信時のコールバック */
  onSubmit: (data: { email: string }) => void;
  /** API エラーメッセージ（サーバーエラー等。フォーム上部に Alert 表示） */
  apiError: string | null;
  /** 送信中フラグ（ボタン・入力フィールドの disabled 制御） */
  isPending: boolean;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `onSubmit` | `(data: { email: string }) => void` | Yes | バリデーション通過後の送信コールバック | PasswordResetRequestPage |
| `apiError` | `string \| null` | Yes | API エラーメッセージ（500 等。SEC-011 により認証エラーは発生しない） | useRequestPasswordReset Hook の error |
| `isPending` | `boolean` | Yes | ミューテーション実行中フラグ | useRequestPasswordReset Hook の isPending |

### PasswordResetRequestComplete

- 配置: `pages/password-reset/PasswordResetRequestComplete.tsx`
- 責務: メール送信完了メッセージを表示する。送信完了状態でのみ表示される。SEC-011 に準拠し、メールアドレスの登録有無に関わらず同一のメッセージを表示する
- 対応セクション: `50_detail_design/screens/auth-password-reset-request.md` &sect;3.2, &sect;7

```typescript
// Props なし（表示のみのコンポーネント）
```

### FormAlert

- 配置: `components/ui/FormAlert.tsx`（auth-signup.md と共通）
- 責務: フォーム上部にエラーメッセージを Alert コンポーネントで表示する。メッセージが null の場合は非表示
- 対応セクション: `50_detail_design/screens/auth-password-reset-request.md` &sect;2.2, &sect;6

```typescript
interface FormAlertProps {
  /** 表示するエラーメッセージ。null の場合は非表示 */
  message: string | null;
  /** Alert の severity（デフォルト: 'error'） */
  severity?: 'error' | 'warning' | 'info' | 'success';
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `message` | `string \| null` | Yes | エラーメッセージ | 親コンポーネント |
| `severity` | `'error' \| 'warning' \| 'info' \| 'success'` | No | Alert の種別 | 固定値 |

### SubmitButton

- 配置: `components/ui/SubmitButton.tsx`（auth-signup.md と共通）
- 責務: フォーム送信ボタン。送信中は disabled + スピナー表示
- 対応セクション: `50_detail_design/screens/auth-password-reset-request.md` &sect;2.3

```typescript
interface SubmitButtonProps {
  /** ボタンのラベルテキスト */
  label: string;
  /** 送信中フラグ（true の場合 disabled + スピナー表示） */
  loading: boolean;
  /** ボタンの type 属性（デフォルト: 'submit'） */
  type?: 'submit' | 'button';
  /** ボタンの MUI color（デフォルト: 'primary'） */
  color?: 'primary' | 'error' | 'success' | 'secondary';
  /** ボタンの variant（デフォルト: 'contained'） */
  variant?: 'contained' | 'outlined' | 'text';
  /** フル幅表示（デフォルト: true） */
  fullWidth?: boolean;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `label` | `string` | Yes | ボタンテキスト | 固定値（「リセットメール送信」） |
| `loading` | `boolean` | Yes | 送信中フラグ | useRequestPasswordReset Hook の isPending |

### AuthNavLinks

- 配置: `pages/auth/AuthNavLinks.tsx`（auth-signup.md と共通）
- 責務: 認証画面下部のナビゲーションリンクを表示する。フォーム表示状態と送信完了状態の両方で表示する
- 対応セクション: `50_detail_design/screens/auth-password-reset-request.md` &sect;8

```typescript
interface AuthNavLink {
  /** リンクの前に表示する説明テキスト */
  prefix: string;
  /** リンクテキスト */
  label: string;
  /** 遷移先パス */
  to: string;
}

interface AuthNavLinksProps {
  /** 表示するリンクの配列 */
  links: AuthNavLink[];
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `links` | `AuthNavLink[]` | Yes | ナビゲーションリンクの配列 | 固定値 |

---

## 4. データフロー

```
POST /api/auth/password-reset
  → useRequestPasswordReset()（← state-management.md §3 認証系）
    → PasswordResetRequestPage
      → [isSubmitted === false]
        → PasswordResetRequestForm (props: onSubmit, apiError, isPending)
          → FormAlert (props: message = apiError)
          → AppTextField x 1 (props: React Hook Form の Controller 経由)
          → SubmitButton (props: label, loading = isPending)
      → [isSubmitted === true]
        → PasswordResetRequestComplete (props: なし)
      → AuthNavLinks (props: links)
```

### 使用する Hook

| Hook | 用途 | 定義元 |
|------|------|-------|
| `useRequestPasswordReset` | パスワードリセット要求 API のミューテーション | `state-management.md §3 認証系` |
| `useAuth` | 認証状態の確認（認証済みリダイレクト判定に AuthLayout が使用） | `state-management.md §3 認証系` |

---

## 5. 状態管理

| 状態 | カテゴリ | 管理方法 | スコープ |
|------|---------|---------|---------|
| フォーム入力値（email） | フォーム | React Hook Form + Zod（passwordResetRequestSchema） | PasswordResetRequestForm 内 |
| フィールドバリデーションエラー | フォーム | React Hook Form の formState.errors | PasswordResetRequestForm 内 |
| API ミューテーション状態（isPending, error） | サーバー | useRequestPasswordReset Hook（TanStack Query useMutation） | PasswordResetRequestPage |
| API エラーメッセージ | UI | useState（useRequestPasswordReset の onError から変換） | PasswordResetRequestPage |
| isSubmitted（フォーム表示/送信完了の切替） | UI | useState<boolean>（初期値: false） | PasswordResetRequestPage |

---

## 6. 共通コンポーネント使用箇所

| 共通コンポーネント | 使用箇所 | カスタマイズ |
|------------------|---------|------------|
| `AuthLayout`（← common-components.md） | PasswordResetRequestPage のルートラッパー | children にフォーム or 完了メッセージ + AuthNavLinks を配置 |
| `AppTextField`（← common-components.md） | PasswordResetRequestForm 内の 1 つの入力フィールド | name: email、label: メールアドレス。errorMessage は React Hook Form の errors から取得 |

---

## 7. 表示条件・認可

| コンポーネント | 表示条件 | 操作可否条件 | 根拠 |
|--------------|---------|-------------|------|
| `PasswordResetRequestPage` | 未認証ユーザーのみアクセス可能 | - | `screens/auth-password-reset-request.md` &sect;2.1 |
| `AuthLayout` | 認証済みユーザーがアクセスした場合、ダッシュボード（SCR-DASH-001）にリダイレクト | - | `ui_flow.md` &sect;5.2 |
| `PasswordResetRequestForm` | isSubmitted === false の場合に表示 | 送信中は全フィールド + ボタンが disabled | `screens/auth-password-reset-request.md` &sect;3.1, &sect;2.3 |
| `PasswordResetRequestComplete` | isSubmitted === true の場合に表示 | - | `screens/auth-password-reset-request.md` &sect;3.2, &sect;7 |
| `AuthNavLinks` | 常時表示（フォーム表示・送信完了の両状態で表示） | - | `screens/auth-password-reset-request.md` &sect;8 |

---

## 8. 画面詳細仕様との対応

| 50_detail_design セクション | 対応コンポーネント | 備考 |
|---------------------------|------------------|------|
| &sect;1 基本情報 | PasswordResetRequestPage | URL パス、対応 API |
| &sect;2.1 レイアウト | AuthLayout | 画面中央配置、ヘッダー・サイドナビなし |
| &sect;2.2 エラー表示方針 | FormAlert | フォーム上部のアラート表示 |
| &sect;2.3 ボタン操作中の状態 | SubmitButton, PasswordResetRequestForm | disabled + スピナー |
| &sect;2.4 レート制限 | FormAlert | レート制限エラーをアラート表示 |
| &sect;3.1 フォーム表示状態 | PasswordResetRequestForm | メールアドレス入力 + 送信ボタン |
| &sect;3.2 送信完了状態 | PasswordResetRequestComplete | 送信完了メッセージ + 迷惑メール注意書き |
| &sect;4 入力項目 | AppTextField x 1 | email |
| &sect;5 バリデーションルール | PasswordResetRequestForm（passwordResetRequestSchema） | クライアント: Zod、SEC-011: 未登録メールでも成功 |
| &sect;6 エラー表示 | FormAlert, AppTextField | サーバーエラー -> FormAlert、フィールドエラー -> AppTextField.errorMessage |
| &sect;7 成功時の動作 | PasswordResetRequestPage | isSubmitted を true に切り替え。SEC-011 準拠で常に成功扱い |
| &sect;8 画面遷移 | AuthNavLinks | ログイン画面へのリンク |
| &sect;9 API リクエスト/レスポンス | useRequestPasswordReset Hook | POST /api/auth/password-reset |

---

## 9. テスト追跡用 設計識別子

**コンポーネント参照形式**: `55_ui_component/screens/auth-password-reset-request.md §{ComponentName}`

| 識別子 | 種別 | テスト対象の概要 |
|--------|------|----------------|
| `55_ui_component/screens/auth-password-reset-request.md §PasswordResetRequestPage` | コンポーネント単体テスト | 認証済みリダイレクト、フォーム/完了状態の切替、成功時に常に完了状態に遷移（SEC-011） |
| `55_ui_component/screens/auth-password-reset-request.md §PasswordResetRequestForm` | コンポーネント単体テスト | フォーム入力・バリデーション・送信・エラー表示・disabled 制御 |
| `55_ui_component/screens/auth-password-reset-request.md §PasswordResetRequestComplete` | コンポーネント単体テスト | 送信完了メッセージの表示（迷惑メール注意書きを含む） |
| `55_ui_component/screens/auth-password-reset-request.md §FormAlert` | コンポーネント単体テスト | message が null の場合は非表示、値がある場合は Alert 表示 |
| `55_ui_component/screens/auth-password-reset-request.md §SubmitButton` | コンポーネント単体テスト | loading 時の disabled + スピナー表示 |
| `55_ui_component/screens/auth-password-reset-request.md §AuthNavLinks` | コンポーネント単体テスト | リンクテキストとリンク先の描画（ログイン画面へのリンク） |
| `55_ui_component/state-management.md §useRequestPasswordReset` | Hook 単体テスト | API 呼び出し・成功時のコールバック・エラーハンドリング |
