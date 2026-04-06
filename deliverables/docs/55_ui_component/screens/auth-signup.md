# サインアップ -- コンポーネント設計

## この文書の役割

| 項目 | 内容 |
|---|---|
| 目的 | 1 画面のコンポーネントツリー・Props 型定義・データフローを定義する |
| 正本情報 | コンポーネント分割・Props 型・データフロー・状態管理 |
| 扱わない内容 | 入力項目・バリデーション詳細（50_detail_design/screens/auth-signup.md）、共通コンポーネント詳細（common-components.md） |
| 主な参照元 | `50_detail_design/screens/auth-signup.md`, `55_ui_component/common-components.md`, `55_ui_component/state-management.md` |
| 主な参照先 | `60_test/test_cases/*.md` |

---

## 1. 基本情報

| 項目 | 内容 |
|---|---|
| 画面 ID | `SCR-AUTH-001` |
| 画面名 | サインアップ |
| URL パス | `/signup` |
| 画面詳細仕様 | `50_detail_design/screens/auth-signup.md` |

---

## 2. コンポーネントツリー

```
SignupPage
└── AuthLayout（← common-components.md）
    ├── SignupForm
    │   ├── FormAlert
    │   ├── AppTextField（← common-components.md）[会社名]
    │   ├── AppTextField（← common-components.md）[ユーザー名]
    │   ├── AppTextField（← common-components.md）[メールアドレス]
    │   ├── AppTextField（← common-components.md）[パスワード]
    │   └── SubmitButton
    └── AuthNavLinks
```

---

## 3. コンポーネント定義

### SignupPage

- 配置: `pages/signup/SignupPage.tsx`
- 責務: サインアップ画面のページコンポーネント。AuthLayout でラップし、SignupForm を配置する。useSignup Hook を呼び出し、ミューテーション結果（成功・エラー）を SignupForm に伝播する。サインアップ成功時にトークンを AuthStore に保存し、ダッシュボードに遷移する
- 対応セクション: `50_detail_design/screens/auth-signup.md` &sect;1, &sect;7, &sect;8

```typescript
// Props なし（ページコンポーネント）
```

### SignupForm

- 配置: `pages/signup/SignupForm.tsx`
- 責務: サインアップフォームの入力・バリデーション・送信を管理する。React Hook Form + Zod（signupSchema）でクライアントサイドバリデーションを行い、送信時に onSubmit コールバックを呼び出す。API エラーはフォーム上部の FormAlert に表示する
- 対応セクション: `50_detail_design/screens/auth-signup.md` &sect;3, &sect;4, &sect;5, &sect;6

```typescript
interface SignupFormProps {
  /** フォーム送信時のコールバック */
  onSubmit: (data: SignupInput) => void;
  /** API エラーメッセージ（フォーム上部に Alert 表示） */
  apiError: string | null;
  /** 送信中フラグ（ボタン・入力フィールドの disabled 制御） */
  isPending: boolean;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `onSubmit` | `(data: SignupInput) => void` | Yes | バリデーション通過後の送信コールバック | SignupPage |
| `apiError` | `string \| null` | Yes | API エラーメッセージ（409, 429, 500 等） | useSignup Hook の error |
| `isPending` | `boolean` | Yes | ミューテーション実行中フラグ | useSignup Hook の isPending |

### FormAlert

- 配置: `components/ui/FormAlert.tsx`
- 責務: フォーム上部にエラーメッセージを Alert コンポーネントで表示する。メッセージが null の場合は非表示。認証画面ではトースト通知を使用しないため、API エラー表示はこのコンポーネントが担う
- 対応セクション: `50_detail_design/screens/auth-signup.md` &sect;2.2, &sect;6

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

- 配置: `components/ui/SubmitButton.tsx`
- 責務: フォーム送信ボタン。送信中は disabled + スピナー表示。認証画面・業務画面で共通利用する
- 対応セクション: `50_detail_design/screens/auth-signup.md` &sect;2.3

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
| `label` | `string` | Yes | ボタンテキスト | 固定値（「サインアップ」） |
| `loading` | `boolean` | Yes | 送信中フラグ | useSignup Hook の isPending |

### AuthNavLinks

- 配置: `pages/auth/AuthNavLinks.tsx`
- 責務: 認証画面下部のナビゲーションリンクを表示する。画面ごとに表示するリンクが異なるため、links 配列を受け取る
- 対応セクション: `50_detail_design/screens/auth-signup.md` &sect;8

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
POST /api/auth/signup
  → useSignup()（← state-management.md §3 認証系）
    → SignupPage
      → SignupForm (props: onSubmit, apiError, isPending)
        → FormAlert (props: message = apiError)
        → AppTextField x 4 (props: React Hook Form の Controller 経由)
        → SubmitButton (props: label, loading = isPending)
      → AuthNavLinks (props: links)
```

### 使用する Hook

| Hook | 用途 | 定義元 |
|------|------|-------|
| `useSignup` | サインアップ API のミューテーション | `state-management.md §3 認証系` |
| `useAuth` | 認証状態の確認（認証済みリダイレクト判定に AuthLayout が使用） | `state-management.md §3 認証系` |

---

## 5. 状態管理

| 状態 | カテゴリ | 管理方法 | スコープ |
|------|---------|---------|---------|
| フォーム入力値（company_name, user_name, email, password） | フォーム | React Hook Form + Zod（signupSchema） | SignupForm 内 |
| フィールドバリデーションエラー | フォーム | React Hook Form の formState.errors | SignupForm 内 |
| API ミューテーション状態（isPending, error） | サーバー | useSignup Hook（TanStack Query useMutation） | SignupPage |
| API エラーメッセージ | UI | useState（useSignup の onError から変換） | SignupPage |
| JWT トークン | クライアント | AuthStore（setTokens） | グローバル |

---

## 6. 共通コンポーネント使用箇所

| 共通コンポーネント | 使用箇所 | カスタマイズ |
|------------------|---------|------------|
| `AuthLayout`（← common-components.md） | SignupPage のルートラッパー | children に SignupForm + AuthNavLinks を配置 |
| `AppTextField`（← common-components.md） | SignupForm 内の 4 つの入力フィールド | name: company_name / user_name / email / password、label: 会社名 / ユーザー名 / メールアドレス / パスワード。パスワードは type="password"。errorMessage は React Hook Form の errors から取得 |

---

## 7. 表示条件・認可

| コンポーネント | 表示条件 | 操作可否条件 | 根拠 |
|--------------|---------|-------------|------|
| `SignupPage` | 未認証ユーザーのみアクセス可能 | - | `screens/auth-signup.md` &sect;2.1 |
| `AuthLayout` | 認証済みユーザーがアクセスした場合、ダッシュボード（SCR-DASH-001）にリダイレクト | - | `ui_flow.md` &sect;5.2 |
| `SignupForm` | 常時表示 | 送信中は全フィールド + ボタンが disabled | `screens/auth-signup.md` &sect;2.3 |

---

## 8. 画面詳細仕様との対応

| 50_detail_design セクション | 対応コンポーネント | 備考 |
|---------------------------|------------------|------|
| &sect;1 基本情報 | SignupPage | URL パス、対応 API |
| &sect;2.1 レイアウト | AuthLayout | 画面中央配置、ヘッダー・サイドナビなし |
| &sect;2.2 エラー表示方針 | FormAlert | フォーム上部のアラート表示 |
| &sect;2.3 ボタン操作中の状態 | SubmitButton, SignupForm | disabled + スピナー |
| &sect;2.4 レート制限 | FormAlert | レート制限エラーをアラート表示 |
| &sect;3 画面レイアウト | SignupForm | フォームフィールドの配置順序 |
| &sect;4 入力項目 | AppTextField x 4 | company_name, user_name, email, password |
| &sect;5 バリデーションルール | SignupForm（signupSchema） | クライアント: Zod、サーバー: apiError |
| &sect;6 エラー表示 | FormAlert, AppTextField | API エラー -> FormAlert、フィールドエラー -> AppTextField.errorMessage |
| &sect;7 成功時の動作 | SignupPage | トークン保存 + ダッシュボード遷移 |
| &sect;8 画面遷移 | AuthNavLinks | ログイン画面へのリンク |
| &sect;9 API リクエスト/レスポンス | useSignup Hook | POST /api/auth/signup |

---

## 9. テスト追跡用 設計識別子

**コンポーネント参照形式**: `55_ui_component/screens/auth-signup.md §{ComponentName}`

| 識別子 | 種別 | テスト対象の概要 |
|--------|------|----------------|
| `55_ui_component/screens/auth-signup.md §SignupPage` | コンポーネント単体テスト | 認証済みリダイレクト、サインアップ成功時のトークン保存・ダッシュボード遷移 |
| `55_ui_component/screens/auth-signup.md §SignupForm` | コンポーネント単体テスト | フォーム入力・バリデーション・送信・エラー表示・disabled 制御 |
| `55_ui_component/screens/auth-signup.md §FormAlert` | コンポーネント単体テスト | message が null の場合は非表示、値がある場合は Alert 表示 |
| `55_ui_component/screens/auth-signup.md §SubmitButton` | コンポーネント単体テスト | loading 時の disabled + スピナー表示 |
| `55_ui_component/screens/auth-signup.md §AuthNavLinks` | コンポーネント単体テスト | リンクテキストとリンク先の描画 |
| `55_ui_component/state-management.md §useSignup` | Hook 単体テスト | API 呼び出し・トークン保存・エラーハンドリング |
