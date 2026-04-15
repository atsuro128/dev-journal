# AppSelect の outlined 切り欠きとラベル位置が不整合（displayEmpty を無条件 true にしている）

## 発見日
2026-04-15

## カテゴリ
implementation / frontend / ui

## 影響度
低（視覚的な違和感のみ。機能には影響しない）

## 発見経緯
Step 11-A Phase 3 SMK 再検証中、SMK-012（明細スライドパネル）で ItemForm のカテゴリフィールドを目視確認した際、「ラベルが浮き上がっていない（フィールド内側にいる）状態でも、outlined フィールドの上辺に切り欠き（notched outline）が入っている」ことに気付いた。

## 関連ステップ
Step 8-7（共通 UI コンポーネント実装）/ Step 10（機能実装）/ Step 11-A

## ブロッカー
なし（後追い改善事項）

## 概要

`expense-saas/frontend/src/components/ui/AppSelect.tsx` で、MUI `<Select>` に対し `displayEmpty` を **無条件で true** に設定している一方、`<InputLabel>` 側に `shrink` を指定していない。これにより以下の不整合が発生する。

- `displayEmpty=true` → MUI の OutlinedInput が「表示要素あり」と判断し、`notched=true`（ラベル幅ぶんの切り欠きが上辺に常に開く）
- `<InputLabel shrink>` 未設定 → 値が空（`value=""`）のときラベルはフィールド内側（非 shrink 位置）に降りる
- 結果: **ラベルは内側にいるのに上辺の切り欠きは空いている**、という見た目の不整合

さらに、`displayEmpty` は本来 placeholder MenuItem を表示するための機能だが、`AppSelect` のテンプレート的利用箇所（例: `ItemForm`）は `placeholder` prop を渡していないため、内部の empty MenuItem 自体生成されない（line 97-101 の条件分岐で false）。**displayEmpty が空回りしている**状態。

## 事実

### 該当コード

`expense-saas/frontend/src/components/ui/AppSelect.tsx` L83-107（抜粋）:

```tsx
<FormControl size="small" fullWidth={fullWidth} error={!!errorMessage} required={required} disabled={disabled}>
  <InputLabel id={labelId}>{label}</InputLabel>  {/* shrink 指定なし */}
  <Select
    labelId={labelId}
    id={name}
    name={name}
    value={value}
    label={label}
    onChange={handleChange}
    displayEmpty                                  {/* ← 無条件 true */}
    SelectDisplayProps={selectDisplayProps}
  >
    {placeholder && (
      <MenuItem value=""><em>{placeholder}</em></MenuItem>
    )}
    {options.map((option) => (
      <MenuItem key={option.value} value={option.value}>{option.label}</MenuItem>
    ))}
  </Select>
  {errorMessage && <FormHelperText>{errorMessage}</FormHelperText>}
</FormControl>
```

### 観測される画面

`ItemForm` のカテゴリフィールド（`expense-saas/frontend/src/pages/reports/ItemForm.tsx` L133-148）。
新規明細作成時（`categoryId=""`）、ラベル「カテゴリ」がフィールド内側にあるにもかかわらず、フィールド上辺の outlined 枠線が「カテゴリ」の幅ぶん途切れている。

## 修正方針

### 案 A（推奨・最小修正）: displayEmpty を placeholder 指定時のみ有効化

```tsx
displayEmpty={!!placeholder}
```

- placeholder なし（ItemForm 等）: 通常の MUI Select 挙動 → 値未選択時はラベルが内側、切り欠きなし
- placeholder あり: 従来どおり displayEmpty 有効

ただし placeholder ありの利用箇所も改めて検証し、`<InputLabel shrink>` 追加が必要かを確認すること。MUI 的には `displayEmpty` を使うなら `<InputLabel shrink>` を併用するのが正攻法。

### 案 B（フル対応）: displayEmpty / shrink / notched を整合させる

```tsx
const shouldDisplayEmpty = !!placeholder;
<InputLabel id={labelId} shrink={shouldDisplayEmpty || undefined}>{label}</InputLabel>
<Select ... displayEmpty={shouldDisplayEmpty}>
```

`shrink={undefined}` にしておけば MUI のデフォルト挙動（focus / value 連動）に戻る。

## 影響範囲

`AppSelect` を利用している全画面のカテゴリ・ステータス・ロール選択フィールド。
影響は視覚的なものに限られるが、利用箇所が広いため修正後はビジュアル回帰確認が必要。

## 修正対象ファイル

- `expense-saas/frontend/src/components/ui/AppSelect.tsx`
- 必要に応じて `placeholder` を渡している既存利用箇所（フィルタ系）の挙動再確認

## 完了条件

- ItemForm のカテゴリフィールドで「ラベル位置と切り欠きの状態が一致」する
- `placeholder` を渡すフィルタ系の利用箇所で従来どおりプレースホルダーが表示される
- `AppSelect.test.tsx` がもしあれば通過する。なければ最低限の表示テストを追加する

## 関連
- PR #51（issue 089/090/091/092 対応）はカテゴリラベル重複と placeholder 削除が論点だったが、この notched 不整合は別の事象で PR #51 のスコープ外
