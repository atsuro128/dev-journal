# ReportForm の日付フィールドで onBlur バリデーションが発火しない（Controller が field.onBlur を渡していない）

## 発見日
2026-04-19

## カテゴリ
implementation / frontend / validation / form-integration

## 影響度
中（日付関連バリデーションがリアルタイム動作せず、保存押下まで気付けない UX 劣化）

## 発見経緯
Step 11-A ローカル動作確認（SMK-022 リアルタイムバリデーション: 日付論理）で、レポート作成画面の開始日に `2026-04-30`、終了日に `2026-04-01` を入力してフォーカスを外してもエラーが表示されず、保存ボタン押下で初めて「開始日は終了日以前を指定してください」が表示される挙動を観察。タイトル欄（SMK-020 PASS 済み）は onBlur で動作しており、日付フィールド固有の問題と判明。

## 関連ステップ
Step 5（詳細設計 screens/report-create.md / report-edit.md）/ Step 10-B（レポート機能実装）/ Step 11-A

## ブロッカー
なし（submit 時の最終バリデーションは機能している）

## 問題

### 設計仕様

`50_detail_design/screens/report-create.md` / `report-edit.md`:

| V | 項目 | ルール | タイミング |
|---|------|-------|-----------|
| V3 | 期間 | 開始日は終了日以前 | **入力時（リアルタイム）** |
| V4 | 開始日/終了日 | 必須 | **入力時（リアルタイム）** |

### 実装の挙動

#### ReportForm の `useForm` 設定は正しい

`expense-saas/frontend/src/components/report/ReportForm.tsx:83-84`:
```tsx
mode: 'onBlur',
reValidateMode: 'onChange',
```

タイトル欄は `{...field}` で spread しているため `field.onBlur` が渡り、onBlur バリデーションが正しく動作（SMK-020 PASS）。

#### 日付フィールドは Controller が `field.onBlur` を渡していない

`expense-saas/frontend/src/components/report/ReportPeriodField.tsx:36-50`:
```tsx
<Controller
  name="periodStart"
  control={control}
  render={({ field }) => (
    <AppDatePicker
      name="periodStart"
      label="開始日"
      value={field.value ?? null}
      onChange={field.onChange}       // ← onChange だけ明示的に渡す
      // field.onBlur が渡っていない
      errorMessage={periodStartError}
      required
      disabled={disabled}
    />
  )}
/>
```

同様に終了日の Controller も `field.onBlur` を渡していない。

#### AppDatePicker も onBlur prop を受け取らない

`expense-saas/frontend/src/components/ui/AppDatePicker.tsx`:
```tsx
export interface AppDatePickerProps {
  name: string;
  label: string;
  value: string | null;
  onChange: (value: string | null) => void;
  errorMessage?: string;
  required?: boolean;
  disabled?: boolean;
  // onBlur prop 定義なし
}
```

結果: 日付フィールドの blur イベントが React Hook Form に届かず、onBlur バリデーションが発火しない。

### 再現手順

1. レポート作成画面を開く
2. タイトルに任意の値を入力
3. 開始日に `2026-04-30` を入力
4. 終了日に `2026-04-01` を入力してフォーカスを外す → エラー表示なし
5. 保存ボタン押下 → 「開始日は終了日以前を指定してください」が表示

## 根本原因

- `ReportPeriodField` の Controller が `field.onBlur` を子コンポーネントに渡していない
- `AppDatePicker` 自体が `onBlur` prop を受け取る定義を持っていない

タイトル欄のように `{...field}` で一括 spread する方式と対比すると、明示的に個別プロパティを渡す設計にしたことで onBlur が取りこぼされている。

## 修正方針

### 1. AppDatePicker に onBlur を追加

```tsx
export interface AppDatePickerProps {
  ...
  /** フォーカスアウト時のコールバック（RHF 連携用） */
  onBlur?: () => void;
}

export default function AppDatePicker({ ..., onBlur }) {
  return (
    <TextField
      ...
      onBlur={onBlur}
    />
  );
}
```

### 2. ReportPeriodField で field.onBlur を渡す

```tsx
<AppDatePicker
  ...
  value={field.value ?? null}
  onChange={field.onChange}
  onBlur={field.onBlur}  // 追加
  ...
/>
```

## 影響範囲

- `ReportForm` の開始日・終了日フィールド（症状確認済み）
- 他に `AppDatePicker` を Controller 経由で使っている箇所があれば同様の影響あり（要確認）
- ItemForm の日付は `register('expenseDate')` 直接なので本 issue とは独立（issue 118 で別途修正予定）

## 修正対象ファイル

- `expense-saas/frontend/src/components/ui/AppDatePicker.tsx`（onBlur prop 追加）
- `expense-saas/frontend/src/components/report/ReportPeriodField.tsx`（Controller から onBlur 伝播）
- 関連テスト（`AppDatePicker.test.tsx` があれば onBlur 伝播テスト追加、`ReportPeriodField.test.tsx` で onBlur 検証）

## 完了条件

- 開始日・終了日のいずれかにフォーカスアウトした時点で、期間論理エラー・必須エラーがリアルタイム表示される
- タイトル欄と同じ UX（onBlur で即座にエラー表示）になる
- 既存の submit 時バリデーションは引き続き通過する
- 対応するテストが追加され通過する

## 次セッション着手手順

1. ブランチ作成: `fix/119-date-field-onblur`
2. `AppDatePicker.tsx` に onBlur prop を追加
3. `ReportPeriodField.tsx` で field.onBlur を渡す
4. テスト追加（onBlur 発火 → エラー表示）
5. ローカルで SMK-022 相当の操作を再実行して視覚確認
6. PR 作成
