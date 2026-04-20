# ItemForm のリアルタイムバリデーションが動作しない（useForm の mode 未指定）

## 発見日
2026-04-19

## カテゴリ
implementation / frontend / validation / spec-violation

## 影響度
中（設計仕様違反。ユーザーが保存ボタン押下まで入力エラーに気付けない UX 劣化）

## 発見経緯
Step 11-A ローカル動作確認（SMK-021 リアルタイムバリデーション: 範囲）で、明細追加フォームの金額欄に `0` / `-1` を入力してフォーカスを外してもエラーが表示されず、保存ボタン押下で初めてエラー表示される挙動を観察。ReportForm（タイトル必須）は SMK-020 で onBlur リアルタイム動作を確認済み（PASS）のため、ItemForm 固有の不具合と判明。

## 関連ステップ
Step 5（詳細設計 screens/report-detail.md）/ Step 10-E（明細機能実装）/ Step 11-A

## ブロッカー
なし（機能はサブミット時に最終的にブロックされるため、データ整合性は担保されている）

## 問題

### 設計仕様

`50_detail_design/screens/report-detail.md §バリデーション`:

| V | 項目 | ルール | メッセージ | タイミング |
|---|------|-------|-----------|-----------|
| V1 | 日付 | 必須 | 「日付を入力してください」 | 入力時（リアルタイム） |
| V2 | 金額 | 必須 | 「金額を入力してください」 | 入力時（リアルタイム） |
| V3 | 金額 | 正の整数 | 「正の金額を入力してください」 | **入力時（リアルタイム）** |
| V4 | 金額 | 整数（小数不可） | 「円単位の整数で入力してください」 | **入力時（リアルタイム）** |
| V5 | カテゴリ | 必須 | 「カテゴリを選択してください」 | 入力時（リアルタイム） |
| V6 | 摘要 | 必須 | 「摘要を入力してください」 | 入力時（リアルタイム） |
| V7 | 摘要 | 500文字以内 | 「摘要は500文字以内で入力してください」 | **入力時（リアルタイム）** |

すべての明細バリデーションがリアルタイム動作を要求している。

### 実装の挙動

`expense-saas/frontend/src/pages/reports/ItemForm.tsx:85-99`:

```tsx
const {
  register,
  control,
  handleSubmit,
  formState: { errors },
} = useForm<ItemFormValues>({
  resolver: zodResolver(itemFormSchema),
  defaultValues: defaultValues ?? { ... },
  // mode 未指定 → デフォルトの 'onSubmit'
});
```

React Hook Form の `useForm` は `mode` 未指定時デフォルトで `'onSubmit'`。この場合、バリデーションは `handleSubmit` 発火時のみ実行され、フィールドの onBlur / onChange ではエラーが評価されない。

### 対比: ReportForm は正しく設定されている

`expense-saas/frontend/src/components/report/ReportForm.tsx:79-85`:

```tsx
} = useForm<ReportFormValues>({
  resolver: zodResolver(reportFormSchema),
  defaultValues: defaultValues ?? DEFAULT_VALUES,
  // V1/V3/V4: フォーカスアウト時にバリデーション、V2: 入力時（リアルタイム）
  mode: 'onBlur',
  reValidateMode: 'onChange',
});
```

SMK-020（タイトル必須）は onBlur で正しくエラー表示されることを確認済み。

### 再現手順

1. Member ログイン → draft レポート詳細 → 「+ 明細追加」
2. 金額欄に `0` を入力してフォーカスアウト → エラー表示なし
3. 金額欄に `-1` を入力してフォーカスアウト → エラー表示なし
4. 保存ボタン押下 → ここで初めて「正の金額を入力してください」が表示

### 付随観察: `type="number"` で `e` が入力可能

金額欄で `e` を入力できる（`1e5` 等の指数表記）。HTML `<input type="number">` の仕様上許容されるが、設計は「整数のみ」を要求。Zod スキーマ `.int().positive()` は submit 時に弾くが、入力段階での抑止がない。

## 根本原因

`ItemForm` で `useForm` 設定時に `mode` / `reValidateMode` の指定が漏れている。

## 修正方針

### 必須: mode 設定を追加

`ReportForm` と揃える:

```tsx
useForm<ItemFormValues>({
  resolver: zodResolver(itemFormSchema),
  defaultValues: defaultValues ?? { ... },
  mode: 'onBlur',
  reValidateMode: 'onChange',
});
```

これで V1〜V7 すべてがリアルタイム動作する。

### 検討: `e` 入力の抑止（別論点、本 issue 内で扱うか要判断）

- オプションA: `inputMode="numeric"` + `onKeyDown` で `e`, `-`, `.` を抑止
- オプションB: type="text" + pattern + custom parse
- オプションC: 現状維持（submit 時の Zod `.int()` で弾く）

推奨: 本 issue は mode 設定のみ対応、`e` 抑止は別 issue で扱うか post-MVP 対応。

## 修正対象ファイル

- `expense-saas/frontend/src/pages/reports/ItemForm.tsx`（`useForm` に `mode: 'onBlur'`, `reValidateMode: 'onChange'` 追加）
- `expense-saas/frontend/src/pages/reports/__tests__/ItemForm.test.tsx`（onBlur でエラー表示されるテストを追加 / 既存テストの確認）

## 完了条件

- 金額に `0`, `-1`, `1.5`, 空文字のいずれを入力してフォーカスアウトしても対応するエラーメッセージが即座に赤字で表示される
- 日付未入力・カテゴリ未選択・摘要未入力・501文字以上の摘要でも同様にリアルタイム表示される
- 既存の onSubmit バリデーションも引き続き通過する
- ItemForm 単体テストに onBlur バリデーションケースを追加

## 次セッション着手手順

1. ブランチ作成: `fix/118-item-form-realtime-validation`
2. `ItemForm.tsx` に `mode: 'onBlur'`, `reValidateMode: 'onChange'` を追加
3. `ItemForm.test.tsx` に各フィールドの onBlur エラー表示テストを追加
4. ローカルで SMK-021 相当の操作を再実行して視覚確認
5. PR 作成
