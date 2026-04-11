# auth store に currentUser を保持しており state-management.md と乖離

## 発見日
2026-04-11

## カテゴリ
implementation

## 影響度
低

## 発見経緯
review

## 関連ステップ
Step 10（機能実装）

## ブロッカー
なし

## 問題

state-management.md では「ユーザー情報（name, email, role, tenant）はサーバー状態として TanStack Query の `['auth', 'me']` クエリで管理する」と定義されている。

しかし実装では `stores/auth.ts` にモジュール変数 `currentUser` を保持し、`setCurrentUser` / `getCurrentUser` で同期的にアクセスしている。

### 使用箇所
- `api/auth.ts`: `/api/auth/me` 取得後に `setCurrentUser` で保存
- `components/layout/AppLayout.tsx`: `getCurrentUser()` でユーザー情報を同期取得
- `hooks/useAuth.ts`: `getCurrentUser()` でロール情報を同期取得

### 設計との相違
- `useCurrentUser` Hook（TanStack Query）は存在し動作している
- store の `currentUser` は初回ロード時の同期アクセス用（TanStack Query のローディング状態を回避するため）

## 影響

- `useCurrentUser` と `getCurrentUser()` の二重管理により、データの不整合が発生する可能性
- アカウント切り替え時に store のキャッシュが残る可能性

## 提案

- `AppLayout` / `useAuth` を `useCurrentUser` Hook 経由に変更
- ローディング状態は Suspense またはスケルトン UI で対応
- `stores/auth.ts` からの `currentUser` 関連を削除

対応は影響範囲（UX 変更を伴う）を考慮し、Step 11 以降で実施する。

---

## 解決内容
stores/auth.ts の currentUser 関連を削除、api/auth.ts（未使用）を削除、useAuth を { isAuthenticated } のみに簡素化。AppLayout / ReportDetailPage / ReportEditPage を useCurrentUser Hook に移行。useLogout Hook を新設しサーバー側ログアウトを実装。useLogin / useSignup に prefetchQuery 追加。PR #36 でマージ済み。

## 解決日
2026-04-11
