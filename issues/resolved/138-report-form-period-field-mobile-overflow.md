# スマホ幅（375px）でレポート作成フォームが横幅に収まらない（ReportPeriodField の flex 横並び）

## 発見日
2026-04-22

## カテゴリ
implementation / ui-design / responsive

## 影響度
中（スマホ幅でレポート作成・編集フォームが viewport を超過し、横スクロールしないと全体が見えない）

## 発見経緯
Step 11-A SMK-062（スマホ幅でのフォーム操作）実施中。DevTools でスマホ幅（375px）に設定し、`/reports/new`（レポート作成画面）を開くと、対象期間フィールド（開始日 + 終了日）が横並びのため横幅に収まらず、ページ全体が横スクロール状態になる。

## 関連ステップ
Step 5.5（UI コンポーネント設計）/ Step 10-C（レポート作成・編集）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし（機能動作は正常。スマホ UX の品質問題）

## 問題

### 現状の挙動

`ReportPeriodField.tsx` は対象期間の開始日・終了日を `display: 'flex'` で**常に横並び**に配置している。

```tsx
// ReportPeriodField.tsx L33-34
<Box sx={{ display: 'flex', alignItems: 'flex-start', gap: 1 }}>
  {/* 開始日 AppDatePicker */}
  ...
  {/* 〜 セパレータ */}
  ...
  {/* 終了日 AppDatePicker */}
</Box>
```

`AppDatePicker` は MUI の `DatePicker` + `TextField` ベースで、デフォルトの min-width が確保されるため、375px 幅では **2 つのピッカー + 区切りテキスト**が入りきらず、ページレベルの横 overflow を発生させる。

### 設計書との整合

**ui-guidelines.md § レスポンシブ対応**:
> レイアウトは MUI の `Grid` / `Stack` で構成し、ブレークポイントに応じてカラム数を変更する

→ 本実装は `Box` + `display: 'flex'` 固定で、ブレークポイント対応なし。

**smoke_check.md SMK-062 期待**:
> 入力フィールドが画面幅に収まり、キーボード表示時にも操作対象が隠れない

→ 現状は画面幅に収まっていない。

### 影響範囲

- `/reports/new` レポート作成画面（`ReportCreatePage` → `ReportForm` → `ReportPeriodField`）
- `/reports/:id/edit` レポート編集画面（`ReportEditPage` → `ReportForm` → `ReportPeriodField`）
- レポート一覧の期間フィルタでも `AppDatePicker` を横並びにしている場合は同様の影響（要別途確認）

## 修正方針

### 案 A: `Stack` でブレークポイント対応（推奨）

`Box` + `display: 'flex'` を MUI `Stack` に置換し、スマホ幅では縦積みに切り替える。区切りテキスト「〜」はスマホ幅では非表示にする。

```tsx
import Stack from '@mui/material/Stack';

<Stack
  direction={{ xs: 'column', sm: 'row' }}
  alignItems={{ xs: 'stretch', sm: 'flex-start' }}
  spacing={{ xs: 1, sm: 1 }}
>
  {/* 開始日 AppDatePicker */}
  ...
  {/* 区切りテキスト（sm 以上のみ表示） */}
  <Typography
    variant="body1"
    sx={{ mt: 1, flexShrink: 0, lineHeight: '40px', display: { xs: 'none', sm: 'block' } }}
  >
    〜
  </Typography>
  {/* 終了日 AppDatePicker */}
  ...
</Stack>
```

**効果**:
- スマホ幅（xs）: 開始日・終了日を**縦積み**で全幅表示、区切りテキスト非表示
- タブレット・PC 幅（sm 以上）: 従来通り横並び + 区切りテキスト表示
- `ui-guidelines.md` のブレークポイント方針（`Stack` 使用）に準拠

### 案 B: `AppDatePicker` 側に min-width 指定で無理矢理収める（不採用）

個々の min-width を小さくしても日付形式の文字列（YYYY-MM-DD）が切れる可能性があり、UX が劣化する。

### 推奨

**案 A**。`ui-guidelines.md` のレスポンシブ方針（`Stack` でブレークポイント対応）と整合する最小修正。

## 修正対象ファイル

- `expense-saas/frontend/src/components/report/ReportPeriodField.tsx` — `Box` → `Stack`、`direction` をブレークポイント対応、区切りテキストをスマホ幅で非表示
- （必要に応じ）既存テスト `ReportForm` 系のスナップショットを再生成

## 完了条件

- スマホ幅（375px）でレポート作成画面 (`/reports/new`) がページ全体横スクロールなしで表示される
- 対象期間の開始日・終了日が縦積みで全幅表示される
- PC 幅（1280px）では従来通り横並び + 区切りテキスト表示
- レポート編集画面 (`/reports/:id/edit`) も同様
- 既存テストが通る（React Testing Library のテストは DOM 構造に直接依存しないためほぼ影響なし）

## 関連 SMK / issue

- **SMK-062**（現在 FAIL）: 本 issue で解消対象
- **#137**（スマホ幅でレポート一覧・明細一覧がページ全体横スクロール）: 別原因（TableContainer 未使用）、本 issue とは独立。明細スライドパネルの横伸びは #137 の二次影響
- 要別途確認: レポート一覧の期間フィルタ（`ReportListPage` のフィルタ部分で `AppDatePicker` 横並び使用がある場合は同じ問題）

---

## 解決内容

PR #89 にて対応。`ReportPeriodField.tsx` の `Box` + `display: 'flex'` 固定レイアウトを MUI `Stack` に置換し、`direction={{ xs: 'column', sm: 'row' }}` によるブレークポイント対応を実装。スマホ幅（xs）では開始日・終了日を縦積みで全幅表示し、区切りテキスト「〜」は非表示。

- 解決 PR: #89
- マージ commit: 1369e50

## 解決日

2026-04-22
