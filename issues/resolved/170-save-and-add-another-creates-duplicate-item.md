# 「保存して続けて追加」ボタン押下で同じ明細が 2 件登録される（明細追加モーダル、データ整合性 blocker）

## 発見日
2026-05-05

## カテゴリ
bug / frontend / data-integrity

## 影響度
高（blocker — データ整合性。ユーザーが気づかず重複申請してしまう）

## 発見経緯
Step 11-A SMK-052 再検証中にユーザーが副次発見。新規レポート作成画面（`/reports/new`）の明細追加モーダルで「保存して続けて追加」ボタンを押下すると、同じ明細が 2 件登録される。

## 関連ステップ
Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
あり（データ整合性に関わる。ユーザー業務として重複明細が登録されることで経費精算金額が誤って増える可能性）

## 問題

### 現象

新規レポート作成中（`/reports/new`）の明細追加モーダルで:

1. 金額・カテゴリ・日付・摘要を入力
2. 「**保存して続けて追加**」ボタンを押下
3. **同じ明細が 2 件登録される**

### 観察事実

- **再現性**: 100%
- **成功トースト**: 1 回のみ表示（FE の onSuccess は 1 回しか発火していない）
- **リロード後**: 重複表示が残存 → **DB に 2 件 INSERT されている**
- 「保存」ボタン（モーダルを閉じるパターン）の挙動は未確認（要再検証）

### 期待動作

- 「保存して続けて追加」押下 → 1 件のみ登録 → モーダルがリセット → 続けて入力可能になる

### 推定原因

トーストは 1 回だが DB は 2 件 = **FE の onSuccess は 1 回しか呼ばれていないが、API リクエストは 2 回飛んでいる**可能性が高い。

候補:
- ボタンの `onClick` と form の `onSubmit` の両方が発火（multiple handlers wired to same event）
- TanStack Query / `useMutation` の二重呼び出し（dedup されていない）
- React 18 StrictMode 起因の useEffect 二重発火 + mutate 副作用
- `onClick` 内で `mutate` を呼びつつ、`type="submit"` も付いていてフォーム送信もトリガーされている

要 DevTools Network タブで POST `/api/reports/{reportId}/items` リクエスト数を確認して切り分け。

### 影響範囲

- 新規レポート作成（`/reports/new`）の明細追加モーダル「保存して続けて追加」ボタン経路
- 既存 draft レポート編集（`/reports/{id}/edit`）の明細追加モーダルでも同じボタンが存在する場合は同根（要確認）

## 影響

- ユーザーが気づかず重複明細を登録 → レポート合計金額が誤った値になり、承認・支払処理に影響
- ユーザーが気づいた場合でも、削除操作の追加負担
- データ整合性の信頼性低下（業務系 SaaS としての品質問題）

## 提案

### Step 1: 実装の現状把握

- `expense-saas/frontend/src/pages/reports/ItemForm.tsx`（明細追加フォーム）の「保存して続けて追加」ボタンのイベントハンドラ実装を確認
- `useMutation` 呼び出し経路の特定（onClick 直接 vs onSubmit 経由）
- DevTools Network タブで実際の POST リクエスト数を確認して二重発火経路を特定

### Step 2: 修正

- 重複イベント発火の原因に応じて以下のいずれか:
  - `<button type="button">` への変更（form submit 由来なら）
  - `onClick` 内の `e.preventDefault()` + `e.stopPropagation()`
  - `useMutation` の `isPending` 中はボタン disabled で物理的に二重押下を防ぐ
  - `mutateAsync` を使い結果を await してから次のリセット処理に進む

### Step 3: 回帰テスト

- Vitest で「保存して続けて追加」押下後に `mutate` が 1 回のみ呼ばれることを検証
- E2E で 1 回押下後に DB に 1 件のみ登録されることを検証（Step 11-C で実装予定）

## 修正対象候補ファイル

- `expense-saas/frontend/src/pages/reports/ItemForm.tsx` または `ItemEditPanel.tsx`（実装場所要確認）
- 対応する `*.test.tsx`

## 完了条件

- 「保存して続けて追加」ボタン押下で 1 件のみ DB 登録される
- 編集モードでも同様（要確認）
- 回帰テスト追加

## 関連 SMK

- SMK-052 副次発見（2026-05-05 再検証時）
- 既存の SMK 観点には含まれていないため、Step 11-C の E2E 設計時に新規ケース追加を検討

## 解決ログ

### 解決日
2026-05-05

### PR
- PR #132（マージコミット: `6bcbc2b`）
- ブランチ: `step11/170-save-and-add-another-duplicate-item`

### 原因
issue #115 blocker 2 対応コミット（`a83a87d`）で「明細作成 POST」を `ReportDetailPage`（親）から `ItemSlidePanel`（子）に移管した際に、親側の `handleItemSaveAndContinue` 内の `createItem.mutate` 呼び出しを削除し忘れた + 子の `handleSaveAndContinue` 内に `onItemSaveAndContinue` 優先呼び出し分岐を残したため、mode='add' の「保存して続けて追加」で **POST が 2 回**発火していた。トーストは 1 回しか見えていなかった（`apiToast` と `toast` の重複）ため発覚が遅れ、SMK-052 の実機再検証で副次発見された。

### 修正内容
- `ReportDetailPage.handleItemSaveAndContinue` 関数を削除（2 回目 POST の発生源）
- `<ItemSlidePanel onItemSaveAndContinue=...>` prop を削除
- `ItemSlidePanel.handleSaveAndContinue` の `onItemSaveAndContinue` 分岐を削除し `onSaveAndContinueProp()` 一本化
- `formKey` インクリメントを `ReportDetailPage` の `onSaveAndContinue` コールバックに移植（フォーム再マウント維持）
- 回帰テスト追加（ITM-FE-109 / ITM-FE-110）: `ReportDetailPage` 全体をマウントし、添付なし / 添付ありの両経路で `POST /api/reports/:id/items` が 1 回のみ呼ばれることを検証
- 旧バグ再現確認実施（旧実装に戻して FAIL → 新実装で PASS）
- テスト設計書 `dev-journal/deliverables/docs/60_test/test_cases/items.md` §12.3 に新規 ID（ITM-FE-109/110）を追記

### レビュー
- 内部レビュー（reviewer エージェント）: PASS
- codex レビュー: 初回 REQUEST CHANGES（回帰テストが旧バグ再現条件を満たしていない）→ 対応後 APPROVE 相当
- ローカル CI: FE lint / tsc / test（786 tests）/ build すべて PASS

### 完了条件の達成
- ✅ 「保存して続けて追加」ボタン押下で 1 件のみ DB 登録される
- ✅ 編集モードはボタン非表示（`ItemForm.tsx:336` `isAdd && onSaveAndContinue`）のため影響外
- ✅ 回帰テスト追加（ReportDetailPage 経由の結合テスト 2 ケース）
