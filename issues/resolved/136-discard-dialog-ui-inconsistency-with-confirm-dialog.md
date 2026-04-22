# 明細編集の変更破棄ダイアログが ConfirmDialog 未使用で UI 不統一（共通コンポーネントに寄せる）

## 発見日
2026-04-22

## カテゴリ
implementation / ui-design

## 影響度
低〜中（機能動作は正常だが、同種ダイアログ間でボタンスタイル・レイアウトが揃わず UX 一貫性を損なう）

## 発見経緯
Step 11-A SMK-038（添付削除 draft のみ）実施中の副次発見。添付ファイル削除確認ダイアログを操作した直後に明細編集中の変更破棄ダイアログを操作したところ、ボタンスタイル（variant）やレイアウトが揃っていないことに気付いた。

## 関連ステップ
Step 5.5（UI コンポーネント設計）/ Step 10-C/10-G（レポート詳細・添付）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし（UI 統一性の問題で動作には影響しない）

## 問題

### 現状の挙動

レポート詳細画面では 2 系統のダイアログ実装が併存している。

| ダイアログ | 用途 | 実装 | ボタンスタイル |
|-----------|------|------|--------------|
| 添付ファイル削除確認 | 添付の削除操作 | `ConfirmDialog`（共通コンポーネント）経由 | キャンセル = `outlined` / 削除する = `contained` `color=error` |
| 明細編集の変更破棄 | Drawer 閉じ時の未保存警告 | MUI `Dialog` / `DialogTitle` / `DialogContent` / `DialogActions` を直接使用 | キャンセル = 既定 `text` / 破棄 = 既定 `text` `color=error` |

添付ファイル削除確認ダイアログ側が `ConfirmDialog` 準拠で、ボタンの `variant`（`outlined` + `contained`）により視覚的に確認ボタンが強調されている。一方、明細編集の変更破棄ダイアログは MUI のデフォルト Button（`text` variant）のみで、2 つのボタンに視覚的な強弱が付かず、操作ミスを誘発しやすい。

### 該当コード

**添付ファイル削除確認（AttachmentArea.tsx L208-219）**:

```tsx
<ConfirmDialog
  open={true}
  title="添付ファイルの削除"
  message="この添付ファイルを削除しますか?"
  confirmLabel="削除する"
  confirmColor="error"
  cancelLabel="キャンセル"
  onConfirm={handleConfirmDelete}
  onCancel={handleCancelDelete}
/>
```

→ `ConfirmDialog` 経由で `Button variant="outlined"`（キャンセル）+ `Button variant="contained" color="error"`（削除する）が描画される。

**明細編集の変更破棄（ItemSlidePanel.tsx L586-608）**:

```tsx
<Dialog
  open={true}
  onClose={handleDiscardCancel}
  aria-labelledby="discard-dialog-title"
  aria-describedby="discard-dialog-description"
>
  <DialogTitle id="discard-dialog-title">変更を破棄しますか？</DialogTitle>
  <DialogContent>
    <DialogContentText id="discard-dialog-description">
      編集内容は保存されていません。破棄するとこれまでの変更が失われます。
    </DialogContentText>
  </DialogContent>
  <DialogActions>
    <Button onClick={handleDiscardCancel}>キャンセル</Button>
    <Button onClick={handleDiscard} color="error">
      破棄
    </Button>
  </DialogActions>
</Dialog>
```

→ MUI `Dialog` を直接使用。`Button` の `variant` 指定がないため text variant で描画され、ConfirmDialog と見た目が揃わない。

### 設計書との整合

`dev-journal/deliverables/docs/55_ui_component/common-components.md` L204-583 で `ConfirmDialog` は共通コンポーネントとして定義されており、用途として「提出・削除・承認・却下・支払完了・明細削除・添付削除・明細保存時の期間外警告（ITM-007）」が列挙されている。ただし**「編集中の変更破棄」は明示列挙されていない**。

ただし common-components.md の設計意図（「確認操作を担うため共通コンポーネントとして定義」）から、同種の確認ダイアログ（破棄確認）も ConfirmDialog を使うのが自然である。

### 正とする側

**添付ファイル削除ダイアログ側（ConfirmDialog 経由）が正**。明細編集の変更破棄ダイアログを ConfirmDialog に寄せる。

## 修正方針

### 案 A: ItemSlidePanel を ConfirmDialog 経由に置換（推奨）

`ItemSlidePanel.tsx` L586-608 の生 `Dialog` 実装を `ConfirmDialog` に置換する。

```tsx
{isDiscardDialogOpen && (
  <ConfirmDialog
    open={true}
    title="変更を破棄しますか？"
    message="編集内容は保存されていません。破棄するとこれまでの変更が失われます。"
    confirmLabel="破棄"
    confirmColor="error"
    cancelLabel="キャンセル"
    onConfirm={handleDiscard}
    onCancel={handleDiscardCancel}
  />
)}
```

**波及する変更**:

- `ItemSlidePanel.tsx` から `Dialog` / `DialogTitle` / `DialogContent` / `DialogContentText` / `DialogActions` の import が不要になれば削除
- `common-components.md` L583 の ConfirmDialog 用途説明に「編集中の変更破棄」を追記
- 既存テスト（`ItemSlidePanel.test.tsx`）で `getByRole('dialog')` 等の取り回しが変わる可能性 → テストコードの調整が必要

### 案 B: そのまま放置（不採用）

動作上問題なく、用途の意味論も若干違う（操作前確認 vs 離脱警告）ため残す選択肢もあるが、UX 一貫性の観点で非推奨。

### 推奨

**案 A**。理由:

- 同じ画面内での UI 統一性（ボタン variant + レイアウト）が揃う
- `ConfirmDialog` は既に破棄に必要な機能（title / message / confirmLabel / confirmColor）をすべて持つため追加実装不要
- 設計書 `common-components.md` の ConfirmDialog 用途リストに「変更破棄」を追記することで、将来の類似ダイアログ追加時にも共通コンポーネント使用が徹底される

## 修正対象ファイル

- `expense-saas/frontend/src/pages/reports/ItemSlidePanel.tsx` — 生 Dialog を ConfirmDialog に置換、不要 import 削除
- `expense-saas/frontend/src/pages/reports/__tests__/ItemSlidePanel.test.tsx` — ダイアログ要素の取得方法を調整（必要に応じ）
- `dev-journal/deliverables/docs/55_ui_component/common-components.md` — L583 の ConfirmDialog 用途説明に「変更破棄」を追記

## 完了条件

- 明細編集の変更破棄ダイアログが ConfirmDialog 経由で描画され、ボタン variant・レイアウトが添付削除ダイアログと揃う
- 既存テストが通る（ItemSlidePanel の破棄確認テスト群: ATT-FE-057〜071 相当）
- `common-components.md` の ConfirmDialog 用途に「編集中の変更破棄」が追記されている
- レポート詳細画面で明細編集 → 変更 → 閉じる → 破棄ダイアログ表示 → キャンセル/破棄 の双方が従来どおり動作する

## 関連 issue

- #108（AttachmentArea 永続化案内文の撤去）: 破棄ダイアログの実装経緯。ItemSlidePanel L109 コメントで #108 課題 2 として破棄確認ダイアログを追加した経緯が記載されている

---

## 解決内容

PR #91 にて対応。`ItemSlidePanel.tsx` の生 MUI `Dialog` 実装（L586-608）を `ConfirmDialog` コンポーネントに置換し、ボタン variant・レイアウトを添付削除ダイアログと統一した。`common-components.md` の ConfirmDialog 用途説明に「編集中の変更破棄」を追記（dev-journal コミット 049c128 に付随）。

- 解決 PR: #91
- マージ commit: 2d1b1ed

## 解決日

2026-04-22
