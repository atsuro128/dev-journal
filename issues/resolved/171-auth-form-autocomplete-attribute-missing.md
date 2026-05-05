# 認証フォームのメールアドレス欄に `autocomplete` 属性が欠落しており、ブラウザ・パスワードマネージャーがサジェスト候補を出さない

## 発見日
2026-05-05

## カテゴリ
implementation

## 影響度
中

## 発見経緯
user-report

## 関連ステップ
Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし

## 問題

### 現象

ログイン画面（`/login`）のメールアドレス欄にフォーカスしても、ブラウザのパスワードマネージャーに保存済みの候補（メールアドレス）がサジェストされない。一方、パスワード欄ではサジェストが出る。

### 原因

`frontend/src/pages/login/LoginForm.tsx:48-67` のメール欄・パスワード欄ともに **`autoComplete` 属性が未指定**。

| フィールド | 現状 | あるべき値 |
|------------|------|------------|
| email | `type="email"` のみ | `autoComplete="username"` |
| password | `type="password"` のみ | `autoComplete="current-password"` |

Chrome は `type="password"` を見て自動的にパスワード欄として認識するため、パスワード欄ではサジェストが出る。一方メール欄は `autoComplete="username"` が無いとパスワードマネージャーが「ログインフォームのユーザー名フィールド」として紐付けにくく、保存されていても候補として出にくい挙動になる。

W3C / WHATWG の HTML 仕様および Chrome / 1Password / Bitwarden 等の公式推奨は、ログインフォームでは `autocomplete="username"` + `autocomplete="current-password"` をペアで付与すること。

### 影響範囲（要確認）

同様の漏れが他の認証関連フォームにもある可能性が高い。

| フォーム | 期待される autocomplete |
|----------|------------------------|
| ログイン（`pages/login/LoginForm.tsx`） | email: `username`, password: `current-password` |
| サインアップ（`pages/signup/`配下） | email: `email`, password: `new-password` |
| パスワードリセット要求（`pages/password-reset/PasswordResetRequestForm.tsx`） | email: `email` |
| パスワードリセット実行（`pages/password-reset/`配下） | password: `new-password` |

## 影響

- メールアドレスのサジェストが出ないため、ユーザーが毎回手入力する必要があり UX が劣化する
- パスワードマネージャー（1Password, Bitwarden, ブラウザ標準）が保存対象として正しく認識しない場合、自動入力が機能しないリスクがある
- 業務継続には支障ないが、繰り返し発生する不便のため放置すべきでない

## 提案

### Step 1: 影響範囲の確認

各認証フォームの実装ファイルを Read し、`autoComplete` 属性の有無を確認する。

### Step 2: 修正

該当する `<AppTextField>` または `<input>` に以下を付与する:

```tsx
// ログイン画面
<AppTextField {...register('email')} type="email" autoComplete="username" ... />
<AppTextField {...register('password')} type="password" autoComplete="current-password" ... />

// サインアップ画面
<AppTextField {...register('email')} type="email" autoComplete="email" ... />
<AppTextField {...register('password')} type="password" autoComplete="new-password" ... />

// パスワードリセット要求
<AppTextField {...register('email')} type="email" autoComplete="email" ... />

// パスワードリセット実行
<AppTextField {...register('password')} type="password" autoComplete="new-password" ... />
```

### Step 3: 回帰テスト

- 各フォームの単体テスト（既存の `LoginForm.test.tsx` 等）に `autoComplete` 属性が期待値で付与されていることのアサーションを追加
- テスト関数名候補: `sets_autocomplete_username_on_email_field` / `sets_autocomplete_current_password_on_password_field` 等

## 修正対象候補ファイル

- `expense-saas/frontend/src/pages/login/LoginForm.tsx`
- `expense-saas/frontend/src/pages/signup/`（実装ファイル要確認）
- `expense-saas/frontend/src/pages/password-reset/PasswordResetRequestForm.tsx`
- `expense-saas/frontend/src/pages/password-reset/`（実行フォーム実装ファイル要確認）
- 対応する `*.test.tsx`

## 完了条件

- 全認証フォームで適切な `autoComplete` 属性が付与されている
- ブラウザのパスワードマネージャーがメールアドレス欄を「ユーザー名フィールド」として認識し、サジェストを表示する（手動動作確認）
- 各フォームの単体テストに `autoComplete` 属性検証が追加されている

## 関連 SMK

- 既存の SMK 観点には含まれていない
- Step 11-A スモークチェックリストに「認証フォームのオートフィル動作」観点の追加を検討

## 解決ログ

### 解決日
2026-05-05

### PR
- PR #133（マージコミット: `bdb9170`）
- ブランチ: `step11/171-auth-form-autocomplete`

### 修正内容
4 つの認証フォームに `autoComplete` 属性を追加:

| ファイル | 追加属性 |
|----------|---------|
| `LoginForm.tsx` | email: `username` / password: `current-password` |
| `SignupForm.tsx` | email: `email` / password: `new-password` |
| `PasswordResetRequestForm.tsx` | email: `email` |
| `PasswordResetForm.tsx` | new_password / confirm_password ともに `new-password` |

加えて単体テストに `toHaveAttribute('autocomplete', ...)` 検証を追加（AUTH-FE-077〜083）。テスト設計書 `dev-journal/deliverables/docs/60_test/test_cases/auth.md` に新規 ID を追記。

### 影響範囲確認
`grep -rn 'type="email"|type="password"' frontend/src/` で全件確認し、認証 4 フォーム以外に対象なしを確認（`pages/settings`/`account`/`profile` 等のディレクトリも未作成）。

### レビュー
- 内部レビュー（reviewer エージェント）: PASS（blocker 0 / warning 0、info 3 件修正不要）
- codex レビュー: APPROVE 相当（指摘事項なし）
- ローカル CI: FE lint / tsc / 認証関連テスト 67 PASS

### 完了条件の達成
- ✅ 全認証フォームに適切な `autoComplete` 属性が付与されている
- ✅ 各フォームの単体テストに autoComplete 属性検証が追加されている（AUTH-FE-077〜083）
- ⏭️ 手動動作確認（ブラウザでメールアドレス候補が表示される）はユーザー側で実施予定

## 追加対応ログ（PR #133 マージ後の手動確認で残課題判明）

### 確認日
2026-05-05

### 残課題

PR #133 マージ後の Edge ブラウザでの手動動作確認で、**ログイン画面のメールアドレス欄をクリックしても保存済みアカウント候補のドロップダウンが表示されない**ことが判明した。一方、パスワード欄をクリックするとペア候補が表示され、選択するとメール欄も自動入力される。

### 原因（高確度の仮説）

`type="email"` と `autoComplete="username"` の **オートフィル経路の衝突**。

| 経路 | トリガ |
|------|--------|
| パスワードマネージャー候補（クレデンシャルペア） | `autoComplete="username"` + `autoComplete="current-password"` |
| 連絡先帳由来のメアド候補（Autofill API） | `type="email"` |

Chromium / Edge では `type="email"` フィールドは Autofill API 経路（連絡先帳メアド候補）に流れ、パスワードマネージャー候補表示が抑制される挙動が観測される。GitHub.com・X.com・Slack 等の主要ログインフォームは `type="text"` + `autoComplete="username"` の組み合わせを採用してこの問題を回避している。

#### 比較例（GitHub.com）

```html
<input type="text" name="login" id="login_field" autocomplete="username" autocorrect="off" autocapitalize="off" required>
```

- `type="text"`: パスワードマネージャー候補表示経路に乗せる
- `autocomplete="username"`: パスワードマネージャー連携

### 修正方針

**LoginForm のみ対象**（Signup / PasswordResetRequest は `autoComplete="email"` で連絡先候補経路を使うため `type="email"` のままで問題なし）。

```tsx
<AppTextField
  {...register('email')}
  name="email"
  id="email"
  type="text"           // email から text に変更
  inputMode="email"     // モバイルで email キーボード表示を維持
  autoComplete="username"
  required
  ...
/>
```

#### バリデーションへの影響
- HTML5 の `type="email"` ネイティブ検証は失われる
- LoginForm.tsx は `<form noValidate>` で HTML5 検証を無効化済みであり、Zod (loginSchema) でクライアントサイドバリデーションを実施しているため実害なし

### 完了条件（追加）

- ✅ Edge / Chrome でログイン画面のメールアドレス欄をクリックすると保存済みアカウント候補が表示される
- ✅ モバイル端末で `inputMode="email"` により `@` キー付きキーボードが表示される（HTML5 type=email と等価の UX）
- ✅ Zod バリデーションが従来どおり動作する（メアド形式チェック）
- ✅ 単体テストに `type="text"` + `inputMode="email"` の検証を追加

### 追加対応の最終解決ログ

#### 解決日（追加対応）
2026-05-05

#### PR
- PR #134（マージコミット: `b4f32da`）
- ブランチ: `step11/171-login-form-input-type`

#### 修正内容
`LoginForm.tsx` のメールアドレス欄を以下に変更:
- `type="email"` → `type="text"`
- `inputProps={{ inputMode: 'email' }}` を追加（AppTextField の `inputProps` は `slotProps.htmlInput` にマージされ、DOM 上で `<input inputmode="email">` として出力される）
- `autoComplete="username"` は維持

SignupForm / PasswordResetRequestForm は変更なし（`autoComplete="email"` 連絡先候補経路で問題ないため）。

単体テスト `LoginForm.test.tsx` の AUTH-FE-077 に以下のアサーションを追加:
- `expect(emailInput).toHaveAttribute('type', 'text')`
- `expect(emailInput).toHaveAttribute('inputmode', 'email')`

テスト設計書 `test_cases/auth.md` の AUTH-FE-077 前提条件・期待結果を `type="text"` + `inputMode="email"` 検証を含む形に更新。

#### レビュー
- 内部レビュー（reviewer エージェント）: PASS（blocker 0 / warning 0、info 1 件修正不要）
- codex レビュー: 指摘事項なし
- ローカル CI: FE lint / tsc / LoginForm テスト 10 PASS / build PASS

#### 手動動作確認
- Edge ブラウザで `/login` のメールアドレス欄クリック → 保存済みアカウント候補ドロップダウンが表示されることを確認: **PASS**
- 候補選択でメアド + パスワードがペア自動入力されることを確認: **PASS**
- ログインボタン押下で従来どおりログイン成功（Zod バリデーション動作）: **PASS**

#### 完了条件の達成（最終）
- ✅ Edge でメールアドレス欄クリック時に保存済みアカウント候補が表示される（手動確認 PASS）
- ✅ DOM 上 `<input type="text" inputmode="email" autocomplete="username">` が出力される
- ✅ Zod バリデーションが従来どおり動作する
- ✅ 単体テスト追加 PASS
