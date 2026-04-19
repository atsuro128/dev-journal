# 日付フィールドを空にすると Zod デフォルトエラー "Invalid input: expected string, received null" が露出する

## 発見日
2026-04-19

## カテゴリ
implementation / frontend / validation / i18n / ux

## 影響度
中（日本語 UI に英語のシステムメッセージが混入。SMK-051「エラーメッセージの自然さ」にも抵触する恐れ）

## 発見経緯
Step 11-A ローカル動作確認（SMK-022 日付論理）の過程で以下の現象を発見:

1. 開始日 > 終了日 で保存ボタンを押して「開始日は終了日以前を指定してください」エラーを表示
2. 開始日または終了日の年部分（`2026`）を backspace で消して `yyyy` プレースホルダー状態に戻す
3. エラー表示が `Invalid input: expected string, received null` に変わる

カスタム日本語メッセージ（「開始日を入力してください」）ではなく、Zod v4 のデフォルト英語メッセージがそのまま UI に表示されている。

## 関連ステップ
Step 5（詳細設計 screens/report-create.md）/ Step 10-B（レポート機能実装）/ Step 11-A

## ブロッカー
なし（機能は動作している）

## 問題

### 再現手順

1. レポート作成画面を開く
2. タイトル・開始日・終了日を入力して保存ボタン押下（任意で 開始日>終了日 にしてエラー出力）
3. 開始日の入力欄にフォーカスを当て、年部分（`2026`）に対して backspace を連打してクリア
4. エラーメッセージが `Invalid input: expected string, received null` に変化

### 根本原因

#### 1. AppDatePicker が空値を null に変換して onChange へ流す

`expense-saas/frontend/src/components/ui/AppDatePicker.tsx:39-42`:
```tsx
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
  const val = e.target.value;
  onChange(val || null);   // ← 空文字 → null
};
```

#### 2. Zod スキーマは string を期待し null を受け付けない

`expense-saas/frontend/src/components/report/ReportForm.tsx:28-30`:
```ts
periodStart: z.string().min(1, '開始日を入力してください'),
periodEnd: z.string().min(1, '終了日を入力してください'),
```

`z.string()` は null を受け付けないため、型ミスマッチで **Zod v4 のデフォルトエラーメッセージ**（`"Invalid input: expected string, received null"`）が発生する。カスタムメッセージ `'開始日を入力してください'` は `.min(1)` にのみ紐付いているため、型レベルのエラーには適用されない。

#### 3. 型整合性の問題

- `ReportFormValues = z.infer<typeof reportFormSchema>` では `periodStart: string`（非 null）
- 実行時は `null` が流れる → 型アノテーションと実体が乖離
- TypeScript のコンパイル時チェックで拾えない

## 修正方針

### 案A（推奨）: AppDatePicker が空文字を返すように統一

`AppDatePicker` の `onChange(val || null)` を `onChange(val)`（= 空文字をそのまま返す）に変更。ReportFormValues の型（`string`）と実装の動作を一致させる。

- Zod の `.min(1, '開始日を入力してください')` がそのまま発火する
- 型安全性が保たれる
- AppDatePicker の `value` prop 型も `string` に単純化できる

**注意**: null を返す仕様を前提にした呼び出し側があれば、そちらも合わせて修正が必要。要調査。

### 案B: Zod スキーマで null を許容しカスタムメッセージを付与

```ts
periodStart: z
  .string({ error: '開始日を入力してください' })
  .nullable()
  .transform((v) => v ?? '')
  .pipe(z.string().min(1, '開始日を入力してください')),
```

複雑で保守性が下がるため非推奨。

### 案C: AppDatePicker の value/onChange 型を string のみにする

案A とセット。`value: string` / `onChange: (value: string) => void` に型を厳格化し、`null` 経路を消す。型システムで再発を防止できる。

推奨: **案A + 案C の組み合わせ**。

## 影響範囲

- `AppDatePicker` を使う全ての画面:
  - `ReportForm`（レポート作成・編集）
  - その他の使用箇所は要調査（一覧画面のフィルタ日付は TextField 直接使用のため影響なし）
- `ReportFormValues` の型と実行時挙動が一致することによる副次的改善

## 修正対象ファイル

- `expense-saas/frontend/src/components/ui/AppDatePicker.tsx`（`null` 経路削除、型を `string` に）
- `expense-saas/frontend/src/components/report/ReportPeriodField.tsx`（value 型変更の影響対応）
- `expense-saas/frontend/src/components/report/ReportForm.tsx`（schema で必要に応じて補強）
- 関連テスト

## 完了条件

- 日付フィールドを空にした時のエラーメッセージが「開始日を入力してください」/「終了日を入力してください」と日本語で表示される
- 英語のデフォルトエラーメッセージが UI に露出しない
- `AppDatePicker` の value / onChange の型が `string` に統一され、null 経路が排除されている
- 関連テストが更新され通過する

## 次セッション着手手順

1. ブランチ作成: `fix/120-date-field-null-leak`
2. `AppDatePicker.tsx` の value / onChange 型を `string` に変更、`onChange(val || null)` → `onChange(val)` に修正
3. 呼び出し側（`ReportPeriodField` 等）の型エラーを解消
4. テスト更新（空値時のメッセージ検証、null を返さないことの検証）
5. ローカルで再現手順を実行し日本語メッセージ表示を確認
6. PR 作成（issue 119 と独立。修正ファイルが重複するため同一 PR 化も検討可）

## 関連 issue

- issue 119: ReportForm 日付フィールドの onBlur バリデーション不発火
  - 同じ `AppDatePicker` / `ReportPeriodField` を触るため、同一 PR でまとめて対応する選択肢もある
