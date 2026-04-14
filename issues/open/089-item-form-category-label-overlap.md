# 明細フォームのカテゴリ欄でラベル「カテゴリ」と placeholder「カテゴリを選択」が重複表示

## 発見日
2026-04-14

## カテゴリ
implementation / frontend / ui

## 影響度
低（視覚的不具合、機能は動作）

## 発見経緯
Step 11-A ローカル動作確認 SMK-012 前提データ準備のため、明細を複数追加した際に観察

## 関連ステップ
Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし

## 概要

明細追加/編集フォーム（レポート詳細のスライドパネル内）のカテゴリ欄で、InputLabel「カテゴリ」と Select 内 placeholder「カテゴリを選択」が視覚的に重なって表示される。MUI の Select + InputLabel + placeholder 同時指定時の典型的な表示崩れ。

設計（`report-detail.md` §6 パネルレイアウト L258-259）では `[カテゴリを選択 v]` の 1 行表示のみで、InputLabel は想定されていない。

## 事実

- 実装箇所: `expense-saas/frontend/src/pages/reports/ItemForm.tsx` L134-149
  ```tsx
  <Controller
    name="categoryId"
    control={control}
    render={({ field }) => (
      <AppSelect
        name="categoryId"
        label="カテゴリ"           // ← L140
        options={categories}
        value={field.value}
        onChange={field.onChange}
        placeholder="カテゴリを選択"  // ← L144
        errorMessage={errors.categoryId?.message}
        disabled={isView || (isPending && !isView)}
      />
    )}
  />
  ```
- 観察: label「カテゴリ」が InputLabel として上部に固定表示され、Select の中に placeholder「カテゴリを選択」が表示される。両者が重なって見える
- 参考設計: `dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md:258-259`

## 修正方針

以下のいずれかを採用:
- **案 A**: `label` を外し `placeholder="カテゴリを選択"` のみにする（設計書に準拠）
- **案 B**: `placeholder` を外し `label="カテゴリ"` のみにする（MUI 標準スタイル）
- **案 C**: `AppSelect` 内部で `label` と `placeholder` を共存させるときの見た目を CSS で修正（InputLabel を `shrink=false` にしない、または Select 内部表示を抑制）

案 A（設計書に準拠）を推奨。

## 修正対象ファイル

- `expense-saas/frontend/src/pages/reports/ItemForm.tsx` L140 または L144
- 必要に応じて `expense-saas/frontend/src/components/ui/AppSelect.tsx`（共通コンポーネント修正の場合）

## 完了条件

- 明細フォームのカテゴリ欄にラベルとプレースホルダーが重ならずに表示される
- 既存テスト（ItemForm.test.tsx, AppSelect.test.tsx）が通過する
- 設計書と実装が一致する

## 関連 issue

- 090（明細編集時プリフィルなし）— 同じ `ItemForm.tsx` 内のバグ。バンドル PR 推奨
