# 403 認可エラー時にフィードバックが表示されない（SMK-007 FAIL）

## 発見日
2026-04-13

## カテゴリ
implementation / frontend / ux

## 影響度
中（非ブロッカー、UX 改善）

## 発見経緯
Step 11-A ローカル動作確認 SMK-007（認可エラー時のリダイレクト）で、Member が `/settings/tenant` を直接 URL で開いた際、ダッシュボードへ黙ってリダイレクトされることを確認。smoke_check.md SMK-007 は「(403 画面 or トースト) AND リダイレクト」を要求しており、仕様乖離のため FAIL 判定。

## 関連ステップ
Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし（セキュリティ・機能は OK。UX のみ NG）

## 概要

Member ロールで `http://localhost:5173/settings/tenant` を直接開くと、ダッシュボード（`/`）へ黙ってリダイレクトされる。403 画面もトーストも表示されず、ユーザーは「なぜリダイレクトされたか」を理解できない。

- **セキュリティ**: OK（コンテンツリークなし）
- **機能**: OK（リダイレクト動作）
- **UX**: NG（ユーザーに理由が伝わらない）

smoke_check.md SMK-007 の期待結果:
> 「(403 画面 or トースト) AND リダイレクト」

## 事実

- 再現手順: Member ロール（`test-member@example.com`）でログイン → URL バーに `/settings/tenant` を直接入力 → ダッシュボードへ遷移、フィードバック無し
- 実装根拠: `expense-saas/frontend/src/pages/admin/TenantPage.tsx` L30-34 の useEffect が `navigate('/')` を呼ぶのみで、トースト発火処理がない
- 他の管理系ページ（admin 配下）でも同様の可能性あり

## 修正方針

1. **PrivateRoute または AppLayout で共通ハンドラを追加**: 認可エラー検知時に「この画面にアクセスする権限がありません」トーストを発火
2. **または各管理系ページの useEffect でトースト発火**: TenantPage、UserListPage、CategoryPage 等で個別対応
3. 共通化を優先し、管理系ページが新規追加されたときに漏れないようにする

## 修正対象ファイル（推定）

- `expense-saas/frontend/src/pages/admin/TenantPage.tsx`
- `expense-saas/frontend/src/pages/admin/UserListPage.tsx`（もし同等の構造があれば）
- `expense-saas/frontend/src/pages/admin/CategoryPage.tsx`（同上）
- または共通コンポーネント（`PrivateRoute.tsx` / `AppLayoutOutlet.tsx` 等）でハンドリング

## 完了条件

- Member が `/settings/tenant` を直接開くと、ダッシュボードへリダイレクトされると同時に「この画面にアクセスする権限がありません」トーストが表示される
- 他の管理系画面でも同様の挙動が保証される
- SMK-007 が PASS になる
