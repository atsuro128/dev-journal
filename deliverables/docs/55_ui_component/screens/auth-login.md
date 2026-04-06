# ログイン -- コンポーネント設計

## この文書の役割

| 項目 | 内容 |
|---|---|
| 目的 | 1 画面のコンポーネントツリー・Props 型定義・データフローを定義する |
| 正本情報 | コンポーネント分割・Props 型・データフロー・状態管理 |
| 扱わない内容 | 入力項目・バリデーション詳細（50_detail_design/screens/auth-login.md）、共通コンポーネント詳細（common-components.md） |
| 主な参照元 | `50_detail_design/screens/auth-login.md`, `55_ui_component/common-components.md`, `55_ui_component/state-management.md` |
| 主な参照先 | `60_test/test_cases/*.md` |

---

## 1. 基本情報

| 項目 | 内容 |
|---|---|
| 画面 ID | `SCR-AUTH-002` |
| 画面名 | ログイン |
| URL パス | `/login` |
| 画面詳細仕様 | `50_detail_design/screens/auth-login.md` |

---

## 2. コンポーネントツリー

```
LoginPage
└── AuthLayout（← common-components.md）
    ├── LoginForm
    │   ├── FormAlert
    │   ├── AppTextField（← common-components.md）[メールアドレス]
    │   ├── AppTextField（← common-components.md）[パスワード]
    │   └── SubmitButton
    └── AuthNavLinks
```

---

## 3. コンポーネント定義

### LoginPage

- 配置: `pages/login/LoginPage.tsx`
- 責務: ログイン画面のページコンポーネント。AuthLayout でラップし、LoginForm を配置する。useLogin Hook を呼び出し、ミューテーション結果（成功・エラー）を LoginForm に伝播する。ログイン成功時にトークンを AuthStore に保存し、ダッシュボード（またはリダイレクト元）に遷移する
- 対応セクション: `50_detail_design/screens/auth-login.md` &sect;1, &sect;7, &sect;8

```typescript
// Props なし（ページコンポーネント）
```

### LoginForm

- 配置: `pages/login/LoginForm.tsx`
- 責務: ログインフォームの入力・バリデーション・送信を管理する。React Hook Form + Zod（loginSchema）でクライアントサイドバリデーションを行い、送信時に onSubmit コールバックを呼び出す。API エラー（認証失敗・レート制限・サーバーエラー）はフォーム上部の FormAlert に表示する
- 対応セクション: `50_detail_design/screens/auth-login.md` &sect;3, &sect;4, &sect;5, &sect;6

```typescript
interface LoginFormProps {
  /** フォーム送信時のコールバック */
  onSubmit: (data: LoginInput) => void;
  /** API エラーメッセージ（フォーム上部に Alert 表示） */
  apiError: string | null;
  /** 送信中フラグ（ボタン・入力フィールドの disabled 制御） */
  isPending: boolean;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `onSubmit` | `(data: LoginInput) => void` | Yes | バリデーション通過後の送信コールバック | LoginPage |
| `apiError` | `string \| null` | Yes | API エラーメッセージ（401, 429, 500 等） | useLogin Hook の error |
| `isPending` | `boolean` | Yes | ミューテーション実行中フラグ | useLogin Hook の isPending |

### FormAlert

- 配置: `components/ui/FormAlert.tsx`
- 責務: フォーム上部にエラーメッセージを Alert コンポーネントで表示する。メッセージが null の場合は非表示。認証画面ではトースト通知を使用しないため、API エラー表示はこのコンポーネントが担う
- 対応セクション: `50_detail_design/screens/auth-login.md` &sect;2.2, &sect;6

Props 型定義は `common-components.md §FormAlert` を参照。

### SubmitButton

- 配置: `components/ui/SubmitButton.tsx`
- 責務: フォーム送信ボタン。送信中は disabled + スピナー表示
- 対応セクション: `50_detail_design/screens/auth-login.md` &sect;2.3

Props 型定義は `common-components.md §SubmitButton` を参照。

### AuthNavLinks

- 配置: `pages/auth/AuthNavLinks.tsx`
- 責務: 認証画面下部のナビゲーションリンクを表示する
- 対応セクション: `50_detail_design/screens/auth-login.md` &sect;8

Props 型定義は `common-components.md §AuthNavLinks` を参照。

---

## 4. データフロー

```
POST /api/auth/login
  → useLogin()（← state-management.md §3 認証系）
    → LoginPage
      → LoginForm (props: onSubmit, apiError, isPending)
        → FormAlert (props: message = apiError)
        → AppTextField x 2 (props: React Hook Form の Controller 経由)
        → SubmitButton (props: label, loading = isPending)
      → AuthNavLinks (props: links)
```

### 使用する Hook

| Hook | 用途 | 定義元 |
|------|------|-------|
| `useLogin` | ログイン API のミューテーション | `state-management.md §3 認証系` |
| `useAuth` | 認証状態の確認（認証済みリダイレクト判定に AuthLayout が使用） | `state-management.md §3 認証系` |

---

## 5. 状態管理

| 状態 | カテゴリ | 管理方法 | スコープ |
|------|---------|---------|---------|
| フォーム入力値（email, password） | フォーム | React Hook Form + Zod（loginSchema） | LoginForm 内 |
| フィールドバリデーションエラー | フォーム | React Hook Form の formState.errors | LoginForm 内 |
| API ミューテーション状態（isPending, error） | サーバー | useLogin Hook（TanStack Query useMutation） | LoginPage |
| API エラーメッセージ | UI | useState（useLogin の onError から変換） | LoginPage |
| JWT トークン | クライアント | AuthStore（setTokens） | グローバル |
| リダイレクト元パス | UI | React Router の location.state（from） | LoginPage |

---

## 6. 共通コンポーネント使用箇所

| 共通コンポーネント | 使用箇所 | カスタマイズ |
|------------------|---------|------------|
| `AuthLayout`（← common-components.md） | LoginPage のルートラッパー | children に LoginForm + AuthNavLinks を配置 |
| `AppTextField`（← common-components.md） | LoginForm 内の 2 つの入力フィールド | name: email / password、label: メールアドレス / パスワード。パスワードは type="password"。errorMessage は React Hook Form の errors から取得 |

---

## 7. 表示条件・認可

| コンポーネント | 表示条件 | 操作可否条件 | 根拠 |
|--------------|---------|-------------|------|
| `LoginPage` | 未認証ユーザーのみアクセス可能 | - | `screens/auth-login.md` &sect;2.1 |
| `AuthLayout` | 認証済みユーザーがアクセスした場合、ダッシュボード（SCR-DASH-001）にリダイレクト | - | `ui_flow.md` &sect;5.2 |
| `LoginForm` | 常時表示 | 送信中は全フィールド + ボタンが disabled | `screens/auth-login.md` &sect;2.3 |

---

## 8. 画面詳細仕様との対応

| 50_detail_design セクション | 対応コンポーネント | 備考 |
|---------------------------|------------------|------|
| &sect;1 基本情報 | LoginPage | URL パス、対応 API |
| &sect;2.1 レイアウト | AuthLayout | 画面中央配置、ヘッダー・サイドナビなし |
| &sect;2.2 エラー表示方針 | FormAlert | フォーム上部のアラート表示 |
| &sect;2.3 ボタン操作中の状態 | SubmitButton, LoginForm | disabled + スピナー |
| &sect;2.4 レート制限 | FormAlert | レート制限エラーをアラート表示（5 req/min/IP） |
| &sect;3 画面レイアウト | LoginForm | フォームフィールドの配置順序 |
| &sect;4 入力項目 | AppTextField x 2 | email, password |
| &sect;5 バリデーションルール | LoginForm（loginSchema） | クライアント: Zod、サーバー: apiError。SEC-011 準拠の統一エラーメッセージ |
| &sect;6 エラー表示 | FormAlert, AppTextField | 認証失敗 -> FormAlert（SEC-011 統一メッセージ）、フィールドエラー -> AppTextField.errorMessage |
| &sect;7 成功時の動作 | LoginPage | トークン保存 + ダッシュボード遷移（リダイレクト元がある場合はそちらに遷移） |
| &sect;8 画面遷移 | AuthNavLinks | パスワードリセット・サインアップ画面へのリンク |
| &sect;9 API リクエスト/レスポンス | useLogin Hook | POST /api/auth/login |
| &sect;11 認証フロー分岐 | LoginPage, FormAlert | SEC-011: 全認証失敗を同一エラーメッセージで表示 |

---

## 9. テスト追跡用 設計識別子

**コンポーネント参照形式**: `55_ui_component/screens/auth-login.md §{ComponentName}`

| 識別子 | 種別 | テスト対象の概要 |
|--------|------|----------------|
| `55_ui_component/screens/auth-login.md §LoginPage` | コンポーネント単体テスト | 認証済みリダイレクト、ログイン成功時のトークン保存・ダッシュボード遷移、リダイレクト元がある場合の遷移 |
| `55_ui_component/screens/auth-login.md §LoginForm` | コンポーネント単体テスト | フォーム入力・バリデーション・送信・エラー表示・disabled 制御 |
| `55_ui_component/screens/auth-login.md §FormAlert` | コンポーネント単体テスト | message が null の場合は非表示、値がある場合は Alert 表示 |
| `55_ui_component/screens/auth-login.md §SubmitButton` | コンポーネント単体テスト | loading 時の disabled + スピナー表示 |
| `55_ui_component/screens/auth-login.md §AuthNavLinks` | コンポーネント単体テスト | リンクテキストとリンク先の描画（パスワードリセット + サインアップ） |
| `55_ui_component/state-management.md §useLogin` | Hook 単体テスト | API 呼び出し・トークン保存・SEC-011 統一エラーメッセージのハンドリング |
