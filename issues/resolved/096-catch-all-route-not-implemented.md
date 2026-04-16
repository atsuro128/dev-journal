# 未定義 URL に対する catch-all ルートが未実装（真っ白な画面）

## 発見日
2026-04-14

## カテゴリ
implementation / frontend / ux / security-hardening

## 影響度
中（UX 改善 + 認可エラーとの非対称性によるわずかな情報開示）

## 発見経緯
Step 11-A ローカル動作確認中、issue 088（403 認可エラーフィードバック）の対応文言を検討する過程で発見。「権限のない画面に直接アクセスした場合」と「定義されていない URL に直接アクセスした場合」の挙動が非対称になっており、攻撃者から区別可能な状態であることを確認。

## 関連ステップ
Step 8-3（フロントエンド初期化）/ Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし（Step 11-A の必須対応外。後追い改善事項）

## 概要

`expense-saas/frontend/src/App.tsx` の `<Routes>` に **catch-all ルート（`<Route path="*" />`）が定義されていない**。そのため、定義されていない任意の URL（例: `/admin/nonexistent`、`/foo/bar/baz` 等）に直接アクセスすると、React Router がどのルートにもマッチさせず **真っ白な画面** が表示される。

## 事実

### 実装

`expense-saas/frontend/src/App.tsx` L24-55:

```tsx
<BrowserRouter>
  <Routes>
    {/* 認証不要ルート */}
    <Route path="/login" element={<LoginPage />} />
    <Route path="/signup" element={<SignupPage />} />
    <Route path="/password-reset" element={<PasswordResetRequestPage />} />
    <Route path="/password-reset/:token" element={<PasswordResetPage />} />

    {/* 認証必須ルート */}
    <Route element={<PrivateRoute />}>
      <Route element={<AppLayoutOutlet />}>
        <Route path="/" element={<Navigate to="/dashboard" replace />} />
        <Route path="/dashboard" element={<DashboardPage />} />
        {/* ... 既存ルート定義 ... */}
        <Route path="/settings/tenant" element={<TenantPage />} />
      </Route>
    </Route>
    {/* ← ここに catch-all <Route path="*" ... /> がない */}
  </Routes>
</BrowserRouter>
```

### 観測される挙動

| URL 例 | 現状の表示 |
|--------|---------|
| `/dashboard` | DashboardPage（正常） |
| `/settings/tenant`（Member ログイン中） | TenantPage がレンダリング → useEffect で `navigate('/')`（黙りリダイレクト、issue 088 で対応予定） |
| `/admin/nonexistent` | **真っ白な画面**（catch-all なし） |
| `/foo/bar/baz` | **真っ白な画面**（同上） |

### 関連する非対称性

issue 088（403 認可エラーフィードバック）の対応により、**認可エラー**時はトースト表示 + ダッシュボードへリダイレクトされる予定。一方、**未定義 URL** は今のまま真っ白な画面。これにより:

- 「真っ白 = 未定義 URL」
- 「トースト + リダイレクト = 定義済みだが認可不足」

の差分から、攻撃者が React Router に定義されているルートを推測可能。実害は小さいが、ポートフォリオ品質の観点では UX/セキュリティ改善余地。

## 修正方針

### 1. catch-all ルートの追加

`App.tsx` の `<Routes>` 末尾に追加:

```tsx
{/* どのルートにもマッチしないパスは 404 ページを表示 */}
<Route path="*" element={<NotFoundPage />} />
```

PrivateRoute の **外側** に置くか **内側** に置くかは設計判断:

- **外側に配置**: 未ログインでも 404 ページが見える。よりシンプル。リンクから /login へ誘導する設計が必要
- **内側に配置**: 未ログインは catch-all マッチ前に PrivateRoute で /login にリダイレクト。ログイン済みのみ NotFoundPage を見る
- **推奨**: 内側配置。未ログイン時は /login 強制で挙動が一貫する

### 2. NotFoundPage コンポーネントの新設

新規ファイル: `expense-saas/frontend/src/pages/error/NotFoundPage.tsx`

```tsx
// 404 ページ。未定義 URL に対する共通エラー画面。
import { Link } from 'react-router-dom';
import Box from '@mui/material/Box';
import Typography from '@mui/material/Typography';

export default function NotFoundPage() {
  return (
    <Box sx={{ textAlign: 'center', py: 8 }}>
      <Typography variant="h4">お探しのページが見つかりません</Typography>
      <Typography variant="body1" sx={{ mt: 2 }}>
        URL が間違っているか、ページが移動された可能性があります。
      </Typography>
      <Box sx={{ mt: 4 }}>
        <Link to="/dashboard">ダッシュボードへ戻る</Link>
      </Box>
    </Box>
  );
}
```

### 3. 設計書追記

`dev-journal/deliverables/docs/50_detail_design/screens/screens.md` または `error-handling.md` 等に「未定義 URL の挙動 = 404 ページ表示」を追記。

### 4. テスト追加

- `App.test.tsx`（もしあれば）: 未定義 URL に対して NotFoundPage がレンダリングされることを検証
- `NotFoundPage.test.tsx`: 表示要素・ダッシュボードリンクの存在確認

## 修正対象ファイル

- `expense-saas/frontend/src/App.tsx` — catch-all ルート追加
- 新規: `expense-saas/frontend/src/pages/error/NotFoundPage.tsx`
- 新規: `expense-saas/frontend/src/pages/error/__tests__/NotFoundPage.test.tsx`
- 既存: `expense-saas/frontend/src/__tests__/App.test.tsx`（もしあれば）
- 設計書: `dev-journal/deliverables/docs/50_detail_design/screens/screens.md` または該当箇所

## 完了条件

- 未定義 URL（例: `/admin/xyz123`、`/foo`）にアクセスすると NotFoundPage が表示される
- ダッシュボードへの戻り導線がある
- 未ログイン時は /login にリダイレクトされる（PrivateRoute 内配置の場合）
- 既存テストが通過し、新規テストが追加されて通過する

## 関連 issue

- 088: 403 認可エラー時のフィードバック欠落 — 本 issue と合わせて対応すると、認可エラーと未定義 URL の挙動差を最小化できる（ただし 088 が案 A 採用なので完全統一にはならない）
- 後追い対応: Step 11-A 完了後に着手

## 解決

PR #59 でスカッシュマージ（2026-04-16）。
- `NotFoundPage.tsx` 新設（MUI Box + Typography + React Router Link）
- `App.tsx` に `<Route path="*" element={<NotFoundPage />} />` を PrivateRoute > AppLayoutOutlet 内末尾に追加
- テスト 3 件追加（NFP-001〜003）
- 内部レビュー PASS + codex APPROVE 相当

## 解決日
2026-04-16
