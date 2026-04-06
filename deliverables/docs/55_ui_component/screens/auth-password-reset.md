# パスワードリセット実行 -- コンポーネント設計

## この文書の役割

| 項目 | 内容 |
|---|---|
| 目的 | 1 画面のコンポーネントツリー・Props 型定義・データフローを定義する |
| 正本情報 | コンポーネント分割・Props 型・データフロー・状態管理 |
| 扱わない内容 | 入力項目・バリデーション詳細（50_detail_design/screens/auth-password-reset.md）、共通コンポーネント詳細（common-components.md） |
| 主な参照元 | `50_detail_design/screens/auth-password-reset.md`, `55_ui_component/common-components.md`, `55_ui_component/state-management.md` |
| 主な参照先 | `60_test/test_cases/*.md` |

---

## 1. 基本情報

| 項目 | 内容 |
|---|---|
| 画面 ID | `SCR-AUTH-004` |
| 画面名 | パスワードリセット実行 |
| URL パス | `/password-reset/:token` |
| 画面詳細仕様 | `50_detail_design/screens/auth-password-reset.md` |

---

## 2. コンポーネントツリー

```
PasswordResetPage
└── AuthLayout（← common-components.md）
    ├── PasswordResetForm  [表示条件: viewState === 'form']
    │   ├── FormAlert
    │   ├── AppTextField（← common-components.md）[新しいパスワード]
    │   ├── AppTextField（← common-components.md）[確認用パスワード]
    │   └── SubmitButton
    ├── PasswordResetComplete  [表示条件: viewState === 'complete']
    ├── PasswordResetTokenInvalid  [表示条件: viewState === 'token-invalid']
    └── AuthNavLinks  [表示条件: viewState === 'form']
```

---

## 3. コンポーネント定義

### PasswordResetPage

- 配置: `pages/password-reset/PasswordResetPage.tsx`
- 責務: パスワードリセット実行画面のページコンポーネント。AuthLayout でラップし、3つの表示状態（フォーム表示・完了・トークン無効）を切り替える。URL パラメータからトークンを取得し、useExecutePasswordReset Hook を呼び出す。API レスポンスの INVALID_TOKEN エラーでトークン無効状態に遷移する。トークンの事前検証は行わない（画面仕様 &sect;7 準拠）
- 対応セクション: `50_detail_design/screens/auth-password-reset.md` &sect;1, &sect;7, &sect;8, &sect;9

```typescript
/** 画面の表示状態 */
type PasswordResetViewState = 'form' | 'complete' | 'token-invalid';

// Props なし（ページコンポーネント）
```

### PasswordResetForm

- 配置: `pages/password-reset/PasswordResetForm.tsx`
- 責務: 新しいパスワードの入力フォームを管理する。React Hook Form + Zod（passwordResetSchema）でクライアントサイドバリデーションを行う。confirm_password は new_password との一致チェックを含む。送信時に onSubmit コールバックを呼び出す。API エラー（サーバーエラー等）はフォーム上部の FormAlert に表示する
- 対応セクション: `50_detail_design/screens/auth-password-reset.md` &sect;3.1, &sect;4, &sect;5, &sect;6

```typescript
interface PasswordResetFormProps {
  /** フォーム送信時のコールバック */
  onSubmit: (data: { new_password: string }) => void;
  /** API エラーメッセージ（サーバーエラー等。フォーム上部に Alert 表示） */
  apiError: string | null;
  /** 送信中フラグ（ボタン・入力フィールドの disabled 制御） */
  isPending: boolean;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `onSubmit` | `(data: { new_password: string }) => void` | Yes | バリデーション通過後の送信コールバック。confirm_password はクライアント側でのみ使用し API には送信しない | PasswordResetPage |
| `apiError` | `string \| null` | Yes | API エラーメッセージ（500 等。INVALID_TOKEN はページレベルで処理） | useExecutePasswordReset Hook の error |
| `isPending` | `boolean` | Yes | ミューテーション実行中フラグ | useExecutePasswordReset Hook の isPending |

### PasswordResetComplete

- 配置: `pages/password-reset/PasswordResetComplete.tsx`
- 責務: パスワード変更完了メッセージを表示する。「ログイン画面へ」ボタンで SCR-AUTH-002 に遷移する
- 対応セクション: `50_detail_design/screens/auth-password-reset.md` &sect;3.2, &sect;8

```typescript
// Props なし（表示のみのコンポーネント。遷移先は固定）
```

### PasswordResetTokenInvalid

- 配置: `pages/password-reset/PasswordResetTokenInvalid.tsx`
- 責務: トークン無効・期限切れメッセージを表示する。「パスワードリセット画面へ」ボタンで SCR-AUTH-003 に遷移する。API レスポンスの INVALID_TOKEN エラーを受けて表示される
- 対応セクション: `50_detail_design/screens/auth-password-reset.md` &sect;3.3, &sect;5（S1, S2）

```typescript
// Props なし（表示のみのコンポーネント。遷移先は固定）
```

### FormAlert

- 配置: `components/ui/FormAlert.tsx`（auth-signup.md と共通）
- 責務: フォーム上部にエラーメッセージを Alert コンポーネントで表示する。メッセージが null の場合は非表示
- 対応セクション: `50_detail_design/screens/auth-password-reset.md` &sect;2.2, &sect;6

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
- 対応セクション: `50_detail_design/screens/auth-password-reset.md` &sect;2.3

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
| `label` | `string` | Yes | ボタンテキスト | 固定値（「パスワードを変更」） |
| `loading` | `boolean` | Yes | 送信中フラグ | useExecutePasswordReset Hook の isPending |

### AuthNavLinks

- 配置: `pages/auth/AuthNavLinks.tsx`（auth-signup.md と共通）
- 責務: 認証画面下部のナビゲーションリンクを表示する。フォーム表示状態でのみ表示する
- 対応セクション: `50_detail_design/screens/auth-password-reset.md` &sect;9

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
PUT /api/auth/password-reset/:token
  → useExecutePasswordReset()（← state-management.md §3 認証系）
    → PasswordResetPage
      → [viewState === 'form']
        → PasswordResetForm (props: onSubmit, apiError, isPending)
          → FormAlert (props: message = apiError)
          → AppTextField x 2 (props: React Hook Form の Controller 経由)
          → SubmitButton (props: label, loading = isPending)
        → AuthNavLinks (props: links)
      → [viewState === 'complete']
        → PasswordResetComplete (props: なし)
      → [viewState === 'token-invalid']
        → PasswordResetTokenInvalid (props: なし)
```

### 使用する Hook

| Hook | 用途 | 定義元 |
|------|------|-------|
| `useExecutePasswordReset` | パスワードリセット実行 API のミューテーション | `state-management.md §3 認証系` |
| `useAuth` | 認証状態の確認（認証済みリダイレクト判定に AuthLayout が使用） | `state-management.md §3 認証系` |

---

## 5. 状態管理

| 状態 | カテゴリ | 管理方法 | スコープ |
|------|---------|---------|---------|
| フォーム入力値（new_password, confirm_password） | フォーム | React Hook Form + Zod（passwordResetSchema） | PasswordResetForm 内 |
| フィールドバリデーションエラー | フォーム | React Hook Form の formState.errors | PasswordResetForm 内 |
| API ミューテーション状態（isPending, error） | サーバー | useExecutePasswordReset Hook（TanStack Query useMutation） | PasswordResetPage |
| API エラーメッセージ | UI | useState（useExecutePasswordReset の onError から変換） | PasswordResetPage |
| viewState（フォーム/完了/トークン無効の切替） | UI | useState&lt;PasswordResetViewState&gt;（初期値: 'form'） | PasswordResetPage |
| token（URL パラメータ） | UI | React Router の useParams | PasswordResetPage |

---

## 6. 共通コンポーネント使用箇所

| 共通コンポーネント | 使用箇所 | カスタマイズ |
|------------------|---------|------------|
| `AuthLayout`（← common-components.md） | PasswordResetPage のルートラッパー | children に表示状態に応じたコンポーネントを配置 |
| `AppTextField`（← common-components.md） | PasswordResetForm 内の 2 つの入力フィールド | name: new_password / confirm_password、label: 新しいパスワード / 確認用パスワード。両方とも type="password"。errorMessage は React Hook Form の errors から取得 |

---

## 7. 表示条件・認可

| コンポーネント | 表示条件 | 操作可否条件 | 根拠 |
|--------------|---------|-------------|------|
| `PasswordResetPage` | 未認証ユーザーのみアクセス可能 | - | `screens/auth-password-reset.md` &sect;2.1 |
| `AuthLayout` | 認証済みユーザーがアクセスした場合、ダッシュボード（SCR-DASH-001）にリダイレクト | - | `ui_flow.md` &sect;5.2 |
| `PasswordResetForm` | viewState === 'form' の場合に表示 | 送信中は全フィールド + ボタンが disabled | `screens/auth-password-reset.md` &sect;3.1, &sect;2.3 |
| `PasswordResetComplete` | viewState === 'complete' の場合に表示 | 「ログイン画面へ」ボタンが常時有効 | `screens/auth-password-reset.md` &sect;3.2 |
| `PasswordResetTokenInvalid` | viewState === 'token-invalid' の場合に表示（API が INVALID_TOKEN エラーを返した場合） | 「パスワードリセット画面へ」ボタンが常時有効 | `screens/auth-password-reset.md` &sect;3.3, &sect;5（S1, S2） |
| `AuthNavLinks` | viewState === 'form' の場合のみ表示 | - | `screens/auth-password-reset.md` &sect;9 |

---

## 8. 画面詳細仕様との対応

| 50_detail_design セクション | 対応コンポーネント | 備考 |
|---------------------------|------------------|------|
| &sect;1 基本情報 | PasswordResetPage | URL パス（:token パラメータ含む）、対応 API |
| &sect;2.1 レイアウト | AuthLayout | 画面中央配置、ヘッダー・サイドナビなし |
| &sect;2.2 エラー表示方針 | FormAlert | フォーム上部のアラート表示 |
| &sect;2.3 ボタン操作中の状態 | SubmitButton, PasswordResetForm | disabled + スピナー |
| &sect;2.4 レート制限 | FormAlert | レート制限エラーをアラート表示 |
| &sect;3.1 フォーム表示状態 | PasswordResetForm | 新しいパスワード + 確認用パスワード + 送信ボタン |
| &sect;3.2 完了状態 | PasswordResetComplete | 完了メッセージ + 「ログイン画面へ」ボタン |
| &sect;3.3 トークン無効状態 | PasswordResetTokenInvalid | 無効メッセージ + 「パスワードリセット画面へ」ボタン |
| &sect;4 入力項目 | AppTextField x 2 | new_password, confirm_password |
| &sect;5 バリデーションルール | PasswordResetForm（passwordResetSchema） | クライアント: Zod（パスワード一致チェック含む）、サーバー: INVALID_TOKEN -> viewState 切替、VALIDATION_ERROR -> フィールドエラー |
| &sect;6 エラー表示 | FormAlert, AppTextField, PasswordResetTokenInvalid | サーバーエラー -> FormAlert、フィールドエラー -> AppTextField.errorMessage、トークン無効 -> PasswordResetTokenInvalid |
| &sect;7 画面表示時の初期処理 | PasswordResetPage | URL パラメータからトークン取得、事前検証なし |
| &sect;8 成功時の動作 | PasswordResetPage | viewState を 'complete' に切り替え |
| &sect;9 画面遷移 | AuthNavLinks, PasswordResetComplete, PasswordResetTokenInvalid | フォーム: ログイン画面リンク、完了: ログイン画面ボタン、無効: パスワードリセット要求画面ボタン |
| &sect;10 API リクエスト/レスポンス | useExecutePasswordReset Hook | PUT /api/auth/password-reset/:token |

---

## 9. テスト追跡用 設計識別子

**コンポーネント参照形式**: `55_ui_component/screens/auth-password-reset.md §{ComponentName}`

| 識別子 | 種別 | テスト対象の概要 |
|--------|------|----------------|
| `55_ui_component/screens/auth-password-reset.md §PasswordResetPage` | コンポーネント単体テスト | 認証済みリダイレクト、3つの表示状態（form/complete/token-invalid）の切替、URL パラメータからのトークン取得 |
| `55_ui_component/screens/auth-password-reset.md §PasswordResetForm` | コンポーネント単体テスト | フォーム入力・バリデーション（パスワード一致チェック含む）・送信・エラー表示・disabled 制御 |
| `55_ui_component/screens/auth-password-reset.md §PasswordResetComplete` | コンポーネント単体テスト | 完了メッセージの表示、「ログイン画面へ」ボタンの遷移先 |
| `55_ui_component/screens/auth-password-reset.md §PasswordResetTokenInvalid` | コンポーネント単体テスト | 無効メッセージの表示、「パスワードリセット画面へ」ボタンの遷移先 |
| `55_ui_component/screens/auth-password-reset.md §FormAlert` | コンポーネント単体テスト | message が null の場合は非表示、値がある場合は Alert 表示 |
| `55_ui_component/screens/auth-password-reset.md §SubmitButton` | コンポーネント単体テスト | loading 時の disabled + スピナー表示 |
| `55_ui_component/screens/auth-password-reset.md §AuthNavLinks` | コンポーネント単体テスト | リンクテキストとリンク先の描画（ログイン画面へのリンク） |
| `55_ui_component/state-management.md §useExecutePasswordReset` | Hook 単体テスト | API 呼び出し・成功時のコールバック・INVALID_TOKEN エラーハンドリング・サーバーエラーハンドリング |
