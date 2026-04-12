# App.tsx の LoginPage import がスケルトンを参照している

## 発見日
2026-04-11

## カテゴリ
implementation

## 影響度
高

## 発見経緯
user-report

## 関連ステップ
Step 10（機能実装）/ Step 11（システムテスト・UAT — 11-A ローカル動作確認）

## ブロッカー
なし

## 問題

`frontend/src/App.tsx` の import が Step 8 で作成されたスケルトン `pages/LoginPage.tsx` を参照しており、Step 10 で実装された `pages/login/LoginPage.tsx` が使われていない。

```typescript
// 現状（App.tsx:2）
import LoginPage from './pages/LoginPage';  // スケルトン（<div>LoginPage</div> のみ）

// 正しくは
import LoginPage from './pages/login/LoginPage';  // Step 10 で実装済み
```

ログイン画面にアクセスすると「LoginPage」というテキストだけが表示され、ログインフォームが表示されない。

## 影響

- ログインできないためアプリ全体が使用不能
- 他の認証画面（Signup, PasswordReset）も同様のパターンで未接続の可能性あり

## 提案

1. `App.tsx` の import パスを `pages/login/LoginPage` に修正
2. 他の認証系ページ（Signup, PasswordReset）の import も確認・修正
3. 不要なスケルトン `pages/LoginPage.tsx` を削除

---

## 解決内容

## 解決日
