# ReportDetailPage の削除/提出/承認等アクション系エラーが画面に表示されない（itemApiError の表示経路の欠落）

## 発見日
2026-04-22

## カテゴリ
implementation / ui-design

## 影響度
中（アクション系 API エラーが画面に表示されず、ユーザーが失敗に気付かない）

## 発見経緯
auto-test（#134 PR #84 のローカルテスト実行中に、「明細削除で FORBIDDEN エラー時に err.message が画面に表示される」テストが失敗し、実装経路を調査した結果、既存バグであることが判明）

## 関連ステップ
Step 10（機能実装）/ Step 11-A（ローカル動作確認）/ issue #134（FE エラーハンドラ統一）

## ブロッカー
なし（#134 PR #84 は案 B でテスト側を mock 検証に切り替えて先行マージ予定、本 issue は後追い実装として Step 11-D 前までに解消）

## 問題

### 現状の挙動

`ReportDetailPage.tsx` の明細削除・提出・承認等アクション系 onError はすべて `setItemApiError(message)` を呼んでいるが、`itemApiError` は `ItemSlidePanel` の `apiError` prop としてのみ画面に渡されている（L584）。

```tsx
// ReportDetailPage.tsx:584
<ItemSlidePanel
  apiError={itemApiError}
  ...
/>
```

`ItemSlidePanel` は `panelState !== 'closed'` のときだけ open になるので（L571）、**パネルが閉じている操作（明細削除ダイアログ → 削除するボタン押下、レポート提出・承認・支払操作）でのエラーは state は更新されるが画面には一切表示されない**。

### 該当経路（#134 修正後）

| 箇所 | 処理 | 画面表示経路 |
|------|------|------------|
| `handleDeleteItemConfirm`（L413-429） | 削除 mutate onError で `setItemApiError(message)` | **見えない**（パネル閉） |
| `handleSubmitConfirm` 系（L429-480 周辺） | 提出 mutate onError で `setItemApiError` | **見えない** |
| `handleApproveConfirm` 系（L480-510 周辺） | 承認 mutate onError で `setItemApiError` | **見えない** |
| `handleItemSubmit`（L434-480） | 追加/更新 mutate onError で `setItemApiError` | 見える（パネル開） |

追加・更新はパネル開のため表示されるが、削除・提出・承認等は panel 閉の操作なので `itemApiError` を表示する術がない。

### #134 との関係

#134 の修正は「onError で err.message をそのまま使う」ことが目的で、**表示経路自体は既存のまま維持**している。つまり本 issue は #134 以前から存在していた**既存バグ**で、#134 修正時に新規追加された回帰テスト（「明細削除で FORBIDDEN エラー時に err.message が表示される」）で初めて顕在化した。

### 再現手順

1. 明細削除ボタン押下 → 確認ダイアログ「削除する」押下
2. サーバー側で 403 FORBIDDEN が返る状況を再現（権限のないユーザーでの削除試行、あるいは DevTools Network で API をブロック）
3. 画面上にエラー表示がないことを確認（期待: トーストやバナーで「この操作を行う権限がありません。」表示）

提出・承認・支払でも同様。

## 影響

- UX: 中（失敗に気付かないため誤操作や重複操作を招く可能性）
- 設計整合: 中（`state-management.md` §6.5 の「ユーザー向けメッセージは `err.message` をそのまま表示する」方針に対し、**表示経路が存在しないケース**が検出漏れ）
- データ: なし
- セキュリティ: なし

## 提案

### 方針: アクション系エラーは Toast（AppToast）で表示

パネル閉状態での操作（削除・提出・承認・支払）の onError を `setItemApiError` ではなく `setToast({ open: true, severity: 'error', message })` に変更する。

### 変更スコープ

| ファイル | 変更内容 |
|---------|--------|
| `frontend/src/pages/reports/ReportDetailPage.tsx` L413-429（`handleDeleteItemConfirm`） | `setItemApiError(message)` → `setToast({ open: true, severity: 'error', message })` |
| 同上 提出/承認/支払/却下系 onError | 上記と同様に `setToast` に変更 |
| `itemApiError` state の残存 | パネル内操作（追加・更新）のみで使われるよう用途を限定。それ以外は `setToast` 経由に統一 |

### 判断論点

**論点: エラー表示の経路統一**
- **案 A (推奨)**: パネル閉操作は `setToast`、パネル開操作は `setItemApiError`（現状維持）の住み分け
  - 利点: パネル内エラーは入力値に関連するのでパネル内表示が自然、パネル外エラーは Toast が自然
  - 欠点: 2 経路が残る
- **案 B**: 全て `setToast` に統一、`itemApiError` を撤去
  - 利点: 単一経路で分かりやすい
  - 欠点: パネル内での入力エラーと混同しやすい、state-management.md §6.5 のパターンと乖離

→ **推奨: 案 A**。パネル内は既存のまま維持、パネル外の削除/提出/承認/支払/却下のみ Toast へ。

### テスト

#134 PR #84 で追加した mock 検証テスト（案 B で `expect(setItemApiError).toHaveBeenCalledWith(...)` を検証）を、本 issue 対応後に「`setToast` が期待されるメッセージで呼ばれる」形に差し替える。

## 完了条件

- 明細削除で FORBIDDEN エラー時にトーストで「この操作を行う権限がありません。」が表示される
- レポート提出・承認・支払・却下の各操作で同様にトースト表示される
- 既存の明細追加・編集のパネル内エラー表示（`itemApiError` → `apiError` prop）は変更されない
- 既存テスト通過 + 回帰テスト追加（トースト表示の検証）
- Step 11-D（横断レビュー）までに解消されている

## 関連

- **#134**（PR #84）: FE エラーハンドラのハードコード文言統一。本 issue のバグを顕在化させた回帰テストを追加したが、表示経路の修正は本 issue のスコープに分離
- `frontend/src/pages/reports/ReportDetailPage.tsx` L413-510（アクション系 onError ハンドラ群）
- `dev-journal/deliverables/docs/55_ui_component/state-management.md` §6.5（FE エラーハンドリング方針）

---

## 解決内容

PR #90 にて対応。`ReportDetailPage.tsx` のアクション系 onError（明細削除・レポート提出・承認・支払・却下）で `setItemApiError` を呼んでいた箇所を `setToast({ open: true, severity: 'error', message })` に変更し、パネル閉状態でのエラーをトーストで表示するように修正した。パネル開時の明細追加・編集エラーは `itemApiError`（`apiError` prop 経由）のまま維持。

- 解決 PR: #90
- マージ commit: 72f0df5

## 解決日

2026-04-22
