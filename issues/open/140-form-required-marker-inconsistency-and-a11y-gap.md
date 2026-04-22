# フォーム必須マーカー（`*`）と HTML5 `required` 属性の不統一を全フォームで解消（ReportForm タイトルのみ `*` 表示の根本原因）

## 発見日
2026-04-22

## カテゴリ
implementation / ui-design / a11y

## 影響度
中（視覚的 UX の不整合 + HTML5 `required` 属性欠落による a11y 品質低下。機能動作は Zod バリデーションで担保されているためブロッカーではない）

## 発見経緯
Step 11-A SMK-050（ラベル・ボタンの日本語）実施中、ユーザーが「レポート作成画面でタイトルだけ `*` 付き、対象期間には付いていないのは意図通りか？」と指摘。仕様書・実装を横断調査した結果、**フォーム / コンポーネント / 設計書の三重で `required` の扱いに不統一がある**ことが判明した。

## 関連ステップ
Step 5.5（UI コンポーネント設計）/ Step 10-A（認証 UI）/ Step 10-B（レポート UI）/ Step 10-D（明細 UI）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし（バリデーションは Zod で担保、機能動作に問題なし）

## 問題

### 現状の挙動マトリクス

| フォーム / フィールド | `required` prop 渡し | 視覚的 `*` | HTML5 `required` 属性（a11y） |
|---------------------|------|----------|----------|
| **ReportForm**: タイトル | ✅ | ✅ 表示（MUI 自動付与） | ✅ あり |
| **ReportForm**: 開始日 / 終了日 | ✅ | ❌ 抑止（`AppDatePicker` の意図的抑止） | ✅ あり（slotProps で保持） |
| **ItemForm**: 日付 / 金額 / カテゴリ / 摘要 | ❌ **未指定** | ❌ 表示なし | ❌ **なし** |
| **LoginForm**: メール / パスワード | ❌ **未指定** | ❌ 表示なし | ❌ **なし** |
| **SignupForm**: 4 フィールド | ❌ **未指定** | ❌ 表示なし | ❌ **なし** |
| **PasswordResetForm**: 2 フィールド | ❌ **未指定** | ❌ 表示なし | ❌ **なし** |
| **PasswordResetRequestForm**: メール | ❌ **未指定** | ❌ 表示なし | ❌ **なし** |

→ **全フィールド必須の前提なのに、`ReportForm` だけが `required` prop を渡しており、それも視覚的に半分だけ `*` 表示される中途半端な状態**。他のフォームでは HTML5 `required` 属性すら付いておらず a11y 品質が劣る。

### コンポーネント側の挙動差

| コンポーネント | `required` prop 渡し時の挙動 |
|--------------|--------------------------|
| `AppTextField` | MUI のデフォルト（ラベルに `*` 自動付与） |
| `AppDatePicker` | `slotProps.htmlInput.required` にのみ設定し、ラベルの `*` を**意図的に抑止**（テスト互換性のため、`AppDatePicker.tsx` L63-66 コメント参照） |
| `AppSelect` | `FormControl` の `required` prop 経由でラベルに `*` 自動付与 |

→ 3 つの共通コンポーネントで `required` 挙動が異なる（`AppDatePicker` のみ抑止）。

### 設計書との対比

| 設計書 | `*` 表記 | 実装 |
|--------|---------|------|
| `report-create.md` L49, L52 | 「タイトル \*」「対象期間 \*」 | タイトルのみ ✅、対象期間 ❌ |
| `report-edit.md` | 同様と推定（未確認） | 同上 |
| `report-detail.md` L252-261 | 「日付 \*」「金額（円） \*」「カテゴリ \*」「摘要 \*」 | 全フィールド ❌ |
| `login.md` / `signup.md` / `password-reset*.md` | （要確認） | 全フィールド ❌ |

→ 設計書の想定と実装が乖離している。特に明細フォームは設計書が「全フィールド `*`」を指定しているのに、実装は全フィールド `*` なし。

### 根本原因

1. **ReportForm と ItemForm / 認証系フォームで `required` prop の渡し方が不統一**（実装者間のポリシー欠落）
2. **`AppTextField` / `AppDatePicker` / `AppSelect` でラベル側 `*` の抑止方針が不統一**（`AppDatePicker` のみテスト互換性重視で抑止）
3. **設計書は MUI の仕様差を考慮せず、全フィールドに `*` を想定**（実装上再現が困難）

### ユーザー影響

- **視覚**: レポート作成画面で「タイトル `*` だけが孤立した印象」→ ユーザーの違和感として報告された
- **a11y**: ItemForm / 認証系フォームで HTML5 `required` 属性が欠落 → スクリーンリーダーが「必須」と読み上げない、ブラウザネイティブの submit 阻止も効かない（Zod が代替で動作するので実害は軽微）
- **トレーサビリティ**: 設計書と実装が乖離 → 将来の保守性低下

## 修正方針

### 採用案: 必須マーカー（`*`）を全フォームで廃止 + HTML5 `required` 属性を統一付与

全フォーム全フィールドが必須である本プロジェクトの状態を活かし、以下を実施する:

#### 1. コンポーネント側の統一（`*` 抑止）

`AppTextField` と `AppSelect` を `AppDatePicker` と同じ抑止パターンに揃える。

**`AppTextField.tsx`**:

```tsx
export default function AppTextField({
  name,
  label,
  errorMessage,
  required,
  ...rest
}: AppTextFieldProps) {
  return (
    <TextField
      name={name}
      label={label}
      size="small"
      fullWidth
      error={!!errorMessage}
      helperText={errorMessage ?? rest.helperText}
      // required は input 要素にのみ設定し、ラベルへの `*` 付与を避ける
      // （AppDatePicker と対称化、getByLabelText 完全一致のため）
      slotProps={{
        htmlInput: required ? { required: true } : undefined,
        ...(rest.slotProps ?? {}),
      }}
      {...rest}
    />
  );
}
```

**`AppSelect.tsx`**:

- `FormControl` の `required` prop を**渡さない**（これを渡すと MUI がラベルに `*` を付与する）
- 代わりに `Select` の `inputProps` 経由で input 要素に `required` 属性を付与

```tsx
<FormControl
  size="small"
  fullWidth={fullWidth}
  error={!!errorMessage}
  // required は渡さない（FormControl がラベルに * を付与するのを防ぐ）
  disabled={disabled}
>
  <InputLabel id={labelId} shrink={shouldShrink}>{label}</InputLabel>
  <Select
    ...
    inputProps={{
      ...(required ? { required: true } : {}),
      ...(readOnly ? { readOnly: true } : {}),
    }}
  >
    ...
  </Select>
</FormControl>
```

**`AppDatePicker.tsx`**: 現状維持（既に抑止済み）

#### 2. 全フォームで `required` prop を統一付与（a11y 向上）

全フォーム全フィールドに `required` prop を渡す。視覚的変化は `ReportForm` のタイトル（`*` が消える）のみ。他フォームは視覚不変・a11y のみ向上。

**対象フォーム**:

| フォーム | 現状 | 修正 |
|---------|------|------|
| `ReportForm` | タイトル・開始日・終了日すべて `required` 付き | 変更なし（既に付与済み、視覚的には `*` が消える） |
| `ItemForm` | 全フィールド未指定 | 日付 / 金額 / カテゴリ / 摘要 に `required` 追加 |
| `LoginForm` | 全フィールド未指定 | メール / パスワード に `required` 追加 |
| `SignupForm` | 全フィールド未指定 | 全 4 フィールドに `required` 追加 |
| `PasswordResetForm` | 全フィールド未指定 | 新パスワード / 確認 に `required` 追加 |
| `PasswordResetRequestForm` | 未指定 | メール に `required` 追加 |

#### 3. 設計書を実装に合わせて修正

`*` 表記を削除する。対象:

- `dev-journal/deliverables/docs/50_detail_design/screens/report-create.md` L49, L52
- `dev-journal/deliverables/docs/50_detail_design/screens/report-edit.md`（`*` 表記があれば）
- `dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md` L252-261
- `dev-journal/deliverables/docs/50_detail_design/screens/login.md`（確認）
- `dev-journal/deliverables/docs/50_detail_design/screens/signup.md`（確認）
- `dev-journal/deliverables/docs/50_detail_design/screens/password-reset*.md`（確認）
- `dev-journal/deliverables/docs/50_detail_design/ui-guidelines.md`（必須マーカー仕様があれば）

また、**各画面設計書の「入力項目」テーブルの「必須」列は維持**（仕様情報として必要）。変更するのはレイアウト図の `*` 表記のみ。

#### 4. ユーザー向け注記（任意・追加検討）

全フォームで `*` を廃止した後、「このフォームは全項目必須です」の旨を表示するか検討。

- **案 A（推奨）**: 注記なし。バリデーションエラーが blur / submit で発火するため、ユーザーは空欄で先に進もうとした瞬間に気付く
- **案 B**: 各フォーム先頭に `<Typography variant="caption">すべての項目が必須です</Typography>` を追加

本 issue では **案 A** を採用し、注記追加は post-MVP UX polish として別途検討。

### 不採用案

#### 案 Y: 設計書を現状に合わせて「タイトルだけ `*`」を正とする

- 実装バグ（`required` prop 未指定）と視覚不統一が残る
- a11y 問題も残る
- 却下

#### 案 Z: MUI フローティングラベルから離脱して FormControl + 外部ラベル形式に全面書き換え

- スコープが肥大（全フォーム全面書き換え）
- MVP 内対応には過剰
- 却下

## 修正対象ファイル

### コード

- `expense-saas/frontend/src/components/ui/AppTextField.tsx` — `slotProps.htmlInput.required` 抑止パターンに変更
- `expense-saas/frontend/src/components/ui/AppSelect.tsx` — `FormControl` の `required` 抑止、`Select.inputProps` で付与
- `expense-saas/frontend/src/pages/reports/ItemForm.tsx` — 4 フィールドに `required` prop 追加
- `expense-saas/frontend/src/pages/login/LoginForm.tsx` — 2 フィールドに `required` prop 追加
- `expense-saas/frontend/src/pages/signup/SignupForm.tsx` — 4 フィールドに `required` prop 追加
- `expense-saas/frontend/src/pages/password-reset/PasswordResetForm.tsx` — 2 フィールドに `required` prop 追加
- `expense-saas/frontend/src/pages/password-reset/PasswordResetRequestForm.tsx` — 1 フィールドに `required` prop 追加

### テスト（要調整）

- `getByLabelText('タイトル *')` 等の `*` 付きラベル完全一致が残っていれば `getByLabelText('タイトル')` に修正
- 既存の `getByLabelText('開始日')` 等はそのまま機能（`AppDatePicker` は既に抑止済み）

grep で `getByLabelText.*\*` を検索して対象ファイルを列挙する。

### 設計書

- `dev-journal/deliverables/docs/50_detail_design/screens/report-create.md`
- `dev-journal/deliverables/docs/50_detail_design/screens/report-edit.md`
- `dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md`
- `dev-journal/deliverables/docs/50_detail_design/screens/login.md`
- `dev-journal/deliverables/docs/50_detail_design/screens/signup.md`
- `dev-journal/deliverables/docs/50_detail_design/screens/password-reset*.md`
- `dev-journal/deliverables/docs/50_detail_design/ui-guidelines.md`（必須マーカー仕様の記述があれば）

## 完了条件

- 全フォーム全フィールドで視覚的 `*` 非表示（ラベルに `*` が付かない）
- 全フォーム全必須フィールドに HTML5 `required` 属性が付与される（devtools で input を Inspect して確認可能）
- バリデーション動作は従来どおり（空入力で blur / submit 時にエラー文言表示）
- `getByLabelText` の完全一致が全フォームで機能する
- 設計書の `*` 表記が削除され、実装と整合する
- 既存テストが通る（`*` 付きラベル検索を使っていれば修正）

## 関連 SMK / issue

- **SMK-050**（ラベル・ボタンの日本語）: 本 issue 解消で「タイトルだけ `*` 付きの違和感」が解消
- **#118 / #119 / #120**: ItemForm / ReportForm のバリデーション改修系。本 issue と同じフォーム群を触るが、対象が異なる（バリデーション挙動 vs 必須マーカー表示）ため別 issue
- **post-MVP**: ユーザー向け注記（「すべての項目が必須です」）追加は別途検討（UX polish として）

## 補足: 本 issue が解消する「3 つの不整合」

1. **コンポーネント間の不整合**（`AppTextField` / `AppDatePicker` / `AppSelect` で `required` 挙動が異なる）→ 全コンポーネントで抑止パターンに統一
2. **フォーム間の不整合**（ReportForm だけ `required` 付与、他は未付与）→ 全フォームで一律 `required` 付与
3. **設計書と実装の乖離**（設計書は `*` を想定、実装は一部のみ）→ 設計書を実装に合わせて修正
