# 認証フロー UX バグ 2 件（AppLayout 未適用 + ログイン 401 のリダイレクト誤動作）

## 発見日
2026-04-13

## カテゴリ
implementation / frontend

## 影響度
高

## 発見経緯
Step 11-A ローカル動作確認の SMK-001 / SMK-002 実施中に発見

## 関連ステップ
Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
**あり** — Step 11-A の SMK-002 以降ほぼ全項目をブロックする

## 解決
- 解決日: 2026-04-13
- 解決 PR: #47（マージコミット e05ddf9）
- 対応概要:
  - §1: `PrivateRoute` + `AppLayoutOutlet` を新規作成し、`App.tsx` のネストルートで認証必須ルート全てに統一適用。`AppLayout` のレンダー中 navigate を useEffect 化、管理系ページの自前 AppLayout ラップを除去
  - §2: `api/client.ts` で `/api/auth/login` と `/api/auth/refresh` の 401 はリフレッシュ処理をスキップして即 ApiClientError を throw
- 追加テスト: `PrivateRoute.test.tsx`（3 ケース）、`api/client.test.ts`（4 ケース）
- レビュー: 内部レビュー 2 ラウンド（FIX → PASS）、codex APPROVE 相当

## 概要

Step 11-A スモークテスト着手時に、認証フローに関する 2 つの実装漏れ・実装不整合を発見した。両者ともログイン直後の体験を直接損ねるため、Step 11-A 続行前に修正が必要。

---

## §1 認証済みページに AppLayout が適用されていない

### 事実

- `expense-saas/frontend/src/components/layout/AppLayout.tsx` で `AppHeader`（ユーザーメニュー・ログアウトボタン）+ `AppSidebar`（グローバルナビ）+ メインコンテンツコンテナを統合する共通レイアウトが実装済み
- しかし `expense-saas/frontend/src/App.tsx` では Routes 配下の各ページが `AppLayout` でラップされていない
- `AppLayout` を実際に使っているのは管理系 2 画面のみ（`pages/admin/TenantPage.tsx`, `pages/admin/AllReportsPage.tsx`）
- それ以外の認証必須ページは「素のコンポーネント」が直接描画される:
  - `DashboardPage`, `ReportListPage`, `ReportDetailPage`, `ReportCreatePage`, `ReportEditPage`, `ApprovalListPage`, `PaymentListPage`
- 認証ガード（`PrivateRoute`）も存在しない

### 影響

- ヘッダー無し → ログアウトボタンに到達できず、ロール切替テストができない
- サイドバー無し → グローバルナビによるページ遷移ができない、SMK-005 / SMK-006（ロール別の非表示要素確認）が実施不能
- 認証ガード無し → 未ログインで `/dashboard` 等にアクセス可能、SMK-007 / SMK-008（403 / 未ログインリダイレクト）が実施不能
- Step 11-A の SMK-002 以降ほぼ全項目をブロック

### 修正方針

1. **AppLayout の適用**: `App.tsx` で認証必須ルート群を `AppLayout` でラップする
   - 既存パターン（TenantPage / AllReportsPage の「ページ内で `<AppLayout>...</AppLayout>` を返す」方式）と整合させるか、レイアウトルート方式（`<Route element={<AppLayoutOutlet />}>...</Route>`）に統一するかは実装エージェントが判断
   - 重複ラップ（TenantPage / AllReportsPage の自前 AppLayout）が発生しないよう注意
2. **PrivateRoute の追加**:
   - 未ログイン時は `/login` にリダイレクト、`location.state.from` に元 URL を保持（SMK-008 の return_to 要件）
   - ロール不一致時は `/dashboard` にリダイレクト + 403 トースト（SMK-007）
   - 認証必須ルートに適用
3. **AppLayout の navigate 動作確認**:
   - 現在 `AppLayout.tsx:64` の未ログイン時リダイレクトはレンダー中に `navigate` を呼んでおり React の警告対象。`useEffect` に移すか PrivateRoute に責務を移す

### 参照
- 設計: `dev-journal/deliverables/docs/50_detail_design/screens/`（ヘッダー・サイドバー仕様）
- screens.md §4.1 §4.2（認証済みレイアウト）
- 認可: `dev-journal/deliverables/docs/50_detail_design/authz.md`

---

## §2 ログイン誤入力時にエラーメッセージが一瞬で消える

### 事実

- `expense-saas/frontend/src/api/client.ts` L77-101 の `apiClient` が **すべての 401 レスポンス**で `refreshAccessToken()` を起動する
- 未ログイン状態で `POST /api/auth/login` に誤った認証情報を送ると:
  1. API が 401 を返す
  2. `apiClient` がアクセストークン期限切れと誤解して `doRefresh()` を呼ぶ
  3. 未ログインのためリフレッシュトークンが存在せず、`client.ts:23` `window.location.href = '/login'` で **ハードリロード**
  4. `LoginPage` が再マウントされ `apiError` ステートが消失
  5. 結果として「メールアドレスまたはパスワードが正しくありません」が一瞬出てすぐ消える

### 影響

- 認証失敗時のフィードバックが事実上機能しない
- SEC-011（認証失敗の統一メッセージ）の可視性が損なわれる
- ユーザーが原因不明のままログイン画面に戻される（UX 重大）
- SMK-029（エラー表示の自動消去）の検証も阻害される

### 修正方針

`apiClient` で **ログインエンドポイント `/api/auth/login` の 401 はリフレッシュ処理をスキップ**して呼び出し元（`useLogin`）にそのまま例外を投げる。

```ts
// 例
if (res.status === 401 && !path.startsWith('/api/auth/login')) {
  // 既存のリフレッシュ処理
}
```

`/api/auth/refresh` 自体の 401 も同様にスキップ対象（無限ループ防止）。

### 参照
- 設計: `dev-journal/deliverables/docs/55_ui_component/state-management.md`（apiClient の 401 ハンドリング設計）
- セキュリティ: `dev-journal/deliverables/docs/50_detail_design/security.md` SEC-011

---

## 修正対象ファイル（推定）

- `expense-saas/frontend/src/App.tsx` — ルーティングのレイアウト統合 + PrivateRoute 適用
- `expense-saas/frontend/src/components/auth/PrivateRoute.tsx` — 新規作成
- `expense-saas/frontend/src/components/layout/AppLayout.tsx` — レンダー中 navigate の useEffect 化
- `expense-saas/frontend/src/api/client.ts` — ログインエンドポイントの 401 リフレッシュ除外
- `expense-saas/frontend/src/pages/admin/TenantPage.tsx` / `AllReportsPage.tsx` — 自前 AppLayout 重複の整理（ラップ方式統一時）
- 関連テストの更新（既存テストが AppLayout 前提でない場合の修正）

## 完了条件

- ログイン後のすべての認証必須画面でヘッダー・サイドバーが表示される
- ヘッダー右上のユーザーメニューからログアウトできる
- 未ログインで `/dashboard` を直接開くと `/login` にリダイレクトされ、ログイン後に元 URL に戻る
- 誤った認証情報でログインしたとき、「メールアドレスまたはパスワードが正しくありません」が画面上に残り続ける（手動で消すか、画面遷移するまで）
- 既存テストが通る（AppLayout 統合後の振る舞いに合わせて必要なテストは更新する）
- Step 11-A SMK-001 が再実施可能、SMK-002 以降のロール切替テストに進める
