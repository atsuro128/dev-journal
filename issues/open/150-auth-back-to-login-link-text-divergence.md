# 認証画面「ログイン画面に戻る」リンクテキストの設計乖離と UX 違和感

## 発見日
2026-04-25

## カテゴリ
implementation / ui-design

## 影響度
中（ユーザーが直接目にする文言。機能動作に影響なし、「ログイン」が重複する表記の違和感が UX を損なう）

## 発見経緯
Step 11-A Phase 5 SMK-099（パスワードリセット要求画面の「ログイン画面に戻る」リンク確認）実施時に発見。画面上に「ログイン画面へ戻るログイン」という表記が出力され、「ログイン」が重複していることが判明。

## 関連ステップ
Step 5（画面詳細設計）/ Step 5.5（UI コンポーネント設計）/ Step 10-B（認証 UI 実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし

## 関連 issue
なし

## 問題

### 表示上の症状

`PasswordResetRequestPage` および `PasswordResetPage` の画面下部に「ログイン画面へ戻るログイン」が表示される。「ログイン」が重複しており、不自然な表記になっている。

### 実装構造

`AuthNavLinks` コンポーネント（`frontend/src/pages/auth/AuthNavLinks.tsx`）は `prefix + label` 構成でリンクを描画する。他画面では「リード文 + クリッカブル名詞」として自然な日本語になるが、パスワードリセット系画面では prefix 自体が完結した文になっているため、後続の label「ログイン」が浮いてしまう。

| 画面 | prefix | label | 結果 |
|---|---|---|---|
| LoginPage | 「アカウントをお持ちでない方は」 | 「新規登録」 | 自然 |
| LoginPage | 「パスワードを忘れた方は」 | 「パスワードリセット」 | 自然 |
| SignupPage | 「既にアカウントをお持ちの方は」 | 「ログイン」 | 自然 |
| PasswordResetRequestPage | 「ログイン画面へ戻る」 | 「ログイン」 | 違和感（重複） |
| PasswordResetPage | 「ログイン画面へ戻る」 | 「ログイン」 | 違和感（重複） |

該当箇所:
- `expense-saas/frontend/src/pages/password-reset/PasswordResetRequestPage.tsx:40`
- `expense-saas/frontend/src/pages/password-reset/PasswordResetPage.tsx:53`

### 設計との乖離

設計書 `dev-journal/deliverables/docs/50_detail_design/screens/auth-password-reset-request.md`（L120, L141, L204, L221, L227）はすべて **「ログイン画面に戻る」**（助詞「に」、1 つのリンクテキスト）と規定している。

実装は **「ログイン画面へ戻る」（prefix）＋「ログイン」（label）** に分割されており、助詞も `に → へ` に変わっている。

## 影響

- ユーザーがパスワードリセット要求画面・パスワードリセット実行画面でログイン画面へ戻る際、リンク表記が不自然で意図が伝わりにくい
- 設計書に定められたテキストと実装が一致しておらず、SMK の確認基準を満たさない

## 提案

**採用方針（合意済み: A 案）**: 「ログイン画面に戻る」全体を 1 つのクリッカブルテキストにする（設計書通り）。

### 修正案 A-1: AuthNavLinks の prefix を任意化（推奨）

`prefix` を `prefix?: string`（オプショナル）にし、空または未指定の場合は label のみを表示。
`PasswordResetRequestPage` / `PasswordResetPage` の links 配列を `{ label: 'ログイン画面に戻る', to: '/login' }` に変更。

メリット: コンポーネント側を最小変更、他画面パターン（リード文＋クリッカブル名詞）も維持
デメリット: prefix の用途分岐がコンポーネント内で発生

### 修正案 A-2: link-only の独立した呼び出し

`AuthNavLinks` を変更せず、PasswordReset 系画面では `MuiLink` を直接使用して「ログイン画面に戻る」のみ表示。

メリット: コンポーネントを汚さない
デメリット: パターン分散（同種ナビ用途で共通コンポーネントを使わない例ができる）

実装時は A-1 を推奨（共通コンポーネントの自然な拡張、変更範囲が局所化）。

## 修正対象ファイル

| ファイル | 変更種別 |
|---|---|
| `expense-saas/frontend/src/pages/auth/AuthNavLinks.tsx` | A-1: `prefix` を `?:` に変更し、空時は表示しない条件分岐を追加 |
| `expense-saas/frontend/src/pages/auth/__tests__/AuthNavLinks.test.tsx` | prefix なし時のテストケース追加 |
| `expense-saas/frontend/src/pages/password-reset/PasswordResetRequestPage.tsx` | links 配列を `{ label: 'ログイン画面に戻る', to: '/login' }` に変更 |
| `expense-saas/frontend/src/pages/password-reset/PasswordResetPage.tsx` | 同上 |
| `expense-saas/frontend/src/pages/password-reset/__tests__/PasswordResetPage.test.tsx` | リンクテキスト assert を「ログイン画面に戻る」に更新（既存テストがあれば） |

設計書側は修正不要（既に「ログイン画面に戻る」と記載されており、実装が乖離している側）。

## 完了条件

- パスワードリセット要求画面（`/password-reset`）の画面下部に「ログイン画面に戻る」（一語）が表示される
- パスワードリセット実行画面（`/password-reset/:token`）も同様
- リンクは押下でログイン画面（`/login`）に遷移する
- 「ログイン」が重複表示されない
- `AuthNavLinks` のテストが通る（既存パターン + prefix なしパターン）
- `PasswordResetPage` のテストが通る

## MVP 区分

MVP（UX 違和感、ユーザーが直接目にする）

## ラベル

- type: bug / ux
- area: frontend / docs（設計乖離）

## 関連 SMK

SMK-099（Phase 5、パスワードリセット要求画面の「ログイン画面に戻る」リンク確認）

## 関連設計書

- `dev-journal/deliverables/docs/50_detail_design/screens/auth-password-reset-request.md`（設計書、修正不要）
- `dev-journal/deliverables/docs/55_ui_component/screens/auth-password-reset-request.md`（UI コンポーネント設計、確認が必要）

## やらないこと

- 設計書の修正（既に「ログイン画面に戻る」と書かれているため）
- 他画面（Login / Signup）の AuthNavLinks 利用の見直し（自然な表現になっているため）

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->

---

## メタ情報

| 項目 | 値 |
|---|---|
| ID | 150 |
| 起票日 | 2026-04-25 |
| 起票者 | Claude Code（指揮役） |
