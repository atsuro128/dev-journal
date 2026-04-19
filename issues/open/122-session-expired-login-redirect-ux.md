# セッション期限切れ時のサイレントリダイレクト UX 改善（post-MVP）

## 発見日
2026-04-19

## カテゴリ
ux / frontend / auth / post-mvp

## 影響度
低〜中（post-MVP）

## 発見経緯
Step 11-A ローカル動作確認（SMK-024 401 セッション切れ）の検証中、両方のトークンを無効化した際の挙動が「サイレントにログイン画面へリダイレクト」であり、ユーザーに対してセッション切れを説明するフィードバックがないことに気付いた。state-management.md §エラー分類では「自動遷移（Snackbar 不要）」と定義されており、現状の実装は設計通りだが、UX として「最低ライン」であり post-MVP で改善の余地がある。

## 関連ステップ
post-MVP（運用フェーズ）

## ブロッカー
なし

## ⚠ MVP スコープ外

本 issue は MVP スコープ外の UX 改善項目を記録するもの。MVP リリース前には対応しない。管理方式は `ops-080` で検討中。

## 現状（MVP 時点）

### 実装
- `api/client.ts` が 401 を受けると refresh_token で交換を試行
- 交換失敗時はトークンをクリアしてログイン画面にリダイレクト
- **画面遷移のみでフィードバックなし**（state-management.md §エラー分類: 「自動遷移（Snackbar 不要）」）

### 設計書の状態
- `55_ui_component/state-management.md:375`: 「自動遷移（Snackbar 不要）」← 実装準拠
- `60_test/manual_checklists/smoke_check.md` SMK-024: 「『セッションが切れました』とトースト表示」← **state-management.md と矛盾**

smoke_check.md の記述は state-management.md 策定以前の古い要件が残った不整合と判断。本 issue 対応時に smoke_check.md も合わせて更新する。

## 問題

### UX 上の懸念
- 作業中に突然ログイン画面へ飛ばされ、理由が示されない
- ユーザーが「エラーか？」「自分の操作が悪かったのか？」と不安になる
- 復帰後、直前の画面に戻れない（`return_to` 未実装）

### 業界ベストプラクティスとの乖離
主要プロダクトの一般解:
- GitHub: ログインページ上部に `Please sign in again.` フラッシュ
- Microsoft 365: リダイレクト + `Your session has expired` バナー
- AWS Console: 同様のバナー表示

共通パターン: **リダイレクト + ログイン画面内のインラインアラート**（トーストではない、画面構造に組み込まれた説明）。

### 採用しない選択肢
- 中間「セッション期限切れ」画面: 1 画面増える割に価値が薄い
- トースト: 画面構造的に不自然（トーストは操作フィードバック用途）
- 完全サイレント（現状）: 最低ライン

## 改善方針（post-MVP）

### 1. ログイン画面にインラインアラートを追加

`api/client.ts` のリダイレクト時に URL クエリ or navigation state で「session_expired」フラグを渡し、ログイン画面でフォーム上部に `<Alert severity="info">` を表示:

```
セッションの有効期限が切れました。再ログインしてください。
```

### 2. return_to の実装

リダイレクト時に現在の URL を query param 等に保持し、ログイン成功後に元画面へ復帰。  
SMK-008（未ログインリダイレクト）の期待結果にも「元の遷移先が return_to に保持される」と記載があり、こちらと設計を揃える。

### 3. 発火条件の整理
- アクセストークンのみ失効 + refresh 成功: 無言（現状維持、UX ベスト）
- refresh も失敗: リダイレクト + インラインアラート + return_to
- 意図的ログアウト: リダイレクトのみ（アラート不要）

## 影響範囲
- `expense-saas/frontend/src/api/client.ts`（401 ハンドリング）
- `expense-saas/frontend/src/pages/auth/LoginPage.tsx`（URL クエリ / state 受け取り + アラート表示）
- `expense-saas/frontend/src/stores/auth.ts`（リダイレクト経路の調整）
- `dev-journal/deliverables/docs/55_ui_component/state-management.md`（「Snackbar 不要」をインラインアラート方針へ更新）
- `dev-journal/deliverables/docs/60_test/manual_checklists/smoke_check.md` SMK-024（期待結果を新方針に揃える）

## 対応条件

- refresh 失敗時のリダイレクト先ログイン画面にインラインアラート表示
- return_to によりログイン後に元画面へ復帰
- 関連設計書（state-management.md, smoke_check.md）の整合性確保
- テスト追加（Playwright E2E: セッション切れ → ログイン画面アラート → 再ログイン → 元画面復帰）

## 関連

- ops-080: MVP スコープ外管理の方針検討中。本 issue の管理先は ops-080 決定後に確定
- SMK-008: 未ログイン時リダイレクト（return_to を前提としており、本 issue とセットで整合性確保）
- SMK-024: 401 セッション切れ（現状の smoke_check.md と state-management.md の矛盾を本 issue 対応で解消）
