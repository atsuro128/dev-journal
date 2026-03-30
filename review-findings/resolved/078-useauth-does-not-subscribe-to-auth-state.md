# 078: useAuth が認証状態の変更を購読していない

## 指摘概要
`useAuth()` はモジュール変数をその場で読むだけで、`setTokens()` / `clearTokens()` による変更を React 側へ通知しない。これではログイン・ログアウト・リフレッシュ後に認証ガードやヘッダー表示が自動更新されず、Step 10 で使う認証状態の共通契約が未固定のまま残る。

## 根拠
- `expense-saas/frontend/src/stores/auth.ts` L1-L18 は単なるモジュールスコープ変数で、購読 API を持たない。
- `expense-saas/frontend/src/hooks/useAuth.ts` L3-L5 は `getAccessToken()` の結果を返すだけで、`useState` / `useSyncExternalStore` / Context などの React 状態管理に接続していない。
- `ai-dev-framework/guide/work-breakdown/step8-foundation.md` L202-L208 では Step 10 へ渡すものとして「API クライアント基盤（認証トークン管理含む）」を固定することを要求している。
- `dev-journal/deliverables/docs/30_arch/architecture.md` L284-L297 は認証状態管理をフロントエンド責務として定義している。

## 判定
高 / 共通契約未固定

## 修正方針案
- `stores/auth.ts` に subscribe/unsubscribe を持たせ、`useAuth()` を `useSyncExternalStore` などで接続する。
- もしくは Context か状態管理ライブラリへ寄せ、認証状態変更時に UI が再評価される方式を 8-5 の責務として固定する。
- 最低でも「何を store に保持し、どの API で購読するか」を決め、Step 10 で別方式が乱立しない状態にする。

## 対応: 対応不要

**理由:** 8-5 の責務は「API クライアント基盤」であり、トークン管理（stores/auth.ts）と API クライアント（client.ts）は正しく動作する。client.ts は getAccessToken() でトークンを取得し、Authorization ヘッダーを付与し、401 時にリフレッシュ・リトライを行う。この API 通信基盤としての契約は固定されている。

useAuth フックの React リアクティブ性（ログイン/ログアウト後の UI 自動更新）は、API クライアントの責務ではなく UI 実装の責務。Step 10 のページコンポーネント実装で認証ガード（ProtectedRoute）やレイアウトコンポーネントを作る際に、その時点で最適な方式（useSyncExternalStore / Context / 状態管理ライブラリ）を選択すべき。

「別方式が乱立する」リスクについては、Step 10 のチケットで方式を統一すれば済む問題であり、8-5 で先行して固定する必然性はない。API クライアント層とUI層の責務を混ぜると、8-5 のスコープが肥大化する。
